# 4장. 공용 오믹스 데이터를 AWS로 가져오기 - SRA와 Open Data

## 학습 목표

- SRA 데이터를 AWS에서 다루는 기본 흐름을 설명할 수 있다.
- AWS Open Data 프로그램이 연구자에게 주는 의미를 이해할 수 있다.
- 공개 데이터 반입과 직접 참조의 차이를 구분할 수 있다.

## 핵심 질문

- SRA 데이터는 AWS에서 어떻게 접근하는가
- 모든 공개 데이터가 S3에 있는 것은 아닌데, 어떤 전략으로 가져와야 하는가
- AWS Open Data는 단순 저장소가 아니라 어떤 생태계를 만드는가

## "다운로드"에서 "직접 접근"으로 바뀐 패러다임

공용 유전체 데이터를 AWS에서 다룬다고 할 때 초보자가 가장 먼저 바로잡아야 할 표현은 `다운로드`라는 말이다. 특히 NCBI SRA(Sequence Read Archive)의 경우, AWS에서 데이터를 쓴다는 것은 많은 경우 NCBI 서버에서 무엇인가를 새로 요청해 받는 일이 아니라, 이미 S3에 공개되어 있는 데이터를 직접 접근(access)하는 일에 가깝다. NCBI의 `SRA in the Cloud`와 `Download SRA sequence data using AWS` 문서도 이 점을 설명한다. 즉 public SRA data는 AWS Registry of Open Data를 통해 S3와 HTTPS 경로로 제공되며, 연구자는 이를 EC2나 다른 AWS 계산 계층에서 바로 읽을 수 있다. 이 구조가 의미하는 바는 크다. 유전체 데이터 접근의 출발점이 `download -> analyze`에서 `query -> access in place -> compute in cloud`로 이동하고 있기 때문이다 (NCBI 2024a; NCBI 2024b).

이 변화는 단순한 편의성 이상의 의미를 가진다. 예전에는 원하는 run accession을 찾은 다음, 이를 FTP나 다른 경로로 길게 내려받고, 로컬 스토리지에 저장한 뒤, 다시 분석 환경으로 옮기는 단계가 필수에 가까웠다. 그러나 공용 데이터가 이미 S3 안에 있다면, 같은 클라우드 안의 계산 자원이 이를 직접 읽어 올 수 있다. 따라서 병목은 전송 자체보다 메타데이터를 어떻게 검색하고, 어떤 accession만 선택하고, 어떤 형식으로 바로 후속 파이프라인에 연결할 것인가로 이동한다. 이 장에서 SRA는 단순한 다운로드 실습이 아니라, 공용 유전체 데이터 접근 패러다임이 어떻게 바뀌고 있는지를 보여 주는 대표 사례로 읽어야 한다.

## S3 direct access와 `--no-sign-request`

SRA 데이터를 AWS에서 다루는 가장 클라우드 네이티브한 방식은 direct S3 access이다. NCBI 문서는 `s3://sra-pub-run-odp/`, `s3://sra-pub-src-2/`, `s3://sra-pub-sars-cov2/` 같은 공개 버킷을 예시로 들며, `aws s3 ls s3://sra-pub-run-odp/ --no-sign-request`와 같은 명령으로 일부 데이터를 계정 없이도 조회할 수 있다고 설명한다. 이 예시는 교육적으로 매우 중요하다. 많은 초보자가 "클라우드 접근은 복잡한 인증과 비용이 따라붙는다"고 생각하지만, 공개 SRA 자산의 상당 부분은 적어도 탐색 단계에서 매우 직접적으로 접근할 수 있기 때문이다. 즉 공용 데이터에 대해 AWS는 때로 로그인한 조직 사용자만의 환경이 아니라, 공개 연구 인프라의 일부로 기능한다.

