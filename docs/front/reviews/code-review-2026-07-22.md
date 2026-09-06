# VIVAC-frontend 코드 리뷰 (2026-07-22)

## 요약

Next.js 15(App Router) + React 19 + NextAuth v5 + TanStack Query 5 + Zustand 조합의 turborepo(pnpm) 모노레포. `apps/web/src` 21개 파일 1,171줄, `packages/*` 13개 파일 217줄 — 실질적으로는 홈 정적 화면과 Google 로그인(NextAuth v5 서버 세션)만 구현된 초기 단계이며, `apps/web/src/features/spots/*` 등 실 데이터 기능은 아직 `main`에 없다(별도 브랜치 `feat/spot-detail-ui`에 진행 중, 이번 리뷰 범위인 `main` 워킹트리에는 미포함).

타입 안전성(`strictTypeChecked`, `no-explicit-any: error` 준수 확인)과 DESIGN.md 토큰 사용 규율은 기본기가 탄탄하다. 반면 인증 토큰 refresh 체인에는 `apiClient` 호출부가 아직 없어 잠복 중인 실결함이 여러 개 있고, 배포 인프라는 프론트 배포 시 다른 서비스(api/console)에 실제 순단을 일으키는 상태다. `packages/*` 3종은 상당 부분이 빈 배럴(`export {}`)로, 아직 필요 없는 추상화가 먼저 깔려 있다. 8일 전(`docs/backlog/codebase-review-260714.md`) 내부 리뷰에서 지적된 Tier 1 항목 중 시크릿 주입 문제는 이번에 해결이 확인됐고(잘된 점 참고), 나머지는 대부분 미해결로 재확인됐다.

| 심각도 | 건수 |
|---|---|
| Critical | 1 |
| High | 3 |
| Medium | 6 |
| Low / Nit | 7 |

## Critical

### C1. 프론트엔드 배포마다 nginx 공유 컨테이너가 내려가 api.vivac.app·console.vivac.app 순단 — `infra/nginx.conf:8-54`, `.github/workflows/deploy.yml:86-87`
- **문제:** `infra/nginx.conf`가 `vivac.app`(→`web:3000`), `api.vivac.app`(→`api:8000`), `console.vivac.app`(→`console:3000`) 3개 도메인을 한 nginx 컨테이너에서 라우팅한다. 그런데 `infra/docker-compose.yml:1-40`은 이 저장소(frontend) 소유의 `web`+`nginx` 서비스만 정의하고 `api`/`console` 컨테이너는 정의하지 않는다 — 즉 세 서비스가 같은 EC2 호스트에서 별도 repo의 compose로 떠서 `vivac-network`(external) 하나를 공유하는 구조다. `.github/workflows/deploy.yml:86-87`의 `docker compose down` → `docker compose up -d`는 frontend 배포마다 이 공유 nginx까지 내렸다 올리므로, api/console이 살아있어도 그 몇 초간 vivac.app만이 아니라 api.vivac.app·console.vivac.app까지 502/연결거부를 겪는다.
- **근거:** `infra/nginx.conf:25-37`(api.vivac.app 블록), `:39-54`(console.vivac.app 블록), `infra/docker-compose.yml:19-27`(nginx 서비스, 이 repo가 소유), `.github/workflows/deploy.yml:85-88`(`pull`→`down`→`up -d`). 서비스 경계상 이 nginx는 frontend 하나만의 자산이 아닌데 frontend 배포 파이프라인이 통째로 재기동시킨다.
- **제안:** 최소 변경으로는 `docker compose down` 대신 `docker compose up -d --no-deps --force-recreate web`처럼 자기 서비스만 재기동. 근본적으로는 도메인별 edge 라우팅(nginx 대신 CloudFront 오리진 분기, 또는 별도 소유의 edge compose)으로 옮겨 frontend 배포가 다른 repo의 ingress를 건드리지 않게 분리.

## High

