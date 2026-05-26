# BlueSky Ransomware Investigation

**Lab:** [CyberDefenders — BlueSky Ransomware](https://cyberdefenders.org/blueteam-ctf-challenges/bluesky-ransomware/)  
**Analyst:** Myname  
**Date:** 28 April 2024  
**Severity:** High  
**Verdict:** True Positive

---

## Background

A high-profile corporation managing critical data across diverse industries reported a significant security incident. Their network was impacted by a suspected ransomware attack. Key files were encrypted, causing disruptions and raising concerns about potential data compromise. Early signs point to a sophisticated threat actor.

---

## Artifact Sources

- PCAP (network capture)
- Windows Event Log (.evtx)
- Ransomware sample (PE executable)

## Tools Used

- Wireshark — network traffic analysis
- CyberChef — payload decoding (UTF-16 LE, Base64)
- VirusTotal — malware hash analysis and behavioral sandbox review
- Event Viewer — Windows event log analysis

---

## Timeline

> Timestamp discrepancy observed between sources. PCAP dated 28/04/2024, Event Log dated 23/04/2024. Likely due to lab environment setup on separate dates. Event sequence used for timeline rather than absolute timestamps.

```
00:29:56  Port scan begins — SYN scan from 87.96.21.84
00:30:13  MSSQL Server login successful (sa account)
00:30:13  xp_cmdshell enabled
00:30:xx  Payload dropped — LkUYP.exe, Gjmwb.vbs written to %TEMP%
xx:xx:xx  C2 established via winlogon.exe injection (Event Log, time discrepancy noted)
00:32:10  checking.ps1 downloaded and executed (privilege check + Defender disabled)
00:32:10  Persistence established — scheduled task LPupdate created
00:32:12  Credential dumping — hashes saved to C:\ProgramData\hashes.txt
00:32:21  Ransomware deployed (BlueSky)
```

---

## IOCs

**Files**

- `LkUYP.exe` — dropped to %TEMP% via xp_cmdshell — hash unknown (not recoverable from PCAP)
- `Gjmwb.vbs` — dropper script, executes LkUYP.exe — hash unknown (not recoverable from PCAP)
- `hashes.txt` — dumped credentials, saved to `C:\ProgramData\hashes.txt`
- `javaw.exe.ransomware` — BlueSky ransomware binary  
  SHA-256: `3e035f2d7d30869ce53171ef5a0f761bfb9c14d94d9fe6da385e20b8d96dc2fb`
- `# DECRYPT FILES BLUESKY #.txt` — ransom note (source: VirusTotal behavioral analysis)
- `# DECRYPT FILES BLUESKY #.html` — ransom note (source: VirusTotal behavioral analysis)

**URLs**

- `http://87.96.21.84/checking.ps1`
- `http://87.96.21.84/del.ps1`
- `http://87.96.21.84/Invoke-PowerDump.ps1`
- `http://87.96.21.84/extracted_hosts.txt`
- `http://87.96.21.84/javaw.exe` (inferred via SMB)

**C2 IP**

- `87.96.21.84` — Warsaw, Poland — Orange Mobile (AS5617, TPNET)  
  Mobile carrier IP range. Unusual for C2 infrastructure — possible VPN/proxy or mobile tethering.

**Malware Family**

- BlueSky ransomware

**Process**

- `winlogon.exe` — C2 injection target

**Registry Keys**

- `HKLM:\SOFTWARE\Microsoft\Windows Defender\` — DisableAntiSpyware, DisableRoutinelyTakingAction, DisableRealtimeMonitoring, SubmitSamplesConsent, SpynetReporting

---

## Findings

### 1. Reconnaissance (TA0043) — T1595 Active Scanning

Port scan detected from `87.96.21.84` via rapid SYN packets to multiple destination ports with no corresponding ACK.

![Fig 1: SYN scan showing port scanning activity](screenshots/fig1-syn-scan.png)

### 2. Initial Access (TA0001) — T1190 Exploit Public-Facing Application

Successful MSSQL Server authentication observed via TDS packet. Credentials `sa`/`cyb3rd3f3nd3r$` transmitted in plaintext. Method of credential acquisition unknown — brute force, credential stuffing, or prior knowledge cannot be confirmed from available artifacts.

![Fig 2: TDS login packet showing plaintext credentials](screenshots/fig2-tds-login.png)

### 3. Execution (TA0002)

**T1059.003 — Windows Command Shell**  
`xp_cmdshell` enabled via `sp_configure`, spawning cmd.exe to write dropper files `LkUYP.exe` and `Gjmwb.vbs` to `%TEMP%`.

![Fig 3: xp_cmdshell TDS packet](screenshots/fig3-xp-cmdshell.png)

![Fig 4: CyberChef decoded payload revealing dropper filenames](screenshots/fig4-cyberchef-decoded.png)

**T1059.001 — PowerShell**  
`checking.ps1`, `del.ps1`, and `Invoke-PowerDump.ps1` downloaded and executed post-compromise.

### 4. Privilege Escalation / C2 (TA0004 / TA0011) — T1055 Process Injection

Metasploit C2 injected into `winlogon.exe`, confirmed via Event ID 400 with `HostName=MSFConsole` and `HostApplication=winlogon.exe`. Specific injection technique not determinable from available artifacts.

![Fig 5: Event ID 400 showing MSFConsole and winlogon.exe](screenshots/fig5-event-id-400.png)

### 5. Defense Impairment (TA0112) — T1685 Disable or Modify Tools

Windows Defender disabled via `checking.ps1`. Registry keys set to `1` under `HKLM:\SOFTWARE\Microsoft\Windows Defender` and `WinDefend` service stopped and disabled.

![Fig 6: checking.ps1 HTTP stream](screenshots/fig6-checking-ps1.png)

### 6. Persistence (TA0003) — T1053.005 Scheduled Task

Scheduled task `\Microsoft\Windows\MUI\LPupdate` created via `schtasks.exe`, configured to run hourly with SYSTEM privileges executing `del.ps1` from `C:\ProgramData\`.

![Fig 7: schtasks command in HTTP stream](screenshots/fig7-schtasks.png)

### 7. Credential Access (TA0006) — T1003 OS Credential Dumping

`Invoke-PowerDump.ps1` downloaded and executed via Base64 encoded command. Password hashes saved to `C:\ProgramData\hashes.txt`. Specific dumping method not confirmed from available artifacts.

![Fig 8: CyberChef decoded Base64 command](screenshots/fig8-base64-decoded.png)

### 8. Lateral Movement (TA0008) — T1021.002 SMB/Windows Admin Shares

Credentials from `hashes.txt` used with `Invoke-SMBExec` against hosts from `extracted_hosts.txt` to spread ransomware across the network.

### 9. Impact (TA0040) — T1486 Data Encrypted for Impact

BlueSky ransomware (`javaw.exe.ransomware`) deployed. Files encrypted with `.bluesky` extension appended. Ransom notes dropped on victim machines.

![Fig 9: VirusTotal detection — 62/71 vendors, BlueSky family confirmed](screenshots/fig9-virustotal-detection.png)

![Fig 10: VirusTotal files dropped — ransom notes identified](screenshots/fig10-virustotal-files-dropped.png)

---

## Limitations

- Ransomware delivery via SMB inferred from script analysis, not directly observed in PCAP
- Credential acquisition method unconfirmed

---

## Actions Taken

- Block `87.96.21.84` at firewall/perimeter
- Isolate affected hosts from network
- Reset `sa` account credentials, disable if not required for operations
- Disable `xp_cmdshell` on MSSQL Server
- Implement Event ID 15457 alerting for future `xp_cmdshell` enable attempts
- Remove scheduled task `\Microsoft\Windows\MUI\LPupdate`
- Sweep all endpoints for `# DECRYPT FILES BLUESKY #` files to identify blast radius
- Reset all credentials recovered from `hashes.txt`
- Restore encrypted files from clean backup

---

## Escalation Reason

Confirmed ransomware deployment across network via SMB lateral movement. Multiple hosts potentially affected. Credential dumping indicates full domain compromise possible. Requires incident response team engagement and management notification.
