# 2장. EC2, S3, EBS, IAM Role 이해하기

## 학습 목표

- EC2 인스턴스, S3, EBS의 역할 차이를 구분할 수 있다.
- 블록 스토리지와 객체 스토리지의 차이를 유전체 데이터 사례로 설명할 수 있다.
- IAM User와 IAM Role의 차이를 이해하고 기본 보안 원칙을 설명할 수 있다.

## 핵심 질문

- EC2는 무엇을 위한 서비스인가
- S3와 EBS는 각각 언제 쓰는가
- 왜 액세스 키를 파일에 박아 넣는 대신 IAM Role을 써야 하는가

## 리전, 가용 영역, 계정이라는 기본 단위

AWS를 처음 배울 때 가장 먼저 익혀야 하는 것은 개별 서비스보다 공간적 단위와 관리 단위다. 리전(region)은 물리적으로 떨어진 지리적 영역이고, 가용 영역(availability zone, AZ)은 한 리전 안에서 전력과 네트워크 측면에서 어느 정도 분리된 데이터센터 묶음이라고 이해하면 된다. 계정(account)은 이런 자원을 누구의 소유와 과금 체계 아래에서 운영할 것인지를 결정하는 관리 단위다. 오믹스 분석에서는 계산 자원과 저장 자원이 같은 리전에 있는지가 매우 중요하다. 같은 AWS 안에 있어도 리전이 달라지면 데이터 전송 비용과 지연 시간이 커질 수 있고, 워크플로 전체의 단순성도 떨어진다.

이 개념이 중요한 이유는 유전체 데이터가 크기 때문이다. 몇 메가바이트짜리 로그 파일을 옮길 때는 잘 드러나지 않지만, 수백 기가바이트 FASTQ나 여러 샘플의 BAM, CRAM을 옮기기 시작하면 위치가 비용과 성능을 직접 좌우한다. 따라서 초급자도 "어느 리전에 버킷을 만들 것인가", "어느 리전에서 EC2를 띄울 것인가", "공개 데이터가 어느 리전에 있는가"를 먼저 확인하는 습관을 가져야 한다. 이후 배우게 될 S3, EC2, Batch, HealthOmics, EMR 모두 이 기본 단위 위에서 움직인다. 서비스 이름을 모를 때도 `내 데이터는 어디에 있고, 내 계산은 어디에서 돌아가는가`를 먼저 묻는다면 실수를 크게 줄일 수 있다.

## EC2 인스턴스 이름 읽기 - `m`, `c`, `r`, `i`, `f`, `hpc`

Amazon EC2는 AWS에서 가장 기본적인 계산 서비스다. 가장 단순하게 말하면, 필요할 때 원하는 크기의 가상 서버를 켜는 서비스다. 오믹스 분석에서는 정렬, 변이 호출, QC, 노트북 실습, 맞춤형 파이프라인 실행처럼 "직접 소프트웨어를 설치하고 돌려야 하는 계산"이 주로 EC2 위에서 돌아간다. 초보자에게는 인스턴스 종류가 너무 많아 보이지만, 실제로는 이름을 읽는 규칙만 이해해도 전체 지형이 빠르게 정리된다. 핵심은 어떤 인스턴스가 더 비싼지가 아니라, 어떤 병목에 맞추어 설계되었는지를 읽는 것이다.

AWS 인스턴스 이름에서 가장 먼저 볼 것은 패밀리(family)다. `m`은 general purpose, `c`는 compute optimized, `r`은 memory optimized, `i`나 `d` 계열은 storage optimized, `f`는 FPGA 같은 가속기를 쓰는 accelerated computing, `hpc`는 긴밀한 병렬 통신이 필요한 high-performance computing 계열로 이해할 수 있다. 그 뒤의 숫자는 세대(generation), 마지막 크기 표시는 인스턴스 크기(size)를 뜻한다. 따라서 `r7i.4xlarge` 같은 이름을 보면, 최신 세대의 메모리 중심 인스턴스라는 사실부터 파악하면 된다. 이 책에서 중요한 것은 모든 이름을 외우는 일이 아니라, 이름에서 의도를 읽는 감각을 기르는 일이다 (AWS 2024a).

