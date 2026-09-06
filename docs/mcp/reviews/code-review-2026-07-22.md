# vivac-mcp 코드 리뷰 (2026-07-22)

## 요약

Python 3.12 + `uv` + 공식 `mcp` SDK(`FastMCP`, streamable-http) + `httpx` 스택, `src/`에 실질 코드 5개 파일뿐인 기획 초기 단계지만 README 기준 이미 `https://mcp.vivac.app`에 배포되어 실 트래픽을 받는 상태다. `docs/plan.md`가 사전에 짚어둔 공백(지역 축약어, 목록 응답 부실, 대중교통 데이터 부재) 3개 모두 코드에 정확히 반영되어 있어 기획-구현 정합성은 이 단계 코드베이스치고 상당히 높다. 다만 `vivacapi-core`를 호출하는 유일한 통로인 `client.py`가 직접 테스트를 전혀 갖고 있지 않은 점, 그리고 네트워크/타임아웃 예외 처리와 로깅이 전무해 실패가 운영자에게 보이지 않는 점은 이미 배포된 서비스 기준으로 보면 가볍게 넘길 사안은 아니다.

| 심각도 | 건수 |
|---|---|
| Critical | 0 |
| High | 3 |
| Medium | 2 |
| Low / Nit | 2 |
| 기획단계 결정 필요 | 5 |

## Critical

해당 없음.

## High

### H1. `client.py`가 직접 테스트되지 않음 — `src/vivac_mcp/client.py:12-47`
- **문제:** `vivacapi-core`와 실제로 HTTP 통신하는 코드는 `client.py`의 `list_spots`/`get_spot` 두 함수가 전부인데, 이 파일을 직접 검증하는 테스트가 하나도 없다.
- **근거:** `tests/test_server.py`의 세 테스트 모두 `monkeypatch.setattr(server, "list_spots", fake_list_spots)` / `monkeypatch.setattr(server, "get_spot", fake_get_spot)`로 `client.py` 함수 자체를 통째로 대체한다(`tests/test_server.py:16-17`, `34-35`, `50-51`). `tests/` 디렉터리에 `test_client.py`나 `conftest.py`가 없어 쿼리 파라미터 조립(`params` dict), `base_url`/`timeout` 설정, status code 분기, `response.json()["items"]` 파싱 등 외부 계약과 맞닿는 로직이 회귀 검증 대상에서 완전히 빠져 있다. 외부 API 계약 변경이나 파라미터 이름 오타 같은, 이 종류 프로젝트에서 가장 흔히 터지는 버그를 지금 테스트가 하나도 못 잡는다.
- **제안:** `httpx.MockTransport` 등으로 `list_spots`/`get_spot`을 직접 호출하는 테스트를 최소 1~2개 추가한다 — 정상 응답 파싱, non-200 시 `VivacApiError` 발생, 404 시 `None` 반환 정도만 있어도 이 계층의 가장 큰 위험은 잡힌다.

### H2. 네트워크/타임아웃 예외가 처리되지 않고, 실패가 어디에도 기록되지 않음 — `src/vivac_mcp/client.py:27-35`, `39-47`
- **문제:** `list_spots`/`get_spot` 모두 `response.status_code != 200`인 경우만 `VivacApiError`로 감싸 던진다. `vivacapi-core`가 완전히 죽어있거나(connection refused) 응답이 느려 10초 타임아웃에 걸리는 경우, `client.get(...)` 호출 자체(`client.py:30`, `42`)가 `httpx.ConnectError`/`httpx.TimeoutException`을 던지는데 이건 잡히지 않고 원본 타입 그대로 `search_backpacking_spots`까지 전파된다. 즉 실패 원인에 따라 호출자가 받는 예외 타입이 `VivacApiError`(구조화)와 raw `httpx.*Error`(비구조화)로 제각각이다.
- **근거:** 설치된 `mcp` SDK(`.venv/lib/python3.12/site-packages/mcp/server/lowlevel/server.py:585-590`)를 직접 확인한 결과, tool 함수가 던진 예외는 `except Exception as e: return self._make_error_result(str(e))`로 잡혀 MCP 프로토콜의 `CallToolResult(isError=True, ...)`로 변환된다 — 그래서 vivac-mcp 프로세스 자체가 죽지는 않는다. 하지만 이 경로 어디에도 `logger` 호출이 없다(같은 파일 588-590줄, `tools/base.py:116-117`의 `ToolError` 래핑도 마찬가지). vivac-mcp 쪽에서 별도로 로깅하지 않는 한, `vivacapi-core` 장애나 타임아웃이 발생해도 vivac-mcp 서버 로그에는 아무 흔적도 안 남고 사용자에게는 raw 예외 문자열만 그대로 노출된다. README상 이미 프로덕션 배포된 서비스라는 점을 고려하면 장애 발생 시 운영자가 이를 알아챌 방법이 지금은 없다.
- **제안:** `client.py`에서 `httpx.HTTPError`(타임아웃/연결 오류 포함)까지 함께 잡아 `VivacApiError`로 통일하고, 그 지점에서 최소 `logger.exception(...)` 한 줄이라도 남긴다. 예외 타입을 하나로 통일하면 향후 tool이 늘어나도 같은 처리를 반복하기 쉬워진다.

