# Cortex Setup — HIVE-01

## Overview

Cortex is the observable enrichment engine that TheHive depends on. It takes IPs, file hashes, domains, and URLs from cases and runs them through analyzer modules (VirusTotal, AbuseIPDB, Shodan) automatically, returning enrichment results back to TheHive.

**Cortex must be deployed and running before TheHive is configured** — TheHive requires the Cortex API key during its own setup.

**Deployed on:** HIVE-01 (192.168.0.40, Ubuntu 22.04)

---

## Why a New VM

Cortex and TheHive were originally planned for INFRA-01 (Ubuntu 24.04). During Phase 2 setup it was discovered that the StrangeBee installer does not support Ubuntu 24.04. Rather than rebuild INFRA-01 and disturb the existing osTicket deployment, a new dedicated VM was created.

**INFRA-01** remains as-is running osTicket. The osTicket integration with Shuffle is purely API-based over the flat network — no co-location required.

---

## HIVE-01 VM Specifications

| Setting | Value |
|---|---|
| VM ID | 106 |
| Name | HIVE-01 |
| OS | Ubuntu 22.04 LTS |
| CPU | 1 socket, 4 cores |
| RAM | 8192 MB |
| Disk | 100GB (local-lvm) |
| IP | 192.168.0.40 (static) |
| SSH | addo@192.168.0.40 |

> **Note:** When creating the VM in Proxmox, select **local-lvm** as the disk storage — not local. VM disks on local storage consume the Proxmox root filesystem which has limited space.

![HIVE-01 VM creation confirmation in Proxmox] <img width="689" height="471" alt="image" src="https://github.com/user-attachments/assets/f9ec20ad-34c9-4d69-814d-aff2b3b6b41e" />

---

## Pre-Deployment: Proxmox Storage Cleanup

Before deploying HIVE-01, Proxmox root storage was at 99.98% capacity. This was caused by:

- ISO images accumulating in `/var/lib/vz/template/iso/` (~20GB)
- VM disk images stored on local instead of local-lvm (~71GB)
- An orphaned VM (104) no longer needed

![Proxmox storage pools — local and local-lvm] <img width="810" height="385" alt="image" src="https://github.com/user-attachments/assets/68830fe4-f472-4d15-a41f-8b9c63994ade" />


![local-lvm usage — 12.91% of 876GB used] <img width="853" height="375" alt="image" src="https://github.com/user-attachments/assets/1b1a17bc-69cf-4505-8637-6872e9668e85" />



![local-lvm VM disk inventory before cleanup] <img width="1203" height="460" alt="image" src="https://github.com/user-attachments/assets/a09f0c20-8b59-4531-b106-d355575d603e" />


### Steps taken to recover space

**Delete ISOs no longer needed** (all VMs already installed):
```bash
rm /var/lib/vz/template/iso/kali-linux-2026.1-installer-amd64.iso
rm /var/lib/vz/template/iso/SERVER_EVAL_x64FRE_en-us.iso
rm /var/lib/vz/template/iso/ubuntu-24.04.4-live-server-amd64.iso
rm /var/lib/vz/template/iso/virtio-win-0.1.285.iso
rm /var/lib/vz/template/iso/WS01_user_.iso
```
> The Ubuntu 22.04 ISO was kept until HIVE-01 finished installing.

**Remove orphaned VM 104:**
```bash
qm unlock 104
qm destroy 104
```

**Remove orphaned WAZUH-01 disk image on local** (active disk already on local-lvm):
```bash
rm /var/lib/vz/images/103/vm-103-disk-0.qcow2
qm set 103 --delete unused0
```

**Migrate KALI-01 disk from local to local-lvm:**
```bash
qm stop 105
qm move-disk 105 scsi0 local-lvm --delete
```

**Result:** Proxmox root storage recovered from 99.98% to 9% used (82GB free).

---

## Pre-Deployment: INFRA-01 Resource Check

Before starting, INFRA-01 was verified and scaled down since it only runs osTicket:

![INFRA-01 resources after scaling — 7.8GB RAM, 4 cores] <img width="811" height="317" alt="image" src="https://github.com/user-attachments/assets/b3027bd2-d2c7-4c49-a5ab-cfb68f314376" />


INFRA-01 disk was also unallocated — the Proxmox resize had been applied but the LVM partition had not been extended. Fixed with:

```bash
sudo growpart /dev/sda 3
sudo pvresize /dev/sda3
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

![INFRA-01 disk before LVM extension — lsblk showing sda at 120GB but LVM only 10GB] <img width="730" height="331" alt="image" src="https://github.com/user-attachments/assets/ceab78ec-4802-41f9-a1d0-294e1f2868b5" />


![INFRA-01 disk after LVM extension — root filesystem expanded to 117GB] <img width="956" height="666" alt="image" src="https://github.com/user-attachments/assets/114e9de8-c432-48a4-accb-71ee37dfede7" />


---

## Static IP Configuration

Ubuntu 22.04 Server uses **netplan**, not nmcli. Configure static IP as follows:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Replace contents with:

```yaml
network:
  ethernets:
    ens18:
      dhcp4: no
      addresses:
        - 192.168.0.40/24
      routes:
        - to: default
          via: 192.168.0.1
      nameservers:
        addresses:
          - 192.168.0.10
  version: 2
