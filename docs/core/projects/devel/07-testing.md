# 테스트 전략

> 소스: `.claude/rules/testing.md`, `tests/conftest.py`, `tests/` 디렉터리 전체.

## 1. 프레임워크

- `pytest` + `pytest-asyncio`(`asyncio_mode = "auto"` — `async def test_*`에 별도 마커 불필요)
- 실행: `uv run pytest` (전체) / `uv run pytest tests/path/to/test_file.py` (단일 파일)
- `make test`는 `.env.test`를 사용해 `vivac_test` DB 대상으로 실행한다.

## 2. 픽스처 (`tests/conftest.py`)

| 픽스처 | 용도 |
|---|---|
| `client` | DB 미사용, 순수 HTTP 계층만 검증할 때 (예: CORS, 헬스체크) |
| `db_client` / `db_session` | DB가 필요한 테스트용. 트랜잭션을 열고 테스트 종료 후 롤백해 테스트 간 상태를 격리 — DB에 실제 커밋되는 side effect를 남기지 않는다 |
| `apply_migrations`(autouse) | 세션 시작 시 `vivac_test` DB에 alembic 마이그레이션을 1회 적용 |

## 3. 파일 구성 원칙

- 파일명은 `test_<대상>.py`로 기능/라우터/crud 단위와 1:1 매칭한다.
- **라우터 테스트와 crud 테스트를 분리**한다 — HTTP 계층 검증(status, 응답 envelope)과 쿼리 로직 검증(필터/정렬/페이지네이션)을 같은 파일에 섞지 않는다.

## 4. 현재 테스트 파일 목록 (`tests/`)

| 파일 | 대상 |
|---|---|
| `test_admin_auth_backend.py`, `test_admin_auth_router.py` | SQLAdmin 인증 백엔드 / `/v1/admin/auth/google` |
| `test_auth_router.py` | `/v1/auth/*` |
| `test_require_staff.py` | `require_staff` 의존성 |
| `test_cors.py` | CORS 설정 (main에도 이미 존재하는 6개 무관 실패가 pre-existing 이슈로 기록돼 있음) |
| `test_errors.py` | `ErrorCode` ↔ HTTP status 매핑 |
| `test_health.py` | 헬스체크 |
| `test_explore_router.py` | 공개 탐색 API |
| `test_spot_search_crud.py` | 검색 crud 로직 |
| `test_spot_trust_tier_decay.py` | trust_tier 감쇠 배치 |
| `test_internal_spots_crud.py`, `test_internal_spots_bulk.py`, `test_internal_spots_delete.py` | 내부 spot 어드민 crud/bulk/삭제 |
| `test_internal_spot_business_info_crud.py`, `test_internal_spot_business_info_bulk.py` | 내부 사업자정보 crud/bulk |
| `test_internal_spot_groups.py` | 내부 그룹 어드민 |
| `test_internal_spot_options.py` | 옵션 화이트리스트 어드민 |
| `test_internal_jobs.py` | job 상태 조회 |
| `test_internal_review_reports_router.py` | 리뷰 신고 어드민 |
| `test_job_worker.py` | 워커 사이클 단위 테스트 |
| `test_spot_bulk_schema.py` | bulk 요청 스키마 검증 |
| `test_spot_groups.py` | 앱용 그룹 crud/로직 |
| `test_spot_images.py` | 이미지 presign/register |
| `test_spot_reviews_crud.py`, `test_spot_reviews_router.py` | 리뷰 crud/라우터 |
| `test_invites_router.py` | 초대/리퍼럴 (17개) |
| `test_user_crud.py` | 유저 crud |
| `test_uid_constraint.py` | shortuuid CHECK 제약 |
| `test_audit_history.py` | 감사 로그 이력 조회 |

## 5. 무엇을 테스트하는가

- **에러 케이스**: `core/errors.py`의 `ErrorCode`가 올바른 status/코드로 매핑되는지 확인 (`test_errors.py`).
- **보안 성격의 로직**: 화이트리스트 기반 필터/정렬처럼 임의 컬럼 주입을 방지하는 로직은 화이트리스트 밖 입력이 거부되는 케이스를 반드시 포함한다.
- **DB 접근**: `AsyncSession` + `await` 기반 crud 함수는 mock으로 대체하지 않고 실제 DB(`db_session`)로 검증한다.
- **권한 등급**: `require_role` 적용 엔드포인트는 하위 등급 403, 해당 등급 이상 통과를 모두 커버한다(예: `test_internal_spot_groups.py`의 STAFF vs MANAGER 케이스).

## 6. 로컬 실행 환경

- 로컬 개발/테스트는 반드시 `.env.local` / `.env.test`를 사용한다 — `.env`가 아니다.
- git worktree를 여러 개 병행할 때, 로컬 Postgres 5432 포트를 공유하면 서로 다른 브랜치의 alembic 마이그레이션 히스토리가 충돌할 수 있다. 이 경우 `DB_PORT`를 워크트리별로 분리(예: 5433)해 독립 컨테이너를 띄우는 방식으로 우회한다 — 실제로 `feature/spot-assignment-reassign`, `feature/reusable-referral-invite` 등 작업에서 이 문제가 반복 발생했다.

## 7. CI (`ci.yml`)

PR마다 다음을 실행한다.

1. `ruff` (lint + format)
2. `alembic upgrade head` (마이그레이션 적용 가능성 검증)
3. `pytest` (전체 테스트)
