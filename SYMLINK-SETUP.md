# 심볼릭 링크 설정 — 각 repo에서 vivac-cowork 문서 참조하기

`vivac-cowork`를 작업 중인 repo에 심볼릭 링크로 연결하면, 그 repo 안에서 바로 공유 기획 문서(PRODUCT.md 등)와 해당 repo 전용 문서(`docs/docs-<repo명>/`)를 함께 참조할 수 있습니다.

## 1. 링크 걸기

작업할 repo 루트에서 다음 명령을 실행합니다. `<vivac-cowork 경로>` 자리에는 이 저장소를 clone한 절대경로를 넣습니다.

```bash
cd <작업할 repo 경로>
ln -s "<vivac-cowork 경로>" .context
```

예시 (이 워크스페이스 기준 — 경로는 사람마다 다를 수 있으니 본인 환경에 맞게 바꿔서 사용합니다):

| repo | 실행 위치 | 명령 |
|---|---|---|
| VIVAC-frontend | `~/CursorProjects/vivac/VIVAC-frontend` | `ln -s ~/Documents/Claude/Projects/vivac-cowork .context` |
| vivac-console | `~/CursorProjects/vivac/vivac-console` | `ln -s ~/Documents/Claude/Projects/vivac-cowork .context` |
| vivac-mcp | `~/CursorProjects/vivac/vivac-mcp` | `ln -s ~/Documents/Claude/Projects/vivac-cowork .context` |
| vivacapi-core | `~/CursorProjects/vivac/vivacapi-core` | `ln -s ~/Documents/Claude/Projects/vivac-cowork .context` |
| vivacapi-etl | `~/CursorProjects/vivac/vivacapi-etl` | `ln -s ~/Documents/Claude/Projects/vivac-cowork .context` |

## 2. git에 커밋되지 않도록 처리

절대경로 심볼릭 링크는 clone 위치가 사람마다 달라, 커밋하면 다른 환경에서 깨집니다. 각 repo의 `.gitignore`에 한 줄을 추가합니다.

```
.context
```

## 3. 확인

```bash
ls -la .context                          # 링크 경로 확인
cat .context/PRODUCT.md                  # 공유 문서가 읽히는지 확인
ls .context/docs/docs-<repo명>/          # 해당 repo 전용 문서 폴더 확인
```

## 4. CLAUDE.md에서 참조 범위 명시 (권장)

에이전트가 필요 없는 다른 repo 문서까지 훑어 컨텍스트를 낭비하지 않도록, 각 repo의 CLAUDE.md에 기본 참조 범위를 적어두는 것을 권장합니다.

```markdown
## 공유 컨텍스트

`.context/`는 `vivac-cowork` 저장소로 가는 심볼릭 링크입니다.

- 기본으로 참고: `.context/PRODUCT.md`, `.context/docs/docs-<repo명>/`
- 다른 repo 맥락(`.context/docs/docs-<다른repo명>/`)은 명시적으로 요청받았을 때만 참고합니다.
```

`<repo명>` 자리에는 실제 폴더 이름(`vivac-console`, `vivacapi-core` 등)을 넣습니다.
