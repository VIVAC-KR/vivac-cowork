# 문서 인벤토리 및 SSOT 마이그레이션 계획

> ⚠️ **이 문서는 `main` 머지 이전(문서 69개) 기준으로 작성됐습니다.** 2026-09-07 `main`의 23커밋을 머지한 결과 문서가 107개가 됐고, 여기서 미해결로 보고한 여러 항목이 이미 해소됐습니다 — `PRODUCT_TEMP.md`·`TEMP.md` 삭제, 기능 명세의 `docs/features/` 분리, `PRODUCT.md` 2026-08-10 개정, `docs/design/` 편입에 따른 링크 정리, `openapi.json` 버전 명시 동기화 관행이 그렇습니다. 최신 기준 분석은 [boundary-conflicts-260907.md](boundary-conflicts-260907.md)를 참고하세요.

> 작성일: 2026-09-07
> 목적: 기존 `docs/` 문서를 SSOT 기반 문서 체계로 옮기기 위한 사전 조사입니다. 이 문서는 조사 결과와 계획만 담으며, 이번 작업에서 기존 문서를 이동·수정·삭제하지 않았습니다.
> 범위: `docs/` 하위 Markdown 69개 전수. `docs/openapi.json`(생성물)과 `.specify/memory/constitution.md`는 문서 체계 판단에 필요한 범위에서만 참조합니다.
> 표기: 요청한 10개 항목 중 Purpose와 Scope를 한 칸으로, Duplicates와 Conflicts를 한 칸으로 합쳤습니다. 셀 폭 때문이며 내용은 모두 담았습니다. 충돌 항목은 `C1`~`C10`, 중복 항목은 `D1`~`D8`로 본문 뒤 목록과 연결됩니다.

---

# 1. 전체 문서 인벤토리

## 1.1 `docs/` 루트 — 여러 repo에 걸친 제품 문서

| Path | Type | Purpose / Scope | Potential SSOT | Dependencies | Duplicates / Conflicts | Status | Action |
|---|---|---|---|---|---|---|---|
| `PRODUCT.md` | Product | 제품 정의 확정본. 문제·타겟·차별화·플랫폼·MVP 범위·데이터 전략·수익 모델·로드맵·미해결 이슈 | **제품 정의·문제·타겟·핵심 가치·MVP 범위의 SSOT** | `feature-spec.md`, `data-pipeline.md`, `business-feature-roadmap.md`, `archive/planning-source/*` | 기능 정의가 `feature-spec.md`·`business-feature-roadmap.md`와 3중 정의(D4). MVP 범위가 실제 구현과 불일치(C7) | Active | **Keep** — SSOT 지정, 기능 범위 절만 링크로 축약 |
| `PRODUCT_TEMP.md` | Product | 병합 완료 표식만 남은 4줄 스텁 | 없음 | `PRODUCT.md` | `PRODUCT.md`로 병합 완료(D7) | Duplicate | **Archive** — `archive/planning-source/`로 이동 |
| `feature-spec.md` | Feature | 실서비스 점검 기반 화면 단위 기능 기획. 정상구현/반쪽구현/API선구현/완전미구현 4갈래 | **화면 단위 기능 명세의 SSOT**(단, Feature 단위 분해 필요) | `PRODUCT.md`, `ia.md`, `STATUS.md`, `business-feature-roadmap.md` | 기능 정의 3중(D4). 구현 상태 기록 4중(D3). `business-feature-roadmap.md` 1.2와 상세페이지 존재 여부 충돌(C1) | Active | **Split** — 기능별 Feature Specification으로 분해 |
| `business-feature-roadmap.md` | Feature | 성장·리텐션·수익화·신뢰 4관점 기능 후보 16건, 코드 대조 근거 포함 | 기능 후보 백로그의 SSOT(제안 12건 한정) | 코드베이스(`vivacapi-core`), `PRODUCT.md` | 기능 정의 3중(D4). 완료 3건은 결정 기록과 성격 중복. 1.2 전제가 낡음(C1) | Active(일부 Outdated) | **Split** — 완료 3건은 Decision, 제안 12건은 Backlog |
| `ia.md` | IA | 사이트맵, 화면 인벤토리, 내비게이션 구조, 미결 배치 3건 | **정보 구조·화면 구조·라우팅의 SSOT** | `feature-spec.md`, `PRODUCT.md`, `STATUS.md` | 화면 목록이 `feature-spec.md` §0과 부분 중복(D4) | Active | **Keep** — SSOT 지정 |
| `data-pipeline.md` | Decision | `pipeline_status`/`trust_tier` 필드 설계와 확정 정책(tier 3 노출, 안전 속성 표시 규칙) | **데이터 신뢰도·노출 정책(비즈니스 규칙)의 SSOT** | `vivacapi-core` 구현 | 상태값 정의가 `core/enums.md`·`core/erd.md`와 3중(D1) | Active | **Keep** — 정책은 SSOT 유지, 값 목록은 `core/enums.md` 링크로 대체 |
| `STATUS.md` | Status | 미해결 이슈·진행 상황·팀 결정 대기·결정 로그 종합 스냅샷 | **구현 상태·결정 로그의 집계 SSOT**(파생 문서) | 거의 모든 문서 | 구현 상태 4중 기록(D3). §6 결정 로그가 개별 문서 결정과 중복(D3) | Active(수동 동기화) | **Keep** — 단, 파생 문서임을 명시하고 §6은 ADR 인덱스로 전환 |
| `INDEX.md` | Meta | 전체 문서 마스터 인덱스 + 문서별 상태 열 | 문서 위치 탐색의 SSOT | 모든 문서 | 상태 열이 `STATUS.md`와 중복(D3). design 링크 8건 깨짐(C10) | Active | **Merge** — 상태 열 제거해 `STATUS.md`로 위임, 탐색 기능만 유지 |
| `CONTEXT_SCOPE.md` | Meta | 각 repo `CLAUDE.md`가 import하는 공유 컨텍스트 참고 범위 | AI agent 컨텍스트 로딩 범위의 SSOT | `.claude/rules/vivac-docs-authoring.md` | "`docs/`는 심볼릭 링크"라는 서술이 이 저장소 기준으로는 틀림(C8) | Outdated(부분) | **Merge** — `docs/meta/DOCUMENTATION.md`로 흡수 |
| `TEMP.md` | Other | 스크래치 메모(`hello22!`) | 없음 | 없음 | 없음 | Outdated | **Delete** |
| `temp-adr/_template.md` | Meta | ADR 템플릿(Context/Drivers/Options/Decision/Consequences) | ADR 템플릿 후보 | 없음 | `front/templates/adr-template.md`와 포맷 상이한 2중 템플릿(D5) | Unknown(미도입) | **Merge** — 단일 ADR 템플릿으로 통합 |
| `temp-specs/_template.md` | Meta | Feature Specification 템플릿 17절(FR/BR/AC/Non-Goals/Open Questions 포함) | Feature Spec 템플릿 후보 | 없음 | Spec Kit `.specify/templates/spec-template.md`와 역할 중복(D5) | Unknown(미도입) | **Merge** — Spec Kit 템플릿과 단일화 |
| `temp-product/README.md` | Meta | 0바이트 빈 파일 | 없음 | 없음 | 없음 | Unknown | **Delete** |

