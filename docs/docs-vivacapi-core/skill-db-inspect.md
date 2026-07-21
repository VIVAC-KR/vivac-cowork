# Claude Skill 초안: `db_inspect`

> 본 문서는 `.claude/skills/db_inspect/SKILL.md` 로 옮길 **초안**입니다.
> 검토/수정 후 실제 위치로 이동하면 슬래시 커맨드(`/db_inspect`)로 호출 가능합니다.

---

## 목적

데이터베이스 스키마(ORM 모델 + Alembic 마이그레이션 + 연결 설정)를 종합적으로 점검하여
**보안/위험/정합성 보고서**를 `docs/db-security-review-YYYY-MM-DD.md` 형식으로 생성합니다.

`/db_inspect`를 호출하면 매번 최신 스키마 상태를 분석하여 새 보고서를 생성합니다.

---

## 트리거 / 사용 시점

- 새 모델/컬럼 추가 PR 직전
- 분기 별 정기 점검 (월 1회 권장)
- 마이그레이션이 운영에 적용되기 전 사전 검토
- 외부 보안 감사 대비 셀프 체크

---

## 인자

- `$ARGUMENTS` (선택): 점검 범위를 좁히는 키워드. 예) `users`, `spot`, `auth`. 미지정 시 전체.

---

## SKILL.md 본문 (그대로 복사하여 사용)

