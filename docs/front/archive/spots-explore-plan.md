# ⚠️ Archived

이 문서는 더 이상 유효하지 않습니다.

대체 문서:
없음 — 문서가 설명하는 `apps/web/src/features/spots/`(SpotCard, SpotListView, SpotMapView, SpotExplorer) 구현이 현재 코드베이스에 존재하지 않습니다(확인일 2026-07-15 기준, 홈 화면 개편 과정에서 제거된 것으로 추정). `/spots` 리스트·지도 탐색 페이지를 재구현할 경우 이 문서의 컴포넌트 분해·placeholder 톤 결정을 참고 자료로 활용할 수 있습니다.

Archive Date:
2026-07-15

Reason:
문서가 참조하는 `Spot` 타입, MSW 목 데이터, `features/spots/*` 컴포넌트가 현재 코드베이스에 존재하지 않습니다. `docs/backlog/codebase-review-260714.md`에서 코드와 모순되는 문서로 이미 식별된 바 있습니다.

---

# 장소 탐색 페이지 — 리스트/지도 토글 보기 (디자인 우선) 계획

## 배경 (Context)

`/spots` (장소 탐색) 페이지는 현재 헤더만 있는 빈 골격이다. 야영 가능 장소를
**리스트**와 **지도** 두 가지 보기로 제공하되, 사용자가 토글로 전환할 수 있게 한다.

이번 작업은 **컴포넌트 디자인(마크업 + Tailwind 스타일)만** 구현한다.
실제 데이터 페칭과 상태 관리 연동은 이번 범위에서 제외하고, 후속 작업으로 미룬다.

### 확정된 결정 (이번 범위)

- **지도 SDK는 추후 적용**: 지도 보기는 실제 지도 SDK 없이 placeholder 디자인으로
  구현하고, 나중에 SDK(Kakao/Naver/Leaflet 등 — 후속 선택)를 드롭인할 수 있도록 구조만 잡는다.
- **보기 전환은 토글**: 리스트/지도 중 한 번에 하나만 표시. 데스크톱/모바일 공통.
- **데이터·상태 관리 미연동**: `useSpots`(쿼리)·`useSpotFilterStore`(필터 스토어)는
  **이번엔 연결하지 않는다.** 컴포넌트는 정적 샘플 데이터를 props로 받아 디자인만 확정한다.
  (보기 전환을 위한 로컬 UI 토글 상태는 디자인 상호작용에 필요한 최소 범위로만 사용)
- **필터 UI 제외**: 필터 선택 UI는 만들지 않는다.

### 재사용 자산 (디자인 참고용)

- 도메인 타입: `Spot` 등 (`packages/shared/src/types/index.ts`) — 샘플 데이터 타입으로 사용
- MSW 목 데이터 3건 (`packages/api/src/mocks/handlers.ts`) — 샘플 데이터 모양/내용 참고
- 기존 `apps/web/src/features/spots/SpotList.tsx`는 데이터 연동 데모 골격이므로
  **이번엔 건드리지 않고 그대로 둔다** (후속 데이터 연동 단계에서 새 컴포넌트로 대체 예정).

## 구현 계획 (모두 `apps/web/src/features/spots/` 하위, 신규)

모든 컴포넌트는 컨벤션 준수: 함수 선언문 export, Props는 파일 내 `interface`로 정의,
플랫폼 의존(DOM) 컴포넌트는 `apps/web`에 위치.

### 1. `SpotCard.tsx` — 순수 프레젠테이션 카드

- `interface SpotCardProps { spot: Spot }` (props로 데이터 주입, 자체 페칭 없음).
- 스팟 이름, 야영 가능 여부(`legalStatus`) 배지, 설명, 지역(`region`), 시설 아이콘/태그 정도를
  카드 디자인으로 렌더. 기존 `SpotList.tsx`의 카드 마크업을 디자인 기준선으로 참고.

### 2. `SpotListView.tsx` — 리스트 보기 (프레젠테이션)

- `interface SpotListViewProps { spots: Spot[] }`.
- `spots`를 매핑해 `<SpotCard />` 목록 렌더. 빈 목록 시 "표시할 장소가 없습니다" 안내 디자인 포함.
- 데이터 페칭/로딩/에러 로직 없음 (props만 그림).

### 3. `SpotMapView.tsx` — 지도 보기 placeholder (프레젠테이션)

- `interface SpotMapViewProps { spots: Spot[] }`.
- 실제 지도 SDK가 없으므로 지도 영역을 자리표시 박스로 디자인:
  - "지도 보기는 준비 중입니다 (지도 SDK 연동 예정)" 안내 문구/일러스트 영역
  - 임시로 각 스팟의 이름·좌표(`latitude`, `longitude`)를 핀 카드/목록 형태로 표시
- 후속 SDK 연동 시 이 컴포넌트 내부만 교체하면 되도록 props 인터페이스 유지.

### 4. `SpotExplorer.tsx` — 보기 전환 컨테이너

- `"use client"`. 토글 버튼 2개(`리스트` / `지도`) 디자인 + 선택 강조.
- `useState<"list" | "map">("list")`로 보기 모드 로컬 관리(전역 스토어 미사용).
- 정적 샘플 데이터 배열(`Spot[]`, MSW 목 데이터와 동일한 모양으로 파일 상단에 상수 정의)을
  보유하고, 현재 보기에 따라 `<SpotListView spots={...} />` / `<SpotMapView spots={...} />`에 전달.
- 후속 단계에서 이 샘플 배열을 `useSpots()` 결과로 교체하면 데이터 연동 완료.

### 5. 페이지 연결 — `apps/web/src/app/(protected)/spots/page.tsx` (수정)

- 서버 컴포넌트 유지. 기존 헤더 아래에 `<SpotExplorer />` 렌더.
- `import { SpotExplorer } from "@/features/spots/SpotExplorer";` (feature 내부 barrel 미사용).

## 변경하지 않는 것

- `useSpots`, `useSpotFilterStore`, 스키마, MSW 핸들러 — 변경/연동 없음.
- 기존 `SpotList.tsx` — 그대로 둠.
- 필터 관련 코드 일체.

## 컴포넌트 트리

```
app/(protected)/spots/page.tsx        (server)
└── SpotExplorer.tsx                   (client) — 토글 상태 + 샘플 데이터 보유
    ├── [view = "list"] SpotListView   (props: spots)
    │                   └── SpotCard   (props: spot)
    └── [view = "map"]  SpotMapView    (props: spots) — 지도 placeholder
```

## 검증 방법

1. 개발 서버 실행 후 `/login` → Google 로그인 → `/spots` 접근.
2. 기본 보기가 **리스트**이고 샘플 스팟 카드들이 디자인대로 표시되는지 확인.
3. **지도** 토글 클릭 → placeholder 디자인 + 샘플 스팟 좌표 핀 표시 확인.
4. 토글을 오가며 리스트 ↔ 지도 전환 UI가 정상 동작하는지 확인.
5. 데스크톱/모바일 폭에서 레이아웃이 깨지지 않는지 확인.
6. lint/타입체크 통과 확인.

## 후속 작업 (이번 범위 밖)

- 지도 SDK 선택 및 `SpotMapView` 실제 지도 연동.
- `SpotExplorer`의 샘플 데이터를 `useSpots()`로 교체 (데이터 연동).
- `useSpotFilterStore` 기반 필터 UI 구현.
