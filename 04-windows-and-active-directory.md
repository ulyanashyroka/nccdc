# 04 Windows And Active Directory

**Source:** NECCDC Tryout Briefing, plus explicitly illustrative practice material. See the [complete guide](README.md).

## 6. Windows and Active Directory

The briefing asks you to **learn the GUI once, then repeat in PowerShell**. Most commands below require domain access, correct modules, and appropriate privileges. On a standalone Windows machine, AD-specific cmdlets will not work.

### 6.1 AD PowerShell — users, groups, disable, move

**Beginner:** Active Directory centrally manages domain identities. An organizational unit (OU) is an administrative container; groups grant access.

**Scenario:** “A departed employee can still log in. Check account status and group memberships.”

```powershell
Import-Module ActiveDirectory
Get-ADUser -Identity jsmith -Properties Enabled,LastLogonDate,DistinguishedName |
  Select-Object Name,Enabled,LastLogonDate,DistinguishedName
Get-ADPrincipalGroupMembership -Identity jsmith | Select-Object Name
Get-ADGroupMember -Identity 'Restaurant-Staff' | Select-Object Name,ObjectClass
```

**Intermediate:** find the correct account and confirm written authorization before disabling it:

```powershell
Disable-ADAccount -Identity jsmith -WhatIf          # simulate action
# Disable-ADAccount -Identity jsmith                 # [CHANGE] only after approval
```

To preview moving an account to another OU:

```powershell
Get-ADUser jsmith | Move-ADObject -TargetPath 'OU=FormerStaff,DC=restaurant,DC=lab' -WhatIf
```

**Advanced:** distinguish local users from domain users, group nesting, inherited access, and the impact of moving an object to a new OU with different GPOs.

### 6.2 Privileged groups — who should administer what?

**Scenario:** “A normal cashier appears to have administrative permissions.”

```powershell
Get-ADGroupMember -Identity 'Domain Admins' -Recursive |
  Select-Object Name,SamAccountName,ObjectClass
Get-LocalGroupMember -Group Administrators       # on member server
whoami /groups
```

**Beginner:** Domain Admins are broadly privileged in the domain; local Administrators control a particular server/workstation. **Intermediate:** verify whether access was intentional, identify membership path, and apply least privilege after authorization. **Advanced:** understand tiering, privileged service accounts, nested groups, and auditing membership changes.

### 6.3 Group Policy — `gpresult`, `gpupdate`, applied GPOs

**Scenario:** “A password policy or security setting is different from what IT expects.”

```powershell
gpresult /r
gpresult /scope computer /v
gpresult /h "$env:TEMP\gp-report.html"
Get-GPInheritance -Target 'OU=Cashiers,DC=restaurant,DC=lab'   # GroupPolicy module
Get-GPResultantSetOfPolicy -ReportType Html -Path 'C:\Temp\rsop.html'
```

**Beginner:** GPOs can apply at site/domain/OU levels, with scope/security filtering and inheritance. **Intermediate:** use results to identify which policy actually applied; understand that a configured GPO might be filtered out. **Advanced:** inspect enforcement, loopback processing, WMI filters, conflicts and refresh failures. `gpupdate /force` **[CHANGE]** reprocesses policy and can affect the system; do not use it to cover up a policy misunderstanding.

### 6.4 Shares and NTFS — both permission layers

**Scenario:** “A user can access the share but cannot modify a file.”

```powershell
Get-SmbShare
Get-SmbShareAccess -Name 'RestaurantDocs'
Get-Acl 'C:\Shares\RestaurantDocs' | Format-List
icacls 'C:\Shares\RestaurantDocs'
Get-LocalGroupMember -Group 'Users'
```

**Beginner:** SMB share permissions and NTFS permissions are separate restrictions; effective network access must satisfy both. **Intermediate:** inspect inherited versus explicit ACEs and group membership. **Advanced:** evaluate effective rights, explicit deny, ownership, inheritance, auditing and SMB session identity. Example preview of granting a lab group a file right:

```powershell
icacls 'C:\Shares\RestaurantDocs' /grant 'RESTAURANT\Cashiers:(OI)(CI)M' /T
```

**[CHANGE]** This recursively grants Modify; only perform with verified requirements, a tested path, and a backup. Prefer minimal scope over `/T` when possible.

### 6.5 Remote Desktop — allowed users, NLA, firewall scope

**Scenario:** “The office manager cannot connect to the back-office server via RDP.”

```powershell
Get-Service TermService
Get-NetTCPConnection -State Listen -LocalPort 3389
Get-LocalGroupMember -Group 'Remote Desktop Users'
Get-NetFirewallRule -DisplayGroup 'Remote Desktop' |
  Select-Object DisplayName,Enabled,Direction,Action,Profile
Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server' |
  Select-Object fDenyTSConnections
Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' |
  Select-Object UserAuthentication
```

**Beginner:** `fDenyTSConnections=1` generally disables incoming RDP; `UserAuthentication=1` commonly indicates Network Level Authentication (NLA) is required. **Intermediate:** distinguish RDP service, listener, network, group membership and NLA/authentication failure. **Advanced:** examine user-right assignments, policy overrides, gateway infrastructure and narrow allowed source networks. Do not disable NLA or expose RDP broadly just to make a test connect.

### 6.6 Autoruns — scheduled tasks, services, Run keys

**Scenario:** “A mystery program starts again after every reboot.”

```powershell
Get-ScheduledTask | Select-Object TaskName,TaskPath,State
Get-CimInstance Win32_Service |
  Select-Object Name,State,StartMode,PathName
Get-ItemProperty 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Run'
Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Run'
```

**Beginner:** persistence can come from legitimate scheduled tasks, services or registry Run keys. **Intermediate:** inspect execution paths, authors, timestamps, digital signatures and related logs; compare with approved baseline. **Advanced:** use Microsoft's Autoruns tool to review additional autostart locations, then contain and document verified malicious persistence without deleting unknown business tasks.

---

