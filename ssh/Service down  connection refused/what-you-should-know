# Day 1 — SSH Fundamentals + Service Troubleshooting

Today's CTF only requires a small portion of your notes.

## Five things to understand

1. SSH client
2. SSH server
3. TCP port
4. systemd service
5. Listening socket

---

## 1. SSH client vs SSH server

When you type:

```bash
ssh user@server
```

your machine is the **SSH client**.

The remote machine runs an SSH server daemon called `sshd`.

On Ubuntu/Debian, the systemd service is normally called `ssh`.

So:

```bash
systemctl status ssh
```

means: *"What is the status of the SSH server service?"*

Don't confuse:

| Name   | What it is                        |
|--------|-----------------------------------|
| `ssh`  | client command                    |
| `sshd` | server daemon                     |
| `ssh`  | Ubuntu systemd service name       |

---

## 2. SSH needs a TCP connection

Normally:

```text
client ───── TCP ─────> server
                         port 22
                         ↓
                        sshd
```

SSH normally listens on **TCP port 22**.

You can check listening TCP sockets with:

```bash
ss -tlnp
```

Understand the options:

| Option | Meaning              |
|--------|----------------------|
| `-t`   | TCP                  |
| `-l`   | listening            |
| `-n`   | don't resolve names  |
| `-p`   | show process         |

So `ss -tlnp` basically means:

> Show me TCP ports that are listening and which process owns them.

You may see:

```text
LISTEN 0 128 0.0.0.0:22 ...
```

That means something is listening on TCP port 22.

---

## 3. systemctl

Linux services are commonly managed by **systemd**.

Commands you need today:

```bash
systemctl status ssh    # Check the service
systemctl start ssh     # Start it
systemctl stop ssh      # Stop it
systemctl restart ssh   # Restart it
```

For today's CTF, the most important one is:

```bash
systemctl status ssh
```

Don't immediately restart things when troubleshooting. **First inspect.**

That's an important SRE habit:

```text
Observe → understand → change → verify
```

not:

```text
Something is broken → restart everything
```

---

## 4. Understanding `Connection refused`

Suppose you run:

```bash
ssh localhost
```

and receive something like:

```text
ssh: connect to host localhost port 22: Connection refused
```

Think about the TCP layer first.

Your client reached the machine, but there is no service accepting the connection on that port.

Conceptually:

```text
SSH client
    |
    | TCP connection
    ↓
localhost:22
    |
    X
    |
   sshd
```

Possible causes include:

- `sshd` isn't running
- SSH is listening on another port
- nothing is listening on port 22

For Day 1, we are primarily interested in the **first one**.
