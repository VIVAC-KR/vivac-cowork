# 탐색 API 지도 확장 — 구현 회신

| | |
|---|---|
| **작성일** | 2026-08-05 |
| **대상 요청** | "VIVAC 탐색 API 확장 요청 — 지도 기반 탐색 도입" (2026-08-05) |
| **브랜치** | `feature/search-map-api` |
| **상태** | 구현 완료 — 계약 확정 |

---

## 0. 요약 — 계약 확정 여부

| 항목 | 결정 | 요청 대비 |
|---|---|---|
| §2 목록 좌표 필드 | 확정 | **nullable로 내려감** (아래 §2 참조) |
| §3 좌표 없는 스팟 제외 | 확정 | env 플래그, 기본 OFF |
| §4 bbox | 확정 | 요청대로 |
| §5 total | 확정 | 캡 10,000 |
| §6 지도용 경량 응답 | 확정 | **별도 엔드포인트** `GET /v1/explore/spots/map`, limit 상한 8,000 |
| §7 커서 스코프 검증 | 확정 | `400 CURSOR_SCOPE_MISMATCH` |

FE는 이 문서의 응답 형태로 목을 만들면 된다. 아래 §9에 우선 확인 요청 3건 회신.

---

## 1. 확정 계약

### `GET /v1/explore/spots`

```
q, category, region_province, bbox, cursor, limit(1-50)
```

```json
{
  "items": [{
    "uid": "spot_a1b2c3",
    "title": "남이섬 오토캠핑장",
    "trust_tier": 2,
    "thumbnail_url": "https://cdn.vivac.app/...",
    "region_short": "강원",
    "category": ["AUTO_CAMPING"],
    "latitude": 37.7907,
    "longitude": 127.5262
  }],
  "next_cursor": "a1b2c3d4e5f6:spot_a1b2c3",
  "has_more": true,
  "total": 137,
  "total_capped": false
}
```

### `GET /v1/explore/spots/map` (신규)

```
q, category, region_province, bbox, limit(1-8000, 기본 8000)
```

```json
{
  "items": [{"uid": "spot_a1b2c3", "latitude": 37.7907, "longitude": 127.5262, "trust_tier": 2}],
  "truncated": false
}
```

### `GET /v1/explore/spots/{uid}`

변경 없음. `EXPLORE_REQUIRE_COORDINATES=true`일 때만 좌표 없는 spot이 `404 SPOT_NOT_FOUND`.

### 신규 에러 코드

| 코드 | status | 발생 조건 |
|---|---|---|
| `CURSOR_SCOPE_MISMATCH` | 400 | 커서 발급 시점의 bbox와 요청의 bbox가 다름 |
| `VALIDATION_ERROR` | 422 | bbox 형식/범위 오류 |

---

## 2. 좌표 필드 (§2)

`SpotListItem`에 `latitude` / `longitude` 추가. **요청과 달리 nullable로 내린다.**

**근거.** 요청의 non-nullable 논리는 "§3이 좌표 없는 스팟을 제외하므로 응답에 도달한
항목은 좌표를 갖는다"에 의존한다. 그런데 §3의 롤아웃 조건(좌표 적재 완료 전에는 켤 수
없음) 때문에 그 규칙은 플래그 뒤에 있고, 기본값은 OFF다. 플래그가 꺼진 상태에서
스키마를 non-nullable로 선언하면 목록 조회가 **매 요청 500**이 된다 — 현재 prod
spot은 좌표가 전부 NULL이기 때문이다.

OpenAPI 스키마는 정적이라 두 설정 중 한쪽에만 참일 수 없다. 배포된 실제 동작에 대해
거짓말하지 않는 쪽을 택했다.

**FE의 null 분기를 없애는 대안은 §6이다.** `SpotMapItem.latitude/longitude`는
non-nullable이고, 지도 엔드포인트는 플래그와 무관하게 항상 좌표 보유 spot만 반환한다.
null 분기가 실제로 아픈 곳은 핀 렌더링이므로, 보장이 필요한 지점에 보장을 뒀다.

좌표 적재가 끝나고 `EXPLORE_REQUIRE_COORDINATES=true`로 전환한 뒤, 목록 필드도
non-nullable로 좁히는 것은 별도 PR로 가능하다 (FE 호환 깨지지 않는 방향의 변경).