## 1.2 `docs/archive/planning-source/` — 기획 원본

| Path | Type | Purpose / Scope | Potential SSOT | Dependencies | Duplicates / Conflicts | Status | Action |
|---|---|---|---|---|---|---|---|
| `VIVAC_기획서_문제정의.md` | Product | 최초 기획서 원본. 문제 정의·솔루션·리서치 인용 | 없음(근거 보존용) | `PRODUCT.md` | `PRODUCT.md`로 대체됨(D7) | Outdated(폐기 표기 완료) | **Keep as Archive** |
| `VIVAC_기획서_문제정의_수정안.md` | Product | 위 문서의 개정안. 시장·경쟁 분석, JTBD 상세, KPI 표 | 시장·경쟁 분석 배경 근거 | `PRODUCT.md`, `research_backpacking_market.md` | `PRODUCT.md`로 병합(D7). 플랫폼 전제가 `PRODUCT.md`와 상충(C5, 헤더로 완화됨) | Outdated(폐기 표기 완료) | **Keep as Archive** |
| `VIVAC_기획서_문구수정안_오프라인_로그인.md` | Product | PWA 폐기·소셜 로그인 문구 수정 제안 | 없음 | 위 수정안 | 수정안에 반영 완료(D7) | Outdated(폐기 표기 완료) | **Keep as Archive** |
| `research_backpacking_market.md` | Product | 2026-05 시장·문화 리서치 보고서(페르소나 근거) | **사용자 정의·시장 근거의 SSOT** | `PRODUCT.md` | 타겟 서술이 `PRODUCT.md`와 중복이나 상세도 다름(D7) | Active(유효 근거) | **Rename/Move** — `archive/`가 아닌 `docs/research/`로 승격 |
| `VIVAC_데이터실현성_스파이크_검증설계.md` | Product | 합법성 데이터 실현성 검증 설계(미실행) | **데이터 실현성 검증 설계의 SSOT** | `PRODUCT.md` 미해결 이슈, `feature-spec.md` §4.2 | archive 폴더 의미와 상태 불일치(D8) | Active(미실행) | **Move** — `archive/` 밖으로 이동 |
| `VIVAC_공유자제vs데이터해자_논의1페이저.md` | Product | 데이터 해자 방향 팀 결정용 1페이저(미해결) | **해당 결정 논의의 SSOT** | `PRODUCT.md` 미해결 이슈, `STATUS.md` §4 | archive 폴더 의미와 상태 불일치(D8) | Active(미결정) | **Move** — `archive/` 밖으로 이동 |

## 1.3 `docs/core/` — vivacapi-core

