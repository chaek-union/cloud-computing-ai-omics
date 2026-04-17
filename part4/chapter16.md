# Chapter 16. 비용, 보안, 재현성을 함께 설계하기

## 학습 목표

- 클라우드 분석에서 비용, 보안, 재현성이 왜 동시에 중요해지는지 설명할 수 있다.
- 유전체 데이터의 민감성과 접근 제어 원칙을 이해할 수 있다.
- 태그, 로그, 버전 관리, 환경 기록 같은 운영 습관을 설명할 수 있다.

## 핵심 질문

- 왜 계산보다 저장과 전송이 더 큰 비용이 될 수 있는가
- 공개 데이터와 민감한 임상 데이터는 어떻게 다르게 다뤄야 하는가
- 실험을 다시 재현하려면 무엇을 남겨야 하는가

## 16.1 비용 구조 이해하기

클라우드 분석의 비용을 이야기할 때 초보자는 보통 인스턴스 시간부터 떠올린다. 실제 청구서에서 눈에 띄는 항목이 compute인 경우가 많기 때문이다. 그러나 오믹스 분석이 커질수록 비용은 훨씬 더 복합적인 구조를 가진다. 데이터를 얼마나 오래 저장하는지, 몇 번 다시 읽는지, 리전이나 가용 영역을 넘어 몇 번 이동하는지, 실패한 작업을 얼마나 자주 처음부터 다시 실행하는지, 로그를 얼마 동안 남기는지가 모두 비용을 만든다. 다시 말해 클라우드 비용은 "어떤 서버를 썼는가"보다 "어떤 생애주기로 데이터를 굴렸는가"에 더 가깝다.

이 점은 오믹스 분석에서 특히 중요하다. FASTQ와 BAM, CRAM, VCF는 파일 하나하나가 크고, 중간 산물은 순간적으로 더 커지기 쉽다. 정렬과 recalibration, joint calling, annotation, cohort aggregation을 거치는 동안 같은 샘플이 여러 형식으로 존재할 수 있고, 각 단계의 임시 파일이 예상보다 오래 남으면 저장비가 조용히 누적된다. 반면 compute는 대개 실행이 끝나면 비용도 멈춘다. 그래서 경험이 쌓일수록 연구자는 "비용은 서버가 아니라 생애주기에서 샌다"는 감각을 갖게 된다.

Table 1은 교육용으로 단순화한 비용 구조다. 이 표의 핵심은 절대액을 외우는 것이 아니라, 어떤 설계 선택이 어떤 종류의 비용으로 이어지는지를 연결해서 보는 데 있다. 이 장에서는 원칙 수준에서만 다루고, 세부 누수 지점은 다음 장에서 더 실전적으로 해부한다.

Table 1. 유전체 클라우드 분석의 대표 비용 축

| 비용 축 | 대표 원인 | 초보자가 놓치기 쉬운 점 |
|---|---|---|
| 컴퓨트 | 과도한 인스턴스 크기, 잘못된 instance family, 불필요한 재실행 | 가장 잘 보이지만 동시에 가장 통제하기 쉬운 항목이다 |
| 스토리지 | 원본 장기 보관, 중간 산물 방치, lifecycle 미설계 | 분석이 끝난 뒤에도 계속 남아 청구서를 만든다 |
| 요청 비용 | 작은 파일 폭증, 반복적인 LIST/GET, 무분별한 테이블 스캔 | 저장 단가보다 눈에 덜 띄지만 장기적으로 누적된다 |
| 데이터 이동 | 리전 간 이동, AZ 간 이동, 외부 다운로드 공유 | "같은 AWS 안이니 무료"라고 오해하기 쉽다 |
| 반복 실행 | resume 부재, checkpoint 부재, 환경 불일치로 인한 재시도 | 비용이자 재현성 저하의 신호다 |