### H3. `asyncio.gather`가 하나의 실패로 전체 결과를 날림 — `src/vivac_mcp/server.py:55-56`
- **문제:** `details = await asyncio.gather(*(get_spot(uid) for uid in uids))`는 `return_exceptions=True`가 없다. fan-out 대상 `uid` 중 하나라도 `get_spot`이 예외(타임아웃, 5xx 등)를 던지면 `gather` 전체가 그 예외로 즉시 실패하고, 이미 성공했을 나머지 상세 조회 결과까지 전부 버려진다.
- **근거:** 바로 다음 줄 `return [detail for detail in details if detail is not None]`은 "스팟이 존재하지 않아 404 → `None`"인 경우는 명시적으로 관대하게 처리하도록 설계돼 있다(`tests/test_server.py:43-55`의 `test_spot_not_found_during_fanout_is_dropped`가 이를 검증). 하지만 "일시적 오류로 실패"하는 경우는 같은 관대함이 적용되지 않고 H2와 맞물려 전체 tool 호출이 통째로 에러 처리된다 — 5건 중 4건은 정상 조회됐는데 1건의 일시적 실패 때문에 사용자가 아무 결과도 못 받는 상황이 생길 수 있다.
- **제안:** `asyncio.gather(..., return_exceptions=True)`로 바꾸고, 반환값 필터링 시 `Exception` 인스턴스도 `None`과 함께 걸러낸다. 부분 성공을 살리는 쪽이 이 tool의 "확인 안 된 정보는 지어내지 말 것"이라는 기존 설계 방향과도 더 잘 맞는다.

## Medium

### M1. MCP tool 파라미터에 스키마 제약/설명이 전혀 없음 — `src/vivac_mcp/server.py:25-31`
- **문제:** `search_backpacking_spots`의 개별 파라미터(`region`, `keywords`, `category`, `limit`)에 `Field(description=...)` 등 필드 단위 메타데이터가 없다. 특히 `limit: int = 5`는 상한/하한이 없고, `category: list[str] | None`은 유효값을 알려줄 방법이 아예 없다.
- **근거:** `vivacapi-core`의 `/v1/explore/spots`는 `limit: int = Query(20, ge=1, le=50)`로 범위를 강제한다(`vivacapi-core/vivacapi/api/v1/endpoints/explore.py:28`). vivac-mcp의 tool 스키마에는 이 제약이 없어, LLM이 범위 밖 `limit`을 보내면 core가 422를 반환하고 그게 H2의 미처리 예외 경로를 그대로 타 raw 에러 문자열로 노출된다. `category`는 상황이 더 나쁘다 — `vivacapi-core`의 `Spot.category`가 `ARRAY(String)`일 뿐 enum이 아니라서(`vivacapi-core/vivacapi/models/spot.py:115`) core 쪽에도 유효값 목록이 없고, vivac-mcp도 이를 하드코딩하거나 노출하지 않는다. LLM은 사실상 대화 맥락에서 카테고리 코드를 추측해서 보낼 수밖에 없다.
- **제안:** `limit`에 최소 `Field(default=5, ge=1, le=50)` 정도의 제약을 달아 core의 제약과 스키마 단계에서 일치시킨다. `category`는 알려진 값 목록이 있다면(없다면 F3 참고) tool description에라도 예시를 명시해 LLM이 임의 값을 만들어내지 않도록 한다.

