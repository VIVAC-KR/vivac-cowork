# 문서 마이그레이션 — 사람이 결정해야 하는 6건

> ⚠️ **이 문서는 `main` 머지 이전(문서 69개) 기준으로 작성됐습니다.** 2026-09-07 `main`의 23커밋을 머지한 결과 문서가 107개가 됐고, 여기서 미해결로 보고한 여러 항목이 이미 해소됐습니다 — `PRODUCT_TEMP.md`·`TEMP.md` 삭제, 기능 명세의 `docs/features/` 분리, `PRODUCT.md` 2026-08-10 개정, `docs/design/` 편입에 따른 링크 정리, `openapi.json` 버전 명시 동기화 관행이 그렇습니다. 최신 기준 분석은 [boundary-conflicts-260907.md](boundary-conflicts-260907.md)를 참고하세요.

> 작성일: 2026-09-07
> 배경: [doc-inventory-260907.md](doc-inventory-260907.md)의 마이그레이션 계획 중, AI agent가 단독으로 정할 수 없는 항목만 뽑아 선택지와 근거를 정리했습니다.
> 결정 권한의 근거: Constitution 원칙 VI는 제품 방향, MVP 범위, 비즈니스 정책, 정본 문서의 정의를 사람의 권한으로 규정합니다. 아래 6건은 모두 그 범주에 속합니다.
> 사용법: 각 결정의 "선택" 칸에 고르신 안을 적어주시면 그대로 진행합니다.

---

# 결정 1 — Spec Kit 워크플로를 어느 저장소에서 돌릴 것인가

**선택**: (미정)

## 확인된 사실

- Spec Kit의 `create-new-feature.sh:191`은 명세 위치를 `SPECS_DIR="$REPO_ROOT/specs"`로 **하드코딩**합니다. 플래그로 바꿀 수 없고 `docs/` 하위로 옮길 수도 없습니다.
- 다만 git 브랜치를 강제로 만들지는 않습니다. 기능 상태는 `SPECIFY_FEATURE` 환경변수와 `.specify/feature.json`으로 추적되므로, 브랜치 없이 한 저장소 안에서 여러 명세를 병행할 수 있습니다.
- 현재 Spec Kit과 Constitution은 `vivac-cowork`에만 설치되어 있습니다. 실제 구현은 `vivacapi-core`, `VIVAC-frontend`, `vivac-console`, `vivacapi-etl`, `vivac-mcp` 5개 저장소에서 일어납니다.
- 로컬에 실제로 체크아웃된 저장소는 `vivac-cowork`, `vivac-frontend`, `vivacapi-core` 3개입니다.
- VIVAC의 기능은 대부분 저장소를 가로지릅니다. 예를 들어 그룹 기능은 API가 core, 화면이 front에 있고, 데이터 검증 화면은 console과 core에 걸쳐 있습니다.

## 선택지

| 안 | 구조 | 장점 | 단점 |
|---|---|---|---|
| **A. cowork 단일 허브** | 모든 명세를 `vivac-cowork/specs/`에 두고, 구현만 각 저장소에서 수행 | Constitution 1개로 통제됩니다. 저장소를 가로지르는 기능을 한 명세로 표현할 수 있습니다. 문서 허브가 이미 cowork이므로 추가 이동이 없습니다 | `/speckit-plan`·`/speckit-tasks`가 코드 없는 저장소에서 실행됩니다. 명세와 코드가 물리적으로 떨어져 있어 구현 시 교차 참조가 필요합니다 |
| **B. 저장소별 설치** | 5개 저장소에 각각 Spec Kit을 설치하고, Constitution은 cowork를 정본으로 두고 각 저장소가 참조 | 명세가 코드 옆에 있어 Plan·Tasks 단계가 자연스럽습니다 | Constitution 사본이 5개로 늘어 원칙 V(하나의 사실은 한 곳에서 정의)와 정면으로 부딪힙니다. 저장소를 가로지르는 기능을 표현할 곳이 없습니다 |
| **C. 하이브리드** | 여러 저장소에 걸친 제품 레벨 명세는 cowork, 한 저장소에 갇힌 기술 명세는 해당 저장소 | 각각의 장점을 취합니다 | "이 기능은 어디에 쓰는가"를 매번 판단해야 하고, 그 판단 기준 자체가 새 규칙이 됩니다 |