스토리지 계층 선택도 이 틀 안에서 이해해야 한다. `S3 Standard`는 단순하고 빠르지만 계속 비싸고, `Intelligent-Tiering`은 접근 패턴이 흔들리는 데이터에 유리하며, `Standard-IA`와 Glacier 계열은 장기 보관에 도움이 되지만 retrieval 비용과 최소 저장 기간을 함께 고려해야 한다. 여기서 중요한 것은 가장 싼 storage class를 고르는 일이 아니라, 지금 이 데이터가 `hot`, `warm`, `cold` 중 어디에 해당하는지를 꾸준히 구분하는 습관이다. 비용 최적화는 구매가 아니라 분류의 문제라는 점을 학생에게 초반부터 심어 줄 필요가 있다.

## 16.2 최소 권한, 암호화, 접근 제어

유전체 데이터는 공개 참조 자원과 민감한 환자 데이터가 한 프로젝트 안에 함께 존재하기 쉽다. 따라서 보안은 "모든 것을 다 잠근다"가 아니라, 데이터 등급에 따라 다른 접근 규칙을 적용하는 일이다. gnomAD나 SRA처럼 공개 접근이 전제된 자원은 직접 읽는 편이 자연스럽지만, 환자 샘플 기반 FASTQ나 임상 annotation 테이블은 같은 방식으로 다루면 안 된다. 연구실 운영에서 중요한 것은 이런 차이를 문장으로 명시하는 것이다. 어떤 데이터가 공개용인지, 어떤 데이터가 내부 분석 전용인지, 어떤 결과가 외부 협업자에게 공유 가능한지를 미리 정해 두지 않으면 기술적 설정도 일관되기 어렵다.

최소 권한 원칙은 이 구분을 AWS 권한 모델로 옮긴 것이다. 사람은 필요한 계정과 리소스만 보아야 하고, 워크플로는 필요한 입력을 읽고 필요한 출력 위치에만 쓸 수 있어야 한다. 장기 액세스 키를 노트북이나 스크립트에 넣어 두는 방식은 작은 연구실에서는 편해 보일 수 있지만, 팀이 커질수록 누가 어떤 권한으로 무엇을 했는지 추적하기 어려워진다. 따라서 사람은 IAM Identity Center나 federation을 통해 임시 권한으로 접근하고, 실행 주체는 IAM Role을 통해 필요한 범위만 허용하는 구조가 기본이 된다.

암호화도 마찬가지다. 저장 중 암호화와 전송 중 암호화는 이제 특별한 기능이 아니라 기본 운영 가정으로 보는 편이 맞다. 다만 암호화만으로 충분하다고 생각하면 안 된다. 누가 그 데이터에 접근할 수 있는지, 어떤 네트워크 경로를 타는지, 감사 로그를 남기는지까지 함께 설계해야 실제 보호가 된다. 민감한 임상 데이터가 포함된 프로젝트에서는 S3 버킷 정책, KMS 키 접근 권한, private network 경로, 로그 감사, 외부 공유 절차가 한 묶음으로 이해되어야 한다. 반대로 공개 데이터는 지나치게 폐쇄적으로 다루기보다, 적절한 direct access를 활용해 복사와 중복 저장을 줄이는 편이 더 나을 수 있다.

Table 2는 데이터 등급에 따라 접근 원칙이 어떻게 달라지는지 교육용으로 정리한 것이다. 이는 법률 자문표가 아니라 운영 감각을 잡기 위한 첫 지도다. 실제 프로젝트에서는 기관의 IRB, 계약, 데이터 사용 동의 범위, 국가별 규정이 더해질 수 있으므로, 민감 데이터는 항상 기관 정책과 함께 검토해야 한다.

Table 2. 데이터 등급별 기본 접근 원칙

| 데이터 유형 | 대표 예시 | 기본 원칙 |
|---|---|---|
| 공개 참조 데이터 | gnomAD, SRA public data, 공개 annotation | 가능한 한 직접 참조하고 불필요한 복사를 줄인다 |
| 내부 연구 데이터 | 비식별 cohort table, 내부 QC 결과 | 프로젝트 단위 권한과 태그로 접근 범위를 제한한다 |
| 민감 임상 데이터 | 환자 유래 read set, 임상 annotation, 식별 가능 메타데이터 | 최소 권한, 암호화, 감사 로그, 공유 승인 절차를 함께 설계한다 |

