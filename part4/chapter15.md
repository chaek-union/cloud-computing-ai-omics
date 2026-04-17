# Chapter 15. 시퀀싱 데이터에서 변이 해석까지 - 통합 예제

## 학습 목표

- 시퀀싱 센터 데이터 handoff부터 최종 질의와 해석까지의 전체 흐름을 설명할 수 있다.
- data lake, lakehouse, warehouse가 하나의 워크플로 안에서 어떻게 이어지는지 이해할 수 있다.
- S3, HealthOmics, Athena, Bedrock, Quick이 각각 어느 단계에서 역할을 하는지 설명할 수 있다.

## 핵심 질문

- 시퀀싱 회사에서 데이터를 넘겨받은 뒤, 무엇부터 구조화해야 하는가
- 원시 데이터, 분석용 테이블, 요약 리포트를 왜 같은 저장 방식으로 다루면 안 되는가
- 변이 해석까지 가는 통합 워크플로에서 AWS 서비스들은 어떤 순서로 연결되는가

## 15.1 통합 워크플로를 하나의 이야기로 보기

이 책의 앞 장들은 개별 서비스와 개별 개념을 설명했지만, 실제 연구실 운영은 늘 통합 워크플로의 형태로 나타난다. 시퀀싱 회사에서 FASTQ나 CRAM이 도착하고, 이를 등록하고, 전처리와 정렬과 변이 호출을 돌리고, annotation을 붙이고, cohort 수준으로 질의하고, 마지막에 해석과 보고를 수행하는 과정은 하나의 긴 파이프라인이다. 학생에게는 이 흐름을 서비스별 목록보다 `데이터가 어떻게 상태를 바꾸며 이동하는가`로 보여 주는 편이 훨씬 이해하기 쉽다. 같은 AWS라도 어떤 단계는 data lake에 가깝고, 어떤 단계는 lakehouse에, 어떤 단계는 warehouse에 가깝다. 따라서 통합 예제 장에서는 기술 명칭보다 데이터의 생애주기를 먼저 보는 것이 중요하다.

가장 단순한 실전 예제는 다음과 같이 그릴 수 있다. vendor 또는 시퀀싱 센터가 데이터를 S3나 HealthOmics sequence store로 전달한다. 연구실은 이를 sample metadata와 연결하고, Nextflow 또는 HealthOmics workflow를 돌려 정렬과 variant calling을 수행한다. 결과 VCF와 annotation table은 Athena나 Spark에서 질의 가능한 구조로 정리되고, 마지막에는 Bedrock이나 dashboard 계층을 통해 사람이 해석하고 공유한다. 이 흐름은 단순한 자동화가 아니라, 원시 파일이 점점 더 구조화된 연구 자산으로 변해 가는 과정이다.

## 15.2 Vendor handoff - read set 등록과 메타데이터 정리

통합 워크플로의 첫 단계는 파일 자체보다 메타데이터다. 시퀀싱 센터에서 파일이 도착했다고 해서 분석 준비가 끝난 것은 아니다. 샘플 ID, 라이브러리 정보, 시퀀싱 플랫폼, 참조 유전체 버전, 승인 상태, 프로젝트 코드, 환자 또는 코호트와의 연결 규칙이 함께 정리되어야 한다. HealthOmics sequence store와 read set 개념이 유용한 이유는 바로 이 단계에서 데이터의 등록과 provenance를 더 명시적으로 관리할 수 있기 때문이다. 단순히 폴더에 파일을 복사해 두는 방식은 빠를 수 있지만, 이후 누가 어떤 샘플을 어떤 파이프라인으로 분석했는지 추적하기가 훨씬 어려워진다.

실전에서는 메타데이터 JSON이나 manifest 파일이 워크플로의 진짜 출발점이 된다. 예를 들어 `sample_id`, `read_set_id`, `reference`, `pipeline_version`, `phenotype_group` 같은 정보가 구조화되어 있으면, 이후 Lambda나 Step Functions, HealthOmics workflow가 이를 기준으로 자동화될 수 있다. 반대로 파일만 있고 메타데이터가 흩어져 있으면, 아무리 좋은 클라우드 인프라를 가져와도 운영은 쉽게 불안정해진다. 학생에게는 `파일을 받는다`보다 `샘플을 등록한다`는 표현을 가르치는 편이 더 정확하다. 분석은 파일을 대상으로 하는 것 같지만, 실제 운영은 등록된 샘플과 메타데이터를 대상으로 이루어지기 때문이다.

## 15.3 Data lake, lakehouse, warehouse가 한 파이프라인 안에서 만나는 방식