### H1. Refresh token 만료/폐기 시 stale access token이 유지되어 자동 로그아웃되지 않음 — `apps/web/src/auth.ts:48-54`
- **문제:** `serverRefreshToken`이 백엔드 401(만료·폐기된 refresh token)로 `null`을 반환하면 `if (refreshed) {...}` 블록만 있고 else가 없어 `token.accessToken`/`token.refreshToken`이 만료 전 값 그대로 유지된다. `SessionBridge.tsx:32-35`의 `registeredRefresh`는 `update({refresh:true})`가 돌려주는 이 "변경 없는" 세션에서 여전히 truthy한 `accessToken`을 읽어 `next?.accessToken ?? null`로 **non-null 값을 반환**한다. 이어서 `packages/api/src/client/index.ts:73-81`의 `if (!newToken)`은 이 stale 토큰을 성공으로 오인해 `clearAuth()`를 건너뛰고 만료된 토큰으로 원 요청을 재시도한다(`original._retry` 가드 덕에 무한루프는 아니지만, 재시도도 401로 실패하고 그대로 reject된다).
- **근거:** 세 파일에 걸친 체인을 직접 추적해 확인. 현재는 `apiClient`(axios 인스턴스)를 실제로 호출하는 코드가 앱에 없어(아래 L1 참고) 미발현이지만, refresh 로직 자체의 결함이라 첫 클라이언트 데이터 페칭 기능이 붙는 즉시 발현된다 — 사용자는 "로그인된 것처럼 보이지만 모든 요청이 401"인 상태에 조용히 갇히고 재로그인 유도가 없다.
- **제안:** `auth.ts`의 refresh 분기에 `else` 추가 — `refreshed === null`이면 `token.accessToken`/`refreshToken`을 지우거나 에러 플래그를 세션에 실어, `SessionBridge`/interceptor가 이를 명확한 실패로 인식하고 `clearAuth()`(signOut)를 타도록 한다.

### H2. `NEXT_PUBLIC_API_BASE_URL`(코드가 읽는 이름) vs `NEXT_PUBLIC_API_URL`(빌드가 주입하는 이름) 불일치 — `packages/api/src/client/index.ts:11-17`
- **문제:** `apiClient`의 `resolveBaseUrl()`은 `process.env.NEXT_PUBLIC_API_BASE_URL`을 읽지만, `Dockerfile:24-25`와 `.github/workflows/deploy.yml:36-37`는 빌드 시 `NEXT_PUBLIC_API_URL`을 주입한다. `NEXT_PUBLIC_*`는 Next.js가 빌드 타임에 정적 치환하므로 런타임에 값을 맞출 방법이 없다 — 이름이 어긋난 채로 배포되면 주입값은 항상 무시되고 `apiClient`는 항상 `"/api"` fallback으로 baseURL이 고정된다.
- **근거:** `packages/api/src/client/index.ts:13`, `Dockerfile:24-25`, `.github/workflows/deploy.yml:36-37`를 대조 확인. 현재 `apiClient` 호출부가 없어 실피해는 없지만(`/api` fallback 자체는 nginx 프록시와 맞물려 우연히 동작할 수도 있음), 의도한 값이 실제로는 한 번도 전달되지 않는 죽은 배선 상태.
- **제안:** 변수명을 한쪽으로 통일(`NEXT_PUBLIC_API_URL`로 맞추거나 Dockerfile/CI를 `NEXT_PUBLIC_API_BASE_URL`로 수정). `/api` fallback만으로 충분하다면 반대로 빌드 인자 자체를 정리.

### H3. 보안 헤더 전무 + `accessToken`이 브라우저에서 읽히는 세션 JSON에 그대로 실림 — `apps/web/next.config.ts`, `apps/web/src/auth.ts:58-64`
- **문제:** `auth.ts`의 `session` 콜백(58-64행)이 백엔드 access token을 `session.accessToken`에 담아 `/api/auth/session` 응답과 클라이언트의 `useSession()`으로 그대로 노출한다(refresh token은 서버 JWT에만 있어 상대적으로 안전 — 이 설계 자체는 양호). 그런데 `next.config.ts`에는 `headers()`가 전혀 없어 CSP·X-Frame-Options·HSTS·X-Content-Type-Options 등 어떤 방어 헤더도 없다. 현재 코드베이스에 `dangerouslySetInnerHTML`이나 사용자 콘텐츠 렌더링은 없어(grep 확인) 당장 뚫을 XSS 벡터는 안 보이지만, access token을 브라우저 JS가 읽을 수 있게 설계한 이상 XSS 한 번의 파급력이 크고, clickjacking 등 XSS와 무관한 기본 방어도 비어 있다.
- **근거:** `apps/web/next.config.ts` 전체(26줄, `headers` export 없음), `apps/web/src/auth.ts:58-64`.
- **제안:** `next.config.ts`에 `headers()`로 최소 `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Referrer-Policy`, CSP를 추가. 근본적으로는 access token을 세션에서 빼고 서버(proxy route 또는 middleware)가 요청마다 `auth()`로 세션을 읽어 `Authorization` 헤더를 주입하는 BFF 형태로 옮기면 브라우저 JS가 토큰을 아예 볼 수 없다.

