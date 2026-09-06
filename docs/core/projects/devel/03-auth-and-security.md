# 인증 & 보안

> 소스: `vivacapi/core/deps.py`, `core/security.py`, `core/config.py`, `.claude/rules/security.md`, 2026-07-11 보안 점검(backlog), 2026-05-02 DB 보안 리뷰.

## 1. 인증 3가지 흐름

모두 Google ID Token 검증에서 시작하지만 이후 발급되는 토큰/세션 종류가 다르다.

| 흐름 | 엔드포인트 | 토큰 | 만료 |
|---|---|---|---|
| 앱 사용자 | `POST /v1/auth/google` | HS256 JWT access + refresh | access 30분 / refresh 7일 |
| vivac-console (staff) | `POST /v1/admin/auth/google` | HS256 JWT access(+`email`, `is_staff` 클레임) | 8시간 |
| SQLAdmin (`/admin`) | 로그인 폼 → `AdminAuth` | 서명된 세션 쿠키 | 세션(현재 기본값 14일) |

- JWT payload: `sub`(user uid), `type`(`access`/`refresh`), `iat`, `exp`.
- refresh 토큰은 **완전 stateless** — 서버에 저장/회수 수단이 없다. 유출 시 만료 전까지 유효한 것을 알고 선택한 트레이드오프(운영 단순성 우선). 회수가 필요해지면 refresh 토큰만 DB에 `jti`를 저장하는 방식으로 전환한다.
- staff 판정은 **매 요청 DB 재조회**(`require_staff`) — 토큰의 `is_staff` 클레임은 표시용이며 권한 판단에 쓰지 않는다.
- staff 로그인은 `verify_staff_google_login`(`core/deps.py`)에서 다음을 모두 통과해야 한다:
  1. Google ID Token 유효성
  2. `ALLOWED_EMAIL_DOMAIN` 화이트리스트(설정된 경우만)
  3. DB에 해당 email로 가입된 유저 존재
  4. `User.is_staff == True`
  5. `User.is_active == True`

## 2. 의존성 계층 (`core/deps.py`)

```
get_current_user          Bearer JWT 검증, 없으면 401, is_active=False면 403
get_current_user_optional  헤더 없으면 None, 있으면 get_current_user와 동일 검증
                            (비로그인도 허용하는 공개 조회에서 로그인 시 추가 정보 노출용)
require_staff              get_current_user + is_staff 체크, 아니면 403
require_role(min_role)     require_staff 위에 얹어 staff_role 등급 비교, 미달 시 403
```

`CurrentStaff = Annotated[User, Depends(require_staff)]` 타입 alias로 라우터에서 재사용한다.

`_STAFF_ROLE_RANK`로 등급 비교를 한다 — `StrEnum`은 선언 순서가 아니라 값(str)으로 비교되므로 별도 랭크 dict가 필요하다.

## 3. staff 권한 등급 (`StaffRole`)

`User.is_staff`(bool)는 콘솔 접근 여부의 큰 게이트로 그대로 두고, 그 안에서 세부 권한은 `User.staff_role`(`STAFF < MANAGER < SUPERUSER`, 기본값 `STAFF`)로 나눈다. 라우터 단위 `require_staff`(coarse gate) 위에, 등급 제한이 필요한 개별 엔드포인트에만 `Depends(require_role(StaffRole.XXX))`를 얹는다.

### 현재 등급 매핑

| 등급 | 엔드포인트 | 이유 |
|---|---|---|
| **MANAGER 이상** | `POST /v1/internal/spots/assignments` | 타 staff에게 검증 작업 할당 |
| **MANAGER 이상** | `PATCH /v1/internal/spots/assignments`, `POST /v1/internal/spots/assignments/transfer`, `PATCH /v1/internal/spots/{uid}/assignment` | 검증 담당자 재할당/이전 |
| **MANAGER 이상** | `DELETE /v1/internal/spots/{uid}`, `POST /v1/internal/spots/{uid}/restore` | 소프트 삭제/복구 |
| **MANAGER 이상** | `POST/DELETE /v1/internal/spot-options`, `/{field}/{code}` | 화이트리스트 옵션값 추가/삭제 |
| **MANAGER 이상** | `DELETE /v1/internal/groups/{uid}` | 그룹 삭제(비가역) |
| **MANAGER 이상** | `POST/PATCH/DELETE /v1/internal/groups/{uid}/members/*` | 임의 유저에게 `owner`까지 포함한 역할 강제 부여/박탈 — 권한 상승 리스크 |
| **SUPERUSER** | `POST /v1/internal/spots/bulk` | 최대 5000행 파괴적 upsert |
| STAFF만 | 그 외 internal 엔드포인트 | spot group 조회/메타 수정/단일 spot 제거, spot-business-info bulk 포함 |

