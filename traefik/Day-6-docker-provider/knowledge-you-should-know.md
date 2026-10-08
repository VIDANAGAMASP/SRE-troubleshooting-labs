# Day 6 — Docker Provider + Labels

Today we make an important jump:

Until now, you configured Traefik using a file provider:

```text
/etc/traefik/dynamic.yml
        ↓
     Traefik
```

Today, Traefik will discover routing configuration from Docker containers themselves.

This is one of the most important concepts you'll need for real-world Traefik troubleshooting.

---

## 1. Knowledge needed

You need only these Docker concepts today:

### Container

A Docker container is a running instance of an image:

```text
Docker image
     ↓
  container
```

Basic commands:

```bash
docker ps
docker ps -a
docker inspect <container>
docker logs <container>
```

---

## 2. Docker labels

Docker containers can have metadata called labels.

For example:

```yaml
labels:
  - "traefik.enable=true"
```

Traefik's Docker provider reads these labels and dynamically creates its configuration.

Conceptually:

```text
Docker container
      |
      | labels
      ↓
Traefik Docker Provider
      |
      ↓
Router + Service
```

So instead of manually writing:

```yaml
http:
  routers:
    myapp:
      rule: "Host(`myapp.local`)"
      service: myapp
```

you can attach the configuration to the container.

---

## 3. The important labels

A basic Traefik Docker configuration looks like:

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.myapp.rule=Host(`myapp.local`)"
  - "traefik.http.services.myapp.loadbalancer.server.port=8080"
```

Think of these as:

```text
traefik.enable
        ↓
Should Traefik consider this container?

router rule
        ↓
When should this request match?

service port
        ↓
Where should Traefik send it?
```

---

## 4. Very important: Docker port vs application port

Suppose the container contains an application listening on:

```text
:8080
```

Traefik needs to know which container port contains the application.

That's what this does:

```text
traefik.http.services.myapp.loadbalancer.server.port=8080
```

This is different from:

```bash
docker ps
```

showing a published host port such as:

```text
0.0.0.0:9000->8080/tcp
```

For today's lab, don't worry about Docker port publishing yet. We will deliberately keep the architecture simple.

---

## 5. Provider mental model

You now have two ways Traefik can obtain dynamic configuration.

### File provider

```text
/etc/traefik/dynamic.yml
             ↓
       File Provider
             ↓
           Traefik
```

### Docker provider

```text
Docker containers
       ↓
Docker Provider
       ↓
     Traefik
```

The important troubleshooting question becomes:

> Where did Traefik get this router/service configuration from?

---

## 6. Today's architecture

```text
                    Docker
                       |
              +--------+--------+
              |                 |
          Traefik            webapp
             :80              :8080
              |
              |
         Docker Provider
              ↑
              |
        Docker labels
```

The client sends:

```text
http://127.0.0.1
```

Traefik should route it to the container.
