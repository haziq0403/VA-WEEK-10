| Details | Information |
|---|---|
| Name | Haziq Danial Bin Nor Azan |
| Student ID | 52215225213 |
| Programme | Bachelor Of Information Technology (Hons) In Computer System Security |
| Course | IKB21403 - Vulnerability Analysis |
| Lecturer | Nor Adani Kamal Mohamad Nasir |

---

# Lab 2 — Core Analyst Skill (CVE / CVSS / CWE Research)

## Objective

Manually research 3 vulnerabilities from Lab 1 using CVE.org and NVD. For each finding, break down the CVSS vector string metric by metric, map it to a CWE, and evaluate whether it is actually exploitable in the lab environment.

> A scanner gives you a number. A professional analyst understands what that number means — and whether those conditions exist in your environment.

---

## Lab Environment

| Component | Description |
|---|---|
| Attacker Machine | Kali Linux |
| Victim Machine | Metasploitable2 |
| Network Type | VirtualBox Host-Only Network |
| Victim IP Address | 192.168.56.106 |

---

## Vulnerabilities Selected

| # | Vulnerability | CVE | Category |
|---|---|---|---|
| 1 | Samba Badlock | CVE-2016-2118 | Protocol / Service |
| 2 | SSL POODLE | CVE-2014-3566 | Cryptographic |
| 3 | VNC Weak Password | No CVE | Authentication / Configuration |

---

## Step 1 — Understand the Objective

![Step 1 - Objective](images/lab2_p03.png)

This lab trains analysts to not blindly trust scanner output. The key question for each finding is:

**"Even though CVSS says X, is this actually exploitable HERE?"**

Factors that affect real exploitability:
- Is the vulnerable service actually running?
- Is the port reachable from the attacker machine?
- Does the attack require special conditions (MITM, active session, credentials)?
- Are those conditions realistic in this specific lab environment?

---

## Step 2 — Research Finding 1: Samba Badlock (CVE-2016-2118)

**Research source:** `https://nvd.nist.gov/vuln/detail/CVE-2016-2118`

![Step 2 - Samba CVE Info and CVSS Breakdown](images/lab2_p04.png)

**CVE Information:**

| Field | Details |
|---|---|
| CVE ID | CVE-2016-2118 |
| Published | April 12, 2016 |
| Affected Software | Samba versions prior to 4.2.11 / 4.3.8 / 4.4.2 |
| Description | MS-SAMR and MS-LSAD protocols in Samba accept inadequate authentication levels. MITM attacker can downgrade to CONNECT level and take over DCE/RPC connection to access SAM database. |

**CVSS v3.0 Score: 7.5 HIGH**

Vector: `CVSS:3.0/AV:N/AC:H/PR:N/UI:R/S:U/C:H/I:H/A:H`

**Vector Breakdown:**

| Metric | Value | Meaning |
|---|---|---|
| Attack Vector (AV) | Network (N) | Exploitable remotely — no physical access needed |
| Attack Complexity (AC) | **High (H)** | Requires MITM position on network — harder to achieve |
| Privileges Required (PR) | None (N) | No credentials needed on target |
| User Interaction (UI) | **Required (R)** | Victim must initiate an SMB connection for attack to work |
| Scope (S) | Unchanged (U) | Cannot affect other components |
| Confidentiality (C) | High (H) | Full read access to SAM database including hashes |
| Integrity (I) | High (H) | Can modify SAM database data |
| Availability (A) | High (H) | Can disable critical services |

**Key insight:** `AC:H` and `UI:R` are the two metrics that make this less immediately dangerous than the CVSS score suggests. Both preconditions must be met simultaneously.

---

## Step 3 — Samba Badlock: CWE Mapping and Lab Exploitability

![Step 3 - Samba CWE and Exploitability](images/lab2_p05.png)

**CWE Mapping:**

| Field | Details |
|---|---|
| CWE ID | CWE-287: Improper Authentication |
| Why | Samba accepts CONNECT level authentication instead of requiring SIGN or SEAL — allows the connection to be hijacked without proper credential verification |