세대와 접미사까지 함께 읽으면 정보가 훨씬 풍부해진다. `c5`는 Xeon 기반의 5세대 compute optimized, `c6i`는 Intel Ice Lake 기반의 6세대, `c6a`는 AMD EPYC 기반의 6세대, `c6g`는 AWS Graviton2 기반, `c7g`는 Graviton3, `c8g`는 Graviton4 기반 최신 세대를 뜻한다. 접미사 `i`는 Intel, `a`는 AMD, `g`는 Graviton이라는 프로세서 종류를 가리키고, 여기에 다시 붙는 `d`는 NVMe 로컬 SSD(instance store)를, `n`은 네트워크 대역폭 강화를, `e`는 메모리 확장을 의미한다. 따라서 `c8gd.8xlarge`는 `Graviton4 기반 compute optimized, 로컬 NVMe 포함, 8xlarge 크기`라고 한 문장으로 풀어 읽을 수 있다. 같은 논리로 `r7iz`는 `Ice Lake 기반 메모리 최적화에 고주파(z) 변형`, `m7a.xlarge`는 `AMD 기반 7세대 general purpose`로 읽힌다.

크기 표시도 규칙적이다. `large`, `xlarge`, `2xlarge`, `4xlarge`, `8xlarge`, `12xlarge`, `16xlarge`, `24xlarge`, `48xlarge`, `metal` 순으로 vCPU와 메모리가 대체로 두 배씩 늘어난다. 같은 `c7g` 안에서 `c7g.large`(2 vCPU / 4 GiB)부터 `c7g.16xlarge`(64 vCPU / 128 GiB)까지 스펙이 선형적으로 확장되므로, 우선 패밀리와 세대를 고르고 나서 크기만 필요에 맞게 고르면 된다. 오믹스 분석에서는 인스턴스를 고를 때 구체적인 이름이 곧 예산과 병목을 결정하므로, 다음 Table 1-1처럼 최근 세대의 대표 선택지를 머릿속에 하나씩 갖고 있는 편이 실용적이다.

Table 1-1. 2026년 기준 compute optimized 계열 대표 예시

| 이름 | 프로세서 | 로컬 SSD | 대표 크기 | 오믹스에서 자주 쓰는 맥락 |
|---|---|---|---|---|
| `c5.9xlarge` | Intel Xeon (5세대) | 없음 | 36 vCPU / 72 GiB | 기존 파이프라인 호환, 레거시 워크로드 |
| `c6i.8xlarge` | Intel Ice Lake | 없음 | 32 vCPU / 64 GiB | 표준 variant calling, bwa, GATK |
| `c6a.16xlarge` | AMD EPYC | 없음 | 64 vCPU / 128 GiB | CPU 집약 Batch, 가격 대비 CPU 시간 |
| `c7g.16xlarge` | AWS Graviton3 | 없음 | 64 vCPU / 128 GiB | ARM 빌드 가능한 최신 툴 (samtools, bcftools) |
| `c7gd.8xlarge` | Graviton3 | NVMe SSD 2×1.9 TB | 32 vCPU / 64 GiB | 정렬 sort, shard 기반 scratch |
| `c8g.24xlarge` | Graviton4 | 없음 | 96 vCPU / 192 GiB | 최신 대규모 CPU 병렬 작업 |
| `c8gd.16xlarge` | Graviton4 | 로컬 NVMe 포함 | 64 vCPU / 128 GiB | 최신 scratch-heavy 정렬·sort |

