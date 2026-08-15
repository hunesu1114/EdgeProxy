# edge

104 서버의 최전방 리버스 프록시. **80/443 을 점유하는 유일한 컨테이너다.**

```
                    인터넷
                      │  :80 / :443
              ┌───────▼────────┐
              │   edge-proxy   │  TLS 종단 + 도메인 라우팅
              └───┬────────┬───┘
       filehub 도메인      boram-khs.duckdns.org
                  │        │
     ┌────────────▼──┐  ┌──▼─────────┐
     │filehub-frontend│  │ boram-web  │   각각 정적 파일 + /api 프록시
     └───────┬───────┘  └─────┬──────┘
     filehub-backend      boram-backend
```

앱은 각자 저장소의 compose 로 배포한다. **앱을 배포해도 이 스택은 건드리지 않는다.**
도메인이 늘어날 때만 여기를 수정한다.

## 왜 앱마다 프록시를 두지 않는가

원래 FileHub 는 자체 `filehub-proxy` 로 TLS 를 끝냈다. 앱이 하나일 때는 문제가 없었지만,
두 번째 앱이 들어오면서 80/443 을 나눠 쓸 수 없게 됐다.

앞에 edge 를 두면서 앱별 프록시는 **하는 일이 없어진다.** 그 존재 이유가 TLS 종단이었는데
edge 가 이미 끝냈기 때문이다. 남겨두면 한 요청이 nginx 를 세 번 지나고,
`client_max_body_size` 나 `X-Forwarded-Proto` 를 계층마다 맞춰야 해서
나중에 업로드가 조용히 깨지는 지점만 늘어난다. 그래서 앱별 프록시는 제거했다.

---

## 최초 설정

### 1. 공용 네트워크 생성

앱 compose 들이 `external: true` 로 참여한다. **이게 먼저 있어야 앱이 뜬다.**

```bash
docker network create edge-net
```

### 2. 인증서 배치

acme.sh 로 도메인별 인증서를 발급하고 아래 구조로 설치한다.

```
/etc/edge/certs/
├─ default/     # 도메인 불일치 요청용. 아무 인증서나 하나 복사해도 된다
│  ├─ fullchain.pem
│  └─ privkey.pem
├─ filehub/
└─ boram/
```

DuckDNS 는 HTTP-01 이 공유기 때문에 막히는 경우가 많아 **DNS-01** 을 쓴다.

```bash
export DuckDNS_Token="발급받은_토큰"
```

```bash
acme.sh --issue --dns dns_duckdns -d boram-khs.duckdns.org
```

```bash
acme.sh --install-cert -d boram-khs.duckdns.org \
  --key-file       /etc/edge/certs/boram/privkey.pem \
  --fullchain-file /etc/edge/certs/boram/fullchain.pem \
  --reloadcmd      "docker exec edge-proxy nginx -s reload"
```

`--reloadcmd` 덕분에 갱신될 때마다 nginx 가 알아서 다시 읽는다.
이게 없으면 인증서는 갱신됐는데 nginx 는 옛것을 물고 있어 어느 날 만료된다.

### 3. FileHub 도메인 채우기

`conf.d/filehub.conf` 의 `__FILEHUB_DOMAIN__` 두 곳을 실제 도메인으로 바꾼다.

### 4. 기동

```bash
docker compose up -d
```

```bash
docker compose exec proxy nginx -t
```

---

## 도메인 추가

1. `conf.d/<이름>.conf` 를 만든다 (`boram.conf` 를 복사해 고치는 게 빠르다)
2. 인증서를 `/etc/edge/certs/<이름>/` 에 설치한다
3. 앱 compose 가 `edge-net` 에 참여하고 `container_name` 이 고정되어 있는지 확인한다
   — 프록시는 컨테이너 이름으로 찾아간다
4. 반영

```bash
docker compose exec proxy nginx -t && docker compose exec proxy nginx -s reload
```

`nginx -t` 로 먼저 검사한다. 설정이 틀린 채로 reload 하면 **기존 설정이 유지되긴 하지만**,
컨테이너를 재시작하는 순간 전체가 죽는다.

---

## 문제가 생겼을 때

| 증상 | 확인 |
|---|---|
| 502 Bad Gateway | 뒤쪽 컨테이너가 죽었거나 `edge-net` 에 없다. `docker network inspect edge-net` |
| 엉뚱한 사이트가 뜬다 | 해당 도메인의 server 블록이 없어 기본 서버로 갔다 |
| 인증서 만료 | `acme.sh --list` 로 갱신 상태 확인. `--reloadcmd` 가 걸려 있는지도 |
| 413 Request Entity Too Large | `client_max_body_size` 가 edge < 앱 순으로 작으면 앞단에서 먼저 막힌다 |

```bash
docker compose logs -f proxy
```
