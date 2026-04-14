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
