# 용어집 (Glossary)

이 용어집은 본문에 처음 등장하는 자리에서 간단히 정의하되, 책 전체에서 반복해서 나오는 핵심 용어를 한곳에 모은 것이다. 클라우드 인프라, 생물정보학, 오믹스 데이터 모델, AI·자동화 용어를 분야별로 정리했다. 각 장을 읽다 모호한 용어가 나오면 여기서 다시 확인하면 된다.

---

## 1. AWS 인프라 용어

| 용어 | 정의 |
|---|---|
| Region | AWS가 서비스하는 물리적 지리 영역. 같은 리전 안에 두면 비용과 지연이 작다. |
| Availability Zone (AZ) | 한 리전 안에서 전력·네트워크가 분리된 데이터센터 묶음. |
| EC2 (Elastic Compute Cloud) | 원하는 크기의 가상 서버를 필요할 때 켜는 계산 서비스. |
| S3 (Simple Storage Service) | 파일이 아닌 객체(object)로 데이터를 저장하는 클라우드 저장소. |
| EBS (Elastic Block Store) | EC2 인스턴스에 붙는 블록 스토리지. 운영체제 디스크, 임시 작업 공간에 적합. |
| FSx for Lustre | 고성능 POSIX 공유 파일시스템. S3와 연동해 대규모 scratch 용도로 쓴다. |
| IAM User | 사람 혹은 프로그램적 주체에 발급되는 자격 증명 단위. |
| IAM Role | 실행 주체(인스턴스, 서비스 등)에 일시적 권한을 부여하는 방식. 액세스 키를 코드에 박는 대신 쓴다. |
| VPC | AWS 계정 안의 가상 네트워크. 서브넷, 라우팅, VPC endpoint를 포함한다. |
| VPC Endpoint | 인터넷을 거치지 않고 AWS 서비스(S3 등)에 접근하는 내부 통로. |
| Spot Instance | 여유 자원을 할인가로 쓰는 EC2. 2분 interruption notice 후 회수될 수 있다. |
| Batch | 대규모 잡을 큐에 넣고 EC2/Fargate에서 실행시키는 관리형 서비스. |
| Lambda | 이벤트에 반응해 짧게 돌아가는 서버리스 함수 실행 환경. |
| EMR | Spark, Hadoop 같은 분산 프레임워크를 AWS에서 관리형으로 돌리는 플랫폼. |
| HealthOmics | 유전체·오믹스 전용 storage, workflow, analytics를 묶은 AWS 서비스. |
| SageMaker (SageMaker AI) | 머신러닝 모델 학습과 배포를 위한 통합 서비스. 2024년 12월 `SageMaker AI`로 개명. |
| Bedrock | foundation model(Claude 등)을 AWS 계정 안에서 호출·관리하는 운영 계층. |
| AgentCore | Bedrock 위에서 도구 사용, 질의 orchestration을 담당하는 응용 계층. |
| Athena | S3 위의 데이터를 SQL로 질의하는 서버리스 쿼리 서비스. |
| Glue (Data Catalog) | S3 위 데이터의 스키마와 테이블 메타데이터를 관리하는 카탈로그. |
| DataSync | 온프레미스, 다른 클라우드, 시퀀싱 센터에서 AWS로 데이터를 옮기는 관리형 서비스. |
| ECR | 컨테이너 이미지를 저장·배포하는 AWS 레지스트리. |
| ECS / Fargate | 컨테이너 실행 서비스. Fargate는 인스턴스를 직접 관리하지 않는 형태. |
| EKS | AWS에서 관리형으로 돌리는 Kubernetes. |
| ParallelCluster | Slurm/HPC 스타일 클러스터를 AWS에서 간편하게 띄우는 도구. |
| Step Functions | 여러 서비스의 실행 흐름을 상태 기계(state machine)로 엮는 오케스트레이션. |
| MWAA | AWS가 관리하는 Apache Airflow 환경. |
| CloudWatch | 로그, 메트릭, 이벤트를 수집·조회하는 관측 서비스. |
| S3 Tables | 트랜잭션·스키마가 관리되는 S3 기반 Iceberg 테이블 스토리지. |
| Quick (Amazon Q / QuickSight) | 자연어·시각화 기반 분석 인터페이스 계열. |
| Mountpoint for Amazon S3 | AWS 공식 S3 FUSE 마운트 도구. 읽기 전용과 append 위주. |
| S3 Intelligent-Tiering | 접근 빈도에 따라 S3 스토리지 계층을 자동 조정하는 저장 클래스. |
| S3 Glacier | 장기 보관용 저비용 S3 스토리지 계층. |
| Lifecycle rule | S3 객체의 전이·삭제 규칙. incomplete multipart upload 정리에 중요. |

