# PowerShell Commands Reference
## Hybrid Identity — Microsoft Entra Connect

This document contains all PowerShell commands used in the
hybrid identity implementation, organized by category with
explanations of what each command does and why it is used
in an enterprise context.

---

## Table of Contents
- [Network Diagnostics](#network-diagnostics)
- [Active Directory User Management](#active-directory-user-management)
- [UPN Configuration](#upn-configuration)
- [Entra Connect Sync Management](#entra-connect-sync-management)
- [Firewall Configuration](#firewall-configuration)
- [DNS Configuration](#dns-configuration)
- [Offboarding and Lifecycle Management](#offboarding-and-lifecycle-management)

---

## Network Diagnostics

### Test connectivity to DC01
```powershell
ping 192.168.56.10
```
**Purpose:** Verify host machine can reach the domain
controller before beginning any configuration. 100% packet
loss indicates a network adapter or firewall issue.

---

### Test internet connectivity from DC01
```powershell
Test-NetConnection -ComputerName portal.azure.com -Port 443
```
**Purpose:** Confirm DC01 can reach Microsoft Azure over
HTTPS. Entra Connect requires this connection to push
identity data to the cloud. TcpTestSucceeded must be True.

---

### View all network adapters and their status
```powershell
Get-NetAdapter | Format-Table Name, Status, InterfaceIndex
```
**Purpose:** Identify all network adapters on DC01 and
confirm which are active. Used to diagnose connectivity
issues when DC01 cannot reach the internet or the host network.

---

### View full IP configuration
```powershell
ipconfig /all
```
**Purpose:** Confirm IP addresses, subnet masks, default
gateways, and DNS server assignments for all adapters.
DC01 should show both 192.168.56.10 (Host-Only) and
a 10.x.x.x address (NAT) when both adapters are active.

---

## Active Directory User Management

### List all AD users with UPN
```powershell
Get-ADUser -Filter * -Properties UserPrincipalName |
Select-Object Name, SamAccountName, UserPrincipalName |
Format-Table -AutoSize
```
**Purpose:** Audit all user accounts and their current UPNs
before and after migration. Use this to verify the bulk
update completed correctly.

---

### Create a new AD user
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
**Purpose:** Provision a new user in on-premises AD to test
the hybrid identity sync cycle. The UPN must use the
cloud-routable suffix for Entra ID to accept the account.

---

### Disable a user account — offboarding
```powershell
Disable-ADAccount -Identity tuser2
```
**Purpose:** Simulate employee offboarding by disabling
the on-premises account. After a delta sync, this status
propagates to Entra ID and blocks all Microsoft 365 access
automatically. This is the single most important offboarding
action in a hybrid identity environment.

---

### Enable a user account
```powershell
Enable-ADAccount -Identity tuser2
```
**Purpose:** Re-enable a disabled account. Status propagates
to Entra ID on the next sync cycle.

---

### Delete a user account
```powershell
Remove-ADUser -Identity tuser2 -Confirm:$false
```
**Purpose:** Permanently remove a user from AD. The account
will be removed from Entra ID on the next sync cycle.

---

### Reset a user password
```powershell
Set-ADAccountPassword -Identity jsmith `
    -NewPassword (ConvertTo-SecureString "NewPassword123!" -AsPlainText -Force) `
    -Reset
```
**Purpose:** Reset a user password on-premises. With
Password Hash Synchronization enabled, the new password
hash syncs to Entra ID automatically — no separate cloud
password reset required.

---

## UPN Configuration

### Add UPN suffix to Active Directory forest
```powershell
Set-ADForest -UPNSuffixes @{Add="paccylab.onmicrosoft.com"}
```
**Purpose:** Register the cloud tenant domain as a valid
UPN suffix in on-premises AD. Required before updating
any user UPNs. Without this step, the suffix is not
recognized as valid by Active Directory.

---

### Bulk update all user UPNs
```powershell
Get-ADUser -Filter * -SearchBase "DC=techbridge,DC=local" |
ForEach-Object {
    $newUPN = $_.SamAccountName + "@paccylab.onmicrosoft.com"
    Set-ADUser $_ -UserPrincipalName $newUPN
    Write-Host "Updated: $newUPN"
}
```
**Purpose:** Migrate all existing users from the private
@techbridge.local suffix to the cloud-routable
@paccylab.onmicrosoft.com suffix. Entra ID rejects UPNs
using non-routable private domain suffixes. This command
updates all accounts simultaneously in under 60 seconds
regardless of user count.

---

### Verify UPN update completed
```powershell
Get-ADUser -Filter * -Properties UserPrincipalName |
Where-Object {$_.UserPrincipalName -like "*paccylab*"} |
Select-Object Name, UserPrincipalName
```
**Purpose:** Confirm all users now have the correct
cloud-routable UPN suffix after the bulk migration.

---

## Entra Connect Sync Management

### Force immediate delta sync
```powershell
Start-ADSyncSyncCycle -PolicyType Delta
```
**Purpose:** Trigger an immediate synchronization cycle
instead of waiting for the default 30-minute schedule.
Use this after any identity change that requires urgent
cloud propagation — particularly during offboarding.

**Delta sync** processes only changes since the last sync.
It is fast (seconds) and low-impact on system resources.

---

### Force full synchronization
```powershell
Start-ADSyncSyncCycle -PolicyType Initial
```
**Purpose:** Trigger a complete re-sync of all objects
from AD to Entra ID. Use this after major changes such
as bulk UPN migrations or OU restructuring. Takes longer
than delta sync but ensures complete consistency.

---

### Check sync scheduler status
```powershell
Get-ADSyncScheduler
```
**Purpose:** View the current sync schedule including
when the next sync cycle will run, whether the scheduler
is enabled, and the sync interval. Useful for confirming
sync is running on schedule.

---

### Check sync connector run status
```powershell
Get-ADSyncConnectorRunStatus
```
**Purpose:** View the current state of the sync connectors.
Use this to confirm a sync cycle completed successfully
or to diagnose why sync may have stalled.

---

### View sync errors
```powershell
Get-ADSyncCSObject -ConnectorName "paccylab.onmicrosoft.com" |
Where-Object {$_.HasSyncError -eq $true}
```
**Purpose:** Identify any objects that failed to sync
and the specific error associated with each. First
diagnostic step when users are not appearing in Entra ID
after a sync cycle.

---

## Firewall Configuration

### Allow ICMP ping through Windows Firewall
```powershell
netsh advfirewall firewall add rule `
    name="Allow ICMPv4-In" `
    protocol=icmpv4:8,any `
    dir=in `
    action=allow
```
**Purpose:** Enable ping responses on DC01. Windows Server
blocks ICMP by default. This rule allows network
connectivity testing from the host machine without
disabling the firewall entirely.

---

### Verify firewall rule was created
```powershell
netsh advfirewall firewall show rule name="Allow ICMPv4-In"
```
**Purpose:** Confirm the ICMP rule exists and is enabled.

---

## DNS Configuration

### Add external DNS forwarders
```powershell
Add-DnsServerForwarder -IPAddress 8.8.8.8
Add-DnsServerForwarder -IPAddress 8.8.4.4
```
**Purpose:** Configure DC01's DNS server to forward
external name resolution requests to Google's public DNS.
DC01 handles all internal techbridge.local queries itself,
but forwards anything it cannot resolve (like
portal.azure.com) to Google. Required for DC01 to reach
Microsoft cloud services.

---

### Verify DNS forwarders
```powershell
Get-DnsServerForwarder
```
**Purpose:** Confirm external DNS forwarders are configured
correctly. Should show 8.8.8.8 and 8.8.4.4.

---

### Set DNS server address on specific adapter
```powershell
Set-DnsClientServerAddress `
    -InterfaceIndex 8 `
    -ServerAddresses "8.8.8.8","8.8.4.4"
```
**Purpose:** Assign specific DNS servers to a network
adapter by interface index. Use Get-NetAdapter to find
the correct interface index first.

---

## Offboarding and Lifecycle Management

### Complete offboarding sequence
```powershell
# Step 1 — Disable the account immediately
Disable-ADAccount -Identity tuser2

# Step 2 — Force immediate sync to cloud
Start-ADSyncSyncCycle -PolicyType Delta

# Step 3 — Verify account is disabled in AD
Get-ADUser -Identity tuser2 -Properties Enabled |
Select-Object Name, Enabled
```
**Purpose:** The complete three-step offboarding sequence
for any departing user. Disabling on-prem and forcing
delta sync ensures all Microsoft 365 access is revoked
within minutes — not hours.

---

### Verify account status in AD
```powershell
Get-ADUser -Identity tuser2 -Properties Enabled |
Select-Object Name, SamAccountName, Enabled, UserPrincipalName
```
**Purpose:** Confirm a specific user's account status
before and after offboarding actions.

---

### Find all disabled accounts in AD
```powershell
Search-ADAccount -AccountDisabled |
Select-Object Name, SamAccountName, UserPrincipalName
```
**Purpose:** Audit all disabled accounts in Active
Directory. Useful for compliance reporting and
identifying stale accounts.

---

## Related Documentation
- [Setup Guide](setup-guide.md)
- [Troubleshooting Scenarios](troubleshooting.md)
- [Screenshots](screenshots/)
