# 야영장 상세페이지 UI 설계 문서

- 작성일: 2026-07-15
- 작성자: ui-designer (Claude)
- 범위: P1-1-B 장소 탐색 UI 중 상세페이지 (M5. 장소 상세 페이지) — **시각/컴포넌트 레이어**
- 선행 문서(기획, 수정하지 않음): [`spot-detail-screen-design-260715.md`](./spot-detail-screen-design-260715.md)
- 참고 소스: `DESIGN.md`(디자인 시스템 — **단일 진실 공급원**), `apps/web/src/app/globals.css`(DESIGN.md 토큰 배선), `apps/web/src/features/home/*`·`apps/web/src/components/TopBar.tsx`(기존 코딩 컨벤션), `docs/spots-explore-plan.md`(장소 탐색 리스트/지도 페이지 설계 — 같은 feature의 형제 문서), `.openapi.json`(BE 실제 스키마 스냅샷, 참고용)
- **v2 (2026-07-15)**: DESIGN.md 우선 정책 적용. v1의 "DESIGN.md 미배선 → 회색조 치환" 전제가 폐기되면서 §0, §4.2, §11.4, §12.3, §13이 개정됐다. 상세는 §14 버전 히스토리.
- **v3 (2026-07-21)**: **구현 반영.** PR #23에서 이 스펙을 구현하며 확인된 사실로 §1(라우트 경로), §1.1(`SpotSpecGrid` 시그니처), §2.1(스크롤 감지 계약), §2.7(지도 비율·상세주소 색), §8.1(`loading.tsx` 제약), §10(대비비 표·`sr-only` 규칙)을 개정했다. 각 지점에 `[v3]` 표기를 뒀다. 미해결 항목을 모을 `spot-detail-ui-followups.md`는 아직 작성되지 않았다(링크 없음).

이 문서는 기획 초안의 정보구조·필드 매핑·섹션 순서·설계 원칙 4가지를 그대로 따른다. 여기서는 "그 구조가 실제로 어떻게 생겼고 어떤 컴포넌트로 만들어지는지"만 정의한다. 기획 초안의 결정을 뒤집는 부분은 모두 **섹션 11(비판적 검토)** 에 근거와 트레이드오프를 명시했다.

---

## 0. 토큰 체계 — DESIGN.md 우선 정책

> **v2 개정**: `DESIGN.md`가 `globals.css`에 정식 배선되면서, v1이 전제했던 *"DESIGN.md는 미배선이므로 회색조 Tailwind 클래스로 치환한다"* 는 원칙은 **폐기됐다.** 이 문서는 이제 **DESIGN.md 우선 정책**을 따른다 — DESIGN.md에 해당 토큰이 있으면 무조건 그것을 쓰고, 없을 때만 새로 정의한다. v1의 관련 판단(§4.2의 `yellow-*` 재사용, §11.4의 `warning`/`link` 미채택, §12.3, §13)은 전부 뒤집혔으며 개정 근거를 각 섹션에 남겼다.

### 0.1 확정된 사실

1. **`DESIGN.md`는 `globals.css`의 `@theme`에 배선돼 있다.** `bg-canvas`, `text-ink`, `border-hairline`, `text-display-lg`, `rounded-pill`, `shadow-level-3` 같은 클래스가 **실제로 존재한다.** v1이 "존재하지 않는다"고 적은 것은 더 이상 사실이 아니다.
2. **fluid 토큰은 폐기됐다.** `--text-*` clamp 토큰과 fluid `--spacing`이 제거되고 DESIGN.md의 고정 px 스케일(base unit 4px)로 대체됐다. 따라서 `p-4` = 정확히 16px이며 DESIGN.md `{spacing.md}`와 1:1 대응한다.
3. **브레이크포인트가 DESIGN.md 기준으로 바뀌었다.** `sm:`=600px(Tablet) · `md:`=960px(Desktop) · `lg:`=1200px(Wide) · `xl:`=1400px(Ultra). Tailwind 기본값(640/768/1024/1280)이 **아니다** — 이 문서의 모든 브레이크포인트 표기는 새 값을 뜻한다.
4. **다크 램프가 파생됐다.** DESIGN.md는 라이트 전용이지만 모든 시맨틱 토큰의 다크값이 `globals.css`에 정의됐다. 컴포넌트는 토큰만 쓰면 다크모드가 자동으로 따라온다 — v1 §10의 "다크모드 미대응 한계 고지"는 **더 이상 유효하지 않다.**
5. **모노톤 원칙은 DESIGN.md 시맨틱 팔레트로 흡수됐다.** `RecommendedSpotCard`의 `legalStatus` 배지가 `green/red/yellow-*` → `success/error/warning` 토큰으로 이관됐다.

### 0.2 색상 토큰 (치환 없이 그대로 사용)

| 용도 | 클래스 | 라이트 | 다크 |
|---|---|---|---|
| 제목·본문 최상위 | `text-ink` | `#171717` | `#ededed` |
| 보조 텍스트 | `text-body` | `#4d4d4d` | `#a1a1a1` |
| 최하위(placeholder·결측·아이콘) | `text-mute` | `#888888` | `#7e7e7e` |
| 페이지·카드 배경 | `bg-canvas` | `#ffffff` | `#0a0a0a` |
| 인셋 배경(스펙 셀, 준비중 카드) | `bg-canvas-soft` | `#fafafa` | `#111111` |
| 더 깊은 인셋(칩 배경, 이미지 placeholder) | `bg-canvas-soft-2` | `#f5f5f5` | `#1a1a1a` |
| 구분선·카드 보더 | `border-hairline` | `#ebebeb` | `#2e2e2e` |
| 강한 보더(아웃라인 칩·버튼) | `border-hairline-strong` | `#a1a1a1` | `#4d4d4d` |
| CTA 솔리드 | `bg-primary text-on-primary` | 검정 위 흰색 | **흰색 위 검정**(폴라리티 플립) |
| 인라인 링크 | `text-link` | `#0070f3` | `#3291ff` |
| 허용·성공 | `bg-success-soft text-success-deep` | `#e3f5e7` / `#0f7a35` | `#12301c` / `#62c073` |
| 경고 | `bg-warning-soft text-warning-deep` | `#ffefcf` / `#ab570a` | `#3b2410` / `#f5a623` |
| 오류 | `bg-error-soft text-error-deep` | `#f7d4d6` / `#c50000` | `#3c1618` / `#ee0000` |

