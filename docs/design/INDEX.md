# docs/design/ 문서 구조 안내

`docs/`와 동일한 원칙(Reference/Decisions/Backlog/Archive 분리)을 디자인 문서에 적용한다. 전체 우선순위·작성 원칙은 [`docs/INDEX.md`](../INDEX.md)를 함께 참고.

## 1. 문서 구조

| 폴더 | 역할 |
|---|---|
| `docs/design/reference/` | 현재 확정된 디자인 스펙만 저장. "왜"는 쓰지 않는다 — 현재 디자인 기준만 유지한다. |
| ├─ `reference/pages/` | 페이지 단위 UI 스펙 (예: `spot-detail.md`) |
| ├─ `reference/components/` | 재사용 컴포넌트 명세 (현재 비어 있음) |
| ├─ `reference/patterns/` | UX 패턴 (현재 비어 있음) |
| └─ `reference/tokens/` | 디자인 토큰 (현재 비어 있음 — 실제 토큰 정본은 vivac-frontend 루트의 `DESIGN.md`와 `apps/web/src/app/globals.css`에 배선되어 있다) |
| `docs/design/decisions/` | 디자인 의사결정 기록. 디자인 변경 이유, UX 리뷰, 사용자 테스트 결과, 레이아웃 변경 배경, 버전 히스토리 (예: `spot-detail-design-decisions.md`). |
| `docs/design/design-ref/` | 참고 자료. 레퍼런스 이미지, 경쟁 서비스, 무드보드 (예: `design-ref/spot-carousel/`). 구현 기준이 아니다. |
| `docs/design/archive/` | 폐기된 디자인. 삭제하지 않고 보관한다 (현재 비어 있음). |

디자인 관련 미해결 후속 논의·검증 필요 항목은 design 폴더가 아니라 `docs/backlog/`에 둔다 (예: `docs/backlog/spot-detail-design-followups.md`) — 프로젝트 전체 backlog를 한곳에서 추적하기 위함이다.

## 2. Source of Truth 우선순위

1. `docs/reference/`
2. `docs/design/reference/`
3. Source Code
4. `docs/decisions/` (`docs/design/decisions/` 포함)
5. `docs/backlog/`
6. `docs/design/archive/` / `docs/archive/`

## 3. 반드시 먼저 읽어야 하는 문서

- vivac-frontend 루트의 `DESIGN.md` — 디자인 토큰·컴포넌트 원시값의 단일 진실 공급원. `npx getdesign`으로 재생성되므로 이 문서를 직접 수정하지 않는다(그래서 `docs/`로 옮기지 않고 vivac-frontend에 남겨뒀다). 편차(원칙을 의도적으로 어긴 부분)는 `apps/web/src/app/globals.css` 헤더 주석이 정본이다.
- [`docs/design/reference/pages/spot-detail.md`](reference/pages/spot-detail.md) — 야영장 상세페이지 UI 스펙 (v2.2, DESIGN.md 우선 정책 적용)
- [`docs/design/decisions/spot-detail-design-decisions.md`](decisions/spot-detail-design-decisions.md) — 위 스펙의 의사결정 경위·버전 히스토리

## 4. 문서 작성 원칙

- Reference는 "왜"를 쓰지 않는다. 현재 디자인 기준만 유지한다.
- Decisions에는 디자인 변경 이유, 검토했던 대안, 트레이드오프를 적는다.
- 아직 검증되지 않았거나 후속 논의가 필요한 항목은 `docs/backlog/`로 보낸다.
- Archive는 구현 기준이 아니며, 반드시 대체 문서를 명시한다.

## 5. 알려진 한계 (2026-07-15 재구성 시점 기준)

- `docs/design/reference/pages/spot-detail.md`가 "선행 문서(기획)"로 인용하는 `spot-detail-screen-design-260715.md`는 vivac-cowork·vivac-frontend 어느 저장소에도 존재하지 않는다. 별도 보관 중이거나 아직 작성되지 않은 것으로 보인다 — 확인 필요.
- 같은 문서가 `docs/spots-explore-plan.md`(현재 `docs/archive/spots-explore-plan.md`)를 "형제 문서"이자 활성 참고 자료로 인용하지만, 해당 문서가 설명하는 `apps/web/src/features/spots/*` 구현은 현재 코드베이스에 존재하지 않아 archive 처리했다. `/spots` 리스트·지도 탐색 페이지를 재구현할 때 이 참조를 최신 상태로 다시 확인해야 한다.
