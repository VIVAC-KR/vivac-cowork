# DOCUMENTATION — 문서 작성 방법

> **초안입니다.** 2026-09-07 작성, 같은 날 1차 피드백 반영. 승인 전까지 이 문서는 규범이 아니라 제안입니다.
> 이 문서가 담지 않는 것: 어떤 문서가 정본인지는 [SSOT.md](SSOT.md), 불변 원칙과 우선순위는 [constitution.md](constitution.md).
> 대체 대상: 승인되면 `.claude/rules/vivac-docs-authoring.md`, `docs/INDEX.md` §8의 규칙을 이 문서로 일원화합니다. `docs/front/INDEX.md`의 SoT 우선순위·작성 원칙과 `docs/CONTEXT_SCOPE.md`의 중복 서술은 2026-09-08 이 문서로 위임 완료했습니다.

## 1. 어디에 쓸 것인가

먼저 [SSOT.md](SSOT.md) §1.1의 계층 판정을 거친 뒤, 계층 안에서 폴더를 고릅니다.

### 1.1 제품 계층 — `docs/product/`

| 문서 성격 | 위치 |
|---|---|
| 제품 정의·범위·데이터 정의 | `docs/product/PRODUCT.md` (단일 파일) |
| 정보 구조 | `docs/product/ia.md` |
| 기능 후보·로드맵 | `docs/product/business-feature-roadmap.md` |

### 1.2 기능 명세 계층 — `docs/features/`

화면 하나를 만들 때 필요한 기능 계약을 **자족적으로** 담는 계층입니다. 구현자가 다른 문서를 열지 않고도 그 화면을 만들 수 있어야 하므로, 정본이 정의한 값을 문서 안에 옮겨 적습니다. 옮겨 적는 조건은 §4.1 규칙 5와 Constitution 원칙 V의 한정 규정을 따릅니다.

**플랫폼 무관입니다.** 레이아웃, 영역 ID, 반응형 규칙은 여기 쓰지 않습니다. 그것은 `docs/design/`의 몫입니다.

| 문서 성격 | 위치 |
|---|---|
| 화면별 기능 명세 | `docs/features/<화면>.md` |
| 라우트 없이 여러 화면이 공유하는 로직 | `docs/features/_shared/<주제>.md` |

화면 명세는 §7의 `docs/meta/templates/feature-template.md`가 정한 10개 절 구성(1. 목적과 비목표 / 2. 사용자와 진입 / 3. 표시 항목 계약 / 4. 안전·신뢰 표시 계약 / 5. 상호작용 / 6. 상태 / 7. 입력과 검증(조건부) / 8. 문구·메시지 / 9. 수용 기준 / 10. 검토 체크리스트)을 따릅니다.

`_shared/`의 공용 로직 문서는 대응하는 라우트가 없어 화면 명세가 아닙니다. 10개 절 구성과 자족성 요구가 적용되지 않으며, 각 화면 명세가 참조하는 방식으로 씁니다.

### 1.3 아키텍처 계층 — `docs/architecture/`

여러 저장소에 걸친 시스템 구조와 계약만 담습니다. 한 저장소 안에서 닫히는 구조 문서는 저장소 계층(`docs/<약칭>/`)에 씁니다.

| 문서 성격 | 위치 |
|---|---|
| 저장소를 넘는 데이터·필드 설계 | `docs/architecture/<주제>.md` |
| API 계약 | `docs/architecture/openapi.json` |

### 1.4 리서치 계층 — `docs/research/`

| 문서 성격 | 위치 |
|---|---|
| 사용자·시장 조사 근거 | `docs/research/<주제>.md` |

### 1.5 저장소 계층 — `docs/<약칭>/`

| 문서 성격 | 위치 |
|---|---|
| 미착수·우선순위 대기 항목 | `docs/<약칭>/backlog/` |
| 날짜별 코드리뷰·문서감사 스냅샷 | `docs/<약칭>/reviews/` |
| 장애 기록 | `docs/<약칭>/incidents/` |
| 여러 결정이 묶인 기능·API 설계 문서 | `docs/<약칭>/projects/` |
| 안정적 레퍼런스(아키텍처, ERD, 필드 매핑) | `docs/<약칭>/` 루트 |

새 폴더는 그 유형의 문서가 실제로 생길 때만 만듭니다. 빈 폴더를 미리 만들지 않습니다.

