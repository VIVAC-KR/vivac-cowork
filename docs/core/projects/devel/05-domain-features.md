# 도메인 기능 상세 — Group / Invite / Review / Image

## 1. Spot Group (컬렉션)

> 소스: `models/spot_group.py`, `crud/spot_group.py`, `endpoints/spot_groups.py`(앱), `endpoints/internal_spot_groups.py`(어드민).

유저가 스팟을 모아 관리하는 컬렉션. `spot_groups` ↔ `users`는 `spot_group_members`(역할 포함) N:M, `spot_groups` ↔ `spots`는 `spot_group_spots` N:M.

### 1.1 공개 범위(`GroupVisibility`)

| 값 | 읽기 권한 | 멤버 초대 |
|---|---|---|
| `private` | owner만 | 불가 |
| `invite_only` | 멤버만 | 가능 |
| `public` | 비로그인 포함 누구나 | 가능 |

### 1.2 그룹 내 역할(`GroupRole`)

`VIEWER < CONTRIBUTOR < EDITOR < OWNER` 순으로 권한 누적. 한 그룹에 OWNER가 여러 명일 수 있다(공동 소유).

### 1.3 접근 제어 패턴 (`endpoints/spot_groups.py`)

- `_get_readable_group`: `PUBLIC`이면 누구나, 아니면 멤버만 통과. **존재를 숨기려고 403이 아닌 404를 반환**한다(비멤버에게 private 그룹의 존재 여부 자체를 노출하지 않기 위함) — `get_current_user_optional`을 써서 비로그인도 `PUBLIC` 그룹은 조회 가능하게 한다.
- `_get_membership_or_404`: 멤버가 아니면 역시 404.
- `require_group_role(min_role)`: 멤버십 확인 위에 역할 등급까지 검사하는 의존성 팩토리 — `core/deps.py`의 `require_role` 패턴과 동일한 구조.
- 멤버 초대(`POST /v1/groups/{uid}/members`)는 `OWNER`만 가능하고, `PRIVATE` 그룹은 초대 자체가 막힌다.
- last-owner 안전장치: 그룹을 owner 0명 상태로 만드는 역할 변경/제거는 `409 SPOT_GROUP_LAST_OWNER_REQUIRED`.

### 1.4 어드민 API와의 차이

`vivac-console`용 `/v1/internal/groups/*`는 멤버십 체크를 건너뛰고 `require_staff`(+일부 `MANAGER` 이상)로만 게이트한다. CRUD 로직은 `crud/spot_group.py`를 그대로 재사용한다.

| 항목 | 앱 API (`/v1/groups`) | 어드민 API (`/v1/internal/groups`) |
|---|---|---|
| 대상 | "내가 멤버인 그룹"만 | 아무 유저의 그룹이나 |
| 페이지네이션 | 단순 offset/limit | Refine simple-rest(`_start`/`_end`/`_sort`/`_order` + `X-Total-Count`) |
| `PRIVATE` 그룹 초대 | 403 (`SPOT_GROUP_INVITE_NOT_ALLOWED`) | 어드민은 우회 가능(지원/모더레이션 목적) |
| 존재하지 않는 리소스 응답 | 404(존재 은닉) | 동일하게 404 |
| 파괴적 작업(삭제, 역할 강제 변경) | 소유자 본인만 | `MANAGER` 이상 |

전체 엔드포인트 표는 [02-api-reference.md](./02-api-reference.md) 4.5 참고.

## 2. Invite / 리퍼럴 링크

> 소스: `models/invite.py`, `crud/invite.py`, `endpoints/invites.py`, `endpoints/auth.py`.

공유 링크 기반 초대. **그룹 초대**와 **일반 앱 리퍼럴**을 하나의 `Invite` 엔티티로 처리한다 — `group_uid`가 있으면 그룹 초대, 없으면 리퍼럴.

### 2.1 스키마 개요

```
Invite
  uid              PK, 공유 링크 토큰 겸용(별도 token 컬럼 없음)
  inviter_uid      FK users.uid
  group_uid        FK spot_groups.uid, nullable, ON DELETE CASCADE
  group_role       GroupRole, nullable (group_uid 있을 때만)
  status           pending / accepted / revoked
  accepted_by_uid  FK users.uid, nullable
  accepted_at      nullable

User.referred_by_uid  FK users.uid, nullable — 신규 가입 시 1회만 세팅, 이후 불변
```

`group_uid`가 `ON DELETE CASCADE`라 그룹이 삭제되면 관련 invite도 함께 삭제된다(수락 시점에 그룹이 사라져 있는 경합 상태를 원천 차단).

### 2.2 재사용 정책 — 그룹 초대와 리퍼럴이 다르다

| 구분 | 재사용 여부 | 배경 |
|---|---|---|
| 그룹 초대 (`group_uid` 있음) | **1회용**. 수락되면 `status=ACCEPTED`로 종료 | 최초 설계 그대로 |
| 일반 리퍼럴 (`group_uid` 없음) | **재사용 가능**. 수락 후에도 `status`가 `PENDING`으로 유지돼 같은 링크로 여러 명이 반복 가입 가능 | 2026-07-20 `feature/reusable-referral-invite`에서 정책 역전. 단톡방/SNS에 링크 하나로 여러 명을 유입시키는 표준 리퍼럴 패턴을 지원하기 위함. `consume_invite_for_signup`에서 `group_uid is None`일 때만 `ACCEPTED` 전환을 생략하도록 수정 — 스키마/마이그레이션 변경 없이 소진 정책 하나만 바뀐 것 |