**Lab Exploitability Analysis:**

| Question | Answer |
|---|---|
| Is port 445 open? | YES — confirmed by Nessus scan |
| Reachable from Kali? | YES — same Host-Only network |
| Credentials required? | NO |
| Special conditions? | YES — must be in MITM position between client and server |

**Conclusion:** LIKELY EXPLOITABLE with conditions.

Kali and Metasploitable2 are on the same subnet, so ARP poisoning MITM is possible using tools like `ettercap` or `arpspoof`. However, a victim must also be actively initiating an SMB connection. In this isolated lab with no other users, that connection would need to be simulated manually.

The `AC:H` metric in the CVSS vector correctly reflects this difficulty.

---

## Step 4 — Research Finding 2: SSL POODLE (CVE-2014-3566)

**Research source:** `https://nvd.nist.gov/vuln/detail/CVE-2014-3566`

![Step 4 - SSL POODLE CVE and CVSS Breakdown](images/lab2_p06.png)

**CVE Information:**

| Field | Details |
|---|---|
| CVE ID | CVE-2014-3566 |
| Name | POODLE — Padding Oracle On Downgraded Legacy Encryption |
| Published | October 14, 2014 |
| Affected Software | Any software supporting SSLv3 fallback |
| Description | SSLv3 uses nondeterministic CBC padding — inherently vulnerable to padding oracle attacks. This is a design flaw that cannot be patched. The protocol must be disabled entirely. |

**Important — two different CVSS scores exist:**

| CVSS Version | Score | Rating |
|---|---|---|
| CVSS v2.0 | **4.3** | MEDIUM |
| CVSS v3.0 | **9.8** | CRITICAL |

> This discrepancy shows why you must always check which CVSS version a report is citing. The same vulnerability looks very different depending on the version used.

**CVSS v3.0 Vector Breakdown:**

`CVSS:3.0/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`

| Metric | Value | Meaning |
|---|---|---|
| Attack Vector (AV) | Network (N) | Attacker intercepts SMTP / PostgreSQL traffic |
| Attack Complexity (AC) | **Low (L)** | SSLv3 session can be forced by any client |
| Privileges Required (PR) | None (N) | No credentials needed |
| User Interaction (UI) | **None (N)** | Passive MITM — victim does not need to do anything |
| Confidentiality (C) | High (H) | Encrypted data can be fully decrypted |
| Integrity (I) | High (H) | Can inject data into decrypted session |
| Availability (A) | High (H) | Can disrupt SSL/TLS communications entirely |

---

## Step 5 — SSL POODLE: CWE Mapping and Lab Exploitability

![Step 5 - SSL CWE and Exploitability](images/lab2_p07.png)

**CWE Mapping:**

| Field | Details |
|---|---|
| CWE ID | CWE-327: Use of a Broken or Risky Cryptographic Algorithm |
| Why | SSLv3's CBC padding scheme is fundamentally flawed — nondeterministic padding enables padding oracle attacks. Cannot be patched, only disabled. |
| Also related | CWE-326: Inadequate Encryption Strength — SSLv3 supports weak 40-bit RC4 and DES cipher suites |

**Lab Exploitability Analysis:**

| Question | Answer |
|---|---|
| SSLv3 confirmed on ports 25 & 5432? | YES — confirmed by Nessus |
| Reachable from Kali? | YES — same network |
| Credentials required? | NO |
| Special conditions? | YES — must intercept an active SSLv3 session |

**Conclusion:** LIKELY EXPLOITABLE — but limited practical impact in this isolated lab.

SMTP (port 25) and PostgreSQL (port 5432) are rarely actively used in this lab setup, so there may be no live sessions to intercept at the time of the attack. In a real production environment with active email and database clients, this would be HIGHLY CRITICAL.

---

## Step 6 — Research Finding 3: VNC Weak Password (No CVE)

![Step 6 - VNC No CVE Explanation](images/lab2_p08.png)

**Why is there no CVE?**

