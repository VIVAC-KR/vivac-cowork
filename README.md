# vivac-cowork

VIVAC 서비스의 **기획/협업 문서 저장소**입니다. 코드는 없고, 제품 기획·정책·설계 문서를 관리합니다. API 서버 코드는 별도 저장소 `vivacapi-core`에 있습니다.

## 문서 구성

> 📍 문서가 많아 길을 잃기 쉽습니다 — **[docs/INDEX.md](docs/INDEX.md)**(전체 문서 마스터 인덱스)와 **[docs/STATUS.md](docs/STATUS.md)**(미해결 이슈·진행 상황·핵심 결정 로그 종합)를 먼저 보세요.

| 파일 | 내용 |
|---|---|
| [docs/INDEX.md](docs/INDEX.md) | 전체 문서(제품 공유 문서 + 5개 repo 전용 폴더 + design/) 마스터 인덱스. "이 내용 어디 있지?"의 첫 진입점입니다. |
| [docs/STATUS.md](docs/STATUS.md) | 코드 리뷰 오픈 이슈, 로드맵 진행 상황, 팀 결정 대기 항목, 핵심 결정 로그를 한 곳에 모은 트래커. |
| [docs/product/PRODUCT.md](docs/product/PRODUCT.md) | 제품 정의 — 문제·가설, MVP 범위와 단계, 데이터 정의, 결정 로그, 열린 항목. 현재 확정본입니다 (2026-08-10 개정). 기능 명세는 `docs/features/`로 분리했습니다. |
| [docs/features/](docs/features/README.md) | 화면별 기능 명세 — 홈 · 검색/필터/지도 탐색 · 스팟 상세 · 계정/인증. 각 화면을 10개 절로 자족적으로 기술해, 구현자가 다른 문서를 열지 않고 한 화면을 만들 수 있게 합니다. 라우트 없는 공용 로직은 `docs/features/_shared/`에 둡니다 (2026-08-10 PRODUCT.md §5에서 분리). |
| [docs/product/ia.md](docs/product/ia.md) | 정보 구조(IA) — 사이트맵, 화면 인벤토리, 내비게이션 구조. |
| [docs/architecture/data-pipeline.md](docs/architecture/data-pipeline.md) | 스팟 데이터 파이프라인 설계 — `pipeline_status`(RAW→PUBLISHED 처리 단계), `trust_tier`(신뢰도 1~3등급) 필드 정의와 확정 정책. |
| [docs/product/business-feature-roadmap.md](docs/product/business-feature-roadmap.md) | 비즈니스 기능 로드맵 — 성장/리텐션/수익화/신뢰 4개 관점의 기능 후보와 우선순위. |
| [docs/architecture/](docs/architecture/) | 여러 저장소에 걸친 시스템 구조 — 데이터 파이프라인 설계, OpenAPI 계약(`openapi.json`). 한 저장소 안에서 닫히는 구조 문서(`docs/core/architecture.md` 등)는 저장소별 폴더에 남습니다. |
| [docs/research/](docs/research/) | 사용자·시장 리서치 — 제품 판단의 근거가 되는 조사 자료. |
| [docs/meta/](docs/meta/) | 문서 거버넌스 — Constitution, 정본 목록(SSOT), 문서 작성 규칙. |
| [docs/archive/planning-source/](docs/archive/planning-source/) | PRODUCT.md 병합에 쓰인 원본 기획·리서치 문서 모음 (2026-08-04, 프로젝트 루트에 흩어져 있던 `docs_to_be_merge/`와 시장조사 자료를 통합). 폐기된 문서와 아직 미해결·미실행인 문서가 섞여 있으므로 각 파일 상단 상태 표시를 먼저 확인하세요. 리서치 원본은 [docs/research/](docs/research/)로 옮겼습니다. |
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

`docs/front/`도 같은 규칙을 따릅니다. 과거에는 `VIVAC-frontend` repo의 `docs/`를 복사해 미러링했으나, 심볼릭 링크로 전환한 뒤로는 이 폴더가 원본입니다 — `VIVAC-frontend`에는 자체 `docs/`가 없고 이 폴더를 그대로 가리킵니다. 여기서 고친 내용이 곧 그 저장소의 문서입니다.

`vivac-ios`, `vivac-mobile-test`는 자체 `docs/` 폴더가 없어 취합 대상에서 제외했습니다. `vivacapi-core-org`는 `vivacapi-core`의 오래된 개인 포크로, 내용이 전부 `vivacapi-core` 쪽이 최신·상위호환이라 제외했습니다(대조 확인 완료, 2026-07-21).