직접 접근 방식의 장점은 데이터를 다른 저장소로 다시 옮기지 않아도 된다는 데 있다. 예를 들어 EC2, AWS Batch, EMR, HealthOmics 같은 계산 계층이 같은 클라우드 안에서 public SRA bucket을 바로 읽는다면, 중간 복사본과 불필요한 저장 비용을 줄일 수 있다. NCBI 문서는 S3 URL을 통한 접근이 자유롭고, S3 URL의 경우 inter-region data transfer fee가 없다고 설명한다. 물론 사용자가 자기 결과를 저장하거나 별도 계산 자원을 운영하는 비용은 여전히 발생할 수 있다. 하지만 핵심은 공용 원본에 접근하는 출발점이 훨씬 가벼워졌다는 점이다 (NCBI 2024b).

## SRA Toolkit - format conversion과 표준 접근 경로

그렇다고 해서 direct S3 access가 항상 유일한 길은 아니다. `sra-tools`는 여전히 SRA 생태계에서 매우 중요한 표준 도구이며, 특히 `prefetch`, `fastq-dump`, `fasterq-dump` 같은 유틸리티는 많은 워크플로에서 계속 사용된다. direct S3 access가 "공용 객체에 바로 닿는 경로"라면, SRA Toolkit은 "SRA 포맷을 사용자가 원하는 형식으로 가져오고 변환하는 경로"라고 이해할 수 있다. 따라서 이 장에서는 둘을 경쟁 관계로 설명하기보다, 서로 다른 목적의 도구로 설명하는 편이 더 정확하다. 즉 이미 알고 있는 run accession을 빠르게 cloud-native하게 다루고 싶다면 direct S3 access가 더 자연스러울 수 있고, SRA 고유 포맷을 표준 도구로 처리하거나 특정 변환 단계를 거치고 싶다면 Toolkit이 적합할 수 있다.

실제 교육에서는 이 두 경로를 대비해 보여 주는 것이 좋다. 같은 accession을 대상으로 할 때 `aws s3 cp`는 빠른 직접 복사 경로를 보여 주고, `prefetch`와 `fasterq-dump`는 표준화된 포맷 변환 경로를 보여 준다. 학생은 여기서 "어떤 도구가 더 정답인가"를 외우기보다, 어떤 상황에서 어떤 접근이 더 자연스러운가를 배워야 한다. SRA Toolkit의 존재는 클라우드 시대에도 파일 형식과 변환 단계가 여전히 중요하다는 사실을 상기시킨다. 즉 클라우드는 데이터 이동을 줄여 주지만, 형식과 해석의 문제를 사라지게 하지는 않는다.

## Athena metadata query와 cohort selection

공용 데이터가 커질수록 더 중요한 것은 원본 파일을 만지는 일보다 먼저 메타데이터를 좁히는 일이다. NCBI는 AWS Athena를 통한 cloud-native metadata search를 공식적으로 안내하고 있으며, `Get Started in Athena`와 예제 쿼리 문서는 SRA metadata를 SQL로 검색하는 흐름을 제공한다. 이 구조는 교육적으로 매우 강력하다. 학생은 먼저 organism, assay, platform, project, accession 범위를 Athena로 좁히고, 그 다음 필요한 run만 direct access나 Toolkit으로 넘기는 흐름을 배울 수 있다. 즉 대규모 공개 데이터 시대의 첫 단계는 파일 다운로드가 아니라 metadata query다 (NCBI 2024c).

이 점은 연구 설계에도 직접 연결된다. 예전에는 관심 있는 데이터셋을 대충 골라 먼저 내려받은 뒤, 로컬에서 정리하는 경우가 많았다. 반면 Athena 기반 접근은 어떤 cohort를 만들고 싶은지, 어떤 속성으로 샘플을 필터링할 것인지, 어떤 조건을 만족하는 run만 계산에 태울 것인지부터 명시하게 만든다. 이는 단순한 기술 선택이 아니라 분석 사고방식의 변화다. Table 1처럼 `먼저 메타데이터를 좁히고 -> accession을 정하고 -> 데이터에 접근하고 -> 계산을 시작한다`는 흐름을 체화하면, 학생은 더 큰 공개 데이터 환경에서도 같은 원리로 확장할 수 있다.

Table 1. SRA on AWS의 세 가지 기본 접근 경로

