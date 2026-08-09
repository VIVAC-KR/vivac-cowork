# VIVAC 진행 상황 & 결정 로그

문서 곳곳에 흩어진 "지금 뭐가 열려있고, 뭐가 급하고, 뭐가 결정됐는가"를 한 곳에 모았습니다. 전체 문서 목록은 [INDEX.md](INDEX.md) 참고. 이 문서는 각 원본 문서의 상태를 요약만 할 뿐 원본을 대체하지 않습니다 — 상세는 항상 링크를 따라가세요.

> 이 문서는 2026-08-04 기준 스냅샷입니다(§7만 2026-08-10 기준). 개별 리뷰 문서의 `[열림]`/`[완료]` 태그가 바뀌면 이 요약도 함께 갱신하세요(자동 동기화 아님).

---

## 🔴 지금 가장 급한 것

1. **[Critical] `ADMIN_SESSION_SECRET`이 git history에 실제 값으로 커밋됨** — 커밋 `746ecfc`, `main`에서 여전히 reachable. 이 키를 아는 사람은 `/admin` 세션 쿠키를 위조해 무인증으로 SQLAdmin에 진입, 아무 유저나 staff로 승격 가능. **rotate 여부 확인 필요** — 실제 배포에 재사용됐는지부터 확인 권장. → [core/code-review-2026-07-22.md](core/code-review-2026-07-22.md#critical)
2. **[Critical] vivacapi-etl — prod DB 비밀번호가 소스코드에 하드코딩되어 git history에 평문으로 존재** — 커밋 `3914727`, 아직 커밋 안 된 `scripts/upload_source1.py`에도 동일 값 중복. **즉시 rotate 필요.** 원문에는 실제 값이 있었으나 이 요약 문서 체인에서는 redact 처리함 — 실제 값 확인·rotate는 vivacapi-etl repo에서 직접. → [etl/reviews/code-review-2026-07-22.md](etl/reviews/code-review-2026-07-22.md)
3. **[BLOCKER] front — NextAuth 런타임 secret 컨테이너 주입 누락** 외 Tier 1(배포 즉시 장애/보안) 3건, 전부 아직 열림. → [front/backlog/codebase-review-260714.md](front/backlog/codebase-review-260714.md) 상세는 아래 §2 참고.
4. **팀 결정 대기 2건** (아래 §4) — 데이터 실현성 스파이크 착수 여부, 공유자제↔데이터해자 방향.

---

## 1. 코드 리뷰 오픈 이슈 현황 (repo별)

| repo | 리뷰 문서 | 열림 | 완료 | 진행중 | 보류 | 비고 |
|---|---|---|---|---|---|---|
| front | [front/backlog/codebase-review-260714.md](front/backlog/codebase-review-260714.md) | **14** | 1 | 1 | 1 | 🔴 Tier1(배포 즉시) 4건 전부 열림 — 아래 §2 |
| console | [console/reviews/codebase-review-260714.md](console/reviews/codebase-review-260714.md) | 1 | 20 | 2 | 1 | 대부분 정리됨 |
| core | [core/code-review-2026-07-22.md](core/code-review-2026-07-22.md) | Critical 1 / High 1 / Medium 3 / Low 4 | — | — | — | 완료 표기 트래킹이 없는 문서 — 처리 여부 확인 필요 |
| core | [core/backlog.md](core/backlog.md) | 5건 | — | — | — | 우선순위 미정 (이미지 인프라, DB 백업 이중화, audit_log 보관정책, rate limiting, 수정이력 화면 고도화) |
| core | [core/backlog/*.md](core/backlog/) | 6건 | — | — | — | 2026-07-11 점검 — 보안 4 / 성능 2, 심각도 대부분 낮음~중간 |
| mcp | [mcp/reviews/code-review-2026-07-22.md](mcp/reviews/code-review-2026-07-22.md) | 다수 | — | — | — | `client.py` 테스트 부재, 예외처리·로깅 전무 — 완료 표기 트래킹 없음 |
| etl | [etl/reviews/code-review-2026-07-22.md](etl/reviews/code-review-2026-07-22.md) | 다수 | — | — | — | 🔴 Critical 1(위 §0 참고) 포함 — 완료 표기 트래킹 없음 |

## 2. front Tier 1 — 배포 즉시 장애/보안 (전부 열림)

[front/backlog/codebase-review-260714.md](front/backlog/codebase-review-260714.md) 원문 기준:

1. NextAuth 런타임 secret 컨테이너 주입 누락 (BLOCKER)
2. access token 브라우저 노출 (XSS → 토큰 탈취)
3. CloudFront 인증 응답 캐시 포이즈닝 (개연성 높음, repo만으론 미검증)
4. nginx 공유 소유권 → 배포마다 타 서비스 순단

## 3. business-feature-roadmap.md 진행 상황 (16개 항목)

| 상태 | 건수 | 항목 |
|---|---|---|
| ✅ 완료 | 3 | 1.1 재사용 리퍼럴 링크, 4.2 trust_tier 신선도 감쇠(PR #117), 4.4 검증 담당자 재할당(PR #118) |
| ⏸ 보류 | 1 | 1.2 스팟 상세 공개 공유 카드(OG 메타) — ⚠️ **재확인 필요(2026-08-04)**: "상세페이지 자체가 없다"(2026-07-20 기준)는 전제가 더 이상 맞지 않음. `/spots/{uid}`가 실제로 구현·배포돼 있음을 실서비스에서 확인([feature-spec.md §1.2](feature-spec.md#12-스팟-상세-spotsuid)). OG 메타만 다시 검토하면 될 수 있음 |
| 제안(미착수) | 12 | 1.3, 1.4, 2.1~2.4, 3.1~3.4, 4.1, 4.3 |

의존성 없이 바로 착수 가능한 항목(난이도 하): `1.1`✅, `1.3(a)`, `2.2`, `4.2`✅, `4.3`, `4.4`✅ — 미완료는 `1.3(a)`, `2.2`, `4.3` 3건.

## 4. 팀 결정이 필요한 것 (제품/기획)

[PRODUCT.md](PRODUCT.md) "미해결 이슈" 원문 기준:

| 이슈 | 필요한 결정 | 근거 문서 |
|---|---|---|
| 데이터 실현성 검증 미실행 | 스파이크 착수 여부·시점 | [archive/planning-source/VIVAC_데이터실현성_스파이크_검증설계.md](archive/planning-source/VIVAC_데이터실현성_스파이크_검증설계.md) |
| 공유 자제 ↔ 데이터 해자 | 해자를 UGC로 둘지 공급자 주도 데이터 품질로 둘지 | [archive/planning-source/VIVAC_공유자제vs데이터해자_논의1페이저.md](archive/planning-source/VIVAC_공유자제vs데이터해자_논의1페이저.md) |
| 구현 범위가 PRODUCT.md MVP 정의보다 넓음 (그룹·초대·리퍼럴·리뷰 이미 구현됨) | MVP 정의를 넓혀 반영할지, 별도 로드맵 트랙으로 유지할지 | [business-feature-roadmap.md](business-feature-roadmap.md) |

## 5. 문서 자체의 알려진 결함 (내용은 맞지만 참조·구조가 낡음)

각 문서가 스스로 표시해둔 것들을 모았습니다 — 코드를 고치는 문제가 아니라 문서만 갱신하면 되는 항목입니다.

| 문서 | 문제 | 조치 |
|---|---|---|
| [front/reference/frontend/api-proxy.md](front/reference/frontend/api-proxy.md) | `route.ts`를 유일한 프록시로 설명하나 실제로는 `next.config.ts`의 `rewrites()`가 우선 적용 | 갱신 필요 |
| [core/projects/vivac-console-backend.md](core/projects/vivac-console-backend.md), [vivac-console-frontend.md](core/projects/vivac-console-frontend.md) | 경로가 `/v1/admin/*`로 돼 있으나 실제는 `/v1/internal/*` | [core/architecture.md](core/architecture.md)가 최신 기준임을 이미 각주로 명시함 — 추가 조치 불요 |
| [core/projects/spot-invites.md](core/projects/spot-invites.md) | "초대 1회용" 결정이 business-feature-roadmap 1.1에서 일반 리퍼럴에 한해 뒤집힘 | 이미 각주 반영됨 — 추가 조치 불요 |
| [design/reference/pages/spot-detail.md](../../vivac-frontend/design/reference/pages/spot-detail.md) | 후속 이슈 문서 링크가 실제 파일명·경로와 다름 | ✅ 2026-08-04 수정 완료(vivac-frontend 별도 커밋) |
| [front/archive/auth-implementation.md](front/archive/auth-implementation.md) | 대체 문서가 "없음"으로 표시 — NextAuth v5 전환 후 신규 Reference 문서 미작성 | Reference 문서 신규 작성 필요 |
| [core/skill-db-inspect.md](core/skill-db-inspect.md) | `.claude/skills/db_inspect/SKILL.md`로 옮겨야 할 초안인데 아직 이 자리에 있음 | 이동 또는 초안임을 계속 유지할지 결정 필요 |
| [feature-spec.md](feature-spec.md) | PRODUCT.md의 "기능 범위"·MVP 4순위·차별점 문구·수익 모델을 인용하는데, 2026-08-10 개정으로 해당 서술이 바뀌거나 다른 절로 이동함 | 새 [PRODUCT.md](PRODUCT.md) 기준으로 재정렬 필요 |
| [ia.md](ia.md) | PRODUCT.md "플랫폼" 절을 참조 — 새 문서에서는 §3.4 | 참조 위치 갱신 필요 |
| [STATUS.md](STATUS.md) (이 문서) | §4·§5가 구 PRODUCT.md "미해결 이슈" 원문을 기준으로 작성됨 | 새 [PRODUCT.md](PRODUCT.md) §7.4(팀 결정 대기)와 대조 필요 |
| [core/projects/business/](core/projects/business/README.md) | `vivacapi-core`의 낡은 `docs/core/projects/PRODUCT.md` 사본에서 파생되어 폐기된 기획(감성 큐레이션·2030 감성 캠퍼·오프라인 MVP)을 되살렸음. 2026-08-10 정리 | **사본을 제거해야 재발이 막힘** — `vivacapi-core` 측 조치 필요 |

## 6. 핵심 결정 로그 (연대순)

| 날짜 | 결정 | 문서 |
|---|---|---|
| 2026-05-02 | DB 스키마 보안 점검 기준선 수립 | [core/security/db-security-review-2026-05-02.md](core/security/db-security-review-2026-05-02.md) |
| 2026-05-17 | 비동기 Job 워커: 외부 브로커 없이 FastAPI 내장 폴링 워커로 | [core/projects/async-job-worker-design.md](core/projects/async-job-worker-design.md) |
| 2026-05-19 | 탐색 API 계약을 OpenAPI로 확정(VVC-105) | [core/projects/vvc-105-explore-api-spec.md](core/projects/vvc-105-explore-api-spec.md) |
| 2026-06-07 | console을 별도 repo로 분리, `/v1/admin/*`(이후 `/v1/internal/*`로 변경) | [core/projects/vivac-console-backend.md](core/projects/vivac-console-backend.md) |
| 2026-07-11 | `pipeline_status`/`trust_tier` 필드명 확정 | [data-pipeline.md](data-pipeline.md) |
| 2026-07-15 | 검색은 Elasticsearch 대신 PostgreSQL FTS+trigram | [core/projects/spot-search-postgres-fts.md](core/projects/spot-search-postgres-fts.md) |
| 2026-07-15 | 상세페이지 UI 스펙 확정(v2, DESIGN.md 우선 정책) | [design/decisions/spot-detail-design-decisions.md](../../vivac-frontend/design/decisions/spot-detail-design-decisions.md) |
| 2026-07-16 | 초대/리퍼럴을 단일 `Invite` 엔티티로 통합 | [core/projects/spot-invites.md](core/projects/spot-invites.md) |
| 2026-07-20 | 재사용 리퍼럴 링크·trust_tier 감쇠·검증 담당자 재할당 구현 완료 | [business-feature-roadmap.md](business-feature-roadmap.md) |
| 2026-07-21 | MCP Connector·CLI 기획 착수 | [mcp/projects/plan.md](mcp/projects/plan.md), [mcp/projects/cli-plan.md](mcp/projects/cli-plan.md) |
| 2026-07-28 | 검색 라우팅 골격만 우선 구축(질의·필터는 후속) | [front/projects/search.md](front/projects/search.md) |
| 2026-08-03 | nginx stale upstream으로 89분 장애 | [core/troubleshooting/2026-08-03-nginx-stale-upstream-502.md](core/troubleshooting/2026-08-03-nginx-stale-upstream-502.md) |
| 2026-08-04 | 기획 문서 통합 — PRODUCT.md 확정, 원본은 archive로 이동 | [PRODUCT.md](PRODUCT.md), [archive/planning-source/](archive/planning-source/) |
| 2026-08-04 | 실서비스(vivac.app) 점검으로 화면·API 구현 현황 확정, 상세 화면 기획 4갈래로 통합 | [feature-spec.md](feature-spec.md) |
| 2026-08-06 | vivacapi-core 코드 전수 조사로 비즈니스/개발 관점 문서 세트(18건) 신규 작성 | [core/projects/business/](core/projects/business/README.md), [core/projects/devel/](core/projects/devel/README.md) |
| 2026-08-06 | 7개 repo에 docs 심볼릭 링크 구조 확대 적용, 각 repo 로컬에 남아있던 미이전 문서(console/front/mcp/etl) 5건을 vivac-cowork로 병합 | [SYMLINK-SETUP.md](../SYMLINK-SETUP.md) |

## 7. MVP 구현 현황 (2026-08-10)

기획·계약·수용 기준은 [PRODUCT.md](PRODUCT.md)를 따른다. 이 절은 달성 여부만 추적한다.

### 7.1 플랫폼·기능

| 항목 | 상태 |
|---|---|
| 웹 — 검색·목록·필터·상세 | 구현 완료 |
| 웹 — 지도 탐색 | 미구현 (SDK 미통합) |
| **iOS 앱** | **미착수** (`apps/mobile` 없음) |
| 공개 스팟 | 191건 |
| 좌표 보유 | 54건 (28.3%) |
| 노지 스팟 | 0건 |

*데이터는 2026-08-07 전수 기준*

### 7.2 데이터 채움률 — 현재 수치

목표치는 [PRODUCT.md](PRODUCT.md) §4.5.

**전수 집계 — 191건 (2026-08-07)**

| 항목 | 현재 |
|---|---|
| 좌표 | 28.3% |
| 지역 | 28.8% |
| 카테고리 | 3.7% (전부 자연휴양림) |
| 썸네일 | 0% |
| 신뢰도 등급 | 100% (전부 2등급) |

**상세 표본 집계 — 30건**

| 항목 | 현재 |
|---|---|
| 주소 | 27% |
| 연락처 | 23% |
| 예약·안내 링크 | 17% |
| 화기 유형 | 10% |
| 테마 | 13% |
| 설명 | 10% |

### 7.3 유형 분포와 편중 판단

**유형 분포 (제목 기준)** — 자연휴양림 169 / 야영장·캠핑장 12 / 기타 10

편중은 적재 순서에 따른 중간 상태다. 모집단 추정(휴양림 약 170, 국립공원 야영장 30~40,
지자체 야영장 수백)에 비추어 최종 400~600건대에서 휴양림 비중은 30~40%로 희석된다.
**미검증 추정.**

### 7.4 미달 항목 종합

| 항목 | 위치 | 성격 |
|---|---|---|
| **iOS 앱 전체 미착수** | §3.4 | 1단계 **완료 조건** (§3.2) |
| 지도 SDK 미통합 | 5.4 | 1단계 **완료 조건** (§3.2) |
| 필터 옵션 부족 | 5.3 | 데이터 마일스톤 (§2.2) |
| 상세 필드 공백 | 5.5 | 데이터 마일스톤 (§2.2) |

웹은 데이터 확보와 SDK 통합이 남았고, iOS는 구축부터 배포까지 전부 남았다.

### 7.5 기능별 현재 상태·미달

현재 상태는 별도 표기가 없으면 웹 기준이며, iOS는 전 기능 미착수다.

| 기능 | 현재 상태 | 미달 |
|---|---|---|
| §5.1 홈 | 구현 완료 | 그룹 구성이 운영 화면 없이 고정 식별자에 묶여 있다. 썸네일 채움률 0%(§7.2)라 카드가 비어 보인다 |
| §5.2 검색·목록 | 구현 완료 | — |
| §5.3 필터 | 구현 완료. 단 **노출 옵션이 실데이터에 미달**한다 | 카테고리 — 자연휴양림 1개만 노출 / 지역 — 4개 (임시값) |
| §5.4 지도 탐색 | **미구현.** 2모드 UI·상태 관리·시트·API 클라이언트는 완료, 지도 SDK 미통합으로 지도 영역은 플레이스홀더 | — |
| §5.5 스팟 상세 | 구현 완료. 리뷰 영역은 플레이스홀더(미구현) | 표시 항목 대부분이 비어 있다(§7.2). 화면은 완성됐고 데이터가 따라오지 않은 상태다. 출처는 스키마에 필드가 없어 미구현이다 |
| §5.6 계정·인증 | 구현 완료 | — |
| §5.7 공통 상태 | 구현 완료 | — |

### 7.6 미착수

| 항목 | 위치 |
|---|---|
| 사용자 행동 계측 도입 | MVP 달성 직후 (§3.2 2단계) |
| 데이터 실현성 스파이크 | 설계는 완료, 미실행 |
| 그룹(컬렉션) 기능 | API는 구현됨. 운영 화면 없이 홈 캐러셀이 고정 식별자에 묶여 있음 (§5.1) |
| 홍보 랜딩페이지 | 라우트 없음. `/`는 서비스 홈이지 홍보 랜딩이 아님 |
| GA 등 분석 계측 | 전무. CBT 수요 측정용 계측도 없음 |
| CBT 모집 수단 | 미정. 랜딩 대신 TestFlight로 대체 가능 |
| 카카오 로그인 | MVP 직후 순차 도입 (§5.6) |

**UGC — MVP 범위 제외, 추후 개발**

| 항목 | 현재 |
|---|---|
| 리뷰·평점 | 상세 화면 플레이스홀더 (§5.5). API는 구현됨 |
| 스팟 제보 ("새 스팟" / "정보가 틀렸어요") | 미구현 |
| 사진·GPS 첨부 폼 | 미구현 |

---

## 갱신 방법

- 리뷰 문서의 `[열림]`을 `[완료]`로 바꿀 때, §1·§2 표도 같이 갱신하세요.
- business-feature-roadmap.md 항목 상태가 바뀌면 §3도 같이 갱신하세요.
- 새로운 팀 결정이 나면 §4에서 지우고 §6에 한 줄 추가하세요.
