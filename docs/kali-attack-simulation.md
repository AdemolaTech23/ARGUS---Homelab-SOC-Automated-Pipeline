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

## Verification

After installation, network connectivity was confirmed from the terminal:

```bash
ping 192.168.0.1   # gateway
ping 8.8.8.8       # internet
```

**Results:**

- Gateway (`192.168.0.1`): 4 packets transmitted, 4 received, **0% packet loss**
- Internet (`8.8.8.8`): 4 packets transmitted, 4 received, **0% packet loss**

Both confirmed KALI-01 is live on the `192.168.0.0/24` network with full internet access.

![KALI-01 network connectivity verified]

<img width="655" height="515" alt="image" src="https://github.com/user-attachments/assets/cbb9424b-bf1c-428d-aa9d-986f29121808" />
(screenshots/KALI01-vm-installed.png)

---

## Current Lab Environment

| Hostname | Role | OS | IP Address |
|---|---|---|---|
| `DC01` | Domain Controller | Windows Server 2022 | `192.168.0.10` |
| `WS01` | Workstation | Windows 10 Pro | `192.168.0.70` |
| `INFRA01` | Infrastructure Server | Ubuntu 24.04.4 LTS | `192.168.0.50` |
| `WAZUH01` | SIEM | Ubuntu 22.04 LTS | `192.168.0.60` |
| `KALI-01` | Attack Simulation | Kali Linux 2026.1 | `192.168.0.x` |

---

## Attack Simulation
