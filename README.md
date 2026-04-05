# SOC-Incident-Analysis-Lab
Home SOC lab simulating cyber attacks with Splunk, Kali Linux, and Windows. MITRE ATT&CK mapping included.

# SOC Incident Analysis Lab

## Overview
A complete home lab simulating a Security Operations Center (SOC) environment for L1 monitoring and incident triage practice.

## Technologies Used
| Tool | Purpose |
|------|---------|
| Oracle VirtualBox | Hypervisor for VMs |
| Kali Linux | Attacker machine |
| Windows 10 | Target machine |
| Splunk Enterprise | SIEM for log collection |
| Sysmon | Endpoint logging |
| Nmap | Network reconnaissance |
| Hydra | Brute force attacks |
| Metasploit | Reverse shell payloads |

## Lab Architecture
![Architecture](images/architecture.png)

## Attacks Simulated & MITRE Mapping

| Attack | MITRE ID | Tactic | Detection Method |
|--------|----------|--------|------------------|
| Nmap Port Scan | T1046 | Discovery (TA0007) | Event ID 5156 - 47+ ports in 60s |
| RDP Brute Force | T1110 | Credential Access (TA0006) | Event ID 4625 - 10+ failures/5min |
| Reverse Shell | T1573 | Command & Control (TA0011) | Sysmon Event ID 1 & 3 |

## Splunk Detection Queries

### Port Scan Detection
```spl
index=main EventCode=5156
| bin _time span=1m
| stats dc(Destination_Port) as ports by Source_IP, _time
| where ports > 10
.....,............
