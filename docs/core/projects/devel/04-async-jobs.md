# 비동기 Job 워커 & 배치

> 소스: `vivacapi/workers/*.py`, `docs/core/projects/async-job-worker-design.md`(설계 노트 원문), `scripts/decay_trust_tier.py`.

## 1. 설계 요지

bulk upsert처럼 오래 걸리는 작업은 `jobs` 테이블에 등록하고 즉시 `202 Accepted`를 반환한다. 실제 처리는 **API 프로세스 안의 asyncio task**가 수행한다 — 별도 워커 프로세스나 Redis/RabbitMQ 같은 외부 브로커는 쓰지 않는다. 단일 Lightsail 인스턴스라는 전제 위에서 가장 단순한 방식을 택했다.

## 2. 동작 흐름

```
[앱 부팅 = lifespan 진입]
   │
   ├─ 1. orphan 정리: status='running'인 job → 'failed'(error="orphaned")
   │
   └─ 2. asyncio.create_task()로 워커 spawn
          │
   [워커 루프, 2초마다]
          │
          ├─ 3. SELECT ... WHERE status='pending' FOR UPDATE SKIP LOCKED LIMIT 1
          ├─ 4. status='running', started_at=now() 갱신 + 커밋 (락 해제)
          ├─ 5. JobType → 핸들러 매핑 dispatch
          └─ 6. 결과 기록: status='succeeded'|'failed', finished_at, result/error
   │
[앱 종료 = lifespan 종료]
   │
   └─ 7. 워커 task.cancel() + 종료 대기
```

## 3. 왜 이 방식인가 (대안 비교)

| 대안 | 평가 | 채택 여부 |
|---|---|---|
| 외부 브로커(Celery/Dramatiq/arq + Redis) | 재시도/스케줄/우선순위 등 풍부하지만 Redis 인프라 추가 필요 | ❌ 현 트래픽 규모(수동 트리거 기반, 저빈도)엔 과함 |
| FastAPI `BackgroundTasks` | 의존성 0, 코드 한 줄이지만 **영속성 없음**(인스턴스 죽으면 유실), 상태 추적 불가 | ❌ 진행 상태 폴링 API가 필요하고 데이터 유실이 허용 안 됨 |
| PostgreSQL `LISTEN/NOTIFY` 병행 | latency는 최적이지만 LISTEN 전용 커넥션 점유 + 알림 유실 대비 폴링을 결국 병행해야 함 | ❌ 복잡도 대비 이득 작음(2초 폴링이 수동 트리거 시나리오에서 무시 가능) |
| 별도 워커 프로세스 | API 죽어도 워커 생존, 독립 스케일 가능하지만 Lightsail 단일 인스턴스라 격리 이점 약함 | ⏸ 워커가 무거워지면 재검토 |
| SQS/Cloud Tasks | 매니지드지만 vendor lock 비용 대비 효용 낮음 | ❌ |

## 4. 결정 사항

| # | 항목 | 결정 | 이유 |
|---|---|---|---|
| 1 | DB 세션 | 매 사이클 새 세션 | 커넥션 회수 + 트랜잭션 격리 |
| 2 | 핸들러 등록 | 명시적 dict(`HANDLERS = {JobType.X: handler}`) | 단순, 디버깅 쉬움 |
| 3 | 트랜잭션 경계 | claim 트랜잭션과 작업 트랜잭션 분리 | 락을 빠르게 해제, 진행 상태 폴링 가능 |
| 4 | 핸들러 예외 | traceback 전체를 `error` 컬럼에 기록 | 운영 디버깅에 유용 |
| 5 | 폴링 주기 | 2초 | DB 부하 무시 가능 + 수동 트리거 시나리오라 latency 무관 |
| 6 | 동시성 | 1(단일 워커) | 데이터 정합성 우선. `SKIP LOCKED`라 향후 N 워커로 자연 확장 가능 |
| 7 | 자동 재시도 | 없음 | 운영자가 새 job을 생성하는 단순 정책 |

## 5. 핵심 함수 (`vivacapi/workers/job_worker.py`)

```
cleanup_orphaned_jobs(db)                  # running → failed 일괄 전환
claim_next_job(db) -> Job | None            # SKIP LOCKED 기반 1건 claim
process_job(db, job)                        # 핸들러 dispatch + 결과 기록
run_worker_cycle(session_factory)           # 1사이클 = claim + process (단위 테스트에서 직접 호출)
job_worker_loop(session_factory)            # 무한 루프 + 2초 sleep
```

