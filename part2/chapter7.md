# Chapter 7. Spot, Batch, Fleet로 하는 초대형 permutation 운영

## 학습 목표

- 수천 개 규모의 짧은 permutation workload를 AWS에서 어떻게 설계할지 설명할 수 있다.
- Ansible, AWS Batch, EC2 Fleet/Spot Fleet가 각각 맡아야 할 역할을 구분할 수 있다.
- Spot interruption, quota, scheduling overhead를 고려한 실전 아키텍처를 제시할 수 있다.

## 핵심 질문

- Ansible로 수천 개 Spot 인스턴스를 직접 띄워도 되는가
- 10-20분짜리 permutation job은 Spot과 Batch에 잘 맞는가
- 2026년 기준으로는 왜 Batch나 Fleet가 더 자연스러운가

## 7.1 옛 방식의 미덕 - Ansible + Spot 인스턴스

먼저 결론부터 분명히 해 두자. `Ansible로 수천 개 Spot 인스턴스를 직접 만들어 10-20분짜리 permutation을 돌리는 구조`는 오늘도 기술적으로 가능하다. 실제로 짧고 독립적인 계산을 대량으로 분산시키는 데 Spot은 잘 맞고, Ansible은 같은 환경을 빠르게 복제하는 데 매우 유용하다. 따라서 이 방식은 "옛날이라서 틀린 방식"이 아니라, 클라우드 초창기 연구 현장에서 매우 실용적이었던 경험이다. 이 장의 목표는 그 경험을 부정하는 것이 아니라, 그 경험이 오늘날 어떤 상위 계층으로 이어졌는지를 보여 주는 데 있다.

이 구조의 장점은 직관성이다. 인스턴스를 띄우고, playbook으로 필요한 패키지를 설치하고, 각 서버에 반복 범위를 나눠 주고, 결과를 모으면 된다. 팀이 이미 SSH와 Linux 운영에 익숙하고, 한 번성 대량 실행을 빠르게 해치워야 한다면 여전히 나쁜 선택은 아니다. 특히 계산 단위가 매우 단순하고, 중앙 큐나 컨테이너 계층을 새로 배우는 비용이 더 클 때는 직접 제어가 오히려 편할 수 있다.

하지만 규모가 수천 개 수준으로 커지면, 이 구조는 곧바로 `서버를 띄울 수 있는가`의 문제가 아니라 `실패를 어떻게 다룰 것인가`의 문제로 바뀐다. 어떤 인스턴스가 중간에 회수되었는지, 어떤 범위가 아직 끝나지 않았는지, 어디서부터 재실행해야 하는지, quota 때문에 실제로 몇 대가 떠 있는지, 서로 다른 인스턴스 타입이 섞여도 결과 수집 규칙이 유지되는지 같은 질문이 더 중요해진다. 즉 Ansible + Spot 구조는 가능하지만, 오케스트레이션을 사람이 더 많이 떠안는 구조라는 점을 분명히 이해해야 한다.

## 7.2 기술적으로 가능한가 - Spot vCPU quota와 Fleet quota

수천 개 Spot 인스턴스 launch가 가능한지 묻는다면, 공식 문서는 분명히 "가능하다"고 답한다. EC2 Fleet과 Spot Fleet 문서는 tens, hundreds, or thousands of EC2 instances를 한 번의 요청으로 시작할 수 있다고 설명하며, fleet당 target capacity 기본 quota는 10,000, 리전 전체 target capacity quota는 100,000으로 제시한다 (AWS 2026a). 즉 "수천 개 인스턴스" 자체는 비정상적인 요구가 아니다. 문제는 이것이 곧바로 아무 계정에서나 기본 설정으로 가능한 것은 아니라는 점이다.

실제 병목은 대개 Spot vCPU quota와 capacity availability다. Spot quota는 인스턴스 개수로 관리되지 않고 vCPU 단위로 관리되며, 계정과 리전별로 적용된다 (AWS 2026b). 예를 들어 기본 계정에서 Standard Spot 요청 quota가 매우 낮게 시작할 수 있으므로, 1,000개의 `c` 계열 인스턴스를 바로 띄우려 하면 quota에 먼저 막힐 수 있다 (AWS 2026c). 따라서 "기술적으로 가능하다"와 "내 계정에서 지금 바로 가능하다"는 다른 문장이다. 초대형 permutation을 준비할 때는 launch template보다 먼저 현재 Spot quota와 필요한 target capacity를 계산해야 한다.

