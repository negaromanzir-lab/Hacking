# About Windows

## Overview

Notes on the Windows internals that show up during assessment: how identity and tokens work, how processes and services are structured, where configuration lives, and which defaults are permissive enough to matter. Written as a reference to read alongside the practical material in [`../tryHackmeNotes-main/`](../tryHackmeNotes-main/).

## Repository Structure

- Identity, tokens, and groups
- Processes, services, and scheduled tasks
- The registry
- File system and executable formats
- Active Directory fundamentals
- PowerShell and built-in tooling
- Event logs and auditing

## Identity, Tokens, and Groups

Windows does not have `sudo`. Privilege is carried by the access token that every process holds, and a token records the user, the enabled groups, and the privileges that apply.

```powershell
whoami
whoami /all                       # full token: groups, privileges, integrity level
whoami /priv
whoami /groups
net user <user>                   # account details
net localgroup                    # local groups and members
net group "Domain Admins"         # domain group membership
```

The details that change the outcome of a check:

- **Integrity levels.** `High` can usually write to system paths, `Medium` (a normal interactive user) generally cannot, `Low` is sandboxed and browser-origin code. `whoami /groups` labels the level as `Mandatory Label\...`.
- **Enabled vs. deny-only groups.** A SID can be present in the token but flagged deny-only, which is what happens to admin SIDs in a filtered token. Membership in a group is therefore not the same as the privileges of that group being available.
- **UAC split token.** With UAC on, administrators get two tokens: a full one for elevation and a filtered one for normal use. `net session` in `cmd` is a common way to observe whether a token is elevated.
- **Local Administrators is special.** A domain admin joining a machine does not automatically land in the local group unless the machine is domain-joined and Group Policy adds them, so check the local group and not only domain membership.

## Processes, Services, and Scheduled Tasks

```powershell
Get-Process
Get-CimInstance Win32_Process -Filter "Name='notepad.exe'" | Select CommandLine
tasklist /v
tasklist /svc                              # services hosted in each process
sc query type= service state= all
sc qc <service>                            # binary path, account, start type
```

Reading a service configuration is the highest-value step: `binPath`, `ServiceDll` in the registry, `ImagePath`, and the `ObjectName` it runs as. A service that runs as `LocalSystem` and loads a DLL from a writable path is a privilege escalation path, not a stylistic issue.

Persistence and execution scheduling:

```powershell
Get-ScheduledTask | Where-Object State -ne 'Disabled'
schtasks /query /fo LIST /v
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

Common auto-start locations: `Winlogon\Userinit` and `Shell`, `Run`/`RunOnce` under both `HKLM` and `HKCU`, the `Image File Execution Options` debugger keys, and WMI event subscriptions.

## The Registry

The registry is a tree of configuration hives, and each hive maps to a configuration surface on disk.

| Hive | File | Holds |
| --- | --- | --- |
| `HKLM` | `%SystemRoot%\System32\config\SYSTEM` | Machine-wide settings, services |
| `HKCU` | `%SystemRoot%\System32\config\USER` | Per-user settings |
| `HKCR` | merged view | File associations, COM classes |
| `HKU` | per-user hives | Loaded user profiles |
| `HKCC` | `%SystemRoot%\System32\config\SYSTEM` | Current computer config |

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall"
reg query "HKLM\SYSTEM\CurrentControlSet\Services" /s /f "LocalSystem"
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Run"
```

Uninstall keys are a clean way to inventory installed software, including the ones no shortcut suggests is installed. Services live at `HKLM\SYSTEM\CurrentControlSet\Services\<name>`, with `ImagePath`, `ObjectName`, and `Start` (2 = auto, 3 = manual, 4 = disabled).

## File System and Executable Formats

```powershell
Get-ChildItem -Path C:\ -Recurse -Force -ErrorAction SilentlyContinue |
  Where-Object { $_.Attributes -match 'Hidden|System' } | Select FullName
icacls C:\Projects
```

Every NTFS file carries a DACL, and the owner plus the DACL decide what an access token can do. Useful concepts:

- **Inheritance (`ACE` flags).** Permissions flow from parent to child unless inheritance is broken, so a fix applied at the root and a break applied at a child directory can coexist.
- **Null DACL / empty DACL.** A null DACL grants everyone full control; an empty DACL grants nobody. Either is worth flagging whenever it appears on a directory a service writes to.
- **The BUILTIN\Administrators ACE that allows *S-1-5-32-544*.** Because the SID is used rather than the name, a rename of the group does not change the behaviour.
- **Writable service directories.** Same class of problem as on Linux: a directory a privileged service executes from is an escalation primitive.

Notable format details: the **PE** header (an MZ stub plus a PE header, optional COFF header, optional header, section table), with the security directory at Data Directory index 4 pointing at the certificate table. A file signed but modified afterwards is a normal, and useful, observation. **Alternate data streams** mean a zone identifier (`Zone.Identifier`) and payload can share a name, and `dir /r` reveals them.

