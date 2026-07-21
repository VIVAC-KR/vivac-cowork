# source1 변환 수정 이력 (Changelog)

> 원천: `data/source1.json` (GoCamping API basedList, 2,897건)
> 산출물: `data/source1_spots.json`, `data/source1_spot_business_info.json`
> 변환 스크립트: `scripts/transform_source1.py`

원본 raw data(`data/source1.json`)는 어떠한 경우에도 직접 수정하지 않습니다. 모든 정제는 변환 스크립트에서 일어나며, 본 문서는 그 정제 규칙의 추가/변경 이력을 시간 순으로 기록합니다.

---

## v1.0 — 2026-05-21 (초기 변환)

관련 커밋: `2642b96 feat(source1): transform raw JSON to spots/spot_business_info bulk insert format`

### 1. 결정적 UID 생성
- `spots.uid`: `uuid5(NS_SPOT="00000000-0000-0000-0000-000000000001", f"{src}:{contentId}")`
- `spot_business_info.uid`: `uuid5(NS_BIZ="00000000-0000-0000-0000-000000000002", f"{src}:{contentId}")`
- 재실행 시 동일한 contentId는 동일한 uid를 가짐 (멱등성 확보).

### 2. 기본 정제 규칙
- **빈 문자열 / 공백만 있는 값** → `None`
- **`Y` / `N`** → `True` / `False` (`has_liability_insurance` ← `insrncAt`)
- **`가능` / `불가능`** → `True` / `False` (`is_pet_allowed` ← `animalCmgCl`)
- **숫자형 변환 실패** → `None` (`latitude`, `longitude`, `total_area_m2`)

### 3. 좌표 매핑
- `latitude` ← `mapY` (float)
- `longitude` ← `mapX` (float)

### 4. 카테고리 매핑 (초기)
- `category` ← `[induty]` (단일 값을 1-요소 배열로 wrap)
  - ⚠️ v1.1에서 화이트리스트 split 방식으로 변경됨

### 5. NOT NULL 기본값
- `rating_avg = 0.0`
- `review_count = 0`
- (스키마상 NOT NULL이지만 원천에 값이 없어 기본값 채움)

### 6. Enum 매핑 (Notion 페이지 기준)
8개 enum 컬럼에 코드 매핑 적용:
- `business_type` (BUSINESS_TYPE): `민간/공립/국립/지자체/자연휴양림/국립공원/국민여가` → `PRIVATE/PUBLIC_LOCAL/...`
- `operation_type` (OPERATION_TYPE): `직영/위탁` → `SELF_OPERATED/OUTSOURCED`
- `operating_status` (OPERATING_STATUS): `운영/휴장/폐업` → `OPEN/TEMPORARILY_CLOSED/PERMANENTLY_CLOSED`
- `fire_pit_type` (FIRE_PIT_TYPE): `개별/공동취사장/불가` → `INDIVIDUAL/SHARED/NOT_ALLOWED`
- `themes` (THEMES): 12종 한국어 → 영문 코드
- `nearby_facilities` (NEARBY_FACILITIES): 11종 한국어 → 영문 코드
- `has_equipment_rental` (HAS_EQUIPMENT_RENTAL): 7종 한국어 → 영문 코드
- `amenities` (AMENITIES): 12종 한국어 → 영문 코드

매핑 불가 값은 `null`로 처리하고 unmapped 로그 출력.

### 7. 날짜 매핑
- `licensed_at` ← `prmisnDe` (ISO 8601 yyyy-mm-dd 문자열)

### 8. 좌석/면적
- `total_area_m2` ← `allar` (float)
  - ⚠️ v1.1에서 `0` → `null` 처리 추가

---

## v1.1 — 2026-05-21 (P0/P1/P2 데이터 품질 수정)

관련 검토 문서: `docs/source1_transform_review.md`
관련 메모: `memory/project_source1_pending_fixes.md`

검토 결과 발견된 데이터 품질 이슈 중 **8건**을 본 변환 스크립트에 반영합니다. (P2-5 region_city는 원천 결함이라 후속 작업.)

### P0-1. `category` 화이트리스트 + 쉼표 split

- **변경 전**: `induty` 원문 값을 1-요소 배열로 wrap
  - 예: `"일반야영장,침대,TV,에어컨,..."` → `["일반야영장,침대,TV,에어컨,..."]`