메모리형(`r6i`, `r7g`, `r8g`, `r7iz`)이나 storage-optimized(`i4i`, `i7ie`, `i8g`, `im4gn`, `is4gen`), 범용(`m6i`, `m7g`, `m8g`)도 같은 규칙으로 읽으면 된다. 예를 들어 Hail joint genotyping처럼 메모리 압박이 큰 작업에는 `r7iz.32xlarge`(고주파 Xeon, 128 vCPU / 1,024 GiB)가, sort 중간 파일이 많은 정렬 워크플로에는 로컬 NVMe가 붙은 `i7ie.6xlarge`나 `c8gd.8xlarge`가 자연스럽다. 반대로 "뭘 골라야 할지 모르겠다"는 상황에서는 `m` 계열 최신 세대(`m8g.xlarge` 같은 것)를 기본값으로 두고, 실제 실행 로그를 보면서 병목을 확인한 뒤 `c`·`r`·`i`로 바꿔 나가는 편이 안전하다. 실제 구체 인스턴스 종류와 온디맨드 가격은 AWS EC2 Instance Types 페이지에서 최신 목록을 확인할 수 있다 (AWS 2024a).

## 오믹스 분석용 컴퓨트 선택 - 메모리형, CPU형, IO형, 가속형

오믹스 분석에서 인스턴스를 고를 때 초급자가 가장 흔히 하는 실수는 "일단 큰 서버면 좋다"라고 생각하는 것이다. 실제로는 CPU가 병목인지, 메모리가 병목인지, 로컬 디스크 I/O가 병목인지, 아니면 특정 가속기를 쓸 수 있는지에 따라 선택이 달라져야 한다. 예를 들어 정렬과 일반적인 variant calling은 CPU 사용량이 높고 샘플 단위 병렬화가 쉬우므로 `c` 계열이 잘 맞는 경우가 많다. 반대로 Hail aggregation, joint genotyping, 대형 annotation join처럼 메모리 압박이 큰 단계는 `r` 계열이 더 자연스럽다. Notebook 기반 탐색이나 가벼운 분석 서버는 `m` 계열로도 충분한 경우가 많다.

로컬 scratch 공간을 많이 쓰는 작업은 따로 생각해야 한다. 정렬 중간 산물, shard 파일, 대형 sort, 일시적인 matrix 변환처럼 짧은 시간 동안 많은 읽기·쓰기를 반복하는 작업은 `i` 또는 `d` 계열이 유리할 수 있다. 반면 FPGA 기반 가속 소프트웨어를 쓸 수 있을 때만 `f` 계열이 의미가 있다. 예를 들어 DRAGEN처럼 하드웨어 가속을 전제로 설계된 파이프라인은 일반 CPU 인스턴스와 다른 성능·비용 구조를 보일 수 있지만, 모든 분석이 자동으로 빨라지는 것은 아니다. 즉 accelerated instance는 "범용 고성능"이 아니라, 해당 가속기를 이해하는 소프트웨어와 함께 쓸 때만 강한 특수 선택지라고 보는 편이 정확하다.

Table 1은 오믹스 분석에서 자주 만나는 workload와 인스턴스 패밀리를 교육용으로 정리한 것이다. 실제 선택에서는 데이터 크기, 사용 도구, 병렬화 방식, 비용 제약까지 함께 봐야 한다. 그럼에도 이 정도의 첫 지도가 있으면 초급자도 최소한 `왜 이 인스턴스를 골랐는가`를 설명할 수 있다. 좋은 선택은 늘 가장 비싼 인스턴스가 아니라, 병목에 가장 잘 맞는 인스턴스다. 이 판단 습관은 이후 Spot, Batch, EMR, HealthOmics를 사용할 때도 그대로 이어진다.

Table 1. 오믹스 분석에서의 대표 인스턴스 선택

| 인스턴스 패밀리 | 대표 병목 | 유전체 예시 |
|---|---|---|
| `m` | 균형형 | 노트북 서버, 일반 분석 환경, 소규모 테스트 |
| `c` | CPU | 정렬, 일반 batch 작업, 샘플 단위 variant calling |
| `r` | 메모리 | joint genotyping, 대형 annotation join, Hail 집계 |
| `i` / `d` | 로컬 I/O | sort, shard 기반 중간 파일 처리, scratch-heavy workflow |
| `f` | 가속기 | DRAGEN 같은 FPGA 기반 파이프라인 |
| `hpc` | 긴밀한 병렬 통신 | EFA가 필요한 tightly coupled 계산 |

## S3, EBS, FSx - 데이터가 머무는 자리

