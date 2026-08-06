# vivac-console 리팩터링 — 2026-08-02

- 범위: repo 전체 (`src/` + 루트 설정). 브랜치 `feature/refactor`.
- 방법: sub agent 병렬 audit 4개(components / app routes / lib+config / 신규 기능 보안·정합성) → 메인 세션이 헤드라인 finding 직접 검증 → sub agent 2개 병렬 적용(components/admin, app routes) + 메인 세션이 lib·auth·proxy·ui 처리 → 전체 diff를 리뷰 agent가 재검증 → 지적된 regression 수정.
- 결과: 36개 파일 수정, 신규 5개(148줄), 파일 3개 삭제. **-782 / +459줄** (신규 포함 실질 약 -180줄).
- 검증: `pnpm typecheck` 0, `pnpm lint` 0 error(경고 1, 아래 참조), `pnpm build` 성공.
- 이전 리뷰: [`codebase-review-260714.md`](./codebase-review-260714.md).

---

## 🔴 보안

### [완료] 🔴 proxy matcher가 경로 "포함" 확장자를 전부 skip → server action 무인증 호출 가능
- 위치: `src/proxy.ts:25` (구 matcher `"/((?!_next|favicon.ico|.*\\.svg|.*\\.png).*)"`)
- 배경: 부정 lookahead의 `.*\.png`에 `$` 앵커가 없어 **경로 중간**에 `.png`가 있어도 매칭됐다. `new RegExp(...).test("/spot-groups/abc.png/edit") === false` — 즉 middleware가 아예 실행되지 않는다.
- 노출: `"use server"` action은 세션 없이도 POST 가능한 공개 엔드포인트다. `/spot-groups/x.png/edit`로 `Next-Action` 헤더를 붙여 POST하면 proxy를 건너뛰고 `queueDbDump` 같은 전체 DB 덤프 action이 실행된다. 260714 리뷰에서 이 matcher를 `[열림] … 현재 해당 route 없어 무해`로 남겨뒀는데, 그 뒤 들어온 spot-groups / DB dump 커밋이 바로 그 route를 만들었다.
- 처리(2중):
  1. matcher를 `"/((?!api/auth|_next/static|_next/image|.*\\.(?:svg|png|jpg|jpeg|gif|webp|ico)$).*)"`로 교체 — 확장자는 경로 **끝**에서만 제외. 정적 자산·`/api/auth`는 그대로 통과, `.png`가 낀 임의 경로는 이제 proxy를 탄다(regex 실측 확인).
  2. `src/lib/api.ts`의 `authHeaders()`가 토큰 없으면 `ApiError(401)`을 던진다. backend 호출은 전부 이 함수를 지나므로 proxy를 우회한 경로가 또 생겨도 여기서 막힌다. (기존에는 `Bearer ` 빈 헤더를 그대로 보내 backend 판단에 의존)

### [완료] 🟠 `window.location.href = download_url` — scheme 검증 없음
- 위치: `src/components/admin/db-dump-button.tsx`
- 260714에서 `OpenLink`에 적용한 것과 같은 부류. `new URL()` 파싱 후 `http:`/`https:`만 허용.

## 🟠 Correctness

### [완료] picker 2종의 stale response / 미처리 rejection / lint error
- 위치: `spot-picker.tsx`, `group-picker.tsx`
- debounce가 타이머만 취소하고 in-flight 요청은 두므로 **늦게 도착한 옛 응답이 최신 결과를 덮어썼다**(느린 광역 검색 → 좁힌 검색 → 옛 결과 표시). effect마다 `cancelled` 플래그로 폐기.
- `.catch` 부재로 API 500이 unhandled rejection이 되고 화면상 "결과 없음"과 구분 불가 → 에러 문구 표시.
- effect 본문 동기 `setState` 제거(기존 `pnpm lint` error 2건 해소). 빈 입력 처리는 렌더에서 파생.
- 이미 선택된 항목을 서버 `_end: 10` **이후에** 걸러내 드롭다운이 비는 문제 → `limit`에 선택 개수만큼 여유분.

