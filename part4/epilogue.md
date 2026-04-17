# 에필로그. 클라우드 이후의 오믹스 연구

이 책을 통해 우리는 AWS의 개별 서비스보다 더 큰 변화를 보았다. 오믹스 연구는 더 이상 파일을 한 번 내려받아 개인 서버에서 끝내는 작업이 아니라, 데이터가 생성되고, 등록되고, 구조화되고, 질의되고, 해석되고, 공유되는 하나의 플랫폼적 과정이 되었다. 이 변화는 단지 계산 장소가 연구실 밖으로 옮겨갔다는 뜻이 아니다. 데이터가 생성되는 곳, 협업이 이루어지는 방식, 재현성을 확보하는 방법, 그리고 연구자가 코드를 쓰고 질문을 던지는 방식 자체가 함께 바뀌고 있다는 뜻이다. 클라우드는 유전체학을 더 빠르게 만들었을 뿐 아니라, 무엇을 연구 인프라라고 부를 것인가를 다시 정의하게 만들었다.

앞으로의 연구자는 한 가지 역할만으로는 충분하지 않을 가능성이 크다. 데이터 해석 능력만으로도, 인프라 운영 능력만으로도, 생성형 AI 도구 활용 능력만으로도 부족할 수 있다. 대신 좋은 연구자는 데이터의 위치를 이해하고, 분석 계층을 구조화하고, 비용과 재현성을 함께 설계하며, 필요할 때는 AI를 빠른 초안 작성과 질의 인터페이스로 활용할 줄 아는 사람이 될 것이다. 그러나 이 변화가 연구자를 기계적으로 만들 필요는 없다. 오히려 좋은 인프라가 갖추어질수록 연구자는 더 중요한 과학적 질문과 해석에 집중할 수 있다.

이 책의 마지막 메시지는 크고 거창한 것이 아니다. 자신의 연구 주제에 맞는 작은 첫 프로젝트를 하나 설계해 보는 것에서 시작하면 된다. 공개 데이터를 S3에서 직접 읽어 보는 일, 작은 Nextflow 파이프라인을 HealthOmics나 Batch에서 돌려 보는 일, Hail로 cohort 질의를 한 번 해 보는 일, annotation 결과를 Athena로 요약해 보는 일, AI 도구로 작은 분석 스크립트와 문서를 함께 만들어 보는 일 중 하나면 충분하다. 클라우드 이후의 오믹스 연구는 모든 것을 한꺼번에 배우는 사람의 것이 아니라, 데이터와 계산과 해석의 관계를 조금씩 더 잘 설계해 가는 사람의 것이기 때문이다.

## 10년의 기록

2026년 4월, 덴마크의 Novo Nordisk가 OpenAI와 전사 차원의 전략적 파트너십을 발표했다. 이 협력은 특정 연구팀이 AI 도구를 도입하는 수준이 아니라, 신약 발굴·제조·임상·상업 운영에 이르는 조직 전체를 AI 인프라 위에 재구성하겠다는 선언에 가깝다. 경영진은 이 파트너십의 목적을 "과학자를 대체하는 것이 아니라 supercharge하는 것"이라고 표현했고, 기술 도입과 함께 전사 AI 리터러시 교육까지 계약에 포함되었다 (Reuters 2026). Eli Lilly, Novartis 등 주요 제약사들도 비슷한 방향으로 움직이고 있으며, 이는 바이오 산업이 "AI를 실험적으로 써 보는 단계"에서 "AI를 인프라 자체로 깔아 두는 단계"로 옮겨 가고 있다는 신호다. 이 소식을 접하면서, 나는 AWS 클라우드를 유전체 연구에 써 온 지난 10년을 자연스럽게 돌아보게 되었다.

출발점은 박사후 연구원을 시작한 2015년 말이었다. UCSF 서버에 쌓여 있던 수백 TB의 유전체 데이터를 로컬에서 처리하다 대형 서버가 자주 멈췄고, 결국 James Simons 박사가 설립한 자폐 연구 재단(SFARI)의 지원 아래 AWS로의 이주를 결정했다. 처음에는 SGE cluster와 Intel Lustre를 직접 세우며 유전학을 전공한 연구자가 인프라 엔지니어 역할까지 겸해야 했지만, 시간이 흐르면서 htslib의 클라우드 네이티브 접근이 자리를 잡고, 관리형 서비스들이 하나둘 정리되면서, 연구자는 낯선 엔지니어링 문제 대신 본래의 과학적 질문에 다시 집중할 수 있게 되었다. 공동 연구자가 버킷 접근 권한만 받으면 같은 분석을 그대로 재실행할 수 있게 되면서, 재현성과 국제 협업의 감각도 함께 바뀌었다. 10년이 지난 지금, 그때 우리가 고생하며 쌓던 인프라는 대부분 한두 개의 관리형 서비스로 대체되었고, 산업계는 그 위에 foundation model과 오믹스 데이터를 묶는 다음 단계로 이미 넘어가 있다. 지금의 학생에게 건네고 싶은 말은 단순하다. 10년 뒤에 다루게 될 인프라는 이 책의 내용을 훨씬 넘어설 것이므로, 서비스 이름이 아니라 그 서비스가 풀려고 했던 문제를 읽는 감각을 만들어 두면 된다. 그 감각이 있으면 다음 10년의 바이오 인프라가 어떤 모양이 되든, 그 위에서 자기 연구 질문을 계속 던질 수 있다.

## Further Reading

- AWS. [Genomics on AWS resources](https://aws.amazon.com/health/genomics-resources/).
- AWS. [Genomics Data Transfer, Analytics, and Machine Learning on AWS](https://docs.aws.amazon.com/whitepapers/latest/genomics-data-transfer-analytics-and-machine-learning/introduction.html).

## References

- AWS. 2026a. [Genomics on AWS resources](https://aws.amazon.com/health/genomics-resources/).
- AWS. 2026b. [Genomics Data Transfer, Analytics, and Machine Learning on AWS](https://docs.aws.amazon.com/whitepapers/latest/genomics-data-transfer-analytics-and-machine-learning/introduction.html).
- Reuters. 2026. ["Wegovy-maker Novo Nordisk partners with OpenAI to speed drug development."](https://www.reuters.com/legal/litigation/wegovy-maker-novo-nordisk-partners-with-openai-speed-drug-development-2026-04-14/) April 14, 2026.
