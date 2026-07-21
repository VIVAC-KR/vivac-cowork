# source1 → spots / spot_business_info 변환 작업 기록

> 작성일: 2026-05-21
> 변환 스크립트: `scripts/transform_source1.py`
> 입력: `data/source1.json` (GoCamping 야영장 데이터, 2,897건)
> 출력:
> - `data/source1_spots.json` — `spots` 테이블 bulk insert용
> - `data/source1_spot_business_info.json` — `spot_business_info` 테이블 bulk insert용

---

## 1. 작업 목표

GoCamping API에서 수집한 원천 JSON(`source1.json`)을 다음 두 PostgreSQL 테이블에 bulk insert 가능한 JSON 배열 형태로 변환:

- `spots` (마이그레이션 `62795cd8d6c0` 적용 — `source`, `external_id` 컬럼 포함)
- `spot_business_info` (FK: `spot_uid → spots.uid`)

또한 Notion에 정의된 enum 매핑을 적용하여 한글 원본 값을 enum 코드로 변환.

---

## 2. 필드 매핑

### 2.1 spots

| DB 컬럼 | 원천 필드 | 변환 |
|---|---|---|
| `uid` | — | `uuid5(NS_SPOT, "{src}:{contentId}")` 결정적 UUID |
| `title` | `facltNm` | 그대로 |
| `address` | `addr1` | 빈 문자열 → null |
| `address_detail` | `addr2` | 빈 문자열 → null |
| `region_province` | `doNm` | 그대로 |
| `region_city` | `sigunguNm` | 그대로 |
| `postal_code` | `zipcode` | 그대로 |
| `phone` | `tel` | 빈 문자열 → null |
| `description` | `intro` | 빈 문자열 → null |
| `tagline` | `lineIntro` | 빈 문자열 → null |
| `latitude` | `mapY` | 문자열 → float |
| `longitude` | `mapX` | 문자열 → float |
| `altitude` | — | null |
| `unit_count` | — | null |
| `is_fee_required` | — | null |
| `is_pet_allowed` | `animalCmgCl` | "가능"→true, "불가능"→false, else null |
| `pet_policy` | — | null |
| `has_equipment_rental` | `eqpmnLendCl` | 쉼표 split → enum 배열 |
| `themes` | `themaEnvrnCl` | 쉼표 split → enum 배열 |
| `fire_pit_type` | `brazierCl` | enum 단일값 |
| `amenities` | `sbrsCl` | 쉼표 split → enum 배열 |
| `nearby_facilities` | `posblFcltyCl` | 쉼표 split → enum 배열 |
| `camp_sight_type` | — | null |
| `rating_avg` | — | `0.0` (NOT NULL) |
| `review_count` | — | `0` (NOT NULL) |
| `website_url` | `homepage` | 빈 문자열 → null |
| `booking_url` | `resveUrl` | 빈 문자열 → null |
| `features` | `featureNm` | 그대로 |
| `category` | `induty` | 단일 값을 1-요소 배열로 wrap |
| `total_area_m2` | `allar` | 문자열 → float |
| `has_liability_insurance` | `insrncAt` | "Y"→true, "N"→false |
| `source` | `src` | "src1" |
| `external_id` | `contentId` | 그대로 |

### 2.2 spot_business_info

| DB 컬럼 | 원천 필드 | 변환 |
|---|---|---|
| `uid` | — | `uuid5(NS_BIZ, "{src}:{contentId}")` |
| `spot_uid` | — | 동일 contentId의 `spots.uid` |
| `business_reg_no` | `bizrno` | 그대로 |
| `tourism_business_reg_no` | `trsagntNo` | 그대로 |
| `business_type` | `facltDivNm` | enum 변환 |
| `operation_type` | `mangeDivNm` | enum 변환 |
| `operating_agency` | `mgcDiv` | 빈 문자열 → null |
| `operating_status` | `manageSttus` | enum 변환 |
| `national_park_no` | — | null |
| `national_park_office_code` | — | null |
| `national_park_serial_no` | — | null |
| `national_park_category_code` | — | null |
| `licensed_at` | `prmisnDe` | ISO 날짜 문자열 |

---

## 3. Enum 매핑

Notion 참조:
- Spots: <https://www.notion.so/Spots-3476249ea05a80b5b7b1d014c52cffc8>
- enum_mappings: <https://www.notion.so/enum_mappings-3476249ea05a80dc8738f45f9709b59a>

### 3.1 business_type (`facltDivNm`)