---

## 2. 유전체·변이 분석 용어

| 용어 | 정의 |
|---|---|
| WGS (Whole-Genome Sequencing) | 한 사람의 전체 유전체를 읽는 시퀀싱. 한 명당 수십~수백 GB 규모. |
| WES (Whole-Exome Sequencing) | 단백질 코딩 영역을 집중해 읽는 시퀀싱. |
| FASTQ | 시퀀싱 원시 리드와 품질값을 담은 텍스트 포맷. |
| BAM / CRAM | 정렬된 리드를 담는 이진 포맷. CRAM은 참조 기반 압축으로 더 작다. |
| VCF | 변이(variant)를 기록하는 표준 텍스트 포맷. |
| BGZF | BAM/CRAM/VCF에 쓰이는 블록 기반 gzip 변형. 랜덤 접근을 가능하게 한다. |
| Variant calling | 정렬 결과에서 변이를 호출하는 단계. |
| Joint genotyping | 여러 샘플의 중간 산물(g.vcf 등)을 합쳐 cohort 수준에서 변이를 재호출하는 단계. |
| VEP (Variant Effect Predictor) | 변이의 기능적 효과를 예측하는 Ensembl 도구. |
| ClinVar | 변이의 임상적 의미를 공유하는 NIH 공개 데이터베이스. |
| Pathogenic variant | 질병과 인과적 연관이 확립된 변이. |
| Allele frequency (AF) / Allele number (AN) | 코호트에서 대립유전자가 관찰된 비율과 전체 수. |
| Multiallelic site | 한 위치에 여러 대립유전자가 있는 경우. |
| Burden test | 유전자 단위로 rare variant의 누적 효과를 검정하는 방법. |
| gnomAD | Broad Institute가 관리하는 대규모 인구 변이 빈도 참조 데이터. |
| SRA (Sequence Read Archive) | NCBI가 관리하는 공개 시퀀싱 원시 데이터 저장소. |
| DRAGEN | Illumina의 FPGA 가속 유전체 분석 파이프라인. |
| Manifest | 분석에 들어가는 입력 파일의 경로·메타데이터 목록 파일. |

---

## 3. 분산 분석·데이터 모델 용어

| 용어 | 정의 |
|---|---|
| Spark | 분산 데이터 처리 프레임워크. EMR의 주된 실행 엔진. |
| Parquet | 칼럼 기반 압축 저장 포맷. 대규모 질의에 유리. |
| Iceberg | S3 등 객체 스토리지 위에서 트랜잭션과 스키마 진화를 지원하는 테이블 포맷. |
| EMRFS | EMR이 S3를 HDFS처럼 쓰게 해 주는 파일시스템 계층. |
| Hail | Spark 위에서 유전체 데이터를 다루는 분산 분석 라이브러리. |
| MatrixTable | Hail의 행=변이, 열=샘플 행렬 자료구조. |
| VariantDataset (VDS) | reference block과 variant data를 분리한 sparse, split 표현. cohort-scale 분석 표현. |
| Local alleles (LGT, LAD, LPL, LA) | multiallelic site에서 전역 인덱스 대신 지역 인덱스를 쓰는 VDS 표현. |
| Nextflow / WDL / CWL | 재현 가능한 워크플로 정의 언어. HealthOmics가 지원. |

---

## 4. single-cell·spatial·이미징 오믹스 용어

