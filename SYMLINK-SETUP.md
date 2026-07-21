# 심볼릭 링크 설정 — 각 repo에서 vivac-cowork 문서 참조하기

`vivac-cowork`의 `docs/` 폴더를 작업 중인 repo의 `docs/` 자리에 심볼릭 링크로 연결하면, 그 repo 안에서 바로 공유 기획 문서(`PRODUCT.md` 등)와 해당 repo 전용 문서(`docs-<repo명>/`)를 함께 참조할 수 있습니다.

`vivac-cowork` 저장소 루트 전체가 아니라 **`docs/` 폴더만** 공유됩니다. `CLAUDE.md`, `README.md` 등 `vivac-cowork` 저장소 자체에 대한 설명이나 기획 작업용 파일은 이 링크에 포함되지 않고, 각 개발 repo에는 노출되지 않습니다.

## 1. 기존 로컬 docs 정리

작업할 repo에 이미 자체 `docs/` 폴더가 있다면(대부분 있습니다), 그 안의 md 문서가 이미 `vivac-cowork`의 `docs/docs-<repo명>/`로 옮겨졌는지 먼저 확인합니다. 옮겨졌다면 로컬 `docs/`는 삭제해도 안전합니다.

```bash
cd <작업할 repo 경로>
git rm -r docs   # 이미 vivac-cowork로 옮겨진 문서라면 안전하게 삭제
git commit -m "docs: remove local docs, now managed in vivac-cowork"
```

아직 옮기지 않은 문서가 있다면, 먼저 그 내용을 `vivac-cowork`의 `docs/docs-<repo명>/`로 옮긴 뒤 진행합니다.

## 2. 링크 걸기

```bash
ln -s "<vivac-cowork 경로>/docs" docs
```

`<vivac-cowork 경로>` 자리에는 이 저장소를 clone한 절대경로를 넣습니다.

예시 (이 워크스페이스 기준 — 경로는 사람마다 다를 수 있으니 본인 환경에 맞게 바꿔서 사용합니다):

| repo | 실행 위치 | 명령 |
|---|---|---|
| VIVAC-frontend | `~/CursorProjects/vivac/VIVAC-frontend` | `ln -s ~/Documents/Claude/Projects/vivac-cowork/docs docs` |
| vivac-console | `~/CursorProjects/vivac/vivac-console` | `ln -s ~/Documents/Claude/Projects/vivac-cowork/docs docs` |
| vivac-mcp | `~/CursorProjects/vivac/vivac-mcp` | `ln -s ~/Documents/Claude/Projects/vivac-cowork/docs docs` |
| vivacapi-core | `~/CursorProjects/vivac/vivacapi-core` | `ln -s ~/Documents/Claude/Projects/vivac-cowork/docs docs` |
| vivacapi-etl | `~/CursorProjects/vivac/vivacapi-etl` | `ln -s ~/Documents/Claude/Projects/vivac-cowork/docs docs` |

## 3. git에 커밋되지 않도록 처리

절대경로 심볼릭 링크는 clone 위치가 사람마다 달라, 커밋하면 다른 환경에서 깨집니다. 각 repo의 `.gitignore`에 한 줄을 추가합니다.

```
docs
```

## 4. 확인

```bash
ls -la docs                          # 심볼릭 링크인지 확인 (화살표로 표시됨)
cat docs/PRODUCT.md                  # 공유 문서가 읽히는지 확인
ls docs/docs-<repo명>/               # 해당 repo 전용 문서 폴더 확인
```

## 5. CLAUDE.md에서 참조 범위 명시 (권장)

에이전트가 필요 없는 다른 repo 문서까지 훑어 컨텍스트를 낭비하지 않도록, 각 repo의 CLAUDE.md에 기본 참조 범위를 적어두는 것을 권장합니다.

```markdown
## 공유 컨텍스트

`docs/`는 `vivac-cowork` 저장소의 `docs/` 폴더로 가는 심볼릭 링크입니다.

- 기본으로 참고: `docs/PRODUCT.md`, `docs/docs-<repo명>/`
- 다른 repo 맥락(`docs/docs-<다른repo명>/`)은 명시적으로 요청받았을 때만 참고합니다.
```

`<repo명>` 자리에는 실제 폴더 이름(`vivac-console`, `vivacapi-core` 등)을 넣습니다.
