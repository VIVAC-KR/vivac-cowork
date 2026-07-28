# vivacapi-core 코드 리뷰 (2026-07-22)

## 요약

FastAPI + SQLAlchemy asyncio(asyncpg) + PostgreSQL, JWT/Google ID token auth 백엔드. 소스 약 6.6K LOC(`vivacapi/`), 테스트 약 6.7K LOC / 290 test.

전반적으로 규율이 잘 잡힌 코드베이스다. layer 분리(routers/crud/schemas/models/core)가 일관되고, 에러 봉투·화이트리스트 정렬/필터·router 단위 `require_staff` + endpoint 단위 `require_role` 패턴이 CLAUDE.md 규약대로 지켜진다. 모든 DB 접근이 `AsyncSession` + `await` 기반이고, 사용자 입력으로 컬럼을 고르는 곳은 예외 없이 dict 화이트리스트를 거친다(임의 `getattr` 없음). 심각한 문제는 대부분 코드 자체가 아니라 git history에 커밋된 secret 하나이며, 나머지는 동시성 edge case와 audit/일관성 gap 수준이다.

| 심각도 | 건수 |
|---|---|
| Critical | 1 |
| High | 1 |
| Medium | 3 |
| Low / Nit | 4 |

---

## Critical

### C1. 실제 `ADMIN_SESSION_SECRET` 값이 git history에 커밋됨 — `example.env:65` (commit `746ecfc`)

- **문제:** `example.env`(현재는 `.env.example`로 대체됨)의 `ADMIN_SESSION_SECRET`에 placeholder가 아닌 **실제 생성된 secret 값**이 들어간 채 커밋됐다: `ADMIN_SESSION_SECRET=[REDACTED]`. 파일의 다른 모든 secret(`JWT_SECRET_KEY`, `DB_PASSWORD` 등)은 `CHANGE_ME_TO_A_RANDOM_SECRET_KEY` 같은 placeholder인데 이 값만 진짜다. 커밋 `746ecfc0`("feat: set staff auth gateway (#75)", 2026-06-07)에서 추가됐고, 이후 파일이 지워졌지만 **`main`에서 여전히 reachable**하다(`git cat-file -p 746ecfc:example.env`로 확인).
- **근거:** 이 값은 `main.py:68`의 SQLAdmin `SessionMiddleware` 서명 키다. `/admin` 세션 쿠키는 이 키로 서명되므로, 키를 아는 공격자는 임의 user_uid로 admin 세션 쿠키를 위조해 `/admin`에 무인증 진입할 수 있고, SQLAdmin에서 아무 유저의 `is_staff`/`is_active`를 토글할 수 있다(security.md: SQLAdmin은 staff_role 미반영, 모든 staff가 평면 권한). 게다가 `config.py`의 prod 검증(`_validate_prod_requirements`)은 이 값을 **통과시킨다** — `CHANGE_ME`도 없고 32자 이상이라, 누군가 `cp .env.example .env.production` 후 이 값을 그대로 두면 부팅이 성공한다. 즉 실제 배포에 재사용됐을 개연성이 있다.
- **제안:** (1) 이 secret을 즉시 rotate — dev/prod에서 실제 사용 중이면 교체하고 기존 admin 세션을 무효화한다. (2) history purge는 부담이 크니 rotate가 1순위다(reachable 커밋이라 지금도 노출 상태). (3) 재발 방지로 `.env.example`의 모든 secret은 반드시 `CHANGE_ME_*` placeholder만 두는 규칙을 명문화하고, gitleaks 등 pre-commit secret scan 도입을 검토한다.

---

## High

### H1. check-then-insert 경합 시 `IntegrityError`가 잡히지 않아 500 반환 (의도된 409/도메인 에러 아님)

