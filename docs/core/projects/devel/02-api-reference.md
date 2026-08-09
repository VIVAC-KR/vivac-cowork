# API 레퍼런스

> 소스: `vivacapi/api/v1/routers.py` + `endpoints/*.py` 전수 조사 (2026-08-01 기준). 기존 `architecture.md`의 엔드포인트 표보다 최신/상세하다 — 이 문서를 1차 참고로 삼는다.
> 항상 최신 확정 계약이 필요하면 `make openapi`(→ `docs/openapi.json`, git 미추적) 또는 서버 실행 후 `/docs`(Swagger)를 사용한다.

## 1. 공통 규약

- 모든 에러 응답은 전역 예외 핸들러를 거쳐 `{"error": {"code", "message", "details"}}` 형식으로 통일된다. Enum 값은 [01-data-model.md](./01-data-model.md)의 `ErrorCode` 표 참고.
- 도메인 에러는 `HTTPException`을 직접 던지지 않고 `AppException(ErrorCode, message)`를 사용한다.
- `/v1/internal/*` 라우터는 **라우터 단위**로 `Depends(require_staff)`가 걸려 있다 — 개별 엔드포인트가 아니라 `include_router(..., dependencies=[Depends(require_staff)])`.
- 일부 내부 엔드포인트는 라우터 단위 `require_staff`(coarse gate) 위에 엔드포인트별로 `Depends(require_role(StaffRole.MANAGER|SUPERUSER))`를 추가로 얹는다 — 아래 표의 "권한" 컬럼 참고.
- `vivac-console`은 `/v1/internal/...`만 호출한다. 유일한 예외는 로그인(`/v1/admin/auth/google`).

## 2. 공개 API (비로그인)

| Method | Path | 설명 |
|---|---|---|
| `GET` | `/health` | 헬스체크 |
| `POST` | `/v1/auth/google` | Google ID Token 로그인 → JWT 쌍 발급. `invite_uid` 포함 시 신규가입 자동 초대 소비 |
| `POST` | `/v1/auth/refresh` | 토큰 갱신 |
| `GET` | `/v1/explore/spots` | 공개(`PUBLISHED`) spot 목록/검색 (`q`, `category`, `region_province`, 커서 페이지네이션) |
| `GET` | `/v1/explore/spots/{uid}` | 공개 spot 상세 |
| `GET` | `/v1/explore/spots/{uid}/images` | spot 이미지 목록 (CDN/presigned URL) |
| `GET` | `/v1/groups/{group_uid}` | `PUBLIC`/`INVITE_ONLY` 그룹 상세 (비로그인도 조회 가능, `get_current_user_optional`) |
| `GET` | `/v1/groups/{group_uid}/spots` | 위 그룹의 스팟 목록 |
| `GET` | `/v1/groups/{group_uid}/members` | 위 그룹의 멤버 목록 |
| `GET` | `/v1/invites/{uid}` | 초대 링크 미리보기 (수락 전 클릭 시 열람) |
| `GET` | `/v1/explore/spots/{spot_uid}/reviews` | 스팟 리뷰 목록 |

## 3. 인증 필요 (앱 유저, `Bearer JWT`)

| Method | Path | 설명 |
|---|---|---|
| `GET` | `/v1/auth/me` | 현재 사용자 정보 |
| `POST` | `/v1/groups` | 그룹 생성 |
| `GET` | `/v1/groups` | 내 그룹 목록 |
| `PATCH` | `/v1/groups/{group_uid}` | 그룹 정보 수정 |
| `DELETE` | `/v1/groups/{group_uid}` | 그룹 삭제 |
| `POST` | `/v1/groups/{group_uid}/spots` | 그룹에 스팟 추가 |
| `DELETE` | `/v1/groups/{group_uid}/spots/{spot_uid}` | 그룹에서 스팟 제거 |
| `POST` | `/v1/groups/{group_uid}/members` | 멤버 초대 (`OWNER`만, `PRIVATE` 그룹은 불가) |
| `PATCH` | `/v1/groups/{group_uid}/members/{user_uid}` | 멤버 역할 변경 |
| `DELETE` | `/v1/groups/{group_uid}/members/{user_uid}` | 멤버 추방 |
| `POST` | `/v1/invites` | 초대 링크 발급 (그룹 초대 or 일반 리퍼럴) |
| `POST` | `/v1/invites/{uid}/accept` | 기존 로그인 유저의 그룹 초대 수락 |
| `POST` | `/v1/explore/spots/{spot_uid}/reviews` | 리뷰 작성 |
| `PATCH` | `/v1/explore/spots/{spot_uid}/reviews/{review_uid}` | 리뷰 수정 (작성자 본인) |
| `DELETE` | `/v1/explore/spots/{spot_uid}/reviews/{review_uid}` | 리뷰 삭제 (작성자 본인 또는 모더레이터) |
| `POST` | `/v1/explore/spots/{spot_uid}/reviews/{review_uid}/reports` | 리뷰 신고 |
| `POST` | `/v1/admin/auth/google` | 콘솔 staff Google 로그인 (인증 전이라 별도 prefix) |

