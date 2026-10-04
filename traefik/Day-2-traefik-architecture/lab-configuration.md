# Day 2 Backend Routing Failure

## Killercoda Setup

Open a fresh Ubuntu Playground again:

[Killercoda Ubuntu Playground](https://killercoda.com/playgrounds/scenario/ubuntu)

Paste the entire script:

```bash
#!/bin/bash

set -e

echo "=========================================="
echo " Traefik Day 2 CTF - Environment Setup"
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

echo "[1/8] Downloading Traefik ${TRAEFIK_VERSION}..."

wget -q --show-progress \
  "https://github.com/traefik/traefik/releases/download/${TRAEFIK_VERSION}/traefik_${TRAEFIK_VERSION}_linux_amd64.tar.gz" \
  -O traefik.tar.gz

echo "[2/8] Extracting..."

tar -xzf traefik.tar.gz

echo "[3/8] Installing..."

install -m 755 traefik /usr/local/bin/traefik

traefik version

# -------------------------------------------------
# 3. Directories
# -------------------------------------------------

mkdir -p /etc/traefik
mkdir -p /var/log

# -------------------------------------------------
# 4. STATIC CONFIGURATION
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
# 5. DYNAMIC CONFIGURATION
#
# Intentional CTF failure:
# The router points to a service named "backend".
# The service definition intentionally points to
# the WRONG backend port.
# -------------------------------------------------

cat > /etc/traefik/dynamic.yml <<'EOF'
http:

  routers:

    myapp:
      rule: "PathPrefix(`/`)"
      entryPoints:
        - web
      service: backend

  services:

    backend:
      loadBalancer:
        servers:
          - url: "http://127.0.0.1:9999"
EOF

# -------------------------------------------------
# 6. Create backend application
# -------------------------------------------------

mkdir -p /opt/traefik-day2

cat > /opt/traefik-day2/backend.py <<'EOF'
from http.server import BaseHTTPRequestHandler, HTTPServer

class Handler(BaseHTTPRequestHandler):

    def do_GET(self):
        body = b"Hello from the Day 2 backend!\n"

        self.send_response(200)
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
# 7. Backend systemd service
# -------------------------------------------------

cat > /etc/systemd/system/day2-backend.service <<'EOF'
[Unit]
Description=Traefik Day 2 Backend
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /opt/traefik-day2/backend.py
Restart=on-failure

[Install]
WantedBy=multi-user.target
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
# Start services
# -------------------------------------------------

systemctl daemon-reload

systemctl enable day2-backend.service >/dev/null 2>&1
systemctl enable traefik.service >/dev/null 2>&1

systemctl restart day2-backend.service
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
echo "Traefik static config:"
echo "  /etc/traefik/traefik.yml"

echo
echo "Traefik dynamic config:"
echo "  /etc/traefik/dynamic.yml"

echo
echo "Backend:"
echo "  127.0.0.1:8080"

echo
echo "Traefik:"
echo "  127.0.0.1:80"

echo
echo "Services:"
echo "  traefik.service"
echo "  day2-backend.service"

echo
echo "=========================================="
echo " ENVIRONMENT CREATED"
echo "=========================================="
echo
echo "The environment contains an intentional"
echo "Traefik routing/backend failure."
echo
echo "Start investigating."
echo
```

---

Challenge

### Environment

You have:

```text
                    ┌──────────────────┐
                    │ Python Backend   │
                    │ 127.0.0.1:8080   │
                    └────────▲─────────┘
                             │
                             │
Client → Traefik :80 → Router → Service
```

Traefik is using:

```text
Static configuration
        ↓
File provider
        ↓
Dynamic configuration
```

### Symptoms

The backend application is supposed to be available through Traefik.

But something isn't working correctly.

Your first task is to determine where the request path breaks.

### Target

This should eventually succeed:

```bash
curl -v http://127.0.0.1:80/
```

and return the backend's response.

You should also be able to access the backend directly:

```bash
curl -v http://127.0.0.1:8080/
```

### Goal

Determine why:

```text
Client
 ↓
Traefik
 ↓
Router
 ↓
Service
 ↓
Backend
```

is not producing the expected response.

Fix the problem and verify the complete request path.

### Restrictions

You may use:

```text
systemctl
ps
ss
curl
cat
grep
tail
```

You may inspect both Traefik configuration files.

Do not:

- reinstall Traefik
- recreate the environment
- use Docker
- use Kubernetes
- add middleware
- use the dashboard/API yet
- blindly restart everything

Most importantly:

**Test the backend directly before assuming Traefik is the problem.**
