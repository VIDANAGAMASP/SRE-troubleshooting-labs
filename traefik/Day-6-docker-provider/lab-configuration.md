# Day 6 CTF — Docker Provider

## Killercoda setup

Use a fresh Ubuntu environment.

Run this complete setup:

```bash
#!/bin/bash

set -e

echo "=========================================="
echo " Traefik Day 6 - Docker Provider CTF"
echo "=========================================="

# -------------------------------------------------
# 1. Install dependencies
# -------------------------------------------------

apt-get update -qq
apt-get install -y -qq curl wget ca-certificates

# -------------------------------------------------
# 2. Install Docker
# -------------------------------------------------

if ! command -v docker >/dev/null 2>&1; then
    curl -fsSL https://get.docker.com | sh
fi

systemctl enable docker >/dev/null 2>&1 || true
systemctl start docker

# -------------------------------------------------
# 3. Install Traefik
# -------------------------------------------------

TRAEFIK_VERSION="v3.7.13"

cd /tmp

rm -f traefik.tar.gz
rm -f traefik

wget -q \
  "https://github.com/traefik/traefik/releases/download/${TRAEFIK_VERSION}/traefik_${TRAEFIK_VERSION}_linux_amd64.tar.gz" \
  -O traefik.tar.gz

tar -xzf traefik.tar.gz

install -m 755 traefik /usr/local/bin/traefik

# -------------------------------------------------
# 4. Traefik directories
# -------------------------------------------------

mkdir -p /etc/traefik
mkdir -p /var/log

# -------------------------------------------------
# 5. Static configuration
# -------------------------------------------------

cat > /etc/traefik/traefik.yml <<'EOF'
entryPoints:
  web:
    address: ":80"

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false

log:
  level: INFO
  filePath: "/var/log/traefik.log"
EOF

# -------------------------------------------------
# 6. Traefik systemd service
# -------------------------------------------------

cat > /etc/systemd/system/traefik.service <<'EOF'
[Unit]
Description=Traefik Reverse Proxy
After=docker.service
Requires=docker.service

[Service]
Type=simple
ExecStart=/usr/local/bin/traefik --configFile=/etc/traefik/traefik.yml
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
EOF

# -------------------------------------------------
# 7. Create backend Docker image
# -------------------------------------------------

mkdir -p /opt/day6-app

cat > /opt/day6-app/index.html <<'EOF'
DOCKER APPLICATION
Traefik successfully reached the container.
EOF

cat > /opt/day6-app/Dockerfile <<'EOF'
FROM python:3.12-alpine

WORKDIR /app

COPY index.html /app/index.html

EXPOSE 8080

CMD ["python3", "-m", "http.server", "8080", "--bind", "0.0.0.0"]
EOF

docker build -t day6-webapp /opt/day6-app >/dev/null

# -------------------------------------------------
# 8. Start intentionally broken container
#
# INTENTIONAL FAILURE:
#
# The container has Traefik labels, but the router
# rule is deliberately wrong.
# -------------------------------------------------

docker rm -f day6-webapp 2>/dev/null || true

docker run -d \
  --name day6-webapp \
  --label "traefik.enable=true" \
  --label 'traefik.http.routers.day6.rule=Host(`wrong.local`)' \
  --label "traefik.http.routers.day6.entrypoints=web" \
  --label "traefik.http.services.day6.loadbalancer.server.port=8080" \
  day6-webapp >/dev/null

# -------------------------------------------------
# 9. Start Traefik
# -------------------------------------------------

systemctl daemon-reload
systemctl enable traefik.service >/dev/null 2>&1
systemctl restart traefik.service

sleep 5

echo
echo "=========================================="
echo " ENVIRONMENT CREATED"
echo "=========================================="

echo
echo "Traefik:"
echo "  http://127.0.0.1:80"

echo
echo "Container:"
echo "  day6-webapp"

echo
echo "Container application port:"
echo "  8080"

echo
echo "Try:"
echo "  curl -v http://127.0.0.1"
echo
echo "Direct container test:"
echo "  docker inspect day6-webapp"
echo
echo "Container logs:"
echo "  docker logs day6-webapp"

echo
echo "=========================================="
echo " Start investigating."
echo "=========================================="
```

---

## 8. Your CTF

### Environment

You have:

```text
Client
   |
   | HTTP :80
   ↓
Traefik
   |
   | Docker Provider
   ↓
Docker container
   |
   | :8080
   ↓
Web application
```

The application is running inside Docker.

Traefik is supposed to discover the application automatically.

### Symptoms

When you access:

```bash
curl -v http://127.0.0.1
```

you don't get the expected application response.

However, you have reason to believe the Docker container itself is running.

### Your mission

Determine why Traefik isn't routing the request correctly.

You need to investigate all of these layers:

```text
Docker
   ↓
Container
   ↓
Labels
   ↓
Traefik Docker Provider
   ↓
Router
   ↓
Service
   ↓
Application
```

### Useful commands

Start with:

```bash
docker ps
```

Then:

```bash
docker inspect day6-webapp
```

Look specifically for:

```text
Labels
```

Check:

```bash
docker logs day6-webapp
```

Check Traefik:

```bash
systemctl status traefik
```

And:

```bash
tail -f /var/log/traefik.log
```

You can also inspect the container's network information:

```bash
docker inspect day6-webapp
```

Goal Reach with curl -v http:127.0.0.1:80 and curl -H "HOST: myapp.local" 127.0.0.1:80
