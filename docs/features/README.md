# 기능 명세

> 제품 정의·MVP 범위·데이터 정의는 [PRODUCT.md](../product/PRODUCT.md)가 기준이다. 구현 현황은 [STATUS.md](../STATUS.md) §7.

| 문서 | 대상 화면·라우트 | 내용 |
|---|---|---|
| [home.md](home.md) | 홈 `/` | 진입점. 검색으로 유도하고 큐레이션된 스팟을 노출한다 |
| [search.md](search.md) | 검색 `/search` | 조건에 맞는 스팟을 찾고 비교한다 — 목록 · 필터 · 지도 탐색 |
| [spot-detail.md](spot-detail.md) | 스팟 상세 `/spots/{uid}` | 한 스팟의 정보를 확인하고 예약·문의로 이어진다 |
| [auth.md](auth.md) | 로그인 `/login` | 로그인 상태를 유지한다 |
| [_shared/states.md](_shared/states.md) | 라우트 없음 (전 화면 공통) | 로딩·빈 결과·오류를 일관되게 처리합니다 |

각 화면 문서는 [meta/templates/feature-template.md](../meta/templates/feature-template.md)의 구조를 따릅니다. 화면 문서는 자기완결형 기능 명세서이며, 정본이 정의한 값을 옮겨 적을 때는 정본을 함께 표시합니다([constitution.md](../meta/constitution.md) 원칙 V 한정 규정).

`_shared/`는 라우트가 없는 공통 규칙 문서를 모읍니다. 화면 문서가 아니므로 자기완결 요구를 적용하지 않고, 각 화면 문서가 참조합니다.

> 위 목록의 문서들은 아직 새 구조로 마이그레이션되지 않았습니다.
