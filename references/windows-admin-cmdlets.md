# Windows administration — essential PowerShell cmdlets

Curated reference for admin tasks on Windows 10/11 + Windows Server. Grouped by domain. Every cmdlet listed here is current (non-deprecated) — `Get-WmiObject`, `Get-EventLog`, `wmic`, `netsh advfirewall`, `diskpart` scripts, and `schtasks.exe` XML are intentionally excluded in favor of their modern replacements.

Sources: [Microsoft Learn — Windows Server admin cmdlets](https://learn.microsoft.com/en-us/training/modules/manage-windows-server-settings-use-powershell-cmdlets/), [windows-powershell-docs on Context7](https://context7.com/microsoftdocs/windows-powershell-docs), [TechTarget — 25 basic admin cmdlets](https://www.techtarget.com/searchwindowsserver/tip/Top-25-Windows-PowerShell-commands-for-administrators).

---

## 1. Processes and services

| Cmdlet | Purpose | Example |
| --- | --- | --- |
| `Get-Process` | List running processes (returns `Process` objects with `CPU`, `WS`, `Id`) | `Get-Process \| Sort-Object WS -Descending \| Select -First 10` |
| `Stop-Process` | Kill by ID or name | `Stop-Process -Name notepad -Force` |
| `Start-Process` | Launch an exe/MSI, optionally elevated | `Start-Process msiexec -ArgumentList '/i app.msi /qn' -Verb RunAs -Wait` |
| `Wait-Process` | Block until a process exits | `Wait-Process -Id 1234 -Timeout 60` |
| `Get-Service` | Service state | `Get-Service spooler` |
| `Start-Service` / `Stop-Service` / `Restart-Service` | State change | `Restart-Service DHCP` |
| `Set-Service` | Configure startup type, account | `Set-Service bits -StartupType Automatic` |
| `New-Service` / `Remove-Service` | Create or delete a service | `New-Service -Name MyAgent -BinaryPathName "C:\bin\agent.exe"` |

## 2. CIM / WMI (replaces `Get-WmiObject` + `wmic`)

Always use **`Get-CimInstance`**. `Get-WmiObject` is removed in PS 7; `wmic` is removed in Windows 11.

| Class | What it surfaces |
| --- | --- |
| `Win32_OperatingSystem` | OS version, install date, last boot, total RAM |
| `Win32_ComputerSystem` | Manufacturer, model, domain, RAM, current user |
| `Win32_Processor` | CPU(s), core count, clock |
| `Win32_PhysicalMemory` | DIMMs and their capacity/speed |
| `Win32_LogicalDisk` | Drives with free / total space |
| `Win32_DiskDrive` / `Win32_Volume` | Physical disks / mounted volumes |
| `Win32_Product` | MSI-installed apps (slow — triggers reconfig, use `Get-Package` instead when possible) |
| `Win32_BIOS` | BIOS vendor, serial number |
| `Win32_NetworkAdapterConfiguration` | Legacy network config (prefer `Get-NetIPConfiguration`) |
| `Win32_Service` | Richer service detail (PID, path, account) than `Get-Service` |
| `Win32_Process` | Richer process info (path, command line) than `Get-Process` |

Examples:
```powershell
Get-CimInstance Win32_OperatingSystem | Select Caption, Version, LastBootUpTime, @{N='UptimeDays';E={(New-TimeSpan $_.LastBootUpTime (Get-Date)).Days}}
Get-CimInstance Win32_LogicalDisk -Filter "DriveType=3" | Select DeviceID, @{N='FreeGB';E={[math]::Round($_.FreeSpace/1GB,1)}}, @{N='TotalGB';E={[math]::Round($_.Size/1GB,1)}}
Get-CimInstance Win32_Product -Filter "Name LIKE '%Chrome%'" | Select Name, Version, Vendor
```

Invoke methods on instances via `Invoke-CimMethod`:
```powershell
Get-CimInstance Win32_Process -Filter "Name='notepad.exe'" | Invoke-CimMethod -MethodName Terminate
```

## 3. Event log (replaces `Get-EventLog`)

Always **`Get-WinEvent`** — supports the modern `Microsoft-Windows-*` logs that the legacy `Get-EventLog` misses.

```powershell
# Last 50 System errors
Get-WinEvent -LogName System -MaxEvents 50 | Where LevelDisplayName -eq 'Error'

# Filter server-side (10-100x faster than pipeline Where)
Get-WinEvent -FilterHashtable @{ LogName='Application'; Id=1000,1001; StartTime=(Get-Date).AddHours(-24) }

# Recent app crashes
Get-WinEvent -FilterHashtable @{ LogName='Application'; ProviderName='Application Error' } -MaxEvents 20

# Pretty-print one event in full
Get-WinEvent -LogName System -MaxEvents 1 | Select-Object * | Format-List
```

Write events with `New-WinEvent` if you have a registered provider, or `Write-EventLog` to the classic logs.

## 4. Networking (replaces `ipconfig` / `netsh` / `route print` / `arp`)

| Cmdlet | Purpose |
| --- | --- |
| `Get-NetIPConfiguration` | Adapter + IP + DNS summary (the `ipconfig /all` replacement) |
| `Get-NetAdapter` | NICs with status, speed, MAC |
| `Get-NetIPAddress` | IPv4/IPv6 assignments |
| `New-NetIPAddress` / `Remove-NetIPAddress` | Assign / remove IP |
| `Set-DnsClientServerAddress` | Set resolver | `-InterfaceAlias Ethernet -ServerAddresses 1.1.1.1,8.8.8.8` |
| `Get-NetRoute` / `New-NetRoute` | Routing table |
| `Test-Connection` | `ping` replacement with object output |
| `Test-NetConnection -Port N` | `telnet` replacement (TCP reachability + route + DNS) |
| `Resolve-DnsName` | `nslookup` replacement |
| `Get-NetTCPConnection` / `Get-NetUDPEndpoint` | `netstat -ano` replacement |
| `Get-NetFirewallRule` / `New-NetFirewallRule` / `Set-NetFirewallRule` / `Remove-NetFirewallRule` | Firewall (replaces `netsh advfirewall`) |
| `Get-SmbShare` / `New-SmbShare` / `Grant-SmbShareAccess` | SMB shares (replaces `net share`) |
| `Get-SmbConnection` / `Get-SmbSession` | Active SMB clients / sessions |

## 5. Storage (replaces `diskpart` scripts)

```powershell
Get-Disk                        # all physical disks
Get-Volume                      # all volumes incl. drive letter, size, health
Get-Partition -DiskNumber 0     # partitions on disk 0

# Provision a new data drive end-to-end
Initialize-Disk 1 -PartitionStyle GPT
New-Partition -DiskNumber 1 -UseMaximumSize -AssignDriveLetter |
    Format-Volume -FileSystem NTFS -NewFileSystemLabel Data -Confirm:$false

# Resize / extend
Resize-Partition -DriveLetter D -Size 500GB

# Storage Spaces
New-StoragePool -FriendlyName Pool1 -StorageSubsystemFriendlyName "Windows Storage*" -PhysicalDisks (Get-PhysicalDisk -CanPool $true)
New-VirtualDisk -StoragePoolFriendlyName Pool1 -FriendlyName VD1 -ResiliencySettingName Mirror -Size 2TB -ProvisioningType Thin
```

BitLocker: `Enable-BitLocker`, `Disable-BitLocker`, `Get-BitLockerVolume`, `Backup-BitLockerKeyProtector`, `Unlock-BitLocker`.

## 6. Scheduled tasks (replaces `schtasks.exe` XML)

```powershell
$action  = New-ScheduledTaskAction -Execute 'pwsh.exe' -Argument '-File C:\bin\daily.ps1'
$trigger = New-ScheduledTaskTrigger -Daily -At 3am
$prin    = New-ScheduledTaskPrincipal -UserId SYSTEM -RunLevel Highest
Register-ScheduledTask -TaskName 'DailyJob' -Action $action -Trigger $trigger -Principal $prin

Get-ScheduledTask | Where State -eq Ready
Start-ScheduledTask 'DailyJob'
Get-ScheduledTaskInfo 'DailyJob'    # last-run time, last result
Unregister-ScheduledTask 'DailyJob' -Confirm:$false
```

## 7. Registry (`HKLM:` and `HKCU:` PSDrives, not `reg.exe`)

```powershell
# Read
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion' -Name ProductName, CurrentBuild
(Get-Item 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion').GetValue('ProductName')

# Enumerate
Get-ChildItem 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall' |
    ForEach-Object { Get-ItemProperty $_.PSPath } |
    Where-Object DisplayName -ne $null | Select DisplayName, DisplayVersion

# Write (HKLM needs elevation)
New-Item 'HKCU:\Software\MyApp' -Force | Out-Null
Set-ItemProperty 'HKCU:\Software\MyApp' -Name 'Version' -Value '1.0' -Type String
Remove-ItemProperty 'HKCU:\Software\MyApp' -Name 'Version'
Remove-Item 'HKCU:\Software\MyApp' -Recurse
```

## 8. Packages and updates

| Task | Cmdlet / command |
| --- | --- |
| Install a winget package | `winget install --id=<Id> --silent --accept-source-agreements --accept-package-agreements --disable-interactivity` |
| Upgrade everything | `winget upgrade --all --silent --accept-package-agreements --accept-source-agreements --disable-interactivity --include-unknown` |
| List installed apps | `Get-Package` (faster, doesn't trigger MSI reconfig like `Win32_Product`) |
| PS module install | `Install-PSResource -Name <Name> -Scope CurrentUser -TrustRepository -Quiet` |
| PS module update | `Update-PSResource -Name <Name>` |
| List installed hotfixes | `Get-HotFix` |
| Windows Update via PS (requires PSWindowsUpdate) | `Install-PSResource PSWindowsUpdate; Get-WindowsUpdate; Install-WindowsUpdate -AcceptAll -AutoReboot` |
| List Windows features / roles | `Get-WindowsFeature` (Server) / `Get-WindowsOptionalFeature -Online` (Client) |
| Enable a feature | `Install-WindowsFeature -Name <Name>` (Server) / `Enable-WindowsOptionalFeature -Online -FeatureName <Name>` (Client) |
| DISM operations | `Get-WindowsImage`, `Add-WindowsPackage`, `Repair-WindowsImage` |
| Appx | `Get-AppxPackage`, `Add-AppxPackage`, `Remove-AppxPackage` |

## 9. Active Directory (RSAT module `ActiveDirectory`)

```powershell
# Users
Get-ADUser -Filter "Name -like 'John*'" -Properties *
New-ADUser -Name 'jdoe' -SamAccountName jdoe -AccountPassword (Read-Host -AsSecureString) -Enabled $true
Set-ADUser jdoe -Title 'Senior Engineer' -Department IT
Disable-ADAccount jdoe

# Groups
New-ADGroup -Name Devs -GroupScope Global -GroupCategory Security
Add-ADGroupMember Devs -Members jdoe
Get-ADGroupMember Devs -Recursive

# Computers / OUs
Get-ADComputer -Filter * -Properties OperatingSystem
New-ADOrganizationalUnit -Name Servers -Path 'DC=corp,DC=local'
```

## 10. Hyper-V

```powershell
New-VM -Name vm01 -MemoryStartupBytes 4GB -Generation 2 -SwitchName 'Default Switch'
Get-VM
Start-VM vm01 ; Stop-VM vm01 -Force ; Checkpoint-VM vm01 -SnapshotName baseline
Set-VM vm01 -ProcessorCount 4 -DynamicMemory -MemoryMinimumBytes 2GB -MemoryMaximumBytes 8GB
Get-VMNetworkAdapter vm01 | Set-VMNetworkAdapterVlan -Access -VlanId 100
```

## 11. Defender (replaces `mpcmdrun.exe` wrapping)

```powershell
Get-MpComputerStatus              # RealTimeProtection, signatures, quick scan time
Get-MpPreference                  # current config
Add-MpPreference -ExclusionPath 'C:\Users\yohan\dscode','C:\Users\yohan\ms-repos'
Set-MpPreference -DisableRealtimeMonitoring $true    # (requires admin + often Tamper Protection off)
Start-MpScan -ScanType QuickScan
Get-MpThreatDetection             # recent detections
```

## 12. Remoting

```powershell
# One-off
Invoke-Command -ComputerName srv01 -ScriptBlock { Get-Service dhcp }

# Interactive
Enter-PSSession -ComputerName srv01
# ... work ...
Exit-PSSession

# Persistent session (avoid reconnect overhead)
$s = New-PSSession -ComputerName srv01,srv02
Invoke-Command -Session $s -ScriptBlock { Get-Service dhcp }
Remove-PSSession $s

# Over SSH (PS 7 cross-platform)
Invoke-Command -HostName srv01 -UserName admin -ScriptBlock { Get-Service dhcp }
```

## 13. Group Policy (RSAT module `GroupPolicy`)

```powershell
New-GPO -Name 'Baseline Servers' -Comment 'Standard hardening'
Get-GPO -All
Get-GPOReport -Name 'Baseline Servers' -ReportType Html -Path baseline.html
Set-GPRegistryValue -Name 'Baseline Servers' -Key 'HKLM\SOFTWARE\Policies\Example' -ValueName Enabled -Type DWord -Value 1
Backup-GPO -Name 'Baseline Servers' -Path C:\gpo-backups
Restore-GPO -Name 'Baseline Servers' -Path C:\gpo-backups
New-GPLink -Name 'Baseline Servers' -Target 'OU=Servers,DC=corp,DC=local' -LinkEnabled Yes
```

## 14. IIS (module `IISAdministration`, replaces `appcmd.exe`)

```powershell
Import-Module IISAdministration
Get-IISSite
New-IISSite -Name 'MyApp' -PhysicalPath 'C:\inetpub\MyApp' -BindingInformation '*:8080:'
Start-IISSite MyApp ; Stop-IISSite MyApp
New-IISAppPool 'MyAppPool' -Force
Set-IISAppPool MyAppPool -Attribute @{ 'managedRuntimeVersion' = 'v4.0' }
```

## 15. ACLs and privileges

```powershell
# Read ACL
(Get-Acl 'C:\Share\data').Access | Format-Table IdentityReference, FileSystemRights, AccessControlType

# Grant Modify to a user
$acl = Get-Acl 'C:\Share\data'
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule('CORP\jdoe','Modify','ContainerInherit, ObjectInherit','None','Allow')
$acl.AddAccessRule($rule)
Set-Acl 'C:\Share\data' $acl

# Take ownership
takeown /f 'C:\path' /r /d Y     # (the built-in is still the cleanest for bulk takeover)
icacls 'C:\path' /grant Administrators:F /T

# Elevation check
[bool]([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
```

## 16. Diagnostics one-liners

```powershell
# Top 10 memory hogs
Get-Process | Sort-Object WS -Descending | Select-Object -First 10 Name, Id, @{N='MB';E={[int]($_.WS/1MB)}}, CPU

# Disks with <10% free
Get-CimInstance Win32_LogicalDisk -Filter 'DriveType=3' |
    Where-Object { ($_.FreeSpace / $_.Size) -lt 0.10 } |
    Select DeviceID, @{N='FreePct';E={[math]::Round(100*$_.FreeSpace/$_.Size,1)}}

# Recent crashes (last 24 h)
Get-WinEvent -FilterHashtable @{ LogName='Application'; ProviderName='Application Error'; StartTime=(Get-Date).AddDays(-1) }

# Uptime
(Get-Date) - (Get-CimInstance Win32_OperatingSystem).LastBootUpTime

# Listening ports with owning process
Get-NetTCPConnection -State Listen | ForEach-Object {
    [pscustomobject]@{
        Local  = "$($_.LocalAddress):$($_.LocalPort)"
        PID    = $_.OwningProcess
        Name   = (Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue).ProcessName
    }
} | Sort-Object Local
```

## 17. Common pitfalls on Windows

- **`Win32_Product` triggers MSI reconfiguration** — each query can cause slow repair cycles. Prefer `Get-Package` or the registry `HKLM:\...\Uninstall` walk.
- **`Get-Acl` + `Set-Acl` on `HKLM:` subkeys** is flaky — prefer `icacls` for registry ACLs.
- **`Set-Service -StartupType Disabled`** on a currently-running service doesn't stop it — chain with `Stop-Service`.
- **PS 5.1 `Out-File` defaults to UTF-16 LE with BOM** — always pass `-Encoding utf8` explicitly when the file will be read by a non-PS tool.
- **`-Recurse` on `Remove-Item` in a folder with reparse points** follows them by default — add `-Force` and be mindful, or use `Get-ChildItem -Force -Attributes !ReparsePoint -Recurse` first.
- **`Stop-Process -Force` doesn't wait** — follow with `Wait-Process -Timeout 10` if the next step depends on it being gone.