### [완료] DB dump 폴링이 겹치거나 영영 안 풀림
- `setInterval` + async 콜백 → 조회가 2.5s보다 오래 걸리면 tick이 겹치고, 늦은 tick이 완료된 작업을 "다운로드 준비 중…"으로 되돌려 버튼이 영구 비활성.
- 응답 후 다음 tick을 예약하는 `setTimeout` 체인으로 교체 + 전체 시한은 **별도 타이머**로 계측(조회 요청 하나가 hang해도 5분 뒤 풀린다).

### [완료] ClickableRow 안의 `<a>`와 행 onClick이 동시 발화
- 목록 행이 `router.push`로 이동하는데 행 안에 다른 href를 가진 `<Link>`가 있었다(spot-business-info의 스팟 제목). 어느 쪽이 이길지 비결정적.
- `e.target.closest("a")`면 행 핸들러가 빠지도록 수정 — 모든 목록 페이지에 한 번에 적용되는 root-cause 지점.

### [완료] 그 외
- `apiCreate`/`apiDelete`/`apiMutate`가 transport 실패(backend 재시작 등)에 throw → server action 전체가 error boundary로 날아갔다. 이제 에러 **문자열**로 흡수, 호출부는 기존 `if (error)` 분기 그대로.
- `parseErrorBody`가 삭제·생성 실패에도 "저장 실패"를 붙이던 문구 → "요청 실패".
- `spots/[uid]/edit`의 그룹 조회 `.catch(() => [])`가 500·네트워크 실패까지 삼켜 "소속 그룹 0개"로 표시 → 404만 degrade.
- `spots/page.tsx`의 `rating_avg.toFixed(1)` null 가드(주변 셀은 전부 `?? "-"`인데 여기만 무방비, API가 null이면 페이지 전체 500).
- 검색 form이 `assigned_to_uid` hidden input을 안 넘겨 My Queue에서 검색하면 담당자 필터가 사라지던 문제.
- `spot-groups/new`가 스팟을 순차 루프로 추가(50개면 왕복 50회 직렬) → `Promise.all`. 부분 실패 시 `saved=1`과 `error=`를 같이 붙여 초록·빨강 배너가 동시에 뜨던 것도 수정.
- `spot-groups/[uid]/edit` 스팟 제거 시 `spots_offset` 유실 → 유지. 페이지 수가 줄면 마지막 페이지로 클램프(빈 화면에 갇히지 않게).
- `FacetFilter`가 `saved`/`error`를 다음 URL로 실어날라 "저장되었습니다"가 재출현하던 문제.
- `auth.ts`의 `API_BASE_URL` 미검증 + `catch {}` 무음 → fail-fast + `console.error`.
- `searchParams`를 `Record<string, string | undefined>`로 단언하던 3개 목록 페이지 — 실제로는 `string[]`이 올 수 있고 `apiList`가 `"a,b"`로 stringify해 필터가 조용히 오작동. `listQuery`에서 정규화.
- `error.tsx`가 `error.message`를 그대로 렌더(dev에선 내부 API 응답 본문 노출, prod에선 마스킹돼 아무 정보 없음) → 고정 문구 + `digest`.

## 🟡 중복 제거 / 철거

