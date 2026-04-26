# 19장. 산업계 사례와 데이터 브로커로 배우는 AWS 오믹스 운영

## 학습 목표

- 제약사, 바이오텍, 플랫폼 기업이 AWS를 어떻게 다르게 사용하는지 분류할 수 있다.
- Seven Bridges, Basepair, Research Gateway 같은 데이터 브로커 또는 협업 플랫폼의 역할을 설명할 수 있다.
- AWS Marketplace가 무엇이며, SaaS/private offer 방식으로 어떤 유전체 AI 플랫폼을 조달할 수 있는지 설명할 수 있다.
- 연구실이 산업계 사례에서 무엇을 바로 배울 수 있는지 운영 원칙으로 정리할 수 있다.

## 핵심 질문

- 왜 어떤 조직은 직접 EC2와 Batch를 운영하고, 어떤 조직은 HealthOmics나 외부 플랫폼을 얹는가
- 데이터 브로커는 단순 저장 대행이 아니라 무엇을 대신해 주는가
- AWS Marketplace에서 사는 것과 직접 구축하는 것은 무엇이 다른가
- 산업계 사례에서 반복해서 보이는 공통 아키텍처는 무엇인가

## 산업계는 무엇을 AWS에 올리는가

산업계 사례를 볼 때 가장 먼저 버려야 할 오해는 "모두가 같은 이유로 AWS를 쓴다"는 생각이다. 제약사, 바이오텍, 진단 기업, 플랫폼 회사는 비슷한 서비스를 쓰더라도 전혀 다른 병목을 해결하려는 경우가 많다. 어떤 조직은 RNA-seq이나 WGS 파이프라인의 throughput을 높이기 위해 클라우드를 택하고, 어떤 조직은 공개 데이터와 임상 메타데이터를 한곳에 모으기 위해, 또 어떤 조직은 협업형 분석 플랫폼을 제공하기 위해 AWS를 선택한다. 즉 산업계의 AWS 사용 사례는 기술 그 자체보다 `무엇을 병목으로 느꼈는가`를 먼저 읽어야 이해가 된다. 이 장에서는 사례 나열보다 운영 모델의 차이를 보는 것이 더 중요하다.

가장 크게 보면 다섯 가지 운영 모델이 보인다. 첫째는 `직접 구축`이다. EC2, Batch, FSx, S3를 직접 조합해 파이프라인과 데이터 계층을 운영하는 방식이다. 둘째는 `관리형 서비스`다. HealthOmics처럼 workflow 운영 부담을 줄이는 계층을 얹는다. 셋째는 `협업 플랫폼` 또는 data broker 모델이다. Basepair, Seven Bridges처럼 데이터는 고객 환경에 두고 orchestration과 UI를 제공하는 경우가 많다. 넷째는 `hybrid cloud`다. 기관 내부 자원, science cloud, AWS를 함께 쓴다. 다섯째는 `marketplace procurement`다. 직접 플랫폼을 만들지 않고 AWS Marketplace를 통해 SaaS나 in-silico lab 플랫폼을 조달하는 방식이다. 이 다섯 가지를 구분하면 산업계 사례가 훨씬 덜 복잡해 보인다.

## Takeda, ImmunoScape, Goldfinch로 보는 scale-out 패턴

Takeda 사례는 관리형 서비스가 대규모 오믹스 운영에서 어떤 차이를 만드는지 보여 주는 대표 예다. AWS 소개에 따르면 Takeda는 on-prem Slurm 기반 RNA-seq 운영을 HealthOmics로 옮기면서 20,000 samples를 2일 만에 처리하고, 일일 10,000 samples 규모의 throughput과 70% 이상의 비용 절감 효과를 얻었다고 설명한다. 여기서 중요한 점은 속도 그 자체만이 아니다. immutable run log, scientist self-service, tag 기반 데이터 관리처럼 운영과 재현성의 개선이 함께 언급된다는 사실이 더 중요하다. 즉 클라우드는 단지 더 빨리 도는 서버가 아니라, 더 예측 가능하게 운영되는 분석 시스템을 만들기 위한 수단이 된다.

