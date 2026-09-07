# Docker 빌드 및 배포 가이드

## 구성 개요

pnpm + Turborepo 모노레포 구조에서 `apps/web` (Next.js 15)를 Docker 이미지로 빌드해 Docker Hub에 배포하는 구성입니다.

## 변경된 파일

| 파일 | 내용 |
|------|------|
| `Dockerfile` | 멀티스테이지 빌드 (builder → runner) |
| `.dockerignore` | 빌드 컨텍스트 최적화 |
| `apps/web/next.config.ts` | `output: "standalone"` 추가 |
| `apps/web/src/app/Providers.tsx` | ESLint import/order 수정 |
| `apps/web/src/features/auth/GoogleLoginButton.tsx` | ESLint no-misused-promises 수정 |
| `apps/web/src/features/auth/useAuthInit.ts` | 존재하지 않는 규칙 eslint-disable 주석 제거 |

## Dockerfile 구조

### Stage 1: builder
- `node:20-alpine` + `pnpm@10.19.0` (corepack)
- `package.json` / `pnpm-lock.yaml` 먼저 복사 → `pnpm install` → 소스 복사 순서로 레이어 캐시 최적화
- `pnpm --filter @vivac/web build` 로 Next.js 빌드

### Stage 2: runner
- `node:20-alpine` 베이스, non-root 유저(`nextjs`) 실행
- standalone 출력물만 복사 → 최종 이미지 **213MB**
- `CMD ["node", "apps/web/server.js"]`

## 환경 변수

### 빌드 타임 (`NEXT_PUBLIC_*`)
Next.js가 빌드 시 번들에 인라인하므로 `docker build --build-arg`로 주입해야 합니다. 런타임 `-e`로는 적용되지 않습니다.

| 변수 | 설명 |
|------|------|
| `NEXT_PUBLIC_GOOGLE_CLIENT_ID` | Google OAuth 클라이언트 ID |
| `NEXT_PUBLIC_API_URL` | 백엔드 API 베이스 URL |

### 런타임
서버 사이드 전용 시크릿이 생기면 `docker run --env-file .env.production`으로 주입합니다.

## 로컬 빌드 및 실행

```bash
# 빌드
docker build \
  --build-arg NEXT_PUBLIC_GOOGLE_CLIENT_ID=<client-id> \
  --build-arg NEXT_PUBLIC_API_URL=https://api.example.com \
  -t vivac-web .

# 실행
docker run -p 3000:3000 vivac-web
# → http://localhost:3000
```

## Docker Hub 푸시

```bash
docker tag vivac-web vivacadmin/vivac-web:latest
docker push vivacadmin/vivac-web:latest
```

## 배포 아키텍처

```
GitHub Push (main)
    ↓
GitHub Actions
  ├─ docker build (GitHub Secrets → --build-arg)
  └─ docker push → Docker Hub (vivacadmin/vivac-web)
         ↓
       EC2
  ├─ docker pull vivacadmin/vivac-web:latest
  └─ docker run -p 3000:3000 ...
```

CI/CD 워크플로우는 `.github/workflows/docker-publish.yml` 참고.
