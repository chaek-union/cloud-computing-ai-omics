# Chapter 6. 대규모 작업 자동화 - Ansible, AWS Batch, Lambda

## 학습 목표

- 반복 작업 자동화가 왜 중요한지 설명할 수 있다.
- Ansible, AWS Batch, Lambda가 각각 어떤 종류의 문제에 적합한지 구분할 수 있다.
- 오믹스 분석에서 대규모 잡 관리 전략을 개괄할 수 있다.

## 핵심 질문

- 인스턴스를 여러 대 띄워 permutation test를 돌릴 때 무엇이 가장 힘든가
- 구성 관리와 배치 실행은 어떻게 다른가
- Lambda는 어디까지 유용하고 어디서 한계가 있는가

## 6.1 수작업 운영이 무너지는 순간

오믹스 분석을 처음 AWS로 옮길 때는 한두 번의 실습이 모든 것을 단순해 보이게 만든다. 인스턴스 하나를 띄우고, 필요한 패키지를 설치하고, 스크립트를 올리고, 데이터를 S3에서 내려받아 돌리면 된다. 이 단계에서는 "클라우드가 생각보다 쉽다"는 인상이 생긴다. 그러나 같은 작업을 100번, 1,000번 반복해야 하는 순간 문제가 달라진다. 이제 병목은 CPU 수가 아니라, 누락된 패키지 버전, 서로 다른 설정 파일, 실패한 잡의 추적, 결과 수집, 다시 실행 범위 계산 같은 운영 문제로 이동한다.

대표적인 예가 permutation test다. 5분짜리 계산 하나는 사람이 직접 돌려도 어렵지 않다. 하지만 10,000번을 반복해야 하면, 그때부터 필요한 것은 "더 빠른 컴퓨터"보다 "동일한 환경을 반복 가능하게 만들고, 실패를 복구하고, 결과를 안전하게 모으는 구조"다. 이 지점에서 자동화는 편의 기능이 아니라 연구 설계의 일부가 된다. 분석이 커질수록 중요한 것은 서버를 얼마나 멋지게 띄우는가가 아니라, 같은 작업을 얼마나 일관되게 여러 번 실행할 수 있는가이다.

따라서 이 장에서는 자동화를 세 층으로 나누어 생각하는 것이 좋다. 첫째는 여러 서버의 환경을 같은 상태로 맞추는 `구성 관리(configuration management)`이고, 둘째는 많은 잡을 큐에 넣고 재시도와 우선순위를 관리하는 `배치 실행(batch execution)`이며, 셋째는 S3 업로드, 메타데이터 생성, 상태 변화처럼 계산 주변의 작은 사건을 이어 붙이는 `이벤트 자동화(event automation)`다. Ansible, AWS Batch, Lambda는 각각 이 세 층에서 강점을 가진다. 세 도구를 경쟁 관계로 이해하기보다, 서로 다른 운영 문제를 푸는 도구로 이해하는 편이 훨씬 정확하다.

## 6.2 Spot Instance로 짧고 많은 계산 싸게 돌리기

짧고 독립적인 작업을 많이 돌릴 때 가장 먼저 떠올릴 수 있는 자원이 EC2 Spot Instance다. Spot은 남는 용량을 할인된 가격으로 사용하는 방식이므로, 정렬 batch, permutation test, Monte Carlo simulation, bootstrap, 파라미터 스윕처럼 개별 작업이 독립적이고 다시 실행하기 쉬운 workload와 잘 맞는다. AWS 공식 문서도 Spot을 stateless, fault-tolerant, flexible workload에 적합한 자원으로 설명한다 (AWS 2026a). 오믹스 분석에서는 특히 샘플 단위로 쪼개기 쉬운 잡이나, 실패해도 일부 shard만 재실행하면 되는 작업이 좋은 후보가 된다.

