# vivac-console 코드 리뷰 (2026-07-22)

## 요약

Next.js 16 (App Router) + TypeScript, pnpm 기반 내부 운영 콘솔. `src/` 약 3,766 LOC, Server Components 중심에 편집 폼만 client component. 데이터는 전부 `src/lib/api.ts` 헬퍼를 거쳐 `vivacapi-core`의 `/v1/internal/*`를 호출한다.

전반적으로 건강한 코드베이스다. **가장 중요한 아키텍처 규칙("DB 직접 접근 금지, `/v1/internal`만 사용")은 완전히 지켜지고 있다** — DB 드라이버 의존성이 전무하고(`pg`/`prisma`/`drizzle` 없음), 모든 네트워크 호출이 `/internal/*` 경로이며, 유일한 `/admin` 사용처는 로그인 예외(`auth.ts:58`)뿐이다. 토큰을 session에서 빼내 서버 전용 헬퍼로 읽는 처리, `OpenLink`의 `javascript:` 차단, encodeURIComponent 일관 적용 등 직전 리뷰(2026-07-14)에서 지적된 항목들이 실제로 반영돼 있다. 다만 **에러 응답 envelope 계약이 백엔드 실제 동작과 어긋나 있어** 모든 mutation 실패가 raw JSON으로 노출되는 실질 버그가 남아 있다.

| 심각도 | 건수 |
|---|---|
| Critical | 0 |
| High | 1 |
| Medium | 2 |
| Low / Nit | 6 |

---

## Critical

해당 없음. (no-direct-DB / only-`/v1/internal` 규칙 준수, client 번들 secret 유출 없음, 도달 가능한 XSS 없음)

---

## High

### H1. 에러 응답 envelope 계약 불일치 — mutation 실패가 raw JSON으로 노출 — `src/lib/api.ts:60-78`

- **문제:** `parseErrorBody`는 FastAPI 기본 형태인 `{"detail": "..."}`(문자열) 및 422의 `detail` 배열만 파싱한다. 그런데 백엔드 `vivacapi-core`는 **모든** 에러를 전역 exception handler로 감싸 `{"error": {"code", "message", "details"}}` envelope로 반환한다. `parseErrorBody`에는 `parsed.error.message`를 읽는 분기가 없어 어느 branch에도 걸리지 않고 `message = body`(raw JSON 문자열)로 폴백한다. 결과적으로 사용자는 `저장 실패 (422) {"error":{"code":"VALIDATION_ERROR","message":"허용되지 않는 상태 전이입니다: ENRICHED -> PUBLISHED","details":null}}` 같은 JSON 덩어리를 그대로 본다.
- **근거:**
  - 백엔드는 `AppException`(`vivacapi-core/vivacapi/main.py:105-107`), `RequestValidationError`(`main.py:110-118`), starlette `HTTPException`(`main.py:135-138`)까지 전부 `_error_response`(`main.py:92-102`)의 `{"error":{...}}` envelope로 감싼다. `{"detail":...}` 형태는 이 백엔드에서 **절대 생성되지 않는다.** 즉 `parseErrorBody`가 겨냥하는 shape 자체가 존재하지 않는다.
  - 이 헬퍼는 `apiMutate`/`apiCreate`/`apiDelete`(`api.ts:81-119`) 전체가 공유한다. spot/business-info 저장, group member 추가·역할변경·제거, spot-option 추가·삭제, group 삭제 등 **모든 쓰기 작업의 에러**가 영향을 받는다. 특히 group member 중복(`SPOT_GROUP_MEMBER_ALREADY_EXISTS`), option 중복, last-owner 제약 같은 도메인 에러가 사람이 읽을 메시지 대신 JSON으로 뜬다.
  - `docs/pipeline-status-review-api.md:83,88-89`는 "백엔드가 `422 + {"detail": "..."}`를 주고 콘솔이 `detail`을 파싱하도록 수정했다"고 기록하지만, 실제 배포된 백엔드는 envelope를 반환하므로 **문서가 잘못된 계약을 서술**하고 있고 파서는 그 잘못된 가정 위에 만들어졌다.
- **제안:** `parseErrorBody`에 envelope 분기를 최우선으로 추가한다. 한 줄이면 된다: `JSON.parse` 후 `if (parsed.error?.message) return \`저장 실패 (${status}) ${parsed.error.message}\`;`를 `detail` 검사보다 앞에 둔다. (하위호환으로 기존 `detail` 분기는 남겨둬도 무방하나 이 백엔드에선 죽은 코드다.) 아울러 `docs/pipeline-status-review-api.md`의 "구현 결과" 계약 서술을 envelope 기준으로 정정 권장.

---

## Medium

### M1. spots 검색 form이 `assigned_to_uid`를 누락 — My Queue에서 검색하면 조용히 담당 범위를 벗어남 — `src/app/(admin)/spots/page.tsx:113-120`

