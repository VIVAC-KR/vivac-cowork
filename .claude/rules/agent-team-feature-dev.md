# 신규 기능 개발 Agent Team 운영 규칙

새 기능 개발을 시작할 때 아래 3개 subagent 정의로 Agent Team을 구성한다. (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`는 `.claude/settings.local.json`에서 이미 활성화돼 있다.)

## 팀원 정의

| subagent | 파일 | 관점 |
|---|---|---|
| `architect` | `.claude/agents/architect.md` | 기술 아키텍처 (Python/AWS/PostgreSQL), 필요 시 구현 repo(`vivacapi-core`) 코드 작업 포함 |
| `ux-designer` | `.claude/agents/ux-designer.md` | UX/UI (화면 흐름·인터랙션·접근성) |
| `devils-advocate` | `.claude/agents/devils-advocate.md` | 비판적 검토 (반대 관점, 리스크·엣지케이스) |

## Spawn 방법

이름을 고정해서 이후에도 바로 참조할 수 있게 한다:

```
Spawn three teammates for <기능 이름> 논의:
- architect agent type으로 "architect"
- ux-designer agent type으로 "designer"
- devils-advocate agent type으로 "critic"
```

## devil's advocate 자동 개입 원칙

subagent 정의만으로는 자동 트리거가 안 된다. 팀 리더(메인 세션)가 아래 시점마다 능동적으로 `critic`(devils-advocate) 팀원에게 검토를 요청한다:

- 기능 스코프/정책이 확정될 때
- 아키텍처·데이터 모델 선택이 나올 때
- 주요 UX 흐름이 확정될 때

검토 요청 후 리더는 devils-advocate의 반박을 반영할지 정리해서 진행 여부를 판단한다. 근거 없는 반박은 기각해도 되지만, 기각 이유를 남긴다.
