# 12장. SageMaker로 배우는 오믹스 머신러닝

## 학습 목표

- SageMaker의 기본 역할을 설명할 수 있다.
- 오믹스 데이터 분석에서 SageMaker를 언제 고려할 수 있는지 이해할 수 있다.
- 노트북, 학습 작업, 모델 배포를 서로 구분할 수 있다.
- 전통적인 생물정보학 파이프라인과 머신러닝 워크플로우가 어떻게 연결되는지 설명할 수 있다.

## 핵심 질문

- SageMaker는 단순한 Jupyter 서버와 무엇이 다른가
- 유전체와 전사체 데이터 분석에서 어떤 유형의 모델 작업에 적합한가
- 전통적인 생물정보학 파이프라인과 머신러닝 워크플로우는 어떻게 연결되는가
- Notebook, Training Job, Endpoint를 각각 언제 써야 하는가

## SageMaker를 오믹스 분석에서 어디에 놓아야 하는가

오믹스 분석에서 SageMaker를 이해하는 가장 좋은 방법은 그것을 "생물정보학 파이프라인을 대체하는 도구"로 보지 않는 것이다. 정렬, 변이 호출, 발현 정량, QC처럼 원시 데이터를 분석 가능한 형태로 바꾸는 과정은 여전히 Nextflow, WDL, CWL, AWS Batch, HealthOmics 같은 워크플로 계층이 더 자연스럽다. 반면 SageMaker는 이렇게 정리된 표, feature matrix, embedding, label, 이미지 타일, sequence representation 위에 탐색, 모델 학습, 추론, 실험 관리, 배포를 얹는 계층에 더 가깝다. 즉 SageMaker는 생물정보학의 앞단을 대체하기보다, 그 위에 머신러닝과 상호작용형 분석을 올리는 역할을 맡는다.

서비스 이름의 변화도 여기서 짚고 넘어갈 필요가 있다. AWS 공식 문서에 따르면 2024년 12월 3일 기존 `Amazon SageMaker`는 `Amazon SageMaker AI`로 이름이 바뀌었고, 같은 날 "next generation of Amazon SageMaker"가 데이터, 분석, AI를 아우르는 상위 플랫폼으로 발표되었다. 따라서 오늘 문서를 읽을 때 `SageMaker`라는 이름은 두 층의 의미를 가진다. 하나는 모델을 만들고 학습하고 배포하는 `SageMaker AI`이고, 다른 하나는 Unified Studio, Lakehouse, Governance 등을 포함한 더 큰 통합 플랫폼이다. 학생이 이 날짜와 구조를 알고 있으면 문서 이름이 섞여 보여도 훨씬 덜 혼란스럽다 (AWS 2024a; AWS 2026a).

오믹스 관점에서 보면 이 구조는 오히려 자연스럽다. 데이터 저장과 대규모 워크플로 실행은 S3, HealthOmics, Batch, Athena 같은 계층에 맡기고, 그 결과를 가지고 분류, 회귀, 군집화, 차원 축소, 임베딩 생성, foundation model 학습과 추론을 수행하는 계층으로 SageMaker를 두면 된다. 따라서 이 장에서 SageMaker는 "Notebook 도구"로 좁게 보아서는 안 된다. 노트북은 진입점일 뿐이고, 실제 강점은 데이터와 계산과 모델 운영을 하나의 관리형 흐름으로 묶어 준다는 데 있다.

## Notebook, Studio, Unified Studio, 학습 작업은 어떻게 다른가

초보자가 가장 많이 헷갈리는 부분은 SageMaker의 화면과 실행 단위를 한 덩어리로 보는 것이다. 하지만 실제로는 역할이 분리되어 있다. 노트북은 탐색과 실험을 위한 대화형 환경이고, 학습 작업(training job)은 코드와 데이터를 받아 독립적으로 실행되는 관리형 배치 학습이며, 실시간 endpoint는 학습이 끝난 모델을 서비스형 추론으로 노출하는 계층이다. 따라서 "SageMaker를 쓴다"는 말은 단순히 JupyterLab을 켠다는 뜻이 아니라, 필요에 따라 다른 실행 모드를 선택한다는 뜻이다.

