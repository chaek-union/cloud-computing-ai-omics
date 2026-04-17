# Chapter 3. 데이터는 옮기지 말고 연결하라 - gnomAD와 S3

## 학습 목표

- gnomAD가 S3 링크를 제공하는 이유를 설명할 수 있다.
- gnomAD의 Hail VDS가 왜 단순 압축 파일이 아니라 분석 친화적 데이터 표현인지 설명할 수 있다.
- 데이터 복사 중심 사고와 데이터 현지 분석(data-local analysis) 사고의 차이를 이해할 수 있다.
- 공개 유전체 데이터를 AWS에서 직접 참조하는 기본 전략을 설명할 수 있다.

## 핵심 질문

- 왜 예전에는 FTP나 Globus로 데이터를 옮겨야 했는가
- 왜 지금은 S3 링크를 연결하는 방식이 더 합리적인가
- gnomAD 같은 대규모 자원은 사용자에게 무엇을 바꾸게 했는가
- 왜 gnomAD의 VDS는 "큰 VCF 하나"와 다른 사고방식을 요구하는가

## 3.1 공개 데이터 배포 방식의 변화

대규모 유전체 공공 자원을 다루는 방식은 지난 몇 년 사이에 뚜렷하게 바뀌었다. 예전에는 연구자가 FTP나 Globus 같은 전송 경로를 통해 큰 파일을 자기 기관 서버로 내려받고, 그 복사본 위에서 필터링과 통계 분석을 수행하는 방식이 표준에 가까웠다. 이 방식은 데이터셋이 작고 갱신 주기가 느릴 때는 어느 정도 합리적이었다. 그러나 수십만 명에서 백만 명 규모의 코호트가 등장하면서 같은 데이터를 여러 기관이 반복해서 저장하고, 서로 다른 시점의 복사본을 유지하고, 매번 이동 비용을 감당하는 방식은 점점 비현실적이 되었다. 공개 데이터 자원이 클라우드 객체 스토리지에 놓이기 시작한 배경에는 바로 이 구조적 한계가 있다.

gnomAD는 이 변화를 설명하기에 가장 좋은 사례다. gnomAD 팀은 2020년 다중 클라우드 접근 공지에서, 데이터를 공개 클라우드에 두는 이유를 중복 복사 제거, 자원 절약, 다른 클라우드 데이터셋과의 통합, 그리고 더 넓은 접근성 확보라는 네 가지 관점에서 설명했다. 이 메시지는 단순히 다운로드 링크의 위치가 바뀌었다는 뜻이 아니다. 오히려 연구자가 대규모 참조 자원을 자기 서버로 가져오는 대신, 공용 객체 스토리지에 있는 자원을 직접 참조하고 그 근처에서 계산해야 한다는 새로운 분석 습관을 뜻한다. 이때 핵심 문장이 바로 `bring compute to data`이다. 즉 클라우드 시대의 유전체 분석은 파일을 먼저 옮기는 일보다, 어디에 있는 데이터를 어떤 방식으로 바로 읽을 것인가를 먼저 설계하는 일에 가깝다 (gnomAD team 2020; AWS Open Data Registry 2026).

## 3.2 gnomAD 데이터 포털과 S3 배포 구조

AWS Open Data Registry는 gnomAD를 공개 S3 자원으로 소개하고, 대표 버킷으로 `s3://gnomad-public-us-east-1/`를 안내한다. 초보자에게 중요한 점은 여기서 `S3 버킷(bucket)`과 `객체 경로(object path)`라는 개념을 함께 이해하는 것이다. 버킷은 데이터가 담기는 최상위 저장 공간이고, 개별 파일이나 테이블은 그 아래 객체 키를 통해 접근된다. 로컬 디스크의 폴더와 비슷하게 보일 수 있지만, 실제로는 POSIX 파일시스템이 아니라 객체 스토리지이므로 접근 방식과 비용 구조가 다르다. 오믹스 연구자 입장에서는 이것이 "어디에 있는 공개 자원을 어떻게 읽을 것인가"라는 실용적 질문으로 곧바로 연결된다.

