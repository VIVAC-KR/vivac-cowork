# Enum 값 정리

`vivacapi-core` 백엔드에서 쓰는 모든 `StrEnum`의 실제 값(DB/API에 저장되는 값)과 의미 정리.
소스: `vivacapi/models/*.py`, `vivacapi/core/errors.py`.

## JobStatus (`vivacapi/models/job.py`)

Bulk upsert 등 비동기 작업(Job)의 진행 상태.

| 값 | 의미 |
|---|---|
| `pending` | 대기 중, 아직 시작 안 함 |
| `running` | 실행 중 |
| `succeeded` | 성공 완료 |
| `failed` | 실패 |

## JobType (`vivacapi/models/job.py`)

Job이 수행하는 작업 종류.

| 값 | 의미 |
|---|---|
| `spots_bulk_upsert` | spot 대량 upsert |
| `spot_business_info_bulk_upsert` | spot 사업자 정보 대량 upsert |

## InviteStatus (`vivacapi/models/invite.py`)

공유 링크 기반 초대(그룹 초대 또는 앱 리퍼럴)의 상태.

| 값 | 의미 |
|---|---|
| `pending` | 대기 중, 미수락 |
| `accepted` | 수락됨 |
| `revoked` | 철회됨 |

## MembershipTier (`vivacapi/models/user.py`)

사용자 멤버십 등급.

| 값 | 의미 |
|---|---|
| `free` | 무료 회원 |
| `member` | 유료(정회원) |

## StaffRole (`vivacapi/models/user.py`)

`is_staff=True`인 사용자 내부의 세부 권한 등급. `STAFF < MANAGER < SUPERUSER` 순으로 권한 누적. `is_staff=False`면 의미 없음.

| 값 | 의미 |
|---|---|
| `staff` | 기본 콘솔 접근 권한 |
| `manager` | staff 이상 — 검증 작업 할당, 그룹 삭제/멤버 역할 강제 변경 등 |
| `superuser` | 최상위 — spot bulk upsert(최대 5000행) 등 파괴적 작업 |

## SpotImageRole (`vivacapi/models/spot_image.py`)

Spot 이미지의 용도 구분.

| 값 | 의미 |
|---|---|
| `thumbnail` | 대표 이미지 |
| `detail` | 상세 이미지 |

## GroupVisibility (`vivacapi/models/spot_group.py`)

Spot 그룹 공개 범위.

| 값 | 의미 |
|---|---|
| `private` | owner만 접근, 멤버 초대 불가 |
| `invite_only` | 멤버만 조회 가능(읽기 권한은 private와 동일), 멤버 초대는 허용 |
| `public` | 비로그인 포함 누구나 조회 가능 |

## GroupRole (`vivacapi/models/spot_group.py`)

Spot 그룹 내 멤버 역할. `VIEWER < CONTRIBUTOR < EDITOR < OWNER` 순으로 권한 누적. 한 그룹에 OWNER가 여러 명일 수 있음(공동 소유).

| 값 | 의미 |
|---|---|
| `viewer` | 조회만 가능 |
| `contributor` | 콘텐츠 기여 가능 |
| `editor` | 편집 권한 |
| `owner` | 소유자, 그룹 관리 권한 |

## SpotOptionField (`vivacapi/models/spot_field_option.py`)

관리형 화이트리스트를 쓰는 `spots`의 배열 컬럼 이름. 값이 실제 컬럼명과 동일(코드에서 `getattr(Spot, field)`로 바로 매칭).

| 값 | 의미 |
|---|---|
| `category` | spot 카테고리 |
| `amenities` | 편의시설 |
| `nearby_facilities` | 주변 시설 |
| `has_equipment_rental` | 장비 대여 여부 관련 옵션 |

## PipelineStatus (`vivacapi/models/spot.py`)

Spot 데이터 파이프라인 진행 상태. `PUBLISHED`만 공개 API에 노출됨.

| 값 | 의미 |
|---|---|
| `RAW` | 원본 수집 데이터 |
| `ENRICHED` | 데이터 보강 완료 |
| `CURATED` | 큐레이션 완료 |
| `REVIEWED` | 검수 완료 |
| `PUBLISHED` | 공개됨 (공개 API 노출 대상) |
| `REJECTED` | 반려됨 |

## ErrorCode (`vivacapi/core/errors.py`)

전역 에러 응답 봉투(`error.code`)에 쓰이는 코드. 값 자체가 사람이 읽을 수 있는 스네이크/파스칼 형태라 별도 human-readable 매핑 없이 값 = 의미. 각 코드가 매핑되는 HTTP status만 정리.

| 값 | HTTP status |
|---|---|
| `UNAUTHORIZED` | 401 |
| `FORBIDDEN` | 403 |
| `NOT_FOUND` | 404 |
| `USER_NOT_FOUND` | 404 |
| `JOB_NOT_FOUND` | 404 |
| `SPOT_NOT_FOUND` | 404 |
| `SPOT_BUSINESS_INFO_NOT_FOUND` | 404 |
| `SPOT_OPTION_NOT_FOUND` | 404 |
| `SPOT_OPTION_ALREADY_EXISTS` | 409 |
| `SPOT_GROUP_NOT_FOUND` | 404 |
| `SPOT_GROUP_MEMBER_NOT_FOUND` | 404 |
| `SPOT_GROUP_MEMBER_ALREADY_EXISTS` | 409 |
| `SPOT_GROUP_SPOT_ALREADY_EXISTS` | 409 |
| `SPOT_GROUP_LAST_OWNER_REQUIRED` | 409 |
| `SPOT_GROUP_INVITE_NOT_ALLOWED` | 403 |
| `REVIEW_NOT_FOUND` | 404 |
| `REVIEW_ALREADY_EXISTS` | 409 |
| `REVIEW_REPORT_ALREADY_EXISTS` | 409 |
| `INVITE_NOT_FOUND` | 404 |
| `INVITE_NOT_ACCEPTABLE` | 409 |
| `VALIDATION_ERROR` | 422 |
| `INTERNAL_ERROR` | 500 |
| `SERVICE_UNAVAILABLE` | 503 |