## 권장 — A

이유는 세 가지입니다.

1. `feature-spec.md`가 정리한 12개 기능 중 단일 저장소로 끝나는 것은 사실상 없습니다. 지도 탐색은 front 화면과 core 위경도 데이터가 함께 필요하고, 제보는 front 폼과 core 신규 모델이 함께 필요합니다.
2. B는 Constitution을 5벌 복제하는데, 이는 Constitution 스스로가 금지한 구조입니다.
3. Plan 단계에서 "이 명세는 core와 front에서 구현한다"고 명시하면 코드가 같은 저장소에 없다는 단점은 대부분 해소됩니다.

**이 결정이 막고 있는 것**: Phase 3(Feature Specification 분해) 전체. 위치가 정해지지 않으면 명세를 하나도 만들 수 없습니다.

---

# 결정 2 — Feature Spec·ADR 템플릿을 무엇으로 단일화할 것인가

**선택**: (미정)

## 확인된 사실

현재 후보가 서로 다른 성격으로 4개 존재합니다.

| 템플릿 | 위치 | 구성 |
|---|---|---|
| Spec Kit 기본 | `.specify/templates/spec-template.md` | User Story(P1/P2/P3 우선순위) 중심 + Requirements + Success Criteria + Assumptions. 각 스토리가 독립 배포 가능해야 한다는 전제 |
| VIVAC 자체안 | `docs/temp-specs/_template.md` | 17절. Problem·Goal·User Flow·FR·**Business Rules**·Data Requirements·**Edge Cases**·**Error Handling**·AC·Non-Goals·Open Questions·Change Log·DoD |
| ADR(간결) | `docs/front/templates/adr-template.md` | 결정 이유만 기록. "현재 스펙은 여기 쓰지 말고 reference에 링크" 원칙 명시 |
| ADR(상세) | `docs/temp-adr/_template.md` | 8절. Context·Decision Drivers·Options Considered·Decision·Consequences·Implementation Notes·Status History |

`.specify/scripts/bash/common.sh:497-500`이 템플릿 해석 순서를 정의합니다 — ① `.specify/templates/overrides/` ② presets ③ extensions ④ `.specify/templates/`(core). 즉 **`.specify/templates/overrides/spec-template.md`에 수정본을 두면 core 템플릿을 건드리지 않고 덮어쓸 수 있고**, Spec Kit 업그레이드도 안전합니다.

## 선택지

| 안 | 내용 | 장점 | 단점 |
|---|---|---|---|
| **A. Spec Kit 기본 그대로** | `temp-specs/`·`temp-adr/` 폐기 | 워크플로와 100% 호환됩니다. 유지 비용이 없습니다 | Business Rules·Edge Cases·Error Handling 절이 없습니다. VIVAC에서 이 세 절은 "합법성 표기", "데이터 없음", "tier 3 안전 속성" 같은 안전 관련 판단이 들어가는 자리입니다 |
| **B. VIVAC 자체안 유지** | Spec Kit 템플릿을 `temp-specs/` 것으로 교체 | 안전·정책 관련 절이 확보됩니다 | 17절은 기능 하나당 작성 부담이 큽니다. Spec Kit의 User Story 우선순위 개념(P1만 구현해도 동작하는 슬라이스)이 사라집니다 |
| **C. Spec Kit + VIVAC 절 추가** | `.specify/templates/overrides/spec-template.md`에 Business Rules·Edge Cases·Error Handling 3절만 덧붙임 | 워크플로 호환을 유지하면서 안전 절을 확보합니다 | 오버라이드 레이어를 관리해야 하고, Spec Kit 업그레이드 시 병합 확인이 필요합니다 |

