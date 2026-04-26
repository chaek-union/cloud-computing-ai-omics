# 20장. 비용이 새는 지점과 해결 전략 - 1PB WGS 운영에서 배우기

## 학습 목표

- 유전체 워크플로에서 자주 놓치는 숨은 비용 항목을 설명할 수 있다.
- 비용이 커지는 메커니즘을 `스토리지`, `요청`, `컴퓨트`, `데이터 이동`, `로그`로 나누어 이해할 수 있다.
- AWS의 lifecycle rule, right-sizing, Spot, retention 설정 같은 구체적 대응책을 제시할 수 있다.

## 핵심 질문

- 왜 "컴퓨트만 많이 썼다"고 생각했는데 실제 청구서는 다른 곳에서 커지는가
- 1PB급 WGS 프로젝트에서 비용은 어느 한 항목이 아니라 어떤 상호작용으로 만들어지는가
- 숨은 비용은 피할 수 없는가, 아니면 설계와 운영으로 줄일 수 있는가

## 데이터 크기보다 데이터 생애주기가 더 중요하다

대규모 오믹스 프로젝트의 비용을 이해할 때 가장 먼저 버려야 할 생각은 `데이터가 크니까 비싸다`라는 단순한 설명이다. 실제로는 같은 1PB라도 무엇을 얼마나 오래, 어떤 방식으로 읽고, 얼마나 자주 옮기고, 중간 산물을 어떻게 남겨 두느냐에 따라 비용 구조가 크게 달라질 수 있다. 즉 데이터 크기 자체보다 데이터 생애주기(data lifecycle)가 더 중요하다. 학생은 이 장에서 청구서를 보는 법보다, 비용이 만들어지는 메커니즘을 보는 법을 먼저 배워야 한다. 그렇게 해야 숨은 비용이 갑자기 튀는 이유를 이해할 수 있다.

교육용으로 1,000명 WGS, 총 데이터 약 1PB 수준을 생각해 보자. raw FASTQ가 400-600 TB, processed BAM/CRAM이 300-500 TB, intermediate가 300-800 TB, derived data가 10-50 TB 정도라고 가정할 수 있다. 여기서 중요한 것은 최종 보관량이 아니라 workflow 중간의 순간 최대 저장량이다. 실제 운영에서는 intermediate가 원본보다 더 커지는 경우가 흔하며, 순간적으로는 2-3PB에 가까운 저장량이 보일 수도 있다. 따라서 비용 설계는 최종 산물만 보고 세울 수 없고, 워크플로 전체의 생애주기를 함께 봐야 한다.

## Incomplete multipart upload와 보이지 않는 유령 바이트

대형 FASTQ, BAM, CRAM 업로드의 기본 방식은 S3 multipart upload다. 이 방식 자체는 효율적이지만, 업로드가 중간에 실패하거나 파이프라인이 멈추면 완성되지 않은 part가 S3에 남을 수 있다. 문제는 사용자가 파일이 "아직 없다"고 느껴도, 저장 비용은 이미 발생하고 있다는 점이다. 이 때문에 incomplete multipart upload는 대규모 오믹스 프로젝트에서 대표적인 `보이지 않는 비용`이 된다. 파일이 눈에 잘 보이지 않아도 바이트는 이미 남아 있기 때문이다.

AWS는 이를 줄이기 위해 `AbortIncompleteMultipartUpload` lifecycle rule을 제공한다. 교육용으로는 `7일 뒤 미완료 업로드 자동 중단` 같은 정책을 기본값처럼 가르치는 것이 좋다. 완성되지 않은 파일도 비용을 만든다는 점이 핵심이다. 따라서 업로드 실패가 잦거나, 샘플 반입 파이프라인이 길거나, 외부 기관이 데이터를 자주 전달하는 환경일수록 이 규칙은 사실상 필수다. 비용 최적화는 화려한 절약 기술보다 먼저 이런 기본 hygiene에서 시작한다.