| 경로 | 언제 적합한가 | 대표 예시 |
|---|---|---|
| Public direct access | accession을 이미 알고 있고, public data를 바로 읽고 싶을 때 | `aws s3 ls s3://sra-pub-run-odp/ --no-sign-request` |
| Toolkit-based access | SRA 포맷 변환이나 표준 도구 흐름이 필요할 때 | `prefetch`, `fasterq-dump` |
| Cloud Data Delivery | Toolkit으로 직접 받을 수 없는 original file 또는 특정 restricted data가 필요할 때 | 사용자 S3 bucket으로 직접 전달 |

## Cloud Data Delivery - original file과 restricted data 전달

사용자가 흔히 떠올리는 `request` 개념에 가장 가까운 것은 Cloud Data Delivery Service다. NCBI 공식 문서는 SRA Toolkit이 모든 original submitted file을 직접 제공할 수는 없기 때문에, cloud data delivery를 통해 source file이나 기타 특정 파일을 사용자 bucket으로 전달한다고 설명한다. 여기서 중요한 것은 이것이 일반적인 의미의 "내 컴퓨터로 다운로드"가 아니라는 점이다. 데이터는 사용자의 AWS 또는 GCP bucket으로 직접 전달되며, AWS의 경우 목적 bucket이 `us-east-1` 리전에 있어야 한다는 제약이 있다. 따라서 학생은 public data의 direct access와, 별도의 delivery 요청이 필요한 경우를 구분해 배워야 한다 (NCBI 2024d).

이 distinction은 운영상 매우 중요하다. public data는 공용 bucket을 바로 읽는 방식이 자연스럽고, restricted data나 original file 일부는 사용자의 cloud bucket으로 controlled delivery하는 방식이 더 적합하다. 즉 모든 것을 같은 방식으로 처리하지 않는다는 점이 핵심이다. 실제 연구에서는 공개 데이터와 승인 기반 데이터가 함께 등장할 수 있으므로, 어떤 자산이 direct access 대상이고 어떤 자산이 delivery 대상인지를 구분하는 능력이 필요하다. 이 장에서 SRA는 단지 데이터 접근 기술이 아니라, 데이터 거버넌스와 접근권한이 클라우드 안에서 어떻게 구현되는지를 배우는 출발점이 된다.

## DataSync와 하이브리드 데이터 반입

SRA 같은 공개 데이터 자원은 직접 참조가 중요하지만, 우리 연구실이 직접 생산한 데이터는 여전히 반입해야 하는 경우가 많다. 시퀀싱 센터, 병원, 온프레미스 HPC, 다른 클라우드에 있는 FASTQ, BAM, CRAM을 AWS로 옮겨야 할 때는 AWS DataSync가 매우 실용적인 도구가 된다. DataSync는 NFS, SMB, HDFS, object storage와 AWS 스토리지 사이의 대용량 전송, 무결성 검증, 자동화된 동기화를 지원한다. 따라서 이 장에서는 `공개 데이터는 가능한 한 직접 참조하고, 자체 데이터는 필요에 따라 반입한다`는 대비를 짚어 두는 것이 좋다. 이 둘을 같은 문제로 보지 않을수록 전체 데이터 전략이 더 잘 정리된다.

DataSync를 함께 소개하는 이유는 학생이 클라우드 활용을 공개 데이터 읽기에만 한정해서 생각하지 않게 하기 위해서다. 실제 연구 운영에서는 public open data, controlled data, 자체 생성 데이터가 함께 존재한다. SRA가 보여 주는 것은 공용 데이터의 direct access 모델이고, DataSync가 보여 주는 것은 하이브리드 환경에서 자체 데이터를 반입하는 운영 모델이다. 결국 AWS를 잘 쓴다는 말은 모든 데이터를 일괄적으로 복사하는 것도 아니고, 모든 데이터를 일괄적으로 현지 참조하는 것도 아니다. 데이터의 성격에 따라 `참조할 것`, `반입할 것`, `전달 요청할 것`을 구분하는 것이 핵심이다.

## 비용과 운영 주의점

