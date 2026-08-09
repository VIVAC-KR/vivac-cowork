# vivacapi-core — 개발자 문서

> `vivacapi-core` 저장소(VIVAC 백엔드 API)의 개발자용 문서 모음. 2026-08-01 기준 코드를 직접 조사해 작성했다.
> 제품/운영 관점 문서는 [`../business/`](../business/README.md) 참고. 기존 기능별 설계 노트 원문은 상위 폴더([`../`](../))에 그대로 남아 있다.

## 읽는 순서 (신규 합류자 기준)

1. [00-overview.md](./00-overview.md) — 기술 스택, 프로젝트 구조, 레이어드 아키텍처, 로컬 개발 명령어
2. [01-data-model.md](./01-data-model.md) — ERD, 테이블별 컬럼/제약/인덱스, 전체 Enum
3. [02-api-reference.md](./02-api-reference.md) — 전체 엔드포인트 인벤토리 (공개/인증/내부 어드민)
4. [03-auth-and-security.md](./03-auth-and-security.md) — 인증 3가지 흐름, staff 권한 등급, 보안 이슈
5. [04-async-jobs.md](./04-async-jobs.md) — 비동기 job 워커, bulk upsert, audit log, trust_tier 배치
6. [05-domain-features.md](./05-domain-features.md) — Spot Group / Invite / Review / 이미지 스토리지 상세
7. [06-search.md](./06-search.md) — 탐색 검색(PostgreSQL FTS + trigram) 설계
8. [07-testing.md](./07-testing.md) — 테스트 전략, 픽스처, 파일 목록
9. [08-infra-and-deployment.md](./08-infra-and-deployment.md) — AWS Lightsail 인프라, CI/CD
10. [09-known-issues-and-tech-debt.md](./09-known-issues-and-tech-debt.md) — 알려진 버그/보안/성능 이슈

## 분야별 빠른 링크

| 궁금한 것 | 문서 |
|---|---|
| 새 엔드포인트를 어디에 추가해야 하나 | [00-overview.md](./00-overview.md) §3-4 (레이어 구조) |
| 특정 테이블의 컬럼/제약이 궁금 | [01-data-model.md](./01-data-model.md) |
| 이 API가 staff 권한이 필요한가 | [02-api-reference.md](./02-api-reference.md), [03-auth-and-security.md](./03-auth-and-security.md) §3 |
| 로그인 토큰이 왜 이렇게 동작하나 | [03-auth-and-security.md](./03-auth-and-security.md) §1-2 |
| bulk upsert가 실패하면 어떻게 되나 | [04-async-jobs.md](./04-async-jobs.md) §6-7 |
| 그룹 초대는 되는데 리퍼럴은 왜 재사용 가능한가 | [05-domain-features.md](./05-domain-features.md) §2.2 |
| 검색이 오타를 어떻게 처리하나 | [06-search.md](./06-search.md) |
| 테스트가 왜 실제 DB를 쓰나 | [07-testing.md](./07-testing.md) §2 |
| 배포는 어떻게 트리거되나 | [08-infra-and-deployment.md](./08-infra-and-deployment.md) §1, §5 |
| 지금 알려진 버그/부채가 뭐가 있나 | [09-known-issues-and-tech-debt.md](./09-known-issues-and-tech-debt.md) |

## 문서 유지 원칙

- 이 폴더는 **현재 코드 상태의 스냅샷**이다. 라우터/모델/설정이 바뀌면 관련 문서도 같은 PR에서 갱신하는 것을 권장한다.
- 기능 설계 히스토리(왜 이렇게 결정했는가)가 필요하면 상위 폴더의 개별 설계 노트(`../async-job-worker-design.md`, `../spot-invites.md` 등)를 먼저 확인한다 — 이 폴더는 그 결정들의 **현재 결과**를 종합한 것이다.