| Path | Type | Purpose / Scope | Potential SSOT | Dependencies | Duplicates / Conflicts | Status | Action |
|---|---|---|---|---|---|---|---|
| `core/architecture.md` | Architecture | API 아키텍처 개요(라우터·모델·설정), living document | **백엔드 기술 아키텍처의 SSOT** | `erd.md`, `openapi.json`, 코드 | `openapi.json`을 "git 미추적"으로 서술하나 실제로는 추적 중(C2). `/v1/internal/*` 경로 기준의 최신본이라 `projects/vivac-console-*.md`와 충돌(C3) | Active | **Keep** — SSOT 지정, C2 문구 수정 |
| `core/erd.md` | Architecture | 전체 테이블 ERD(Mermaid), 제약·인덱스 목록 | **데이터 구조(스키마)의 SSOT** | `vivacapi/models/` | `pipeline_status`/`trust_tier` 정의가 `data-pipeline.md`·`enums.md`와 3중(D1) | Active | **Keep** — SSOT 지정 |
| `core/enums.md` | Architecture | 백엔드 `StrEnum` 전체 값과 의미 | **Enum 값의 SSOT** | `vivacapi/models/`, `core/errors.py` | `PipelineStatus` 값이 `data-pipeline.md`·`erd.md`와 3중(D1) | Active | **Keep** — SSOT 지정 |
| `core/backlog.md` | Status | 우선순위 미정 백로그 5건(이미지 인프라, DB 백업, audit_log 보관, rate limit 등) | core 백로그의 SSOT | `backlog/*.md`, `STATUS.md` | 항목 상태가 `STATUS.md` §1과 중복(D3). auth rate limit 중복은 해소됨 | Active | **Keep** |
| `core/backlog/*.md` (7건) | Status | 2026-07-11 점검 발견 이슈 개별 기록(보안 4·성능 2, `admin-list-scale`·`auth-rate-limit`·`deploy-tag-injection`·`pipeline-status-index`·`private-image-exposure`·`prod-allowed-email-domain`·`sqladmin-session-cookie`) | 개별 이슈의 SSOT | `core/backlog.md`, `STATUS.md` | 건수·상태가 `STATUS.md` §1과 중복(D3) | Active | **Keep** |
| `core/code-review-2026-07-22.md` | Status | 전체 코드 리뷰. Critical 1건(git history secret 노출) 포함 | 해당 리뷰 결과의 SSOT | `STATUS.md` §1 | 완료 트래킹이 없어 `STATUS.md`가 처리 여부를 확정 못 함(D3) | Active(추적 미비) | **Keep** — 항목별 상태 표기 추가 필요 |
| `core/reviews/known-issues.md` | Status | vivacapi-core 자체 문서 감사 결과 취합, 문서 간 모순 3건 반영 | 문서 결함 추적의 SSOT | `architecture.md`, `backlog.md`, `projects/*` | `STATUS.md` §5와 중복(D3) | Active | **Merge** — `STATUS.md` §5로 통합 검토 |
| `core/security/db-security-review-2026-05-02.md` | Status | DB 스키마 보안 점검(2026-05-02), 후속 처리 현황 갱신됨 | 해당 점검의 SSOT | `erd.md` | 없음 | Active | **Keep** |
| `core/troubleshooting/2026-08-03-nginx-stale-upstream-502.md` | Decision | nginx 502 장애 89분 기록·재발 방지 | 해당 장애의 SSOT | 인프라 구성 | `front/decisions/incidents/`와 폴더 명칭 체계가 다름(D6) | Active | **Rename** — `core/decisions/incidents/`로 통일 |
| `core/infra/lightsail-setup.md` | Architecture | AWS Lightsail 프로비저닝 표준 절차 | 인프라 프로비저닝의 SSOT | 없음 | 없음 | Active | **Keep** |
| `core/test-setup.md` | Architecture | 테스트 환경 구성(2026-05-02, `feature/test-setup` 기준) | 테스트 환경의 SSOT | 없음 | 작성 시점이 오래되어 현행 확인 필요 | Unknown | **Keep** — 현행성 검증 필요 |
| `core/skill-db-inspect.md` | Other | `db_inspect` Claude Skill 초안. 실제 위치로 미이동 | 없음(문서가 아닌 도구) | `.claude/skills/` | 문서 체계에 속하지 않는 산출물이 docs에 체류 | Unknown | **Move** — `.claude/skills/db_inspect/SKILL.md`로 이동하거나 폐기 |
| `core/projects/vvc-105-explore-api-spec.md` | Feature | 탐색 API(검색·필터·리스트·상세) BE 스펙 확정 노트 | 탐색 API 계약의 설계 근거 | `openapi.json`, `spot-search-postgres-fts.md` | API 정의가 `openapi.json`·`architecture.md`와 분산(D2) | Active | **Keep** — Feature Spec 후보 |
| `core/projects/spot-search-postgres-fts.md` | Decision | 검색을 Elasticsearch 대신 PostgreSQL FTS+trigram으로 결정 | 검색 방식 결정의 SSOT | `vvc-105-explore-api-spec.md` | 없음 | Active | **Rename** — ADR로 승격 |
| `core/projects/spot-invites.md` | Feature | 초대·리퍼럴을 단일 `Invite` 엔티티로 통합한 설계 | 초대 도메인의 SSOT | `business-feature-roadmap.md` 1.1 | "1회용" 결정이 로드맵 1.1에서 뒤집힘(C4, 각주로 완화됨) | Active(부분 갱신) | **Keep** |
| `core/projects/spot-groups-admin-api.md` | Feature | Spot Group 어드민 API `/v1/internal/groups/*` 계약 | 그룹 어드민 API의 SSOT | `vivac-console-backend.md` | API 정의 분산(D2) | Active | **Keep** |
| `core/projects/spot-detail-fields.md` | Feature | 상세 응답 필드 확장(front 요청 대응) | 상세 응답 스키마 변경 근거 | `front/backlog/spot-detail-schema-request.md` | 필드 목록이 `openapi.json`과 중복(D2) | Active(완료) | **Archive** — 완료 작업 기록 |
| `core/projects/spot-bulk-and-admin.md` | Feature | 스팟 일괄 적재·내부 백오피스 백엔드 구성 | 벌크 적재 설계의 SSOT | `etl/*` | 없음 | Active | **Keep** |
| `core/projects/async-job-worker-design.md` | Decision | 외부 브로커 없이 FastAPI 내장 폴링 워커 채택 | 워커 구조 결정의 SSOT | `architecture.md` | 없음 | Active | **Rename** — ADR로 승격 |
| `core/projects/vivac-console-backend.md` | Feature | console용 admin API 계약(설계 스냅샷) | 없음(대체됨) | `architecture.md` | 경로가 `/v1/admin/*`로 실제와 불일치(C3, 각주로 완화됨) | Outdated | **Archive** |
| `core/projects/vivac-console-frontend.md` | Feature | console repo 초기 세팅(설계 스냅샷) | 없음(대체됨) | 위 문서 | 위와 동일(C3) | Outdated | **Archive** |

## 1.4 `docs/front/` — VIVAC-frontend(원본 repo 미러)

