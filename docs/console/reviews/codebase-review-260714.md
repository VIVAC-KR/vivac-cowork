# vivac-console 코드베이스 리뷰 — 2026-07-14

- 범위: repo 전체 (src/ 약 2,500 LOC + Dockerfile, infra/docker-compose.yml, .github/workflows/deploy.yml, next.config.ts)
- 방법: sub agent 3개 병렬 (correctness / security / architecture) + 메인 세션이 헤드라인 finding 전부 파일 직접 검증
- 계기: 최근 기능 러시(Browse/My Queue 분리, pipeline_status 검증 화면, edit form 개선) 이후 전반 점검
- 코드 수정 없음. 아래는 미처리 이슈 누적 기록.

상태 값: `[열림]` / `[진행중]` / `[완료]` / `[보류]`

---

## 🔴 Tier 1 — 지금 깨져 있거나 실질 노출

### [완료] 🔴 My Queue가 필터 없이 전체 목록을 보여줌 (`assigned_to_uid` drop)
- 위치: `src/app/(admin)/spots/page.tsx:61` (링크 생성: `dashboard/page.tsx:97,115`, `admin-shell.tsx:41`)
- 배경: d1011bb에서 Browse/My Queue 분리하며 링크 3곳에 `?assigned_to_uid=`를 붙였지만, spots 목록 페이지는 `FACETS` 3개(pipeline_status, region_province, source) + q/sort/page만 API로 전달한다.
- 현재 상태: My Queue 링크·사이드바·대시보드 카드 클릭 시 **필터 없는 전체 목록**이 뜬다. 최근 커밋의 핵심 기능이 조용히 동작하지 않는 상태. correctness/architecture 두 agent가 독립적으로 발견, 메인 세션 확인 완료.
- 제안: `apiList` params에 `assigned_to_uid: sp.assigned_to_uid` pass-through 추가 + `buildQuery` 보존 목록에도 포함. (backend `_FILTERABLE` whitelist에 있는지 함께 확인)
- 시점: 즉시 — 팀원이 My Queue를 믿고 작업하면 남의 항목을 건드림.

### [완료] 🔴 backend admin JWT가 `/api/auth/session` 응답으로 브라우저에 노출
- 위치: `src/auth.config.ts:33` (`session.accessToken = token.accessToken`)
- 배경: server component/action에서 `auth()`로 토큰을 읽기 위해 session에 넣었지만, NextAuth session callback 산출물은 `/api/auth/session` JSON으로 클라이언트에도 그대로 내려간다. 클라이언트에서 `accessToken`을 쓰는 곳은 전무(grep 확인 — `useSession` 사용 0건).
- 현재 상태: httpOnly cookie 보호가 무의미. XSS·악성 extension 하나면 8h 유효 admin token 탈취. 단독으로는 악용 불가하지만 CSP 부재(아래 항목)와 결합 시 파급 큼.
- 제안: session callback에서 `accessToken` 제거, 서버에서는 JWT cookie를 decode하는 server-only helper(또는 NextAuth v5 `unstable_update`/jwt 직접 접근)로 읽기.
- 시점: 이번 주 내. console이 `https://console.vivac.app`으로 internet-reachable이므로 실질 노출.

### [완료] 🔴 `OpenLink`가 `javascript:` scheme URL을 그대로 렌더
- 위치: `src/components/admin/spot-edit-form.tsx:371-388`
- 배경: c85e45d에서 추가한 웹사이트 바로가기. `new URL(url)` 파싱 성공 여부만 검사하는데, `new URL("javascript:alert(1)")`도 성공한다. `website_url`은 ETL이 외부에서 수집한 값 — 신뢰 불가 입력.
- 현재 상태: 오염된 `website_url`이 들어오면 admin 세션에서 클릭 한 번으로 실행되는 stored XSS 벡터. 위 accessToken 노출과 결합하면 token 탈취 체인 완성.
- 제안: `const u = new URL(url); if (u.protocol !== "http:" && u.protocol !== "https:") return null;`
- 시점: 즉시 — 한 줄 수정.

