# SSH Troubleshooting — 20-Day Roadmap

| Day | Knowledge you need                                      | Main troubleshooting skill                  |
|-----|---------------------------------------------------------|---------------------------------------------|
| 1   | SSH client/server, TCP port, `ss`, `systemctl`, `ssh`   | Service down / connection refused           |
| 2   | `sshd_config`, `Port`, `sshd -t`, reload/restart        | Wrong SSH port                              |
| 3   | Users, `$HOME`, `.ssh`, permissions, ownership          | Bad `.ssh` permissions                      |
| 4   | Public/private keys, `authorized_keys`                  | Public-key authentication                   |
| 5   | `ssh -i`, key selection, fingerprints                   | Wrong private key                           |
| 6   | `ssh-agent`, `ssh-add`, `IdentitiesOnly`                | Too many authentication failures            |
| 7   | `known_hosts`, host keys, fingerprints                  | Host key verification                       |
| 8   | `PasswordAuthentication`, `PermitRootLogin`             | Authentication configuration                |
| 9   | `AllowUsers`, `AllowGroups`                             | User access restrictions                    |
| 10  | `journalctl`, `/var/log/auth.log`, SSH logs             | Server-side diagnosis                       |
| 11  | `sshd -t`, safe config changes                          | Broken `sshd_config`                        |
| 12  | TCP states, `ss`, `nc`                                  | Listener/network distinction                |
| 13  | Firewall basics                                         | SSH blocked by firewall                     |
| 14  | IP routing, timeout vs refused                          | Network-path troubleshooting                |
| 15  | DNS, `/etc/hosts`, hostname resolution                  | SSH works by IP but not hostname            |
| 16  | SSH client config                                       | `~/.ssh/config` problems                    |
| 17  | Bastion + `ProxyJump`                                   | Multi-hop SSH                               |
| 18  | `scp`, `sftp`, `rsync`                                  | File-transfer troubleshooting               |
| 19  | `tmux`, `screen`, connection persistence                | Long-running SSH work                       |
| 20  | Everything above                                        | Production-style SSH incident               |
