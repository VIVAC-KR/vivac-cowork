# 개발자 개요 — vivacapi-core

> 대상: `vivacapi-core` 저장소에 처음 합류하는 엔지니어
> 최종 정리일: 2026-08-01 (소스: `vivacapi-core` main 브랜치 코드 + `docs/core/architecture.md` 등 기존 문서 통합)
> 이 폴더(`devel/`)는 **개발자가 알아야 하는 모든 것**을 분야별로 나눈 것이다. 제품/운영 관점은 [`../business/`](../business/README.md) 참고.

---

## 1. 한 줄 요약

VIVAC은 캠핑/노지/차박 스팟 정보를 큐레이션하는 서비스다. `vivacapi-core`는 이 서비스의 단일 백엔드로, **공개 앱 API**(비로그인 탐색 + 로그인 유저 기능)와 **내부 운영 API**(`vivac-console`이라는 별도 Next.js 콘솔이 호출)를 하나의 FastAPI 프로세스에서 함께 서빙한다.

## 2. 기술 스택

| 구분 | 기술 | 비고 |
|---|---|---|
| Framework | FastAPI | Python 3.12+ |
| ORM | SQLAlchemy 2.x (asyncio) | 전 계층 `AsyncSession` + `await` |
| Database | PostgreSQL 16 | `asyncpg` 드라이버, local은 Docker Compose, prod는 AWS Lightsail Managed PostgreSQL |
| Migration | Alembic | autogenerate + 수동 마이그레이션 병행 (감사 트리거, CHECK 제약 등) |
| Auth | Google OAuth 2.0 ID Token → 자체 JWT(HS256) | 앱/콘솔/SQLAdmin 3가지 흐름, 상세는 [03-auth-and-security.md](./03-auth-and-security.md) |
| Admin UI | SQLAdmin(`/admin`, 비상용) + vivac-console(Refine, 일상 운영) | |
| Storage | S3(presigned URL) + CloudFront CDN | 이미지만 해당, 상세는 [05-domain-features.md](./05-domain-features.md) |
| Package Manager | `uv` | `uv sync`, `uv run ...` |
| Lint/Format | `ruff` (기본 line-length 88) | |
| Test | `pytest` + `pytest-asyncio`(`asyncio_mode=auto`) | 실제 Postgres 사용, 트랜잭션 롤백 격리 |
| Local Infra | Docker Compose | |
| Prod Infra | AWS Lightsail Instance($3.50) + Lightsail Managed PostgreSQL($15) | 상세는 [08-infra-and-deployment.md](./08-infra-and-deployment.md) |

## 3. 프로젝트 구조

```
vivacapi-core/
├── vivacapi/
│   ├── main.py              # FastAPI 앱, 미들웨어, 전역 예외 핸들러, SQLAdmin 마운트, lifespan(워커 기동)
│   ├── core/                 # 횡단 관심사
│   │   ├── config.py         # pydantic-settings 환경 설정 (prod 부팅 검증 포함)
│   │   ├── database.py       # SQLAlchemy async 엔진/세션/Base
│   │   ├── deps.py           # FastAPI 의존성: get_current_user, require_staff, require_role
│   │   ├── errors.py         # ErrorCode + AppException (표준 에러 봉투)
│   │   ├── limits.py         # bulk 요청 크기 제한
│   │   ├── security.py       # Google ID Token 검증, JWT 생성/디코딩
│   │   ├── storage.py        # S3 presigned URL / CDN URL 헬퍼
│   │   ├── nickname.py       # 랜덤 닉네임 생성
│   │   └── region.py         # 시/도 축약
│   ├── models/                # SQLAlchemy ORM
│   ├── schemas/                # Pydantic 요청/응답 모델
│   ├── crud/                    # DB 쿼리 함수 (정렬/필터 화이트리스트 포함)
│   ├── api/v1/
│   │   ├── routers.py          # /v1 라우터 조립
│   │   └── endpoints/          # auth, explore, spot_groups, spot_reviews, invites, admin_auth, internal_*
│   ├── admin/                    # SQLAdmin 인증 백엔드
│   └── workers/                  # 인프로세스 비동기 잡 워커 + bulk upsert 핸들러
├── alembic/                        # DB 마이그레이션
├── scripts/                          # export_openapi.py, decay_trust_tier.py, upload_spots_local.py
├── tests/                              # pytest (실제 PostgreSQL, 트랜잭션 롤백 격리)
├── docker-compose.yml                   # Local PostgreSQL
├── infra/docker-compose.yml               # Prod 컨테이너 정의 (deploy.yml이 사용)
├── Makefile                                 # run / db-up / migrate / test / openapi / release
└── .env.example                               # 환경 변수 템플릿
```

## 4. 레이어드 아키텍처

