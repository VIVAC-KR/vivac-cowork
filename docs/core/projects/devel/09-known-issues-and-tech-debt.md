# 알려진 이슈 & 기술 부채

> 여러 출처(2026-07-11 보안/성능 점검, 2026-07-20 문서 감사, `docs/core/backlog.md`)를 취합했다. 착수 시 원본 backlog 파일에서 지우고 이슈/PR로 전환하는 것이 저장소 관례다.

## 1. 보안 (상세는 [03-auth-and-security.md](./03-auth-and-security.md) 6장)

| 심각도 | 이슈 | 위치 |
|---|---|---|
| 🟠 중간 | 비공개 이미지(`is_public=False`)가 공개 탐색 API로 노출 | `crud/spot_image.py::list_images_by_spot` |
| 🟠 중간 | SQLAdmin 세션 쿠키 하드닝 부재 (`https_only=False`, 14일 만료) | `main.py`, `admin/auth.py` |
| 🟡 낮음 | prod `ALLOWED_EMAIL_DOMAIN` 미설정을 부팅 검증이 막지 않음 | `core/config.py` |
| 🟡 낮음 | auth 엔드포인트 rate limit 부재 | `endpoints/auth.py`, `admin_auth.py` |
| 🟡 낮음 | `deploy.yml` git tag 이름 셸 injection 패턴 | `.github/workflows/deploy.yml:88,121-123` |

## 2. 성능 (2026-07-11 sql-pro 점검, 데이터 누적 시 발현 예정)

| 이슈 | 위치 | 트리거 시점 |
|---|---|---|
| 어드민 목록 count 서브쿼리 + 선행 와일드카드 `ILIKE '%검색어%'`가 인덱스를 못 탐 (seq scan 2회: count + 본쿼리) | `crud/spot.py:86-95`, `crud/spot_business_info.py:40-49` | spots 수만 건 이상 + 어드민 목록 체감 지연 |
| `spots.pipeline_status`는 `PUBLISHED` partial index뿐 — 어드민의 `RAW`/`CURATED` 필터는 seq scan | `models/spot.py`, `crud/spot.py`(`_FILTERABLE`) | ETL 대량 유입으로 spots 수만 건 이상 |

수정 방향: title 검색은 `pg_trgm` GIN 인덱스, pipeline_status는 일반 인덱스 추가(partial index는 유지). count의 정확한 total이 필요한 이상(Refine `X-Total-Count` 계약) 근사치 전환은 별도 트레이드오프 검토가 필요하다. 두 항목 모두 프론트 계약에는 영향 없음.

## 3. 인프라 백로그

| 항목 | 상태 |
|---|---|
| 이미지 기능 운영 세팅(S3 버킷, CloudFront 오리진, IAM 권한, CORS) | v0.5.8 배포됨, 인프라 미설정 — 코드는 있지만 이미지 API가 계속 503 |
| DB 백업 이중화(pg_dump → S3) | 합의됐으나 착수 보류. RDS/Lightsail Free Tier retention 1일 고정 문제와 얽혀 있음 |
| `audit_log` 보관정책 | 무한 증가 대응 미정 — 데이터 쌓여 실측 필요해지는 시점에 재검토 |
| 수정 이력 화면 고도화 | 현재 최신 100건 고정, 페이지네이션 없음 (수요 발생 시) |

## 4. 데이터/설계 후속 과제

| 항목 | 배경 |
|---|---|
| `trust_tier` 갱신 쓰기 경로 부재 | `last_verified_at` 컬럼과 감쇠 배치는 있지만, 실제 재검증 시 이 컬럼을 갱신하는 PATCH/bulk 경로가 없다. 검증해도 감쇠 워터마크가 안 움직여 배치가 계속 강등시킬 수 있음 |
| ETL bulk upsert의 `pipeline_status` 강등 위험 | 기존 `PUBLISHED` row를 재업서트할 때 payload에 `pipeline_status`가 포함되면 강등될 수 있음. ETL 쪽 status 미포함 원칙 vs core의 강등 차단, 미결정 |
| `trust_tier` 자동 판정 로직 | 소스별 기본값 + 4대 속성(화기/접근성/편의시설/예약) 완비 체크 기반 자동 판정 로직 없음, 현재는 수동 판정 |
| 스팟 자체 신고(폐쇄/접근불가) 모델 부재 | 리뷰 신고(`spot_review_reports`)만 있고 스팟 자체 상태 이상을 알릴 통로 없음. 비즈니스 관점은 [../business/05-monetization-and-roadmap.md](../business/05-monetization-and-roadmap.md) 4.1 |

## 5. 문서 자체의 알려진 갭 (2026-07-20 문서 감사 기준)

- `docs/core/projects/vivac-console-backend.md` / `vivac-console-frontend.md`는 `/v1/admin/*` 경로 기준의 **낡은 설계 스냅샷**이다. 실제 구현은 `/v1/internal/*`로 확정됐다 — 이 devel 폴더와 [02-api-reference.md](./02-api-reference.md)가 최신 기준.
- `docs/core/skill-db-inspect.md`가 `.claude/skills/db_inspect/SKILL.md`로 이관될 예정이었으나, 해당 디렉터리가 아직 없다.
- `architecture.md`/`erd.md`(저장소 문서)에 최종 갱신일이 없어 실제 코드와의 괴리 정도를 문서만으로 판단하기 어렵다 — 이 devel 폴더는 2026-08-01 코드 기준으로 재조사해 작성됐다.
- `docs/samples/spots_bulk_sample.json`, `spot_business_info_bulk_sample.json` 샘플 파일이 관련 문서 어디에서도 링크되지 않음.

## 6. 마이그레이션 작업 시 주의 (트러블슈팅 기록)

- 브랜드 뉴 enum + 브랜드 뉴 테이블을 함께 만들 때는 `sa.Enum(...)`을 컬럼에 직접 넣고 `create_table`이 타입 생성까지 관리하게 둔다. `create_type=False` 사전생성 조합은 **기존 컬럼에 enum을 추가**할 때(`add_column`)만 쓰는 패턴 — 섞어 쓰면 `DuplicateObjectError`가 난다.
- 여러 git worktree를 병행할 때 로컬 Postgres 5432 포트를 공유하면 서로 다른 브랜치의 alembic 히스토리가 충돌한다. `DB_PORT`를 워크트리별로 분리해 우회한다.
- `ruff format` 실행 시 기존 파일 다수가 재작성 대상으로 뜨는 pre-existing drift가 있다(로컬 ruff 버전과 저장소 기존 스타일 간 불일치로 추정) — 무관 파일은 건드리지 않고, 자신이 변경한 파일만 포맷을 맞춘다.