- **변경 후**: 쉼표 split 후 화이트리스트 필터링
  - 화이트리스트: `일반야영장`, `자동차야영장`, `카라반`, `글램핑`, `캠프닉`
  - 예: `"일반야영장,카라반,글램핑"` → `["일반야영장","카라반","글램핑"]`
  - 예: `"일반야영장,침대,TV,..."` → `["일반야영장"]`
- **부대시설 노이즈 처리**: `induty`에서 제거되는 부대시설 토큰(`침대`, `TV`, `에어컨`, `냉장고`, `유무선인터넷`, `난방기구`, `취사도구`, `내부화장실`, `내부샤워실` 등)은 이미 `sbrsCl`/`posblFcltyCl` 원천 필드를 통해 `amenities`/`nearby_facilities`에 매핑되고 있으므로 정보 손실 없음.

### P0-2. `is_pet_allowed` "가능(소형견)" 보존 (A안)

- **변경 전**: `"가능(소형견)"` 457건이 `null`로 손실
- **변경 후**:
  - `is_pet_allowed = True`
  - `pet_policy = "SMALL_DOG_ONLY"`
- 영향: 펫 가능 정보 보존 (`is_pet_allowed` null 비율 29.7% → ~14%)

### P1-1. `business_reg_no` 정규화

- **변경 전**: `bizrno` 원문 그대로
- **변경 후** (~60건 정제):
  1. `strip()` 적용
  2. 모든 공백 제거: `re.sub(r"\s+", "", v)`
  3. 한국 사업자등록번호 패턴 `\d{3}-\d{2}-\d{5}` 매칭 검증
  4. 비매칭 값(`"제99-99-9999호"`, `"관광사-업자-업자 없음"` 등)은 `null`로 폐기
- 예: `"999-99-9999 "` → `"999-99-99999"` 검증 후 보존

### P1-2. URL 정규화

- **대상 컬럼**: `website_url` (~174건), `booking_url` (~140건)
- **변경 후** 처리 순서:
  1. `strip()` 적용
  2. `http://` 또는 `https://` 시작 → 그대로 보존
  3. `www.` 시작 → `https://` prefix 추가
  4. 알려진 단축링크(`naver.me/`, `bit.ly/`, `me2.do/` 등) → `https://` prefix 추가
  5. 그 외 (한글 텍스트, 도메인 형식 아님 등) → `null`
- 예: `" https://jscamp.three-four.co.kr/"` → `"https://jscamp.three-four.co.kr/"`
- 예: `"www.campingkorea.or.kr"` → `"https://www.campingkorea.or.kr"`
- 예: `"naver.me/xFrKvXPn"` → `"https://naver.me/xFrKvXPn"`
- 예: `"리버뷰캠핑장 "` → `null`

### P2-1. `total_area_m2 = 0 → null`

- **변경 전**: `allar = "0"` 1,777건이 `0.0`으로 저장
- **변경 후**: `to_float()` 결과가 `0.0`인 경우 `null` 처리
- 이유: 의미상 "미입력"을 면적 0m²로 잘못 표기한 것을 정정

### P2-3. `phone` 트레일링 `~N` 절단

- **변경 전**: `"033-671-4568~9"` 등 4건이 원문 그대로
- **변경 후**: `re.sub(r"~\d+$", "", v)` → 시작 번호만 보존
- 예: `"033-671-4568~9"` → `"033-671-4568"`

### P2-4. `features` 트레일링 콤마 제거

- **변경 전**: `"덕전마을 안쪽,"` 등 2건이 끝에 쉼표 잔존
- **변경 후**: `v.rstrip(", ")` 적용
- 예: `"덕전마을 안쪽,"` → `"덕전마을 안쪽"`

---

### P2-2. `postal_code` 6자리 → 5자리 변환 (도로명주소 lookup 방식)

- **변경 전**: 6자리 199건, 7자리 6건, 빈값 18건 — 5자리는 2,674건뿐
- **변경 후**: 5자리 2,801건(96.7%), null 96건(3.3%), 6자리/7자리 잔존 0건

