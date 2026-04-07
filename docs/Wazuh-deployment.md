## Phase 1 — Server Preparation (Cloned VM Cleanup)

### Overview

As part of the SOC environment buildout, a new Ubuntu-based server named **WAZUH01** was prepared to host the Wazuh platform.

This virtual machine was created in proxmox by cloning the existing **INFRA01** system, a dedicated server for osTicket. Because of this, it initially retained the original system identity and network configuration.

This phase ensures the server is uniquely identifiable and properly configured before deploying Wazuh.

---

### Objective

Prepare the cloned server for its role as the dedicated Wazuh server in the lab environment.

**Target configuration:**

| Setting        | Value              |
|---------------|--------------------|
| Hostname      | wazuh01            |
| IP Address    | 192.168.0.60       |
| Gateway       | 192.168.0.1        |
| DNS Server    | 192.168.0.10       |
| Search Domain | lab.local          |

---

## Step 1 — Changed Hostname from INFRA01 to WAZUH01

Because the VM was cloned from INFRA01, it initially retained the original hostname.

The hostname was updated so the system could function as a separate infrastructure node.

### Command used

```bash
sudo hostnamectl set-hostname wazuh01
```
```bash
hostnamectl
```



Explanation

hostnamectl set-hostname wazuh01 changes the system hostname
hostnamectl verifies the updated system identity
Result

The server hostname was successfully changed from infra01 to wazuh01.

📸 Screenshot: <img width="472" height="105" alt="image" src="https://github.com/user-attachments/assets/8c89588f-d851-4fb9-ab0e-6bb912ba4897" />

## Step 2 — Updated Local Host Resolution File

The `/etc/hosts` file was updated so the system correctly resolves its new hostname locally.

### File edited
```bash
sudo nano /etc/hosts
```

### Entry used
```text
127.0.1.1 wazuh01
```

### Explanation
The `/etc/hosts` file allows the system to resolve local hostnames before querying DNS.

Updating this entry ensures the server recognizes itself as **wazuh01**.

### Result
Local hostname resolution now correctly reflects the new server identity.

📸 Screenshot: <img width="632" height="173" alt="image" src="https://github.com/user-attachments/assets/09552f6f-9670-48cf-810c-335e4e02d52e" />

## Step 3 — Configured Static IP Address with Netplan

The cloned server required a permanent IP address so it could reliably function as an infrastructure node.

### File edited
```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

### Configuration applied
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      dhcp4: no
      addresses:
        - 192.168.0.60/24
      routes:
        - to: default
          via: 192.168.0.1
      nameservers:
        addresses:
          - 192.168.0.10
        search:
          - lab.local
```

### Explanation
This configuration:
- disables **DHCP**
- assigns static IP `192.168.0.60`
- sets the default gateway to `192.168.0.1`
- points DNS to the domain controller `192.168.0.10`
- adds the search domain `lab.local`

### Why this matters
Infrastructure servers should not rely on dynamically assigned IP addresses.

A static address ensures **WAZUH01** can always be reached consistently for:
- Wazuh dashboard access
- agent communication
- log ingestion
- documentation accuracy

### Apply configuration
```bash
sudo netplan apply
```

### Result
WAZUH01 was successfully configured with a static IP address and domain-aware DNS settings.

📸 Screenshot: 

<img width="574" height="227" alt="image" src="https://github.com/user-attachments/assets/805f07df-29b6-4c27-8ee2-c034755ba000" />

## Step 4 — Verified Interface, IP Address, and Network Connectivity

After applying the network configuration, the server’s interface details and network connectivity were verified.

### Commands used
```bash
ip a
ping 192.168.0.1
ping 192.168.0.10
```

### Explanation
The `ip a` command displays all network interfaces and their assigned IP addresses.

The relevant interface for this VM is:
- `ens18`

Expected IP address:
- `192.168.0.60/24`

The ping tests confirm network connectivity:

- `ping 192.168.0.1` verifies that WAZUH01 can communicate with the local gateway, which is required for:
  - network routing
  - internet access (package installation, updates)
  - communication across the lab network

