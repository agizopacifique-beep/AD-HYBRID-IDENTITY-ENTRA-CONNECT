# Setup Guide — Hybrid Identity with Microsoft Entra Connect

## Overview
This guide documents the complete implementation process for 
extending an on-premises Windows Server 2022 Active Directory 
environment to Microsoft Entra ID using Microsoft Entra Connect.

---

## Prerequisites

| Requirement | Details |
|---|---|
| Domain Controller | Windows Server 2022 with AD DS installed |
| Domain | techbridge.local fully configured |
| Azure Account | Microsoft account with Entra ID tenant |
| Internet Access | DC01 must reach portal.azure.com and login.microsoftonline.com |
| PowerShell | Run as Administrator on DC01 |
| VirtualBox | Dual adapter — NAT + Host-Only |

---

## Phase 1 — Verify Network Connectivity

Before starting, confirm DC01 can reach both the local 
network and the internet simultaneously.

### Verify host can reach DC01
```powershell
ping 192.168.56.10
```
Expected: 4 replies, 0% packet loss

### Verify DC01 can reach the internet
```powershell
Test-NetConnection -ComputerName portal.azure.com -Port 443
```
Expected: TcpTestSucceeded : True

### If DC01 has no internet access
1. Open Control Panel → Network and Internet → Network Connections
2. Check if the NAT adapter (Ethernet) is disabled
3. Right-click → Enable
4. Re-run the connectivity test

> **Note:** DC01 requires two active adapters simultaneously:
> - Adapter 1 (NAT): Internet access via host machine
> - Adapter 2 (Host-Only): Local lab network 192.168.56.0/24

---

## Phase 2 — Prepare Azure Tenant

### Step 1 — Create Microsoft Entra ID Tenant
1. Go to https://azure.microsoft.com/en-us/free
2. Sign in with a Microsoft account
3. Complete tenant setup
4. Note your tenant domain: yourname.onmicrosoft.com

### Step 2 — Create Dedicated Sync Service Account
1. Go to https://entra.microsoft.com
2. Navigate to Users → New User → Create new user
3. Fill in:

| Field | Value |
|---|---|
| User principal name | syncadmin@paccylab.onmicrosoft.com |
| Display name | EntraConnectSync |
| Password | Strong password — write it down |

4. Click Assignments tab → Add role → Global Administrator
5. Click Review + create → Create

> **Why a dedicated account:** Automated sync services must
> never share credentials with human admin accounts. A dedicated
> service account ensures sync continues even if admin passwords
> change, and provides a clean audit trail.

### Step 3 — Disable Security Defaults
1. Go to Entra Admin Center → Identity → Overview → Properties
2. Scroll to bottom → Manage Security Defaults
3. Set to Disabled
4. Reason: My organization uses Conditional Access

### Step 4 — Exclude Sync Account from MFA Policies
1. Go to Entra Admin Center → Conditional Access → Policies
2. Open each MFA policy:
   - Multifactor authentication for all users
   - Multifactor authentication for admins
3. For each policy:
   - Click Users → Exclude tab
   - Add syncadmin@paccylab.onmicrosoft.com
   - Save

---

## Phase 3 — Prepare On-Premises Active Directory

### Step 5 — Add UPN Suffix

**Why:** On-premises users have @techbridge.local UPNs which
Entra ID cannot accept. Adding the cloud domain as a UPN
suffix allows users to be updated to a routable identity.

1. On DC01 open Server Manager → Tools →
   Active Directory Domains and Trusts
2. Right-click Active Directory Domains and Trusts at top
3. Click Properties
4. In Alternative UPN suffixes box type:
5. 5. Click Add → OK

### Step 6 — Bulk Update All User UPNs

**Why:** Adding the suffix registers it as valid but does
not update existing users. This PowerShell command updates
every user in the domain simultaneously.

```powershell
Get-ADUser -Filter * -SearchBase "DC=techbridge,DC=local" |
ForEach-Object {
    $newUPN = $_.SamAccountName + "@paccylab.onmicrosoft.com"
    Set-ADUser $_ -UserPrincipalName $newUPN
    Write-Host "Updated: $newUPN"
}
```

Expected output — each user printed as updated:
---

## Phase 4 — Install and Configure Entra Connect

### Step 7 — Download Entra Connect

1. Go to https://entra.microsoft.com
2. Navigate to Identity → Hybrid Management →
   Microsoft Entra Connect → Connect Sync
3. Click the Access tab
4. Download the installer
5. Save to DC01 desktop

> **Note:** As of early 2026, Entra Connect is no longer
> available from the Microsoft Download Center. Download
> only from the Entra Admin Center.

### Step 8 — Disable IE Enhanced Security Configuration

**Why:** Entra Connect uses an embedded Internet Explorer
browser component to handle Microsoft authentication.
IE Enhanced Security Configuration strips JavaScript from
all sites — including Microsoft login pages — breaking
the authentication flow entirely.

1. Open Server Manager → Local Server
2. Find IE Enhanced Security Configuration — click On
3. Set Administrators to Off
4. Set Users to Off
5. Click OK

### Step 9 — Run the Entra Connect Installer

1. Double-click the downloaded installer on DC01
2. Accept the license agreement
3. Click Use express settings

### Step 10 — Connect to Microsoft Entra ID

1. Enter:
2. 2. Enter the DC01 Administrator password
3. Click Next

### Step 12 — Complete Configuration

1. Review the configuration summary
2. Click Install
3. Wait for configuration to complete
4. Click Exit when Configuration complete screen appears

> **The first full sync runs automatically on completion.**
> All AD users will begin appearing in Entra ID immediately.

---

## Phase 5 — Validate Synchronization

### Step 13 — Verify Users in Entra ID

1. Go to https://entra.microsoft.com
2. Navigate to Users → All Users
3. Confirm AD users appear with On-premises synced: Yes

### Step 14 — Force Delta Sync

```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

### Step 15 — Test User Provisioning

```powershell
# Create test user on-prem
New-ADUser `
    -Name "Test User" `
    -SamAccountName "tuser2" `
    -UserPrincipalName "tuser2@paccylab.onmicrosoft.com" `
    -AccountPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force) `
    -Enabled $true

# Force immediate sync
Start-ADSyncSyncCycle -PolicyType Delta
```

Then check Entra ID — user should appear in under 2 minutes.

### Step 16 — Test Account Deprovisioning

```powershell
# Disable user on-prem
Disable-ADAccount -Identity tuser2

# Force immediate sync
Start-ADSyncSyncCycle -PolicyType Delta
```

Then check Entra ID → click the user → confirm
Account status: Disabled

---

## Expected Results

| Test | Expected Result |
|---|---|
| Users in Entra | All AD users visible with On-premises synced: Yes |
| Provisioning speed | New user appears in Entra in under 2 minutes |
| Deprovisioning | Disabled on-prem shows Disabled in Entra automatically |
| Manual cloud steps | Zero — all changes flow from on-premises AD |

---

## Related Documentation
- [PowerShell Commands Reference](powershell-commands.md)
- [Troubleshooting Guide](troubleshooting.md)
- [Screenshots](screenshots/)
- 