| Path | Type | Purpose / Scope | Potential SSOT | Dependencies | Duplicates / Conflicts | Status | Action |
|---|---|---|---|---|---|---|---|
| `front/INDEX.md` | Meta | front repo 문서 구조·SoT 우선순위·작성 원칙 | front repo 문서 규칙의 SSOT | `.claude/rules/vivac-docs-authoring.md` | 폴더 분류 체계가 cowork 작성 규칙과 다름(D6). SoT 우선순위에서 코드가 3순위인데 Constitution은 5순위(C9). design 링크 2건 깨짐(C10) | Active(충돌) | **Merge** — `docs/meta/DOCUMENTATION.md`로 규칙 단일화 |
| `front/archive/auth-implementation.md` | Architecture | 구 인증 구현(react-oauth/google) | 없음 | NextAuth v5 코드 | 대체 문서 "없음"으로 표기된 공백(C6 계열) | Outdated | **Keep as Archive** — 대체 Reference 신규 작성 필요 |
| `front/archive/spots-explore-plan.md` | Feature | `/spots` 리스트·지도 탐색 설계(코드 제거됨) | 없음 | `feature-spec.md` §4.1 | 지도 탐색 재기획이 `feature-spec.md` §4.1에 있어 참고 관계(D4) | Outdated | **Keep as Archive** |
| `front/backlog/codebase-review-260714.md` | Status | 2026-07-14 전체 코드베이스 리뷰. Tier 1 4건 전부 열림 | front 미해결 이슈의 SSOT | `STATUS.md` §1·§2 | `STATUS.md`와 중복 요약(D3) | Active | **Keep** |
| `front/backlog/spot-detail-design-followups.md` | Status | 상세페이지 UI 후속 논의·미해결 목록 | 해당 후속 이슈의 SSOT | vivac-frontend `design/` | design 링크 2건 깨짐(C10) | Active | **Keep** |
| `front/backlog/spot-detail-schema-request.md` | Feature | 상세페이지용 BE 스키마 확장 요청 | 없음(처리 완료) | `core/projects/spot-detail-fields.md` | 요청 필드 목록 중복(D2) | Outdated(처리됨) | **Archive** |
| `front/decisions/incidents/cloudfront-nextjs-rsc-caching.md` | Decision | CloudFront+Next.js RSC 캐싱 장애 기록 | 해당 장애의 SSOT | 인프라 | 없음 | Active | **Keep** |
| `front/projects/search.md` | Feature | 검색 라우팅 골격 설계(2026-07-28) | 검색 화면 라우팅의 SSOT | `ia.md`, `feature-spec.md` §1.3 | 라우팅 정의가 `ia.md`와 중복(D4) | Active(골격만) | **Keep** |
| `front/reference/frontend/api-proxy.md` | Architecture | Next.js API 프록시 구조·환경변수 | 프론트 프록시 구조의 SSOT | `next.config.ts` | `route.ts`를 유일 프록시로 설명하나 실제는 `rewrites()` 우선(C6) | Outdated(부분) | **Keep** — 갱신 필요 |
| `front/reference/infra/docker-deployment.md` | Architecture | Docker 빌드·배포 구성 | front 배포 구성의 SSOT | 없음 | 없음 | Active | **Keep** |
| `front/templates/adr-template.md` | Meta | ADR 작성 템플릿 | ADR 템플릿 후보 | 없음 | `temp-adr/_template.md`와 2중 템플릿(D5) | Active | **Merge** |
| `front/templates/incident-template.md` | Meta | 인시던트 기록 템플릿 | 인시던트 템플릿의 SSOT | 없음 | 없음 | Active | **Keep** |
| `front/templates/reference-template.md` | Meta | Reference 문서 템플릿 | Reference 템플릿의 SSOT | 없음 | 없음 | Active | **Keep** |

## 1.5 `docs/console/` — vivac-console

| Path | Type | Purpose / Scope | Potential SSOT | Dependencies | Duplicates / Conflicts | Status | Action |
|---|---|---|---|---|---|---|---|
| `console/audit-history-api.md` | Architecture | 수정 이력 API 프론트 연동 명세 | audit API 연동의 SSOT | `core/architecture.md`, `openapi.json` | API 정의 분산(D2) | Active | **Keep** |
| `console/spot-sdp-field-mapping.md` | Architecture | 콘솔 화면 필드 ↔ DB 컬럼 1:1 매핑표 | 콘솔 화면-데이터 매핑의 SSOT | `core/erd.md` | 컬럼 정의가 `erd.md`와 중복(D2). `trust_tier`가 콘솔에 없다는 갭 기록 | Active | **Keep** |
| `console/projects/pipeline-status-review-api.md` | Feature | 데이터 검증 화면용 BE API 요청 명세 + 구현 결과 | 검증 화면 API의 SSOT | `data-pipeline.md`, `core/architecture.md` | `pipeline_status` 전이 규칙이 `data-pipeline.md`와 중복(D1) | Active(구현 완료·미push) | **Keep** |
| `console/reviews/codebase-review-260714.md` | Status | 2026-07-14 코드베이스 리뷰. 열림 1·완료 20 | console 미해결 이슈의 SSOT | `STATUS.md` §1 | `STATUS.md`와 중복 요약(D3) | Active | **Keep** |

## 1.6 `docs/etl/` — vivacapi-etl

| Path | Type | Purpose / Scope | Potential SSOT | Dependencies | Duplicates / Conflicts | Status | Action |
|---|---|---|---|---|---|---|---|
| `etl/source1_transform.md` | Architecture | source1(GoCamping 2,897건) → spots 변환 작업 기록 | source1 변환 규칙의 SSOT | `core/erd.md` | 필드 매핑이 `console/spot-sdp-field-mapping.md`와 부분 중복(D2) | Active | **Keep** |
| `etl/decisions/source1_transform_changelog.md` | Decision | 변환 규칙 변경 이력 | 변환 규칙 이력의 SSOT | 위 문서 | 없음 | Active | **Keep** |
| `etl/reviews/source1_transform_review.md` | Status | 변환 결과 검토 보고서 | 해당 검토의 SSOT | 위 두 문서 | 없음 | Active | **Keep** |
| `etl/naver_favorites_import.md` | Architecture | 네이버 지도 즐겨찾기 → spots 임포트 기록 | 해당 임포트의 SSOT | `core/erd.md` | 없음 | Active | **Keep** |

## 1.7 `docs/mcp/` — vivac-mcp

| Path | Type | Purpose / Scope | Potential SSOT | Dependencies | Duplicates / Conflicts | Status | Action |
|---|---|---|---|---|---|---|---|
| `mcp/projects/plan.md` | Feature | MCP Connector 기획 — 자연어 스팟 검색 | MCP 제품 기획의 SSOT | `PRODUCT.md`, 탐색 API | `PRODUCT.md` MVP 범위에 없는 제3 소비자 트랙(C7 계열) | Active(기획 단계) | **Keep** |
| `mcp/projects/cli-plan.md` | Feature | CLI 기획 — `plan.md` 위의 3번째 소비자 | CLI 기획의 SSOT | 위 문서 | 위와 동일(C7 계열) | Active(열린 질문 있음) | **Keep** |

---

# 2. 정보 유형별 위치 지도

요청하신 정보 종류가 현재 어디에 흩어져 있는지입니다. **볼드**는 향후 SSOT로 지정할 곳입니다.

