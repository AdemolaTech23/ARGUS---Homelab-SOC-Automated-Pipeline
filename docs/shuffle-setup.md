# Shuffle SOAR — Deployment and Wazuh Integration

This document covers the deployment of Shuffle on the physical Windows machine and its integration with Wazuh as the first step in the ARGUS automation pipeline.

**Target pipeline position:**
```
KALI-01 attack → Wazuh alert → Shuffle webhook → TheHive case → Cortex enrichment → osTicket ticket
```

---

## Prerequisites

| Requirement | Details |
|---|---|
| Physical Windows machine | 192.168.0.101, 24GB RAM, 1TB external drive (A:) |
| Docker Desktop | Installed to A:\Docker, data at A:\DockerData |
| Git | v2.54.0 |
| WAZUH-01 | Running at 192.168.0.60 |
| All VMs reachable | Flat 192.168.0.0/24 network via vmbr0 |

---

## Step 1 — Set Static IP on Windows Machine

The Windows machine needs a fixed IP so Wazuh always knows where to send webhooks.

Check current IP:
```
ipconfig
```

Set static IP:
```
netsh interface ip set address "Ethernet" static 192.168.0.101 255.255.255.0 192.168.0.1
```

Set primary DNS to Google, alternate to DC-01:
```
netsh interface ip set dns "Ethernet" static 8.8.8.8
netsh interface ip add dns "Ethernet" 192.168.0.10 index=2
```

**Why:** Wazuh's ossec.conf webhook URL is hardcoded to the Windows machine IP. If the IP changes via DHCP, the integration breaks. DC-01 as alternate DNS allows `lab.local` hostname resolution when needed.