EC2가 계산을 위한 서버라면, EBS와 S3는 데이터를 두는 서로 다른 방식이다. Amazon EBS는 인스턴스에 붙는 블록 스토리지(block storage)로, 운영체제 디스크나 작업용 파일시스템처럼 "서버에 연결된 디스크"에 가깝다. 파일을 마운트해서 사용하고, 정렬 중간 파일이나 임시 작업 공간처럼 빠른 랜덤 I/O가 필요한 작업에 잘 맞는다. 반면 Amazon S3는 객체 스토리지(object storage)다. 폴더처럼 보이지만 실제로는 경로가 아니라 객체 키(key)를 가진 파일 저장소이며, 주로 URI 형태로 접근한다. FASTQ, BAM, CRAM, VCF, reference, 결과 파일, 공유 데이터셋처럼 오래 보관하고 여러 서비스가 함께 읽는 자산은 S3에 두는 편이 자연스럽다.

초보자에게는 `EBS는 작업대`, `S3는 데이터 호수`라는 비유가 꽤 유용하다. 작업대 위에는 지금 손으로 만지며 처리할 물건을 올려 두고, 호수에는 여러 사람이 함께 참고할 자료와 오래 보관할 자산을 둔다고 생각하면 된다. 이 둘 사이의 차이는 오믹스 분석에서 매우 실용적이다. 예를 들어 BAM 정렬 과정에서는 임시 파일과 sort 작업이 많으므로 EBS나 로컬 scratch가 중요하지만, 완성된 결과와 원본 데이터는 S3에 보관하는 편이 낫다. 즉 어디에 저장할지는 단순한 취향이 아니라, 접근 패턴과 워크플로 단계의 문제다.

FSx for Lustre는 이 둘 사이를 연결하는 특수한 선택지로 짧게 이해하면 좋다. 이는 POSIX 파일시스템이 필요하고, 여러 계산 노드가 빠르게 같은 파일 트리를 읽고 써야 할 때 유용한 고성능 공유 스토리지다. S3와 연동해 대규모 원본을 가져와 scratch처럼 처리하고 다시 결과를 S3로 돌려보내는 방식이 대표적이다. 모든 연구실이 처음부터 FSx를 배울 필요는 없지만, `S3만으로는 불편한 고성능 파일시스템 수요`가 있다는 사실을 아는 것은 중요하다. 특히 구조적 변이 분석이나 대규모 intermediate 파일을 다루는 워크플로에서는 이런 계층이 큰 차이를 만들 수 있다 (AWS 2024d).

Table 2. 유전체 데이터 유형별 저장 위치의 첫 선택

| 데이터 유형 | 우선 저장 위치 | 이유 |
|---|---|---|
| 원시 FASTQ | S3 | 장기 보관, 공유, 여러 서비스의 공통 입력 |
| 정렬 중간 파일 | EBS 또는 로컬 scratch | 높은 I/O와 짧은 수명 |
| 최종 BAM/CRAM, VCF | S3 | 재사용과 공유가 쉬움 |
| 운영체제와 도구 설치 | EBS | 인스턴스에 붙는 시스템 디스크 |
| 다수 노드가 함께 읽는 고성능 작업 데이터 | FSx for Lustre | 공유 파일시스템과 높은 처리량 |

## S3 Standard, Intelligent-Tiering, Glacier 계층과 비용 감각

S3를 이해할 때 또 하나 중요한 것은 모든 객체가 같은 비용 구조를 갖지 않는다는 점이다. 가장 단순한 `S3 Standard`는 자주 읽고 쓰는 데이터에 적합한 기본 계층이다. 접근 패턴이 불규칙하거나 예측하기 어려운 경우에는 `S3 Intelligent-Tiering`이 유용할 수 있다. 반면 `Standard-IA`나 Glacier 계열은 저장 단가를 낮추는 대신, 읽어올 때의 retrieval fee나 최소 저장 기간(minimum storage duration), 복원 지연을 고려해야 한다. 따라서 초보자는 "가장 싼 클래스가 항상 가장 경제적이다"라는 생각부터 버리는 편이 좋다.