실제 현장에서 이 문제가 얼마나 자주 발생하는지는 외부 엔지니어링 문서에서도 확인할 수 있다. AWS Storage Blog의 ["Discovering and deleting incomplete multipart uploads to lower Amazon S3 costs"](https://aws.amazon.com/blogs/storage/discovering-and-deleting-incomplete-multipart-uploads-to-lower-amazon-s3-costs/)는 ListMultipartUploads API로 잔존 part를 찾아내고 lifecycle로 정리하는 절차를 설명한다. Corey Quinn은 Last Week in AWS에서 ["Quieting Noisy Multipart Upload Costs"](https://www.lastweekinaws.com/blog/quieting-noisy-multipart-upload-costs/) 같은 글로 이 문제를 "S3 청구서에서 가장 많이 간과되는 항목"이라고 반복해서 지적한다. 또 AWS Config 관리형 규칙 [`s3-lifecycle-policy-check`](https://docs.aws.amazon.com/config/latest/developerguide/s3-lifecycle-policy-check.html)나 [`s3-version-lifecycle-policy-check`](https://docs.aws.amazon.com/config/latest/developerguide/s3-version-lifecycle-policy-check.html)는 이런 정책이 버킷에 실제로 설정되어 있는지를 계정 차원에서 감시할 수 있게 해 준다. 유전체 연구실이 수십 수백 TB를 다루기 시작하면, 이 작은 규칙 하나가 매달 눈에 띄는 청구서 차이를 만든다.

## Small object, random access, request cost

S3 비용은 저장 단가만으로 결정되지 않는다. request cost는 유전체 분석에서 자주 과소평가되지만, 실제로는 매우 중요할 수 있다. BAM과 CRAM의 random access, index 기반 반복 조회, 수십만 개 small object, intermediate 재읽기, console 탐색에 따른 LIST/GET 요청이 겹치면 저장 비용보다 요청 비용이 더 크게 느껴질 수도 있다. 특히 `파일 크기`보다 `파일 개수와 접근 패턴`이 비용을 좌우한다는 점이 중요하다. 따라서 데이터 구조를 잘못 설계하면 저장 자체는 싸도 운영 비용은 빠르게 커진다.

이 문제는 클라우드 네이티브라는 이름 아래 small object를 지나치게 많이 만드는 구조에서 더 자주 드러난다. 예를 들어 chunk를 너무 잘게 나눈 배열 데이터, 지나치게 세분화된 intermediate shard, 정리되지 않은 임시 산물은 모두 요청 비용과 listing overhead를 키운다. 해결책은 일괄적으로 하나의 큰 파일로 돌아가는 것이 아니라, `작업 패턴에 맞는 적절한 object granularity`를 잡는 것이다. S3를 단순한 하드디스크로 떠올리는 순간 요청 비용은 눈에 띄지 않게 불어난다. 객체 스토리지는 저장 방식만이 아니라 비용 구조도 다르다.

## Over-provisioned compute와 right-sizing

compute 비용은 보통 가장 눈에 잘 띄기 때문에, 사람들은 종종 그것만을 문제로 생각한다. 그러나 compute는 동시에 가장 통제하기 쉬운 비용이기도 하다. 낭비의 주된 원인은 `병목과 맞지 않는 인스턴스 선택`, `불필요하게 큰 자원`, `Spot 미사용`, `과도한 shard로 인한 orchestration overhead`다. 예를 들어 실제 병목이 디스크 I/O인데 CPU 코어만 늘리거나, 메모리 사용량은 낮은데 `r` 계열을 쓰면 비용은 금방 새어 나간다. 즉 compute는 비싸서 문제가 되는 경우보다, 잘못 배치해서 문제가 되는 경우가 더 많다.

해결의 핵심은 right-sizing이다. 처음부터 가장 큰 인스턴스를 고르기보다 `m -> c/r/i/f`처럼 병목에 따라 점진적으로 맞추는 습관이 중요하다. 샘플 단위 병렬화가 가능하면 작은 인스턴스를 많이 쓰는 편이 나을 수 있고, Spot이 잘 맞는 작업이라면 On-Demand만 고집할 이유가 없다. 또한 workflow의 resume, retry, call caching, checkpoint가 잘 설계되어 있으면 더 공격적으로 저렴한 자원을 활용할 수 있다. 학생은 여기서 비용 최적화가 곧 성능 최적화와 다르지 않다는 사실을 배우게 된다.

## Intermediate data explosion과 lifecycle 설계

유전체 분석에서 intermediate data는 자주 과소평가된다. sorted BAM, recalibrated BAM, temporary shard, feature matrix, 중간 summary table은 최종 결과보다 더 큰 저장량을 차지할 수 있다. 프로젝트를 시작할 때는 raw FASTQ와 최종 VCF만 떠올리기 쉽지만, 실제 청구서를 키우는 것은 종종 분석 중간의 잠깐 존재하는 대형 파일들이다. 문제는 이 intermediate가 분석이 끝난 뒤에도 아무 규칙 없이 남아 있는 경우가 많다는 점이다. 이때 storage cost는 조용히, 그러나 꾸준히 누적된다.

따라서 lifecycle 설계는 필수다. 어떤 intermediate는 7일 뒤 지워도 되지만, 어떤 것은 재현성을 위해 한 달 보관해야 할 수 있다. 어떤 결과는 바로 Parquet나 요약 테이블로 변환한 뒤 원본 intermediate를 없애도 되지만, 어떤 단계는 재분석을 위해 더 오래 남겨야 할 수 있다. 핵심은 `분석이 끝난 뒤 중간 산물이 어떻게 처리될지`를 미리 정하는 것이다. 생애주기 규칙이 없는 data lake는 쉽게 data swamp가 되고, 비용은 그 늪의 가장 빠른 신호로 나타난다.

## Egress, cross-AZ, cross-region data movement

협업 단계에서 가장 위험한 비용 가운데 하나가 데이터 이동이다. 많은 사용자가 같은 AWS 안에서는 이동이 다 무료라고 생각하지만, 실제로는 그렇지 않다. 인터넷으로 나가는 egress는 눈에 보이는 비용 항목이고, 같은 리전 안에서도 가용 영역이 다르면 비용이 생길 수 있으며, 리전 간 이동은 따로 과금되는 대상이다. 유전체 데이터는 한 번 움직일 때 TB 단위가 되기 쉽기 때문에, 작은 설계 실수가 곧 큰 청구서로 이어진다. 따라서 compute를 데이터가 있는 곳 가까이에 배치하는 원칙은 성능뿐 아니라 비용 측면에서도 매우 중요하다.

이 문제는 특히 외부 협업에서 자주 드러난다. 데이터를 collaborator에게 넘기거나, 다른 클라우드로 이동하거나, 로컬로 대량 다운로드하는 순간 비용 구조가 바뀐다. 따라서 좋은 운영 원칙은 `가능하면 데이터는 제자리에 두고, 사람과 계산을 그쪽으로 가져간다`는 것이다. 공개 데이터에서 `bring compute to data`가 중요했던 이유가 여기서 다시 나타난다. 공개 데이터 접근과 자체 데이터 협업은 결국 같은 비용 원리 위에 놓여 있다.

## CloudWatch logs, metrics, metadata retention

로그는 작아 보여서 종종 무시되지만, 장기 운영에서는 결코 작은 비용이 아니다. CloudWatch Logs는 기본적으로 로그를 무기한 보관할 수 있으므로, retention을 설정하지 않으면 Batch, EMR, HealthOmics, Lambda, Step Functions에서 나온 로그가 계속 쌓인다. 개별 로그 이벤트는 작더라도, 대량 샘플 처리와 긴 기간이 겹치면 누적 비용은 무시하기 어려워진다. 게다가 Logs Insights로 반복 스캔하는 비용까지 더해지면, "로그는 작으니 괜찮다"는 말은 더 이상 맞지 않는다. 로그도 data lifecycle의 일부로 봐야 한다.

해결 방향은 비교적 단순하다. workflow 성격에 따라 7일, 30일, 90일 같은 retention 정책을 명시하고, 장기 보관이 정말 필요한 로그만 따로 남기는 것이다. 학생에게는 이것을 단순 절약 기술보다 `운영 규율`로 가르치는 편이 좋다. 좋은 운영은 문제가 생겼을 때 필요한 로그는 충분히 남기되, 의미 없는 로그를 영구 보관하지 않는다. 결국 로그도 intermediate data와 다르지 않다. 무엇을 얼마나 오래 남길지 미리 정하지 않으면, 조용히 비용을 만들어 낸다.

## 1,000 WGS, ~1PB 환경을 비용 구조로 읽는 법

교육용으로 1,000명 WGS, 약 1PB 규모의 프로젝트를 생각해 보자. 여기서 비용은 하나의 큰 항목이 지배하는 경우보다, storage growth와 request frequency, inefficient compute usage, data movement가 상호작용하며 생겨난다. raw FASTQ를 오래 남기면 storage cost가 생기고, 이를 작은 shard로 계속 읽으면 request cost가 커지고, 분석을 위해 다른 리전의 compute를 쓰면 transfer cost가 붙고, 병목과 맞지 않는 인스턴스를 쓰면 compute cost가 늘어난다. 즉 청구서는 기술 요소의 목록이 아니라, 운영 설계의 결과물이다. 비용은 곧 시스템이 어디에서 비효율적인지를 드러내는 지표다.

이 예제에서 학생에게 남겨야 할 핵심 메시지는 세 가지다. 첫째, `Data size is not the real problem — data lifecycle is.` 둘째, `Compute is controllable — storage and transfer are where costs silently accumulate.` 셋째, `The biggest costs often come from things you did not explicitly plan.` 이 세 문장은 단순한 슬로건이 아니라, 실제 운영 경험을 요약한 원칙이다. 학생이 이 원칙을 이해하면, 아직 자기 프로젝트가 1PB가 아니더라도 큰 규모를 설계할 감각을 미리 배울 수 있다.

## 비용 누수 진단표와 해결 플레이북

비용 문제를 해결할 때 중요한 것은 공포가 아니라 진단이다. incomplete multipart upload가 보인다면 lifecycle rule부터 확인한다. request cost가 크다면 small object와 random access pattern을 의심한다. compute 비용이 높다면 인스턴스 right-sizing과 Spot 사용 여부를 본다. storage가 급격히 늘어난다면 intermediate retention을 점검한다. data transfer가 크다면 cross-region, cross-AZ, egress 패턴을 추적한다. logs 비용이 계속 늘면 retention 설정이 없는지부터 본다.

Table 1. 비용 누수 신호와 첫 조치

| 신호 | 먼저 의심할 원인 | 첫 조치 |
|---|---|---|
| 저장량이 파일 목록보다 크게 보인다 | incomplete multipart upload | `AbortIncompleteMultipartUpload` lifecycle rule 확인 |
| S3 request 비용이 예상보다 크다 | small object, 반복 LIST/GET, random access | object granularity와 access pattern 점검 |
| EC2 비용이 샘플 수에 비례하지 않는다 | over-provisioning, 병목과 맞지 않는 instance | CPU, memory, I/O 사용률을 보고 right-sizing |
| 분석이 끝났는데 storage가 계속 증가한다 | intermediate retention 부재 | scratch와 intermediate prefix에 보관 기간 설정 |
| 협업 뒤 transfer 비용이 튄다 | cross-region, egress, cross-AZ 이동 | compute와 collaborator 접근 위치 재설계 |
| CloudWatch 비용이 조용히 늘어난다 | log retention 미설정 | log group별 retention 기본값 설정 |

숨은 비용은 피할 수 없는 숙명이 아니라, 파악하면 줄일 수 있는 설계 문제다. 유전체 분석은 데이터가 크기 때문에 비용이 생기는 것이 아니라, 큰 데이터를 어떤 습관으로 운영하는가에 따라 비용이 달라진다. 좋은 연구 인프라는 빠르기만 한 인프라가 아니라, 어디서 비용이 만들어지는지를 설명할 수 있는 인프라다. 이 감각이 생기면 청구서는 두려움의 대상이 아니라 설계를 고치는 피드백 도구가 된다.

## 핵심 개념 정리

- 오믹스 프로젝트의 비용은 데이터 크기 자체보다 데이터 생애주기와 접근 패턴에 더 크게 좌우된다.
- incomplete multipart upload, small object 요청, intermediate data, transfer, 로그는 대표적인 숨은 비용 축이다.
- compute 비용은 눈에 잘 띄지만 동시에 가장 통제하기 쉬운 항목이다.
- 비용 최적화는 절약 기술보다 `미리 설계한 lifecycle과 retention 규칙`에서 출발한다.

## 복습 질문

1. incomplete multipart upload가 왜 `유령 데이터`라고 불릴 수 있는가?
2. 저장 비용보다 request 비용이 더 커질 수 있는 유전체 작업의 예를 들어 보라.
3. 1PB급 WGS 환경에서 비용을 줄이기 위한 설계 원칙 세 가지를 제시하라.

## Further Reading

- AWS. [Aborting incomplete multipart uploads using a bucket lifecycle configuration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpu-abort-incomplete-mpu-lifecycle-config.html).
- AWS. [Amazon S3 pricing](https://aws.amazon.com/s3/pricing/).
- AWS. [CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/).

## References

- AWS. 2026a. [Aborting incomplete multipart uploads using a bucket lifecycle configuration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpu-abort-incomplete-mpu-lifecycle-config.html).
- AWS. 2026b. [Amazon S3 pricing](https://aws.amazon.com/s3/pricing/).
- AWS. 2026c. [Amazon EC2 On-Demand pricing](https://aws.amazon.com/ec2/pricing/on-demand/).
- AWS. 2026d. [Understanding data transfer charges in CUR](https://docs.aws.amazon.com/cur/latest/userguide/cur-data-transfers-charges.html).
- AWS. 2026e. [Working with log groups and log streams](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html).
- AWS. 2026f. [CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/).