## Medium

### M1. `SessionBridge` 마운트 이전 구간에 refresh가 호출되면 정상 세션도 강제 로그아웃될 수 있음 — `apps/web/src/features/auth/SessionBridge.tsx:16,31-36`
- **문제:** 모듈 스코프 변수 `registeredRefresh`(16행)는 `null`로 시작해 `useEffect`(31-36행, 컴포넌트 커밋 이후에만 실행)에서 비로소 실제 함수로 채워진다. 그 사이 시점에 `apiClient` 인터셉터가 401을 받아 `refresh()`를 호출하면 `configureAuth`에 등록된 `refresh: async () => (registeredRefresh ? registeredRefresh() : null)`가 `null`을 반환 — 실제로는 refresh가 가능한 상황인데도 실패로 간주되어 `clearAuth()`(signOut)가 호출된다.
- **근거:** `SessionBridge.tsx:16`(모듈 변수 초기값), `:31-36`(effect 내 대입 시점), `packages/api/src/client/auth.ts:21`(null 시 실패 취급). 창은 좁지만(마운트~커밋 사이 1틱) 결정론적으로 존재하는 레이스.
- **제안:** `registeredRefresh`를 effect가 아니라 모듈 로드 시점에 즉시 유효한 함수(`getSession`/`update`를 직접 참조)로 초기화하거나, `SessionBridge` 마운트 전에는 interceptor가 refresh를 시도하지 않도록 게이트.

### M2. `middleware.ts`의 `PROTECTED_PREFIXES`/`config.matcher` 이중 관리 트랩 — `apps/web/src/middleware.ts:14,37`
- **문제:** 라우트 보호 여부는 `PROTECTED_PREFIXES` 배열(14행, 현재 `[]`)로 판단하지만, 미들웨어 자체가 실행되는 조건은 별도의 `config.matcher`(37행, 현재 `["/login"]`로 고정)다. 향후 `/mypage` 같은 보호 라우트를 `PROTECTED_PREFIXES`에만 추가하고 `matcher`에 등록하지 않으면, 코드는 "보호되는 것처럼" 보이지만 미들웨어 자체가 그 경로에서 실행되지 않아 게이트가 조용히 무력화된다.
- **근거:** 코드 내 주석(11-12행, 35행)이 이미 이 함정을 경고하고 있음 — 즉 알려진 위험이 문서화만 된 채 구조적으로는 막혀 있지 않음. 현재는 `PROTECTED_PREFIXES=[]`라 실피해 없음.
- **제안:** `matcher`를 `PROTECTED_PREFIXES`에서 파생하거나, 첫 보호 라우트 추가 시점에 broad matcher(`["/((?!_next|api).*)"]` 등) + 코드 내 게이팅으로 전환.

### M3. `eslint-plugin-react-hooks` 미설치 — hooks 관련 규칙의 안전망이 없음 — `tooling/eslint-config/index.js`, `apps/web/eslint.config.js`
- **문제:** 두 eslint flat config 어디에도 `eslint-plugin-react-hooks`가 없다(`package.json`/lockfile 전체 검색 결과 의존성 자체가 미설치). `@next/eslint-plugin-next`(Next 전용 규칙만 포함)와는 별개 패키지라 `react-hooks/rules-of-hooks`(조건부 훅 호출 등 문법 오류)와 `react-hooks/exhaustive-deps`(의존성 배열 누락)가 CI에서 전혀 잡히지 않는다.
- **근거:** `pnpm-lock.yaml` 및 모든 `package.json`에서 `react-hooks` 미검출. 위 M1의 `SessionBridge` 레이스가 바로 이 안전망 부재의 실사례 — `exhaustive-deps`가 있었어도 이 특정 패턴(모듈 변수 대입)은 못 잡았겠지만, 앞으로 `useEffect` 의존성 누락류 버그가 생겨도 아무도 못 잡는다는 뜻.
- **제안:** `eslint-plugin-react-hooks`를 `tooling/eslint-config`에 추가하고 `recommended` 규칙 세트를 켠다.

