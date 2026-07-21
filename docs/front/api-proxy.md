# Next.js API 프록시 설정

## 왜 프록시가 필요한가

브라우저에서 FastAPI 백엔드를 직접 호출하면 두 가지 문제가 생깁니다.

1. **CORS**: 도메인이 다르면 브라우저가 요청을 차단
2. **백엔드 IP 노출**: `NEXT_PUBLIC_*` 환경변수는 번들에 인라인되어 누구나 볼 수 있음

해결책: **브라우저 → Next.js 서버 → FastAPI** 구조로 백엔드 호출을 서버 사이드에서 중계합니다.

## 요청 흐름

```
브라우저
  → https://vivac.app/api/v1/auth/google   (HTTPS)
  → nginx (포트 80)
  → Next.js web:3000
  → /api/[...path]/route.ts  (프록시 라우트)
  → http://<API_PROXY_ORIGIN>/v1/auth/google  (내부 네트워크)
  → FastAPI (Lightsail)
```

## 프록시 라우트

`apps/web/src/app/api/[...path]/route.ts`

- `/api/v1/...` 요청을 받아 `/api` prefix를 제거한 뒤 `API_PROXY_ORIGIN`으로 포워딩
- `hop-by-hop` 헤더 필터링 (`transfer-encoding`, `connection` 등)
- `req.arrayBuffer()`로 요청 바디 읽기 (Next.js App Router 호환)
- fetch 실패 시 502 응답 + 콘솔 에러 로그

## Next.js 환경변수 두 종류

### 빌드 타임 — `NEXT_PUBLIC_*`

```dockerfile
ARG NEXT_PUBLIC_GOOGLE_CLIENT_ID
ENV NEXT_PUBLIC_GOOGLE_CLIENT_ID=$NEXT_PUBLIC_GOOGLE_CLIENT_ID
```

- **빌드 시 번들에 인라인** → 브라우저 JS에 포함되어 누구나 볼 수 있음
- 변경하려면 **반드시 재빌드** 필요
- GHA에서 `--build-arg`로 주입

| 변수 | 설명 |
|------|------|
| `NEXT_PUBLIC_GOOGLE_CLIENT_ID` | Google OAuth 클라이언트 ID |
| `NEXT_PUBLIC_API_URL` | (현재 미사용, apiClient는 `/api` 폴백 사용) |

### 빌드 타임 — 서버 사이드 전용 (단, 빌드 때 번들에 포함)

```dockerfile
ARG API_PROXY_ORIGIN
ENV API_PROXY_ORIGIN=$API_PROXY_ORIGIN
```

- **서버(Node.js) 라우트 핸들러에서만** 사용 → 브라우저에 노출 안 됨
- **주의:** Next.js standalone 빌드에서 webpack이 `process.env.*` 참조를 빌드 타임에 정적 치환함.
  컨테이너 런타임 환경변수로만 주입하면 읽히지 않음. **Dockerfile에 ARG/ENV로 넣어야** 정상 동작.
- 변경 시 반드시 재빌드 필요

| 변수 | 설명 |
|------|------|
| `API_PROXY_ORIGIN` | FastAPI 백엔드 주소 (예: `http://x.x.x.x:80`) |

## 환경변수 주입 경로

```
GitHub Secret (API_PROXY_ORIGIN)
  → GHA build 스텝 (--build-arg)
  → Dockerfile ARG → ENV
  → Next.js 빌드 때 webpack이 process.env.API_PROXY_ORIGIN 치환
  → 번들에 값이 정적으로 포함됨
```

GHA 워크플로우 `build-args`에 포함:

```yaml
build-args: |
  NEXT_PUBLIC_GOOGLE_CLIENT_ID=${{ secrets.NEXT_PUBLIC_GOOGLE_CLIENT_ID }}
  NEXT_PUBLIC_API_URL=${{ secrets.NEXT_PUBLIC_API_URL }}
  API_PROXY_ORIGIN=${{ secrets.API_PROXY_ORIGIN }}
```

## apiClient 동작 원리

`packages/api/src/client/index.ts`의 `resolveBaseUrl()`:

```typescript
function resolveBaseUrl(): string {
  return (
    process.env.NEXT_PUBLIC_API_BASE_URL ??  // 설정 안 됨
    process.env.EXPO_PUBLIC_API_BASE_URL ??  // 모바일 앱용
    "/api"                                   // ← 웹은 이 폴백 사용
  );
}
```

웹 앱은 `NEXT_PUBLIC_API_BASE_URL`을 설정하지 않으므로 baseURL이 `/api`가 됩니다.  
axios가 `/v1/auth/me`를 붙이면 → `/api/v1/auth/me` → Next.js 프록시 라우트가 캐치.

## 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| `ECONNREFUSED 127.0.0.1:8080` | `API_PROXY_ORIGIN` 미설정, fallback 사용됨 | EC2 `.env` 파일 확인 또는 GHA 재배포 |
| `500 Internal Server Error` (프록시 단) | fetch 실패 또는 hop-by-hop 헤더 충돌 | CloudWatch `/vivac/web` 로그에서 `[proxy] fetch failed:` 확인 |
| `502 upstream_unavailable` | FastAPI 서버 다운 또는 네트워크 차단 | Lightsail 방화벽, FastAPI 프로세스 확인 |
