---
paths:
  - "docs/**/*.md"
---

# vivac 문서 작성 규칙

`docs/` 하위에 새 문서를 쓰거나 기존 문서를 고칠 때 이 규칙을 따릅니다. 사용자가 "문서 작성해줘"라고만 요청해도, 아래 기준으로 알맞은 카테고리 폴더에 정형 포맷으로 작성합니다.

상세 규칙은 [`docs/meta/DOCUMENTATION.md`](../../docs/meta/DOCUMENTATION.md)를 따릅니다. 이 문서는 그중 자주 쓰는 부분을 간추린 것입니다.

## 1. 어느 폴더에 쓸지

| 문서 성격 | 위치 |
|---|---|
| 미착수 · 우선순위 대기 항목 | `docs/<repo>/backlog/` |
| 날짜별 코드리뷰 / 문서감사 스냅샷 | `docs/<repo>/reviews/` |
| 장애 기록 | `docs/<repo>/incidents/` |
| 여러 결정이 묶인 기능·API 설계 문서 | `docs/<repo>/projects/` |
| 제품 정의·범위·정보 구조·로드맵 | `docs/product/` |
| 화면별 기능 명세 | `docs/features/` |
| 여러 repo에 걸친 시스템 구조·계약 | `docs/architecture/` |
| 사용자·시장 리서치 | `docs/research/` |
| 그 외 안정적 레퍼런스(아키텍처, ERD, 필드 매핑 등) | `docs/<repo>/` 루트 |

결정 기록(ADR)은 `docs/` 안에 쓰지 않습니다. 저장소를 넘는 결정은 `vivac-cowork/adr/`, 그 저장소 안에서 닫히는 결정은 **해당 저장소의 `adr/`**(심볼릭 링크 대상이 아니라 그 저장소 git)로 갑니다. 판정표는 [`docs/meta/SSOT.md`](../../docs/meta/SSOT.md) §1.3, 명명·머리말 규칙은 [`DOCUMENTATION.md`](../../docs/meta/DOCUMENTATION.md) §5입니다. `docs/<repo>/decisions/`는 폐지 중인 폴더이므로 새로 만들지 않습니다.

`<repo>`는 지금 작업 중인 repo 약칭: `front`(VIVAC-frontend) / `console`(vivac-console) / `mcp`(vivac-mcp) / `core`(vivacapi-core) / `etl`(vivacapi-etl). 어느 카테고리에도 뚜렷이 안 맞으면 사용자에게 먼저 확인합니다.

## 2. 파일명

- kebab-case, 확장자 `.md`
- 날짜가 의미 있는 문서(리뷰, backlog 개별 항목)는 끝에 `-YYMMDD`를 붙입니다. 예: `codebase-review-260714.md`
- 새 폴더(`reviews/`, `incidents/` 등)는 그 유형 문서가 실제로 생길 때만 만듭니다. 빈 폴더를 미리 만들지 않습니다.

## 3. 톤/말투

- 본문 문장은 "~입니다/~합니다" 서비스 말투
- 기술 용어(함수명, 엔드포인트, 필드명 등)는 영문 그대로, 코드 인용은 백틱으로 표기
- 서비스명은 항상 `vivac`/`VIVAC`, 한글 표기(비바크, 비박 등) 금지

## 4. 카테고리별 템플릿

화면별 기능 명세·ADR·인시던트는 [`docs/meta/templates/`](../../docs/meta/templates/)의 템플릿(`feature-template.md` · `adr-template.md` · `incident-template.md`)을 씁니다. 아래는 저장소 계층 폴더용 포맷입니다.

### 변경 이력 — 규칙이 누적되는 영역

데이터 변환 규칙처럼 결정이 계속 쌓이는 영역은 ADR 대신 버전별 변경 이력 포맷을 씁니다.

```markdown
# <대상> 변경 이력

## v1.0 — YYYY-MM-DD (<한 줄 요약>)

### <변경 항목>
- 변경 전:
- 변경 후:
- 이유:
```

### backlog/ — 미착수 항목

```markdown
# <문제 한 줄 요약>

- **심각도**: 낮음/중간/높음 (<보안/성능/운영 등 분류>)
- **출처**: <점검 날짜 및 계기>

## 문제

## 수정 방향

## 영향
```

### reviews/ — 코드리뷰·문서감사 스냅샷

```markdown
# <대상> 리뷰 — YYYY-MM-DD

- 범위:
- 방법:
- 계기:

상태 표기: `[열림]` / `[진행중]` / `[완료]` / `[보류]`

## 🔴 Tier 1 — <심각도 그룹>

### [상태] 🔴 <제목>
- 위치:
- 현재 상태:
- 제안:
```

### projects/ — 여러 결정이 묶인 기능·API 설계

```markdown
# <기능/API 이름>

> 작성일: YYYY-MM-DD
> 배경:

## 1. 한 줄 요약

## 2. 결정 사항 요약

| 항목 | 결정 | 근거 |
|---|---|---|

## 3. 엔드포인트/스키마 (해당 시)

## N. Out of Scope
```

## 5. 원칙

- 새 문서가 기존 문서와 **정의**가 겹치면 새로 쓰지 않고 기존 문서를 갱신하거나 링크로 참조합니다. 같은 사실을 두 곳에서 정의하지 않습니다.
- **한정된 예외는 `docs/features/`입니다.** 화면별 기능 명세는 구현자가 다른 문서를 열지 않고 한 화면을 만들 수 있어야 하므로 자족적으로 씁니다. 정본이 정의한 값을 옮겨 적을 수 있으며, 이때 (1) 값마다 정본 문서와 절을 표시하고 (2) 정본과 동일한 값을 적고 (3) 정본을 고칠 때 함께 갱신합니다(Constitution 원칙 V 한정 규정). 정본 표시가 없는 값은 새 정의가 되므로 예외에 해당하지 않습니다. 여기서 "중복이니 링크로 대체하자"고 판단하지 않습니다.
- 결정이 나중에 뒤집히면 원본 문서를 지우지 않고 각주/경고로 반영 사실을 남깁니다 (예: [projects/spot-invites.md](../../docs/core/projects/spot-invites.md) 상단 각주 참고).
- 다른 repo 전용 폴더(`docs/<다른 repo>/`) 문서는 명시적으로 요청받았을 때만 참고하거나 수정합니다.
