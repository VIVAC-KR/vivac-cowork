# 심볼릭 링크 설정 — 각 repo에서 vivac-cowork 문서 참조하기

`vivac-cowork`의 `docs/` 폴더를 작업 중인 repo의 `docs/` 자리에 심볼릭 링크로 연결하면, 그 repo 안에서 바로 공유 기획 문서(`PRODUCT.md` 등)와 해당 repo 전용 문서(`docs/<repo 약칭>/`)를 함께 참조할 수 있습니다.

| repo | 약칭 폴더 |
|---|---|
| `VIVAC-frontend` | `docs/front/` |
| `vivac-console` | `docs/console/` |
| `vivac-infra` | `docs/infra/` |
| `vivac-ios` | `docs/ios/` |
| `vivac-mcp` | `docs/mcp/` |
| `vivacapi-core` | `docs/core/` |
| `vivacapi-etl` | `docs/etl/` |

`vivac-infra`, `vivac-ios`는 아직 이 repo에 문서가 없어 `docs/infra/`, `docs/ios/` 폴더가 존재하지 않습니다. `docs/` 심볼릭 링크 자체는 문제없이 걸리며, 첫 문서를 그 repo용으로 작성하는 순간 폴더가 생깁니다(`.claude/rules/vivac-docs-authoring.md` 원칙상 빈 폴더를 미리 만들어두지 않습니다).

`vivac-cowork` 저장소 루트 전체가 아니라 **`docs/` 폴더만** 공유됩니다. `CLAUDE.md`, `README.md` 등 `vivac-cowork` 저장소 자체에 대한 설명이나 기획 작업용 파일은 이 링크에 포함되지 않고, 각 개발 repo에는 노출되지 않습니다.

`VIVAC-frontend`는 여기에 더해 Spec Kit 설정이 필요합니다 — 8절을 참고하세요. 다른 repo에는 해당하지 않습니다.

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
| vivac-infra | `~/CursorProjects/vivac/vivac-infra` | `ln -s ~/Documents/Claude/Projects/vivac-cowork/docs docs` |
| vivac-ios | `~/CursorProjects/vivac/vivac-ios` | `ln -s ~/Documents/Claude/Projects/vivac-cowork/docs docs` |
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

## 8. Spec Kit 설정 (VIVAC-frontend 전용)

`vivac-cowork`는 SSOT 문서 저장소로만 사용하며 Spec Kit을 두지 않습니다. SDD 워크플로는 이를 도입한 구현 저장소에서만 돕니다 — 2026-09-07 기준 **`VIVAC-frontend` 한 곳**입니다. 다른 repo에는 설치하지 않습니다.

Spec Kit은 명세(`specs/`)와 코드가 같은 repo에 있다고 전제하는 도구입니다. `specs/`는 `docs/` 바깥이라 이 문서의 심볼릭 링크로는 공유되지 않으며, 공유할 필요도 없습니다. 저장소를 넘는 확정 계약은 `docs/product/features/`에 있고, `specs/`는 그 repo의 변경 단위만 담습니다.

### 8.1 선행 조건

`docs` 심볼릭 링크가 먼저 걸려 있어야 합니다(위 2절). 아래 Constitution 연결이 그 링크를 타고 들어가기 때문입니다.

### 8.2 설치

`vivac-cowork`에 설치했을 때 사용한 옵션은 아래와 같습니다. 같은 값으로 맞춥니다.

| 항목 | 값 |
|---|---|
| `speckit_version` | `1.0.4` |
| `ai` / `integration` | `claude` |
| `ai_skills` | `true` (`/speckit-*` 스킬 방식) |
| `script` | `sh` |
| `feature_numbering` | `sequential` |
| `here` | `true` (현재 폴더에 설치) |

설치 후 생성되는 `.specify/init-options.json`이 위 표와 일치하는지 확인합니다. 다르면 재설치하거나 값을 맞춥니다.

`.specify/`와 `specs/`는 git에 커밋합니다. `.specify/.gitignore`가 체크아웃별 상태(`feature.json` 등)를 알아서 제외합니다.

### 8.3 Constitution 연결

Constitution은 `vivac-cowork/docs/meta/constitution.md`가 정본입니다. Spec Kit은 이를 `$REPO_ROOT/.specify/memory/constitution.md`에서 읽으므로 심볼릭 링크로 연결합니다.

```bash
mkdir -p .specify/memory
ln -s ../../docs/meta/constitution.md .specify/memory/constitution.md
```

**이 링크는 4절과 반대로 git에 커밋합니다.** `docs`나 `.claude/rules/...`와 달리 repo 내부 상대경로라 clone 위치와 무관하게 동작하기 때문입니다. 커밋해두면 사람마다 따로 설정할 필요가 없습니다.

복사하지 않고 링크하는 이유는 3절과 같습니다 — 원본이 `vivac-cowork`에 하나뿐이어야 Constitution이 갈라지지 않습니다. 이는 Constitution 원칙 V가 요구하는 바이기도 합니다.

`docs` 링크가 없으면 이 링크는 끊긴 상태가 됩니다. worktree를 새로 만들 때는 6절의 `.worktreeinclude`가 `docs`를 함께 복사하는지 확인하세요.

### 8.4 기능 이름 규칙

기능 폴더 이름은 영문 설명에서만 슬러그가 만들어집니다. **한글로 설명할 때는 `--short-name`을 반드시 함께 지정합니다.** 생략하면 `specs/001-`처럼 이름 없는 폴더가 생깁니다.

```bash
/speckit-specify --short-name map-explore "지도에서 스팟을 탐색하는 기능"
```

### 8.5 확인

```bash
ls -la .specify/memory/constitution.md        # 심볼릭 링크인지 확인
head -3 .specify/memory/constitution.md       # 내용이 읽히는지 확인 (끊겼으면 실패)
cat .specify/init-options.json                # 8.2 표와 일치하는지 확인
```

경로 생성만 미리 확인하려면 실제 파일을 만들지 않는 dry-run을 씁니다.

```bash
bash .specify/scripts/bash/create-new-feature.sh --dry-run --short-name map-explore "지도 탐색"
# BRANCH_NAME: 001-map-explore
# SPEC_FILE:   <repo>/specs/001-map-explore/spec.md
```
