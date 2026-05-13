# Kali Linux Setup & Attack Simulation — KALI-01

> Part of the [ARGUS Home SOC Lab](../README.md) project.  
> **Phase 1 — Step 5:** Attack simulation platform deployed and verified on network.

---

## Objective

Deploy a Kali Linux VM on the Proxmox host to serve as the dedicated attack simulation machine for the lab. KALI-01 will be used to generate real attack traffic against lab endpoints, validate Wazuh detections, and test the full SOC pipeline end to end.

---

## VM Configuration

KALI-01 was provisioned in Proxmox using the Kali Linux 2026.1 x86_64 installer ISO.

| Setting | Value |
|---|---|
| VM ID | 105 |
| Name | `KALI-01` |
| CPU | 2 cores (x86-64-v2-AES) |
| Memory | 3040 MB (~3 GB) |
| Disk | local storage, qcow2, iothread enabled |
| Network | VirtIO, `vmbr0` bridge, firewall enabled |
| OS Type | Linux (l26) |
| Node | pve |

> **Note:** KALI-01 is currently on `vmbr0` alongside the rest of the lab. Once pfSense is implemented, it will be moved to `vmbr3` (isolated attack network segment) to better reflect a real adversary network position.

![KALI-01 Proxmox VM configuration]

<img width="727" height="535" alt="image" src="https://github.com/user-attachments/assets/598b5e16-29e3-4c79-b8b2-a40a3574aa09" />
(screenshots/KALI01-vm-configuration.png)

---

## Installation Steps

1. Downloaded **Kali Linux 2026.1 x86_64 Installer ISO** from kali.org and uploaded to Proxmox ISO storage
2. Created VM in Proxmox with VirtIO disk and network adapter
3. Booted into **Graphical Install**
4. Configured hostname as `kali01`, domain left blank, user `kali`
5. Selected guided partitioning — entire disk, single partition
6. Kept default software selection — Kali desktop, top 10 tools, standard utilities
7. Installed GRUB bootloader to `/dev/sda`
8. Booted into Kali desktop

---

## Network Configuration

### Static IP Assignment

To ensure consistent identification within the lab, KALI-01 was configured with a static IP address using `nmcli`:

```bash
sudo nmcli con mod "Wired connection 1" \
  ipv4.addresses 192.168.0.80/24 \
  ipv4.gateway 192.168.0.1 \
  ipv4.dns 192.168.0.10 \
  ipv4.method manual

sudo nmcli con up "Wired connection 1"
```

Output confirmed:

```
Connection successfully activated
```

Interface `eth0` was confirmed with the assigned static address:

```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 192.168.0.80/24 brd 192.168.0.255 scope global noprefixroute eth0
```

![KALI-01 static IP configured]

<img width="772" height="507" alt="image" src="https://github.com/user-attachments/assets/cdef6a67-2718-45be-91a7-3bd1e264b78c" />

---
## Verification

After static IP assignment, connectivity to the gateway and domain controller was confirmed:

```bash
ping 192.168.0.1    # gateway
ping 192.168.0.10   # DC01
```

**Results:**

- Gateway (`192.168.0.1`): 4 packets transmitted, 4 received, **0% packet loss**
- DC01 (`192.168.0.10`): 4 packets transmitted, 4 received, **0% packet loss**

![KALI-01 connectivity to gateway and DC01 verified]

<img width="778" height="513" alt="image" src="https://github.com/user-attachments/assets/fba59c1f-3f72-4014-84fd-5bf1289c7957" />


---

## Current Lab Environment

| Hostname | Role | OS | IP Address |
|---|---|---|---|
| `DC01` | Domain Controller | Windows Server 2022 | `192.168.0.10` |
| `WS01` | Workstation | Windows 10 Pro | `192.168.0.70` |
| `INFRA01` | Infrastructure Server | Ubuntu 24.04.4 LTS | `192.168.0.50` |
| `WAZUH01` | SIEM | Ubuntu 22.04 LTS | `192.168.0.60` |
| `KALI-01` | Attack Simulation | Kali Linux 2026.1 | `192.168.0.80` |

---

## Attack Simulation

### Pre-Attack: False Positive Investigation & Alert Tuning

Before running any attacks, the Wazuh Threat Hunting dashboard was reviewed to establish a baseline. The Events tab was set to **Last 24 hours** and filtered for rule level 15 and above.