omics 연구에서 data warehouse, data lake, lakehouse는 경쟁 개념이 아니라 서로 다른 데이터 상태를 가리키는 편이 더 맞다. 원시 FASTQ, BAM, CRAM, raw image는 보통 data lake 계층에 가깝다. 이 단계의 핵심은 원본 보존과 유연한 저장이며, 스키마를 엄격히 강제하기보다 나중 분석을 위해 자산을 남겨 두는 데 의미가 있다. 반면 annotation이 붙은 variant table, cohort filtering table, count matrix summary, quality summary 같은 반복 질의용 데이터는 lakehouse 계층에 더 가깝다. 여기서는 Parquet, S3 Tables, Glue catalog, Athena 같은 구조가 중요해진다. 마지막으로 임상 리포트용 요약 표, 운영 대시보드, PI용 cohort summary는 warehouse에 가까운 계층으로 볼 수 있다.

이 구분은 이론적 분류가 아니라 실제 설계 도구다. 원시 FASTQ를 바로 Athena로 질의하려 하거나, 임상 리포트를 만들 데이터를 raw data zone에 그대로 두면 운영이 금방 복잡해진다. 반대로 반복 질의가 필요한 annotation 결과를 계속 VCF와 TSV 상태로만 남겨 두면, cohort 분석과 dashboard 작성이 매우 비효율적이 된다. 따라서 통합 예제에서는 `원시 데이터는 lake에, 반복 질의 데이터는 lakehouse에, 요약과 보고는 warehouse에`라는 큰 원칙을 분명히 해 두는 것이 좋다. 학생은 이를 통해 서비스 선택보다 먼저 데이터 계층을 나누는 사고를 배우게 된다.

Table 1. 통합 오믹스 워크플로에서의 데이터 계층

| 계층 | 대표 데이터 | AWS 예시 |
|---|---|---|
| Data lake | FASTQ, BAM, CRAM, raw image, read set | S3, HealthOmics sequence store |
| Lakehouse | annotation table, cohort filtering table, QC summary | S3 Tables, Parquet, Glue catalog, Athena |
| Warehouse | dashboard dataset, clinical report mart, 운영 요약표 | Athena 결과셋, Amazon Quick, 별도 보고 계층 |

## 15.4 전처리와 분석 - HealthOmics, Nextflow, Batch의 역할

원시 데이터가 등록되면 다음 단계는 workflow 실행이다. 이때 연구실은 크게 두 가지 길을 선택할 수 있다. 하나는 HealthOmics처럼 관리형 워크플로 계층을 사용하는 것이고, 다른 하나는 Batch와 Nextflow 같은 범용 조합을 직접 운영하는 것이다. 전자는 실행 환경을 표준화하고 재시도와 provenance를 더 쉽게 관리하는 데 강하고, 후자는 더 큰 자유도와 커스터마이징을 제공한다. 통합 예제 장에서는 어느 것이 더 우월하다고 말하기보다, `관리형과 직접 운영은 운영 철학이 다르다`는 점을 보여 주는 편이 좋다.

예를 들어 WGS나 panel 분석의 전처리 파이프라인은 `input manifest -> alignment -> variant calling -> QC -> output VCF/BAM`의 큰 흐름을 가진다. 이 과정은 workflow 실행 단계에서는 여전히 compute 중심처럼 보이지만, 사실은 동시에 데이터 계층을 바꾸는 단계이기도 하다. 원시 FASTQ는 정렬과 변이 호출을 거치며 분석 결과 자산으로 변하고, 이후 annotation과 cohort query를 위해 다른 구조로 다시 정리된다. 학생은 여기서 "파이프라인이 끝났다"보다 "데이터가 다음 계층으로 변환되었다"는 시각을 가져야 한다. 그래야 후속 Athena 질의와 Bedrock 인터페이스가 왜 필요한지 자연스럽게 이해된다.

## 15.5 테이블화와 cohort query - Athena가 들어오는 지점

변이 해석으로 가기 전에 반드시 필요한 단계가 구조화된 테이블화다. VCF 자체는 여전히 매우 중요하지만, cohort 질의나 운영 요약, phenotype과의 결합, frequency 비교 같은 작업에는 컬럼형 테이블과 SQL 계층이 훨씬 편리한 경우가 많다. Athena는 바로 이 지점에서 강점을 가진다. annotation이 끝난 variant table이나 sample-level QC table, phenotype metadata를 Glue catalog와 연결해 두면, 연구자는 복잡한 파서 없이도 반복 질의를 수행할 수 있다. 이 과정이 바로 lakehouse 사고의 실전적 구현이다.