ADR은 별도로 A(간결) / B(상세) 중 하나를 골라야 합니다. `front/templates/adr-template.md`는 미러라 여기서 고칠 수 없다는 점을 유의하세요(결정 4 참고).

## 권장 — Spec은 C, ADR은 상세(`temp-adr`) 기반

Spec에 C를 권하는 이유는 Constitution 원칙 I·II 때문입니다. 두 원칙은 "안전 관련 속성은 확인되지 않음으로 표시", "노출 게이트와 신뢰도 표시는 별개 축"을 요구하는데, 이건 Functional Requirement가 아니라 Business Rule과 Edge Case 자리에 적히는 내용입니다. 그 절이 없는 템플릿을 쓰면 원칙 IV의 검증 기준("안전·합법성 동작은 검증 없이 완료로 표시할 수 없다")을 만족시킬 자리가 사라집니다.

ADR에 상세안을 권하는 이유는 Options Considered와 Status History 절 때문입니다. 원칙 V가 "뒤집힌 결정은 원래 결정과 그 이유를 함께 남긴다"고 요구하는데, 간결안에는 그 자리가 없습니다. 실제로 이미 뒤집힌 결정이 2건(Invite 1회용, admin 경로) 있습니다.

**이 결정이 막고 있는 것**: Phase 0의 `temp-*` 폴더 처분, Phase 3의 명세 작성, Phase 4의 ADR 도입.

---

# 결정 3 — `docs/openapi.json`을 계속 추적할 것인가

**선택**: (미정)

## 확인된 사실

- 파일 크기 228KB. git 이력은 커밋 1건뿐이며(`2d4a2c1`), `TEMP.md` 스크래치 메모와 함께 들어왔습니다.
- 현재 작업 트리에 **634줄 추가·23줄 삭제가 미커밋 상태**로 남아 있습니다. 언제 어떤 경위로 갱신됐는지 기록이 없습니다.
- `core/architecture.md` 7번째 줄은 이 파일을 "git 미추적"이라고 서술합니다. 실제와 반대입니다(C2).
- `vivac-cowork`에는 core 코드가 없으므로 이 저장소에서는 `make openapi`를 실행할 수 없습니다. 즉 갱신은 항상 사람이 다른 저장소에서 복사해 와야 합니다.

## 선택지

| 안 | 내용 | 장점 | 단점 |
|---|---|---|---|
| **A. 계속 추적, 규칙 없음** | 현행 유지 | 추가 작업이 없습니다 | 지금과 같은 "언제 갱신됐는지 모르는 634줄"이 반복됩니다. 문서 서술과도 계속 어긋납니다 |
| **B. 추적 중단** | `.gitignore`에 추가하고 파일 삭제 | `architecture.md` 서술과 일치합니다. 저장소가 가벼워집니다 | front·console·mcp 담당자가 API 계약을 볼 수단이 없어집니다. cowork에서는 재생성할 수 없기 때문입니다 |
| **C. 추적하되 갱신 규칙 명시** | 유지하되 "core 릴리스 태그 시점에만 갱신, 커밋 메시지에 core 버전 명시"를 `docs/meta/DOCUMENTATION.md`에 규칙으로 넣고 `architecture.md` 문구 수정 | 공유 수단을 유지하면서 출처와 시점이 명확해집니다. Constitution 원칙 II의 "확인 시점 표기"와 같은 취지입니다 | 규칙을 지켜야 하고, 지키지 않으면 A와 같아집니다 |

## 권장 — C

`vivac-cowork`는 코드 없는 문서 허브이고, 5개 저장소가 공유하는 API 계약을 볼 수 있는 유일한 장소입니다. B는 그 기능을 없앱니다. 다만 A는 이미 실패했다는 증거가 지금 작업 트리에 남아 있으므로, 갱신 시점 규칙이 함께 가야 합니다.