또 하나 놓치기 쉬운 점은 capacity가 항상 즉시 확보되지 않는다는 사실이다. Spot best practices 문서는 필요한 aggregate capacity가 언제나 즉시 주어지거나 계속 유지된다고 보장하지 않는다고 분명히 말한다 (AWS 2026d). 따라서 permutation workload가 Spot과 잘 맞는다고 해서, 원하는 시점에 원하는 수만큼 인스턴스를 한 번에 확보할 수 있다고 가정하면 안 된다. 다중 인스턴스 타입, 다중 가용 영역, 유연한 shard 설계가 중요한 이유가 여기에 있다.

Table 1. 초대형 Spot permutation에서 먼저 확인할 항목

| 항목 | 왜 중요한가 | 대표 확인 포인트 |
|---|---|---|
| Spot vCPU quota | 계정이 허용하는 총 Spot 규모를 결정함 | 필요한 인스턴스 총 vCPU 수가 현재 quota 안에 들어오는가 |
| Fleet target capacity quota | 한 fleet과 리전 전체에서 요청 가능한 규모를 제한함 | fleet당 10,000, 리전 전체 100,000 기본 quota 확인 |
| 가용 용량 | 실제로 어느 타입·AZ에서 Spot이 확보되는지 좌우함 | 여러 인스턴스 타입과 여러 AZ를 열어 두었는가 |
| 작업 단위 길이 | interruption에 대한 내성을 결정함 | 실패 시 다시 돌릴 shard가 충분히 짧은가 |

## 7.3 왜 요즘은 Batch나 Fleet가 더 자연스러운가

오늘날 `Ansible + 개별 Spot request`보다 `AWS Batch 또는 EC2 Fleet`가 더 자연스러운 첫 선택지인 이유는, 운영의 무게중심이 이미 인스턴스 생성보다 상위 계층으로 옮겨갔기 때문이다. AWS 자체도 `RequestSpotInstances` API를 legacy API로 보고 강하게 권장하지 않는다 (AWS 2026e). 대신 Spot best practices는 EC2 Fleet, Auto Scaling, 통합 서비스, price-capacity-optimized 전략 같은 더 현대적인 요청 계층을 권한다 (AWS 2026d). 메시지는 명확하다. Spot을 쓰지 말라는 뜻이 아니라, Spot을 직접 원시 API 수준에서 다루기보다 interruption-aware orchestration 계층과 함께 쓰라는 뜻이다.

EC2 Fleet는 직접 제어를 유지하면서도 다중 인스턴스 타입과 다중 서브넷, 다중 용량 풀을 하나의 요청으로 관리하게 해 준다. 자체 스케줄러를 가지고 있거나, 컨테이너보다 인스턴스 수준 제어를 더 중요하게 여기는 팀에게는 매우 유용하다. 반면 AWS Batch는 그 위에 잡 큐, array job, retry, timeout, 로그, Spot-backed compute environment를 얹어 주므로, 많은 연구실에게 더 높은 수준의 기본값이 된다. 쉽게 말해 Fleet는 `capacity orchestration`에 가깝고, Batch는 `job orchestration`에 가깝다. permutation처럼 실제 관심사가 "몇 대의 서버"보다 "몇 개의 반복 계산"에 있을 때는 후자가 더 자연스럽다.

또한 Batch는 인스턴스 다양화와 Spot 전략을 서비스 수준에서 흡수하기 쉬운 장점이 있다. AWS Batch 문서는 Spot 기반 컴퓨트 환경에서 `SPOT_CAPACITY_OPTIMIZED` 전략과 다양한 인스턴스 타입 허용을 권장한다 (AWS 2026f). 이 조합은 연구자가 직접 용량 풀을 손으로 고르는 부담을 줄여 준다. 결국 Batch나 Fleet가 더 권장되는 이유는 "더 멋진 서비스"라서가 아니라, 사람 손으로 처리해야 할 실패 복구와 capacity 분산을 더 구조적으로 다룰 수 있기 때문이다.

## 7.4 Array job, shard, retry, timeout으로 permutation 설계하기

Permutation workload를 현대적으로 설계할 때 가장 중요한 단위는 인스턴스가 아니라 shard다. 예를 들어 100,000회 permutation을 해야 한다면, 이를 1회 단위로 다루는 대신 100회나 500회씩 묶어 child job 하나가 일정 범위를 책임지도록 설계하는 편이 더 낫다. 이렇게 하면 스케줄링 오버헤드를 줄이고, Spot interruption이 나더라도 실패한 범위만 다시 실행할 수 있다. AWS Batch array job은 바로 이런 구조에 적합하며, array size는 2에서 10,000까지 가능하다 (AWS 2026g). 각 child job은 `AWS_BATCH_JOB_ARRAY_INDEX`를 사용해 서로 다른 random seed나 iteration range를 받도록 구성할 수 있다.

