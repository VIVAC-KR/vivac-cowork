# 인증 로직 구현 보고서

**프로젝트:** VIVAC — 국내 백패킹/미니멀 캠핑 유저를 위한 통합 장소 탐색 앱  
**작성일:** 2026-06-17  
**최종 수정:** 2026-06-17  
**대상:** 프론트엔드 개발자

---

## 목차

1. [프로젝트 컨텍스트](#1-프로젝트-컨텍스트)
2. [아키텍처 결정사항](#2-아키텍처-결정사항)
3. [인증 플로우](#3-인증-플로우)
4. [토큰 저장 전략](#4-토큰-저장-전략)
5. [백엔드 API 명세](#5-백엔드-api-명세)
6. [Axios 인터셉터 설계](#6-axios-인터셉터-설계)
7. [구현 파일 목록](#7-구현-파일-목록)
8. [환경변수](#8-환경변수)
9. [구현 진행 상태](#9-구현-진행-상태)
10. [알려진 이슈 및 기술 노트](#10-알려진-이슈-및-기술-노트)
11. [미결 사항](#11-미결-사항)

---

## 1. 프로젝트 컨텍스트

| 항목 | 내용 |
|---|---|
| 프로젝트명 | VIVAC |
| 모노레포 도구 | Turborepo + pnpm |
| 프레임워크 | Next.js 15 (App Router) |
| 언어 | TypeScript (strict mode) |
| 주요 라이브러리 | TanStack Query, Zustand, Axios |

---

## 2. 아키텍처 결정사항

### next-auth 미사용 이유

이 프로젝트에서는 `next-auth`를 사용하지 않는다. 이유는 다음과 같다.

- **백엔드가 인증의 주체다.** 세션 관리, 유저 DB 모두 백엔드에서 담당한다.
- `next-auth`는 **프론트엔드가 인증 주체**일 때 적합한 라이브러리다. 백엔드 중심 구조에 억지로 적용하면 next-auth 세션과 백엔드 세션을 동시에 관리해야 하는 역할 중복이 발생한다.

### 채택 라이브러리

**`@react-oauth/google` (v0.13.5)** — Google Identity Services의 공식 React 래퍼 라이브러리.  
`GoogleLogin` 컴포넌트를 사용하며, 이 컴포넌트는 `credential` 필드로 **ID 토큰(JWT)** 을 반환한다.

> `useGoogleLogin` 훅은 access_token을 반환하므로 id_token이 필요한 이 프로젝트에 적합하지 않다.

---

## 3. 인증 플로우

### 3-1. 로그인 플로우

```
사용자 → Google 로그인 버튼 클릭
  → GoogleLogin 컴포넌트 → Google 인증 팝업
  → credential(id_token) 발급
  → POST /v1/auth/google { id_token }
  → 백엔드 응답: { access_token, refresh_token, token_type: "bearer" }
  → refresh_token → localStorage
  → access_token → Zustand (메모리)
  → GET /v1/auth/me (Authorization: Bearer {access_token})
  → 유저 정보 → Zustand
  → 메인 페이지(/)로 리디렉션
```

### 3-2. 앱 초기화 플로우 (새로고침 대응)

페이지 새로고침 시 메모리(Zustand)에 저장된 access_token이 사라진다. 이를 복원하기 위해 앱 로드 시 다음 플로우를 실행한다.

```
앱 로드 → useAuthInit 훅 실행
  → localStorage에서 refresh_token 확인
      → 있음: POST /v1/auth/refresh { refresh_token }
             → 새 access_token → Zustand 저장
             → GET /v1/auth/me → 유저 정보 → Zustand 저장 → 로그인 상태 복원
      → 없음: 비로그인 상태 유지
```

이 플로우는 `useAuthInit` 훅에서 처리하며, `AuthInit` 컴포넌트를 통해 `Providers.tsx`에서 한 번만 실행된다.

---

## 4. 토큰 저장 전략

| 토큰 | 저장 위치 | 이유 |
|---|---|---|
| access_token | Zustand (메모리) | 수명이 짧고 API 요청마다 사용. 메모리가 XSS에 가장 안전 |
| refresh_token | localStorage | 새로고침 후에도 세션 유지 필요. 백엔드가 body로 반환하므로 HttpOnly 쿠키 불가 |

### 트레이드오프 노트

- `refresh_token`을 `localStorage`에 저장하는 것은 XSS 취약점이 있는 페이지에서 탈취될 수 있다.
- 현재 백엔드는 `Set-Cookie(HttpOnly)` 방식이 아닌 response body로 토큰을 반환하므로 프론트엔드에서 `HttpOnly` 쿠키를 사용할 수 없다.
- MVP 단계에서는 이 트레이드오프를 허용 가능한 수준으로 판단한다. 추후 백엔드와 협의해 `HttpOnly` 쿠키 방식으로 전환 가능하다.

### localStorage 접근 레이어 분리

`packages/store`는 플랫폼 무관 패키지이므로 DOM API(`localStorage`)를 직접 사용하지 않는다.  
localStorage 접근은 브라우저 레이어(`apps/web`)에서만 수행한다.

| 위치 | 역할 |
|---|---|
| `packages/store/src/auth/useAuthStore.ts` | 메모리 상태만 관리 (localStorage 미접근) |
| `apps/web/src/features/auth/GoogleLoginButton.tsx` | 로그인 성공 시 refresh_token 저장 |
| `apps/web/src/app/Providers.tsx` | 로그아웃 시 refresh_token 제거, configureAuth 주입 |

---

## 5. 백엔드 API 명세

### 5-1. 로그인 (Google OAuth)

```
POST /v1/auth/google
```

**요청 바디:**
```json
{
  "id_token": "<Google에서 발급한 ID 토큰 (JWT)>"
}
```

**응답:**
```json
{
  "access_token": "string",
  "refresh_token": "string",
  "token_type": "bearer"
}
```

### 5-2. 유저 정보 조회

```
GET /v1/auth/me
Authorization: Bearer {access_token}
```

**응답:**
```json
{
  "uid": "string",
  "email": "string",
  "nickname": "string",
  "name": null,
  "picture": null,
  "is_active": false,
  "membership_tier": "member",
  "identity_verified_at": "ISO8601",
  "onboarding_survey_completed_at": "ISO8601",
  "created_at": "ISO8601",
  "updated_at": "ISO8601",
  "is_identity_verified": false,
  "has_completed_onboarding_survey": true
}
```

### 5-3. 토큰 갱신

```
POST /v1/auth/refresh
```

**요청 바디:**
```json
{
  "refresh_token": "string"
}
```

**응답:**
```json
{
  "access_token": "string",
  "token_type": "bearer"
}
```

---

## 6. Axios 인터셉터 설계

`packages/api/src/client/index.ts`에서 Axios 인스턴스를 생성하고 인터셉터를 설정한다.

### configureAuth 패턴

`packages/api`는 `packages/store`에 의존하지 않는다(순환 의존 방지). 대신 `configureAuth` 함수로 getter/setter를 주입받는다. 주입은 `apps/web/src/app/Providers.tsx` 모듈 로드 시 1회 실행된다.

```
packages/api/src/client/auth.ts
  └── configureAuth({ getAccessToken, getRefreshToken, setAccessToken, clearAuth })
        ← apps/web/src/app/Providers.tsx 에서 useAuthStore.getState()로 주입
```

### 요청 인터셉터

```
요청 발생 → getAccessToken() → Zustand에서 access_token 읽기
         → headers.Authorization = `Bearer ${access_token}` 설정
         → 요청 전송
```

### 응답 인터셉터 (401 자동 갱신)

```
응답 수신
  → 401이 아님: 정상 반환
  → 401 + 재시도 플래그 없음:
      → 다른 갱신 요청 진행 중? → queue에 대기
      → getRefreshToken() → localStorage에서 refresh_token 읽기
      → 없음: clearAuth() → 로그아웃
      → 있음: raw axios로 POST /v1/auth/refresh (인터셉터 루프 방지)
          → 성공: setAccessToken(new_token) → queue 처리 → 원래 요청 재시도
          → 실패: clearAuth() → queue 거부
```

---

## 7. 구현 파일 목록

| 파일 | 역할 | 상태 |
|---|---|---|
| `packages/shared/src/types/auth.ts` | `User`, `AuthTokens` Zod 스키마 + 타입 | ✅ 완료 |
| `packages/api/src/client/auth.ts` | `configureAuth` — 인터셉터 getter/setter 주입 모듈 | ✅ 완료 |
| `packages/api/src/client/index.ts` | Axios 인스턴스 + 요청/응답 인터셉터 | ✅ 완료 |
| `packages/api/src/queries/auth.ts` | `loginWithGoogle`, `fetchMe`, `refreshAccessToken` | ✅ 완료 |
| `packages/store/src/auth/useAuthStore.ts` | 인증 전역 상태 (accessToken, user, isAuthenticated) | ✅ 완료 |
| `apps/web/src/features/auth/GoogleLoginButton.tsx` | Google 로그인 버튼 컴포넌트 | ✅ 완료 |
| `apps/web/src/features/auth/useAuthInit.ts` | 앱 로드 시 refresh_token으로 세션 복원 훅 | ✅ 완료 |
| `apps/web/src/app/Providers.tsx` | GoogleOAuthProvider + configureAuth + AuthInit 연결 | ✅ 완료 |
| `apps/web/src/app/login/page.tsx` | 로그인 페이지 UI | ✅ 완료 |

### 패키지 의존 관계

```
packages/shared/src/types/auth.ts
        ↑
packages/store/src/auth/useAuthStore.ts
packages/api/src/client/auth.ts
        ↑
packages/api/src/client/index.ts
packages/api/src/queries/auth.ts
        ↑
apps/web/src/features/auth/GoogleLoginButton.tsx
apps/web/src/features/auth/useAuthInit.ts
        ↑
apps/web/src/app/Providers.tsx  ← configureAuth 주입 진입점
```

---

## 8. 환경변수

`apps/web/.env.local`에 다음을 설정한다.

```env
NEXT_PUBLIC_API_BASE_URL=<백엔드 서버 URL>
NEXT_PUBLIC_GOOGLE_CLIENT_ID=<Google Cloud Console에서 발급한 클라이언트 ID>
```

> `NEXT_PUBLIC_` 접두사가 붙어야 클라이언트 사이드에서 접근 가능하다 (Next.js 규칙).

**주의:** 환경변수 변경 후 Next.js 개발 서버를 반드시 재시작해야 한다.

---

## 9. 구현 진행 상태

### 완료

- [x] `@react-oauth/google` 패키지 설치
- [x] `packages/shared` — User, AuthTokens Zod 스키마 + 타입 정의
- [x] `packages/store` — useAuthStore 구현 (accessToken, user, isAuthenticated)
- [x] `packages/api` — configureAuth 패턴으로 인터셉터 설계 및 구현
- [x] `packages/api` — loginWithGoogle, fetchMe, refreshAccessToken 함수 구현
- [x] `apps/web` — GoogleLoginButton 컴포넌트 구현 (GoogleLogin 컴포넌트 기반)
- [x] `apps/web` — useAuthInit 훅 구현 (세션 복원)
- [x] `apps/web` — Providers.tsx에 GoogleOAuthProvider 및 configureAuth 연결
- [x] `apps/web` — 로그인 페이지 UI 구현
- [x] Google Cloud Console OAuth 클라이언트 ID 발급 및 환경변수 설정
- [x] `NEXT_PUBLIC_API_BASE_URL` 환경변수 설정

### 중단 (백엔드 협의 필요)

- [ ] **백엔드 CORS 설정** — 현재 프론트엔드(`http://localhost:3000`)에서 백엔드(`http://3.39.122.225:80`)로의 요청이 CORS 정책으로 차단됨. 백엔드에서 허용 Origin 추가 필요.

### 미착수

- [ ] 로그인 성공/실패 에러 토스트 UI 연동
- [ ] 로그아웃 기능 구현
- [ ] 비로그인 접근 제어 (미들웨어 또는 페이지 레벨 가드)

---

## 10. 알려진 이슈 및 기술 노트

### Next.js 환경변수 정적 치환 주의

Next.js는 `NEXT_PUBLIC_*` 환경변수를 **빌드 타임에 코드에 직접 삽입**하며, 정확히 `process.env.NEXT_PUBLIC_*` 패턴만 인식한다.

```ts
// ❌ 인식 안 됨 — 브라우저에서 undefined 반환
const env = process.env;
env.NEXT_PUBLIC_API_BASE_URL;

// ✅ 올바른 패턴
process.env.NEXT_PUBLIC_API_BASE_URL;
```

`packages/api/src/client/index.ts`의 `resolveBaseUrl()` 함수가 이 패턴으로 수정되어 있다.

### Google Cloud Console 승인된 Origin 등록 필수

`GoogleLogin` 컴포넌트는 Google 서버와 직접 통신하므로, Cloud Console에서 사용 중인 도메인을 **승인된 JavaScript 원본**에 등록해야 한다. 미등록 시 `The given origin is not allowed for the given client ID` 오류 발생.

- 개발: `http://localhost:3000`
- 프로덕션: 실제 서비스 도메인

---

## 11. 미결 사항

| # | 항목 | 담당 | 상태 |
|---|---|---|---|
| 1 | 백엔드 CORS 허용 Origin 추가 (`http://localhost:3000`, 프로덕션 도메인) | 백엔드 (최진혁) | **협의 필요** |
| 2 | `POST /v1/auth/refresh` 응답에 `refresh_token` 포함 여부 확인 (token rotation 여부) | 백엔드 (최진혁) | 미확인 |
| 3 | 에러 토스트 UI 컴포넌트 설계 및 연동 | 프론트엔드 | 미착수 |
| 4 | 로그아웃 API 엔드포인트 존재 여부 확인 | 백엔드 (최진혁) | 미확인 |
| 5 | 비로그인 접근 제어 범위 결정 (어느 페이지에 가드를 걸 것인지) | 프론트엔드 | 미착수 |