> 어뷰징 방지(재사용 횟수 제한 등)는 의도적으로 미구현 상태다. 대량 가짜 가입 리스크는 [../business/05-monetization-and-roadmap.md](../business/05-monetization-and-roadmap.md) 1.1 참고.

### 2.3 엔드포인트별 동작

| 엔드포인트 | 동작 |
|---|---|
| `POST /v1/invites` | 발급. `group_uid` 있으면: 그룹 존재 확인(404) → `PRIVATE`면 403 → 요청자가 `OWNER` 아니면 403(멤버 아니면 404로 존재 은닉) |
| `GET /v1/invites/{uid}` | 미리보기, 인증 불필요(공유 링크 클릭 시 로그인 전 열람). `inviter_nickname`/`group_name`/`status` 반환 |
| `POST /v1/invites/{uid}/accept` | 기존 로그인 유저의 그룹 합류. `status != PENDING` 또는 `group_uid is None`이면 409(`INVITE_NOT_ACCEPTABLE`) — 리퍼럴 초대를 이 엔드포인트로 열면 합류할 그룹이 없어 거부됨 |
| `POST /v1/auth/google` (`invite_uid` 포함) | 신규가입 자동 수락. `consume_invite_for_signup`이 invite가 없거나 `PENDING`이 아니면 **조용히 무시**(best-effort) — 초대 링크가 깨져 있다고 로그인 자체가 막히면 안 되기 때문 |

### 2.4 기존 `SpotGroupMember.invited_by_uid`와의 관계

건드리지 않았다. `invited_by_uid`는 `users.uid` FK라 비회원을 가리킬 수 없어 "이미 가입한 유저를 그룹에 초대"(`POST /v1/groups/{uid}/members`) 용도로만 유지되고, "비회원을 초대해 신규 가입시키는" 시나리오는 `Invite` + `User.referred_by_uid`가 별도로 처리한다.

## 3. Spot Review & 신고

> 소스: `models/spot_review.py`, `models/spot_review_report.py`, `endpoints/spot_reviews.py`, `endpoints/internal_review_reports.py`.

- 유저당 스팟별 리뷰 1개(`UNIQUE(spot_uid, user_id)`), `rating`은 0~5 범위 CHECK.
- 작성/수정/삭제는 본인만. 삭제는 작성자 본인 또는 모더레이터(`_is_moderator`)도 가능.
- 리뷰 신고(`POST .../reviews/{review_uid}/reports`)는 별도 `spot_review_reports` 테이블에 쌓이고, `vivac-console`이 `GET /v1/internal/review-reports`로 모아 본다.
- **스팟 자체**(리뷰가 아니라)에 대한 폐쇄/접근불가 신고 기능은 아직 없다 — 비즈니스 관점 공백은 [../business/05-monetization-and-roadmap.md](../business/05-monetization-and-roadmap.md) 4.1 참고.

## 4. 이미지 스토리지

> 소스: `core/storage.py`, `models/spot_image.py`, `endpoints/internal_spot_images.py`, `endpoints/explore.py`.

업로드는 3단계로 나뉜다 — API 서버는 파일 바이트를 직접 받지 않고 presigned URL 발급·검증만 담당한다.

```
1. Presign   POST /v1/internal/spots/{uid}/images/presign
             서버가 키를 spots/{uid}/{shortuuid}{ext} 형태로 생성
             (content_type으로 확장자 결정, jpeg/png/webp만 허용)
             → S3 presigned PUT URL 발급. 키를 서버가 생성하므로
               클라이언트가 임의 경로에 쓰지 못한다.
                     │
2. 직접 업로드  클라이언트(콘솔/앱)가 발급받은 URL로 S3에 직접 PUT
             (API 서버·EC2 우회)
                     │
3. Register  POST /v1/internal/spots/{uid}/images
             s3_key가 spots/{uid}/ 하위 경로인지 검증(다른 spot 경로
             등록 방지) + object_exists(S3 head_object)로 실제 업로드
             됐는지 재확인한 뒤에만 spot_image row 생성
```

조회 시 `is_public=True`는 CDN URL, `False`는 presigned GET URL을 반환한다. **`is_public`은 서빙 방식 구분이지 접근 제어가 아니다** — 두 경우 모두 (의도상) 공개 API에 노출되도록 설계됐다. 다만 현재 공개 엔드포인트 구현에 `is_public` 필터가 빠져 있는 상태이므로 [09-known-issues-and-tech-debt.md](./09-known-issues-and-tech-debt.md)의 알려진 이슈를 함께 참고한다.

S3(`S3_BUCKET`/`CDN_BASE_URL`) 미설정 시 이미지 API만 503(`SERVICE_UNAVAILABLE`)을 반환하고 나머지 기능은 정상 동작한다.