This is a **configuration weakness**, not a software flaw. The VNC software itself works exactly as designed — the problem is that the administrator configured a trivially guessable password (`password`). Configuration issues are tracked under CWE, not CVE.

| Field | Details |
|---|---|
| CVE ID | None — Nessus Plugin ID: 61708 |
| Why no CVE | Configuration issue, not a software bug. VNC works as designed. |
| Research source | MITRE CWE (cwe.mitre.org/data/definitions/521.html) |

Since there is no CVE, there is no official CVSS score either. The CVSS must be calculated manually — this is a real analyst skill.

---

## Step 7 — Calculate CVSS Manually for VNC

![Step 7 - VNC Manual CVSS Calculation](images/lab2_p09.png)

**Manually Calculated CVSS v3.1: 10.0 CRITICAL**

Vector: `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`

| Metric | Value | Justification |
|---|---|---|
| Attack Vector (AV) | Network (N) | Port 5900 accessible remotely over any network |
| Attack Complexity (AC) | Low (L) | Just connect to port 5900 and type `password` — no special conditions |
| Privileges Required (PR) | None (N) | Zero prior credentials or account needed |
| User Interaction (UI) | None (N) | Attack is immediate — victim does not need to do anything |
| Scope (S) | Unchanged (U) | Access limited to target system itself |
| Confidentiality (C) | High (H) | Full graphical desktop = all files, data, credentials visible |
| Integrity (I) | High (H) | Can create, modify, or delete any file — can install malware |
| Availability (A) | High (H) | Can shut down, reboot, or fully disable the system |

Every single metric is at maximum — CVSS 10.0 is fully justified.

---

## Step 8 — VNC: CWE Mapping and Lab Exploitability

![Step 8 - VNC CWE and Exploitability](images/lab2_p10.png)

**CWE Mapping:**

| Field | Details |
|---|---|
| CWE ID | CWE-521: Weak Password Requirements |
| Why | VNC allows authentication with a single common dictionary word. No minimum complexity, no lockout policy, no MFA enforced. |
| Also related | CWE-287: Improper Authentication — system does not adequately prevent unauthorised access |
| CAPEC | CAPEC-70: Try Common (Default) Usernames and Passwords |

**Lab Exploitability Analysis:**

| Question | Answer |
|---|---|
| VNC confirmed on port 5900? | YES — confirmed by Nessus |
| Reachable from Kali? | YES |
| Password known? | YES — `password` |
| Special conditions? | NONE |

**Verification command:**

```bash
vncviewer 192.168.56.106
# Enter password: password
# Result: Full root graphical desktop access immediately
```

**Conclusion:** DEFINITELY EXPLOITABLE. Zero conditions required. Most immediately dangerous finding in the lab.

---

## Step 9 — Summary Comparison

![Step 9 - Summary Comparison Table](images/lab2_p11.png)

| Finding | CVE | CVSS | CWE | Attack Vector | Exploitable in Lab? |
|---|---|---|---|---|---|
| Samba Badlock | CVE-2016-2118 | 7.5 HIGH | CWE-287 | Network / High complexity | LIKELY — needs MITM + active SMB session |
| SSL POODLE | CVE-2014-3566 | 9.8 CRITICAL | CWE-327 | Network / Low complexity | LIKELY — needs active SSLv3 session |
| VNC Weak Password | No CVE | 10.0 CRITICAL | CWE-521 | Network / Low complexity | ✅ DEFINITELY — zero conditions |

---

## Key Learning: CVSS Score ≠ Actual Business Risk

From this lab, three important lessons:

**1. High CVSS ≠ immediately exploitable**
SSL POODLE has CVSS 9.8 but requires intercepting an active encrypted session. In this isolated lab, that may never happen.

**2. No CVE ≠ no risk**
VNC has no CVE, but it is the most immediately dangerous finding. Configuration weaknesses are just as dangerous as software bugs.

**3. CVSS version matters**
SSL POODLE has CVSS v2 score of 4.3 (Medium) and CVSS v3 score of 9.8 (Critical). Always check which version is being cited.

---

*IKB21403 Vulnerability Analysis · UniKL MIIT · 2026*