Unexpectedly, **29 critical alerts** were already present — all originating from `DC-01` before any attack simulation had begun.

![Wazuh Threat Hunting dashboard — 29 pre-existing critical alerts]

<img width="1893" height="881" alt="image" src="https://github.com/user-attachments/assets/56aa20c8-200e-4649-b1b3-6151c3c0a9bc" />


**Alert details:**

| Field | Value |
|---|---|
| `rule.id` | `92213` |
| `rule.level` | `15` |
| `rule.description` | Executable file dropped in folder commonly used by malware |
| `rule.mitre.technique` | Ingress Tool Transfer (`T1105`) |
| `rule.mitre.tactic` | Command and Control |
| `data.win.eventdata.image` | `C:\Windows\system32\cleanmgr.exe` |
| `data.win.eventdata.targetFilename` | `WimProvider.dll` (dropped in Temp folder) |
| `data.win.eventdata.user` | `LAB\Administrator` |
| `data.win.system.eventID` | `11` (File Created) |
| `data.win.system.channel` | `Microsoft-Windows-Sysmon/Operational` |

![Alert document details — cleanmgr.exe DLL drop]

<img width="1885" height="925" alt="image" src="https://github.com/user-attachments/assets/48f13467-e066-43c7-bb6b-73eead4fe8ba" />


![Rule 92213 details — Ingress Tool Transfer]

<img width="1903" height="976" alt="image" src="https://github.com/user-attachments/assets/df2188ee-9cb7-443b-85c0-33e63fe9ec2a" />


**Verdict: False positive.** Windows Disk Cleanup (`cleanmgr.exe`) legitimately extracts DLL files to Temp folders during normal operation. The Sysmon EID 11 rule was too broad and flagged this routine behavior as suspicious.

---

### Alert Tuning — Suppression Rule

Rather than disabling the rule entirely, a targeted suppression rule was written to silence it only when the offending process is specifically `cleanmgr.exe`. Any other process dropping executables into Temp folders will still trigger a full level 15 alert.

`DC01` was used to SSH into `WAZUH01` to edit the local rules file:

```powershell
ssh addo@192.168.0.60
```

![SSH into WAZUH01 from DC01]

<img width="644" height="507" alt="image" src="https://github.com/user-attachments/assets/0c2e2b80-7e98-4ad6-bad9-9d845406f96a" />
(screenshots/DC01-ssh-into-wazuh01.png)

The local rules file was opened:

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

**Before** — default example rule only:

![local_rules.xml before changes]

<img width="729" height="356" alt="image" src="https://github.com/user-attachments/assets/8e5753d8-9024-4b81-936a-7e9b5876fa52" />

(screenshots/DC01-ssh-wazuh01-old-config.png)

The following suppression rule was added:

```xml
<group name="sysmon,">
  <rule id="100002" level="0">
    <if_sid>92213</if_sid>
    <field name="win.eventdata.image">cleanmgr.exe</field>
    <description>False positive - Windows Disk Cleanup dropping temp DLL</description>
  </rule>
</group>
```

**After** — suppression rule added below the existing group:

![local_rules.xml after suppression rule added]

<img width="843" height="443" alt="image" src="https://github.com/user-attachments/assets/bea8f4af-5e07-4c84-837f-81e156d02671" />

(screenshots/DC01-ssh-wazuh01-new-config.png)

**Rule breakdown:**

| Element | Purpose |
|---|---|
| `if_sid: 92213` | Only applies when the parent rule fires |
| `field: win.eventdata.image` | Matches specifically on `cleanmgr.exe` |
| `level: 0` | Silences the alert completely |
| `id: 100002` | Custom rule IDs start at 100000 to avoid conflicts with built-in Wazuh rules |

The Wazuh manager was restarted to load the new rule with no syntax errors reported.

---

### Key Takeaways

- Always investigate unexpected alerts before adding attack noise they may reveal real tuning opportunities
- Alert tuning is a core detection engineering skill reducing false positives without weakening coverage
- Wazuh is agent-based, not network-based — it reads endpoint logs, not packet traffic. Basic network scans may not trigger alerts without more aggressive techniques or a dedicated network IDS such as Suricata

---

### Attack 1 — SMB Brute Force Against DC01

#### Objective

Simulate a credential brute force attack against the domain administrator account on `DC01` over SMB and confirm detection in Wazuh.

---

#### Step 1 — Reconnaissance: Port Scan