**함께 결정할 것**: 현재 미커밋 634줄을 어떻게 할지입니다. 어느 core 버전에서 나온 것인지 확인되면 그 정보와 함께 커밋하고, 확인이 안 되면 되돌린 뒤 다음 릴리스에 맞춰 새로 생성하는 편이 깨끗합니다.

**이 결정이 막고 있는 것**: Phase 0 기준선 정리, 충돌 C2 해소.

---

# 결정 4 — `docs/front/` 미러의 규칙 충돌(C9)을 어디서 고칠 것인가

**선택**: (미정)

## 확인된 사실

- `docs/INDEX.md`는 `docs/front/`를 "VIVAC-frontend 저장소 자체 `docs/` 구조를 **수정 없이 복사**"한다고 명시합니다. 즉 여기서 고쳐도 다음 미러링에 덮어씁니다.
- `front/INDEX.md` §2의 SoT 우선순위: 1) reference 2) design reference 3) **Source Code(실제 코드가 최종 사실이다)** 4) decisions 5) backlog 6) archive
- Constitution Precedence: 1) Constitution 2) 제품 정의·승인된 Spec 3) 정본 참조 문서 4) CLAUDE.md 5) **코드와 관행**
- 두 규칙은 서로 반대 방향입니다. front 규칙은 "코드가 사실이므로 문서를 고친다", Constitution은 "문서가 의도이고 코드가 현실이므로 **틀린 쪽을** 고친다"입니다.
- 이건 문구 차이가 아니라 실제 판단이 갈리는 지점입니다. 예를 들어 C6(api-proxy 문서가 `route.ts`를 설명하나 실제는 `rewrites()`)에서 front 규칙은 문서만 고치면 끝이지만, Constitution은 "`route.ts`가 미사용 코드로 남아 있는 것이 의도인가"를 먼저 묻습니다.

## 선택지

| 안 | 내용 | 장점 | 단점 |
|---|---|---|---|
| **A. 미러 그대로, 상위 규칙만 명시** | `docs/meta/DOCUMENTATION.md`에 "저장소 간 충돌 시 Constitution이 상위"라고 적고 front repo는 손대지 않음 | front 저장소를 건드리지 않습니다 | front 저장소에서 작업하는 agent는 `front/INDEX.md`만 읽으므로 여전히 반대로 판단합니다. 충돌이 실질적으로 남습니다 |
| **B. front 저장소에서 수정** | `VIVAC-frontend/docs/INDEX.md` §2를 Constitution Precedence에 맞춰 고치고 미러를 갱신 | 충돌이 근본에서 사라집니다. 수정 범위는 문단 하나입니다 | 별도 저장소에 커밋이 필요하고, front 담당자의 동의가 필요합니다 |
| **C. 미러 중단** | front 문서를 cowork 체계로 흡수하고 미러링을 끝냄 | 규칙이 하나가 됩니다 | front 저장소에서 자기 문서를 볼 수 없게 됩니다. 변경 범위가 가장 큽니다 |

## 권장 — B

A는 문제를 문서상으로만 덮습니다. 실제로 판단을 내리는 주체(front 저장소에서 작업하는 사람과 agent)가 읽는 문서가 안 바뀌기 때문입니다. C는 이득에 비해 변경이 큽니다. B는 한 문단 수정으로 끝나고, 같은 작업을 하면서 `front/INDEX.md` §5의 알려진 한계(C6)도 함께 정리할 수 있습니다.

**이 결정이 막고 있는 것**: Phase 1(메타 계층 신설). 규칙 충돌을 남긴 채로 `DOCUMENTATION.md`를 쓰면 그 문서 자체가 모순을 담게 됩니다.

---

# 결정 5 — MVP 범위에 그룹·초대·리퍼럴·리뷰를 포함할 것인가 (C7)

**선택**: (미정)

