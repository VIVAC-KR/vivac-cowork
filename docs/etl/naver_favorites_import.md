# 네이버 지도 즐겨찾기 → spots 임포트 작업 기록

> 작성일: 2026-07-15
> 스크립트: `scripts/parse_naver_favorites.py`, `scripts/upload_naver_favorites_csv.py`
> 입력: 사람이 DevTools로 저장한 즐겨찾기 폴더 HTML (파일)
> 출력: `data/naver_favorites_upload.csv` → prod `spots` 테이블

---

## 1. 작업 배경

VIVAC 콘솔에 노출할 스팟을 네이버 지도 즐겨찾기 공유 폴더
(`map.naver.com/p/favorite/sharedPlace/folder/...`)에서 가져오기 위한 파이프라인.
소스는 나중에 카카오맵/구글맵 즐겨찾기로 바뀔 수 있음 — `spots.source` 컬럼에
자유 텍스트로 구분값만 넣으면 되므로 소스 교체 자체는 스키마 변경 없이 가능.

**자동 크롤링을 하지 않는 이유**: `map.naver.com/robots.txt`가 `/p/` 하위 경로를
전부 Disallow 하고 있고, 네이버 이용약관(`policy.naver.com/rules/service.html`)에
"네이버의 사전 허락 없이 자동화된 수단(매크로/봇/스파이더/스크래퍼)으로 ... 게시물
등을 수집"하는 행위를 명시적으로 금지하는 조항이 있음. 이 조항은 주기적 polling뿐
아니라 1회성 자동 요청에도 적용됨. 그래서 이 파이프라인은 **코드에서 네이버로 나가는
네트워크 요청이 0건** — 사람이 자기 브라우저로 직접 페이지를 열람하고 DevTools에서
렌더링된 HTML을 복사해 로컬 파일로 저장한 것만 파싱한다.

## 2. 파이프라인 구성

```
사람: 네이버 즐겨찾기 폴더 열람 → DevTools Elements 패널에서
      목록 영역 outerHTML 복사 → 로컬 파일로 저장
  │
  ▼
scripts/parse_naver_favorites.py <저장한 파일> --source naver_favorite
  │  (표준 라이브러리 html.parser로 로컬 파일만 파싱, 네트워크 요청 없음)
  ▼
data/naver_favorites_upload.csv
  │
  ▼
사람: CSV 육안 검토
  │
  ▼
scripts/upload_naver_favorites_csv.py
  │  (SSH 터널 + psycopg2, ON CONFLICT (source, external_id) DO NOTHING)
  ▼
prod DB spots 테이블 (pipeline_status=RAW로 삽입)
  │
  ▼
vivac-console `/spots` 화면 (소스=naver_favorite, 상태=RAW 필터)
에서 사람이 검수 → ENRICHED → ... → PUBLISHED로 승격 (기존 화면, 변경 없음)
```

## 3. 필드 매핑

파서가 HTML에서 뽑아내는 값은 제한적이고, 나머지는 콘솔 검수 단계에서 사람이 채우는
것을 전제로 한다.

| DB 컬럼 | 값 | 비고 |
|---|---|---|
| `uid` | `uuid4` → base64url 22자 | `transform_spots_csv.py`의 `new_uid()`와 동일 방식 |
| `source` | CLI `--source` 값 (기본 `naver_favorite`) | 소스 바뀌면 이 인자만 바꾸면 됨 |
| `external_id` | 즐겨찾기 항목 링크의 `/place/{숫자}` | `(source, external_id)` unique 제약으로 재실행 시 중복 방지 |
| `title` | 링크 블록 안 첫 텍스트 | ⚠️ 4장 참고 |
| `address` | 링크 블록 안 나머지 텍스트 이어붙임 | ⚠️ 4장 참고 |
| `rating_avg` | `0` (고정) | NOT NULL, 서버 기본값 없음 — 명시적으로 채움 |
| `review_count` | `0` (고정) | 〃 |
| `pipeline_status` | `RAW` (고정) | 컬럼을 INSERT에 명시하면 서버 `server_default`가 안 먹으므로 직접 씀 |
| 그 외 전부 | 빈 값 → `NULL` | 좌표·전화번호·카테고리 등. 콘솔 검수 단계에서 채움 |