하지만 Spot을 단순히 "싼 EC2"로만 이해하면 곤란하다. Spot은 언제든 회수될 수 있고, 일반적으로 중단 2분 전에 interruption notice가 제공된다 (AWS 2026b). 따라서 계산 상태가 인스턴스 안에만 남아 있거나, 중간 결과를 주기적으로 외부 저장소에 기록하지 않는 구조는 Spot과 궁합이 나쁘다. 반대로 입력 manifest와 결과 저장 위치가 S3에 있고, 작업 단위가 짧고 독립적이며, 실패한 조각만 다시 태울 수 있다면 Spot은 매우 강력한 선택이 된다. 여기서 핵심은 인스턴스보다 워크로드를 Spot 친화적으로 설계하는 것이다.

실전 운영에서는 세 가지 습관이 중요하다. 첫째, 작업을 가능한 한 짧고 독립적인 단위로 나눈다. 둘째, 결과와 체크포인트를 인스턴스 내부가 아니라 S3 같은 durable storage에 남긴다. 셋째, 한 종류의 인스턴스에 집착하지 않고 여러 타입과 여러 가용 영역에 유연하게 열어 둔다. Spot의 본질은 "예측 가능한 서버"가 아니라 "가용한 용량을 탄력적으로 활용하는 시장"에 가깝기 때문이다. 비용 절감은 계산 자체보다 이런 운영 원칙을 잘 지킬 때 생긴다.

## 6.3 Ansible로 환경을 반복 가능하게 만들기

Ansible의 강점은 많은 서버를 같은 상태로 만드는 데 있다. 예를 들어 50개의 단일 코어 인스턴스에 같은 R 버전과 패키지, 같은 스크립트, 같은 입력 경로를 배포하고 싶다면, 사람이 SSH로 하나씩 들어가 설정하는 방식은 금방 무너진다. Ansible은 이런 반복 작업을 playbook으로 선언해 두고 여러 인스턴스에 동일하게 적용할 수 있게 해 준다. 따라서 Ansible은 "대규모 계산을 스케줄링하는 도구"라기보다, "같은 계산 환경을 빠르게 복제하는 도구"로 이해하는 편이 맞다.

오믹스 분석에서 Ansible은 특히 bootstrap 단계에서 유용하다. 인스턴스 launch 이후 공통 패키지 설치, 환경 변수 설정, reference와 스크립트 배치, 실행 범위 분배, 로그 수집 지점 통일 같은 작업을 코드로 남길 수 있기 때문이다. 이 접근의 가장 큰 장점은 재현성이다. 서버 하나를 손으로 설정하면 그때그때는 빠를 수 있지만, 몇 주 뒤 같은 실험을 재현하거나 다른 학생이 이어받을 때 거의 항상 문제가 생긴다. 반면 playbook은 "무엇을 설치했고 어떤 순서로 설정했는가"를 문서이자 실행 코드로 남긴다.

그렇다고 Ansible이 잡 큐까지 알아서 관리해 주는 것은 아니다. Ansible로도 여러 인스턴스에 범위 기반 작업을 나눠 줄 수 있지만, 실패 재시도, 동적 스케일링, 수천 개 잡의 상태 추적, 인터럽션 대응 같은 기능은 본질적으로 구성 관리보다 상위의 오케스트레이션 문제다. 그래서 오늘날의 실전 구조에서는 `Ansible = provisioning과 bootstrap`, `AWS Batch = 잡 큐와 실행`, `S3 = 입력과 결과의 durable state`처럼 역할을 분리하는 편이 더 자연스럽다. 이 구분을 이해하면 Ansible을 과소평가하지도, 과대평가하지도 않게 된다.

Table 1. Ansible, AWS Batch, Lambda의 역할 차이

| 도구 | 가장 잘 푸는 문제 | 유전체 예시 |
|---|---|---|
| Ansible | 여러 서버에 같은 환경을 반복 배포 | permutation용 EC2에 R, 패키지, 스크립트 설치 |
| AWS Batch | 많은 잡을 큐에 넣고 재시도·스케일링 관리 | 샘플 단위 QC, Monte Carlo, containerized batch pipeline |
| AWS Lambda | 이벤트 기반의 작은 자동화 | S3 업로드 감지 후 검증, 배치 실행 시작, 알림 발송 |

