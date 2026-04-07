# ARGUS — Home SOC Automated Pipeline

A fully orchestrated SOC environment built on Proxmox, integrating Wazuh, 
Splunk, Elastic, TheHive, Shuffle, and MISP to simulate enterprise threat 
detection and automated incident response.

## Lab Environment

| System    | Role                          | IP Address     |
|-----------|-------------------------------|----------------|
| DC01      | Active Directory + DNS        | 192.168.0.10   |
| WS01      | Workstation (Windows 10 Pro)  | 192.168.0.102  |
| INFRA01   | Ticketing / osTicket          | 192.168.0.50   |
| WAZUH01   | SIEM (Wazuh)                  | 192.168.0.60   |

## Documentation

- [Wazuh Deployment](docs/wazuh-deployment.md)
- [Wazuh Agent Deployment](docs/wazuh-agent-deployment.md)
- [Sysmon Deployment — Windows](docs/sysmon-deployment-windows.md)
- [Elastic Deployment](docs/elastic-deployment.md)
- [Splunk Deployment](docs/splunk-deployment.md)
- [TheHive Setup](docs/thehive-setup.md)
- [Shuffle Setup](docs/shuffle-setup.md)
- [MISP Setup](docs/misp-setup.md)