하이브리드 클라우드가 여전히 중요한 이유도 여기서 드러난다. 모든 데이터를 같은 규칙으로 한꺼번에 클라우드로 밀어 넣는 것이 정답은 아니다. 법적 또는 네트워크 제약이 큰 데이터는 기관 내부에 두고, 공개 데이터 활용과 대규모 병렬 계산은 AWS에서 수행하는 혼합 구조가 더 현실적일 수 있다. 중요한 것은 어디에 두느냐 자체보다, 데이터 등급과 권한 모델이 그 위치 선택과 맞물려 있어야 한다는 점이다.

## 16.3 컨테이너와 환경 고정

재현성은 흔히 Docker를 쓰면 끝나는 문제처럼 설명되지만, 실제로는 훨씬 넓다. 같은 결과를 다시 얻으려면 최소한 워크플로 정의, 컨테이너 이미지, 입력 파라미터, 참조 유전체 버전, annotation 자원 버전, 실행 시점의 서비스 설정을 함께 고정해야 한다. 스크립트 파일만 공유하는 방식은 종종 "무엇을 실행했는가"는 남겨도 "어떤 환경에서 실행했는가"를 충분히 남기지 못한다. 그래서 WDL, CWL, Nextflow 같은 workflow language가 중요하다. 이들은 파이프라인의 논리뿐 아니라 실행 단위와 입력, 출력, 의존성을 더 명시적으로 표현하게 만든다.

AWS HealthOmics의 private workflow 기능은 이런 관점에서 유용하다. HealthOmics는 WDL, CWL, Nextflow 정의를 직접 받아들이고, workflow versioning과 call caching 같은 기능을 제공한다. workflow versioning은 "업데이트된 워크플로가 언제부터 어떤 결과를 만들었는가"를 분리해서 기록하게 해 주고, call caching은 이미 계산된 태스크 출력을 재사용해 비용을 줄이는 동시에 같은 작업을 불필요하게 반복하지 않게 해 준다. 이 기능들은 단순한 편의 기능이 아니라, 재현성을 운영 습관으로 끌어올리는 장치라고 보는 편이 맞다.

컨테이너도 태그만으로는 부족하다. `latest` 같은 느슨한 태그는 나중에 같은 이름이 다른 이미지를 가리킬 수 있기 때문이다. 따라서 재현성을 정말 중요하게 생각한다면 이미지 digest, workflow definition 버전, parameter JSON, reference build, annotation database 버전을 함께 남겨야 한다. 오믹스 분석에서는 GRCh37과 GRCh38의 차이만으로도 결과 해석이 크게 달라질 수 있고, ClinVar나 VEP annotation release 차이도 의미 있는 변화를 만든다. 결국 재현 가능한 분석은 "같은 코드를 다시 돌린다"보다 "같은 문맥을 다시 구성할 수 있다"에 더 가깝다.

## 16.4 동등성 검증과 provenance 남기기

재현성 논의에서 자주 빠지는 마지막 단계가 동등성 검증이다. 환경을 고정했다고 해서 결과가 자동으로 과학적으로 동일하다고 단정할 수는 없다. 실제 운영에서는 인스턴스 세대가 바뀌고, 워크플로가 새 버전으로 이전되고, FPGA나 GPU 같은 가속 계층이 바뀌기도 한다. 이때 중요한 것은 "문제 없이 돌아갔다"가 아니라 "동등한 결과를 만들었는가"를 검증하는 것이다. 이런 태도는 연구실 운영을 훨씬 성숙하게 만든다.