### 1.6 명세 계층 — 각 구현 저장소 `specs/`

`vivac-cowork`에는 없습니다. Spec Kit을 도입한 구현 저장소에만 존재하며, 사람이 직접 폴더를 만들지 않고 그 저장소에서 `/speckit-specify`로 생성합니다. 경로는 `$REPO_ROOT/specs`로 고정돼 있습니다.

기능 이름 슬러그는 영문에서만 생성되므로 **한글 설명을 줄 때는 `--short-name`을 반드시 함께 지정합니다.** 생략하면 `001-`처럼 이름 없는 폴더가 만들어집니다.

```bash
/speckit-specify --short-name map-explore "지도에서 스팟을 탐색하는 기능"
```

### 1.7 결정 계층 — `adr/`

영향 범위로 위치가 갈립니다. 판정표는 [SSOT.md](SSOT.md) §1.3에 있습니다. 규칙은 아래 §5.

## 2. 파일명

- kebab-case, 확장자 `.md`
- 날짜가 의미 있는 문서(리뷰, backlog 개별 항목, 스냅샷)는 끝에 `-YYMMDD`를 붙입니다. 예: `codebase-review-260714.md`
- 번호로 읽는 순서를 강제하는 문서 묶음은 `NN-` prefix를 씁니다. 예: `01-data-model.md`
- ADR은 §5.1의 명명 규칙을 따릅니다.

## 3. 톤과 표기

- 본문 문장은 "~입니다/~합니다" 서비스 말투로 씁니다.
- 서비스명은 항상 `vivac`/`VIVAC`입니다. 한글 표기(비바크 등)는 쓰지 않습니다. 다만 `비박`은 야영 형태를 가리키는 도메인 용어로 쓸 수 있습니다.
- 기술 용어(함수명, 엔드포인트, 필드명)는 영문 그대로 백틱으로 표기합니다.
- 규범을 기술할 때는 MUST / MUST NOT / SHOULD를 씁니다.

## 4. 기능 명세: `features/`와 `specs/`의 경계

두 곳 모두 "이 기능이 무엇을 해야 하는가"를 다루지만 **축과 수명이 다릅니다.** `docs/features/`는 화면 축의 지속 문서이고, `specs/`는 변경 단위 축의 일시 문서입니다. **경계를 명시하지 않으면 지금 유효한 계약이 어느 쪽인지 판별할 수 없습니다.** 문제는 같은 서술이 두 곳에 있다는 것이 아니라, 수명이 다른 두 문서가 각자 계약을 주장하는 것입니다.

| | `docs/features/` | `<repo>/specs/<NNN-feature>/` |
|---|---|---|
| 저장소 | `vivac-cowork` | 구현 저장소 |
| 축 | **화면** (홈·검색·상세·인증) + 라우트 없는 공용 로직 `_shared/` | **변경 단위** (이번에 만들 것) |
| 답하는 질문 | 지금 이 화면의 계약이 무엇인가 | 무엇을 어떻게 바꿀 것인가 |
| 수명 | 지속. 제품이 사는 동안 유지 | 일시. 구현이 끝나면 이력 |
| 상태 | 확정만 기록 | `Draft` → 승인 → 구현 → 완료 |
| 구성 | 10개 번호 절 (§1.2 · `templates/feature-template.md`) | User Story → Requirements → Success Criteria |
| 이어지는 것 | 없음. **화면 계약의 정본**이자, 제품·데이터 정의에 대해서는 **정본을 지목한 사본을 싣는 문서**입니다(§4.1 규칙 5) | `plan.md` → `tasks.md` → 구현 |
| 성격 | Reference | Change proposal |

### 4.1 경계를 지키는 규칙