gnomAD를 S3에서 제공한다는 사실은 두 가지 변화를 동시에 의미한다. 첫째, 사용자는 큰 참조 자원을 매번 개인 저장소에 복사하지 않아도 된다. 둘째, 같은 클라우드 안에서 EC2, EMR, Hail, Athena 같은 계산 계층이 공용 데이터를 직접 읽을 수 있으므로, 분석 출발점이 훨씬 일관되고 재현 가능해진다. 예를 들어 어떤 연구자가 같은 버전의 gnomAD release를 같은 S3 경로에서 읽는다면, 최소한 참조 자원의 위치와 버전에 대해서는 같은 출발선을 공유하게 된다. 따라서 S3 위의 gnomAD는 단순한 다운로드 저장소가 아니라, 협업용 공용 참조 레이어(shared reference layer)로 이해하는 편이 더 정확하다.

Table 1은 전통적 다운로드 방식과 S3 직접 참조 방식의 차이를 교육용으로 단순화한 것이다. 여기서 중요한 것은 "무조건 클라우드가 빠르다"가 아니라, 데이터 이동을 줄일수록 저장 중복과 운영 복잡성이 함께 줄어든다는 점이다. 특히 코호트가 커질수록 이동 비용과 정합성 관리 비용이 더 크게 작용하므로, 참조 자원은 가능한 한 공용 위치에 두고 계산을 옮기는 방식이 유리해진다. 이 점은 뒤에서 다룰 SRA, EMR, Hail, HealthOmics 장에도 반복해서 등장하는 공통 원리다.

Table 1. 공개 유전체 자원 접근 방식의 비교

| 항목 | 전통적 다운로드 방식 | S3 직접 참조 방식 |
|---|---|---|
| 출발점 | 파일을 기관 서버로 먼저 복사 | 공용 객체 스토리지 경로를 직접 참조 |
| 저장 요구 | 기관별 중복 복사본이 쉽게 늘어남 | 공용 자원을 공유하고 로컬 복사를 줄임 |
| 갱신 관리 | 새 release가 나오면 다시 동기화 필요 | 버전된 경로를 기준으로 직접 참조 가능 |
| 협업 | 각자 다른 복사본을 쓸 가능성 큼 | 같은 경로와 같은 release를 공유하기 쉬움 |
| 대규모 코호트 적합성 | 이동과 저장 비용이 커짐 | 계산을 데이터 가까이에서 수행하기 쉬움 |

## 3.3 Hail VDS - 파일이 아니라 분산 데이터셋으로 보기

gnomAD를 이야기할 때 자주 등장하는 `VDS`는 여기서 주의 깊게 설명해야 하는 개념이다. 초보자는 VDS를 단순히 "VCF보다 더 잘 압축한 형식"으로 이해하기 쉽지만, 실제로는 그보다 훨씬 분석 지향적인 표현이다. Hail 문서에 따르면 VariantDataset(VDS)은 cohort-level genomic data를 표현하기 위한 희소(sparse), 분리(split) 구조이며, `reference_data`와 `variant_data`를 별도의 MatrixTable로 나누어 저장한다. 다시 말해 모든 샘플과 모든 위치를 빽빽하게 펼쳐 저장하는 방식이 아니라, 참조 블록과 실제 변이 정보를 나누어 저장함으로써 대규모 코호트에 더 잘 맞는 질의와 계산을 가능하게 한다. 따라서 VDS는 저장 형식인 동시에 계산 구조다 (Hail 2026).

