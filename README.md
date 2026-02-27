# Cloud_Active_Directory_Keberoasting_Lab


````markdown
# AWS Active Directory External Exposure → Kerberoasting → Remediation (Step-by-Step + Notes)

> Project type: Cloud Security / Active Directory Attack Simulation (External Attacker)  
> Environment: AWS
> Goal: Show how a misconfigured cloud perimeter can expose AD services to external attacks, demonstrate impact (SMB auth + Kerberoasting), then remediate and verify.

---

## Table of Contents
- [Why I built this](why-i-built-this)
- [High-level story](high-level-story)
- [Architecture](architecture)
- [Tools used](tools-used)
- [Build Log (Step-by-step)](build-log-step-by-step)
  - [Phase 1 — AWS Network Setup](phase-1--aws-network-setup)
  - [Phase 2 — Launch Domain Controller + Configure AD](phase-2--launch-domain-controller--configure-ad)
  - [Phase 3 — Launch Domain Member Server (Client)](phase-3--launch-domain-member-server-client)
  - [Phase 4 — External Recon (Kali)](phase-4--external-recon-kali)
  - [Phase 5 — SMB Authentication & Password Spray](phase-5--smb-authentication--password-spray)
  - [Phase 6 — Kerberoasting](phase-6--kerberoasting)
  - [Phase 7 — Remediation & Verification](phase-7--remediation--verification)
- [Troubleshooting Notes (Real issues I hit)](troubleshooting-notes-real-issues-i-hit)
- [Screenshots / Evidence](screenshots--evidence)
- [Findings](findings)
- [MITRE ATT&CK Mapping](mitre-attck-mapping)
- [Remediation Summary](remediation-summary)
- [Next Steps (Privilege Escalation)](next-steps-privilege-escalation)

---

## Why I built this

A lot of cloud breaches don’t start with “zero-days.”  
They start with exposed services, weak credentials, and identity systems (AD) reachable from places they shouldn’t be.

I built this lab to simulate a realistic situation:

- A company deploys Active Directory in AWS
- Cloud firewall rules (Security Groups) expose AD ports externally (even if “restricted”)
- An external attacker enumerates services, validates credentials, and performs Kerberoasting
- The defender then remediates by reducing attack surface and verifying the fix

This repo is written as a walkthrough + my thinking as I went.

---

## High-level story

External attacker path:
1. Identify exposed AD ports (88/389/445/3389)
2. Validate SMB authentication using weak credentials
3. Enumerate SPNs and extract Kerberoastable TGS hash
4. Prove impact, then fix it

Defender path:
1. Reduce externally exposed ports (close 88/389/445)
2. Keep only RDP open to my IP for management
3. Re-scan to confirm the attack path is dead

---

## Architecture

- VPC: 10.0.0.0/16
- Subnet: 10.0.1.0/24 (public for simplicity)
- Instances:
  - AD-DC-01 — Windows Server 2022 Domain Controller
  - AD-CLIENT-01 — Windows Server 2022 domain member (used as “client” since Windows 10 AMI wasn’t available without Marketplace costs)
- Attacker: Local Kali VM (outside AWS)

> Important design note: This project intentionally used a public subnet to simulate external exposure.  
> In real environments, AD should not be internet-reachable.

---

## Tools used

AWS / Cloud
- EC2, VPC, Subnets, Route Tables, Internet Gateway, Security Groups

Windows / AD
- AD DS role, DNS, dsa.msc, setspn

Kali / Attacker tools
- nmap
- smbclient
- crackmapexec
- impacket-GetUserSPNs

---

# Build Log (Step-by-step)

## Phase 1 — AWS Network Setup

### 1) Create VPC
- VPC CIDR: 10.0.0.0/16

### 2) Create Subnet
- Subnet CIDR: 10.0.1.0/24

### 3) Internet Gateway + Route Table
- Create IGW and attach to VPC
- Create a new route table (kept the default “main” untouched)
- Add route:
  - 0.0.0.0/0 → Internet Gateway
- Associate route table to subnet

✅ At this point, the subnet had internet access.

---

## Phase 2 — Launch Domain Controller + Configure AD

### 1) Launch EC2: Domain Controller
- Name: AD-DC-01
- OS: Windows Server 2022 Base (Amazon-owned, not Marketplace)
- Type: t3.micro (kept cost low)
- Storage: 30GB gp3