### M2. 이 repo의 CLAUDE.md/rules 문서가 vivacapi-core를 그대로 복제한 것으로 보이며 실제 코드와 맞지 않음 — `CLAUDE.local.md:47-83`, `.claude/rules/testing.md:8-12`, `.claude/rules/api-conventions.md`, `.claude/rules/security.md`
- **문제:** 리뷰 기준 문서로 참고한 `CLAUDE.md`/`CLAUDE.local.md`/`.claude/rules/*.md`가 vivacapi-core repo의 내용을 거의 그대로 옮겨온 것으로 보인다. 이 repo에는 존재하지 않는 아키텍처를 사실인 것처럼 서술한다.
- **근거:** `CLAUDE.local.md:47-49`는 "Framework: FastAPI / Database: SQLAlchemy asyncio + PostgreSQL / Auth: JWT Bearer"라고 명시하고, `:51-60`은 `routers`/`crud`/`schemas`/`models`/`core` 4계층 원칙을, `:62-83`은 `vivac-api/vivacapi/...`(alembic, docker-compose.yml, example.env 포함) 트리를 그린다 — 그러나 vivac-mcp의 실제 구조는 `src/vivac_mcp/{client.py, server.py, core/{config.py, region.py}}`뿐이고 DB/ORM/JWT/routers/crud layer 자체가 없다. `.claude/rules/testing.md:8-12`는 `db_client`/`db_session`/`apply_migrations` picture(`conftest.py`)를 픽스처 규약으로 설명하지만, vivac-mcp `tests/`에는 `conftest.py`조차 없고 실제 테스트는 `monkeypatch`만 쓴다. `.claude/rules/api-conventions.md`(전체)와 `.claude/rules/security.md`(전체)는 `/v1/internal/...`, `AppException`/`ErrorCode`, `StaffRole`, SQLAdmin 등 vivacapi-core 고유의 REST/인증 체계를 다루는데 vivac-mcp는 MCP 프로토콜 서버라 이 중 어느 것도 적용되지 않는다.
- **제안:** 지금은 코드량이 적어 실질 피해가 크지 않지만, 이 문서들은 (사람이든 AI agent든) 다음 작업자가 "리뷰 잣대"로 그대로 신뢰할 문서다. 앞으로 코드가 늘어나기 전에 vivac-mcp 실제 구조에 맞게 다시 쓰거나, 최소한 복제본이라는 표시라도 남겨두는 편이 낫다.

## Low / Nit

### L1. region 후보 루프가 순차 호출 — `src/vivac_mcp/server.py:42-52`
- **문제:** `for region_province in candidates:` 루프는 후보(최대 2개: 예 "강원" → 강원도/강원특별자치도)를 순차적으로 `await`한다. 같은 함수 안 상세 조회 fan-out(`:55`)은 `asyncio.gather`로 병렬화되어 있는 것과 대조적이다.
- **근거:** `len(merged) >= limit`이면 `break`하는 조기 종료 최적화(`:51-52`) 때문에 의도적으로 순차 처리했을 가능성이 있다 — 첫 후보만으로 `limit`이 채워지면 두 번째 호출 자체를 생략할 수 있다. 병렬화하면 이 이점은 사라진다.
- **제안:** 의도된 트레이드오프라면 짧은 주석(예: `ponytail:` 스타일)으로 "왜 순차인지"를 남겨두면 다음 사람이 "버그인가?"로 헷갈리지 않는다. 아니라면 `asyncio.gather`로 통일한다.

### L2. `.gitignore`가 `.env.example`을 예외 처리하지만 해당 파일이 없음 — `.gitignore:19-21`
- **문제:** `.env*`를 무시하되 `!.env.example`로 예외를 둬서 example 파일은 커밋되도록 의도한 패턴인데, repo에 `.env.example`(혹은 `example.env`) 파일 자체가 없다.
- **근거:** repo 루트/서브디렉터리를 `*.env*` 패턴으로 찾아도 매칭되는 파일이 없다. 현재는 README(`README.md:47`)가 `VIVACAPI_BASE_URL` 등 환경변수를 프로즈로 설명해 기능적 공백은 아니다.
- **제안:** 사소하지만 예외 패턴을 추가한 의도가 있다면 `.env.example` 파일을 만들어 채우거나, 안 만들 거면 `.gitignore`의 예외 줄을 정리한다.

## 기획단계 결정 필요 (foundational)