학생에게는 이 단계가 왜 중요한지 구체적 질문으로 보여 주는 것이 좋다. 예를 들어 `BRCA1과 BRCA2의 pathogenic variant를 가진 샘플은 몇 명인가`, `이 phenotype group에서 rare damaging variant burden은 어떤가`, `특정 ancestry group에서 allele frequency가 다른가` 같은 질문은 Athena와 구조화된 테이블이 있을 때 훨씬 자연스럽다. 반대로 모든 것을 VCF 원본 위에서만 해결하려 하면, 분석은 가능하더라도 반복성과 협업성이 크게 떨어진다. 즉 변이 해석의 시작은 모델이 아니라, 질의 가능한 테이블을 만드는 일이다.

## 15.6 자연어 해석과 보고 - Bedrock과 Quick의 위치

마지막 단계에서 비로소 생성형 AI와 시각화 계층이 등장한다. Bedrock은 구조화된 데이터가 이미 준비된 뒤에, 자연어 질문을 더 쉽게 던지고 결과를 요약하고 보고 초안을 만드는 해석 계층으로 이해하는 편이 정확하다. 이것은 `AI가 원시 VCF를 이해한다`는 뜻이 아니다. 오히려 앞 단계에서 annotation, 테이블화, provenance 정리가 충분히 이루어졌기 때문에, 마지막에 자연어 인터페이스가 유용해지는 것이다. 따라서 학생에게는 `AI가 먼저가 아니라 데이터 정돈이 먼저`라는 메시지를 분명히 남겨야 한다.

Amazon Quick 같은 시각화 계층도 비슷하다. Quick은 직접 유전체 계산을 수행하는 서비스가 아니라, cohort dashboard, 운영 지표, 결과 공유를 위한 상위 요약 계층에 더 가깝다. 즉 Bedrock이 질의와 설명을 돕는 자연어 계층이라면, Quick은 요약과 관찰을 돕는 시각화 계층이다. 통합 워크플로의 끝은 반드시 거대한 알고리즘이 아니라, 사람이 이해하고 공유할 수 있는 형태로 결과를 정리하는 것이다. 이 점까지 포함해야 시퀀싱 데이터에서 해석까지의 흐름이 완성된다.

## 핵심 개념 정리

- 통합 오믹스 워크플로는 `vendor handoff -> workflow -> table화 -> 질의 -> 해석`의 흐름으로 보는 것이 가장 직관적이다.
- 원시 데이터, 분석용 테이블, 요약 보고용 데이터는 같은 계층에 두지 않는 것이 좋다.
- Athena와 같은 SQL 계층은 변이 해석 이전에 cohort 질의를 구조화하는 데 매우 중요하다.
- Bedrock과 Quick은 분석의 시작점이 아니라, 구조화된 데이터 위에 얹히는 해석과 시각화 계층이다.

## 복습 질문

1. 시퀀싱 센터에서 파일이 도착한 뒤, 왜 파일 복사보다 메타데이터 등록이 먼저인가?
2. data lake, lakehouse, warehouse는 통합 omics 워크플로 안에서 각각 어떤 역할을 하는가?
3. Bedrock과 Quick은 왜 raw data 단계가 아니라 구조화된 분석 결과 단계에서 더 유용한가?

## Further Reading

- AWS. [Sequence stores](https://docs.aws.amazon.com/omics/latest/dev/sequence-stores.html).
- AWS. [Workflow definition specifics for Nextflow](https://docs.aws.amazon.com/omics/latest/dev/workflow-definition-nextflow.html).
- AWS Machine Learning Blog. [Accelerating genomics variant interpretation with AWS HealthOmics and Amazon Bedrock AgentCore](https://aws.amazon.com/blogs/machine-learning/accelerating-genomics-variant-interpretation-with-aws-healthomics-and-amazon-bedrock-agentcore/).

## References

- AWS. 2026a. [Sequence stores](https://docs.aws.amazon.com/omics/latest/dev/sequence-stores.html).
- AWS. 2026b. [Accessing HealthOmics read sets with Amazon S3 URIs](https://docs.aws.amazon.com/omics/latest/dev/s3-access.html).
- AWS. 2026c. [Workflow definition specifics for Nextflow](https://docs.aws.amazon.com/omics/latest/dev/workflow-definition-nextflow.html).
- AWS. 2026d. [What is Amazon Athena?](https://docs.aws.amazon.com/athena/latest/ug/what-is.html).
- AWS. 2026e. [What is Amazon Quick?](https://docs.aws.amazon.com/quick/latest/userguide/what-is.html).
- AWS Machine Learning Blog. 2025. [Accelerating genomics variant interpretation with AWS HealthOmics and Amazon Bedrock AgentCore](https://aws.amazon.com/blogs/machine-learning/accelerating-genomics-variant-interpretation-with-aws-healthomics-and-amazon-bedrock-agentcore/).
- AWS Industries Blog. 2024. [Orchestrating multiple AWS HealthOmics workflows at scale](https://aws.amazon.com/blogs/industries/orchestrating-multiple-aws-healthomics-workflows-at-scale/).
