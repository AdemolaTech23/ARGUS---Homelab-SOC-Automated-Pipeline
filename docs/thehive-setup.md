# TheHive Setup — HIVE-01

## Overview

TheHive is the security case management platform at the centre of the ARGUS automation pipeline. Every Wazuh alert above a configured threshold automatically creates a case in TheHive, which is then enriched via Cortex and escalated to osTicket through Shuffle.

**Deployed on:** HIVE-01 (192.168.0.40, Ubuntu 22.04) — same VM as Cortex
**Port:** 9000
**Version:** 5.5.3-1

> **Dependency:** Cortex must be fully deployed and running before TheHive is configured. See [`cortex-setup.md`](cortex-setup.md) for VM creation, storage cleanup, Java, Docker, and Elasticsearch setup — all prerequisites are covered there and are not repeated here.

---

## Architecture

TheHive and Cortex share the same HIVE-01 VM and the same Elasticsearch 7.x instance. TheHive uses Cassandra as its primary database backend.

| Service | Port | Purpose |
|---|---|---|
| Cassandra | 9042 | TheHive primary database |
| Elasticsearch | 9200 | TheHive index engine (shared with Cortex) |
| Cortex | 9001 | Enrichment engine |
| TheHive | 9000 | Case management dashboard |

---

## Installation

### Step 1 — Install Cassandra 4.1.x

Cassandra is TheHive's primary database. It must be installed and running before TheHive starts.

Add the Cassandra repository:

```bash
wget -qO - https://downloads.apache.org/cassandra/KEYS | sudo gpg --dearmor -o /usr/share/keyrings/cassandra-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/cassandra-archive-keyring.gpg] https://debian.cassandra.apache.org 41x main" | sudo tee /etc/apt/sources.list.d/cassandra.list
sudo apt update
```

![Cassandra repository added and apt updated] <img width="1022" height="288" alt="image" src="https://github.com/user-attachments/assets/adf9ad5d-7c32-4c85-8564-12c47e2ff4a3" />
ng)

Install and start Cassandra:

```bash
sudo apt install -y cassandra
sudo systemctl enable cassandra
sudo systemctl start cassandra
sudo systemctl status cassandra
# Expected: active (running)
```

![Cassandra service running — active status confirmed] <img width="904" height="286" alt="image" src="https://github.com/user-attachments/assets/5633e0e7-3933-4ed1-95d7-149ad56e4d1b" />

> **Note:** During `apt update` you may see a 404 error for the Docker Debian repository — this is a known leftover from the Cortex installation. It does not affect Cassandra installation.

---

### Step 2 — Install TheHive

The StrangeBee APT repository is deprecated. Install TheHive directly from the `.deb` package:

```bash
wget -O /tmp/thehive.deb https://thehive.download.strangebee.com/5.5/deb/thehive_5.5.3-1_all.deb
sudo apt install -y /tmp/thehive.deb
```

![TheHive 5.5.3-1 downloaded (445MB) and installed] <img width="1012" height="305" alt="image" src="https://github.com/user-attachments/assets/bf07acf0-0d8f-4e48-a1ad-d63d7f631749" />


---

## Configuration

### Step 3 — Edit application.conf

```bash
sudo nano /etc/thehive/application.conf
```

The default config is correct for a local setup. Two changes required:

**1. Add Cortex connector block** at the bottom of the file:

```
play.modules.enabled += connector.cortex.CortexConnector
cortex {
  servers = [
    {
      name = "Cortex"
      url = "http://127.0.0.1:9001"
      auth {
        type = "bearer"
        key = "YOUR_CORTEX_API_KEY"
      }
    }
  ]
}
```

> Replace `YOUR_CORTEX_API_KEY` with the API key generated from the Cortex dashboard. Since both services are on the same VM, the URL uses localhost.

**2. Update the base URL:**

Find:
```
application.baseUrl = "http://localhost:9000"
```

Change to:
```
application.baseUrl = "http://192.168.0.40:9000"
```

---

### Step 4 — Create File Storage Directory

```bash
sudo mkdir -p /opt/thp/thehive/files
sudo chown -R thehive:thehive /opt/thp/thehive
```

---

## Starting TheHive

```bash
sudo systemctl enable thehive
sudo systemctl start thehive
sudo systemctl status thehive
# Expected: active (running)
```

![TheHive service active — configuration steps and running status] <img width="1066" height="409" alt="image" src="https://github.com/user-attachments/assets/f78b5a2b-c4ef-4256-8466-cc474532a38b" />

