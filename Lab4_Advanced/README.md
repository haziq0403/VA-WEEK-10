| Details | Information |
|---|---|
| Name | Haziq Danial Bin Nor Azan |
| Student ID | 52215225213 |
| Programme | Bachelor Of Information Technology (Hons) In Computer System Security |
| Course | IKB21403 - Vulnerability Analysis |
| Lecturer | Nor Adani Kamal Mohamad Nasir |

---

# Lab 4 — Advanced (Risk-Based Vulnerability Prioritisation)

## Objective

Re-prioritise all 5 vulnerabilities from Lab 1 using a contextual **risk scoring framework** instead of CVSS score alone. Build a remediation priority list ordered by real-world risk — not by CVSS number.

> Blindly fixing vulnerabilities in CVSS order wastes budget and leaves high-exposure targets unprotected.

---

## Lab Environment

| Component | Description |
|---|---|
| Attacker Machine | Kali Linux |
| Victim Machine | Metasploitable2 |
| Network Type | VirtualBox Host-Only Network |
| Victim IP Address | 192.168.56.106 |

---

## Risk Scoring Formula

```
Risk Score = Exploitability (E) × Impact (I) × Exposure (Ex)

Maximum score = 5 × 5 × 5 = 125
```

| Dimension | What it measures |
|---|---|
| **Exploitability (E)** | Is there a public exploit? Easy to use? Require special conditions? |
| **Impact (I)** | Worst-case outcome — RCE? Data loss? Privilege escalation? DoS? |
| **Exposure (Ex)** | Internet-facing? Reachable from Kali? Internal only? Segmented? |

---

## Step 1 — Understand Why CVSS Alone is Not Enough

![Step 1 - Risk Framework Explanation](images/lab4_p03.png)

**The core problem with CVSS-only prioritisation:**

- A CVSS 10.0 vulnerability on an internal database unreachable from outside may be **less urgent** than a CVSS 5.0 vulnerability on an internet-facing login page with no authentication
- CVSS measures severity in an ideal attack scenario — not in **your** environment
- Fixing vulnerabilities by CVSS order alone wastes time and budget

**Lab context:**
Metasploitable2 (`192.168.56.106`) is directly reachable from Kali on the same Host-Only network. This means Exposure is elevated for ALL findings compared to a segmented real-world network.

---

## Step 2 — Learn the Scoring Scale

![Step 2 - Scoring Legend](images/lab4_p04.png)

| Score | Exploitability | Impact | Exposure |
|---|---|---|---|
| **5/5** | Public exploit, one-click, no skill | Full system compromise / root | Internet-facing, no auth |
| **4/5** | Known exploit, easy to use | RCE or full data access | Reachable from LAN / lab |
| **3/5** | PoC exists, moderate skill needed | Privilege escalation / data leak | Restricted internal access |
| **2/5** | No public exploit, hard to exploit | Partial data access / DoS | Requires MITM or credentials |
| **1/5** | Theoretical only, very complex | Minimal / informational | Fully segmented / isolated |

---

## Step 3 — Apply Risk Scores to All 5 Vulnerabilities

![Step 3 - Risk Scoring Table](images/lab4_p05.png)

| Vulnerability | CVSS | E | I | Ex | Risk Score | CVSS Rank | Risk Rank |
|---|---|---|---|---|---|---|---|
| VNC Weak Password (Port 5900) | 10.0 | 5/5 | 5/5 | 5/5 | **125** | #1 | #1 |
| Bind Shell Backdoor (Port 1524) | 9.8 | 5/5 | 5/5 | 5/5 | **125** | #2 | #1 |
| Ubuntu Linux EOL (OS-level) | 10.0 | 3/5 | 5/5 | 4/5 | **60** | #1 | #3 |
| SSL v2/v3 Detection (Ports 25, 5432) | 9.8 | 2/5 | 3/5 | 3/5 | **18** | #2 | #4 |
| Samba Badlock CVE-2016-2118 (Port 445) | 7.5 | 2/5 | 3/5 | 3/5 | **18** | #5 | #4 |

**Key observation:**
- Ubuntu EOL and VNC both have CVSS 10.0 — same score
- But Risk Score is 60 vs 125 — very different real risk
- Ubuntu EOL drops to #3 because actually exploiting it requires finding a specific unpatched CVE with a working exploit
- VNC stays at #1 because exploiting it requires typing one word