```

Apply and fix permissions:

```bash
sudo netplan apply
sudo chmod 600 /etc/netplan/00-installer-config.yaml
```

> SSH will drop when `netplan apply` runs. Reconnect on 192.168.0.40. The DHCP lease on the old IP will expire on its own.

---

## Installation

### Step 1 — Install Dependencies

```bash
sudo apt install -y wget curl gnupg apt-transport-https git ca-certificates
```

### Step 2 — Install Java 11

Cortex is a Java application and requires Java 11 specifically.

```bash
sudo apt install -y openjdk-11-jdk
```

![Java 11 installation on HIVE-01] <img width="892" height="214" alt="image" src="https://github.com/user-attachments/assets/11690696-e458-48dc-8fe9-16b1c854d441" />


Set the JAVA_HOME environment variable so Cortex can find Java:

```bash
echo JAVA_HOME="/usr/lib/jvm/java-11-openjdk-amd64" | sudo tee -a /etc/environment
source /etc/environment
```

Verify:

```bash
java -version
# Expected: openjdk version "11.x.x"
```

![Java 11 version confirmed on HIVE-01] <img width="822" height="138" alt="image" src="https://github.com/user-attachments/assets/cf2bbc63-71c0-4722-9afa-80413fdc0ba3" />


### Step 3 — Install Docker

The StrangeBee install script contains a bug where it adds the Debian Docker repository instead of the Ubuntu one, causing installation to fail. Docker must be installed manually before running the script.

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor --yes -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu jammy stable" | sudo tee /etc/apt/sources.list.d/docker.list

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
```

Verify Docker is working:

```bash
sudo docker run hello-world
```

### Step 4 — Install Elasticsearch 7.x

The StrangeBee install script handles Elasticsearch correctly. Run the script and select **option 3 (Install Cortex with Docker-based analyzers)**:

```bash
wget -q -O /tmp/install_script.sh https://scripts.download.strangebee.com/latest/sh/install_script.sh
bash /tmp/install_script.sh
```

> The script will warn about Ubuntu 22.04 compatibility — type `yes` to continue. It installs Elasticsearch 7.17.x and configures it correctly, then fails on the Docker step (known bug). That is expected — Docker is already installed from Step 3.

![StrangeBee script failing on Docker step — expected, Docker already installed] <img width="947" height="256" alt="image" src="https://github.com/user-attachments/assets/9fef0376-0587-4f02-bce7-c00f52cdcf93" />


Elasticsearch installed: `7.17.29`

![Elasticsearch running — active status confirmed] <img width="939" height="310" alt="image" src="https://github.com/user-attachments/assets/71174678-b63d-446d-8d3f-3ca5c63eb34a" />

### Step 5 — Install Cortex

The StrangeBee APT repository has been deprecated. The old repository URL `deb.thehive-project.org` no longer resolves. Install Cortex directly from the `.deb` package:

![Deprecated APT repository failing to resolve] <img width="848" height="366" alt="image" src="https://github.com/user-attachments/assets/6b3382a7-8d86-4e11-87e9-1f490ba24567" />

```bash
wget -O /tmp/cortex.deb https://cortex.download.strangebee.com/3.2/deb/cortex_3.2.1-2_all.deb
sudo apt install -y /tmp/cortex.deb
```

![Cortex .deb downloaded and installed successfully] <img width="911" height="322" alt="image" src="https://github.com/user-attachments/assets/51779baf-ab23-42ef-80ec-d1375414ce03" />

---

## Configuration

### Step 6 — Generate Secret Key

Cortex requires a secret key for cryptographic functions. Generate one:

```bash
cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 64 | head -n 1
```

![Secret key generated] <img width="641" height="18" alt="image" src="https://github.com/user-attachments/assets/52c24e6d-036f-4c41-959e-fe3c94736573" />


Save the output — it will be added to the config file.

### Step 7 — Edit application.conf

```bash
sudo nano /etc/cortex/application.conf
```

**Two changes required:**

**1. Uncomment and set the secret key** (line 9):
```
play.http.secret.key="YOUR_64_CHARACTER_KEY_HERE"
```
> The `#` at the start of this line must be removed. Cortex will crash on startup if the key remains commented out.

**2. Update the analyzer catalog URL:**

Find the analyzer URLs block and replace the old URL:
```
# Old (dead):
"https://download.thehive-project.org/analyzers.json"

# Replace with:
"https://catalogs.download.strangebee.com/latest/json/analyzers.json"
```

![application.conf — secret key uncommented and analyzer URL updated] <img width="589" height="90" alt="image" src="https://github.com/user-attachments/assets/b9dcdf66-e1aa-44b1-b6cb-44add1f852ea" />


