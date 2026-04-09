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

With agents enrolled and active, the final step was to confirm that Sysmon event data from `DC01` was flowing through to the Wazuh dashboard.

### Configuring the Agent to Collect Sysmon Logs

By default, the Wazuh agent does not collect from the Sysmon event channel. The agent configuration file was updated on `DC01` to explicitly include the Sysmon operational log as a monitored source.

The following block was added to `ossec.conf` on `DC01`:

```xml

  Microsoft-Windows-Sysmon/Operational
  eventchannel

```

![ossec.conf updated with Sysmon localfile entry]

<img width="615" height="517" alt="image" src="https://github.com/user-attachments/assets/19d3679b-2ed2-4c1a-9739-3621f4ffcff8" />
(screenshots/DC01-new-localfile-added.png)

After saving the configuration, the Wazuh agent service was restarted to apply the changes:

```powershell
Restart-Service WazuhSvc
Get-Service WazuhSvc
```

Output confirmed:

```
Status   Name       DisplayName
------   ----       -----------
Running  WazuhSvc   Wazuh
```

![Wazuh agent service restarted on DC01]

<img width="629" height="132" alt="image" src="https://github.com/user-attachments/assets/4a643f8f-d2e9-425f-bcde-380aa336d391" />
(screenshots/DC01-wazuh-service-restarted.png)

---

### Generating a Test Event

To verify that Sysmon logs were being ingested, a simulated lateral movement attempt was executed on `DC01` using a failed network authentication command:

```powershell
net use \\DC01 /user:fakeuser wrongpassword
```

This intentionally triggered a failed logon attempt and a process creation event captured by Sysmon Event ID 1.

Output on `DC01`:

```
System error 1326 has occurred.
The user name or password is incorrect.
```

![Test command run on DC01]

<img width="675" height="145" alt="image" src="https://github.com/user-attachments/assets/55e25faa-a0cb-467d-81b6-dd265d38dcc9" />
(screenshots/DC01-command-ran-to-generate-log.png)

---

### Sysmon Event Visible in Wazuh

The event was successfully captured by Sysmon, shipped by the Wazuh agent, and appeared in the Wazuh dashboard with full process metadata.

**Key fields from the ingested alert:**

| Field | Value |
|---|---|
| `agent.name` | `DC-01` |
| `agent.ip` | `192.168.0.10` |
| `manager.name` | `wazuh01` |
| `data.win.system.eventID` | `1` (Process Create) |
| `data.win.system.channel` | `Microsoft-Windows-Sysmon/Operational` |
| `data.win.eventdata.image` | `C:\Windows\System32\net.exe` |
| `data.win.eventdata.commandLine` | `net.exe use \\DC01 /user:fakeuser wrongpassword` |
| `data.win.eventdata.parentImage` | `powershell.exe` |
| `data.win.eventdata.user` | `LAB\Administrator` |
| `rule.id` | `92037` |
| `rule.description` | A net.exe connection to a remote resource was started by powershell.exe |
| `rule.mitre.id` | `T1567` |
| `rule.mitre.tactic` | Exfiltration |
| `rule.level` | `3` |

![Sysmon log ingested and visible in Wazuh dashboard]

<img width="1474" height="511" alt="image" src="https://github.com/user-attachments/assets/a944025b-a2d7-4f64-ae71-57fce8347fb4" />
(screenshots/DC01-sysmon-log-in-wazuh.png)

---

## Conclusion

All three endpoints — `DC01`, `WS01`, and `INFRA01` — were successfully enrolled into the Wazuh manager as active agents. Sysmon log ingestion was confirmed on `DC01` by generating a simulated authentication event and verifying the resulting alert appeared in the Wazuh dashboard with full process telemetry and MITRE ATT&CK mapping.

The Wazuh platform is now collecting enriched endpoint telemetry across the lab environment. Sysmon ingestion will next be verified on `WS01` before moving into active threat simulation.

**Next:** Verify Sysmon log ingestion on `WS01`, then proceed to Kali Linux VM setup and first real alert generation.