ImmunoScape 사례는 startup형 패턴을 보여 준다. 이 회사는 Nextflow와 AWS Batch를 중심으로 필요할 때만 확장하는 구조를 택했고, 샘플이 들어올 때만 인프라를 키우는 방식으로 운영했다. 이 구조의 핵심은 `spiky workload`다. 항상 큰 클러스터를 사 두는 것보다, 분석 수요가 몰릴 때만 확장하는 것이 더 합리적인 경우가 많다. Goldfinch와 Loka가 보여 준 구조적 변이 분석 사례도 비슷한 교훈을 준다. 많은 파일과 높은 I/O가 필요한 파이프라인에서 `Cromwell + AWS Batch + FSx for Lustre` 같은 조합은 단순히 S3만으로 운영할 때보다 훨씬 더 적절할 수 있다. 즉 산업계 사례는 서비스 이름보다 `어떤 병목에 어떤 조합이 맞았는가`를 보여 주는 교육 재료다.

Table 1. 산업계 사례에서 보이는 대표 운영 모델

| 운영 모델 | 대표 사례 | 주된 병목 | 핵심 AWS 계층 |
|---|---|---|---|
| 직접 구축 | Loka/Goldfinch | 파일 수, I/O, 구조적 변이 파이프라인 | Batch, FSx, S3 |
| 관리형 서비스 | Takeda | throughput, 운영 표준화, self-service | HealthOmics, S3 |
| startup scale-out | ImmunoScape | spiky workload, 빠른 생산화 | Nextflow, Batch, S3 |
| 협업 플랫폼 | Basepair, Seven Bridges | 인프라 진입 장벽, 공유와 provenance | IAM role 연결, managed UI |
| 조달형 플랫폼 | Helical | foundation model 기반 in-silico lab | AWS Marketplace, SaaS |

## Seven Bridges, Basepair, Research Gateway로 보는 data broker 패턴

data broker를 단순히 `데이터를 대신 저장해 주는 업체`로 설명하면 본질을 놓치게 된다. 이런 플랫폼의 핵심은 데이터 이동을 최소화하면서, 사용자가 인프라를 직접 엔지니어링하지 않고도 분석과 협업을 수행하게 해 주는 데 있다. Basepair는 고객 계정의 S3나 HealthOmics store에 IAM Role로 연결되는 connected cloud 모델을 강조한다. 즉 데이터는 고객 AWS 계정 안에 남아 있고, SaaS는 orchestration과 visualization 계층만 제공한다. 이 구조는 민감 데이터와 협업 플랫폼을 동시에 다루려는 조직에게 매우 매력적이다.

Seven Bridges는 더 오래된 세대의 대표적 data broker다. AWS 위에서 협업형 분석 플랫폼을 제공하며, CWL portability, NIH 데이터 접근, provenance 추적, GUI 기반 파이프라인 실행을 강하게 내세워 왔다. 여기서 학생이 배워야 할 점은, 모든 연구자가 인프라 엔지니어일 필요는 없다는 사실이다. 플랫폼은 자유도를 조금 줄이는 대신, 재현성과 협업과 접근성을 높이는 경우가 많다. Relevance Lab Research Gateway는 또 다른 변형으로, self-service catalog, shared S3 storage, AWS Batch/ParallelCluster, budget과 cost reporting을 묶어 기관 포털처럼 제공한다. 이 사례는 데이터 브로커가 단순 분석 도구를 넘어 기관 운영 플랫폼이 될 수도 있음을 보여 준다.

## AWS Marketplace와 private offer로 조달하는 유전체 AI 플랫폼

AWS Marketplace는 third-party software, data, services를 찾고 사고 배포하고 관리하는 curated catalog다. 유전체 책에서는 이를 단순한 앱 스토어보다 `조달 계층`으로 설명하는 편이 더 정확하다. 즉 사용자는 AWS bill과 계약 구조 안에서 외부 솔루션을 도입할 수 있고, 때로는 private offer를 통해 맞춤 가격과 조건을 협상할 수 있다. 학생이 이 구조를 이해하면, 모든 연구 조직이 직접 플랫폼을 만들 필요는 없다는 현실을 자연스럽게 받아들일 수 있다. 연구 인프라는 기술만이 아니라 조달과 계약의 문제이기도 하다.

