# 심볼릭 링크 설정 — 각 repo에서 vivac-cowork 문서 참조하기

`vivac-cowork`의 `docs/` 폴더를 작업 중인 repo의 `docs/` 자리에 심볼릭 링크로 연결하면, 그 repo 안에서 바로 공유 기획 문서(`PRODUCT.md` 등)와 해당 repo 전용 문서(`docs/<repo 약칭>/`)를 함께 참조할 수 있습니다.

| repo | 약칭 폴더 |
|---|---|
| `VIVAC-frontend` | `docs/front/` |
| `vivac-console` | `docs/console/` |
| `vivac-mcp` | `docs/mcp/` |
| `vivacapi-core` | `docs/core/` |
| `vivacapi-etl` | `docs/etl/` |

`vivac-cowork` 저장소 루트 전체가 아니라 **`docs/` 폴더만** 공유됩니다. `CLAUDE.md`, `README.md` 등 `vivac-cowork` 저장소 자체에 대한 설명이나 기획 작업용 파일은 이 링크에 포함되지 않고, 각 개발 repo에는 노출되지 않습니다.

## 1. 기존 로컬 docs 정리

작업할 repo에 이미 자체 `docs/` 폴더가 있다면(대부분 있습니다), 그 안의 md 문서가 이미 `vivac-cowork`의 `docs/<repo 약칭>/`로 옮겨졌는지 먼저 확인합니다. 옮겨졌다면 로컬 `docs/`는 삭제해도 안전합니다.

```bash
cd <작업할 repo 경로>
git rm -r docs   # 이미 vivac-cowork로 옮겨진 문서라면 안전하게 삭제
git commit -m "docs: remove local docs, now managed in vivac-cowork"
```

아직 옮기지 않은 문서가 있다면, 먼저 그 내용을 `vivac-cowork`의 `docs/<repo 약칭>/`로 옮긴 뒤 진행합니다.

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

## 3. 문서 작성 규칙도 자동으로 로드되게 걸기 (권장)

`vivac-cowork`의 `.claude/rules/vivac-docs-authoring.md`에는 카테고리 폴더 선택 기준, 파일명, 톤, 문서별 템플릿이 정리돼 있습니다. 이 파일은 `docs/`가 아니라 `.claude/rules/` 밑에 있어 `docs` 심볼릭 링크만으로는 안 딸려옵니다 — 같은 위치(`.claude/rules/`)에 한 번 더 심볼릭 링크를 걸어야 합니다. 이렇게 두면 Claude가 `docs/**/*.md`를 다룰 때마다 이 규칙이 자동으로 컨텍스트에 로드됩니다(파일 안 `paths:` frontmatter로 스코프가 걸려 있습니다).

```bash
mkdir -p .claude/rules
ln -s "<vivac-cowork 경로>/.claude/rules/vivac-docs-authoring.md" .claude/rules/vivac-docs-authoring.md
```

## 4. git에 커밋되지 않도록 처리

절대경로 심볼릭 링크는 clone 위치가 사람마다 달라, 커밋하면 다른 환경에서 깨집니다. 각 repo의 `.gitignore`에 두 줄을 추가합니다.

```
docs
.claude/rules/vivac-docs-authoring.md
```

## 5. 확인

```bash
ls -la docs                                        # 심볼릭 링크인지 확인 (화살표로 표시됨)
cat docs/PRODUCT.md                                # 공유 문서가 읽히는지 확인
ls docs/<repo 약칭>/                               # 해당 repo 전용 문서 폴더 확인
ls -la .claude/rules/vivac-docs-authoring.md       # 규칙 심볼릭 링크 확인
```

## 6. worktree에서도 자동으로 걸리게 하기 (권장)

Claude Code에서 `git worktree`를 새로 만들면 gitignore된 파일(`docs`, `.claude/rules/vivac-docs-authoring.md` 심볼릭 링크 포함)은 기본적으로 새 worktree에 복사되지 않습니다. repo 루트에 `.worktreeinclude` 파일을 만들고 아래처럼 적어두면, gitignore된 파일 중 이 패턴에 매칭되는 것만 새 worktree 생성 시 자동으로 복사됩니다(심볼릭 링크는 링크 그대로 복사되어 같은 `vivac-cowork` 경로를 계속 가리킵니다).

```
docs
.claude/rules/vivac-docs-authoring.md
```

`.claude/settings.json`의 `worktree.symlinkDirectories`는 여기 쓰지 않습니다 — 그건 같은 repo 안 원본 worktree의 디렉터리를 라이브로 공유하는 용도(예: `node_modules`)라 성격이 다르고, 쓰기 시 심볼릭 링크가 일반 파일로 바뀌거나 worktree 정리가 실패하는 알려진 버그가 있습니다.

## 7. CLAUDE.md에서 참조 범위 명시 (권장)

에이전트가 필요 없는 다른 repo 문서까지 훑어 컨텍스트를 낭비하지 않도록, 각 repo의 CLAUDE.md에 아래 한 줄을 추가해 `docs/CONTEXT_SCOPE.md`를 import합니다.

```markdown
@docs/CONTEXT_SCOPE.md
```

내용을 복붙하지 않고 import로 참조하는 이유는 `.claude/rules/vivac-docs-authoring.md`와 같습니다 — 원본이 `vivac-cowork`에 하나뿐이라, 참고 범위 안내를 고쳐도 각 repo의 CLAUDE.md를 따로 손댈 필요가 없습니다.

`docs/`가 심볼릭 링크라 이 import는 "외부 경로"로 취급됩니다. repo마다 처음 한 번 Claude Code가 승인 다이얼로그를 띄우고, 승인하면 그다음부터는 자동으로 로드됩니다.