### Step 8 — Add Cortex User to Docker Group

Cortex runs as the `cortex` system user. It needs permission to run Docker containers for analyzers:

```bash
sudo usermod -aG docker cortex
```

---

## Starting Services

Start Elasticsearch first — Cortex depends on it:

```bash
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
sudo systemctl status elasticsearch
# Expected: active (running)
```

Start Cortex:

```bash
sudo systemctl enable cortex
sudo systemctl start cortex
sudo systemctl status cortex
```

![Cortex service running — active status confirmed] <img width="899" height="227" alt="image" src="https://github.com/user-attachments/assets/b265203d-347c-4c03-9356-87148813c89c" />


Verify Cortex is listening on port 9001:

```bash
sudo ss -tlnp | grep 9001
# Expected: LISTEN on *:9001
```

---

## First-Run Initialization

### Known Issue — UI Database Button Broken

Navigating to `http://192.168.0.40:9001` shows a blank screen on first run. This is a known bug in Cortex 3.2 — the "Update Database" button at `/index.html#!/maintenance` renders but does not execute the migration.

![Cortex blank screen on first load] <img width="1018" height="722" alt="image" src="https://github.com/user-attachments/assets/ef441e7a-1bcc-4ee4-bf70-ef9d7e7e03bd" />


![Browser console showing user init not found 404 error] <img width="947" height="427" alt="image" src="https://github.com/user-attachments/assets/0fe14e2a-4f0e-4f1d-a137-2957d346886e" />


![Update Database button visible but non-functional] <img width="933" height="296" alt="image" src="https://github.com/user-attachments/assets/46889c2f-46c7-4d90-997e-7d00ee1b875a" />

The organization and admin user must be created manually via the API.

**Create the default organization:**

```bash
curl -X POST "http://127.0.0.1:9001/api/organization" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "cortex",
    "description": "Default organization",
    "status": "Active"
  }'
```

**Create the admin user:**

```bash
curl -X POST "http://127.0.0.1:9001/api/user" \
  -H "Content-Type: application/json" \
  -d '{
    "login": "admin",
    "name": "Admin",
    "roles": ["superadmin"],
    "password": "YOUR_ADMIN_PASSWORD"
  }'
```

![Organization and admin user created via API] <img width="877" height="463" alt="image" src="https://github.com/user-attachments/assets/474c8c55-91ee-41a2-8fd1-64c6895642e2" />


**Login:**

Navigate to `http://192.168.0.40:9001/index.html#!/login` and sign in with the admin credentials.

---

## Verification

Cortex dashboard accessible at `http://192.168.0.40:9001`

| Check | Expected |
|---|---|
| Elasticsearch status | active (running) |
| Cortex status | active (running) |
| Port 9001 | LISTEN |
| Dashboard | Organizations page visible after login |
| Organization | cortex — Active |
| Version | 3.2.1+2 |

![Cortex dashboard — Organizations page, cortex org active, version 3.2.1+2] <img width="991" height="548" alt="image" src="https://github.com/user-attachments/assets/e2971b10-ce3f-402d-b92a-25c88109d061" />


---

## Troubleshooting

### Cortex crashes on startup — exit code 255

Check the application log:

```bash
sudo cat /var/log/cortex/application.log | tail -50
```

Most common cause: secret key is still commented out in `application.conf`. Verify with:

```bash
sudo grep "secret.key" /etc/cortex/application.conf
```

The line must not start with `#`.

![Cortex log analysis — identifying crash cause] <img width="868" height="356" alt="image" src="https://github.com/user-attachments/assets/a47b25ea-8e47-429d-be5d-d4445e9da3c6" />


### Cortex starts but port 9001 not listening

![Cortex startup issue — port not binding] <img width="1017" height="143" alt="image" src="https://github.com/user-attachments/assets/3eb37500-1721-4c38-b07d-6b8f6a897f1f" />

Check if Elasticsearch is running first:

```bash
curl http://127.0.0.1:9200
```

Cortex will not bind its port if it cannot reach Elasticsearch on startup.

---

## Lessons Learned

- **Ubuntu 24.04 is not supported** by the StrangeBee installer — use 22.04
- **The StrangeBee install script has a Docker bug** — it adds the Debian repo instead of Ubuntu. Install Docker manually before running the script
- **The APT repository for Cortex is deprecated** — install via direct `.deb` download from `cortex.download.strangebee.com`
- **The secret key must be uncommented** in `application.conf` — Cortex crashes silently if it remains commented out
- **The UI database initialization button is broken** in Cortex 3.2 — create the organization and admin user via the API directly
- **VM disks must be placed on local-lvm**, not local — local storage shares the Proxmox root filesystem

---

## Next Step

TheHive is deployed next and connects to Cortex using the API key generated from the Cortex dashboard.

→ Continue in [`thehive-setup.md`](thehive-setup.md)
