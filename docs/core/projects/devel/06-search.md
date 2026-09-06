# 탐색 검색 설계 — PostgreSQL FTS + trigram

> `GET /v1/explore/spots?q=...`의 검색 구현. 소스: `crud/spot.py::search_spots`, 원 설계 노트 `docs/core/projects/spot-search-postgres-fts.md`.

## 1. 한 줄 요약

Elasticsearch 없이 PostgreSQL 확장(`pg_trgm` + `tsvector`)만으로 검색을 구현했다. `q` 없으면 기존 목록 동작(전체 목록, `uid` 오름차순 cursor) 그대로 유지, `q` 있으면 검색 모드로 분기한다.

## 2. Elasticsearch를 지금 도입하지 않는 이유

| 검토 항목 | 현재 상태 | 판단 |
|---|---|---|
| 데이터 규모 | spot 수천~수만 단위(큐레이션 플랫폼, 대량 크롤링형 아님) | GIN 인덱스로 충분 |
| 인프라 비용 | ES 클러스터 별도 운영(EC2 t2.micro급 1대로 core도 겨우 도는 인프라) | 추가 서버 운영 부담이 이득보다 큼 |
| 동기화 문제 | Postgres → ES 동기화 파이프라인(CDC/outbox) 신규 구축 필요 | 지금 그 파이프라인이 없음 — 신규 실패 지점 추가 |
| 검색 요구 수준 | 오타 허용 + 부분 매칭 + 필드별 가중치 정도 | `pg_trgm` + `tsvector`로 충족 가능 |
| 한국어 형태소 분석 | 필요 없음(짧은 캠핑장 키워드 위주) | nori 같은 전용 analyzer 아직 불필요 |

### 전환을 고려할 조건 (2개 이상 해당 시 재검토)

1. spot 수가 수십만 이상으로 늘어 GIN 인덱스 스캔 비용이 p95 응답시간에 실측으로 영향
2. 동의어 사전, 복합명사 형태소 분해 등 `simple` config + trigram으로 감당 안 되는 검색 품질 요구
3. 검색과 동시에 대량 facet aggregation이 필요해질 때 (지금은 `spot_field_options` 화이트리스트가 이미 알려진 값 집합이라 불필요)
4. 검색 트래픽 증가로 본 DB(쓰기 트래픽과 동일 인스턴스)와 부하 분리가 필요해질 때

전환 시에는 Postgres를 source of truth로 유지하고 ES/OpenSearch를 검색 전용 read replica 성격으로 붙이며(outbox 패턴 또는 Debezium CDC), 애플리케이션 코드는 검색 쿼리가 `crud/spot.py::search_spots` 함수 하나에 몰려 있어 라우터/스키마 변경 없이 이 함수 내부만 교체 가능한 구조다. 지금 시점에 어댑터 계층을 미리 만들지는 않는다(YAGNI).

## 3. 매칭 대상 필드와 기술

| 필드 | 역할 | 기술 | 가중치 |
|---|---|---|---|
| `title` | 스팟 이름 | `tsvector`(`simple` config) | A(최고) |
| `tagline` | 한줄설명 | `tsvector`(`simple` config) | B |
| `description` | 상세 설명 | `tsvector`(`simple` config) | C |
| `address` | 주소 | `tsvector`(`simple` config) | D(최저) |
| `title` | 오타/부분어 보완 | `pg_trgm` similarity | 별도 보정치 |
| `category` | 카테고리 | array `&&` 정확 매칭 | 필터(랭킹 미개입) |
| `region_province` | 지역 | 등호 매칭 | 필터(랭킹 미개입) |

`category`/`region_province`는 `spot_field_options` 화이트리스트로 관리되는 구조화 값이라 자유텍스트 랭킹에 섞지 않고 `WHERE` 절 필터로만 사용한다.

### `simple` config를 쓰는 이유

Postgres 기본 `english` config는 영어 스테머라 한글에 적용되지 않는다. `simple`은 스테밍 없이 공백/구두점 기준 토큰화만 수행 — 한글에서 스테밍이 없다는 게 오히려 안전(잘못된 어간 추출로 인한 오탐 없음). 대신 짧은 한글 단어의 부분 매칭이 약해지는 단점은 trigram으로 보완한다.