그룹 관련 라우트의 권한 규칙(`require_group_role`, `_get_readable_group` 등) 상세는 [05-domain-features.md](./05-domain-features.md) 참고.

## 4. 내부 어드민 API (`/v1/internal/*`, 라우터 단위 `require_staff`)

`vivac-console`(Refine simple-rest)이 호출한다. 목록형 엔드포인트는 `_start`/`_end`/`_sort`/`_order` 쿼리 + 응답 헤더 `X-Total-Count` 규약을 따르고, 정렬/필터 컬럼은 화이트리스트로 제한된다(밖의 값은 422). 예외적으로 `internal_spots.py` 목록은 검색/필터/정렬을 지원하는 offset 페이지네이션이다.

### 4.1 Spot (`/v1/internal/spots`)

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| `GET` | `` | STAFF | 목록 (검색·필터·정렬·offset 페이지네이션) |
| `GET` | `/stats` | STAFF | 대시보드 통계 (My Queue 포함) |
| `GET` | `/distinct/{field}` | STAFF | 필터 드롭다운 옵션 |
| `POST` | `/assignments` | **MANAGER 이상** | 검증 대기 spot 할당 (`assigned_to_uid IS NULL`인 spot만) |
| `PATCH` | `/assignments` | **MANAGER 이상** | spot uid 목록 일괄 (재)할당 — 기존 할당 여부 무관 |
| `POST` | `/assignments/transfer` | **MANAGER 이상** | 담당자 A→B로 검증 대기 물량 이전 (`count`개) |
| `GET` | `/{uid}` | STAFF | 상세 |
| `PATCH` | `/{uid}` | STAFF | 부분 수정 (`pipeline_status` 전이 화이트리스트 적용) |
| `PATCH` | `/{uid}/assignment` | **MANAGER 이상** | 단건 재할당/해제 (`user_uid: null`이면 해제) |
| `DELETE` | `/{uid}` | **MANAGER 이상** | 소프트 삭제 |
| `POST` | `/{uid}/restore` | **MANAGER 이상** | 소프트 삭제 복구 |
| `GET` | `/{uid}/history` | STAFF | 감사 로그 기반 수정 이력 |
| `GET` | `/{uid}/groups` | STAFF | 이 스팟이 속한 그룹 목록 |
| `POST` | `/bulk` | **SUPERUSER** | 대량 upsert 잡 등록 (202, 최대 5000행 / 5 MiB) |

> 라우팅 순서 주의: `PATCH /assignments`(고정 경로)는 파일 내에서 `PATCH /{uid}`(동적 경로)보다 먼저 선언돼야 한다 — `{uid}`가 `"assignments"` 문자열을 그대로 삼켜버릴 수 있다. `distinct/{field}` vs `/{uid}`도 동일 규칙.

### 4.2 Spot 이미지 (`/v1/internal/spots`)

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| `POST` | `/{uid}/images/presign` | STAFF | 업로드용 presigned PUT URL 발급 |
| `POST` | `/{uid}/images` | STAFF | 업로드 완료 이미지 등록 |

### 4.3 Spot 사업자 정보 (`/v1/internal/spot-business-info`)

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| `GET` | `` | STAFF | 목록 |
| `GET` | `/{uid}` | STAFF | 상세 |
| `PATCH` | `/{uid}` | STAFF | 수정 |
| `GET` | `/{uid}/history` | STAFF | 수정 이력 |
| `POST` | `/bulk` | STAFF | 대량 upsert 잡 등록 (202) |

### 4.4 Spot 옵션 화이트리스트 (`/v1/internal/spot-options`)

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| `GET` | `` | STAFF | 옵션값 목록 조회 (`category`/`amenities`/`nearby_facilities`/`has_equipment_rental`) |
| `POST` | `` | **MANAGER 이상** | 옵션값 추가 |
| `DELETE` | `/{field}/{code}` | **MANAGER 이상** | 옵션값 삭제 |

