# SSOT — 정본 문서 목록

> **초안입니다.** 2026-09-07 작성, 같은 날 1차 피드백 반영. 승인 전까지 이 문서는 규범이 아니라 제안입니다.
> 근거: Constitution 원칙 V가 "어떤 문서가 정본인지의 목록은 이 문서가 관리한다"고 위임했습니다.
> 이 문서가 담지 않는 것: 폴더 분류·파일명·작성 방법은 [DOCUMENTATION.md](DOCUMENTATION.md), 불변 원칙과 우선순위는 [constitution.md](constitution.md).
> **경로 표기**: 이 문서의 경로는 현재 구조와 일치합니다. `docs/product/`·`docs/architecture/`·`docs/research/` 신설에 따른 파일 이동은 2026-09-08에 끝났습니다.

## 1. 저장소 경계

VIVAC은 8개 저장소로 구성되지만 **제품 문서의 정본은 `vivac-cowork` 한 곳입니다.**

각 구현 저장소는 자기 `docs/`를 `vivac-cowork/docs/`로 심볼릭 링크해 참조합니다(설정 방법은 [SYMLINK-SETUP.md](../../SYMLINK-SETUP.md)). 즉 구현 저장소에 로컬 `docs/`는 존재하지 않으며, `docs/<약칭>/`이 그 저장소의 문서 자리입니다. **예외는 ADR입니다** — §1.3 참고.

| 계층 | 위치 | 정본이 되는 것 |
|---|---|---|
| **제품** | `vivac-cowork/docs/product/` | 제품 정의, MVP 범위, 정보 구조, 로드맵 |
| **기능 명세** | `vivac-cowork/docs/features/` | 화면별 확정 계약 — 목적·동작·확정 계약·수용 기준 |
| **아키텍처** | `vivac-cowork/docs/architecture/` | 여러 저장소에 걸친 시스템 구조와 계약 |
| **리서치** | `vivac-cowork/docs/research/` | 사용자·시장 조사 근거 |
| **명세** | 각 구현 저장소 `specs/` | 그 저장소의 변경 단위 명세 (Spec Kit을 도입한 저장소만) |
| **결정** | `vivac-cowork/adr/` | 제품·정책·시스템 전반의 의사결정 |
| **거버넌스** | `vivac-cowork/docs/meta/` | Constitution, 정본 목록, 작성 규칙 |
| **저장소별 문서** | `vivac-cowork/docs/<약칭>/` | 해당 저장소의 아키텍처, 설계 노트, 백로그, 코드리뷰 |
| **저장소별 결정** | 각 구현 저장소 `adr/` | 그 저장소 내부의 구현 결정 |
| **코드** | 각 구현 저장소 | 구현, 빌드·배포 설정, 생성물 |

약칭: `front`(VIVAC-frontend) · `console`(vivac-console) · `core`(vivacapi-core) · `etl`(vivacapi-etl) · `mcp`(vivac-mcp) · `infra`(vivac-infra) · `ios`(vivac-ios)

> `docs/architecture/`는 여러 저장소에 걸친 것만 담습니다. `docs/core/architecture.md`, `docs/core/erd.md`, `docs/front/reference/`처럼 한 저장소 안에서 닫히는 구조 문서는 저장소 계층에 남습니다. 이름이 같아도 계층이 다릅니다.

### 1.1 계층 판정 기준

**"이 내용이 저장소 하나만의 사정인가"**로 판단합니다.

| 질문 | 예 → 위치 |
|---|---|
| 사용자에게 보이는 동작이나 제품 약속을 정의하는가 | 제품 계층 |
| 여러 저장소가 함께 지켜야 하는 계약인가 | 제품 계층 또는 `specs/` |
| 왜 그렇게 정했는지를 남기는가 | 결정 계층 (§1.3) |
| 한 저장소의 내부 구조·도구·운영만 다루는가 | `docs/<약칭>/` |
| 코드를 읽으면 알 수 있고 코드와 함께 바뀌는가 | 코드(문서화하지 않음) |