## 🟠 Tier 2 — Correctness / 운영 안정성

### [완료] 🟠 API 오류를 전부 404로 위장
- 위치: `src/app/(admin)/spots/[uid]/edit/page.tsx:88-92`, `spot-business-info/[uid]/edit/page.tsx:59-63`
- 현재 상태: `apiFetch` 실패(500, 네트워크 단절, 401 포함)를 모두 `notFound()` 처리 — backend 장애가 "존재하지 않는 페이지"로 은폐되어 디버깅을 방해.
- 처리: `apiFetch`/`apiList`가 status를 담은 `ApiError`를 던지도록 변경, 두 edit page 모두 `err instanceof ApiError && err.status === 404`일 때만 `notFound()`, 그 외 rethrow.

### [완료] 🟠 server action 수동 fetch 중복 → 이미 drift 발생
- 위치: `src/app/(admin)/spots/[uid]/edit/page.tsx:48-78` (`saveSpot`), `spot-business-info/[uid]/edit/page.tsx:27-49` (`saveBusinessInfo`)
- 배경: 두 action이 `src/lib/api.ts`를 우회하고 auth header + fetch + 에러 처리를 각자 복사. `saveSpot`은 dc2952c에서 `{"detail":...}` 파싱을 얻었지만 `saveBusinessInfo`는 못 받음 — 중복이 만든 drift의 실증.
- 처리: `api.ts`에 `apiMutate(path, data)` 추가 (404/422 detail 배열 파싱 포함, `d.msg`/`d.loc` join). 두 server action은 `return apiMutate(...)` 한 줄로 축소.

### [완료] 🟠 제출/반려 버튼의 `pendingStatusRef`가 validation 실패 시 리셋 안 됨
- 위치: `src/components/admin/spot-edit-form.tsx:109,153,319-339`
- 현재 상태: "제출" 클릭 → phone validation 실패 → ref에 `"CURATED"` 잔존 → 이후 의도치 않은 status로 전송될 수 있었음.
- 처리: `handleSubmit(onSubmit, () => { pendingStatusRef.current = null })`로 invalid 콜백에서 리셋. (`sbi-edit-form.tsx`는 pendingStatusRef 자체가 없어 해당 없음 확인)

### [완료] 🟠 deploy heredoc unquoted + `.env` 권한
- 위치: `.github/workflows/deploy.yml:66-73`
- 현재 상태: `cat > .env <<EOF`가 unquoted라 secret 안의 `$`·backtick이 EC2 shell에서 확장됨. `.env`가 기본 umask로 다른 사용자도 읽기 가능.
- 처리: `<<'EOF'`로 quoting + `chmod 600 .env` 추가. (`${{ secrets.* }}`는 GHA가 워크플로 파싱 단계에서 치환하므로 heredoc quoting과 무관하게 정상 동작 확인)

### [완료] 🟠 Dockerfile base image `node:20-alpine` EOL
- 위치: `Dockerfile:1,18`
- 처리: builder/runner 두 스테이지 모두 `node:22-alpine`으로 bump.

