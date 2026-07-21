# 기획: VIVAC CLI — 비개발자 대상 자연어 캠핑 정보 검색

> 작성일: 2026-07-21
> 전제 문서: `docs/plan.md`(VIVAC MCP Connector) — 이 기획은 그 위에 세워진다. MCP 서버 자체를 다시 만들지 않고 **세 번째 소비자**로 재사용하는 안이다.
> 대상 시나리오(유저 정의): "개발 지식이 전혀 없는 VIVAC 일반 고객이, Claude/Gemini/GPT 등 LLM에 연동해 캠핑 정보를 자연어로 검색하면 LLM을 통해 답을 얻는" 제품.

## 1. 핵심 개념과 첫 번째 모순

"CLI"와 "개발 지식 전혀 없는 일반 고객"은 원래 상충한다. 터미널을 찾아 열고, 실행 경로를 알고, 커맨드를 입력하는 행위 자체가 이미 진입장벽이다. 그래서 이 기획에서 "CLI"는 **셸 커맨드 모음이 아니라, 더블클릭으로 실행되는 단일 바이너리 안에 자연어 채팅 루프가 들어있는 TUI 앱**을 뜻하는 것으로 정의한다(4항에서 배포 형태로 다시 다룸). 이 정의를 확정하지 않으면 이후 설계가 전부 흔들리므로 7항 열린 질문 1번으로 다시 명시한다.

## 2. 왜 되는가 (기존 자산 + 새로 확인한 사실)

`docs/plan.md`가 만드는 `vivac-mcp` 원격 MCP 서버(HTTPS, Streamable HTTP/SSE)는 Claude Desktop 커넥터 전용이 아니다. 웹서치로 확인한 2026-07 기준 3대 LLM API 현황:

