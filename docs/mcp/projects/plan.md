# 기획: VIVAC MCP Connector — 자연어 백패킹 스팟 검색

> 작성일: 2026-07-21
> 예시 시나리오: 유저가 Claude(웹/데스크톱)에서 "경남에 백패킹 갈만한 장소가 어디있을까? 난 차가 없어"라고 물으면, Claude가 이 connector를 통해 VIVAC 데이터로 답한다.

## 1. 핵심 개념

Claude.ai/Desktop의 **Connector**는 내부적으로 **remote MCP 서버**(HTTPS, SSE/Streamable HTTP)다. 일반 유저 대상이므로 로컬 stdio MCP(개발자용, 설치 필요) 아니라 **원격 MCP 서버**로 간다 — 유저는 Claude 설정에서 "VIVAC" connector 추가만 하면 끝, 설치 불필요.

## 2. 왜 되는가 (기존 자산)

`GET /v1/explore/spots`, `GET /v1/explore/spots/{uid}`(`vivacapi-core`)는 이미 **완전 공개, 비로그인, JWT 불필요**. 인증 없는 read-only MCP tool로 감싸기 좋은 조건 — 새 인증 계층 안 만들어도 됨.

## 3. 코드 대조로 확인한 공백 3개 (이 기획의 핵심 리스크)

`vivacapi-core` 실제 코드(`models/spot.py`, `core/region.py`, `api/v1/endpoints/explore.py`) 확인 결과:

| # | 문제 | 근거 | 영향 |
|---|---|---|---|
| 1 | **"차 없음"(대중교통 접근성) 데이터 자체가 없음** | `Spot` 컬럼 전체 확인 — `nearby_facilities`(자유배열)는 있어도 구조화된 "transit_accessible" 류 필드 전무 | "차 없어" 조건은 지금 DB로 필터 불가능. LLM이 텍스트 필드 보고 추론하거나 "확인 필요" 명시하는 수밖에 없음 |
| 2 | **지역 축약어 매칭 안 됨** | `core/region.py`의 `SIDO_ABBR`는 풀네임→약칭 **한 방향만**(경상남도→경남). 역방향 매핑 없음. `list_spots`의 `region_province` 필터는 정확일치 | "경남에"라고 물으면 `region_province=경남`으로 넘길 시 DB엔 "경상남도"로 저장돼 있어 **0건** 나올 위험 |
| 3 | **목록 응답이 LLM 답변 구성에 부실** | `SpotListItem` = `uid/title/trust_tier/category/region_short/thumbnail_url`뿐(`explore.py` 확인) — 주소·평점·테마·인근시설·예약링크 없음 | LLM이 "이유"까지 대려면 후보 하나하나 상세(`/spots/{uid}`) 추가호출 필요 → N+1 호출, 응답 느려짐/컨텍스트 커짐 |

## 4. 설계 방향

```
Claude.ai (유저) → Connector(remote MCP server, 이 repo) → vivacapi-core 공개 API
```

- 이 repo는 `vivacapi-core`를 호출하는 소비자 중 하나 — DB 직접 접근 없음, 항상 공개 API 경유(워크스페이스 CLAUDE.md의 의존 방향 원칙 그대로)
- **MCP tool 1개로 시작**: `search_backpacking_spots` — 입력은 자유텍스트 아니라 **구조화 파라미터**(region, no_car: bool, themes 등). 자연어 파싱은 Claude 자체가 이미 하므로 tool 안에 별도 NLU 안 만듦
- tool 내부 처리:
  1. region 정규화 — 역방향 매핑(약칭→풀네임)을 이 repo 쪽에 둔다(vivacapi-core 안 건드리는 게 가장 빠름)
  2. `list_spots`/`search_spots` 호출
  3. 후보 일부는 상세(`/spots/{uid}`)까지 fan-out해서 그라운딩용 필드 채워 반환
- tool description에 **"반환된 필드 밖 정보는 지어내지 말 것"** 명시 — 할루시네이션(예약 가능 여부, 실제 교통편 등) 방지가 제일 중요한 안전장치
- 인증: V1은 없음(공개데이터). 대신 **rate limit 필요** — `/v1/explore/*`엔 현재 아무 제한도 없음(`vivacapi-core`의 `backlog/auth-rate-limit-260711.md`는 `/v1/auth/*`만 다룸, explore는 대상 아니었음). Connector가 트래픽을 끌면 새로운 고트래픽 유입점 — 이 repo(또는 앞단 프록시)에서 반드시 제한

## 5. 범위 제안

| Phase | 내용 |
|---|---|
| **V1(MVP)** | 원격 MCP 서버 1개, tool 1개, region 정규화, 상세 fan-out, rate limit. "차없음"은 정확 필터 대신 관련 텍스트필드 노출해서 LLM이 답변에 "확인 필요" 단서 달게 함 |
| **V2** | `vivacapi-core`의 `Spot`에 대중교통 접근성 구조화 필드 신설(ETL 반영 필요 — `vivacapi-core` 쪽 별도 `docs/projects/*.md` 설계 대상), tool에 `no_car` 실제 필터로 승격 |
| **V3** | 개인화(그룹/찜 연동), 다중 turn 대화 상태(이전 추천 기억), 예약 링크 클릭 트래킹 연계 |

## 6. 열린 질문

1. 배포 위치: `vivacapi-core`와 같은 Lightsail 인스턴스에 컨테이너 추가 vs 별도 인프라 — 트래픽 예측 안 된 상태라 보수적으로 같은 인스턴스로 시작 권장
2. "차없음" 데이터 확보를 V1에 억지로 우겨넣을지, V2로 미룰지 — 데이터 없이 필터 흉내내면 잘못된 정보 줄 위험 큼. **V2로 미루는 쪽 권장**
3. 트래픽 스파이크 대비 — 지금 인프라가 t2/t3 micro급이라 Connector 바이럴 시 별도 검토 필요
4. ~~스택 미정~~ → **결정: Python 3.12 + `uv` + 공식 `mcp` SDK(`FastMCP`, transport=`streamable-http`) + `httpx`.** `vivacapi-core`와 툴체인 통일, 별도 웹프레임워크 불필요(SDK가 Starlette/uvicorn 내장). 착수 순서는 `docs/cli-plan.md` 8.1 참고 — repo 스캐폴딩 → region 역매핑 → `vivacapi-core` httpx 클라이언트 → MCP tool 구현 → 로컬 검증 → rate limit(리버스프록시 레벨) → 배포