이 차이는 gnomAD v4.0 release note에서 매우 인상적인 숫자로 드러난다. 2023년 11월 공개된 v4.0 공지에 따르면, gnomAD는 release용 730,947 exomes를 만들기 위해 실제로 955,213 samples 규모의 전체 데이터를 처리했으며, 이 전체 v4 dataset은 VDS 형식 덕분에 18 TB만 사용했다. 같은 데이터를 전통적 project VCF로 저장했다면 897 TB가 필요했을 것이라고 설명한다. 이 수치는 VDS가 단순한 파일 포맷 차이를 넘어, 대규모 코호트를 어떻게 표현할 것인가의 문제임을 잘 보여 준다. 즉 VDS는 저장 공간만 아끼는 기술이 아니라, 대규모 callset을 분석 가능한 구조로 유지하기 위한 데이터 모델이라고 보는 편이 더 정확하다 (gnomAD Production Team 2023).

또한 VDS는 무엇을 저장하느냐만 바꾸는 것이 아니라, 무엇을 질문할 수 있느냐도 바꾼다. gnomAD v4.1 공지는 VDS 덕분에 all callable sites에서 allele number를 유지할 수 있었고, 변이가 관찰되지 않은 위치까지 포함한 allele number 파일을 제공할 수 있었다고 설명한다. 이는 기존 callset 처리에서 변이가 없는 위치의 정보가 쉽게 사라지던 문제와 대비된다. 학생 입장에서는 이것을 "저장 효율이 좋아졌다"보다 "질문 가능한 범위가 넓어졌다"라고 이해하는 편이 훨씬 유익하다. 유전체 데이터 구조는 단순히 디스크 공간을 아끼는 문제가 아니라, 과학적 해석의 범위를 결정하는 문제이기 때문이다 (Chao et al. 2024).

## 3.4 공용 참조 레이어와 재현성

공용 클라우드 자원의 가장 큰 장점 가운데 하나는 재현성과 협업의 출발점을 맞추기 쉽다는 점이다. 전통적 방식에서는 같은 gnomAD release를 연구실마다 따로 보관하고, 어느 시점에 어떤 버전을 받았는지, 어떤 전처리를 추가했는지, 로컬에서 어떤 인덱스를 다시 만들었는지 추적하기 어려운 경우가 많았다. 반면 공용 객체 스토리지에 있는 versioned release를 직접 참조하면, 적어도 참조 자원의 위치와 버전에 대해서는 훨씬 명확하게 기록할 수 있다. 이후 동일한 Hail 버전, 동일한 container image, 동일한 workflow를 결합하면 분석 출발점이 더 단단해진다. 재현성이란 단지 논문의 마지막 단계에서 코드를 공개하는 일이 아니라, 분석에 들어가기 전부터 같은 공용 레이어를 공유하는 운영 습관이라는 점을 이 장에서 강조할 필요가 있다.

gnomAD가 2024년 8월 browser Hail Tables를 공개한 것도 같은 흐름으로 읽을 수 있다. 이 발표의 핵심은 웹 브라우저 뒤에서만 보이던 데이터 구조가, 이제 연구자가 재사용 가능한 Hail Table 자산으로 직접 접근 가능해졌다는 점이다. 학생들에게는 이것이 매우 중요한 전환이다. 브라우저에서 보는 요약 정보와 실제 분석 자원이 더 가까워질수록, "웹에서 본 내용을 다시 로컬에서 재구성해야 하는 불필요한 노동"이 줄어든다. 공개 참조 자원이 단순한 보기용 포털을 넘어서 분석 가능한 테이블과 라이브러리 형태로 확장될수록, 공용 참조 레이어라는 개념은 더 강해진다 (gnomAD browser 2024).

## 3.5 Hail, Spark, Athena, 도구 생태계와의 연결

gnomAD를 S3에서 읽는다고 해서 모든 분석이 자동으로 해결되는 것은 아니다. 중요한 것은 그 위에 어떤 질의 계층과 어떤 도구를 얹느냐이다. Hail은 대규모 희소 유전체 데이터를 분산 연산 환경에서 다루는 대표적 도구이고, Spark 기반 분석 계층과 함께 사용할 때 진가를 발휘한다. AWS 쪽에서는 Athena-ready lakehouse 자산이나 공개 테이블 자원이 추가되면서, 반드시 거대한 VCF를 모두 내려받지 않아도 특정 cohort filter, annotation query, 빈도 확인 같은 작업을 더 직접적으로 설계할 수 있게 되었다. 즉 같은 공용 S3 자원이라도 어떤 도구로 접근하느냐에 따라 사용 경험은 크게 달라진다.