| 정보 | 현재 정의 위치 | 다중 정의 | 정의 상충 |
|---|---|---|---|
| 제품 정의 | **`PRODUCT.md`**, `PRODUCT_TEMP.md`(폐기), archive 3건 | 예 | 아니오(archive 헤더로 해소) |
| 제품 비전 | **`PRODUCT.md`** 표제·차별화 | 아니오 | 아니오 |
| 문제 정의 | **`PRODUCT.md`** "문제", archive 기획서 2건, `research_backpacking_market.md` | 예 | 아니오 |
| 사용자 정의 | **`PRODUCT.md`** "타겟", **`research_backpacking_market.md`**(페르소나 근거), archive 수정안(JTBD) | 예 | 아니오(상세도 차이) |
| 핵심 가치 | **`PRODUCT.md`** "차별화", `data-pipeline.md`(신뢰 표시 정책) | 부분 | 아니오 |
| MVP 범위 | **`PRODUCT.md`**, `feature-spec.md`, `business-feature-roadmap.md`, `ia.md` | 예 | **예 — C7** |
| 기능 정의 | `PRODUCT.md` "기능 범위", **`feature-spec.md`**(화면), `business-feature-roadmap.md`(백엔드), `core/projects/*` | 예 | **예 — C1** |
| 정보 구조 | **`ia.md`** | 아니오 | 아니오 |
| 화면 구조 | **`ia.md`**, `feature-spec.md`, `console/spot-sdp-field-mapping.md`(콘솔) | 예 | 아니오 |
| 라우팅 | **`ia.md`**, `feature-spec.md`, `front/projects/search.md` | 예 | 아니오(미결 3건은 `ia.md` §4) |
| 데이터 구조 | **`core/erd.md`**, `core/enums.md`, `data-pipeline.md`, `console/spot-sdp-field-mapping.md`, `etl/source1_transform.md` | 예 | 아니오(값은 일치) |
| 기술 아키텍처 | **`core/architecture.md`**, `front/reference/*`, `core/infra/lightsail-setup.md` | 아니오(repo별 분리) | 아니오 |
| API 정의 | **`docs/openapi.json`**(생성물), `core/architecture.md`, `core/projects/*` 4건, `console/*` 2건 | 예 | **예 — C2, C3** |
| 비즈니스 규칙 | **`data-pipeline.md`**(신뢰·노출), `PRODUCT.md`(신뢰 확보 방식), `core/projects/spot-invites.md`(초대 소진) | 예 | **예 — C4** |
| 사용자 흐름 | **`ia.md`** 사이트맵, `feature-spec.md` | 예 | 아니오 |
| 구현 상태 | **`STATUS.md`**, `INDEX.md` 상태 열, `feature-spec.md` §0, `business-feature-roadmap.md` 태그, 각 리뷰·백로그 | 예(4중) | 아니오(동기화 지연 위험) |
| 기술적 의사결정 | `STATUS.md` §6, `data-pipeline.md`, `core/projects/*`, `etl/decisions/`, `front/decisions/` | 예 | 아니오 — **ADR 체계 부재가 근본 문제** |
| 향후 계획 | `PRODUCT.md` 로드맵, `business-feature-roadmap.md`, `feature-spec.md` §5, `mcp/projects/*` | 예 | 부분(C7) |
| TODO | `STATUS.md`, `core/backlog.md`, `core/backlog/*`, `front/backlog/*`, 각 리뷰 문서 | 예 | 아니오 |
| 중복된 제품 정의 | `PRODUCT.md` ↔ archive 3건 ↔ `PRODUCT_TEMP.md` | 예 | 아니오(정리 완료) |

---

# 3. SSOT 후보

## 3.1 이미 존재하며 SSOT로 지정 가능한 문서

| 문서 | 담당 정보 | 지정 이유 |
|---|---|---|
| `.specify/memory/constitution.md` | 불변 원칙, 우선순위, 결정 권한의 경계 | 2026-09-07 v1.0.0 비준 완료. 다른 문서가 침범하지 않는 유일한 거버넌스 계층입니다. |
| `docs/PRODUCT.md` | 제품 정의, 문제, 타겟, 핵심 가치, MVP 범위, 수익 모델 | 이미 3개 원본을 병합한 확정본이고, 다른 문서가 모두 이 문서를 참조하는 구조입니다. |
| `docs/ia.md` | 정보 구조, 화면 인벤토리, 라우팅, 내비게이션 | 유일 정의처이며 미결 사항까지 스스로 표시하고 있습니다. |
| `docs/data-pipeline.md` | 데이터 노출 게이트·신뢰도 표시 정책 | Constitution 원칙 II가 직접 참조하는 정본이며, 정책 판단 근거가 이 문서에만 있습니다. |
| `docs/core/erd.md` | DB 스키마 | 모델 코드에서 파생되는 유일한 전체 스키마 뷰입니다. |
| `docs/core/enums.md` | Enum 값 | 값 목록의 유일한 집약처입니다. |
| `docs/core/architecture.md` | 백엔드 기술 아키텍처 | living document로 관리되고 있고, 경로 변경 시 이미 최신 기준 역할을 하고 있습니다. |
| `docs/openapi.json` | API 계약 | 코드에서 생성되므로 사람이 쓴 어떤 문서보다 정확합니다. |
| `docs/STATUS.md` | 구현 상태·결정 로그 집계 | 유일한 횡단 집계처입니다. 단 파생 문서이므로 SSOT가 아니라 **집계 뷰**로 성격을 명시해야 합니다. |

## 3.2 신설이 필요한 SSOT

| 신설 문서 | 담당 정보 | 필요한 이유 |
|---|---|---|
| `docs/meta/SSOT.md` | 어떤 정보의 정본이 어느 문서인지의 목록 | Constitution 원칙 V가 "정본 목록은 SSOT 문서가 관리한다"고 위임했으나 문서가 없습니다. |
| `docs/meta/DOCUMENTATION.md` | 폴더 분류, 파일명, 템플릿, 작성 방법 | 현재 규칙이 `.claude/rules/vivac-docs-authoring.md`·`front/INDEX.md`·`INDEX.md` 3곳에 흩어져 있고 서로 다릅니다(D6). |
| `specs/` (저장소 루트) | Feature Specification 개별 문서 | `feature-spec.md` 하나가 12개 기능을 담고 있어 Spec Kit의 기능 단위 워크플로와 맞지 않습니다. **경로는 Spec Kit `create-new-feature.sh:191`에 `$REPO_ROOT/specs`로 하드코딩되어 있어 `docs/` 하위로 옮길 수 없습니다.** 어느 저장소의 루트인지는 아래 결정 1을 따릅니다. |
| `docs/decisions/` (ADR) | 기술·제품 의사결정 | 결정이 `projects/`·`STATUS.md` §6·본문 각주에 흩어져 있고 ADR 번호 체계가 없습니다. |