- **위치:** `crud/spot_group.py:117-127`(`add_member`), `:183-198`(`add_spot`), `crud/spot_review_report.py:14-33`(`create_report`), `crud/spot_review.py:48-68`(`create_review`).
- **문제:** 네 함수 모두 "먼저 SELECT로 존재 확인 → 없으면 INSERT" 패턴인데, 대응하는 unique 제약(`SpotGroupMember` 복합 PK, `SpotGroupSpot` 복합 PK, `uq_review_reporter`, 부분 unique index `uq_spot_user_review_active`)이 있다. 같은 리소스로 동시에 두 요청이 들어오면 둘 다 SELECT를 통과한 뒤 한쪽 INSERT가 `IntegrityError`를 던지는데, 이 예외를 잡지 않아 전역 `unhandled_exception_handler`로 흘러 **500 INTERNAL_ERROR**가 나간다. 의도는 각각 `SPOT_GROUP_MEMBER_ALREADY_EXISTS`(409), `SPOT_GROUP_SPOT_ALREADY_EXISTS`(409), `REVIEW_REPORT_ALREADY_EXISTS`(409), `REVIEW_ALREADY_EXISTS`(409)다.
- **근거:** 같은 파일 계층에 **정답 패턴이 이미 존재한다** — `crud/spot_field_option.py:26-33`의 `create_field_option`은 `try: commit except IntegrityError: rollback; raise AppException(...ALREADY_EXISTS)`로 경합을 제대로 409로 변환한다. code-style.md(화이트리스트/일관성)와 api-conventions.md(도메인 에러는 `AppException`+`ErrorCode`로) 취지상, 나머지 네 곳도 이 패턴을 따라야 한다. `create_review`는 사용자 대면 경로(리뷰 중복 제출)라 실제로 재현되기 쉽다.
- **제안:** 각 INSERT `commit`을 `try/except IntegrityError`로 감싸 `rollback()` 후 해당 `AppException(...ALREADY_EXISTS)`를 던진다(선-check는 happy-path 최적화로 남겨도 되고 제거해도 된다 — 제약이 최종 방어선). `create_field_option`을 그대로 참고.

---

## Medium

### M1. `assign_spots`(POST /v1/internal/spots/assignments)에 `set_audit_user` 누락 → audit `changed_by`가 NULL

- **위치:** `api/v1/endpoints/internal_spots.py:135-148`.
- **문제:** 이 endpoint는 `crud_spot.assign_spots`로 spots.`assigned_to_uid`를 UPDATE한다(audit 트리거 대상 테이블). 그런데 형제 할당 endpoint들 — `bulk_assign_spots`(:166), `transfer_spot_assignments`(:188), `reassign_spot`(:219) — 이 모두 UPDATE 직전 `await crud_audit.set_audit_user(db, staff.uid)`를 호출하는 것과 달리, `assign_spots`만 호출하지 않는다. 결과적으로 이 경로의 audit_log 항목은 `changed_by`가 NULL로 남아 "누가 배정했는지"가 기록되지 않는다.
- **근거:** `crud/audit.py:14-21`의 `set_audit_user`는 `SET LOCAL app.user_id`로 트랜잭션 스코프에 actor를 심고, DB 트리거가 그 값을 `audit_log.changed_by`에 넣는다. 호출을 빼면 그 트랜잭션의 쓰기는 actor 미상으로 기록된다. 주요 배정 경로가 형제들과 비일관적이며 incident 조사 시 감사 추적에 구멍이 생긴다.
- **제안:** `assign_spots`에서 `crud_spot.assign_spots(...)` 호출 앞에 `await crud_audit.set_audit_user(db, staff.uid)` 한 줄 추가. (다만 `assign_spots` crud가 자체적으로 `commit`하므로, 같은 세션·트랜잭션 내에서 SET LOCAL이 유지되도록 호출 순서만 지키면 된다 — 형제 endpoint와 동일.)

### M2. SQLAdmin `SessionMiddleware` 쿠키 하드닝 미적용 (backlog 미해결, C1로 위험 가중)

- **위치:** `main.py:64-71`(`Admin(...)` 생성), 하드닝 코드 부재.
- **문제:** SQLAdmin이 장착하는 `SessionMiddleware`가 기본값 그대로다 — `https_only=False`(Secure 플래그 없음), `max_age` 기본 14일. `docs/core/backlog/sqladmin-session-cookie-260711.md`에 이미 등록된 이슈인데 코드에 반영되지 않았다(`grep https_only|same_site|max_age` → config 주석 외 없음). admin이 `http://`로 한 번이라도 접근하면 세션 쿠키가 평문 전송될 수 있고, admin JWT(8h)와 달리 세션은 2주간 유효하다.
- **근거:** C1(세션 서명 키 노출)과 결합하면 위험이 커진다 — 긴 수명 + 노출 키 + Secure 미설정. security.md의 "ADMIN_SESSION_SECRET을 JWT와 분리한 이유(어드민 세션 노출 방지)" 취지와도 어긋난다.
- **제안:** backlog 문서의 방향대로 `AdminAuth`/`Admin`에 `SessionMiddleware`를 `https_only=True, same_site="strict", max_age=8*3600`로 override. 프론트 영향 없음(재로그인 주기만 14일→8h).

### M3. `decay_stale_trust_tiers`가 구현·테스트됐지만 어떤 scheduler에도 연결되지 않음 (dormant)

