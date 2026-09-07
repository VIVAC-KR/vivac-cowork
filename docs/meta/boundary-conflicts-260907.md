# 저장소 경계 원칙과 충돌하는 문서 — 변경 필요 목록

> 2026-09-07 작성. 근거 원칙은 [SSOT.md](SSOT.md) §1.
> **이 문서는 보고용입니다. 아직 어떤 문서도 이동·수정·삭제하지 않았습니다.**
> 조사 기준: `main` 머지 후 `docs/` 하위 Markdown 107개.

## 1. 요약

| 등급 | 건수 | 성격 |
|---|---|---|
| 🔴 경계 위반 | 4건 | 제품 계층 정보가 저장소 계층에 정의돼 있습니다 |
| 🟠 규칙 충돌 | 3건 | 저장소별로 다른 판정 기준을 선언합니다 |
| 🟡 잔재 | 4건 | 대체됐으나 파일이 남아 있습니다 |

---

## 2. 🔴 경계 위반 — 제품 정의가 저장소 계층에 있음

### B1. `docs/core/projects/business/` (8개 문서)

`vivacapi-core` 폴더 안에 제품 전체를 서술하는 문서 묶음이 있습니다.

| 문서 | 중복 대상 |
|---|---|
| `00-product-overview.md` | 이미 폐기 스텁이며 `PRODUCT.md`를 가리킵니다 — 조치 불요 |
| `01-domain-glossary.md` | `PRODUCT.md` §4, `data-pipeline.md`, `core/enums.md` |
| `02-core-features.md` | `docs/features/`, `PRODUCT.md` §5 |
| `03-admin-console-operations.md` | 고유 — 운영 업무는 `docs/console/`이 적절합니다 |
| `04-data-pipeline-and-quality.md` | `data-pipeline.md`, `PRODUCT.md` §4.3 |
| `05-monetization-and-roadmap.md` | `PRODUCT.md` §7.5, `business-feature-roadmap.md` |
| `06-known-risks-and-open-decisions.md` | `PRODUCT.md` §7, `STATUS.md` §4 |

**문제**: 제품 정의·수익화·로드맵·열린 결정은 제품 계층의 정보인데 저장소 계층에 있습니다. 게다가 `PRODUCT.md`는 2026-08-10에 개정됐고 이 묶음은 2026-08-01 기준이라 **이미 낡았습니다** — 예를 들어 MVP 우선순위가 개정 전 기준입니다.

**제안**: `01`·`02`·`04`·`05`·`06`은 제품 계층 정본으로 내용을 흡수하거나, 스냅샷임을 명시하고 `archive/`로 옮깁니다. `03`은 `docs/console/`로 이동합니다. 흡수할 내용이 있는지는 문서별 대조가 필요합니다.

### B2. `docs/core/projects/devel/` (10개 문서)

저장소 계층에 있는 것 자체는 맞습니다. 다만 기존 정본과 같은 내용을 다시 씁니다.

| 문서 | 기존 정본 |
|---|---|
| `01-data-model.md` | `core/erd.md` + `core/enums.md` |
| `02-api-reference.md` | `docs/openapi.json` + `core/architecture.md` |
| `06-search.md` | `core/projects/spot-search-postgres-fts.md` |
| `08-infra-and-deployment.md` | `core/infra/lightsail-setup.md` |
| `09-known-issues-and-tech-debt.md` | `core/backlog.md`, `core/backlog/*`, `core/code-review-2026-07-22.md` |

**제안**: `devel/`을 "신규 합류자용 안내"로 성격을 좁히고 상세는 정본 링크로 대체하거나, 반대로 `devel/`을 정본으로 승격하고 기존 문서를 정리합니다. 어느 쪽이든 지금처럼 둘 다 두면 원칙 V 위반입니다.

### B3. `docs/front/projects/search-map-explore.md`, `docs/front/backlog/search-map-schema-request.md`

검색·지도 탐색의 요구사항과 BE 계약 요청이 front 폴더에 있습니다. 같은 기능의 제품 계층 명세는 `docs/features/search.md`입니다.

**문제**: 하나의 기능이 두 계층에서 정의됩니다. 결정 1의 "제품 기능 요구사항을 저장소별로 중복 작성하지 않는다"에 직접 걸립니다.

**제안**: 제품 요구사항은 `features/search.md`(또는 `specs/`)에 남기고, front 문서는 프론트 구현 방식만 다루도록 범위를 좁힙니다.

### B4. `docs/PRODUCT.md` §4.3 ↔ `docs/data-pipeline.md`