### 1.2 중복 작성 금지

제품 기능의 요구사항을 저장소별로 다시 쓰지 않습니다. **확정 계약은 `docs/features/` 한 곳에 있고, 각 저장소는 그것을 참조해 구현합니다.**

```text
vivac-cowork/docs/features/search.md   ← 확정 계약 (모든 저장소가 참조)
        ├─→ VIVAC-frontend 구현   (specs/ 로 변경 단위를 쪼갬)
        ├─→ vivacapi-core 구현     (docs/core/ + adr/ 로 진행)
        └─→ vivac-infra 구현
```

저장소 내부의 구현 방법은 그 저장소가 정합니다.

### 1.3 ADR의 이원 구조

결정의 **영향 범위**로 위치를 정합니다.

| 결정의 범위 | ADR 위치 |
|---|---|
| 제품 비즈니스 로직 | `vivac-cowork/adr/` |
| 제품 요구사항·Feature 정책 | `vivac-cowork/adr/` |
| 여러 저장소에 영향을 주는 기술 결정 | `vivac-cowork/adr/` |
| 전체 시스템 아키텍처 결정 | `vivac-cowork/adr/` |
| 특정 저장소의 구현 결정 | 해당 저장소 `adr/` |
| 특정 프레임워크·라이브러리 사용 방식 | 해당 저장소 `adr/` |
| 저장소 내부 구조 결정 | 해당 저장소 `adr/` |
| 단순 구현 변경 | ADR 불필요 |

두 ADR의 관계는 **Parent ADR / Related ADR**로 연결합니다. 규칙은 [DOCUMENTATION.md](DOCUMENTATION.md) §5.

저장소 `adr/`는 `docs/` 심볼릭 링크 대상이 아니라 그 저장소의 git에 직접 들어갑니다. 코드와 함께 리뷰·버전 관리됩니다.

**저장소 ADR은 `vivac-cowork`에서 색인하지 않습니다.** 로컬 결정은 그 저장소 안에서만 활용합니다. 다른 저장소나 비즈니스 로직이 알아야 할 결정은 정의상 cowork ADR이므로, cowork만 봐도 제품·시스템 결정은 전수 조회됩니다.

따라야 하는 것이 둘 있습니다.

1. `docs/STATUS.md` §6 결정 로그는 **cowork ADR만** 다룹니다. 저장소 내부 결정은 담지 않으며, 그 범위를 문서 머리말에 밝힙니다.
2. 저장소 ADR이 다른 저장소나 비즈니스 로직에 영향을 준다는 것이 드러나면, 그 결정은 저장소 ADR로 끝낼 수 없습니다. 팀 논의 → cowork ADR 기록 → 기획 문서 수정으로 올려야 하며, 이 절차는 [DOCUMENTATION.md](DOCUMENTATION.md) §5.4가 강제합니다.

## 2. 정본 목록

### 2.1 제품 계층 — `docs/product/`

| 정보 | 정본 |
|---|---|
| 제품 정의, 문제·가설, MVP 판정 기준, 단계 로드맵 | `docs/product/PRODUCT.md` |
| 데이터 정의(category 어휘, 노지 등재 요건, 채움률 목표) | `docs/product/PRODUCT.md` §4 |
| 신뢰도·노출 **정책** | `docs/product/PRODUCT.md` §4.3 |
| 정보 구조·사이트맵·라우팅 | `docs/product/ia.md` |
| 기능 후보와 우선순위 | `docs/product/business-feature-roadmap.md` |

### 2.2 기능 명세 계층 — `docs/features/`

| 정보 | 정본 |
|---|---|
| 화면별 확정 계약 | `docs/features/<화면>.md` |

목적 → 동작 → 확정 계약 → 수용 기준 4블록으로 기술합니다. 저장소를 넘는 계약이므로 모든 구현 저장소가 이 문서를 참조합니다.

### 2.3 아키텍처 계층 — `docs/architecture/`