```
Client (App / vivac-console / 브라우저)
        │ HTTP
        ▼
Routers (api/v1/endpoints)  ── HTTP 입출력, 검증, DI. 비즈니스 로직 최소화
        │                       internal/*은 라우터 단위 require_staff
        ▼
CRUD (crud/)  ──────────────── DB 쿼리. 정렬/필터 화이트리스트, 상태 전이 규칙
        │
        ▼
Models (models/)  ──────────── SQLAlchemy ORM 테이블 정의
        │
        ▼
PostgreSQL 16  ──────────────── Local: Docker Compose / Prod: Lightsail Managed DB
                                감사 트리거(audit_log)가 spots·spot_business_info 기록
```

의존 방향 원칙(`CLAUDE.local.md`):

- `routers`는 HTTP 입출력과 DI만 담당한다.
- `crud`는 DB 접근/쿼리를 담당한다.
- `schemas`는 요청/응답 모델을 정의한다.
- `models`는 SQLAlchemy ORM 테이블을 정의한다.
- `core`는 설정/보안/DB 연결 등 횡단 관심사를 담당한다.
- 비즈니스 로직은 가능하면 `crud` 또는 명시적 서비스 계층으로 옮기고 라우터는 얇게 유지한다.

`workers`는 API 프로세스 안에서 asyncio task로 도는 잡 워커다 — 별도 프로세스가 아니다. 상세는 [04-async-jobs.md](./04-async-jobs.md).

## 5. 로컬 개발 명령어

```bash
# 의존성 설치
uv sync
uv sync --group dev

# 로컬 DB 기동 (Docker Compose)
make db-up          # = docker compose --env-file .env.local up -d db

# 마이그레이션 적용
make migrate         # = alembic upgrade head

# 개발 서버
make run              # = uvicorn vivacapi.main:app --reload

# 테스트 (vivac_test DB, .env.test 사용)
make test

# OpenAPI 스펙 추출 (docs/openapi.json, git 미추적)
make openapi

# 배포 (main 브랜치에서만, 버전 태그 push)
make release v=v0.x.0
```

> **로컬 실행/테스트는 반드시 `.env.local`을 사용한다.** `.env`가 아니다 — `Makefile`의 `ENV` 변수 기본값이 `.env.local`이며, git worktree에서도 메인 레포의 `.env.local`을 공통으로 찾도록 설계돼 있다.

## 6. 이 문서 폴더의 구성

| 문서 | 다루는 내용 |
|---|---|
| [01-data-model.md](./01-data-model.md) | ERD, 테이블별 컬럼/제약/인덱스, 전체 Enum 값 |
| [02-api-reference.md](./02-api-reference.md) | 전체 엔드포인트 인벤토리 (공개/인증/내부), 에러 코드, API 설계 규약 |
| [03-auth-and-security.md](./03-auth-and-security.md) | 인증 3가지 흐름, staff 권한 등급, 알려진 보안 이슈 |
| [04-async-jobs.md](./04-async-jobs.md) | 비동기 job 워커 설계, bulk upsert, audit log, trust_tier 감쇠 배치 |
| [05-domain-features.md](./05-domain-features.md) | Spot Group, Invite/리퍼럴, Review, 이미지 스토리지 — 도메인별 상세 |
| [06-search.md](./06-search.md) | 탐색 검색(PostgreSQL FTS + trigram) 설계 |
| [07-testing.md](./07-testing.md) | 테스트 전략, 픽스처, 실행 방법 |
| [08-infra-and-deployment.md](./08-infra-and-deployment.md) | AWS Lightsail 인프라, CI/CD, 환경 변수 |
| [09-known-issues-and-tech-debt.md](./09-known-issues-and-tech-debt.md) | 알려진 버그/보안 이슈/성능 이슈, backlog 취합 |

## 7. 코드 스타일 핵심 규칙 (요약)

전체 규칙은 저장소 `.claude/rules/*.md` 참고. 여기서는 자주 실수하는 항목만 요약한다.

- **화이트리스트 패턴**: 사용자 입력으로 정렬/필터 컬럼 등 ORM 속성을 동적으로 고를 때는 반드시 명시적 dict 화이트리스트를 거친다. 임의 속성 문자열을 바로 `getattr`하지 않는다.
- **crud 함수명**: `동사_대상_수식어` 순서 (예: `list_spots_admin`, `get_history`). 어드민/내부 전용 로직은 `_admin` 등으로 구분.
- **라우터 파일**: 도메인 단위로 쪼갠다. 하나의 거대 라우터에 몰아넣지 않는다.
- **Enum**: `enum.StrEnum` 사용.
- **주석**: 비즈니스 규칙의 "왜"가 코드만으로 드러나지 않을 때만 남긴다. 함수가 하는 일 자체를 설명하는 주석은 지양.
- **`/v1/internal/*`만**: `vivac-console`과 통신할 땐 반드시 `/v1/internal/...` 형식만 쓴다. 유일한 예외는 로그인(`/v1/admin/auth/google`).