- **F1. Rate limit이 실제로 걸려 있는지 이 repo만으로는 확인 불가.** `docs/plan.md:37`은 `/v1/explore/*`가 완전 비인증 공개 API라 Connector가 트래픽을 끌면 반드시 리버스프록시 레벨에서 rate limit을 걸어야 한다고 명시한다. 그런데 README는 이미 `✅ 배포됨`이라고 밝히고 있고, 이 repo(`infra/docker-compose.yml`, `Dockerfile`)에는 rate limit 관련 설정이 전혀 없다 — 설계상 nginx/CloudFront 쪽(다른 repo/인프라)에 두기로 했으니 범위 밖일 수 있지만, 실제로 적용됐는지는 vivac-mcp만 봐서는 검증할 수 없다. 이미 프로덕션 트래픽을 받는 상태이므로 확인이 필요하다.
- **F2. 에러 처리/로깅 정책 미정 (H2 연장).** 지금은 client.py 두 함수에서만 문제지만, tool이 늘어날수록 같은 구멍이 반복 복제될 가능성이 크다. "실패 시 어디까지 잡고, 어떻게 로깅하고, MCP 클라이언트에 뭘 보여줄지"를 패턴으로 먼저 정해두는 편이 나중에 N개 호출부를 한 번에 고치는 것보다 싸다.
- **F3. `category` 화이트리스트/enum 도입 여부.** M1에서 지적한 대로 vivac-mcp도 vivacapi-core도 유효한 category 값을 구조화된 형태로 갖고 있지 않다. workspace의 `code-style.md` 관례(`enum.StrEnum` 사용)를 참고해 짧은 화이트리스트를 도입할지, 아니면 이대로 자유 텍스트로 둘지 결정이 필요하다.
- **F4. CLAUDE.md류 문서 정리 (M2 연장).** 지금 정리해두면 비용이 작지만, 코드가 더 쌓인 뒤에는 "무엇이 진짜고 무엇이 복사본 잔재인지" 가려내는 비용이 커진다.
- **F5. CI(lint/test 자동화) 부재.** `.github/workflows` 등 자동화 파이프라인이 repo에 없어, 이번 리뷰에서 나온 것과 같은 회귀를 push/PR 단계에서 잡아줄 장치가 아직 없다. PR 3개가 이미 머지된 이력이 있어(`git log`) 워크플로우 자체는 갖춰져 가는 중으로 보이므로, `uv run ruff check`/`uv run pytest`를 doing PR 단계에 붙이는 결정을 미루지 않는 편이 좋다.

## 잘된 점

- `docs/plan.md` §3이 미리 짚어둔 공백 3개(지역 축약어, 목록 응답 부실, 대중교통 데이터 부재)가 모두 코드에 반영돼 있다 — `region.py`의 역매핑, `server.py`의 상세 fan-out, tool docstring의 "확인이 필요하다고 답변에 명시할 것" 지시. 기획-구현 정합성이 이 단계 코드베이스치고 눈에 띄게 높다.
- `client.py:10-11`의 `ponytail:` 주석(호출마다 새 `AsyncClient`를 쓰는 이유와 트래픽 늘 때의 전환 방향을 명시)은 의도된 단순화를 표시하는 모범적인 예다.
- 스택 선택이 `docs/plan.md` §6.4 / `docs/cli-plan.md` §8.2의 결정과 `pyproject.toml`이 정확히 일치한다(Python 3.12, `uv`, `mcp[cli]` FastMCP streamable-http, `httpx`) — 계획과 구현 사이 드리프트가 없다.
- `region.py`의 가장 까다로운 분기(약칭 하나에 풀네임 여러 개가 매핑되는 케이스, 예: 강원)가 `tests/test_region.py`에 정확히 커버돼 있다.
- 시크릿 관련 문제 없음 — `.env*`가 `.gitignore`/`.dockerignore`에서 제외되고, working tree와 git 전체 히스토리를 뒤져도 자격증명/키 패턴이 나오지 않는다. `eval`/`exec`/`pickle`/`subprocess` 등 위험한 호출도 없고, 쿼리 파라미터는 문자열 결합 없이 `httpx`의 `params=` dict로만 전달돼 injection 여지가 없다.
- Dockerfile이 멀티스테이지 + 비루트 유저 + `uv sync --frozen`으로 구성돼 있어 첫 Dockerfile치고 기본기가 탄탄하다.
