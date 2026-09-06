# vivacapi-etl 코드 리뷰 (2026-07-22)

## 요약

Python(`uv`) 기반 스팟 데이터 수집/변환 파이프라인. `src/`(24개 파일, ~1,875 lines) + `scripts/` 구성이며, GoCamping Open API(source1) 하나만 실제 구현되어 있고 source2~7은 TODO stub. 전반적으로 데이터 정제 로직(`transform_source1.py`)과 변경 이력 문서화(`docs/*.md`)는 꼼꼼한 편이지만, prod DB에 직접 쓰는 스크립트들이 **자격증명을 코드에 하드코딩해 git에 커밋**했고 **가드 없는 `TRUNCATE ... CASCADE`**를 prod에 실행하는 구조라 데이터 유실/유출 리스크가 크다. 요청받은 `spots-master-edit-project-42202d2a545b.json` 서비스 계정 키는 실제 private key가 맞지만 **git에는 커밋된 적이 없음**(`.gitignore`가 정상 동작 중)을 확인했다 — 다만 여전히 로컬에 평문으로 남아있는 활성 자격증명이라 별도 권고사항으로 다룬다.

| 심각도 | 건수 |
|---|---|
| Critical | 2 |
| High | 1 |
| Medium | 4 |
| Low / Nit | 4 |

## Critical

### C1. 운영 DB 비밀번호가 평문으로 git에 커밋됨 — `scripts/bulk_insert_source1.py:19-28`
- **문제:** prod RDS 접속 정보(SSH host/key 경로, DB host/user/password/name)가 소스코드에 하드코딩되어 있고, 이 파일은 커밋 `3914727`(`chore(source1): checkpoint WIP source1 tooling and bulk insert scripts`)로 git 이력에 실제로 들어가 있다. `DB_PASSWORD = "[REDACTED — 실제 값은 rotate 후 vivacapi-etl repo 자체 문서/git history 참고]"`(27번 줄)는 `git show HEAD:scripts/bulk_insert_source1.py`로 재확인 — 완전한 평문 상태로 원격 저장소 이력에 남아있다. 동일한 값이 아직 커밋되지 않은 `scripts/upload_source1.py:12-21`(20번 줄 `DB_PASSWORD`)에도 중복되어 있어, 이 파일이 커밋되는 순간 노출이 반복된다.
- **근거:** `git log --oneline -- scripts/bulk_insert_source1.py` → `3914727`; `git show HEAD:scripts/bulk_insert_source1.py` 출력에 동일 값 포함. `docs/naver_favorites_import.md:84-87`도 이 문제를 자체적으로 인지하고 기록해 둠("`bulk_insert_source1.py`/`upload_source1.py`는 prod SSH/DB 자격증명을 스크립트에 하드코딩해 커밋되어 있음").
- **제안:** RDS 비밀번호를 즉시 로테이션하고, 두 스크립트 모두 `upload_naver_favorites_csv.py`가 이미 쓰고 있는 패턴(`python-dotenv` + `os.environ[...]`, `example.env` 템플릿)으로 통일한다. 이미 히스토리에 박힌 비밀번호는 로테이션이 유일한 실질적 해결책이며(git history rewrite는 강제 push가 필요해 별도 논의 필요), 로테이션 전까지는 해당 RDS 인스턴스의 접근 IP를 제한하는 등 임시 완화도 고려.

