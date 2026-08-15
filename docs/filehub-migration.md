# FileHub 를 edge 뒤로 옮기기

FileHub 는 지금 자체 `filehub-proxy` 로 80/443 을 점유하고 TLS 를 끝낸다.
edge 를 도입하면 그 역할이 겹치므로 **`proxy` 서비스를 제거**하고,
`frontend` 를 공용 네트워크에 붙여 edge 가 찾아갈 수 있게 한다.

> 운영 중인 서비스를 건드리는 작업이다. 아래 순서대로 하면 다운타임은 수십 초다.

---

## 바뀌는 것

```
[전]  인터넷 → filehub-proxy(:443, TLS) → filehub-frontend → filehub-backend
[후]  인터넷 → edge-proxy(:443, TLS) ──→ filehub-frontend → filehub-backend
```

`filehub-frontend` 이하는 그대로다. **앞단만 바뀐다.**

---

## 1. `docker-compose.prod.yaml` 수정

### 1-1. `frontend` 서비스에 네트워크 추가

```yaml
  frontend:
    image: ${DOCKERHUB_USERNAME}/filehub-frontend:${IMAGE_TAG:-latest}
    container_name: filehub-frontend
    restart: unless-stopped
    depends_on:
      - backend
    networks:            # ← 추가
      - default          #   backend 를 부르기 위한 기존 내부망
      - edge-net         #   edge 가 찾아오는 공용망
```

`container_name: filehub-frontend` 는 이미 있다. **edge 가 이 이름으로 찾아가므로 바꾸면 안 된다.**

### 1-2. `proxy` 서비스 통째로 제거

```yaml
  # proxy:
  #   image: nginx:alpine
  #   ...
  #   ports:
  #     - "80:80"
  #     - "443:443"
```

### 1-3. `configs:` 블록 통째로 제거

`proxy_conf` 는 위 proxy 전용이었다. 남겨두면 쓰이지 않는 설정만 떠돈다.

### 1-4. 파일 맨 아래에 네트워크 선언 추가

```yaml
networks:
  default:
  edge-net:
    name: edge-net
    external: true
```

---

## 2. 배포 워크플로 수정

`.github/workflows/deploy.yml` 의 헬스체크가 `https://localhost/` 를 본다.
edge 로 옮기면 도메인 기반 라우팅이라 Host 헤더가 필요하다.

```bash
curl -fsSk -H "Host: 실제도메인" https://localhost/
```

또 edge 네트워크가 없으면 `up` 이 실패하므로, 배포 스크립트에 한 줄 넣어 둔다.

```bash
docker network inspect edge-net >/dev/null 2>&1 || docker network create edge-net
```

---

## 3. 서버에서 실행

edge 를 먼저 올려야 한다. FileHub 의 proxy 를 내리는 순간 443 이 비기 때문이다.

```bash
docker network create edge-net
```

### 3-1. 인증서 옮기기

FileHub 인증서는 지금 `/etc/filehub/certs` 에 있다. edge 구조에 맞춰 복사한다.

```bash
sudo mkdir -p /etc/edge/certs/filehub /etc/edge/certs/default
sudo cp /etc/filehub/certs/*.pem /etc/edge/certs/filehub/
sudo cp /etc/filehub/certs/*.pem /etc/edge/certs/default/
```

acme.sh 의 `--install-cert` 대상 경로도 새 위치로 바꾼다. 안 바꾸면
**갱신은 되는데 edge 는 옛 인증서를 계속 물고 있다가 어느 날 만료된다.**

```bash
acme.sh --install-cert -d 실제도메인 \
  --key-file       /etc/edge/certs/filehub/privkey.pem \
  --fullchain-file /etc/edge/certs/filehub/fullchain.pem \
  --reloadcmd      "docker exec edge-proxy nginx -s reload"
```

### 3-2. 전환

```bash
cd ~/filehub && docker compose -f docker-compose.prod.yaml --env-file .env.prod up -d --remove-orphans
```

`--remove-orphans` 가 있어야 compose 에서 지운 `filehub-proxy` 가 실제로 내려간다.
이게 없으면 컨테이너가 남아 443 을 계속 잡고 있어 edge 가 못 뜬다.

```bash
cd ~/edge && docker compose up -d
```

### 3-3. 확인

```bash
docker compose -f ~/edge/docker-compose.yml exec proxy nginx -t
```

```bash
curl -fsSk -H "Host: 실제도메인" https://localhost/ >/dev/null && echo OK
```

---

## 되돌리기

문제가 생기면 edge 를 내리고 FileHub 의 이전 compose 로 되돌린다.

```bash
cd ~/edge && docker compose down
cd ~/filehub && git checkout docker-compose.prod.yaml && docker compose -f docker-compose.prod.yaml --env-file .env.prod up -d
```

인증서 원본(`/etc/filehub/certs`)을 지우지 않았으므로 그대로 다시 동작한다.
**전환이 확실히 안정된 뒤에 지운다.**
