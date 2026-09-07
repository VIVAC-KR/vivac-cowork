# CloudFront + Next.js RSC 캐싱 이슈

## 증상

- Google OAuth 로그인 후 `/login` 페이지에 RSC 페이로드 원문이 텍스트로 노출
- `Cross-Origin-Opener-Policy` 헤더가 브라우저에 도달하지 않음
- Response Header에서 확인된 상태:
  ```
  content-type: text/x-component
  x-cache: Hit from cloudfront
  cache-control: s-maxage=31536000
  ```

## 원인

Next.js App Router는 두 종류의 응답을 반환한다.

| 요청 유형 | Content-Type | 용도 |
|-----------|-------------|------|
| 일반 페이지 로드 | `text/html` | 브라우저 렌더링 |
| 클라이언트 사이드 내비게이션 (RSC fetch) | `text/x-component` | React RSC 클라이언트 처리 |

CloudFront Cache Policy의 `Vary` 헤더에 `RSC`, `Next-Router-State-Tree` 등이 포함되지 않으면, 두 요청을 같은 캐시 키로 처리한다.

RSC 응답(`text/x-component`)이 먼저 캐시되면, 이후 일반 페이지 로드 요청에도 같은 RSC 페이로드가 반환된다. 브라우저는 `text/x-component`를 렌더링할 수 없어 원문 텍스트로 표시한다.

또한 CloudFront가 오래된 응답을 캐시하고 있어, Next.js 서버에서 설정한 `Cross-Origin-Opener-Policy` 헤더가 브라우저에 전달되지 않았다.

## COOP 이슈와의 연관

Google OAuth는 팝업에서 인증 완료 후 `postMessage`로 부모 창에 결과를 전달한다. `Cross-Origin-Opener-Policy: same-origin` 헤더가 이를 차단하면 Google 라이브러리가 리다이렉트 모드로 폴백하면서 전체 페이지 이동이 발생하고, 이 과정에서 CloudFront가 잘못된 캐시(RSC 페이로드)를 서빙하게 된다.

## 해결

### 1. next.config.ts — COOP 헤더 추가

```ts
async headers() {
  return [
    {
      source: "/(.*)",
      headers: [
        { key: "Cross-Origin-Opener-Policy", value: "same-origin-allow-popups" },
      ],
    },
  ];
},
```

### 2. CloudFront Cache Policy 수정

Cache Policy에 다음 헤더를 캐시 키로 추가한다 (`vivac-nextjs-rsc` 정책):

```
RSC
Next-Router-State-Tree
Next-Url
Next-Router-Prefetch
```

이렇게 하면 CloudFront가 RSC 요청과 HTML 요청을 별도의 캐시로 관리한다.

### 3. CloudFront 캐시 무효화

설정 변경 전 쌓인 잘못된 캐시를 제거한다.

```
Invalidation path: /*
```

## 재발 방지

- CloudFront 앞에 Next.js를 배포할 때는 반드시 RSC 관련 헤더를 Cache Policy에 포함해야 한다.
- 새 Distribution을 생성하거나 Cache Policy를 교체할 때 위 헤더 목록을 체크한다.
