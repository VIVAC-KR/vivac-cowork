# 공유 컨텍스트 참고 범위

`docs/`는 `vivac-cowork` 저장소의 `docs/` 폴더로 가는 심볼릭 링크입니다.

- 기본으로 참고: `docs/PRODUCT.md`, `docs/features/`, `docs/data-pipeline.md`, `docs/business-feature-roadmap.md`, 그리고 지금 작업 중인 repo에 대응하는 `docs/<약칭>/` 폴더 (front/console/mcp/core/etl 중 하나)
- `docs/features/`는 화면별 기능 명세입니다. 플랫폼 무관한 서비스 기능 축이며, 필요한 기능 파일만 골라 읽습니다.
- 다른 repo의 `docs/<다른 약칭>/` 폴더는 명시적으로 요청받았을 때만 참고합니다.
- 문서 작성 규칙(`.claude/rules/vivac-docs-authoring.md`)은 연결해뒀다면 문서를 쓸 때 자동으로 로드되므로 따로 참고할 필요는 없습니다.