- **위치:** `crud/spot.py:281-313`. 호출부는 `tests/test_spot_trust_tier_decay.py`뿐(프로덕션 코드에 caller 없음 — grep 확인).
- **문제:** trust_tier 신선도 감쇠 배치 로직이 완성돼 있고 테스트 6건까지 있지만, `main.py` lifespan은 `job_worker_loop`와 `startup_orphan_cleanup`만 띄우고 이 함수를 주기적으로 부르는 경로가 없다. 즉 기능이 실행되지 않는 채로 잠들어 있다.
- **근거:** 코드+테스트만 있고 wiring이 없으면 "구현됐다"는 착각을 유발한다(Simplicity First 관점에선 미완의 기능). 의도적으로 보류한 상태라면 그 사실이 코드/문서에 드러나야 한다.
- **제안:** 주기 실행이 필요하면 lifespan에 경량 스케줄러(예: worker loop 내 일 1회 tick) 또는 외부 cron endpoint로 연결한다. 아직 보류라면 함수 근처에 "스케줄러 미연결(의도적)" 주석 한 줄로 상태를 명시한다.

---

## Low / Nit

### L1. `_STAFF_ROLE_RANK` 중복 정의 — `core/deps.py:18-22` ↔ `api/v1/endpoints/spot_reviews.py:23-27`

`core/deps.py`의 등급 랭크 dict가 `spot_reviews.py`에 그대로 복붙돼 있다(`_is_moderator` 판정용). `core/deps.py`에서 import하면 단일 출처가 된다. 규약 위반은 아니고 DRY nit.

### L2. `http_exception_handler`가 미매핑 status를 `INTERNAL_ERROR` code로 라벨링 — `main.py:124-138`

`_STATUS_TO_CODE`에 401/403/404/422/503만 있어, 405(Method Not Allowed)·409 등 프레임워크 레벨 HTTPException은 status는 유지되지만 봉투 `code`가 `INTERNAL_ERROR`로 나간다. api-conventions.md의 "모든 에러를 표준 봉투로"는 지켜지나 code 라벨이 오해를 부른다. 실제 앱 코드가 이들 status를 직접 던지진 않아 영향은 작다. `_STATUS_TO_CODE`에 405/409 매핑을 추가하면 해소.

### L3. `update_group_member_role`(internal)의 `get_user_by_id` null 미검증 — `api/v1/endpoints/internal_spot_groups.py:205-212`

`user = await crud_user.get_user_by_id(db, member.user_uid)` 뒤 `user.nickname`을 바로 참조한다. FK 무결성상 정상적으론 None이 아니지만, 만약 없으면 `AttributeError` → 500. 공개 `spot_groups.py`의 대응 endpoint는 member만 반환해 이 노출이 없다. 방어적 null-check(또는 404) 권장.

### L4. 보안성 로직 테스트 gap 2건 (testing.md: 화이트리스트/감사 성격 로직은 반드시 커버)

(a) `internal_review_reports`의 sort 화이트리스트 거부 케이스가 테스트되지 않았다 — `internal_spots`(`phone`), `business_info`(`business_reg_no`), `groups`(`not_a_field`)는 거부 테스트가 있는데 review-reports(`created_at`만 허용)만 빠졌다. (b) M1과 연결해, `assign_spots`의 audit `changed_by` 귀속을 검증하는 테스트가 없어 누락이 드러나지 않았다. 두 케이스 각각 짧은 테스트 추가 권장.

---

## 잘된 점

- **화이트리스트 규율이 전면적으로 지켜진다.** 사용자 입력으로 정렬/필터 컬럼을 고르는 모든 곳(`_ADMIN_SORTABLE`/`_FILTERABLE`, spot/group/business_info/review-report)이 dict 화이트리스트를 거치고, 밖의 값은 폴백 없이 422로 거부한다. 임의 속성 `getattr`은 유일하게 `spot_field_option.py:46`에만 있는데 그 `field`는 `SpotOptionField` StrEnum으로 검증된 값이라 안전하다.
- **에러 봉투·권한 계층이 일관적이다.** 도메인 에러는 예외 없이 `AppException`+`ErrorCode`로 던지고(코드 전역에 `raise HTTPException` 0건), router 단위 `require_staff` 위에 등급 제한이 필요한 endpoint에만 `require_role(StaffRole.XXX)`를 얹는 패턴이 security.md 매핑과 정확히 일치한다(bulk=SUPERUSER, 할당/그룹삭제/멤버관리=MANAGER).
- **동시성·트랜잭션 처리가 견고하다.** 스팟 할당·job claim에 `FOR UPDATE SKIP LOCKED`, bulk upsert에 중첩 SAVEPOINT(행별 격리 + 실패 시 전체 롤백), 부팅 시 orphan job 정리까지 실전 시나리오를 챙겼다.
- **테스트가 계층별로 촘촘하다.** router(status/envelope)와 crud(필터/정렬/페이지네이션)를 분리하고, 화이트리스트 거부·권한 등급별 403·last-owner 안전장치·audit 기록 흐름까지 290개 테스트로 커버한다.