| 원본 | enum |
|---|---|
| 민간 | `PRIVATE` |
| 공립 | `PUBLIC_LOCAL` |
| 국립 | `PUBLIC_NATIONAL` |
| 지자체 | `LOCAL_GOVERNMENT` |
| 자연휴양림 | `FOREST_RECREATION` |
| 국립공원 | `NATIONAL_PARK` |
| 국민여가 | `PUBLIC_LEISURE` |
| (빈 값) | `null` |

### 3.2 operation_type (`mangeDivNm`)

| 원본 | enum |
|---|---|
| 직영 | `SELF_OPERATED` |
| 위탁 | `OUTSOURCED` |

### 3.3 operating_status (`manageSttus`)

| 원본 | enum |
|---|---|
| 운영 | `OPEN` |
| 휴장 | `TEMPORARILY_CLOSED` |
| 폐업 | `PERMANENTLY_CLOSED` |

### 3.4 fire_pit_type (`brazierCl`)

| 원본 | enum |
|---|---|
| 개별 | `INDIVIDUAL` |
| 공동취사장 | `SHARED` |
| 불가 | `NOT_ALLOWED` |
| (빈 값) | `null` |

### 3.5 themes (`themaEnvrnCl`, list)

| 원본 | enum |
|---|---|
| 여름물놀이 | `SUMMER_SWIMMING` |
| 가을단풍명소 | `AUTUMN_FOLIAGE` |
| 걷기길 | `HIKING_TRAIL` |
| 봄꽃여행 | `SPRING_FLOWERS` |
| 겨울눈꽃명소 | `WINTER_SNOW` |
| 일몰명소 | `SUNSET` |
| 낚시 | `FISHING` |
| 일출명소 | `SUNRISE` |
| 수상레저 | `WATER_SPORTS` |
| 액티비티 | `ACTIVITY` |
| 스키 | `SKI` |
| 항공레저 | `AIR_SPORTS` |

### 3.6 nearby_facilities (`posblFcltyCl`, list)

| 원본 | enum |
|---|---|
| 산책로 | `TRAIL` |
| 계곡 물놀이 | `VALLEY_SWIMMING` |
| 낚시 | `FISHING` |
| 어린이놀이시설 | `PLAYGROUND` |
| 강/물놀이 | `RIVER_SWIMMING` |
| 해수욕 | `BEACH_SWIMMING` |
| 농어촌체험시설 | `RURAL_EXPERIENCE` |
| 수상레저 | `WATER_SPORTS` |
| 운동장 | `SPORTS_FIELD` |
| 청소년체험시설 | `YOUTH_EXPERIENCE` |
| 수영장 | `SWIMMING_POOL` |

### 3.7 has_equipment_rental (`eqpmnLendCl`, list)

| 원본 | enum |
|---|---|
| 화로대 | `FIRE_PIT` |
| 릴선 | `EXTENSION_CORD` |
| 난방기구 | `HEATER` |
| 텐트 | `TENT` |
| 식기 | `COOKWARE` |
| 침낭 | `SLEEPING_BAG` |
| 일산화탄소감지기 | `CO_DETECTOR` |

### 3.8 amenities (`sbrsCl`, list)

| 원본 | enum |
|---|---|
| 전기 | `ELECTRICITY` |
| 온수 | `HOT_WATER` |
| 장작판매 | `FIREWOOD_SALE` |
| 무선인터넷 | `WIFI` |
| 운동시설 | `FITNESS` |
| 물놀이장 | `WATER_PLAYGROUND` |
| 놀이터 | `PLAYGROUND` |
| 마트.편의점 | `CONVENIENCE_STORE` |
| 산책로 | `TRAIL` |
| 트렘폴린 | `TRAMPOLINE` |
| 운동장 | `SPORTS_FIELD` |
| 덤프스테이션 | `DUMP_STATION` |

---

## 4. 주요 변환 규칙

- **빈 문자열 정규화**: 원천 JSON의 `""` 값은 모두 `null`로 변환.
- **boolean 변환**: `"Y"/"N"` → `true/false`; `"가능"/"불가능"` → `true/false`. 그 외는 `null`.
- **숫자 변환**: `allar`, `mapX`, `mapY`는 문자열 → `float`. 변환 실패 시 `null`.
- **리스트 split**: `eqpmnLendCl`, `themaEnvrnCl`, `sbrsCl`, `posblFcltyCl`는 쉼표 split 후 각 요소를 enum 매핑. 빈 결과는 `null`.
- **UUID 생성**: `uuid5`를 사용한 결정적 UUID — 재실행해도 동일 spot은 동일 uid 유지.
    - `spots.uid` 네임스페이스: `00000000-0000-0000-0000-000000000001`
    - `spot_business_info.uid` 네임스페이스: `00000000-0000-0000-0000-000000000002`
    - seed: `"{src}:{contentId}"` (예: `"src1:1984"`)
