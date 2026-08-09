# VIVAC — 비즈니스/제품 문서

> 프로덕트 오너, PM, 운영팀을 위한 문서 모음. 2026-08-01 기준 코드/기획 자료를 조사해 작성했다.
> 개발자 관점 문서는 [`../devel/`](../devel/README.md) 참고.

## 읽는 순서

1. [00-product-overview.md](./00-product-overview.md) — 제품이 무엇을 해결하는가, 차별화, 타겟, MVP 범위
2. [01-domain-glossary.md](./01-domain-glossary.md) — pipeline_status, trust_tier, 권한 등급 등 핵심 비즈니스 용어
3. [02-core-features.md](./02-core-features.md) — 탐색/그룹/초대/리뷰 등 핵심 기능별 유저 플로우
4. [03-admin-console-operations.md](./03-admin-console-operations.md) — 내부 운영 콘솔, 권한 등급별 업무, 모더레이션
5. [04-data-pipeline-and-quality.md](./04-data-pipeline-and-quality.md) — 데이터 파이프라인 단계와 신뢰도 정책
6. [05-monetization-and-roadmap.md](./05-monetization-and-roadmap.md) — 수익화 후보, 성장/리텐션/신뢰 기능 로드맵
7. [06-known-risks-and-open-decisions.md](./06-known-risks-and-open-decisions.md) — 의사결정이 필요한 리스크/공백

## 분야별 빠른 링크

| 궁금한 것 | 문서 |
|---|---|
| VIVAC이 어떤 문제를 푸는 서비스인가 | [00-product-overview.md](./00-product-overview.md) |
| "미검증" 뱃지는 왜 붙는가 | [01-domain-glossary.md](./01-domain-glossary.md), [04-data-pipeline-and-quality.md](./04-data-pipeline-and-quality.md) |
| 그룹/초대 기능이 정확히 뭘 할 수 있나 | [02-core-features.md](./02-core-features.md) §4-5 |
| 운영팀이 콘솔에서 뭘 하나 | [03-admin-console-operations.md](./03-admin-console-operations.md) |
| 다음에 어떤 기능을 만들면 좋을까 | [05-monetization-and-roadmap.md](./05-monetization-and-roadmap.md) |
| 지금 당장 결정해야 하는 게 뭐가 있나 | [06-known-risks-and-open-decisions.md](./06-known-risks-and-open-decisions.md) |

## 문서 성격

- 이 폴더는 **현재 상태의 스냅샷 + 로드맵**이다. 상태 표기(`제안`/`🚧 진행중`/`✅ 완료`/`⏸ 보류`)가 바뀌면 해당 문서를 갱신한다.
- 기능이 실제로 설계 단계로 들어가면 상위 폴더에 개별 설계 문서(`../<feature-slug>.md`)를 새로 만드는 것이 이 저장소의 관례다 — 이 폴더의 로드맵 항목은 그 설계 문서로 이어지는 입구 역할을 한다.
- Notion/Linear로 옮길 때: 각 파일을 프로젝트/페이지 1개로, 표의 각 행(특히 [05-monetization-and-roadmap.md](./05-monetization-and-roadmap.md)의 기능 목록)을 이슈/태스크 1개로 매핑하는 것을 권장한다.
