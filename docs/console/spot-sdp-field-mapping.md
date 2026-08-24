# Spot / Business Info SDP — 화면 필드 ↔ DB 컬럼 매핑

> **목적**: 콘솔의 Spot SDP(`/spots`, `/spots/[uid]/edit`)와 Business Info SDP
> (`/spot-business-info`, `/spot-business-info/[uid]/edit`) 화면에 노출되는
> 한글 라벨과, 그 값이 실제로 어느 DB 컬럼에서 오는지 1:1로 정리한 문서다.
>
> **쓰임**:
> - 사람이 볼 때: 화면의 "한줄설명"이 DB 어느 컬럼인지 즉시 찾기 위함.
> - AI agent가 작업할 때: "화로 유형 필드 고쳐줘" 같은 한글 요청을 받았을 때
>   `fire_pit_type` 컬럼/필드명으로 바로 매핑해 코드를 찾도록 돕기 위함. 반대로
>   API 응답 필드명만 보고 화면 어디에 뜨는지 역추적할 때도 사용.
> - 콘솔은 DB에 직접 접근하지 않고 `vivacapi-core`의 `/v1/internal/*` API만
>   호출한다 (`vivac-console/CLAUDE.md`). 아래 "DB 컬럼"은 API 응답 JSON 키이며,
>   `vivacapi-core`의 `vivacapi/models/spot.py`, `vivacapi/models/spot_business_info.py`
>   SQLAlchemy 모델과 대조해 실제 테이블 컬럼명과 동일함을 확인했다 (2026-07-14 기준).
> - 컬럼이 추가/변경되면 이 문서도 같이 갱신할 것 — 소스 오브 트루스는 어디까지나
>   `vivacapi-core`의 모델 정의다.

## 1. Spot SDP — `spots` 테이블

### 1.1 목록 화면 (`/spots`, `src/app/(admin)/spots/page.tsx`)

| 화면 라벨 | DB 컬럼 | 타입 | 비고 |
|---|---|---|---|
| 이름 | `title` | string | 정렬 가능 |
| 상태 | `pipeline_status` | string (enum) | `RAW`\|`ENRICHED`\|`CURATED`\|`REVIEWED`\|`PUBLISHED`\|`REJECTED`. 필터 패싯 |
| 소스 | `source` | string \| null | 필터 패싯 |
| 도/광역시 | `region_province` | string \| null | 정렬 가능, 필터 패싯 |
| 시/군/구 | `region_city` | string \| null | |
| 평점 | `rating_avg` | float | 정렬 가능 |
| 리뷰 | `review_count` | int | 정렬 가능 |
| 수정일 | `updated_at` | datetime \| null | 정렬 가능, 기본 정렬 기준 |

### 1.2 편집 화면 (`/spots/[uid]/edit`, `src/components/admin/spot-edit-form.tsx`)

섹션 구분은 화면 UI의 `<section>` 구획과 동일하다.

**기본 정보**

| 화면 라벨 | DB 컬럼 | 타입 | 비고 |
|---|---|---|---|
| 이름 | `title` | string | 필수 |
| 한줄설명 | `tagline` | string \| null | |
| 설명 | `description` | string \| null | |
| 카테고리 | `category` | string[] \| null | 콤마 구분 태그로 편집 |
| 태그 | `themes` | string[] \| null | 콤마 구분 태그로 편집. 화면 라벨은 "태그"지만 DB 컬럼명은 `themes` |

**연락처 / 링크**

| 화면 라벨 | DB 컬럼 | 타입 | 비고 |
|---|---|---|---|
| 전화번호 | `phone` | string \| null | 숫자·하이픈만 허용 |
| 웹사이트 | `website_url` | string \| null | |
| 예약 링크 | `booking_url` | string \| null | |

**위치**

| 화면 라벨 | DB 컬럼 | 타입 | 비고 |
|---|---|---|---|
| 주소 | `address` | string \| null | |
| 상세 주소 | `address_detail` | string \| null | |
| 도/광역시 | `region_province` | string \| null | |
| 시/군/구 | `region_city` | string \| null | |
| 우편번호 | `postal_code` | string \| null | |
| 위도 | `latitude` | float \| null | |
| 경도 | `longitude` | float \| null | |
| 고도(m) | `altitude` | float \| null | |

**시설 / 정책**