## 3.3 SSOT로 지정하면 안 되는 문서

- `docs/INDEX.md` — 탐색용 인덱스입니다. 상태 열을 유지하는 한 `STATUS.md`와 영구히 어긋납니다.
- `docs/business-feature-roadmap.md` — 제안·완료·보류가 섞인 혼합 문서입니다. 분해 전에는 어떤 정보의 정본도 될 수 없습니다.
- `docs/feature-spec.md` — 현재는 사실상 기능 명세의 정본이지만, 12개 기능을 한 파일에 담고 있어 기능별 변경 이력을 추적할 수 없습니다.
- `docs/archive/**` — Constitution Precedence상 구현 근거로 쓰지 않습니다. 단 유효한 2건은 archive 밖으로 빼야 합니다(D8).

---

# 4. 중복 / 충돌

## 4.1 중복 (같은 정보를 여러 곳에서 정의)

| ID | 내용 | 관련 문서 | 위험도 |
|---|---|---|---|
| D1 | `pipeline_status` 값 목록이 3곳에 정의 | `data-pipeline.md`, `core/enums.md`, `core/erd.md`(+`console/projects/pipeline-status-review-api.md`) | 낮음 — 현재 값은 일치하나 값 추가 시 3곳 갱신 필요 |
| D2 | API 스키마·필드 목록이 설계 문서와 `openapi.json`에 이중 기재 | `core/projects/*` 4건, `console/*` 2건, `front/backlog/spot-detail-schema-request.md` | 중간 — 설계 스냅샷이 코드보다 먼저 낡습니다 |
| D3 | 구현 상태가 4중 기록 | `STATUS.md`, `INDEX.md` 상태 열, `feature-spec.md` §0, 각 문서 자체 태그 | **높음** — 수동 동기화이고 `STATUS.md`가 스스로 "자동 동기화 아님"이라고 밝혔습니다 |
| D4 | 기능 정의가 3중 | `PRODUCT.md` 기능 범위, `feature-spec.md`, `business-feature-roadmap.md` | **높음** — 관점이 달라 어느 것이 정본인지 불명확합니다 |
| D5 | 템플릿 2중 | `temp-adr/_template.md` ↔ `front/templates/adr-template.md`, `temp-specs/_template.md` ↔ `.specify/templates/spec-template.md` | 중간 — 어느 템플릿을 쓸지 결정되지 않았습니다 |
| D6 | 문서 작성 규칙이 3곳 | `.claude/rules/vivac-docs-authoring.md`, `front/INDEX.md`, `INDEX.md` §8 | **높음** — 폴더 분류 체계 자체가 서로 다릅니다(아래 C9 참고) |
| D7 | 제품 정의가 `PRODUCT.md`와 archive 3건에 중복 | `PRODUCT.md`, archive 3건 | 낮음 — archive 헤더로 이미 해소되었습니다 |
| D8 | `archive/`에 유효 문서 2건이 섞여 있음 | `VIVAC_데이터실현성_스파이크_검증설계.md`, `VIVAC_공유자제vs데이터해자_논의1페이저.md` | 중간 — "archive는 구현 근거로 쓰지 않는다"는 규칙과 충돌합니다 |

## 4.2 충돌 (서로 다른 정의를 갖고 있음)

| ID | 충돌 내용 | 문서 A | 문서 B | 판정 |
|---|---|---|---|---|
| C1 | 스팟 상세 페이지 존재 여부 | `business-feature-roadmap.md` 1.2 — "상세 페이지가 repo 역사상 한 번도 구현된 적 없음"(2026-07-20) | `feature-spec.md` §1.2 — "`/spots/{uid}` 정상 구현 확인"(2026-08-04, 실서비스 점검) | **B가 최신·실측**. A 갱신 필요. `STATUS.md`가 이미 재확인 필요로 표시했습니다 |
| C2 | `docs/openapi.json`의 git 추적 여부 | `core/architecture.md` L7 — "`docs/openapi.json`(git 미추적)" | 실제로 `docs/openapi.json`이 이 저장소에 커밋되어 있고 현재 수정 상태입니다 | **코드(실제 상태)가 현실**. A 문구 수정 필요 |
| C3 | 어드민 API 경로 | `core/projects/vivac-console-{backend,frontend}.md` — `/v1/admin/*` | `core/architecture.md`·실제 구현 — `/v1/internal/*` | **B가 정본**. 각주로 완화되어 있으나 두 문서는 Archive 처리가 적절합니다 |
| C4 | Invite 소진 정책 | `core/projects/spot-invites.md` — "1회용, 재사용 불가" | `business-feature-roadmap.md` 1.1 — 일반 리퍼럴은 재사용 가능(구현 완료) | **B가 최신**. 각주 반영됨. `group_uid` 유무로 분기되는 규칙을 본문에 반영하는 편이 안전합니다 |
| C5 | 플랫폼 전제 | archive `기획서_문제정의_수정안.md` — iOS 네이티브 우선 + PWA 배제 | `PRODUCT.md` — 웹 MVP 우선 → React Native 확장 | **B가 정본**. archive 헤더로 이미 경고되어 있습니다 |
| C6 | 프론트 API 프록시 경로 | `front/reference/frontend/api-proxy.md` — `route.ts`가 유일 프록시 | 실제 — `next.config.ts`의 `rewrites()`가 우선, `route.ts`는 미사용 | **코드가 현실**. A 갱신 필요 |
| C7 | MVP 범위 | `PRODUCT.md` MVP 정의 — 그룹·초대·리퍼럴·리뷰 미포함 | 실제 구현 — 4개 모두 `vivacapi-core`에 구현·배포됨 | **미해결**. Constitution 원칙 III·VI상 사람의 결정이 필요하며 AI가 임의로 범위를 넓힐 수 없습니다 |
| C8 | `docs/`의 실체 | `CONTEXT_SCOPE.md` — "`docs/`는 vivac-cowork의 docs로 가는 심볼릭 링크입니다" | 이 저장소(`vivac-cowork`)에서는 `docs/`가 실제 폴더입니다 | **문서가 다른 repo 관점으로 쓰였음**. 주체를 명시하도록 수정 필요 |
| C9 | 문서 간 SoT 우선순위 | `front/INDEX.md` §2 — 1)reference 2)design reference 3)**코드** 4)decisions 5)backlog 6)archive | Constitution Precedence — 1)Constitution 2)제품 정의·승인된 Spec 3)정본 참조 문서 4)CLAUDE.md 5)**코드와 관행** | **정면 충돌**. front 규칙은 "코드가 최종 사실", Constitution은 "문서가 의도이고 코드가 현실"입니다. 통일 필요 |
| C10 | 문서 내 링크 19건이 깨짐 | `INDEX.md` 8건, `STATUS.md` 2건, `front/*` 4건 — 모두 `vivac-frontend/design/` 경로 | `business-feature-roadmap.md` 5건 — <code>[2.3](상태변경 알림)</code> 형태의 잘못된 Markdown 링크 | design 링크는 저장소 밖 상대 경로여서 체크아웃 배치에 의존합니다. roadmap 5건은 단순 표기 오류입니다 |