- **알 수 없는 enum 값**: 스크립트가 추적하여 종료 시 경고 출력. 모든 source1.json 값은 정의된 enum 안에 포함됨을 확인.

---

## 5. 검증 결과

```
spots: 2897 -> data/source1_spots.json
spot_business_info: 2897 -> data/source1_spot_business_info.json
All enum values mapped successfully.
```

- `spots.uid` 유일성 ✅
- `(source, external_id)` 유일성 ✅ (마이그레이션 `62795cd8d6c0`의 unique 제약 만족)
- `spot_business_info.uid` 유일성 ✅
- 모든 `spot_business_info.spot_uid`가 `spots.uid`에 존재 (FK 무결성 보장) ✅
- 알 수 없는 enum 값 0건

### 원천 데이터 enum 분포 확인 결과

| 필드 | 분포 |
|---|---|
| `facltDivNm` | 민간 2,385 / 지자체 253 / (빈) 142 / 국립공원 37 / 공립 33 / 자연휴양림 28 / 국민여가 16 / 국립 3 |
| `mangeDivNm` | 직영 2,778 / 위탁 119 |
| `manageSttus` | 운영 2,745 / 휴장 152 |
| `brazierCl` | 개별 2,243 / (빈) 551 / 불가 66 / 공동취사장 37 |

리스트형 enum 필드도 모두 정의 범위 안에 있음을 확인.

---

## 6. 샘플 출력

### spots 샘플 (contentId=1984, "아빠가 만든 캠핑장")

```json
{
  "uid": "c6b26a33-a88e-5397-80c3-81e05ab99b76",
  "title": "아빠가 만든 캠핑장",
  "address": "강원특별자치도 영월군 북면 덕전길 240",
  "region_province": "강원특별자치도",
  "region_city": "영월군",
  "postal_code": "26208",
  "latitude": 37.2556317630274,
  "longitude": 128.393013232493,
  "is_pet_allowed": false,
  "fire_pit_type": "INDIVIDUAL",
  "amenities": ["ELECTRICITY","WIFI","FIREWOOD_SALE","HOT_WATER","WATER_PLAYGROUND","PLAYGROUND","FITNESS"],
  "themes": ["SUMMER_SWIMMING"],
  "has_equipment_rental": ["TENT","EXTENSION_CORD","FIRE_PIT","HEATER"],
  "nearby_facilities": ["VALLEY_SWIMMING","YOUTH_EXPERIENCE"],
  "rating_avg": 0.0,
  "review_count": 0,
  "category": ["일반야영장"],
  "total_area_m2": 4000.0,
  "has_liability_insurance": true,
  "source": "src1",
  "external_id": "1984"
}
```

### spot_business_info 샘플

```json
{
  "uid": "deed9f81-1f4f-5181-bfd6-d9bee1c1ce71",
  "spot_uid": "c6b26a33-a88e-5397-80c3-81e05ab99b76",
  "business_reg_no": "225-04-55378",
  "tourism_business_reg_no": "2015000031",
  "business_type": "PRIVATE",
  "operation_type": "SELF_OPERATED",
  "operating_status": "OPEN",
  "licensed_at": "2015-06-29"
}
```

---

## 7. 재실행 방법

```bash
python3 scripts/transform_source1.py
```

결정적 UUID 사용으로 idempotent — 동일 입력은 동일 출력을 생성하므로 upsert 시 안전.

---

## 8. 향후 개선 여지 (해결 안 됨)

- `category`는 현재 `induty` 단일값을 1-요소 배열로 wrap 중. 향후 카테고리 enum이 정의되면 매핑 추가 필요.
- 다음 필드는 매핑 미정의 상태로 `null`:
    - `altitude`, `unit_count`, `is_fee_required`, `pet_policy`, `camp_sight_type`
    - `national_park_*` (4개) — `mgcDiv`(operating_agency)가 채워질 때 함께 검토 필요할 수 있음.
- 원천에는 `gnrlSiteCo`/`autoSiteCo`/`glampSiteCo`/`caravSiteCo`/`indvdlCaravSiteCo` 등 사이트 수 필드가 있음 — `unit_count`로 합산하는 정책이 정해지면 적용 가능.