| 화면 라벨 | DB 컬럼 | 타입 | 비고 |
|---|---|---|---|
| 유료 여부 | `is_fee_required` | boolean \| null | |
| 반려동물 허용 | `is_pet_allowed` | boolean \| null | |
| 반려동물 정책 | `pet_policy` | string \| null | |
| 사이트 수 | `unit_count` | int \| null | |
| 총 면적(㎡) | `total_area_m2` | float \| null | |
| 화로 유형 | `fire_pit_type` | string \| null | |
| 사이트 유형 | `camp_sight_type` | string \| null | |
| 편의시설 | `amenities` | string[] \| null | 콤마 구분 태그로 편집 |
| 주변 시설 | `nearby_facilities` | string[] \| null | 콤마 구분 태그로 편집 |
| 렌탈 장비 | `has_equipment_rental` | string[] \| null | ⚠️ 컬럼명은 `has_*`(bool처럼 보임)지만 실제 타입은 대여 가능 장비 목록 배열 |
| 특이사항 | `features` | string \| null | |
| 배상책임보험 | `has_liability_insurance` | boolean \| null | |

### 1.3 편집 화면 상단 — 라벨 없이 텍스트로만 노출

`Field` 컴포넌트 라벨이 아니라 페이지 헤더에 문장 형태로 박혀 있는 값들.

| 화면 표기 | DB 컬럼 | 비고 |
|---|---|---|
| `UID: {값}` | `uid` | PK, 22자 shortuuid |
| `파이프라인 상태` 배지 | `pipeline_status` | 편집 폼 상단 배지 (목록의 "상태"와 동일 컬럼) |
| `소스: {값}` | `source` | |
| `ID: {값}` | `external_id` | 소스 시스템 원본 ID |
| `수정일: {값}` | `updated_at` | |

## 2. Business Info SDP — `spot_business_info` 테이블

### 2.1 목록 화면 (`/spot-business-info`, `src/app/(admin)/spot-business-info/page.tsx`)

| 화면 라벨 | DB 컬럼 | 타입 | 비고 |
|---|---|---|---|
| 스팟 | `spot_uid` → `spots.title` | string | `spot_uid`로 조인된 `spots.title` 표시, 클릭 시 `/spots/{spot_uid}/edit`로 이동 |
| 사업유형 | `business_type` | string \| null | 정렬 가능 |
| 운영상태 | `operating_status` | string \| null | 정렬 가능 |
| 수정일 | `updated_at` | datetime \| null | 정렬 가능, 기본 정렬 기준 |

### 2.2 편집 화면 (`/spot-business-info/[uid]/edit`, `src/components/admin/sbi-edit-form.tsx`)

**사업자 정보**

| 화면 라벨 | DB 컬럼 | 타입 | 비고 |
|---|---|---|---|
| 사업자 등록번호 | `business_reg_no` | string \| null | |
| 관광사업 등록번호 | `tourism_business_reg_no` | string \| null | |
| 사업유형 | `business_type` | string \| null | |
| 운영유형 | `operation_type` | string \| null | |
| 운영기관 | `operating_agency` | string \| null | |
| 운영상태 | `operating_status` | string \| null | |
| 허가일 | `licensed_at` | date \| null | |

**국립공원**

| 화면 라벨 | DB 컬럼 | 타입 | 비고 |
|---|---|---|---|
| 국립공원 번호 | `national_park_no` | int \| null | |
| 사무소 코드 | `national_park_office_code` | string \| null | |
| 일련번호 | `national_park_serial_no` | string \| null | |
| 카테고리 코드 | `national_park_category_code` | string \| null | |

### 2.3 편집 화면 상단 — 라벨 없이 텍스트로만 노출

| 화면 표기 | DB 컬럼 | 비고 |
|---|---|---|
| `Spot: {값}` (링크) | `spot_uid` | FK → `spots.uid`, unique (spot당 business info 최대 1건) |
| `수정일: {값}` | `updated_at` | |

## 3. 화면에 노출되지 않는 필드 (참고용)

콘솔 화면 어디에도 한글 라벨로 뜨지 않지만 코드/DB에는 존재해서 AI agent가
헷갈릴 수 있는 필드들.

| 필드 | 테이블 | 비고 |
|---|---|---|
| `assigned_to_uid` | `spots` | 검증 담당 staff FK(`users.uid`). "제출/반려" 버튼 노출 여부 판단에만 쓰이고(`spot-edit-form.tsx`), 목록 필터 쿼리 파라미터(`assigned_to_uid`)로만 존재. 화면에 값 자체를 보여주는 라벨은 없음 |
| `created_at` | `spots`, `spot_business_info` | API 응답에는 있으나 두 SDP 어디에도 렌더링되지 않음 |
| `trust_tier` | `spots` | DB/모델에는 존재하나 콘솔 `SpotDetail` 타입에 아예 없음 — 콘솔에서 조회·수정 불가 |
| `uid` (business info) | `spot_business_info` | PK. 화면엔 URL 경로 세그먼트로만 쓰이고 라벨로 노출되진 않음 |