2025년 1월 공개된 gnomAD toolbox는 이 흐름을 현재형으로 잘 보여 준다. 이 공지는 v4 exomes와 genomes short variant release VCF 전체가 약 1.4 TB이므로, 모든 사용자가 이를 통째로 내려받아 처리하는 방식은 현실적이지 않다고 지적한다. 대신 toolbox를 통해 로컬 복사 없이 gnomAD data를 더 쉽게 질의하도록 돕는 방향을 제시한다. 이 사례는 "클라우드에 올려 두었다"는 사실만으로 충분하지 않고, 그 위에 query-friendly 도구와 테이블이 있어야 실제 접근성이 높아진다는 점을 잘 보여 준다. 따라서 이 장의 결론은 단순하다. gnomAD는 공개 다운로드 자원이면서 동시에 공용 분석 플랫폼의 일부이고, S3는 그 플랫폼의 기반 저장 계층이다.

## 핵심 개념 정리

- gnomAD가 S3 링크를 제공하는 이유는 데이터를 더 많이 복사하라는 뜻이 아니라, 있는 곳에서 바로 읽으라는 뜻이다.
- 객체 스토리지 위의 공개 자원은 협업용 공용 참조 레이어로 기능할 수 있다.
- Hail VDS는 단순 압축 포맷이 아니라, 대규모 코호트를 희소하고 분산 분석 친화적으로 표현하는 구조다.
- 공개 데이터의 가치가 커질수록 `download -> analyze`보다 `query -> access in place -> compute` 패턴이 더 중요해진다.

## 복습 질문

1. gnomAD가 S3를 통해 데이터를 공개하는 이유를 저장, 협업, 계산 관점에서 설명해 보라.
2. VDS가 전통적 project VCF와 다른 점은 무엇이며, 왜 이것이 대규모 코호트 분석에 중요한가?
3. 공용 객체 스토리지 위의 참조 자원이 재현성과 협업을 어떻게 바꾸는가?

## Further Reading

- AWS Open Data Registry. [Genome Aggregation Database (gnomAD)](https://registry.opendata.aws/broad-gnomad/).
- gnomAD browser. [gnomAD v4.0](https://gnomad.broadinstitute.org/news/2023-11-gnomad-v4-0/).
- gnomAD browser. [gnomAD toolbox](https://gnomad.broadinstitute.org/news/2025-01-gnomad-toolbox/).

## References

- AWS Open Data Registry. 2026. [Genome Aggregation Database (gnomAD)](https://registry.opendata.aws/broad-gnomad/).
- Chao K, Wilson M, Goodrich J, gnomAD Production Team. 2024. [gnomAD v4.1](https://gnomad.broadinstitute.org/news/2024-04-gnomad-v4-1).
- gnomAD browser. 2024. [Release gnomAD browser tables](https://gnomad.broadinstitute.org/news/2024-08-release-gnomad-browser-tables/).
- gnomAD browser. 2025. [gnomAD toolbox](https://gnomad.broadinstitute.org/news/2025-01-gnomad-toolbox/).
- gnomAD browser. 2026. [gnomAD v4.1.1](https://gnomad.broadinstitute.org/news/2026-03-gnomad-v4-1-1/).
- gnomAD Production Team. 2023. [gnomAD v4.0](https://gnomad.broadinstitute.org/news/2023-11-gnomad-v4-0/).
- gnomAD team. 2020. [Open access to gnomAD data on multiple cloud providers](https://gnomad.broadinstitute.org/news/2020-10-open-access-to-gnomad-data-on-multiple-cloud-providers/).
- Hail. 2026. [VariantDataset](https://hail.is/docs/0.2/vds/hail.vds.VariantDataset.html).