## 6.4 AWS Batch로 대규모 잡 큐 운영하기

AWS Batch는 "서버를 내가 직접 배열하느냐"보다 "작업을 큐와 정책으로 관리하느냐"에 초점을 두는 서비스다. 사용자는 잡 정의(job definition), 잡 큐(job queue), 컴퓨트 환경(compute environment)을 설정하고, 실제 인스턴스 선택과 확장은 상당 부분 Batch에 맡긴다. 이 구조의 핵심 이점은 대규모 반복 작업에서 사람이 직접 인스턴스 상태를 관리하지 않아도 된다는 점이다. 즉 Batch는 EC2를 없애는 서비스가 아니라, EC2 위에 관리형 스케줄러 계층을 얹는 서비스다 (AWS 2026c).

오믹스 분석에서는 이 점이 매우 중요하다. permutation, Monte Carlo, 샘플 단위 QC, 컨테이너화된 alignment step, cohort shard 분석처럼 서로 독립적인 잡이 많은 workload는 Batch와 잘 맞는다. 특히 array job은 Monte Carlo simulation이나 parametric sweep처럼 공통 정의를 가진 병렬 작업에 효율적이며, 공식 문서상 배열 크기는 2부터 10,000까지 가능하다 (AWS 2026d). 즉 permutation 10,000회를 "서버 10,000대"로 생각하기보다, "같은 정의를 가진 자식 잡 10,000개"로 생각하는 방식으로 전환할 수 있다.

Batch의 또 다른 장점은 실패를 운영 수준에서 다룰 수 있다는 점이다. 재시도 정책, timeout, CloudWatch 로그, Spot 기반 컴퓨트 환경, 다양한 인스턴스 타입 허용 같은 기능을 조합하면, 사람이 잡 하나하나를 손으로 돌보지 않아도 된다. 이때 중요한 설계 원칙은 짧은 작업을 너무 잘게 쪼개지 않는 것이다. AWS Batch 문서는 몇 초짜리 초단기 작업은 스케줄링 오버헤드가 더 클 수 있으므로, 필요하면 여러 작업을 묶어 3-5분 이상 실행되도록 binpack하는 방식을 권장한다 (AWS 2026e). 반대로 10-20분 정도의 독립 작업은 Batch와 잘 맞는 길이다.

실전에서는 `manifest -> array index -> shard range -> S3 output` 구조가 특히 유용하다. 부모 잡은 입력 manifest와 공통 파라미터를 가리키고, 각 자식 잡은 환경 변수나 배열 인덱스를 사용해 서로 다른 샘플, seed, permutation range를 처리한다. 이렇게 하면 결과 수집과 재실행 범위를 명확히 할 수 있고, 실패한 child job만 다시 실행하기도 쉬워진다. 즉 Batch의 핵심 가치는 서버 생성 자동화가 아니라, 반복 계산을 `추적 가능한 단위`로 바꾸는 데 있다.

## 6.5 Lambda로 만드는 작은 자동화

Lambda는 종종 "서버리스니까 뭐든 할 수 있는 계산 서비스"로 오해되지만, 오믹스 분석에서의 가장 좋은 역할은 대규모 계산 엔진이 아니라 작은 접착제(glue)다. 즉 Lambda는 alignment나 variant calling을 직접 오래 수행하기보다, S3 업로드 감지, 메타데이터 검사, manifest 생성, HealthOmics나 Batch 실행 시작, 상태 알림 같은 짧은 이벤트 기반 작업에 더 적합하다. 이 차이를 분명히 이해해야 한다. 장시간 계산은 Batch, EMR, HealthOmics 같은 계층에 맡기고, Lambda는 흐름을 연결하는 쪽에 두는 편이 설계가 훨씬 깨끗해진다.

