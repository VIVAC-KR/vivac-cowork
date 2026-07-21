# 코드베이스 전반 리뷰 — 2026-07-14

- 범위: `apps/web`, `packages/*`, `infra/*`, `Dockerfile`, `.github/workflows`, `docs/*`
- 방법: sub agent 3개 병렬(code-reviewer / security-auditor / architect-reviewer) + 메인 세션 파일 직접 검증
- 계기: NextAuth v5 서버 세션 전환(788291c) 직후 전반 점검
- 코드 수정 없음. 아래는 미처리 이슈 누적 기록.

상태 값: `[열림]` / `[진행중]` / `[완료]` / `[보류]`

---

## 🔴 Tier 1 — 배포 즉시 장애 / 보안

### [열림] 🔴 NextAuth 런타임 secret 컨테이너 주입 누락 (BLOCKER)
- 위치: `infra/docker-compose.yml:7-8`, `.github/workflows/deploy.yml:78`, `Dockerfile:24-31`
- 배경: NextAuth v5는 프로덕션에서 `AUTH_SECRET`(세션 JWE 암호화·CSRF 서명)과 Google provider의 `AUTH_GOOGLE_ID`/`AUTH_GOOGLE_SECRET`을 런타임에 요구한다.
- 현재 상태: compose `environment:`는 `API_PROXY_ORIGIN` 하나만 주입. build-args·Dockerfile 어디에도 `AUTH_*` 없음. `deploy.yml:78`의 `echo "...=..." > .env`는 `>`라 매 배포 `.env`를 한 줄로 truncate → 호스트에 수동 배치해도 compose가 참조 안 하고 덮어씀. auth commit이 main에 있어 **다음 배포에서 `/api/auth/*`가 `MissingSecret`으로 붕괴**.
- 제안: `AUTH_SECRET`/`AUTH_GOOGLE_ID`/`AUTH_GOOGLE_SECRET`을 compose `environment:`(또는 `env_file:`)로 명시 주입하고 SSM/Secrets Manager로 관리. `.env` 생성을 `>>` 또는 관리형 시크릿으로 교체.
- 시점: 다음 배포 전 필수.

### [열림] 🔴 access token 브라우저 노출 (XSS → 토큰 탈취)
- 위치: `apps/web/src/auth.ts:59`, `apps/web/src/features/auth/SessionBridge.tsx:39`, `apps/web/next.config.ts`
- 배경: session 콜백이 `accessToken`을 세션에 실어 `/api/auth/session` JSON으로 브라우저에 노출 → store 미러 → axios. XSS 한 번이면 api.vivac.app bearer 탈취.
- 현재 상태: refresh token은 서버 JWT에만 둠(양호). 그러나 `next.config.ts`에 `headers()` 부재 → CSP/HSTS/X-Frame-Options 전무로 XSS 완화 수단 없음. 영향 증폭.
- 제안: BFF 전환 — proxy route가 `auth()`로 서버측 세션 읽어 `Authorization` 주입, session·store에서 accessToken 제거. `next.config.ts` 보안 헤더 추가(CSP, HSTS, X-Frame-Options: DENY, X-Content-Type-Options: nosniff, COOP).
- 시점: secret 주입 다음 우선순위.

### [열림] 🔴 CloudFront 인증 응답 캐시 포이즈닝 (개연성 높음, repo만으론 미검증)
- 위치: CloudFront distribution 설정(코드 미관리), `apps/web/src/app/api/[...path]/route.ts:45-48`
- 배경: 앞단 CloudFront가 `/api/auth/session`(accessToken 포함)·`/api/v1/*` bearer 응답을 URL 키로 캐시하면 사용자 A 응답을 B에게 서빙 가능.
- 현재 상태: `docs/cloudfront-nextjs-rsc-caching.md`가 CF 공격적 캐싱 incident 1회 기록 → 가설 아님. CF 설정이 repo에 없어 실제 캐시 정책 검증 불가.
- 제안: `/api/*`·`/api/auth/*` behavior를 CachingDisabled + Authorization/Cookie forward-but-no-cache. proxy 응답에 `Cache-Control: no-store` 강제(defense-in-depth). 매 배포 `/*` invalidation은 증상만 덮음.
- 시점: BFF 작업과 함께.

