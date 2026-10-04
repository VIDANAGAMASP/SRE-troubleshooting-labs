# Day 2 — Entrypoint → Router → Service → Backend

Day 1 taught you how to determine whether Traefik itself is alive and listening.

Today we move one layer deeper.

You will learn the most important Traefik mental model:

```text
Client
  ↓
Entrypoint :80
  ↓
Router
  ↓
Service
  ↓
Backend
```

The lab will have one backend and one routing failure. No Docker, no Kubernetes, no middleware yet.

---

## 1. Knowledge needed

### Linux

You need:

```text
ps
ss
curl
grep
cat
systemctl
```

You already know these from Day 1.

### Networking

You need to understand that there are now two separate connections:

```text
Connection 1

curl
 ↓
Traefik :80
```

and:

```text
Connection 2

Traefik
 ↓
Backend :8080
```

This distinction is extremely important.

**Traefik can be perfectly reachable while the backend is completely broken.**

For example:

```text
curl
 ↓
Traefik :80       ← WORKS
 ↓
Router            ← WORKS
 ↓
Service           ← WORKS
 ↓
Backend :8080     ← BROKEN
```

In that situation, restarting Traefik doesn't solve the actual problem.

---

## 2. Traefik Note

### The four fundamental pieces

Today we introduce:

#### 1. Entrypoint

The entrypoint accepts the incoming connection.

```text
:80
 ↓
web
```

#### 2. Router

A router decides: *"Does this HTTP request belong to this route?"*

For example:

```yaml
http:
  routers:
    myapp:
      rule: "PathPrefix(`/`)"
      service: myapp
```

Conceptually:

```text
HTTP request
     ↓
Router
     ↓
Does the rule match?
     ↓
Yes
     ↓
Send to myapp service
```

Don't worry about complex rules yet. We're using the simplest possible rule today.

#### 3. Service

A Traefik service represents the destination behind the router.

Conceptually:

```text
Router
   ↓
Service
   ↓
Backend
```

A service can eventually contain multiple backend servers:

```text
             ┌── Backend 1
Router → Service ── Backend 2
             └── Backend 3
```

That's how load balancing eventually fits into the model.

#### 4. Backend

The backend is the actual application/server receiving the request.

Today's backend will simply be a tiny local HTTP server:

```text
127.0.0.1:8080
```

So the complete path is:

```text
curl
  ↓
127.0.0.1:80
  ↓
Traefik web entrypoint
  ↓
Router
  ↓
Service
  ↓
127.0.0.1:8080
  ↓
Backend
```

---

## 3. Static vs Dynamic Configuration

This is where the distinction becomes important.

### Static

Controls Traefik itself:

```text
entryPoints:
providers:
api:
log:
```

For example:

```yaml
entryPoints:
  web:
    address: ":80"
```

### Dynamic

Controls HTTP routing:

```text
http:
  routers:
  services:
  middlewares:
```

Today's router and service will be supplied through a **file provider**.

That means:

```text
Static configuration
       ↓
Traefik
       ↓
File provider
       ↓
Dynamic configuration
       ↓
Router
       ↓
Service
       ↓
Backend
```

This is your first exposure to a provider.

---

## 4. Traefik Theory — File Provider

A provider is a source from which Traefik obtains dynamic configuration.

Later you'll learn:

```text
Providers
   ├── Docker
   ├── Kubernetes
   └── File
```

Today:

```text
File
 ↓
dynamic.yml
 ↓
router + service
```

The static configuration tells Traefik:

```yaml
providers:
  file:
    filename: /etc/traefik/dynamic.yml
```

The dynamic configuration can then contain:

```yaml
http:
  routers:
  services:
```

This gives us a clean separation:

```text
/etc/traefik/traefik.yml
        │
        │ static
        ↓
     Traefik
        ↑
        │ dynamic
/etc/traefik/dynamic.yml
```

---

## 5. Commands for today

**Check Traefik**

```bash
traefik version
```

**Check listeners**

```bash
ss -tlnp
```

**Test Traefik**

```bash
curl -v http://127.0.0.1:80/
```

**Test the backend directly**

```bash
curl -v http://127.0.0.1:8080/
```

This is particularly important.

You're comparing:

```text
Backend directly
```

against:

```text
Backend through Traefik
```

If this works:

```bash
curl http://127.0.0.1:8080/
```

but this fails:

```bash
curl http://127.0.0.1:80/
```

then your backend itself may be fine.

That narrows the investigation toward Traefik.

**Inspect configuration**

```bash
cat /etc/traefik/traefik.yml
```

and:

```bash
cat /etc/traefik/dynamic.yml
```

**Check logs**

```bash
cat /var/log/traefik.log
```

or:

```bash
tail -f /var/log/traefik.log
```