### [완료] 🟠 security header 전무
- 위치: `next.config.ts:3`
- 처리: `headers()`에 `X-Frame-Options: DENY`, `Content-Security-Policy: frame-ancestors 'none'`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin` 추가. script-src 등 전체 CSP는 Google 로그인·카카오맵 링크·OpenStreetMap iframe·Pretendard CDN에 영향 줄 수 있어 브라우저 실측 없이 넣지 않음 — 필요 시 별도 검증 후 추가.

### [완료] 🟠 빈 문자열 facet 파라미터가 backend로 전달
- 위치: `src/app/(admin)/spots/page.tsx:114-116` (hidden input), `src/lib/api.ts:35-37`
- 처리: `apiList`에서 `v !== undefined && v !== ""`로 skip. (backend `== ""` 필터 해석 여부는 여전히 vivacapi-core 미확인이나, 프론트에서 빈 값을 안 보내는 게 항상 안전한 방향이라 즉시 적용)

### [열림] 🟠 대시보드 "Pending Review (Mine)" 숫자와 링크 결과 불일치 가능
- 위치: `src/app/(admin)/dashboard/page.tsx:35,108-118`
- 현재 상태: 카드 숫자는 `my_assigned_total - my_completed`(전 상태 포함), 링크는 `pipeline_status=ENRICHED`만 필터 — `my_completed`의 backend 정의에 따라 숫자와 목록 건수가 어긋날 수 있음. **미확정** — vivacapi-core의 stats 정의 확인 필요.
- 제안: stats API에 ENRICHED+assigned 카운트 추가하거나 링크/수치 정의 일치.
- 시점: vivacapi-core의 `my_completed` 정의를 먼저 확인해야 함 — repo 간 계약 이슈라 이번 배치에서 보류.

### [완료] 🟠 `signIn` callback fetch에 try/catch 없음
- 위치: `src/auth.ts:55-61`
- 현재 상태: backend 다운 시 fetch throw → 사용자에게 의미 없는 "Configuration" 에러.
- 처리: fetch를 try/catch로 감싸 실패 시 `return false`.

## 🟡 Tier 3 — 정리 / 철거 / hardening

### [완료] 🟡 중복 type 정의 — `SpotDetail` × 2, `BusinessInfoDetail` × 2
- 위치: `spots/[uid]/edit/page.tsx:8-45` vs `spot-edit-form.tsx:14-45` (sbi 동일 패턴)
- 처리: `src/lib/types.ts` 신설, `SpotDetail`/`BusinessInfoDetail` 각 한 벌로 통합. 4개 파일(두 edit page + 두 form)이 `import type`으로 참조. (FastAPI openapi.json 생성은 장기 과제로 보류)

### [완료] 🟡 `NEXT_PUBLIC_API_BASE_URL` — server 전용인데 public prefix
- 위치: `src/lib/api.ts:3`, `src/auth.ts:25`, `deploy.yml:72`, `.env.example`
- 처리: `API_BASE_URL`로 rename, `api.ts`는 미설정 시 모듈 로드 시점에 throw(fail-fast)로 변경. `.env.local`(gitignore, 로컬 전용)도 함께 갱신 안 하면 로컬 build가 즉시 깨지는 걸 build 검증 중 실제로 확인·수정.

### [완료] 🟡 배포가 `:latest` 고정 — SHA pin/rollback 불가 + down→up downtime
- 위치: `infra/docker-compose.yml:3`, `deploy.yml:74-78`
- 처리: compose image를 `${DOCKER_IMAGE}` 변수로, deploy 스크립트가 `export DOCKER_IMAGE=<repo>:${{ github.sha }}`로 pin. `docker compose down` 제거하고 `pull` + `up -d`만 (recreate로 충분).

### [완료] 🟡 dead config `ALLOWED_EMAIL_DOMAIN`
- 위치: `.env.example:13`
- 처리: 삭제. (Google provider `hd` param 추가는 별도 defense-in-depth 판단이 필요해 보류)

### [완료] 🟡 문서 drift — README `/v1/admin/*` vs 실제 `/v1/internal/*`
- 위치: `README.md:12,18`
- 처리: 실제 경로(`/v1/internal/*`, 로그인은 `/admin/auth/google`)로 정정. 상위 workspace `vivac/CLAUDE.md`는 이 repo git 범위 밖(별도 관리 파일)이라 손대지 않음 — 별도로 정정 필요.

### [완료] 🟡 자잘한 correctness 묶음
- `spots/page.tsx` + `spot-business-info/page.tsx` — `Number(page)` → `parseInt(page, 10) || 1`로 NaN 방지 (양쪽 페이지 모두 적용).
- `admin-shell.tsx:102` — `pathname.startsWith(item.href.split("?")[0])`로 querystring 제외하고 비교, My Queue nav active 표시 정상화.
- `spots/[uid]/edit/page.tsx` — biList/history를 `Promise.all`로 병렬화 + `biList[0]?.uid` 가드.
- `spot-edit-form.tsx` — phone validate를 `v.trim()` 후 `/^\d[\d-]*\d$/`로 검사, 저장 값도 trim.
- `api.ts:40` — Tier 2에서 `ApiError`로 이미 body 포함 처리됨 (추가 조치 불요, 확인만).
- `spots/[uid]/edit/page.tsx`, `spot-business-info/[uid]/edit/page.tsx` — uid 전체 사용처(`apiFetch`/`apiMutate`/history) `encodeURIComponent` 적용.

### [진행중] 🟡 hardening / 인지 항목 (당장 조치 불요로 표시됐던 것 중 일부만 처리)
- [완료] `deploy.yml` — `permissions: contents: read` + `concurrency: prod-deploy` 추가.
- [완료] `package.json` — `shadcn`을 `dependencies`→`devDependencies`로 이동 (런타임에 안 쓰는 CLI), `pnpm-lock.yaml` 갱신.
- [열림] `appleboy/*` action mutable tag pin — 정확한 commit SHA 확인이 필요해 이번 배치에서 보류 (잘못 pin하면 배포가 깨짐).
- [완료] `src/proxy.ts:25` matcher (svg/png만 제외) — `refactor-260802.md` 🔴 항목에서 실제 우회 경로(`/spot-groups/x.png/edit`)가 생겨 exploit으로 격상, 확장자 매칭을 경로 끝(`$`)으로 한정해 수정.
- [열림] `src/proxy.ts:20` callbackUrl 없음 — UX 개선, 로그인 페이지 쪽 처리도 같이 필요해 별도 작업으로 분리.
- [열림] `package.json:18` next-auth beta 의존 — stable 릴리스 대기.
- [열림] `Makefile` — 결함 아님, 취향 문제라 유지. CI lint/typecheck gate 부재가 진짜 공백이나 새 워크플로 추가는 별도 결정 필요.
- [열림] `auth.config.ts:15` 8h 만료 UX — 별도 설계 필요.
- [열림] `layout.tsx:30-33` Pretendard self-host — 폰트 파일 준비 등 별도 작업.
- [열림] `spot-edit-form.tsx:151-185` PATCH 전체 필드 전송 — 의도된 트레이드오프로 확인, 코드 변경 불요.

## 권장 실행 순서
1. **Tier 1 세 건 한 번에**: `assigned_to_uid` pass-through + `OpenLink` protocol 검사 + session에서 `accessToken` 제거 — 앞 둘은 각 몇 줄, 셋째는 반나절.
2. **에러 처리 통합 PR**: `apiMutate` helper 신설(404/422 파싱 포함) → `saveSpot`/`saveBusinessInfo` 복사본 삭제 → `notFound()` 오용 수정 → `apiList` body 포함. Tier 2의 4건이 한 PR로 해소.
3. **배포 위생 PR**: node:22 bump + heredoc quoting + chmod 600 + security headers + `permissions:` — 전부 설정 수준.
4. Tier 3은 손 닿을 때: 중복 type 통합과 `NEXT_PUBLIC_` rename부터.

## Agent 리뷰 정정 기록 (재현 시 참고)
- architecture agent의 "compose image가 `vars.DOCKER_IMAGE`와 달라 stale image 배포" — 과장. var가 같은 값이면 정상 동작하고, 실제 문제는 `:latest` 고정으로 SHA pin/rollback이 불가능한 것. 그렇게 재분류.
- architecture agent가 대시보드 링크(`dashboard/page.tsx:97,115`)를 assigned_to_uid drop의 발생지로 인용 — 정확했고 메인 세션이 3개 링크 사이트 모두 grep으로 확인. correctness agent와 독립 일치.
- security agent의 "NEXT_PUBLIC 값이 현재 bundle에 유출 안 됨" — 메인 세션 확인 결과 맞음 (Docker build 시 env 부재로 undefined inline). "우연히 안전" 상태라는 판정 유지.
- correctness agent의 대시보드 myPending 불일치 — backend `my_completed` 정의를 이 repo에서 확인 불가라 **미확정(PLAUSIBLE)**으로 강등. vivacapi-core 확인 필요.
- 세 agent 간 모순 없음. critical/high 전 항목 메인 세션이 원본 파일로 재확인.
