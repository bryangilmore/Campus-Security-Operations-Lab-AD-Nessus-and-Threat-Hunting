# Campus-Security-Operations-Lab-AD-Nessus-and-Threat-Hunting
Home lab simulating a small campus network with pfSense, Active Directory, Nmap, Nessus, and basic threat hunting in Windows event logs.

## 1. Overview

This lab simulates a small Windows domain behind a pfSense firewall, with a domain controller, Windows client, and a Nessus Essentials scanner.  

Goals:

- Build and harden a basic AD environment
- Discover hosts and services with Nmap
- Run unauthenticated and authenticated Nessus scans
- Generate suspicious activity and hunt it in Windows event logs

## 2. Lab Topology

![GNS3 topology](screenshots/00-gns3-topology.png)

**Key components**

- pfSense firewall, routing between lab networks
- WIN-DC1, Windows Server domain controller, 10.10.20.10
- WIN-CLT1, Windows 11 domain joined client, 10.10.30.50
- Nessus Essentials on WIN-CLT1, scanning both hosts

## 3. Network Configuration

### pfSense interfaces

![pfSense ifconfig](screenshots/01-pfSense-ifconfig.png)

- `em0` 10.10.254.2, transit network toward GNS3 router
- `em1` 172.16.0.1, management or WAN side used by GNS3

### Domain controller IP

![DC1 ipconfig](screenshots/02-dc1-ipconfig.png)

- IP 10.10.20.10, subnet 255.255.255.0
- Default gateway 10.10.20.1 (router toward pfSense)
- DNS points to itself

### Client IP and connectivity

![Client ipconfig](screenshots/03-client-ipconfig.png)  
![Client connectivity to DC](screenshots/04-client-connectivity-to-dc.png)

- Client IP 10.10.30.50
- Client can ping 10.10.20.10 and use it as domain controller

## 4. Active Directory Design

### OUs and users

![AD OUs and users](screenshots/05-ad-ous-and-users.png)

- `OU=Campus-Users`
  - `OU=Students`
  - `OU=Staff`

Example test users:

- `Alex Student`
- `Sam Staff`

### Security groups

![AD security groups](screenshots/06-ad-security-groups.png)

- `GG-Students`
- `GG-Staff`

Groups are used for Group Policy filtering so students and staff receive different restrictions.

### Group Policy objects

![GPO list](screenshots/07-gpos-list.png)

Custom GPOs:

- `GG_Students_Lockdown`
- `GG_Staff_Standard`

Linked to the appropriate OUs:

![OU GPO links](screenshots/08-ou-gpo-links.png)

### GPO results on clients

Student:

![gpresult student](screenshots/09-gpresult-student.png)

Staff:

![gpresult staff](screenshots/10-gpresult-staff.png)

This confirms the correct policies are applied to each user type through security group filtering.

## 5. Network Discovery and Port Scanning

### 5.1 Host discovery

![Nmap host discovery](screenshots/11-nmap-host-discovery.png)

Used `nmap -sn 10.10.30.0/24` to discover live hosts. The scan identified:

- 10.10.30.1, router
- 10.10.30.50, client

### 5.2 Service scans

Domain controller:

![Nmap DC1](screenshots/12-nmap-dc1.png)

Client:

![Nmap client](screenshots/13-nmap-client.png)

These show typical Windows services open on the DC, for example RPC, SMB, and directory related ports, and workstation services on the client. These screenshots go into the “network enumeration” section of the write-up.

## 6. Nessus Vulnerability Management

### 6.1 Scan configuration

![Nessus scan config](screenshots/14-nessus-scan-config.png)

- Policy based on Basic Network Scan
- Targets, 10.10.20.10 and 10.10.30.50
- Authenticated Windows credentials, `CAMPUS\administrator`

### 6.2 Unauthenticated scan results

Hosts summary:

![Unauthenticated hosts summary](screenshots/15-nessus-unauthenticated-hosts-summary.png)

Domain controller:

![Unauthenticated DC1 vulns](screenshots/16-nessus-unauthenticated-dc1-vulns.png)

Client:

![Unauthenticated client vulns](screenshots/17-nessus-unauthenticated-client-vulns.png)

### 6.3 Authenticated scan results

Domain controller:

![Authenticated DC1 vulns](screenshots/18-nessus-authenticated-dc1-vulns.png)

Hosts summary:

![Authenticated hosts summary](screenshots/19-nessus-authenticated-hosts-summary.png)

The authenticated scan provides deeper findings such as missing patches, insecure configuration settings, and local privilege escalation issues.

Raw exports are in the `nessus` folder:

- `nessus/campus-lab-baseline-vuln-scan.html`
- `nessus/campus-lab-baseline-vuln-scan.csv`

### 6.4 High level remediation plan

Examples of actions that could be taken based on typical Nessus findings:

- Apply missing Windows security updates on WIN-DC1 and WIN-CLT1
- Review and harden SMB configuration, disable SMBv1 where possible
- Enforce strong password and lockout policies through Group Policy
- Reduce exposed services on the client, for example disable unused management ports

(You can refine this list based on the actual high and critical findings in the CSV.)

## 7. Mini Threat Hunt

### 7.1 Simulated attack activity

A small noisy event was created by trying repeatedly to reach the DC admin share with incorrect credentials.

![Simulated failed logons](screenshots/20-threat-simulated-failed-logons.png)

Command used on WIN-CLT1:

```powershell
for ($i = 0; $i -lt 5; $i++) {
    try {
        net use \\WIN-DC1\C$ /user:CAMPUS\fakeuser WrongPass!1
    } catch {}
}