- `ping 192.168.0.10` confirms that WAZUH01 can reach **DC01**, which provides:
  - Active Directory services
  - DNS resolution for the lab environment
  - core infrastructure communication

### Result
The WAZUH01 server successfully:
- displayed the configured static IP address on interface **ens18**
- communicated with the gateway at **192.168.0.1** with **0% packet loss**
- reached **192.168.0.10** with **0% packet loss**

This confirms proper interface configuration, network connectivity, and communication with the domain controller.

📸 Screenshot: <img width="727" height="364" alt="image" src="https://github.com/user-attachments/assets/101b4d74-5839-45db-8bf3-c8412cc41bae" />


---

## Phase Summary

At the end of this phase, WAZUH01 was successfully prepared as a standalone infrastructure server.

### Confirmed working:
- Hostname changed to **wazuh01**
- `/etc/hosts` updated correctly
- Static IP configured as **192.168.0.60**
- Gateway set to **192.168.0.1**
- DNS set to **192.168.0.10**
- Search domain set to **lab.local**
- Connectivity to gateway verified
- Connectivity to DC01 verified

---

## Current Lab Environment

| System   | Role                              | IP Address       |
|----------|----------------------------------|------------------|
| DC01     | Active Directory + DNS            | 192.168.0.10     |
| INFRA01  | Ticketing / Infrastructure Server | 192.168.0.50     |
| WAZUH01  | Security Monitoring (Wazuh)       | 192.168.0.60     |

---

## Why This Step Matters

Although this phase appears simple, it is foundational to the overall SOC environment.

Before installing Wazuh, the server must have:
- a unique system identity
- consistent and predictable network addressing
- correct internal DNS configuration
- verified communication with core infrastructure systems

This mirrors real-world enterprise deployment practices, where infrastructure validation is completed before deploying security monitoring platforms.

---

## Deployment Adjustment — OS Compatibility Issue

During the initial attempt to deploy Wazuh, an issue was encountered related to operating system compatibility.

The server (WAZUH01) was running **Ubuntu Server 24.04 LTS**, which resulted in compatibility issues with the Wazuh installation script and package dependencies.

### Issue Identified

While executing the installation process, it became evident that:
- Certain Wazuh components are not fully supported on Ubuntu 24.04
- Installation scripts and dependencies are more stable and documented on Ubuntu 22.04 LTS
- Community and official documentation primarily reference Ubuntu 22.04 environments

This created a risk of:
- incomplete installation
- broken services (indexer, dashboard, or manager)
- additional troubleshooting overhead with limited documentation support

📸 Screenshot: <img width="956" height="205" alt="image" src="https://github.com/user-attachments/assets/2d1acd1e-af3d-4499-89ae-202d66d56881" />


---

## Decision Made

Instead of forcing compatibility or troubleshooting unsupported behavior, the environment was adjusted to align with a **stable and widely supported platform**.

The decision was made to:
- replace Ubuntu 24.04 with **Ubuntu Server 22.04 LTS**
- rebuild the WAZUH01 server using the supported OS version
- proceed with installation using a known, stable baseline

---

## Why This Decision Matters

In real-world environments, engineers prioritize:
- stability over novelty
- supported configurations over experimental setups
- documented deployment paths over unsupported workarounds

Choosing a well-supported OS version ensures:
- smoother installation process
- better compatibility with security tools (Wazuh, TheHive, Shuffle)
- easier troubleshooting using existing documentation and community resources

---

## Key Takeaway

This step highlights an important principle in infrastructure and security engineering:

> Successful deployments are not just about making things work, but making them work reliably within supported and documented environments.

---

## Next Step

Rebuild WAZUH01 using **Ubuntu Server 22.04 LTS** and proceed with a clean installation of the Wazuh stack.

---

## Step — Rebuilt WAZUH01 on Supported OS

Following the decision to move to a supported platform, a new virtual machine was deployed using **Ubuntu Server 22.04 LTS** within the Proxmox environment.

### VM Configuration

The new WAZUH01 server was provisioned with the following specifications:

- **CPU:** 1 core (x86-64-v2-AES)
- **Memory:** 8 GB RAM
- **Storage:** 100 GB (qcow2)
- **Network:** VirtIO (vmbr0 bridge)
- **OS Image:** ubuntu-22.04.5-live-server-amd64.iso