최근 AWS HPC 블로그의 F1 대 F2 평가 사례는 이를 매우 실무적으로 보여 준다. 이 검증은 단순한 성능 비교로 끝나지 않고, command provenance, DRAGEN metrics, VCF concordance를 함께 확인했다. 즉 더 빠르고 더 싸다는 주장만 한 것이 아니라, 동일한 파이프라인이 동일한 변이 결과를 유지하는지 `bcftools isec` 같은 도구로 검증했다. 이런 접근은 학생에게 재현성을 가르칠 때도 좋은 모델이 된다. 컨테이너를 썼다는 사실만으로 만족하지 말고, 바뀐 환경이 결과를 바꾸지 않았는지 실제 산출물 수준에서 확인해야 한다는 메시지를 주기 때문이다.

또한 바이트 수준 동일성과 과학적 동일성을 구분하는 감각도 필요하다. AWS HealthOmics storage 문서는 imported file과 exported file 사이에 bitwise equivalence가 항상 보존되지는 않을 수 있지만, HealthOmics ETag와 provenance metadata를 통해 read set의 semantic identity를 유지한다고 설명한다. 이는 교육적으로 아주 좋은 예시다. 과학적으로 같은 read set을 가리키는 정보와, 압축 방식이나 내부 표현 때문에 바이트가 정확히 일치하는지는 다른 문제라는 뜻이기 때문이다. 따라서 provenance는 단지 파일 이름을 적는 일이 아니라, 그 데이터가 어디서 왔고 어떤 의미적 동일성을 가지는지 남기는 일이다.

Table 3은 최소한 남겨 두어야 할 provenance 항목을 정리한 것이다. 이 목록이 완벽한 표준은 아니지만, 여기의 절반만 충실히 남겨도 재현성과 협업 품질은 크게 좋아진다.

Table 3. 재현성을 위한 최소 provenance 기록

| 항목 | 예시 | 남기는 이유 |
|---|---|---|
| 입력 데이터 위치 | S3 URI, sequence store read set ID | 어떤 원본을 썼는지 추적하기 위해 |
| 샘플 메타데이터 | subject ID, sample ID, library ID | 샘플 혼동을 방지하기 위해 |
| 워크플로 버전 | workflow name, version, git commit | 어떤 논리로 실행했는지 남기기 위해 |
| 컨테이너 정보 | image URI, digest | 실행 환경을 다시 구성하기 위해 |
| 파라미터 파일 | JSON/YAML parameter set | 같은 실행 조건을 복원하기 위해 |
| 참조 자원 | reference build, annotation release | 해석 차이의 원인을 추적하기 위해 |
| 동등성 검증 결과 | metrics diff, concordance summary | 새 환경 전환 시 결과 보존을 확인하기 위해 |

## 16.5 연구실 운영 규칙 만들기

좋은 운영 규칙은 연구를 느리게 만드는 절차가 아니라, 같은 실수를 반복하지 않게 해 주는 압축된 기억이다. 특히 학생과 포닥이 자주 바뀌는 연구실에서는 "다음 사람도 같은 방식으로 할 수 있게 만드는 규칙"이 곧 생산성이다. 이런 규칙은 길 필요가 없다. 오히려 짧고 반복 가능할수록 좋다.

예를 들어 다음과 같은 다섯 가지 원칙만 지켜도 연구실 운영 품질은 눈에 띄게 좋아진다. 첫째, 모든 실행에는 manifest와 parameter file을 남긴다. 둘째, 원본 경로와 결과 경로를 절대 섞지 않는다. 셋째, 태그 없는 주요 리소스는 만들지 않는다. 넷째, 30분 이상 걸리는 batch workload는 resume, retry, checkpoint 전략을 함께 설계한다. 다섯째, 로그 retention과 storage lifecycle을 기본값으로 명시한다. 이 다섯 가지는 비용을 줄이기 위한 규칙이면서 동시에 보안을 돕고, 재현성도 높이는 규칙이다.

Table 4. 연구실 운영 규칙의 첫 버전

