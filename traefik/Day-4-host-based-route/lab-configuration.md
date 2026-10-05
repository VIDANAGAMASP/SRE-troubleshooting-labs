# Day 4 CTF — Host Routing

## Killercoda setup

Use a fresh Ubuntu Playground.

Paste the entire script:

```bash
#!/bin/bash

set -e

echo "=========================================="
echo " Traefik Day 4 CTF - Host Routing"
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
# 3. Create directories
# -------------------------------------------------

mkdir -p /etc/traefik
mkdir -p /var/log
mkdir -p /opt/traefik-day4

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

cat > /opt/traefik-day4/backend.py <<'EOF'
from http.server import BaseHTTPRequestHandler, HTTPServer

class Handler(BaseHTTPRequestHandler):

    def do_GET(self):

        host = self.headers.get("Host", "")

        if host == "api.example.com":
            body = b"API APPLICATION\n"
            status = 200

        elif host == "web.example.com":
            body = b"WEB APPLICATION\n"
            status = 200

        else:
            body = b"Backend received an unknown Host\n"
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

cat > /etc/systemd/system/day4-backend.service <<'EOF'
[Unit]
Description=Traefik Day 4 Backend
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /opt/traefik-day4/backend.py
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# -------------------------------------------------
# 7. Dynamic configuration
#
# Two host-based routers.
#
# INTENTIONAL FAILURE:
# One Host rule contains an incorrect hostname.
# -------------------------------------------------

cat > /etc/traefik/dynamic.yml <<'EOF'
http:

  routers:

    api:
      rule: "Host(`api.example.com`)"
      entryPoints:
        - web
      service: backend

    web:
      rule: "Host(`www.example.com`)"
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
# 8. Traefik service
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

systemctl enable day4-backend.service >/dev/null 2>&1
systemctl enable traefik.service >/dev/null 2>&1

systemctl restart day4-backend.service
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
echo "Expected hosts:"
echo "  api.example.com"
echo "  web.example.com"

echo
echo "Dynamic configuration:"
echo "  /etc/traefik/dynamic.yml"

echo
echo "=========================================="
echo " Start investigating."
echo "=========================================="
```

---

## CTF Challenge

### Environment

You have:

```text
                         Traefik :80
                              |
                    ┌─────────┴─────────┐
                    ↓                   ↓
             Host: api.example   Host: web.example
                    ↓                   ↓
                  Router              Router
                    \                   /
                     \                 /
                      └──────┬────────┘
                             ↓
                       Backend :8080
```

Both applications are supposed to be reachable through Traefik.

### Symptoms

One hostname works as expected.

Another hostname does not.

You have been told:

> "The server is up, Traefik is running, and the backend is healthy."

Your job is to determine whether that statement is actually sufficient to explain the problem.

### Target

Test at least:

```bash
curl -v -H "Host: api.example.com" http://127.0.0.1/

curl -v -H "Host: web.example.com" http://127.0.0.1/
```

Also test an unknown host:

```bash
curl -v -H "Host: unknown.example.com" http://127.0.0.1/
```

And test the backend directly where useful.

### Goal

Determine:

1. Why one hostname works.
2. Why the other doesn't.
3. Whether the failure is TCP, HTTP, router matching, service, or backend.
4. Which configuration controls the routing decision.
5. Fix the problem.
6. Verify both intended hosts end-to-end.

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
- DNS changes
- `/etc/hosts` changes
- blindly restarting Traefik

You don't need real DNS. The Host header is enough.

### The key question

When you see:

```text
curl → 127.0.0.1:80
```

and it connects successfully, don't stop there.

Ask:

> What HTTP Host did I actually send, and which Traefik router should match it?

Your troubleshooting chain is now:

```text
TCP
 ↓
Entrypoint
 ↓
HTTP request
 ↓
Host header
 ↓
Router rule
 ↓
Service
 ↓
Backend
```