- `/admin`(SQLAdmin)은 아직 `staff_role`을 반영하지 않는다 — 모든 staff가 다른 사용자의 `is_staff`/`is_active`를 토글할 수 있는 단일 평면이다. SQLAdmin까지 등급을 반영하려면 별도 작업이 필요하다.
- `/admin`에서 사용자 생성/삭제는 막혀 있다(`can_create/can_delete = False`) — 계정 생성은 Google 로그인 흐름만이 유일한 경로다.
- `staff_role`을 `SUPERUSER`로 올리는 API는 아직 없다 — DB에서 직접 값을 바꾸거나 SQLAdmin으로 부여한다(bootstrap 단계의 의도된 공백).

## 4. 이미지 `is_public`의 의미

`spot_images.is_public`은 **서빙 방식 구분**(True=CDN URL, False=presigned URL)이지 접근 제어가 아니다. 코드 상 두 경우 모두 공개 API에 노출된다. 외부 비노출 이미지가 필요해지면 별도 접근 제어 필드를 도입해야 한다. 현재 이게 실제로 보안 이슈로 이어지는 사례는 6장 참고.

## 5. 설정 검증 (`core/config.py`)

`pydantic-settings` 기반. `ENVIRONMENT=prod`면 부팅 시 `_validate_prod_requirements`가 다음을 검증해 실패 시 부팅 자체를 막는다.

- placeholder 값 사용 여부
- 32자 미만의 약한 `JWT_SECRET_KEY`/`ADMIN_SESSION_SECRET`
- 잘못된 `CORS_ALLOWED_ORIGINS`(미설정/localhost 포함/`*` 포함)
- 잘못된 `DB_HOST`

시크릿 필드는 `SecretStr`로 감싸 repr/직렬화에서 마스킹된다.

| 변수 | 설명 | 기본값 |
|---|---|---|
| `ENVIRONMENT` | local/dev/prod | `local` |
| `DB_HOST`/`DB_PORT`/`DB_NAME`/`DB_USER`/`DB_PASSWORD` | DB 접속 정보 | host/port만 기본값 |
| `GOOGLE_CLIENT_ID` | Google OAuth Client ID | 필수 |
| `ALLOWED_EMAIL_DOMAIN` | 어드민 로그인 허용 이메일 도메인 | 없음(제한 안 함) |
| `JWT_SECRET_KEY` | JWT 서명 키 (prod 32자 이상) | 필수 |
| `JWT_ALGORITHM` | JWT 알고리즘 | `HS256` |
| `JWT_ACCESS_TOKEN_EXPIRE_MINUTES` | access 만료(분) | `30` |
| `JWT_REFRESH_TOKEN_EXPIRE_DAYS` | refresh 만료(일) | `7` |
| `JWT_ADMIN_ACCESS_TOKEN_EXPIRE_HOURS` | 어드민 access 만료(시간) | `8` |
| `ADMIN_SESSION_SECRET` | SQLAdmin 세션 키 (JWT 키와 분리, prod 32자 이상) | 필수 |
| `CORS_ALLOWED_ORIGINS` | 허용 origin (콤마 구분) | local만 localhost:3000 |
| `AWS_REGION` | S3 리전 | `ap-northeast-2` |
| `S3_BUCKET` / `S3_ENDPOINT_URL` / `CDN_BASE_URL` | 이미지 스토리지 (미설정 시 이미지 API 503) | 없음 |
| `S3_PRESIGN_EXPIRE_SECONDS` | presigned URL 만료(초) | `3600` |

## 6. 알려진 보안 이슈 (2026-07-11 점검, 미해결분)

