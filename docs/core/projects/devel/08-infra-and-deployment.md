# 인프라 & 배포

> 소스: `docs/core/infra/lightsail-setup.md`(프로비저닝 원문), `.github/workflows/deploy.yml`, `.github/workflows/ci.yml`, `infra/docker-compose.yml`, `Makefile`.

## 1. 아키텍처

```
GitHub (tag push: v*.*.*)
      │
      ▼
GitHub Actions (deploy.yml) ──── docker buildx (linux/amd64) push ──▶ Docker Hub
      │
      ▼ (SSH: ec2-user)
Lightsail Instance  ◀──── private network ───▶  Lightsail Managed PostgreSQL
($3.50 / Amazon Linux 2023 / Docker)             ($15 Standard, 1GB RAM)
      │
      ├─ alembic upgrade head (버전 태그 이미지로)
      ├─ docker compose up -d (IMAGE_TAG 고정 — 마이그레이션과 동일 이미지)
      └─ /health 헬스체크
```

**배포는 `main` push가 아니라 버전 태그(`v*.*.*`) push 시 트리거된다** — `make release v=v0.x.0`이 `main` 브랜치·클린 워킹트리를 확인한 뒤 태그를 push한다.

## 2. 비용 (초기 3개월 무료 티어 이후)

| 리소스 | 비용 |
|---|---|
| Lightsail Instance ($3.50 번들, 1 vCPU/512MB/20GB SSD) | $3.50/월 |
| Lightsail Managed PostgreSQL ($15 Standard, 1GB RAM) | $15/월 |
| Static IP (attach 상태) | $0 |
| 송수신 1TB/월 | $0 (번들 포함) |
| **합계** | **약 $18.50/월** |

무료 티어는 신규 Lightsail 사용자에게 1회만 제공된다. 절대 켜면 안 되는 항목: 자동/수동 스냅샷, Load Balancer($18/월), CDN/Distribution, 4번째 이상 DNS Zone, Static IP 미할당 방치.

## 3. 프로비저닝 절차 요약

1. SSH 키 페어 등록 (`vivac-prod-key`)
2. Lightsail Instance 생성 — Amazon Linux 2023, `ec2-user`, `$3.50/월` 번들, **자동 스냅샷 OFF**
3. 인스턴스 방화벽: SSH(22, 가능하면 본인 IP만), HTTP(80), HTTPS(443)
4. Static IP 생성 후 인스턴스에 attach
5. Lightsail Managed PostgreSQL 생성 — PostgreSQL 16.x, `$15 Standard`, **Public mode OFF**(같은 리전·계정의 Lightsail 인스턴스만 접근 가능)
6. 인스턴스에서 DB 연결 확인 (`psql`)
7. 인스턴스에 Docker 설치
8. `/home/ec2-user/vivac/.env` 작성 (`chmod 600`) — deploy.yml이 이 경로를 전제
9. 로컬에서 `docker buildx build --platform linux/amd64 ... --push` — Apple Silicon 등에서 그냥 빌드하면 `linux/arm64`가 되어 인스턴스에서 `exec format error` 발생
10. 수동 `docker run` sanity check + `alembic upgrade head` + `/health` 확인
11. (선택) 도메인 연결 + Caddy로 Let's Encrypt 자동 SSL

전체 명령어와 스크린샷 수준 상세는 저장소 `docs/core/infra/lightsail-setup.md` 원문 참고.

## 4. 환경 변수 (prod `.env`)

`core/config.py`의 `_validate_prod_requirements`가 부팅 시 검증한다 — 실패하면 부팅 자체가 안 된다. 상세는 [03-auth-and-security.md](./03-auth-and-security.md) 5장.

- `JWT_SECRET_KEY` / `ADMIN_SESSION_SECRET`은 로컬/dev와 **절대 동일한 값을 쓰지 않는다**. 서로도 달라야 한다.
- `CORS_ALLOWED_ORIGINS`에 `localhost`나 `*`가 섞이면 prod 부팅 실패.
- 이미지 스토리지(`S3_BUCKET`/`CDN_BASE_URL`)는 선택 — 미설정 시 이미지 API만 503.

## 5. CI/CD 워크플로우

| 워크플로우 | 트리거 | 내용 |
|---|---|---|
| `ci.yml` | PR | `ruff` lint/format → `alembic upgrade head` → `pytest` |
| `deploy.yml` | 버전 태그(`v*.*.*`) push | Docker 이미지 빌드(linux/amd64) → Docker Hub push(버전 태그 + latest) → SSH로 Lightsail 접속 → `alembic upgrade head` → `docker compose up -d`(고정 `IMAGE_TAG`) → `/health` 확인 |

**알려진 이슈**: `deploy.yml`의 SSH 스크립트가 git tag 이름을 `${{ }}` 문자열 치환으로 그대로 삽입하고 있어 셸 injection 패턴에 해당한다(tag push 권한자로 실효는 한정적). 상세는 [09-known-issues-and-tech-debt.md](./09-known-issues-and-tech-debt.md).

## 6. 백업

RDS/Lightsail Managed DB 자동 백업 retention이 Free Tier 제한으로 1일 고정이며, 상향 시도 시 `FreeTierRestrictionError`가 발생하고 콘솔에서도 해제할 수 없다(AWS Support 티켓 필요). 별도 `pg_dump` → S3 백업 이중화 방안이 논의됐으나 **착수는 보류** 상태다. 상세 계획은 저장소 `docs/core/backlog.md`의 "DB 백업 이중화" 섹션 참고.

## 7. 자원 정리 순서 (인스턴스 삭제 시)

컨테이너 정지 → 인스턴스 삭제 → Static IP 삭제 → DB 삭제 → DNS Zone/SSH key 삭제. Static IP를 인스턴스보다 먼저 detach하면 시간당 과금이 발생하므로, 인스턴스를 먼저 삭제해 자동 detach되는 흐름이 안전하다.