| 용어 | 정의 |
|---|---|
| scRNA-seq | 세포 단위 RNA 발현 측정. 한 실험에서 수만~수백만 세포가 나온다. |
| Spatial transcriptomics | 조직 절편의 위치 정보와 함께 RNA 발현을 측정하는 기술. |
| AnnData | 세포×유전자 행렬과 `obs`, `var`, `obsm`, `layers`를 함께 다루는 데이터 모델. |
| h5ad | AnnData의 HDF5 기반 저장 backend. 단일 파일. |
| Zarr | N차원 배열을 chunk 단위로 저장해 객체 스토리지에서 부분 접근을 가능하게 하는 포맷. |
| Chunk | Zarr 배열의 최소 접근 단위. 크기 설계가 성능과 비용을 좌우한다. |
| Sharding (zarr v3) | 작은 chunk를 묶어 객체 수를 줄이는 기법. |
| OME-Zarr | 생물학 이미지와 다중 해상도 피라미드를 위한 Zarr 기반 표준. |
| SpatialData | Images, Labels, Points, Shapes, Tables를 한 좌표계에서 다루는 프레임워크. |
| TileDB-SOMA | larger-than-memory 질의와 corpus-wide API를 제공하는 플랫폼형 single-cell 저장 계층. |
| Vitessce | 웹 기반 single-cell·spatial 시각화 컴포넌트 모음. |
| CELLxGENE Census | CZI가 제공하는 corpus-wide single-cell 질의 플랫폼. TileDB-SOMA 기반. |
| gimVI | 짝지어지지 않은 scRNA-seq와 spatial 데이터를 통합해 미측정 유전자를 추정하는 모델. |

---

## 5. AI·자동화 용어

| 용어 | 정의 |
|---|---|
| Foundation model | 범용 대규모 언어·멀티모달 모델. Claude, Llama 등이 포함된다. |
| Claude | Anthropic의 foundation model 계열. Bedrock을 통해 AWS에서 호출 가능. |
| Kiro | 요구사항·설계·작업 파일을 명시적으로 남기는 spec-driven AI IDE. |
| RAG (Retrieval-Augmented Generation) | 외부 데이터에서 관련 문서를 먼저 검색해 모델 답변을 근거 있게 만드는 방식. |
| Human-in-the-loop | AI 출력에 사람의 검토·승인을 결합해 책임 체계를 유지하는 설계. |
| Provenance | 데이터가 어디서 왔고 어떤 처리를 거쳤는지 추적 가능한 기록. |
| Reproducibility | 같은 입력·코드·환경에서 같은 결과가 나오도록 보장하는 성질. |

---

## 6. 운영·비용 용어

| 용어 | 정의 |
|---|---|
| Bring compute to data | 데이터를 옮기지 않고 데이터가 있는 곳에서 계산을 돌리는 원칙. |
| Egress | 리전·AZ를 벗어나는 데이터 전송. 상당한 비용 요인. |
| Cross-AZ / cross-region | 가용 영역·리전 사이의 데이터 이동. 구성에 따라 과금된다. |
| Array job | Batch가 같은 정의를 N개 복제해 병렬 실행하는 잡. |
| Shard | 큰 작업을 재실행 단위로 나누기 쉽게 쪼갠 묶음. |
| Retry / timeout | 일시적 실패를 자동 복구하거나 무한 대기를 방지하는 설정. |
| Incomplete multipart upload | 중단된 업로드가 남긴 "유령 바이트". 7일 lifecycle rule로 정리한다. |
| Right-sizing | 병목에 맞는 인스턴스 패밀리·크기를 고르는 습관. |
| Presigned URL | 제한된 시간 동안 유효한 서명 URL. 외부 공유에 쓴다. |
| Requester pays | 접근 비용을 요청자(다운로드자)가 부담하는 S3 설정. |

---

## 찾는 용어가 없다면

본문의 첫 등장 시 정의를 찾아보거나, 각 장 말미의 `핵심 개념 정리`와 `Further Reading`을 참고한다. AWS 공식 용어는 해당 서비스 문서 링크가 References에 정리되어 있다.
