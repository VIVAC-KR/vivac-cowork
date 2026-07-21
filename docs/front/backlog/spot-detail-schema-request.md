# 야영장 상세페이지(spot-detail) — BE 스키마 요청

- 작성일: 2026-07-16
- 목적: `GET /v1/explore/spots/{uid}` 응답(`SpotDetail`)에 아래 필드 추가를 BE에 요청
- 배경: 현재 `.openapi.json` 기준 `SpotDetail`은 `uid`, `title`, `address`, `website_url` 4개 필드만 정의돼 있다. 반면 ETL 벌크 적재 스키마(`SpotBulkRow`)에는 아래 필드 대부분이 이미 존재한다 — 신규 수집이 아니라 **기존 데이터를 조회 API에 노출**하는 작업일 가능성이 높다.
- 상세 UI 스펙 근거: [`design/reference/pages/spot-detail.md`](../../design/reference/pages/spot-detail.md), [`docs/backlog/spot-detail-design-followups.md`](spot-detail-design-followups.md)

## 요청 필드 목록

모든 필드는 스팟별로 값이 없을 수 있다는 전제(nullable)로 UI가 설계되어 있다. "SpotBulkRow 존재 여부"는 `.openapi.json` 대조 결과이며 BE 확인 전까지는 추정치다.

| # | FE 필드명 | 타입 | Nullable | 용도 | SpotBulkRow 존재 여부 |
|---|---|---|---|---|---|
| 1 | `name` (title) | string | N | 장소명 (H1) | 존재 (`SpotDetail`에 이미 있음) |
| 2 | `tagline` | string | Y | 한줄설명 | 존재 (`tagline`) |
| 3 | `categories` | string[] | Y (빈 배열) | 카테고리 칩 | 존재 (`category`) |
| 4 | `tags` | string[] | Y (빈 배열) | 태그 칩 | 존재 (`themes`) |
| 5 | `isFeeRequired` | boolean | Y | 유료 여부 배지 | 존재 (`is_fee_required`) |
| 6 | `isPetAllowed` | boolean | Y | 반려동물 허용 배지 | 존재 (`is_pet_allowed`, `pet_policy`) |
| 7 | `note` | string | Y | 특이사항 배너 (자유 텍스트) | 존재 (`features`) |
| 8 | `campSightType` | string | Y | 사이트 유형 | 존재 (`camp_sight_type`) |
| 9 | `unitCount` | number | Y | 사이트 수 | 존재 (`unit_count`) |
| 10 | `totalAreaM2` | number | Y | 총 면적(평균 면적 계산용) | 존재 (`total_area_m2`) |
| 11 | `firePitType` | string | Y | 화로 유형 | 존재 (`fire_pit_type`) |
| 12 | 이용요금 | — | — | 스펙 그리드 5번째 항목 | **시스템 전체 미수집** — 신규 필드 필요 여부 확인 |
| 13 | `latitude` | number | Y | 위치/지도 | 존재 (`latitude`) |
| 14 | `longitude` | number | Y | 위치/지도 | 존재 (`longitude`) |
| 15 | `address` | string | Y | 주소 | 존재 (`SpotDetail`에 이미 있음) |
| 16 | `addressDetail` | string | Y | 상세주소 | 미확인 — 존재 여부 확인 필요 |
| 17 | `description` | string | Y | 상세설명 | 존재 (`description`) |
| 18 | `amenities` | string[] | Y (빈 배열) | 편의시설 목록 | 존재 (`amenities`) |
| 19 | `nearbyFacilities` | string[] | Y (빈 배열) | 주변시설 목록 | 존재 (`nearby_facilities`) |
| 20 | `equipmentRental` | string[] | Y (빈 배열) | 렌탈장비 목록 | 존재 (`has_equipment_rental`) — ⚠️ 필드명은 boolean처럼 보이나 실제 타입이 배열인지 BE 확인 필요 |
| 21 | `phone` | string | Y | 액션바 전화 | 존재 (`phone`) |
| 22 | `websiteUrl` | string | Y | 액션바 웹사이트 | 존재 (`SpotDetail`에 이미 있음) |
| 23 | `bookingUrl` | string | Y | 액션바 예약 링크 | 존재 (`booking_url`) |
| 24 | `imageUrl` | string | Y | 대표 이미지 | 미확인 — 존재 여부 확인 필요 |
| 25 | `ratingAvg` | number | N (기본값 0) | 평점 | 존재 (`rating_avg`, 기본값 0) |
| 26 | `reviewCount` | number | N (기본값 0) | 리뷰 수 | 존재 (`review_count`, 기본값 0) |

## BE에 함께 확인받을 질문

1. `category`(배열)와 `camp_sight_type`(단일 문자열)이 실제로 다른 층위의 분류가 맞는지 — UI는 두 값이 정규화 후 동일하면 중복 표시를 생략하도록 방어 처리했다.
2. `rating_avg`/`review_count`가 "미수집" 상태인지, 아니면 리뷰가 없어 "0"인 정상 데이터 상태인지. 후자라면 UI 문구를 "준비 중"이 아니라 "아직 리뷰가 없어요 · 평점 0.0"으로 바꿔야 한다.
3. 이용요금 필드를 신규로 추가할 계획이 있는지, 있다면 예상 시점.
4. `note`(특이사항)에 구조화된 심각도(`severity`: 예 `info`/`warning`/`critical`) enum을 추가할 수 있는지 — 현재는 단일 텍스트라 UI가 심각도를 구분하지 못한다.
5. `phone`/`websiteUrl`/`bookingUrl` 3개가 모두 null인 스팟이 실제 데이터에서 몇 % 인지 (액션바 승격 규칙의 실효성 판단용).
