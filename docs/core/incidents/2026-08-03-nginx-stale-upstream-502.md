# nginx가 옛 upstream IP를 물어 api.vivac.app 전 경로 502

- **발생** 2026-08-03 10:35 UTC (19:35 KST)
- **복구** 2026-08-03 12:04 UTC (21:04 KST)
- **지속** 89분
- **영향** 외부에서 `api.vivac.app` 전 경로 502. `/health` 포함. 내부 컨테이너 간 호출은 정상이라 콘솔 작업은 계속 됐다.
- **트리거** vivacapi-core `v0.21.1` 배포

## 증상

```
$ curl -sI https://api.vivac.app/health
HTTP/2 502
server: nginx/1.31.3
x-cache: Error from cloudfront
```

배포 워크플로는 success로 끝나 있었다. EC2·컨테이너·CloudWatch 어디에도 에러가 없었다.

## 원인

`VIVAC-frontend/infra/nginx.conf`가 upstream을 정적으로 지정하고 있었다.

```nginx
location / {
    proxy_pass http://api:8000;
}
```

`proxy_pass`에 변수 없이 호스트명을 쓰면 **nginx는 설정 로드 시 1회만 DNS를 해석하고 그 IP를 계속 사용한다.** 컨테이너가 재생성돼 주소가 바뀌어도 모른다.

배포 시점의 실제 값:

| | 주소 |
|---|---|
| nginx가 물고 있던 upstream | `172.24.0.2:8000` |
| `vivac-api-1` 실제 주소 | `172.24.0.7` |

`vivac-front-nginx-1`은 6일째 재기동 없이 떠 있었다.

```
2026/08/03 11:58:53 [error] 21#21: *8239 connect() failed (111: Connection refused)
while connecting to upstream, client: 13.124.199.11, server: api.vivac.app,
request: "GET /health HTTP/1.1", upstream: "http://172.24.0.2:8000/health"
```

### 왜 이번 배포에서만

이전 배포들도 api 컨테이너를 매번 재생성했지만 터지지 않았다.

| | 재생성된 컨테이너 |
|---|---|
| ~v0.21.0 | api 1개 |
| v0.21.1 | api + redis 2개 |

v0.21.1은 `infra/docker-compose.yml`의 redis 서비스에 `command`(maxmemory)와 `logging`을 추가했다. compose는 설정이 바뀐 서비스를 재생성하므로 이번엔 두 컨테이너가 함께 내려갔다 올라왔다.

하나만 재생성하면 방금 비운 주소를 그대로 회수할 가능성이 높다. 둘을 동시에 재생성하면 배정이 밀려 주소가 뒤바뀔 수 있다. redis 변경은 방아쇠였을 뿐, 결함은 처음부터 nginx 설정에 있었다.

## 왜 89분이나 걸렸나

두 가지가 겹쳤다.

**1. 배포 헬스체크가 잘못된 경로를 봤다.** `deploy.yml`이 `http://localhost:8000/health`만 확인했다. 이건 EC2 호스트에서 api 컨테이너로 직결되는 경로라 nginx를 거치지 않는다. 외부가 완전히 죽은 상태에서 워크플로는 초록불로 끝났다.

**2. API 로그가 정상으로 보였다.** 배포 후에도 200 응답 로그가 계속 찍혔다. 출처 IP를 보고서야 전부 내부 호출이었음을 알았다.

```
172.24.0.1  →  EC2 호스트 (배포 스크립트의 헬스체크)
172.24.0.5  →  console 컨테이너
172.24.0.3  →  front-web SSR 프록시
```

세 출처 모두 nginx를 우회해 api를 직접 부른다. nginx를 거친 외부 요청은 배포 직후부터 단 한 건도 성공하지 못했다. **"최근 요청이 200"은 외부 정상의 근거가 못 된다** — 출처 IP를 반드시 함께 봐야 한다.

## 복구

```bash
docker exec vivac-front-nginx-1 nginx -s reload
```

reload가 설정을 다시 읽으면서 upstream 이름을 재해석한다. 즉시 복구됐고 데이터 손실이나 롤백은 없었다.

## 재발 시 대응

증상이 "외부 502인데 컨테이너는 전부 Up"이면 이 건과 같다.

```bash
# 1. api 자체는 살아있는가 (EC2 안에서)
curl -s localhost:8000/health

# 2. nginx가 어느 주소로 가려다 실패하는가
docker logs --tail 30 vivac-front-nginx-1 2>&1 | grep -i "connect() failed"

# 3. 실제 주소와 대조
docker inspect vivac-api-1 --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'

# 4. 1이 정상이고 2의 upstream IP가 3과 다르면 확정 → 복구
docker exec vivac-front-nginx-1 nginx -s reload
```

api뿐 아니라 `vivac-front-web-1`, `vivac-console-console-1`도 같은 결함을 공유한다. 해당 컨테이너가 재생성되면 `vivac.app`, `console.vivac.app`에서 동일한 증상이 난다.

## 조치

**완료 — 감지** (`vivacapi-core` #146, v0.21.2)

배포 후 `https://api.vivac.app/health` 검증을 `[5/5]` 단계로 추가했다. 사용자가 실제로 타는 CloudFront → nginx → api 경로를 통과해야 배포가 완료된다. 실패 시 nginx upstream 에러를 덤프한다.

**필요 — 근본 수정** (`VIVAC-frontend`, `infra/nginx.conf`)

`proxy_pass`를 변수로 바꾸고 resolver를 두면 요청 시점마다 재해석된다.

```nginx
resolver 127.0.0.11 valid=10s;
set $upstream http://api:8000;
proxy_pass $upstream;
```

`127.0.0.11`은 docker 내장 DNS의 고정 주소다. resolver 없이 변수만 쓰면 요청 시점에 `no resolver defined to resolve api`로 실패한다.

api / web / console 세 블록 모두 적용해야 한다.

## 배운 것

- **헬스체크는 사용자가 타는 경로를 그대로 타야 한다.** 프록시를 건너뛴 체크는 프록시 장애를 원리적으로 못 잡는다.
- **컨테이너 IP는 재생성마다 바뀔 수 있다.** 우연히 유지되던 것에 기대고 있었고, 그 우연은 재생성 대상이 하나 늘자 깨졌다.
- **로그의 200은 출처를 봐야 의미가 생긴다.** 내부 호출만으로도 정상처럼 보인다.
