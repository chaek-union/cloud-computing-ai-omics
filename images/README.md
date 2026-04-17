# images/

이 디렉터리는 책에 쓰이는 개념도, 경로도, 실험 설계 도식을 모아 둔다.

## 파일명 규칙

- `ch{장번호}_fig{번호}_{짧은설명}.{확장자}` 형식을 따른다. 예) `ch8_fig1_vds_vs_vcf.svg`, `ch13_fig2_zarr_chunk.png`.
- 본문에서 참조할 때는 `Figure 8-1`, `Figure 13-2`처럼 장번호와 그림번호를 함께 표기한다.
- 벡터(SVG, PDF)가 가능하면 우선 사용하고, PNG는 300dpi 이상.

## 우선순위 다이어그램 목록

다음 다이어그램이 있으면 학습 효과가 크다. 본문에는 이미 해당 개념의 텍스트 박스 또는 ASCII 구조가 들어가 있으나, 정식 그림으로 교체할 수 있다.

1. `ch1_fig1_cloud_layers.svg` — AWS 서비스 지형도 (저장·계산·워크플로·질의·AI 5계층).
2. `ch2_fig1_storage_map.svg` — S3, EBS, FSx의 접근 패턴 비교.
3. `ch8_fig1_vds_vs_vcf.svg` — VCF의 dense 표현 대 VDS의 sparse split 표현.
4. `ch9_fig1_healthomics_layers.svg` — Storage / Workflows / Analytics 3계층.
5. `ch10_fig1_bgzf_index.svg` — BGZF 블록과 인덱스를 이용한 랜덤 접근.
6. `ch13_fig1_chunk_access.svg` — `download → open` vs `read chunks in place`.
7. `ch13_fig2_data_model_stack.svg` — AnnData / h5ad·Zarr / OME-Zarr / TileDB-SOMA 계층.
8. `ch15_fig1_pipeline_overview.svg` — read set → workflow → table → query → interpretation.
9. `ch18_fig1_bedrock_stack.svg` — Raw data → annotation → table → query → natural language 계층.
10. `ch20_fig1_cost_leaks.svg` — 비용 누수 6축 다이어그램.

## 그림 제작 도구

이 책의 공식 다이어그램은 [D2](https://d2lang.com)로 작성한다. 각 그림은 `.d2` 소스와 그로부터 생성한 `.svg`를 나란히 보관한다.

```bash
d2 images/chX_figY_name.d2 images/chX_figY_name.svg
```

### AWS 서비스 카테고리 팔레트

| 카테고리 | 배경 | 테두리 | 라벨 |
|---------|-----|-------|------|
| Storage | `#F3F8EC` | `#7AA116` | `#4F6B0C` |
| Compute / Workflows | `#FFF3E0` | `#FF9900` | `#B36B00` |
| Analytics | `#EFE4FF` | `#8C4FFF` | `#5B2A9E` |
| Gen-AI (Bedrock) | `#FCE4EC` | `#CC2264` | `#8B1543` |
| Network / Anchor | `#E7F1FF` | `#146EB4` | `#0F4C81` |
| Warning / Bad path | `#FDF3E7` | `#D13212` | `#8B1C00` |
| 본문 텍스트 | — | — | `#232F3E` (Squid Ink) |

## 라이선스

저자 직접 제작 그림은 책 본문과 동일한 라이선스를 따른다. 외부 그림은 출처를 캡션에 명시한다.