1. **`specs/`는 델타만 씁니다.** 현재 계약을 옮겨 적지 않고 `docs/features/<화면>.md`를 링크한 뒤 바뀌는 부분만 기술합니다. `specs/`는 수명이 짧은 변경 제안이라 그 안의 사본을 정본과 맞춰 줄 장치가 없고, 구현이 끝나 문서가 이력이 되는 순간 그대로 낡습니다. 반대로 `features/`는 지속 문서이며 아래 규칙 5의 정본 지목·동반 갱신 의무를 지므로 사본이 낡지 않게 유지됩니다.
2. **완료의 정의에 반영을 포함합니다.** 저장소가 다르므로 자동으로 따라가지 않습니다. 구현이 끝나면 확정된 계약을 `features/`에 반영해야 그 명세가 완료입니다. 반영하지 않으면 `features/`가 낡고 §6의 충돌로 되돌아옵니다.
3. **`features/`에는 진행 중인 것을 쓰지 않습니다.** 확정되지 않은 항목은 `specs/`에 있거나 `PRODUCT.md` §7 열린 항목에 있습니다.
4. **한 화면을 여러 명세가 건드릴 수 있습니다.** 반대로 한 명세가 여러 화면을 바꿀 수도 있습니다. 1:1 대응을 가정하지 않습니다.
5. **`features/`는 정본이 정의한 값을 옮겨 적을 수 있습니다.** Constitution 원칙 V의 한정 규정에 따라 세 조건을 **모두** 지킬 때만 성립합니다. (1) 옮겨 적은 값마다 정본 문서와 절을 함께 표시합니다. (2) 정본과 다른 값을 적거나 새 값을 정하지 않습니다 — 어긋나면 정본이 맞고 기능 명세가 틀린 것입니다. (3) 정본을 수정할 때 그 값을 옮겨 적은 기능 명세를 함께 갱신합니다. 정본 표시가 없는 값은 이 문서가 새로 정의한 것이 되어 한정을 벗어나며, 원칙 V의 재정의 금지에 걸립니다.

> 예: `features/search.md`는 "정렬 — MVP 미제공"을 확정 계약으로 못박고 있습니다. 정렬을 도입하려면 `specs/`에 명세를 만들고, 구현 완료 시 `features/search.md`의 그 줄을 갱신합니다. 명세 안에 정렬 계약을 남긴 채 끝내면 두 곳이 갈라집니다.

## 5. ADR

### 5.1 명명과 번호

| 위치 | 파일명 | 예 |
|---|---|---|
| `vivac-cowork/adr/` | `ADR-NNN-<slug>.md` | `ADR-007-map-sdk-selection.md` |
| `<repo>/adr/` | `<약칭>-ADR-NNN-<slug>.md` | `core-ADR-003-fts-index-strategy.md` |

번호는 각 위치에서 독립적으로 증가합니다. 저장소 접두어를 붙이는 이유는 서로 다른 저장소의 `ADR-003`이 충돌하지 않게 하기 위해서입니다.

### 5.2 Parent ADR / Related ADR

모든 ADR은 머리말에 아래를 적습니다.

```markdown
> Status: Proposed | Accepted | Superseded
> Date: YYYY-MM-DD
> Parent ADR: <상위 결정, 없으면 "없음">
> Related ADR: <함께 읽어야 하는 결정, 없으면 생략>
> Superseded By: <대체한 ADR, 해당 시>
> Cross-repo impact: <저장소 ADR 전용 — §5.4>
> 영향 저장소: <cowork ADR 전용 — 아래 규칙 2>
> 승인자: <승인한 사람, 승인 전에는 "미승인">
```

필드마다 적용 대상이 다릅니다.

| 필드 | cowork ADR | 저장소 ADR |
|---|---|---|
| `Status` · `Date` · `Parent ADR` · `승인자` | 필수 | 필수 |
| `Related ADR` · `Superseded By` | 해당할 때만 | 해당할 때만 |
| `Cross-repo impact` | **쓰지 않습니다** | 필수 |
| `영향 저장소` | 필수 | 쓰지 않습니다 |

`승인자`를 적는 이유는 Constitution 원칙 VI가 승인을 사람의 권한으로 규정하기 때문입니다. 누가 승인했는지 남지 않으면 그 원칙이 지켜졌는지 확인할 수 없습니다. AI agent는 초안을 쓰고 선택지를 제시할 수 있지만 이 칸을 채우거나 `Status`를 `Accepted`로 올릴 수 없습니다.

관계 규칙입니다.

