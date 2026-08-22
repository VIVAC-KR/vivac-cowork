# Linear 이슈 등록 형식

Linear에 새 issue를 등록할 때(`mcp__claude_ai_Linear__save_issue` 등) description 본문은 아래 섹션 구성을 따름.

## 본문 템플릿

```markdown
## 🎯 Goal

## 📌 Scope
### In Scope
* 

### Out of Scope
* 

## ✅ Acceptance Criteria
- [ ] 

## 🔗 Related
> 

## Notes
* 
```

- 섹션 순서·이모지·제목 문구 그대로 유지. 내용 없는 섹션도 헤더는 남김(빈 채로 두거나 "해당 없음")
- 기술 용어(함수명, 엔드포인트, 필드명 등)는 영문 그대로, 코드 인용은 백틱으로 표기
- 서비스명은 항상 `vivac`/`VIVAC`, 한글 표기(비바크, 비박 등) 금지

<!-- ## Metadata 섹션

`Metadata`(URL, Identifier, Status, Priority, Assignee, Labels, Project, Created, Updated)는 Linear가 자동으로 붙이는 부분. description 본문에 직접 쓰지 않고, 아래 필드를 도구 파라미터로 지정.

| Metadata 항목 | 대응 파라미터 |
|---|---|
| Status | `status` |
| Priority | `priority` |
| Assignee | `assignee` |
| Labels | `labels` |
| Project | `project` | -->

Project 지정 시 대상 프로젝트 없으면 새로 만들기 전에 사용자 확인.
