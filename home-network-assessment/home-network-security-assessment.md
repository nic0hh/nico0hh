# Home Network Security Assessment
### Passive Traffic Analysis and Asset Discovery

## 1. Summary

In scanning the traffic going through my Pi-hole DNS filter with tcpdump, I discovered both unencrypted HTTP and exposed SSH which pose risks to network security. A couple of unidentified devices were discovered along with a lot of IoT background noise. A controversial anti-cheat gaming software was checked upon game launch without any suspicious behaviour to report. Overall, the Pi-hole DNS integrity was confirmed but since DNS analysis only reveals destination, further investigation would require process-level monitoring, extended capture, and sandbox analysis for kernel drivers.

## 2. Scope and Methodology

**Capture time:** 27/03/2026, 12:30–13:00

**Tools used:**
- tcpdump (Raspberry Pi 4B)
- Nmap 7.99
- Wireshark
- VirusTotal

**Network overview:**
- 18 hosts discovered
- Pi-hole DNS filtering with Cloudflare upstream
- Mix of wired and wireless devices

**Traffic intentionally generated:**
- Nmap scans from PC
- Game launch (Naraka: Bladepoint with NEAC Protect)
- General browsing

## 3. Asset Discovery (Nmap)

### 3.1 Network Inventory

Nmap discovered 18 active hosts on the 192.168.2.0/24 subnet. The table below summarises devices, network addresses, and any notes.

| IP Address | MAC Vendor | Device Type | Notes |
|---|---|---|---|
| 192.168.2.1 | Sagemcom Broadband SAS | Router/Modem | 4 unexpected ports open |
| 192.168.2.2 | Sonos | Smart Speaker | Proprietary protocol 0x6969 |
| 192.168.2.4 | Sonos | Smart Speaker | - |
| 192.168.2.5 | Sonos | Smart Speaker | - |
| 192.168.2.6 | Apple | iPhone/iPad | - |
| 192.168.2.7 | Apple | iPhone/iPad | - |
| 192.168.2.8 | Unknown | Unknown | Determined Apple device |
| 192.168.2.9 | Arcadyan | Network Device | HTTP admin interface open |
| 192.168.2.10 | Arcadyan | Network Device | HTTP admin interface open |
| 192.168.2.14 | Unknown | Unknown | Determined Apple device |
| 192.168.2.18 | Beijing Xiaomi | IoT desk light | All ports closed |
| 192.168.2.22 | Apple | iPhone/iPad | - |
| 192.168.2.25 | Liteon Technology | Unknown | All ports filtered |
| 192.168.2.32 | Raspberry Pi | Pi-hole DNS #1 | SSH exposed. Port 3000 open (Grafana dashboard) |
| 192.168.2.35 | Samsung Electronics | Smart TV | All ports closed |
| 192.168.2.202 | Raspberry Pi | Pi-hole DNS #2 | SSH exposed. Port 3000 open (Grafana dashboard) |
| 192.168.2.254 | Sagemcom Broadband SAS | Gateway | ISP management ports present, all closed |
| 192.168.2.34 | - | Windows Workstation | SMB ports 139/445 open |

### 3.2 Device Identification Notes

Two hosts presented unknown MAC vendors: 192.168.2.8 and 192.168.2.14. Details of open ports revealed port 62078 (iphone-sync) on both devices, indicating Apple iOS devices with MAC randomisation enabled. This is a privacy feature introduced in iOS 14, where devices present a randomised MAC address per network, preventing tracking via hardware address. While negligible in this assessment, MAC randomisation has implications for network asset management as device inventory based solely on MAC vendor lookup will fail to identify these devices.

Furthermore, hosts 192.168.2.6, 192.168.2.7, and 192.168.2.22 are all identified Apple MACs, making a total of 5 alongside the unidentified ones. However, there are only 4 Apple devices in the environment. This discrepancy is attributed to MAC address randomisation causing a single device to register as multiple hosts during the extended scan window of 3,859 seconds. This demonstrates how MAC randomisation can inflate host count during static network scanning.