1. **저장소 ADR이 cowork ADR을 구현할 때 `Parent ADR`로 그 cowork ADR을 가리킵니다.** 링크는 저장소 → cowork **단방향**입니다. 역방향 목록을 cowork에 두면 저장소가 늘 때마다 갱신해야 해 낡습니다.
2. **cowork ADR은 `영향 저장소`를 나열합니다.** 어느 저장소가 이 결정을 구현해야 하는지만 적고, 그쪽 ADR 번호는 적지 않습니다.
3. **`Parent ADR`이 없는 저장소 ADR은 그 저장소 안에서 닫힌 결정입니다.** 다른 저장소에 영향이 있다면 저장소 ADR이 아니라 cowork ADR이어야 합니다.
4. **상위 결정이 뒤집히면 자식 ADR을 검토합니다.** cowork ADR을 `Superseded`로 바꿀 때, 그 ADR을 `Parent`로 가리키는 저장소 ADR이 여전히 유효한지 확인해야 합니다. 확인 결과를 새 ADR에 적습니다.
5. **결정이 아니면 ADR로 쓰지 않습니다.** 장애는 `docs/<약칭>/incidents/`, 미해결 항목은 `docs/<약칭>/backlog/`, 현재 유효한 사실은 `docs/<약칭>/` 루트나 `docs/architecture/`에 씁니다.

### 5.3 무엇을 ADR로 남기는가

[SSOT.md](SSOT.md) §1.3의 판정표를 따릅니다. 판단이 서지 않으면 **영향 범위가 저장소를 넘는지**만 봅니다. 넘으면 cowork, 안 넘으면 저장소입니다. 단순 구현 변경은 ADR을 만들지 않습니다.

### 5.4 저장소 ADR의 영향 범위 신고

저장소 ADR은 그 저장소 안에서만 활용하며 `vivac-cowork`에서 색인하지 않습니다([SSOT.md](SSOT.md) §1.3). 그래서 **저장소 안에 갇혀서는 안 되는 결정**이 그대로 묻히지 않도록, 모든 저장소 ADR은 머리말에 아래 필드를 반드시 포함합니다.

**cowork ADR은 이 필드를 쓰지 않습니다.** 이 필드는 "이 결정을 상위로 올려야 하는가"를 묻는 것인데, cowork ADR은 이미 상위 결정이므로 질문이 성립하지 않습니다.

```markdown
> Cross-repo impact: none
```

값은 둘 중 하나만 허용합니다.

| 값 | 의미 |
|---|---|
| `none` | 이 저장소 안에서 닫힌 결정입니다 |
| cowork ADR 링크 | 다른 저장소나 비즈니스 로직에 영향이 있어 상위 결정으로 올렸습니다 |

`Parent ADR`과 혼동하지 않습니다. `Parent ADR`은 "이 결정이 어느 상위 결정을 구현하는가"이고, `Cross-repo impact`는 "이 결정이 상위로 올라가야 하는가"입니다. 방향이 반대입니다.

#### 영향이 있을 때의 절차

저장소 ADR만으로 끝낼 수 없습니다. 순서대로 진행합니다.

1. 저장소 ADR을 `Proposed` 상태로 두고 **머지하지 않습니다**
2. 팀에서 영향 범위를 논의합니다
3. `vivac-cowork/adr/`에 상위 ADR을 기록합니다
4. 영향받는 기획 문서(`docs/product/PRODUCT.md`, `docs/features/`, `docs/product/ia.md` 등)를 수정합니다
5. 저장소 ADR의 `Cross-repo impact`에 3에서 만든 cowork ADR을 링크하고 `Parent ADR`도 채운 뒤 머지합니다

#### 기계 검증

이 필드는 자기 신고이므로 신고만으로는 강제가 되지 않습니다. **CI가 신고 내용과 실제 변경을 대조해 모순을 잡습니다.**

> `Cross-repo impact: none`으로 신고된 PR이 저장소 경계를 넘는 산출물을 함께 변경했다면 **검사를 실패시킵니다.**

경계를 넘는 산출물은 저장소마다 다르며, 예를 들어 `vivacapi-core`에서는 OpenAPI 명세 생성 지점, DB 마이그레이션, `pipeline_status`·`trust_tier` 제약, 전역 에러 코드가 여기 해당합니다. **저장소별 대상 경로 목록은 아직 확정되지 않았습니다** — CI를 도입하는 시점에 각 저장소에서 정합니다.

빠져나갈 길은 하나만 둡니다. 메인테이너가 `no-product-impact` 라벨을 명시적으로 붙이는 경우입니다. 라벨은 누가 언제 붙였는지 남으므로, 나중에 문제가 되어도 "아무도 몰랐다"가 아니라 "누가 판단했다"가 됩니다.

