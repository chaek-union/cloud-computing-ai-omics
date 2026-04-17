# 14장. AWS에서 Claude와 Kiro로 하는 바이브 코딩

## 학습 목표

- Kiro와 Claude on Bedrock이 각각 어떤 역할을 하는지 설명할 수 있다.
- 유전체 분석 코드 작성에서 agentic coding 도구가 어디까지 유용한지 구분할 수 있다.
- 바이브 코딩을 실험적 생산성 도구로 쓰되, 과학적 검증은 왜 별도로 필요하게 되는지 이해할 수 있다.

## 핵심 질문

- Kiro는 일반 IDE나 단순 코드 자동완성과 무엇이 다른가
- Claude를 AWS Bedrock 위에서 쓰는 것은 어떤 운영상 의미를 가지는가
- AI가 파이프라인 초안을 잘 만든다고 해서 왜 결과 검증까지 맡길 수는 없는가

## 바이브 코딩을 어떻게 가르칠 것인가

`바이브 코딩`이라는 표현은 편리하지만, 교과서에서는 조금 더 조심해서 써야 한다. 학생에게 중요한 것은 AI가 "대충 감으로 코드를 써 준다"는 인상이 아니라, 요구사항을 더 빨리 문서화하고 반복 작업을 자동화하고 디버깅을 보조하는 도구로 쓰인다는 사실이다. omics 분석에서는 작은 스크립트 하나가 곧바로 논문 결과나 임상 해석으로 이어질 수 있기 때문에, 코드 생성 속도와 결과의 신뢰도를 반드시 분리해서 생각해야 한다. 따라서 이 장의 목표는 AI 도구를 찬양하거나 두려워하게 만드는 것이 아니라, 어디까지 자동화하고 어디서부터 사람이 다시 검토해야 하는지 경계를 그리게 하는 것이다. 그 경계가 분명할수록 AI 보조 도구는 연구 생산성을 높이는 실용 도구가 된다.

2026년 기준으로 AWS 문맥에서 이 문제를 설명하기에 좋은 조합은 Kiro와 Claude on Bedrock이다. Kiro는 agentic coding IDE로서 spec-driven workflow를 강조하고, Claude on Bedrock은 AWS 계정과 IAM, 리전, 모델 접근 정책 안에서 생성형 모델을 호출하게 해 준다. 즉 하나는 개발 환경에 가깝고, 다른 하나는 모델 접근 계층에 가깝다. 이 둘을 같은 것으로 가르치면 학생이 개념을 혼동하기 쉽다. Kiro는 코딩 작업을 조직하는 도구이고, Bedrock은 모델을 안전하게 호출하는 운영 계층이라는 구분이 먼저다.

## Kiro - 감이 아니라 specification을 남기는 IDE

Kiro의 가장 큰 교육적 장점은 "생각나는 대로 코드를 쓴다"는 흐름보다, `requirements.md`, `design.md`, `tasks.md`처럼 중간 산물을 남기며 코딩하게 만든다는 점이다. 이 방식은 omics 연구와 특히 잘 맞는다. 유전체 분석 파이프라인은 입력 형식, 참조 데이터 버전, 출력 구조, 품질 기준이 조금만 어긋나도 결과 해석이 크게 달라질 수 있기 때문이다. 따라서 학생이 AI 도구를 쓸 때도 먼저 문제를 구조화하고, 입출력 조건과 예외를 적고, 작업 단계를 나누는 습관을 들이는 것이 중요하다. Kiro는 바로 이런 점에서 단순 자동완성 도구보다 더 교육적이다.

예를 들어 어떤 학생이 "VCF를 읽어 rare damaging variant를 필터링하고 샘플별 요약표를 만든다"는 작은 과제를 한다고 하자. 전통적인 바이브 코딩은 곧바로 코드를 생성하게 만들 수 있다. 반면 Kiro 스타일은 먼저 입력이 단일 VCF인지 cohort VCF인지, annotation이 이미 붙어 있는지, 출력이 TSV인지 Parquet인지, 결측값을 어떻게 처리할지, 재현 가능한 실행 방법은 무엇인지를 specification으로 적게 만든다. 이 과정은 느려 보이지만, 실제로는 나중에 생기는 대부분의 오류를 줄여 준다. omics 분석에서는 코드 작성 속도보다 `무엇을 하기로 했는지 명시적으로 남기는 일`이 더 중요할 때가 많다.

