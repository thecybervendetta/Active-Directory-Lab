# Case Study: Active Directory Penetration Testing Lab

**Project:** AltSchool Africa Cybersecurity Capstone / Independent Research  
**Role:** Security Researcher / Penetration Tester  
**Technologies:** VMware Workstation Pro, Windows Server 2022, Windows 11, Kali Linux, Nmap, Enum4linux, Ldapsearch, Kerbrute, Hydra, Smbmap, Smbclient  

## 📌 Executive Summary
In this project, I engineered an isolated, virtualized Active Directory (AD) lab environment to conduct comprehensive network security testing. The environment consisted of a Windows Server 2022 Domain Controller, a Windows 11 domain-joined client, and a dual-homed Kali Linux attacker machine. The goal was to simulate an attacker with zero prior knowledge, systematically map the network, enumerate domain information, and successfully exploit weak credentials to achieve lateral movement.

## 🏗️ Architecture & Lab Topology
To ensure isolation from the host network while still allowing necessary testing functionality, the lab was carefully segmented:
*   **Windows Server 2022 (Domain Controller):** Configured with AD DS and DNS. IP: `192.168.1.10` (Host-Only).
*   **Windows 11 (Client):** Joined to the `company.local` domain. IP: `192.168.1.20` (Host-Only).
*   **Kali Linux (Attacker):** Dual-homed setup. NAT adapter for internet access (package upgrades), and Host-Only adapter (`192.168.1.30`) for direct communication with the lab subnet.

## ⚔️ The Attack Path: From Zero Knowledge to Exploitation

### 1. Reconnaissance
Starting with only the subnet address, I initiated a stealthy `nmap` TCP SYN scan against `192.168.1.0/24`. 
```bash
nmap -sS -T4 -F --open 192.168.1.0/24
```
*   **Discovery:** Identified two active hosts (`192.168.1.10` and `192.168.1.20`).
*   **Open Ports:** Exposed critical AD ports including 53 (DNS), 88 (Kerberos), 135 (RPC), 139 (NetBIOS), 389 (LDAP), and 445 (SMB).

### 2. Enumeration
With SMB exposed, I utilized `enum4linux` on both targets. This revealed the domain name (`COMPANY`), the Domain Controller's hostname (`WIN-BS07N70U9V`), and the client hostname (`WINCLIENT`).

Next, I queried LDAP to confirm the fully qualified domain name (FQDN):
```bash
ldapsearch -x -H ldap://192.168.1.10 -s base -b "" namingContexts
```
*   **Result:** Discovered the FQDN: `company.local`.

Using the FQDN, I executed a **Kerberos Enumeration** via `kerbrute` on port 88 to discover valid user principals:
```bash
kerbrute usernum -d company.local --dc 192.168.1.10 namelist.txt
```
*   **Result:** Successfully validated the username `administrator@company.local`.

### 3. Exploitation & Password Attack
After an anonymous SMB login attempt via `smbclient` confirmed that authentication was required, I initiated an active dictionary attack using `hydra` against the `winclient` machine.

> [!NOTE]
> **Troubleshooting Hydra SMBv2:** The default version of Hydra (9.5) threw "invalid reply" errors because it only supported SMBv1, which was disabled on modern Windows OS. I successfully troubleshot this by compiling **Hydra 9.7dev** from source, enabling native SMBv2 password cracking.

```bash
hydra -l 'winclient@company.local' -P /usr/share/wordlists/fasttrack.txt -s 445 192.168.1.20 smb2
```
*   **Result:** Successfully cracked the password (`Password12`) for the `winclient` account.

### 4. Post-Exploitation & Lateral Movement Mapping
Armed with valid credentials, I used `smbmap` to enumerate the accessible network shares on both the client and the Domain Controller. 
```bash
smbmap -u 'winclient' -p 'Password12' -d company -H 192.168.1.20
```
*   **Result:** Discovered limited Read-Only access to highly sensitive shares including `IPC$`, `NETLOGON`, and `SYSVOL`. I successfully authenticated directly into the `IPC$` share via `smbclient`.

## 🛡️ Findings & Hardening Recommendations

The assessment successfully exposed significant vulnerabilities in default Active Directory configurations:
1.  **Weak Passwords & Lack of MFA:** The immediate breach of the `winclient` account highlighted the necessity for enforced complex password policies and Multi-Factor Authentication (MFA) for domain accounts.
2.  **Unrestricted SMB:** SMBv2 brute-forcing was possible due to a lack of rate limiting. Recommendation: Enforce SMB signing, disable obsolete protocols, and implement alert mechanisms for spray-like authentication behavior.
3.  **Over-Exposed Services:** Telemetry (LSAP, DNS SRV) freely leaked host and domain structures. Recommendation: Implement host-based firewalls to restrict management ports to known administrative subnets and adhere strictly to the principle of least privilege.

---
*This lab demonstrates my practical ability to provision infrastructure from scratch, execute systematic penetration testing methodologies, solve complex tooling issues on the fly, and deliver actionable security remediations.*
