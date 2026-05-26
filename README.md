# AD-HYBRID-IDENTITY-ENTRA-CONNECT

![Windows Server](https://img.shields.io/badge/Windows_Server-2022-0078D6?style=for-the-badge&logo=windows&logoColor=white)
![Microsoft Azure](https://img.shields.io/badge/Microsoft_Azure-0089D6?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![Microsoft Entra ID](https://img.shields.io/badge/Microsoft_Entra_ID-0078D4?style=for-the-badge&logo=microsoft&logoColor=white)
![PowerShell](https://img.shields.io/badge/PowerShell-5391FE?style=for-the-badge&logo=powershell&logoColor=white)
![Active Directory](https://img.shields.io/badge/Active_Directory-003399?style=for-the-badge&logo=windows&logoColor=white)
![Security+](https://img.shields.io/badge/CompTIA-Security%2B-FF0000?style=for-the-badge&logo=comptia&logoColor=white)
![AZ-104](https://img.shields.io/badge/Microsoft-AZ--104_In_Progress-0089D6?style=for-the-badge&logo=microsoft&logoColor=white)

---

# On-Premises Active Directory to Cloud Hybrid Identity

### Extending Enterprise Identity Infrastructure to Microsoft Entra ID
#### Microsoft Entra Connect | Password Hash Sync | Automated Lifecycle Management

---

## Table of Contents

- [Project Overview](#project-overview)
- [Business Use Case](#business-use-case)
- [Objectives](#objectives)
- [Technologies Used](#technologies-used)
- [Lab Environment](#lab-environment)
- [Architecture Diagram](#architecture-diagram)
- [Implementation Steps](#implementation-steps)
- [Security Considerations](#security-considerations)
- [PowerShell Commands](#powershell-commands)
- [Challenges and Troubleshooting](#challenges-and-troubleshooting)
- [Key Skills Demonstrated](#key-skills-demonstrated)
- [Screenshots](#screenshots)
- [Lessons Learned](#lessons-learned)
- [Future Improvements](#future-improvements)
- [Resume Summary](#resume-summary)
- [Recruiter Impact Statement](#recruiter-impact-statement)
- [Contact](#contact)

---

## Project Overview

This project demonstrates the end-to-end design and implementation
of a hybrid identity architecture that bridges an on-premises
Windows Server 2022 Active Directory environment with Microsoft
Entra ID (formerly Azure AD) using Microsoft Entra Connect.

Hybrid identity is the backbone of modern enterprise IT.
Organizations do not operate purely on-premises or purely in the
cloud — they operate in both simultaneously. This project replicates
that real-world enterprise reality, where on-premises Active Directory
remains the authoritative identity source while Microsoft Entra ID
extends that identity into the cloud, enabling access to Microsoft
365, Azure resources, and SaaS applications through a single
synchronized identity.

> **Key Result:** User accounts created in on-premises Active
> Directory automatically appeared in Microsoft Entra ID in under
> 2 minutes, with zero manual intervention in the cloud. Account
> disables propagated to the cloud automatically, blocking all
> Microsoft 365 access instantly.

---

## Business Use Case

In enterprise environments, hybrid identity solves a critical
operational challenge: users need seamless access to both
on-premises resources (file servers, printers, internal
applications) and cloud resources (Microsoft 365, Azure,
SharePoint Online, Teams) using a single set of credentials.

**Real-world scenarios this project addresses:**

| Scenario | Business Impact |
|---|---|
| Employee onboarding | HR creates one AD account — user gets M365 access automatically |
| Employee offboarding | Disable one AD account — all cloud access revoked instantly |
| Password management | One password works everywhere — on-prem and cloud |
| Identity governance | Single source of truth for all user identities |
| Compliance and audit | Centralized identity management simplifies audit reporting |
| Access control | Conditional Access enforces security policies at sign-in |

---

## Objectives

- [x] Deploy and configure Windows Server 2022 as a domain controller
- [x] Establish Active Directory Domain Services with users and OUs
- [x] Create a Microsoft Entra ID tenant for cloud identity
- [x] Configure UPN suffixes to align on-premises and cloud domains
- [x] Bulk migrate existing user UPNs via PowerShell automation
- [x] Deploy and configure Microsoft Entra Connect with Express Settings
- [x] Implement Password Hash Synchronization
- [x] Create dedicated sync service account following least-privilege
- [x] Validate sub-2-minute user provisioning from AD to Entra ID
- [x] Validate automated access revocation via account disable
- [x] Configure Conditional Access policies and MFA exclusions
- [x] Document and resolve real-world implementation challenges

---

## Technologies Used

| Category | Technology |
|---|---|
| On-Premises OS | Windows Server 2022 Standard Evaluation |
| Client OS | Windows 10 Pro |
| Directory Services | Active Directory Domain Services (AD DS) |
| Cloud Identity | Microsoft Entra ID (formerly Azure AD) |
| Sync Engine | Microsoft Entra Connect v2.6.3.0 |
| Sync Method | Password Hash Synchronization (PHS) |
| Cloud Portal | Microsoft Azure Portal / Entra Admin Center |
| Scripting and Automation | PowerShell 5.1 |
| Hypervisor | Oracle VirtualBox |
| Networking | VirtualBox Host-Only + NAT Adapters |
| DNS | Windows Server DNS + Google DNS Forwarders |
| Security Policies | Conditional Access, MFA, Security Defaults |
| Certifications | CompTIA Security+ (Certified) / AZ-104 (In Progress) |

---

## Lab Environment

| Component | Details |
|---|---|
| Domain Controller | DC01 — Windows Server 2022 — 192.168.56.10 |
| On-Premises Domain | techbridge.local |
| Cloud Tenant | Paccylab.onmicrosoft.com |
| Client Machine | PaccySOClab — Windows 10 Pro — Domain Joined |
| Hypervisor | Oracle VirtualBox |
| Network — Adapter 1 | NAT — Internet connectivity via host WiFi |
| Network — Adapter 2 | Host-Only — Lab network 192.168.56.0/24 |
| DNS | Internal AD DNS + External Forwarders 8.8.8.8 / 8.8.4.4 |
| Sync Service Account | syncadmin@paccylab.onmicrosoft.com |
| AD Users Synced | 11 on-premises accounts |

---

## Architecture Diagram
┌─────────────────────────────────┐       ┌──────────────────────────────────┐
│     ON-PREMISES ENVIRONMENT     │       │       MICROSOFT AZURE CLOUD      │
│                                 │       │                                  │
│  ┌───────────────────────────┐  │       │  ┌────────────────────────────┐  │
│  │           DC01            │  │       │  │     Microsoft Entra ID     │  │
│  │   Windows Server 2022     │  │ SYNC  │  │   Paccylab.onmicrosoft.com │  │
│  │   techbridge.local        │◄─┼───────┼─►│                            │  │
│  │   AD DS + DNS             │  │       │  │  ✓ Users Synced            │  │
│  └───────────────────────────┘  │       │  │  ✓ Passwords Synced        │  │
│              │                  │       │  │  ✓ Status Synced           │  │
│  ┌───────────▼───────────────┐  │       │  └────────────────────────────┘  │
│  │     Entra Connect         │  │       │              │                   │
│  │     v2.6.3.0              │──┼───────┼──►  Microsoft 365               │
│  │  Password Hash Sync       │  │       │     Teams / SharePoint           │
│  └───────────────────────────┘  │       │     Azure Resources              │
│                                 │       │                                  │
│  ┌───────────────────────────┐  │       │                                  │
│  │       PaccySOClab         │  │       │                                  │
│  │     Windows 10 Pro        │  │       │                                  │
│  │     Domain Joined         │  │       │                                  │
│  └───────────────────────────┘  │       │                                  │
└─────────────────────────────────┘       └──────────────────────────────────┘
192.168.56.0/24                          Cloud Tenant
---

## Implementation Steps

### Phase 1 — On-Premises Infrastructure

**Step 1 — Domain Controller Configuration**
- Deployed Windows Server 2022 in VirtualBox
- Configured static IP: 192.168.56.10 / Subnet: 255.255.255.0
- Installed Active Directory Domain Services role via Server Manager
- Promoted server to domain controller for techbridge.local
- Configured DNS server with internal zone and external forwarders

**Step 2 — Active Directory User Provisioning**
- Created Organizational Units for structured identity management
- Provisioned 11 user accounts including:
  jsmith, bjones, mwilliams, sconnor, edavis, jwilson, ltaylor
- Configured Windows 10 Pro client (PaccySOClab) joined to domain

**Step 3 — Network Configuration**
- Configured VirtualBox dual-adapter architecture:
  - Adapter 1: NAT — internet via host WiFi
  - Adapter 2: Host-Only — lab network 192.168.56.0/24
- Verified DC01 reachability and internet connectivity
- Added Google DNS forwarders for external name resolution

---

### Phase 2 — Cloud Identity Preparation

**Step 4 — Microsoft Entra ID Tenant Setup**
- Created Microsoft Entra ID tenant: Paccylab.onmicrosoft.com
- Configured tenant-level settings for hybrid identity readiness

**Step 5 — Dedicated Sync Service Account**
- Created syncadmin@paccylab.onmicrosoft.com
- Assigned Global Administrator role
- Excluded from all MFA Conditional Access policies
- Configured as a non-interactive service account

**Step 6 — UPN Suffix Alignment**
- Registered paccylab.onmicrosoft.com as alternate UPN suffix
  in Active Directory Domains and Trusts on DC01
- Required to make on-premises identities routable to Entra ID

---

### Phase 3 — Identity Synchronization

**Step 7 — Bulk UPN Migration via PowerShell**
- Migrated all 11 users from @techbridge.local to
  @paccylab.onmicrosoft.com using a single PowerShell script
- Completed in under 60 seconds with zero errors

**Step 8 — Entra Connect Deployment**
- Downloaded Entra Connect v2.6.3.0 from Entra Admin Center
- Installed on DC01 using Express Settings
- Authenticated cloud side with syncadmin credentials
- Authenticated on-premises side with TECHBRIDGE\Administrator
- Initial full synchronization completed automatically

**Step 9 — Sync Validation**
- Forced delta sync via PowerShell
- Verified all 11 AD users appeared in Entra ID within 2 minutes
- Confirmed On-premises synced: Yes for all AD accounts

---

### Phase 4 — Lifecycle Management Validation

**Step 10 — Provisioning Test**
- Created new user tuser2 in on-premises AD via PowerShell
- Triggered immediate delta sync
- User appeared in Entra ID in under 2 minutes
- Zero manual steps required in the cloud

**Step 11 — Deprovisioning Test**
- Disabled tuser2 in on-premises AD via PowerShell
- Triggered delta sync
- Account status changed to Disabled in Entra ID
- All Microsoft 365 access automatically blocked

---

## Security Considerations

### Least Privilege
The dedicated sync service account was created exclusively
for Entra Connect authentication. It performs no interactive
logins, has no mailbox, and exists solely to authenticate
the sync engine — following least privilege for service accounts.

### Conditional Access and MFA
When Security Defaults were disabled, Microsoft automatically
activated four Conditional Access policies including MFA
enforcement for all users and admins. The sync service account
was explicitly excluded to allow unattended automated
authentication while maintaining full MFA for human accounts.

### Password Hash Synchronization Security
PHS transmits only a hash of a hash — never plaintext
passwords. All synchronization occurs over TLS-encrypted
connections. Microsoft stores only the processed hash.

### Role-Based Access Control
Global Administrator was assigned to the sync service account
with the understanding that in production this would be
scoped to the minimum required permissions using the
Hybrid Identity Administrator role.

### Identity Protection
AD remains the single source of truth. All identity changes
flow from on-premises governance processes before reflecting
in the cloud — preventing unauthorized cloud-side modifications.

### Audit Logging
All sync activity is logged in the Entra Connect event log
on DC01 and in Microsoft Entra ID audit logs, providing
a complete audit trail across both environments.

---

## PowerShell Commands

### Add UPN Suffix to Active Directory
```powershell
# Register cloud-routable UPN suffix in on-premises AD
Set-ADForest -UPNSuffixes @{Add="paccylab.onmicrosoft.com"}
```

### Bulk UPN Migration — 11 Users in Under 60 Seconds
```powershell
Get-ADUser -Filter * -SearchBase "DC=techbridge,DC=local" |
ForEach-Object {
    $newUPN = $_.SamAccountName + "@paccylab.onmicrosoft.com"
    Set-ADUser $_ -UserPrincipalName $newUPN
    Write-Host "Updated: $newUPN"
}
```

### Create Test User for Sync Validation
```powershell
New-ADUser `
    -Name "Test User" `
    -GivenName "Test" `
    -Surname "User" `
    -SamAccountName "tuser2" `
    -UserPrincipalName "tuser2@paccylab.onmicrosoft.com" `
    -AccountPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force) `
    -Enabled $true
```

### Force Immediate Delta Sync
```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

### Verify Sync Scheduler Status
```powershell
Get-ADSyncScheduler
```

### Disable User — Offboarding
```powershell
Disable-ADAccount -Identity tuser2
```

### Allow ICMP for Network Diagnostics
```powershell
netsh advfirewall firewall add rule `
    name="Allow ICMPv4-In" `
    protocol=icmpv4:8,any `
    dir=in action=allow
```

### Configure External DNS Forwarders
```powershell
Add-DnsServerForwarder -IPAddress 8.8.8.8
Add-DnsServerForwarder -IPAddress 8.8.4.4
```

---

## Challenges and Troubleshooting

### Challenge 1 — DC01 No Internet Access
**Symptom:** Browser on DC01 showing DNS_PROBE_FINISHED_NO_INTERNET

**Root Cause:** NAT network adapter was present in VirtualBox
but had been disabled inside Windows Server Network Connections

**Resolution:**
Control Panel → Network and Internet → Network Connections →
found Ethernet adapter disabled → right-clicked → Enable.
Internet access restored immediately.

**Lesson:** Always verify adapter status inside the guest OS.
VirtualBox can show an adapter configured while Windows has
it disabled internally.

---

### Challenge 2 — Users Not Syncing to Entra ID
**Symptom:** Entra Connect ran with no errors but zero
users appeared in Entra ID

**Root Cause:** All user UPNs using @techbridge.local —
a non-routable private domain rejected by Entra ID

**Resolution:**
Added paccylab.onmicrosoft.com as alternate UPN suffix in
Active Directory Domains and Trusts, then bulk updated
all 11 accounts via PowerShell in under 60 seconds.

**Lesson:** Entra ID only accepts UPNs matching a verified
tenant domain. Planning UPN alignment before deployment
is essential in any hybrid identity project.

---

### Challenge 3 — Entra Connect Authentication Blocked
**Symptom:** Login popup showed JavaScript error then
MFA prompt that could not be completed

**Root Cause:**
1. IE Enhanced Security Configuration stripping JavaScript
   from Microsoft authentication endpoints
2. Conditional Access MFA policies blocking service account

**Resolution:**
1. Disabled IE ESC via Server Manager → Local Server
2. Excluded syncadmin from MFA Conditional Access policies
   in Entra Admin Center

**Lesson:** Service accounts must be excluded from
interactive security policies before Entra Connect
deployment. IE ESC is a common Windows Server blocker
for embedded browser authentication flows.

---

### Challenge 4 — Account Disable Not Reflecting in Cloud
**Symptom:** User disabled in AD but still active in Entra

**Root Cause:** Default sync cycle runs every 30 minutes

**Resolution:**
```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```
Account reflected as Disabled in Entra within seconds.

**Lesson:** For urgent offboarding, always force a delta
sync immediately. Include this as a mandatory step in
all offboarding runbooks.

---

## Key Skills Demonstrated

**Identity and Access Management**
Microsoft Entra ID · Hybrid Identity · Entra Connect
Password Hash Sync · Conditional Access · MFA Management
Service Account Governance · Least Privilege Design

**Active Directory Administration**
Windows Server 2022 · AD DS · Domain Controller Setup
UPN Management · User Lifecycle · DNS Administration

**Cloud Administration**
Microsoft Azure · Entra Admin Center · Tenant Configuration
Security Defaults · Cloud Identity Management

**PowerShell Automation**
Bulk identity migration · Automated provisioning
Delta sync triggers · AD management · Status verification

**Networking and Infrastructure**
Dual-adapter hypervisor configuration · DNS troubleshooting
Hybrid network architecture · NAT and Host-Only networking

**Security**
CompTIA Security+ Certified · Conditional Access design
MFA policy management · Audit logging · Identity protection

**Troubleshooting**
Systematic multi-layer diagnosis · Authentication failures
Network connectivity · Hypervisor networking · Sync failures

---

## Screenshots

| Screenshot | Description |
|---|---|
| ![Tenant Overview](screenshots/azure-tenant-overview.png) | Entra ID tenant — Paccylab.onmicrosoft.com |
| ![Sync Admin](screenshots/sync-admin-account.png) | Dedicated sync service account in Entra |
| ![UPN Suffix](screenshots/upn-suffix-added.png) | Cloud UPN suffix added to on-premises AD |
| ![Bulk Update](screenshots/upn-bulk-update-powershell.png) | PowerShell bulk UPN migration — 11 accounts |
| ![Install](screenshots/entra-connect-install.png) | Entra Connect installer running on DC01 |
| ![Configured](screenshots/entra-connect-configured.png) | Configuration complete — sync initiated |
| ![Sync Status](screenshots/sync-status-powershell.png) | Delta sync triggered and verified |
| ![Users Synced](screenshots/users-synced-entra.png) | AD users visible in Microsoft Entra ID |
| ![User Created](screenshots/user-created-on-prem.png) | New user created in on-premises AD |
| ![User Appeared](screenshots/user-appeared-cloud.png) | User appeared in Entra in under 2 minutes |
| ![User Disabled](screenshots/user-disabled-blocked.png) | Disabled on-prem — blocked in cloud |

---

## Lessons Learned

**Hybrid identity requires upfront planning.**
UPN alignment, service account design, network validation,
and security policy configuration must all be planned before
deployment — not discovered during implementation.

**Source of truth architecture is everything.**
When AD is the single source of truth, provisioning and
deprovisioning become predictable, auditable, and scalable.

**Service accounts are a unique security category.**
Automated services cannot complete MFA challenges.
Designing exclusions before deploying security policies
is essential operational practice.

**Troubleshooting requires systematic layer isolation.**
The most complex issue had two independent root causes.
Fixing one without the other would have left the problem
partially unsolved. Methodical isolation of each layer
is what resolves compound failures.

**Delta sync is a critical operational skill.**
Waiting 30 minutes for a sync cycle during an urgent
offboarding is operationally unacceptable. Knowing how
to force and verify an immediate sync is essential.

---

## Future Improvements

| Improvement | Business Value |
|---|---|
| Microsoft Intune Integration | Extend hybrid identity to device management and MDM |
| Granular Conditional Access | Location, device, and risk-based access policies |
| Microsoft Defender for Cloud | Cloud security posture management and threat protection |
| Microsoft Sentinel SIEM | Centralized security monitoring and incident response |
| Privileged Identity Management | Just-in-time admin access and privileged role governance |
| Zero Trust Architecture | Never trust, always verify across all resources |
| Single Sign-On (SSO) | Seamless access across all enterprise applications |
| Azure AD Application Proxy | Secure remote access to on-premises applications |
| Hybrid Azure AD Join | Extend device identity management to on-premises machines |
| Password Writeback | Enable self-service password reset from the cloud |

---

## Resume Summary

Designed and implemented a hybrid identity architecture
connecting an on-premises Windows Server 2022 Active
Directory environment to Microsoft Entra ID using Microsoft
Entra Connect with Password Hash Synchronization. Achieved
sub-2-minute automated user provisioning from on-premises
AD to the cloud with zero manual steps required in Entra
after initial configuration. Validated that account disables
on-premises automatically block all Microsoft 365 access.
Bulk migrated 11 user UPNs to a cloud-routable domain via
PowerShell in under 60 seconds. Diagnosed and resolved
real-world challenges including network adapter
misconfiguration, UPN suffix mismatches, IE Enhanced
Security Configuration blocking authentication, and
Conditional Access MFA policies blocking service account
sync. Holds CompTIA Security+ and is actively pursuing
Microsoft AZ-104.

**ATS Keywords:** Microsoft Entra ID · Azure AD ·
Hybrid Identity · Entra Connect · Active Directory ·
Password Hash Sync · Windows Server 2022 · PowerShell ·
Conditional Access · MFA · Identity and Access Management ·
Cloud Administration · System Administration ·
Technical Support · CompTIA Security+

---

## Recruiter Impact Statement

> This project demonstrates hands-on experience with the
> exact identity infrastructure that enterprises depend on
> every day. Every major organization running Microsoft 365
> operates a hybrid identity environment. The skills shown
> here — deploying Entra Connect, managing sync cycles,
> configuring Conditional Access, automating identity
> lifecycle with PowerShell, and resolving real authentication
> failures — are the same skills required on day one in a
> Technical Support, Systems Administrator, or Cloud Support
> role. This is not theoretical knowledge. Every step was
> implemented, tested, broken, fixed, and documented in a
> fully functional lab environment built from scratch.

---

## Acknowledgments

- Microsoft Documentation — Entra Connect deployment guides
- Microsoft Learn — Hybrid Identity and AZ-104 learning paths
- Oracle VirtualBox Documentation — Network configuration

---

## Contact

**Pacifique Agizo**
*Technical Support | Systems Administrator | Cloud Support*

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Pacifique_Agizo-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/agizopacifique)
[![GitHub](https://img.shields.io/badge/GitHub-agizopacifique--beep-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/agizopacifique-beep)

*Open to Technical Support, Systems Administrator,
and Cloud Support opportunities.*

---

*Every result documented in this project is real,
tested, and reproducible.*
