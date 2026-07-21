# 수정 이력(Audit History) API — 프론트 연동 가이드

`spots` / `spot_business_info` 두 테이블의 변경 이력을 화면에 표시하기 위한 API 명세.
백엔드는 DB 트리거로 모든 변경(INSERT/UPDATE/DELETE)을 `audit_log`에 기록하고,
아래 엔드포인트로 조회한다.

## 엔드포인트

둘 다 **staff 인증 필요** — `Authorization: Bearer <JWT>`.

```
GET /v1/internal/spots/{spot_uid}/history
GET /v1/internal/spot-business-info/{business_info_uid}/history
```

> ⚠️ business-info는 `spot_uid`가 아니라 **business_info 레코드 자체의 uid**로 조회한다.
> 먼저 해당 레코드의 uid를 확보해야 한다.

## 응답 형태

최신순(`changed_at` desc), 최대 100건.

```json
[
  {
    "changed_at": "2026-07-01T12:55:00+00:00",
    "action": "UPDATE",
    "changed_by": "abc123...",
    "changed_by_name": "홍길동",
    "changes": {
      "review_count": { "before": 8, "after": 9 },
      "rating_avg":   { "before": 3.5, "after": 4.9 }
    }
  }
]
```

### 필드 의미 / 화면 처리

| 필드 | 설명 / 화면 |
|---|---|
| `changed_at` | 변경 시각. 타임존 포함 ISO → 로컬시간 변환 |
| `action` | `INSERT`=생성 / `UPDATE`=수정 / `DELETE`=삭제 |
| `changed_by` | 변경한 유저 uid (원시값) |
| `changed_by_name` | 표시용 이름. **`null`이면 "시스템/일괄작업"으로 표기** |
| `changes` | **바뀐 필드만** 담김. `{ 필드명: { before, after } }` |

## 반드시 처리해야 할 엣지 케이스

- `action=INSERT` → `before`가 전부 `null` (신규 생성)
- `action=DELETE` → `after`가 전부 `null`
- `changes`가 빈 객체 `{}` 가능 → 의미 있는 변경 없음. "변경사항 없음" 또는 행 숨김
- `changed_by_name`이 `null` → 워커/시스템 변경. 이름 대신 "시스템" 라벨
- 값 타입 다양 → 배열 필드(`themes`, `amenities` 등)는 배열, 숫자/불리언/문자열 그대로. 타입별 렌더 필요
- `updated_at`은 diff에서 **의도적으로 제외**됨 (매번 바뀌어 노이즈)

## 프론트가 갖춰야 할 것

- **필드명 → 한글 라벨 매핑** — `changes`의 키는 DB 컬럼명(영문 snake_case).
  예: `review_count`→"리뷰 수", `rating_avg`→"평점". 프론트에 매핑 테이블 필요.
- **페이지네이션 없음** — 현재 최신 100건 고정.

## 아직 없는 것 (필요 시 백엔드에 요청)

- 필드 라벨 API (영문 컬럼명 → 한글 라벨을 백엔드가 내려줌)
- 페이지네이션 (100건 초과 이력 조회)