This configuration aligns with Wazuh’s resource requirements for an all-in-one deployment (manager, indexer, dashboard).

📸 Screenshot: Proxmox VM configuration summary 

<img width="748" height="545" alt="image" src="https://github.com/user-attachments/assets/e5ea53c6-f42d-48dd-b06e-21c8668beea5" />

---

## Installation Context

This deployment replaces the previously cloned instance that was based on Ubuntu 24.04.

By rebuilding the system from a clean ISO using a supported OS version, the environment is now:

- aligned with official Wazuh documentation
- free from legacy configuration conflicts
- ready for a predictable installation process

This ensures that any issues encountered moving forward are related to configuration or deployment steps, rather than underlying OS compatibility.

---

## Why This Step Matters

Rebuilding instead of modifying an unsupported system reduces complexity and eliminates hidden variables.

In enterprise environments, this approach is standard practice when:

- base system compatibility is uncertain
- deployment consistency is required across environments
- long-term maintainability is a priority

Starting from a clean, supported OS provides a reliable foundation for security tooling deployment.

---

---

## Step — Base System Configuration (22.04)

With Ubuntu Server 22.04 successfully deployed, the WAZUH01 system was configured to align with the existing lab environment and infrastructure standards.

This phase mirrors the configuration previously completed on earlier systems within the lab. To avoid redundancy, detailed command-level steps are not repeated here, as they are already documented in prior infrastructure setup sections.

### Configuration Completed

The following baseline configurations were applied:

- Hostname set to **wazuh01**
- Local host resolution updated (`/etc/hosts`)
- Static IP configured as **192.168.0.60**
- Default gateway set to **192.168.0.1**
- DNS configured to use **192.168.0.10** (DC01)
- Domain search suffix set to **lab.local**

These settings ensure the system is fully integrated into the lab’s domain-aware network structure.

---

## Verification

System configuration was validated to confirm proper alignment with network and infrastructure requirements.

The following checks were performed:

- OS version confirmed as **Ubuntu 22.04.5 LTS**
- Network interface (**ens18**) shows correct static IP assignment
- Hostname correctly applied and persistent
- Successful connectivity to:
  - **Gateway (192.168.0.1)**
  - **Domain Controller (192.168.0.10)**

All tests returned expected results with **0% packet loss**, confirming stable network communication.

📸 Screenshot: System configuration and connectivity verification

<img width="907" height="783" alt="image" src="https://github.com/user-attachments/assets/e243b1c2-ac36-4a3d-b436-fe75db547771" />

---

## Why This Step Matters

Although this configuration phase is routine, it is critical for ensuring consistency across all infrastructure nodes within the lab.

By reusing a standardized configuration approach:

- deployment becomes predictable and repeatable
- troubleshooting complexity is reduced
- integration with centralized services (DNS, Active Directory) is ensured

This reflects real-world practices where baseline system configuration is standardized across environments before deploying security or monitoring platforms.

---

## Status

The WAZUH01 server is now fully prepared and aligned with the lab environment.

The system is:
- correctly identified
- network-stable
- domain-aware
- ready for application deployment

---

## Wazuh Installation — Command Execution

> "Execution of the Wazuh all-in-one installation on WAZUH01 using the official installer."

### Installation Method

The Wazuh stack was deployed using the official all-in-one installer to align with a single-node architecture.

---