### M4. `/api/[...path]/route.ts`가 `next.config.ts`의 rewrite와 완전히 중복 — 항상 rewrite가 이겨 route.ts는 도달 불가능한 코드 — `apps/web/next.config.ts:13-22`, `apps/web/src/app/api/[...path]/route.ts:1-60`
- **문제:** `next.config.ts`의 `rewrites()`(배열 반환 = `afterFiles` 처리)가 `/api/:path((?!auth/).*)`를 `API_PROXY_ORIGIN`으로 직접 외부 rewrite한다. Next.js의 라우팅 순서상 `afterFiles` rewrite는 동적 라우트보다 먼저 매치되고, 매치된 destination이 외부 URL이므로 Next 내부 라우터로 돌아오지 않는다 — 즉 60줄짜리 catch-all Route Handler(`route.ts`, hop-by-hop 헤더 스트리핑·에러 핸들링 포함)가 실행될 일이 없다. `docs/INDEX.md:52`도 이 사실을 알려진 한계로 이미 기록해 두고 있다.
- **근거:** 두 파일이 동일한 `/api/*`(단, `/api/auth/*` 제외) 범위를 이중으로 구현. `route.ts`는 `AbortSignal.timeout` 등 타임아웃 처리도 없어 살아있었다면 그 자체로도 개선 여지가 있었을 코드지만, 현재는 순수 도달 불가능 코드.
- **제안:** 하나만 남긴다. 위 H3의 BFF 전환(서버에서 Authorization 주입)을 하려면 `route.ts` 쪽을 살리고 `rewrites()`를 제거하는 방향이 더 유리.

### M5. 서버 세션을 `SessionProvider`에 전달하지 않아 로그인 상태에서도 첫 렌더에 "로그인" 링크가 플래시됨 — `apps/web/src/app/Providers.tsx:17`, `apps/web/src/components/TopBar.tsx:14,130-140`
- **문제:** `Providers.tsx:17`의 `<SessionProvider>`가 `session` prop 없이 렌더링된다. `layout.tsx`(서버 컴포넌트)도 `auth()`를 호출해 세션을 내려주지 않는다. 그 결과 `useSession()`은 클라이언트에서 `/api/auth/session`을 fetch할 때까지 `status: "loading"`이고, `TopBar.tsx:130`의 `status === "authenticated" ? 로그아웃 : 로그인` 삼항연산자가 `loading`을 `unauthenticated`와 동일하게 취급해 실제로는 로그인된 사용자에게도 "로그인" 링크가 잠깐 보였다가 로그아웃 버튼으로 바뀐다.
- **근거:** `Providers.tsx:1-26` 전체(session prop 없음), `TopBar.tsx:14`(`status`만 구조분해, `loading` 분기 없음), `:130`(삼항연산자).
- **제안:** `layout.tsx`에서 `const session = await auth()`로 세션을 읽어 `<Providers session={session}>` → `<SessionProvider session={session}>`로 내려주면 최초 렌더부터 정확한 상태로 시작한다.

### M6. core 에러 envelope(`{error:{code,message}}`)를 파싱하지 않고 HTTP status만 사용 — `apps/web/src/features/auth/serverAuthApi.ts:26-28,37-39,53-55`
- **문제:** `serverLoginWithGoogle`/`serverFetchMe`/`serverRefreshToken` 세 함수 모두 `!res.ok`일 때 응답 바디를 읽지 않고 상태 코드만으로 `Error`를 던지거나(로그인/me) `null`을 반환한다(refresh). vivacapi-core는 모든 에러를 `{error:{code,message,details}}` 봉투로 통일해 내려주는데(레포 컨벤션), 프론트는 이 봉투를 어디서도 파싱하지 않아 `INVALID_GOOGLE_TOKEN` 같은 구체적 원인이 전부 버려지고 로그가 상태 코드로만 남는다.
- **근거:** 세 함수 모두 동일 패턴 반복(26-28, 37-39, 53-55행). 로그인 흐름은 사용자가 가장 먼저 마주치는 경로라 실패 원인 유실의 디버깅 비용이 크다.
- **제안:** `!res.ok` 분기에서 `await res.json()`으로 봉투를 읽어 `code`/`message`를 로그 또는 `Error` 메시지에 포함(응답 바디가 봉투 형식이 아닐 catch 대비 필요).