**처리 방식**:
1. **신 우편번호 lookup DB 빌드** — `scripts/build_postal_lookup.py`
   - 원천: 우정사업본부 "지역별 주소 DB" (zipcode_DB.zip, 18개 권역 텍스트, UTF-8, 파이프 구분, 26 컬럼)
   - 산출: `data/postal_lookup.sqlite` (6,451,266 행, 34,401 고유 우편번호, ~1.2GB)
   - 인덱스: `(province, city, road_name, bldg_no)`
2. **`zipcode` 정규화 로직**:
   - 5자리(`\d{5}`) → 그대로 보존
   - 그 외(6자리/7자리/빈값) → `addr1` 파싱 후 lookup
3. **`addr1` 파싱**: `province / city(1~2 토큰) / town(읍·면, optional) / road_name(~로/길/대로) / bldg_no` 추출. 시도 약어(`충북` → `충청북도` 등) 정규화 포함.
4. **lookup fallback 순서**: 정확 매칭 → `bldg_no_sub=0` → town 무시 → city+town 무시.
5. **매칭 실패 → `null`** (보수적 정책. 추측 매칭으로 잘못된 우편번호를 넣지 않음.)

**남은 96건의 null 사유**:
- 지번주소 혼입 (예: `"부산광역시 강서구 대저1동 1-5번지"` — 도로명 없음)
- 행정구역 분구 표기 차이 (예: source1 "화성시" vs DB "화성시 만세구")
- 도로명+번호가 DB에 등록되지 않은 신규/오기 주소

> **참고**: 사용자가 별도로 제공한 `data/postal_code_old` (구 우편번호 DB)는 본 변환에 사용되지 않았음. 신 DB의 `구우편번호` 컬럼은 헤더에만 있고 값이 비어있어 직접 (구→신) 매핑이 불가했고, source1.json에 이미 도로명주소(`addr1`)가 있어 도로명 lookup 경로가 더 정확/직접적이었음.

---

## 후속 작업

### P2-5. `region_city = null` 38건

원천(`sigunguNm`)이 빈 값인 경우. 변환 단계에서 보강 불가.

**처리 계획**:
1. `region_province + address`로 `sigunguNm` 역추적 (postal lookup DB의 도로명→city 활용 가능)
2. 또는 원천 보강 후 재크롤링

---

## v1.1 적용 결과 (검증)

`scripts/transform_source1.py` 재실행 후 산출물(`data/source1_spots.json`, `data/source1_spot_business_info.json`) 검증:

| 항목 | v1.0 | v1.1 | 변화 |
|---|---|---|---|
| spots / biz 행 수 | 2,897 / 2,897 | 2,897 / 2,897 | — |
| NOT NULL 위반 | 0 | 0 | — |
| 유일성·FK 무결성 | OK | OK | — |
| `postal_code` 5자리 | 2,674 (92.3%) | **2,801 (96.7%)** | +127 |
| `category` null | 2,581 | **4** | -2,577 |
| `category` 다중원소 | 0 | **1~4 elem 분포 정확** | — |
| `pet_policy=SMALL_DOG_ONLY` | 0 | **457** | +457 (보존) |
| `is_pet_allowed=True` | 646 | **1,103** | +457 |
| `total_area_m2 == 0.0` | 1,777 | **0** | -1,777 (→ null) |
| `business_reg_no` invalid | ~60건 | **0** | 정제 완료 |
| `website_url` invalid (http/https/null 외) | ~174건 | **0** | |
| `booking_url` invalid | ~140건 | **0** | |
| `phone` trailing `~` | 4 | **0** | |
| `features` trailing `,` | 2 | **0** | |

unmapped enum: 0건 (모두 정상 매핑).

---

## 운영 원칙

1. **원천 불변**: `data/source1.json`은 절대 직접 수정하지 않음. 모든 정제는 변환 스크립트에서 수행.
2. **결정적 변환**: 동일 입력 → 동일 출력 (uuid5 결정성 보장).
3. **재실행 안전**: `scripts/transform_source1.py`는 언제든 재실행 가능. 기존 산출물 덮어씀.
4. **품질 검증**: 변환 후 `docs/source1_transform_review.md` 형식의 검토 보고서를 갱신.
5. **Changelog 갱신**: 정제 규칙 변경 시 본 문서에 버전 추가.