여기에 최근의 UI 변화가 겹친다. AWS 문서에 따르면 2023년 11월 30일 이전의 SageMaker Studio 경험은 `SageMaker Studio Classic`으로 이름이 바뀌었고, 기존 workload 유지용으로만 남아 있으며 신규 온보딩은 중단되었다. 이후의 `SageMaker Studio`는 최신 ML workflow용 웹 경험이고, 2024년 12월 3일 preview로 공개된 `SageMaker Unified Studio`는 데이터, SQL, AI, ML 도구를 한 프로젝트 안에서 함께 다루는 상위 통합 환경이다. 즉 오늘날 학생이 새로 배우는 기준점은 `Studio Classic`이 아니라 `Studio`와 `Unified Studio`라고 이해하는 편이 맞다 (AWS 2026b; AWS 2026c; AWS 2026d).

Table 1은 오믹스 분석에 필요한 주요 실행 모드를 정리한 것이다. 중요한 것은 어떤 화면이 더 최신인가보다, 어떤 작업이 대화형 탐색에 맞고 어떤 작업이 배치 실행에 맞는가를 구분하는 일이다.

Table 1. SageMaker의 주요 실행 모드

| 구성 요소 | 핵심 역할 | 오믹스 예시 |
|---|---|---|
| Notebook 또는 JupyterLab | 데이터 탐색, 시각화, 가벼운 실험 | PCA, UMAP, QC plot, feature engineering |
| SageMaker Studio | 최신 ML 중심 개발 환경 | 모델 실험, 코드 작성, job 관리 |
| SageMaker Unified Studio | 데이터와 AI를 함께 다루는 통합 프로젝트 공간 | Athena 결과 탐색, 데이터 자산 공유, ML 작업 연결 |
| Training Job | 독립적이고 재현 가능한 관리형 학습 | genotype matrix 분류기 학습, 딥러닝 모델 pretraining |
| Processing Job 또는 스크립트 실행 | 전처리와 feature preparation | VCF 정리, cohort matrix 변환, 임베딩 계산 |
| Real-time Endpoint | 실시간 추론 서비스 | sequence embedding API, variant effect scoring API |
| Batch Transform 또는 오프라인 추론 | 대량 예측 배치 실행 | 샘플 집합 전체에 대한 risk score 계산 |

## Amazon Genomics CLI와 Nextflow 결과를 SageMaker로 연결하기

오믹스 머신러닝의 첫 번째 실제 패턴은 "무거운 생물정보학 계산은 워크플로 엔진이 처리하고, 결과 해석과 모델링은 SageMaker가 맡는다"는 구조다. AWS HPC 블로그의 2022년 예제는 이를 매우 교육적으로 보여 준다. 이 글에서 저자들은 Amazon Genomics CLI(AGC)가 AWS Batch 기반 실행 환경을 자동 구성해 `fetchngs`로 SRA 데이터를 FASTQ로 바꾸고, 이어서 Sarek 파이프라인으로 variant calling을 수행한 다음, 최종 VCF 결과를 SageMaker notebook으로 넘겨 상호작용형 분석과 시각화를 진행한다. 이 흐름은 학생에게 매우 중요한 메시지를 준다. 머신러닝은 생물정보학 파이프라인 이후에 얹히는 "3차 분석 계층"이라는 것이다 (Nocaj et al. 2022).