Before launching the attack, an Nmap SYN scan was run from `KALI-01` against `DC01` to identify open and filtered ports:

```bash
sudo nmap -sS -p 22,80,443,445,3389,5985,389,135 192.168.0.10
```

**Results:**

| Port | State | Service |
|---|---|---|
| `22` | filtered | ssh |
| `80` | filtered | http |
| `135` | open | msrpc |
| `389` | open | ldap |
| `443` | filtered | https |
| `445` | open | microsoft-ds (SMB) |
| `3389` | open | ms-wbt-server (RDP) |
| `5985` | open | wsman (WinRM) |

Port `445` (SMB) was confirmed open — the primary attack surface for credential brute forcing in an Active Directory environment.

![Nmap port scan results against DC01] <img width="637" height="250" alt="image" src="https://github.com/user-attachments/assets/ace92893-ae92-4f4b-b1f1-3a3f6192c33e" />

---

#### Step 2 — Attempted Brute Force with Hydra (Failed)

Initial brute force attempts were made using Hydra across multiple protocols:

- **SMB** — Hydra could not speak SMB2/SMB3, returned `invalid reply`
- **RDP** — Connection failed, likely due to Network Level Authentication (NLA) or GPO restriction on `DC01`
- **WinRM** — Port 5985 not responding as expected

Hydra is not suited for modern Windows environments running SMB2/SMB3. A more capable tool was required.

---

#### Step 3 — CrackMapExec Installation & Brute Force (Succeeded)

CrackMapExec was installed on `KALI-01`:

```bash
sudo apt install crackmapexec -y
```

![CrackMapExec installed on KALI-01] <img width="797" height="176" alt="image" src="https://github.com/user-attachments/assets/b4a3cbfc-818b-4a45-a610-e8a0c11f87f4" />


An SMB brute force attack was launched against `DC01` targeting the `administrator` account using the `rockyou.txt` wordlist:

```bash
crackmapexec smb 192.168.0.10 -u administrator -p /usr/share/wordlists/rockyou.txt
```

CrackMapExec successfully spoke SMB2/SMB3 and began hammering `DC01` with authentication attempts. Each failed attempt returned `STATUS_LOGON_FAILURE`.

```
SMB  192.168.0.10  445  DC01  [-] lab.local\administrator:123456 STATUS_LOGON_FAILURE
SMB  192.168.0.10  445  DC01  [-] lab.local\administrator:password STATUS_LOGON_FAILURE
SMB  192.168.0.10  445  DC01  [-] lab.local\administrator:iloveyou STATUS_LOGON_FAILURE
...
```

![CrackMapExec brute force running against DC01] <img width="1030" height="683" alt="image" src="https://github.com/user-attachments/assets/dbf39e8b-582b-4670-9f49-08b53e0c4045" />


---

#### Step 4 — Detection on DC01 (Windows Event Viewer)

The brute force generated **3,977 Event ID 4625** (An account failed to log on) entries within the last hour, visible in the Windows Security event log on `DC01`.

**Event details:**

| Field | Value |
|---|---|
| Event ID | `4625` |
| Log | Security |
| Task Category | Logon |
| Keywords | Audit Failure |
| Computer | `DC01.lab.local` |
| Description | An account failed to log on |

![DC01 Event Viewer — mass 4625 logon failures] <img width="1029" height="701" alt="image" src="https://github.com/user-attachments/assets/72f0dacb-fd71-46a1-8c1e-83247cb71648" />

---

#### Step 5 — Wazuh Alert Detection

Wazuh detected the brute force pattern and fired **Rule 60204 — Multiple Windows Logon Failures** across **500+ hits** within the attack window.

**Alert summary:**

| Field | Value |
|---|---|
| `rule.id` | `60204` |
| `rule.description` | Multiple Windows Logon Failures |
| `rule.level` | `10` (High) |
| `agent.name` | `DC-01` |
| `MITRE Technique` | T1110 — Brute Force |
| `MITRE Tactic` | Credential Access |

Full event data confirmed the complete attack chain:

| Field | Value |
|---|---|
| Source IP | `192.168.0.80` (KALI-01) ✅ |
| Target account | `administrator@lab.local` ✅ |
| Logon Type | `3` (Network / SMB) ✅ |
| Auth Package | NTLM ✅ |
| Status | `0xC000006D` / SubStatus `0xC000006A` (bad password) ✅ |