10-20분짜리 permutation job은 Batch에 잘 맞는 길이다. AWS Batch 문서는 수 초짜리 짧은 작업은 스케줄러 오버헤드가 더 클 수 있으니 필요하면 binpack해서 3-5분 이상이 되도록 권장한다 (AWS 2026h). 즉 10-20분은 "너무 짧아서 Batch에 부적합한" 길이가 아니라, 오히려 scheduler overhead와 interruption recovery 사이의 균형이 좋은 편이다. 반대로 각 permutation이 너무 짧다면, 여러 개를 한 child job 안에서 루프 돌게 묶는 설계가 더 현실적이다.

retry와 timeout은 별도로 생각해야 한다. Batch의 timeout은 잡이 너무 오래 걸릴 때 강제로 종료하기 위한 장치이고, `attemptDurationSeconds`는 각 시도에 적용된다. 공식 문서는 timeout으로 종료된 작업은 자동 retry되지 않으며, 자체 실패한 작업만 retry 정책에 따라 다시 시도될 수 있다고 설명한다 (AWS 2026i). 따라서 permutation 설계에서는 "실패"와 "멈춤"을 같은 사건으로 다루면 안 된다. 예를 들어 stuck process를 막기 위해 timeout을 걸고, Spot interruption이나 일시적 실패는 재시도 정책으로 흡수하며, 이미 완료한 shard는 S3 manifest로 체크하는 식의 분리가 필요하다.

Table 2. permutation shard 설계의 실전 원칙

| 설계 항목 | 권장 방향 | 이유 |
|---|---|---|
| child job 길이 | 대개 수 분 이상, 필요하면 10-20분 수준 | scheduler overhead를 줄이고 interruption recovery 단위를 적절히 유지 |
| 상태 저장 | S3 manifest, 로그, 결과 파일에 남김 | 인스턴스 손실 시에도 진행 상태를 복원 가능 |
| 실패 처리 | retry와 timeout을 분리 | stuck process와 일시적 실패는 성격이 다름 |
| 작업 분배 | array index 기반 seed/range 분배 | 재실행 범위를 명확하게 제한 가능 |

## 7.5 Ansible은 어디에 남는가 - provisioning과 bootstrap

그렇다면 오늘날 Ansible은 더 이상 쓸모가 없는가? 전혀 그렇지 않다. 다만 역할이 바뀌었다고 보는 편이 정확하다. Ansible은 수천 개 permutation child job의 주 오케스트레이터라기보다, VPC, IAM, launch template, AMI bootstrap, 공통 패키지 설치, 진단 스크립트 배포 같은 provisioning 층에서 여전히 강하다. 즉 `서버 환경을 준비하는 문제`에서는 Ansible이 매우 유용하고, `수천 개의 짧은 잡을 장애 복구와 함께 운영하는 문제`에서는 Batch나 Fleet가 더 적합한 경우가 많다.

이 구분은 Ansible 공식 문서에도 어느 정도 드러난다. `amazon.aws.ec2_instance` 모듈은 EC2 인스턴스 생성과 관리를 지원하지만 Spot instance 생성 자체는 지원하지 않고, 이를 위해 별도의 `amazon.aws.ec2_spot_instance` 모듈을 사용하도록 안내한다 (Ansible Community Documentation 2026a; 2026b). 즉 현대적인 Ansible workflow에서도 Spot은 AWS 계층을 직접 존중하면서 다뤄야 한다. 다시 말해, "Ansible이 있으니 AWS의 스케줄링 계층은 필요 없다"가 아니라, "Ansible은 준비와 부트스트랩에, Fleet/Batch는 실제 확장과 운영에"라는 역할 분리가 더 자연스럽다.

소규모 연구실의 일회성 대량 실행이라면, 여전히 `Ansible + Spot` 조합이 가장 빠른 해법일 수 있다. 중요한 것은 이를 보편적 기본값으로 일반화하지 않는 것이다. 반복적으로 돌아오는 대형 permutation, 장기간 유지할 재현 가능한 운영, 여러 사람이 함께 쓰는 큐 환경에서는 상위 오케스트레이션 계층의 가치가 훨씬 커진다.

## 7.6 추천 아키텍처 - 2026년 기준 실전 선택표

2026년 기준으로 가장 현실적인 구조는 세 층으로 정리할 수 있다. 첫째, `Ansible = provisioning과 bootstrap`. 둘째, `AWS Batch 또는 EC2 Fleet = 실제 확장과 interruption-aware orchestration`. 셋째, `S3 + manifest + logs = durable state와 observability`. 이 세 층을 분리하면, 인프라 준비와 실행 제어와 상태 보존이 뒤섞이지 않는다. 결국 초대형 permutation 운영의 핵심은 "서버를 많이 띄우는 기술"이 아니라, `중단되어도 이어서 갈 수 있는 계산 구조`를 만드는 일이다.