## Claude on Bedrock - 모델을 AWS 운영 체계 안으로 가져오기

Claude를 AWS Bedrock 위에서 쓰는 이유는 모델 품질만이 아니다. Bedrock은 IAM 권한, 리전, 모델 접근 승인, 로깅과 같은 운영 요소를 AWS 계정 안으로 가져온다. 연구실 관점에서는 이것이 매우 중요하다. 같은 모델을 쓰더라도 누가 어떤 리전에서 어떤 권한으로 호출하는지, 어떤 프로젝트가 어떤 비용을 발생시키는지, 어떤 데이터와 함께 연결되는지를 더 명시적으로 관리할 수 있기 때문이다. Claude on Bedrock이나 Claude Code on Bedrock 문서는 이런 운영 준비가 model access, IAM permissions, region selection, model pinning 같은 문제와 연결된다고 설명한다.

교과서에서는 이를 "Claude를 어디서 쓰느냐"의 문제가 아니라, "생성형 AI를 연구 인프라 안에 어떻게 배치하느냐"의 문제로 설명하는 편이 좋다. 예를 들어 로컬 개발 환경에서는 Kiro 같은 IDE를 쓰고, 실제 모델 호출은 Bedrock을 통해 수행하게 하면, 코드 작성과 모델 운영의 경계를 더 분명히 나눌 수 있다. 이런 구조는 연구실이나 기관 단위 협업에서 특히 유용하다. 즉 Bedrock은 코딩 경험 그 자체보다, 코딩 도구가 조직의 보안과 비용 체계 안에서 움직이게 하는 계층으로 이해해야 한다.

## 유전체 분석에서 AI 코딩 도구가 실제로 잘하는 일

omics 분석에서 AI 코딩 도구가 가장 잘하는 일은 의외로 "새로운 알고리즘 발명"이 아니다. 오히려 파일 파서 초안 만들기, manifest 생성, CLI 스크립트 뼈대 작성, 문서화, 반복적 리팩터링, 작은 시각화 코드 작성, 테스트 케이스 제안, 에러 메시지 해석 같은 주변부 작업에서 강하다. 이 영역은 사람이 하자니 시간이 아깝고, 그렇다고 완전히 방치하면 코드 품질이 빨리 떨어지는 부분이다. AI 도구는 이런 작업을 빠르게 도와 주면서, 연구자가 더 중요한 분석 설계와 해석에 시간을 쓰게 만든다. 따라서 학생에게는 "AI가 어떤 코드를 가장 잘 보조하는가"를 구체적으로 가르치는 편이 훨씬 실용적이다.

실제 예시를 들면, AI는 Nextflow 입력 manifest를 샘플 시트에서 생성하는 작은 Python 스크립트, VCF annotation 결과를 cohort summary table로 바꾸는 SQL 초안, Hail QC 결과를 시각화하는 notebook 셀, README와 실행 예시를 자동으로 정리하는 작업에 매우 유용하다. 반면 통계 모델의 타당성 판단, rare variant 기준 정의, phenotype encoding, confounder 선택, clinical interpretation 같은 부분은 여전히 사람이 책임져야 한다. 즉 AI는 코딩의 많은 부분을 빠르게 하지만, 과학적 질문 자체를 대신 정하지는 못한다. 이 구분이 흐려지는 순간 바이브 코딩은 생산성 도구가 아니라 오류 증폭기가 될 수 있다.

## 바이브 코딩의 위험 - 환각, 잘못된 라이브러리, 검증 누락

AI 코딩 도구의 가장 큰 위험은 코드가 그럴듯해 보인다는 점이다. 특히 omics 분야는 포맷과 라이브러리의 차이가 복잡해서, 존재하지 않는 함수나 오래된 API, 잘못된 파일 형식 가정이 섞여 들어가기 쉽다. 더 큰 문제는 이런 오류가 종종 즉시 드러나지 않는다는 것이다. 파이프라인이 끝까지 돌아가더라도, 잘못된 phenotype merge나 변이 필터링 기준 때문에 결과가 조용히 왜곡될 수 있다. 따라서 이 장에서는 "AI가 쓴 코드는 우선 의심하고, 사람이 검증한 뒤에만 연구 코드로 승격한다"는 원칙을 명시해야 한다.

