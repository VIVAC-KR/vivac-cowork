# 기능 명세

> 제품 정의·MVP 범위·데이터 정의는 [PRODUCT.md](../PRODUCT.md)가 기준이다. 구현 현황은 [STATUS.md](../STATUS.md) §7.

| 문서 | 대상 화면·라우트 | 내용 |
|---|---|---|
| [home.md](home.md) | 홈 `/` | 진입점. 검색으로 유도하고 큐레이션된 스팟을 노출한다 |
| [search.md](search.md) | 검색 `/search` | 조건에 맞는 스팟을 찾고 비교한다 — 목록 · 필터 · 지도 탐색 |
| [spot-detail.md](spot-detail.md) | 스팟 상세 `/spots/{uid}` | 한 스팟의 정보를 확인하고 예약·문의로 이어진다 |
| [auth.md](auth.md) | 로그인 `/login` | 로그인 상태를 유지한다 |
| [_common.md](_common.md) | 라우트 없음 (전 화면 공통) | 로딩·빈 결과·오류를 일관되게 처리한다 |

각 문서는 **목적 → 동작 → 확정 계약 → 수용 기준** 4블록으로 기술한다.
