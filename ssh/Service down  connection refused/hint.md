# Day 1 — SSH Troubleshooting Flow

## Diagnostic questions

```text
Can I connect?
      ↓
Is the SSH service running?
      ↓
Is something listening on port 22?
      ↓
Is sshd actually listening?
```

## Commands and the question each one answers

| Command                  | Question                          |
|--------------------------|-----------------------------------|
| `ssh localhost`          | Can I actually establish SSH?     |
| `systemctl status ssh`   | Is the SSH service running?       |
| `ss -tlnp \| grep :22`   | Is something listening on TCP 22? |
