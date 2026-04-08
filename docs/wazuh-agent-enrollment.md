# Wazuh Agent Enrollment — Windows/Linux Endpoints

> Part of the [ARGUS Home SOC Lab](../README.md) project.  
> **Phase 2** — Completed after Sysmon deployment on all Windows endpoints.

---

## Objective

Enroll all endpoints into the Wazuh manager so that host-level telemetry, including enriched Sysmon events, is collected, indexed, and visible through the Wazuh dashboard.

---

## Sysmon Integration Context

Prior to agent enrollment, Sysmon v15.20 was deployed on both `DC01` and `WS01` using the SwiftOnSecurity configuration. This ensures that from the moment the Wazuh agent connects, it immediately begins shipping enriched event data covering process creation, file activity, DNS queries, and network connections.

See: [Sysmon Deployment — Windows Endpoints](sysmon-deployment-windows.md)

---

## Endpoints

| Hostname | Role | OS | IP Address |
|---|---|---|---|
| `DC01` | Domain Controller | Windows Server 2022 | `192.168.0.10` |
| `WS01` | Workstation | Windows 10 Pro | `192.168.0.70` |
| `INFRA01` | Infrastructure Server | Ubuntu 24.04.4 LTS | `192.168.0.50` |

> **Note:** Prior to agent enrollment, `WS01` was reconfigured from a dynamically assigned IP to a static address. All lab endpoints use static addressing to ensure consistent identification within Wazuh and across the broader SOC pipeline.

### WS01 Static IP Configuration

| Setting | Value |
|---|---|
| IP Address | `192.168.0.70` |
| Subnet Mask | `255.255.255.0` |
| Default Gateway | `192.168.0.1` |
| Preferred DNS | `192.168.0.10` |

![WS01 static IP assignment]

<img width="406" height="454" alt="image" src="https://github.com/user-attachments/assets/b6eda738-f627-4869-a7ab-c1bb91ab07f7" />


---

## Installation Steps

Agent deployment was initiated from the Wazuh dashboard using the built-in deployment wizard under **Endpoints > Deploy new agent**. The process was repeated for each endpoint with the appropriate package and agent name selected.

---

### DC01 — Windows Server 2022

The following options were configured in the wizard:

- **Package:** MSI 32/64 bits (Windows)
- **Server address:** `192.168.0.60`
- **Agent name:** `DC-01`
- **Group:** Default

![DC01 agent deployment wizard]

<img width="1787" height="884" alt="image" src="https://github.com/user-attachments/assets/9456cfda-fc85-4545-aa50-240aaf34c504" />


The wizard generated the following PowerShell commands, run directly on `DC01`:

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.4-1.msi `
  -OutFile $env:tmp\wazuh-agent.msi
msiexec.exe /i $env:tmp\wazuh-agent /q WAZUH_MANAGER='192.168.0.60' WAZUH_AGENT_NAME='DC-01'

NET START Wazuh
```

Output confirmed:

```
The Wazuh service is starting.
The Wazuh service was started successfully.
```

![DC01 agent installation and service start]

<img width="1920" height="999" alt="image" src="https://github.com/user-attachments/assets/ba5955da-7d12-4cb5-8d61-704e8b5d5532" />


---

### WS01 — Windows 10 Pro

The following options were configured in the wizard:

- **Package:** MSI 32/64 bits (Windows)
- **Server address:** `192.168.0.60`
- **Agent name:** `WS-01`
- **Group:** Default

![WS01 agent deployment wizard]

<img width="1003" height="813" alt="image" src="https://github.com/user-attachments/assets/23ed9edf-085e-4a69-8fd0-75b59027abd3" />


Commands run on `WS01`:

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.4-1.msi `
  -OutFile $env:tmp\wazuh-agent.msi
msiexec.exe /i $env:tmp\wazuh-agent /q WAZUH_MANAGER='192.168.0.60' WAZUH_AGENT_NAME='WS-01'

NET START Wazuh
```

Output confirmed:

```
The Wazuh service is starting.
The Wazuh service was started successfully.
```

![WS01 agent installation and service start]

<img width="844" height="323" alt="image" src="https://github.com/user-attachments/assets/7aef6bd1-859d-4196-b469-bc63fd265957" />


---

### INFRA01 — Ubuntu 24.04.4 LTS

For the Linux endpoint, the wizard generated a `wget` based command using the `.deb` package. The following commands were run on `INFRA01`:

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.4-1_amd64.deb \
  && sudo WAZUH_MANAGER='192.168.0.60' WAZUH_AGENT_NAME='INFRA-01' \
  dpkg -i ./wazuh-agent_4.14.4-1_amd64.deb

sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

The package downloaded at 10.4 MB/s and was installed successfully. The agent service was enabled and started via `systemctl`.

![INFRA01 agent installation]

<img width="1005" height="677" alt="image" src="https://github.com/user-attachments/assets/ef946654-9bf0-4e29-8aba-0119178552d1" />


---

## Agent Registration

After all three agents were started, the Wazuh dashboard confirmed successful registration across all endpoints.

### DC01

| ID | Name | IP Address | Group | OS | Version | Status |
|---|---|---|---|---|---|---|
| 001 | DC-01 | 192.168.0.10 | default | Windows Server 2022 Standard Evaluation | v4.14.4 | ✅ Active |


### WS01

| ID | Name | IP Address | Group | OS | Version | Status |
|---|---|---|---|---|---|---|
| 002 | WS-01 | 192.168.0.70 | default | Windows 10 Pro | v4.14.4 | ✅ Active |


### INFRA01

| ID | Name | IP Address | Group | OS | Version | Status |
|---|---|---|---|---|---|---|
| 003 | INFRA-01 | 192.168.0.50 | default | Ubuntu 24.04.4 LTS | v4.14.4 | ✅ Active |


---

## Dashboard Verification

With all three agents enrolled and reporting, the Wazuh Endpoints summary confirmed the following:

- **Agents by status:** 3 Active, 0 Disconnected, 0 Pending
- **Top 5 OS:** Windows (2), Ubuntu (1)
- **Top 5 Groups:** default (3)

| ID | Name | IP Address | Group | OS | Version | Status |
|---|---|---|---|---|---|---|
| 001 | DC-01 | 192.168.0.10 | default | Windows Server 2022 Standard Evaluation | v4.14.4 | ✅ Active |
| 002 | WS-01 | 192.168.0.70 | default | Windows 10 Pro | v4.14.4 | ✅ Active |
| 003 | INFRA-01 | 192.168.0.50 | default | Ubuntu 24.04.4 LTS | v4.14.4 | ✅ Active |

![All 3 agents active in Wazuh dashboard]

<img width="1885" height="569" alt="image" src="https://github.com/user-attachments/assets/1785fcfa-b226-4646-a497-34b6ec912de4" />


---

## Sysmon Log Ingestion

---

## Conclusion