![Static IP assigned](https://raw.githubusercontent.com/AdemolaTech23/ARGUS---Homelab-SOC-Automated-Pipeline/main/docs/screenshots/Ip_statically_assigned.png)

---

## Step 2 — Install Docker Desktop

Docker Desktop is required to run Shuffle's containers on Windows. It uses WSL 2 (Windows Subsystem for Linux) as a backend — a lightweight Linux VM that provides the Linux kernel containers need. This is different from Linux-native Docker (used on HIVE-01) where Docker talks directly to the OS kernel.

**Why WSL 2 matters:** Windows doesn't natively support Linux containers. Docker Desktop creates a hidden WSL 2 Linux VM, and all containers run inside it. The experience feels identical from the command line, but the architecture is:

```
Windows (your machine)
    └── WSL 2 Linux VM (managed by Docker Desktop)
            └── Shuffle containers (share the WSL 2 kernel)
```

![Windows Subsystem for Linux](https://raw.githubusercontent.com/AdemolaTech23/ARGUS---Homelab-SOC-Automated-Pipeline/main/docs/screenshots/Windows_Subsystem_for_linux_enabled.png)

Download Docker Desktop from `https://www.docker.com/products/docker-desktop/` and install to the external drive:

```
"C:\Users\Ademo\Downloads\Docker Desktop Installer.exe" install --installation-dir="A:\Docker"
```

**Configuration options during install:**
- All-users installation ✅
- Use WSL 2 instead of Hyper-V ✅
- Allow Windows Containers — leave unchecked

After installation, set Docker data directory to external drive:

**Docker Desktop → Settings → Resources → Advanced → Disk image location → `A:\DockerData`** → Apply & Restart

**Note:** VMware Workstation Pro 15.5.5+ supports running alongside Hyper-V/WSL 2. Older versions may cause VM slowdowns.

---

## Step 3 — Install Git

Git is required to clone the Shuffle repository from GitHub.

Download Git 2.54.0 for Windows (64-bit) from `https://git-scm.com/download/win`. Run the installer with all default options.

Verify in Git Bash:
```
git --version
```

Expected output:
```
git version 2.54.0.windows.1
```

![Git installed](https://raw.githubusercontent.com/AdemolaTech23/ARGUS---Homelab-SOC-Automated-Pipeline/main/docs/screenshots/Installed.png)

---

## Step 4 — Clone and Start Shuffle

Clone the Shuffle repository in Git Bash:
```
git clone https://github.com/Shuffle/Shuffle
```

**What cloning does:** Downloads an exact copy of the Shuffle project from GitHub — all source code, configuration files, and the `docker-compose.yml` file that tells Docker which containers to spin up and how to configure them.

Navigate into the folder:
```
cd Shuffle
```

Start all Shuffle containers:
```
docker-compose up -d
```

The `-d` flag runs containers in detached mode — they run in the background without occupying the terminal.

**What docker-compose does:** Reads `docker-compose.yml` and starts four containers:

| Container | Role |
|---|---|
| shuffle-frontend | Web UI served on port 3001 |
| shuffle-backend | Core API and workflow logic on port 5001 |
| shuffle-orborus | Worker that executes workflow steps |
| shuffle-opensearch | Database storing workflows and execution history |

![Shuffle started via Docker](https://raw.githubusercontent.com/AdemolaTech23/ARGUS---Homelab-SOC-Automated-Pipeline/main/docs/screenshots/Shuffle_started_via_docker.png)

Allow Windows Defender Firewall access when prompted — select **Private networks only**.

![Windows Defender Firewall alert](https://raw.githubusercontent.com/AdemolaTech23/ARGUS---Homelab-SOC-Automated-Pipeline/main/docs/screenshots/Windows_security_Alert.png)

Verify all containers are running:
```
docker ps
```

All four containers should show `Up` status.

---

## Step 5 — Shuffle First Login

Open browser and navigate to:
```
http://localhost:3001
```

Or from other machines on the network:
```
http://192.168.0.101:3001
```

Create the administrator account on first load.

![Shuffle frontend loaded](https://raw.githubusercontent.com/AdemolaTech23/ARGUS---Homelab-SOC-Automated-Pipeline/main/docs/screenshots/Shuffle_Front_end_loaded_locally.png)

---

## Step 6 — Configure Wazuh Webhook Integration

### Create Webhook Trigger in Shuffle

1. In Shuffle, click **Workflows** → **New Workflow**
2. Name: `Wazuh-to-TheHive`, Description: `Receives Wazuh alerts via webhook and creates TheHive cases`
3. In the workflow editor, drag the **Webhook** trigger from the Triggers panel onto the canvas
4. Click the webhook node — copy the **Webhook URI** shown in the right panel
5. Replace `localhost` with `192.168.0.101` in the URL
6. Click **Start** then **Save**

**Why build from scratch (not a template):** Templates auto-generate steps configured for generic environments — wrong IPs, wrong API keys, wrong field mappings. Building manually means you understand exactly what each step does and the workflow is correctly configured for your environment. It also demonstrates genuine understanding on a portfolio project.

![Webhook URI in Shuffle](https://raw.githubusercontent.com/AdemolaTech23/ARGUS---Homelab-SOC-Automated-Pipeline/main/docs/screenshots/Webhook_given.png)

**Note on webhook IDs:** Each webhook trigger node gets a unique ID. If you recreate the workflow or add a new webhook node, a new ID is generated. Always update `ossec.conf` to match the current active webhook ID.

### Configure Wazuh to Send Alerts to Shuffle

SSH into WAZUH-01:
```
ssh addo@192.168.0.60
```

Edit the main Wazuh configuration file:
```
sudo nano /var/ossec/etc/ossec.conf
```

**What ossec.conf is:** The master configuration file for the entire Wazuh installation. The name `ossec` comes from OSSEC (Open Source HIDS SECurity), the open source project Wazuh was forked from in 2015. The directory name `/var/ossec/` is a legacy of that origin. All configuration files live in `/var/ossec/etc/` following Linux convention — configuration always goes in `etc`.

Add this block before the closing `</ossec_config>` tag:

```xml
<integration>
  <name>shuffle</name>
  <hook_url>http://192.168.0.101:3001/api/v1/hooks/YOUR_WEBHOOK_ID</hook_url>
  <level>6</level>
  <alert_format>json</alert_format>
</integration>
```

**XML structure breakdown:**
- `<integration>` — opens the integration block
- `<name>shuffle</name>` — tells Wazuh which integration script to run
- `<hook_url>` — destination URL for alert data
- `<level>6</level>` — minimum alert level threshold. Only alerts level 6+ are forwarded
- `<alert_format>json</alert_format>` — format Shuffle expects
- `</ossec_config>` — closes the entire configuration file

The valid tags and their meanings are defined by Wazuh's schema — documented at `https://documentation.wazuh.com`. Detection engineers refer to this constantly when building integrations.

Save and exit: `Ctrl+X` → `Y` → `Enter`

Restart Wazuh to apply:
```
sudo systemctl restart wazuh-manager
```

Verify restart:
```
sudo systemctl status wazuh-manager
```

Expected: `active (running)`

![Wazuh configured with Shuffle webhook](https://raw.githubusercontent.com/AdemolaTech23/ARGUS---Homelab-SOC-Automated-Pipeline/main/docs/screenshots/SSH_into_wazuh-configured-connection-to-shuffle-webhook.png)

---

## Step 7 — Verify Integration

### Confirm Wazuh is Sending

Check the integration log on WAZUH-01:
```
sudo tail -f /var/ossec/logs/integrations.log
```

Each line shows an alert file being sent to the webhook URL:
```
/tmp/shuffle-<timestamp>-<id>.alert  http://192.168.0.101:3001/api/v1/hooks/<webhook-id>
```

**What this output means:** Wazuh writes each alert to a temporary file in `/tmp/`, then sends it to the webhook URL. Seeing your webhook ID in these lines confirms Wazuh is sending to the correct destination.

### Generate a Test Alert

From KALI-01, create a small test password list:
```
echo -e "password\n123456\nwelcome1\nPassword1\nSummer2024" > /tmp/test-passwords.txt
```

Run SMB brute force against DC-01:
```
nxc smb 192.168.0.10 -u jasmine.rodgers -p /tmp/test-passwords.txt --ignore-pw-decoding
```

**`--ignore-pw-decoding` explained:** The rockyou.txt wordlist contains passwords with non-UTF-8 characters from older encodings. This flag tells nxc to skip undecodable characters instead of crashing.

Expected output: `STATUS_LOGON_FAILURE` for first attempts, then `STATUS_ACCOUNT_LOCKED_OUT` after hitting the 3-attempt lockout policy.

![Kali attack simulation](https://raw.githubusercontent.com/AdemolaTech23/ARGUS---Homelab-SOC-Automated-Pipeline/main/docs/screenshots/Kali_Attack_simulation___shuffle_webhook_test__.png)

### Confirm Alerts in Wazuh

Expected rules to fire:

| Rule ID | Description | Level |
|---|---|---|
| 60122 | Logon Failure - Unknown user or bad password | 5 |
| 60115 | User account locked out (multiple login errors) | 9 |
| 92652 | NTLM Anonymous Logon / possible pass-the-hash | 6 |

Rule 60115 (level 9) and Rule 92652 (level 6) both exceed the level 6 webhook threshold and will be forwarded to Shuffle.

![Alerts generated in Wazuh](https://raw.githubusercontent.com/AdemolaTech23/ARGUS---Homelab-SOC-Automated-Pipeline/main/docs/screenshots/Alert_generated_in_wazuh___shuffle_webhook_test__.png)

### Confirm Shuffle Received Alerts

In Shuffle, click the **Webhook 1** node in the workflow canvas. The right panel shows the received alert JSON:

```json
{
  "severity": 3,
  "pretext": "WAZUH Alert",
  "title": "User account locked out (multiple login errors)",
  "rule_id": "60115",
  "timestamp": "2026-06-20T16:57:09.882+0000"
}
```

![Shuffle receives Wazuh alerts](https://raw.githubusercontent.com/AdemolaTech23/ARGUS---Homelab-SOC-Automated-Pipeline/main/docs/screenshots/Shuffle_recieves_wazuh_alerts.png)
![Shuffle receives Wazuh alerts detail](https://raw.githubusercontent.com/AdemolaTech23/ARGUS---Homelab-SOC-Automated-Pipeline/main/docs/screenshots/Shuffle_recieves_wazuh_alerts_1.png)
![All Wazuh alerts in Shuffle](https://raw.githubusercontent.com/AdemolaTech23/ARGUS---Homelab-SOC-Automated-Pipeline/main/docs/screenshots/Wazuh___all_alert_generated__.png)

**Pipeline verified:** KALI-01 → DC-01 → Wazuh → Shuffle ✅

---

## Alert Suppression — DC01$ Machine Account Noise

During testing, DC-01 generates high-volume machine account logon/logoff events (Rules 60106 and 60137) from DC01$ that flood the alert view. These are normal Domain Controller background activity — not actionable.

Add suppression rule to `/var/ossec/etc/rules/local_rules.xml`:

```xml
<group name="windows,">
  <rule id="100003" level="0">
    <if_sid>60106,60137</if_sid>
    <field name="win.eventdata.targetUserName">DC01\$</field>
    <description>False positive - DC01$ machine account logon/logoff noise</description>
  </rule>
</group>
```

Restart Wazuh after adding: `sudo systemctl restart wazuh-manager`

---

## Lessons Learned

- **Docker Desktop on Windows uses WSL 2** — a full Linux VM runs silently in the background. All containers run inside it, not natively on Windows
- **VMware Workstation Pro + Hyper-V** — version 15.5.5+ supports coexistence. Older versions may cause VM slowdowns
- **Docker data must be redirected to external drive** — C: drive space is limited; set disk image location in Docker Desktop settings before pulling any images
- **Install Docker to external drive via CLI** — use `--installation-dir` flag in PowerShell
- **Shuffle webhook IDs change** — every new webhook trigger node gets a unique ID. Always update ossec.conf to match the active webhook ID
- **Shuffle workflow must be saved AND webhook started** — saving the workflow is not enough; the webhook trigger must be explicitly started via the Start button
- **Docker containers need internet access** — if containers can't pull images after first run, restart Docker Desktop completely; WSL 2 DNS can get stuck
- **Wazuh integrations.log** — always check this first when debugging webhook delivery. It shows exactly what URL each alert is being sent to
- **localhost vs IP in webhook URLs** — Shuffle's webhook URI uses `localhost` by default. Always replace with `192.168.0.101` for external systems like Wazuh to reach it

---

## Verification Table

| Check | Command/Location | Expected Result |
|---|---|---|
| Docker containers running | `docker ps` | 4 containers Up |
| Shuffle UI accessible | `http://192.168.0.101:3001` | Login page loads |
| Wazuh sending webhooks | `sudo tail -f /var/ossec/logs/integrations.log` | Lines with webhook ID |
| Shuffle receiving alerts | Shuffle workflow → Webhook node | Alert JSON visible |
| Attack detected in Wazuh | Search `rule.id:60115` | Account lockout alert |

---

## Next Step

With Wazuh alerts flowing into Shuffle, the next step is building the playbook actions to automate case creation in TheHive.

→ Continue in [`shuffle-playbooks.md`](shuffle-playbooks.md)