### 3.3 Findings

| Finding | Devices | Port | Risk | Recommendation |
|---|---|---|---|---|
| Unencrypted HTTP admin interfaces | 192.168.2.9 and .10 (Arcadyan) | Port 80 | Credentials transmitted in plaintext. | Enable HTTPS on admin interface; if unsupported by firmware, restrict access by IP or disable remote admin. |
| Local DNS running | 192.168.2.9 and .10 (Arcadyan) | Port 53 | Could bypass Pi-hole DNS filtering. | Investigate whether any hosts are resolving via Arcadyan rather than Pi-hole. Configure devices to force DNS through Pi-hole at the router level. |
| SMB exposure on workstation | 192.168.2.34 (Windows PC) | Ports 135, 139, 445 | Historically exploited protocol (WannaCry). SMBv1 disabled but SMBv2/3 still exposed. Unnecessary exposure on home network. | Block ports 135, 139, 445 in Windows Firewall. |
| SSH exposed on both Pi-hole instances | 192.168.2.32 and .202 | Port 22 | Brute force surface. Pi-hole compromise would allow DNS manipulation affecting all network traffic. | Upgrade to key-based authentication, disable password authentication. |
| LLMNR active on network | Network-wide | - | Credential capture via broadcast poisoning. | Protocol redundant with Pi-hole DNS stack: disable. |
| Unidentified device | 192.168.2.25 (Liteon) | All ports filtered | Unidentified devices cannot be monitored or assessed for malicious behaviour, and if compromised, might not be recognised as anomalous. | Cross-reference DHCP lease timestamps against known devices, isolate to guest VLAN if unidentified. |

## 4. Traffic Analysis

### Sonos Proprietary Protocol (0x6969)

High volume of traffic from three Sonos speakers observed. Proceeded to isolate meaningful traffic:

- Standard UPnP and mDNS exclusion: `!(udp.port == 1900) && !(udp.port == 5353)`. This was insufficient since Sonos on ethernet uses a proprietary protocol (EtherType 0x6969).

- Extended filters applied: `!(eth.type == 0x6969) && !(udp.port == 1900) && !(udp.port == 5353)`

Traffic remained entirely within the local subnet with no external connections observed. Classified as benign.

### Pi-hole DNS Integrity

- DNS queries were then isolated to reduce noise further, filtering to queries only and excluding responses: `dns.flags.response == 0 && !(eth.type == 0x6969) && !(udp.port == 1900) && !(udp.port == 5353)`

- Pi-hole DNS integrity check by excluding Pi-hole instances and Cloudflare upstream queries: `dns && ip.dst == 1.1.1.1 && !(ip.src == 192.168.2.202) && !(ip.src == 192.168.2.32)`. The Pi-hole instances were excluded as they legitimately forward queries to Cloudflare upstream — any other host querying 1.1.1.1 directly would indicate a bypass. No such traffic was detected. All DNS queries observed during the capture window were routed through Pi-hole, confirming the filtering stack is functioning correctly.

### NEAC Protect Anti-Cheat (Naraka: Bladepoint)

NEAC Protect is a kernel-level anti-cheat driver, at times being accused of being spyware online.

- The initial query for "neac" did not yield any results: `dns.qry.name contains "neac"`

- Filtering to the owner company yielded 5 queries: `dns.qry.name contains "netease"`

All 5 DNS queries were routed through Pi-hole and all were observed during game launch only. Domain and resolved IP were checked against VirusTotal and came back clean. As such, no unexpected behaviour was identified at the DNS layer. It is worth noting that kernel-level drivers operate below the visibility of passive DNS capture and a complete behavioural assessment of NEAC Protect is needed to fully declare it harmless.

## 5. Limitations and Further Investigation

This assessment was limited to passive DNS layer analysis and active Nmap scanning over a 30-minute window. The following require extended methods:

- **NEAC Protect kernel driver:** Full behavioural analysis requires process-level monitoring and sandbox execution.

- **Inter-device traffic:** Lateral movement indicators between hosts are not visible in a short passive capture window.