```markdown
---
name: db_inspect
description: ORM 모델 + Alembic 마이그레이션 + DB 연결 설정을 종합 점검하고 docs/db-security-review-YYYY-MM-DD.md 보고서를 생성합니다.
disable-model-invocation: true
allowed-tools: Bash(ls *) Bash(cat *) Bash(grep *) Bash(rg *) Bash(date *) Bash(uv run alembic *) Read Write
---

# DB 스키마 보안/위험 점검

ORM 모델, Alembic 마이그레이션, DB 연결 설정 전반을 점검하여 위험 항목을 식별하고
`docs/db-security-review-YYYY-MM-DD.md` 보고서를 생성합니다.

## 절차

### 1단계: 점검 범위 수집 (병렬 실행)

```bash
date +%Y-%m-%d
ls app/models/
ls alembic/versions/
uv run alembic current
uv run alembic heads
```

`Read` 도구로 다음을 모두 읽어 들이세요:
- `app/models/*.py` 전체
- `alembic/versions/*.py` 전체 (혹은 head 기준 최신 N개)
- `app/core/database.py`, `app/core/config.py`

`$ARGUMENTS`가 지정된 경우 해당 키워드를 포함하는 모델/마이그레이션만 점검 대상으로 좁히세요.

### 2단계: 체크리스트 기반 점검

**아래 체크리스트의 각 항목을 모두 평가**하고, 해당하는 위험을 보고서에 기록합니다.
하나라도 건너뛰지 말고, 발견되지 않은 항목도 "발견 없음"으로 명시하세요.

#### A. 식별자/PK
- [ ] PK가 SERIAL/auto-increment인가? → 열거 가능성(enumeration), IDOR 위험
- [ ] 외부 노출용 별도 ID(UUID 등)가 있는가?
- [ ] 외래키에 인덱스가 있는가? (없으면 join/cascade delete 성능 저하)
- [ ] FK의 `ON DELETE` / `ON UPDATE` 정책이 명시적인가?

#### B. 유니크/제약
- [ ] UNIQUE 컬럼이 대소문자/공백 정규화 없이 비교되는가? (특히 `email`)
- [ ] 정규화 컬럼에 functional unique index(`lower(...)`)가 있는가?
- [ ] CHECK 제약으로 도메인이 좁혀져 있는가? (시간 정합성, 값 범위, enum)
- [ ] NOT NULL이어야 할 컬럼이 nullable로 잘못 선언돼 있지 않은가?
- [ ] partial unique index가 필요한 soft-delete/active 컬럼이 있는가?

#### C. PII/민감정보
- [ ] PII (email/이름/사진/전화/주소) 평문 저장 여부와 보존 정책
- [ ] 인증정보/토큰/비밀키가 DB 컬럼에 저장돼 있는가?
- [ ] 감사 로그(audit log) 또는 변경 이력 테이블 존재 여부
- [ ] 민감 컬럼에 대한 마스킹/암호화(at-rest 외) 필요성

#### D. URL/문자열 입력
- [ ] URL 컬럼의 길이뿐 아니라 스킴/도메인 검증이 있는가?
- [ ] 사용자 입력 문자열의 최대 길이가 합리적인가? (DoS 방지)
- [ ] 정규식 CHECK 제약이나 애플리케이션 검증이 있는가?

#### E. 시간/타임스탬프
- [ ] 모든 datetime 컬럼이 `TIMESTAMPTZ` (timezone=True)인가?
- [ ] `created_at`, `updated_at` 자동 갱신 동작이 있는가?
- [ ] `*_at` 컬럼이 `created_at`보다 과거가 될 수 있는가? (CHECK 필요)

#### F. 인증/IdP
- [ ] OAuth subject 컬럼이 nullable=False인 채 단일 IdP에 락인돼 있는가?
- [ ] 다중 IdP 확장 시 `auth_identities` 분리 가능성

#### G. 마이그레이션 안전성
- [ ] `# please adjust!` 등 autogenerate 미검토 흔적이 남아있는가?
- [ ] downgrade가 운영에서 위험한 destructive 동작을 포함하는가?
- [ ] NOT NULL 컬럼 추가 시 server_default나 백필 단계가 누락됐는가?
- [ ] 인덱스를 운영에서 추가할 때 `CONCURRENTLY` 옵션이 필요한가? (대용량 테이블)

#### H. 연결/풀/로깅
- [ ] `echo` 활성 조건이 운영에서 켜질 위험이 있는가? (SQL/PII 로그 노출)
- [ ] `pool_size`, `max_overflow`가 운영 인스턴스 수 × DB max_connections와 정합한가?
- [ ] `expire_on_commit=False`로 인한 stale 데이터 위험
- [ ] DSN/비밀번호가 로그/예외 메시지에 노출되는 경로가 있는가?
- [ ] `pydantic-settings`의 `SecretStr` 적용 여부

#### I. 인덱스/성능
- [ ] 자주 조회되는 컬럼/조합에 인덱스가 있는가?
- [ ] 사용되지 않을 가능성이 있는 인덱스(중복/광범위)가 있는가?
- [ ] full-text/JSON 컬럼에 적절한 인덱스 타입(GIN 등)이 있는가?

### 3단계: 보고서 생성

`Write` 도구로 `docs/db-security-review-{오늘날짜}.md` 파일을 생성합니다.
이미 같은 날 보고서가 있으면 파일명에 `-2`, `-3` 등을 붙여 중복을 피하세요.

보고서는 **반드시 아래 섹션을 모두 포함**합니다:

1. **메타정보** — 작성일, 대상 브랜치, 점검 범위, 현재 alembic head
2. **점검 대상 요약** — 테이블별 컬럼 표, 연결 설정 요약
3. **식별된 위험 항목** — 항목별로:
   - 심각도(🔴 High / 🟠 Medium / 🟡 Low / ⚪ Info)
   - 식별자(`H-1`, `M-1`, …)
   - **문제** / **영향** / **완화** 3섹션
4. **우선순위 권장 액션** — 표 (우선순위 / 항목 / 작업량)
5. **본 보고서가 다루지 않은 것** — 점검 범위 외의 명시적 한계

### 4단계: 결과 안내

생성된 보고서 경로와 식별된 High/Medium 항목 수를 한 줄로 사용자에게 알립니다.

## 주의사항

- **DB에 직접 연결하지 않습니다.** 정적 분석(파일 읽기)만으로 보고서를 작성합니다.
  실제 DB 메타데이터(`pg_indexes` 등)가 필요한 항목은 "운영 점검 필요"로만 표시.
- **추측을 사실로 적지 않습니다.** 직접 본 코드 근거 없이 위험을 단정하지 마세요.
  근거가 약한 항목은 ⚪ Info 또는 "검증 필요"로 표시.
- **수정을 시도하지 않습니다.** 보고서 생성만 수행하고, 실제 마이그레이션이나 코드 수정은
  사용자의 별도 지시를 받은 후 진행합니다.
- 한글 보고서 톤을 유지하고, 컬럼 이름/파일 경로는 `code` 인용으로 명확히 표기합니다.
```

---

## 적용 방법

1. 이 문서의 ` ```markdown ... ``` ` 블록 내용을 복사
2. `.claude/skills/db_inspect/SKILL.md` 파일로 저장 (디렉토리 새로 생성 필요)
3. Claude Code를 재시작하거나 슬래시 커맨드 목록을 갱신
4. `/db_inspect` 또는 `/db_inspect users` (특정 키워드)로 호출

---

## 향후 개선 아이디어

- 단계 1에 `pg_stat_*` 등 운영 DB 메타 조회 옵션 추가 (DSN을 read-only 사용자로 한정).
- 각 위험 항목에 대한 자동 수정 PR을 생성하는 별도 스킬(`/db_fix`)과 연계.
- 이전 보고서와의 diff 출력 (이미 해소된 항목 / 새로 등장한 항목).
- CI에서 `/db_inspect`를 호출해 새 마이그레이션 머지 전 PR 코멘트로 보고서 첨부.