| 심각도 | 이슈 | 위치 | 상세 |
|---|---|---|---|
| 🟠 중간 | 비공개 이미지가 공개 API로 노출 | `crud/spot_image.py::list_images_by_spot` | `GET /v1/explore/spots/{uid}/images`(비로그인 가능)가 `is_public=False` 이미지도 presigned GET URL로 반환한다 — 필터 부재. 수정: 공개 endpoint는 `is_public=True`만 반환하고, staff 전용 `/v1/internal/spots/{uid}/images` 전체 조회 엔드포인트를 신설해야 vivac-console의 비공개 이미지 조회가 유지된다 |
| 🟠 중간 | SQLAdmin 세션 쿠키 하드닝 부재 | `vivacapi/main.py`, `admin/auth.py` | `SessionMiddleware` 기본값 — `https_only=False`, `max_age` 14일. admin JWT(8h)와 불일치. `https_only=True`, `same_site="strict"`, `max_age=8*3600`으로 override 필요 |
| 🟡 낮음 | prod에서 `ALLOWED_EMAIL_DOMAIN` 미설정 허용 | `core/config.py:_validate_prod_requirements` | 도메인 화이트리스트가 prod에서 조용히 꺼진 채 배포될 수 있음. DB `is_staff` 체크가 최종 방어선이라 직접 우회는 아니지만 defense-in-depth 공백 |
| 🟡 낮음 | auth 엔드포인트 rate limit 부재 | `endpoints/auth.py`, `admin_auth.py` | `/v1/auth/google`, `/v1/auth/refresh`, `/v1/admin/auth/google`에 무제한 요청 가능. 토큰 brute-force는 비현실적이나 Google API 호출 유발/계정 enumeration/DoS 여지. reverse proxy(CloudFront/nginx) 레벨 rate limit로 해결 권장 |
| 🟡 낮음 | `deploy.yml` tag 이름 셸 injection 패턴 | `.github/workflows/deploy.yml:88,121-123` | git tag 이름이 SSH 스크립트에 `${{ }}` 문자열 치환으로 삽입됨. tag push 권한자로 실효는 한정적이나 GitHub Actions의 대표적 injection 패턴. `envs:`로 환경변수 전달 + `"$IMAGE"` 참조로 수정 필요 |

## 7. DB 보안 리뷰 이력 (2026-05-02, 대부분 해결됨)

| 항목 | 상태 | 비고 |
|---|---|---|
| 정수 PK 열거 가능성 | ✅ 해결 | 전 테이블 shortuuid `VARCHAR(22)` PK로 전환 |
| email 대소문자 민감성 | ✅ 해결 | 소문자 정규화 + `lower(email)` unique index |
| PII 감사 로그 부재 | ✅ 부분 해결 | `audit_log` + 트리거 도입(spots/spot_business_info만, users는 미적용) |
| picture URL 검증 누락 | ⏸ 보류 | Google OAuth 클레임에서만 저장(사용자 입력 경로 없음) — 위험 낮음으로 판단 |
| DB 자격증명 처리 | ✅ 해결 | `SecretStr` 적용, `database_url` 직렬화 제외 |
| `expire_on_commit` 부수효과 | ✅ 해당 없음 | 권한 분기는 요청마다 새 세션에서 재조회 |

전체 상세는 저장소 `docs/core/security/db-security-review-2026-05-02.md` 원문 참고.

## 8. 보안 관련 코드 규칙 요약

- 새 도메인 에러는 `HTTPException`을 직접 던지지 말고 `AppException` + `ErrorCode`를 사용한다.
- 사용자 입력으로 ORM 속성을 동적으로 고를 때(정렬/필터 컬럼)는 반드시 명시적 dict 화이트리스트를 거친다. 임의 속성 문자열을 바로 `getattr`하지 않는다 — 검증되지 않은 입력이 DB 컬럼에 직접 매핑되는 것을 막기 위함이다.
- 이미지 업로드는 서버가 S3 키를 생성(`spots/{uid}/{shortuuid}{ext}`)하므로 클라이언트가 임의 경로에 쓸 수 없다. 등록 시에도 `s3_key`가 해당 spot 하위 경로인지 검증하고 `head_object`로 실제 업로드 여부를 재확인한다.
