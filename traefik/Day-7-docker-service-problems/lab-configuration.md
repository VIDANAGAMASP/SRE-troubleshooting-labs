# Day 7 CTF — Docker 502 Bad Gateway

## Challenge architecture

You'll have:

```text
                    Docker

             +-------------------+
             |                   |
             |     Traefik       |
             |       :80         |
             |                   |
             +---------+---------+
                       |
                       |
                  ??? network
                       |
             +---------+---------+
             |                   |
             |    webapp         |
             |                   |
             |      :8080        |
             +-------------------+
```

The container exists.

The labels exist.

The router exists.

But something in the backend connectivity path is wrong.

---

## 8. Killercoda setup

Use a fresh Ubuntu environment.

Run:

```bash
#!/bin/bash
set -e

TRAEFIK_VERSION="v3.7.13"

apt-get update
apt-get install -y wget curl docker.io

systemctl enable --now docker

# --------------------------------------------------
# Install Traefik
# --------------------------------------------------

cd /tmp

wget -q \
  "https://github.com/traefik/traefik/releases/download/${TRAEFIK_VERSION}/traefik_${TRAEFIK_VERSION}_linux_amd64.tar.gz" \
  -O traefik.tar.gz

tar -xzf traefik.tar.gz traefik
install -m 755 traefik /usr/local/bin/traefik

# --------------------------------------------------
# Clean previous lab
# --------------------------------------------------

docker rm -f day7-webapp day7-traefik 2>/dev/null || true
docker network rm traefik-net backend-net 2>/dev/null || true

# --------------------------------------------------
# Networks
# --------------------------------------------------

docker network create traefik-net
docker network create backend-net

# --------------------------------------------------
# Backend
#
# IMPORTANT:
# Backend is ONLY connected to backend-net.
# --------------------------------------------------

docker run -d \
  --name day7-webapp \
  --network backend-net \
  python:3.12-alpine \
  sh -c '
    mkdir -p /app &&
    printf "%s\n" \
      "<html><body>" \
      "<h1>DAY 7 APPLICATION</h1>" \
      "<p>Traefik reached the Docker backend successfully.</p>" \
      "</body></html>" \
      > /app/index.html &&
    cd /app &&
    python3 -m http.server 8080
  '

# --------------------------------------------------
# Traefik
#
# Traefik is ONLY connected to traefik-net.
#
# It can discover the container through Docker API,
# but it CANNOT reach the container because they have
# no common Docker network.
# --------------------------------------------------

cat > /tmp/traefik.yml <<'EOF'
entryPoints:
  web:
    address: ":80"

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false

log:
  level: DEBUG
EOF

docker run -d \
  --name day7-traefik \
  --network traefik-net \
  -p 80:80 \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -v /tmp/traefik.yml:/etc/traefik/traefik.yml:ro \
  traefik:${TRAEFIK_VERSION} \
  --configFile=/etc/traefik/traefik.yml

# --------------------------------------------------
# Add correct Traefik labels to backend
# --------------------------------------------------

docker stop day7-webapp
docker rm day7-webapp

docker run -d \
  --name day7-webapp \
  --network backend-net \
  --label traefik.enable=true \
  --label 'traefik.http.routers.day7.rule=Path(`/`)' \
  --label traefik.http.services.day7.loadbalancer.server.port=8080 \
  python:3.12-alpine \
  sh -c '
    mkdir -p /app &&
    printf "%s\n" \
      "<html><body>" \
      "<h1>DAY 7 APPLICATION</h1>" \
      "<p>Traefik reached the Docker backend successfully.</p>" \
      "</body></html>" \
      > /app/index.html &&
    cd /app &&
    python3 -m http.server 8080
  '

sleep 5

echo
echo "=========================================="
echo " DAY 7 LAB READY"
echo "=========================================="
echo
echo "Test:"
echo "  curl -v http://127.0.0.1/"
echo
echo "Expected initial result:"
echo "  HTTP 502 Bad Gateway"
echo
echo "Backend:"
echo "  docker inspect day7-webapp"
echo
echo "Traefik:"
echo "  docker logs day7-traefik"
echo
echo "Networks:"
echo "  docker network ls"
echo "  docker network inspect traefik-net"
echo "  docker network inspect backend-net"
echo
```

---

## 9. CTF Challenge

### Symptoms

You have a running Traefik instance and a running Docker application.

The Docker application should be available through:

```text
http://127.0.0.1
```

But the request fails with a `502 Bad Gateway`.

Your job is to determine why.

### Your target

Make:

```bash
curl -v http://127.0.0.1
```

return:

```text
DAY 7 APPLICATION
```

### Investigate systematically

Start with:

```bash
docker ps
```

Then:

```bash
docker inspect day7-webapp
```

Look at:

```text
Labels
Networks
IP address
```

Inspect:

```bash
docker network ls
```

Then:

```bash
docker network inspect traefik-net
```

and:

```bash
docker network inspect backend-net
```

Check Traefik:

```bash
systemctl status traefik
```

Then:

```bash
tail -f /var/log/traefik.log
```

You should pay particular attention to what Traefik says about the backend connection.

### Restrictions

You may:

- inspect Docker
- inspect networks
- inspect container configuration
- inspect labels
- inspect Traefik logs
- modify Docker networking
- reconnect a container to a network
- restart the affected container

You may not:

- change the application
- change the application port
- replace Traefik
- use Kubernetes
- use the dashboard/API
- create a file-provider configuration
- randomly recreate everything

### The key question

Don't just say:

> "It's a 502."

Prove:

```text
Is Traefik running?             ✓ / ✗
        ↓
Did the router match?            ✓ / ✗
        ↓
Did Traefik discover service?    ✓ / ✗
        ↓
Is container running?            ✓ / ✗
        ↓
Is application listening?        ✓ / ✗
        ↓
Can Traefik reach the container? ✓ / ✗
```
