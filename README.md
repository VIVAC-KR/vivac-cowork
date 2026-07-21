# vivac-cowork

VIVAC 서비스의 **기획/협업 문서 저장소**입니다. 코드는 없고, 제품 기획·정책·설계 문서를 관리합니다. API 서버 코드는 별도 저장소 `vivacapi-core`에 있습니다.

## 문서 구성

| 파일 | 내용 |
|---|---|
| [docs/PRODUCT.md](docs/PRODUCT.md) | 제품 개요 — 문제 정의, 차별화, 타겟, MVP 범위, 데이터 전략, 수익 모델, 로드맵. 현재 확정본입니다. |
| [docs/PRODUCT_TEMP.md](docs/PRODUCT_TEMP.md) | 기획 개정 초안 (2026-07-11 PM 분석 반영) — beachhead 타겟(차박러), 신뢰도 투명성 중심 재포지셔닝, MVP 범위 조정, 가설 검증 계획/KPI. 확정 시 PRODUCT.md로 병합 예정입니다. |
| [docs/data-pipeline.md](docs/data-pipeline.md) | 스팟 데이터 파이프라인 설계 — `pipeline_status`(RAW→PUBLISHED 처리 단계), `trust_tier`(신뢰도 1~3등급) 필드 정의와 확정 정책. |
| [docs/business-feature-roadmap.md](docs/business-feature-roadmap.md) | 비즈니스 기능 로드맵 — 성장/리텐션/수익화/신뢰 4개 관점의 기능 후보와 우선순위. |
| [.claude/rules/vivac-docs-authoring.md](.claude/rules/vivac-docs-authoring.md) | `docs/` 하위 문서 작성 규칙(카테고리, 파일명, 톤, 템플릿). `docs/`가 아니라 `.claude/rules/`에 있어, 각 repo에서 이 경로 그대로 한 번 더 심볼릭 링크를 걸면 문서 작성 시 자동 로드됩니다. |
| [docs/CONTEXT_SCOPE.md](docs/CONTEXT_SCOPE.md) | 공유 컨텍스트 기본 참고 범위 안내. 각 repo CLAUDE.md에서 `@docs/CONTEXT_SCOPE.md`로 import합니다. |
| [SYMLINK-SETUP.md](SYMLINK-SETUP.md) | 각 repo에서 이 저장소의 `docs/`를 심볼릭 링크로 연결하는 방법. |
| [CLAUDE.md](CLAUDE.md) | Claude Code용 프로젝트 지침. `vivac-cowork` 저장소 자체 작업용이며, `docs/`가 아니라 저장소 루트에 있어 다른 repo로는 공유되지 않습니다. |

## repo별 문서 (`docs/<repo>/`)

각 repo에서만 참고하는 맥락(아키텍처, 백로그, 코드리뷰, 결정사항, ETL 작업 기록 등)은 원본 repo가 아니라 여기서 관리합니다. 각 repo는 이 저장소의 `docs/` 폴더를 자기 `docs/` 자리에 심볼릭 링크로 걸어 공유 문서(PRODUCT.md 등)와 자기 전용 폴더(`<repo>/`)를 함께 참조합니다. 저장소 루트(CLAUDE.md 등)는 링크 대상에 포함되지 않습니다 — 링크 설정 방법은 [SYMLINK-SETUP.md](SYMLINK-SETUP.md) 참고.

| 폴더 | 원본 repo |
|---|---|
| [docs/front/](docs/front/) | `VIVAC-frontend` |
| [docs/console/](docs/console/) | `vivac-console` |
| [docs/mcp/](docs/mcp/) | `vivac-mcp` |
| [docs/core/](docs/core/) | `vivacapi-core` |
| [docs/etl/](docs/etl/) | `vivacapi-etl` |

### 폴더 내부 규칙

문서 성격별로 표준 하위 폴더(`decisions/`, `backlog/`, `reviews/`, `projects/`)를 씁니다. 카테고리 기준, 파일명, 톤, 문서별 템플릿은 [.claude/rules/vivac-docs-authoring.md](.claude/rules/vivac-docs-authoring.md)에 정리돼 있습니다 — 각 repo의 `.claude/rules/`에 같은 경로로 심볼릭 링크를 걸어두면 문서를 쓸 때 자동으로 로드됩니다(설정 방법은 [SYMLINK-SETUP.md](SYMLINK-SETUP.md) 참고).

`vivac-ios`, `vivac-mobile-test`는 자체 `docs/` 폴더가 없어 취합 대상에서 제외했습니다. `vivacapi-core-org`는 `vivacapi-core`의 오래된 개인 포크로, 내용이 전부 `vivacapi-core` 쪽이 최신·상위호환이라 제외했습니다(대조 확인 완료, 2026-07-21).
