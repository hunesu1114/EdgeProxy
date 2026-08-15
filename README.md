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

### 3. 기동

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
4. `master` 에 push 한다. 나머지는 아래 CI/CD 가 한다

---

## 배포 (CI/CD)

`master` 에 push 하면 GitHub Actions 가 검증 후 서버에 반영한다.

```
master push
   │
   ├─ ① 검증    더미 인증서로 nginx -t · compose config · upstream 규칙
   └─ ② 배포    설정 전송 → 실제 인증서로 재검증 → 자리 교체 → reload → 두 도메인 확인
```

PR 은 ①만 돌고 배포되지 않는다.

### 이 저장소의 배포가 특별히 조심스러운 이유

edge 는 **모든 사이트가 지나는 단일 길목이다.** 설정 한 줄이 잘못되면 boram 과 FileHub 가
동시에 죽는다. 그래서 배포가 세 겹으로 막는다.

1. **CI 에서 먼저 본다** — 자체 서명 인증서로 경로만 채우고 `nginx -t` 를 돌린다.
   문법 오류는 서버에 닿지도 못한다.
2. **서버에서 실제 인증서로 다시 본다** — CI 의 더미 인증서로는 만료·체인 문제를 못 잡는다.
   `~/edge.new` 에 풀어 **일회용 컨테이너**로 검사하므로 돌고 있는 edge 는 영향받지 않는다.
   검증을 통과한 뒤에야 `conf.d/` 를 덮어쓴다.
   — 먼저 덮어쓰면 깨진 파일이 자리를 잡고, 그 뒤 아무 이유로든 컨테이너가 재시작되는
   순간 전체가 죽는다.
3. **`restart` 가 아니라 `reload`** — reload 는 새 설정이 잘못되면 예전 설정으로 계속
   서비스하지만, restart 는 그대로 죽는다.

### 검사 항목 하나 (upstream)

`proxy_pass http://boram-web:80;` 처럼 컨테이너 이름을 **직접** 쓰면, 그 앱이 내려가
있을 때 nginx 가 기동 자체를 못 해 **다른 사이트까지 함께 죽는다.**
`resolver` + 변수(`set $boram_upstream ...`)를 써야 요청 시점에 이름을 푼다.
실수로 되돌아가지 않도록 CI 가 막는다.

### GitHub Secrets

저장소 → Settings → Secrets and variables → Actions

| 이름 | 설명 |
|---|---|
| `SERVER_HOST` | 104 서버 주소 |
| `SERVER_USER` | SSH 사용자 |
| `SERVER_SSH_KEY` | SSH 개인키 전문 |
| `SERVER_PORT` | SSH 포트 (22 면 생략 가능) |

boram / FileHub 저장소에 이미 등록한 것과 같은 값이다.
**인증서와 관련된 Secret 은 없다.** 인증서는 서버의 `/etc/edge/certs` 에만 있고
저장소에는 들어오지 않는다.

### 배포 확인이 502 를 통과시키는 이유

배포 끝에 두 도메인을 확인하는데 `502` 는 성공으로 친다.
502 는 **edge 는 멀쩡한데 뒤쪽 앱이 아직 안 떠 있다**는 뜻이다.
앱 사정 때문에 edge 배포가 실패하면 안 된다.

### 손으로 반영해야 할 때

```bash
cd ~/edge && docker compose exec proxy nginx -t && docker compose exec proxy nginx -s reload
```

> **`nginx -t` 는 파일만 검사한다. 반드시 reload 까지 해야 반영된다.**
> 파일을 고쳐놓고 reload 를 빠뜨리면 nginx 는 옛 설정으로 계속 돌아간다.
> 겉으로는 고친 것처럼 보이는데 동작은 그대로라, 엉뚱한 곳을 한참 뒤지게 된다.

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
