# Day 8 — Traefik Dashboard + API

So far you've been troubleshooting Traefik mostly from the outside:

```text
curl
docker
systemctl
logs
config files
```

Today you learn how to inspect what Traefik itself currently knows.

This is important because there is a difference between:

> What you think you configured

and:

> What Traefik actually loaded at runtime.

---

## 1. New concept: Traefik API

Traefik has an internal API that exposes runtime information such as:

```text
Entrypoints
Routers
Services
Middlewares
Providers
```

Conceptually:

```text
                 Traefik
              /           \
             /             \
       HTTP traffic       API
           ↓                ↓
       Routers          Runtime state
           ↓
       Services
           ↓
        Backend
```

The API is primarily a visibility/troubleshooting tool in today's lab.

---

## 2. Dashboard vs API

They are related but different.

### API

Returns structured information.

For example, conceptually:

```text
/api/http/routers
/api/http/services
```

You can query it using:

```text
curl
```

### Dashboard

Provides a human-friendly web interface showing things like:

```text
HTTP
 ├── Routers
 ├── Services
 └── Middlewares
```

For today's CTF, use the API first.

---

## 3. Enabling the API

Static configuration can contain:

```yaml
api:
  dashboard: true
```

This enables the dashboard.

For the API itself:

```yaml
api:
  dashboard: true
```

Traefik exposes its internal API through the API/dashboard mechanism.

For a simple local lab, we'll expose it on a dedicated entrypoint.

---

## 4. Why this matters for troubleshooting

Imagine:

```bash
curl http://127.0.0.1
```

returns `404`.

You inspect:

```text
/etc/traefik/dynamic.yml
```

and think:

> "Router looks correct."

But perhaps:

- the file wasn't loaded
- the router has a different name
- the rule isn't what you expected
- the service wasn't discovered
- the Docker provider generated something unexpected

Runtime inspection can answer:

> What does Traefik actually know right now?

---

## 5. Important mental model

Before today:

```text
Configuration
     ↓
Traefik
     ↓
Traffic
```

Now:

```text
                 ┌──→ Traffic
                 │
Configuration → Traefik
                 │
                 └──→ API → Runtime state
```

The API gives you a window into Traefik's internal state.

---

## 6. Useful commands

Check whether Traefik is running:

```bash
systemctl status traefik
```

Check listening sockets:

```bash
ss -tlnp
```

Query the API:

```bash
curl -s http://127.0.0.1:8080/api/http/routers
```

Pretty-print JSON:

```bash
curl -s http://127.0.0.1:8080/api/http/routers | python3 -m json.tool
```

Services:

```bash
curl -s http://127.0.0.1:8080/api/http/services | python3 -m json.tool
```

Middlewares:

```bash
curl -s http://127.0.0.1:8080/api/http/middlewares | python3 -m json.tool
```

You can also use:

```bash
curl -s http://127.0.0.1:8080/api/rawdata | python3 -m json.tool
```

`rawdata` is particularly useful because it gives you a broad view of Traefik's runtime configuration.

---

## 7. One important distinction

A configuration file might say:

```yaml
router:
    rule: Host(`example.com`)
```

But don't assume that means the router is active.

Your troubleshooting process should become:

```text
Configuration says X
        ↓
Runtime API says Y
        ↓
Actual request behaves Z
```

Then investigate the difference.

This is a much stronger SRE approach than simply editing configuration until the request works.