---

## 3. 좌표 없는 스팟 제외 (§3)

`EXPLORE_REQUIRE_COORDINATES` (bool, 기본 `false`) 환경변수.

켜면 `crud/spot.py`의 `_apply_explore_filters()`가 `latitude IS NOT NULL AND
longitude IS NOT NULL`을 건다. 이 함수 하나를 목록/검색/카운트/지도/상세가 모두
경유하므로, **엔드포인트별로 조건을 각각 걸어 어긋나는 경우가 구조적으로 없다** —
요청의 "어느 모드에서 보든 같은 집합" 요구가 코드 한 곳에서 보장된다.

**구현 위치를 `pipeline_status`가 아니라 조회 필터로 정한 이유.** 발행 조건에 넣으면
좌표가 뒤늦게 채워진 spot을 다시 `PUBLISHED`로 승격시키는 배치가 필요하고, 좌표가
지워지면 발행 상태가 뒤집힌다. 파이프라인 상태는 "사람이 검수를 끝냈는가"를 뜻하고
좌표 보유는 "지도에 그릴 수 있는가"를 뜻해, 서로 다른 축이다. 플래그로 껐다 켰다
해야 한다는 롤아웃 요구도 조회 시점 필터가 아니면 만족시킬 수 없다.

**상세도 404다.** 목록에서만 빼면 북마크·공유된 URL은 계속 열려, 검색으로는 찾을 수
없는 페이지가 링크로만 접근 가능해진다. `get_spot_by_uid(published_only=True)`에서
같은 필터를 태워, 상세와 이미지 목록이 함께 404가 된다.

어드민 경로(`published_only=False`)는 영향받지 않는다 — 좌표를 채우려면 콘솔에서
그 spot이 보여야 한다.

### 롤아웃 절차

1. 좌표 적재 (진행 상황은 §9 참조)
2. dev에서 `EXPLORE_REQUIRE_COORDINATES=true` → 탐색 결과 개수 확인
3. prod env 반영 후 재배포

---

## 4. bbox (§4)

`bbox=min_lng,min_lat,max_lng,max_lat` — 경도 먼저 (GeoJSON/OGC).
`vivacapi/core/geo.py`의 `parse_bbox()`가 파싱/검증한다.

**검증을 넣은 이유.** 요청이 지적한 대로 순서 실수는 에러 없이 빈 결과만 준다. 다만
한국 좌표에서는 조용히 넘어가지 않는다 — 경도(124~132)를 위도 자리에 넣으면 위도
유효 범위(-90~90)를 벗어나므로 `422`로 걸린다. 그 외에 `min >= max`인 뒤집힌 상자도
거부하고, 메시지에 `order is lng,lat — not lat,lng`를 넣었다.

날짜변경선을 넘는 bbox(`min_lng > max_lng`)는 지원하지 않는다. 국내 전용 서비스라
필요해질 때 추가한다.

`region_province + bbox`는 요청대로 **방어하지 않는다** — 두 값이 같이 오면 그냥
AND로 결합된다 (FE가 bbox 전송 시 region을 제거하는 계약).

### 인덱스

```sql
CREATE INDEX ix_spots_coordinates ON spots (latitude, longitude)
WHERE latitude IS NOT NULL AND longitude IS NOT NULL AND deleted_at IS NULL;
```

PostGIS·GiST는 도입하지 않았다. 8,000건 규모에서 복합 btree + partial index로 충분하고,
PostGIS는 확장 설치와 RDS 파라미터 그룹 변경까지 딸려온다. 실측상 이 규모에서는
플래너가 seq scan을 고르는 경우도 많다 — 인덱스는 데이터가 커질 때를 위한 보험이지
지금의 병목 해소가 아니다.

서버 사이드 클러스터링도 넣지 않았다 (요청의 판단과 동일).

### 사이드 이펙트 — 기존 버그 수정