### [열림] 🔴 nginx 공유 소유권 → 배포마다 타 서비스 순단
- 위치: `infra/nginx.conf`, `infra/docker-compose.yml:19-27`, `.github/workflows/deploy.yml:80`
- 배경: `nginx.conf`가 vivac.app + api.vivac.app + console.vivac.app 라우팅. deploy가 `docker compose down`으로 이 nginx까지 내림.
- 현재 상태: frontend 배포 시 API·console도 outage. static upstream `proxy_pass http://api:8000`은 resolver 없어 **기동 시점에** 세 upstream 전부 resolve → 컨테이너 부재 시 nginx 기동 실패(host not found in upstream) → 타 repo 컨테이너 상태가 이 repo 배포 성패를 좌우.
- 근본 원인: repo 위치가 아니라 (a) 앱 배포의 `docker compose down`, (b) static upstream의 부팅 의존. repo를 어디 두든 이 둘 안 고치면 그대로.
- 제안 (lazy 순):
  1. **repo 안 옮기고 순단 제거 (~4줄)** — deploy를 `down` 대신 자기 서비스만: `docker compose up -d --no-deps --force-recreate web`. nginx에 `resolver 127.0.0.11 valid=10s` + 변수 proxy_pass(`set $up_api api:8000; proxy_pass http://$up_api;`)로 lazy resolution → upstream 부재해도 부팅(요청 시 502만). 이것만으로 배포 순단 소멸, 구조 변경 0.
  2. **소유권 정리(필요 시)** — edge nginx는 인프라지 앱 소유 아님. 별도 compose project로 분리(`docker compose -p edge -f edge.yml up -d`), 앱 배포는 자기 서비스만 recreate. **backend repo에 넣지 말 것**(core는 유일 상류인데 하류 도메인을 알게 되는 의존성 역전). 새 git repo도 55줄엔 과함 — 라우트 편집 잦아지면 그때 `vivac-infra` repo.
  3. **장기** — CloudFront가 앞단이므로 subdomain 라우팅을 CloudFront(서브도메인별 distribution→origin)로 올려 공유 nginx 은퇴 가능. 지금은 과함.
- 시점: 1번은 배포 안정화 위해 조기. 2·3은 인프라 정리 시.

---

## 🟠 Tier 2 — correctness 버그 (대부분 잠재: `apiClient` 호출처 0건)

> 참고: `apiClient`(axios)를 실제로 호출하는 코드가 없어 아래 4·6·7은 spots 등 client 데이터 기능 착수 시 발현. BFF 전환하면 이 기계(store+SessionBridge+interceptor+refresh queue)가 통째로 삭제되어 함께 소멸.

### [열림] 🟠 refresh 실패 시 stale token 유지 + 로그아웃 안 됨
- 위치: `apps/web/src/auth.ts:49-54`
- 배경: `serverRefreshToken`이 `null`(만료·폐기) 반환 시 처리.
- 현재 상태: `if (refreshed)` false → 옛 token 그대로 return. `SessionBridge`가 옛 accessToken(non-null) 반환 → interceptor `if(!newToken)` false → `clearAuth` 안 탐 → 죽은 토큰으로 무한 재시도, 로그아웃 불가.
- 제안: `refreshed === null`이면 token 필드 clear(또는 error 플래그) → 클라이언트 signOut 유도.
- 시점: BFF 미채택 시 즉시. 채택 시 무의미.

### [열림] 🟠 middleware 인증 게이트 trap
- 위치: `apps/web/src/middleware.ts:14,37`
- 배경: 게이팅은 `PROTECTED_PREFIXES` 기반, 실행 조건은 `config.matcher`.
- 현재 상태: `PROTECTED_PREFIXES=[]`, matcher `["/login"]` 고정. `PROTECTED_PREFIXES`에 `/mypage` 추가해도 matcher 미등록이면 미들웨어 미실행 → 게이트처럼 보이나 안 걸림. 주석이 이 함정으로 유도.
- 제안: matcher를 `PROTECTED_PREFIXES`에서 파생하거나 broad matcher + 코드 게이팅.
- 시점: 첫 보호 라우트 추가 전.

### [열림] 🟠 env 변수명 불일치 (baseURL 항상 /api fallback)
- 위치: `packages/api/src/client/index.ts:13`, `Dockerfile:24`, `.github/workflows/deploy.yml:37`
- 배경: client는 `NEXT_PUBLIC_API_BASE_URL` 읽음, 빌드는 `NEXT_PUBLIC_API_URL` 주입.
- 현재 상태: 주입값 무시, 항상 `/api` fallback. `NEXT_PUBLIC_*`는 빌드 타임 정적 치환(memory: project-nextjs-webpack-env)이라 런타임 변경 불가. apiClient 미사용이라 현재 무해.
- 제안: 이름 통일 또는 `/api` 의도 확정 후 dead build-arg 삭제.
- 시점: client 데이터 기능 착수 전.

