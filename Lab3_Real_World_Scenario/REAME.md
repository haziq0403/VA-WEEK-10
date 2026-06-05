| Details | Information |
|---|---|
| Name | Haziq Danial Bin Nor Azan |
| Student ID | 52215225213 |
| Programme | Bachelor Of Information Technology (Hons) In Computer System Security |
| Course | IKB21403 - Vulnerability Analysis |
| Lecturer | Nor Adani Kamal Mohamad Nasir |

---

# Lab 3 — Real-World Scenario (Manual Validation)

## Objective

Manually validate scanner findings using command-line tools. For each finding, make a professional verdict:

- ✅ **True Positive** — finding is real and confirmed by manual check
- ❌ **False Positive** — scanner reported something that does not actually exist
- ⚠️ **Accepted Risk** — finding is real but low enough severity to monitor without immediate action

> Blind reporting = bad analyst. A professional never pastes scanner output into a report without verifying it first.

---

## Lab Environment

| Component | Description |
|---|---|
| Attacker Machine | Kali Linux |
| Victim Machine | Metasploitable2 |
| Network Type | VirtualBox Host-Only Network |
| Victim IP Address | 192.168.56.106 |

---

## Manual Validation Tools

| Tool | Command | Purpose |
|---|---|---|
| Nmap ssl-enum-ciphers | `nmap --script ssl-enum-ciphers -p PORT IP` | Check what SSL/TLS ciphers a port actually offers |
| Nmap service version | `nmap -sV -p PORT IP` | Detect exact service and version on a port |
| Netcat banner grab | `nc -nv IP PORT` | Connect and read the raw service banner |
| curl header grab | `curl -I http://IP` | Read HTTP response headers including server version |

---

## Findings Selected for Validation

| Finding | Service | Scanner Claim |
|---|---|---|
| A | SSL Weak Cipher (Port 443) | SSLv2/v3 on HTTPS — CRITICAL |
| B | Open Port No Auth (Port 1524) | Bind Shell Backdoor — CRITICAL |
| C | Outdated Services (Ports 21, 22, 80) | Multiple outdated versions — HIGH |

---

## Step 1 — Understand the Objective

![Step 1 - Objective and Tools](images/lab3_p03.png)

Three validation scenarios — each teaches a different lesson:

| Finding | Lesson |
|---|---|
| A — SSL Cipher | Scanner can get the **port** wrong even when the vulnerability is real |
| B — Bind Shell | Two tools confirming the same finding = very high confidence true positive |
| C — Outdated Services | Banner grabbing confirms exact version to match against CVE databases |

---

## Step 2 — Finding A: What the Scanner Reported

![Step 2 - Finding A Scanner Report](images/lab3_p04.png)

**Scanner claim:**

| Field | Details |
|---|---|
| Finding | SSL Version 2 and 3 Protocol Detection |
| Severity | CRITICAL (CVSS 9.8) |
| Scanner claim | Target accepts SSLv2 and SSLv3 connections with weak cipher suites |
| Affected port | Port 443/tcp (HTTPS) |

**Manual validation command:**

```bash
nmap --script ssl-enum-ciphers -p 443 192.168.56.106
```

**What this command does:**
- Uses the Nmap ssl-enum-ciphers NSE script
- Connects to port 443 and attempts to negotiate SSL/TLS
- Lists all cipher suites the server is willing to accept

**Actual result:**

```
PORT      STATE    SERVICE
443/tcp   filtered https
```

Port 443 is **FILTERED** — no HTTPS web server is running on Metasploitable2.

---

## Step 3 — Finding A: Analysis and Verdict

![Step 3 - Finding A Technical Justification](images/lab3_p05.png)

**Analysis:**

The scanner's SSL findings are VALID — but they apply to port 25 (SMTP) and port 5432 (PostgreSQL), NOT port 443. The ssl-enum-ciphers script returned no SSL data from port 443 because no service is listening there.

If we only checked port 443:
- Port 443 → ❌ **FALSE POSITIVE** — no HTTPS service exists
- Ports 25 & 5432 → ✅ **TRUE POSITIVE** — SSL weakness is real on these ports