이 패턴의 장점은 두 가지다. 첫째, 파이프라인 단계와 모델링 단계를 분리함으로써 각 단계에 가장 적합한 실행 환경을 선택할 수 있다. 정렬과 변이 호출은 대개 workflow engine, container, batch queue가 더 잘 맞는다. 반면 결과를 정리해 분류 모델을 실험하고 시각화하는 일은 노트북 환경이 더 자연스럽다. 둘째, 입력과 출력을 S3에 두면 두 계층이 느슨하게 연결된다. 즉 파이프라인이 끝난 뒤 생성된 VCF, TSV, parquet, summary table을 SageMaker가 그대로 읽어 이어서 탐색하고 학습할 수 있다.

학생이 이 구조를 이해하면 SageMaker를 불필요하게 과장하지 않게 된다. SageMaker는 BWA나 GATK를 대신 돌려 주는 서비스가 아니다. 하지만 BWA와 GATK가 만든 결과를 feature table로 정리하고, 유전자별 패턴을 시각화하고, 예측 모델이나 군집 모델을 훈련하는 데에는 매우 적합하다. 좋은 오믹스 시스템은 이 두 세계를 경쟁 관계로 놓지 않고, `workflow for transformation`과 `SageMaker for learning and interpretation`으로 분업시킨다.

## 1000 Genomes 예제로 보는 tertiary analysis

이 장의 두 번째 핵심 패턴은 `HealthOmics + Athena + SageMaker` 조합이다. AWS Industries 블로그는 2025년 1월 29일, 1000 Genomes 데이터를 사용해 population variation을 분석하는 흐름을 제시했다. 이 예제의 교육적 가치는 PCA 자체보다, 대규모 변이 데이터를 어떻게 tertiary analysis용 구조로 연결하는지 보여 준다는 데 있다. 글의 흐름은 대략 이렇다. 원본 변이 자산을 AWS에서 관리 가능한 저장 계층으로 가져오고, 필요한 cohort를 Athena로 질의해 좁힌 다음, SageMaker Studio에서 PCA와 시각화를 수행한다 (Tzouvanas 2025).

이 예제가 중요한 이유는 대규모 유전체 분석에서 머신러닝 전처리가 곧 데이터 질의 문제라는 사실을 잘 보여 주기 때문이다. PCA를 하려면 결국 샘플 x 변이 행렬 또는 요약 feature matrix가 필요하다. 그런데 그 행렬을 만들기 전 단계에서 이미 cohort filtering, annotation selection, population label 정리, missingness 처리 같은 일이 발생한다. 즉 오믹스 머신러닝은 "바로 모델부터 돌리는 일"이 아니라, 질의 가능한 저장 계층과 상호작용형 분석 계층을 연결하는 일에 훨씬 가깝다.

Table 2는 이 패턴을 학습용으로 요약한 것이다. 이 구조는 1000 Genomes에만 해당하지 않는다. 암 코호트, rare disease cohort, single-cell feature matrix, proteomics abundance table에도 거의 그대로 확장된다.

Table 2. 오믹스 ML에서 자주 쓰는 연결 패턴

| 단계 | 주된 서비스 | 의미 |
|---|---|---|
| 원시 또는 준정리 데이터 저장 | S3, HealthOmics | 재사용 가능한 데이터 자산 보관 |
| 메타데이터 필터링과 cohort selection | Athena 또는 SQL 계층 | 필요한 샘플과 feature 범위 좁히기 |
| feature matrix 생성 | Processing job, notebook, workflow 후처리 | 모델 입력 형태로 변환 |
| 탐색과 시각화 | Notebook, Studio, Unified Studio | PCA, clustering, QC, 가설 형성 |
| 모델 학습 | SageMaker Training Job | 재현 가능한 학습과 대규모 실험 |
| 배포 또는 배치 추론 | Endpoint, Batch Transform | 실제 사용 또는 대규모 scoring |

## 실용 모델에서 genomic foundation model까지