Helical의 `Bio Foundation Model Platform for In-Silico Labs` listing은 이 조달형 모델을 설명하기 좋은 사례다. listing 기준 이 제품은 SaaS이며, private offer, deployed on AWS, 추가 인프라 비용 가능성 같은 구조를 가진다. 또한 `Model Library`, `Model Personalization`, `In-Silico Experiments` 세 모듈을 통해 single-cell, mRNA, DNA foundation model을 조합해 target identification, biomarker discovery, patient stratification, RNA design을 지원한다고 설명한다. 이 사례를 책에서는 "자동화 실험실"을 물리 실험실의 대체가 아니라 `foundation model + proprietary data adaptation + large-scale virtual experiment orchestration` 계층으로 설명하는 편이 정확하다. 다만 AWS Marketplace listing은 제품 개요와 계약 구조를 보여 주는 자료이지, 성능에 대한 독립 검증 자료가 아니라는 caveat도 함께 적어야 한다.

## 공개 데이터 민주화와 hybrid cloud의 공존

산업계 사례를 읽다 보면 한쪽에서는 데이터를 AWS 안에서 바로 읽는 공개 데이터 민주화가 진행되고, 다른 한쪽에서는 여전히 hybrid cloud가 중요한 현실이라는 사실이 동시에 드러난다. Amazon Science의 SRA 기사는 NIH Sequence Read Archive가 AWS Open Data Sponsorship Program을 통해 네이티브하게 접근 가능해졌다고 설명한다. 이 흐름은 `데이터가 있는 곳으로 계산을 가져가는` 방식이 공개 데이터 생태계에서 얼마나 중요해졌는지를 보여 준다. 그러나 모든 조직이 이 모델 하나로 수렴하는 것은 아니다. 법적, 윤리적, 네트워크적 이유로 일부 데이터는 여전히 기관 내부나 다른 science cloud에 남을 수 있다.

Kyoto University의 hybrid cloud 논문은 이런 현실을 잘 보여 준다. 대형 기관에서는 on-prem, 학술용 클라우드, 공용 클라우드 AWS를 함께 묶는 구성이 오히려 더 현실적일 수 있다. 학생은 여기서 "클라우드가 전통 인프라를 그대로 대체한다"는 단선적 서사를 버려야 한다. 실제 연구 인프라는 대개 혼합형이다. 중요한 것은 어느 한 플랫폼을 신앙처럼 택하는 것이 아니라, 데이터 위치를 유지하면서 계산과 워크플로 계층을 적절히 조합하는 능력이다.

## 연구실이 바로 가져올 수 있는 운영 원칙

산업계 사례를 연구실 수준으로 번역하면 몇 가지 운영 원칙이 남는다. 첫째, 모든 것을 직접 구축하려고 하지 말고, 팀의 역량과 반복 사용 패턴에 맞는 계층을 고른다. 둘째, 데이터는 가능한 한 원래 위치에 두고, orchestration과 해석 계층을 붙이는 방향을 우선 고려한다. 셋째, 플랫폼을 도입할 때는 기술보다 provenance, budget visibility, 협업 방식, access control이 더 중요한 경우가 많다. 넷째, 조달형 플랫폼을 사용할 때는 편의성만 보지 말고, 데이터가 어디에 남는지, 어떤 권한 모델을 쓰는지, 어떤 부분이 독립 검증되지 않았는지를 함께 점검해야 한다.

AWS 유전체 운영의 미래는 한 가지 서비스가 아니라, 데이터 위치를 유지하면서 compute와 workflow 계층, 그리고 필요할 경우 조달 계층까지 조합하는 능력에 있다. 어떤 조직은 HealthOmics로 갈 것이고, 어떤 조직은 Batch와 EKS를 유지할 것이며, 어떤 조직은 Basepair나 Seven Bridges 같은 data broker를 선택할 것이다. 학생은 이 다양성을 보고 혼란스러워할 필요가 없다. 오히려 같은 질문을 던지면 된다. `이 조직은 어떤 병목을 풀려고 이 패턴을 택했는가?` 이 질문에 답할 수 있다면 산업계 사례는 단순한 홍보 자료가 아니라 좋은 교과서가 된다.