| 정보 | 정본 |
|---|---|
| 신뢰도·노출 **필드 설계**(`pipeline_status`·`trust_tier`) | `docs/architecture/data-pipeline.md` |
| API 계약 | `docs/architecture/openapi.json` — `vivacapi-core`에서 생성해 동기화합니다 |

### 2.4 리서치 계층 — `docs/research/`

| 정보 | 정본 |
|---|---|
| 사용자·시장 조사 근거 | `docs/research/research_backpacking_market.md` |

### 2.5 명세 계층 — 각 저장소 `specs/`

`vivac-cowork`는 SSOT 저장소이며 Spec Kit을 사용하지 않습니다. SDD 워크플로는 이를 도입한 구현 저장소에서만 돕니다. 2026-09-07 기준 `VIVAC-frontend` 한 곳입니다.

| 정보 | 정본 |
|---|---|
| Feature Specification | `<repo>/specs/<NNN-feature>/spec.md` |
| Implementation Plan | `<repo>/specs/<NNN-feature>/plan.md` |
| Tasks | `<repo>/specs/<NNN-feature>/tasks.md` |

경로는 Spec Kit이 `$REPO_ROOT/specs`로 고정하므로 바꿀 수 없습니다.

**저장소 `specs/`는 제품 요구사항의 정본이 아닙니다.** 저장소를 넘는 확정 계약은 `docs/features/`에 있고, `specs/`는 그 계약을 그 저장소에서 어떻게 바꿔 나가는지를 담습니다. 경계는 [DOCUMENTATION.md](DOCUMENTATION.md) §4를 따릅니다.

Spec Kit이 참조하는 Constitution은 각 저장소에서 상대경로 심볼릭 링크로 연결합니다.

```bash
# 예: VIVAC-frontend (docs 심볼릭 링크가 이미 걸려 있어야 합니다)
mkdir -p .specify/memory
ln -s ../../docs/meta/constitution.md .specify/memory/constitution.md
```

저장소 내부 상대경로라 git에 커밋되며, 사람마다 따로 설정할 필요가 없습니다.

### 2.6 결정 계층 — `adr/`

| 정보 | 정본 |
|---|---|
| 제품·정책·시스템 전반 결정 | `vivac-cowork/adr/ADR-NNN-<slug>.md` |
| 저장소 내부 구현 결정 | `<repo>/adr/<약칭>-ADR-NNN-<slug>.md` |

기존에 결정을 담고 있는 문서는 영향 범위에 따라 행선지가 갈립니다.

| 문서 | 결정의 성격 | 행선지 |
|---|---|---|
| `docs/map-explore-discussion.md` | 지도 탐색 제품 결정의 근거 | `vivac-cowork/adr/` |
| `docs/design/decisions/spot-detail-design-decisions.md` | 상세페이지 UI 결정 — 제품 계약에 가까움 | `vivac-cowork/adr/` (판단 필요) |
| `docs/core/projects/spot-search-postgres-fts.md` | Elasticsearch 대신 PostgreSQL FTS | `vivacapi-core/adr/` |
| `docs/core/projects/async-job-worker-design.md` | 외부 브로커 없이 내장 폴링 워커 | `vivacapi-core/adr/` |
| `docs/etl/decisions/source1_transform_changelog.md` | 변환 규칙 변경 이력 | `vivacapi-etl/adr/` |
| `docs/front/decisions/incidents/*`, `docs/core/troubleshooting/*` | 사건 기록 | `docs/<약칭>/incidents/` |

> 인시던트 기록은 ADR이 아닙니다. 결정이 아니라 사건이므로 저장소로 내보내지 않고 `docs/<약칭>/incidents/`에 남깁니다.
> 저장소 `adr/`로 가는 문서는 `vivac-cowork`에서 제거됩니다. 이동 시점과 방법은 §4의 실행 계획 항목입니다.

### 2.7 거버넌스 계층