여기서 좋은 습관은 네 가지다. 첫째, AI가 만든 코드는 반드시 작은 샘플 데이터로 먼저 테스트한다. 둘째, 실행 결과뿐 아니라 중간 산물의 개수, 헤더, 샘플 수, 변이 수 같은 sanity check를 확인한다. 셋째, reference와 annotation 버전을 명시적으로 고정한다. 넷째, README와 test를 함께 생성하게 만들어 나중에 사람이 검토하기 쉽게 만든다. AI 코딩 도구의 진짜 가치는 코드 한 번에 있지 않고, 검증 가능한 초안을 빨리 만들게 해 주는 데 있다. 학생이 이 원칙을 익히면 바이브 코딩은 훨씬 안전한 생산성 도구가 된다.

## 작은 오믹스 프로젝트에서의 추천 워크플로

교육용으로는 너무 큰 프로젝트보다 작은 end-to-end 예제를 반복하는 편이 좋다. 예를 들어 `샘플 시트 -> FASTQ QC 스크립트 -> 요약표 -> README`나 `VCF -> filtering script -> variant summary -> plots` 같은 흐름은 AI 코딩 도구를 가르치기에 적합하다. 먼저 요구사항을 글로 적고, Kiro 같은 도구로 태스크를 쪼개고, Claude로 코드 초안을 만든 뒤, 작은 입력으로 실행하고, 결과를 수동 검토하고, 마지막에 test와 문서를 붙이는 순서를 반복하면 된다. 이 구조는 AI 도구를 쓰면서도 과학적 통제력을 잃지 않게 해 준다.

AI 코딩 도구는 omics 분석을 대신하는 과학자가 아니라, 초안 작성과 반복 작업을 보조하는 연구 보조원에 가깝다. Kiro는 생각을 구조화하게 해 주고, Claude on Bedrock은 모델 호출을 AWS 운영 체계 안에 두게 해 준다. 둘을 잘 조합하면 연구 생산성은 분명히 높아진다. 그러나 최종 분석 기준, 데이터 품질 판단, 결과 해석, 임상적 의미 부여는 여전히 사람이 책임져야 한다. 바이브 코딩의 핵심은 속도가 아니라, 빠른 초안을 검증 가능한 연구 자산으로 바꾸는 규율이다.

## 핵심 개념 정리

- Kiro는 spec-driven coding workflow를 강조하는 IDE이고, Bedrock은 모델 운영 계층이다.
- AI 코딩 도구는 파일 파서, 문서화, 작은 자동화 코드, 테스트 초안 같은 반복 작업에 특히 강하다.
- omics 분석에서는 그럴듯한 코드보다 검증 가능한 코드가 더 중요하다.
- 바이브 코딩은 결과를 자동 신뢰하는 방식이 아니라, 초안을 빨리 만들고 사람이 검토하는 방식으로 써야 안전하다.

## 복습 질문

1. Kiro와 Claude on Bedrock은 각각 어떤 문제를 해결하며, 왜 같은 도구로 보면 안 되는가?
2. omics 분석에서 AI 코딩 도구가 잘하는 일과 사람이 끝까지 책임져야 하는 일은 각각 무엇인가?
3. AI가 생성한 코드를 연구 코드로 승격하기 전에 어떤 검증 절차가 필요한가?

## Further Reading

- Kiro. [Official site](https://kiro.dev/).
- Anthropic. [Claude on Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-on-amazon-bedrock).
- Anthropic. [Claude Code on Amazon Bedrock](https://code.claude.com/docs/en/amazon-bedrock).

## References

- Anthropic. 2026a. [Claude on Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-on-amazon-bedrock).
- Anthropic. 2026b. [Claude Code on Amazon Bedrock](https://code.claude.com/docs/en/amazon-bedrock).
- AWS. 2026. [Model parameters for Anthropic Claude models](https://docs.aws.amazon.com/bedrock/latest/userguide/model-parameters-claude.html).
- Kiro. 2026a. [Official site](https://kiro.dev/).
- Kiro. 2026b. [Pricing](https://kiro.dev/pricing/).
- Lee et al. 2026. [Vibe Coding Omics Data Analysis Applications](https://pubs.acs.org/doi/10.1021/acs.jproteome.5c00984).
- NCBI. 2026. [GeneGPT repository](https://github.com/ncbi/GeneGPT).
