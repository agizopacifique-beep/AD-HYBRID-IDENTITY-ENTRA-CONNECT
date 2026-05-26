# Troubleshooting Guide
## Hybrid Identity — Microsoft Entra Connect

This document captures real issues encountered during the
implementation of this hybrid identity project, with exact
symptoms, root causes, resolutions, and lessons learned.

Every scenario in this guide was experienced and resolved
during this lab — not theoretical.

---

## Table of Contents
- [Scenario 1 — DC01 No Internet Access](#scenario-1--dc01-no-internet-access)
- [Scenario 2 — Users Not Syncing to Entra ID](#scenario-2--users-not-syncing-to-entra-id)
- [Scenario 3 — Entra Connect Authentication Blocked](#scenario-3--entra-connect-authentication-blocked)
- [Scenario 4 — Account Disable Not Propagating to Cloud](#scenario-4--account-disable-not-propagating-to-cloud)
- [Quick Diagnostic Reference](#quick-diagnostic-reference)

---

## Scenario 1 — DC01 No Internet Access

### Symptom
Browser on DC01 showing:
PowerShell connectivity test failing:
```powershell
Test-NetConnection -ComputerName portal.azure.com -Port 443
# WARNING: Name resolution of portal.azure.com failed
# PingSucceeded: False
```

---

### Root Cause
The NAT network adapter was present and correctly configured
in VirtualBox but had been disabled inside Windows Server's
Network Connections panel. The operating system was not
using the adapter even though the hypervisor showed it
as configured.

---

### Resolution

**Step 1 — Check adapter status inside DC01:**
1. Open Control Panel → Network and Internet →
   Network Connections
2. Look for the Ethernet adapter showing as disabled
   (greyed out icon)
3. Right-click → Enable
4. Internet access restored immediately — no reboot required

**Step 2 — Verify internet is now reachable:**
```powershell
Test-NetConnection -ComputerName portal.azure.com -Port 443
# Expected: TcpTestSucceeded : True
```

---

### Why This Happens
VirtualBox manages adapters at the hypervisor level.
Windows Server manages them at the OS level. These are
independent — a VirtualBox adapter can be fully configured
while Windows has it administratively disabled. Always
verify adapter status inside the guest OS, not just in
VirtualBox settings.

---

### Prevention
After adding or changing any network adapter in VirtualBox,
always verify inside the VM that the adapter is enabled
and has obtained an IP address before proceeding.

---

### Lesson Learned
Troubleshooting network issues requires checking every
layer independently — hypervisor configuration, OS adapter
status, IP assignment, DNS resolution, and firewall rules.
Assuming the hypervisor setting is sufficient is a common
mistake.

---

## Scenario 2 — Users Not Syncing to Entra ID

### Symptom
Entra Connect configuration completed successfully with
no errors but zero users appeared in Microsoft Entra ID
after the initial sync.

---

### Root Cause
All on-premises user accounts had UPNs using the private
domain suffix @techbridge.local:
Microsoft Entra ID only accepts UPNs that match a verified
domain registered in the tenant. The private .local suffix
is non-routable and unverifiable — Entra ID rejected all
sync objects silently.

---

### Resolution

**Step 1 — Add cloud domain as UPN suffix in AD:**
1. Open Server Manager → Tools →
   Active Directory Domains and Trusts
2. Right-click Active Directory Domains and Trusts
3. Click Properties
4. In Alternative UPN suffixes box type:
5. 5. Click Add → OK

**Step 2 — Bulk update all user UPNs via PowerShell:**
```powershell
Get-ADUser -Filter * -SearchBase "DC=techbridge,DC=local" |
ForEach-Object {
    $newUPN = $_.SamAccountName + "@paccylab.onmicrosoft.com"
    Set-ADUser $_ -UserPrincipalName $newUPN
    Write-Host "Updated: $newUPN"
}
```

**Step 3 — Force delta sync and verify:**
```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```

Then navigate to Entra Admin Center → Users → All Users
and confirm users appear with On-premises synced: Yes.

---

### Why This Happens
Entra ID is a cloud service that operates on verified
internet domains. Private Active Directory domains using
.local suffixes were never designed to be routable on
the internet. Hybrid identity requires all synced
identities to use a domain that Entra ID can verify
belongs to your tenant.

---

### Prevention
Before deploying Entra Connect, always audit all user
UPNs and confirm they use a routable domain suffix that
matches a verified domain in your Entra tenant.

---

### Lesson Learned
UPN alignment between on-premises AD and Entra ID is
the most common cause of sync failures in new hybrid
identity deployments. This should be the first thing
verified before installing Entra Connect.

---

## Scenario 3 — Entra Connect Authentication Blocked

### Symptom
Two sequential failures during Entra Connect sign-in:

**Failure 1 — JavaScript error:**
**Failure 2 — MFA prompt that could not be completed:**
---

### Root Cause
Two independent issues required separate fixes:

**Root Cause 1 — IE Enhanced Security Configuration**
Windows Server enables IE Enhanced Security Configuration
(IE ESC) by default to protect servers from malicious
websites. Entra Connect uses an embedded Internet Explorer
browser component to handle Microsoft authentication.
IE ESC strips JavaScript from all websites — including
Microsoft authentication endpoints — breaking the login
flow entirely.

**Root Cause 2 — Conditional Access MFA Policies**
When Security Defaults were disabled, Microsoft
automatically activated four Conditional Access policies
including Multifactor authentication for all users.
The sync service account (syncadmin) was subject to this
policy and could not complete the interactive MFA
registration challenge through the embedded browser.

---

### Resolution

**Fix 1 — Disable IE Enhanced Security Configuration:**
1. Open Server Manager → Local Server
2. Find IE Enhanced Security Configuration — shows as On
3. Click On
4. Set Administrators to Off
5. Set Users to Off
6. Click OK

**Fix 2 — Exclude sync account from MFA policies:**
1. Go to https://entra.microsoft.com
2. Navigate to Conditional Access → Policies
3. Open Multifactor authentication for all users:
   - Click Users → Exclude tab
   - Search and select syncadmin@paccylab.onmicrosoft.com
   - Save
4. Repeat for Multifactor authentication for admins

**Fix 3 — Retry Entra Connect authentication:**
1. Return to Entra Connect wizard
2. Click Previous then Next to reload the sign-in prompt
3. Sign in with syncadmin credentials
4. Login completes successfully

---

### Why This Happens
IE ESC is a Windows Server security feature designed to
protect interactive server sessions. It was never designed
with embedded authentication flows in mind. Entra Connect's
embedded browser inherits these restrictions.

MFA Conditional Access policies are critical security
controls for human accounts but are incompatible with
automated service accounts that authenticate without
human interaction. Service accounts must always be
explicitly excluded from MFA requirements.

---

### Prevention
Before installing Entra Connect on any Windows Server:
1. Disable IE Enhanced Security Configuration
2. Create and configure the sync service account
3. Verify the service account is excluded from all
   MFA Conditional Access policies
4. Test sign-in from a browser before running the wizard

---

### Lesson Learned
Compound failures — where two separate root causes must
both be resolved — require systematic layer-by-layer
diagnosis. Fixing only the JavaScript error would have
exposed the MFA issue. Fixing only the MFA issue without
addressing IE ESC would have left authentication broken.
Always resolve each layer completely before moving to the next.

---

## Scenario 4 — Account Disable Not Propagating to Cloud

### Symptom
User account disabled in on-premises Active Directory
via PowerShell but account still showing as active in
Microsoft Entra ID after several minutes.

```powershell
Disable-ADAccount -Identity tuser2
# No output — command succeeded
# But Entra ID still shows account as active
```

---

### Root Cause
Microsoft Entra Connect runs on a default synchronization
schedule of every 30 minutes. Disabling an account in AD
does not immediately push the change to Entra ID — it
waits for the next scheduled sync cycle.

In time-sensitive offboarding scenarios, a 30-minute
delay before cloud access is revoked is operationally
unacceptable and a potential security risk.

---

### Resolution

**Force immediate delta sync after any offboarding action:**
```powershell
# Step 1 — Disable the account
Disable-ADAccount -Identity tuser2

# Step 2 — Force immediate sync
Start-ADSyncSyncCycle -PolicyType Delta

# Step 3 — Verify in AD
Get-ADUser -Identity tuser2 -Properties Enabled |
Select-Object Name, Enabled
```

Then verify in Entra Admin Center:
1. Navigate to Users → All Users
2. Click the disabled user
3. Confirm Account status: Disabled

---

### Why This Happens
Entra Connect is designed for efficiency — running
continuous sync would consume significant resources.
The 30-minute default schedule is a balance between
near-real-time sync and system performance. Delta sync
only processes changes since the last cycle, making
it fast and low-impact when triggered manually.

---

### Prevention
Include forced delta sync as a mandatory step in all
offboarding runbooks:

---

### Lesson Learned
Default sync schedules are designed for normal operations,
not emergency offboarding. Every IT team operating a
hybrid identity environment should have a documented
offboarding procedure that includes forcing a delta sync
immediately after disabling any account.

---

## Quick Diagnostic Reference

| Symptom | First Check | Command |
|---|---|---|
| DC01 no internet | Check NAT adapter enabled in Network Connections | `Test-NetConnection -ComputerName portal.azure.com -Port 443` |
| Users not in Entra | Check UPN suffixes | `Get-ADUser -Filter * -Properties UserPrincipalName` |
| Sync not running | Check scheduler | `Get-ADSyncScheduler` |
| Sync errors | Check connector errors | `Get-ADSyncConnectorRunStatus` |
| Auth failing in wizard | Check IE ESC and MFA policies | Server Manager → Local Server |
| Account disable not syncing | Force delta sync | `Start-ADSyncSyncCycle -PolicyType Delta` |
| Ping to DC01 failing | Check firewall | `netsh advfirewall firewall show rule name=all` |
| No IP on NAT adapter | Check adapter status | `Get-NetAdapter` |

---

## Related Documentation
- [Setup Guide](setup-guide.md)
- [PowerShell Commands Reference](powershell-commands.md)
- [Screenshots](screenshots/)
- 
