# Splunk SPL Queries — SIEM Detection Lab

All queries used during the SIEM Detection Lab project, documented for reference.

---

## Basic Log Verification

```spl
index=*
```
Confirms all indexed data. Returned 3,901 events across `auth.log` and `syslog` sources.

```spl
index=main source="/var/log/auth.log"
```
Filters to authentication log events only from the Ubuntu VM.

---

## Attack Detection Queries

### Failed Login Attempts (Brute Force)
```spl
index=main source="/var/log/auth.log" "Failed password"
```
Returns all failed SSH login events. Detected 24 events during the brute force simulation, all originating from `192.168.56.1`.

### Successful Logins
```spl
index=main source="/var/log/auth.log" "Accepted password"
```
Returns successful SSH authentications. Used to contrast against failed attempts.

### Sudo / Privilege Escalation
```spl
index=main source="/var/log/auth.log" "sudo"
```
Captures all sudo session events including the command executed, working directory, TTY, and user context.

---

## Dashboard Panel Queries

### Panel 1 — Failed Login Attempts Over Time
```spl
index=main "Failed password" | timechart count by host
```
Timechart showing volume of failed logins per host over time. Shows a clear spike during brute force simulation.

### Panel 2 — Top Source IPs of Failed Logins
```spl
index=main "Failed password" | rex "from (?P<src_ip>\d+\.\d+\.\d+\.\d+)" | top src_ip
```
Extracts the source IP from the raw log text using regex and ranks IPs by failed login count. `192.168.56.1` (Windows host) ranked #1.

### Panel 3 — Successful vs Failed Logins (Comparison)
```spl
index=main ("Failed password" OR "Accepted password") 
| eval status=if(searchmatch("Failed"),"Failed","Success") 
| timechart count by status
```
Side-by-side timechart of failed vs. successful logins. Clearly shows the brute force spike alongside the single successful login event.

---

## Notes on SPL Concepts Used

| Concept | What it does |
|---|---|
| `index=main` | Scopes search to the main index |
| `source="/var/log/auth.log"` | Filters to a specific log file source |
| `timechart count by host` | Groups event counts over time, split by host |
| `rex "from (?P<src_ip>...)"` | Extracts a named field from raw event text using regex |
| `top src_ip` | Returns the most frequent values of a field |
| `eval status=if(...)` | Creates a new field based on a conditional expression |
| `searchmatch("Failed")` | Returns true if the raw event contains the string |