공용 데이터 접근이 쉬워졌다고 해서 비용을 잊어도 되는 것은 아니다. NCBI 문서는 public SRA data의 자유 접근과 무료 access를 강조하지만, 사용자가 EC2를 운영하거나 Athena query를 돌리거나 결과를 자기 버킷에 저장하는 비용은 여전히 발생할 수 있다. 또한 AWS의 일반적인 S3 비용 모델에는 Requester Pays 같은 개념도 존재하므로, 학생은 "공용 데이터 자유 접근"과 "모든 S3 bucket이 동일한 비용 규칙을 갖는다"를 혼동하지 않도록 배워야 한다. 특히 Athena는 메타데이터 탐색에 매우 강력하지만, 쿼리 결과 저장과 스캔 비용을 고려해야 하므로 무제한 무료 도구처럼 가르치면 안 된다. 비용 감각은 direct access의 장점을 과장하지 않고, 어떤 단계에서 누구의 비용이 발생하는지를 구분하는 태도에서 출발한다.

현대 유전체학에서 공용 데이터 접근은 더 이상 "먼저 받아서 저장"이 아니라, "먼저 찾고, 필요한 것만 정하고, 가능한 한 데이터가 있는 곳에서 계산"하는 방향으로 바뀌고 있다. SRA는 이 전환을 가장 잘 보여 주는 사례다. direct S3 access, Toolkit-based conversion, Cloud Data Delivery, DataSync 기반 반입은 서로 대체 관계가 아니라, 서로 다른 종류의 데이터와 다른 운영 문제를 푸는 경로들이다. 학생이 이 네 가지를 구분할 수 있게 되면, 이후 어떤 공개 유전체 자원을 AWS에서 보더라도 같은 원리로 접근 전략을 설계할 수 있게 된다.

## 핵심 개념 정리

- SRA를 AWS에서 다룬다는 것은 종종 `다운로드`보다 `직접 접근`에 가깝다.
- public data, toolkit conversion, cloud delivery는 서로 다른 목적의 접근 경로다.
- 대규모 공개 데이터 시대에는 원본 파일보다 메타데이터를 먼저 질의하는 습관이 중요하다.
- 공개 데이터 직접 참조와 자체 데이터 반입은 다른 문제이며, DataSync는 후자에 더 가깝다.

## 복습 질문

1. public SRA data를 AWS에서 다룰 때 `download`보다 `direct access`라는 표현이 더 정확한 이유는 무엇인가?
2. direct S3 access와 SRA Toolkit은 각각 어떤 상황에서 더 적합한가?
3. Cloud Data Delivery와 DataSync는 어떤 점에서 서로 다른 문제를 해결하는가?

## Further Reading

- NCBI. [SRA in the Cloud](https://www.ncbi.nlm.nih.gov/sra/docs/sra-cloud/).
- NCBI. [Download SRA sequence data using Amazon Web Services (AWS)](https://www.ncbi.nlm.nih.gov/sra/docs/sra-aws-download/).
- AWS Open Data Registry. [NIH NCBI Sequence Read Archive (SRA) on AWS](https://registry.opendata.aws/ncbi-sra/).

## References

- Amazon Science. 2020. [AWS democratizes access to the largest genomic sequences repository — NIH’s Sequence Read Archive](https://www.amazon.science/latest-news/aws-democratizes-access-to-the-largest-genomic-sequences-repository-nihs-sequence-read-archive).
- AWS Open Data Registry. 2026. [NIH NCBI Sequence Read Archive (SRA) on AWS](https://registry.opendata.aws/ncbi-sra/).
- NCBI. 2024a. [SRA in the Cloud](https://www.ncbi.nlm.nih.gov/sra/docs/sra-cloud/).
- NCBI. 2024b. [Download SRA sequence data using Amazon Web Services (AWS)](https://www.ncbi.nlm.nih.gov/sra/docs/sra-aws-download/).
- NCBI. 2024c. [Get Started in Athena](https://www.ncbi.nlm.nih.gov/sra/docs/sra-athena/).
- NCBI. 2024d. [Cloud Data Delivery Service](https://www.ncbi.nlm.nih.gov/sra/docs/data-delivery/).
- NCBI. 2024e. [Search in Athena](https://www.ncbi.nlm.nih.gov/sra/docs/sra-athena-examples/).