유전체 데이터는 생애주기(lifecycle)가 비교적 잘 구분되므로 storage class 사고방식이 특히 중요하다. 분석 직후의 FASTQ와 결과물은 재분석과 검증이 자주 일어나므로 `hot` 데이터에 가깝고, 이 단계에서는 `S3 Standard`가 자연스럽다. 몇 주나 몇 달이 지나 접근 빈도가 줄어들면 `Intelligent-Tiering`이나 `Standard-IA`가 적절할 수 있다. 장기 보관 의무가 있는 원시 데이터나 거의 꺼내지 않는 보관본은 `Glacier Instant Retrieval`, `Glacier Flexible Retrieval`, `Glacier Deep Archive` 같은 colder tier를 검토할 수 있다. 하지만 재분석이 잦은 데이터를 너무 빨리 Deep Archive로 밀어 넣으면 저장비는 줄어도 복원 지연과 retrieval 비용 때문에 프로젝트 전체 비용은 오히려 올라갈 수 있다 (AWS 2024b).

실전에서는 lifecycle rule을 통해 데이터를 단계적으로 이동시키는 패턴을 많이 쓴다. 예를 들어 업로드 직후에는 `S3 Standard`에 두고, 일정 기간이 지나면 `Intelligent-Tiering`이나 `Standard-IA`로 옮기고, 장기 보관 대상만 Glacier 계열로 내리는 식이다. 이 구조는 학생들에게 단순 암기보다 훨씬 유익하다. 중요한 것은 storage class 이름 자체가 아니라, 어떤 데이터가 아직 활발히 사용 중인지, 어떤 데이터가 보관 단계로 들어갔는지를 지속적으로 구분하는 습관이다. 비용 감각은 청구서를 본 뒤에 배우는 것이 아니라, 저장소를 설계할 때부터 함께 배워야 한다.

## IAM Role과 최소 권한 원칙

IAM(Identity and Access Management)은 AWS에서 누가 무엇에 접근할 수 있는지를 결정하는 권한 체계다. 초보자는 보통 IAM User와 액세스 키부터 떠올리지만, 오믹스 분석 운영에서 더 중요한 개념은 IAM Role이다. IAM User는 사람이 로그인하거나 API를 호출할 때 사용하는 장기 자격 증명에 가깝고, IAM Role은 EC2, Batch, Lambda, HealthOmics 같은 실행 주체에 임시 권한을 부여하는 방식이라고 이해하면 된다. 이 차이는 보안뿐 아니라 운영 편의성에서도 매우 크다. 액세스 키를 파일이나 스크립트에 박아 넣지 않고도, 인스턴스가 필요한 S3 버킷만 읽도록 안전하게 설정할 수 있기 때문이다.

최소 권한 원칙(least privilege)은 "필요한 일만 할 수 있도록 최소한의 권한만 준다"는 원칙이다. 예를 들어 어떤 EC2 인스턴스가 특정 입력 버킷을 읽고 결과 버킷에 쓰기만 하면 된다면, 전체 S3 전체 권한을 줄 이유가 없다. 이 원칙은 학생에게 추상적인 보안 문구로 설명하기보다, 실제 분석 흐름으로 보여 주는 편이 훨씬 잘 이해된다. `EC2가 S3를 읽기 위해 Role을 붙인다`는 한 문장을 중심으로 생각하면, 왜 액세스 키를 코드에 넣는 습관이 위험한지 바로 드러난다. 유전체 데이터는 민감 정보일 수 있기 때문에, 권한 관리의 기본기를 초반에 정확히 배우는 것이 이후 모든 장의 안전장치가 된다 (AWS 2024c; AWS 2024e).

## 초보자가 자주 하는 실수

입문자에게서 되풀이되는 실수는 크게 네 가지로 볼 수 있다. 첫째, EBS와 S3를 같은 종류의 저장소라고 생각하는 것이다. EBS는 서버에 붙는 디스크이고, S3는 여러 서비스가 공유하는 객체 저장소이므로, 둘을 혼동하면 데이터 배치와 비용 감각이 모두 흐려진다. 둘째, 인스턴스 크기만 보고 선택하는 것이다. 실제로는 메모리가 필요한 작업에 CPU형 인스턴스를 쓰거나, I/O 병목 작업에 큰 CPU만 늘리면 성능은 거의 오르지 않고 비용만 커진다.

