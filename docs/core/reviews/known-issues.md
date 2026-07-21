# 알려진 문서 이슈 (vivacapi-core docs)

> vivacapi-core `docs/audit-2026-07-20/08-cross-cutting-issues.md`(2026-07-20, docs 15개 전수 감사)에서 발견된 항목 중, 이 저장소로 취합하면서 아직 유효한 것만 정리했다. 원본 감사 문서 자체(00~07번 파일)는 개별 문서를 보존 수준으로 재구성한 중복 산출물이라 이 저장소로는 옮기지 않았다 — 원본 소스 문서([architecture.md](../architecture.md), [backlog.md](../backlog.md), [projects/](../projects/) 등)가 정본이다.

## 이번 취합 과정에서 반영한 것 (더 이상 열린 이슈 아님)

- **auth rate limit 중복 기재** — [backlog.md](../backlog.md)의 일반 항목이 [backlog/auth-rate-limit-260711.md](../backlog/auth-rate-limit-260711.md)(더 상세)와 중복이었음. backlog.md 쪽을 링크 참조로 대체.
- **`spot-invites.md` ↔ 비즈니스 로드맵 1.1의 결정 역전** — [projects/spot-invites.md](../projects/spot-invites.md)의 "1회용, 재사용 불가" 결정이 [business-feature-roadmap.md](../../business-feature-roadmap.md) 1.1(2026-07-20)에서 일반 리퍼럴에 한해 뒤집혔음에도 본문에 반영이 안 돼 있었음. 각주 추가.
- **`vivac-console-frontend.md` 낡음 경고 누락** — 짝 문서 [projects/vivac-console-backend.md](../projects/vivac-console-backend.md)는 "실제 경로는 `/v1/admin/*`이 아니라 `/v1/internal/*`" 경고가 있었는데 frontend.md에는 없었음. 동일 경고 추가.

## 아직 열려 있는 문서 갭 (원본 문서 자체는 수정하지 않음, 참고용 기록)

1. **`skill-db-inspect.md` 미이관** — `.claude/skills/db_inspect/SKILL.md`로 옮길 초안인데, vivacapi-core 저장소의 `.claude/skills/`에 해당 디렉터리가 없음(2026-07-20 확인). 이관되거나 보류 결정이 나면 갱신 필요.
2. **`test-setup.md`와 `.claude/rules/testing.md`의 우선순위 불명** — test-setup.md(2026-05-02)가 도입한 픽스처·규칙을 `.claude/rules/testing.md`가 흡수해 정식화했는데, 어느 쪽이 1차 참고 문서인지 명시가 없음.
3. **`architecture.md`/`erd.md`에 최종 갱신일 없음** — "living document"라고만 표기돼 있어, 실제 코드와 얼마나 어긋났는지 문서만 보고는 알 수 없음.
4. **샘플 JSON 미참조** — vivacapi-core `docs/samples/spots_bulk_sample.json`, `spot_business_info_bulk_sample.json`을 [projects/spot-bulk-and-admin.md](../projects/spot-bulk-and-admin.md) 등 관련 문서 어디에서도 링크하지 않음.
5. **`vvc-105-explore-api-spec.md` 후속 이슈 상태 불명** — VVC-117(필터)/118(이미지필드)/119(정렬확정) 완료 여부가 원문서 범위 밖으로 명시돼 있는데, [projects/spot-search-postgres-fts.md](../projects/spot-search-postgres-fts.md)가 사실상 VVC-119 관련 정렬 로직을 구현했음에도 원 스펙 문서에 역참조가 없음.

## 코드 대조 검증 결과 (2026-07-20 기준, 5건 샘플 확인 — 전수 아님)

| 항목 | 문서 주장 | 코드 확인 결과 |
|---|---|---|
| [backlog/private-image-exposure-260711.md](../backlog/private-image-exposure-260711.md) | `list_images_by_spot`에 `is_public` 필터 없음 | ✅ 확인, **미해결** |
| [backlog/prod-allowed-email-domain-260711.md](../backlog/prod-allowed-email-domain-260711.md) | `_validate_prod_requirements`가 `ALLOWED_EMAIL_DOMAIN` 안 검사 | ✅ 확인, **미해결** |
| [backlog/pipeline-status-index-260711.md](../backlog/pipeline-status-index-260711.md) | `pipeline_status`는 partial index뿐 | ✅ 확인, **미해결** |
| [skill-db-inspect.md](../skill-db-inspect.md) | `.claude/skills/db_inspect/SKILL.md`로 이관 예정 | ✅ 확인, **미이관** |
| [backlog.md](../backlog.md) 이미지 인프라 항목 | `S3_BUCKET`/`CDN_BASE_URL` 미설정 | ✅ 확인, **여전히 미설정** |

이 검증은 2026-07-20 시점 vivacapi-core 코드 기준이다 — 이후 코드가 바뀌었다면 재확인 필요.