| 규칙 | 이유 | 최소 구현 |
|---|---|---|
| 실행 전 manifest 고정 | 샘플 혼동과 재현성 저하를 막기 위해 | 입력 URI, sample ID, reference build를 표준 템플릿에 기록 |
| 원본과 결과 분리 | 덮어쓰기 사고를 막기 위해 | `raw/`, `processed/`, `reports/` prefix 고정 |
| 태그 필수화 | 비용 배분과 권한 제어를 쉽게 하기 위해 | `project_id`, `owner`, `data_class` 최소 태그 적용 |
| retry와 checkpoint 사용 | Spot, 실패 복구, 비용 절감을 위해 | workflow 수준 resume와 task retry 기본 활성화 |
| retention 명시 | 로그와 중간 산물의 무한 축적을 막기 위해 | log group retention과 lifecycle rule을 프로젝트 시작 시 설정 |

이 장의 핵심은 비용, 보안, 재현성이 서로 다른 세 과목이 아니라는 점이다. 같은 데이터를 오래 두고, 아무나 접근하게 하고, 실행 문맥을 남기지 않으면 결국 셋 모두가 동시에 나빠진다. 반대로 최소 권한, 표준 경로, 버전 고정, provenance 기록, lifecycle 설계를 초반부터 습관으로 만들면 셋 모두가 함께 좋아진다. 클라우드 운영이 어려운 이유는 요소가 많기 때문이 아니라, 이 요소들이 서로 얽혀 있기 때문이다. 이 얽힘을 이해하는 순간 연구자는 비로소 "서비스를 쓰는 사람"에서 "연구 인프라를 설계하는 사람"으로 한 단계 올라간다.

## 핵심 개념 정리

- 클라우드 비용은 compute만의 문제가 아니라 저장, 이동, 요청, 반복 실행이 함께 만드는 구조다.
- 보안은 모든 데이터를 똑같이 잠그는 일이 아니라 데이터 등급에 맞는 권한 모델을 설계하는 일이다.
- 재현성은 컨테이너 하나로 끝나지 않으며, workflow version, parameter, reference, provenance 기록까지 포함한다.
- 동등성 검증은 재현성의 마지막 단계이며, 환경 전환 시 결과가 실제로 유지되는지 확인하게 해 준다.

## 복습 질문

1. 왜 오믹스 분석에서는 compute 비용보다 저장과 전송과 재실행 비용이 더 크게 문제 될 수 있는가?
2. 민감한 임상 데이터와 공개 참조 데이터는 왜 같은 권한 모델로 다루면 안 되는가?
3. 컨테이너를 사용하고도 재현성이 부족해질 수 있는 이유를 provenance 관점에서 설명해 보라.

## Further Reading

- AWS. [IAM best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html).
- AWS HPC Blog. [Evaluating next-generation cloud compute for large-scale genomic processing](https://aws.amazon.com/blogs/hpc/evaluating-next-generation-cloud-compute-for-large-scale-genomic-processing/).
- Strozzi et al. [Scalable Workflows and Reproducible Data Analysis for Genomics](https://pubmed.ncbi.nlm.nih.gov/31278683/).

## References

- AWS. 2026a. [IAM best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html).
- AWS. 2026b. [HealthOmics storage](https://docs.aws.amazon.com/omics/latest/dev/sequence-stores.html).
- AWS. 2026c. [Private workflows in HealthOmics](https://docs.aws.amazon.com/omics/latest/dev/private-workflows.html).
- AWS. 2026d. [Amazon S3 storage classes overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html).
- AWS Batch. 2026. [Use Amazon EC2 Spot best practices for AWS Batch](https://docs.aws.amazon.com/batch/latest/userguide/bestpractice6.html).
- AWS HPC Blog. 2026. [Evaluating next-generation cloud compute for large-scale genomic processing](https://aws.amazon.com/blogs/hpc/evaluating-next-generation-cloud-compute-for-large-scale-genomic-processing/).
- AWS Industries Blog. 2023. [Secure your genomic workflows and data with AWS HealthOmics](https://aws.amazon.com/blogs/industries/secure-your-genomic-workflows-and-data-with-aws-healthomics/).
- Strozzi F, Janssen R, Wurmus R, et al. 2019. [Scalable Workflows and Reproducible Data Analysis for Genomics](https://pubmed.ncbi.nlm.nih.gov/31278683/).