오믹스 머신러닝이라고 하면 초보자는 곧바로 거대한 딥러닝 모델부터 떠올리기 쉽다. 하지만 실제 연구 현장에서는 분류, 회귀, 차원 축소, 임베딩 생성, 이상치 탐지 같은 비교적 실용적인 작업이 여전히 매우 중요하다. 예를 들어 환자 샘플의 subtype 분류, 발현 signature 기반 score 예측, variant burden을 이용한 population clustering, single-cell embedding 계산은 모두 SageMaker가 잘 맞는 작업이다. 특히 입력이 이미 S3에 있고, 전처리 결과를 notebook으로 확인한 뒤 학습 작업으로 넘기고 싶을 때 SageMaker의 관리형 실행 모델이 강점을 보인다.

그 위에 더 큰 확장으로 foundation model 계열이 있다. AWS Machine Learning 블로그의 2024년 글은 AWS HealthOmics sequence store를 오믹스 데이터 저장 계층으로 사용하고, SageMaker Training을 이용해 HyenaDNA 계열 genomic language model을 pre-train하는 예제를 제시한다. 여기서 중요한 메시지는 "오믹스 ML의 최신 확장도 결국 S3 또는 HealthOmics에 정리된 데이터 위에서 관리형 학습 작업을 돌리는 구조"라는 점이다. 즉 foundation model이 등장해도 데이터 계층과 학습 계층의 분리는 여전히 유지된다 (Ariyawansa and Handley 2024).

다만 모든 팀이 foundation model stack을 직접 구축할 필요는 없다. 대부분의 연구실과 바이오텍 팀은 먼저 작은 supervised task, embedding 생성, notebook 기반 탐색에서 더 큰 가치를 얻는다. 학생에게는 이 순서를 짚어 주는 편이 중요하다. 먼저 `작은 문제를 안정적으로 푸는 SageMaker 사용법`을 익히고, 그다음에야 분산 학습과 대형 모델로 확장하는 것이 바람직하다. 특히 single-cell과 spatial omics의 데이터 접근 계층은 따로 복잡하므로, 그 부분은 [13장](chapter13.md)에서 다루는 chunked storage와 remote access 개념과 함께 보는 편이 정확하다.

## 비용과 운영 원칙을 어떻게 잡아야 하는가

SageMaker의 가장 흔한 오해는 "관리형 서비스니까 알아서 싸고 알아서 효율적일 것"이라는 기대다. 실제로는 Notebook, Training Job, Endpoint의 비용 구조가 다르므로 작업 종류에 맞춰 선택해야 한다. 잠깐 탐색하고 그림을 그릴 일이라면 대화형 노트북이 적합하지만, 수 시간 또는 수일이 걸리는 학습은 Training Job으로 분리하는 편이 좋다. 모델을 한 번만 대량 추론할 것이라면 실시간 endpoint보다 batch 방식이 낫다. 결국 비용 최적화의 핵심은 같은 도구를 오래 켜 두는 것이 아니라, 실행 모드를 목적에 맞게 분리하는 데 있다.

오믹스 분석에서는 데이터 위치도 중요하다. Training Job과 입력 S3 버킷, HealthOmics sequence store, Athena query result가 같은 리전 안에 있어야 운영이 단순하고 전송 비용 위험도 줄어든다. 또한 notebook에서 우연히 잘 돌아간 코드를 곧바로 논문용 생산 파이프라인으로 착각해서는 안 된다. 재현성 있는 학습을 원하면 코드 버전, 데이터 snapshot, 파라미터, 컨테이너 이미지를 분리해 관리형 job으로 승격시켜야 한다. 이 원칙은 작은 random forest에서 대형 genomic language model에 이르기까지 모두 같다.

SageMaker는 오믹스 분석의 전 과정을 혼자 담당하는 서비스가 아니다. 대신 S3와 HealthOmics에 정리된 데이터, Batch와 Nextflow가 만든 결과, Athena가 좁혀 준 cohort를 바탕으로, 탐색에서 학습과 배포까지 이어지는 관리형 ML 계층을 제공한다. 따라서 오믹스에서 SageMaker를 잘 쓴다는 것은 "클라우드에 Jupyter를 띄운다"가 아니라, 데이터 파이프라인과 모델 파이프라인의 경계를 분리하고 그 사이를 안정적으로 연결하는 일에 가깝다.