여섯 건 중 **유일하게 순수한 제품 결정**이며, 나머지 다섯 건과 성격이 다릅니다.

## 확인된 사실

- `PRODUCT.md`의 MVP 정의는 4가지입니다. ① 공공 데이터 통합 ② 노지 스팟 지도 표시 ③ 합법성 검증 로직 ④ UGC 인프라(후순위).
- 그런데 `vivacapi-core`에는 MVP 정의에 없는 **그룹(컬렉션)·초대·리퍼럴·리뷰**가 이미 구현·배포되어 있습니다.
- 화면은 없습니다. `feature-spec.md` §3.1·§3.2가 "API 선구현, 화면 없음"으로 분류했습니다.
- 반대로 MVP 정의의 핵심인 **지도 기반 탐색은 화면·데이터 모두 없습니다**(`feature-spec.md` §4.1, `/map` 404, 위경도 대부분 `null`).
- 가장 큰 차별점인 **합법성 정도 표기도 전혀 없습니다**(§4.2). 선행 조건인 데이터 실현성 스파이크가 미실행입니다.
- `PRODUCT.md`는 이 상황을 스스로 미해결 이슈로 기록해 두었습니다.

## 선택지

| 안 | 내용 | Constitution 원칙 III 통과 여부 |
|---|---|---|
| **A. 4개 모두 MVP에 편입** | 현실을 정의에 맞춤. 화면 4종을 MVP 범위로 승격 | 어렵습니다. 리퍼럴·초대가 "어디에서 합법적이고 안전하게 야영할 수 있는가"에 어떻게 기여하는지 한 문장으로 설명하기 곤란합니다 |
| **B. 4개 모두 로드맵 이후 트랙** | MVP 정의를 지키고, 구현된 API는 화면 없이 대기 | 통과합니다. 다만 배포된 API가 계속 미사용으로 남고, `feature-spec.md`가 "백엔드가 준비돼 ROI가 높다"고 평가한 것과 배치됩니다 |
| **C. 선별 편입** | 그룹만 MVP, 초대·리퍼럴·리뷰는 이후 트랙 | 통과합니다. 그룹은 "찾은 장소를 모아 다시 찾아본다"는 탐색 가치의 연장으로 설명됩니다 |
| **D. 결정 보류, 현행 유지** | 지금처럼 미해결로 둠 | 원칙 III의 "이미 구현된 기능이라는 사실은 범위에 포함될 근거가 되지 않는다"에 걸립니다. 보류 자체가 기본값을 "포함"으로 만들고 있습니다 |

## 권장 — C

세 가지 이유입니다.

1. 그룹은 핵심 질문에 한 문장으로 연결됩니다. "여러 곳을 뒤지지 않게 한다"는 문제 정의의 연장선에서, 찾은 장소를 다시 찾을 수 있게 하는 기능이기 때문입니다. 초대·리퍼럴은 성장 수단이지 사용자 가치가 아니고, 리뷰는 UGC라 `PRODUCT.md`가 이미 4순위로 배치했습니다.
2. `feature-spec.md`가 그룹 화면을 "백엔드 이미 준비됨, 화면만 만들면 됨(ROI 높음)"으로 평가했습니다. 편입 비용이 가장 낮습니다.
3. 무엇보다 **A를 택하면 지도·합법성보다 그룹·리퍼럴이 먼저 완성될 위험**이 생깁니다. `PRODUCT.md`는 "단순 통합은 차별점이 아니며 전환을 만드는 것은 합법성 검증"이라고 스스로 못박았는데, 그 두 축이 아직 0%인 상태입니다.

## 함께 정할 것

이 결정과 짝을 이루는 **데이터 실현성 스파이크 착수 여부**도 같이 답해 주시는 편이 좋습니다. `PRODUCT.md`와 `STATUS.md` §4에 팀 결정 대기로 올라 있고, `feature-spec.md` §4.2는 "스파이크 결과에 따라 화면 기획 자체가 달라진다"며 기획을 의도적으로 비워 두었습니다. 스파이크를 안 하기로 하면 MVP 3순위(합법성 검증)를 명세로 만들 수 없습니다.