가장 일반적인 권장안은 `AWS Batch array jobs + Spot-backed managed compute environment`다. 반복적이고 재현 가능한 대규모 운영에 잘 맞고, child job 단위 재시도와 timeout, 로그 추적을 서비스 수준에서 다루기 쉽기 때문이다. 자체 스케줄러를 유지하면서 인스턴스 수준 제어를 더 세밀하게 하고 싶다면 `EC2 Fleet/Spot Fleet + 자체 오케스트레이션`이 맞을 수 있다. 반면 학습 비용을 최소화하고 빠르게 한 번 돌려야 하는 연구라면 `Ansible + Spot`도 여전히 실용적이다. 중요한 것은 가능성과 권장을 구분하는 것이다.

Table 3. 2026년 기준 permutation 운영 선택표

| 상황 | 가장 자연스러운 선택 | 이유 |
|---|---|---|
| 일회성 대량 실행, 팀 규모 작음 | Ansible + Spot | 빠르게 환경을 맞추고 직접 제어하기 쉬움 |
| 반복적이고 재현 가능한 대규모 운영 | AWS Batch array jobs + Spot-backed compute environment | queue, retry, timeout, scaling을 구조적으로 관리 가능 |
| 자체 scheduler 유지, 세밀한 capacity 제어 필요 | EC2 Fleet/Spot Fleet | 인스턴스 수준 제어와 capacity orchestration에 유리 |

이 장의 핵심 문장은 다음과 같이 정리할 수 있다. `Ansible can still provision and coordinate Spot-based infrastructure, but for launching and managing thousands of short-lived permutation jobs, AWS Batch or EC2 Fleet provides a more scalable and interruption-aware orchestration layer.` 유전체 분석자는 여기서 특정 도구의 승패를 배우는 것이 아니라, 어떤 계층이 어떤 운영 부담을 흡수하는지 배우는 것이 중요하다. 그래야 같은 원리를 permutation뿐 아니라 bootstrap, simulation, parametric sweep, cohort shard 분석에도 확장할 수 있다.

## 핵심 개념 정리

- 수천 개 Spot 인스턴스 launch 자체는 기술적으로 가능하지만, 실제 병목은 quota, capacity, failure recovery에 있다.
- Spot permutation 설계의 핵심 단위는 인스턴스가 아니라 shard와 durable state다.
- Batch는 job orchestration에, Fleet는 capacity orchestration에, Ansible은 provisioning과 bootstrap에 더 가깝다.
- 2026년 기준 권장 기본값은 대체로 `Batch 또는 Fleet 중심 + Ansible 보조` 구조다.

## 복습 질문

1. "기술적으로 가능하다"와 "기본 계정에서 바로 가능하다"는 왜 다른 말인가?
2. permutation workload에서 shard, retry, timeout을 분리해 설계해야 하는 이유는 무엇인가?
3. Batch와 Fleet는 각각 어떤 층의 오케스트레이션을 담당하며, Ansible은 어디에 남는가?

## Further Reading

- AWS. [Quotas for EC2 Fleet and Spot Fleet](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/fleet-quotas.html).
- AWS. [Array jobs](https://docs.aws.amazon.com/batch/latest/userguide/array_jobs.html).
- AWS. [Use Amazon EC2 Spot best practices for AWS Batch](https://docs.aws.amazon.com/batch/latest/userguide/bestpractice6.html).

## References

- AWS. 2026a. [Quotas for EC2 Fleet and Spot Fleet](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/fleet-quotas.html).
- AWS. 2026b. [Spot Instance quotas](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-limits.html).
- AWS. 2026c. [Amazon EC2 instance type quotas](https://docs.aws.amazon.com/ec2/latest/instancetypes/ec2-instance-quotas.html).
- AWS. 2026d. [Best practices for Amazon EC2 Spot](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html).
- AWS. 2026e. [RequestSpotInstances](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_RequestSpotInstances.html).
- AWS. 2026f. [Use Amazon EC2 Spot best practices for AWS Batch](https://docs.aws.amazon.com/batch/latest/userguide/bestpractice6.html).
- AWS. 2026g. [Array jobs](https://docs.aws.amazon.com/batch/latest/userguide/array_jobs.html).
- AWS. 2026h. [When to use AWS Batch](https://docs.aws.amazon.com/batch/latest/userguide/bestpractice1.html).
- AWS. 2026i. [Job timeouts](https://docs.aws.amazon.com/batch/latest/userguide/job_timeouts.html).
- Ansible Community Documentation. 2026a. [amazon.aws.ec2_instance module](https://docs.ansible.com/ansible/latest/collections/amazon/aws/ec2_instance_module.html).
- Ansible Community Documentation. 2026b. [amazon.aws.ec2_spot_instance module](https://docs.ansible.com/ansible/latest/collections/amazon/aws/ec2_spot_instance_module.html).