### 4.5 Spot Group 어드민 (`/v1/internal/groups`)

멤버십 체크 없이 아무 유저의 그룹이나 조회/모더레이션 가능. 앱 API(`/v1/groups/*`)와 달리 `PRIVATE` 그룹에도 강제 개입할 수 있다.

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| `POST` | `` | STAFF | 그룹 생성 |
| `GET` | `` | STAFF | 목록 (`name_like`/`visibility`/`user_uid` 필터) |
| `GET` | `/{group_uid}` | STAFF | 상세 |
| `PATCH` | `/{group_uid}` | STAFF | 메타 수정 (모더레이션) |
| `DELETE` | `/{group_uid}` | **MANAGER 이상** | 삭제 (하드 삭제, cascade) |
| `GET` | `/{group_uid}/members` | STAFF | 멤버 목록 (nickname/email join) |
| `POST` | `/{group_uid}/members` | **MANAGER 이상** | 멤버 강제 추가 (`PRIVATE` 그룹도 허용 — 앱 API와의 의도적 차이) |
| `PATCH` | `/{group_uid}/members/{user_uid}` | **MANAGER 이상** | 역할 강제 변경 |
| `DELETE` | `/{group_uid}/members/{user_uid}` | **MANAGER 이상** | 멤버 강제 제거 |
| `GET` | `/{group_uid}/spots` | STAFF | 그룹 내 스팟 목록 |
| `POST` | `/{group_uid}/spots` | STAFF | 스팟 추가 |
| `DELETE` | `/{group_uid}/spots/{spot_uid}` | STAFF | 스팟 강제 제거 (단일 항목, 가역적) |

last-owner 안전장치는 어드민도 동일 적용 — 그룹을 owner 0명 상태로 만들 수 없다(강제로 없애려면 그룹 자체를 삭제).

### 4.6 리뷰 신고 (`/v1/internal/review-reports`)

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| `GET` | `` | STAFF | 리뷰 신고 목록 조회 |

### 4.7 Job (`/v1/internal/jobs`)

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| `GET` | `/{job_id}` | STAFF | bulk 잡 상태 조회 (`status`, `result`, `error`) |

## 5. 운영 UI

| Path | 설명 |
|---|---|
| `/admin` | SQLAdmin — 사용자 계정 상태/권한 관리 (staff 세션 필요, 비상용) |
| `/docs` | Swagger UI (OpenAPI) |

## 6. vivac-console 목록 API 규약 (Refine simple-rest)

`internal_spots.py`, `internal_spot_business_info.py` 등 목록 조회 엔드포인트는 Refine의 simple-rest data provider 규약을 따른다.

| 항목 | 규약 |
|---|---|
| Query params | `_start`, `_end`, `_sort`, `_order` |
| 응답 헤더 | `X-Total-Count`에 전체 개수 반환 (CORS `expose_headers` 등록 필요) |
| 정렬/필터 컬럼 | 화이트리스트(`_ADMIN_SORTABLE`, `_FILTERABLE` 패턴)로 관리, 임의 컬럼 주입 방지 |

## 7. 공개 탐색 API 페이지네이션 (cursor)

`GET /v1/explore/spots`는 offset이 아닌 **opaque cursor 방식**을 쓴다.

- `next_cursor`는 `base64(JSON)`으로 인코딩된 불투명 토큰 — 클라이언트는 echo만 한다.
- 기본 목록(검색어 없음)은 `uid` 단일 cursor, 검색 모드(`q` 있음)는 `{score, rating_avg, uid}` composite cursor — 서로 다른 포맷이라 섞어 보내면 `422 VALIDATION_ERROR`.
- 정렬 기준을 바꾸고 이전 cursor를 재사용하면 422로 거부된다.
- 상세 설계 근거는 [06-search.md](./06-search.md) 참고.

## 8. 인증 흐름 요약

3가지 인증 흐름이 있고 모두 Google ID Token 검증에서 시작한다. 상세는 [03-auth-and-security.md](./03-auth-and-security.md).

| 흐름 | 엔드포인트 | 토큰 | 만료 |
|---|---|---|---|
| 앱 사용자 | `POST /v1/auth/google` | HS256 JWT access + refresh | 30분 / 7일 |
| vivac-console (staff) | `POST /v1/admin/auth/google` | HS256 JWT access(+`email`,`is_staff` 클레임) | 8시간 |
| SQLAdmin (`/admin`) | 로그인 폼 → `AdminAuth` | 서명된 세션 쿠키 | 세션(14일, [09](./09-known-issues-and-tech-debt.md) 참고) |
