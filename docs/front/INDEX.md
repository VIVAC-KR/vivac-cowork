# front 문서 구조 안내

`VIVAC-frontend` 저장소가 참조하는 문서 폴더입니다. 실체는 `vivac-cowork/docs/front/`에 있고, 그 저장소에서는 `docs/front/`로 보입니다.

> **문서 작성 규칙과 충돌 판정은 [docs/meta/DOCUMENTATION.md](../meta/DOCUMENTATION.md)를 따릅니다.** 폴더 분류·파일명·톤은 §1, 문서 간 규칙은 §6, 문서와 코드가 다를 때의 판정은 §8입니다. 이 문서는 front 폴더의 현재 구성과 먼저 읽을 문서만 안내합니다.

## 1. 폴더 구성

| 폴더 | 담긴 것 |
|---|---|
| 루트 | 안정적 레퍼런스 — 프론트엔드 아키텍처·API 연동(`api-proxy.md`), 배포·인프라 구성(`docker-deployment.md`) |
| `incidents/` | 장애 원인·대응 기록 (`cloudfront-nextjs-rsc-caching.md`) |
| `projects/` | 여러 결정이 묶인 기능·API 설계 문서 |
| `backlog/` | 미해결 문제 — 개선 아이디어, 기술 부채, 코드리뷰 결과 |
| `reviews/` | 날짜별 코드리뷰·문서감사 스냅샷 |
| `archive/` | 폐기 문서. 대체 문서·폐기일·사유를 상단에 명시합니다 |

폴더 분류는 다른 저장소 폴더와 같은 표준을 씁니다. 기준은 [DOCUMENTATION.md](../meta/DOCUMENTATION.md) §1.5입니다.

디자인 문서는 [`docs/design/`](../design/INDEX.md)에 별도로 있습니다.

## 2. 먼저 읽어야 하는 문서

- [`api-proxy.md`](api-proxy.md) — Next.js API 프록시 구조와 환경변수
- [`docker-deployment.md`](docker-deployment.md) — Docker 빌드·배포 구성
- [`incidents/cloudfront-nextjs-rsc-caching.md`](incidents/cloudfront-nextjs-rsc-caching.md) — CloudFront 캐싱 장애 사례. 인프라 변경 전 필독입니다
- [`backlog/codebase-review-260714.md`](backlog/codebase-review-260714.md) — 알려진 미해결 이슈 전체 목록(배포·보안 우선순위 포함)
- `VIVAC-frontend` 루트의 `DESIGN.md` — 디자인 토큰 정본. `npx getdesign`으로 재생성되므로 `docs/design/`으로 옮기지 않았습니다

## 3. 알려진 한계 (2026-07-15 재구성 시점 기준)

- [`api-proxy.md`](api-proxy.md)는 `apps/web/src/app/api/[...path]/route.ts`를 유일한 프록시로 설명하지만, 실제로는 `next.config.ts`의 `rewrites()`가 우선 적용되고 해당 `route.ts`는 미사용 코드입니다([`backlog/codebase-review-260714.md`](backlog/codebase-review-260714.md) Tier 3 참고). 프록시 흐름·환경변수 설명 자체는 유효하나 이 부분은 갱신이 필요합니다.
- NextAuth v5 전환(commit `788291c`) 이후의 인증 아키텍처를 다루는 레퍼런스 문서가 아직 없습니다. [`archive/auth-implementation.md`](archive/auth-implementation.md)는 전환 이전 구현을 설명하므로 archive 처리했습니다.
