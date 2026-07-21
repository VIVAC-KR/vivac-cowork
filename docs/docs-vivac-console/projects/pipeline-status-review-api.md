# 데이터 검증(pipeline_status) 화면 — 백엔드(vivacapi-core) 요청 명세

> **구현 완료 (core 브랜치 `feature/pipeline-status-transition-validation`, 미push).**
> 아래는 원 요청 스펙이며, 실제 반영 상태는 문서 하단 "구현 결과" 참고.

콘솔에 staff용 "raw 데이터 검증" 화면을 만들었다. `pipeline_status=ENRICHED`인 spot을
staff가 직접 구글 검색 등으로 대조하며 필드를 수정하고, 제출하면 `CURATED`로,
반려하면 `REJECTED`로 전이시킨다. 이 화면이 돌아가려면 core 쪽 API 3곳에 변경이 필요하다.

콘솔 쪽 구현은 완료됨 (`src/components/admin/spot-edit-form.tsx`,
`src/app/(admin)/spots/page.tsx`). 아래는 core가 맞춰줘야 하는 계약.

> ⚠️ 경로 표기 정정: 아래 예시는 `/v1/admin/internal/spots`로 적었으나 오타다.
> 콘솔이 실제로 호출하는 경로는 `/v1/internal/spots` (`admin` 세그먼트 없음) —
> 실제 배포된 core 라우터와 일치함. 콘솔 코드(`src/lib/api.ts`, `src/app/(admin)/spots/**`)
> 전부 `/internal/spots...`만 쓰고 있어 404 이슈 없음, 확인 완료.

## 전제

- `pipeline_status` enum은 이미 존재: `RAW`, `ENRICHED`, `CURATED`, `REVIEWED`, `PUBLISHED`, `REJECTED`
- 인증/권한: 기존과 동일. `Authorization: Bearer <JWT>`, `is_staff` 체크만. **신규 롤 불필요** (세부 권한 도입 안 하기로 결정)
- 이 화면은 **pull 방식** — 특정 staff에게 항목을 배정하는 기능 없음. `assigned_to` 같은 필드 불필요

## 1. `GET /v1/admin/internal/spots` — 필드 노출 + 필터 추가

- 응답 각 아이템에 `pipeline_status` 포함 (지금은 빠져 있음)
- 쿼리 파라미터 `pipeline_status`를 `_FILTERABLE` 화이트리스트에 추가 (기존 `region_province`, `source`와 동일한 방식)

```
GET /v1/admin/internal/spots?pipeline_status=ENRICHED&_start=0&_end=25
```

## 2. `GET /v1/admin/internal/spots/distinct/pipeline_status` — 기존 distinct 엔드포인트 재사용

`region_province`, `source`와 동일한 기존 패턴. `pipeline_status`만 화이트리스트에 추가하면 됨. 신규 엔드포인트 아님.

```json
["RAW", "ENRICHED", "CURATED", "REVIEWED", "PUBLISHED", "REJECTED"]
```

## 3. `PATCH /v1/admin/internal/spots/{uid}` — `pipeline_status` 필드 수신 + 전이 검증

콘솔은 제출/반려 버튼을 누르면 수정된 필드들과 함께 `pipeline_status`를 payload에 실어 보낸다.

```json
PATCH /v1/admin/internal/spots/{uid}
{
  "title": "...",
  "address": "...",
  ...,
  "pipeline_status": "CURATED"
}
```

**서버에서 반드시 검증할 것**: 이 엔드포인트로 들어오는 `pipeline_status` 전이는
`ENRICHED → CURATED` 또는 `ENRICHED → REJECTED` **이 두 가지만 허용**. 그 외 값(예: 현재
상태가 `RAW`인데 `PUBLISHED`로 점프, 혹은 `CURATED`에서 다시 `ENRICHED`로 되돌리기 등)은
거부. 이 화면은 검증 큐 하나만 처리하는 용도라 다른 단계 전이는 별도 워크플로우 몫.

전이 거부 시 4xx + 에러 메시지 텍스트만 반환하면 됨. 콘솔은 응답 바디를 그대로
alert 박스에 띄운다 (`spot-edit-form.tsx`의 `error` 상태 — 별도 에러 코드 파싱 안 함,
사람이 읽을 메시지면 충분).

## 4. `/v1/admin/internal/spots/{uid}/history` — 확인만 하면 됨

기존 audit 트리거가 컬럼 단위로 diff를 잡는 방식이면 `pipeline_status` 변경도 자동으로
`changes.pipeline_status: { before, after }`로 잡힐 것으로 예상. 별도 로직 추가 없이
그런지만 확인 부탁. (`docs/audit-history-api.md` 참고 — `updated_at`처럼 의도적으로
diff에서 제외해야 하는 컬럼은 아님, 오히려 이 화면의 핵심 감사 로그라 반드시 잡혀야 함)

## 스코프 아닌 것

- staff 세부 권한/롤 — 안 함, 기존 isStaff 그대로
- 특정 staff에게 항목 배정(assign) — 안 함, pull 방식
- `ENRICHED` 이후 단계(`REVIEWED`, `PUBLISHED`) 전이 — 이 화면 스코프 아님, 필요시 별도 요청

## 구현 결과 (core 브랜치 `feature/pipeline-status-transition-validation`)

| 항목 | 상태 |
|---|---|
| 1. 목록 필터 + `pipeline_status` 노출 | 기존 v0.6.0에 이미 있었음. 회귀 테스트만 추가 |
| 2. distinct 화이트리스트 | 기존에 이미 있었음. 회귀 테스트만 추가 |
| 3. PATCH 전이 검증 | 신규. `crud/spot.py`에 `ALLOWED_PIPELINE_TRANSITIONS` 화이트리스트, endpoint에서 현재 상태 조회 후 검증. 위반 시 `422` + `{"detail": "허용되지 않는 상태 전이입니다: ENRICHED -> PUBLISHED"}` |
| 4. audit history | 기존 로직으로 충족 (row snapshot 트리거, `updated_at` 외 전체 diff 대상이라 `pipeline_status`도 자동 포함). 확인용 테스트만 추가 |

`make test` 154개 전체 통과(신규 8개), `ruff check` 통과. ETL bulk upsert 경로는 이 PATCH를 안 거쳐서 영향 없음.

**콘솔 쪽 반영한 것**: `saveSpot`(`spots/[uid]/edit/page.tsx`)이 422 에러 바디를 `{"detail": "..."}` JSON으로 파싱해서
`detail` 문자열만 alert에 표시하도록 수정 — 전엔 raw JSON 텍스트가 그대로 노출됐음.

**미push 상태이므로 아직 프론트에서 실제 검증은 못 함.** core가 push하면 이 브랜치 기준으로 로컬 통합 테스트 진행.