신뢰도 등급 기준표가 두 문서에 있습니다.

**제안**: `PRODUCT.md`는 사용자에게 보이는 **정책**("3등급은 미검증 뱃지, 안전 속성은 확인되지 않음"), `data-pipeline.md`는 **필드 설계**(`pipeline_status`·`trust_tier` 저장·제약·인덱스)로 나눕니다. 등급 기준표는 한쪽에만 두고 다른 쪽은 링크합니다. [SSOT.md](SSOT.md) §2.1에 이 분리를 이미 반영했습니다.

---

## 3. 🟠 규칙 충돌 — 저장소별로 다른 판정 기준

### B5. `docs/front/INDEX.md` §2 — SoT 우선순위

front 폴더가 자체 우선순위를 선언합니다. **코드가 3순위**로 `decisions/`보다 위입니다. Constitution Precedence는 코드가 5순위입니다.

실제로 판정이 갈립니다. 문서와 코드가 다를 때 front 규칙은 문서를 고치고 끝내지만, Constitution은 어느 쪽이 틀렸는지 먼저 판단합니다.

**제안**: §2를 삭제하고 [DOCUMENTATION.md](DOCUMENTATION.md) §6으로 위임합니다. 과거에는 이 파일이 다른 저장소의 미러라 여기서 고칠 수 없었지만, 현재는 심볼릭 링크 구조라 **이 파일이 원본이므로 바로 수정 가능합니다.**

### B6. `docs/front/INDEX.md` §1 — 폴더 구조

`reference/`·`templates/` 등 front 전용 분류를 선언합니다. 다른 저장소 폴더는 `decisions/`·`backlog/`·`reviews/`·`projects/`를 씁니다.

**제안**: 실제 폴더는 그대로 두되 분류 기준 서술은 [DOCUMENTATION.md](DOCUMENTATION.md) §1.5로 통합합니다.

### B7. `.claude/rules/vivac-docs-authoring.md`

`docs/features/`, `docs/design/`, `docs/meta/`, `specs/`가 생기기 전에 작성돼 이들을 다루지 않습니다. 또 "여러 repo에 걸친 product 맥락은 `docs/` 루트"라고만 해 제품 계층과 명세 계층을 구분하지 않습니다.

**제안**: [DOCUMENTATION.md](DOCUMENTATION.md)로 내용을 옮기고, 이 파일은 그 문서를 가리키는 포인터로 축소합니다. 각 저장소가 `.claude/rules/`에 심볼릭 링크로 거는 자동 로드 경로이므로 파일 자체는 유지해야 합니다.

---

## 4. 🟡 잔재 — 대체됐으나 남아 있음

| 문서 | 상태 | 제안 |
|---|---|---|
| `docs/feature-spec.md` | `docs/features/`로 대체됨. `archive/planning-source/feature-spec-260804.md`에 스냅샷이 이미 보관됐고 `README.md`도 더는 링크하지 않습니다 | 삭제 |
| `docs/CONTEXT_SCOPE.md` | 참고 범위에 `features/`·`specs/`·`meta/`가 없습니다. "`docs/`는 심볼릭 링크입니다"라는 서술이 cowork 본체에서는 틀립니다 | 갱신 후 [DOCUMENTATION.md](DOCUMENTATION.md)로 흡수 |
| `docs/business-feature-roadmap.md` | `PRODUCT.md` §7.5와 `core/projects/business/05`가 같은 내용을 담습니다 | 3자 대조 후 정본 하나로 정리 |
| `docs/temp-adr/`, `docs/temp-specs/`, `docs/temp-product/` | 미도입 템플릿. `temp-product/README.md`는 0바이트 | 결정 2 확정 후 처분 |

---

## 5. 조치 순서 제안

1. **B5·B6·B7** — 규칙 충돌을 먼저 없앱니다. 판정 기준이 하나여야 이후 이동의 근거가 생깁니다.
2. **B4** — 제품 계층 내부의 중복을 정리합니다. 범위가 작고 정본이 분명합니다.
3. **B1** — `core/projects/business/`를 대조해 흡수 또는 아카이브합니다. 가장 큰 작업입니다.
4. **B2** — `core/projects/devel/`의 성격을 정합니다.
5. **B3** — 검색·지도 기능의 계층을 정리합니다. 진행 중인 기능이라 담당자 확인이 필요합니다.
6. **§4 잔재** — 마지막에 정리합니다.

각 단계는 무엇을 왜 고치는지 먼저 보고한 뒤 진행합니다.
