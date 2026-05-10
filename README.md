# Active Directory Attack & Detection Lab

A full-stack cybersecurity home lab simulating real-world enterprise attacks and detection using a custom Active Directory environment and Elastic SIEM.

---

## Lab Overview

This project demonstrates the full cycle of cybersecurity operations: building enterprise infrastructure, introducing common misconfigurations, executing real-world attack techniques, and detecting every attack using a SIEM.

---

## Architecture

| Component | Technology |
| :--- | :--- |
| Hypervisor | VirtualBox |
| Network | Isolated NAT Network (10.0.0.0/24) |
| Domain Controller | Windows Server 2022 |
| Workstations | Windows 10 Pro (x2) |
| Attack Machine | Kali Linux |
| SIEM | Elastic Stack (Elasticsearch + Kibana) |
| Log Shipper | Winlogbeat |

### Network Topology

- **DC01** — 10.0.0.5 (Static) — Domain Controller, DNS, DHCP
- **WS01** — DHCP — Domain-joined workstation
- **WS02** — DHCP — Domain-joined workstation
- **Kali** — 10.0.0.100 (DHCP) — Attack machine
- **SIEM** — 10.0.0.102 (Static) — Elasticsearch + Kibana

All VMs connected via VirtualBox isolated NAT Network `AD-Lab-Net` (10.0.0.0/24). No traffic reaches the host or external networks.

---

## Infrastructure Build

### Domain Controller Setup

Windows Server 2022 Standard with Desktop Experience was installed on a VirtualBox VM with 2 CPUs, 2GB RAM, and a 50GB disk. A static IP of 10.0.0.5 was assigned — Domain Controllers must have fixed addresses. The server was promoted to a Domain Controller for a new forest named `homelab.local`. DNS was installed automatically with Active Directory. DHCP was configured with a scope of 10.0.0.100-200 to assign IPs to workstations and the attacker machine.

### Organizational Units & Users

Two OUs were created: `IT` and `Sales`, both protected from accidental deletion. User accounts were created and assigned to groups. Two users were intentionally added to Domain Admins to create high-value targets for privilege escalation attacks.

| Username | Display Name | OU | Group Membership |
| :--- | :--- | :--- | :--- |
| jsmith | John Smith | Sales | Domain Users |
| tbrown | Tina Brown | Sales | Domain Admins |
| ftashi | Fade Tashi | IT | Domain Admins |
| sfaja | Sel Faja | IT | Domain Users |

### Workstation Deployment

Two Windows 10 Pro VMs were created with 2 CPUs, 2GB RAM, and 50GB disks each. Windows 10 Pro is required for domain join capability. Both were joined to `homelab.local` and authenticated with domain user accounts.

### Attacker Machine

Kali Linux was imported from a pre-built VirtualBox image and connected to AD-Lab-Net. It received IP 10.0.0.100 from DC01's DHCP server. Tools installed include Nmap, CrackMapExec, Impacket, Responder, and John the Ripper.

### SIEM Deployment

Ubuntu Server 22.04 was installed with 2 CPUs, 4GB RAM, and a 60GB disk. It has two network adapters — NAT for internet during package installation, and AD-Lab-Net (10.0.0.102 static) for log ingestion. Elasticsearch 8.x and Kibana 8.x were installed from the official Elastic APT repository. Winlogbeat was installed on DC01 to forward Windows Security Event Logs to Elasticsearch. The index pattern `winlogbeat-*` was created in Kibana confirming log ingestion.

---

## Deliberate Vulnerabilities

Seven misconfigurations were intentionally introduced to simulate real-world administrative mistakes. Each enables a specific attack technique.

| # | Vulnerability | Detail | Attack Enabled |
| :--- | :--- | :--- | :--- |
| 1 | Weak user password | jsmith given a simple, guessable password | Password spraying |
| 2 | Domain Admin weak password | tbrown given a common password | Credential compromise |
| 3 | SPN on user account | ftashi registered with HTTP/svcadmin.homelab.local | Kerberoasting |
| 4 | Kerberos pre-auth disabled | sfaja set to DoesNotRequirePreAuth | AS-REP roasting |
| 5 | Open network share | C:\Public with Everyone Full Control | SMB share enumeration |
| 6 | LLMNR enabled | Multicast enabled via registry | LLMNR/NBT-NS poisoning |
| 7 | SMB signing disabled | Server-side SMB signing turned off | SMB relay attacks |

---

## Attack Simulation

All attacks were performed from Kali Linux, starting with only network access — no credentials or domain information.

### 1. Network Discovery

The entire subnet was scanned to identify live hosts. DC01 was found at 10.0.0.5.

### 2. Port Scanning & Service Detection

