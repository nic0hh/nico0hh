# Threat Intelligence Report: Play Ransomware (aka PlayCrypt)

**Date:** April 2026  
**Analyst:** Nicholaus
**Classification:** TLP:WHITE  
**Primary Source:** CISA/FBI Advisory AA23-352A (Updated June 2025)

---

## 1. Threat Actor Overview

Play, also known as PlayCrypt, has been active since June 2022 and was one of the most active ransomware groups in 2024. They are a presumably closed group, designed according to their own leak site to "guarantee the secrecy of deals." Play uses a double extortion model where they encrypt systems after exfiltrating data. The ransom note does not include a ransom demand or payment instructions, but instructs victims to contact the group via email. A portion of victims have also been contacted by telephone.

---

## 2. Targeted Sectors and Geography

Play has targeted a wide range of businesses and critical infrastructure across North America, South America, and Europe, with approximately 900 confirmed victims as of May 2025. The group appears to favour high-value targets over mass deployment campaigns.

---

## 3. Attack Lifecycle

Play gains initial access through stolen credentials purchased on the dark web, exploitation of public-facing applications including FortiGate VPN and Microsoft Exchange ProxyNotShell, and abuse of external-facing services such as RDP and VPN.

Once in the network they use AdFind to run Active Directory queries and with their own information stealer, Grixba, they enumerate network information and scan for AV software. Subsequently, unsecured credentials are identified and Mimikatz is used to dump said credentials and gain domain administrator access.

To avoid detection, Play will disable any found anti-virus software and remove log files to cover their tracks. They will specifically target Microsoft Defender with PowerShell scripts. Play uses Cobalt Strike and SystemBC for lateral movement, along with PsExec to move laterally via SMB/Windows Admin Shares and transfer tools across the network.

Play will then execute WinPEAS via PowerShell to enumerate privilege escalation paths and executables are distributed domain-wide through Group Policy Objects and lateral tool transfer.

Play will segment compromised data into compressed .RAR format using WinRAR and transfer it to attacker-controlled external accounts with WinSCP. Since Play recompiles its encryptor for every attack, the unique file hashes generated defeat signature-based AV detection. The group uses AES-RSA hybrid encryption with intermittent encryption processing, alternating 1MB chunks rather than entire files, thus accelerating deployment. Encrypted files are left with a .PLAY extension and a ReadMe.txt is dropped in C:/Users/Public/Music. Ransom demands are not included in the note itself — victims are instead directed to contact Play via email. The combination of intermittent encryption and unique per-attack binaries means traditional signature-based defenses are largely ineffective against Play before the payload detonates.

---

## 4. MITRE ATT&CK Technique Mapping

| Tactic | Technique ID | What Play Does |
|---|---|---|
| Initial Access | T1078 | Abuse of valid accounts likely purchased on the dark web |
| Initial Access | T1190 | Exploit public-facing applications (FortiGate VPN, Microsoft Exchange ProxyNotShell, SimpleHelp RMM) |
| Initial Access | T1133 | Abuse of external-facing services such as RDP and VPN |
| Discovery | T1016 | Grixba information-stealer used to enumerate network information |
| Discovery | T1518.001 | Scan for AV software |
| Defense Evasion | T1562.001 | GMER, IOBit, PowerTool used to disable AV software |
| Defense Evasion | T1070.001 | Log files removed to cover tracks |
| Defense Evasion | T1059.001 | PowerShell scripts used to target Microsoft Defender |
| Lateral Movement | T1021.002 | PsExec used to move laterally via SMB/Windows Admin Shares |
| Lateral Movement | T1570 | Transfer tools across network |
| Credential Access | T1552 | Unsecured credentials identified |
| Credential Access | T1003 | Mimikatz used to dump credentials and gain domain administrator access |
| Execution | T1059.001 | Remote code execution following SimpleHelp exploitation |
| Execution | T1059.001 | WinPEAS executed via PowerShell to identify privilege escalation paths* |
| Execution | T1484.001 | Executables distributed domain-wide via Group Policy Objects |
| Exfiltration | T1560.001 | Compromised data segmented and compressed into .RAR format using WinRAR |
| Exfiltration | T1048 | Data transferred via WinSCP to actor-controlled external accounts |
| Impact | T1027 | Play recompiles its encryptor for every attack, generating unique hashes that defeat signature-based detection |
| Impact | T1486 | Files encrypted using AES-RSA hybrid encryption with intermittent encryption, processing alternating 1MB chunks |
| Impact | T1657 | Double extortion: victims face both encryption and threatened publication of stolen data to a Tor leak site |