- **문제:** 목록 상단 검색 `<form>`은 GET 제출이라 URL을 form 내부 필드로 통째로 교체한다. hidden input이 `sort`, `order`, FACETS 3개(`pipeline_status`/`region_province`/`source`)와 `q`뿐이고 **`assigned_to_uid`가 빠져 있다.** 사용자가 My Queue(`/spots?pipeline_status=ENRICHED&assigned_to_uid=X`)에서 스팟명을 검색하면 결과 URL에서 `assigned_to_uid`가 사라져, 자기 할당분이 아니라 **모든 담당자의 ENRICHED 스팟**을 보게 된다. 본인 큐를 검색한다고 믿는 상태에서 남의 항목이 노출된다.
- **근거:** 목록 fetch(`page.tsx:62-70`)와 `buildQuery`(sort/pagination 링크, `page.tsx:76-82`)는 이미 `assigned_to_uid`를 보존하도록 고쳐져 있는데, 검색 form의 hidden input 목록만 갱신이 누락됐다. 이는 직전 리뷰에서 Tier 1(critical)으로 다뤘던 "My Queue가 필터 없이 전체를 보여줌"과 **동일한 버그 클래스**가 검색 경로에만 재발한 것이다. (Submit/Reject 버튼은 `assigned_to_uid === currentUserId` 가드가 있으나 일반 Save는 아무 스팟에나 열려 있어, 잘못된 스코프에서 편집이 가능하다.)
- **제안:** form에 hidden input 한 줄 추가 — `<input type="hidden" name="assigned_to_uid" value={assignedToUid ?? ""} />`. `FacetFilter`처럼 URLSearchParams 보존 방식으로 통일하는 것도 방법.

### M2. Business Info 목록에서 스팟명 링크가 행 클릭에 먹힘 — 스팟이 아니라 사업정보로 이동 — `src/app/(admin)/spot-business-info/page.tsx:101-109`, `src/components/admin/clickable-row.tsx:14-21`

- **문제:** 행 전체가 `ClickableRow`(→ `/spot-business-info/{uid}/edit`)인데, 첫 셀 안에 **다른 목적지**를 가리키는 `<Link href="/spots/{spot_uid}/edit">`(스팟 편집)가 중첩돼 있다. Next의 `Link`는 클릭 시 `preventDefault`만 하고 `stopPropagation`은 하지 않으므로, 이벤트가 `<tr>`로 버블링돼 행의 `onClick`이 뒤이어 실행된다. 두 `router.push`가 같은 tick에 호출되고 **나중에 실행되는 행 쪽(business-info edit)이 이긴다.** 결과적으로 "스팟으로 가기" 링크가 죽어서, 스팟명을 눌러도 사업정보 편집 화면에 도착한다.
- **근거:** `ClickableRow`(`clickable-row.tsx`)는 target 검사 없이 `<tr onClick>`에 무조건 `router.push(href)`. spots 목록·spot-groups 목록의 중첩 링크는 행과 목적지가 동일(edit chevron)이라 무해하지만, SBI 목록만 목적지가 갈린다. (`spot-sdp-field-mapping.md:104`에 "클릭 시 `/spots/{spot_uid}/edit`로 이동"으로 명시된 의도된 affordance가 무력화됨.)
- **제안:** 스팟명 `Link`의 `onClick`에서 `e.stopPropagation()` 호출, 또는 `ClickableRow`가 `e.target.closest("a")` 존재 시 행 네비게이션을 건너뛰도록 가드.

---

## Low / Nit

### L1. `apiMutate`/`apiCreate`/`apiDelete` 사실상 중복 — `src/lib/api.ts:81-119`
세 함수가 HTTP method와 body 유무만 다르고 나머지(fetch → ok면 null → 아니면 `parseErrorBody`)가 동일하다. `apiSend(method, path, data?)` 하나로 접을 수 있다. 직전 리뷰가 `saveSpot`/`saveBusinessInfo` 중복을 `apiMutate`로 통합했는데, 이후 groups/options 기능이 같은 패턴을 다시 세 벌로 늘렸다. (기능 영향 없음, 정리 차원.)

### L2. mutation 후 `revalidatePath`/`router.refresh` 부재 — `src/components/admin/spot-edit-form.tsx:259` 외
저장 성공 시 client form들이 `router.push`만 하고 캐시 무효화를 하지 않는다(`revalidatePath`/`revalidateTag`/`router.refresh` 사용처 grep 결과 0건). 목록 fetch가 `cache: "no-store"`이고 Next 16 dynamic 페이지의 Router Cache staleTime이 0이라 현재는 최신값이 보일 가능성이 높지만, Server Action의 정석은 영향 route를 `revalidatePath`로 무효화하는 것이다. Next 버전/설정 변화에 취약한 암묵적 의존이라 명시화 권장. (group member/spot/option action은 서버 `redirect`를 써서 상대적으로 덜 취약.)