### 2) Connect via RDP
- Download .rdp file from AWS
- Decrypt Windows password using key pair
- Login as Administrator

### 3) Set Static IP inside Windows
This is critical for AD stability.

Inside the server:
- ncpa.cpl → Ethernet → IPv4 properties
- Set:
  - IP: (the current private IP from ipconfig, e.g. `10.0.1.10`)
  - Mask: 255.255.255.0
  - Gateway: 10.0.1.1
  - DNS: 127.0.0.1 (because this will become DNS server)

### 4) Install Active Directory Domain Services
Server Manager → Add Roles → Active Directory Domain Services  
Then promote to DC:
- New forest: corp.local

After reboot, login:
- corp\Administrator

### 5) Create Users + Service Account
Opened AD Users and Computers:
- Win + R → dsa.msc

Created:
- jsmith / Password123!
- dbenz / Password123!
- cmark / Welcome123!
- itadmin / Admin123! (added to Domain Admins)
- svc-sql / Service123! (service account)

### 6) Configure SPN (for Kerberoasting)
On AD-DC-01 (Admin cmd):

```powershell
setspn -A MSSQLSvc/sql.corp.local:1433 corp\svc-sql
setspn -L corp\svc-sql
````

✅ Verified SPN exists.

---

## Phase 3 — Launch Domain Member Server (Client)

### Why not Windows 10?

In my region, Windows 10 AMIs were not available without Marketplace vendor images (extra cost).
To keep the project cost-safe, I used another Windows Server as the “client.”

### 1) Launch EC2: Domain Member

* Name: AD-CLIENT-01
* Windows Server 2022 Base
* t3.micro
* 30GB storage
![Launch, Domain Member](https://i.imgur.com/O9x2EwZ.png)

### 2) Set Client DNS to DC IP

On AD-CLIENT-01:

* ipconfig (note current private IP)
* ncpa.cpl → IPv4 properties:

  * **DNS:** DC private IP (example: 10.0.1.10)

Test:

```powershell
ping <DC_PRIVATE_IP>
nslookup corp.local
```
![Setting Up Domain Member](https://i.imgur.com/vFwOfeL.png)
✅ Ping + DNS resolution worked.

### 3) Join Domain

* sysdm.cpl → Computer Name → Change → Domain: corp.local
* Auth: `corp\Administrator`

![Joining domain](https://i.imgur.com/VQuFNMA.png)

Reboot.

### 4) Login challenge (real issue)

I attempted RDP login as jsmith and got:

> “account wasn’t authorized for remote login”

This is normal. Non-admin users usually aren’t allowed RDP on servers by default.
I logged in using `itadmin` (Domain Admin) for management and verification.

---

## Phase 4 — External Recon (Kali)

Attacker machine: **Local Kali VM** (outside AWS)

### 1) Validate external exposure

```bash
nmap -sS -Pn -p 88,389,445,3389 <DC_PUBLIC_IP>
```
![Recon With Kali](https://i.imgur.com/KD12oKc.png)
✅ Result: all ports open (misconfiguration state confirmed)

---

## Phase 5 — SMB Authentication & Password Spray

### 1) SMB discovery script odd behavior

I ran:

```bash
nmap --script smb-os-discovery -p 445 <DC_PUBLIC_IP>
```

It appeared “filtered” even though port 445 was “open” on basic scans.
![Filtered Scan](https://i.imgur.com/Xf9T26z.png)

This was a useful lesson:

* **Nmap “open”** can mean handshake succeeds
* SMB scripts can fail due to negotiation/protocol constraints

### 2) SMB connection timeout (NT_STATUS_IO_TIMEOUT)

smbclient initially timed out.
Fix: force SMB2 (modern Windows Server requires SMB2/SMB3).

✅ Working command:

```bash
smbclient -L //<DC_PUBLIC_IP>/ -U jsmith --option='client min protocol=SMB2'
```

![Force SMB2](https://i.imgur.com/zKZoEku.png)

### 3) Password spray simulation

Used known weak passwords (lab-created accounts).
Result: successful authentication for users where the password was correct.

(Example testing approach)

```bash
crackmapexec smb <DC_PUBLIC_IP> -u cmark -p Password123!
crackmapexec smb <DC_PUBLIC_IP> -u dbenz -p Password123!
```
![Password Reuse](https://i.imgur.com/rqk1Vtg.png)
![Password Reuse](https://i.imgur.com/e33Vg29.png)

✅ Demonstrated how weak password reuse increases blast radius.

---

## Phase 6 — Kerberoasting

### 1) Enumerate SPNs

```bash
impacket-GetUserSPNs corp.local/jsmith:Password123! -dc-ip <DC_PUBLIC_IP>
```
![Enumerate SPNs](https://i.imgur.com/vEpjjJH.png)
✅ Saw svc-sql in the results.

### 2) Request Kerberos service ticket (hash extraction)

```bash
impacket-GetUserSPNs corp.local/jsmith:Password123! -dc-ip <DC_PUBLIC_IP> -request
```
![Enumerate SPNs](https://i.imgur.com/cF3i0lZ.png)
✅ Output included a `$krb5tgs$...` hash (Kerberoastable).

This proved:

* external Kerberos reachability
* SPN-based ticket retrieval
* offline crack pathway

---

## Phase 7 — Remediation & Verification

### 1) Close AD ports externally (Security Group)

On **DC Security Group**, removed external access for:

* 445 (SMB)
* 389 (LDAP)
* 88 (Kerberos)
* 
![DC Security](https://i.imgur.com/7SyTf1d.png)

Kept:

* 3389 (RDP) open **only to my IP** for management

### 2) Re-test scan

```bash
nmap -sS -Pn -p 88,389,445,3389 <DC_PUBLIC_IP>
```

✅ Result:

* 88 filtered
* 389 filtered
* 445 filtered
* 3389 open

![Re-test](https://i.imgur.com/tspKKbF.png)
Remediation verified.

---

# Troubleshooting Notes (Real issues I hit)

## 1) “I can’t switch user / it keeps logging in as Administrator”

Cause:

* RDP cached credentials (“Remember me” / saved creds)

Fix:

* Cleared saved credentials on my local PC (Credential Manager)
* Used “Use a different account” when reconnecting

## 2) “jsmith not authorized for remote login”

Cause:

* On Windows Server, normal domain users usually aren’t allowed RDP by default

Fix:

* Logged in with itadmin (Domain Admin)
* (Optional improvement) Add normal users to **Remote Desktop Users** if you want them to RDP

## 3) SMB timeouts but port shows open

Cause:

* SMB negotiation/protocol differences; modern Windows requires SMB2/3

Fix:

* Force SMB2:

```bash
smbclient -L //<DC_PUBLIC_IP>/ -U jsmith --option='client min protocol=SMB2'
```

---

# Findings

## Finding 1 — Active Directory ports exposed externally

* **Impact:** external discovery + authentication path exists
* **Risk:** high in real environments

## Finding 2 — Weak password reuse across multiple users

* **Impact:** multiple accounts compromised quickly via password spray
* **Risk:** high

## Finding 3 — Kerberoasting possible via service account SPN

* **Impact:** service ticket hash extracted for offline cracking
* **Risk:** high (often leads to escalation)

---

# MITRE ATT&CK Mapping

* **T1046** — Network Service Discovery
* **T1078** — Valid Accounts
* **T1110** — Brute Force (Password Spraying pattern)
* **T1558.003** — Steal or Forge Kerberos Tickets: Kerberoasting

---

# Remediation Summary

✅ Removed external exposure of:

* Kerberos (88)
* LDAP (389)
* SMB (445)

✅ Left only:

* RDP (3389) restricted to my IP for management

**Recommended “real world” upgrades:**

* Put DC in **private subnet**
* Use a **bastion host** or SSM Session Manager
* Enforce strong password policy + lockout thresholds
* Service account hygiene: long passwords, managed accounts, least privilege

---

# Next Steps (Privilege Escalation)

Now that initial access + Kerberoast hash extraction is proven, the next phase is:

1. **Offline cracking simulation** (controlled)
2. Validate whether svc-sql has elevated rights
3. Attempt lateral movement / privileged actions (lab-only)
4. Apply defensive controls:

   * service account hardening
   * password policy
   * remove unnecessary privileges
   * segmentation

> Next repo update will document the escalation chain and defenses applied.

---

## Disclaimer

This project was conducted in a controlled lab environment on systems I own and configured for testing.
No unauthorized systems were targeted.