## Active Directory Fundamentals

```powershell
Get-ADUser -Filter * -Properties MemberOf
Get-ADGroup -Filter * -Properties Member
Get-ADComputer -Filter * -Properties OperatingSystem
Get-ADOrganizationalUnit -Filter *
klist                        # current TGT and cached tickets
nltest /domain_trusts
```

Structures worth knowing before an assessment:

- **The domain is a database replicated through the forest.** A Domain Admin in any domain of a forest that trusts in the right direction can usually escalate across the forest, so map trusts early.
- **Delegation.** Unconstrained delegation stores a reusable TGT, which means the first request that authenticates to a host with delegation enabled can capture it.
- **GPOs** apply in order: Local, Site, Domain, OU, with Block Inheritance as the explicit override. A GPO that grants a group `SeDebugPrivilege` is a common escalation route, and so is a writable GPO or SYSVOL file.
- **AD CS.** A Domain Controller with an enrolled CA can mint certificates, and templates with `ENROLLEE SUPPLIES SUBJECT` or an enrollee-supplied SAN are a frequent path to escalation.

## PowerShell and Built-in Tooling

```powershell
Get-ExecutionPolicy -List
Get-Module -ListAvailable
#: do not run history-logged commands on a target during an engagement
```

`-ExecutionPolicy` is a safety control, not a security boundary, so it should not be treated as one when reasoning about what is possible. Execution context matters for a defender reading logs: `4104` (script block) and `4103` (module logging) need to be enabled ahead of time, otherwise the evidence is simply not collected.

Tools already present on the box:

```powershell
Get-ChildItem, Get-Content, Select-String, Get-Process
Get-Service, Get-ScheduledTask, Get-NetTCPConnection
Get-LocalUser, Get-LocalGroupMember, Get-WmiObject
```

Built-ins are worth preferring for an assessment you may have to justify: the binary already exists on the host, so it leaves a smaller footprint than a tool you uploaded, and it inherits the token you already hold.

## Event Logs and Auditing

```powershell
Get-WinEvent -LogName Security -MaxEvents 50
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} -MaxEvents 20
wevtutil el | Select-String -Pattern 'Security|PowerShell|Sysmon'
```

Event IDs that carry most of the signal:

| ID | Meaning |
| --- | --- |
| 4624 | Successful logon |
| 4625 | Failed logon |
| 4648 | Logon with explicit credentials (`runas`) |
| 4672 | Special privileges assigned to new logon |
| 4688 | Process creation (needs command-line auditing on) |
| 4697 | Service installed |
| 4720 | User account created |
| 4732 | Member added to a global/local group |
| 4768 | Kerberos TGT requested |
| 4769 | Kerberos service ticket requested |
| 5140 | Network share accessed |
| 7045 | Service installed (System log) |
| 4104 | PowerShell script block logged |

The two most common gaps: command-line auditing is off by default for 4688, and PowerShell script block logging is off by default, so "nothing in the logs" often means "nothing was configured to be logged".

## Hardening Checklist

- Local Administrators limited to accounts that need it; no stale domain admin accounts cached locally.
- LSA protection enabled, with the `RunAsPPL` setting for credential isolation.
- Credential Guard / Defender for Credential Guard where the platform supports it.
- Defender for Endpoint on, tamper protection on, exclusions reviewed.
- Windows Firewall enabled on all profiles with a documented allow list.
- SMBv1 disabled; SMB signing and LDAP signing required.
- Remote PowerShell and WinRM restricted to management hosts.
- 4688 command-line auditing, PowerShell 4104 logging, and Sysmon installed and forwarding off-host.

## Related Material

- [`../tryHackmeNotes-main/Windows commands/`](../tryHackmeNotes-main/Windows%20commands/)
- [`../tryHackmeNotes-main/Windows PrivEsc/`](../tryHackmeNotes-main/Windows%20PrivEsc/)
- [`../tryHackmeNotes-main/Kerberos/`](../tryHackmeNotes-main/Kerberos/)
- [`../tryHackmeNotes-main/Powershell/`](../tryHackmeNotes-main/Powershell/)
- [`../tryHackmeNotes-main/Windows AppLocker/`](../tryHackmeNotes-main/Windows%20AppLocker/)
- [`../About Networking/`](../About%20Networking/)

## Scope and Legal Notice

Everything here is for systems you own or have explicit written authorisation to test. These techniques are dual-use and equally valuable to someone deciding whether a machine is worth targeting, so keep notes on authorisation scope, timestamps, and the specific reason each finding matters in your report.

## Contribution Guidelines

1. Lead with the command, then explain which part of the output actually matters and why.
2. Say which Windows version or edition a behaviour applies to; defaults differ between client and server SKUs.
3. Pair each finding with the corresponding defensive control, not just the offensive technique.
4. Keep screenshots and command output free of real domain names, usernames, and machine names.
