# Linear 프로젝트 등록 형식

Linear에 project를 만들거나 description을 채울 때(`mcp__claude_ai_Linear__save_project` 등) 본문은 아래 섹션 구성을 따름. 이모지·말투는 [linear-issue-format](./linear-issue-format.md)과 맞추되, 섹션 구성 자체는 프로젝트 스코프(여러 이슈를 묶는 상위 단위)에 맞게 다르게 구성함.

## 본문 템플릿

```markdown
## 🎯 Goal

## 🗺️ Background

## 📌 Scope
### In Scope
* 

### Out of Scope
* 

## 🔗 Related Issues
> 

## Notes
* 
```

- 섹션 순서·이모지·제목 문구 그대로 유지. 내용 없는 섹션도 헤더는 남김(빈 채로 두거나 "해당 없음")
- 기술 용어(함수명, 엔드포인트, 필드명 등)는 영문 그대로, 코드 인용은 백틱으로 표기
- 서비스명은 항상 `vivac`/`VIVAC`, 한글 표기(비바크, 비박 등) 금지

## 섹션별 작성 기준

- `Goal`: 프로젝트에 묶인 이슈들을 관통하는 상위 목표 1~2문장. 개별 이슈의 Goal을 나열하지 않음
- `Background`: 이슈 포맷에는 없는 섹션. 왜 지금 이 프로젝트가 필요한지 — 계기, 배경 상황, 관련된 다른 프로젝트/이슈와의 연결. "무엇을 할지"가 아니라 "왜 시작됐는지"를 씀
- `Scope`: 프로젝트 범위. In Scope 항목마다 관련 이슈 링크를 `([XXX-123](URL))` 형태로 함께 표기
- `Related Issues`: 프로젝트에 속한 이슈를 `> [XXX-123](URL) — 한 줄 설명 (담당 영역, 상태)` 형태로 나열. 완료 여부는 체크박스가 아니라 상태 표기(Done/In Progress/Todo 등)로 표현 — 프로젝트는 이슈처럼 단일 완료 조건이 아니라 여러 이슈의 진행 상태 모음이기 때문
- `Notes`: 프로젝트 진행 중 알게 된 맥락, 후속 이슈로 분리된 사항, 팀 간 조율 내용 등

이슈 포맷에 있는 `Acceptance Criteria`는 프로젝트에는 두지 않음 — 프로젝트 완료 여부는 `Related Issues`의 상태로 판단하므로 별도 체크리스트가 중복됨.

Project 설명 작성 전 프로젝트에 할당된 issue 목록을 먼저 확인하고, 그 내용을 근거로 작성.
