# AD-HYBRID-IDENTITY-ENTRA-CONNECT
Hybrid identity project extending on-premises Windows Server 2022 Active Directory to Microsoft Entra ID using Entra Connect — automated user provisioning, password hash sync, and account lifecycle management.

## Overview
Extended an on-premises Windows Server 2022 Active Directory environment 
to a hybrid identity architecture using Microsoft Entra Connect, 
demonstrating enterprise-grade identity synchronization and automated 
account lifecycle management.

## Business Value Delivered

| Outcome | Detail |
|---|---|
| ⚡ Sub-2-minute provisioning | New accounts created on-prem appear in cloud automatically |
| 🔒 Instant access revocation | Account disable on-prem blocks all Microsoft 365 access automatically |
| 🔄 Zero manual cloud steps | No admin intervention required in Entra after initial configuration |
| 📋 Bulk identity migration | 11 user UPNs migrated to routable domain via PowerShell in under 60 seconds |

## Environment

| Component | Details |
|---|---|
| Domain Controller | Windows Server 2022 — DC01 |
| Domain | techbridge.local |
| Cloud Tenant | Paccylab.onmicrosoft.com |
| Sync Tool | Microsoft Entra Connect v2.6.3.0 |
| Hypervisor | VirtualBox |

## What Was Implemented

- Hybrid identity bridge between on-premises AD and Microsoft Entra ID
- Password Hash Synchronization for seamless cloud authentication
- Automated user lifecycle management — provisioning and deprovisioning
- Bulk UPN migration via PowerShell for all existing AD users
- Dedicated sync service account following least-privilege principles
- Delta sync configured for near real-time identity updates

## Key Skills Demonstrated

- Microsoft Entra ID
- Hybrid Identity Architecture
- Microsoft Entra Connect
- Password Hash Sync
- Active Directory Administration
- PowerShell Automation
- Azure Identity Management
- Conditional Access Policy Management

## Results

### User Provisioning
Created a new user on-prem and triggered a delta sync.
The account appeared in Microsoft Entra ID in under 2 minutes
with zero manual steps required in the cloud.

### Account Deprovisioning
Disabled a user account on-prem via PowerShell.
After delta sync, the account showed as Disabled in Entra ID,
blocking all Microsoft 365 access automatically.

### Bulk UPN Migration
Migrated 11 user accounts from @techbridge.local to
@paccylab.onmicrosoft.com using a single PowerShell script
in under 60 seconds.

## Documentation
- [Setup Guide](setup-guide.md)
- [PowerShell Commands](powershell-commands.md)
- [Troubleshooting Scenarios](troubleshooting.md)
- [Screenshots](screenshots/)