예를 들어 시퀀싱 센터가 FASTQ를 S3에 올리면, Lambda가 업로드 이벤트를 받아 파일명과 샘플 메타데이터를 검증하고, 필요한 manifest를 만들고, 그다음 Step Functions나 HealthOmics, Batch를 호출하는 흐름을 생각할 수 있다. 이 구조에서는 Lambda가 계산의 몸통이 아니라, 계산을 시작시키고 상태를 이어 주는 손잡이가 된다. 오믹스 분석 파이프라인이 커질수록 이런 작은 자동화는 체감상 매우 큰 차이를 만든다. 사람에게 메일을 보내는 것보다 먼저, 기계가 다음 단계를 준비하게 만드는 것이다.

따라서 Lambda의 한계도 함께 이해해야 한다. 길고 무거운 계산, 큰 메모리와 긴 실행 시간이 필요한 작업, 대용량 scratch를 요구하는 작업은 Lambda에 맞지 않는다. Lambda는 어디까지나 `이벤트에 반응해 짧게 실행되는 함수`라는 정체성을 가진다. 이 선을 넘기려 할수록 설계가 불안정해진다. 반대로 이 선을 정확히 지키면, 복잡한 유전체 운영에서 사람의 클릭을 크게 줄일 수 있다.

## 6.6 Batch, EKS, ParallelCluster의 선택

자동화가 깊어질수록 "그렇다면 Batch 대신 Kubernetes나 Slurm을 바로 배우는 것이 낫지 않은가"라는 질문이 나온다. 이 질문에 대한 가장 실용적인 답은, 먼저 팀이 무엇을 이미 운영하고 있는지 보라는 것이다. AWS Batch는 배치 잡을 빠르게 운영하고 싶지만 자체 스케줄러를 관리하고 싶지 않은 팀에게 잘 맞는다. Amazon EKS는 이미 Kubernetes를 플랫폼으로 쓰고 있고, 배치 작업뿐 아니라 API, UI, 노트북, 추론 서비스까지 같은 클러스터에서 운영하려는 조직에 더 자연스럽다. AWS ParallelCluster는 Slurm 중심의 전통적 HPC 운영 방식과 공유 파일시스템을 AWS로 가져오고 싶은 경우에 더 적합하다 (AWS 2026f; AWS 2026g).

초급 연구실의 첫 선택지로는 대체로 Batch가 가장 단순하다. 컨테이너화된 잡을 큐에 넣고, 필요할 때만 확장하고, Spot과 On-Demand를 섞어 쓰기 쉽기 때문이다. EKS는 강력하지만 그만큼 플랫폼 복잡도가 크다. ParallelCluster 역시 매우 유용하지만, 이는 `HPC 클러스터 운영`을 AWS로 옮기는 쪽에 가깝다. 즉 세 서비스는 우열 관계라기보다 출발점이 다르다. 현재 팀이 Kubernetes나 Slurm을 이미 중심 도구로 삼고 있지 않다면, 많은 유전체 batch workflow의 기본값은 여전히 Batch 쪽이 더 자연스럽다.

Table 2. 대규모 배치 실행 계층의 선택 기준

| 계층 | 가장 잘 맞는 상황 | 주의점 |
|---|---|---|
| AWS Batch | containerized batch workload, 반복 실행, Spot 활용 | 초단기 작업은 binpack이 필요할 수 있음 |
| Amazon EKS | 배치 외에도 다양한 서비스와 Kubernetes 운영이 필요한 팀 | 클러스터와 RBAC 운영 복잡도가 큼 |
| AWS ParallelCluster | Slurm 기반 HPC와 공유 파일시스템이 중요한 환경 | 전통적 클러스터 운영 개념을 여전히 이해해야 함 |

## 6.7 어떤 도구를 언제 선택할 것인가

지금까지의 내용을 하나로 정리하면, 도구 선택의 핵심은 "얼마나 많은 계산을 하느냐"보다 "무엇을 자동화하려 하느냐"에 있다. 여러 서버에 같은 환경을 배포하는 것이 문제라면 Ansible이 맞다. 많은 잡을 큐에 넣고 재시도와 스케일링을 관리하는 것이 문제라면 Batch가 맞다. 파일 업로드나 메타데이터 이벤트를 감지해 다음 단계를 시작시키는 것이 문제라면 Lambda가 맞다. 즉 자동화는 하나의 도구로 끝나는 일이 아니라, 문제 층에 따라 서로 다른 도구를 조합하는 일이다.