체크박스("영향이 있습니까? [ ]")를 쓰지 않는 이유가 여기 있습니다. 체크박스는 영향을 인지하지 못한 사람 앞에서 아무것도 막지 못하고, 검토했다는 착각만 만듭니다. 이 규칙은 **판정을 묻는 대신 신고와 diff를 대조**합니다.

## 6. 문서 간 규칙

Constitution 원칙 V에서 나옵니다.

- **재기술 금지가 기본값** — 다른 문서가 정의한 것을 다시 정의하지 않고 링크합니다.
- **한정된 예외는 `docs/features/` 한 곳입니다** — 구현자가 다른 문서를 열지 않고 한 화면을 만들 수 있어야 하므로, 기능 명세는 정본이 정의한 값을 옮겨 적을 수 있습니다. 정본 지목 표시 · 정본과 동일한 값 · 정본 수정 시 동반 갱신 세 조건을 모두 지킬 때만 성립하며, 정본 표시가 없는 값은 새 정의가 되어 예외를 벗어납니다(§4.1 규칙 5).
- **정본 명시** — 요약·집계 문서는 머리말에 "이 문서는 파생이며 정본이 아닙니다"를 적습니다.
- **뒤집힌 결정 보존** — 결정이 바뀌면 원래 결정과 바뀐 이유를 함께 남깁니다. 지우지 않습니다.
- **폐기 표시** — archive로 옮길 때 대체 문서·폐기일·사유를 상단에 적습니다. 대체 문서가 없으면 "없음"과 그 사유를 적습니다.
- **새 문서를 만들면** `docs/INDEX.md`에 한 줄을 추가합니다.

## 7. 템플릿

| 유형 | 템플릿 |
|---|---|
| 기능 명세(화면) | [`docs/meta/templates/feature-template.md`](templates/feature-template.md) — §1.2의 10개 절 구성 |
| ADR | [`docs/meta/templates/adr-template.md`](templates/adr-template.md) — §5.2의 머리말 항목을 포함합니다 |
| 인시던트 | [`docs/meta/templates/incident-template.md`](templates/incident-template.md) |
| Feature Specification (`specs/`) | `.specify/templates/spec-template.md` — Spec Kit을 도입한 구현 저장소가 관리합니다. 초안은 [`templates/frontend-spec-template-draft.md`](templates/frontend-spec-template-draft.md) |

> `docs/meta/templates/` 3종이 정본 템플릿입니다. `docs/front/templates/` 3종과 `docs/temp-adr/`·`docs/temp-specs/`·`docs/temp-product/`는 2026-09-08 삭제했습니다([SSOT.md](SSOT.md) §4 U3). Reference 템플릿은 만들지 않았습니다 — 필요 여부부터 판단할 항목이며, 그때까지 안정적 레퍼런스 문서에는 정해진 템플릿이 없습니다.

## 8. 문서 충돌 판정

Constitution Conflict Resolution을 따릅니다. **저장소별로 다른 우선순위를 두지 않습니다.**

1. Constitution
2. 제품 정의(`PRODUCT.md`)와 승인된 Feature Specification
3. 정본 참조 문서([SSOT.md](SSOT.md) §2)
4. `CLAUDE.md` 및 실행 지침
5. 코드와 관행

문서와 코드가 다르면 **문서가 의도이고 코드가 현실입니다.** 어느 쪽이 틀렸는지 판단해 고치고, 의도를 확정할 수 없으면 사람에게 묻습니다. 조용히 맞추지 않습니다. 안전·합법성이 걸린 충돌은 보수적인 쪽을 잠정 기준으로 삼고 즉시 보고합니다.

## 9. 저장소별 예외

없습니다. `docs/front/`를 포함한 모든 폴더가 이 문서의 규칙을 따릅니다.

> 과거 `docs/front/`는 VIVAC-frontend 저장소의 `docs/`를 복사한 미러였고 자체 규칙(`front/INDEX.md`)을 가졌습니다. 현재는 그 저장소가 `docs/`를 심볼릭 링크로 연결하므로 미러가 아니라 원본입니다. 충돌하던 SoT 우선순위와 작성 원칙은 2026-09-08 이 문서로 위임했고, `front/INDEX.md`는 폴더 구성과 먼저 읽을 문서만 안내합니다.