### C2. `bulk_insert_source1.py`가 prod에 가드 없는 `TRUNCATE ... CASCADE` 실행 — `scripts/bulk_insert_source1.py:19-28, 94-96`
- **문제:** `SSH_HOST`/`DB_HOST`가 상수로 고정되어 있어 이 스크립트는 실행할 때마다 항상 prod를 대상으로 하며, 대상 환경을 바꿀 방법이 없다. 94-96번 줄에서 확인/드라이런 없이 곧바로 `TRUNCATE spot_business_info, spots RESTART IDENTITY CASCADE;`를 실행한다. `CASCADE`는 `spots`/`spot_business_info`를 참조하는 다른 모든 테이블(예: 이미지, 리뷰, 즐겨찾기 등 FK로 연결된 테이블이 있다면 전부)까지 연쇄로 비운다 — source1 재적재 목적의 스크립트가 실수로 한 번만 실행돼도 prod에서 spots와 무관해 보이는 데이터까지 광범위하게 유실될 수 있다.
- **근거:** 94-96번 줄 `TRUNCATE ... CASCADE`. `docs/naver_favorites_import.md:120-121`은 "`upload_naver_favorites_csv.py`는 ... `bulk_insert_source1.py`처럼 테이블을 TRUNCATE하지 않음 — 다른 소스 데이터를 지우지 않는다"라고 명시해, 이 스크립트의 파괴적 특성이 이미 팀 내부적으로 알려진 리스크임을 뒷받침한다.
- **제안:** 최소한 (1) 실행 전 `input("prod를 TRUNCATE합니다. 계속하려면 'yes' 입력: ")` 같은 명시적 확인 단계, (2) `--dry-run` 기본값, (3) 대상 host를 환경변수로 분리해 실수로 prod를 가리키는 상황을 피하는 장치가 필요하다. 근본적으로는 TRUNCATE+재적재 대신 `upload_source1.py`/`upload_naver_favorites_csv.py`처럼 `ON CONFLICT` 기반 upsert로 전환하는 것을 권장 — 최초 1회성 시딩이 목적이었다면 그 사실과 "재실행 금지"를 스크립트 docstring에 명시하고, 재실행이 필요 없어진 시점에 삭제하는 것도 고려.

## High

### H1. `bulk_insert_source1.py`가 uid를 base64url로 인코딩하지 않아 시스템 전체 uid 포맷과 불일치 — `scripts/bulk_insert_source1.py:55-57` (전체 파일에 encode 로직 없음)
- **문제:** `load_rows()`(55-57번 줄)는 `source1_spots.json`/`source1_spot_business_info.json`의 `uid`/`spot_uid` 값을 변환 없이 그대로 튜플에 담아 insert한다. 그런데 `docs/source1_transform.md:237`의 샘플을 보면 `transform_source1.py`가 만드는 `uid`는 표준 UUID 문자열(36자, 예: `"c6b26a33-a88e-5397-80c3-81e05ab99b76"`)이고, 시스템의 실제 uid 포맷은 `base64url` 22자다 — `scripts/upload_source1.py:42-47`(`UUID_COLS`, `encode_uid()`)가 정확히 이 변환을 수행하고, `docs/naver_favorites_import.md:58`도 "`uuid4` → base64url 22자 | `transform_spots_csv.py`의 `new_uid()`와 동일 방식"이라고 명시해 이것이 표준임을 확인해준다. 즉 `bulk_insert_source1.py`는 `upload_source1.py`에 있는 uid 인코딩 스텝이 통째로 빠진 상태로, 이 스크립트를 실행하면 `spots.uid`/`spot_business_info.uid`/`spot_uid`가 36자 UUID 문자열로 prod에 들어가 나머지 파이프라인(naver_favorites, transform_spots_csv 등)이 만드는 22자 포맷과 뒤섞인다.
- **근거:** `scripts/bulk_insert_source1.py`에는 `encode_uid`/`UUID_COLS`가 전혀 없음(grep 결과 0건) 대비 `scripts/upload_source1.py:45-47`, `scripts/parse_naver_favorites.py:44-45`, `scripts/transform_spots_csv.py:38-39`는 모두 동일한 `base64.urlsafe_b64encode(uuid...).rstrip(b"=")` 패턴 사용. `bulk_insert_source1.py`는 git에 커밋된 유일한 버전(커밋 `3914727`)이고, uid 인코딩을 제대로 하는 `upload_source1.py`는 그보다 나중(mtime 기준)에 작성되었으나 아직 커밋되지 않은 상태다.
- **제안:** `bulk_insert_source1.py`를 커밋된 상태로 유지할 계획이라면 `upload_source1.py`와 동일한 `encode_uid`/`UUID_COLS` 처리를 이식한다. 더 근본적으로는 두 스크립트 중 하나로 통합해(M4 참고) uid 인코딩이 한 곳에서만 정의되도록 하는 편이 이런 종류의 불일치 재발을 막는다.

