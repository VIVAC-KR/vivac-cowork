# 검색·지도 탐색 UI 설계 문서

- 작성일: 2026-08-10
- 범위: `/search` 목록 모드 · 지도 모드 — **시각/컴포넌트 레이어**
- 선행 문서(계약, 수정하지 않음): [`docs/features/search.md`](../../../features/search.md) — 확정 계약 정본
- 설계 경위: [`docs/front/projects/search-map-explore.md`](../../../front/projects/search-map-explore.md)
- 참고 소스: vivac-frontend 루트의 `DESIGN.md`(디자인 시스템 — **단일 진실 공급원**), `apps/web/src/app/globals.css`(토큰 배선), [`spot-detail.md`](spot-detail.md)(같은 feature의 형제 문서), [`docs/PRODUCT.md`](../../../PRODUCT.md) §4.3(핵심 속성 정의)

이 문서는 `docs/features/search.md`가 정한 계약을 뒤집지 않는다. "그 계약이 실제로 어떻게 생겼고 어떤 컴포넌트로 만들어지는지"만 정의한다.

---

## 0. 토큰 체계

[`spot-detail.md §0`](spot-detail.md)이 정의한 **DESIGN.md 우선 정책**을 그대로 따른다. 색상·타이포·모양·여백 토큰을 여기서 다시 정의하지 않는다.

이 문서에서 추가로 지키는 제약 2가지:

