# Day 3 CTF — HTTP Troubleshooting

## Lab overview

This time the environment contains three HTTP paths:

```text
                 Traefik :80
                      |
             ┌────────┼────────┐
             ↓        ↓        ↓
            /       /api     /broken
             |        |        |
             ↓        ↓        ↓
          backend   backend   broken
```

Your job is not simply to "make curl return 200."

You must determine what each HTTP response tells you.

---

## 9. Killercoda setup

Use a fresh Ubuntu Playground.

Paste the complete script:

```bash
#!/bin/bash

set -e

echo "=========================================="
echo " Traefik Day 3 CTF - HTTP Troubleshooting"
echo "=========================================="

# -------------------------------------------------
# 1. Install dependencies
# -------------------------------------------------

apt-get update -qq
apt-get install -y -qq curl wget ca-certificates python3

# -------------------------------------------------
# 2. Install Traefik
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
# 3. Directories
# -------------------------------------------------

mkdir -p /etc/traefik
mkdir -p /var/log
mkdir -p /opt/traefik-day3

# -------------------------------------------------
# 4. Static configuration
# -------------------------------------------------

cat > /etc/traefik/traefik.yml <<'EOF'
entryPoints:
  web:
    address: ":80"

providers:
  file:
    filename: "/etc/traefik/dynamic.yml"
    watch: true

log:
  level: INFO
  filePath: "/var/log/traefik.log"
EOF

# -------------------------------------------------
# 5. Backend application
# -------------------------------------------------

cat > /opt/traefik-day3/backend.py <<'EOF'
from http.server import BaseHTTPRequestHandler, HTTPServer

class Handler(BaseHTTPRequestHandler):

    def do_GET(self):

        if self.path == "/":
            body = b"Day 3 backend: ROOT OK\n"
            status = 200

        elif self.path == "/api":
            body = b"Day 3 backend: API OK\n"
            status = 200

        else:
            body = b"Backend: resource not found\n"
            status = 404

        self.send_response(status)
        self.send_header("Content-Type", "text/plain")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()

        self.wfile.write(body)

    def log_message(self, format, *args):
        print("[BACKEND]", format % args)

server = HTTPServer(("127.0.0.1", 8080), Handler)

print("Backend listening on 127.0.0.1:8080")

server.serve_forever()
EOF

# -------------------------------------------------
# 6. Backend systemd service
# -------------------------------------------------

cat > /etc/systemd/system/day3-backend.service <<'EOF'
[Unit]
Description=Traefik Day 3 Backend
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /opt/traefik-day3/backend.py
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# -------------------------------------------------
# 7. Dynamic Traefik configuration
#
# There are intentionally multiple routing cases.
# -------------------------------------------------

cat > /etc/traefik/dynamic.yml <<'EOF'
http:

  routers:

    root:
      rule: "Path(`/`)"
      entryPoints:
        - web
      service: backend

    api:
      rule: "Path(`/api`)"
      entryPoints:
        - web
      service: backend

    broken:
      rule: "Path(`/broken`)"
      entryPoints:
        - web
      service: broken-service

  services:

    backend:
      loadBalancer:
        servers:
          - url: "http://127.0.0.1:8080"

    broken-service:
      loadBalancer:
        servers:
          - url: "http://127.0.0.1:9999"
EOF

# -------------------------------------------------
# 8. Traefik systemd service
# -------------------------------------------------

cat > /etc/systemd/system/traefik.service <<'EOF'
[Unit]
Description=Traefik Reverse Proxy
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/traefik --configFile=/etc/traefik/traefik.yml
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
EOF

# -------------------------------------------------
# 9. Start everything
# -------------------------------------------------

systemctl daemon-reload

systemctl enable day3-backend.service >/dev/null 2>&1
systemctl enable traefik.service >/dev/null 2>&1

systemctl restart day3-backend.service
systemctl restart traefik.service

sleep 3

echo
echo "=========================================="
echo " ENVIRONMENT CREATED"
echo "=========================================="

echo
echo "Traefik:"
traefik version

echo
echo "Traefik:"
echo "  http://127.0.0.1:80"

echo
echo "Backend:"
echo "  http://127.0.0.1:8080"

echo
echo "Static configuration:"
echo "  /etc/traefik/traefik.yml"

echo
echo "Dynamic configuration:"
echo "  /etc/traefik/dynamic.yml"

echo
echo "Services:"
echo "  traefik.service"
echo "  day3-backend.service"

echo
echo "=========================================="
echo " ENVIRONMENT CREATED"
echo "=========================================="
echo
echo "Investigate the HTTP behaviour."
echo
```

---

## 10. CTF Challenge

### Environment

You have:

```text
                    Traefik :80
                         |
              ┌──────────┼──────────┐
              ↓          ↓          ↓
             /          /api      /broken
              |          |           |
              └─────┬────┘           X
                    ↓
             Backend :8080
```

### Symptoms

There are different behaviours depending on the URL.

You must investigate them rather than assuming they all have the same cause.

### Target

Determine why the different requests produce different HTTP responses.

Test at least:

```bash
curl -v http://127.0.0.1:80/

curl -v http://127.0.0.1:80/api

curl -v http://127.0.0.1:80/broken
```

Also compare them against:

```bash
curl -v http://127.0.0.1:8080/
```

and appropriate backend paths.

### Goal

For each failure, determine:

1. Did TCP connectivity work?
2. Did the request reach Traefik?
3. Did a router match?
4. Which service did the router select?
5. Could that service reach its backend?
6. Was the final response generated by Traefik or the backend?

Your answer should distinguish between:

```text
Traefik-generated response
```

and:

```text
Backend-generated response
```

That's an important SRE skill.

### Restrictions

You may use:

```text
curl
ss
ps
systemctl
cat
grep
tail
```

You may inspect:

```text
/etc/traefik/traefik.yml
/etc/traefik/dynamic.yml
```

Do not use:

- Docker
- Kubernetes
- dashboard/API
- middleware
- modifying the configuration before you understand the failure
- repeatedly restarting services

### Your troubleshooting objective

Build the request path for each test:

```text
curl
 ↓
TCP :80
 ↓
Entrypoint
 ↓
HTTP request
 ↓
Router
 ↓
Service
 ↓
Backend
 ↓
HTTP response
```

For every response, ask:

> "Who generated this response?"

Goal:

>All endpoints should reach backend and return a response from backend