## Medium

### M1. Google 서비스 계정 개인키가 repo root에 평문으로 존재 — `spots-master-edit-project-42202d2a545b.json`
- **문제:** 파일을 열어 확인한 결과 `"type": "service_account"`, `"private_key": "-----BEGIN PRIVATE KEY-----..."`, `client_email` 등 실제 사용 가능한 Google Cloud 서비스 계정 키가 맞다. **다만 git 커밋 이력에는 없다** — `git ls-files`, `git log --all -- spots-master-edit-project-42202d2a545b.json` 모두 결과 없음, `git check-ignore -v`로 `.gitignore:79`(`spots-master-edit-project-*.json`)에 정상적으로 매칭됨을 확인했다. 즉 "커밋된 시크릿"은 아니지만, 로컬 디스크(repo 루트, 644 권한)에 상시 평문으로 놓여있는 활성 자격증명이라는 점은 변하지 않는다 — `git add -f`, zip/백업 전송, 디스크 유출 등으로 여전히 노출될 수 있는 경로가 남아있다.
- **근거:** 파일 내용(1-13번 줄) 육안 확인, `.gitignore:79`, `git check-ignore -v spots-master-edit-project-42202d2a545b.json` exit 0.
- **제안:** Critical로 볼 필요는 없지만(실제 VCS 유출은 없음), repo 트리 바깥(예: `~/.config/vivacapi-etl/` 또는 OS 키체인/시크릿 매니저)으로 옮기고 `.env`의 `GOOGLE_SERVICE_ACCOUNT_FILE`은 절대경로로 그 위치를 가리키도록 바꾸는 것을 권장. 이 키가 다른 경로로 공유된 적이 없는지도 한 번 확인 필요.

### M2. `transform_source1.py`가 한글 텍스트 파일 입출력에 encoding을 명시하지 않음 — `scripts/transform_source1.py:352,454,456`
- **문제:** `SRC.open()`(352번 줄), `OUT_SPOTS.open("w")`/`OUT_BIZ.open("w")`(454, 456번 줄) 모두 `encoding=` 인자 없이 플랫폼 기본 인코딩에 의존한다. 저장하는 JSON은 `ensure_ascii=False`로 한글을 원문 그대로 쓰므로, macOS(현재 개발 환경)에서는 기본 인코딩이 UTF-8이라 문제없이 동작하지만, `LANG`이 `C`/`POSIX`인 Linux/Docker 환경(CI, 배포 서버 등)에서 실행하면 `UnicodeEncodeError`/`UnicodeDecodeError`로 즉시 크래시한다. 같은 저장소의 `src/vivacapi_etl/storage.py:18`(`encoding="utf-8"` 명시), `scripts/upload_naver_favorites_csv.py:53`(`encoding="utf-8-sig"`)는 이미 이 패턴을 지키고 있어 `transform_source1.py`만 예외적으로 빠져있다.
- **근거:** 위 3개 줄에 `encoding=` 부재. 대조군: `storage.py:18-19`.
- **제안:** 세 곳 모두 `encoding="utf-8"` 추가. 한 줄짜리 수정으로 향후 CI/Docker 이전 시 잠재 크래시를 없앨 수 있다.

### M3. `contentId` 누락 시 서로 다른 스팟이 동일 uid로 충돌할 수 있음 — `scripts/transform_source1.py:369`
- **문제:** `content_id = str(r.get("contentId"))`는 `contentId`가 없으면 `None`을 그대로 문자열화해 `"None"`을 uid seed(`f"{src}:{content_id}"`)에 사용한다. `contentId`가 비어있는 행이 2건 이상이면 `uuid5`가 결정적이므로 둘 다 동일한 `spot_uid`/`biz_uid`를 갖게 되어, `bulk_insert_source1.py`의 `TRUNCATE` 후 plain INSERT(ON CONFLICT 없음)에서 unique violation으로 전체 배치가 실패하거나, upsert 경로에서는 조용히 한쪽 데이터가 다른 쪽을 덮어쓸 수 있다. 같은 파일의 다른 모든 필드는 `empty_to_none` 등으로 결측을 꼼꼼히 방어하는 것과 대비되게, 식별자로 쓰이는 이 필드만 무방비 상태다.
- **근거:** 369번 줄. GoCamping API가 실제로 `contentId`를 비우는 경우가 있는지는 이번 리뷰에서 원본 데이터로 확인하지 않았으나(정적 리뷰 범위), 코드상 방어 로직이 전무하다.
- **제안:** `contentId`가 없거나 빈 문자열이면 해당 행을 skip하고 경고 로그를 남기는 명시적 가드 추가.