셋째, 액세스 키를 설정 파일이나 노트북에 하드코딩하는 것이다. 처음에는 편해 보여도, 이 습관은 팀 협업과 장기 운영에서 큰 위험이 된다. IAM Role을 써야 할 자리에 장기 키를 넣어 두면 누가 어떤 권한을 언제 썼는지 통제하기 어려워지고, 비밀 유출 가능성도 커진다. 마지막으로 storage class를 무작정 가장 싼 쪽으로 선택하는 실수도 많다. 저장 단가만 보고 Glacier Deep Archive를 선택했다가, 나중에 데이터를 꺼내는 시간과 비용 때문에 연구 일정 전체가 꼬이는 일은 생각보다 흔하다.

EC2는 계산, EBS는 붙는 디스크, S3는 오래 두고 공유하는 객체 저장소, IAM Role은 실행 주체에 임시 권한을 주는 방식. 이렇게 네 축을 먼저 잡아 두면 충분하다. 그다음부터는 서비스 이름이 아니라 병목과 데이터 생애주기를 기준으로 선택하면 된다. 즉 좋은 AWS 운용은 화려한 서비스 조합보다, 적절한 계산 자원과 적절한 저장 위치와 적절한 권한을 고르는 기초에서 시작한다. 이 기본기가 잡혀 있어야 뒤에서 다룰 Batch, EMR, HealthOmics, Hail, Bedrock도 자연스럽게 이해할 수 있다.

## 핵심 개념 정리

- EC2는 계산 자원이고, 인스턴스 패밀리는 병목의 종류를 반영한다.
- EBS는 인스턴스에 붙는 블록 스토리지이고, S3는 장기 보관과 공유에 적합한 객체 스토리지다.
- FSx for Lustre는 여러 노드가 함께 쓰는 고성능 파일시스템이 필요할 때 고려하는 보조 계층이다.
- S3 storage class는 저장 단가만이 아니라 retrieval fee, minimum duration, 복원 지연까지 함께 봐야 한다.
- IAM Role은 액세스 키를 코드에 넣지 않고 실행 주체에 임시 권한을 주는 기본 보안 장치다.

## 복습 질문

1. `c` 계열과 `r` 계열 인스턴스는 각각 어떤 유전체 workload에 더 잘 맞는가?
2. 정렬 중간 파일과 최종 VCF를 같은 저장소에 두지 않는 이유는 무엇인가?
3. IAM User 대신 IAM Role을 우선 고려해야 하는 이유를 실제 EC2와 S3 예시로 설명해 보라.

## Further Reading

- AWS Samples. [IAM role creation example in `aws-genomics-workflows`](https://github.com/aws-samples/aws-genomics-workflows/blob/master/docs/core-env/create-iam-roles.md).
- AWS. [Using Amazon FSx for Lustre for genomics workflows on AWS](https://aws.amazon.com/blogs/storage/using-amazon-fsx-for-lustre-for-genomics-workflows-on-aws/).
- AWS. [Secure your genomic workflows and data with AWS HealthOmics](https://aws.amazon.com/blogs/industries/secure-your-genomic-workflows-and-data-with-aws-healthomics/).

## References

- AWS. 2024a. [Amazon EC2 instance type names](https://docs.aws.amazon.com/ec2/latest/instancetypes/instance-type-names.html).
- AWS. 2024b. [Amazon S3 storage classes overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html).
- AWS. 2024c. [IAM best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html).
- AWS. 2024d. [Using Amazon FSx for Lustre for genomics workflows on AWS](https://aws.amazon.com/blogs/storage/using-amazon-fsx-for-lustre-for-genomics-workflows-on-aws/).
- AWS. 2024e. [Secure your genomic workflows and data with AWS HealthOmics](https://aws.amazon.com/blogs/industries/secure-your-genomic-workflows-and-data-with-aws-healthomics/).