**이 결정이 막고 있는 것**: Phase 2 전체, Phase 3의 분해 대상 개수(12개인지 8개인지).

---

# 결정 6 — 분해 후 `feature-spec.md`를 어떻게 남길 것인가

**선택**: (미정)

## 확인된 사실

- `feature-spec.md`는 두 가지 성격이 섞여 있습니다. ① 2026-08-04 실서비스·API·코드를 대조해 확인한 **관측 기록** ② 그 위에 세운 **신규 기획**.
- ①은 다른 어디에도 없는 정보입니다. 실제 화면을 열어 확인한 결과이기 때문입니다.
- ②는 Phase 3에서 개별 명세로 빠져나갑니다.

## 선택지

| 안 | 내용 | 장점 | 단점 |
|---|---|---|---|
| **A. 전량 분해 후 archive** | 기획을 모두 명세로 옮기고 원본은 폐기 처리 | 기능 정의 3중(D4)이 완전히 끊깁니다 | 실서비스 점검이라는 관측 기록이 archive로 들어가 "구현 근거로 쓰지 않는" 문서가 됩니다 |
| **B. 관측 기록만 남김** | §0 현황 요약과 §7 운영 조치만 남겨 "2026-08-04 화면 현황 스냅샷"으로 축소하고, 기획 부분은 명세로 이관 | 관측 가치를 지키면서 중복을 끊습니다 | 문서 성격이 바뀌므로 이름도 바꿔야 합니다(예: `screen-status-260804.md`) |
| **C. 그대로 두고 명세만 신설** | 원본 유지 | 작업이 없습니다 | D4가 그대로 남습니다. 명세와 원본이 갈라지는 순간 어느 쪽이 정본인지 다시 모호해집니다 |

## 권장 — B

C는 이번 마이그레이션이 풀려는 문제를 그대로 남깁니다. A와 B의 차이는 관측 기록의 취급인데, `STATUS.md`가 이미 이 문서의 점검 결과를 여러 곳에서 인용하고 있어 살려 두는 편이 낫습니다.

**이 결정이 막고 있는 것**: Phase 3 마무리.

---

# 부록 — 제가 결정해도 되는 것

아래는 사람의 판단이 필요 없다고 보고 제 재량으로 처리할 항목입니다. 다르게 보신다면 알려 주세요.

| 항목 | 처리 방향 | 근거 |
|---|---|---|
| `docs/TEMP.md`, `docs/temp-product/README.md`, 루트 0바이트 파일 삭제 | 삭제 | 내용이 없고 `INDEX.md`·`STATUS.md`가 이미 정리 대상으로 표시했습니다 |
| `INDEX.md` 상태 열 제거 | 제거 후 `STATUS.md` 위임 | 원칙 V의 중복 정의 금지에 해당합니다 |
| `archive/`의 유효 문서 2건 이동 | `archive/` 밖으로 | 문서 스스로 "폐기 문서 아님"이라고 표시하고 있습니다 |
| `research_backpacking_market.md` 승격 | `docs/research/`로 | 위와 같습니다 |
| 완료된 설계 스냅샷 archive 처리 | `vivac-console-*.md` 등 | 문서에 이미 각주로 대체 관계가 명시돼 있습니다 |
| 깨진 링크 19건 수정 | 수정 | 사실 오류입니다 |
| C1·C2·C6·C8 문구 수정 | 실제 상태에 맞춰 수정 | 문서-코드 불일치이며 의도 판단이 필요 없습니다 |
| `core/troubleshooting/` → `core/decisions/incidents/` | 이름 통일 | 분류 체계 정리입니다 |

단 위 항목도 Constitution Conflict Resolution에 따라 **수정 전에 무엇을 왜 고치는지 먼저 보고**하고 진행합니다.