## 6. Bulk Upsert 잡

### 6.1 Job 종류

| JobType | 처리 대상 | 핸들러 |
|---|---|---|
| `spots_bulk_upsert` | `spots` 테이블 대량 upsert | `workers/spots_bulk.py` |
| `spot_business_info_bulk_upsert` | `spot_business_info` 테이블 대량 upsert | `workers/spot_business_info_bulk.py` |

### 6.2 upsert 키

`Spot.external_id` + `source` 조합(`UNIQUE(source, external_id)`)이 upsert 매칭 키다. 페이로드에 `external_id`가 있으면 해당 행에 대해 upsert, 없으면 신규 insert. `spot_business_info`는 `spot_external_id`로 대상 `spots` 행을 조회해 매핑한다.

### 6.3 부분 실패 정책

**전체 트랜잭션 롤백** — 한 행이라도 검증/처리에 실패하면 전체를 롤백하고 `job.status = FAILED`, `job.result`에 행별 실패 사유를 기록한다. 자동 재시도는 없다.

### 6.4 페이로드 한도

| 항목 | 상한 |
|---|---|
| 요청당 행 수 | 5,000행 |
| 요청당 페이로드 크기 | 5 MiB (`core/limits.py::enforce_spots_bulk_size`) |

초과 시 `400 VALIDATION_ERROR`.

## 7. 장애 처리

- 워커가 처리 중 인스턴스가 죽으면 해당 job은 `RUNNING` 상태로 남는다.
- 앱 재부팅 시 lifespan에서 `RUNNING` 상태 job을 `FAILED`(`error="orphaned"`)로 전환 후 워커를 시작한다.
- graceful shutdown 중 long-running task를 cancel하면 `RUNNING` 상태로 남을 수 있는데, 이는 다음 부팅 시 orphan 정리로 보완한다. 이중 처리 위험은 핸들러 멱등성으로 별도 해결해야 하는 영역(아직 명시적 멱등 보장 없음).

## 8. Audit Log와의 연동

`spots`/`spot_business_info`의 쓰기는 DB 트리거로 `audit_log`에 기록되는데, 변경 주체(`changed_by`)는 `set_config('app.user_id', ...)`(SET LOCAL)로 주입해야 남는다. 워커에서는 `process_job`이 이 주입을 담당한다 — job을 생성한 `created_by` uid를 감사 로그의 행위자로 남긴다.

## 9. trust_tier 신선도 감쇠 배치 (온디맨드 job이 아닌 독립 스크립트)

`Spot.last_verified_at`(nullable) 기준 180일 경과(NULL 포함) + `PUBLISHED` + 미삭제 스팟을 대상으로:

- tier 1/2 → 한 단계 하향(숫자 증가)
- 이미 최하위인 tier 3 → `assigned_to_uid`를 비워 재검증 큐로 되돌림(공개 상태는 유지)
- 감쇠 시 `last_verified_at`을 현재 시각으로 갱신(watermark) — 갱신하지 않으면 배치를 돌릴 때마다 tier가 연쇄적으로 무너지는 버그가 생긴다

이 배치는 `jobs` 테이블의 온디맨드 큐 패턴이 아니라 **독립 스크립트**(`scripts/decay_trust_tier.py`) + 호스트 crontab(주 1회 권장) 패턴으로 구현돼 있다 — "DB 백업 이중화" 계획의 전례를 따른 것이다.

> **스코프 밖(후속 필요)**: 스팟이 실제로 재검증될 때 `last_verified_at`을 갱신해주는 쓰기 경로가 아직 없다 — PATCH(`internal_spots.py`)나 bulk upsert에서 `trust_tier`를 설정해도 `last_verified_at`은 그대로 NULL/과거값으로 남는다.

## 10. 향후 검토 (Out of Scope, 아직 손대지 않음)

- bulk upsert가 CPU 바운드로 판명되면 `asyncio.to_thread()` 적용 또는 워커 프로세스 분리
- 작업이 5분 이상 길어지면 graceful shutdown 패턴 추가
- 작업 빈도가 분당 수십 건 이상으로 늘면 LISTEN/NOTIFY 또는 N 워커로 확장
- 자동 재시도(지수 백오프 등)