기존 `list_spots()`(= `q` 없는 목록 모드)는 `category`·`region_province`를 **받고도
무시**하고 있었다. `search_spots()`에만 필터가 걸려 있었다. 요청의 "`category` + `bbox`
= AND"는 `q` 없이도 성립해야 하므로, 공통 필터 함수로 합치면서 이 누락도 함께 고쳤다.
`q` 없이 `category`나 `region_province`만 보내던 기존 호출은 **이제 실제로 필터링된다** —
동작 변경이니 FE에서 이 조합을 쓰고 있었다면 확인 바란다.

---

## 5. total (§5)

캡 **10,000** (`crud/spot.py`의 `TOTAL_CAP`).

요청의 예시값 1,000 대신 10,000을 택했다. 전체 규모가 8,000건 미만이면 캡 1,000은
흔하게 걸려서 사실상 "1000+"만 보여주게 되는데, 정작 사용자가 궁금한 "이 지도 영역에
몇 곳"은 대개 세 자리 수다. 10,000이면 현재 데이터에서 `total`은 사실상 항상 정확하고,
`total_capped`는 계약에 남아 있어 데이터가 커져도 FE 변경 없이 캡이 작동한다.

비용은 `LIMIT 10001` 서브쿼리 위의 `COUNT(*)`이므로, 최악의 경우에도 10,001행에서
멈춘다. 필터(q/category/region/bbox)는 목록 조회와 **동일한 조건**을 태운다.

---

## 6. 지도용 경량 응답 (§6)

`fields=map` 파라미터 대신 **별도 엔드포인트** `GET /v1/explore/spots/map`.

**근거.** `fields`는 응답 스키마를 유니온으로 만들어 OpenAPI가 "둘 중 하나"밖에 말하지
못한다. FE 타입 생성이 흐려지고, `limit` 상한이 50과 8,000으로 갈리는 것도 파라미터
값에 따라 검증 규칙이 달라지는 조건부 로직이 된다. 커서·`total`이 지도 모드에서는
무의미하다는 점도 응답 형태가 갈리는 이유다.

**요청이 우려한 "검색·필터 로직 한 벌 유지"는 유지된다** — 두 엔드포인트 모두
`_apply_explore_filters()` + `_search_terms()`를 공유한다. 갈라진 것은 HTTP 계약뿐이고
쿼리 조건은 한 곳이다.

**limit 상한 8,000은 열었다.** 요청의 계산(경량 페이로드 → gzip 후 100KB 안팎)에
동의한다. 항목당 필드가 4개(uid 22바이트 + float 2 + smallint)라 JSON raw로도
8,000건 ≈ 700KB, gzip 후 100KB 수준이다. 기본값도 8,000으로 뒀다 — 지도는 bbox 안
전부를 원하는 것이 정상 요구라, 기본값이 잘라내면 조용히 잘못된 그림을 준다.

`truncated`가 상한에 걸렸는지 알려준다. `false`면 `len(items)`가 곧 해당 조건의 전체
개수이므로 지도 모드에는 `total`을 따로 두지 않았다.

라우트는 `/spots/{uid}`보다 **먼저** 등록해야 한다 (안 그러면 `map`이 uid로 매칭된다).
회귀 방지 테스트 있음: `test_map_route_is_not_shadowed_by_detail_route`.

---

## 7. 커서 스코프 검증 (§7)

커서에 bbox 지문(sha256 앞 12자)을 박고, 요청 bbox와 다르면 `400 CURSOR_SCOPE_MISMATCH`.

- 목록 커서: `{scope}:{uid}` — bbox 없으면 scope는 빈 문자열이라 `:{uid}`
- 검색 커서: 기존 base64 JSON에 `"s"` 키 추가

bbox로 발급한 커서를 **bbox 없이** 재사용하는 경우도 스코프 불일치로 거부한다 (그
커서가 가리키는 위치는 다른 결과 집합의 것이므로).

기존 형식(콜론 없는 맨 uid) 커서는 scope `""`로 해석돼 그대로 동작한다 — 배포 시점에
FE가 들고 있던 커서가 깨지지 않는다.

**정렬(`sort`) 추가 시**: 요청의 예상대로 같은 규칙이 필요하다. `_scope_token()`에
정렬 키를 함께 넣으면 되도록 만들어 뒀다.

---

## 8. 캐시