## Low / Nit

### L1. 워크스페이스 패키지 3종 상당 부분이 실질적으로 빈 배럴 — `packages/shared/src/utils/index.ts:7`, `packages/shared/src/constants/index.ts:6`, `packages/api/src/queries/index.ts:6`
- **문제:** `export {}`만 있는 파일이 각각 "향후 확장 예정" 주석과 함께 패키지 구조(자체 `package.json`/`tsconfig.json`/`eslint.config.js`/`README.md`)를 온전히 갖추고 있다. `@vivac/api`의 실제 알맹이인 axios 인터셉터·refresh 큐(`packages/api/src/client/index.ts`, ~90줄)도 앱 어디서도 호출되지 않고(grep 확인, `configureAuth`만 소비됨), `@tanstack/react-query`는 `QueryProvider`로 배선까지 됐지만 `useQuery`/`useMutation` 호출이 코드베이스 전체에 0건이다. `@vivac/store`도 소비처가 `SessionBridge` 1곳뿐인 15줄 zustand 스토어에 패키지 3중 포장.
- **근거:** CLAUDE.md 2절 "No abstractions for single-use code" / "No flexibility...that wasn't requested"에 해당. 모바일은 별도 repo(`vivac-mobile-test`)라 `workspace:*` 공유가 아직 성립하지 않는 점도 지금 이 분리의 실익을 낮춘다.
- **제안:** 지금 당장 급하지는 않음 — placeholder 배럴은 실사용 시점에 채우고, 정말 여러 곳에서 쓰이기 전까지는 `apps/web/src/lib`에 인라인해도 충분.

### L2. `LogoutButton` 컴포넌트 미사용 — `apps/web/src/features/auth/LogoutButton.tsx`
- **문제:** 앱 전체에서 이 컴포넌트를 import하는 곳이 없다(grep 확인). `TopBar.tsx:133-135`가 `signOut({callbackUrl:"/"})`을 인라인으로 중복 구현해 로그아웃 버튼을 이미 자체 렌더링하고 있다.
- **제안:** 삭제하거나, 두 곳의 로그아웃 버튼 스타일이 실제로 같다면 `TopBar`가 이 컴포넌트를 재사용하도록 교체.

### L3. `MswProvider` + 빈 `handlers` 배열 — dev 첫 렌더를 블로킹하는 장치가 아무것도 모킹하지 않음 — `apps/web/src/app/MswProvider.tsx`, `packages/api/src/mocks/handlers.ts:9`
- **문제:** `NEXT_PUBLIC_API_MOCKING=enabled`일 때 `MswProvider`는 MSW 워커가 `start()`할 때까지 `children` 렌더를 보류한다(정상적인 패턴 자체는 양호). 그런데 `handlers`가 빈 배열이라 워커가떠도 가로채는 요청이 없다 — 로그인이 next-auth 서버 플로우로 전환되며 마지막 핸들러가 사라진 뒤 정리되지 않은 것으로 보인다.
- **제안:** 지금 모킹 대상이 없다면 `MswProvider`/`mocks/browser.ts`/msw 의존성을 함께 제거하고, 필요해지면 재도입.

### L4. Dockerfile의 `NEXT_PUBLIC_GOOGLE_CLIENT_ID` build-arg가 죽은 값 — `Dockerfile:27-28`
- **문제:** NextAuth v5 Google Provider는 `AUTH_GOOGLE_ID`/`AUTH_GOOGLE_SECRET`(런타임 env, `.github/workflows/deploy.yml:82`에서 이미 주입)을 쓴다. `NEXT_PUBLIC_GOOGLE_CLIENT_ID`는 NextAuth 전환 이전 클라이언트 사이드 Google Identity Services 흐름의 잔재로 보이며 현재 코드 어디서도 읽지 않는다.
- **제안:** build-arg와 CI의 `secrets.NEXT_PUBLIC_GOOGLE_CLIENT_ID` 주입 라인을 함께 삭제.