> ⚠️ **`success`는 DESIGN.md의 승인된 편차다(편차 #1).** DESIGN.md 원문은 `success = link = #0070f3`(파랑)으로 묶고 Don't 항목에서 여섯 번째 액센트 색 도입을 금지한다. VIVAC은 **이 규칙을 의도적으로 어기고 초록(`#45a557`)을 추가했다** — 야영 허가 여부는 이 앱의 핵심 가치("어디서 자도 되는가")이고, 그 신호를 파랑으로 표시하면 관용 색상 코드(초록=안전)와 어긋나 명료성이 떨어지기 때문. `link`는 파랑 그대로 유지된다. 편차 레지스트리 정본은 `globals.css`. 결정 경위는 §11.4.

### 0.3 타이포 토큰 (치환 없이 그대로 사용)

크기·행간·자간·굵기가 토큰에 묶여 있으므로 **`font-*`를 따로 붙이지 않는다.**

| 용도 | 클래스 | 스펙 |
|---|---|---|
| 장소명 (H1) | `text-display-md sm:text-display-lg` | 24/600/-0.96px → 32/600/-1.28px |
| 섹션 제목 | `text-display-sm` | 20px / 600 / -0.6px |
| 리드 문단 | `text-body-lg` | 18px / 400 |
| 본문·설명 | `text-body-md` | 16px / 400 |
| 보조 본문 | `text-body-sm` | 14px / 400 / -0.28px |
| 강조 보조 본문 | `text-body-sm-strong` | 14px / 500 |
| 캡션(스펙 라벨·칩) | `text-caption` | 12px / 400 |
| 버튼 (마케팅 스케일) | `text-button-lg` | 16px / 500 |
| 버튼 (인앱 스케일) | `text-button-md` | 14px / 500 |

> DESIGN.md Don't: *"디스플레이를 700으로 올리지 마라"*(상한 600). 따라서 v1이 쓰던 `font-bold`(700)는 제목에 쓰지 않는다.
> **한글 음수 자간 리스크**: display 토큰의 -0.96/-1.28px 자간은 라틴 전제라 한글에서 자소가 빽빽해 보일 수 있다. 실기기 검수 필요 — §12 후속 논의 4번.

### 0.4 모양·깊이 토큰

| DESIGN.md | 클래스 | 값·용도 |
|---|---|---|
| `{rounded.xs}` | `rounded-xs` | 4px |
| `{rounded.sm}` | `rounded-sm` | 6px — 인앱 버튼·폼 인풋 |
| `{rounded.md}` | `rounded-md` | 8px — 카드 기본 |
| `{rounded.lg}` | `rounded-lg` | 12px — 큰 카드 |
| `{rounded.xl}` | `rounded-xl` | 16px |
| `{rounded.pill}` | `rounded-pill` | 100px — 마케팅 CTA |
| `{rounded.full}` | `rounded-full` | 9999px — 배지·아이콘 버튼 |
| Level 1~5 | `shadow-level-1` ~ `shadow-level-5` | 스택 섀도 + inset hairline 링 |

> DESIGN.md Don't: *"카드에 단일 무거운 드롭섀도를 쓰지 마라"* — Tailwind 기본 `shadow-sm`/`shadow-md`/`shadow-xl` 대신 **반드시 `shadow-level-*`** 을 쓴다.
> DESIGN.md Don't: *"같은 화면에서 마케팅 100px pill과 6px 인앱 라운드를 섞지 마라"* — 상세페이지는 전역 `TopBar`를 숨기는 몰입형 화면(§11.5 옵션 B)이므로 버튼은 **마케팅 스케일(`rounded-pill`)로 통일**한다.

### 0.5 여백 토큰

`--spacing: 4px` 고정이라 Tailwind 숫자 스케일이 DESIGN.md와 1:1 대응한다. 별도 매핑 불필요.

`p-1`=4 · `p-2`=8 · `p-3`=12 · `p-4`=16 · `p-6`=24 · `p-8`=32 · `p-10`=40 · `p-12`=48 · `p-16`=64 · `p-24`=96

페이지 거터는 DESIGN.md 규정대로 **모바일 16px / 데스크톱 24px** → `px-4 sm:px-6`.

---

## 1. 컴포넌트 분해도

데이터 페칭 방식(서버 컴포넌트 fetch vs `useSpotDetail` 훅)은 이번 문서 범위 밖이다 — M3 BE 엔드포인트 확정 전까지 결정할 수 없다. 따라서 아래 컴포넌트는 전부 **`spot` 데이터를 props로만 받는 순수 프레젠테이션 컴포넌트**로 설계했다. `docs/spots-explore-plan.md`의 `SpotCard`/`SpotListView`가 택한 것과 동일한 전략이며, 이렇게 하면 페칭 전략이 나중에 뭘로 정해지든 컴포넌트를 그대로 재사용할 수 있다.

> **[v3] 라우트 경로 개정**: `app/(protected)/spots/[uid]/`가 아니라 **`app/spots/[uid]/`** 다. `(protected)` 라우트 그룹은 커밋 `788291c`에서 삭제됐고, `middleware.ts`의 `PROTECTED_PREFIXES`가 빈 배열이며, `GET /v1/explore/spots/{uid}`는 비로그인 접근이 가능한 공개 엔드포인트다. 아래 트리는 개정된 경로를 반영한다.

```
app/spots/[uid]/page.tsx                                     (server) — 라우팅 셸, fetch, notFound()
└── SpotDetailView                                           (server) — 섹션 조합만 담당
    ├── SpotDetailTopBar                                     (client) — 뒤로가기 + 공유, 스크롤 반응
    ├── SpotHeroImage                                        (server)
    ├── SpotHeaderSection                                    (server)
    │   ├── SpotChip × N                       [features/spots/SpotChip.tsx]
    │   ├── SpotFeeBadge                       [features/spots/SpotFeeBadge.tsx]
    │   └── SpotPetBadge                       [features/spots/SpotPetBadge.tsx]
    ├── SpotAlertBanner                                      (server) — 특이사항, null이면 렌더 안 함
    ├── SpotSpecGrid                                         (server)
    │   └── SpotSpecItem × 5                    (사이트유형/사이트수/평균면적/화로유형/이용요금)
    ├── SpotLocationSection                                  (server) — 지도 placeholder + 주소
    ├── SpotDescriptionSection                               (client) — 더보기 토글
    ├── SpotFacilitySection                                  (server)
    │   └── SpotFacilityRow × 3                 (편의/주변/렌탈)
    ├── SpotReviewPlaceholder                                (server)
    └── SpotActionBar                                        (client) — sticky bottom, 전화/웹사이트/예약
```

파일 경로는 두 계층으로 나눈다.

- **`apps/web/src/features/spots/`** (기존 `docs/spots-explore-plan.md`가 이미 이 경로에 `SpotCard.tsx` 등을 두기로 함) — 리스트 카드와 상세페이지가 **함께 재사용할 수 있는 원자 컴포넌트**만 둔다. `SpotChip`, `SpotFeeBadge`, `SpotPetBadge`가 여기 속한다. 리스트 카드에도 언젠가 유료/반려동물 배지가 필요해질 가능성이 높기 때문(현재 `RecommendedSpotCard`엔 없지만, 실제 API 스키마 — 부록 A 참고 — 에 `is_fee_required`/`is_pet_allowed`가 이미 존재).
- **`apps/web/src/features/spots/detail/`** (신규 하위 폴더) — 상세페이지 전용 조합 컴포넌트. 컴포넌트 수가 12개 이상이라 `features/spots/` 루트에 평평하게 두면 리스트/지도 관련 파일과 섞여 탐색성이 떨어진다.

### 1.1 Props 시그니처

> 필드명은 21개 필드 기획 문서 + `.openapi.json`에서 확인한 실제 ETL 스키마(`SpotBulkRow`, 부록 A)를 참고해 camelCase로 추정했다. **BE 스키마 확정 전까지는 추정치**이며, `packages/shared/types`에 정식 Zod 스키마가 생기면 그 타입을 `import`해서 대체해야 한다.

```ts
// apps/web/src/features/spots/SpotChip.tsx
interface SpotChipProps {
  label: string;
}

// apps/web/src/features/spots/SpotFeeBadge.tsx
interface SpotFeeBadgeProps {
  isFeeRequired: boolean | null; // null = 데이터 미확보
}

// apps/web/src/features/spots/SpotPetBadge.tsx
interface SpotPetBadgeProps {
  isPetAllowed: boolean | null;
}
```

```ts
// apps/web/src/features/spots/detail/SpotDetailTopBar.tsx
interface SpotDetailTopBarProps {
  backHref: string; // 리스트로 돌아가는 경로. 예: "/spots"
}

// apps/web/src/features/spots/detail/SpotHeroImage.tsx
interface SpotHeroImageProps {
  imageUrl: string | null; // null = 수집 예정(placeholder) 상태
  alt: string; // 장소명 기반 대체텍스트
}

// apps/web/src/features/spots/detail/SpotHeaderSection.tsx
interface SpotHeaderSectionProps {
  name: string;
  tagline: string | null;
  categories: string[]; // 빈 배열 가능
  tags: string[]; // 빈 배열 가능
  isFeeRequired: boolean | null;
  isPetAllowed: boolean | null;
}

// apps/web/src/features/spots/detail/SpotAlertBanner.tsx
interface SpotAlertBannerProps {
  note: string | null; // null/빈 문자열이면 컴포넌트가 null을 반환(섹션 자체 미노출)
}

// apps/web/src/features/spots/detail/SpotSpecGrid.tsx
interface SpotSpecGridProps {
  campSightType: string | null;
  unitCount: number | null;
  totalAreaM2: number | null; // unitCount와 함께 평균 면적 계산에 사용
  firePitType: string | null;
  categories: string[]; // [v3] campSightType과의 중복 비교에 필요 — 아래 주석 참고
}

// apps/web/src/features/spots/detail/SpotSpecItem.tsx
interface SpotSpecItemProps {
  label: string;
  value: string | null; // null = 이 스팟에 한해 데이터 결측 (섹션 7의 "결측" 케이스)
  isComingSoon?: boolean; // true = 시스템 전체 미수집 필드 (섹션 7의 "수집예정" 케이스). 현재는 이용요금 하나만 해당
}

// apps/web/src/features/spots/detail/SpotLocationSection.tsx
interface SpotLocationSectionProps {
  latitude: number | null;
  longitude: number | null;
  address: string | null;
  addressDetail: string | null;
}

// apps/web/src/features/spots/detail/SpotDescriptionSection.tsx
interface SpotDescriptionSectionProps {
  description: string | null;
}

// apps/web/src/features/spots/detail/SpotFacilitySection.tsx
interface SpotFacilitySectionProps {
  amenities: string[];
  nearbyFacilities: string[];
  equipmentRental: string[];
}

// apps/web/src/features/spots/detail/SpotFacilityRow.tsx
interface SpotFacilityRowProps {
  label: string; // "편의시설" | "주변시설" | "렌탈장비"
  items: string[]; // 빈 배열이면 "정보 없음" 폴백 렌더
}

// apps/web/src/features/spots/detail/SpotActionBar.tsx
interface SpotActionBarProps {
  phone: string | null;
  websiteUrl: string | null;
  bookingUrl: string | null;
}

// apps/web/src/features/spots/detail/SpotReviewPlaceholder.tsx
// props 없음 — 리뷰 기능이 출시되기 전까지 항상 동일한 내용을 렌더

// apps/web/src/features/spots/detail/SpotDetailSkeleton.tsx
// props 없음 — 로딩 상태 전용, 실제 데이터 구조와 무관하게 골격만 그림
```

`평균 면적 = totalAreaM2 / unitCount` 계산은 컴포넌트 내부에 인라인으로 두지 말고 `packages/shared/utils`에 순수 함수로 추출하는 걸 권장한다(예: `calcAverageSiteArea(totalAreaM2, unitCount): number | null`). 리스트 카드에서도 같은 파생값이 필요해질 가능성이 있고, `unitCount`가 0이거나 `null`인 나눗셈 예외 처리를 한 곳에서만 관리하기 위함이다. 다만 이번 문서 범위에서 강제하는 사항은 아니고 구현 시 판단 사항으로 남긴다.

---

## 2. 각 섹션 시각 스펙

### 2.1 `SpotDetailTopBar`

기획 초안 와이어프레임은 "← 상태바 ⇧"로 이미지 위에 얹힌 투명 탑바를 보여준다. **이건 현재 아키텍처와 충돌한다** — `TopBar.tsx`가 `app/layout.tsx`에서 전역으로 항상 렌더링되고, 상세페이지도 예외가 아니다. 이 충돌과 두 가지 대안은 섹션 11.5에서 다룬다. 여기서는 채택안(옵션 B) 기준으로 스펙을 적는다.

- 위치: 히어로 이미지 위에 `absolute` 오버레이. 스크롤이 히어로 이미지 높이를 넘어가면 `bg-canvas`로 전환 + `border-b border-hairline` 추가(기존 `TopBar`의 sticky 규칙과 동일한 셸을 재사용).
- 버튼 2개: 뒤로가기(좌), 공유(우). 둘 다 `rounded-full`, 반투명 상태에서는 `bg-black/30 text-white`(스크림 — 토큰이 아니라 이미지 위 가독성 확보용 중립 오버레이), 솔리드 전환 후에는 `text-body`.
- 터치 타겟: `h-10 w-10` (기존 `TopBar`의 햄버거 버튼과 동일 크기, 44px 하한 근접).
- 높이: `h-16`(64px) — DESIGN.md `nav-bar` height 64px. 전역 `TopBar`도 같은 값으로 마이그레이션됐으므로 전환 시점에 레이아웃 점프가 없다.

> **[v3] 스크롤 감지 계약** — 전환 시점은 `scroll` 이벤트로 높이를 계산하지 않고, `SpotHeroImage`가 심어둔 **`data-spot-hero` 속성을 `IntersectionObserver`로 관찰**해서 잡는다(`rootMargin: "-64px 0px 0px 0px"` — 탑바 높이만큼 루트 상단을 줄여 히어로가 탑바 뒤로 완전히 숨는 순간 전환). 히어로 비율이 브레이크포인트마다 다르므로(`aspect-[4/3]` / `sm:aspect-[16/9]`), 스크롤 임계값을 JS로 계산하면 CSS 브레이크포인트를 중복 구현하게 된다. 이 방식은 비율이 바뀌어도 JS를 손대지 않는다.
>
> `data-spot-hero`는 `SpotHeroImage`와 `SpotDetailTopBar` 사이의 계약이다 — 히어로 마크업을 교체할 때 이 속성을 유지해야 한다.
>
> 위치는 `absolute`가 아니라 **`fixed inset-x-0 top-0`** 이다. 히어로를 지난 뒤에도 탑바가 화면에 남아야 하기 때문.

### 2.2 `SpotHeroImage`

- 컨테이너: `relative aspect-[4/3] sm:aspect-[16/9]` — 모바일은 카드형(`RecommendedSpotCard`와 동일 비율), 데스크톱은 와이드.
- 배경: `bg-canvas-soft-2`(placeholder 상태), 실제 이미지가 들어오면 `object-cover`.
- placeholder 아이콘: `RecommendedSpotCard`가 이미 쓰는 산 모양 SVG 아이콘을 그대로 재사용(`text-mute`) — 이미지 placeholder 시각 언어를 앱 전체에서 하나로 통일.
- 라운드 없음(`rounded-none`) — 화면 폭 전체를 채우는 히어로이므로 카드처럼 라운드 처리하지 않는다. DESIGN.md `{rounded.none}` 용도 정의(*"Full-bleed hero / footer bands"*)와 정확히 일치.

### 2.3 `SpotHeaderSection`

- 컨테이너 패딩: `px-4 py-4 sm:px-6` — DESIGN.md 페이지 거터(모바일 16px / 데스크톱 24px). 홈 화면도 같은 값으로 마이그레이션됐다.
- 칩 로우(카테고리+태그): `flex gap-3 overflow-x-auto [scrollbar-width:none]` — `HomeSearchBar`의 카테고리 칩 가로 스크롤 패턴 그대로 재사용. 칩 순서는 **카테고리 먼저, 태그 나중**(스크롤 안 해도 보이는 왼쪽 영역에 더 판단 중요한 정보 배치).
- 장소명: `text-display-md sm:text-display-lg text-ink`, `line-clamp-2`. (v1의 `font-bold`는 DESIGN.md 상한 600 규칙 위반이라 제거 — display 토큰에 weight 600이 이미 포함돼 있다.)
- 한줄설명(`tagline`): `mt-2 text-body-md text-body`. `null`이면 렌더 안 함.
- 배지 로우(유료/반려동물): `mt-2 flex flex-wrap gap-2` — 배지 상세 스펙은 섹션 3.

### 2.4 `SpotAlertBanner`

섹션 6에서 별도로 상세히 다룬다.

### 2.5 `SpotSpecGrid`

- 그리드: `grid grid-cols-2 gap-3 sm:grid-cols-4` — 모바일 2×2(와이어프레임과 동일), `sm`(**600px**) 이상에서 4열 한 줄.
- 각 `SpotSpecItem` 셀: `rounded-md bg-canvas-soft p-3` (DESIGN.md `card-soft` 크롬).
  - 라벨: `text-caption text-body`
  - 값: `mt-1 text-body-sm-strong text-ink` — 값이 `null`(결측)이면 `text-body-sm text-body`로 톤다운(굵기만 낮춤), 텍스트는 "정보 없음".
  - `isComingSoon`(이용요금 전용): 셀 크롬은 동일하게 유지하고 값 자리만 `text-caption text-body`, "가격 정보 준비 중"으로 표기. 근거는 섹션 5.2-B.
  - ⚠️ **결측·준비중 표기에 `text-mute`를 쓰지 않는다** — 라이트 모드 대비비 3.54:1로 AA 미달(§10). 톤다운은 색이 아니라 **굵기**(`-strong` 제거)로만 만든다.
- 5번째 항목(이용요금)은 4열 그리드에서 마지막 칸에 배치되며, `sm:col-span-1`로 나머지와 동일 폭 유지(특별 취급하지 않아 "여기만 비어있다"는 인상을 줄인다).

> **[v3] 중복 방어와 열 수** — 셀은 총 **5개**(사이트유형/사이트수/평균면적/화로유형/이용요금)다. 따라서 `sm:grid-cols-4`에서 기본 배치는 4+1로 **두 줄**이며, 위 "4열 한 줄" 표현은 셀 4개 기준의 서술이다.
>
> `categories`에 `campSightType`과 정규화 후 동일한 값이 있으면 사이트 유형 셀을 생략한다(의사결정 기록 §11.1). 이때 남는 셀은 **3개가 아니라 4개**이므로 `sm:grid-cols-4`를 그대로 두면 정확히 한 줄이 된다 — 열 수를 3으로 줄이면 오히려 3+1로 깨진다. 의사결정 기록 §11.1의 *"나머지 3개 항목만 보여주는 3열 그리드로 접는다"* 는 이용요금 셀(§5.2-B에서 추가됨)을 계산에 넣지 않은 서술이며, 구현은 **열 수를 바꾸지 않고 셀만 생략**한다.
>
> 비교는 `trim()` + `toLowerCase()` 정규화 후 수행한다. `SpotSpecGrid`가 `categories`를 prop 으로 받는 이유가 이것이다(§1.1).

### 2.6 `SpotActionBar`

섹션 8에서 상세히 다룬다.

### 2.7 `SpotLocationSection`

- 섹션 제목: `text-display-sm text-ink` + `mt-12`(섹션 간 상단 여백, `SpotCarousel`의 `mt-12 sm:mt-16` 규칙과 통일).
- 지도 영역: 실제 SDK 미정 상태이므로 `docs/spots-explore-plan.md`의 `SpotMapView`가 이미 채택한 placeholder 톤("지도 보기는 준비 중입니다") 문구·톤을 그대로 재사용한다 — 같은 앱 안에서 지도 placeholder가 두 가지 다른 말투로 보이면 안 되기 때문. 컨테이너: `aspect-[4/3] sm:aspect-[21/9] rounded-md bg-canvas-soft-2`, 중앙에 안내 텍스트 + 좌표가 있으면 "위도, 경도" 텍스트를 보조로 노출.
- 주소: `mt-3 text-body-md text-body`, 상세주소는 `text-body-sm text-body`로 **크기만** 한 단계 낮춤.

> **[v3] 지도 비율에 `sm:aspect-[21/9]` 추가** — `aspect-[4/3]`은 모바일 폭 전제다. 이 페이지는 `lg`(1200px) 전까지 컨테이너 제약이 없어 전체 폭을 쓰므로, 넓은 화면에서 4:3을 유지하면 높이 800px에 달하는 빈 상자가 된다(좌표가 없는 스팟은 화면 한 장을 안내 문구 한 줄로 채우게 된다). 실제 지도 SDK가 붙어도 와이드 뷰가 자연스러우므로 `sm`부터 21:9로 전환한다.
>
> **[v3] 상세주소·좌표에 `text-mute`를 쓰지 않는다** — 위 원문은 상세주소에 `text-mute`를 지정했으나, 상세주소와 좌표는 모두 의미 전달이 필요한 텍스트라 §10의 "`mute`는 본문 텍스트 금지" 제약에 걸린다. 특히 좌표는 GPS 앱에 옮겨 적을 수 있는 실제 데이터다. 톤다운은 색이 아니라 **크기**(`text-body-sm` / `text-caption`)로 만든다. 대비비 근거는 §10 개정 표 참고.
- `latitude`/`longitude`가 `null`이면 지도 영역 안내 문구를 "위치 정보가 아직 등록되지 않았어요"로 교체하고 핀 표시 생략(주소 텍스트만 노출) — 섹션 8.3 결측 테이블 참고.

### 2.8 `SpotDescriptionSection`

- 본문: `text-body-md text-body leading-relaxed`, 접힌 상태 `line-clamp-3`.
- 더보기 버튼: `mt-2 text-body-sm-strong text-link` — **v2 개정**: DESIGN.md `link-inline` 토큰을 채택한다. v1은 "그레이 + 밑줄"로 처리했으나, 이는 `link` 토큰이 배선돼 있지 않다는 전제에 기댄 우회였다. DESIGN.md 우선 정책 하에서는 클릭 가능한 인라인 어포던스에 `{colors.link}`를 쓰는 것이 정의된 어휘다(§11.4 참고).
- `description`이 `null`이면 섹션 전체 미노출.

### 2.9 `SpotFacilitySection` / `SpotFacilityRow`

- 각 로우: 라벨(`text-body-sm-strong text-ink`) + 아이템 리스트.
- 아이템 칩: `shrink-0 rounded-full bg-canvas-soft px-2 py-0.5 text-caption text-body` — DESIGN.md `badge-secondary` 스펙. `RecommendedSpotCard`의 시설 태그도 같은 값으로 마이그레이션됐다.
- 아이템이 4개 넘어가는 모바일 대응: `flex gap-2 overflow-x-auto [scrollbar-width:none]`.
- `items`가 빈 배열이면 칩 대신 `text-caption text-mute`로 "정보 없음" 한 줄만 노출.

### 2.10 `SpotReviewPlaceholder`

섹션 7에서 상세히 다룬다.

---

## 3. 배지/칩 시스템

DESIGN.md가 정의하는 칩/배지 원시 컴포넌트는 두 가지다.

1. **아웃라인 필 칩** — `rounded-full border border-hairline-strong px-3 py-1.5 text-body-sm text-body` (`HomeSearchBar` 카테고리 칩이 이 스펙으로 마이그레이션됨)
2. **`badge-secondary`** — `rounded-full bg-canvas-soft px-2 py-0.5 text-caption text-body` (DESIGN.md 정의 원문 그대로. `RecommendedSpotCard` 시설 태그가 이 스펙)

이 문서는 여기에 **세 번째, 유료 여부 전용 강조 배지 하나만** 추가한다. 근거: DESIGN.md Don't *"여섯 번째 액센트 색을 도입하지 마라"* 를 지키려면 유료/반려동물에 새 색을 쓸 수 없다. 하지만 유료 여부는 페르소나 A의 예산 판단에 직결되는 정보라 시각적 위계는 필요하다 — 색이 아니라 **명암 반전**(solid vs outline)으로 위계를 만든다. DESIGN.md의 `button-primary`(solid ink) / `button-secondary`(outline) 관계, 그리고 `pricing-card-featured`의 폴라리티 플립과 동일한 발상이다.

| 배지 | 컴포넌트 | 값 | 클래스 | 근거 |
|---|---|---|---|---|
| 카테고리 | `SpotChip` | 배열, N개 | 아웃라인 필 칩 (위 1) | 필터링용 메타데이터, 기존 카테고리 칩과 톤 통일 |
| 태그 | `SpotChip` | 배열, N개 | 아웃라인 필 칩 (위 1), 카테고리와 시각적으로 구분 없음 | 둘 다 "분류 메타데이터"라는 같은 층위 정보라 구분 불필요 — 순서로만 우선순위 표현(§2.3) |
| 유료 여부 | `SpotFeeBadge` | `true`(유료) | `rounded-full bg-primary px-3 py-1 text-caption font-medium text-on-primary` (solid) | 지출이 발생한다는 사실을 강조. 다크모드에서 자동 폴라리티 플립(흰 배지 / 검은 글자) |
| | | `false`(무료) | `rounded-full border border-hairline-strong px-3 py-1 text-caption text-body` (outline) | 무료는 "특이사항 없음"에 가까운 기본값 취급 |
| | | `null` | 배지 자체를 렌더하지 않음 | 색상 없는 애매한 3번째 상태를 만들지 않기 위해. "정보 없음" 배지보다 아예 안 보이는 게 스캔 레이어에 노이즈를 덜 준다 |
| 반려동물 허용 | `SpotPetBadge` | `true` | 아웃라인 필 칩 + 발자국 아이콘(`aria-hidden`) + "반려동물 동반 가능" | 색이 아니라 아이콘 + 텍스트로 의미 전달(접근성 §10과 연결) |
| | | `false`/`null` | 배지 미노출 | "반려동물 불가"를 굳이 강조 배지로 보여줄 필요가 낮고, 없음/false를 시각적으로 구분할 근거(데이터도, 니즈도) 없음 |

### 3.1 4종 이상 나열 시 wrap/우선순위 규칙

기획 초안 와이어프레임이 이미 배지를 두 로우로 분리해 놨다(칩 로우: 카테고리+태그 / 배지 로우: 유료+반려동물). 이 분리 덕분에 실제로 한 로우에 4종 이상이 몰릴 일은 구조적으로 없다. 각 로우 내부 규칙만 정의하면 된다.

- **칩 로우(카테고리+태그)**: 개수 제한 없음, `overflow-x-auto` 가로 스크롤(랩 대신 스크롤 채택 — 헤더 영역에서 세로 공간을 잡아먹지 않기 위해, `HomeSearchBar` 카테고리 칩과 동일 패턴이라 유저가 이미 학습한 인터랙션). 순서: 카테고리 → 태그.
- **배지 로우(유료+반려동물)**: 최대 2개 고정이라 `flex flex-wrap gap-2`면 충분, 스크롤 불필요.
- 새로운 배지 종류가 향후 추가되면(예: 전기 사용 가능 여부) 배지 로우도 3개가 되어 랩이 필요해질 수 있다 — `flex-wrap` 이미 적용돼 있으므로 별도 대응 불필요.

---

## 4. ⚠ 특이사항 경고 배너

### 4.1 심각도 구분 여부 — 하지 않는다

`특이사항`은 자유 텍스트 단일 필드다(부록 A의 `features` 필드 추정과 일치 — 문자열 하나, enum 아님). 구조화된 심각도 필드가 없는 상태에서 "정보/주의/경고" 3단계를 UI가 임의로 나누려면 클라이언트 쪽에서 텍스트 키워드 매칭("운영중단", "폐쇄" 포함 여부 등)을 해야 하는데, 이건 깨지기 쉬운 안티패턴이다(오탐·미탐, 다국어/표기 변형에 취약). **이번 범위에서는 단일 톤 배너로 설계**하고, 심각도 분리가 정말 필요하면 BE에 구조화된 `severity` enum 필드 추가를 요청하는 걸 후속 논의로 남긴다(섹션 12).

### 4.2 톤 — DESIGN.md `warning` 팔레트 채택 **(v2 개정 — v1 결정을 뒤집음)**

> **v1에서 뒤집힌 판단**: v1은 DESIGN.md의 `warning`(`#f5a623`) 계열을 **의도적으로 거부**하고 홈 화면이 쓰던 Tailwind `yellow-*`를 재사용했다. 논거는 둘이었다 — (1) 앱에 "두 벌의 경고색 체계"가 생긴다, (2) `#f5a623`은 Tailwind 기본 팔레트에 정확히 대응하는 클래스가 없어 임의값 하드코딩(`bg-[#f5a623]`)이 필요하다.
>
> **두 논거 모두 무효화됐다.** (1) 홈 화면의 `yellow-*`가 `warning-*` 토큰으로 이관되면서 경고색 체계는 **한 벌뿐이다.** (2) `warning`/`warning-soft`/`warning-deep`이 `globals.css`에 정식 배선돼 `bg-warning-soft` 같은 표준 유틸로 쓸 수 있으므로 하드코딩이 필요 없다.

기획 초안이 이 필드의 성격을 규정한 대로 — *"법적 야영/취사 정보 필드 부재로, 당분간 운영 관련 리스크 정보를 담는 사실상 유일한 필드"* — 이 배너는 홈 화면 `legalStatus` 배지의 "리스크 신호" 역할을 임시로 대신한다. 그 배지가 이제 `warning` 토큰을 쓰므로, 배너도 같은 토큰을 쓰면 앱 전체에서 앰버가 등장하는 자리가 "확인이 필요한 리스크"라는 의미로 단일하게 수렴한다.

- 컨테이너: `mx-4 mt-3 flex gap-2 rounded-md border border-warning bg-warning-soft px-3 py-2.5 sm:mx-6`
- 아이콘: 삼각형 경고 아이콘(`⚠` 유니코드 대신 인라인 SVG), `text-warning-deep`, `aria-hidden`.
- 텍스트: `text-body-sm-strong text-warning-deep`.
- 위치: 헤더 섹션과 스펙 그리드 사이(기획 초안 순서 그대로).
- `note`가 `null`이거나 빈 문자열이면 `SpotAlertBanner`는 `null`을 반환 — 섹션 자체가 사라지고 레이아웃 상 빈 공간이 남지 않는다.
- 다크모드: 배경 `#3b2410` + 텍스트 `#f5a623`으로 자동 반전 — v1 §10이 고지했던 "경고 배너가 다크모드에서 대비가 어색해진다"는 한계는 해소됐다.

### 4.3 접근성

`role="alert"` 또는 `role="status"` 중 어느 걸 쓸지는 섹션 10에서 다룬다.

---

## 5. "수집 예정(placeholder)" 상태의 시각 언어

기획 초안의 점선 박스 처리를 **비판적으로 재검토**한다.

### 5.1 왜 점선 박스가 최선이 아닌가

점선 테두리는 브라우저 개발자 도구, 와이어프레임 툴, 레이아웃 디버깅 오버레이가 관용적으로 "여긴 아직 확정 안 된 자리"를 표시할 때 쓰는 시각 문법이다. 실제 서비스 화면에 그대로 노출하면 유저에게 "이 화면이 아직 완성 안 됐다 / 버그다"라는 인상을 준다 — 기획 초안이 의도한 "데이터 파이프라인 확장 대기 중"이라는 의미와 반대로 읽힐 위험이 있다. 기획 문서의 점선 표기 자체는 스케치 단계의 표기법으로는 합리적이지만, 최종 시각 스펙으로 그대로 가져가는 건 재고가 필요하다.

### 5.2 대안 — 필드 성격에 따라 다르게 처리

3개 필드(리뷰, 대표 이미지, 이용요금)는 성격이 다르다.

- **리뷰, 대표 이미지**: 페이지 안에서 자기 공간을 크게 차지하는 **독립 섹션**이다.
- **이용요금**: 스펙 그리드 안의 **셀 하나**다. 그리드 안에 점선 박스 하나만 다른 모양으로 끼워 넣으면 그리드 리듬이 깨지고 오히려 더 눈에 띄게 "고장난 느낌"을 준다.

이 둘을 하나의 시각 언어로 묶지 않고 컨텍스트에 맞게 나눈다.

**(A) 독립 섹션형 — 리뷰 / 대표 이미지**

점선 대신 **솔리드 소프트 카드**로 처리한다. 앱스토어·OTT 서비스가 "출시 예정" 콘텐츠를 보여줄 때 쓰는 방식과 같은 발상: 채워지지 않은 게 아니라 "예고"로 읽히게 한다.

- 컨테이너: `rounded-lg bg-canvas-soft p-6 text-center shadow-level-1` — 점선 대신 솔리드 배경 + DESIGN.md Level 1 inset hairline 링으로 "의도된 화면"임을 표시. (DESIGN.md `ex-empty-state-card` 스펙과 동일: canvas-soft 배경 + 넉넉한 패딩.)
- 아이콘: 섹션 주제에 맞는 모노톤 아이콘(리뷰 = 말풍선+별, 이미지 = `RecommendedSpotCard`가 쓰는 것과 같은 산 모양 아이콘), `text-mute`, `h-8 w-8`.
- 카피: 라벨성 문구("수집 예정") 대신 **제품 톤의 짧은 문장**으로. 예:
  - 리뷰: "아직 등록된 후기가 없어요. 준비되는 대로 가장 먼저 보여드릴게요."
  - 대표 이미지: "대표 사진은 준비 중이에요."
- 대표 이미지 placeholder는 `SpotHeroImage`가 이미 이 상태를 기본값으로 담당하므로(섹션 2.2), `SpotReviewPlaceholder`만 신규로 이 카드 패턴을 쓴다.
- **레이아웃 안정성**: 카드의 `min-height`를 실제 리뷰 리스트가 들어왔을 때의 예상 첫 화면 높이(리뷰 카드 1~2개 분량, 대략 `min-h-32`)로 고정해둔다. 데이터가 채워지면 `SpotReviewPlaceholder`를 렌더하는 조건(`reviewCount === 0`)만 바뀌고 그 자리에 실제 리뷰 리스트 컴포넌트가 들어서는 구조 — placeholder와 실 데이터 컴포넌트는 같은 부모 슬롯을 공유하되 서로 다른 컴포넌트이므로 "UI 안 바꾸고 값만 채운다"는 제약은 "같은 위치·같은 섹션 타이틀·비슷한 카드 폭을 유지한다"는 의미로 해석했다 — 리뷰처럼 아예 새 컴포넌트 트리(별점, 리뷰 카드 리스트)가 필요한 필드는 값만 갈아끼우는 게 원천적으로 불가능하기 때문. 이용요금처럼 스펙 그리드의 텍스트 한 줄인 필드만 "완전히 동일한 셀에 값만 대체" 조건을 만족시킬 수 있다.

**(B) 그리드 내장형 — 이용요금**

`SpotSpecItem`의 크롬(배경, 라운드, 패딩)은 다른 4개 셀과 **완전히 동일하게 유지**하고, 값 텍스트만 바꾼다.

- 값 자리: `text-caption text-mute` + "가격 정보 준비 중"
- 부동산·여행 예약 서비스가 가격 미정 매물에 흔히 쓰는 "가격 문의"류 관용구와 같은 자리에 놓이는 문구라 사용자에게 낯설지 않다.
- 값만 문자열로 존재하고(`isComingSoon: true`), 실제 요금 데이터가 들어오면 `isComingSoon`을 `false`로, `value`에 "1박 20,000원" 같은 문자열을 채우기만 하면 된다 — 셀 컴포넌트, 그리드 레이아웃, 폰트 크기 전부 변경 없음. 세 placeholder 필드 중 이 제약을 가장 정직하게 만족하는 케이스다.

---

## 6. (배지 시스템에 통합됨)

배지 관련 세부 스펙은 섹션 3에 전부 기술했다.

---

## 7. 액션바 / Primary CTA

### 7.1 구성

- Primary: **예약 페이지로 이동** (`bookingUrl`) — 외부 이동임을 명확히 하는 라벨 + 외부 링크 아이콘(기획 초안 문구 그대로).
- Secondary: 전화(`phone`), 웹사이트(`websiteUrl`) — 아이콘 버튼.

### 7.2 시각 스펙

- 컨테이너: `fixed inset-x-0 bottom-0 z-30 flex items-center gap-3 border-t border-hairline bg-canvas px-4 pt-3 sm:px-6`
- Safe area: `[padding-bottom:calc(var(--spacing)*3+env(safe-area-inset-bottom))]` — Tailwind 유틸리티만으론 `env()`를 못 다루므로 임의 값 프로퍼티로 처리. `--spacing`이 4px 고정이 되면서 이 계산이 정확히 `12px + safe-area`가 된다(fluid 시절에는 값이 흔들렸다). **신규 처리 필요** — 기존 컴포넌트 중 `env(safe-area-inset-*)`을 쓰는 곳이 없으므로 이번이 최초 도입.
- 버튼 높이: `h-12`(48px) — DESIGN.md `button-primary` 마케팅 스케일. 44px 터치 타겟 하한 충족.
- Primary 버튼: `flex-1 rounded-pill bg-primary text-on-primary text-button-lg` — DESIGN.md `button-primary` 그대로. 폭을 넓게 잡아 "이게 이 화면의 목적지"임을 강조. 다크모드에서 흰 pill / 검은 글자로 자동 폴라리티 플립.
- Secondary 버튼(전화/웹사이트): 정사각형 아이콘 버튼 `h-12 w-12 rounded-full border border-hairline text-body` — DESIGN.md `icon-button-circular` 스펙. 라벨 없이 아이콘만, 대신 `aria-label` 필수(섹션 10).
- 항상 화면 하단에 고정(sticky 아님, **fixed**) — 스크롤 중에도 항상 노출해야 하는 게 기획 초안의 "Primary CTA" 취지에 맞다. 스크롤에 따라 나타났다 사라지는 방식은 채택하지 않는다 — 예약이라는 행동이 페이지 어디서든 즉시 가능해야 하는 핵심 전환 지점이기 때문(스크롤 의존 노출은 오히려 전환율을 낮출 위험).
- 페이지 최하단 콘텐츠(`SpotReviewPlaceholder`)에는 액션바 높이만큼 `pb-20`(대략 액션바 실제 높이 + safe area 여유분) 여백을 둬서 콘텐츠가 액션바에 가려지지 않게 한다.

### 7.3 결측 대응

`bookingUrl`, `phone`, `websiteUrl` 셋 다 `null`일 수 있다. 규칙:

| 상태 | 처리 |
|---|---|
| `bookingUrl` 있음 | Primary 버튼 정상 노출 |
| `bookingUrl` 없음, `phone`/`websiteUrl` 중 하나 이상 있음 | Primary 자리에 있는 것 중 우선순위가 높은 것(전화 > 웹사이트)을 승격해 Primary 스타일로 노출. 문구는 "전화 문의" 등으로 대체 |
| 셋 다 없음 | 액션바 자체를 렌더하지 않음. 이 경우 페이지 최하단 여백(`pb-20`)도 제거해 불필요한 빈 공간이 남지 않게 한다 |

이 우선순위 승격 규칙은 기획 초안에 없는 내용이라 **추정/제안**이다 — 액션바가 완전히 비어버리는 스팟이 실제 데이터에서 몇 % 나올지에 따라 재검토가 필요하다(부록 A 참고: BE 스키마 상 세 필드 모두 nullable).

---

## 8. 상태 정의

### 8.1 로딩 — `SpotDetailSkeleton`

전체 페이지 골격을 중립 블록으로 미러링한다. `bg-canvas-soft-2 animate-pulse` 블록으로 히어로 이미지, 제목 두 줄, 배지 로우, 스펙 그리드 4칸, 액션바 자리를 채운다. 액션바 자리는 스켈레톤 상태에서도 `fixed bottom-0`에 동일한 높이로 유지 — 로딩이 끝나는 순간 액션바가 갑자기 나타나면서 레이아웃이 튀는 걸 방지.

> **[v3] `loading.tsx`로 연결하지 말 것 — 404 상태 코드가 깨진다.**
>
> `app/spots/[uid]/loading.tsx`를 두면 세그먼트 전체가 Suspense 경계가 되어 응답이 스트리밍된다(`Transfer-Encoding: chunked`). 셸이 먼저 flush 되면서 HTTP 상태가 **200으로 확정된 뒤에** 페이지 함수가 `notFound()`에 도달하므로, 존재하지 않는 `uid`에 404 페이지 내용이 200 상태로 나간다. `next build && next start` 프로덕션 빌드에서도 동일하게 재현되는 동작이다(dev 전용 아님).
>
> | 조건 | `/spots/{없는uid}` 상태 |
> |---|---|
> | `loading.tsx` 있음 | 200 (본문은 404 페이지) |
> | `loading.tsx` 없음 | 404 |
>
> 따라서 현재 구현은 `loading.tsx`를 두지 않으며, `SpotDetailSkeleton`은 **아직 어디에도 연결돼 있지 않다.** 실제 데이터 페칭을 도입할 때 `notFound()` 판정 **이후**의 하위 트리만 `<Suspense fallback={<SpotDetailSkeleton />}>`로 감싸면 스트리밍과 404 상태 코드를 둘 다 얻을 수 있다.

### 8.2 에러

fetch 실패(네트워크 오류, 5xx) 시 전체 페이지를 대체하는 에러 상태. 최소 구성: 안내 텍스트(`text-body-md text-body`, "장소 정보를 불러오지 못했어요") + 재시도 버튼(DESIGN.md `button-secondary`: `h-12 rounded-pill border border-hairline-strong bg-canvas text-button-lg text-ink`) + `SpotDetailTopBar`(뒤로가기)는 유지 — 에러 상태에서도 리스트로 돌아갈 수 있어야 한다.

404(존재하지 않는 `uid`)는 Next.js `notFound()`로 처리하고 앱 전역 `not-found.tsx`에 위임 — 이 문서 범위 밖(전역 컴포넌트).

### 8.3 부분 결측 — 필드별 폴백표

21개 필드 중 상당수가 스팟별로 `null`일 수 있다는 전제 하에, "시스템 전체 미수집"(placeholder, 섹션 5)과 "이 스팟만 결측"(단순 폴백)을 구분한다.

| 필드 | 결측 시 처리 | 유형 |
|---|---|---|
| 대표 이미지 | 산 모양 아이콘 placeholder (섹션 2.2) | 시스템 미수집 |
| 리뷰/평점 | "준비 중" 카드 (섹션 5.2-A) | 시스템 미수집 |
| 이용요금 | "가격 정보 준비 중" (섹션 5.2-B) | 시스템 미수집 |
| 한줄설명(`tagline`) | 헤더에서 해당 줄 렌더 생략 | 스팟별 결측 |
| 카테고리/태그 | 칩 로우 자체 미노출(빈 배열이면 `overflow-x-auto` 컨테이너도 렌더 안 함) | 스팟별 결측 |
| 유료 여부 | 배지 미노출 (섹션 3) | 스팟별 결측 |
| 반려동물 허용 | 배지 미노출 (섹션 3) | 스팟별 결측 |
| 사이트 유형/사이트 수/화로 유형 | `SpotSpecItem` 값에 "정보 없음"(`text-mute`) | 스팟별 결측 |
| 사이트 총 면적 | `unitCount`도 없으면 평균 면적 계산 불가 → "정보 없음". `unitCount`만 있고 면적이 없어도 동일 | 스팟별 결측 |
| 전화/웹사이트/예약 링크 | 액션바 규칙 (섹션 7.3) | 스팟별 결측 |
| 위도/경도 | 지도 placeholder 텍스트 교체 (섹션 2.7) | 스팟별 결측 |
| 주소/상세주소 | 주소 없으면 위치 섹션 전체를 "위치 정보가 등록되지 않았어요" 한 줄로 축소, 상세주소만 없으면 주소만 노출 | 스팟별 결측 |
| 상세설명 | 섹션 전체 미노출 | 스팟별 결측 |
| 편의/주변/렌탈 | 각 로우별 "정보 없음" (섹션 2.9) | 스팟별 결측 |
| 특이사항 | 배너 전체 미노출 (섹션 4.2) | 스팟별 결측(없는 게 정상 — 특이사항 없는 스팟이 더 많을 것) |

---

## 9. 반응형

> **v2 개정**: fluid 토큰이 폐기되면서 "연속 스케일이 크기 변화를 커버한다"는 v1 전제가 사라졌다. 이제 **모든 크기 변화는 브레이크포인트로만 처리**한다. 또한 브레이크포인트 값 자체가 DESIGN.md 기준으로 바뀌었다(§0.1-3).

DESIGN.md 브레이크포인트 표를 그대로 따른다.

| 구간 | Tailwind 접두사 | 처리 |
|---|---|---|
| < 600px (Mobile, 기준 380px) | (기본) | 단일 컬럼. 3-up 그리드는 1-up으로, 칩 로우는 가로 스크롤 |
| 600~959px (Tablet) | `sm:` | 히어로 비율 `sm:aspect-[16/9]`, 스펙 그리드 `sm:grid-cols-4`, 거터 24px, 장소명 `sm:text-display-lg` |
| 960~1199px (Desktop) | `md:` | 시설 리스트 스크롤 → wrap 전환(§9.2). 구조는 여전히 단일 컬럼 |
| ≥ 1200px (Wide) | `lg:` | **2컬럼 재배치**: 좌측(약 62%) 히어로+헤더+경고배너+스펙그리드+본문+시설, 우측(약 38%) sticky 사이드 패널 |
| ≥ 1400px (Ultra) | `xl:` | 콘텐츠는 `max-w-page`(1400px)에서 중앙 정렬 고정. 밴드만 edge-to-edge |

> v1은 2컬럼 전환을 `lg:`(당시 1024px)로 잡았다. 새 스케일에서 `lg:`는 1200px이므로 **전환 지점이 1024→1200px로 늦춰진다.** DESIGN.md의 Wide 구간 정의와 일치하며, 1024~1199px 태블릿 가로 모드에서 사이드 패널이 눌리는 문제를 오히려 줄인다.

### 9.1 데스크톱 2컬럼 상세

- 컨테이너: `lg:mx-auto lg:flex lg:max-w-5xl lg:gap-8 lg:px-6`
- 우측 사이드 패널: `lg:sticky lg:top-20 lg:w-80 lg:shrink-0 lg:rounded-lg lg:p-6 lg:shadow-level-4` — DESIGN.md `card-marketing-large` 크롬(rounded.lg + Level 4 float stack). 보더 대신 Level 4 섀도에 포함된 inset hairline 링이 경계를 만든다. `SpotActionBar`가 모바일의 `fixed bottom-0` 바가 아니라 이 패널 안의 일반 카드로 렌더링된다. `SpotActionBar`는 `lg:` 여부에 따라 컨테이너 스타일만 다르게 받는 게 아니라, 두 컨텍스트(모바일 fixed bar / 데스크톱 sticky 패널)를 하나의 컴포넌트가 `className`/레이아웃 prop으로 분기하기보다 **부모(`SpotDetailView`)가 반응형 클래스로 감싸는 래퍼 두 개(모바일용 `fixed` 래퍼, 데스크톱용 `lg:sticky` 래퍼) 안에 같은 `SpotActionBar` 내부 버튼 마크업을 배치**하는 방식을 권장 — 컴포넌트 로직은 그대로 두고 바깥 위치 지정만 반응형으로 분기. 위치 지정 로직이 컴포넌트 내부에 섞이면 컴포넌트 자체가 두 가지 책임(버튼 렌더링 + 반응형 포지셔닝)을 갖게 되어 복잡도가 올라간다.
- 데스크톱에서는 `pb-20`(모바일 전용 액션바 여백)이 필요 없으므로 `lg:pb-0`으로 해제.
- 사이드 패널에 이용요금(수집 예정) 요약을 함께 노출할지는 고려했으나 채택하지 않았다 — 스펙 그리드에 이미 있고, 사이드 패널은 액션(전화/웹사이트/예약)에만 집중하는 게 "정독 레이어와 스캔 레이어 분리" 원칙(기획 초안 설계 원칙 1)에 더 맞는다.

### 9.2 시설 리스트 반응형

모바일은 `overflow-x-auto` 가로 스크롤(섹션 2.9), `md:`(**960px**) 이상에서는 폭 여유가 생기므로 `md:flex-wrap md:overflow-visible`로 전환해 스크롤 대신 자연스럽게 줄바꿈.

---

## 10. 접근성

- **외부 링크 CTA**: `SpotActionBar`의 예약/웹사이트 링크는 `target="_blank" rel="noopener noreferrer"` + 시각적으로 숨겨진 보조 텍스트로 새 창 이동을 명시한다. 예: 버튼 안에 `<span className="sr-only">(새 창에서 열림)</span>` 추가. 아이콘만으로 "외부 이동"을 표시하지 않는다(아이콘 온리 정보 전달 금지 원칙과 동일 이유).

> **[v3] `aria-label`이 있는 요소에는 `sr-only`가 통하지 않는다** — 접근성 이름 계산에서 `aria-label`은 요소의 내부 텍스트를 **덮어쓴다.** 따라서 아이콘 온리 버튼(`aria-label` 필수)에 `sr-only`로 새 창 안내를 넣으면 그 텍스트는 낭독되지 않는다. 구현 중 보조 웹사이트 버튼이 정확히 이 상태였고(실효 이름이 `"웹사이트 열기"` 뿐), 위 원칙이 그 버튼에서만 깨져 있었다.
>
> 규칙: **라벨 방식에 따라 나눈다.**
> - 텍스트 라벨이 있는 버튼(Primary): 내부 `<span className="sr-only">(새 창에서 열림)</span>` — 텍스트와 함께 낭독된다
> - 아이콘 온리 버튼(`aria-label` 사용): 안내를 **`aria-label` 자체에 접어 넣는다** — 예 `aria-label="웹사이트 열기 (새 창에서 열림)"`. 이 경우 `sr-only`는 넣지 않는다(죽은 마크업이 된다)
- **아이콘 온리 버튼**: `SpotDetailTopBar`의 뒤로가기/공유, `SpotActionBar`의 전화/웹사이트 버튼은 전부 `aria-label` 필수(예: `aria-label="전화 걸기"`, `aria-label="공유하기"`). `TopBar.tsx`가 이미 햄버거 버튼에 `aria-label="메뉴 열기"`를 쓰는 기존 관행과 동일.
- **경고 배너**: `role="status"` + `aria-live="polite"`를 권장한다. `role="alert"`(assertive)는 페이지 로드와 동시에 스크린리더가 다른 콘텐츠를 끊고 즉시 읽게 만드는데, 이 배너는 "판단에 영향을 주는 정보"이지 긴급 오류 알림이 아니다. 사용자가 페이지에 진입하자마자 강제로 끼어드는 것보다, 문서 순서상 자연스럽게 만나되 배너라는 걸 보조기술이 인지할 수 있게 하는 `status`가 더 적절하다고 판단했다 — 이 부분은 실제 스크린리더 사용성 테스트 전까지는 **제안**으로 남긴다.
- **색상만으로 정보 전달 금지**: 유료/무료 배지(섹션 3)는 solid/outline 명암 차이 + 반드시 "유료"/"무료" 텍스트 라벨을 함께 표기해 색맹 사용자도 텍스트로 구분 가능. 경고 배너도 앰버 배경뿐 아니라 삼각형 경고 아이콘 + 텍스트 조합. legalStatus 배지가 초록/빨강/앰버 트라이어드를 쓰게 되면서(편차 #1) **적록색맹 사용자에게 초록·빨강이 구분되지 않는 문제**가 생기므로, 텍스트 라벨 병기는 선택이 아니라 필수다.
- **탭 순서**: `SpotDetailTopBar`(뒤로가기 → 공유) → 본문 콘텐츠 순서 → `SpotActionBar`(전화 → 웹사이트 → 예약). `fixed` 액션바가 DOM 상 페이지 최하단에 위치해도 시각적 위치와 탭 순서가 어긋나지 않도록 DOM 순서를 실제 마크업 마지막에 둔다(별도 `tabIndex` 조작 불필요).
- **다크모드**: **v2에서 해소됨.** `globals.css`에 모든 시맨틱 토큰의 다크값이 파생돼 있어, 이 문서의 컴포넌트가 토큰만 쓰면 다크모드가 자동으로 따라온다.

- **대비비 실측 결과 (v2.2, WCAG AA 4.5:1 기준)** — 전 토큰 조합을 측정했다.

| 조합 | 라이트 | 다크 |
|---|---|---|
| `success-deep` on `success-soft` | 4.79 ✅ | 6.35 ✅ |
| `warning-deep` on `warning-soft` | 4.51 ✅ | 7.17 ✅ |
| `error-deep` on `error-soft` | 4.55 ✅ | 5.42 ✅ |
| `ink` on `canvas` | 17.93 ✅ | 16.91 ✅ |
| `body` on `canvas` | 8.45 ✅ | 7.66 ✅ |
| `link` on `canvas` | 4.55 ✅ | — |
| CTA (`on-primary` on `primary`) | 17.93 ✅ | — |
| **`mute` on `canvas`** | **3.54 ⚠️** | 4.88 ✅ |

  - **실측 중 발견한 버그**: 다크 `error-deep`이 라이트값 `#ee0000`을 그대로 쓰고 있어 대비비가 **1.3:1**까지 떨어졌었다(빨강은 앰버·초록과 달리 휘도가 본질적으로 낮다). `#ff6166`으로 수정해 5.42:1 확보. 회색조 필터 검증 중에 드러난 문제다.
  - **`mute`(`#888888`)는 라이트 모드에서 일반 텍스트 AA에 미달한다(3.54:1).** 이건 파생값이 아니라 **DESIGN.md 원본값**이라 유지하되, **본문 텍스트에는 절대 쓰지 않고** placeholder·아이콘·보조 캡션 같은 비필수 정보에만 한정한다. 결측 표기("정보 없음")처럼 의미 전달이 필요한 텍스트는 `text-body`를 쓴다.

> **[v3] 위 표는 `canvas` 배경 조합만 측정했다 — 인셋 배경에서는 결과가 달라진다.**
>
> 상세페이지는 `canvas-soft`(스펙 셀·시설 칩·리뷰 카드)와 `canvas-soft-2`(지도 placeholder) 위에도 텍스트를 얹는다. 이 조합들을 추가로 실측했다.
>
> | 조합 | 라이트 | 다크 |
> |---|---|---|
> | `body` on `canvas-soft` | 8.10 ✅ | 7.31 ✅ |
> | `ink` on `canvas-soft` | 17.18 ✅ | 16.13 ✅ |
> | `body` on `canvas-soft-2` | 7.75 ✅ | 6.74 ✅ |
> | **`mute` on `canvas-soft-2`** | **3.25 ❌** | **4.29 ❌** |
>
> **핵심**: `mute`는 `canvas` 위에서 다크가 4.88로 통과하지만, `canvas-soft-2` 위에서는 **다크도 4.29로 미달**한다. 위 표만 보고 *"다크에서는 `mute`를 써도 된다"* 고 읽으면 안 된다 — 인셋 배경 위에서는 **양쪽 모드 모두 미달**이다. `mute`의 허용 범위는 배경과 무관하게 "텍스트가 아닌 것"(아이콘·도형 placeholder)으로 한정하는 편이 안전하다.
>
> 상세페이지가 실제로 만드는 전경/배경 조합 10종은 전부 AA를 통과한다(PR #23에서 `globals.css` 토큰값 기준 산출).

---

## 관련 문서

- 이 스펙에 대한 의사결정 경위(기획 초안 대비 이견, v1→v2→v2.1 변경 이유, 편차 승인 기록, 버전 히스토리)는 [`docs/design/decisions/spot-detail-design-decisions.md`](../../decisions/spot-detail-design-decisions.md) 참고.
- 아직 해결되지 않은 후속 논의, 신규 토큰 필요 목록, BE 스키마 대조 메모는 [`docs/front/backlog/spot-detail-design-followups.md`](../../../front/backlog/spot-detail-design-followups.md) 참고.

