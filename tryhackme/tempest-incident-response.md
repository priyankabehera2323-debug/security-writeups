# TryHackMe: Tempest — Incident Response

**Category:** Digital Forensics & Incident Response (DFIR) | Threat Hunting | SOC Analysis
**Room link:** https://tryhackme.com/room/tempestincident

## Skills Demonstrated
- Sysmon log analysis (Process Creation, Network Connection, File Creation, Registry events)
- Endpoint and network log correlation
- Packet capture analysis (Wireshark, Brim)
- Base64 decoding of attacker command chains
- C2 traffic identification and analysis
- Reverse SOCKS proxy / pivoting technique recognition
- Privilege escalation analysis (token/privilege abuse)
- Windows Event Log analysis for account creation and group membership changes
- Full attack chain reconstruction, initial access through domain-level persistence

## Scenario Overview
Acting as an Incident Responder, I investigated a CRITICAL-severity alert triaged by a SOC analyst. The intrusion began with a malicious document delivered via Chrome download, which executed a chain of commands leading to full SYSTEM-level compromise and persistent administrative access.

## Methodology
Investigation relied on three correlated data sources:
- **Sysmon logs** (parsed via EvtxECmd → Timeline Explorer, visualized in SysmonView)
- **Windows Event Logs** (Security log events for account/group changes)
- **Packet capture** (Wireshark / Brim, filtered by HTTP path and response port)

Evidence integrity was verified via SHA256 hashing of all provided artifacts before analysis began.

## Attack Chain Reconstruction

### 1. Initial Access
A malicious `.doc` file was downloaded via `chrome.exe` and opened in Microsoft Word. Sysmon Process Creation (Event ID 1) and DNS Query (Event ID 22) events, filtered by WinWord.exe's child processes, revealed the document executed a base64-encoded command chain — decoding it exposed the full attacker command and the exploited vulnerability (a known Office remote code execution CVE).

### 2. Stage 2 Payload & Persistence
The decoded command dropped a file to disk and configured it to execute on user logon (Autostart persistence, parented by `explorer.exe`). This stage 2 binary was downloaded from a remote host and established a connection to a C2 server over a specific domain and port.

### 3. Network Correlation
Cross-referencing Sysmon's endpoint findings against the packet capture (Brim/Wireshark) confirmed the C2 channel: HTTP requests carrying encoded commands and command output via a specific request parameter, using a distinct HTTP method and a User-Agent string that revealed the compiled language of the C2 binary.

### 4. Internal Reconnaissance & Lateral Movement
Decoded C2 traffic revealed the attacker enumerating the filesystem and discovering a sensitive file containing stored credentials, followed by enumeration of listening ports to identify a viable remote shell path. The attacker then deployed a reverse SOCKS proxy tool (identified via file hash against known tooling) to pivot into internal-only services, subsequently authenticating to an internal service using the harvested credentials.

### 5. Privilege Escalation
After enumerating current user privileges, the attacker downloaded a known Windows privilege escalation tool (identified via hash lookup), which abused a specific Windows privilege commonly exploited by "Potato-family" escalation techniques to obtain SYSTEM-level access. A second C2 channel was established on a different port, separating low-privilege and high-privilege attacker sessions.

### 6. Domain Persistence & Privilege Consolidation
With SYSTEM access, the attacker attempted to create new local user accounts — an initial attempt failed due to a missing required parameter, visible in the logs as a useful example of imperfect attacker execution. Windows Security Event ID **4720** confirmed successful account creation. The attacker then added the new account to the local Administrators group, confirmed via Event ID **4732**. A final technique was executed to cement long-term, persistent administrative access.

## Full Kill Chain Summary
Phishing document (WinWord.exe) 

    → Base64 payload → dropped file + logon persistence
    → Stage 2 binary downloaded → C2 established
    → Recon via C2 → sensitive file found → credentials harvested
    → Port enumeration → remote shell path identified
    → Reverse SOCKS proxy → internal pivoting
    → Harvested credentials → internal authentication
    → Privilege check → privesc tool → SYSTEM access
    → New admin account created (Event ID 4720)
    → Account added to Administrators group (Event ID 4732)
    → Final persistence mechanism established

## Key Takeaways
- No single log source tells the full story — Sysmon, Windows Event Logs, and network captures each confirmed and extended findings from the others.
- Attacker tradecraft included realistic imperfections (a failed account creation attempt), reinforcing that failed commands are still valuable forensic evidence.
- Reverse SOCKS proxying for internal pivoting is a technique that goes beyond typical single-host compromise scenarios and is worth specifically watching for in real environments.
- Specific Windows Security Event IDs (4720, 4732) are high-value detection points for account creation and privilege group changes.

## Detection & Defensive Recommendations
- Alert on Office processes (WinWord.exe, EXCEL.exe) spawning child processes such as `cmd.exe` or `powershell.exe`.
- Monitor for base64-encoded PowerShell command lines (`-enc` / `-EncodedCommand` flags).
- Flag outbound connections from user-context processes to non-standard external domains/ports.
- Monitor for local account creation (Event ID 4720) and additions to privileged groups (Event ID 4732), especially outside change-management windows.
- Watch for privilege escalation tooling signatures associated with token impersonation privileges.

## Reflection
This was my most comprehensive DFIR investigation to date, covering the full attack lifecycle from phishing to domain-level persistence. It reinforced that effective incident response depends on correlating multiple log sources rather than relying on any single artifact, and gave me hands-on exposure to real-world attacker techniques like C2 pivoting and privilege escalation tooling.
