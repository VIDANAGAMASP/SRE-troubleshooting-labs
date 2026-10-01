# Day 3 — Permission Denied

## Open a fresh Killercoda Ubuntu playground

[Killercoda Linux Playground](https://killercoda.com/playgrounds/scenario/ubuntu)

## Environment setup

Paste:

```bash
apt-get update -qq && \
apt-get install -y openssh-server openssh-client -qq && \
mkdir -p /run/sshd && \
systemctl enable ssh && \
systemctl start ssh && \
useradd -m -s /bin/bash deploy && \
mkdir -p /home/deploy/.ssh && \
ssh-keygen -t ed25519 -f /tmp/ctf_key -N '' -q && \
cp /tmp/ctf_key.pub /home/deploy/.ssh/authorized_keys && \
chown -R deploy:deploy /home/deploy/.ssh && \
chmod 777 /home/deploy/.ssh && \
chmod 666 /home/deploy/.ssh/authorized_keys && \
chmod 755 /home/deploy && \
echo "====================================" && \
echo " SSH CTF #3 ENVIRONMENT CREATED" && \
echo "====================================" && \
echo "User: deploy" && \
echo "Target: localhost" && \
echo "SSH key: /tmp/ctf_key" && \
echo "Goal: restore public-key authentication" && \
echo "===================================="
```

---

## Your mission

```text
╔══════════════════════════════════════════╗
║       SSH CTF #3 — PUBLIC KEY FAILURE    ║
╚══════════════════════════════════════════╝

User:
    deploy

Target:
    localhost

Private key:
    /tmp/ctf_key

Try:
    ssh -i /tmp/ctf_key deploy@localhost

Reported problem:
    Authentication fails.

Goal:
    Restore SSH public-key authentication.

Restrictions:
    - Do not replace the key.
    - Do not create a password for deploy.
    - Do not enable PasswordAuthentication.
    - Do not modify sshd_config.
    - Diagnose the problem before fixing it.

Success:
    ssh -i /tmp/ctf_key deploy@localhost 'whoami'

Expected:
    deploy
```