### L5. `membership_tier`를 zod `z.string()`으로 느슨하게 타이핑 — `packages/shared/src/types/auth.ts:17`
- **문제:** `.openapi.json`상 `MembershipTier`는 `"free" | "member"` 2값 enum인데 프론트 스키마는 임의 문자열을 다 통과시킨다. 런타임 검증도, 이 필드로 분기하는 UI를 나중에 짤 때의 타입 exhaustiveness도 못 받는다.
- **제안:** `z.enum(["free", "member"])`로 교체.

### L6. `authTokensSchema.token_type`이 계약보다 과도하게 엄격 — `packages/shared/src/types/auth.ts:6`
- **문제:** OpenAPI 계약(`TokenResponse`)에서 `token_type`은 `required` 목록에 없는 optional(default `"bearer"`) 필드인데, 프론트는 `z.literal("bearer")`로 **필수 + 정확히 일치**를 요구한다. 백엔드가 계약 범위 내에서 이 필드를 생략하는 응답을 보내는 순간 `.parse()`가 던져 로그인/refresh 전체가 깨진다.
- **제안:** `z.literal("bearer").optional()` 또는 `z.string().optional()`로 완화. 실무 영향은 작지만(FastAPI가 보통 default 값도 직렬화에 포함) 계약 변경에 취약한 지점.

### L7. `SpotCarousel`이 resize를 구독하지 않아 뷰포트 변경 후 화살표 표시가 stale해질 수 있음 — `apps/web/src/features/home/SpotCarousel.tsx:25-27`
- **문제:** `atStart`/`atEnd`는 스크롤 이벤트와 마운트 시 1회(`useEffect(..., [])`)만 갱신된다. 창 크기 조절로 `scrollWidth`/`clientWidth` 관계가 바뀌어도(예: 데스크톱 창을 넓혀 캐러셀이 다 들어오게 됨) 재계산 트리거가 없어 더 이상 스크롤할 게 없는데도 화살표가 남아있을 수 있다.
- **제안:** `resize` 리스너 또는 `ResizeObserver`로 `updateEdges()` 재호출 추가. 우선순위 낮음(데스크톱 전용 UI, 클릭해도 큰 부작용 없음).

## 잘된 점

- `serverAuthApi.ts`가 백엔드 응답을 `any`로 신뢰하지 않고 `@vivac/shared`의 zod 스키마(`authTokensSchema`/`userSchema`)로 런타임 파싱해 타입 드리프트를 막고 있다. `AbortSignal.timeout(8000)`으로 백엔드 무응답 시 로그인 콜백이 무한 대기하지 않도록 상한도 둠.
- 8일 전 내부 리뷰(`docs/backlog/codebase-review-260714.md`)가 지적한 최우선 배포 블로커 — 프로덕션 `AUTH_SECRET`/`AUTH_GOOGLE_ID`/`AUTH_GOOGLE_SECRET` 런타임 미주입 — 가 이후 커밋(`e029255`, `7e1cece`)으로 해결되어 `.github/workflows/deploy.yml:78-84`에서 정상 주입되는 것을 확인했다.
- `eslint-config`가 `strictTypeChecked` + `no-explicit-any: error` 기반이고, 실제로 코드베이스 전체에 `: any`/`as any` 사용이 0건이다(grep 확인) — 규칙이 실제로 지켜지고 있다.
- `QueryProvider`가 `useState(() => new QueryClient(...))` lazy initializer로 QueryClient를 한 번만 생성하고, 그 이유(리렌더 간 캐시 유지, 모듈 전역 생성 시 SSR 요청 간 공유 위험)를 주석으로 남겼다. `MswProvider`도 unmount 후 `setState`를 막는 cleanup guard(`state.active`)를 정석대로 구현했다.
- `globals.css` 헤더가 DESIGN.md 대비 의도적 편차(success 색상 분리, 다크 팔레트 직접 파생)를 사유와 함께 명시적으로 등재하고, `RecommendedSpotCard`의 `legalStatus` 배지는 색맹 사용자를 위해 색상뿐 아니라 아이콘 형태로도 상태를 이중 인코딩하는 등 접근성을 실제로 고려한 흔적이 코드 주석에 남아 있다.
