# SOC Lab: Windows Attack Detection & Security Monitoring with Splunk

## Overview

This lab simulates a real-world security monitoring environment where a **Kali Linux** attacker machine targets a **Windows 2022** victim machine. All logs are forwarded to **Splunk Enterprise** for centralized monitoring, detection, and alerting. The goal is to detect common attack techniques like port scanning (T1046) and RDP brute-forcing (T1110) using Sysmon and Windows Event Logs.

---

## Technologies Used

| Tool | Purpose |
|------|---------|
| Oracle VirtualBox | Hypervisor for all VMs |
| Kali Linux | Attacker machine (pre-installed tools) |
| Windows 2022 | Target / victim machine |
| Splunk Enterprise | SIEM for log ingestion, searching, and alerting |
| Splunk Universal Forwarder | Lightweight agent on Windows to forward logs |
| Sysmon | Advanced endpoint logging (process, network, file changes) |
| Nmap | Network discovery & port scanning |
| Hydra | Password brute-forcing (RDP, SSH, etc.) |


## Prerequisites

- **Host machine** with at least 16 GB RAM and 50 GB free disk space
- **Oracle VirtualBox** (latest version) with Extension Pack
- ISO images:
  - [Kali Linux](https://www.kali.org/get-kali/)
  - [Windows 10](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022)
  - [splunkUniversal Forwarder](https://www.splunk.com/en_us/download/universal-forwarder.html)


### 1. Virtual Machine Configuration

| VM | vCPUs | RAM | Storage | Network |
|----|-------|-----|---------|---------|
| Kali Linux | 2 | 2 GB | 20 GB | NAT Network |
| Windows 10 | 2 | 4 GB | 40 GB | NAT Network |

Create a **NAT Network** in VirtualBox so all VMs can communicate. 

After installing each OS,
FROM kali linux
```bash
ip a
```
notedown the ip address of kali linux

FROM Windows 2022
```cmd
ipconfig
```
notedown the ip address of windows 2022

verify connectivity with `ping 10.0.*.*` from Windows.


### 2. Install Splunk Enterprise (windows 2022 VM machine)

-[splunk Enterprise](https://www.splunk.com/en_us/products/splunk-enterprise.html)

Access the Splunk web interface at **http://10.0.*.*:8000**



### 3. Install Splunk Universal Forwarder IN (Windows 2022)

   Forward Logs to Splunk Enterprise

### Step 1: Create Configuration File

Create a new text file and rename it to `inputs.conf`

### Step 2: Add Configuration

Open `inputs.conf` in Notepad and paste the following configuration:

```ini
[WinEventLog://Application]
index = endpoint
disabled = false

[WinEventLog://Security]
index = endpoint
disabled = false

[WinEventLog://System]
index = endpoint
disabled = false

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = endpoint
disabled = false
renderXml = true
```

Restart the forwarder service:

```powershell
net stop SplunkForwarder
net start SplunkForwarder
```
      

->Step 6: Verify Logs in Splunk Enterprise

Search the following in Splunk Enterprise:
```init
index=endpoint
```
You should now see:

Application logs
Security logs
System logs
Sysmon logs


### 4. Install Sysmon (Windows 10)

Download [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) from Microsoft Sysinternals. Run as Administrator:

```cmd
sysmon64 -accepteula -i sysmonconfig.xml
```


> Use a community config such as [SwiftOnSecurity's sysmonconfig](https://github.com/SwiftOnSecurity/sysmon-config) for comprehensive coverage.

Sysmon events will appear under `Applications and Services Logs → Microsoft → Windows → Sysmon → Operational` and are automatically picked up by the Universal Forwarder.




## Attacks Simulated & MITRE ATT&CK Mapping

| Attack | MITRE ID | Tactic | Detection Method |
|--------|----------|--------|-----------------|
| Nmap Port Scan | T1046 | Discovery (TA0007) | Event ID 5156 — 10+ distinct destination ports from one source in 1 minute |
| RDP Brute Force | T1110 | Credential Access (TA0006) | Event ID 4625 — 10+ failed logins in 5 minutes via RDP |

---


## Running the Attacks (from Kali Linux)

open bash
```bash
sudo apt update
```

### 1. Port Scan

```bash
nmap -sT 10.0.2.15    # TCP connect scan
nmap -sS 10.0.2.15    # SYN scan (requires root)
nmap -sF 10.0.2.15    # FIN scan (stealth)
```

### 2. RDP Brute Force
```bash
apt install hydra
```

```bash
hydra -l Administrator -P /usr/share/wordlists/rockyou.txt rdp://10.0.2.15
```

## Splunk Detection Queries

### Port Scan Detection (T1046)

```spl
index=endpoint EventCode=5156
| bin _time span=1m
| stats dc(Destination_Port) as distinct_ports by Source_IP, _time
| where distinct_ports > 10
| sort - distinct_ports
```

> **Expected result:** Kali IP (10.0.2.10) shows >10 distinct ports scanned within a single minute.

---

### RDP Brute Force Detection (T1110)

```spl
index=endpoint EventCode=4625 Logon_Type=10
| bin _time span=5m
| stats count by Source_Network_Address, _time
| where count > 10
| rename count as failed_attempts
```

> `Logon_Type=10` filters for Remote Interactive (RDP) logons only.

---



## Investigation Dashboard

Navigate to **Dashboards → Create New Dashboard** in Splunk and add the following panels:

| # | Query | Visualization |
|---|-------|---------------|
| 1 | `index=* EventCode=4625 LogonType=10 \| timechart count by Source_Network_Address` | Column Chart |
| 2 | `index=* EventCode=4625 \| stats count by Source_Network_Address \| sort - count` | Bar Chart |
| 3 | `index=* (4625 OR 4624) LogonType=10 \| timechart count span=1m by EventCode` | Line Chart |
| 4 | `index=* LogonType=10 (4625 OR 4624) \| eval result=if(EventCode=4624, "Success", "Failure") \| stats count by result` | Pie Chart |
| 5 | `index=* (4625 OR 4624) LogonType=10 \| table _time, Source_Network_Address, TargetUserName, EventCode \| head 20` | Table |
| 6 | `index=* sourcetype=pfSense action=block dst_port=3389 \| timechart count by src_ip` | Area Chart |

## Future Improvements

- Add **Zeek** for network monitoring
- Add **TheHive** for case management
- Add **Caldera** for attack emulation
- Set up **Slack alerts** from Splunk

## References

- [Splunk Search Reference](https://docs.splunk.com/Documentation/Splunk/latest/SearchReference)
- [Sysmon Documentation](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)

## License

This project is for **educational and research purposes only**.

---

*Happy Hunting! 🔍*