### L3. 백엔드 다운 시 로그인 실패가 "권한 없음"으로 오표기 — `src/auth.ts:63,68` → `src/app/login/page.tsx:5`
`signIn` 콜백이 백엔드 fetch 실패/`!res.ok`에 `return false`를 하면 NextAuth가 `AccessDenied`로 처리하고, 로그인 페이지는 이를 "권한이 없는 계정이거나 등록되지 않은 사용자입니다"로 매핑한다. 즉 일시적 인프라 장애가 영구적 권한 문제처럼 보여 staff가 접근 권한이 없다고 오해할 수 있다. 네트워크 실패와 인증 거부를 구분하는 에러 코드 분기 고려.

### L4. 서드파티 CDN 런타임 로드 + `script-src` CSP 부재 — `src/app/layout.tsx:32`, `src/components/admin/spot-edit-form.tsx:81-90`
Pretendard 폰트를 jsdelivr에서, Daum postcode/Kakao Maps SDK를 각 CDN에서 런타임 `loadScript`로 불러온다. `next.config.ts`의 CSP는 `frame-ancestors 'none'`만 있고 `script-src`가 없다. 직전 리뷰가 로그인·카카오·OSM 영향 우려로 의도적으로 보류한 항목이나, 공급망/가용성 잔여 리스크로 인지 유지 권장(폰트 self-host, CDN 도메인 한정 CSP).

### L5. 검색 파라미터(`title_like`/`name_like`)의 백엔드 화이트리스트 반영 미확인 — `src/app/(admin)/spots/page.tsx:67`, `src/app/(admin)/spot-groups/page.tsx:39`
목록 검색이 `title_like`(spots), `name_like`(groups)를 보낸다. 백엔드 `_FILTERABLE` 화이트리스트에는 `region_province`/`source`/`pipeline_status`/`assigned_to_uid`만 있고(`vivacapi-core/vivacapi/crud/spot.py:31-36`) `*_like` 검색은 별도 경로로 처리되는 것으로 보이나 이 repo에서 확정 불가. 검색이 실제로 서버에서 필터링되는지(무시되고 전체 반환되지 않는지) 통합 확인 필요.

### Nit. `X-Total-Count`는 CORS `expose_headers` 의존 — `src/lib/api.ts:55`
`Number(res.headers.get("x-total-count") ?? 0)`는 헤더가 없으면 total=0 → 페이지네이션이 1페이지로 접힌다. 백엔드 CORS `expose_headers`에 `X-Total-Count` 등록이 전제(`.claude/rules/api-conventions.md` 규약)이며 콘솔에서는 검증 불가.

---

## 잘된 점

- **핵심 아키텍처 규칙 완전 준수:** DB 드라이버 의존성 0, 모든 호출이 `src/lib/api.ts` 경유 `/internal/*`, 유일한 `/admin` 사용은 로그인 예외(`auth.ts:58`). CLAUDE.md/README 계약 그대로.
- **서버 전용 토큰 처리:** backend accessToken을 NextAuth session에 넣지 않고 `getAccessToken`(`auth.ts:86-93`)이 JWT 쿠키에서 직접 decode → `/api/auth/session`으로 브라우저에 노출되지 않음. 직전 리뷰 Tier 1 이슈가 실제로 해소됨.
- **audit-history 계약 충실 이행**(`change-history.tsx`): `changed_by_name` null→"시스템", 빈 `changes`→"변경사항 없음", 타입별 값 렌더(배열 join/불리언 예·아니오), business_info는 record uid로 history 조회(`spot-business-info/[uid]/edit/page.tsx:34`) — 문서 경고 그대로 준수.
- **대시보드 `myPending` 계산이 수학적으로 정확:** `my_assigned_total - my_completed`가 `pipeline_status=ENRICHED&assigned_to_uid=me` 링크 대상과 정확히 일치한다(백엔드 `my_completed` = assigned AND status≠ENRICHED, 그리고 `pipeline_status`가 NOT NULL이므로 여집합 성립 — `vivacapi-core/vivacapi/crud/spot.py:217-233` 확인). 2026-07-14 backlog에서 [열림]으로 남았던 우려는 실제로는 정합함.
- **`OpenLink` protocol 화이트리스트**(`spot-edit-form.tsx:509-527`): ETL이 수집한 신뢰 불가 `website_url`에서 `http:`/`https:`만 허용해 `javascript:` 벡터 차단.
- 화이트리스트 기반 FACET, 전 uid 경로 세그먼트 `encodeURIComponent`, `ApiError.status`로 404 vs 그 외 구분(`notFound()` 오용 방지) 등 기본기가 탄탄함.
