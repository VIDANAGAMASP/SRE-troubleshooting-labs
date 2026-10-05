# Day 4 — Host-Based Routing

Today we introduce the Host header and your first genuinely important Traefik routing rule:

```text
Host(`api.example.com`)
```

This is the foundation for routing multiple applications through the same IP and port.

We will not introduce Docker, Kubernetes, middleware, TLS, or the dashboard yet.

---

## 1. Knowledge needed

### Linux

Same commands you've already learned:

```text
curl
ss
cat
grep
systemctl
```

### Networking

You already know:

```text
IP address → TCP port
```

Today we add:

```text
HTTP Host header
```

Multiple domains can point to the same server:

```text
api.example.com ─┐
                  ├──→ 192.168.1.10:80
web.example.com ─┘
```

Traefik can then distinguish them based on the HTTP request.

---

## 2. HTTP Host header

Consider:

```bash
curl http://127.0.0.1/
```

The request contains approximately:

```http
GET / HTTP/1.1
Host: 127.0.0.1
```

The `Host` header tells the HTTP server which hostname the client is requesting.

You can manually change it:

```bash
curl -v -H "Host: api.example.com" http://127.0.0.1/
```

Notice something important:

```text
TCP destination:
127.0.0.1:80

HTTP Host:
api.example.com
```

Those are different things.

The TCP connection still goes to:

```text
127.0.0.1:80
```

but the HTTP request says:

```text
Host: api.example.com
```

This is extremely useful for local Traefik testing because you don't need actual DNS.

---

## 3. Traefik Host routing

A Traefik router can contain:

```text
rule: "Host(`api.example.com`)"
```

Meaning:

> Only requests whose HTTP Host header is `api.example.com` match this router.

Conceptually:

```text
                  Request
                     |
                     ↓
               Host header
                     |
             ┌───────┴────────┐
             ↓                ↓
      api.example.com    other.example.com
             |                |
             ↓                ↓
          MATCH             NO MATCH
             |
             ↓
          Router
```

---

## 4. Path vs Host

You have now seen:

```text
Path(`/api`)
```

and today:

```text
Host(`api.example.com`)
```

They answer different questions.

### Path

Which URL path was requested?

Example:

```text
/api/users
```

### Host

Which hostname was requested?

Example:

```text
api.example.com
```

You can eventually combine them:

```text
rule: "Host(`api.example.com`) && PathPrefix(`/api`)"
```

But we won't use combined rules yet.

---

## 5. Why Host routing matters

Imagine one server:

```text
                     Traefik :80
                         |
            ┌────────────┼────────────┐
            ↓            ↓            ↓
       api.example   shop.example   blog.example
            ↓            ↓            ↓
         API           Shop          Blog
```

All three applications can use:

```text
TCP port 80
```

Traefik determines where the request goes based on:

```text
Host header
```

This is one of the main reasons reverse proxies are useful.

---

## 6. Testing Host routing

This is the most important command today:

```bash
curl -v -H "Host: api.example.com" http://127.0.0.1/
```

And:

```bash
curl -v -H "Host: shop.example.com" http://127.0.0.1/
```

The TCP destination is identical.

Only the HTTP Host header changes.

---

## 7. Traefik Note — Router matching

Think of the router as a gate:

```text
HTTP request
     |
     ↓
┌──────────────────────┐
│ Router rule          │
│                      │
│ Host(api.example)    │
└──────────┬───────────┘
           |
       Does it match?
        /        \
      YES         NO
       |           |
       ↓           ↓
   Service       No route
```

This means a request can successfully reach Traefik but still fail to reach an application.

That's another layer distinction:

```text
TCP works
   ↓
HTTP reaches Traefik
   ↓
Router doesn't match
```

---

## 8. Commands for today

**Test without Host override**

```bash
curl -v http://127.0.0.1/
```

**Test a specific Host**

```bash
curl -v -H "Host: api.example.com" http://127.0.0.1/
```

**Test another Host**

```bash
curl -v -H "Host: web.example.com" http://127.0.0.1/
```

**Inspect listeners**

```bash
ss -tlnp
```

**Inspect dynamic configuration**

```bash
cat /etc/traefik/dynamic.yml
```

**Inspect Traefik logs**

```bash
cat /var/log/traefik.log
```