- **`src/lib/list-query.ts` 신설** — `sort/order/page/start` 파싱 + `href`/`sortLink`/`sortIndicator`. 목록 3개 페이지가 각자 갖고 있던 `buildQuery` 3벌, `PAGE_SIZE` 3벌 제거. `spot-business-info`가 문자열 결합으로 만들던 링크(인코딩 없음)도 URLSearchParams 경유로 정리.
- **공용 컴포넌트 신설** — `ui/pager.tsx`(4곳), `ui/status-banner.tsx`(10곳), `ui/field.tsx`(4곳), `EmptyRow`(6곳).
- **`api.ts` 축약** — `apiFetch`/`apiList`가 `request()` 공유, 쓰기 4종이 `send()` 공유. `apiFetchOr404`(3곳의 try/catch+notFound 복붙), `redirectResult`(9곳의 redirect 삼항) 추가.
- `spot-search.ts` + `group-search.ts` → `actions/search.ts` 한 파일.
- `delete-option-button.tsx` 삭제 → `ConfirmActionButton` + bind.
- `db-dump.ts`의 try/catch 3벌 → `safe()` 하나.
- `types.ts` — union 타입을 const 배열에서 파생(선언 중복 제거).
- `spot-edit-form.tsx` — `tagsField` 팩토리·`arr()` 인라인, `parseBool` 한 줄, 도달 불가 분기 제거, boolean select 3벌 → 지역 컴포넌트.
- `Table`이 자체 `overflow-x-auto` 컨테이너를 갖고 있었는데 호출부 6곳이 전부 또 감싸고 있어 스크롤 컨테이너 중첩 → 제거. 미사용 `TableFooter`/`TableCaption` 삭제.
- 설정: `tw-animate-css` 의존성 제거(`animate-spin`은 Tailwind 코어), `eslint.config.mjs`의 `globalIgnores`가 `eslint-config-next` 기본값 그대로 복사한 no-op이라 삭제, `.env.example`에 `AUTH_TRUST_HOST` 추가.

## 리뷰 agent 지적 → 처리

- 목록 3개 페이지에서 chevron `<Link>`를 "행 클릭과 중복"이라며 제거했는데, 그 결과 **행에 focusable 요소가 하나도 남지 않아 키보드로 편집 페이지 접근 불가** + cmd/중클릭 새 탭 불가. `<Link>` 복구(위 `closest("a")` 수정 덕에 중복 발화는 없음).
- `ChangeHistory`의 `ACTION[...] ?? fallback`을 "타입상 도달 불가"로 삭제했으나 값은 API에서 그대로 오는 것 — backend가 새 action을 추가하면 `action.cls`에서 크래시. 복구.
- picker 에러 문구가 입력을 비워도 남던 것, DB dump 시한 타이머, 스팟 제거 후 offset 클램프 — 위 항목에 반영.

## [열림] 남은 것

- `spot-edit-form.tsx` — `pnpm lint` 경고 1건(`react-hooks/incompatible-library`). react-hook-form `watch()` 값을 자식 컴포넌트에 넘기는 지점(3×`SearchLink`, 4×`OptionMultiSelect`, 1×`TagsInput`)이 React Compiler 메모이제이션을 막는다. `useWatch({ control, name })`로 전부 바꾸면 사라지지만 폼 동작 변경이라 별도 작업으로 분리.
- `dashboard/page.tsx` — `_start:0,_end:1`로 `X-Total-Count`만 뽑는 왕복 1회. backend stats에 `pending_review_total`을 추가하면 없앨 수 있다(repo 간 계약). 260714의 "Pending Review (Mine) 숫자/링크 불일치"도 같은 API 확인이 선행돼야 함.
- `spot-group-create-form.tsx` / `spot-group-edit-form.tsx` — name+description+visibility 폼이 사실상 동일(-55줄 여지). 이번 pass에서는 `Field`/`StatusBanner` 추출까지만.
- `ChangeHistory`의 React key가 배열 index — `HistoryEntry`에 고유 식별자가 없어 유지.
- `components/ui/button.tsx`, `badge.tsx` — 미사용 variant/size 다수(`secondary`, `link`, `icon-*` 등)와 `Badge`의 `useRender` 다형성(사용처 0). shadcn 생성물 원형을 유지하는 편이 재생성·업그레이드에 유리해 손대지 않음. cva 문자열이라 런타임 비용도 없음.
- `appleboy/*` action SHA pin, next-auth beta 의존, 8h 만료 UX — 260714에서 이월.