---

# 5. Migration Plan

원칙은 **규칙 → 제품 → 명세 → 결정 → 상태 → 정리** 순서입니다. 규칙을 먼저 정하지 않으면 이후 이동이 다시 뒤집힙니다. 각 단계는 독립 커밋으로 끊을 수 있게 나눴습니다.

## Phase 0 이전 — 사람이 먼저 정해야 하는 것

아래 6건은 AI agent가 정할 수 없습니다. Constitution 원칙 VI가 제품 방향·MVP 범위·비즈니스 정책·정본 정의를 사람의 권한으로 못박고 있기 때문입니다. 상세 설명과 선택지는 `docs/meta/migration-decisions-260907.md`를 참고하세요.

| # | 결정 | 막고 있는 단계 |
|---|---|---|
| 1 | Spec Kit 워크플로를 어느 저장소에서 돌릴 것인가 | Phase 3 전체 |
| 2 | Feature Spec·ADR 템플릿을 무엇으로 단일화할 것인가 | Phase 0·3·4 |
| 3 | `docs/openapi.json`을 계속 추적할 것인가 | Phase 0, C2 |
| 4 | `docs/front/` 미러의 규칙 충돌(C9)을 어디서 고칠 것인가 | Phase 1 |
| 5 | MVP 범위에 그룹·초대·리퍼럴·리뷰를 포함할 것인가(C7) | Phase 2·3 |
| 6 | 분해 후 `feature-spec.md`를 어떻게 남길 것인가 | Phase 3 |

## Phase 0 — 기준선 정리 (사람 결정 필요, 반나절)

작업 대상이 아닌 것부터 치웁니다.

1. `docs/TEMP.md`, `docs/temp-product/README.md`(0바이트), 저장소 루트의 0바이트 파일 `"어디에서 합법적이고 안전하게 야영할 수 있는가?"` 삭제
2. `docs/openapi.json`을 계속 추적할지 결정합니다 — 추적한다면 `core/architecture.md`의 "git 미추적" 문구를 고치고(C2), 추적하지 않는다면 `.gitignore`에 넣습니다
3. `docs/temp-adr/`·`docs/temp-specs/`·`docs/temp-product/`를 처분합니다 — 결정 2의 결과에 따릅니다(D5)

**산출물**: 삭제 커밋 1개, 결정 사항 2건

## Phase 1 — 메타 계층 신설 (반나절, 다른 모든 단계의 선행 조건)

1. `docs/meta/DOCUMENTATION.md` 작성 — 폴더 분류·파일명·템플릿·작성 방법을 한 곳으로 통일합니다. 재료는 `.claude/rules/vivac-docs-authoring.md` + `front/INDEX.md` §1·§4 + `INDEX.md` §8이며, **분류 체계 충돌(D6)과 SoT 우선순위 충돌(C9)을 여기서 해소**합니다. Constitution Precedence를 상위 기준으로 삼아야 합니다
2. `docs/meta/SSOT.md` 작성 — §3의 표를 정본 목록으로 옮깁니다. Constitution 원칙 V가 이 문서를 명시적으로 요구합니다
3. `docs/CONTEXT_SCOPE.md`를 `docs/meta/DOCUMENTATION.md`로 흡수하고 C8을 수정합니다

**검증**: Constitution "Scope of This Document"의 후속 TODO 3건이 모두 해소되어야 합니다

## Phase 2 — 제품 계층 확정 (사람 결정 필요)

1. **C7(MVP 범위 vs 구현 범위)을 결정합니다.** 그룹·초대·리퍼럴·리뷰를 MVP에 포함할지, 별도 트랙으로 둘지입니다. 이 결정 없이는 Phase 3의 Feature Spec 분해 범위가 정해지지 않습니다
2. `PRODUCT.md`의 "기능 범위" 절을 `specs/` 인덱스 링크로 축약해 기능 정의 3중(D4)을 끊습니다
3. `PRODUCT_TEMP.md`를 `archive/planning-source/`로 이동합니다
4. `archive/`에서 유효 문서 2건을 꺼냅니다(D8) — 스파이크 설계는 `docs/research/`로, 공유자제 1페이저는 `docs/decisions/`의 미결 항목으로 옮깁니다
5. `research_backpacking_market.md`를 `docs/research/`로 승격합니다

**산출물**: 결정 1건, 이동 4건

## Phase 3 — Feature Specification 분해