## 핵심 개념 정리

- 산업계에서 AWS는 한 가지 방식으로 쓰이지 않으며, 직접 구축, 관리형 서비스, 협업 플랫폼, hybrid cloud, 조달형 플랫폼이 공존한다.
- data broker의 핵심은 저장 대행보다 `데이터 이동 최소화 + 협업 + provenance + 실행 단순화`에 있다.
- AWS Marketplace는 기술 선택이 아니라 조달 계층으로 이해하는 편이 맞다.
- 산업계 사례를 읽을 때는 서비스 이름보다 `해결하려던 병목`을 먼저 보는 습관이 중요하다.

## 복습 질문

1. Takeda, ImmunoScape, Basepair는 각각 어떤 병목을 풀기 위해 AWS를 사용했는가?
2. data broker 모델은 왜 모든 연구실이 직접 인프라를 운영하지 않아도 되게 만드는가?
3. AWS Marketplace를 단순한 앱 스토어보다 조달 계층으로 설명해야 하는 이유는 무엇인가?

## Further Reading

- AWS Industries Blog. [How Takeda accelerates large-scale bioinformatics with AWS HealthOmics](https://aws.amazon.com/blogs/industries/how-takeda-pharmaceuticals-accelerates-large-scale-bioinformatics-with-aws-healthomics/).
- Seven Bridges. [Amazon Web Services page](https://www.sevenbridges.com/amazon/).
- AWS Marketplace. [What is AWS Marketplace?](https://docs.aws.amazon.com/marketplace/latest/userguide/what-is-marketplace.html).

## References

- Amazon Science. 2020. [AWS democratizes access to the largest genomic sequences repository — NIH’s Sequence Read Archive](https://www.amazon.science/latest-news/aws-democratizes-access-to-the-largest-genomic-sequences-repository-nihs-sequence-read-archive).
- AWS APN Blog. 2020. [Accelerating genomics and high-performance computing on AWS with Relevance Lab Research Gateway Solution](https://aws.amazon.com/blogs/apn/accelerating-genomics-and-high-performance-computing-on-aws-with-relevance-lab-research-gateway-solution/).
- AWS Industries Blog. 2024a. [How Takeda Pharmaceuticals accelerates large-scale bioinformatics with AWS HealthOmics](https://aws.amazon.com/blogs/industries/how-takeda-pharmaceuticals-accelerates-large-scale-bioinformatics-with-aws-healthomics/).
- AWS Industries Blog. 2024b. [Biotech startup builds agile drug discovery pipeline with scalable compute](https://aws.amazon.com/blogs/industries/biotech-startup-builds-agile-drug-discovery-pipeline-with-scalable-compute/).
- AWS Solutions. 2025. [Basepair case study](https://aws.amazon.com/solutions/case-studies/basepair-case-study/).
- AWS Marketplace. 2026a. [What is AWS Marketplace?](https://docs.aws.amazon.com/marketplace/latest/userguide/what-is-marketplace.html).
- AWS Marketplace. 2026b. [SaaS products through AWS Marketplace](https://docs.aws.amazon.com/marketplace/latest/buyerguide/buyer-saas-products.html).
- AWS Marketplace. 2026c. [Private offers in AWS Marketplace](https://docs.aws.amazon.com/marketplace/latest/buyerguide/buyer-private-offers.html).
- Helical. 2026. [Bio Foundation Model Platform for In-Silico Labs](https://aws.amazon.com/marketplace/pp/prodview-5jyanvqdackqa).
- Seven Bridges. 2026. [Amazon Web Services page](https://www.sevenbridges.com/amazon/).
- Yokoyama et al. 2023. [Hybrid cloud for genome analysis](https://www.nature.com/articles/s41439-023-00231-2).