## 4. 알려진 한계

- **DOM 구조 추정치임**: 네이버 즐겨찾기 공유 폴더가 SPA라 사전에 실제 마크업을 볼
  방법이 없었음. `FavoriteParser`(`parse_naver_favorites.py`)는 "place 링크가 나올
  때마다 새 항목 시작, 다음 place 링크 전까지의 텍스트를 그 항목 것으로 간주, 첫 줄=제목
  나머지=주소"라는 휴리스틱으로 동작. 실제 저장 파일로 처음 돌려보고 제목/주소가
  이상하게 나오면 셀렉터·휴리스틱 조정 필요.
- 좌표(`latitude`/`longitude`), 전화번호, 카테고리 등은 즐겨찾기 목록 화면만으로는
  확인이 안 돼 비워둠 — 필요하면 콘솔 편집 화면에서 채우거나, 주소 문자열을
  `data/postal_lookup.sqlite`(도로명주소 → 행정구역 lookup, `build_postal_lookup.py`가
  만든 것)로 보강하는 걸 추후 고려 가능 (이번 스코프에서는 안 함).
- `spot_business_info`(사업자 등록 정보)는 채우지 않음 — 즐겨찾기 데이터에는 해당
  정보가 없음.

## 5. 자격증명 관리

`bulk_insert_source1.py`/`upload_source1.py`는 prod SSH/DB 자격증명을 스크립트에
하드코딩해 커밋되어 있음. `upload_naver_favorites_csv.py`는 이 패턴을 따르지 않고
환경변수로 읽음(`python-dotenv`, 이미 의존성에 있었음). 로컬 `.env`에 아래 키를
채워야 실행됨(값은 `upload_source1.py` 상단 상수와 동일):

```
SSH_HOST=
SSH_PORT=22
SSH_USER=
SSH_KEY_PATH=

DB_HOST=
DB_PORT=5432
DB_USER=
DB_PASSWORD=
DB_NAME=
```

템플릿은 `example.env`에 있음.

## 6. 실행 방법

```bash
# 1) 파서 자체 점검
uv run python scripts/parse_naver_favorites.py --selftest

# 2) 저장한 HTML 파싱
uv run python scripts/parse_naver_favorites.py <저장한파일.html> --source naver_favorite
# → data/naver_favorites_upload.csv 생성. 열어서 육안 확인.

# 3) DB 업로드 (.env 채운 뒤)
uv run --with sshtunnel --with psycopg2-binary --with python-dotenv \
    python scripts/upload_naver_favorites_csv.py
```

`upload_naver_favorites_csv.py`는 `ON CONFLICT (source, external_id) DO NOTHING`이라
재실행해도 안전(중복 삽입 안 됨). `bulk_insert_source1.py`처럼 테이블을 TRUNCATE하지
않음 — 다른 소스 데이터를 지우지 않는다.

## 7. 검증 결과 (이번 세션)

- `--selftest`: 픽스처 HTML 2건에서 `external_id`/`title`/`address` 정상 추출 확인.
- 샘플 HTML(2건)로 실제 CSV 생성 확인 — 컬럼 순서, `rating_avg=0`, `review_count=0`,
  `pipeline_status=RAW`, uid 22자 형식 전부 정상.
- 실 데이터(네이버 즐겨찾기 실제 저장 파일)로는 아직 실행 안 함 — 4장의 DOM 휴리스틱
  검증이 남아 있음.

## 8. 향후 개선 여지 (해결 안 됨)

- 실제 페이지 구조 확인 후 `FavoriteParser` 파싱 로직 보정 필요할 가능성 높음.
- 주소 → `region_province`/`region_city`/`postal_code` 자동 보강 (`postal_lookup.sqlite`
  재사용 가능하지만 아직 연결 안 함).
- 소스가 카카오맵/구글맵으로 바뀔 경우: `parse_naver_favorites.py`의 링크 패턴
  (`/place/{숫자}` 정규식)과 텍스트 추출 로직만 소스별로 다시 짜면 되고, 업로드
  스크립트·DB 스키마·콘솔 검수 화면은 그대로 재사용 가능.
