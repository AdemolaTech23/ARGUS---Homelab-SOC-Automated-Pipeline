# Sysmon Deployment — Windows Endpoints

> Part of the [Home SOC Lab](../README.md) project.  
> **Phase 1.5** — Completed after Wazuh server installation, before agent enrollment.

---

## Objective

Deploy Sysmon on all Windows endpoints in the lab environment to enable detailed host-level telemetry. This enriched event data will feed into Wazuh for detection and analysis.

Sysmon is deployed at this stage — after the Wazuh server is operational but before agents are enrolled — so that the Wazuh agent picks up enriched Sysmon logs from the moment it is installed.

---

## Sysmon Overview

Sysmon (System Monitor) is a Windows system service and device driver from Microsoft Sysinternals that logs detailed system activity to the Windows Event Log. It provides visibility into:

- Process creation and termination
- Network connections
- File creation
- Registry changes
- DNS queries

Standard Windows logging does not capture this level of detail by default. Sysmon fills that gap and is a foundational component of most enterprise and SOC monitoring stacks.

---

## Endpoints

| Hostname | Role | OS | IP Address |
|---|---|---|---|
| `DC01` | Domain Controller | Windows Server 2022 | `192.168.0.10` |
| `WS01` | Workstation | Windows 10 Pro | `192.168.0.102` |

Both machines are on the `192.168.0.0/24` subnet with a default gateway of `192.168.0.1`.

![DC01 IP address confirmation]<img width="1011" height="632" alt="image" src="https://github.com/user-attachments/assets/d90134bb-85d0-4d8a-8fcc-ad9b7dcb3937" />


---

## Installation (DC01 + WS01)

Sysmon v15.20 was downloaded and extracted on both endpoints. The Sysmon directory contained the following files after extraction:

- `sysmon-config-master/` — configuration source folder
- `Eula.txt`
- `Sysmon.exe` — 32-bit binary
- `Sysmon64.exe` — 64-bit binary
- `Sysmon64a.exe` — ARM64 binary
- `sysmonconfig-export.xml` — active configuration file applied at install

---

## Configuration (SwiftOnSecurity)

The configuration applied during installation was based on the SwiftOnSecurity Sysmon config — a community-maintained, production-grade ruleset that provides broad detection coverage while filtering out high-volume noise events.

The config was pre-exported as `sysmonconfig-export.xml` and passed directly to the installer via the `-i` flag.

**Command run on both endpoints:**
```cmd
cd Downloads\Sysmon
.\Sysmon64.exe -accepteula -i C:\Users\<user>\Downloads\Sysmon\sysmonconfig-export.xml
```

On `WS01` specifically:
```powershell
PS C:\Users\jasmine.rodgers\Downloads\Sysmon> .\Sysmon64.exe -accepteula -i C:\Users\jasmine.rodgers\Downloads\Sysmon\sysmonconfig-export.xml
```

Installation output confirmed on both machines:
```
System Monitor v15.20 - System activity monitor
By Mark Russinovich and Thomas Garnier
Copyright (C) 2014-2026 Microsoft Corporation

Loading configuration file with schema version 4.50
Sysmon schema version: 4.91
Configuration file validated.
Sysmon64 installed.
SysmonDrv installed.
Starting SysmonDrv.
SysmonDrv started.
Starting Sysmon64..
Sysmon64 started.
```

![WS01 Sysmon installation output]

<img width="1012" height="723" alt="image" src="https://github.com/user-attachments/assets/997d74a6-5dbd-4f9a-8d38-32dad3e5416c" />

![DC01 Sysmon installation output]

<img width="837" height="734" alt="image" src="https://github.com/user-attachments/assets/c7a15a67-0bbe-4f10-b86e-81f82b4b98a8" />


---

## Service Verification

After installation, the Sysmon64 service status was confirmed on both endpoints:
```powershell
Get-Service Sysmon64
```

**Result on both machines:**
```
Status   Name       DisplayName
------   ----       -----------
Running  Sysmon64   Sysmon64
```

`Sysmon64` confirmed in a `Running` state on both `DC01` and `WS01`.

---

## Log Verification

To confirm Sysmon was actively writing events to the Windows Event Log, the five most recent Sysmon events were queried on each machine:
```powershell
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5
```

**WS01 output:**
```
ProviderName: Microsoft-Windows-Sysmon

TimeCreated                Id  LevelDisplayName  Message
-----------                --  ----------------  -------
4/7/2026 5:56:42 PM         1  Information       Process Create:...
4/7/2026 5:56:34 PM        22  Information       Dns query:...
4/7/2026 5:56:33 PM         1  Information       Process Create:...
4/7/2026 5:56:32 PM         1  Information       Process Create:...
4/7/2026 5:56:26 PM        11  Information       File created:...
```

**DC01 output:**
```
ProviderName: Microsoft-Windows-Sysmon

TimeCreated                Id  LevelDisplayName  Message
-----------                --  ----------------  -------
4/7/2026 5:50:50 PM         1  Information       Process Create:...
4/7/2026 5:49:54 PM        11  Information       File created:...
4/7/2026 5:49:54 PM         1  Information       Process Create:...
4/7/2026 5:49:54 PM         1  Information       Process Create:...
4/7/2026 5:40:01 PM        22  Information       Dns query:...
```

Both endpoints were generating Sysmon events across multiple event IDs, confirming the driver and configuration were active.

| Event ID | Description |
|---|---|
| `1` | Process Create |
| `11` | File Created |
| `22` | DNS Query |

![WS01 Sysmon log verification]

<img width="736" height="422" alt="image" src="https://github.com/user-attachments/assets/aa0b8e62-07bf-4a37-b1c9-bcaed1cefa33" />

![DC01 Sysmon log verification]

<img width="887" height="463" alt="image" src="https://github.com/user-attachments/assets/4c1993a1-affa-4100-8b7f-e8bc1ec3c695" />

---

## Conclusion

Sysmon v15.20 was successfully deployed and verified on both Windows endpoints in the lab environment.

Both `DC01` and `WS01` are now:
- running the `Sysmon64` service
- logging events to the `Microsoft-Windows-Sysmon/Operational` channel
- capturing process creation, file activity, and DNS queries in real time

With host-level telemetry confirmed on both machines, the environment is ready for the next phase.

**Next:** [Wazuh Agent Enrollment](../wazuh-agent-deployment/wazuh-agent-deployment.md) — enroll `DC01` and `WS01` into the Wazuh manager and confirm log ingestion from both endpoints.
