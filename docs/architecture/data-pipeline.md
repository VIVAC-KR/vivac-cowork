# 스팟 데이터 파이프라인 — 상태/등급 필드 설계

스팟 데이터 row마다 2개 필드를 추가합니다. **필드명 확정** (2026-07-11, vivacapi-core `feature/spot-pipeline-status-trust-tier` 브랜치에 구현).

## 1. `pipeline_status` — 파이프라인 진행 상태

"이 데이터가 어느 처리 단계까지 왔는가". API 노출 여부를 결정합니다 (`PUBLISHED`만 노출).

| 값 | 단계 | 의미 |
|---|---|---|
| `RAW` | 원천 수집 | 크롤링/수집한 원본 그대로 (신규 row 기본값) |
| `ENRICHED` | 자동 보정 | 크롤링 기반 보정·병합 완료 |
| `CURATED` | 1차 수작업 | 사람이 정제·입력 완료 (= 검수 대기) |
| `REVIEWED` | 최종 검수 | 검수자 승인 완료 |
| `PUBLISHED` | 실 서비스 | production API에 노출 |
| `REJECTED` | 탈락 | 검수 반려, 중복, 폐쇄된 스팟 등 |

- 진행 순서: RAW → ENRICHED → CURATED → REVIEWED → PUBLISHED
- 검수 반려 시 `REJECTED` 또는 `CURATED`로 되돌립니다
- 이름을 `status`가 아닌 `pipeline_status`로 한 이유: 영업 상태(폐쇄/휴장 등) 필드와의 충돌 예약
- tier1/2/3 같은 숫자 표기는 쓰지 않기로 결정했습니다 — 의미가 숨고, 중간 단계 삽입이 안 되기 때문입니다
- 저장: `String` + CHECK constraint (PG native enum은 값 추가 시 마이그레이션 부담), 앱은 `PipelineStatus` StrEnum
- 공개 조회용 partial index (`WHERE pipeline_status = 'PUBLISHED'`)
- 변경 추적이 필요해지면 `status_changed_at`, `status_changed_by` 추가로 충분합니다

## 2. `trust_tier` — 데이터 신뢰도 등급

"이 데이터를 얼마나 믿을 수 있는가". pipeline_status와 별개 축입니다 — status는 노출 게이트, tier는 노출 후 신뢰도 표시와 운영팀 보강 우선순위를 결정합니다.

| 값 | 기준 |
|---|---|
| `1` | 공식 소스(고캠핑, 산림청 등) + 핵심 4대 속성(화기/접근성/편의시설/예약) 모두 채워짐 |
| `2` | 신뢰 가능한 소스지만 일부 속성 누락, 또는 커뮤니티 소스지만 교차 확인됨 |
| `3` | 단일 비공식 소스(블로그, 제보), 미검증 — 현장 확인 필요 |

- 이름에 `trust_` prefix: API 소비자가 `tier`만 보면 의미 불명 + 미래의 다른 tier(제휴/노출 랭킹 등)와 충돌 방지
- 저장: `SmallInteger` 1/2/3 + CHECK, 판정 전에는 `NULL` (범위 비교·정렬 쿼리 때문에 문자열 `"tier1"`이 아닌 정수)
- 운영 활용 예: tier 3인데 조회수 높은 스팟 = 검증 우선순위 1순위

## 확정된 정책

- **tier 3 노출**: `PUBLISHED`면 tier 3도 노출합니다. API 응답에 `trust_tier`를 내려주고 frontend가 "미검증" 뱃지를 렌더링합니다. 노지/비박 커버리지가 차별화 포인트이므로 비노출보다 표시-노출.
  - 안전 관련 속성(화기 가능 여부 등)은 tier 3일 때 frontend에서 일괄 "확인되지 않음"으로 표시합니다. 필드 단위 검증 플래그는 필요해지는 시점에 도입합니다 (현재는 과설계로 판단).
- **기존 row backfill**: 이미 서비스 노출 중이던 데이터이므로 전부 `PUBLISHED`, `trust_tier`는 `NULL`(미판정).

## 남은 것

- ETL(vivacapi-etl) bulk upsert가 기존 `PUBLISHED` row를 재업서트할 때 `pipeline_status`를 payload에 포함하면 강등될 수 있습니다 — ETL 쪽에서 status 미포함 원칙으로 갈지, core에서 강등 차단할지 추후 결정합니다
- `trust_tier` 자동 판정 로직 (소스별 기본값 + 4대 속성 완비 체크)