### M4. prod 적재 스크립트가 두 벌로 나뉘어 있고 서로 다르게 동작 — `scripts/bulk_insert_source1.py` vs `scripts/upload_source1.py`
- **문제:** 두 스크립트 모두 같은 입력 파일(`source1_spots.json`/`source1_spot_business_info.json`)을 같은 prod 테이블에 적재하지만, (a) 하나는 `TRUNCATE` 후 전량 재적재, 하나는 `ON CONFLICT DO NOTHING` upsert(C2/H1 참고), (b) 하나는 uid를 base64url로 인코딩, 하나는 안 함(H1)으로 동작이 갈린다. 코드베이스만 봐서는 어느 쪽이 "현재 정본"인지 알 수 없고, git에 커밋되어 지속적으로 눈에 띄는 쪽(`bulk_insert_source1.py`)이 오히려 uid 버그가 있는 쪽이다. 다음에 이 파이프라인을 재실행할 사람이 잘못된 스크립트를 고를 위험이 실질적으로 존재한다.
- **근거:** 두 파일 비교(위 C1/C2/H1 근거와 동일). 커밋 이력상 `bulk_insert_source1.py`만 git에 있고 `upload_source1.py`는 여전히 untracked.
- **제안:** 하나를 정본으로 정하고 나머지는 삭제하거나 `deprecated_` 접두사 + docstring 경고를 남긴다. 최소한 어느 스크립트가 실제로 prod 시딩에 쓰였는지(또는 쓰일 예정인지) `docs/source1_transform_changelog.md`류 문서에 한 줄이라도 기록해두면 이런 혼선을 막을 수 있다.

## Low / Nit

### L1. 항상 참이 되는 무의미한 assert 절 — `scripts/split_spots_csv.py:52`
- **문제:** `assert len(rows) == 3035 or len(rows) > 0`에서 `len(rows) == 3035` 조건은 `or len(rows) > 0`에 의해 사실상 무력화된다 — rows가 비어있지 않은 한 뒤 조건이 항상 참이므로, 전체 assert는 결과적으로 `len(rows) > 0`만 검증하는 것과 동일하다. "정확히 3035건"을 검증하려던 원래 의도(1회성 데이터 검증 스크립트의 성격상 특정 건수를 기대했을 가능성이 높음)는 실질적으로 동작하지 않는다.
- **근거:** 52번 줄 boolean 로직.
- **제안:** 의도가 "3035건이어야 함"이었다면 `assert len(rows) == 3035, f"expected 3035, got {len(rows)}"`로, "0건만 아니면 됨"이었다면 `== 3035` 절을 제거.

### L2. README.md가 비어있음 — `README.md`
- **문제:** 0 bytes. 설치/실행 방법, 프로젝트 구조에 대한 설명이 전혀 없다. `docs/*.md`는 개별 파이프라인 작업의 변경 이력/검증 보고서라 온보딩 문서 역할은 못 한다.
- **근거:** `wc -l README.md` → 0.
- **제안:** 필수는 아니나, `uv sync` → `uv run python -m vivacapi_etl` 정도의 최소 실행 안내만 있어도 도움이 됨.

