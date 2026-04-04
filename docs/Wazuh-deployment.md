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

## Next Step

Proceed with base system configuration on WAZUH01, including:

- hostname configuration
- static IP assignment
- DNS and domain alignment
- connectivity validation

Once the system is fully prepared, the Wazuh installation process will begin.
