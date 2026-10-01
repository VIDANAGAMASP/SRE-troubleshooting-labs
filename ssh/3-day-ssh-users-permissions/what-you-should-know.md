# Day 3 — SSH Users, .ssh, Permissions & Ownership

Today we move from "Can I reach sshd?" to "Why does sshd reject this user's key?"

This is one of the most important SSH troubleshooting areas.

---

## 1. Linux users have home directories

Suppose we have:

```bash
useradd -m deploy
```

The user gets:

```text
/home/deploy
```

Their SSH configuration normally lives under:

```text
/home/deploy/.ssh/
```

So the basic structure is:

```text
/home/deploy/
└── .ssh/
    └── authorized_keys
```

---

## 2. authorized_keys

This file contains public keys that are allowed to authenticate as that user.

For example:

```text
/home/deploy/.ssh/authorized_keys
```

Conceptually:

```text
Your laptop
    │
    │ private key
    ▼
SSH server
    │
    ├── deploy user
    │      │
    │      └── authorized_keys
    │
    └── checks whether the public key matches
```

Important:

```text
private key → stays with client
public key  → can be placed in authorized_keys
```

**Never copy your private key into `authorized_keys`.**

---

## 3. Linux permissions

Today you need to understand:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

For example:

```text
drwx------ deploy deploy .ssh
-rw------- deploy deploy authorized_keys
```

The important permissions are generally:

```text
~/.ssh              → 700
authorized_keys     → 600
```

Meaning:

```text
700 = rwx------
600 = rw-------
```

You can fix them with:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

---

## 4. Ownership matters too

Suppose:

```text
/home/deploy/.ssh
```

is owned by:

```text
root root
```

instead of:

```text
deploy deploy
```

That can cause SSH authentication problems.

Check:

```bash
ls -ld /home/deploy
ls -ld /home/deploy/.ssh
ls -l /home/deploy/.ssh/authorized_keys
```

Fix ownership with:

```bash
chown -R deploy:deploy /home/deploy/.ssh
```

So your mental model is:

```text
SSH key authentication
        │
        ├── correct user?
        │
        ├── authorized_keys exists?
        │
        ├── correct ownership?
        │
        └── safe permissions?
```

---

## 5. Why SSH cares about permissions

SSH doesn't want another user to be able to manipulate your authentication files.

Imagine:

```text
deploy/.ssh/authorized_keys
```

was writable by everyone.

Another user could potentially add their own public key and gain access as `deploy`.

Therefore SSH is deliberately strict about ownership and permissions.
