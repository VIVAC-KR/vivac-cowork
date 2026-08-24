# Agent Teams

이 프로젝트에서 사용하는 Claude Code Agent Team 구성을 기록합니다. 새 팀이 생기면 이 문서에 추가합니다.

> Agent Teams는 Claude Code의 실험적 기능입니다. `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`가 `.claude/settings.local.json`에서 활성화돼 있습니다. 팀원은 `.claude/agents/`의 subagent 정의를 이름으로 참조해 생성합니다 — 정의 파일은 팀 소속과 무관하게 재사용 가능한 역할이며, 팀은 이 역할들을 조합해 구성한 단위입니다.

---

## 베이스캠프 스쿼드

신규 기능 개발 시 기술/UX/비판적 검토, 3개 관점에서 병렬로 논의하는 팀입니다. 운영 원칙은 [`.claude/rules/agent-team-feature-dev.md`](.claude/rules/agent-team-feature-dev.md)에 있습니다.

### 팀원

| 이름(권장) | subagent | 역할 | 정의 파일 |
|---|---|---|---|
| architect | `architect` | 기술 아키텍처(Python/FastAPI/AWS/PostgreSQL) 설계·검토, 필요 시 구현 repo(`vivacapi-core`) 코드 작업 | [`.claude/agents/architect.md`](.claude/agents/architect.md) |
| designer | `ux-designer` | UX/UI — 화면 흐름·인터랙션·정보 구조·접근성, 디자인 시스템(토큰) 정합성 | [`.claude/agents/ux-designer.md`](.claude/agents/ux-designer.md) |
| critic | `devils-advocate` | 비판적 검토 — 반대 관점에서 가정·리스크·엣지케이스를 구체적 시나리오로 지적 | [`.claude/agents/devils-advocate.md`](.claude/agents/devils-advocate.md) |

### 소집 방법

이름을 고정해서 이후에도 팀원별로 직접 메시지를 보낼 수 있게 합니다.

```
Spawn three teammates for <기능 이름> 논의:
- architect agent type으로 "architect"
- ux-designer agent type으로 "designer"
- devils-advocate agent type으로 "critic"
```

### 운영 원칙 요약

- 이미 문서로 확정된 설계 결정은 스쿼드가 재검토하지 않습니다 — 남은 실행 범위를 찾는 데 집중합니다.
- `critic`(devils-advocate)은 기능 스코프 확정, 아키텍처·데이터 모델 선택, 주요 UX 흐름 확정 등 주요 결정마다 팀 리더가 능동적으로 검토를 요청합니다 (자동 트리거가 아니므로 리더가 챙깁니다).
- 각 팀원의 조사 결과는 팀 리더가 종합해서 Linear project/issue 등 산출물로 정리합니다.

### 사용 이력

- 2026-08-23: `/search` 지도 모드 탐색 기능 — 기존 설계 문서와 실제 코드 상태 괴리, blocker 2건 발견. Linear project + issue 11건 생성. ([project](https://linear.app/lucente/project/유저는-지도에서-위치필터-기반으로-vivac-스팟을-탐색하고-같은-결과를-리스트로도-동시에-볼-수-있습니다-d53d00f08330))