| 정보 | 정본 |
|---|---|
| 불변 원칙, 우선순위, 결정 권한의 경계 | `docs/meta/constitution.md` |
| 정본 목록, 저장소 경계 | 이 문서 |
| 폴더 분류, 파일명, 템플릿, 작성 방법 | `docs/meta/DOCUMENTATION.md` |
| AI agent 실행 지침 | `CLAUDE.md`, `.claude/rules/` |

### 2.8 저장소별 계층 — `docs/<약칭>/`

| 정보 | 정본 |
|---|---|
| 백엔드 아키텍처 | `docs/core/architecture.md` |
| DB 스키마 | `docs/core/erd.md` |
| Enum 값 | `docs/core/enums.md` |
| 프론트 아키텍처·배포 | `docs/front/reference/` |
| 디자인 스펙·토큰 | `docs/design/` |
| 콘솔 화면-데이터 매핑 | `docs/console/spot-sdp-field-mapping.md` |
| ETL 변환 규칙 | `docs/etl/` |

## 3. 정본이 아닌 것

참조·집계·보존 목적이며 정본으로 인용하지 않습니다.

| 문서 | 성격 |
|---|---|
| `docs/INDEX.md` | 탐색용 인덱스 |
| `docs/STATUS.md` | 파생 집계 뷰 |
| `docs/archive/**` | 폐기 보존. Constitution Precedence상 구현 근거로 쓰지 않습니다 |
| `docs/core/projects/business/**` | 2026-08-01 스냅샷. 제품 정의는 `PRODUCT.md`가 정본입니다 |
| `docs/core/projects/devel/**` | 2026-08-01 스냅샷. 스키마·API는 `erd.md`·`enums.md`·`openapi.json`이 정본입니다 |

## 4. 미해결

승인 전 정리가 필요한 항목입니다. 문서별 상세는 [boundary-conflicts-260907.md](boundary-conflicts-260907.md).

| # | 항목 | 성격 |
|---|---|---|
| U1 | §2.6의 저장소 ADR 대상 문서를 각 저장소 `adr/`로 내보내는 시점과 방법 | 실행 계획 필요 |
| U2 | `docs/core/projects/business/`가 제품 정의를 저장소 계층에서 재서술 | 흡수 또는 아카이브 |
| U3 | `docs/meta/templates/` 4종(ADR · Feature 명세 · Reference · Incident)을 새 기준으로 작성 | 항목별 개별 작업 |
| U4 | `VIVAC-frontend`에 Spec Kit 설치와 Constitution 심볼릭 링크 연결 | 별도 저장소 작업 |
| U5 | archive된 `feature-spec-260804.md`의 3·4부(그룹·리뷰·지도·합법성·제보 화면 초안)가 대체 문서 없이 `PRODUCT.md`·`ia.md`에서 현역으로 인용됨 | 승격 위치 결정 필요 |

U3는 기존 템플릿을 개조하지 않고 기준부터 새로 정해 작성합니다. 4종이 모두 나온 뒤 `docs/front/templates/` 3종과 `docs/temp-adr/`·`docs/temp-specs/`·`docs/temp-product/`를 폐기합니다. `docs/temp-specs/`의 Feature Specification 초안은 cowork가 아니라 `VIVAC-frontend`의 Spec Kit 템플릿 오버라이드로 넘깁니다.

**해소됨** — 저장소 ADR의 색인 방식(§1.3)과 기존 `docs/<약칭>/decisions/`의 처분(§2.6)은 2026-09-07 확정했습니다. `docs/`를 관심사별 폴더로 재구성하는 파일 이동과 링크 갱신, `docs/CONTEXT_SCOPE.md`의 참고 범위 갱신은 2026-09-08 완료했습니다. `docs/front/INDEX.md`의 SoT 우선순위·작성 원칙 충돌은 2026-09-08 `docs/meta/DOCUMENTATION.md`로 위임해 해소했습니다. `docs/feature-spec.md`는 archive 사본과 본문이 동일하고 이 파일을 가리키는 링크가 없어 2026-09-08 삭제했습니다 — 내용은 [archive/planning-source/feature-spec-260804.md](../archive/planning-source/feature-spec-260804.md)에 보존돼 있습니다.