지도 응답도 기존 `spots:list:v{version}` 네임스페이스를 쓴다 (`kind=map` 포함해 키 해시).
따라서 spot 수정 시 `bump_spots_version()` 한 번으로 목록·검색·지도가 **함께** 무효화된다.
TTL은 `SPOTS_LIST_CACHE_TTL_SECONDS`(30초) 공유.

캐시 키에 `bbox`가 포함되므로 지도를 움직일 때마다 새 키가 생긴다. bbox는 사실상
무한한 값이라 히트율이 낮지만, TTL 30초로 자연 소멸하므로 메모리 압박은 제한적이다.
"이 지역 재검색" 버튼 방식(자동 재조회 없음)이라 키 생성 속도도 사용자 클릭 속도로 제한된다.

---

## 9. 우선 확인 요청 3건 회신

### ① §2 — 기존 좌표 필드가 WGS84인가

**코드로는 확인 불가. 데이터 소스 확인이 필요하다.**

`spots.latitude/longitude`는 변환 없는 순수 `Float` 컬럼이고, 저장 경로
(`POST /v1/internal/spots/bulk`, `scripts/upload_spots_local.py`, 콘솔 편집) 어디에도
좌표계 변환이나 검증이 없다. **입력값이 그대로 들어간다.** 즉 좌표계는 전적으로
적재 소스가 무엇을 주느냐에 달려 있고, 저장소는 그걸 알지 못한다.

요청의 지적대로 이건 "약간 틀림"이 아니라 "완전히 다른 곳"으로 나타나는 실패 모드다.
적재 시점에 검증하는 것을 제안한다 — 한국 범위(위도 33~39, 경도 124~132)를 벗어난
값을 bulk upsert에서 거부하면, 카텍/TM 같은 미터 단위 좌표계는 값의 크기 자체가
달라(수십만 단위) 첫 행에서 걸린다. 이번 범위에 포함하지 않았고, 별도로 진행할지
알려주면 붙이겠다.

### ② §3 — 좌표 적재 진행 상황과 예상 완료 시점

**백엔드에서 답할 수 있는 범위가 아니다** — 적재 계획은 데이터 파이프라인/운영 쪽 일정이다.

대신 **현재 수치는 콘솔에서 실시간으로 볼 수 있다**: `GET /v1/internal/spots/stats`의
`missing_coordinates` / `total`. 이 두 값으로 진척률을 직접 추적할 수 있으니, 일정
산정에 쓰기 바란다. 코드 쪽 준비는 끝났고 env 한 줄로 전환된다.

### ③ §6 — 지도용 limit 상한을 8,000까지 열 수 있는지

**열었다.** 기본값도 8,000. 위 §6 참조.

---

## 10. 이번 범위에서 제외한 것

- 필터(합법·식수·취사)·정렬·자동완성 — 요청에서 제외된 범위
- 서버 사이드 클러스터링, 줌 레벨별 집계 — 8,000건 규모에서 불필요
- PostGIS / GiST
- 좌표계 검증 (§9-① 참조 — 별도 판단 필요)
- 목록 좌표 필드의 non-nullable 전환 (§2 — 플래그 전환 이후)

---

## 11. 변경 파일

| 파일 | 변경 |
|---|---|
| `vivacapi/core/geo.py` | 신규 — `BBox`, `parse_bbox()` |
| `vivacapi/core/config.py` | `EXPLORE_REQUIRE_COORDINATES` |
| `vivacapi/core/errors.py` | `CURSOR_SCOPE_MISMATCH` (400) |
| `vivacapi/crud/spot.py` | 공통 필터, bbox, 커서 스코프, `count_spots`, `list_spots_map` |
| `vivacapi/schemas/spot.py` | 목록 좌표, `total`/`total_capped`, `SpotMapItem`/`SpotMapResponse` |
| `vivacapi/api/v1/endpoints/explore.py` | `bbox` 파라미터, `/spots/map` |
| `vivacapi/models/spot.py` | `ix_spots_coordinates` |
| `alembic/versions/a7c3e1f9b204_*.py` | 인덱스 마이그레이션 |
| `tests/test_explore_map_router.py` | 신규 — bbox/커서/total/지도 |
| `tests/test_explore_coordinate_gate.py` | 신규 — §3 플래그 ON/OFF 양쪽 |
