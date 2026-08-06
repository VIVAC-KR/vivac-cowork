# VIVAC 문서 마스터 인덱스

`docs/` 전체(제품 공유 문서 + 5개 repo 전용 폴더)와 `design/`(vivac-frontend 전용, 미러링 안 됨)를 한눈에 조감하기 위한 최상위 내비게이션입니다. 각 폴더의 세부 규칙은 [`front/INDEX.md`](front/INDEX.md)(front repo 문서 구조), [`design/INDEX.md`](../../vivac-frontend/design/INDEX.md)(디자인 문서 구조), [`.claude/rules/vivac-docs-authoring.md`](../.claude/rules/vivac-docs-authoring.md)(작성 규칙)를 참고하세요.

진행 상황·미해결 이슈·핵심 결정 이력은 문서마다 흩어져 있어 따로 모았습니다 → **[STATUS.md](STATUS.md)**.

## 0. 지금 가장 먼저 봐야 할 문서

1. [PRODUCT.md](PRODUCT.md) — 제품 정의 확정본
2. [STATUS.md](STATUS.md) — 미해결 이슈·진행 상황·결정 로그 종합
3. [feature-spec.md](feature-spec.md) — 화면 단위 상세 기능 기획(정상구현/반쪽구현/API선구현/완전미구현)
4. [ia.md](ia.md) — 정보 구조(사이트맵·화면 인벤토리·내비게이션)
5. [business-feature-roadmap.md](business-feature-roadmap.md) — 실제 구현 스냅샷 기준 기능 로드맵

## 1. 제품 전체 (docs/ 루트 — 여러 repo에 걸친 맥락)

| 문서 | 내용 | 상태 |
|---|---|---|
| [PRODUCT.md](PRODUCT.md) | 제품 개요 — 문제 정의, 타겟, 차별화, 플랫폼, MVP 범위, 데이터 전략, 수익 모델, 로드맵 | ✅ 확정본 (2026-08-04 통합) |
| [PRODUCT_TEMP.md](PRODUCT_TEMP.md) | 과거 개정 초안 | 폐기 — PRODUCT.md로 병합 완료 |
| [data-pipeline.md](data-pipeline.md) | 스팟 데이터 `pipeline_status`/`trust_tier` 필드 설계 | ✅ 확정, vivacapi-core에 구현됨 |
| [business-feature-roadmap.md](business-feature-roadmap.md) | 성장/리텐션/수익화/신뢰 4관점 기능 후보 — 코드 대조로 근거 검증됨 | 진행 중 (항목별 상태는 STATUS.md 참고) |
| [feature-spec.md](feature-spec.md) | 화면 단위 상세 기능 기획 — 실서비스(vivac.app)·API 대조로 확인한 정상구현/반쪽구현/API선구현/완전미구현 4갈래를 통합 | 🆕 2026-08-04 작성 |
| [ia.md](ia.md) | 정보 구조(IA) — 사이트맵, 화면 인벤토리, 내비게이션 구조, 아직 배치 못 정한 것 | 🆕 2026-08-04 작성 |
| [CONTEXT_SCOPE.md](CONTEXT_SCOPE.md) | 각 repo CLAUDE.md가 import하는 공유 컨텍스트 기본 참고 범위 안내 | ✅ 안정 |
| [TEMP.md](TEMP.md) | 스크래치 메모("hello22!") | 내용 없음 — 정리 대상 |
| [archive/planning-source/](archive/planning-source/) | PRODUCT.md 병합에 쓰인 기획·리서치 원본 (2026-08-04 통합) | 폐기 3건 + 미해결/유효 3건 혼재, 각 파일 상단 상태 헤더 참고 |

## 2. front — VIVAC-frontend (`docs/front/`)

VIVAC-frontend 저장소 자체 `docs/` 구조를 그대로 미러링합니다(수정 없이 복사). 구조 설명은 [front/INDEX.md](front/INDEX.md) 참고.

