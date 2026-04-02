🛡️ SIEM Detection Lab
![Security Monitoring Dashboard](screenshots/security-dashboard.png)

Tools I used: Splunk Enterprise · Ubuntu Server 24.04 LTS · VirtualBox · Windows PowerShell
Skills Demonstrated: Log Injestion/Understanding · Attack simulation · SPL querying · Dashboard visualization · Network configuration

Overview
I build a home lab SIEM environment meant to simulate 3 real-world attack scenarios and practice threat detection. I ran Splunk on a Windows host that ingests authentication logs forwarded from an Ubuntu Server Virtual Machine via the Splunk Universal Forwarder. The lab simulates brute force SSH attacks, successful logins, and privilege escalation.

Structure of the Lab
Windows Desktop (Host)
        │
   VirtualBox
        │
   Ubuntu Server VM  ──►  generates auth logs (/var/log/auth.log, /var/log/syslog)
        │
   Splunk Universal Forwarder  ──►  ships logs via TCP 9997
        │
   Splunk Enterprise (Windows)  ──►  ingests, indexes, and analyzes logs
        │
   Security Monitoring Dashboard  ──►  visualizes attack patterns

Lab Setup: Personally I like to setup my labs into phases breaking the lab down into a few time managable sections.

Phase 1: Get Splunk Enterprise on Windows

Downloaded and installed Splunk Enterprise on the Windows host
Accessed the web interface at http://localhost:8000
Configured receiving port 9997 for Universal Forwarder traffic

Phase 2: Ubuntu Server VM (VirtualBox)
SettingValueNameUbuntu-LogServerOSUbuntu Server 24.04 LTSRAM4096 MBCPU2 coresDisk25 GB dynamicAdapter 1NAT (internet access)Adapter 2Host-Only (lab traffic — 192.168.56.10)
Additional package installed during setup: OpenSSH Server

Phase 3: Splunk Universal Forwarder on Ubuntu
Transferred the .deb package from Windows to the VM via scp over the Host-Only adapter, then installed and configured:
bashsudo dpkg -i splunkforwarder-10.2.1-*.deb
sudo /opt/splunkforwarder/bin/splunk start --accept-license

# Point forwarder at Splunk server
sudo /opt/splunkforwarder/bin/splunk add forward-server 192.168.56.1:9997

# Monitor authentication and system logs
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/auth.log
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/syslog

Attack Simulations
Attack 1: SSH Brute Force
Ran a PowerShell loop from the Windows host attempting 20 SSH connections as a non-existent user:
powershellfor ($i=1; $i -le 20; $i++) {
    ssh fakeuser@192.168.56.10
}
Result: 24 Failed password events captured in Splunk from source IP 192.168.56.1.

Attack 2: Successful SSH Login
SSH'd into the Ubuntu VM using working credentials to show a contrasting Accepted password event.
Result: 1 Accepted password event confirmed in Splunk. I wanted to demonstrates normal v.s. malicious authentication pattern.

Attack 3: Privilege Escalation via Sudo
Ran a sensitive command requiring root privileges inside the VM:
bashsudo cat /etc/shadow
Result: sudo session open/close events logged to auth.log and captured in Splunk with full command context (COMMAND=/usr/bin/cat /var/log/auth.log).

Splunk Detection Queries (SPL)
See queries/splunk-searches.md for the full documented query set.
Query PurposeSPLAll indexed eventsindex=*Auth log eventsindex=main source="/var/log/auth.log"Failed login attemptsindex=main source="/var/log/auth.log" "Failed password"Successful loginsindex=main source="/var/log/auth.log" "Accepted password"Sudo escalation eventsindex=main source="/var/log/auth.log" "sudo"

Security Monitoring Dashboard
I mbuilt a 3-panel dashboard in Splunk named "Security Monitoring Dashboard":
PanelQueryFailed Login Attempts Over Timeindex=main "Failed password" | timechart count by hostTop Source IPs of Failed Loginsindex=main "Failed password" | rex "from (?P<src_ip>\d+\.\d+\.\d+\.\d+)" | top src_ipSuccessful vs Failed Loginsindex=main ("Failed password" OR "Accepted password") | eval status=if(searchmatch("Failed"),"Failed","Success") | timechart count by status
The dashboard clearly shows the spike in failed logins from 192.168.56.1 during the brute force simulation, contrasted against the single successful login event.

Screenshots
[Splunk first login / home screen](screenshots/splunk-dashboard.png) 
[Ubuntu VM network — Host-Only adapter at 192.168.56.10](screenshots/ubuntu-ip-addr.png)
[3,901 events indexed — logs confirmed flowing into Splunk](screenshots/logs-flowing.png)
[Auth log events visible in Splunk Search](screenshots/auth-log-ingestion.png)
[24 Failed password events from brute force simulation](screenshots/splunk-brute-force.png)
[PowerShell brute force loop running](screenshots/powershell-brute-force.png)
[Accepted password event — successful SSH login detected](screenshots/successful-login.png)
sudo cat /etc/shadow privilege escalation captured sudo-escalation.png)
[Final Security Monitoring Dashboard with all 3 panels](screenshots/security-dashboard.png)

Key Findings

3,901 total events ingested across auth.log and syslog sources
24 failed login attempts detected from a single source IP (192.168.56.1) within a 1-minute window — consistent with automated brute force behavior
1 successful login recorded immediately after, illustrating how an attacker could succeed after repeated attempts
Privilege escalation logged with full command context, including working directory, TTY, and the exact command executed
The dashboard's "Top Source IPs" panel correctly identified the attacker IP with the highest failed login count[README.md](https://github.com/user-