**Verdict: FALSE POSITIVE for port 443 / TRUE POSITIVE for ports 25 & 5432**

> **Real-world lesson:** A blind report saying "fix SSL on port 443" would cause a client to waste time and money remediating something that does not exist. Manual verification saves expensive false remediation.

---

## Step 4 — Finding B: What the Scanner Reported

![Step 4 - Finding B Scanner Report](images/lab3_p06.png)

**Scanner claim:**

| Field | Details |
|---|---|
| Finding | Bind Shell Backdoor Detection |
| Severity | CRITICAL (CVSS 9.8) |
| Scanner claim | Port 1524/tcp is open with a shell that requires no authentication |
| Nessus evidence | Executed `id` command — received `uid=0(root) gid=0(root) groups=0(root)` |

**Manual validation command:**

```bash
nmap -sV -p 1524,5900,23 192.168.56.106
```

**What this command does:**
- `-sV` = detect service name and version on each port
- `-p 1524,5900,23` = check these three specific ports
- Returns what service is actually running, not just whether the port is open

**Actual result:**

```
PORT      STATE  SERVICE   VERSION
23/tcp    open   telnet    Linux telnetd
1524/tcp  open   bindshell Metasploitable root shell
5900/tcp  open   vnc       VNC (protocol 3.3)
```

Nmap identifies port 1524 as `bindshell` and labels it **"Metasploitable root shell"** — directly confirming it is a backdoor.

**Additional verification:**

```bash
nc -nv 192.168.56.106 1524
```

Expected result: Immediate `#` root shell prompt — no login required.

---

## Step 5 — Finding B: Analysis and Verdict

![Step 5 - Finding B Verdict](images/lab3_p07.png)

**Analysis:**

Two independent tools — Nessus and Nmap — both confirm the exact same finding. This gives maximum analyst confidence.

| Tool | What it confirmed |
|---|---|
| Nessus | Executed `id` → received `uid=0(root)` |
| Nmap `-sV` | Identified port 1524 as `bindshell` / "Metasploitable root shell" |

When two independent tools agree on a critical finding, the probability of false positive is extremely low.

**Verdict: ✅ TRUE POSITIVE — CRITICAL SEVERITY**

In a real VA engagement, dual-tool confirmation of an unauthenticated root shell triggers **immediate client escalation — same day remediation required**.

---

## Step 6 — Finding C: Validate FTP Service (Port 21)

![Step 6 - FTP Banner Grab](images/lab3_p08.png)

**Scanner claim:** Multiple services running outdated versions with known CVEs.
**Services to validate:** FTP (vsftpd), SSH (OpenSSH), HTTP (Apache)

**Manual validation command for FTP:**

```bash
nc -nv 192.168.56.106 21
```

**What this command does:**
- `nc` = Netcat — a simple TCP connection tool
- `-n` = skip DNS resolution
- `-v` = verbose — show connection status and banner
- Connects to port 21 and reads whatever the FTP server sends back (the banner)

**Actual banner received:**

```
(UNKNOWN) [192.168.56.106] 21 (ftp) open
220 (vsFTPd 2.3.4)
```

**Analysis of vsftpd 2.3.4:**

| Field | Details |
|---|---|
| Version Detected | vsftpd 2.3.4 |
| CVE Reference | CVE-2011-2523 |
| Description | vsftpd 2.3.4 contains a **deliberately introduced backdoor**. Appending `:)` to the username triggers a root shell on port 6200. |
| Is it vulnerable? | YES — this is THE backdoored version |
| Current stable | vsftpd 3.0.5 |

**Verdict: ✅ TRUE POSITIVE — CRITICAL (CVE-2011-2523)**

---

## Step 7 — Finding C: Validate SSH Service (Port 22)

![Step 7 - SSH Banner Grab](images/lab3_p09.png)

**Manual validation command:**

```bash
nc -nv 192.168.56.106 22
```

**Actual banner received:**

```
(UNKNOWN) [192.168.56.106] 22 (ssh) open
SSH-2.0-OpenSSH_4.7p1 Debian-8ubuntu1
```

**Analysis of OpenSSH 4.7p1:**

