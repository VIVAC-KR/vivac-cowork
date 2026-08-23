# VAC-15 — orphan S3 객체 정리, DB-S3 트랜잭션 보장 범위 결정

- 날짜: 2026-08-23
- 관련 이슈: [VAC-15](https://linear.app/lucente/issue/VAC-15/register-안-된-orphan-s3-객체-정리-안-됨)
- 관련 PR: [vivacapi-core#174](https://github.com/VIVAC-KR/vivacapi-core/pull/174)

## 배경

`POST /{uid}/images/presign`이 발급하는 S3 presigned PUT URL은 실제 업로드
권한이지만, 클라이언트가 업로드만 하고 `POST /{uid}/images`(register)를
호출하지 않으면 DB에는 기록이 없는(SpotImage row 없음) orphan S3 객체가
정리 없이 영구히 남는 문제가 있었습니다.

## 결정 1 — presign을 pending prefix로 발급 (Option A 채택)

presign 발급 키를 최종 경로(`spots/{uid}/...`)가 아니라
`uploads/pending/{uid}/...` prefix로 바꾸고, register 성공 시에만 서버가
`copy_object`로 최종 경로에 복사한 뒤 pending 원본을 `delete_object`합니다.
`uploads/pending/` prefix에는 별도로 `vivac-infra`(terraform)에 S3
lifecycle rule을 걸어 N일 후 미등록 객체를 자동 삭제할 계획입니다(이
PR 범위 밖, 별도 확인/요청 필요).

검토했던 대안:

- **Option B (reconciliation 배치)**: 주기적으로 `ListObjectsV2`로 버킷
  전체를 스캔해 DB와 diff. 앱 코드 변경은 적지만 버킷이 커지면 스캔
  비용이 늘고, 이 저장소에는 주기(cron) 실행 인프라 자체가 없어 별도로
  마련해야 함.
- **Option C (presign 발급 기록 + TTL 배치)**: `pending_upload` 테이블을
  추가해 정밀하게 추적. 가장 정확하지만 스키마 추가와 별도 배치가 필요해
  orphan 규모가 커지기 전까지는 과설계로 판단.

Option A를 택한 이유는 register 시점 프로비저닝이 아직 안 된 상태
(`S3_BUCKET`/`CDN_BASE_URL` 미설정, `docs/core/reviews/known-issues.md`)라
지금이 prefix 분리를 적용하기 가장 저렴한 시점이었고, 별도 배치 없이
AWS가 정리를 담당해 유지보수 부담이 가장 적기 때문입니다.

## 결정 2 — register 중 실패 시 보정(compensate) 처리

`code-review` 스킬로 diff를 2회 검토하며 아래 두 문제를 발견해 반영했습니다.

1. **copy 후 DB insert 실패 시 final_key가 추적 안 되는 orphan으로 남는
   문제.** `copy_object(pending, final)` 이후 `create_image`(DB insert)가
   실패하면 final_key 객체에 DB row가 없고, `uploads/pending/` prefix
   밖이라 lifecycle rule도 적용되지 않아 원래 문제(orphan)가 형태만
   바뀌어 재발합니다. → DB insert 실패 시 방금 copy한 final_key를
   `delete_object`로 되돌리고, pending 원본은 그대로 둬 register 재시도가
   가능하게 했습니다.
2. **보정 delete 자체가 실패하면 원래 예외(진짜 원인)가 가려지는 문제.**
   → 보정 delete를 별도 try/except로 감싸 실패 시 로그만 남기고, 원래
   예외는 그대로 재발생시키도록 수정했습니다.

## 결정 3 — DB row와 S3 객체의 완전한 트랜잭션 보장은 시도하지 않음

리뷰 후 "DB row와 S3 객체가 완전히 트랜잭션으로 보장되면 좋겠다"는
요구가 있었으나, Postgres와 S3는 서로 다른 시스템이라 진짜 2PC(분산
트랜잭션)는 불가능합니다. 강화 방안과 기각 사유는 다음과 같습니다.

- **insert-first saga** (presign 직후 `s3_key=pending_key`로 먼저 commit
  → copy → `s3_key`를 final_key로 UPDATE(commit) → pending 원본 delete):
  DB row가 항상 "실재하는 객체"를 가리키게 되어 더 강한 보장을 주지만,
  중간에 서버가 죽으면 row가 pending_key를 가리킨 채로 남을 수 있어
  lifecycle rule이 그 객체를 지우기 전에 복구하는 주기 점검
  (reconciliation)이 필요해집니다. 이는 Option A를 택한 핵심 이유(별도
  배치 불필요)와 정면으로 배치되어 기각.
- **outbox / 2단계 커밋 패턴**: 가장 견고하지만 별도 outbox 테이블과
  워커가 필요해 이 티켓의 스코프를 크게 넘어섬. 기각.

현재 구현에서 남은 유일한 이론적 리스크는 `session.commit()` 성공
직후 `session.refresh()`가 같은 커넥션에서 실패하는 극히 희박한
commit-ack race입니다. 이는 이 PR만의 결함이 아니라 `crud/spot_image.py`의
`update_image`/`delete_image` 등 이 저장소의 모든 DB write에 동일하게
존재하는 구조적 리스크이므로, 이 PR에서 별도로 대응하지 않기로
결정했습니다.

## 결과

"DB row 없는 S3 orphan"(원래 문제)과 "보정 실패로 인한 원인 예외
마스킹"(리뷰에서 발견) 두 가지는 모두 해소했고, 시스템 간 완전한
원자성은 구조적으로 불가능하다는 점과 그 대가로 남는 잔여 리스크의
크기(사실상 무시 가능한 수준)를 명시적으로 기록해 둡니다.
