# 장소 검색 (search)

> 작성일: 2026-07-28
> 배경: 홈 화면 검색창이 `readOnly` 껍데기라 검색 결과를 볼 동선이 없었습니다. 검색 결과 페이지로 진입하는 라우팅·네비게이션 골격을 먼저 세우고, 실제 검색·필터는 후속으로 분리합니다.

## 1. 한 줄 요약

검색창 제출 시 `/search` 결과 페이지로 이동하는 라우팅·네비게이션 골격을 구축했습니다. 검색 질의·결과 리스트·필터는 아직 미구현이며, 향후 작업으로 남겨둡니다.

## 2. 결정 사항 요약

| 항목 | 결정 | 근거 |
|---|---|---|
| 결과 페이지 라우트 | `/search`, 검색어는 쿼리 파라미터 `q` | BE 계약(`GET /v1/explore/spots?q=`)과 파라미터명 일치, 공유·북마크 가능 |
| 검색 입력 컴포넌트 | `features/search/SearchInput.tsx`로 추출해 홈·결과 페이지 공용 | 두 화면에 같은 검색창이 필요 — 로직 중복 방지. `initialQuery`(결과 페이지 프리필), `className`(상황별 마진) props 제공 |
| 빈 검색어 제출 | 파라미터 없이 `/search`로 이동 | 빈 `q`로 인한 무의미한 쿼리스트링 방지 |
| `HomeSearchBar` | 검색 로직을 `SearchInput`으로 넘기고 서버 컴포넌트로 유지 | 클라이언트 훅은 `SearchInput`에만 필요 — 불필요한 `"use client"` 경계 확대 방지 |

## 3. 구성 요소

| 파일 | 상태 | 역할 |
|---|---|---|
| `apps/web/src/app/search/page.tsx` | 신규 | `/search` 라우트. `searchParams`의 `q`를 읽어 결과 뷰에 전달 |
| `apps/web/src/features/search/SearchResultsView.tsx` | 신규 | 상단 검색창 + 검색어 헤더 + "준비 중" placeholder. 실제 결과 리스트 자리 |
| `apps/web/src/features/search/SearchInput.tsx` | 신규 | 검색 폼·입력·이동 로직(클라이언트 컴포넌트). 홈·결과 페이지 공용 |
| `apps/web/src/features/home/HomeSearchBar.tsx` | 변경 | 인라인 검색 폼을 `SearchInput` 사용으로 교체 |

## 4. 향후 작업 로드맵

### 4.1 검색 결과 리스트 UI (다음 착수 · 우선)

- API를 연결하지 않고 `docs/openapi.json`의 `SpotListItem` 스키마를 참고해 **UI만** 설계합니다.
- 리스트 카드 필드: `uid`, `title`, `trust_tier`, `thumbnail_url`, `region_short`, `category`.
- 응답(`SpotListResponse`)은 `items` · `next_cursor` · `has_more` 구조라 `total`이 없습니다 → cursor 기반 **무한 스크롤**을 지향하되, 이번 단계는 자리(placeholder)만 잡습니다.
- loading · empty 상태를 포함합니다.
- 세부 결정(카드 레이아웃, 스켈레톤, 무한 스크롤 트리거 등)은 착수 과정에서 다시 검토합니다.

### 4.2 검색 결과 필터링 (그 다음)

- 데스크톱은 **필터 레일**, 모바일은 **바텀시트** 형태로 분기합니다.
- 필터 차원은 BE 계약과 일치시킵니다: `category`(다중 선택, `string[]`) · `region_province`(단일).
- 필터 상태는 **URL `searchParams`**로 관리합니다(`/search?q=&category=&region_province=`). 별도 전역 스토어(Zustand)를 도입하지 않아 상태관리 복잡성을 낮게 유지합니다 — `q`와 동일한 방식이라 공유·뒤로가기가 자연스럽게 동작합니다.
- 포함 범위: 0건 empty 상태, 필터 초기화(reset), 활성 필터 개수 배지, 접근성(바텀시트 포커스 트랩 · ESC 닫기 · 배경 스크롤락 · 결과 수 announce).

### 4.3 참조 BE 계약

- 진실 공급원: `docs/openapi.json`의 `GET /v1/explore/spots` (쿼리 `q` · `category` · `region_province` · `cursor` · `limit`).
- 설계 배경: [`docs/core/projects/vvc-105-explore-api-spec.md`](../../core/projects/vvc-105-explore-api-spec.md), [`docs/core/projects/spot-search-postgres-fts.md`](../../core/projects/spot-search-postgres-fts.md). 단, openapi.json과 다른 부분이 있으면 openapi.json을 따릅니다.

### 4.4 열린 결정 · 의존성

- **필터 옵션 값 출처**: 허용 `category` 코드와 `region_province` 목록을 내려주는 공개 엔드포인트가 아직 없습니다(BE 내부적으로는 `spot_field_options` 화이트리스트로 관리). UI 단계에서는 로컬 샘플 값으로 대체하고, 옵션 소스는 착수 시 BE와 확정합니다.
- **`sort`**: 정렬 파라미터는 현재 `openapi.json` 계약에 없습니다(VVC-119 대기). 정렬 UI는 이번 범위에서 제외합니다.
- **추가 필터**: fee/pet/style/difficulty 등은 현재 리스트/필터 계약에 없습니다([VVC-117](https://linear.app/lucente/issue/VVC-117) 대기).
- **반응형 검증**: 브라우저 뷰포트가 1440px로 고정돼 있어 resize로 모바일 미디어쿼리를 검증할 수 없습니다 — 좌표 측정으로 우회합니다.
- 4.1 · 4.2는 착수 시 각각 별도 `projects/` 문서(또는 단일 결정은 `decisions/`)로 승격합니다.

## 5. Out of Scope

- 실제 검색·필터 질의 처리 및 결과 데이터 페칭 (BE 연동)
- `/spots` 목록 라우트
- 검색어 자동완성 · 최근 검색어
- 결과 정렬(`sort`)
