# 🕵️ grCyb3r — Hacker's Hideout
### TryHackMe | Splunk (SIEM) Investigation Write-Up

SOC Investigation documenting a two-phase attack involving LFI exploitation and SSH brute-force activity through log analysis and event correlation using Splunk SIEM.

---

## 🛠 Skills Demonstrated

• SIEM Log Analysis (Splunk)  
• Threat Detection  
• Incident Investigation  
• Event Correlation  
• IOC Identification  
• Attack Timeline Reconstruction  
• Web Exploitation Analysis (LFI)  
• SSH Brute-Force Detection  

---

| Field | Details |
|---|---|
| **Platform** | TryHackMe - Ironhack Cybersecurity Bootcamp Final Scenario
| **Room** | [Hacker's Hideout](https://tryhackme.com) |
| **Incident Reference** | grCyb3r-THM 001-2026-03 |
| **Date** | 12 March 2026 |
| **Prepared by** | Fabio Martines Laiola — SOC Investigation Portfolio |
| **Severity** | 🔴 Medium–Critical (Data Exposure & Credential Theft) |
| **Systems Affected** | Web Server (`/var/log/apache2/access.log`), SSH Server (`/var/log/auth.log`) |
| **SIEM Tool** | Splunk 9.1.1 |
| **Status** | ✅ Completed — All Tasks Solved |

---

## 📋 Executive Summary

On 12 March 2026, our team conducted a forensic investigation on the **Hacker's Hideout lab environment** using Splunk SIEM.

The investigation revealed a **two-phase attack**:

1. **Web server compromise via Local File Inclusion (LFI)** — the attacker enumerated server files and exposed sensitive information, including system usernames.
2. **SSH brute force attack using credentials obtained from exposed files** — the attacker successfully logged in as `ironhack`, gaining interactive shell access to the server.

This report details the investigation methodology, findings, and recommended mitigation steps.

---

## 🔎 Incident Scope

| Field | Value |
|---|---|
| **Attack Type** | Brute Force / LFI / SSH Compromise |
| **Attacker IP** | `192.168.178.83` |
| **Tool Used** | Fuzz Faster U Fool (ffuf) |
| **Timeline** | November 2023 (simulated lab logs) |
| **Files Accessed** | 97 (successful HTTP 200 responses) |

---

## 🛠️ Technical Findings

### Phase 1 — Web Server Compromise

**Objective:** Identify suspicious activity in Apache logs.

---

#### Step 1 — Discover available log sources

```spl
index=* | stats count by source
```

![Log sources](image1.png)

**Finding:** Two log sources identified:
- `/var/log/apache2/access.log` — Web server logs
- `/var/log/auth.log` — Authentication logs

---

#### Step 2 — Identify the attacker's IP

```spl
index=* source="/var/log/apache2/access.log"
| rex field=_raw "^(?P<clientip>\d+\.\d+\.\d+\.\d+)"
| stats count by clientip
| sort -count
```

![Attacker IP](image2.png)

**Finding:** Bulk activity (878 events) from `192.168.178.83` → attacker identified.

---

#### Step 3 — Analyze attacker requests

```spl
index=* source="/var/log/apache2/access.log" "192.168.178.83" | head 5
```

![Attacker requests](image3.png)

```spl
index=* source="/var/log/apache2/access.log" "192.168.178.83" "200" | stats count
```

![Files exposed count](image4.png)

```spl
index=* source="/var/log/apache2/access.log" "192.168.178.83" "200" "passwd"
```

![passwd access](image5.png)

**Observations:**
- Requests use the parameter `/index.php?page=` → **Local File Inclusion (LFI)**
- User-Agent: `"Fuzz Faster U Fool v2.0.0-dev"` → attack tool identified
- Sensitive file accessed: `/etc/passwd` → usernames exposed

---

#### 📊 Phase 1 Summary

| Field | Value |
|---|---|
| **Technique** | Brute Force |
| **Attack Type** | Local File Inclusion (LFI) |
| **Tool** | Fuzz Faster U Fool |
| **Files Exposed** | 97 |
| **File with Usernames** | `/etc/passwd` |

---

### Phase 2 — SSH Brute Force Attack

**Objective:** Determine if the attacker leveraged the obtained credentials.

---

#### Step 4 — Count failed SSH login attempts

```spl
index=* source="/var/log/auth.log" "Failed password" "192.168.178.83" NOT "repeated" | stats count
```

![Failed attempts](image6.png)

**Finding:** **195 failed login attempts** → brute force confirmed.

---

#### Step 5 — Check for a successful SSH login

```spl
index=* source="/var/log/auth.log" "Accepted password" "192.168.178.83"
```

![Successful login](image7.png)

**Finding:** Login succeeded — attacker gained shell access as user `ironhack` (PID: 51362).

**Log evidence:**
```
Nov 20 14:41:11 ironhack sshd[51362]: Accepted password for ironhack from 192.168.178.83 port 39016 ssh2
```

> 💡 The PID is embedded in the log format as `sshd[PID]` — `sshd[51362]` identifies the exact SSH process ID of that session.

![Session details](image8.png)

---

#### 📊 Phase 2 Summary

| Field | Value |
|---|---|
| **Technique** | Brute Force |
| **Service Targeted** | SSH (port 22) |
| **Username** | `ironhack` |
| **Failed Attempts** | 195 |
| **Login Successful** | ✅ Yes |
| **PID** | `51362` |

---

## 🔗 Step-by-Step Breach Explanation

1. **Attacker enumerates web server files** using the automated tool (`ffuf`).
2. **Sensitive files are exposed**, including usernames (`/etc/passwd`).
3. **Attacker attempts SSH login** repeatedly using the exposed credentials.
4. **Attacker succeeds**, establishing an interactive session on the server.
5. **Each step is documented in Splunk**, showing timestamps, IPs, and tools used.

---

## ✅ Answers Summary

| Question | Answer |
|---|---|
| Technique used for the first attack | Brute Force |
| Name of the attack | Local File Inclusion (LFI) |
| Tool used by the attacker | Fuzz Faster U Fool |
| IP address of the attacker | `192.168.178.83` |
| Files successfully exposed | `97` |
| Technique for the second attack | Brute Force |
| Service targeted (second attack) | SSH |
| Username used by the attacker | `ironhack` |
| File where username was found | `/etc/passwd` |
| Failed login attempts | `195` |
| Attacker successfully logged in | `y` |
| PID of the SSH session | `51362` |
| Mechanism to block the attack | Intrusion Prevention System (IPS) |

---

## 🛡️ Recommendations & Mitigation

1. **Patch LFI vulnerabilities** — sanitize all user inputs; never pass raw user input to file include functions.
2. **Enable an Intrusion Prevention System (IPS)** — detect and block brute force and abnormal file access patterns in real time.
3. **Implement rate limiting and account lockout** — limit SSH authentication attempts per IP.
4. **Monitor logs regularly** — set up Splunk alerts for anomalies.
5. **Use multi-factor authentication (MFA)** — protect all sensitive accounts.

---

## 📎 Appendix — Splunk SPL Queries Reference

```spl
# Discover log sources
index=* | stats count by source

# Identify attacker IP
index=* source="/var/log/apache2/access.log"
| rex field=_raw "^(?P<clientip>\d+\.\d+\.\d+\.\d+)"
| stats count by clientip
| sort -count

# Analyze LFI requests
index=* source="/var/log/apache2/access.log" "192.168.178.83" | head 5

# Count successfully exposed files
index=* source="/var/log/apache2/access.log" "192.168.178.83" "200" | stats count

# Confirm /etc/passwd access
index=* source="/var/log/apache2/access.log" "192.168.178.83" "200" "passwd"

# Count failed SSH attempts
index=* source="/var/log/auth.log" "Failed password" "192.168.178.83" NOT "repeated" | stats count

# Confirm successful SSH login
index=* source="/var/log/auth.log" "Accepted password" "192.168.178.83"
```

---

grCyb3r | Fabio Martines Laiola  
SOC Investigation Portfolio | TryHackMe Write-Up Series