![Wazuh — Rule 60204 firing across 500+ events] <img width="1904" height="678" alt="image" src="https://github.com/user-attachments/assets/41f26f00-395b-4ec0-8d67-69d8e82dac1e" />


---

### Tool Migration — CrackMapExec to NetExec

CrackMapExec (5.4.0-kali7) failed on Kali 2026.1 due to a Python 3.13 asyncio incompatibility.

![CrackMapExec Python 3.13 crash] <img width="644" height="411" alt="image" src="https://github.com/user-attachments/assets/30f9b33e-b6b1-4bb0-86d0-43bc65bf1008" />


Investigation confirmed the root cause:

![Python 3.13 and CrackMapExec version] <img width="654" height="379" alt="image" src="https://github.com/user-attachments/assets/6d748fbd-3698-4dc6-b0bc-2a8bdabe55d3" />


- Python 3.13.12 is running on KALI-01
- CrackMapExec 5.4.0-kali7 is installed but crashes under Python 3.13
- `pipx install crackmapexec` also failed — package no longer exists on PyPI

![pipx install failure] <img width="662" height="248" alt="image" src="https://github.com/user-attachments/assets/101445d8-dc20-4bb1-9a62-9f8529b0a95d" />

**Resolution:** Installed NetExec (`nxc`) — the actively maintained successor to CrackMapExec:

```bash
sudo apt install netexec -y
nxc --version
```

![NetExec 1.5.1 confirmed] <img width="479" height="441" alt="image" src="https://github.com/user-attachments/assets/0e7a9709-39cc-4a6c-97e1-e04d11fd4b5d" />

---

### Attack 2 — Account Lockout Policy Validation

#### Account Lockout Policy — DC-01

Configured via Group Policy Management Editor on DC-01:

![Account lockout policy configured] <img width="1007" height="607" alt="image" src="https://github.com/user-attachments/assets/2c429614-8e4d-4b39-891a-15dab0a5cf17" />


| Setting | Value |
|---|---|
| Account lockout threshold | 3 invalid logon attempts |
| Account lockout duration | 30 minutes |
| Reset account lockout counter after | 30 minutes |

#### Test Accounts

All domain users listed to select a test account:

![AD users enumerated on DC-01] <img width="777" height="416" alt="image" src="https://github.com/user-attachments/assets/a5c10dd8-6a57-4784-9f0c-25eda960611c" />


`jasmine.rodgers` selected as the test account. Password set to a known value for testing:

```powershell
Set-ADAccountPassword -Identity jasmine.rodgers -NewPassword (ConvertTo-SecureString "TempPassword123!" -AsPlainText -Force) -Reset
Unlock-ADAccount -Identity jasmine.rodgers
```

#### Attack Executed

Initial run with `rockyou.txt` against `administrator` failed with a UTF-8 decode error:

![NetExec UTF-8 error] <img width="714" height="260" alt="image" src="https://github.com/user-attachments/assets/c85bdb85-5d1c-4615-9d0c-d4913b1314bf" />


Fixed with `--ignore-pw-decoding` flag:

```bash
nxc smb 192.168.0.10 -u jasmine.rodgers -p /usr/share/wordlists/rockyou.txt --ignore-pw-decoding
```

After 3 failed attempts, all subsequent attempts returned `STATUS_ACCOUNT_LOCKED_OUT`:

![Jasmine.rodgers account locked out in NetExec] <img width="693" height="460" alt="image" src="https://github.com/user-attachments/assets/fa33f1bd-df9c-4382-8a90-9e16ace3a532" />


#### Detection in Wazuh

Rule 60115 fired for both accounts:

![Two EID 4740 lockout alerts in Wazuh] <img width="1891" height="322" alt="image" src="https://github.com/user-attachments/assets/b2cdbaf7-fca5-4c20-9ed8-0a06de752966" />


![Rule 60115 detail] <img width="1352" height="566" alt="image" src="https://github.com/user-attachments/assets/e6ddbaef-3811-4cc5-bc1e-2bc1794df350" />

| Field | Value |
|---|---|
| `rule.id` | `60115` |
| `rule.description` | User account locked out (multiple login errors) |
| `rule.level` | `9` (High) |
| `data.win.system.eventID` | `4740` |
| MITRE Techniques | T1110 Brute Force, T1531 Account Access Removal |
| MITRE Tactics | Credential Access, Impact |
| Compliance | PCI DSS 8.1.6/11.4, GDPR IV_35.7.d, HIPAA 164.312.a.1, NIST-800-53 AC.7/SI.4 |