A service scan against 10.0.0.5 revealed ports 53 (DNS), 88 (Kerberos), 389 (LDAP), and 445 (SMB) — confirming this is a Domain Controller. The domain name `homelab.local` and hostname `DC01` were identified.

### 3. Anonymous Enumeration

An anonymous LDAP root query leaked the domain structure, functionality levels, and supported authentication methods without needing credentials. Anonymous SMB login was accepted but no shares were listed — SMB1 was disabled. An attempt to dump users without authentication was blocked. This shows Windows Server 2022 has improved default security over older versions, but still leaks useful information.

### 4. Password Spraying

A list of common usernames and passwords was sprayed against the Domain Controller. Valid credentials for jsmith were discovered. This established a foothold as a regular domain user.

### 5. Authenticated LDAP User Dump

Using the jsmith credentials, the entire Active Directory was queried. Ten accounts were enumerated including Administrator, two Domain Admins (tbrown and ftashi), krbtgt, and computer accounts. A regular user can map the entire domain in minutes.

### 6. Kerberoasting

Service Principal Names were queried to find accounts with registered SPNs. ftashi — a Domain Admin — had an SPN registered. A Kerberos service ticket hash was extracted and saved for offline cracking. Rockyou.txt with 14 million passwords did not crack the hash, meaning the password is not in common breach lists.

### 7. AS-REP Roasting

The user list was checked for accounts with Kerberos pre-authentication disabled. sfaja was found to have this setting. An AS-REP hash was extracted without using any password or authentication — only a valid username was needed.

### 8. LLMNR/NBT-NS Poisoning

Responder was run on Kali to listen for LLMNR and NBT-NS broadcast requests. When a user on WS01 attempted to access a non-existent network share, Responder poisoned the response and captured the NTLMv2 hash. This attack requires no exploit — only protocol abuse.

### 9. Offline Hash Cracking

The extracted Kerberos hash was run against rockyou.txt using John the Ripper. The hash was not cracked — the password is not a common one. The NTLMv2 hash from the LLMNR capture was cracked successfully, recovering jsmith's password.

---

## Detection (Blue Team)

### SIEM Architecture

Windows Event Logs from DC01 are forwarded by Winlogbeat to Elasticsearch running on the SIEM server at 10.0.0.102. Kibana provides the query and dashboard interface at port 5601.

### Attack → Detection Mapping

| Attack | Event ID | Description |
| :--- | :--- | :--- |
| Password Spraying | 4625 | Failed logon |
| Password Spraying (success) | 4624 | Successful logon |
| LDAP User Dump | 4662 | Directory service access |
| Kerberoasting | 4769 | Kerberos service ticket requested |
| AS-REP Roasting | 4768 | Kerberos TGT requested |

All five attack types were successfully detected in Kibana Discover by filtering on these Event IDs.

---

## Key Learnings

### Offensive
- Windows Server 2022 blocks anonymous LDAP enumeration by default, but any authenticated user can dump the entire directory
- Password spraying is still effective — one weak password can expose the entire domain
- A low-privilege user can extract Domain Admin hashes through Kerberoasting and AS-REP roasting
- LLMNR/NBT-NS poisoning captures hashes with no exploit — just protocol abuse

### Defensive
- Every attack maps to a specific Windows Event ID, making detection straightforward
- A properly configured SIEM with Winlogbeat detects all five attack techniques in real time
- Disabling LLMNR, NBT-NS, and SMB1 eliminates several attack vectors
- Group Managed Service Accounts prevent Kerberoasting
- Least privilege, strong passwords, and audit logging are the foundation of detection

---

## Tools Used

| Tool | Purpose |
| :--- | :--- |
| VirtualBox | Hypervisor |
| Windows Server 2022 | Domain Controller |
| Windows 10 Pro | Workstations |
| Ubuntu Server 22.04 | SIEM host |
| Kali Linux | Attack platform |
| Nmap | Network scanning |
| CrackMapExec | Password spraying |
| Impacket | Kerberoasting and AS-REP roasting |
| Responder | LLMNR/NBT-NS poisoning |
| John the Ripper | Hash cracking |
| Elasticsearch + Kibana | SIEM |
| Winlogbeat | Log shipping |

---

## Screenshots

### Phase 1: Infrastructure Build

![DC01 First Boot](images/01-dc01-first-boot.png)
*Windows Server 2022 Desktop Experience on first boot. Desktop Experience was chosen over Server Core for visibility while learning.*

![Active Directory Users and OUs](images/02-active-directory-users-ous.png)
*OUs IT and Sales created with user accounts populated. OUs are protected from accidental deletion.*