오믹스 분석의 관점에서 가장 현실적인 기본 조합은 다음과 같다. `짧고 독립적인 대량 계산은 Spot + Batch`, `여러 인스턴스의 초기 환경 맞춤은 Ansible`, `이벤트 연결과 알림은 Lambda`, `입력과 결과의 durable state는 S3`다. 이 구조는 기능을 멋지게 나눠 놓은 추상도가 아니라, 실패 복구와 재현성을 높이는 실전 구조다. 사람 손이 덜 타는 파이프라인일수록 더 큰 규모에서도 흔들리지 않는다.

다음 장의 심화편에서는 바로 이 문제를 permutation workload 관점에서 더 좁혀 본다. 즉 `Ansible + Spot`으로도 가능한 구조와, 오늘날 더 권장되는 `Batch 또는 Fleet` 중심 구조를 비교하면서, 기술적으로 가능한 것과 운영상 권장되는 것을 구분해 볼 것이다. 자동화의 진짜 성숙은 "할 수 있다"에서 끝나지 않고, "오랫동안 반복 가능하게 운영할 수 있다"로 이어질 때 완성된다.

## 핵심 개념 정리

- 자동화의 핵심 병목은 CPU보다도 환경 일관성, 실패 복구, 결과 수집, 재실행 범위 관리에 있다.
- Spot은 짧고 독립적이며 재실행 가능한 workload에 특히 강하다.
- Ansible은 구성 관리와 bootstrap에, AWS Batch는 잡 큐와 스케일링에, Lambda는 이벤트 기반 연결에 가장 잘 맞는다.
- Batch, EKS, ParallelCluster는 같은 문제를 푸는 도구가 아니라 서로 다른 운영 전제를 가진 실행 계층이다.

## 복습 질문

1. permutation test 같은 workload에서 수작업 운영이 빠르게 무너지는 이유는 무엇인가?
2. Spot Instance를 오믹스 분석에 사용할 때 반드시 함께 설계해야 하는 세 가지 운영 원칙은 무엇인가?
3. Ansible과 AWS Batch는 각각 어떤 층의 문제를 풀며, 왜 이를 구분하는 것이 중요한가?

## Further Reading

- AWS. [What is AWS Batch?](https://docs.aws.amazon.com/batch/latest/userguide/what-is-batch.html).
- AWS. [Best practices for Amazon EC2 Spot](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html).
- AWS. [What is AWS ParallelCluster](https://docs.aws.amazon.com/parallelcluster/latest/ug/what-is-aws-parallelcluster.html).

## References

- AWS. 2026a. [Best practices for Amazon EC2 Spot](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html).
- AWS. 2026b. [Spot Instance interruption notices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-instance-termination-notices.html).
- AWS. 2026c. [What is AWS Batch?](https://docs.aws.amazon.com/batch/latest/userguide/what-is-batch.html).
- AWS. 2026d. [Array jobs](https://docs.aws.amazon.com/batch/latest/userguide/array_jobs.html).
- AWS. 2026e. [When to use AWS Batch](https://docs.aws.amazon.com/batch/latest/userguide/bestpractice1.html).
- AWS. 2026f. [Getting started with AWS Batch on Amazon EKS](https://docs.aws.amazon.com/batch/latest/userguide/getting-started-eks.html).
- AWS. 2026g. [What is AWS ParallelCluster](https://docs.aws.amazon.com/parallelcluster/latest/ug/what-is-aws-parallelcluster.html).
- AWS. 2026h. [Job timeouts](https://docs.aws.amazon.com/batch/latest/userguide/job_timeouts.html).
- AWS. 2026i. [Welcome to AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html).
- Ansible Community Documentation. 2026. [amazon.aws.ec2_instance module](https://docs.ansible.com/ansible/latest/collections/amazon/aws/ec2_instance_module.html).