| 문서 | 내용 | 상태 |
|---|---|---|
| [front/INDEX.md](front/INDEX.md) | front repo 문서 구조·SoT 우선순위 안내 | ✅ |
| [front/archive/auth-implementation.md](front/archive/auth-implementation.md) | 구 인증 구현(react-oauth/google) 설명 | 폐기 — NextAuth v5 전환으로 대체, 대체 문서 없음(공백) |
| [front/archive/spots-explore-plan.md](front/archive/spots-explore-plan.md) | `/spots` 리스트·지도 탐색 설계 | 폐기 — 코드가 제거되어 문서만 남음, 재구현 시 참고자료로만 사용 |
| [front/backlog/codebase-review-260714.md](front/backlog/codebase-review-260714.md) | 2026-07-14 전체 코드베이스 리뷰 | 🔴 오픈 이슈 다수 (STATUS.md 참고) |
| [front/backlog/spot-detail-design-followups.md](front/backlog/spot-detail-design-followups.md) | 상세페이지 UI 후속 논의·미해결 사항 | 열림 |
| [front/backlog/spot-detail-schema-request.md](front/backlog/spot-detail-schema-request.md) | 상세페이지용 BE 스키마 확장 요청 | [core/projects/spot-detail-fields.md](core/projects/spot-detail-fields.md)에서 처리됨 |
| [front/backlog/search-map-schema-request.md](front/backlog/search-map-schema-request.md) | 지도 탐색용 BE 계약 요청 (좌표·bbox·total 등 6건) | 🆕 2026-08-04 작성 — BE 회신 대기 |
| [front/decisions/incidents/cloudfront-nextjs-rsc-caching.md](front/decisions/incidents/cloudfront-nextjs-rsc-caching.md) | CloudFront+Next.js RSC 캐싱 장애 기록 | 해결됨(재발 방지용 필독) |
| [front/projects/search.md](front/projects/search.md) | 검색 라우팅 골격 설계 (2026-07-28) | 🚧 골격만 완료, 실제 검색·필터는 후속 |
| [front/projects/search-map-explore.md](front/projects/search-map-explore.md) | 검색 지도 탐색(목록/지도 2모드) 설계 + 목 기반 퍼블리싱 계획 | ✅ 1단계(SDK 없이) 구현 완료 2026-08-06 — 2단계는 지도 SDK 연동, 좌표 실데이터 대기 |
| [front/reviews/code-review-2026-07-22.md](front/reviews/code-review-2026-07-22.md) | 2026-07-22 코드 리뷰 — 인증 토큰 refresh 체인 잠복 결함, 배포 인프라 타 서비스 순단 | 🆕 repo 로컬 docs에서 이전 |
| [front/reference/frontend/api-proxy.md](front/reference/frontend/api-proxy.md) | Next.js API 프록시 구조 | ⚠️ 일부 낡음 — `route.ts` 설명이 실제로는 미사용 코드 (front/INDEX.md 알려진 한계 참고) |
| [front/reference/infra/docker-deployment.md](front/reference/infra/docker-deployment.md) | Docker 빌드·배포 구성 | ✅ |
| [front/templates/*.md](front/templates/) | ADR·incident·reference 작성 템플릿 3종 | ✅ |

## 3. console — vivac-console (`docs/console/`)

| 문서 | 내용 | 상태 |
|---|---|---|
| [console/audit-history-api.md](console/audit-history-api.md) | 수정 이력(Audit History) API 프론트 연동 가이드 | ✅ |
| [console/projects/pipeline-status-review-api.md](console/projects/pipeline-status-review-api.md) | 데이터 검증 화면용 BE API 요청 명세 | ✅ 구현 완료 (core 브랜치, 미push) |
| [console/reviews/codebase-review-260714.md](console/reviews/codebase-review-260714.md) | 2026-07-14 코드베이스 리뷰 | 오픈 이슈 소수 |
| [console/reviews/code-review-2026-07-22.md](console/reviews/code-review-2026-07-22.md) | 2026-07-22 코드 리뷰 — `/v1/internal` 규칙 준수 확인, 에러 envelope 계약 불일치 실버그 1건 | 🆕 repo 로컬 docs에서 이전 |
| [console/reviews/refactor-260802.md](console/reviews/refactor-260802.md) | 2026-08-02 전체 리팩터링 기록(36파일, sub agent 병렬 작업) | 🆕 repo 로컬 docs에서 이전 |
| [console/spot-sdp-field-mapping.md](console/spot-sdp-field-mapping.md) | 화면 필드 ↔ DB 컬럼 매핑표 | ✅ |

## 4. core — vivacapi-core (`docs/core/`)

| 문서 | 내용 | 상태 |
|---|---|---|
| [core/architecture.md](core/architecture.md) | API 아키텍처 개요 (living document) | ✅ |
| [core/erd.md](core/erd.md) | DB ERD (Mermaid) | ✅ |
| [core/enums.md](core/enums.md) | 전체 StrEnum 값 정리 | ✅ |
| [core/backlog.md](core/backlog.md) | 우선순위 미정 백로그 4건(이미지 인프라, DB 백업 이중화, audit_log 보관정책, rate limiting) | 열림 |
| [core/backlog/*.md](core/backlog/) | 2026-07-11 점검 발견 이슈 6건 (인덱스/스케일 2, 보안 4) | 열림 — 심각도 대부분 낮음~중간 |
| [core/code-review-2026-07-22.md](core/code-review-2026-07-22.md) | 전체 코드 리뷰 — Critical 1건(git history 노출 secret), High 1건 | 🔴 Critical 항목 확인 필요 |
| [core/reviews/known-issues.md](core/reviews/known-issues.md) | vivacapi-core 자체 문서 감사 결과 취합 | 문서 간 모순 3건 반영 완료, 잔여 갭 기록 중 |
| [core/security/db-security-review-2026-05-02.md](core/security/db-security-review-2026-05-02.md) | DB 스키마 보안 점검 (2026-05-02) | 후속 처리 현황 갱신됨(2026-07-14) |
| [core/troubleshooting/2026-08-03-nginx-stale-upstream-502.md](core/troubleshooting/2026-08-03-nginx-stale-upstream-502.md) | nginx 502 장애(89분) 기록 | 복구 완료 |
| [core/infra/lightsail-setup.md](core/infra/lightsail-setup.md) | AWS Lightsail 프로비저닝 가이드 | ✅ |
| [core/test-setup.md](core/test-setup.md) | 테스트 환경 구성 | ✅ |
| [core/skill-db-inspect.md](core/skill-db-inspect.md) | `db_inspect` Claude Skill 초안 | 미착수 — 실제 위치로 이동 안 됨 |
| [core/projects/vivac-console-backend.md](core/projects/vivac-console-backend.md) | console용 admin API 계약 | ⚠️ 스냅샷 — 실제 경로 `/v1/admin/*`→`/v1/internal/*`로 변경됨, [architecture.md](core/architecture.md)가 최신 |
| [core/projects/vivac-console-frontend.md](core/projects/vivac-console-frontend.md) | console repo 초기 세팅 | ⚠️ 위와 동일 사유로 일부 낡음 |
| [core/projects/spot-invites.md](core/projects/spot-invites.md) | 초대 링크(Invite) 설계 | ⚠️ "1회용" 결정이 business-feature-roadmap 1.1에서 일반 리퍼럴에 한해 뒤집힘(각주 있음) |
| [core/projects/spot-detail-fields.md](core/projects/spot-detail-fields.md) | 상세 응답 필드 확장 (front 요청 대응) | ✅ 완료 |
| [core/projects/spot-groups-admin-api.md](core/projects/spot-groups-admin-api.md) | Spot Group 어드민 API | ✅ 최신 |
| [core/projects/spot-search-postgres-fts.md](core/projects/spot-search-postgres-fts.md) | 검색 설계 (PostgreSQL FTS+trigram) | ✅ |
| [core/projects/spot-bulk-and-admin.md](core/projects/spot-bulk-and-admin.md) | 스팟 데이터 일괄 적재·백오피스 | ✅ |
| [core/projects/async-job-worker-design.md](core/projects/async-job-worker-design.md) | 비동기 Job 워커 설계 | ✅ |
| [core/projects/vvc-105-explore-api-spec.md](core/projects/vvc-105-explore-api-spec.md) | 탐색 API 스펙(VVC-105) | ✅ 1단계 완료, 후속 VVC-117/118/119 |

### 4.1 core/projects/business, core/projects/devel — 비즈니스/개발 문서 세트

코드·기존 문서를 조사해 신규 합류자용으로 재구성한 관점별 문서 세트(2026-08-01/06 작성). 비즈니스 관점은 `business/`, 개발자 관점은 `devel/`이며 서로 상호 링크되어 있다.

| 문서 | 내용 | 상태 |
|---|---|---|
| [core/projects/business/README.md](core/projects/business/README.md) | 비즈니스 문서 세트 개요 | ✅ |
| [core/projects/business/00-product-overview.md](core/projects/business/00-product-overview.md) | 제품 개요(PRODUCT.md 요약본) | ✅ |
| [core/projects/business/01-domain-glossary.md](core/projects/business/01-domain-glossary.md) | 도메인 용어집(비즈니스 관점) | ✅ |
| [core/projects/business/02-core-features.md](core/projects/business/02-core-features.md) | 핵심 기능 & 유저 플로우 | ✅ |
| [core/projects/business/03-admin-console-operations.md](core/projects/business/03-admin-console-operations.md) | 운영 콘솔 & 내부 운영 프로세스 | ✅ |
| [core/projects/business/04-data-pipeline-and-quality.md](core/projects/business/04-data-pipeline-and-quality.md) | 데이터 파이프라인 & 신뢰도 정책 | ✅ |
| [core/projects/business/05-monetization-and-roadmap.md](core/projects/business/05-monetization-and-roadmap.md) | 수익화 & 비즈니스 기능 로드맵 | ✅ |
| [core/projects/business/06-known-risks-and-open-decisions.md](core/projects/business/06-known-risks-and-open-decisions.md) | 알려진 리스크 & 미결정 사항(비즈니스 관점) | ✅ |
| [core/projects/devel/README.md](core/projects/devel/README.md) | 개발 문서 세트 개요 | ✅ |
| [core/projects/devel/00-overview.md](core/projects/devel/00-overview.md) | 개발자 개요 — vivacapi-core | ✅ |
| [core/projects/devel/01-data-model.md](core/projects/devel/01-data-model.md) | 데이터 모델 — ERD & Enum | ✅ |
| [core/projects/devel/02-api-reference.md](core/projects/devel/02-api-reference.md) | API 레퍼런스(엔드포인트 전수 조사) | ✅ |
| [core/projects/devel/03-auth-and-security.md](core/projects/devel/03-auth-and-security.md) | 인증 & 보안 | ✅ |
| [core/projects/devel/04-async-jobs.md](core/projects/devel/04-async-jobs.md) | 비동기 Job 워커 & 배치 | ✅ |
| [core/projects/devel/05-domain-features.md](core/projects/devel/05-domain-features.md) | 도메인 기능 상세 — Group/Invite/Review/Image | ✅ |
| [core/projects/devel/06-search.md](core/projects/devel/06-search.md) | 탐색 검색 설계 — PostgreSQL FTS + trigram | ✅ |
| [core/projects/devel/07-testing.md](core/projects/devel/07-testing.md) | 테스트 전략 | ✅ |
| [core/projects/devel/08-infra-and-deployment.md](core/projects/devel/08-infra-and-deployment.md) | 인프라 & 배포 | ✅ |
| [core/projects/devel/09-known-issues-and-tech-debt.md](core/projects/devel/09-known-issues-and-tech-debt.md) | 알려진 이슈 & 기술 부채(취합본) | ✅ |

## 5. etl — vivacapi-etl (`docs/etl/`)

| 문서 | 내용 | 상태 |
|---|---|---|
| [etl/source1_transform.md](etl/source1_transform.md) | source1(GoCamping, 2,897건) → spots 변환 작업 기록 | ✅ |
| [etl/decisions/source1_transform_changelog.md](etl/decisions/source1_transform_changelog.md) | 변환 규칙 변경 이력 | ✅ |
| [etl/reviews/source1_transform_review.md](etl/reviews/source1_transform_review.md) | 변환 결과 검토 보고서 | ✅ |
| [etl/naver_favorites_import.md](etl/naver_favorites_import.md) | 네이버 지도 즐겨찾기 → spots 임포트 기록 | ✅ |
| [etl/reviews/code-review-2026-07-22.md](etl/reviews/code-review-2026-07-22.md) | 2026-07-22 코드 리뷰 — prod DB 자격증명 하드코딩·가드 없는 TRUNCATE CASCADE 등 데이터 유실/유출 리스크 | 🔴 repo 로컬 docs에서 이전, 비밀번호 값은 redact 처리(STATUS.md §0 참고) |

## 6. mcp — vivac-mcp (`docs/mcp/`)

| 문서 | 내용 | 상태 |
|---|---|---|
| [mcp/projects/plan.md](mcp/projects/plan.md) | VIVAC MCP Connector 기획 — 자연어 스팟 검색 | 기획 단계 |
| [mcp/projects/cli-plan.md](mcp/projects/cli-plan.md) | VIVAC CLI 기획 — plan.md 위에 세워진 3번째 소비자 | 기획 단계, 열린 질문 있음 |
| [mcp/reviews/code-review-2026-07-22.md](mcp/reviews/code-review-2026-07-22.md) | 2026-07-22 코드 리뷰 — `client.py` 테스트 부재, 네트워크 예외처리·로깅 전무 | 🆕 repo 로컬 docs에서 이전 |

## 7. design — vivac-frontend 전용 (`design/`, 미러링 안 됨)

`docs/`와 달리 vivac-frontend repo에만 있고 vivac-cowork로 복사되지 않습니다. 구조는 [design/INDEX.md](../../vivac-frontend/design/INDEX.md) 참고.

| 문서 | 내용 | 상태 |
|---|---|---|
| [design/INDEX.md](../../vivac-frontend/design/INDEX.md) | 디자인 문서 구조·SoT 우선순위 | ✅ |
| [DESIGN.md](../../vivac-frontend/DESIGN.md) | 디자인 토큰 원시값 정본(자동 생성, 직접 수정 금지) | ✅, 편차는 `globals.css` 헤더가 정본 |
| [design/reference/pages/spot-detail.md](../../vivac-frontend/design/reference/pages/spot-detail.md) | 상세페이지 UI 스펙 v3 | ✅ 구현 반영됨(PR #23) |
| [design/decisions/spot-detail-design-decisions.md](../../vivac-frontend/design/decisions/spot-detail-design-decisions.md) | 위 스펙 의사결정 경위 | ✅ |
| [design/design-ref/spot-carousel/](../../vivac-frontend/design/design-ref/spot-carousel/) | 이미지 캐러셀 레퍼런스 자료 | 참고용 |

## 8. 이 인덱스 유지하는 법

새 문서를 추가할 때 이 표에도 한 줄을 함께 추가하세요. `.claude/rules/vivac-docs-authoring.md`에 "새 문서 작성 시 docs/INDEX.md 갱신"을 규칙으로 추가하는 것을 권장합니다(아직 미반영 — STATUS.md 참고).
