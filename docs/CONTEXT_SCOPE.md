# 공유 컨텍스트 참고 범위

각 구현 저장소의 `docs/`는 `vivac-cowork` 저장소의 `docs/` 폴더를 가리킵니다(연결 방법은 [SYMLINK-SETUP.md](../SYMLINK-SETUP.md)). 이 문서는 그중 **무엇을 기본으로 읽을 것인가**를 정합니다.

## 기본으로 참고

- [product/PRODUCT.md](product/PRODUCT.md) — 제품 정의·MVP 범위·데이터 정의
- [features/](features/README.md) — 화면별 기능 명세. 플랫폼 무관한 서비스 기능 축이며, 각 문서가 자족적이라 해당 화면 파일만 골라 읽으면 됩니다
- [architecture/](architecture/) — 여러 저장소에 걸친 시스템 구조(데이터 파이프라인, API 계약)
- 지금 작업 중인 저장소에 대응하는 `docs/<약칭>/` 폴더 (front · console · core · etl · mcp 중 하나)

## 필요할 때만 참고

- [product/business-feature-roadmap.md](product/business-feature-roadmap.md) — 기능 후보와 우선순위
- [product/ia.md](product/ia.md) — 사이트맵·내비게이션 구조
- [research/](research/) — 사용자·시장 리서치 근거
- [meta/](meta/) — Constitution, 정본 목록, 문서 작성 규칙
- 다른 저장소의 `docs/<다른 약칭>/` 폴더는 명시적으로 요청받았을 때만 참고합니다

## 정본 판단

어떤 정보의 정본이 어느 문서인지는 [meta/SSOT.md](meta/SSOT.md)를 따릅니다.