1. `specs/` 신설(위치는 결정 1을 따름), 템플릿은 결정 2에서 정한 것 하나로 고정합니다
2. `feature-spec.md`의 12개 기능을 개별 Spec으로 분해합니다. Constitution 원칙 IV에 따라 각 Spec은 Acceptance Criteria와 Non-Goals를 반드시 포함합니다. 우선순위는 `feature-spec.md` §5를 따릅니다(🔴 4건 우선)
3. `business-feature-roadmap.md`를 분해합니다 — 완료 3건은 Phase 4의 ADR로, 제안 12건은 `docs/backlog/`로, 보류 1건(1.2)은 C1 해소 후 재판정합니다
4. `feature-spec.md`는 삭제하지 않고 "화면 현황 스냅샷"으로 성격을 좁혀 남기거나, 전량 분해 후 archive 처리합니다

**주의**: 이 단계는 Spec Kit `/speckit-specify` 워크플로와 직접 맞물립니다. Constitution 원칙 VI상 AI는 분해만 하고 요구사항의 의미를 바꿀 수 없습니다

## Phase 4 — 의사결정 기록 통합

1. `docs/decisions/` 신설, ADR 번호 체계(`ADR-001` 등)를 시작합니다
2. 기존 결정 문서를 ADR로 승격합니다 — `core/projects/spot-search-postgres-fts.md`, `core/projects/async-job-worker-design.md`, `data-pipeline.md`의 확정 정책 절, `front/decisions/incidents/*`, `core/troubleshooting/*`
3. `STATUS.md` §6 결정 로그를 ADR 인덱스로 전환합니다 — 요약 대신 링크만 남깁니다
4. `core/troubleshooting/`을 `core/decisions/incidents/`로 이름을 맞춥니다(D6)

## Phase 5 — 상태 기록 단일화

1. `INDEX.md`의 상태 열을 제거하고 `STATUS.md`로 위임합니다(D3)
2. `STATUS.md` 머리말에 "이 문서는 파생 집계 뷰이며 SSOT가 아닙니다"를 명시합니다
3. `core/reviews/known-issues.md`를 `STATUS.md` §5로 통합할지 검토합니다
4. `core/code-review-2026-07-22.md`에 항목별 열림/완료 표기를 추가합니다 — 현재 추적 수단이 없어 `STATUS.md`가 처리 여부를 확정하지 못합니다

## Phase 6 — 충돌 해소

C1~C10을 순서대로 처리합니다. Constitution Conflict Resolution의 "문서가 의도이고 코드가 현실" 기준을 적용하되, **안전·합법성 관련 충돌은 보수적인 쪽을 잠정 기준으로** 삼습니다.

| 우선 | 항목 | 처리 |
|---|---|---|
| 1 | C9 | Phase 1에서 이미 해소 |
| 2 | C7 | Phase 2에서 이미 해소 |
| 3 | C1 | `business-feature-roadmap.md` 1.2 전제를 실측 기준으로 갱신 |
| 4 | C2, C6, C8 | 문서 문구를 실제 코드에 맞춰 수정 |
| 5 | C3 | `vivac-console-{backend,frontend}.md` Archive 처리 |
| 6 | C4 | `spot-invites.md` 본문에 분기 규칙 반영 |
| 7 | C10 | design 링크 14건을 저장소 내부 참조나 명시적 안내로 대체, roadmap 5건 표기 수정 |
| 8 | C5 | 조치 불요(archive 헤더로 완화됨) |

## Phase 7 — 아카이브 정리

1. 완료된 설계 스냅샷을 `archive/`로 이동합니다 — `core/projects/spot-detail-fields.md`, `front/backlog/spot-detail-schema-request.md`, `core/projects/vivac-console-*.md`
2. `core/skill-db-inspect.md`를 `.claude/skills/`로 옮기거나 폐기합니다
3. 모든 archive 문서에 대체 문서·폐기일·사유 헤더가 있는지 확인합니다

## Phase 8 — 검증

1. 링크 체커를 돌려 깨진 링크 0건을 확인합니다
2. `docs/meta/SSOT.md`의 정본 목록과 실제 문서가 1:1로 맞는지 확인합니다
3. 같은 정보가 두 곳에서 정의되지 않는지 확인합니다 — 특히 D1·D3·D4가 해소되었는지 봅니다
4. Constitution 원칙 V의 검증 기준("정본이 아닌 문서에 정의가 재기술되어 있지 않은가")을 통과하는지 확인합니다

## 단계 간 의존 관계

```text
Phase 0 (기준선)
   ↓
Phase 1 (메타 규칙)  ← 모든 이후 단계의 선행 조건
   ↓
Phase 2 (제품·C7 결정)
   ↓
Phase 3 (Spec 분해) ──┐
   ↓                  │
Phase 4 (ADR) ────────┤
   ↓                  │
Phase 5 (상태 단일화) ─┤
   ↓                  │
Phase 6 (충돌 해소) ←──┘
   ↓
Phase 7 (아카이브) → Phase 8 (검증)
```

---

# 6. 이 조사에서 확인한 사실 요약

- 문서 69개 중 Active 52개, Outdated 11개, Duplicate 1개, Unknown 5개입니다(인벤토리 표는 63행이며, `core/backlog/*.md` 7개를 한 행으로 묶었습니다).
- 가장 시급한 구조 문제는 **문서 작성 규칙 자체가 3곳에 서로 다르게 존재하는 것**(D6)과 **SoT 우선순위가 Constitution과 정면 충돌하는 것**(C9)입니다. 이걸 먼저 풀지 않으면 이후 모든 이동이 근거 없이 진행됩니다.
- 제품 계층은 이미 `PRODUCT.md`로 잘 수렴되어 있습니다. 반면 **기능 계층은 3개 문서가 서로 다른 관점으로 같은 기능을 정의**(D4)하고 있어 분해가 필요합니다.
- 구현 상태를 4곳에서 수동 동기화하고 있어(D3) 이미 최소 1건의 실제 불일치(C1)가 발생했습니다.
- ADR 체계가 없어 의사결정이 설계 문서 본문·각주·`STATUS.md` 표에 흩어져 있습니다. Constitution 원칙 V의 "기록되지 않은 결정은 결정이 아닙니다"를 만족하려면 Phase 4가 필요합니다.