---

## Step 4 — Analyse VNC Weak Password (Risk Score: 125/125)

![Step 4 - VNC Risk Analysis](images/lab4_p06.png)

**Priority 1 — IMMEDIATE ACTION REQUIRED**

| Metric | Score | Justification |
|---|---|---|
| Exploitability (E) | **5/5** | Public exploit is trivial — just type `password`. Nessus logged in automatically in 10 seconds. Zero skill required. |
| Impact (I) | **5/5** | Full graphical desktop control as root. Can read/write/delete all files, install malware, pivot to other systems. |
| Exposure (Ex) | **5/5** | Port 5900 open and directly reachable from Kali. No firewall. No authentication barrier except trivial password. |

**Risk Score: 5 × 5 × 5 = 125 / 125**

This is a perfect storm — maximum on every dimension simultaneously. In a real engagement, this triggers an emergency notification to the client within the hour.

---

## Step 5 — Analyse Bind Shell Backdoor (Risk Score: 125/125)

![Step 5 - Bind Shell Risk Analysis](images/lab4_p07.png)

**Priority 1 — IMMEDIATE ACTION REQUIRED (TIE WITH VNC)**

| Metric | Score | Justification |
|---|---|---|
| Exploitability (E) | **5/5** | No exploit code needed. One command: `nc 192.168.56.106 1524` gives root shell. Simpler than VNC. |
| Impact (I) | **5/5** | Root command-line shell — full system control. Can read `/etc/shadow`, install rootkits, pivot to other hosts. |
| Exposure (Ex) | **5/5** | Port 1524 directly reachable. Confirmed by Nessus and Nmap. Zero authentication whatsoever. |

**Risk Score: 5 × 5 × 5 = 125 / 125**

**Command to exploit from Kali:**

```bash
nc 192.168.56.106 1524
# Result: # (root shell — no password, no exploit code needed)
```

The presence of a backdoor also suggests the system **may already be compromised**. A full forensic investigation is mandatory before trusting any data on this host.

---

## Step 6 — Analyse Ubuntu EOL, SSL v2/v3, and Samba Badlock

![Step 6 - Ubuntu EOL and Lower Risk Findings](images/lab4_p08.png)

**Ubuntu EOL — Risk Score: 60/125 (Priority 3)**

| Metric | Score | Justification |
|---|---|---|
| Exploitability (E) | 3/5 | Many CVEs exist but each requires identifying a specific unpatched component and having a working exploit for it. Not trivially automated. |
| Impact (I) | 5/5 | OS-level compromise = complete system takeover. Kernel exploits bypass all other security controls. |
| Exposure (Ex) | 4/5 | Fully reachable on local network. Multiple services exposed. Not internet-facing in lab (hence 4, not 5). |

Despite having the same CVSS as VNC (10.0), Ubuntu EOL ranks third in risk because exploiting it is harder — unlike VNC which just needs someone to type `password`.

---

![Step 6b - SSL and Samba Joint Analysis](images/lab4_p09.png)

**SSL v2/v3 and Samba Badlock — Risk Score: 18/125 (Priority 4)**

Both score only 18/125 despite SSL having the same CVSS (9.8) as the Bind Shell Backdoor.

| Vulnerability | CVSS | E | I | Ex | Score | Key reason for lower rank |
|---|---|---|---|---|---|---|
| SSL v2/v3 | 9.8 CRITICAL | 2/5 | 3/5 | 3/5 | 18 | Requires active MITM position + active session to intercept |
| Samba Badlock | 7.5 HIGH | 2/5 | 3/5 | 3/5 | 18 | Requires MITM + victim must initiate SMB connection |

Both require the attacker to:
1. Achieve man-in-the-middle network position (ARP poisoning, etc.)
2. Wait for a legitimate user to initiate a connection

In this isolated lab with no other active users, both conditions are hard to meet.

---

## Step 7 — Build the Remediation Priority List

![Step 7 - Remediation Priority List](images/lab4_p10.png)

Ordered by **Risk Score** — NOT CVSS:

| Priority | Vulnerability | CVSS | Risk Score | Timeline | Remediation Action |
|---|---|---|---|---|---|
| **P1** | VNC Weak Password (Port 5900) | 10.0 | 125 | **Same Day** | Change VNC password to 16+ char complex. Block port 5900 via firewall. Restrict to trusted IPs only. |
| **P1** | Bind Shell Backdoor (Port 1524) | 9.8 | 125 | **Same Day** | Block port 1524 immediately. Kill backdoor process. Full forensic investigation. Consider system rebuild. |
| **P3** | Ubuntu EOL 8.04 (OS-level) | 10.0 | 60 | **1–2 Weeks** | Plan OS upgrade to Ubuntu 22.04 LTS. Isolate from production network until upgrade complete. |
| **P4** | SSL v2/v3 Detection (Ports 25, 5432) | 9.8 | 18 | **1 Month** | Disable SSLv2/v3 on SMTP and PostgreSQL. Enforce TLS 1.2 minimum. Schedule during maintenance window. |
| **P4** | Samba Badlock CVE-2016-2118 (Port 445) | 7.5 | 18 | **1 Month** | Upgrade Samba to v4.2.11+. Implement network segmentation to limit MITM attack surface. |

---

## Step 8 — Key Lesson: Why Medium CVSS Can Outrank High CVSS

![Step 8 - CVSS vs Risk Comparison](images/lab4_p11.png)

This is the most important concept in Lab 4.

| | Scenario A | Scenario B |
|---|---|---|
| **Vulnerability** | DVWA Default Admin Login (`admin` / `password`) | Samba Badlock CVE-2016-2118 |
| **CVSS Score** | **5.3 MEDIUM** | **7.5 HIGH** |
| **Location** | Port 80 — internet-facing web app | Port 445 — internal SMB service |
| **Authentication Required** | None — just browse to the page | Must have MITM position on network |
| **User Action Needed** | None — attacker initiates directly | Victim must initiate SMB connection |
| **Public Exploit** | No exploit needed — type `admin`/`password` in browser | Requires specialised MITM tools (e.g. Ettercap) |
| **Exploitability Score** | 5/5 — trivially exploitable | 2/5 — requires complex preconditions |
| **Impact Score** | 4/5 — full web app access, data exfil | 3/5 — SAM database access if successful |
| **Exposure Score** | 5/5 — directly reachable, no barriers | 3/5 — internal, requires MITM |
| **Risk Score** | **100/125 — HIGH REAL RISK** | **18/125 — LOW REAL RISK** |
| **Priority** | **P1 — Fix IMMEDIATELY** | P4 — Planned maintenance |

**The MEDIUM CVSS vulnerability is far more dangerous than the HIGH CVSS vulnerability.**

---

## Step 9 — Real-World Analogy

![Step 9 - Real-World Analogy](images/lab4_p13.png)

Think of it this way:

| | Example | Risk |
|---|---|---|
| ☢️ High impact, low access | Nuclear warhead in a locked underground bunker | **LOWER** immediate risk |
| 🚪 Medium impact, easy access | Unlocked front door to your house | **HIGHER** immediate risk |

**CVSS measures the size of the warhead.**
**Risk assessment measures whether someone can actually get to it and set it off.**

This is exactly what separates a professional security analyst from someone who just runs a scanner and pastes the output into a Word document.

---

## Summary: CVSS Rank vs Risk Rank

| Vulnerability | CVSS | CVSS Rank | Risk Score | Risk Rank | Change |
|---|---|---|---|---|---|
| VNC Weak Password | 10.0 | #1 (tie) | 125 | **#1** | = same |
| Bind Shell Backdoor | 9.8 | #2 (tie) | 125 | **#1** | ▲ moved up |
| Ubuntu Linux EOL | 10.0 | #1 (tie) | 60 | **#3** | ▼ moved down |
| SSL v2/v3 Detection | 9.8 | #2 (tie) | 18 | **#4** | ▼ moved down |
| Samba Badlock | 7.5 | #5 | 18 | **#4** | ▲ moved up |

Ubuntu EOL dropped from CVSS rank #1 to risk rank #3.
SSL v2/v3 dropped from CVSS rank #2 to risk rank #4 — despite having CVSS 9.8 Critical.

---

*IKB21403 Vulnerability Analysis · UniKL MIIT · 2026*

