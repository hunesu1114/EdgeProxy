# FileHub 를 edge 뒤로 옮기기 (서버 작업 순서)

**코드 수정은 이미 끝났다.** FileHub 저장소의 `docker-compose.prod.yaml` 에서
`proxy` 서비스와 `configs` 블록을 제거하고, `frontend` 를 `edge-net` 에 붙였다.
워크플로에는 `--remove-orphans` 와 Host 헤더 헬스체크를 넣었다.

여기서는 **서버에서 해야 할 일**만 다룬다.

```
[전]  인터넷 → filehub-proxy(:443, TLS) → filehub-frontend → filehub-backend
[후]  인터넷 → edge-proxy(:443, TLS) ──→ filehub-frontend → filehub-backend
```

---

## 다운타임

**3단계와 4단계 사이 수십 초.** 그 구간에만 FileHub 가 밖에서 안 보인다.
1~2단계는 서비스에 영향이 없으니 미리 다 끝내두고, 3~4단계만 붙여서 실행한다.

---

## 1. 공용 네트워크

```bash
docker network create edge-net
```

## 2. edge 준비 (아직 띄우지 않는다)

443 은 아직 filehub-proxy 가 잡고 있으므로, 파일과 인증서만 배치한다.

```bash
cd ~ && git clone https://github.com/hunesu1114/EdgeProxy.git edge && cd edge
```

### 2-1. 인증서 배치

기존 FileHub 인증서를 edge 구조로 복사한다.

```bash
sudo mkdir -p /etc/edge/certs/filehub /etc/edge/certs/default /etc/edge/certs/boram
sudo cp /etc/filehub/certs/*.pem /etc/edge/certs/filehub/
sudo cp /etc/filehub/certs/*.pem /etc/edge/certs/default/
```

`default` 는 도메인이 맞지 않는 요청을 받아내는 자리다. TLS 핸드셰이크가 성립하려면
인증서가 있어야 해서 아무거나 하나 넣어둔다.

### 2-2. boram 인증서 발급

DuckDNS 는 공유기가 80 을 막는 경우가 많아 **DNS-01** 을 쓴다.

```bash
export DuckDNS_Token="DuckDNS 대시보드의 토큰"
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

### 2-3. FileHub 인증서 갱신 경로 변경

acme.sh 가 지금 `/etc/filehub/certs` 로 설치하고 있다. 그대로 두면
**갱신은 되는데 edge 는 옛 인증서를 계속 물고 있다가 어느 날 만료된다.**

```bash
acme.sh --install-cert -d filehub-khs.duckdns.org \
  --key-file       /etc/edge/certs/filehub/privkey.pem \
  --fullchain-file /etc/edge/certs/filehub/fullchain.pem \
  --reloadcmd      "docker exec edge-proxy nginx -s reload"
```

## 3. filehub-proxy 내리기 (여기서부터 다운타임)

```bash
docker stop filehub-proxy && docker rm filehub-proxy
```

## 4. edge 띄우기

```bash
cd ~/edge && docker compose up -d
```

```bash
docker compose exec proxy nginx -t
```

## 5. 확인

> edge 는 **SNI**(TLS 핸드셰이크에 실려 오는 도메인)로 server 블록을 고른다.
> `-H "Host:"` 만 붙여 `https://localhost` 로 접속하면 SNI 는 `localhost` 로 나가
> 기본 서버에 걸리고, HTTP/2 는 그 불일치를 거부해 `PROTOCOL_ERROR` 를 낸다.
> **서비스가 멀쩡해도 그렇다.** `--resolve` 로 도메인을 그대로 쓰되 접속만 루프백으로 돌린다.

```bash
curl -fsSk --resolve filehub-khs.duckdns.org:443:127.0.0.1 https://filehub-khs.duckdns.org/ >/dev/null && echo "FileHub OK"
```

브라우저에서 `https://filehub-khs.duckdns.org` 도 확인한다.

---

## 6. FileHub 재배포로 마무리

위 3단계에서 컨테이너만 수동으로 지웠으므로, 다음 배포 때 compose 가 정리된 상태로
반영되도록 한 번 돌려준다. FileHub 저장소를 push 하거나 Actions 에서 수동 실행한다.

이때 `--remove-orphans` 가 남은 흔적을 정리하고, 헬스체크는 edge 를 거쳐 확인한다.

---

## 되돌리기

edge 를 내리고 FileHub 를 이전 커밋으로 되돌린다.

```bash
cd ~/edge && docker compose down
```

```bash
cd ~/filehub && git checkout HEAD~1 -- docker-compose.prod.yaml
docker compose -f docker-compose.prod.yaml --env-file .env.prod up -d
```

인증서 원본(`/etc/filehub/certs`)은 그대로 두었으므로 바로 다시 동작한다.
**전환이 안정된 뒤에 지운다.**