**Two lockout alerts confirmed:**

| Account | Timestamp | Status |
|---|---|---|
| `Administrator` | May 11, 2026 @ 15:02:41 | Locked out ✅ |
| `jasmine.rodgers` | May 11, 2026 @ 15:16:36 | Locked out ✅ |

Full event timeline showing logon failures leading to lockout:

![Wazuh event timeline — failures and lockout] <img width="1915" height="578" alt="image" src="https://github.com/user-attachments/assets/935c9e8d-574f-4b24-84fe-7c4511e51a8a" />


> **Key finding:** The built-in Administrator account is subject to lockout policy on Windows Server 2022 — confirmed by testing.

Accounts unlocked post-testing:

```powershell
Unlock-ADAccount -Identity administrator
Unlock-ADAccount -Identity jasmine.rodgers
```

---

### Attack 3 — LDAP Reconnaissance (T1018)

#### Objective

Simulate post-compromise AD enumeration using the valid credential obtained from the brute force phase.

#### Credential Validation

Credential validated against DC-01 and WS-01 over SMB before LDAP enumeration:

![Credential validated against DC-01] <img width="956" height="170" alt="image" src="https://github.com/user-attachments/assets/0df3e5b2-9875-4ff4-a1fe-5a7f10d258f8" />

![Credential validated against WS-01] <img width="1283" height="331" alt="image" src="https://github.com/user-attachments/assets/1fa2c181-1c4b-4af1-8db4-02730d487b1d" />


> **Security gap noted:** WS-01 shows `signing:False` — SMB signing not enforced, making it vulnerable to SMB relay attacks.

#### LDAP Enumeration Executed

```bash
nxc ldap 192.168.0.10 -u jasmine.rodgers -p 'TempPassword123!' -d lab.local --users
```

![All 20 domain users enumerated via LDAP] <img width="1280" height="480" alt="image" src="https://github.com/user-attachments/assets/b177f2a2-59ec-4851-8422-dca90814dad7" />


All 20 domain users returned including usernames, last password set dates, and bad password counts.

#### Detection in Wazuh — Rule 92652

![Rule 92652 firing on DC-01 and WS-01] <img width="1874" height="455" alt="image" src="https://github.com/user-attachments/assets/1e20b9ac-7ff3-47f8-97c8-369e4e686101" />


| Field | Value |
|---|---|
| `rule.id` | `92652` |
| `rule.description` | Successful Remote Logon Detected - NTLM authentication, possible pass-the-hash attack |
| `rule.level` | `6` |
| `data.win.system.eventID` | `4624` |
| Source IP | `192.168.0.80` (KALI-01) |
| Account | `jasmine.rodgers` |
| Auth Package | NTLM V2 |
| Logon Type | `3` (Network) |
| MITRE Techniques | T1550.002 Pass the Hash, T1078.002 Domain Accounts |
| MITRE Tactics | Defense Evasion, Lateral Movement, Persistence, Privilege Escalation, Initial Access |

Alerts fired on both DC-01 and WS-01.

**Why NTLM flagged as possible Pass the Hash:**
KALI-01 is not domain joined so it falls back to NTLM — behaviorally identical to a real Pass the Hash attack from Wazuh's perspective. A SOC analyst would investigate the source IP and correlate surrounding events to determine legitimacy.

Level 3 logon success noise also observed during this window:

![Level 3 Windows Logon Success alerts — DC01$ machine account] <img width="1906" height="595" alt="image" src="https://github.com/user-attachments/assets/0c2def11-7fd8-4507-a637-6572843055d2" />

These are `DC01$` machine account authentications to its own services — normal DC background activity, not actionable. Suppression planned in the dedicated alert tuning phase.

---

## Decision — Pause Attack Simulation, Build Full Pipeline

Remaining attack techniques (persistence, lateral movement, log clearing) will be executed after the Phase 2 automation pipeline is deployed so the complete SOC workflow fires end to end:

```
KALI-01 attack → Wazuh alert → TheHive case → Cortex enrichment → Shuffle playbook → osTicket ticket
```

---

## Next Step

Phase 2 begins with the automation pipeline. TheHive is deployed first as the case management foundation before Shuffle orchestration is configured.

→ Continue in [`thehive-setup.md`](thehive-setup.md)