### 오타/부분 매칭 보완 — `pg_trgm`

`tsvector` 매칭은 토큰 단위라 "글램핑장" 검색어가 "글램핑" 토큰을 못 찾는 경우가 있다. `title` 컬럼에 `pg_trgm` GIN 인덱스를 걸어 `similarity(title, q) > 0.2`인 행도 결과에 포함시킨다(OR 조건). `title`에만 적용하는 이유: 부분/오타 검색의 실사용 빈도가 이름에 가장 높고, 전체 필드에 걸면 인덱스 크기와 false positive만 늘어난다.

## 4. 최종 스코어 계산

```sql
score = ts_rank(search_vector, websearch_to_tsquery('simple', :q))
        + similarity(title, :q) * 0.3
```

- `ts_rank`가 주 스코어(필드 가중치 A/B/C/D 반영).
- `similarity * 0.3`은 trigram 매칭 시 순위 보정치. 계수 0.3은 정식 A/B 테스트 없이 정한 시작값 — 실사용 검색 로그가 쌓이면 재조정 대상.
- 정렬: `score DESC, rating_avg DESC, uid DESC` — 텍스트 관련도 1차, 동률이면 평점, 그래도 동률이면 `uid`로 결정론적 정렬.

## 5. 페이지네이션 (검색 모드 전용 cursor)

검색 결과는 `score` 기준 정렬이라 기존처럼 `uid` 하나만으로는 keyset pagination이 불가능(순서가 뒤바뀔 수 있음).

- cursor = `base64({"r": <score>, "v": <rating_avg>, "u": <uid>})`
- 다음 페이지 조건: `(score, rating_avg, uid) < (last_score, last_rating_avg, last_uid)` 튜플 비교 — 정렬 키 3개를 모두 담아야 동점 구간에서 행이 중복/누락되지 않는다.
- `q` 없는 기본 목록의 기존 cursor(평문 `uid`)는 그대로 유지 — 두 모드는 `q` 유무로 완전히 분리돼 있어 cursor를 섞어 보내면(예: 검색 응답의 cursor를 `q` 없는 요청에 재사용) `422 VALIDATION_ERROR`로 거부한다.

## 6. 결정 사항과 근거 요약

| # | 결정 | 근거 |
|---|---|---|
| ES 대신 Postgres 확장 채택 | 2장 참고 |
| 자유텍스트와 구조화 필터 분리 | 같은 파이프라인에 섞으면 `ts_rank`가 구조화 값 매칭까지 반영해 랭킹이 왜곡됨 |
| `to_tsvector` config `simple` 채택 | `english` 스테머가 한글에 무의미 |
| `search_vector`를 `GENERATED ALWAYS AS ... STORED`로 생성 | 별도 트리거/애플리케이션 코드 없이 Postgres가 자동 갱신 |
| `pg_trgm`을 `title`에만 적용 | 실사용 빈도/인덱스 크기 트레이드오프 |
| trigram 보정 계수(0.3), similarity 임계값(0.2)은 잠정값 | 실사용 로그 없는 상태의 초기값, 출시 후 튜닝 대상(YAGNI로 지금은 미세 튜닝 로직 없음) |
| `category`/`region_province` 필터는 `_FILTERABLE` 화이트리스트 안 거침 | FastAPI가 타입 검증하는 고정 파라미터라 임의 컬럼 지정이 애초에 불가능해 같은 위험이 없음 |
| 검색 결과도 `pipeline_status == PUBLISHED`만 노출 | 기존 공개 API 정책과 동일, 검색이라고 예외 없음 |

## 7. Out of Scope (의도적 제외)

| 항목 | 이유 |
|---|---|
| 동의어 사전 | ES 전환 조건 충족 전까지 불필요 |
| `sort` enum(`popular`/`latest`/`rating`) 완전 구현 | 검색 모드와 무관한 별도 이슈 |
| 지리 반경 검색(`earthdistance`/PostGIS) | 요청 범위 밖 |
| trigram 계수 자동 튜닝 | 실사용 로그 쌓이기 전까지 의미 없음 |