### L3. 1회성 스크립트에 작성자 개인 로컬 경로 하드코딩 — `scripts/build_postal_lookup.py:15`, `scripts/transform_spots_csv.py:14`
- **문제:** `Path("/Users/ask4git/Downloads/postal_code_new/zipcode_DB.zip")`, `Path.home() / "Downloads" / "spots_master - spots_joined.csv"` 등 작성자 로컬 파일 시스템에 의존. 1회성 데이터 가공 스크립트라는 성격을 감안하면 큰 문제는 아니고 삭제를 요구할 사안도 아니지만, 재현하려는 다른 사람(또는 미래의 본인)이 바로 실행할 수 없다는 점은 인지해 둘 만하다.
- **근거:** 각 파일 15번째/14번째 줄.
- **제안:** 별도 조치 불필요. CLI 인자화는 필요해질 때 추가.

### L4. `main()`이 미구현 source2~7을 조용히 "성공" 처리 — `src/vivacapi_etl/__init__.py:9-15`, `src/vivacapi_etl/sources/__init__.py:12-20`, `src/vivacapi_etl/sources/source2~7/__init__.py`
- **문제:** `pyproject.toml`의 `project.scripts`(`vivacapi-etl = "vivacapi_etl:main"`)로 노출된 진입점이 `ALL_SOURCES` 7개를 전부 순회하는데, source2~7은 `# TODO: replace with real API call`만 있고 빈 payload(`{"items": []}` 또는 `[]`)를 반환한다. 에러 없이 `[sourceN] saved -> ...` 로그가 찍혀 정상 완료된 것처럼 보이지만 실제로는 6/7이 아무 일도 하지 않는다.
- **근거:** 각 stub 파일의 `fetch()` 구현.
- **제안:** WIP 단계이므로 시급하지 않음. 다만 `vivacapi-etl` 커맨드를 CI/cron 등에 실제로 연결할 계획이 생기면, 그 전에 stub 소스는 `ALL_SOURCES`에서 빼거나 `fetch()`가 `NotImplementedError`를 던지게 해 "조용한 미완성"을 방지할 것.

## 잘된 점

- **멱등성 설계**: `transform_source1.py`가 `uuid5` 결정적 UUID로 재실행해도 동일 입력 → 동일 출력을 보장(`docs/source1_transform_changelog.md`에 원칙으로 명문화까지 되어 있음).
- **트랜잭션 처리**: DB에 쓰는 스크립트들(`bulk_insert_source1.py`, `upload_naver_favorites_csv.py`)이 `autocommit=False` + 예외 시 `rollback()` 후 `raise`로 재던짐 — 예외를 조용히 삼키지 않고, 부분 커밋도 남기지 않는다.
- **Dry-run 기본값**: `map_website_urls.py`, `extract_forest_urls.py`는 `--write` 플래그 없이는 Google Sheets에 아무것도 쓰지 않고 미리보기만 출력 — 실수 방지 패턴이 DB 스크립트들에도 적용됐으면 하는 아쉬움이 남을 정도로 좋은 예.
- **자체 점검 내장**: `parse_naver_favorites.py --selftest`, `transform_spots_csv.py`의 결과물 assert(uid 유일성, 빈 title 등) — 별도 테스트 프레임워크 없이도 최소한의 회귀 방지 장치를 넣어둠.
- **정책적 판단의 문서화**: 네이버 지도 즐겨찾기 수집을 자동 크롤링 대신 사람이 DevTools로 저장한 HTML만 파싱하도록 설계하고, 그 이유(robots.txt/이용약관)를 `docs/naver_favorites_import.md`에 명시 — 법적/정책 리스크를 코드 설계 단계에서부터 고려한 흔적.
- **데이터 품질에 대한 꼼꼼함**: `transform_source1.py`의 광범위한 결측치 정규화(`empty_to_none`), enum 화이트리스트, unmapped 값 추적 로깅, 그리고 `docs/source1_transform_review.md`/`_changelog.md`에 남긴 정량적 검증(P0~P2 이슈별 영향 건수, before/after 비교표) — 데이터 파이프라인 리뷰 문서로서 모범적인 수준.
- **인코딩 인지**: `build_postal_lookup.py`가 zip 파일명의 cp437→cp949 재인코딩까지 처리하는 등, 한국 공공데이터 특유의 인코딩 문제를 대체로 잘 인지하고 다룸(M2의 예외 케이스만 제외하면).