| Field | Details |
|---|---|
| Version Detected | OpenSSH 4.7p1 Debian-8ubuntu1 |
| Released | 2007 — approximately 19 years old |
| CVE Reference | CVE-2008-5161 |
| CVE Description | CBC mode allows remote attacker to recover partial plaintext. CVSS 2.6 LOW. |
| RegreSSHion affected? | NO — RegreSSHion (CVE-2024-6387) affects 8.5p1–9.7p1 only |

**Analysis:**
Very old version but the only confirmed CVE (CVE-2008-5161) is low severity with no direct RCE path. The main risk is the age of the software and missing modern security features.

**Verdict: ✅ TRUE POSITIVE — LOW / ACCEPTED RISK (monitor, upgrade in planned maintenance)**

---

## Step 8 — Finding C: Validate HTTP Service (Port 80)

![Step 8 - Apache curl Validation](images/lab3_p10.png)

**Manual validation command:**

```bash
curl -I http://192.168.56.106
```

**What this command does:**
- `curl -I` = send an HTTP HEAD request — returns only the response headers, not the page body
- The `Server:` header reveals the web server software and version
- The `X-Powered-By:` header reveals the backend language version

**Actual response headers:**

```
HTTP/1.1 200 OK
Date: Sat, 23 May 2026 14:16:13 GMT
Server: Apache/2.2.8 (Ubuntu) DAV/2
X-Powered-By: PHP/5.2.4-2ubuntu5.10
Content-Type: text/html
```

**Analysis:**

| Software | Version | Status | Key CVEs |
|---|---|---|---|
| Apache | 2.2.8 | EOL — released Feb 2008 | CVE-2017-7679 (mod_mime buffer overflow), CVE-2017-9788 (DoS) |
| PHP | 5.2.4 | EOL since December 2018 | 100+ CVEs including remote code execution |

Both components are End-of-Life with no further patches. The PHP RCE CVEs alone make this a critical finding.

**Verdict: ✅ TRUE POSITIVE — HIGH/CRITICAL**

---

## Step 9 — Final Summary of All Verdicts

![Step 9 - Final Verdict Table](images/lab3_p11.png)

| Finding | Service / Port | Scanner Said | Verdict | Technical Justification |
|---|---|---|---|---|
| A | SSL (Port 443) | CRITICAL | ❌ FALSE POSITIVE (443) / ✅ TRUE POSITIVE (25, 5432) | Port 443 filtered — no HTTPS. SSL real but on SMTP and PostgreSQL. |
| B | Bind Shell (Port 1524) | CRITICAL | ✅ TRUE POSITIVE CRITICAL | Nmap confirms `bindshell` / "Metasploitable root shell". Two tools agree. |
| C1 | vsftpd 2.3.4 (Port 21) | HIGH | ✅ TRUE POSITIVE CRITICAL | Banner confirms backdoored version. CVE-2011-2523 actively exploitable. |
| C2 | OpenSSH 4.7p1 (Port 22) | HIGH | ✅ TRUE POSITIVE ACCEPTED RISK | Banner confirms version. CVE-2008-5161 low severity. No RCE path. |
| C3 | Apache 2.2.8 / PHP 5.2.4 (Port 80) | HIGH | ✅ TRUE POSITIVE CRITICAL | curl confirms both EOL. 100+ CVEs including RCE. Upgrade immediately. |

---

## Step 10 — Key Learning

![Step 10 - Key Learning](images/lab3_p12.png)

| | Bad Analyst | Professional Analyst |
|---|---|---|
| Approach | Copy-paste Nessus output into Word | Verify each finding with Nmap / nc / curl first |
| Finding A | Reports SSL on port 443 | Reports SSL is on ports 25 & 5432 — NOT port 443 |
| Finding B | Reports Critical — done | Confirms with `nmap -sV` AND `nc -nv` for dual verification |
| Finding C | Lists version from scanner | Does banner grab to get exact version, cross-references CVE |
| Result | Client wastes budget on non-existent issues | Client gets precise, actionable intelligence |

**The skill is not running a scanner — it is knowing how to verify, contextualise, and defend every finding you write in a report.**

---

*IKB21403 Vulnerability Analysis · UniKL MIIT · 2026*