### [열림] 🟠 SessionBridge hydration race
- 위치: `apps/web/src/features/auth/SessionBridge.tsx:19,32`
- 배경: `configureAuth`는 모듈 로드 시, `registeredRefresh`는 useEffect에서 설정.
- 현재 상태: effect 커밋 전 401 → `refresh()` null → spurious signOut. apiClient 미사용이라 현재 미발현.
- 제안: `registeredRefresh`를 모듈 스코프에서 `getSession`/직접 `update`로 설정하거나, bridge 마운트 전까지 interceptor refresh gate.
- 시점: BFF 미채택 시.

---

## 🟡 Tier 3 — 철거 (over-engineering, 소비자 0)

### [열림] 🟡 proxy 이중화 — `route.ts`가 dead code
- 위치: `apps/web/next.config.ts:13-22`, `apps/web/src/app/api/[...path]/route.ts`
- 배경: rewrite와 catch-all Route Handler가 둘 다 `/api/*` 담당.
- 현재 상태: 배열형 `rewrites()` = `afterFiles` = dynamic route보다 먼저 매칭 → rewrite가 이김, `[...path]` catch-all은 실행 안 됨(메인 세션 검증 완료). `docs/api-proxy.md`는 이 dead route를 유일 proxy로 문서화.
- 제안: 하나만 남김. BFF 위해 route.ts 유지(+토큰 주입)하고 rewrite 삭제 권장.
- 시점: BFF 작업과 함께.

### [열림] 🟡 packages 3개가 껍데기 (~130 LOC 실콘텐츠)
- 위치: `packages/shared`, `packages/api`, `packages/store`
- 현재 상태: `shared/utils`·`shared/constants`·`api/queries` barrel이 `export {}`, `mocks/handlers` 빈 배열. `store`는 15줄 zustand에 package 3중 포장(소비자 SessionBridge 1곳). 모바일은 별도 repo(`vivac-mobile-test`)라 `workspace:*` 공유 성립 불가.
- 제안: placeholder barrel 삭제, 실사용 코드는 `apps/web/src/lib`로 인라인. 모바일 공유가 실제 요구되는 시점에 추출.
- 시점: 구조 정리 시.

### [열림] 🟡 MSW 풀스택이 핸들러 0개 유지
- 위치: `packages/api/src/mocks/handlers.ts`, `apps/web/src/app/MswProvider.tsx`, `apps/web/src/mocks/browser.ts`
- 현재 상태: `handlers=[]`인데 msw 의존성·`mockServiceWorker.js`·`MswProvider`(dev 첫 렌더 블로킹) 생존. auth 마이그레이션이 마지막 핸들러 지운 시점에 함께 삭제했어야.
- 제안: 전량 삭제, 필요 시 재도입.
- 시점: 구조 정리 시.

### [열림] 🟡 문서 3개 코드와 모순
- 위치: `docs/api-proxy.md`, `docs/spots-explore-plan.md`, `docs/auth-implementation.md`
- 현재 상태: api-proxy(dead route를 유일 proxy로 기술), spots-explore-plan(삭제된 `Spot` 타입·MSW 목·`features/spots/*` 참조), auth-implementation(구 GIS+localStorage 설계).
- 제안: 폐기 표시 또는 갱신. 철거 작업과 일괄.
- 시점: 구조 정리 시.

### [열림] 🟡 dead code / 죽은 build-arg
- 위치: `apps/web/src/features/auth/LogoutButton.tsx`, `Dockerfile:27-28`
- 현재 상태: `LogoutButton` 미사용(TopBar가 signOut 인라인 중복). `NEXT_PUBLIC_GOOGLE_CLIENT_ID` build-arg는 구 GIS 잔재(NextAuth는 `AUTH_GOOGLE_ID` 사용).
- 제안: 둘 다 삭제.
- 시점: 언제든.

---

## 권장 실행 순서

1. **지금 당장**: Tier 1 #1 secret 주입(안 하면 다음 배포 login 사망), #4 nginx deploy 동사(`--no-deps`)+resolver 수정(순단 제거, ~4줄)
2. **보안**: #2 BFF 전환(access token 브라우저 제거 + 보안 헤더) → #3 CloudFront no-cache까지 커버, #5 refresh-fail 로그아웃
3. **철거**: Tier 3. BFF 채택 시 Tier 2 버그 기계 통째로 소멸

가장 값싼 큰 승리: **BFF 전환** — proxy route 서버측 bearer 주입으로 store+SessionBridge+interceptor+refresh queue 삭제, Tier 2 버그 4개 + Tier 3 기계 동시 제거.

## Agent 리뷰 정정 기록 (재현 시 참고)
- code-reviewer의 `processQueue` "null-stringify"는 실제 버그 아님 — truthy 가드 통과 후에만 호출됨.
- code-reviewer는 rewrite/route 우선순위를 반대로 추정("route가 shadow") — 실제는 rewrite가 이김(afterFiles < dynamic route). architect가 정확.