![Domain Admin Membership](images/03-domain-admin-membership.png)
*ftashi and tbrown added to Domain Admins. These privileged accounts become the targets for privilege escalation.*

![Workstation Domain Join](images/04-workstation-domain-join.png)
*WS02 successfully joined to homelab.local. Both workstations are targets for lateral movement and LLMNR poisoning.*

![Kali IP on AD-Lab-Net](images/05-kali-ip-on-ad-lab-net.png)
*Kali Linux received 10.0.0.100 from DC01's DHCP server, confirming network connectivity to the domain.*

### Phase 2: Vulnerability Configuration

![SPN Configuration](images/06-spn-configuration.png)
*SPN HTTP/svcadmin.homelab.local registered on ftashi, a Domain Admin. SPNs should be on managed service accounts, not user accounts.*

![Pre-Authentication Disabled](images/07-pre-auth-disabled.png)
*Kerberos pre-authentication disabled on sfaja. This legacy setting enables AS-REP roasting.*

![LLMNR Enabled](images/08-llmnr-enabled.png)
*LLMNR protocol enabled via registry. This legacy name resolution protocol enables credential capture with Responder.*

![Open SMB Share](images/09-open-smb-share.png)
*C:\Public share created with Everyone Full Control. Overly permissive shares are a common real-world finding.*

### Phase 3: Reconnaissance

![Nmap Discovery and Service Scan](images/10-nmap-discovery-scan.png)
*Nmap confirms DC01 with ports 53, 88, 389, and 445 open — a textbook Domain Controller footprint.*

![LDAP Root DSE Output](images/11-ldap-rootdse-output.png)
*Anonymous LDAP query returns domain name, hostname, and functional levels. No credentials needed.*

![Anonymous SMB Login](images/12-anonymous-smb-login.png)
*Anonymous SMB login accepted but no shares exposed. SMB1 disabled by default on Windows Server 2022.*

### Phase 4: Exploitation

![Password Spray Success](images/13-password-spray-success.png)
*CrackMapExec finds valid credentials for jsmith. The green plus marks the foothold moment.*

![Authenticated LDAP User Dump](images/14-authenticated-ldap-dump.png)
*As a regular user, the entire directory is enumerated — 10 accounts including Domain Admins.*

![Kerberoasting Hash Extraction](images/15-kerberoasting-hash.png)
*Kerberos TGS hash for Domain Admin ftashi extracted. The hash is ready for offline cracking.*

![AS-REP Hash Extraction](images/16-asrep-roasting-hash.png)
*AS-REP hash for sfaja extracted without using any password. Only a username was needed.*

![Responder LLMNR Capture](images/17-responder-llmnr-capture.png)
*NTLMv2 hash for jsmith captured via LLMNR poisoning when accessing a fake network share.*

![Hash Cracking with RockYou](images/18-hash-cracking-rockyou.png)
*John the Ripper runs rockyou.txt (14 million passwords) against the Kerberoasting hash. Hash not cracked.*

### Phase 5: SIEM Detection

![Winlogbeat Configuration](images/19-winlogbeat-config.png)
*Winlogbeat configured to ship DC01 Security logs to Elasticsearch at 10.0.0.102.*

![Kibana Data View](images/20-kibana-data-view.png)
*Index pattern winlogbeat-* created in Kibana, confirming logs are flowing from DC01.*

![Event 4625 - Failed Logons](images/21-kibana-event-4625.png)
*Event ID 4625 detected — failed logon attempts from the password spray attack.*

![Event 4624 - Successful Logon](images/22-kibana-event-4624.png)
*Event ID 4624 detected — successful logon confirming credential compromise.*

![Event 4769 - Kerberoasting](images/23-kibana-event-4769.png)
*Event ID 4769 detected — Kerberos service ticket request from Kerberoasting activity.*

![Event 4768 - AS-REP Roasting](images/24-kibana-event-4768.png)
*Event ID 4768 detected — TGT request from AS-REP roasting activity.*

![Event 4662 - LDAP Enumeration](images/25-kibana-event-4662.png)
*Event ID 4662 detected — directory service access from LDAP user dump.*

---

## Future Enhancements

- Add Suricata IDS for network-based intrusion detection
- Build SOAR playbook for automated incident response
- Implement Group Policy hardening (disable LLMNR, enforce password policies)
- Deploy Microsoft LAPS for local admin password management
- Create custom Kibana dashboards with visual alerting
- Add cloud component (AWS/Azure) for hybrid detection scenarios

---

## Disclaimer

This lab was built for educational purposes only. All attacks were performed against owned infrastructure in an isolated virtual environment. Do not use these techniques against systems without explicit authorization.

---

**Author:** Marie Angeline Pelausa
**Date:** May 2026