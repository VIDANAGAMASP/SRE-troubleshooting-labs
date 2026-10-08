# Day 7 — Docker Backend Connectivity + 502 Bad Gateway

Today is a very important SRE troubleshooting pattern:

> The router works, but Traefik cannot reach the backend.

This is where you'll learn to distinguish:

```text
404 → routing problem
502 → backend communication problem
```

That distinction is extremely useful in real incidents.

---

## 1. What you already know

You now understand:

```text
Client
   ↓
Entrypoint
   ↓
Router
   ↓
Service
   ↓
Backend
```

With Docker:

```text
Docker container
      ↓
Docker labels
      ↓
Docker Provider
      ↓
Traefik Router + Service
```

Today we focus on the final connection:

```text
Traefik
   ↓
Docker network
   ↓
Container
   ↓
Application
```

---

## 2. A crucial distinction

Suppose:

```bash
curl http://127.0.0.1
```

returns:

```text
502 Bad Gateway
```

That tells you something useful.

Traefik itself is probably reachable.

The request likely matched a router.

But Traefik couldn't successfully communicate with the backend.

So don't immediately start changing the router.

Think:

```text
Client
  ↓
Traefik :80       ✓
  ↓
Router            ✓
  ↓
Service           ✓
  ↓
Backend           ✗
```

---

## 3. Docker networking

Containers normally communicate through Docker networks.

Check them with:

```bash
docker network ls
```

Inspect a network:

```bash
docker network inspect <network>
```

A container can belong to one or more networks.

Check a container:

```bash
docker inspect <container>
```

Look under:

```text
NetworkSettings
```

and:

```text
Networks
```

---

## 4. Why this matters to Traefik

Imagine:

```text
Traefik
   |
   | Docker network A
   |
   X
   |
Backend
   |
   | Docker network B
```

Traefik may know the backend exists.

The router may be correct.

The service may even contain the correct port.

But if Traefik cannot reach that container over a usable network, the request fails.

This is a classic:

> Discovery succeeded, connectivity failed.

---

## 5. Container port

Suppose the application listens on:

```text
8080
```

Traefik might have:

```text
traefik.http.services.app.loadbalancer.server.port=8080
```

That tells Traefik:

```text
Send traffic to container port 8080
```

But that doesn't magically guarantee that:

```text
Traefik → container:8080
```

will work.

You still need:

```text
correct network
+
correct container
+
correct port
+
application actually listening
```

---

## 6. Your troubleshooting sequence

For a suspected backend connectivity failure:

### Step 1 — Is Traefik reachable?

```bash
curl -v http://127.0.0.1
```

### Step 2 — Did routing work?

Look at the HTTP status and Traefik logs.

### Step 3 — Is the container running?

```bash
docker ps
```

### Step 4 — Is the application actually listening?

```bash
docker exec <container> ...
```

### Step 5 — Which network is the container using?

```bash
docker inspect <container>
```

### Step 6 — Can Traefik reach that network?

Inspect:

```bash
docker network inspect <network>
```

This is the beginning of real infrastructure troubleshooting.