*CISA displays T1059 for WinPEAS but hyperlinks to T1059.001 (PowerShell). T1059.001 used here as the more precise mapping.

---

## 5. Detection Opportunities

The following detections are listed in order of kill chain stage. Later stage detections — particularly T1486 and T1490 — represent critical priority escalations indicating active or imminent ransomware deployment. Note: T1490 is not explicitly documented in the CISA advisory but has been included as shadow copy deletion is a near-universal ransomware indicator.

**T1078 — Valid Accounts**  
Monitor for authentication anomalies, including impossible travel (successful logins from geographically distant locations within a short timeframe), dormant accounts suddenly becoming active, and successful logins immediately followed by AD enumeration such as AdFind.

**T1562.001 — Impair Defenses**  
Where EDR is deployed, alert on unexpected termination of security software processes. Monitor for Windows registry modifications targeting security software service keys, specifically Event ID 4657. Loss of EDR visibility on a previously active host is itself a red flag and warrants immediate investigation.

**T1070.001 — Clear Windows Event Logs**  
Configure SIEM to alert on unexpected gaps in log ingestion from hosts. A drop in volume from expected active endpoints may indicate log tampering or deletion. This should be correlated with other suspicious activity on the same host.

**T1560.001 / T1048 — Archive via Utility / Exfiltration Over Alternative Protocol**  
Monitor for compression tool execution such as WinRAR on hosts where it is not standard software. Correlate with anomalous outbound data transfer via non-standard protocols. Any large volume transfer via SFTP/FTP to external IPs outside of regular hours requires immediate investigation. Since file compression and transfer can be normal, individual indicators may be benign — detection value lies in correlation across multiple data points.

**T1490 — Inhibit System Recovery** *(Analyst Assessment — not explicitly in CISA advisory)*  
Shadow copy deletion is a near-universal ransomware behaviour and a high-confidence indicator. Execution of commands such as `vssadmin delete shadows` or `wmic shadowcopy delete` should trigger immediate escalation. These commands have almost no legitimate business use and are a strong pre-ransomware indicator.

**T1486 — Data Encrypted for Impact**  
Mass file encryption can be detected via EDR behavioural detection rules monitoring for rapid file modification and unknown extension proliferation. However, detection at this stage indicates the attack has reached its final phase — earlier detection across the kill chain is essential to prevent reaching this point.

---

## 6. Recommendations

1. **Account hygiene and MFA:** Delete dormant accounts, enforce MFA on all remote access, monitor for credential abuse.
2. **Patch management:** Prioritise known exploited vulnerabilities, specifically internet-facing systems like VPN gateways and mail servers.
3. **Restrict remote access:** Disable unnecessary RDP exposure, enforce MFA on all VPN access, audit external-facing services regularly.
4. **Offline/immutable backups:** Maintain backups that are air-gapped or write-protected, test restoration regularly.
5. **Threat hunting and continuous monitoring:** Proactively hunt for dwell-time indicators such as unexpected AD enumeration, C2 beaconing, and privilege escalation activity.

---

## Sources

[1] CISA/FBI/ASD's ACSC Advisory AA23-352A — #StopRansomware: Play Ransomware  
https://www.cisa.gov/news-events/cybersecurity-advisories/aa23-352a  
Last updated June 4, 2025