> TheHive takes 2-3 minutes to fully initialize on first start. Wait before opening the dashboard.

---

## First Login

Navigate to `http://192.168.0.40:9000`

![TheHive 5.5.3-1 login page] <img width="1000" height="668" alt="image" src="https://github.com/user-attachments/assets/c5ff03fe-ef64-44bd-9d44-0d513c6cc2b4" />

Default credentials:
- **Login:** admin@thehive.local
- **Password:** secret

> Change the default password immediately after first login.

---

## Cortex Integration Verification

1. Go to **Admin** → **Platform Management** → **Connectors** → **Cortex**
2. Click **Test server connection**

![TheHive Cortex connector configuration — server URL and API key] <img width="779" height="594" alt="image" src="https://github.com/user-attachments/assets/09ce78c6-506a-4945-94fe-42f0e9b8140d" />


Expected result: **"Cortex configuration has been successfully tested"**

> **Troubleshooting:** If the test returns a 401 Authentication Error, Cortex may be in a broken initialization state. Fix by wiping the Cortex index and reinitializing through the UI:
> ```bash
> curl -X DELETE http://127.0.0.1:9200/cortex_6
> sudo systemctl restart cortex
> ```
> Navigate to `http://192.168.0.40:9001`, click **Update Database**, create the ARGUS organization and admin user through the proper first-run flow, generate a new API key, and update the TheHive connector.

---

## Cortex Analyzer Configuration

With TheHive and Cortex connected, enable analyzers in Cortex so TheHive can trigger enrichment automatically.

In Cortex at `http://192.168.0.40:9001`, go to **Organization** → **Analyzers**.

![Cortex analyzer tab — 268 available analyzers loaded from catalog] <img width="1849" height="839" alt="image" src="https://github.com/user-attachments/assets/bd9c2da1-7472-4dea-a441-c91481c8e228" />


Enable and configure these three analyzers for the Phase 2 pipeline:

| Analyzer | Data Type | Requires |
|---|---|---|
| AbuseIPDB_2_0 | IP address | AbuseIPDB API key |
| Shodan_Host_1_0 | IP address | Shodan API key |
| VirusTotal_GetReport_3_1 | IP, hash, domain, URL | VirusTotal API key |

Get free API keys from:
- **AbuseIPDB:** https://www.abuseipdb.com/register
- **Shodan:** https://account.shodan.io/register
- **VirusTotal:** https://www.virustotal.com/gui/join-us

Click **Enable** on each analyzer and enter the API key when prompted.

![Three analyzers enabled — AbuseIPDB, Shodan, VirusTotal confirmed active](screenshots/Hive01-Cortex_3AnalyzersAdded.png)

### Analyzer Verification

Run a test analysis against a known public IP to confirm all three analyzers are working. In Cortex go to **New Analysis**, enter `8.8.8.8` as an IP observable, and run all three analyzers.

![All three analyzers returning Success against 8.8.8.8 — TLP:AMBER, PAP:AMBER](screenshots/Hive01-Cortex_3AnalyzersQueryAgainstGoogle.png)

Expected results for `8.8.8.8`:
- **AbuseIPDB** — Abuse score 0%, whitelisted (Google DNS)
- **Shodan** — Open ports 53 and 443, owner Google LLC, Mountain View CA
- **VirusTotal** — 0/91 engines flagged, reputation score +543
---

## Verification

| Check | Expected |
|---|---|
| Cassandra status | active (running) |
| Elasticsearch status | active (running) |
| TheHive status | active (running) |
| Port 9000 | accessible from browser |
| Default login | admin@thehive.local / secret (change immediately) |
| Cortex connector | Successfully tested |
| Version | 5.5.3-1 |

---

## Lessons Learned

- **Cassandra must be running before TheHive starts** — TheHive fails silently if Cassandra is unavailable
- **TheHive and Cortex share Elasticsearch** — no separate instance needed
- **Use `127.0.0.1` in the Cortex connector URL** — not the VM IP, since both services are on the same machine
- **TheHive takes 2-3 minutes to start** — opening the dashboard immediately will show a blank page
- **The StrangeBee APT repository is deprecated** — install via direct `.deb` download only
- **Cortex 401 on connector test** means Cortex initialization is broken — wipe the index and reinitialize through the UI properly

---

## Next Step

With TheHive and Cortex deployed and connected, Shuffle is deployed next to orchestrate the full automation pipeline.

→ Continue in [`shuffle-setup.md`](shuffle-setup.md)