### Commands Executed

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```
📸 Screenshot: Wazuh deployement 

<img width="790" height="127" alt="image" src="https://github.com/user-attachments/assets/fd200441-b764-416a-a375-ad08df4411e9" />



### Wazuh Dashboard Access — Password Reset After Installation

After the Wazuh installation completed, the dashboard login did not accept the initial `admin` credentials shown at the end of the installer output.

To restore dashboard access, the internal Wazuh user password was updated directly from the server.

---

### Applied Command

```bash
sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -u admin -p
```

# wazuh-validation

Post-installation operational verification for a Wazuh all-in-one deployment on `WAZUH01`.

---

## Overview

This document covers validation of the Wazuh platform after a single-node all-in-one installation. The goal was to confirm that all three core components were running, required ports were listening, and that the platform was actively ingesting and displaying event data end-to-end.

> **Note:** API verification was excluded from this validation due to an authentication issue that did not affect core platform operation.

---

## Validation Checklist

| Check | Status |
|---|---|
| `wazuh-manager` service running | ✅ |
| `wazuh-indexer` service running | ✅ |
| `wazuh-dashboard` service running | ✅ |
| Required ports listening | ✅ |
| Local event ingestion | ✅ |
| Dashboard data visibility | ✅ |

---

## 1. Dashboard Access

After installation, the Wazuh web interface was accessed to confirm the dashboard was reachable and rendering correctly.

The Overview page loaded successfully and displayed the full Wazuh feature set across four categories:

**Endpoint Security**
- Configuration Assessment
- Malware Detection
- File Integrity Monitoring

**Threat Intelligence**
- Threat Hunting
- Vulnerability Detection
- MITRE ATT&CK

**Security Operations**
- IT Hygiene
- PCI DSS
- GDPR
- HIPAA

**Cloud Security**
- Docker
- Amazon Web Services
- Google Cloud
- GitHub

The Agents Summary panel showed no agents registered yet, which is expected at this stage of deployment. The Last 24 Hours Alerts panel showed 118 medium severity and 121 low severity alerts — generated by the Wazuh server monitoring itself.

<img width="1916" height="956" alt="image" src="https://github.com/user-attachments/assets/77882b09-c7c8-4ea5-bc2a-775a2fb24118" />


---

## 2. Service Validation

### wazuh-manager

```bash
sudo systemctl status wazuh-manager
```

**Result:** `active (running)`

The following backend processes were confirmed running under the manager:

- `wazuh-authd`
- `wazuh-db`
- `wazuh-execd`
- `wazuh-analysisd`
- `wazuh-syscheckd`
- `wazuh-remoted`
- `wazuh-logcollector`
- `wazuh-monitord`
- `wazuh-modulesd`


<img width="1069" height="559" alt="image" src="https://github.com/user-attachments/assets/7ecbd35a-224c-4a37-88f9-e09c40854ff6" />


---

### wazuh-indexer

```bash
sudo systemctl status wazuh-indexer
```

**Result:** `active (running)`

Startup warnings were present in the log output but did not prevent the service from starting or remaining operational. The indexing backend is available for event storage and querying.

<img width="1021" height="364" alt="image" src="https://github.com/user-attachments/assets/848d00fb-3ff7-4fa1-9e2f-8c61cd06ebd6" />


---

## wazuh-dashboard

```bash
sudo systemctl status wazuh-dashboard
```

**Result:** `active (running)`

Service logs showed active request handling, confirming the dashboard frontend was online and responding.

<img width="1148" height="353" alt="image" src="https://github.com/user-attachments/assets/9041bb44-3910-4049-8895-241f3b483b70" />


---

### 3. Port Verification

```bash
sudo ss -telnp | grep -E "1514|1515|55000|9200|5601"
```

| Port | Purpose | Status |
|---|---|---|
| `1514` | Agent-to-manager event communication | ✅ Listening |
| `1515` | Agent enrollment | ✅ Listening |
| `55000` | Wazuh API | ✅ Listening |
| `9200` | Wazuh indexer / OpenSearch backend | ✅ Listening |
| `5601` | Dashboard interface | — |

<img width="540" height="193" alt="image" src="https://github.com/user-attachments/assets/e21528f4-d83e-4d86-9dbc-112811f0a871" />


---

### 4. Local Event Ingestion Test

A manual test event was generated on the Wazuh server to confirm active log collection and processing:

```bash
sudo logger "WAZUH TEST EVENT"
```

The event was successfully collected and appeared in the dashboard with the following metadata:

- `hostname`: `wazuh01`
- `command`: `/usr/bin/logger 'WAZUH TEST EVENT'`
- `user context`
- `agent ID`
- `manager name`

<img width="798" height="188" alt="image" src="https://github.com/user-attachments/assets/97323790-c63e-49f1-83e4-e9df21b84195" />


---