| Provider | API | 원격 MCP 지원 형태 | 확인 근거 |
|---|---|---|---|
| Anthropic | Messages API | `mcp_servers` 파라미터(beta `mcp-client-2025-11-20`) — url/name만 등록하면 tool로 노출 | [MCP connector - Claude Platform Docs](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector) |
| OpenAI | Responses API | `type: "mcp"` tool, 2025-06부터 지원. Streamable HTTP/SSE만, stdio 불가 | [Introducing support for remote MCP servers... - OpenAI](https://community.openai.com/t/introducing-support-for-remote-mcp-servers-image-generation-code-interpreter-and-more-in-the-responses-api/1266973), [OpenAI MCP Support Guide 2026](https://qaskills.sh/blog/openai-mcp-support-guide-2026) |
| Google | Gemini API / Vertex AI | tool type `mcp_server`(name+url) — 자동으로 function-calling 스키마 변환 | [Expanding Managed Agents in Gemini API - Google Blog](https://blog.google/innovation-and-ai/technology/developers-tools/expanding-managed-agents-gemini-api/), [Gemini SDK 🤝 FastMCP](https://gofastmcp.com/integrations/gemini) |

즉 **`vivac-mcp` 서버 하나만 만들어두면 세 프로바이더 모두 같은 URL을 tool로 등록해서 쓸 수 있다.** region 정규화, 상세 fan-out, rate limit 같은 서버 쪽 로직은 100% 재사용 — CLI가 새로 만들어야 하는 건 "프로바이더별 API 호출 + 터미널 렌더링"뿐이다. `docs/plan.md`의 의존 방향 원칙(“이 repo는 vivacapi-core를 호출하는 소비자”)도 그대로 유지된다 — CLI는 데이터에 직접 접근하지 않고 LLM API를 한 홉 거쳐 같은 MCP 서버를 두드린다.

## 3. 코드/현황 대조로 확인한 공백 (리스크)

| # | 문제 | 근거 | 영향 |
|---|---|---|---|
| 1 | "개발지식 0" 유저 + "CLI" 형태 충돌 | 1항 서술 | TUI 실행 바이너리로 갈 것을 강제 — `npx` 등 런타임 설치가 필요한 배포 방식은 탈락 (4항) |
| 2 | LLM 추론 비용 부담 주체 미정 | Connector(`docs/plan.md`)는 유저의 Claude 계정/구독이 추론 비용을 낸다. CLI는 비개발자 유저가 자기 API 키를 발급받을 수 없으므로(결제수단 등록, 콘솔 가입 필요) VIVAC이 자체 API 키로 대신 호출해야 함 | 유저 수만큼 정비례하는 새 변동비 발생 — Connector와는 완전히 다른 비용 구조. 쿼터/과금 설계 없이는 무제한 트래픽 = 무제한 청구서 |
| 3 | 프로바이더 3사 API 스키마 불일치 | 2항 표에서 확인 — MCP tool 등록 방식은 비슷해도 요청/응답/스트리밍 이벤트 포맷은 Anthropic/OpenAI/Gemini 셋 다 다름 | MCP 서버는 공유해도 CLI의 "에이전트 루프"(요청 조립, 스트리밍 파싱, 에러 매핑, tool 호출 승인 흐름)는 프로바이더별로 별도 구현 필요 — 완전한 zero-cost 재사용은 아님 |
| 4 | `docs/plan.md` 3항의 공백 3개를 그대로 승계 | 같은 MCP 서버·같은 `vivacapi-core` 데이터를 쓰므로 (a) 대중교통 접근성 필드 부재 (b) region 역매핑 (c) 목록 응답 부실이 CLI 답변에도 동일하게 나타남 | CLI 자체 문제가 아니라 상류 의존 — `docs/plan.md` V2가 풀려야 CLI 답변 품질도 같이 좋아짐. 이 기획에서 별도로 다시 풀 필요 없음 |
| 5 | 트래픽 폭주 시 이중 비용 | Connector는 트래픽이 몰려도 VIVAC은 `vivacapi-core` 서빙 비용만 진다. CLI는 트래픽이 몰리면 서빙 비용 + LLM API 비용을 동시에 진다 | `docs/plan.md`가 말한 rate limit만으로는 부족 — 요청당 과금이 실제로 발생하므로 "일일 쿼터/사용량 상한"이 rate limit보다 우선순위 높은 안전장치 |

## 4. 설계 방향

### 4.1 배포 형태 — 세 가지 선택지

| 옵션 | 설명 | 비개발자 적합도 | 비고 |
|---|---|---|---|
| A. `npx` 실행형 | `npx @vivac/cli` | 낮음 — Node.js 설치가 전제, 비개발자 PC엔 보통 없음 | 배포/업데이트는 제일 쉬움 |
| B. 단일 실행 바이너리 | Go/Bun 컴파일 또는 `pkg`로 macOS/Windows 바이너리 배포, 다운로드 후 더블클릭 → TUI 채팅창 | 높음 — 런타임 설치 불필요 | 이 기획의 1항 정의와 부합. **권장** |
| C. VIVAC 앱(웹/모바일) 내장 챗봇 탭 | 엄밀히는 CLI 아님. `VIVAC-frontend`/`vivac-mobile-test`에 이미 로그인·배포 인프라 존재 | 제일 높음 — 유저가 이미 쓰는 앱 안 | 대상 유저층 기준으로는 사실상 최종 정답에 가까움 |

권장안: **V1은 B(단일 바이너리)로 빠르게 검증**, 검증되면 C(자체 앱 내장)로 흡수하는 것을 기본 방향으로 잡는다. CLI는 목적지가 아니라 프로토타입 통로.

### 4.2 LLM API 키 소유권

- BYOK(유저가 자기 API 키 입력)는 대상 유저 정의와 정면으로 충돌 — 탈락.
- **VIVAC 소유 API 키로 서버가 대신 호출**하는 방식만 현실적 → 3항 리스크 2, 5를 정면으로 받는다. 쿼터(예: 익명 기준 기기당 일 N회)와 초과 시 처리(대기/차단/로그인 유도)를 반드시 V1 스코프에 포함한다.

### 4.3 아키텍처

```
VIVAC CLI (단일 실행 바이너리, TUI 채팅 루프)
  → 유저가 물은 자연어를 LLM API로 전달 (VIVAC 소유 키)
     - Anthropic Messages API (mcp_servers)
     - OpenAI Responses API (type: "mcp")   ← V2
     - Gemini API (mcp_server tool)          ← V2
       → 각 API가 vivac-mcp 서버 URL을 원격 MCP tool로 호출
         → vivac-mcp (docs/plan.md와 동일 리소스, 신규 구축 아님)
           → vivacapi-core 공개 API (/v1/explore/spots, /v1/explore/spots/{uid})
```

CLI 프로세스 자신은 VIVAC 데이터에 직접 접근하지 않는다 — 워크스페이스 CLAUDE.md의 "core가 유일한 상류" 원칙을 LLM API를 한 홉 거쳐서도 유지한다.

## 5. 사용 시나리오 (구체 예시)

1. 비개발자 유저(VIVAC 앱 기존 회원)가 `vivac.app/cli` 페이지에서 macOS용 바이너리를 내려받아 더블클릭한다.
2. 터미널(또는 자체 TUI) 창이 뜨고 "어떤 여행을 계획 중이세요?" 프롬프트가 나온다.
3. 유저 입력: "경남에 백패킹 갈만한 곳 있어? 나 차 없어"
4. CLI가 VIVAC 소유 Anthropic API 키로 Messages API를 호출하면서, `mcp_servers`에 `vivac-mcp` URL을 등록해둔 상태로 요청을 보낸다.
5. Claude가 `search_backpacking_spots` tool을 호출(region=경남을 서버 쪽 역매핑 로직이 경상남도로 정규화, no_car 힌트 전달).
6. `vivac-mcp`가 `vivacapi-core`에서 목록+상세 fan-out 후 반환.
7. Claude가 답변 생성 — 단, "차 없음"에 대응하는 구조화 필드가 없으므로(`docs/plan.md` 3항 문제 1) "대중교통 정보는 별도 확인이 필요합니다" 같은 단서를 달아 답한다.
8. CLI 터미널에 답변이 출력된다.

이 시나리오에서 6, 7단계는 `docs/plan.md`가 이미 식별한 리스크가 CLI 레이어에서도 그대로 드러나는 지점이다 — CLI가 새로 만드는 문제가 아니라 상류에서 물려받는 문제임을 다시 강조.

## 6. 범위 제안

| Phase | 내용 |
|---|---|
| **V1(프로토타입)** | 단일 프로바이더(Anthropic만 — Messages API의 MCP connector가 제일 먼저 성숙), 단일 플랫폼 바이너리(macOS 우선), VIVAC 소유 키, 익명 기준 기기당 일일 쿼터 하드코딩, 로그인 없음 |
| **V2** | OpenAI/Gemini 추가, VIVAC 계정 로그인 연동(기존 Google 로그인 재사용)으로 유저 단위 쿼터·사용량 추적 전환, Windows/Linux 빌드 |
| **V3** | 별도 배포 중단 검토 — 검증된 기능을 `VIVAC-frontend`/모바일 앱 내 챗봇 탭으로 흡수(4.1의 옵션 C) |

## 7. 열린 질문

1. **"CLI"의 정의 확정** — 1항의 TUI 바이너리 해석이 맞는지, 아니면 실제로 개발자/파워유저 대상 진짜 셸 커맨드를 원하는지(대상 유저 정의와 다시 충돌하니 재확인 필요).
2. LLM API 비용을 VIVAC이 전액 부담할 때 구체적 쿼터·과금 정책(무료 일 N회, 초과 시 로그인 요구 또는 차단).
3. 익명 사용 허용 시 어뷰징 방어 수단(기기 핑거프린팅 vs 로그인 강제) — V1 스코프에 포함할지.
4. 배포 채널 — 자체 사이트 직접 다운로드 vs Homebrew tap/winget vs (4.1 옵션 C) 앱 내장으로 처음부터 직행.
5. 착수 순서 — `docs/plan.md`(Connector, VIVAC 비용 zero)와 이 CLI 기획(VIVAC이 LLM 비용 부담) 중 무엇을 먼저 낼지. **결정: Connector(`docs/plan.md` V1)를 먼저 낸다.** 8항 참고.

## 8. 개발 착수 순서와 스택 (docs/plan.md와 공유 백엔드)

CLI가 호출하는 `vivac-mcp` 서버는 `docs/plan.md` V1과 완전히 같은 코드다 — 새로 만드는 게 아니라 그쪽 착수 순서를 그대로 따른다. CLI 고유 작업(에이전트 루프, 바이너리 패키징)은 이 서버가 로컬에서 검증된 뒤 시작한다.

### 8.1 서버(공유 백엔드) 착수 순서

1. repo 스캐폴딩 (`uv init`, `pyproject.toml`)
2. region 역매핑 — `vivacapi-core`의 `SIDO_ABBR`를 뒤집은 dict. `vivacapi-core` 코드 안 건드림, 제일 작고 독립적
3. `vivacapi-core` 공개 API 클라이언트 (`httpx.AsyncClient`) — `/v1/explore/spots`, `/v1/explore/spots/{uid}` 래핑
4. MCP tool 1개 구현 — `search_backpacking_spots`(목록 조회 → region 역매핑 적용 → 상세 fan-out)
5. 로컬 구동 검증 (MCP inspector로 tool 호출 확인)
6. rate limit — 앱 코드 아니라 **리버스프록시 레벨**(`vivacapi-core`의 `docs/backlog/auth-rate-limit-260711.md`가 이미 이 방향으로 잡아놓은 패턴 재사용)
7. 배포 — 같은 Lightsail 인스턴스에 컨테이너 추가(7항 열린 질문 4의 보수적 선택지)

### 8.2 스택

| 항목 | 선택 | 근거 |
|---|---|---|
| 언어/툴체인 | Python 3.12 + `uv` | `vivacapi-core`와 통일 — 팀이 이미 아는 툴체인 |
| MCP 서버 | 공식 `mcp` SDK의 `FastMCP`, transport=`streamable-http` | 별도 웹프레임워크 불필요, 공식 SDK에 내장(Starlette/uvicorn) |
| API 호출 | `httpx.AsyncClient` | core 개발 의존성에 이미 있는 라이브러리 |
| 테스트 | `pytest` + `pytest-asyncio` | 워크스페이스 전체 컨벤션과 동일 |
| rate limit | 앱 코드 대신 nginx/CloudFront | core 백로그가 이미 잡은 방향과 일치, 새 의존성 없음 |

### 8.3 CLI 고유 스택 (서버 검증 이후, 아직 미확정)

바이너리 패키징 언어(4.1 옵션 B)는 서버와 별개 결정 — Python(PyInstaller/Nuitka) vs Go 재작성 중 선택 필요. 서버가 로컬에서 돌아가는 걸 먼저 확인한 뒤 착수한다(7항 열린 질문 1의 "CLI 정의" 확정과 묶어서 결정).
