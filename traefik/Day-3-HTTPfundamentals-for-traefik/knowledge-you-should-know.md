# Day 3 — HTTP Fundamentals + Reading Traefik Symptoms

Today we add HTTP, because from this point onward you need to distinguish:

> Can I reach Traefik?

from

> Did Traefik understand my HTTP request?

from

> Did Traefik reach the backend?

This is the foundation for everything that follows.

---

## 1. Knowledge needed

### Linux

You already know:

```text
curl
ss
ps
cat
grep
systemctl
```

### Networking

You already understand TCP connectivity.

Today we add the application protocol running on top of TCP:

```text
TCP connection
      ↓
HTTP request
      ↓
HTTP response
```

### HTTP

You need to understand five things:

1. Request
2. Method
3. URL/path
4. Headers
5. Status code

---

## 2. HTTP fundamentals

Suppose you run:

```bash
curl http://127.0.0.1:80/
```

Conceptually, the client sends something like:

```http
GET / HTTP/1.1
Host: 127.0.0.1
```

Traefik receives that HTTP request.

It can then make routing decisions based on information inside the request.

For example:

```http
GET /api/users
Host: api.example.com
```

Later, Traefik could have rules based on:

- Host
- Path
- Headers

Today we'll focus primarily on path and status codes.

---

## 3. `curl -v` is your friend

Run:

```bash
curl -v http://127.0.0.1:80/
```

The important sections are:

```text
> GET /
> Host: 127.0.0.1
```

Those are request headers.

Then:

```text
< HTTP/1.1 200 OK
```

is the response status.

And:

```text
< Content-Type: text/plain
```

is a response header.

So:

```text
curl
 ↓
TCP connection
 ↓
HTTP request
 ↓
Traefik
 ↓
HTTP response
```

---

## 4. HTTP status codes

You don't need to memorize every HTTP status code.

For this course, these are especially important:

| Code | Meaning              | Typical troubleshooting clue                          |
|------|----------------------|---------------------------------------------------------|
| 200  | Success              | Request reached something that successfully responded |
| 301  | Permanent redirect   | Something is redirecting the request                  |
| 302  | Temporary redirect   | Something is redirecting the request                  |
| 404  | Not found            | Often routing/path/application issue                   |
| 502  | Bad gateway          | Proxy couldn't successfully communicate with backend   |
| 503  | Service unavailable  | Requested service/backend unavailable                  |

### Important

These are clues, not absolute diagnoses.

For example:

```text
404
```

doesn't automatically mean:

> "Traefik router is broken."

The backend application could itself return `404`.

Your job is to gather evidence.

---

## 5. Three very different failures

This distinction is extremely important.

### Case A — Traefik isn't reachable

```text
curl
 ↓
X TCP connection
```

Example:

```text
Connection refused
```

The request never reached HTTP routing.

### Case B — Traefik is reachable but no router matches

```text
curl
 ↓
Traefik :80
 ↓
X Router match
```

Traefik itself may return:

```text
404
```

### Case C — Router matches but backend fails

```text
curl
 ↓
Traefik
 ↓
Router ✓
 ↓
Service ✓
 ↓
Backend X
```

This commonly produces:

```text
502
```

### The mental model

Always think:

```text
             Can I connect?
                    ↓
              Entrypoint
                    ↓
             Does router match?
                    ↓
              Middleware
                    ↓
              Service
                    ↓
             Backend reachable?
                    ↓
             Application response
```

---

## 6. Traefik Note — Router rules

We used this on Day 2:

```text
rule: "PathPrefix(`/`)"
```

Today you need to understand what it means.

`PathPrefix(/)` means:

> Match requests whose path begins with `/`.

Examples:

```text
/              ✓
/api           ✓
/api/users     ✓
/hello         ✓
```

Later you'll learn more specific rules:

```text
Path(`/api`)
PathPrefix(`/api`)
Host(`api.example.com`)
```

But today we'll deliberately create different HTTP symptoms so you can practice identifying where the failure occurs.

---

## 7. Useful commands

**Show only response headers**

```bash
curl -I http://127.0.0.1:80/
```

**Verbose request/response**

```bash
curl -v http://127.0.0.1:80/
```

**Test a specific path**

```bash
curl -v http://127.0.0.1:80/api
```

**Test backend directly**

```bash
curl -v http://127.0.0.1:8080/
```

**Check listeners**

```bash
ss -tlnp
```

**Look at Traefik logs**

```bash
cat /var/log/traefik.log
```

or:

```bash
tail -f /var/log/traefik.log
```