## 핵심 개념 정리

- SageMaker는 생물정보학 파이프라인을 대체하기보다, 그 결과 위에 머신러닝과 상호작용형 분석을 얹는 계층이다.
- 2024년 12월 3일 이후 `SageMaker AI`는 기존 ML 서비스 이름이고, `next generation of SageMaker`는 더 큰 통합 플랫폼 이름이다.
- Notebook, Training Job, Endpoint는 각각 탐색, 재현 가능한 학습, 서비스형 추론이라는 서로 다른 역할을 가진다.
- AGC, Nextflow, HealthOmics, Athena와 연결될 때 SageMaker는 오믹스 tertiary analysis의 핵심 계층이 된다.
- foundation model도 중요하지만, 대부분의 실전 가치는 작은 supervised task와 feature engineering, 시각화, 배치 학습에서 먼저 나온다.

## 복습 질문

1. SageMaker를 단순한 Jupyter 서버로 이해하면 놓치게 되는 핵심 기능은 무엇인가?
2. 오믹스 분석에서 workflow engine과 SageMaker가 각각 맡기 좋은 역할은 무엇인가?
3. `HealthOmics + Athena + SageMaker` 패턴이 tertiary analysis에 적합한 이유는 무엇인가?
4. Notebook, Training Job, Endpoint는 어떤 기준으로 선택해야 하는가?

## Further Reading

- AWS. [What is Amazon SageMaker AI?](https://docs.aws.amazon.com/sagemaker/latest/dg/whatis.html).
- AWS. [What is Amazon SageMaker Unified Studio?](https://docs.aws.amazon.com/sagemaker-unified-studio/latest/userguide/what-is-sagemaker-unified-studio.html).
- AWS HPC Blog. [Analyzing Genomic Data using Amazon Genomics CLI and Amazon SageMaker](https://aws.amazon.com/blogs/hpc/analyzing-genomic-data-using-amazon-genomics-cli-and-amazon-sagemaker/).

## References

- Ariyawansa S, Handley S. 2024. [Pre-training genomic language models using AWS HealthOmics and Amazon SageMaker](https://aws.amazon.com/blogs/machine-learning/pre-training-genomic-language-models-using-aws-healthomics-and-amazon-sagemaker/).
- AWS. 2024a. [Introducing the next generation of Amazon SageMaker](https://aws.amazon.com/about-aws/whats-new/2024/12/next-generation-amazon-sagemaker/).
- AWS. 2026a. [What is Amazon SageMaker AI?](https://docs.aws.amazon.com/sagemaker/latest/dg/whatis.html).
- AWS. 2026b. [Amazon SageMaker Studio](https://docs.aws.amazon.com/sagemaker/latest/dg/studio-updated.html).
- AWS. 2026c. [Amazon SageMaker Studio Classic](https://docs.aws.amazon.com/sagemaker/latest/dg/studio.html).
- AWS. 2026d. [What is Amazon SageMaker Unified Studio?](https://docs.aws.amazon.com/sagemaker-unified-studio/latest/userguide/what-is-sagemaker-unified-studio.html).
- Nocaj A, Guruacharya A, Choudhury O, Pang L. 2022. [Analyzing Genomic Data using Amazon Genomics CLI and Amazon SageMaker](https://aws.amazon.com/blogs/hpc/analyzing-genomic-data-using-amazon-genomics-cli-and-amazon-sagemaker/).
- Tzouvanas K. 2025. [Examine genomic variation across populations with AWS](https://aws.amazon.com/blogs/industries/examine-genomic-variation-across-populations-with-aws/).
