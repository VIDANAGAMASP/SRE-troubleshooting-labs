# Day 5 CTF — Path Routing

## Killercoda setup

Use a fresh Ubuntu Playground.

Paste this entire script:

```bash
#!/bin/bash

set -e

echo "=========================================="
echo " Traefik Day 5 CTF - Path Routing"
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
mkdir -p /opt/traefik-day5

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

cat > /opt/traefik-day5/backend.py <<'EOF'
from http.server import BaseHTTPRequestHandler, HTTPServer

class Handler(BaseHTTPRequestHandler):

    def do_GET(self):

        routes = {
            "/": "HOME APPLICATION\n",
            "/api": "API ROOT\n",
            "/api/users": "USER API\n",
            "/api/orders": "ORDER API\n",
            "/api/v1/users": "API V1 USERS\n",
        }

        if self.path in routes:
            body = routes[self.path].encode()
            status = 200
        else:
            body = b"BACKEND 404 - path not found\n"
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
# 6. Backend service
# -------------------------------------------------

cat > /etc/systemd/system/day5-backend.service <<'EOF'
[Unit]
Description=Traefik Day 5 Backend
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /opt/traefik-day5/backend.py
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# -------------------------------------------------
# 7. Dynamic configuration
#
# INTENTIONAL FAILURE:
#
# The API router uses Path() instead of
# PathPrefix().
#
# The backend supports nested /api paths,
# but the Traefik router does not match them.
# -------------------------------------------------

cat > /etc/traefik/dynamic.yml <<'EOF'
http:

  routers:

    api:
      rule: "Path(`/api`)"
      entryPoints:
        - web
      service: backend

    home:
      rule: "Path(`/`)"
      entryPoints:
        - web
      service: backend

  services:

    backend:
      loadBalancer:
        servers:
          - url: "http://127.0.0.1:8080"
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
# 9. Start services
# -------------------------------------------------

systemctl daemon-reload

systemctl enable day5-backend.service >/dev/null 2>&1
systemctl enable traefik.service >/dev/null 2>&1

systemctl restart day5-backend.service
systemctl restart traefik.service

sleep 3

echo
echo "=========================================="
echo " ENVIRONMENT CREATED"
echo "=========================================="

echo
echo "Traefik:"
echo "  127.0.0.1:80"

echo
echo "Backend:"
echo "  127.0.0.1:8080"

echo
echo "Dynamic configuration:"
echo "  /etc/traefik/dynamic.yml"

echo
echo "Backend routes include:"
echo "  /"
echo "  /api"
echo "  /api/users"
echo "  /api/orders"
echo "  /api/v1/users"

echo
echo "=========================================="
echo " Start investigating."
echo "=========================================="
```

---

## CTF Challenge

### Environment

The application has:

```text
/api
/api/users
/api/orders
/api/v1/users
```

The intended architecture is:

```text
Client
   ↓
Traefik :80
   ↓
API Router
   ↓
Backend :8080
```

### Symptoms

The API root appears to work:

```text
/api
```

But some deeper API endpoints don't behave as expected.

The backend itself is known to contain those endpoints.

Your task is to determine where the request stops matching the expected route.

### Goal

Investigate:

```bash
curl -v http://127.0.0.1:80/api

curl -v http://127.0.0.1:80/api/users

curl -v http://127.0.0.1:80/api/orders

curl -v http://127.0.0.1:80/api/v1/users
```

Compare with direct backend requests:

```bash
curl -v http://127.0.0.1:8080/api

curl -v http://127.0.0.1:8080/api/users
```

### Questions you should answer

Don't just fix it.

Determine:

1. Does the backend actually support the failing paths?
2. Does TCP connectivity to Traefik work?
3. Does Traefik receive the request?
4. Which router should match?
5. What exact rule is that router using?
6. Is the problem `Path()` versus `PathPrefix()`?
7. Is Traefik rewriting the path?
8. What does the backend actually receive?

### Restrictions

Do not use:

- Docker
- Kubernetes
- middleware
- dashboard/API
- changing the backend application
- changing the backend port
- restarting everything repeatedly

You may modify the Traefik dynamic configuration **after** you have established the cause.

### Today's troubleshooting model

Your request path is now:

```text
HTTP request
     |
     ├── Host
     |
     └── Path
          |
          ↓
      Entrypoint
          ↓
        Router
          ↓
        Service
          ↓
       Backend
```

And the key question is:

> Is the backend broken, or did Traefik never route the request to it?