- **`text-mute`를 본문 텍스트에 쓰지 않는다** — 라이트 모드 대비비 3.54:1로 AA 미달(`spot-detail.md §10`). 톤다운은 색이 아니라 **크기·굵기**로 만든다.
- **여섯 번째 액센트 색을 도입하지 않는다** (DESIGN.md Don't). 핵심 속성 아이콘은 색이 아니라 **점등/미표시**로 상태를 표현한다.

---

## 1. 컴포넌트 분해도

```
/search  (app/search/page.tsx — Suspense 셸)
└─ SearchResultsView                    ← 클라이언트 경계. 모드 분기·쿼리 소유
   ├─ SearchInput                       검색 입력 (홈과 공용)
   ├─ SearchModeSegment                 데스크톱 모드 토글 (960px+)
   ├─ SearchModeFloatingToggle          모바일 모드 토글 (우하단 플로팅)
   │
   ├─[목록 모드]
   │  ├─ SearchFilterRail               데스크톱 좌측 레일 (960px+)
   │  ├─ SearchFilterMobile             모바일 필터 트리거 + 모달 시트
   │  └─ SearchResultList               가상화 스크롤 컨테이너
   │     └─ SpotResultCard × N
   │        ├─ SpotTrustBadge  ★개정    3등급만 렌더
   │        └─ SpotAttributeIcons ★신규 핵심 속성 4슬롯
   │
   └─[지도 모드]
      ├─ SpotMap ★신규                  네이버 SDK 컨테이너
      │  ├─ SpotMapPin ★신규
      │  ├─ SpotMapCluster ★신규
      │  └─ MapRefetchButton ★신규      "이 지역 재검색"
      ├─ SearchResultList               데스크톱 분할 좌측 / 모바일 시트 내부
      └─ BottomSheet                    모바일 비모달 스냅 시트
         └─ SpotMiniCard ★신규          핀 선택 시 (피크 상태)
```

`★신규` 6개, `★개정` 1개. 나머지는 구현 완료.

### 1.1 신규·개정 컴포넌트 Props

```ts
// ★신규 — 핵심 속성 4슬롯 (PRODUCT.md §4.3)
interface SpotAttributeIconsProps {
  fire: string | null;          // 화기
  amenities: string[] | null;   // 편의시설
  booking: string | null;       // 예약/요금
  access: string | null;        // 접근성
  size?: "sm" | "md";           // sm=카드·미니카드, md=상세
}

// ★개정 — 3등급만 렌더 (기존 TrustBadge 대체)
interface SpotTrustBadgeProps {
  tier: 1 | 2 | 3 | null;       // 1·2·null → null 반환
}

// ★신규
interface SpotMapProps {
  items: SpotMapItem[];         // 좌표 non-null 보장
  selectedUid: string | null;
  onSelect: (uid: string | null) => void;
  onCameraIdle: (bbox: string, center: LatLng, zoom: number) => void;
  className?: string;
}

interface SpotMiniCardProps {
  spot: SpotListItem;
  onClose: () => void;
}

interface MapRefetchButtonProps {
  visible: boolean;             // 카메라가 마지막 조회 bbox를 벗어났을 때만
  loading: boolean;
  onClick: () => void;
}
```

### 1.2 기존 컴포넌트 Props (변경 없음)

| 컴포넌트 | Props |
|---|---|
| `SearchInput` | `initialQuery?` `className?` |
| `FilterChip` | `label` `selected` `onToggle` |
| `FilterGroups` | `selectedCategories` `selectedRegion` `onToggleCategory` `onToggleRegion` |
| `SearchFilterRail` | `className?` |
| `SearchFilterMobile` | `className?` `resultCount` |
| `SearchModeSegment` / `FloatingToggle` | `mode` `onChange` (+`className?`) |
| `SearchResultList` | `query` `className?` |
| `BottomSheet` | `open` `onClose?` `ariaLabel` `snapPoints?` `initialHeight?` `modal?` |

`MapPlaceholder`는 `SpotMap` 도입 시 **삭제**한다.

### 1.3 카드 높이 제약 ⚠

`SearchResultList`가 가상화에 고정 높이를 쓴다 — `CARD_HEIGHT = 108`, `CARD_HEIGHT_SM = 144`(600px+), `CARD_GAP = 16`. 이 값은 **썸네일이 결정**한다(`p-3` 24 + `w-28` 4:3 → 84 = 108).

본문 영역 84px 배분:

| 요소 | 높이 |
|---|---|
| 카테고리 `text-caption` | ~16px |
| 제목 `text-body-md-strong` | ~24px |
| 지역 `text-caption` + `mt-1` | ~20px |
| **핵심 속성 아이콘 행** ★신규 | ~20px |
| 합계 | **~80px ≤ 84px** |

**높이 상수를 바꾸지 않고 들어간다.** 신뢰도 배지는 제목 우측이라 세로 공간을 쓰지 않고, 3등급만 렌더하므로 대부분 비어 있다.

구현 시 실측으로 검증한다 — 84px를 넘으면 상수와 `estimateSize`를 함께 올려야 하며, 한쪽만 바꾸면 가상화 스크롤 위치가 어긋난다.

---

## 2. 검색 입력 · 결과 헤더

### 2.1 `SearchInput`

홈과 검색 화면이 공유한다. DESIGN.md `form-input-lg` 스펙을 따른다.

| 요소 | 스펙 |
|---|---|
| 컨테이너 | `h-12 rounded-md border border-hairline bg-canvas px-4 shadow-level-2`, `flex items-center gap-3` |
| 검색 아이콘 | 20×20, `stroke-width 1.75`, `text-mute`, `aria-hidden` |
| 입력 | `text-body-md text-ink`, placeholder `text-mute` |
| 라벨 | `aria-label="장소 검색"` (시각적 라벨 없음) |

`text-mute` 예외 2건 — §0 제약의 명시적 예외다.

- **검색 아이콘**: `aria-hidden` 장식이라 의미 전달을 지지 않는다.
- **placeholder**: 입력 시 사라지는 힌트이고, 같은 정보를 `aria-label`이 전달한다. WCAG는 placeholder를 유일한 라벨로 쓰는 것을 금지할 뿐 대비를 요구하지 않는다.

`key={initialQuery}`로 리마운트해 URL을 단일 진실원으로 유지한다(비제어 입력).

### 2.2 결과 헤더 — 삼중 중복 제거 ⚠

개정 전 구현은 같은 정보를 **세 번** 말했다 — 검색창의 검색어, 이브로우 "검색 결과", h1 "'{query}' 검색 결과".

세로 비용도 크다. `pt-6`(24) + `h-12`(48) + `mt-4`(16) + 이브로우(20) + `mt-1`(4) + h1 `display-sm`(~32) = **약 144px**. `SearchResultsView`는 `h-[calc(100dvh-64px)]` 고정 프레임이라 이 144px이 그대로 목록·지도 높이에서 빠진다. 667px 기기에서 콘텐츠 영역이 459px로 줄어든다.

**개정** — 이브로우와 h1을 시각적으로 제거하고 결과 수를 그 자리에 둔다.

```
┌─────────────────────────────────┐
│ 🔍 가리왕산                      │
│                                 │
│ 총 7곳            [목록|지도]    │
└─────────────────────────────────┘
```

| 항목 | 값 |
|---|---|
| h1 | `sr-only`로 유지 — `'{query}' 검색 결과`. 시각적으로만 제거한다 |
| 결과 수 | `text-body-sm-strong text-ink`, `총 {n}곳` |
| 모드 토글 | 우측 정렬, 960px+ 에서만 (`SearchModeSegment`) |
| 헤더 높이 | `pt-6`(24) + `h-12`(48) + `mt-4`(16) + 행(20) ≈ **108px** — 36px 회수 |

h1을 `sr-only`로 남기는 근거: 스크린리더 사용자에게는 페이지 제목이 필요하고, 검색 결과 페이지는 색인 대상이 아니므로 시각적 h1이 SEO에 기여하지 않는다.

### 2.3 결과 수는 헤더에만 둔다

개정 전에는 `총 {n}곳`이 **3곳에 중복 렌더**됐다 — 목록 모드 목록 위, 지도 모드 데스크톱 좌측, 지도 모드 모바일 시트 안. 모드·뷰포트마다 위치가 달라 눈이 매번 다시 찾아야 한다.

헤더 한 곳으로 통합한다. 모바일 시트 안의 필터 트리거는 남기되 개수 표기는 뺀다.

| 상태 | 표기 |
|---|---|
| 첫 조회 전 | `—` (0이 아니다 — "필터 때문에 0건"으로 오독된다) |
| 상한 초과 | `10,000+` |
| 정상 | `총 {n}곳` |

### 2.4 검색어 없는 상태 (`/search`)

`q`가 비면 조회하지 않고 안내 화면을 보여준다.

| 요소 | 스펙 |
|---|---|
| 컨테이너 | `max-w-2xl` 중앙 정렬, `md:max-w-4xl lg:max-w-5xl` |
| 검색창 | 동일 |
| 제목 | `text-display-sm text-ink sm:text-display-md` — "검색어를 입력해주세요" |
| 보조 | `mt-8 text-body-md text-body` — "찾고 싶은 야영장·박지·지역을 검색해보세요." |
| 모드 토글 | **미노출** — 보여줄 결과가 없다 |

이 화면은 §2.2의 높이 압박을 받지 않으므로 `display` 스케일 제목을 유지한다.

---

## 3. 결과 카드 `SpotResultCard`

### 3.1 레이아웃

```
┌──────────┬──────────────────────────────┐
│          │ 자연휴양림           ⚠ 미검증 │  ← 카테고리 / 신뢰도(3등급만)
│  썸네일   │ 가리왕산 자연휴양림            │  ← 제목
│  4:3     │ 강원 정선                     │  ← 지역
│          │ 🔥 🚰 📅 🚗                   │  ← 핵심 속성 4슬롯
└──────────┴──────────────────────────────┘
```

| 요소 | 스펙 |
|---|---|
| 카드 | `flex gap-4 rounded-md bg-canvas p-3 shadow-level-3`, 전체가 `<Link>` |
| 썸네일 | `aspect-[4/3] w-28 sm:w-40 rounded-sm bg-canvas-soft-2` — 결측 시 중립 플레이스홀더 SVG |
| 카테고리 | `text-caption text-body`, `truncate`, 복수는 ` · ` 연결 |
| 제목 | `text-body-md-strong text-ink`, `truncate` |
| 지역 | `mt-1 text-caption text-body` |
| 속성 아이콘 행 | `mt-1.5 flex gap-2` |

카테고리를 제목 **위** 서브타이틀로 두는 기존 배치를 유지한다 — 장소 유형이 스캔의 1차 분류축이다.

### 3.2 `SpotTrustBadge` — 3등급만

| tier | 의미 | 렌더 |
|---|---|---|
| `1` | 공식 소스 + 핵심 속성 완비 | **없음** |
| `2` | 신뢰 소스, 일부 누락 | **없음** |
| `3` | 단일 비공식 소스, 미검증 | `⚠ 미검증` |
| `null` | 미판정 | **없음** |

| 항목 | 값 |
|---|---|
| 클래스 | `inline-flex shrink-0 items-center gap-1 rounded-full bg-canvas-soft px-2 py-0.5 text-caption text-body` (DESIGN.md `badge-secondary`) |
| 아이콘 | 12×12 삼각형 경고, `stroke-width 1.2`, `aria-hidden` |
| 위치 | 제목 행 우측, `items-start` |

`bg-warning`을 쓰지 않는다 — DESIGN.md `warning` 팔레트는 상세 화면의 `SpotAlertBanner`(산불통제 등 행동을 요구하는 경고)가 점유한다. 목록에서 미검증 배지가 같은 톤을 쓰면 위계가 무너진다.

> ⚠ **기존 구현이 `PRODUCT.md` §4.3·`data-pipeline.md`와 등급 방향이 반대다.** 두 문서 모두 1이 최고인데 코드는 3을 최고로 매핑하고 1·2·3 전부에 배지를 단다. **같은 수정이 `RecommendedSpotCard`(홈)에도 필요하다** — 동일한 뒤집힌 매핑을 복사해 쓰고 있다.

### 3.3 `SpotAttributeIcons` — 아이콘 어휘

[`PRODUCT.md`](../../../PRODUCT.md) §4.3 핵심 속성 4종. **순서 고정**, 값 없으면 자리를 유지하고 아이콘만 미표시한다.

| 순서 | 속성 | 아이콘 | 근거 |
|---|---|---|---|
| 1 | 화기 | 불꽃 | 백패킹 판단의 1순위 |
| 2 | 편의시설 | 수도꼭지 | 식수·개수대가 편의시설 중 가장 자주 찾는 항목 |
| 3 | 예약/요금 | 달력 | 예약 방식·선착순 여부 |
| 4 | 접근성 | 자동차 | 차량 진입·주차 |

| 항목 | 값 |
|---|---|
| 크기 | 16×16 (`sm`) / 20×20 (`md`) |
| 스타일 | `stroke-width 1.5`, `currentColor`, `fill="none"` |
| 색 | `text-body` (점등) — **액센트 색을 쓰지 않는다** |
| 간격 | `gap-2` |
| 폭 합계 | 16×4 + 8×3 = **88px** (모바일 본문 223px 여유) |
| 접근성 | 행에 `role="list"`, 각 아이콘 `<li>` + `sr-only` 텍스트. SVG는 `aria-hidden` |

미표시 슬롯은 빈 `<li className="size-4">`로 자리만 잡는다 — 카드 간 아이콘 위치가 고정돼야 세로 비교가 된다.

아이콘은 `components/icons/`로 추출한다. 목록 카드·미니카드 최소 두 곳에서 재사용되므로 인라인 SVG를 복제하지 않는다. 아이콘 라이브러리는 도입하지 않는다 — 번들 비용과 기존 인라인 아이콘과의 톤 불일치 때문이다.

### 3.4 점등 규칙 — 긍정 값일 때만

§4.3 속성은 불리언이 아니라 서술값이다("화로대만 가능", "4WD 필요", "도보 40분"). 아이콘 하나로 값을 표현할 수 없으므로 **긍정 값일 때만 점등**한다.

| 값 | 렌더 |
|---|---|
| 긍정 (화기 가능, 차량 진입 가능 등) | 아이콘 점등 |
| 부정 (불피움 불가 등) | **미표시** |
| `null` (미상) | **미표시** |

`spot-detail.md` §3의 `SpotPetBadge`와 동일한 논리다 — *"`false`/`null` 배지 미노출. 없음/false를 시각적으로 구분할 근거(데이터도, 니즈도) 없음."*

> **감수한 것** — "불가"와 "정보 없음"이 카드에서 구분되지 않는다. 특히 **화기 부정값이 목록에서 침묵**한다. 노지에서 화기 가부는 안전·법규 문제이므로, 부정값 전달의 책임은 전적으로 상세 화면이 진다(`SpotSpecGrid` "화로 유형" 셀 + 3등급의 "확인되지 않음" 일괄 표시). 목록에서 화기 아이콘이 없다는 것을 "불 피워도 된다"로 읽을 여지는 없다 — 아이콘 부재는 정보 부재이지 허가가 아니다.

---

## 4. 필터

### 4.1 구성

[`docs/features/search.md`](../../../features/search.md) 계약에 따라 **2종만** 노출한다.

| 그룹 | 옵션 | 선택 |
|---|---|---|
| 카테고리 | 공공 4종 (자연휴양림 · 국립공원 야영장 · 국민여가캠핑장 · 지자체 야영장) | 다중 |
| 지역 | `region_province` 17개 시도 | 단일 |

노지(`BACKPACKING`)는 로드맵 3단계라 칩에 없다. 정렬·자동완성·속성 필터는 MVP에 없다.

### 4.2 `SearchFilterRail` — 데스크톱 (960px+)

`md:grid-cols-[220px_1fr]` 레일 폭 **220px** 고정, `self-start`(스크롤 비동반).

| 요소 | 스펙 |
|---|---|
| 헤더 | `flex items-center justify-between` — `<h2 className="text-body-sm-strong text-ink">필터</h2>` + 활성 개수 |
| 초기화 | `text-body-sm text-link hover:underline`, 활성 필터가 있을 때만 |
| 그룹 | `<fieldset>` + `<legend className="text-body-sm-strong text-ink">`, `mt-3 flex flex-wrap gap-2` |

**17개 시도의 세로 비용** — 시도명은 전부 2글자로 축약된다(서울·부산·대구·인천·광주·대전·울산·세종·경기·강원·충북·충남·전북·전남·경북·경남·제주).

```
칩 폭 = px-2.5(20) + 2글자 caption(~22) ≈ 42px
220px ÷ (42 + gap-2 8) ≈ 4개/줄 → 17개 = 5줄 ≈ 180px

레일 총 높이 ≈ 헤더 20 + 카테고리 2줄 72 + 지역 5줄 180 + 여백 ≈ 350px
```

데스크톱 콘텐츠 영역(약 800px) 안에 들어간다. **접기(accordion)를 도입하지 않는다** — 5줄은 스캔 가능한 범위이고, 접으면 어떤 지역이 있는지 자체가 안 보인다.

### 4.3 `SearchFilterMobile` — 모바일 (<960px)

| 요소 | 스펙 |
|---|---|
| 트리거 | `inline-flex items-center gap-2 rounded-full border border-hairline-strong px-4 py-2 text-body-sm text-body` + 필터 아이콘 16×16 |
| 활성 배지 | `h-5 min-w-5 rounded-full bg-primary px-1 text-caption text-on-primary` — solid 반전으로 위계 |
| 시트 | `BottomSheet` **모달** — `max-w-[500px] max-h-[85vh] min-h-[60vh] px-5 pb-6` |
| 시트 헤더 | 중앙 정렬 `text-body-md-strong text-ink` + 우측 닫기 버튼 |
| 하단 CTA | `결과 {n}개 보기` — 개수를 모르면 `결과 보기` |

시트는 **모달**이다(포커스 트랩·스크롤락·오버레이·ESC 닫기). 지도 모드의 결과 시트가 비모달인 것과 다르다 — 필터는 배경과 동시에 조작할 이유가 없다.

`triggerRef`로 닫힐 때 포커스를 트리거로 되돌린다.

### 4.4 `FilterChip`

| 상태 | 클래스 |
|---|---|
| 선택 | `border-primary bg-primary text-on-primary` (solid 반전) |
| 미선택 | `border-hairline-strong text-body hover:border-ink` |
| 공통 | `shrink-0 rounded-full border px-2.5 py-1 text-caption transition-colors` |

`aria-pressed`로 토글 상태를 전달한다. 표시 전용 `SpotChip`(상세 화면)과 달리 상호작용 요소다.

선택 상태에 `bg-primary`를 쓰는 것은 DESIGN.md `button-primary`/`button-secondary`의 solid↔outline 위계를 그대로 가져온 것이라 여섯 번째 액센트 도입이 아니다.

### 4.5 `text-mute` 위반 1건

`SearchFilterRail`의 활성 개수가 `text-body-sm text-mute`다. **활성 필터 개수는 의미 있는 정보**이므로 §0 제약(본문 텍스트에 `mute` 금지)에 걸린다. `text-body`로 올리고 톤다운은 크기로만 만든다.

`SearchFilterMobile`의 닫기 버튼 `text-mute hover:text-ink`는 **유지**한다 — 본문 텍스트가 아닌 UI 컴포넌트이고, 3.54:1로 WCAG 비텍스트 대비 3:1을 충족한다.

### 4.6 선행 확인 필요 — `region_province` 값 형식 🔴

[`spot-search-postgres-fts.md`](../../../core/projects/spot-search-postgres-fts.md) §4.2에 따르면 `region_province`는 **`spot_field_options` 화이트리스트로 관리되는 구조화 값이고 등호 매칭**이다. 문자열이 한 글자라도 다르면 결과가 0건이다.

현재 `searchFilterOptions.ts`는 `["경기", "강원", "충북", "경남"]`을 **그대로 전송**한다. 화이트리스트 실제 값이 축약형(`경기`)인지 전체명(`경기도`)인지 문서에 없다.

**카테고리에서 이미 한 번 터진 함정과 같은 구조다** — 한글 라벨을 코드 대신 보내 필터가 조용히 0건을 반환했다. 카테고리는 `{ code, labelKo }` 분리로 해소됐지만 지역은 아직 분리되지 않았다.

17개 시도로 확장하기 전에 화이트리스트 실제 값을 확인하고, 확인 결과에 따라 `regionFilterOptions`도 `{ code, labelKo }` 구조로 바꾼다.

---

## 5. 모드 토글

목록 ↔ 지도 전환. 뷰포트에 따라 다른 컨트롤을 쓰고 **동시에 존재하지 않는다**.

| 뷰포트 | 컨트롤 | 위치 |
|---|---|---|
| 960px+ | `SearchModeSegment` | 헤더 우측 (`max-md:hidden`) |
| <960px | `SearchModeFloatingToggle` | 우하단 고정 (`md:hidden`) |

### 5.1 `SearchModeSegment` — 데스크톱

| 요소 | 스펙 |
|---|---|
| 컨테이너 | `inline-flex shrink-0 rounded-md border border-hairline bg-canvas-soft p-0.5`, `role="group"` `aria-label="보기 모드"` |
| 버튼 | `rounded-sm px-3 py-1.5 text-body-sm transition-colors`, `aria-pressed` |
| 선택 | `bg-canvas text-ink shadow-level-1` — 배경 반전 + 그림자로 떠 있는 느낌 |
| 미선택 | `text-body hover:text-ink` ⚠ **개정** |

**개정 근거** — 기존 미선택이 `text-mute hover:text-body`다. "목록"/"지도"는 의미를 지는 **텍스트 라벨**이므로 WCAG 4.5:1이 필요한데 `mute`는 3.54:1로 미달이다(§0). 선택/미선택 구분은 이미 배경(`bg-canvas`)과 `shadow-level-1`이 만들고 있어 색을 낮출 필요가 없다.

`className`은 미디어쿼리로 감싼다(`max-md:hidden`) — 컴포넌트가 `inline-flex`를 자체 보유해 무-미디어쿼리 `hidden`은 캐스케이드에서 밀린다.

### 5.2 `SearchModeFloatingToggle` — 모바일

| 요소 | 스펙 |
|---|---|
| 컨테이너 | `fixed bottom-6 right-4 inline-flex min-h-11 items-center gap-2 rounded-pill bg-primary px-5 py-3 text-body-sm-strong text-on-primary shadow-level-4` |
| 터치 영역 | `min-h-11` = 44px (WCAG 2.5.5) |
| 아이콘 | 16×16, `aria-hidden` |
| 라벨 | **행위형** — 목록 모드에서 "지도", 지도 모드에서 "목록" |
| `aria-label` | "지도로 보기" / "목록으로 보기" |

**우하단 배치 근거** — 하단 중앙은 스냅 시트의 드래그 핸들(피크 상태) 자리다.

**행위형 라벨 근거** — 세그먼트의 "지도"/"목록" 버튼과 접근성 이름이 겹치면 스크린리더가 두 컨트롤을 구분하지 못한다. 뷰포트로 갈라져 동시 존재하지 않더라도 이름을 다르게 둔다.

### 5.3 z-index 계층 ⚠ 결함 2건

기존 구현의 계층:

| 레이어 | z-index | 위치 |
|---|---|---|
| 모달 필터 시트 | `z-50` (`fixed inset-0`) | `BottomSheet` modal |
| **플로팅 토글** | **`z-50`** | `SearchModeFloatingToggle` |
| 비모달 결과 시트 | `z-40` (`fixed inset-x-0 bottom-0`) | `BottomSheet` 스냅 |

**결함 A — 토글이 모달 필터 시트 위에 그려진다.** 둘 다 `z-50`이고 토글이 DOM 후순위(`SearchResultsView` 최하단)라 스태킹에서 이긴다. 필터 시트가 열린 동안 오버레이 위로 토글이 보이고 클릭된다. 포커스 트랩은 키보드만 막는다.

**결함 B — 토글이 결과 시트 콘텐츠를 가린다.** 결과 시트가 `z-40`이라 시트를 90%까지 올리면 우하단 결과 카드 위에 토글이 겹친다.

**개정** — 레이어를 분리한다.

| 레이어 | z-index |
|---|---|
| 모달 필터 시트 | `z-50` |
| 플로팅 토글 | `z-45` |
| 비모달 결과 시트 | `z-40` |

- 결함 A: 토글을 `z-45`로 낮추면 모달(`z-50`) 아래로 들어간다.
- 결함 B: 결과 시트 리스트 하단 패딩을 `pb-24`로 유지해 마지막 카드가 토글에 덮이지 않게 한다(기존 적용됨). 토글이 시트 위에 떠 있는 것 자체는 **의도**다 — 시트를 올린 상태에서도 지도로 돌아갈 수 있어야 한다.

`z-45`는 Tailwind 기본 스케일에 없으므로 `globals.css` `@theme`에 추가하거나 임의값(`z-[45]`)을 쓴다. 임의값을 쓸 경우 `globals.css` 헤더 주석의 편차 목록에 기록한다.

---

## 6. 지도 모드 레이아웃

`?mode=map`. 별도 라우트가 아니라 `/search`의 두 번째 표현이다.

### 6.1 데스크톱 (960px+)

```
┌─────────────────────────────────────────────────┐
│ 🔍 가리왕산              총 7곳     [목록|지도]  │  헤더 (§2.2)
├──────────────────┬──────────────────────────────┤
│ [필터]           │                              │
│                  │        ┌──────────────┐      │
│ ┌──────────────┐ │        │ 이 지역 재검색 │      │
│ │ 카드         │ │        └──────────────┘      │
│ ├──────────────┤ │                              │
│ │ 카드         │ │            지도               │
│ ├──────────────┤ │                              │
│ │ 카드         │ │                              │
│ └──────────────┘ │                              │
└──────────────────┴──────────────────────────────┘
     45 : 55 (960~1199)  ·  40 : 60 (1200+)
```

| 항목 | 값 |
|---|---|
| 컨테이너 | `mt-4 hidden min-h-0 flex-1 gap-6 px-4 pb-6 md:grid`, `max-w-page sm:px-6` |
| 분할 | `md:grid-cols-[45fr_55fr] lg:grid-cols-[40fr_60fr]` — 1200+ 비율은 §9.4에서 35:65에서 개정됨 |
| 좌측 | `flex min-h-0 flex-col` — 필터 트리거 + `SearchResultList` |
| 우측 | `SpotMap className="min-h-0"` |

**목록 모드와 달리 필터가 레일이 아니라 시트다.** 좌측 컬럼 전체를 결과 목록이 쓰므로 220px 레일을 놓을 자리가 없다. 데스크톱에서도 `SearchFilterMobile`의 모달 시트를 재사용한다.

> 이름이 사실과 다르다 — 데스크톱에서도 쓰이므로 `SearchFilterSheet`가 맞다. **동작에는 영향이 없으므로 지금 리네임하지 않고**, 지도 SDK 연동으로 이 파일을 여는 시점에 함께 바꾼다.

목록 모드는 레일, 지도 모드는 시트 — 같은 화면에서 필터 UI가 모드에 따라 바뀌지만 **동시에 보이지 않으므로** 학습 부담이 없다.

`총 {n}곳`은 §2.3에 따라 헤더로 올라가므로 좌측 컬럼 상단에서 **제거**한다. 기존 구현은 이 자리에서 `text-mute`로 §0을 위반하고 있기도 하다.

### 6.2 모바일 (<960px) — 지도는 full-bleed

```
┌─────────────────────────────────┐
│ 🔍 가리왕산                      │  헤더
│ 총 7곳                          │
├─────────────────────────────────┤
│      ┌──────────────┐           │
│      │ 이 지역 재검색 │           │
│      └──────────────┘           │
│                                 │
│            지도 (60%)            │
│                          ┌────┐ │
│                          │목록│ │  플로팅 토글
├─────────────────────────┴────┴─┤
│ ═══                             │  핸들
│ [필터]                          │
│ ┌─────────────────────────────┐ │  시트 40%
│ │ 카드                        │ │
└─────────────────────────────────┘
```

| 항목 | 값 |
|---|---|
| 지도 컨테이너 | `mt-4 min-h-0 flex-1 md:hidden` — **`px-4 pb-4` 제거** |
| 시트 | `BottomSheet` 비모달, `snapPoints=[0, 0.25, 0.55, 0.9]`, `initialHeight={0.4}` |

**full-bleed 근거** — 기존 구현은 지도가 `px-4 pb-4` 안에 들어가 있다. 세 가지가 어긋난다.

1. `pb-4`는 시트가 최소 25%를 덮으므로 **한 번도 보이지 않는다.**
2. 좌우 `px-4`는 375px 화면에서 32px, 약 **8.5%**의 시야를 잘라낸다.
3. 여백 안의 지도는 "카드 안의 이미지"로 읽힌다. 조작 가능한 표면이라는 신호가 약해진다.

카드·목록이 여백을 갖는 것과 모순되지 않는다 — 여백은 **읽는 것**에 주고, **조작하는 표면**은 화면 끝까지 쓴다.

### 6.3 시트 스냅과 지도 가시 영역

| 스냅 | 시트 | 지도 노출 | 용도 |
|---|---|---|---|
| 피크 (`0`) | 핸들만 (44px) | ~93% | 지도를 넓게 훑는다. 핀 선택 시 미니카드(§7) |
| 25% | 카드 1.5장 | 75% | 지도 위주, 결과를 곁눈질 |
| **40%** (초기) | 카드 2~3장 | 60% | 기본 진입 |
| 55% | 카드 4장 | 45% | 핀 선택 시 자동 전환 |
| 90% | 거의 전체 | 10% | 사실상 목록 |

`initialHeight` 40%는 스냅 지점이 아니다 — 첫 진입에서만 쓰이고 드래그하면 가장 가까운 스냅으로 붙는다.

ESC는 최저 스냅으로 접고, 이미 최저면 무시한다(비모달이라 닫히지 않는다).

### 6.4 "이 지역 재검색" 버튼

| 항목 | 값 |
|---|---|
| 위치 | 지도 **상단 중앙**, `absolute top-4 left-1/2 -translate-x-1/2 z-10` |
| 스타일 | `inline-flex min-h-11 items-center gap-2 rounded-pill bg-canvas px-4 py-2 text-body-sm-strong text-ink shadow-level-3` |
| 노출 조건 | 카메라가 마지막 조회 bbox를 벗어났을 때만 |
| 로딩 | 라벨을 "검색 중"으로 바꾸고 `disabled` |
| 전환 | `opacity`·`translate-y` 150ms |

**상단 중앙 근거** — 하단은 시트(모바일)와 플로팅 토글이 점유한다. 상단 중앙은 국내외 지도 서비스가 공유하는 관례라 학습 비용이 없다.

`bg-canvas` + `shadow-level-3`로 지도 위에 떠 있게 한다. `bg-primary`를 쓰지 않는다 — 플로팅 토글이 이미 `bg-primary`이고, 한 화면에 solid primary 버튼이 둘이면 무엇이 주 행동인지 흐려진다.

### 6.5 `MapPlaceholder` 폐기

`SpotMap` 도입 시 삭제한다. 다만 **SDK 로드 실패·스크립트 차단** 시 대체 화면이 필요하므로, 기존 플레이스홀더의 문구·톤(`지도 준비 중`)을 §8 에러 상태로 옮긴다.

---

## 8. 상태 정의

### 8.1 로딩 — 스켈레톤

| 항목 | 값 |
|---|---|
| 개수 | 4장 |
| 카드 | `flex gap-4 rounded-md bg-canvas p-3 shadow-level-3` — 실제 카드와 동일 크롬 |
| 썸네일 | `aspect-[4/3] w-28 sm:w-40 animate-pulse rounded-sm bg-canvas-soft-2` |
| 텍스트 | 2줄 — `h-4 w-2/3`, `mt-2 h-3 w-1/3` |
| 컨테이너 | `aria-hidden="true"` |

**개정 — 로딩을 알리는 라이브 리전이 없다.** 스켈레톤이 `aria-hidden`이라 스크린리더 사용자에게는 아무 일도 일어나지 않는다. 컨테이너 밖에 `sr-only` 라이브 리전을 둔다.

```html
<p role="status" class="sr-only">검색 결과를 불러오는 중</p>
```

스켈레톤을 `aria-hidden`으로 두는 것 자체는 맞다 — 의미 없는 회색 상자를 읽어줄 이유가 없다.

**속성 아이콘 슬롯은 스켈레톤에 넣지 않는다.** 4개 중 몇 개가 켜질지 모르는데 자리를 잡아두면 로드 후 레이아웃이 흔들린다. 텍스트 2줄만으로 카드 높이(108/144px)가 이미 맞는다.

### 8.2 빈 결과

| 항목 | 값 |
|---|---|
| 컨테이너 | `rounded-md border border-hairline bg-canvas p-8 text-center` |
| 제목 | `text-body-md text-body` — "검색 결과가 없어요." |
| 보조 | `mt-1 text-body-sm text-body` |

**보조 문구를 필터 상태에 따라 분기한다.** 기존 구현은 필터가 없을 때도 "필터를 조정해보세요"라고 말한다.

| 조건 | 보조 문구 | 추가 |
|---|---|---|
| 필터 없음 | "다른 키워드로 검색해보세요." | — |
| 필터 있음 | "필터를 조정하거나 다른 키워드로 검색해보세요." | **필터 초기화** 버튼 |

필터가 걸린 채 0건일 때 초기화 버튼을 같은 자리에 두는 것이 핵심이다 — 사용자가 원인(필터)과 해결(초기화)을 한 화면에서 잇는다. `activeCount`는 `useSpotFilters`가 이미 제공한다.

### 8.3 에러

세 가지를 구분한다.

| 상황 | 판정 | 처리 |
|---|---|---|
| **초기 조회 실패** | `status === "error" && data === undefined` | 전면 에러 블록 + 재시도 |
| **다음 페이지 실패** | `isFetchNextPageError` | **리스트 유지** + 하단 인라인 재시도 |
| **커서 무효화** | `CursorScopeMismatchError` | UI 없음 — `resetQueries`로 조용히 처음부터 재조회 |

**다음 페이지 실패 시 자동 로드를 멈추는 것이 중요하다.** 스크롤이 바닥 근처에 머무는 한 트리거 조건이 계속 성립해 재요청이 무한 반복되고, 사용자는 하단 재시도 UI에 도달하지 못한다. 기존 구현이 `!isFetchNextPageError` 가드로 이미 처리하고 있다 — 유지한다.

| 요소 | 스펙 |
|---|---|
| 전면 에러 | `p-8 text-center`, 제목 `text-body-md text-body`, 보조 `text-body-sm text-body` |
| 인라인 에러 | `py-4 text-center`, `text-body-sm text-body` |
| 재시도 버튼 | `min-h-11 rounded-full border border-hairline-strong px-5 py-2.5 text-body-sm text-body` |

재시도 버튼에 `bg-primary`를 쓰지 않는다 — 에러 복구는 사용자가 원해서 온 행동이 아니다. outline으로 두고 주 행동(검색·필터)의 위계를 지킨다.

### 8.4 지도 전용 상태

목록에는 없고 지도에만 있는 상태 두 가지다.

**A. SDK 로드 실패**

스크립트 차단·네트워크 실패·키 오류로 지도를 못 그리는 경우. `MapPlaceholder`의 문구·톤을 여기로 옮긴다(§6.5).

| 항목 | 값 |
|---|---|
| 컨테이너 | `flex items-center justify-center rounded-lg border border-hairline bg-canvas-soft` (모바일 full-bleed면 `rounded-none border-x-0`) |
| 아이콘 | 40×40 지도 실루엣, `aria-hidden` |
| 제목 | `text-body-md text-body` — "지도를 불러오지 못했어요." |
| 보조 | `text-body-sm text-body` — "목록 모드로 결과를 확인할 수 있어요." |
| 행동 | **"목록으로 보기" 버튼** — 지도가 죽어도 결과 자체는 살아 있다 |

**B. 결과는 있는데 핀이 0개** — 좌표 채움률이 28.3%라 흔하다

목록에 7곳이 뜨는데 지도가 비어 있으면 사용자는 지도가 고장났다고 읽는다. 침묵하면 안 된다.

| 항목 | 값 |
|---|---|
| 위치 | 지도 위 중앙 오버레이, `bg-canvas/95 shadow-level-3 rounded-md px-5 py-4` |
| 문구 | "이 결과에는 위치 정보가 아직 없어요." |
| 보조 | "{n}곳을 목록에서 볼 수 있어요." |

지도 자체는 정상 렌더한다(회색 상자로 바꾸지 않는다) — 지도는 살아 있고 표시할 핀이 없을 뿐이다.

### 8.5 `text-mute` 위반 3건

`SearchResultList`에 §0 제약 위반이 남아 있다.

| 위치 | 기존 | 개정 | 근거 |
|---|---|---|---|
| 에러 보조 문구 | `text-mute` | `text-body` | 복구 방법을 안내하는 본문 |
| 빈 결과 보조 문구 | `text-mute` | `text-body` | 다음 행동을 안내하는 본문 |
| "불러오는 중…" | `text-mute` | `text-body` | `role="status"`로 읽어주는 실제 정보 |

세 곳 다 **크기(`text-body-sm`)는 유지**한다. 톤다운은 크기로만 만든다.

---

## 9. 반응형

DESIGN.md 커스텀 브레이크포인트를 쓴다. Tailwind 기본값(640/768/1024/1280)이 아니다.

| 토큰 | 값 | 역할 |
|---|---|---|
| `sm` | 600px | 태블릿 |
| `md` | 960px | 데스크톱 |
| `lg` | 1200px | 와이드 |
| `xl` | 1400px | 울트라와이드 (`max-w-page`와 동일) |

### 9.1 전환 지점 종합

| 구간 | 컨테이너 | 필터 | 모드 토글 | 지도 모드 | 카드 |
|---|---|---|---|---|---|
| **~599** | `max-w-2xl px-4` | 시트 | 플로팅 | 지도 full-bleed + 스냅 시트 | 썸네일 `w-28`, 높이 **108** |
| **600~959** | `max-w-2xl px-6` | 시트 | 플로팅 | 지도 full-bleed + 스냅 시트 | 썸네일 `w-40`, 높이 **144** |
| **960~1199** | `max-w-4xl`(목록) / `max-w-page`(지도) | **레일 220px**(목록) / 시트(지도) | **세그먼트** | **분할 45:55** | 동일 |
| **1200~1399** | `max-w-5xl`(목록) | 동일 | 동일 | **분할 40:60** | 동일 |
| **1400+** | `max-w-page` 1400 상한 | 동일 | 동일 | 동일 | 동일 |

목록 모드와 지도 모드의 **컨테이너 폭이 다르다** — 목록은 `max-w-5xl`(1024)로 읽기 좋은 줄 길이를 지키고, 지도는 `max-w-page`(1400)로 넓게 쓴다. 지도는 읽는 것이 아니라 보는 것이다.

### 9.2 600px — 카드 높이가 바뀐다 ⚠

썸네일이 `w-28`(112) → `w-40`(160)으로 바뀌면서 카드 높이가 **108 → 144px**로 뛴다. 가상화가 고정 높이를 쓰므로 이 전환은 단순한 CSS 변화가 아니다.

`SearchResultList`가 `useMediaQuery("(min-width: 600px)")`로 감지해 `estimateSize`를 갈아끼우고 `virtualizer.measure()`로 전체 위치를 재계산한다. **이 연결이 끊기면 스크롤 위치가 어긋난다** — 카드 마크업을 바꿀 때 `CARD_HEIGHT`/`CARD_HEIGHT_SM` 상수를 함께 갱신해야 하는 이유다(§1.3).

미디어쿼리 하나가 레이아웃과 가상화 양쪽에 걸려 있다는 점에서, 이 600px 경계는 다른 브레이크포인트보다 깨지기 쉽다.

### 9.3 960px — 구조가 바뀐다

가장 큰 전환이다. 세 가지가 동시에 일어난다.

1. **필터가 시트 → 레일**(목록 모드). `SearchFilterRail`은 `hidden md:block`, `SearchFilterMobile`은 `md:hidden`
2. **모드 토글이 플로팅 → 세그먼트**. 각각 `md:hidden` / `max-md:hidden`
3. **지도 모드가 오버레이 → 분할**. 바텀시트가 사라지고 좌우 2컬럼이 된다

`SearchModeSegment`의 숨김에는 **반드시 미디어쿼리 접두사를 쓴다**(`max-md:hidden`). 컴포넌트가 `inline-flex`를 자체 보유해 무-미디어쿼리 `hidden`은 캐스케이드에서 밀린다.

### 9.4 지도 모드 분할 — 1200px 역전과 개정

실측 계산(지도 모드 컨테이너 `max-w-page` + `sm:px-6` 48, `gap-6` 24):

| 뷰포트 | 비율 | 리스트 폭 | 카드 본문 폭 |
|---|---|---|---|
| 1199 | 45:55 | (1199−48−24)×0.45 = **507** | 507 − 200 = **307** |
| 1200 | 35:65 | (1200−48−24)×0.35 = **395** | 395 − 200 = **195** |

카드 내부 고정폭 200 = `p-3`(24) + 썸네일 `w-40`(160) + `gap-4`(16).

**1200px 경계를 넘는 순간 카드 본문이 307 → 195px로 37% 좁아진다.** 화면이 넓어졌는데 카드가 좁아지는 역전이고, **195px는 모바일(223px)보다도 좁다.** 제목·지역·속성 아이콘 행이 모두 이 폭에 들어가야 한다.

원인은 두 가지가 겹친 것이다 — 비율이 리스트에 불리해지는데(45→35), 썸네일은 `sm` 이상에서 계속 `w-40`(160)로 크다.

**개정 — 1200+ 비율을 `40fr 60fr`로 완화한다.**

| 뷰포트 | 비율 | 리스트 폭 | 카드 본문 폭 |
|---|---|---|---|
| 1199 | 45:55 | 507 | 307 |
| 1200 | **40:60** | (1200−48−24)×0.4 = **451** | **251** |
| 1400 | 40:60 | 1328×0.4 = 531 | 331 |

251px는 모바일(223px)보다 넓어 역전이 해소된다. 대가는 1400px에서 지도가 863 → 797px로 **66px 좁아지는 것**이다.

다른 두 후보를 채택하지 않은 이유:
- **지도 모드에서만 썸네일 축소** — 본문 243px로 비슷한 효과를 내고 카드 높이도 108로 낮아져 후보가 더 보이지만, `SearchResultList`가 모드를 알아야 해 **가상화 상수가 모드에 종속된다**(§1.3의 취약점을 하나 더 늘린다).
- **`minmax(440px, 35fr)` 하한** — 가장 정밀하지만 좁은 1200px대에서만 지도가 밀리는 비선형 동작이라 grid 정의를 읽기 어렵게 만든다.

비율 한 값을 바꾸는 것이 가장 적은 결합으로 문제를 없앤다.

### 9.5 세로 공간 — `100dvh` 프레임

`SearchResultsView`가 `h-[calc(100dvh-64px)]` 고정 프레임이다(64px는 TopBar). 내부는 헤더(`shrink-0`) + 콘텐츠(`min-h-0 flex-1`) 구조로, 스크롤은 **리스트 내부에서만** 일어나고 페이지는 스크롤되지 않는다.

`dvh`를 쓰는 이유는 모바일 브라우저 주소창이 접히고 펴질 때 `vh`가 점프하기 때문이다.

| 구성 | 높이 |
|---|---|
| TopBar | 64 |
| 검색 헤더 (§2.2 개정 후) | ~108 |
| 콘텐츠 (목록 또는 지도) | 나머지 |

667px 기기에서 콘텐츠 영역은 약 **495px**이다. 개정 전(144px 헤더) 대비 36px 늘어난다.

### 9.6 터치 영역 — AA 기준

**기준은 WCAG 2.2 §2.5.8 Target Size (Minimum), Level AA = 24×24 CSS px.** §2.5.5의 44×44는 **Level AAA**이며, 이 프로젝트는 대비비를 포함해 AA를 기준선으로 삼는다.

`text-caption` = 12px / line-height 16px 기준 실측:

| 요소 | 높이 | AA (24) | 판정 |
|---|---|---|---|
| `FilterChip` | `py-1`(8) + 16 + border(2) = **26** | ✅ | 현행 유지 |
| 필터 트리거 | `py-2`(16) + 20 + border(2) = **38** | ✅ | `min-h-11` 추가 |
| 플로팅 토글 | `min-h-11` = 44 | ✅ | — |
| 재시도 버튼 | `min-h-11` = 44 | ✅ | — |

**44px는 단독으로 놓인 주요 액션에만 적용한다** — 플로팅 토글, 재시도 버튼처럼 화면에 하나뿐이고 실패 비용이 큰 요소다. 시트·레일 안에서 `gap-2`로 떨어져 나열되는 칩은 AA 기준으로 충분하며, 44px로 키우면 17개 지역 칩이 레일을 칩 벽으로 만들고 카테고리 칩과의 위계도 사라진다.

> `search-map-explore.md` §7.2가 접근성 산출물로 "44px 터치 영역"을 적어둔 것은 **주요 액션 한정**으로 읽는다. 모든 인터랙티브 요소에 44px를 요구하는 뜻이 아니다.

---

## 7. 핀 · 클러스터 · 미니카드

### 7.1 렌더링 구조

`supercluster`가 bbox·zoom을 받아 클러스터를 계산하고, 우리가 네이버 `Marker`로 그린다. SDK 내장 클러스터링을 쓰지 않는다.

```
지도 경량 응답 (uid · lat · lng · trust_tier)
   ↓ Supercluster.load()
카메라 idle → getClusters(bbox, zoom)
   ↓
feature.properties.cluster === true  → SpotMapCluster (2 이상)
feature.properties.cluster !== true  → SpotMapPin (1)
   ↓ cluster_id / uid 로 diff
기존 Marker 재사용 · 신규만 생성 · 사라진 것만 destroy
```

**diff 기반 재사용이 핵심이다.** `cluster_id`가 안정적이므로 카메라가 움직여도 같은 클러스터는 같은 `Marker` 인스턴스를 유지한다. 이것이 전환 애니메이션과 성능 양쪽의 전제다.

> **하드 제약** — 네이버 공식 FAQ가 *"마커 개수가 200개가 넘으면 현저히 느려진다"* 고 명시한다. 웹 SDK에 캔버스/WebGL 마커 클래스가 없어 **모든 마커가 DOM**이다. `getClusters()` 반환량이 200을 넘지 않도록 `radius`를 조정하고, 실측으로 검증한다.

### 7.2 `SpotMapPin` — 개별 스팟 (count = 1)

| 항목 | 값 |
|---|---|
| 형태 | 물방울 핀, 하단 꼭짓점이 좌표 |
| 크기 | 24×32 (`anchor: {x:12, y:32}`) |
| 아이콘 | `ImageIcon` **스프라이트** — 기본/선택 2프레임을 `origin`으로 클리핑 |
| 색 | `bg-primary` 계열 단색 + `canvas` 테두리 |
| z-index | 기본 `100` |

**모든 핀이 같은 디자인이다. 신뢰도로 구분하지 않는다.**

지도 경량 응답에 `trust_tier`가 오지만 핀에는 반영하지 않는다. 근거 셋:
1. 목 데이터 기준 3등급이 약 20%다. 다섯 중 하나가 경고 표시면 그건 예외가 아니라 배경이 된다.
2. §3.2에서 배지를 "예외를 알리는 장치"로 규정했다. 같은 원칙을 지도에도 적용한다.
3. 지도는 **위치 탐색 도구**다. 신뢰도 판단은 카드·상세에서 한다.

`trust_tier`를 응답에 싣는 이유는 **미니카드가 재조회 없이 "미검증"을 표시하기 위함**이다(§7.5).

**`ImageIcon` 스프라이트를 쓰는 이유** — 네이버 SDK에서 마커는 종류와 무관하게 DOM이지만, `HtmlIcon`은 노드 수가 가장 많다. 동시 표시 200개 제약 아래서는 이미지 마커가 안전하다.

### 7.3 선택 상태

| 상태 | 처리 |
|---|---|
| 기본 | `ImageIcon` 스프라이트 기본 프레임 |
| **선택** | `setIcon()`으로 **`HtmlIcon` 교체** + `setZIndex(1000)` |
| 해제 | `setIcon()`으로 스프라이트 복귀 |

**선택된 1개만 `HtmlIcon`이다.** SDK 내장 애니메이션은 `BOUNCE`/`DROP` 2종 프리셋뿐이라 "선택 시 확대" 같은 임의 애니메이션이 불가능하다. `HtmlIcon` + `marker.getElement()`로 DOM을 잡아 CSS transform을 직접 거는 것이 유일한 경로다.

동시 HTML 마커를 1개로 묶으므로 **200개 제약과 충돌하지 않는다.**

| 항목 | 값 |
|---|---|
| 확대 | `scale(1.25)`, `transform-origin: bottom center` |
| 전환 | 150ms `ease-out` |
| 색 | 반전 — `bg-canvas` 바탕에 `primary` 테두리·아이콘 |

### 7.4 `SpotMapCluster` — 2 이상

확정 구간은 `1 / 2~9 / 10~49 / 50+` 4단계이고, **"1"은 클러스터가 아니라 개별 핀**(§7.2)이므로 클러스터 시각은 **3단계**다.

| 구간 | 지름 | 타이포 | 표기 |
|---|---|---|---|
| 2~9 | 32px | `text-caption` (12) | 숫자 그대로 |
| 10~49 | 40px | `text-body-sm` (14) | 숫자 그대로 |
| 50+ | 48px | `text-body-sm-strong` | `50+` |

| 항목 | 값 |
|---|---|
| 형태 | 원형, `bg-primary text-on-primary` |
| 테두리 | `2px solid canvas` — 지도 위 어떤 바탕에서도 경계가 서게 |
| 그림자 | `shadow-level-2` |
| 아이콘 | `HtmlIcon` — 숫자 텍스트가 필요해 `SymbolIcon` 불가 |
| 앵커 | 중심 |

**구간을 크기로만 표현한다. 색을 바꾸지 않는다** — DESIGN.md Don't(여섯 번째 액센트 금지). 크기 차이가 밀도 차이를 직관적으로 전달한다.

클러스터는 개수가 200개 제약 안에서 관리되므로 `HtmlIcon`을 써도 안전하다.

| 인터랙션 | 동작 |
|---|---|
| 클릭 | `getClusterExpansionZoom()` 줌으로 카메라 이동 — 클러스터가 풀리는 최소 줌 |
| 전환 | 신규 등장 `scale(0.8)→1` + `opacity` **150ms**. 구간 변경은 지름·타이포만 transition **200ms** |

`cluster_id`가 유지되는 한 같은 노드이므로 CSS transition이 실제로 동작한다.

### 7.5 `SpotMiniCard` — 데이터 갭 🔴

핀 선택 시 시트가 피크 상태면 지도 위에 미니카드를 띄운다.

| 항목 | 값 |
|---|---|
| 위치 | 시트 핸들 바로 위, `absolute bottom-14 inset-x-4` |
| 크롬 | `rounded-lg bg-canvas p-3 shadow-level-4` |
| 구성 | 썸네일 64×48 + 제목 + 지역 + 속성 아이콘 4슬롯 + 닫기 |
| 진입 | `translate-y` + `opacity` 150ms |
| 탭 | 상세로 이동 |

**문제 — 지도 응답에 표시할 데이터가 없다.**

`spotMapItemSchema`는 `uid · lat · lng · trust_tier` 4필드뿐이다. **제목도 지역도 썸네일도 없다.**

| 경로 | 문제 |
|---|---|
| 목록 캐시에서 조회 | 목록은 커서 페이징이라 **해당 uid가 아직 로드 안 됐을 수 있다.** 8,000건 중 첫 50건만 있는 상태가 일반적이다 |
| `fetchSpotDetail(uid)` 단건 조회 | 매 핀 클릭마다 왕복. 상세 응답 전체를 받아 미니카드 한 줄에 쓰는 낭비 |

**임시 방침 — 캐시 우선, 미스 시 단건 조회.** `queryClient`에서 목록 캐시를 먼저 뒤지고, 없으면 `fetchSpotDetail`을 호출하되 로딩 중에는 미니카드를 스켈레톤으로 띄운다. 두 번째 클릭부터는 react-query 캐시가 받는다.

근본 해법은 **지도 응답에 최소 표시 필드를 추가**하는 것이지만, 8,000건 × 3필드는 경량 응답의 취지를 해친다. 핵심 속성 4종 추가 요청과 **함께 BE에 물어볼 항목**이다 — "지도 응답에 최소 표시 필드를 넣을지, 미니카드용 벌크 조회를 열지".

### 7.6 핀 ↔ 카드 동기화

| 방향 | 트리거 | 동작 |
|---|---|---|
| 카드 → 핀 | 카드 hover (데스크톱) | 해당 핀 강조 — §7.3 선택과 다른 **약한 상태**(`scale(1.1)`, zIndex만 상승) |
| 카드 → 핀 | 카드 클릭 | 핀 선택 + 카메라 이동 |
| 핀 → 카드 | 핀 클릭 | 리스트 스크롤 + 카드 선택 상태 |
| 핀 → 시트 | 핀 클릭 (모바일) | 시트가 피크면 미니카드, 25% 이상이면 **55%로 자동 전환** |

**핀 → 카드에도 §7.5와 같은 갭이 있다.** `virtualizer.scrollToIndex(i)`로 스크롤하려면 그 uid가 로드된 페이지에 있어야 한다. 없으면 스크롤할 대상이 없다.

이 경우 **리스트를 건드리지 않고 미니카드만 띄운다.** 억지로 페이지를 끝까지 로드해 찾는 동작은 하지 않는다 — 8,000건을 순차 로드하게 된다.

### 7.7 접근성 — 지도는 보조 표현

**네이버 SDK의 마커는 키보드 포커스를 받을 수 없다.** 마커 이벤트에 `keydown`/`focus`가 없고 `MarkerOptions`에 `tabIndex`·`role`·`aria-*`가 없다.

`HtmlIcon`이면 `content`에 `tabindex`를 넣어 우회할 수 있지만, **8,000개 마커에 tab 순서를 부여하는 것 자체가 접근성 안티패턴**이다.

→ **목록이 접근성의 정본(canonical)이고 지도는 보조 표현이다.** 지도로만 도달 가능한 정보를 만들지 않는다. 상세는 §10.

지도 컨테이너에 `role="application"` + `aria-label="스팟 지도"`를 주고, 키보드 사용자에게는 목록 모드를 권하는 `sr-only` 안내를 둔다.

---

## 10. 접근성

기준은 **WCAG 2.2 Level AA**다. AAA는 목표로 삼지 않으며, 이 문서에서 44px 터치 영역처럼 AAA에 속하는 항목을 적용한 곳은 그 이유를 함께 적었다(§9.6).

### 10.1 대비 — `text-mute` 금지 원칙

`text-mute`는 라이트 모드에서 **3.54:1**로 AA(4.5:1) 미달이다. 다크에서는 4.65:1로 통과하므로 **라이트 모드에서만 깨진다** — 다크로 확인하면 문제를 못 본다.

| 용도 | `text-mute` | 근거 |
|---|---|---|
| 본문 텍스트·라벨·수치 | ❌ | 4.5:1 필요 |
| `aria-hidden` 장식 아이콘 | ✅ | 의미를 지지 않음 |
| 인터랙티브 요소의 아이콘 | ✅ | 비텍스트 대비 3:1 요건, 3.54:1로 충족 |
| placeholder | ✅ | 입력 시 사라지고 `aria-label`이 같은 정보를 전달 |

**톤다운은 색이 아니라 크기·굵기로 만든다.** `text-body`(8.1:1 / 다크 7.3:1)에 `text-body-sm`을 조합하면 위계가 충분하다.

이 문서에서 적발한 위반은 §4.5·§5.1·§8.5에 개별 기록했다.

### 10.2 지도는 보조 표현 — 목록이 정본

**네이버 SDK 마커는 키보드 포커스를 받을 수 없다.** 마커 이벤트에 `keydown`/`focus`가 없고 `MarkerOptions`에 `tabIndex`·`role`·`aria-*`가 없다(§7.7).

따라서 **지도로만 도달 가능한 정보나 행동을 만들지 않는다.**

| 지도 요소 | 목록 대응 |
|---|---|
| 핀 클릭 → 상세 | 카드 클릭 → 상세 |
| 핀 선택 → 미니카드 | 카드 자체 |
| 클러스터 클릭 → 줌인 | 해당 없음 — 목록은 전부 나열 |
| "이 지역 재검색" | bbox 필터. 목록 모드에서도 URL로 유지됨 |

| 항목 | 값 |
|---|---|
| 지도 컨테이너 | `role="application"` + `aria-label="스팟 지도"` |
| 안내 | `sr-only`로 "지도는 시각적 탐색용입니다. 목록 모드에서 같은 결과를 키보드로 탐색할 수 있습니다." |
| 모드 토글 | 목록 모드가 기본 진입이므로 키보드 사용자는 별도 조작 없이 접근 가능 |

`role="application"`은 스크린리더의 기본 탐색 모드를 끄므로 남용하면 위험하다. 여기서는 **지도 내부가 통째로 마우스 전용이라는 사실을 명시적으로 알리는 용도**이며, 그 안에 키보드로 도달해야 할 콘텐츠를 두지 않는다는 전제 위에 있다.

### 10.3 상태 변화 알림

| 상황 | 처리 |
|---|---|
| 검색 결과 로딩 | `role="status"` `sr-only` — "검색 결과를 불러오는 중" (§8.1) |
| 결과 수 확정 | 헤더의 `총 {n}곳`을 `aria-live="polite"`로 |
| 다음 페이지 로딩 | 기존 `role="status"` 유지 (§8.3) |
| 필터 변경 | 결과 수 라이브 리전이 받는다 — 별도 알림 불필요 |
| 지도 카메라 이동 | **알리지 않는다** — 300ms 디바운스로도 과도한 소음이 된다 |

스켈레톤은 `aria-hidden`으로 두고 라이브 리전이 대신 말한다. 회색 상자를 읽어줄 이유가 없다.

### 10.4 포커스 관리

| 상황 | 동작 |
|---|---|
| 모달 필터 시트 열림 | 포커스 트랩. 첫 포커스는 시트 제목 |
| 모달 필터 시트 닫힘 | **트리거 버튼으로 복귀** (`triggerRef`) |
| 비모달 결과 시트 | **트랩하지 않는다** — 배경(지도)과 공존이 목적 |
| 비모달 시트에서 ESC | 최저 스냅으로 접기. 이미 최저면 무시 |
| 모드 전환 | 포커스를 옮기지 않는다 — 토글 버튼에 머문다 |

**접힌 비모달 시트가 키보드 함정이 되지 않아야 한다.** 시트가 피크(핸들만) 상태일 때 내부 리스트가 여전히 tab 순서에 남아 있으면, 보이지 않는 카드로 포커스가 들어가 사용자가 길을 잃는다. 피크 상태에서는 시트 내용을 `inert`로 두거나 `tabindex="-1"`을 적용한다.

### 10.5 시맨틱 구조

| 요소 | 마크업 |
|---|---|
| 페이지 제목 | `<h1 class="sr-only">'{query}' 검색 결과</h1>` (§2.2) |
| 결과 목록 | `<ul>` / `<li>` — 가상화해도 리스트 시맨틱 유지 |
| 필터 그룹 | `<fieldset>` + `<legend>` |
| 필터 칩 | `<button aria-pressed>` — 토글이지 링크가 아니다 |
| 모드 토글 | `role="group"` + `aria-label="보기 모드"`, 버튼마다 `aria-pressed` |
| 속성 아이콘 행 | `role="list"`, 각 아이콘 `<li>` + `sr-only` 텍스트, SVG는 `aria-hidden` |

**가상화가 리스트 시맨틱을 깨지 않는다.** 현재 구현은 `paddingTop`/`paddingBottom`으로 앞뒤를 대체하고 `<li>`를 flow에 유지하므로 `absolute` 배치 방식과 달리 `<ul>` 구조가 온전하다.

다만 **DOM에 실제로 존재하는 `<li>` 수가 전체 결과 수와 다르다.** 스크린리더가 "목록, 항목 8개"라고 읽는데 실제로는 200곳일 수 있다. `aria-setsize`/`aria-posinset`을 각 `<li>`에 부여해 전체 규모를 알린다.

### 10.6 접근성 이름 충돌

`SearchModeSegment`의 "지도"/"목록" 버튼과 `SearchModeFloatingToggle`이 같은 이름을 가지면 스크린리더가 구분하지 못한다. 뷰포트로 갈라져 동시 존재하지 않더라도 **행위형 라벨**로 이름을 다르게 둔다 — "지도로 보기" / "목록으로 보기" (§5.2).

### 10.7 검증

| 항목 | 방법 |
|---|---|
| 대비비 | 라이트·다크 양쪽 실측. **라이트에서만 깨지는 경우가 있다** |
| 키보드 탐색 | tab만으로 검색 → 필터 → 결과 → 상세 전 경로 도달 |
| 스크린리더 | 결과 수 변화·로딩이 실제로 읽히는지 |
| 터치 영역 | AA 24×24 기준 (§9.6) |
| 시트 함정 | 피크 상태에서 tab이 보이지 않는 카드로 들어가지 않는지 |

`apps/e2e`에 `search-a11y.spec.ts`가 이미 있다. 위 항목 중 자동화 가능한 것(시맨틱 구조·포커스 복귀·라이브 리전 존재)을 여기에 추가한다.

---

## 관련 문서

- 확정 계약(무엇을 만드는가)은 [`docs/features/search.md`](../../../features/search.md)가 정본이다. 이 문서는 그것이 어떻게 생겼는지만 정의한다.
- 설계 경위·대안 검토·결정 근거는 [`docs/front/projects/search-map-explore.md`](../../../front/projects/search-map-explore.md) 참고.
- 상세 화면 스펙은 형제 문서 [`spot-detail.md`](spot-detail.md). 토큰 체계(§0)와 배지 시스템(§3)을 공유한다.
- 구현 현황은 [`docs/STATUS.md`](../../../STATUS.md) §7.
