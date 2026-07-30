# Hack The Box — Write-Ups

A collection of my write-ups for Hack The Box machines. Each one documents the full attack chain — recon through root — with commands, outputs, and screenshots.

> Have completed a few machines before without documenting them — will try to get those in here someday.

---

## Machines

| Machine | OS | Difficulty | Topics |
|---|---|---|---|
| [Silentium](silentium.md) | Linux | Easy | Subdomain fuzzing, Flowise RCE, CVE chaining, password reuse, Gogs, port forwarding, priv esc |
| [Reactor](./Reactor.md) | Linux | Easy | Next.js RCE (CVE-2025-55182), SQLite credential extraction, Node.js debug port hijacking |
| [Kobold](./Kobold.md) | Linux | Easy | mcp server Arcane Docker Management RCE (CVE-2026-23520), Docker socket abuse, chroot privesec  |
| [cctv](./cctv.md) | Linux | Easy | ZoneMinder SQLi (CVE-2024-51482), hash cracking, SSH tunneling, motionEye RCE (CVE-2025-60787) |
| [Nexus](./Nexus.md) | Linux | Medium | Subdomain fuzzing, Gitea commit history credential leak, Krayin CRM RCE (CVE-2026-38526), password reuse, Gitea template-sync path traversal priv esc |
| [Orion](./Orion.md) | Linux | Easy | Craft CMS pre-auth RCE (CVE-2025-32432), leaked DB creds, bcrypt hash cracking, telnetd auth bypass priv esc (CVE-2026-24061) |

---

## About

I'm Shreekara, an aspiring penetration tester working through HTB machines to build practical offensive security skills. These write-ups are my documentation of what I tried, what worked, and what I learned.

- 🔗 [Hack The Box Profile](https://app.hackthebox.com/users/2711239?profile-top-tab=machines&ownership-period=1M&profile-bottom-tab=prolabs)
- 💼 [LinkedIn](https://www.linkedin.com/in/shreekara-k/)
- 🐙 [GitHub](https://github.com/ShreekaraKedlaya)
