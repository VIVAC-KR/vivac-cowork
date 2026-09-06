# docs/ 문서 구조 안내

이 저장소의 문서는 AI·사람 협업자가 "지금 무엇이 사실인지"와 "왜 그렇게 됐는지"를 빠르게 구분할 수 있도록 역할별로 분리되어 있다.

## 1. 문서 구조

| 폴더 | 역할 |
|---|---|
| `docs/reference/` | 현재 유효한 사실(Source of Truth)만 보관. 제품 정책, 기능 명세, API 명세, 데이터 구조, 시스템 아키텍처, 개발 규칙, 배포 방법. 항상 코드와 일치해야 하며 회의 내용·배경 설명은 포함하지 않는다. |
| ├─ `reference/product/` | 제품 정책·기능 명세 (현재 비어 있음 — 문서 생기면 추가) |
| ├─ `reference/frontend/` | 프론트엔드 아키텍처·API 연동 레퍼런스 (예: `api-proxy.md`) |
| ├─ `reference/backend/` | 백엔드 레퍼런스 (현재 비어 있음) |
| ├─ `reference/architecture/` | 시스템 전반 아키텍처 (현재 비어 있음) |
| └─ `reference/infra/` | 배포·인프라 가이드 (예: `docker-deployment.md`) |
| `docs/decisions/` | 의사결정 기록. 기술 선택 이유, UX 결정 이유, 검토했던 대안, 트레이드오프. 현재 스펙이 아니라 "왜"를 기록한다. |
| └─ `decisions/incidents/` | 장애 원인 및 대응 기록 (예: `cloudfront-nextjs-rsc-caching.md`) |
| `docs/projects/` | 여러 결정이 묶인 기능·API 설계 문서. 기능 골격·향후 작업 로드맵을 담고, 개별 결정이 확정되면 Reference 또는 Decisions로 승격한다. (예: `search.md`) |
| `docs/backlog/` | 아직 해결되지 않은 문제. 개선 아이디어, 기술 부채, 코드 리뷰 결과, 추후 검토 사항. 해결되면 Reference 또는 Decisions로 이동한다. |
| `docs/archive/` | 폐기된 문서. 삭제하지 않고 보관하며, 상단에 Archive 헤더(대체 문서·폐기일·사유)를 명시한다. 구현 근거로 사용하지 않는다. |
| `docs/templates/` | 문서 작성 템플릿 (ADR, Reference, Incident). |

디자인 문서 구조는 [`design/INDEX.md`](../design/INDEX.md) 참고.

## 2. Source of Truth 우선순위

문서 간 충돌이 발생하면 항상 아래 우선순위를 따른다.

1. `docs/reference/`
2. `docs/design/reference/`
3. Source Code (실제 코드가 최종 사실이다)
4. `docs/decisions/`
5. `docs/backlog/`
6. `docs/archive/` — 참고용일 뿐, 구현 근거로 쓰지 않는다

## 3. 반드시 먼저 읽어야 하는 문서

- [`docs/reference/frontend/api-proxy.md`](reference/frontend/api-proxy.md) — Next.js API 프록시 구조와 환경변수
- [`docs/reference/infra/docker-deployment.md`](reference/infra/docker-deployment.md) — Docker 빌드·배포 구성
- [`docs/decisions/incidents/cloudfront-nextjs-rsc-caching.md`](decisions/incidents/cloudfront-nextjs-rsc-caching.md) — CloudFront 캐싱 장애 사례 (재발 방지용, 인프라 변경 전 필독)
- [`docs/backlog/codebase-review-260714.md`](backlog/codebase-review-260714.md) — 현재 알려진 미해결 이슈 전체 목록 (배포/보안 우선순위 포함)
- vivac-frontend 루트의 `DESIGN.md` — 디자인 시스템 단일 진실 공급원 (이 문서는 `npx getdesign`으로 재생성되므로 `docs/design/`으로 옮기지 않았다)

## 4. 문서 작성 원칙

- Reference는 현재 사실만 작성한다. 회의 내용, 배경 설명, 트레이드오프 논의는 넣지 않는다.
- Decisions는 이유만 작성한다. 현재 스펙을 다시 적지 않는다.
- Backlog는 미해결 문제만 작성한다. 해결되면 Reference 또는 Decisions로 옮긴다.
- Archive는 구현에 사용하지 않는다. 반드시 대체 문서를 명시한다 (없으면 "없음"과 사유를 적는다).
- 새 문서를 작성할 때는 `docs/templates/`의 템플릿을 기준으로 시작한다.

## 5. 알려진 한계 (2026-07-15 재구성 시점 기준)

- `docs/reference/frontend/api-proxy.md`는 `apps/web/src/app/api/[...path]/route.ts`를 유일한 프록시로 설명하지만, 실제로는 `next.config.ts`의 `rewrites()`가 우선 적용되고 해당 route.ts는 현재 미사용 코드다(`docs/backlog/codebase-review-260714.md` Tier 3 참고). 프록시 흐름·환경변수 설명 자체는 유효하나 이 부분은 갱신이 필요하다.
- NextAuth v5 전환(commit 788291c) 이후의 인증 아키텍처를 다루는 Reference 문서가 아직 없다. `docs/archive/auth-implementation.md`는 전환 이전 구현을 설명하므로 archive 처리했다.
