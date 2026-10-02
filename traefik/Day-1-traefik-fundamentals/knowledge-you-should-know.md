# Day 1 — Traefik Fundamentals + First Deployment

We'll start very gently. Today you only need to understand:

```text
Linux process → Traefik → entrypoint → HTTP request → basic logs
```

We will not use Docker labels, routers, middleware, Kubernetes, TLS, or dynamic service discovery yet. Those come later.

---

## 1. Knowledge needed

### A. Linux

You already know the basics. Today we need only:

- processes
- listening ports
- configuration files
- logs
- `curl`
- `ss`
- `ps`

Useful commands:

```bash
ps aux
ss -tlnp
curl -v http://127.0.0.1:80
```

### B. Networking

You only need one idea:

> A server application must listen on an IP address and port before it can receive TCP connections.

For example:

```text
Client
  |
  | TCP connection
  v
127.0.0.1:80
  |
  v
Traefik
```

If nothing is listening on port 80:

```bash
curl http://127.0.0.1:80
```

will fail before HTTP routing is even relevant.

This gives us our first troubleshooting layer:

```text
TCP connection
      ↓
Is something listening?
      ↓
Traefik
```

---

## 2. Traefik Fundamentals

### What is Traefik?

At its simplest, Traefik is a program that receives requests and decides where those requests should go.

Imagine your server has:

```text
                 ┌── Application A
Internet ──→ Traefik
                 └── Application B
```

Instead of clients connecting directly to every application, they connect to Traefik.

Traefik then determines which backend should receive the request.

### Reverse proxy

A normal client/server request looks like:

```text
Client ───────────────→ Application
```

With a reverse proxy:

```text
Client ───→ Reverse Proxy ───→ Application
```

Traefik is the reverse proxy.

The client doesn't necessarily know which backend application actually handles the request.

---

## 3. Entrypoint

An entrypoint is a network listener in Traefik.

For example:

```yaml
entryPoints:
  web:
    address: ":80"
```

means:

```text
Traefik
   |
   └── web
        |
        └── TCP :80
```

So if Traefik starts successfully with that configuration, you should see something listening on port 80.

Conceptually:

```text
Client
   |
   | TCP :80
   v
Traefik
   |
   v
web entrypoint
```

### Important distinction

An entrypoint is not an application.

It is the place where Traefik accepts incoming connections.

Later we'll put routers behind it:

```text
Client
   ↓
web :80
   ↓
Router
   ↓
Service
   ↓
Backend
```

But today we stop at the entrypoint.

---

## 4. Static configuration

This is your first important Traefik configuration concept.

Traefik has two broad configuration layers:

```text
                 Traefik
                    |
          ┌─────────┴─────────┐
          ↓                   ↓
      Static config       Dynamic config
```

Today we only use static configuration.

Static configuration controls things about Traefik itself, such as:

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

This tells Traefik:

> Start an entrypoint called `web` and listen on port 80.

Later we'll learn that routers/services/middlewares belong to the dynamic configuration side.

---

## 5. Basic request path

For Day 1, simplify the full architecture to:

```text
curl
  |
  | HTTP request
  v
127.0.0.1:80
  |
  v
Traefik
  |
  v
web entrypoint
```

Eventually you'll learn the complete path:

```text
Client
   ↓
IP / DNS
   ↓
Traefik entrypoint
   ↓
Router
   ↓
Middleware
   ↓
Traefik Service
   ↓
Backend
   ↓
Application
```

But don't worry about the bottom half yet.

---

## 6. Commands for today

You are allowed to use these commands.

**Check whether Traefik exists**

```bash
traefik version
```

**Check whether Traefik is running**

```bash
ps aux | grep traefik
```

**Check listening ports**

```bash
ss -tlnp
```

Or specifically:

```bash
ss -tlnp | grep ':80'
```

**Test HTTP**

```bash
curl -v http://127.0.0.1:80
```

The `-v` is important.

It shows information such as:

```text
* Connected to 127.0.0.1
> GET /
> Host: 127.0.0.1
< HTTP/1.1 ...
```

Don't worry about every line yet. We'll progressively learn how to read it.

**Check logs**

For today's setup:

```bash
cat /var/log/traefik.log
```

and:

```bash
tail -f /var/log/traefik.log
```
