# Day 5 — Path-Based Routing: `Path()` vs `PathPrefix()`

Today we add the second major routing dimension: URL paths.

You already know Host routing:

```text
Host("api.example.com")
```

Now you'll learn why these two are not equivalent:

```text
Path(`/api`)
PathPrefix(`/api`)
```

This is a very common source of reverse-proxy incidents.

---

## 1. Knowledge needed

### HTTP

You already know:

```text
Host
Path
Method
Status code
Headers
```

A request like:

```http
GET /api/users HTTP/1.1
Host: api.example.com
```

contains:

```text
Host = api.example.com
Path = /api/users
```

Traefik can use either of these for routing.

### Traefik

You already know:

```text
Entrypoint
    ↓
Router
    ↓
Service
    ↓
Backend
```

and:

```text
Host(...)
Path(...)
PathPrefix(...)
```

Today we focus entirely on the Path part.

---

## 2. Traefik Note — `Path()`

Consider:

```text
rule: "Path(`/api`)"
```

This means the router matches the path:

```text
/api
```

For example:

```text
/api       ✓
```

but:

```text
/api/users     ?
/api/products  ?
/api/v1        ?
```

These are not the same path as `/api`.

The important concept is:

> `Path()` is an exact path matcher.

---

## 3. Traefik Note — `PathPrefix()`

Now:

```text
rule: "PathPrefix(`/api`)"
```

means:

> Match paths beginning with `/api`.

So:

```text
/api            ✓
/api/users      ✓
/api/products   ✓
/api/v1/users   ✓
```

This is useful when an application has an entire API tree:

```text
/api
/api/users
/api/orders
/api/products
```

---

## 4. Visual comparison

### `Path()`

```text
Path(`/api`)

             /api       ✓
             /api/users ✗
             /api/foo   ✗
```

### `PathPrefix()`

```text
PathPrefix(`/api`)

             /api        ✓
             /api/users  ✓
             /api/foo    ✓
             /api/v1/x   ✓
```

This distinction becomes extremely important when applications expose nested routes.

---

## 5. A subtle troubleshooting point

Suppose:

```text
Traefik :80
     ↓
Router
     ↓
Service
     ↓
Backend :8080
```

The backend might work perfectly:

```bash
curl http://127.0.0.1:8080/api/users
```

but this might fail:

```bash
curl http://127.0.0.1:80/api/users
```

That does not necessarily mean the backend is broken.

You need to determine whether:

```text
/api/users
```

actually matched the Traefik router.

---

## 6. Path matching does not rewrite the path

This is another important concept.

Suppose:

```text
rule: "PathPrefix(`/api`)"
```

and the client sends:

```text
GET /api/users
```

Traefik does not automatically transform it into:

```text
/users
```

The backend normally receives:

```text
/api/users
```

Path rewriting is something we'll learn later with middleware, specifically `StripPrefix`.

So:

```text
PathPrefix
```

means:

> "Should this router receive the request?"

It does not mean:

> "Remove this prefix."

That distinction will save you from many debugging mistakes later.

---

## 7. Commands for today

Test different paths:

```bash
curl -v http://127.0.0.1:80/
```

```bash
curl -v http://127.0.0.1:80/api
```

```bash
curl -v http://127.0.0.1:80/api/users
```

```bash
curl -v http://127.0.0.1:80/api/v1/users
```

Inspect the dynamic configuration:

```bash
cat /etc/traefik/dynamic.yml
```

Test the backend directly:

```bash
curl -v http://127.0.0.1:8080/api/users
```

And check logs when necessary:

```bash
tail -f /var/log/traefik.log
```
