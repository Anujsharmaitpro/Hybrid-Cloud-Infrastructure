I'll build this comprehensive study note structure. Given the scope (36 parts), I'll create it in organized sections with visual diagrams for key concepts. Let me start generating supporting diagrams and building the notes in parallel.

Image generation isn't available, but I'll build the complete study notes with rich structure. Given the massive scope (36 parts), I'll organize everything into collapsible sections. Starting with Parts 1–10:

---

# 📘 Complete Study Notes — Enterprise Infrastructure & Operations (L3)

> **Approach:** Structured around the JD. Each section maps to a JD requirement. Priority tags included where the JD calls out emphasis.

---

## PART 1 — Windows Server Administration

> *JD Requirement: Server build, configuration, hardening, patching, upgrades, migration, lifecycle management, file services, core Windows components.*


###Windows Server 2016 / 2019 / 2022

### Architecture
- **Kernel**: Hybrid kernel (ntoskrnl.exe) — combines monolithic kernel with microkernel features
- **HAL** (Hardware Abstraction Layer): Abstracts hardware differences
- **Executive**: Manages objects, processes, threads, security
- **Subsystems**: Win32 (csrss.exe), POSIX (optional), WSL (Windows Server 2019+)
- **Registry**: Centralized configuration store (regedit.exe / reg.exe)
- **Process Model**: Processes → Threads → Handles

### Installation Methods
| Method | Use Case |
|--------|----------|
| **GUI Install** | Full desktop experience, easy management |
| **Server Core** | Minimal UI, reduced attack surface, 50% less patching |
| **Nano Server** | Ultra-lightweight, cloud-optimized (2016/2019 only, removed in 2022) |
| **Unattended Install** | Answer file (unattend.xml) for automated deployments |
| **PXE Boot** | Network-based installation via WDS |
| **Windows ADK/MDT** | Customized deployment images |

### Server Core vs Desktop Experience
- **Server Core**: No Explorer, no Start Menu, managed via PowerShell/remote tools
- Benefits: Smaller footprint, fewer reboots, reduced attack surface
- Limitations: Some MMC snap-ins don't work, requires familiarity with CLI
- **Desktop Experience**: Can be added post-install (Install-WindowsFeature Server-Gui-Mgmt-Infra)

### Version Differences (Quick Reference)
| Feature | 2016 | 2019 | 2022 |
|---------|------|------|------|
| Container support | Basic | Improved | Improved |
| WSL | No | Yes | Yes |
| TPM 2.0 | No | Partial | Yes |
| Shielded VMs | Yes | Yes | Enhanced |
| Storage Spaces Direct | Yes | Yes | Enhanced |
| SMB Direct | Yes | Yes | Enhanced |
| Hotpatch | No | No | Yes |




###Server Manager

### Overview
- Central management tool for Windows Server roles and features
- **Local Server**: Manage single server
- **All Servers**: Manage multiple servers in a server pool
- **Dashboard**: Overview of server roles, health, and pending restarts

### Key Operations
```powershell
# Add Role/Feature
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Remove Role/Feature
Uninstall-WindowsFeature -Name Web-Server

# List installed roles
Get-WindowsFeature | Where-Object {$_.InstallState -eq 'Installed'}
```

### Server Pool Management
- Add servers via Server Manager → Manage → Add Servers
- Requires WinRM configured and appropriate permissions
- Multi-server filtering and simultaneous operations

### Limitations
- Cannot manage Server Core remotely from older OS versions
- Some features require Server Manager to be installed on the managing machine
- Browser-based Server Manager available in Windows 10/11 for remote management




###PowerShell Administration

### Essential Commands
```powershell
# System Info
Get-ComputerInfo
Get-WmiObject Win32_OperatingSystem
Get-HotFix

# Services
Get-Service | Where-Object {$_.Status -eq 'Running'}
Restart-Service -Name W32Time -Force
Set-Service -Name Spooler -StartupType Automatic

# Processes
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10
Get-Process notepad
Stop-Process -Id 1234 -Force

# Events
Get-EventLog -LogName System -Newest 50
Get-WinEvent -LogName Application -MaxEvents 20

# Network
Get-NetIPConfiguration
Get-DnsClientServerAddress
Test-Connection -Count 4 google.com

# Scheduled Tasks
Get-ScheduledTask | Where-Object {$_.State -ne 'Disabled'}
Register-ScheduledTask -TaskName "DailyBackup" -Trigger (New-ScheduledTaskTrigger -Daily -At 2am) -Action (New-ScheduledTaskAction -Execute "C:\Backup\backup.ps1")
```

### PowerShell Remoting
```powershell
# Enable
Enable-PSRemoting -Force

# Remote command
Invoke-Command -ComputerName SRV01,SRV02 -ScriptBlock { Get-Service }

# Session
$session = New-PSSession -ComputerName SRV01
Invoke-Command -Session $session -ScriptBlock { Get-ADUser -Filter * }
Remove-PSSession $session
```




###Local Users/Groups

### Management
```powershell
# Local User
New-LocalUser -Name "jdoe" -Password (ConvertTo-SecureString "P@ssw0rd" -AsPlainText -Force)
Set-LocalUser -Name "jdoe" -PasswordNeverExpires $true
Remove-LocalUser -Name "jdoe"

# Local Group
New-LocalGroup -Name "AppServers"
Add-LocalGroupMember -Group "Administrators" -Member "DOMAIN\jdoe"
Remove-LocalGroupMember -Group "Remote Desktop Users" -Member "jdoe"

# List members
Get-LocalGroupMember -Group "Administrators"
```

### Built-in Accounts
| Account | SID | Purpose |
|---------|-----|---------|
| Administrator | S-1-5-21-...-500 | Full control |
| Guest | S-1-5-21-...-501 | Limited access (disabled by default) |
| SYSTEM | S-1-5-18 | OS services |
| LOCAL SERVICE | S-1-5-19 | Limited system access |
| NETWORK SERVICE | S-1-5-20 | Network access |

### Best Practices
- Rename default Administrator account
- Disable Guest account
- Implement LAPS (Local Administrator Password Solution)
- Limit local group memberships
- Document local admin passwords in a secure vault




###Services

### Key Services
| Service | Name | Purpose |
|---------|------|---------|
| Windows Update | wuauserv | OS patching |
| Server | LanmanServer | File/print sharing |
| Workstation | LanmanWorkstation | Network connections |
| DNS Client | Dnscache | DNS caching |
| DHCP Client | Dhcp | IP address assignment |
| Netlogon | Netlogon | AD authentication |
| DFS Replication | DfsR | DFS replication |
| IIS Admin | W3SVC | Web services |

### Management
```powershell
# Service control
Get-Service -Name wuauserv
Set-Service -Name wuauserv -StartupType Automatic
Restart-Service -Name wuauserv -Force
Stop-Service -Name wuauserv
Start-Service -Name wuauserv

# Service dependencies
sc qc <service_name>
sc qdepend <service_name>
```




###Scheduled Tasks

### Creation
```powershell
# Simple scheduled task
$trigger = New-ScheduledTaskTrigger -Daily -At 2am
$action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-File C:\Scripts\cleanup.ps1"
Register-ScheduledTask -TaskName "Daily Cleanup" -Trigger $trigger -Action $action -User "SYSTEM" -RunLevel Highest
```

### Key Properties
- **Triggers**: Time-based, event-based, or conditional
- **Actions**: Execute program, send email, display message
- **Conditions**: Start only if on AC power, idle, etc.
- **Settings**: Allow task to run on demand, stop if running too long, restart on failure

### Troubleshooting
- Check `Task Scheduler` GUI or `schtasks /query`
- Review Event Viewer → Applications and Services Logs → Microsoft → Windows → TaskScheduler
- Ensure task has appropriate permissions and execution policy allows




###Registry

### Hive Overview
| Hive | Description |
|------|-------------|
| HKEY_LOCAL_MACHINE (HKLM) | Machine-wide settings |
| HKEY_CURRENT_USER (HKCU) | Current user settings |
| HKEY_CLASSES_ROOT (HKCR) | File associations/COM |
| HKEY_USERS (HKU) | All user profiles |
| HKEY_CURRENT_CONFIG (HKCC) | Hardware profile |

### Management
```powershell
# Registry operations
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion" -Name "ProgramFilesDir"
Set-ItemProperty -Path "HKLM:\SOFTWARE\MyApp" -Name "EnableFeature" -Value 1
New-Item -Path "HKLM:\SOFTWARE\MyApp" -Force
Remove-Item -Path "HKLM:\SOFTWARE\MyApp" -Recurse -Force

# Registry via reg.exe
reg query "HKLM\SOFTWARE\Microsoft" 
reg add "HKLM\SOFTWARE\MyApp" /v Version /t REG_SZ /d "1.0" /f
reg delete "HKLM\SOFTWARE\MyApp" /f
```

### Common Troubleshooting Registries
- **Run keys**: HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
- **Services**: HKLM\SYSTEM\CurrentControlSet\Services
- **IE settings**: HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Internet Settings




###Windows Firewall

### Management
```powershell
# Check status
Get-NetFirewallProfile

# Enable/Disable
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True

# Rule management
New-NetFirewallRule -DisplayName "Allow RDP" -Direction Inbound -LocalPort 3389 -Protocol TCP -Action Allow
Get-NetFirewallRule -DisplayName "RDP*" | Get-NetFirewallPortFilter
Remove-NetFirewallRule -DisplayName "Allow RDP"
```

### Key Concepts
- **Profiles**: Domain, Private, Public (applied based on network type)
- **Inbound vs Outbound**: Inbound rules control incoming traffic; outbound control leaving
- **Windows Firewall with Advanced Security** (wf.msc): GUI management
- **GPO-managed firewall**: Domain environments typically configured via GPO




###Windows Update

### Management
```powershell
# Windows Update module
Get-WindowsUpdate  (PSWindowsUpdate module)
Install-WindowsUpdate -AcceptAll -AutoReboot
Get-WUJob
Get-WURebootStatus

# Manual commands
userview.exe  # View update history
wuauclt /detectnow  # Force detection
```

### GPO Configuration Paths
- Computer Configuration → Administrative Templates → Windows Components → Windows Update
- Key settings: Configure Automatic Updates, Set priorities, Specify intranet Microsoft update service

### Troubleshooting
- **Windows Update log**: `C:\Windows\Logs\WindowsUpdate\wu.log` (20H2+) or `C:\Windows\SoftwareDistribution\DataStore\Logs\Edb.log`
- **CBS log**: `C:\Windows\Logs\CBS\CBS.log`
- **DISM repair**: `DISM /Online /Cleanup-Image /RestoreHealth`
- **SFC repair**: `sfc /scannow`

### Patch Types
- **Cumulative Update**: Includes all previous patches for that month
- **Security Update**: Specific security fix
- **Servicing Stack Update (SSU)**: Updates the update mechanism itself
- **Feature Update**: Major version upgrade (e.g., 20H2 → 21H2)




###SMB

### Versions
| Version | OS | Features |
|---------|----|----------|
| SMB 1.0 | NT 3.1 | Legacy, insecure |
| SMB 2.0 | Vista | Performance improvements |
| SMB 2.1 | Server 2008 | Continued improvements |
| SMB 3.0 | Server 2012 | Encryption, multichannel, DAC |
| SMB 3.02 | Server 2012 R2 | Performance tuning |
| SMB 3.1.1 | Server 2016+ | Encryption required, pre-auth integrity |

### Management
```powershell
# Check SMB version
Get-SmbServerConfiguration | Select-Object EnableSMB1Protocol, EnableSMB2Protocol

# Enable SMB 3.1.1 encryption
Set-SmbServerConfiguration -EncryptData $true

# Share management
New-SmbShare -Name "Data" -Path "D:\Data" -FullAccess "Everyone"
Get-SmbShare
Remove-SmbShare -Name "Data" -Force

# SMB session
Get-SmbSession
Get-SmbOpenFile
```

### Security
- Disable SMB 1.0 (major security risk — WannaCry exploited it)
- Require encryption on SMB 3.1.1+
- Use SMB signing for domain environments
- Audit SMB access




###NTFS Permissions

### Permission Types
| Type | Description |
|------|-------------|
| **Read** | View contents, attributes |
| **Read & Execute** | Run applications |
| **List Folder Contents** | View files in folder |
| **Write** | Create/modify files |
| **Modify** | Read + Write + Delete |
| **Full Control** | All permissions + change permissions |

### Special Permissions
- Traverse Folder / Execute File
- List Folder / Read Data
- Read Attributes / Extended Attributes
- Create Files / Write Data
- Create Folders / Append Data
- Delete Subfolders and Files

### Best Practices
- Use NTFS permissions over share permissions when possible (most restrictive wins)
- Group-based permissions > individual user permissions
- Avoid "Everyone" or "Authenticated Users" with broad access
- Document permission changes
- Periodically audit with `icacls` or AccessChaser

```powershell
# Check permissions
icacls D:\Data
Get-Acl D:\Data | Format-List

# Modify permissions
icacls D:\Data /grant "DOMAIN\Group":(OI)(CI)F /remove "Everyone"
```

### Share vs NTFS Permission Evaluation
- **Combined effective permission**: Most restrictive of the two
- **Share permissions**: Only apply to network access (not local)
- **NTFS**: Apply to both local and network
- Always set Share to "Everyone: Full Control" and manage at NTFS level (best practice)




###File Services / DFS

### DFS Namespace
- **Namespace type**: Domain-based (AD integrated) or Standalone
- **Folder targets**: Physical shares across multiple servers
- **Referral order**: Highest priority → lowest priority; site-aware

```powershell
# DFS Namespace management
New-DfsnRoot -Path "\\domain.com\Data" -TargetPath "\\SRV01\Data" -Type DomainV2
New-DfsnFolder -Path "\\domain.com\Data\Reports" -TargetPath "\\SRV02\Reports"
Get-DfsnRoot
Get-DfsnFolder -Path "\\domain.com\Data"
```

### DFS Replication (DFSR)
- **Purpose**: Keep folders synchronized between servers
- **Conflict resolution**: Latest writer wins (by default)
- **Staging folder**: 660MB default (adjust if large files)
- **Quota**: Prevents excessive replication traffic

```powershell
# DFSR management
New-DfsReplicationGroup -GroupName "FileSync"
New-DfsrMembership -GroupName "FileSync" -FolderName "Data" -ContentPath "D:\Data" -ComputerName SRV01
Get-DfsrStatus -GroupName "FileSync"
```

### FSRM (File Server Resource Manager)
- Quota management
- File screening (block certain file types)
- Storage reports
- File management tasks (auto-move based on policies)




###IIS Basics

### Key Components
- **Sites**: Bindings (IP, port, hostname)
- **Application Pools**: Isolated worker processes (w3wp.exe)
- **Sites vs Applications**: A site can contain multiple applications

### Management
```powershell
# IIS module
Import-Module WebAdministration
Get-Website
Start-Website -Name "Default Web Site"
New-Website -Name "MySite" -Port 8080 -PhysicalPath "C:\MySite" -ApplicationPool "MyAppPool"
Get-WebConfigurationProperty -Filter "/system.webServer/security/authentication/anonymousAuthentication" -Name enabled
```

### Application Pool Recycling
- Regular time intervals (default 1740 min)
- Request-based (specific number of requests)
- Memory-based (private memory limit)
- Rapid-fail protection (prevents crash loops)




###Windows Clustering

### Key Concepts
- **Failover Cluster**: Group of independent servers providing high availability
- **Node**: Individual server in the cluster
- **Cluster Name Object (CNO)**: AD computer object for the cluster
- **Virtual Server/Network Name**: Client-facing name
- **Quorum**: Mechanism to determine cluster health

### Cluster Types
- **Failover Cluster**: Traditional Windows Server clustering
- **Storage Spaces Direct**: Hyper-converged storage + compute
- **Stretch Cluster**: Across sites

### Key Tools
- **Failover Cluster Manager** (cluadmin.msc)
- **PowerShell**: `Get-Cluster`, `Get-ClusterResource`, `Get-ClusterNode`
- **Test-Cluster**: Validation before creation

### Cluster Roles
- File Server (including CSV)
- Hyper-V Virtual Machine
- SQL Server
- Print Spooler
- Generic Application




###Failover Cluster Manager

### Cluster Creation Process
1. Validate all nodes (Test-Cluster)
2. Create cluster (New-Cluster)
3. Configure quorum
4. Add cluster roles
5. Configure storage (if applicable)
6. Validate failover

```powershell
# Cluster management
Get-ClusterNode
Get-ClusterResource
Stop-ClusterNode -Name SRV01
Resume-ClusterNode -Name SRV01
Get-ClusterGroup | Move-ClusterGroup -Node SRV02
```

### Quorum Types
- **Node Majority**: Odd number of nodes
- **Node + File Share Witness**: Even nodes + witness
- **Node + Cloud Witness**: Azure cloud witness (recommended for stretched clusters)
- **Dynamic Quorum**: Automatically adjusts when nodes leave
- **Dynamic Witness**: Witness vote adjusts dynamically




###Windows Admin Center

### Overview
- Browser-based management tool for Windows Server
- Replaces many Server Manager functions
- Manages: Server, Cluster, Hyper-V, Storage, Networking, AD
- Can be deployed on Windows Server or as a VM in Azure

### Key Features
- Server overview and health
- Performance monitoring
- Event log viewing
- Feature/dashboard management
- Remote PowerShell
- Cluster management
- Storage management (S2D, Storage Spaces)




###Server Lifecycle Management

### Lifecycle Stages
1. **Build Standards**: Baseline image, naming convention, documentation
2. **Provisioning**: Install, join domain, configure roles
3. **Hardening**: Apply security baselines, remove unnecessary services
4. **Operation**: Monitoring, patching, performance management
5. **Upgrade**: In-place vs migration (side-by-side)
6. **Migration**: To new OS version, to cloud, to new hardware
7. **Decommission**: Data migration, AD cleanup, physical disposal

### Build Standards Checklist
- Naming convention (e.g., SRV-DC-01, SRV-WEB-02)
- Static IP / IPAM integration
- Time sync configuration (NTP)
- WinRM configured for PowerShell remoting
- Defender/AV deployed
- Logging configured (forwarded events)
- Local admin managed (LAPS)
- Monitoring agent deployed

### Server Hardening Checklist
- Disable unnecessary services and protocols
- Apply CIS benchmark recommendations
- Enable audit policies
- Configure TLS 1.2+ only
- Restrict RDP access
- Enable BitLocker (if applicable)
- Remove default accounts or restrict
- Enable Credential Guard / Device Guard
- Configure Windows Defender + Tamper Protection





---

## PART 2 — Active Directory

> *JD Priority: Core infrastructure — AD is the backbone of enterprise authentication and authorization.*


###AD Fundamentals

### Core Concepts
| Term | Description |
|------|-------------|
| **Forest** | Top-level container; shared schema, global catalog, trust |
| **Tree** | Collection of domains sharing a contiguous namespace |
| **Domain** | Security boundary; unit of replication and administration |
| **OU** | Organize objects; apply GPO; delegate permissions |
| **Sites** | Physical/geographic representation; control replication traffic |
| **Domain Controller (DC)** | Server holding AD DS role; authenticates users |
| **Global Catalog (GC)** | Contains partial attribute set of all objects in forest |
| **FSMO** | Single-master operations for specific tasks |
| **Schema** | Defines all object classes and attributes in AD |
| **Configuration Partition** | Forest-wide, contains topology info |
| **AD Database** | NTDS.DIT — main database file |
| **SYSVOL** | Logon scripts, GPOs (replicated via FRS or DFSR) |
| **NTDS** | Active Directory database (ntds.dit + transactions logs) |

### AD Physical Structure
- **Sites & Subnets**: Define replication topology based on geography
- **Site Links**: Define replication schedule and cost between sites
- **Bridgehead Servers**: Route replication between sites
- **KCC (Knowledge Consistency Checker)**: Automatically creates replication topology

### AD Logical Structure
- Forest → Trees → Domains → OUs → Objects
- Trust relationships connect forests/domains

### Global Catalog
- Holds a partial replica of **every domain** in the forest
- Contains attributes marked as "partial attribute set"
- Required for universal group membership resolution
- Required for cross-domain object references
- GC servers are also DCs
- Default GC port: 3268 (LDAP), 3269 (LDAPS)

### SYSVOL
- Shared folder on each DC: `C:\Windows\SYSVOL\sysvol`
- Contains:
  - Group policy objects (GPC)
  - Logon scripts
  - Netlogon share
- Replication methods: **FRS** (legacy) or **DFSR** (current recommendation)

### AD Database (NTDS.DIT)
- Located at `C:\Windows\NTDS\ntds.dit`
- Contains all AD objects
- Transaction logs: `C:\Windows\NTDS\Ntds.dit.log`
- Checkpoint file: `C:\Windows\NTDS\chk`
- **DSA** (Directory System Agent): Process that manages AD database




###User/Computer Administration

### User Lifecycle
```
Create → Enable → Assign Groups → Monitor → Disable → Move to Disabled OU → Delete/Re-enable
```

```powershell
# User management
New-ADUser -Name "John Doe" -SamAccountName "jdoe" -UserPrincipalName "jdoe@domain.com" -Path "OU=Users,DC=domain,DC=com" -AccountPassword (ConvertTo-SecureString "P@ssw0rd" -AsPlainText -Force) -Enabled $true
Set-ADUser -Identity "jdoe" -DisplayName "John Doe" -Department "IT" -Office "Building A"
Unlock-ADAccount -Identity "jdoe"
Disable-ADAccount -Identity "jdoe"
Get-ADUser -Identity "jdoe" -Properties LastLogonDate, PasswordLastSet, LockedOut
Remove-ADUser -Identity "jdoe" -Confirm:$false

# Bulk user creation
Import-CSV "C:\users.csv" | ForEach-Object { New-ADUser -Name $_.Name -SamAccountName $_.Sam -Path $_.OU -AccountPassword (ConvertTo-SecureString $_.Pass -AsPlainText -Force) -Enabled $true }
```

### Computer Accounts
```powershell
# Computer management
New-ADComputer -Name "PC01" -SamAccountName "PC01$" -Path "OU=Computers,DC=domain,DC=com" -Enabled $true
Get-ADComputer -Filter * -Properties OperatingSystem, LastLogonDate
Remove-ADComputer -Identity "PC01" -Confirm:$false

# Find stale computers (no logon for 90 days)
Search-ADAccount -ComputerOnly -Inactive 90
```

### Security Groups vs Distribution Groups
| Feature | Security | Distribution |
|---------|----------|-------------|
| Security descriptor | Yes | No |
| Can be assigned permissions | Yes | No |
| Can be used in ACLs | Yes | No |
| Email-enabled | Optional | Yes |
| Group types | Universal/Global/Universal | Same |

### Group Scope
| Scope | Valid Members | Scope Applied To |
|-------|--------------|-----------------|
| **Domain Local** | Any | Domain Local group's own domain |
| **Global** | Same domain | Any domain in forest |
| **Universal** | Any | Any domain in forest |

### Nested Groups Best Practice
- Use **Universal** groups for cross-domain access
- Global groups contain users from same domain
- Domain Local groups grant access to resources
- A → B → C where A is Domain Local, B is Global, C contains users

### Account Lockout
- **Account Lockout Policy**:
  - Account lockout threshold: e.g., 5 failed attempts
  - Account lockout duration: e.g., 30 minutes
  - Reset counter after: e.g., 5 minutes
- **Fine-Grained Password Policies (FGPP)**: Different policies for different users
- **Managed by** attribute: Can be owned by another user

### Password Policies
- Minimum password length (recommend 14+ characters)
- Password complexity (or passphrase approach)
- Password history (prevent reuse)
- Maximum password age (controversial; NIST now recommends no max for strong passwords)

### Delegation
- Delegate control over specific OUs using "Delegation of Control" wizard
- Principle of least privilege
- Commonly delegated: Reset passwords, manage groups, create/delete users in specific OU

### Computer Trust Relationship
```powershell
# Reset machine account
Reset-ComputerMachinePassword -NewPassword (ConvertTo-SecureString "NewPass" -AsPlainText -Force)
Test-ComputerSecureChannel -Verbose
Test-ComputerSecureChannel -Repair -Credential (Get-Credential)
```




###AD Replication

### Replication Topology
- **Intra-site**: Ring topology by default; full replication mesh (every DC replicates with every other DC in site)
- **Inter-site**: Site link bridges; scheduled, compression enabled

### Key Concepts
| Term | Description |
|------|-------------|
| **KCC** | Knowledge Consistency Checker; auto-creates replication topology |
| **USN (Update Sequence Number)** | Every write gets a USN; DCs know which changes they've seen |
| **Invocation ID** | Unique per DC; used for FSMO role conflict resolution |
| **Replication Latency** | Time delay between change and propagation |
| **Lingering Objects** | Deleted objects not removed in time; cleaned by tombstone policing |
| **Tombstone Lifetime** | Default 180 days (2016+); objects deleted after this period |
| **Repadmin** | Command-line tool for replication management |
| **DCDiag** | Diagnostic tool for DC health |

### Replication Commands
```powershell
# Repadmin
repadmin /replsummary        # Replication summary
repadmin /replstats          # Replication statistics
repadmin /showrepl           # Show replication per DC
repadmin /syncall            # Force sync all
repadmin /options            # Check DC options (IS_GC, GC capable)
repadmin /bridgeheads        # Show bridgehead servers
repadmin /cache              # Show cached (connected) objects

# DCDiag
dcdiag /v                    # Verbose diagnostic
dcdiag /s:DC01               # Test specific DC
dcdiag /e                    # Test all DCs in enterprise
dcdiag /c                    # Check configuration

# Force replication between specific DCs
repadmin /replicate <destDC> <sourceDC> <partition>
# Example:
repadmin /replicate SRV-DC-02 SRV-DC-01 DC=domain,DC=com
```

### Tombstone & Lingering Objects
- **Tombstone**: Deleted object becomes a "tombstone" for tombstone lifetime period
- **Tombstone Lifetime**: Default 180 days (Windows Server 2016+); was 60 days
- **Lingering Objects**: If a DC is offline longer than tombstone lifetime, lingering objects can persist
- **Prevention**: Remove disconnected DCs promptly; run `repadmin /removelingeringobjects`
- **DSRM (Directory Services Restore Mode)**: Used for restore operations

### USN Rollback
- Occurs when a DC restores from backup and has a USN higher than other DCs
- Results in objects being re-replicated incorrectly
- Detected by invocation ID + USN comparison
- Fixed by authoritative restore or VM restore integration with VM-GenerationID

### Inter-site Replication Optimization
- Compression enabled by default (10-100x reduction)
- Schedule replication during off-hours
- Use site links with proper cost metrics
- Configure preferred/emergency bridgehead servers




###FSMO Roles

### Five FSMO Roles

| Role | Scope | Default DC | Responsibility |
|------|-------|-----------|----------------|
| **Schema Master** | Forest-wide | First DC in forest | Controls schema updates (one at a time) |
| **Domain Naming Master** | Forest-wide | First DC in forest | Manages domain add/remove |
| **RID Master** | Domain-wide | First DC in domain | Allocates RID pools to each DC |
| **PDC Emulator** | Domain-wide | First DC in domain (usually) | Time sync, password changes, GPO updates, account lockout |
| **Infrastructure Master** | Domain-wide | First DC in domain | Updates references between domains; disabled when all DCs are GC |

### Operations (Transfer vs Seize)
- **Transfer (graceful)**: `Move-ADDirectoryServerOperationMasterRole`
- **Seize (emergency)**: When DC is permanently offline

```powershell
# Check current FSMO holders
netdom query fsmo

# Transfer role gracefully
Move-ADDirectoryServerOperationMasterRole -Identity "SRV-DC-02" -OperationMasterRole SchemaMaster, DomainNamingMaster, RIDMaster, PDCEmulator, InfrastructureMaster

# Seize role (use with caution!)
Move-ADDirectoryServerOperationMasterRole -Identity "SRV-DC-02" -OperationMasterRole SchemaMaster -Force

# Find which DC holds which role (PowerShell)
(Get-ADDomainController -Filter *).OperationMasterRoles

# Seize specific role
ntdsutil
roles
seize schema master
quit
```

### PDC Emulator Special Roles
- **Time Source**: All DCs sync time from PDC Emulator
- **Password Changes**: Conflicts resolved by PDC Emulator
- **Account Lockout**: PDC Emulator is source of truth for lockout
- **GPO Updates**: Changes to GPOs are processed here
- **LDS/LDS**: May also hold roles
- **Group Policy**: The PDC Emulator processes GPO changes in the domain

### Infrastructure Master
- Updates references to objects in other domains
- Disabled when all DCs are Global Catalogs (every DC knows about all objects)
- If isolated from GC, it cannot update references

### Finding Seized/Orphaned FSMO
- If DC was decommissioned without transferring roles
- Use `netdom query fsmo` or check with AD Recycle Bin
- Re-attach DC with seized role to prevent conflicts




###AD Troubleshooting

### Authentication Failure
1. Check DNS resolution (LDAP/SRV records)
2. Check time sync (5-minute skew tolerance)
3. Check network connectivity to DC
4. Check NTLM/Kerberos
5. Check account lockout/expiration
6. Check secure channel

### Account Lockout Troubleshooting
- **Event IDs**: 4740 (locked out), 4625 (failed logon)
- **Track lockouts**: Use `lockoutstatus.exe` or Event Viewer on all DCs
- **Cause**: Service account, scheduled task, cached credentials, mobile device sync

### DNS Failure (AD Depends on DNS)
- Verify DNS is AD-integrated or properly configured
- Check _msdcs DNS records
- Verify LDAP SRV records exist
- Confirm 127.0.0.1 as primary DNS (after startup)

### Replication Failure Troubleshooting
```powershell
# Step 1: Check replication status
repadmin /replsummary

# Step 2: Check specific DC
repadmin /showrepl <DCName>

# Step 3: Check site topology
repadmin /sites

# Step 4: Check connectivity
repadmin /testprem
Test-NetConnection -ComputerName <DCName> -Port 389
Test-NetConnection -ComputerName <DCName> -Port 636
Test-NetConnection -ComputerName <DCName> -Port 3268

# Step 5: Check DNS
Resolve-DnsName _ldap._tcp.dc._msdcs.domain.com -Type SRV

# Step 6: DCDiag
dcdiag /e /v

# Step 7: NetLogon
nltest /dsgetdc:domain.com
nltest /sc_query:domain.com
```

### SYSVOL Issues
- **FRS vs DFSR**: Check which replication method is in use
- **Event ID 13508 (DFSR)**: DFSR failed to replicate
- **SYSVOL share missing**: Check Netlogon service
- **FRS journal wrap**: `FRSDiag` tool; may need BurFlags registry key to force autoreset
- **DFSR journal wrap**: Check DFSR database, may need `dfsrutil` commands

### Secure Channel Failure
```powershell
# Reset secure channel
Test-ComputerSecureChannel -Repair
Test-ComputerSecureChannel -Verbose

# If fails, check:
# 1. DNS resolution
# 2. Network connectivity
# 3. Time sync
# 4. DC time
# 5. NetLogon service running
```

### DC Unavailable
- Check if DC is powered on and reachable
- Check if DSRM is clean (DSRM password)
- Verify DCDiag results
- Check disk space on system drive
- Check Event Viewer for errors

### Time Synchronization
- All DCs sync time from **PDC Emulator** in the domain
- External time source → PDC Emulator → all other DCs → member servers/clients
- **Event ID**: Time service events in System log
- **Check**: `w32tm /query /status`, `w32tm /stripchart /computer:dc01`
- **Fix**: `w32tm /config /manualpeerlist:time.windows.com /syncfromflags:manual /reliable:true /update`

### GPO Not Applying
- **Check**: `gpresult /r`, `gpresult /h report.html`
- **Common causes**:
  - Security filtering wrong
  - WMI filter doesn't match
  - GPO disabled
  - Inheritance blocked
  - GPO linked to wrong OU/site
  - File replication of GPO not working (SYSVOL)
  - Client not in correct security group

### Duplicate SPN
```powershell
# Find duplicate SPNs
setspn -Q MSSQLSvc
setspn -Q HTTP
setspn -Q LDAP
setspn -Q HOST

# Register SPN
setspn -S MSSQLSvc/FQDN:1433 DOMAIN\serviceAccount
setspn -S HTTP/web.domain.com DOMAIN\serviceAccount

# Remove duplicate
setspn -D MSSQLSvc/oldname DOMAIN\serviceAccount
```

### Kerberos Failure
- **Event IDs**: 4768 (TGT request), 4769 (service ticket), 4771 (pre-auth)
- **Common causes**:
  - Time skew (>5 minutes)
  - SPN duplicate
  - Wrong SPN format
  - Delegation misconfiguration
  - KDC unreachable
- **Diagnosis**:
  ```powershell
  klist          # Show current tickets
  klist tgt      # Show TGT
  klist purge    # Clear tickets
  ```





---

## PART 3 — Group Policy

> *GPOs are the primary mechanism for enforcing configuration across the enterprise.*


###Fundamentals

### Key Terms
| Term | Description |
|------|-------------|
| **GPO** | Group Policy Object — the actual policy container |
| **GPT** | Group Policy Template — the XML files (version number, display name) |
| **GPC** | Group Policy Container — AD object (display name, links, status) |
| **LSDOU** | Local, Site, Domain, OU — processing order (innermost wins) |

### GPO Storage
- **AD**: GPC in AD (CN=Policies, CN=System, ... — contains GPO metadata)
- **SYSVOL**: GPT in SYSVOL (User Version number)
- Version mismatch between AD and SYSVOL = problems

### Processing Order (LSDOU — Innermost Wins)
1. **Local** (local Group Policy)
2. **Site** (site-linked GPOs)
3. **Domain** (domain-linked GPOs)
4. **OU** (OU-linked GPOs — **wins** if multiple apply)

```powershell
# Force GPUpdate
gpupdate /force
gpupdate /target:user
gpupdate /target:computer

# View applied GPOs
gpresult /h C:\gpreport.html /f
gpresult /r
Get-GPResultantSetOfPolicy -Report Html -Path C:\gpreport.html

# Resultant Set of Policy (RSoP)
gpresult /r /scope:computer
gpresult /r /scope:user
```

### Inheritance Control
- **Block Inheritance**: Prevents parent GPOs from applying (right-click OU → Block Inheritance)
- **Enforced (No Override)**: Parent GPOs apply even if child OU blocks inheritance (right-click GPO → Enforced)
- **Order**: Enforced > Blocked > Link Enabled/Disabled

### Security Filtering
- Default: **Authenticated Users** (all authenticated users get it)
- Can narrow to specific groups
- **Common issue**: GPO linked but doesn't apply because security group doesn't include target users/computers
- **Best practice**: Use dedicated security groups, not individual users
- **Note**: `Authenticated Users` includes all authenticated computer accounts too

### WMI Filters
- Query-based filtering (client-side)
- Example: Apply GPO only if Windows 10+ is installed

```xml
<WmiFilter xmlns="http://schemas.microsoft.com/GroupPolicy/2006/02/WmiFilter">
  <Name>Win10Only</Name>
  <Enabled>true</Enabled>
  <Query>
    <Namespace>root\CIMV2</Namespace>
    <QueryString>SELECT * FROM Win32_OperatingSystem WHERE Version LIKE "10.%"</QueryString>
  </Query>
</WmiFilter>
```

```powershell
# Create WMI filter
New-GPWmiFilter -Name "Win10Only" -LiteralPath "LDAP://CN=Win10Only,CN=WMIPolicy,CN=System,DC=domain,DC=com"
```

- **Troubleshooting WMI**: Event ID 8003 in `Applications and Services Logs → Microsoft → Windows → GroupPolicy → WmiFilter`




###Policy Areas

### Password Policy (GPO)
- Location: Computer Configuration → Windows Settings → Security Settings → Account Policies → Password Policy
- Minimum password length
- Password complexity
- Enforce password history
- Maximum password age
- Minimum password age

### Security Policy
- Location: Computer Configuration → Windows Settings → Security Settings
- User Rights Assignment
- Security Options
- Local Policies (Audit, User Rights)

### User Rights Assignment Examples
| Policy | Common Setting |
|--------|---------------|
| Allow log on locally | Administrators, IT group |
| Access this computer from network | Authenticated Users, IT group |
| Deny log on locally | Guest, Service accounts |
| Force logoff when logon hours expire | Administrators |

### Windows Firewall GPO
- Computer Configuration → Administrative Templates → Network → Network Connections → Windows Firewall
- Per profile (Domain, Private, Public) settings
- Inbound/outbound rules

### Registry GPO
- Deploy registry settings via Administrative Templates
- Or use Group Policy Preferences (GPP) → Registry
- **Security concern**: GPP passwords in SYSVOL (known issue, migrate to preference items with item-level targeting)

### Drive Mapping
- GPP → Drive Maps
- Item-level targeting for conditional mapping
- Reconnect option, update at logon

### Software Deployment
- **Assignment**: Auto-install at startup (computer) or login (user)
- **Publishing**: User installs manually when needed
- **MSI/GPO software**: Placed in SYSVOL software folder
- Check `C:\Windows\Logs\Windows\WindowsUpdate` for installation logs

### Logon/Logoff Scripts
- Scripts in SYSVOL `\scripts` folder
- Run order: Logon = computer startup scripts → user logon scripts
- PowerShell scripts preferred (migration from batch)

### Computer Startup/Shutdown Scripts
- Run in SYSTEM context
- Good for security baselines, drive mapping, registry settings

### Folder Redirection
- Redirects folder (e.g., Documents, Desktop) to network share
- **Basic**: Redirects to same folder on target share
- **Redirect to per-user folder**: Creates subfolder per user
- Permissions: Creator/Owner should have full control
- Common: Desktop, Documents, Pictures, AppData

### Administrative Templates
- Two types:
  - **Template** (.adm/.admx): Classic templates
  - **Modern** (.admx): XML-based, centralized in AD (Central Store: `\\domain.com\SYSVOL\domain.com\Policies\PolicyDefinitions`)
- **Policy vs Preference**: Policy = enforce; Preference = configure (can be overridden)




###Troubleshooting GPO

### GPUpdate Issues
```powershell
# Force update
gpupdate /force
# Check for errors in event log
Get-WinEvent -LogName "Application" -FilterHashtable @{ProviderName="Microsoft-Windows-GroupPolicy"} | Where-Object {$_.Id -eq 4016} -ErrorAction SilentlyContinue
```

### GPRESULT Analysis
```powershell
# Full report
gpresult /h C:\gpreport.html /f

# RSoP in PowerShell
Get-GPResultantSetOfPolicy -Report Html -Path C:\gpreport.html

# RSoP Planning mode
gpresult /r /scope:computer
rsop.msc
```

### Common Issues & Solutions
| Issue | Check |
|-------|-------|
| GPO not applying | gpresult → security filtering → WMI filter → link enabled |
| GPO applying to wrong user | Loopback processing setting |
| Slow logon | AD replication, network, script timeout, GPExtensions |
| GPExtensions failure | Event Viewer → Applications and Services Logs → GroupPolicy |
| SYSVOL replication | DFSR health, SYSVOL replication status |
| Permission on GPO | Requires "Read" and "Apply Group Policy" |

### RSoP Troubleshooting Methodology
1. Check `gpresult` output for "Denies" / "Not Applied"
2. Check GPO status (enabled, linked, WMI filter)
3. Check security filtering
4. Check WMI filter result
5. Check inheritance and enforcement
6. Check SYSVOL replication (GPT version match)

### Event Viewer for GPO
- **Path**: Applications and Services Logs → Microsoft → Windows → GroupPolicy → Operational
- **Event IDs**:
  - 4016: GPO processing started
  - 4017: GPO processed successfully
  - 4018: GPO failed
  - 4029: WMI filter result





---

## PART 4 — DNS ⭐ (Very High Priority)

> *DNS is the most critical infrastructure service — AD, Kerberos, LDAP, and everything depends on it.*


###DNS Fundamentals

### DNS Hierarchy
```
.                    (Root - ".")
├── com/             (TLD - Top Level Domain)
│   ├── example.com  (Domain)
│   │   └── www.example.com  (Host)
│   └── org/
├── net/
└── uk/
    └── co.uk/
```

### DNS Resolution Types
| Type | Description |
|------|-------------|
| **Authoritative DNS** | Holds the actual zone data (answer for "this is the authority") |
| **Recursive DNS** | Resolves queries on behalf of clients; queries other DNS servers |
| **Forwarder** | Recursive resolver that forwards non-local queries to specific DNS servers |
| **Resolver** | Client-side DNS resolver (DNS Client service) |
| **Caching Resolver** | Stores previous query results (TTL-based) |

### Key Terms
| Term | Description |
|------|-------------|
| **FQDN** | Fully Qualified Domain Name (e.g., `server1.domain.com.`) |
| **TTL** | Time To Live — seconds a record is cached before refresh |
| **Root** | Top of DNS hierarchy (13 root server clusters, a.root-servers.net through m.root-servers.net) |
| **TLD** | Top Level Domain (.com, .org, .net, .uk) |

### DNS Server Types
- **Primary (Master)**: Writeable copy of zone
- **Secondary (Slave)**: Read-only copy, pulls via zone transfer (AXFR/IXFR)
- **Stub Zone**: Contains only NS records + glue, for delegation
- **Forward Lookup Zone**: Name → IP (A, AAAA records)
- **Reverse Lookup Zone**: IP → Name (PTR records)
- **Conditional Forwarder**: Forwards specific domain queries to designated DNS server

### Root Hints vs Forwarders
- **Root Hints**: List of root DNS servers; used when no forwarder configured
- **Forwarders**: First stop for non-authoritative queries; common in enterprise (forward to internal DNS, which forwards to external)
- **DNS hierarchy**: Client → Local DNS → Forwarders → Root servers (if no forwarder)




###Record Types

### Record Type Reference
| Type | Purpose | Example |
|------|---------|---------|
| **A** | IPv4 address | A record → 192.168.1.10 |
| **AAAA** | IPv6 address | AAAA record → 2001:db8::1 |
| **CNAME** | Alias to another name | www → server1.example.com |
| **PTR** | Reverse lookup (IP → name) | 1.1.168.192.in-addr.arpa → PTR server1.example.com |
| **MX** | Mail server routing | domain.com → MX 10 mail.domain.com |
| **NS** | Name server | domain.com → NS ns1.domain.com |
| **SOA** | Start of Authority (zone info) | domain.com → SOA ns1 admin (serial, refresh, retry, expire, TTL) |
| **SRV** | Service location (used by AD/Kerberos) | _ldap._tcp.dc._msdcs.domain.com → SRV 0 100 389 dc1.domain.com |
| **TXT** | Text (SPF, DKIM, etc.) | domain.com → TXT "v=spf1 mx" |

### SOA Fields
```
Serial:      Version number (increment on changes; YYYYMMDDNN is common format)
Refresh:     How often secondary checks for updates
Retry:       How often secondary retries if refresh failed
Expire:      When secondary stops answering queries if it can't reach primary
Minimum TTL: Default TTL for negative caching (also NXDOMAIN caching)
```

### AD-Specific DNS Records
| Record | Purpose |
|--------|---------|
| `_msdcs.domain.com` | Contains AD-specific info (GC, DC locations) |
| `_ldap._tcp.dc._msdcs.domain.com` (SRV) | Locates domain controllers |
| `_ldap._tcp.Default-First-Site-Name._sites.dc._msdcs.domain.com` | Site-specific DC location |
| `_kerberos._tcp.domain.com` (SRV) | Kerberos KDC location |
| `_kerberos._tcp.Default-First-Site-Name._sites.domain.com` (SRV) | Site-specific KDC |
| `_gc._tcp.domain.com` (SRV) | Global Catalog location |
| `A` record for each DC | DC IP addresses |
| `CNAME` for GC | Aliases for GC servers |




###AD DNS Integration

### AD-Integrated Zones
- Stored in AD (replicated via AD replication — faster than file replication)
- Replication scope: Forest, Domain, or specific group
- **Secure dynamic updates**: Only authenticated AD computers can register
- Auto-created during AD DS installation

### Dynamic Updates
- **Secure**: Only domain-joined computers can update (default for AD zones)
- **Non-Secure**: Any client can update (risk of poisoning; not recommended)
- **Registration**: DHCP can register A/PTR records for clients (registrar: DHCP server)
- **DNS scavenging**: Removes stale records after TTL expires

### Secure Dynamic Update Process
1. DC authenticated with machine account or user credentials
2. DNS record registered in the AD DNS partition
3. Replicated to all DCs via AD replication
4. TTL determines cache duration

### DC Discovery via DNS
- Client queries DNS for `_ldap._tcp.dc._msdcs.domain.com` SRV records
- Returns list of DCs with: priority, weight, port (389), host
- Client picks one based on site/site affinity, random selection for weighted
- If site mismatch → choose next closest site




###DNS Troubleshooting

### Essential Commands
```powershell
# NSLookup (manual query)
nslookup server1.domain.com
nslookup 192.168.1.10 (reverse lookup)
nslookup -type=SRV _ldap._tcp.dc._msdcs.domain.com
nslookup -type=ANY domain.com
nslookup /server=10.0.0.1 query   (query specific server)

# Resolve-DnsName (preferred PowerShell)
Resolve-DnsName -Name server1.domain.com
Resolve-DnsName -Name _ldap._tcp.dc._msdcs.domain.com -Type SRV
Resolve-DnsName -Name server1.domain.com -Server 8.8.8.8   (query specific server)

# IP config DNS
ipconfig /displaydns       (show cached DNS)
ipconfig /flushdns         (clear DNS cache)
ipconfig /registerdns      (re-register DNS)
ipconfig /showdns          (show DNS cache - newer Windows)

# DNS server commands
Get-DnsServerZone           (PowerShell - list zones)
Get-DnsServerResourceRecord -ZoneName "domain.com" -Name "server1"
Add-DnsServerResourceRecord -ZoneName "domain.com" -Name "server1" -IPv4Address "192.168.1.10"
```

### Common Issues & Troubleshooting
| Issue | Diagnosis | Fix |
|-------|-----------|-----|
| **DNS resolution fails** | `Resolve-DnsName server1` on client | Check DNS server, forwarder, record existence |
| **Wrong DNS server** | `ipconfig /all` shows unexpected DNS | Check DHCP scope options, DHCP reservation, manual config |
| **Missing record** | `Resolve-DnsName server1` fails | `ipconfig /registerdns`, check DNS scavenging, check secure dynamic updates |
| **Duplicate record** | Two records returned for same name | Check DNS admin console, identify rogue registration, purge duplicates |
| **Dynamic registration fails** | Client can't register | Check secure updates, DNS permissions, client machine account, DHPO option 81 |
| **Conditional forwarder issue** | External domain queries fail | Check forwarder IP, test with `Resolve-DnsName -Server`, check firewall port 53 |
| **DNS delegation issue** | Subdomain queries fail | Check NS records, glue records (A records for NS hosts), parent domain configuration |
| **Cached wrong record** | `Resolve-DnsName` returns old data | `ipconfig /flushdns`, check TTL, verify DNS server answer |

### DNS Troubleshooting Flow
1. Check client DNS settings (`ipconfig /all`)
2. Query DNS server directly (`nslookup domain.com <dns-server-ip>`)
3. Check zone exists on server
4. Check record exists (`Resolve-DnsName` on server)
5. Check zone type (primary/secondary/stub)
6. Check zone transfers
7. Check forwarding chain
8. Check root hints (if no forwarder)
9. Check firewall (UDP/TCP 53)
10. Check DNS scavenging settings

### DNS Delegation
- Parent zone delegates subdomain to authoritative DNS servers
- Requires: NS records + Glue records (A records for the delegated NS hosts)
- Example: `subdomain.domain.com` → NS ns1.subdomain.com (ns1.subdomain.com's A record must exist in parent zone)





---

## PART 5 — DHCP

> *DHCP provides the network configuration that enables all other services.*


###Core Topics

### Key Concepts
| Term | Description |
|------|-------------|
| **Scope** | Range of IP addresses for a subnet |
| **Lease** | Time period IP is assigned to client |
| **Reservation** | Permanent IP for specific MAC address |
| **Exclusion** | Range excluded from scope (e.g., .1-.10 for servers) |
| **Options** | Additional config (gateway, DNS, domain suffix) |
| **Scope Activation** | Must be activated to hand out IPs |
| **DHCP Relay (IP Helper)** | Forwards DHCP broadcasts across subnets |
| **Failover** | Two DHCP servers for HA |

### DHCP Scope
```powershell
# DHCP Server module
Import-Module DhcpServer

# Add scope
Add-DhcpServerv4Scope -Name "Main Office" -StartRange 192.168.10.101 -EndRange 192.168.10.254 -SubnetMask 255.255.255.0 -State Active

# Exclusion
Set-DhcpServerv4Scope -ScopeId 192.168.10.0 -ExclusionRange 192.168.10.1,192.168.10.50

# Reservation
Add-DhcpServerv4Reservation -ScopeId 192.168.10.0 -IPAddress 192.168.10.10 -MacAddress "00-11-22-AA-BB-CC"

# Check lease
Get-DhcpServerv4Lease -ScopeId 192.168.10.0

# Release/Renew
Ipconfig /release
Ipconfig /renew
```

### Lease Process (DORA)
1. **Discover**: Client broadcasts "Who has 192.168.10.x?"
2. **Offer**: DHCP server responds "I can give you 192.168.10.100"
3. **Request**: Client requests "I want 192.168.10.100"
4. **Acknowledge**: Server confirms "Here you go"

### DHCP Options
| Option | Purpose | Example |
|--------|---------|---------|
| **003 Router** | Default gateway | 192.168.10.1 |
| **006 DNS** | DNS servers | 192.168.10.10, 192.168.10.11 |
| **015 DNS Suffix** | Domain suffix | domain.com |
| **044 WINS** | NetBIOS name server | (legacy) |
| **066 Boot Server** | TFTP server for PXE | 192.168.10.50 |
| **067 Boot File** | Boot file for PXE | `\boot\pxeboot.n12` |
| **042 NTP** | Time server | 192.168.10.10 |

### DHCP Relay (IP Helper)
- On router/L3 switch, configure `ip helper-address` pointing to DHCP server
- Router forwards broadcast as unicast to DHCP server
- Must be configured on each VLAN/subnet
- Common on VLAN interfaces in switches

### DHCP Failover
- **Hot Standby**: One server active, one passive; all leases on active server
- **Load Balance**: Both servers actively assign leases (split: 50/50, 80/20, etc.)
- **State**: Normal, Communication Lost, Quorum Lost, Partner Down, Shut Down

```powershell
# DHCP failover
Add-DhcpServerv4Failover -ComputerName DHCP01 -PartnerServer DHCP02 -Name "MainFailover" -SharedSecret "Password1" -Mode HotStandby -ReservePercent 10 -ScopeId 192.168.10.0

# Check failover status
Get-DhcpServerv4Failover -ComputerName DHCP01

# Monitor
Get-DhcpServerv4Failover -ComputerName DHCP01 | Select-Object *
```




###DHCP Troubleshooting

### Common Issues
| Issue | Cause | Resolution |
|-------|-------|-----------|
| **APIPA (169.254.x.x)** | No DHCP response received | Check DHCP server, relay, network connectivity |
| **No IP address** | DHCP server down, scope inactive, full | Check server, activate scope, free addresses |
| **Wrong gateway** | Option 003 misconfigured | Check option on DHCP server |
| **Wrong DNS** | Option 006 misconfigured | Check option on DHCP server |
| **Scope exhaustion** | Too few addresses / lease too long | Expand scope or shorten lease |
| **DHCP authorization** | DHCP server not authorized in AD | Authorize in DHCP console or `Add-DhcpServer` |
| **Relay failure** | IP helper not configured on router | Configure `ip helper-address` on interface |
| **Duplicate IP** | Two devices with same IP; or DHCP conflict detection off | Enable DHCP conflict detection; check with `arp -a` |
| **Lease not released** | Client doesn't properly release | DHCP server cleanup; `arp -a` check |

### DHCP Authorization
- AD-integrated DHCP servers must be **authorized** in AD
- Unauthorized DHCP server = cannot hand out leases
- Authorize via: DHCP console → right-click server → Authorize
- Or: `Add-DhcpServer -DnsName DHCP01.domain.com -IPAddress 192.168.10.10`

### APIPA Troubleshooting
- APIPA: 169.254.0.0/16
- Server assigns itself address when DHCP fails
- Not routable (link-local only)
- Check:
  - DHCP server operational
  - DHCP relay configured (if across VLAN)
  - Network cable/connectivity
  - Port security on switch





---

## PART 6 — IP Networking & Subnetting ⭐ (Subnets are Required)

> *Foundation for all networking — must be rock solid for enterprise operations.*


###IPv4 Fundamentals

### IP Address Structure
```
Network ID     | Host ID
192.168.10.    | 64  (with /26 mask, first 26 bits = network, last 6 bits = host)
```

### IP Classification (Modern View)
| Type | Range | Purpose |
|------|-------|---------|
| **Class A** | 1.0.0.0 - 126.255.255.255 | Large networks (10.x.x.x private) |
| **Class B** | 128.0.0.0 - 191.255.255.255 | Medium networks (172.16.0.0 private) |
| **Class C** | 192.0.0.0 - 223.255.255.255 | Small networks (192.168.x.x private) |
| **Class D** | 224.0.0.0 - 239.255.255.255 | Multicast |
| **Class E** | 240.0.0.0 - 255.255.255.255 | Experimental |

### Special Addresses
| Address | CIDR | Purpose |
|---------|------|---------|
| **127.0.0.1** | 127.0.0.0/8 | Loopback (localhost) |
| **169.254.0.1** | 169.254.0.0/16 | APIPA (auto-assigned if DHCP fails) |
| **0.0.0.0** | /0 | Default route / "this network" |
| **255.255.255.255** | /32 | Limited broadcast |
| **255.255.255.0** | /24 | Subnet-directed broadcast (network-specific) |
| **Private**: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 | RFC 1918 | Private (non-routable) |
| **Public**: everything else | — | Routable on internet |

### Private IP Ranges (RFC 1918)
- **10.0.0.0 – 10.255.255.255** (/8) — 16.7M addresses
- **172.16.0.0 – 172.31.255.255** (/12) — 1M addresses
- **192.168.0.0 – 192.168.255.255** (/16) — 65K addresses




###Subnetting Deep Dive

### Core Concept
- **CIDR (Classless Inter-Domain Routing)**: /x notation where x = number of network bits
- **Standard boundaries**: /8, /16, /24 (easy to remember)
- **Subdivisions**: /25, /26, /27, /28, /29, /30, /31, /32

### Quick Reference Table
| CIDR | Subnet Mask | Usable Hosts | Block Size | Common Use |
|------|------------|-------------|------------|-----------|
| **/8** | 255.0.0.0 | 16,777,214 | 16,777,216 | Huge private networks |
| **/16** | 255.255.0.0 | 65,534 | 65,536 | Large private networks |
| **/24** | 255.255.255.0 | 254 | 256 | Common LAN |
| **/25** | 255.255.255.128 | 126 | 128 | Split /24 in half |
| **/26** | 255.255.255.192 | 62 | 64 | Split /24 into 4 |
| **/27** | 255.255.255.224 | 30 | 32 | Split /24 into 8 |
| **/28** | 255.255.255.240 | 14 | 16 | Small subnet |
| **/29** | 255.255.255.248 | 6 | 8 | Small point-to-point |
| **/30** | 255.255.255.252 | 2 | 4 | Point-to-point links |
| **/31** | 255.255.255.254 | 2 (special) | 2 | Point-to-point (RFC 3021) |
| **/32** | 255.255.255.255 | 1 (host route) | 1 | Single host / loopback |

### Subnetting Example: 192.168.10.64/26 ⭐

**Given**: `192.168.10.64/26`

**Calculation**:
- **/26** means first 26 bits = network, last 6 bits = host
- Subnet mask: `255.255.255.192` (last octet: 128+64 = 192)
- Block size: 256 - 192 = **64**
- **Network address**: 192.168.10.**64** (multiple of 64)
- **Broadcast address**: 192.168.10.**127** (64 + 64 - 1)
- **First usable IP**: 192.168.10.**65** (network + 1)
- **Last usable IP**: 192.168.10.**126** (broadcast - 1)
- **Number of usable hosts**: 2^6 - 2 = **62** (2 host bits, minus network and broadcast)
- Next subnet starts at: 192.168.10.**128**

**Calculation shortcut**:
```
Third octet of 192 → Binary: 11000000 (128 + 64 = 192)
                    ↑↑↑↑↑↑↑↑
                    Network bits ↑  ↑ Host bits (64+128 = 192 = /26)
```

```powershell
# Verify network details (Windows)
Get-NetIPAddress -IPAddress 192.168.10.64 | Select-Object IPAddress, PrefixLength, PrefixOrigin

# Calculate in PowerShell
$ip = [IPAddress]::Parse("192.168.10.64")
$mask = 26
$network = $ip.Address -shr (32 - $mask) -shl (32 - $mask)
$bcast = $network -bor ((1 -shl (32 - $mask)) - 1)
Write-Host "Network: $network"
Write-Host "Broadcast: $bcast"
Write-Host "Usable hosts: $((1 -shl (32 - $mask)) - 2)"
```

### How to Calculate Any Subnet
1. **Identify CIDR**: e.g., /26
2. **Subnet mask**: Convert /26 to octets (255.255.255.192)
3. **Block size**: 256 - interesting octet value (256 - 192 = 64)
4. **Network address**: Round IP DOWN to nearest block size multiple
5. **Broadcast**: Network + block size - 1
6. **First usable**: Network + 1
7. **Last usable**: Broadcast - 1
8. **Total hosts**: 2^(32 - CIDR) - 2

### VLSM (Variable Length Subnet Masking)
- Different masks for different subnets in same network
- Example: 192.168.10.0/24 split into /26, /26, /27, /27, /30...
- Allocate largest subnets first to avoid waste




###Networking Concepts

### Core Concepts
| Concept | Description |
|---------|-------------|
| **Gateway/Router** | Routes between networks (L3 device) |
| **Routing** | Process of forwarding packets between networks |
| **Static Route** | Manually configured route |
| **Default Route** | Gateway of last resort (0.0.0.0/0) |
| **NAT** | Network Address Translation (private → public) |
| **VLAN** | Virtual LAN (L2 segmentation) |
| **Trunk** | Link carrying multiple VLANs (tagged) |
| **MTU** | Maximum Transmission Unit (default 1500 bytes) |
| **ARP** | Address Resolution Protocol (IP → MAC) |
| **MAC** | Media Access Control (physical address) |
| **TCP** | Connection-oriented, reliable |
| **UDP** | Connectionless, fast |

### Routing
```powershell
# View routing table
route print

# Add static route
route add 10.0.0.0 mask 255.0.0.0 192.168.10.1
route add 0.0.0.0 mask 0.0.0.0 192.168.10.1    (default route)
route -p add ...    (persistent)

# Delete static route
route delete 10.0.0.0
```

### NAT
- **Outbound**: Private IP → Public IP (PAT/NAPT common)
- **Inbound**: Requires port forwarding or DMZ
- **Hyper-V**: Virtual switches can use NAT (NAT network)
- **Windows**: Internet Connection Sharing (ICS), or Routing role

### VLANs
- VLAN ID: 1-4094
- Native VLAN: Untagged traffic (VLAN 1 default, should change)
- **802.1Q**: Tagging standard
- **Trunk**: Carries multiple VLANs, uses 802.1Q tags
- **Access**: Single VLAN, untagged
- **VLAN Trunk Protocol (VTP)**: Cisco proprietary (v1/v2/v3)

### TCP vs UDP
| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Handshake (SYN/SYN-ACK/ACK) | None |
| Reliability | Guaranteed delivery | Best effort |
| Ordering | Maintained | No guarantee |
| Speed | Slower | Faster |
| Use cases | Web, email, file transfer | DNS, DHCP, streaming, VoIP |

### Important Ports
| Port | Protocol | Service |
|------|----------|---------|
| **22** | TCP | SSH |
| **25** | TCP | SMTP |
| **53** | TCP/UDP | DNS |
| **67/68** | UDP | DHCP |
| **88** | TCP/UDP | Kerberos |
| **123** | UDP | NTP |
| **135** | TCP/UDP | RPC Endpoint Mapper |
| **389** | TCP | LDAP |
| **443** | TCP | HTTPS |
| **445** | TCP | SMB |
| **636** | TCP | LDAPS |
| **3268/3269** | TCP | Global Catalog (LDAP/LDAPS) |
| **3389** | TCP/UDP | RDP |
| **5985/5986** | TCP | WinRM (HTTP/HTTPS) |
| **49152-65535** | TCP | Dynamic RPC ports |

```powershell
# Check listening ports
netstat -ano
Get-NetTCPConnection -State Listen

# Check open firewall ports
Get-NetFirewallRule | Where-Object {$_.Enabled -eq 'True'} | Get-NetFirewallPortFilter
```

### Network Troubleshooting Commands
```powershell
# Connectivity
ping 192.168.10.1 -t          (continuous ping)
ping 192.168.10.1 -l 65500    (large ping, test MTU)
ping google.com               (FQDN test)

# Route tracing
tracert 8.8.8.8
pathping 8.8.8.8             (slow but detailed)

# DNS
nslookup google.com
Resolve-DnsName google.com
ipconfig /displaydns
ipconfig /flushdns

# Network details
ipconfig /all
ipconfig /release
ipconfig /renew
ipconfig /registerdns
arp -a                          (ARP cache)
route print                     (routing table)
netstat -ano                    (connections + PID)
Test-NetConnection -ComputerName google.com -Port 443
```





---

## PART 7 — PKI / Certificate Services

> *JD explicitly requires Certificate Services / PKI knowledge.*


###PKI Fundamentals

### Core Concepts
| Term | Description |
|------|-------------|
| **Public Key** | Shared openly; used to encrypt or verify |
| **Private Key** | Kept secret; used to decrypt or sign |
| **Encryption** | Convert plaintext to ciphertext |
| **Digital Signature** | Hash encrypted with private key; proves authenticity and integrity |
| **Certificate** | Electronic document binding public key to identity |
| **CSR** | Certificate Signing Request (request from requester to CA) |
| **CA** | Certificate Authority; issues and signs certificates |
| **Root CA** | Top-level CA; self-signed or trusted by OS/browser trust stores |
| **Intermediate CA** | Issues certificates; signed by Root CA; provides additional layer |
| **Certificate Chain** | Root → Intermediate(s) → End-entity certificate |
| **Trust** | Root CA in trust store → trust all certificates it signs |

### Public/Private Key Encryption
```
[Sender] encrypts with [Receiver's Public Key] → ciphertext → [Receiver] decrypts with [Private Key]
[Signer] signs with [Signer's Private Key] → signature → [Verifier] verifies with [Signer's Public Key]
```

### Certificate Contents
- Subject (CN, O, OU, C)
- Issuer (CA name)
- Validity dates (Not Before / Not After)
- Serial Number
- Public Key (algorithm + key)
- Extensions (SAN, EKU, Key Usage, etc.)
- Signature Algorithm (SHA-256, etc.)
- Thumbprint (fingerprint)

### Key Terms
| Term | Description |
|------|-------------|
| **SAN** | Subject Alternative Name (multiple domains/IPs in one cert) |
| **CN** | Common Name (primary name, e.g., server1.domain.com) |
| **EKU** | Enhanced Key Usage (purpose: server auth, client auth, code signing) |
| **Key Usage** | Technical restriction (digital signature, key encipherment, etc.) |
| **RSA** | Asymmetric algorithm (most common) |
| **ECC** | Elliptic Curve Cryptography (smaller keys, faster) |
| **SHA-256** | Hash algorithm (replacement for deprecated SHA-1) |
| **TLS** | Transport Layer Security (successor to SSL) |

### AD CS (Certificate Services)
- Enterprise CA: Integrated with AD; auto-enrollment; publishes to AD
- Standalone CA: Standalone; manual approval; not AD-integrated
- **Enterprise CA**: Can be Root or Subordinate
- **Standalone CA**: Usually Root (for PKI hierarchy)

### Enterprise vs Standalone
| Feature | Enterprise CA | Standalone CA |
|---------|--------------|---------------|
| AD integration | Yes | No |
| Auto-enrollment | Yes | No |
| Certificate templates | Yes | No (uses inf file) |
| AD publishing | Yes | No |
| Schema extension | Yes | No |
| Use case | Internal PKI | External/Root CA |

### Certificate Templates
- Built-in templates: Web Server, User, Computer, Domain Controller, SubCA, Root CA
- Custom templates: Define specific EKUs, key sizes, validity periods
- Auto-enrollment: Configured via GPO (Computer/User Configuration → Windows Settings → Security Settings → Public Key Policies → Certificate Services Client Auto-Enrollment)

### CRL & OCSP
- **CRL (Certificate Revocation List)**: List of revoked certificates; published by CA
- **OCSP (Online Certificate Status Protocol)**: Real-time check if cert is revoked
- **CRL Distribution Points**: Where CRLs are published (HTTP, LDAP, etc.)
- **AIA (Authority Information Access)**: Where to find OCSP and CA certificate




###Certificate Concepts

### Certificate Types
| Type | Description | Key Usage |
|------|-------------|-----------|
| **Server/SSL** | Web server authentication | Server Authentication (EKU 1.3.6.1.5.5.7.3.1) |
| **Client** | User/device authentication | Client Authentication (EKU 1.3.6.1.5.5.7.3.2) |
| **Code Signing** | Verify software publisher | Code Signing (EKU 1.3.6.1.5.5.7.3.3) |
| **Email (S/MIME)** | Email encryption/signing | Email Protection (EKU 1.3.6.1.5.5.7.3.4) |
| **SubCA** | Intermediate CA | Certificate Signing |
| **Root CA** | Root authority | Certificate Signing |

### SSL/TLS Handshake (Simplified)
1. Client sends "Hello" (supported cipher suites, TLS version)
2. Server responds with "Hello" (chosen cipher, sends certificate)
3. Client verifies certificate chain (CRL/OCSP check)
4. Client generates session key, encrypts with server's public key
5. Server decrypts with private key
6. Both sides use symmetric session key for encrypted communication

### Key Lengths
| Algorithm | Min Recommended | Legacy |
|-----------|----------------|--------|
| **RSA** | 2048-bit (2021+) | 1024-bit (deprecated) |
| **ECC** | 256-bit | 160-bit (SECG) |
| **SHA** | SHA-256 | SHA-1 (deprecated) |

### TLS Versions
| Version | Status | Features |
|---------|--------|----------|
| SSL 2.0 | Insecure | Deprecated |
| SSL 3.0 | Insecure | Deprecated |
| TLS 1.0 | Insecure | Deprecated |
| TLS 1.1 | Insecure | Deprecated |
| **TLS 1.2** | Secure | Current standard |
| **TLS 1.3** | Most secure | 0-RTT, improved security |




###Troubleshooting PKI

### Tools
```powershell
# Certificate management
certlm.msc          # Local machine certificates
certmgr.msc         # Current user certificates

# Certutil
certutil -store My                (list personal certs)
certutil -store Root              (list trusted root CAs)
certutil -viewstore -user My      (view user store)
certutil -repairstore My <serial> (repair cert)
certutil -verifyCTL               (verify CTL)
certutil -ca-info                 (CA info)
certutil -ping                    (test CA responsiveness)
certutil -getchain                (build certificate chain)

# Generate CSR
certreq -new request.inf request.req
certreq -submit request.req

# Import
certutil -importPFX -p "Password" my cert.pfx

# PKIView
PKIView.exe                 (shows PKI status - diagnostic tool from Microsoft)
```

### Common Issues & Fixes
| Issue | Cause | Resolution |
|-------|-------|-----------|
| **Certificate expired** | Validity period ended | Renew certificate, check auto-enrollment |
| **Certificate not trusted** | Root/Intermediate not in trust store | Install root/intermediate, check CRL |
| **Wrong SAN** | Certificate issued for wrong domain | Re-issue with correct SAN |
| **Missing intermediate** | Chain incomplete | Install intermediate, check CRL/AIA publication |
| **CRL unavailable** | CRL distribution point unreachable | Check CDP URLs, network, CRL publication |
| **Auto-enrollment failure** | GPO misconfiguration | Check certificate policy, event logs |
| **Private key missing** | Cert exported without private key | Re-issue or re-import from PFX |
| **Certificate chain failure** | Missing root or intermediate | Install missing certs, verify chain with `certutil -getchain` |
| **TLS handshake failure** | TLS version mismatch, cipher mismatch, expired cert | Align TLS settings, check cipher suites, verify cert validity |

### Troubleshooting Steps
1. Check certificate validity (expiration date)
2. Check certificate chain (`certutil -getchain`)
3. Check if Root/Intermediate CA is in trust store
4. Check CRL distribution points (can client reach them?)
5. Check OCSP responders (can client reach them?)
6. Check auto-enrollment GPO settings
7. Check Event Viewer → Applications and Services Logs → Certificate Services Client





---

## PART 8 — VMware vSphere

> *Second major pillar of the JD.*


###vSphere Architecture

### Core Components
| Component | Description |
|-----------|-------------|
| **ESXi** | Hypervisor (bare-metal installed on server hardware) |
| **vCenter** | Central management (requires Windows/Linux VM) |
| **vSphere Client** | HTML5 management interface (built into vCenter) |
| **Datacenter** | Logical container for clusters/hosts |
| **Cluster** | Group of ESXi hosts (shared resources, HA, DRS) |
| **Host** | ESXi host |
| **VM** | Virtual machine |
| **Datastore** | Storage volume (VMFS, NFS) mounted on ESXi |
| **Network** | vSwitches, port groups, distributed switches |
| **Resource Pool** | Logical allocation of CPU/memory |

### ESXi
- **Type**: Type 1 hypervisor (runs directly on hardware)
- **Management**: ESXi Shell, SSH, DCUI (Direct Console User Interface), vSphere Client
- **Management network**: VMkernel adapter (vmk0 default)
- **VMkernel ports**: Management, vMotion, Storage, vSAN, Fault Tolerance
- **Datastores**: VMFS (local/SAN), NFS (network share), vSAN (distributed)
- **HBA**: Host Bus Adapter (Fibre Channel/iSCSI connectivity)
- **Drivers**: Use VMware-compatible drivers; ESXi has in-guest driver improvements
- **Firmware**: Update via vendor's tool (Dell iDRAC, HPE iLO, Lenovo XClarity)

### vCenter
- **Platform Services Controller (PSC)**: Identity, licensing, SSO (pre-7.0, embedded in 7.0+)
- **Embedded/External PSC**: vCenter can have embedded PSC (small env) or external (large env)
- **vCenter HA**: Active/Passive for vCenter availability
- **Inventory hierarchy**: Datacenter → Cluster → Host → VM

### Templates & Content Libraries
- **Template**: VM converted to template (golden image); can deploy VMs from template
- **Content Library**: Central repository for vSphere objects (templates, ISOs, scripts, OVF)
- **Subscribed library**: Pulls content from remote/published library
- **Lifecycle Manager**: Update ESXi hosts via images (vSphere Lifecycle Manager - Image-based management, replacing original CIM)




###VM Administration

### VM Creation
```powershell
# VMware PowerCLI
Connect-VIServer -Server vcenter.domain.com
New-VM -Name "MyVM" -Template "Windows2022" -Datastore "Datastore1" -VMHost "ESXi01.domain.com" -NetworkName "Production" -DiskGB 100

# Clone VM
New-VM -Name "CloneVM" -VM "SourceVM" -Datastore "Datastore1" -VMHost "ESXi01.domain.com"
```

### VM Components
| Component | Specification | Notes |
|-----------|--------------|-------|
| **CPU** | Cores, sockets, cores per socket | Must match ESXi host CPU compatibility |
| **Memory** | RAM amount | Can be hot-add (dynamic memory) |
| **NIC** | VMXNET3 (recommended), E1000e | Multiple NICs, different port groups |
| **Disk** | Thin/Thick provision, eager/lazy zero | Thin saves space, thick faster |
| **CD/DVD** | ISO mount | Datastore ISO or guest device |

### VM Operations
```powershell
# Power operations
Start-VM -Name "MyVM"
Stop-VM -Name "MyVM" -Confirm:$false
Restart-VM -Name "MyVM"
Suspend-VM -Name "MyVM"

# Snapshot
New-Snapshot -VM "MyVM" -Name "BeforeUpdate" -Description "Pre-patching snapshot"
Get-Snapshot -VM "MyVM"
Remove-Snapshot -Snapshot (Get-Snapshot -VM "MyVM" -Name "BeforeUpdate") -Confirm:$false

# VMware Tools
Update-Tools -VM "MyVM"                    (requires network connection)
Update-Tools -VM "MyVM" -NoReboot:$true
```

### VM Hardware Version
- Updated when vSphere version upgrades
- Newer versions support newer VM features
- Must power off and reconfigure VM to upgrade hardware version

### Hot-Add/Hot-Plug
- CPU hot-add: Enable in VM settings → CPU hot-add
- Memory hot-add/remove: Enable → Memory hot-add/remove
- NIC hot-add: Enable → PCI device hot-add
- Hot-plug requires guest OS support

### Snapshot Best Practices
- **Purpose**: Temporary protection for tasks (patching, upgrades, testing)
- **NOT a backup**: Snapshots don't back up data; they capture VM state
- **Growth**: Delta disks grow over time (can reach 100% of original disk)
- **Consolidation**: Merge delta disks back into base disk
- **Performance**: Multiple snapshots → I/O latency; chain length should be ≤ 3 ideally
- See PART 12 for more snapshot detail





---

## PART 9 — VMware Networking


###VMware Networking

### Standard vSwitch (vSwitch)
| Feature | Description |
|---------|-------------|
| **Created on** | Each ESXi host |
| **Management** | Per-host configuration |
| **Port groups** | VLAN tagging, security policies |
| **uplinks** | Physical NICs (vmnic0, vmnic1, etc.) |
| **Teaming** | Failover order, load balancing |

### Distributed vSwitch (DVS)
| Feature | Description |
|---------|-------------|
| **Scope** | Cluster-wide |
| **Management** | vCenter managed (central) |
| **Port groups** | Apply across all hosts in cluster |
| **NetFlow** | Traffic analysis |
| **LACP** | Link Aggregation Control Protocol supported |
| **Private VLANs** | Supported |

### Port Group
- **VLAN ID**: VLAN tag (0 = trunk/passthrough, 4095 = virtual)
- **Security**: Promiscuous mode, MAC address changes, forged transmits
- **Traffic shaping**: Bandwidth limits

### VMkernel Adapters
| Purpose | Port | Default |
|---------|------|---------|
| **Management** | N/A | vmk0 (default) |
| **vMotion** | TCP 8000 | Requires dedicated VMkernel |
| **vSAN** | TCP 23451 | Dedicated VMkernel recommended |
| **Fault Tolerance** | TCP 8333, 8334 | Dedicated VMkernel |
| **vSphere Replication** | TCP 8043 | Dedicated VMkernel |
| **Provisioning** | TCP 902 | Can share with management |

### NIC Teaming & Load Balancing
| Policy | Description |
|--------|-------------|
| **Route based on IP hash** | Source IP → physical NIC (LACP/teaming) |
| **Route based on source MAC** | Source MAC → physical NIC |
| **Route based on virtual port** | Port ID → physical NIC |
| **Route based on physical NIC load** | Dynamic load balancing (requires NetApp etc. or distributed switch) |
| **Notify switches** | Send notifications to switch about failover |
| **Failover order** | Active/Standby/Unused |

### Key Settings
| Setting | Default | Recommendation |
|---------|---------|---------------|
| **MTU** | 1500 | 9000 for jumbo frames (all devices must support) |
| **Jumbo Frames** | Disabled | Enable if using iSCSI/vMotion 10GbE+ |

### Troubleshooting
```powershell
# VMware PowerCLI network tests
Test-VMCustomization -VM "MyVM"           (test connectivity)

# Check VM port group
Get-VM "MyVM" | Get-NetworkAdapter | Select-Object NetworkName, PortGroupName

# Check vSwitch
Get-VirtualSwitch -VMHost "ESXi01"
Get-VirtualPortGroup -VMHost "ESXi01"

# Check VMkernel
Get-VMHostNetworkAdapter -VMHost "ESXi01" | Where-Object {$_.Type -eq "vmkernel"}
```

| Issue | Check |
|-------|-------|
| VM cannot communicate | Port group, VLAN, security policies, firewall rules |
| VLAN mismatch | Port group VLAN vs physical switch VLAN |
| Port group issue | Check distributed switch membership, host port group config |
| NIC down | Physical NIC link, ESXi NIC status, cable |
| vSwitch issue | Correct VLAN ID, correct port group assignment |
| MTU mismatch | Check MTU on vSwitch, physical switch, VM (all must match) |
| DNS issue | VM DNS settings, external DNS resolution |
| Gateway issue | VM gateway setting, network routing |
| vMotion failure | VMkernel vMotion port, MTU, network isolation, subnet |



---

## PART 10 — VMware Storage


###VMware Storage

### Storage Types
| Type | Description | Protocol |
|------|-------------|----------|
| **VMFS** | VMware Virtual Machine File System; clustered file system | Block (SAN via iSCSI/FC) |
| **NFS** | Network File System; v3/v4 supported | TCP/IP |
| **vSAN** | Distributed storage across SSD/HDD in cluster | Proprietary (RAID-like) |
| **iSCSI** | Block storage over IP | TCP/IP |
| **Fibre Channel** | High-speed block storage | FC protocol |
| **FC Storage** | SAN via Fibre Channel | FC |

### FC Zoning Concepts
- **Zoning**: FC naming convention to restrict which hosts can access which storage ports
- **Single-initiator zoning**: One host WWN to one/more storage ports
- **Single-initiator-single-target**: Most restrictive
- **Zone sets**: Group zones; only active zone set serves traffic
- **Best practice**: Single-initiator for easy troubleshooting
- **WWPN**: World Wide Port Name (each HBA port)
- **WWNN**: World Wide Node Name (each HBA)

### Datastore Operations
```powershell
# Rescan storage
Get-VMHostStorage -VMHost "ESXi01" -RescanAll
Rescan-VMHostStorage -VMHost "ESXi01"

# Mount/unmount
Get-Datastore -Name "Datastore1" | Set-Datastore -State Mounted
Get-Datastore -Name "Datastore1" | Set-Datastore -State Unmounted

# Extend
Extend-Datastore -Datastore "Datastore1" -NewCapacityGB 500 -ExtendType "dataStoreExt"

# Datastore info
Get-Datastore | Select-Object Name, CapacityMB, FreeSpaceMB, Type, VsanSparseSupported
Get-Datastore -Name "Datastore1" | Get-DatastoreBrowser

# Datastore migration (Storage vMotion)
Move-VDisk -VDisk "MyVM Hard disk 1" -Datastore "Datastore2"

# VM migration (Storage vMotion)
Move-VM -VM "MyVM" -Datastore "Datastore2" -DiskStorageFormat Thin
```

### vSAN Concepts
- **Requirements**: Min 3 hosts (all-flash or hybrid), SSD (cache) + HDD/SSD (capacity)
- **Disk groups**: 1 SSD cache + 1-7 capacity disks per host
- **RAID**: RAID-1 (mirror) for FTT=1, RAID-5/6 for FTT=2/3
- **Storage policies**: Define FTT, fault tolerance, encryption, etc.
- **vSAN HA**: Check health, witness host configuration




###Storage Troubleshooting

| Issue | Diagnosis | Resolution |
|-------|-----------|-----------|
| **Datastore full** | Check free space; grow; delete old snapshots; archive | Extend, migrate VMs, remove snapshots, add disk |
| **APD (All Paths Down)** | All paths to storage fail; VMs stun (stop I/O) | Check HBA, cables, switch, storage processor, multipathing |
| **PDL (Permanent Device Loss)** | Path loss detected; normally permanent | Rescan; if confirmed, remove VM from datastore; recover datastore |
| **Storage latency** | High latency affects VM performance | Check multipathing, storage queue depth, HBA settings, storage load |
| **Path failure** | Individual path fails; multipathing should auto-recover | Check cables, switch port, HBA queue depth |
| **Snapshot growth** | Snapshot delta disk grows excessively | Consolidate snapshots; remove unnecessary snapshots |
| **VM stun** | VM stops processing due to storage unavailability (APD/PDL) | Resolve storage connectivity issue |

### Key Storage Commands
```powershell
# Storage diagnostics
Get-VMHostStorage -VMHost "ESXi01"
Get-StoragePath -VMHost "ESXi01"
Get-ScsiLun -VMHost "ESXi01" -Type "disk"

# Multipathing
Get-VMHostMultipathInfo -VMHost "ESXi01"

# Check VM state (stun)
Get-VM "MyVM" | Select-Object Name, PowerState, ExtensionData
```





---

## PART 11 — VMware HA / DRS / vMotion


###HA / DRS / vMotion

### HA (High Availability)
| Feature | Description |
|---------|-------------|
| **Purpose** | Restart VMs on different host if host fails |
| **Admission Control** | Prevents VM overload if host fails (reserve % capacity) |
| **Heartbeat** | VM heartbeat + host heartbeat + datastore heartbeat (VM tools) |
| **Isolation Response** | What VM does when host thinks it's isolated |
| **Host Failure** | VM restarts on remaining healthy hosts |

### HA Admission Control
| Policy | Description |
|--------|-------------|
| **Disabled** | No capacity check (risky) |
| **Cluster Resource Percentage** | Reserve % of cluster capacity (e.g., 25%) |
| **Host Failures Cluster Tolerates** | N host failures worth of capacity |
| **Specific Hosts and Clusters** | Define specific failover targets |

### HA Failure Scenarios
| Failure | Response |
|---------|----------|
| **Host failure** | VMs restart on remaining hosts (ordered by VM restart priority) |
| **Isolation** | VM may shutdown, power off, or do nothing (configured per VM) |
| **Datastore failure** | VM may restart on different host |
| **Network isolation** | Isolation response triggers |
| **PSync timeout** | vMotion or storage vMotion during HA may fail |

### DRS (Distributed Resource Scheduler)
| Feature | Description |
|---------|-------------|
| **Purpose** | Load balance VMs across hosts |
| **Modes** | Fully Automated, Partially Automated, Manual |
| **Rules** | VM/Host rules (affinity, anti-affinity) |
| **Initial placement** | Where to place new VMs |

### VM/Host Rules
| Type | Description |
|------|-------------|
| **Affinity** | VM must run on specific hosts (Group A members only) |
| **Anti-Affinity** | VM must NOT run with other VMs in group (spread across hosts) |
| **Should run** | Soft affinity (recommended) |
| **Must run** | Hard constraint |
| **Incompatible** | VM must NOT run on hosts in group |

### DRS Automation Levels
| Level | Behavior |
|-------|----------|
| **Manual** | DRS suggests migration; admin decides |
| **Partially Automated** | DRS auto-migrates during initial placement; manual for rebalancing |
| **Fully Automated** | DRS makes all decisions (rebalancing, migration) |

### vMotion
| Requirement | Description |
|-------------|-------------|
| **Shared storage** | All hosts must access same VM files (VMFS/NFS) |
| **vMotion network** | Dedicated VMkernel port |
| **CPU compatibility** | Enea or Intel; EVC mode must match (EVC = Intel architecture) |
| **Network** | VM network must exist on destination host |
| **Version** | vCenter supports live migration |

### Storage vMotion
| Requirement | Description |
|-------------|-------------|
| **Datastores** | Source and destination must be accessible by both hosts |
| **Storage type** | VMFS, NFS, vSAN |
| **Network** | VMkernel with file system NFS/iSCSI support |
| **Performance** | Sealed/locked VMs (or VMs with snapshots) may not support |





---

## PART 12 — VMware Snapshots

> *A very common interview topic — must understand the difference between snapshots and backups.*


###VM Snapshots Deep Dive

### Snapshot Purpose
- **Primary use**: Temporary protection during operations (patching, upgrades, testing, modifications)
- **Quick revert**: Go back to known-good state
- **NOT a backup**: Snapshot is NOT a backup — this is a critical distinction

### Snapshot Hierarchy (Delta Disk Chain)
```
Original disk (flat.vmdk) [SEALED - no more changes to this after snapshot]
↓ (delta disk references)
Delta 1: snapshot-1.vmdk [Current delta disk - where new writes go]
↓
Delta 2: snapshot-2.vmdk [If second snapshot taken]
↓
Delta 3: snapshot-3.vmdk [If third snapshot taken - very rare in production]
```

- **SEALED**: Old disk becomes sealed; no more writes; only read operations
- **Current delta disk**: All new writes go here
- **Each snapshot**: Has its own delta disk (delta01, delta02, etc.)
- **Delta disk growth**: Grows from 0 up to the size of the original disk

### Snapshot Growth
- Each delta disk grows dynamically as changes are made
- After multiple snapshots: Each snapshot can grow independently
- **Risk**: Delta disks can grow to 100% of original disk size
- **After consolidation**: All changes merged into original; delta disks deleted

### Snapshot vs Backup
| Feature | Snapshot | Backup |
|---------|----------|--------|
| **Data backup** | ❌ NO | ✅ YES |
| **Independent of source** | ❌ NO (depends on source disk) | ✅ YES |
| **Time-consuming** | ⚡ Instant | 🕐 Slow (data copy) |
| **Releases source disk space** | ❌ NO | ✅ YES |
| **Use for backup only** | ❌ NO | ✅ YES |
| **Use for quick restore** | ✅ YES | ⚠ YES (slow restore) |
| **Requires snapshot manager** | ❌ NO | ✅ YES |
| **Can handle deletion** | ✅ YES (if snapshot chain intact) | ✅ YES (independent backup) |
| **Long-term retention** | ❌ NO (grows indefinitely) | ✅ YES |
| **Snapshot chain break** | ❌ Chain broken = data loss | ✅ Independent |

**Clear operational answer to "Why shouldn't snapshots be treated as backups?"**:
- **Snapshots are dependent on the source disk** — if the source disk fails, ALL snapshots fail simultaneously
- **Snapshots are not a copy of data** — they only capture the *difference* (deltas) from the original; they do NOT backup data independently
- **Snapshots grow over time** — delta disks can consume all available disk space, causing VM failure
- **Snapshots affect performance** — long-running snapshots (more than 24-48 hours) degrade VM performance
- **Snapshot consolidation** is required to merge changes and reclaim space
- **Snapshot chains can break** — if a delta disk in the chain is corrupted or accidentally deleted, ALL snapshots become unrecoverable
- **Best practice**: Limit snapshot lifetime to 24 hours maximum (VMware recommendation), chain length ≤ 3

### Consolidation
- Merges delta disks into the original disk
- Frees snapshot delta disk space
- Required when snapshot chain grows long
- Can trigger manual consolidation via vSphere Client or PowerCLI

```powershell
# Snapshot operations (PowerCLI)
Get-VM "MyVM" | Get-Snapshot -Name "BeforePatch"
Get-VM "MyVM" | Get-Snapshot | Select-Object Name, Created, SizeMB

# Consolidate all snapshots on VM (remove them)
Get-VM "MyVM" | Get-Snapshot | Remove-Snapshot -RunAsync:$true

# Check if consolidation needed (PowerCLI)
$vm = Get-VM "MyVM"
$vm.ExtensionData.Config.Hardware.Device | Where-Object {$_.ExtensionData.Capability.UnmanagedSm}
# For snapshot consolidation status, use:
$vm | Select-Object Name, @{N="SnapshotCount";E={($_.ExtensionData.Config.Hardware.Device | Where-Object {$_.DeviceInfo.Label -eq "Hard disk"}).ExtensionData.CapacityInKB}}

# Consolidate via task
$vm | Get-Snapshot | Remove-Snapshot -Confirm:$false
```

### Troubleshooting
| Issue | Cause | Resolution |
|-------|-------|-----------|
| Failed consolidation | Storage issues, locked files, replication | Check storage, fix file locks, rerun consolidation |
| Snapshot growth | Long-running snapshots, heavy write workloads | Remove unnecessary snapshots, consolidate |
| Delta disk corruption | Storage failure, snapshot chain break | Restore from backup; snapshot chain may not be repairable |
| Power on fails | Unconsolidated snapshot delta disk can't open | Consolidate snapshots before powering on |
| VM stun | Snapshot delta disk on APD/PDL datastore | Fix storage, then consolidate |





---

## PART 13 — VMware Performance


###VMware Performance

### Key Metrics
| Metric | Description | Good State |
|--------|-------------|------------|
| **CPU Ready** | Time VM waited for CPU (sched wait) | < 5-10% (5% is OK, 10% is warning, >10% is critical) |
| **CPU Co-stop** | Time VM stopped due to co-scheduling (multiple vCPUs) | < 2-5% |
| **CPU Usage** | % of CPU resources consumed | < 80% sustained (or depends on guest) |
| **Memory Usage** | % of memory allocated | Should be high (active memory) |
| **Ballooning** | Hypervisor reclaims memory from VM via balloon driver | Some ballooning OK; high levels problematic |
| **Swapping** | Hypervisor swaps VM memory to disk | Avoid if possible; performance impact |
| **Compression** | ESXi compresses swapped pages to memory | Better than swapping; still uses CPU |
| **Disk Latency** | Time for I/O to complete | < 10ms (or < 20ms depending on storage type) |
| **IOPS** | Input/output operations per second | Depends on storage type and config |
| **Throughput** | Data transfer rate (MB/s) | Depends on workload |
| **Network Latency** | Time for network packet round-trip | < 1ms (within datacenter) |
| **Datastore Latency** | Storage latency from VM perspective | < 10ms |

### Memory Management (ESXi)
- **Ballooning**: Balloon driver in VM (VMware Tools) reclaims guest memory → returns to ESXi
- **Swapping**: ESXi swaps VM pages to swap file (vmswap)
- **Compression**: Compressed pages stored in compressed memory area (ram-z)
- **Memory sharing** (TPS - Transparent Page Sharing): Share identical memory pages between VMs (deprecated due to security concerns — Meltdown/Spectre)
- **Idle memory tax**: Unused memory is reclaimed
- **Active memory tax**: Recently used memory kept

### Performance Monitoring (PerfMon/ESXTOP)
- **esxtop**: Command-line performance monitor on ESXi
  - `c` = CPU, `m` = memory, `d` = disk, `n` = network
  - Key indicators: %RDY (CPU Ready), %MLMTD (memory limit), DAVG/cmd (disk latency per command)
- **vRealize Operations**: Advanced analytics
- **vSphere Client**: Performance charts (real-time / 24hr / 7-day)

```powershell
# Performance monitoring via PowerCLI
Get-Stat -Entity "MyVM" -Stat cpu.ready.sum -Realtime -MaxSamples 100
Get-Stat -Entity "MyVM" -Stat mem.usage.average -Realtime
Get-Stat -Entity "MyVM" -Stat disk.throughput.peak.average -Realtime
Get-Stat -Entity "MyVM" -Stat net.throughput.peak.average -Realtime

# Performance chart data
Get-VICompositePerfKey -Stat "cpu.ready.maximum" -Interval (Get-Statistic -Entity "MyVM" | Select-Object -First 1 -ExpandProperty Interval)

# Common perf stats
$stats = @("cpu.usage.average", "cpu.ready.average", "cpu.costop.average", "mem.usage.average", "disk.throughput.peak.average", "disk.latency.peak.average", "net.throughput.peak.average", "net.latency.peak.average")
```

### Performance Troubleshooting Flow
```
Application Issues
      ↓
Guest OS (Windows/Linux) — Check CPU, memory, disk inside VM
      ↓
VM Level — Check VM hardware config (vCPU, RAM, disk type), hypervisor tools
      ↓
ESXi Host — Check CPU ready, memory balloons, swaps, host disk latency
      ↓
Cluster — Check HA/DRS, resource pools, EVC mode, DPM
      ↓
Storage — Check datastore latency, IOPS, multipathing, APD/PDL
      ↓
Network — Check vSwitch, NIC, VMkernel, physical network
```

### CPU Ready Troubleshooting
- **Cause**: Too many vCPUs, CPU contention, host overloaded, co-stopping
- **Fix**: Reduce vCPUs, migrate VM (vMotion/DRS), add host resources
- **Note**: vCPU > physical core count leads to significant co-stopping

### Memory Troubleshooting
- **Ballooning high**: Guest under memory pressure; balloon driver reclaiming
- **Swapping**: VM swapping to disk; ESXI memory reclaim technique
- **Memory leak in guest**: Check with Process Explorer / Task Manager inside VM
- **VM memory too high**: Right-size; VM memory allocated = virtual hardware RAM

### Disk Troubleshooting
- **High latency**: Storage overloaded, queue depth, bad disk, HBA issue
- **IOPS limit**: Storage controller limits, disk speed limits, array cache
- **Throughput**: Bandwidth limited by NIC/storage/HBA
- **All-Flash vs Hybrid**: Different latency/throughput characteristics





---

## PART 14 — Hyper-V

> *JD explicitly requires Hyper-V administration and clustering.*


###Hyper-V Fundamentals

### Core Concepts
| Term | Description |
|------|-------------|
| **Hypervisor** | Type 1 hypervisor (bare-metal) |
| **Host** | Physical server running Hyper-V |
| **VM** | Virtual machine |
| **Generation 1** | BIOS-based VM; legacy hardware ( emulate hardware) |
| **Generation 2** | UEFI-based VM; modern; supports secure boot, boot from VHD/X, TPM |
| **VHDX** | Virtual hard disk format (replaces VHD; 64TB max vs 2TB) |

### Generation 1 vs Generation 2
| Feature | Gen 1 | Gen 2 |
|---------|-------|-------|
| **Firmware** | BIOS | UEFI |
| **Secure Boot** | No | Yes |
| **Boot from VHD/X** | No | Yes |
| **TPM** | No | Yes |
| **SCSI controller** | No | Yes |
| **Network adapter emulation** | Emulated (MAX) | Synthetic (formerly legacy) |
| **Maximum VHDX size** | 2TB | 64TB |
| **Production checkpoints** | No | Yes |

### Disk Types
| Type | Description | Use Case |
|------|-------------|----------|
| **VHDX** | Virtual Hard Disk (recommended) | All modern VMs |
| **VHD** | Legacy Virtual Hard Disk | Gen 1 VMs; backward compat |
| **Dynamic disk** | Expands as needed (thin provisioning) | Dev/test; saves initial space |
| **Fixed disk** | Pre-allocated; maximum size from creation | Production (best performance) |
| **Differencing disk** | Changes relative to parent disk | Test environments; lab |

### Checkpoints
- **Snapshot** of VM state, memory, and virtual hardware
- **Standard checkpoints**: VM state saved (VM aware)
- **Production checkpoints**: VM-aware; uses VSS/VMWS (VMware Worker Service); application-consistent

### Hyper-V Virtual Switches
| Type | Description | Network Access |
|------|-------------|---------------|
| **External** | VM gets direct access to physical network | Yes (via virtual NIC to physical NIC) |
| **Internal** | VM can communicate with host and other VMs | VM ↔ Host only (no external) |
| **Private** | VM can communicate only with other VMs on same host | VM ↔ VM only (no host) |

### vNIC Types
| Type | Description |
|------|-------------|
| **Synthetic** | High-performance; requires integration services (default) |
| **Emulated** | Legacy; lower performance; no integration services needed |
| **MAC Address** | Auto-generated or static |
| **VLAN** | Virtual LAN tagging |
| **Bandwidth management** | Limit VM bandwidth |
| **RDMA** | Remote Direct Memory Access (higher performance) |
| **Accelerated Networking** | SR-IOV based (offload) |
| **Firewall** | Built-in virtual firewall |

### Storage
| Type | Description |
|------|-------------|
| **VHDX** | Virtual hard disk file (.vhdx) |
| **CSV (Cluster Shared Volume)** | Shared storage for Hyper-V cluster (reFS or NTFS) |
| **SMB Storage** | Hyper-V over SMB (SMB 3.0+) |
| **iSCSI** | Block storage via IP |
| **Storage Spaces** | Software-defined storage; uses disks to create storage pool |

### PowerShell Management
```powershell
# VM operations
New-VM -Name "MyVM" -Generation 2 -MemoryStartupBytes 4GB -NewVHDPath "C:\VMs\MyVM.vhdx" -NewVHDSizeBytes 100GB -Path "C:\VMs" -SwitchName "ExternalSwitch"
Get-VM | Where-Object {$_.State -eq 'Running'}
Stop-VM -Name "MyVM" -Force
Restart-VM -Name "MyVM"
Save-VM -Name "MyVM"              (save state)
Resume-VM -Name "MyVM"
Suspend-VM -Name "MyVM"

# Checkpoint operations
Checkpoint-VM -Name "MyVM" -SnapshotName "BeforeUpdate"
Get-VMSnapshot -VMName "MyVM"
Remove-VMSnapshot -VMName "MyVM" -Name "BeforeUpdate" -Confirm:$false

# Integration Services
Get-VMIntegrationService -VMName "MyVM"
Restart-VM -Name "MyVM" -Force

# VM migration
Move-VM -Name "MyVM" -DestinationHost "HyperV02" -IncludeStorage -StorageDestination "C:\VMs"
```

### Hyper-V Operations
- **Export/Import**: `Export-VM`, `Import-VM` (can import to different host)
- **Live migration**: `Move-VM` between Hyper-V hosts
- **Storage migration**: Move VM storage to different location
- **Checkpoints**: `Checkpoint-VM`, `Get-VMSnapshot`
- **Integration Services**: VSS, Time Synchronization, Heartbeat, Guest Services
- **Resource allocation**: Memory, CPU, GPU (discrete device assignment)
- **Virtual TPM**: Available on Gen 2 VMs
- **Secure Boot**: UEFI Secure Boot with Microsoft/Windows certificates
- **Shielded VMs**: Encrypted VM at rest and in transit (BitLocker + TPM + Host Guardian Service)
- **Host Guardian Service**: For shielded VM attestation





---

## PART 15 — Hyper-V Failover Clustering


###Hyper-V Failover Clustering

### Key Concepts
| Term | Description |
|------|-------------|
| **Cluster** | Group of nodes (Hyper-V hosts) |
| **Node** | Server in the cluster |
| **Quorum** | Voting mechanism to determine cluster health |
| **Witness** | Additional vote (file share/cloud) for tie-breaking |
| **CSV (Cluster Shared Volume)** | ReFS/NIFS volume accessible to all nodes |
| **Cluster roles** | Virtual Machine, Hyper-V Replica, etc. |

### Cluster Validation
- Run before creating cluster or after adding nodes
- Validates hardware, networking, storage, configuration

```powershell
# Test cluster
Test-Cluster -Node "HyperV01", "HyperV02"

# Create cluster
New-Cluster -Name "HyperVCluster" -Node "HyperV01", "HyperV02" -StaticAddress 192.168.10.100

# Add node
Add-ClusterNode -Name "HyperV03" -Cluster "HyperVCluster"

# Validate
Test-Cluster -Node "HyperV01", "HyperV02" -Include "Storage Spaces", "Network"
```

### Quorum Configurations
| Type | Description |
|------|-------------|
| **Node Majority** | Odd number of nodes; requires >50% nodes online |
| **Node + File Share Witness** | Even nodes + file share vote for tie-breaker |
| **Node + Cloud Witness** | Azure cloud witness (modern recommendation) |
| **Dynamic Witness** | Automatically adjusts quorum based on cluster size |

### Cluster Shared Volume (CSV)
- ReFS or NTFS formatted volume presented to all nodes
- CSV uses a **block-level** redirection via SMB if a node doesn't own the CSV
- CSV owns the CSV; nodes access CSV through CSV broker
- CSV resync: Syncs CSV metadata after failure

### Cluster Roles
- **Virtual Machine**: Hyper-V VM role; can live migrate between nodes
- **Hyper-V Replica**: Replication roles for DR
- **File Server**: File server role
- **SQL Server**: SQL Server role

### Live Migration
```powershell
# Live migration
Move-VM -Name "MyVM" -DestinationHost "HyperV02" -IncludeStorage -StorageDestination "C:\VMs"

# Check live migration
Get-VM -ComputerName "HyperV02" | Where-Object {$_.State -eq 'Running'}
Get-VM -ComputerName "HyperV01" | Where-Object {$_.State -eq 'Running'}
```

### Cluster Network
- Cluster networks must be dedicated (management + cluster)
- CSV traffic on dedicated network
- Live migration traffic on dedicated network
- Witness traffic on dedicated network (if possible)




###Hyper-V Cluster Troubleshooting

| Issue | Cause | Resolution |
|-------|-------|-----------|
| **Node eviction** | Node loses quorum; cluster determines node unreachable | Check cluster logs, node status, network; if intentional, use `Remove-ClusterNode` |
| **Quorum loss** | Quorum voting fails; cluster goes offline | Add/repair witness, check node status, check network |
| **CSV inaccessible** | CSV ownership issue; node didn't own CSV | Live migrate CSV owner, check CSV state, force CSV reset |
| **Cluster network failure** | Network connectivity between nodes lost | Check NIC teaming, switch, cabling |
| **Storage failure** | Shared storage unavailable | Check storage health, multipathing, backup |
| **VM won't start** | Multiple possible causes (storage, quorum, resource exhaustion) | Check VM configuration, host resources, storage, checkpoints |
| **Live migration failure** | Network, storage, host compatibility | Check VM version, migration network, storage compatibility, host version |

### Troubleshooting Commands
```powershell
# Cluster
Get-ClusterNode | Select-Object Name, State, Status
Get-ClusterGroup | Select-Object Name, State, OwnerNode
Get-ClusterResource | Where-Object {$_.State -ne 'Online'}
Get-ClusterNetwork | Select-Object Name, State, Sense
Get-ClusterSharedVolume | Select-Object Name, State, OwnedBy

# Cluster logs
Get-ClusterLog -Node "HyperV01" -StartTime (Get-Date).AddHours(-2)
# Or inspect: C:\Windows\Cluster\Reports\*.html

# Failover
Stop-ClusterGroup -Name "Virtual Machine" -IncludeSharedVolumes -Confirm:$false
```





---

## PART 16 — Proxmox (Exposure / Nice-to-Have)

> *Lower priority — listed as exposure in JD. Focus on fundamentals.*


###Proxmox VE

### Core Concepts
| Term | Description |
|------|-------------|
| **Proxmox VE** | Open-source virtualization platform (KVM + LXC) |
| **KVM** | Kernel-based Virtual Machine (Linux hypervisor) |
| **LXC** | Linux Containers (lightweight OS-level virtualization) |
| **Cluster** | Multiple Proxmox nodes managed together |
| **Nodes** | Individual Proxmox servers |
| **Storage** | Local or shared storage (Ceph, NFS, ZFS) |
| **Bridges** | Virtual switches (vmbr0, vmbr1, etc.) |
| **VLAN** | 802.1Q tagging on bridges |
| **HA** | High Availability for VMs (fencing via Hardware Watchdog / HA manager) |
| **Ceph** | Distributed storage (block, object, file) |

### Key Differences from VMware
| Feature | VMware | Proxmox |
|---------|--------|---------|
| **Hypervisor** | ESXi (proprietary) | KVM (Linux-based, open source) |
| **Management** | vCenter (Windows-based) | Web UI (Linux-based) |
| **Storage** | VMFS, vSAN | LVM, ZFS, Ceph, NFS |
| **HA** | HA cluster (VM restart) | HA with fencing |
| **Cost** | Commercial / licensing | Free / Open Source |

### Proxmox CLI
```bash
# VM operations
qm create 100 --name "MyVM" --memory 4096 --cores 2 --os l26
qm start 100
qm stop 100
qm restart 100
qm config 100

# List VMs
qm list
qm status 100

# LXC containers
pct create 101 --hostname "MyContainer" --memory 1024 --cores 1 --rootfs local-lx:8
pct start 101
pct stop 101
pct list

# Snapshot
qm snapshot 100 "snapshot1"
qm delsnapshot 100 "snapshot1"
```

### Ceph Concepts (Basic)
- **Monitor (MON)**: Manages cluster map
- **OSD (Object Storage Daemon)**: Stores data
- **PG (Placement Group)**: Groups of OSDs
- **CRUSH map**: Maps data to OSDs
- **Pools**: Logical containers for data
- **RBD**: Block device (VM storage)
- **CephFS**: File system
- **RGW**: Object gateway

### Proxmox Cluster
- Corosync based (uses `user.cfg` for auth)
- HA requires shared storage (Ceph/NFS/iSCSI)
- Quorum: 2 nodes minimum; 3 recommended for split-brain prevention
- HA group: Set of VMs/LXC that can migrate between nodes
- Fencing: STONITH (Shoot The Other Node In The Head) — prevents split-brain
- Watchdog: Required for HA fencing (IPMI, iLO, or software watchdog)

### Proxmox HA
- Requires Corosync cluster
- Requires shared storage
- VMs configured in HA group
- On node failure, VM automatically restarted on remaining node
- Requires fencing to prevent split-brain

### Proxmox Storage
- **Local**: Direct-attached disk
- **ZFS**: Software-defined storage; snapshots built-in
- **Ceph**: Distributed storage (recommended for clusters)
- **NFS**: Network file system
- **iSCSI**: Block storage
- **LVM**: Local logical volume management

### Proxmox Networking
- **Linux bridges**: `vmbr0`, `vmbr1` (created in GUI or `/etc/network/interfaces`)
- **VLAN**: Trunk VLANs on bridges; VM NICs with VLAN tag
- **Open vSwitch**: Advanced SDN option
- **Bonding**: Link aggregation (mode 4 LACP)

### Proxmox Backup
- **Proxmox Backup Server (PBS)**: Dedicated backup solution (deduplicated, compressed)
- **vzdump**: Built-in backup tool (VZ Dump) — creates archive
- **Backup retention**: Policy-based
- **Incremental**: Cumulative incremental backups (PBS is more efficient)

### Proxmox HA Configuration
- Configure shared storage (Ceph/NFS)
- Enable HA in cluster GUI
- Create HA group, assign VMs
- Configure fencing resources (IPMI, iLO, iDRAC)
- Watchdog device (mandatory for HA)
- Check HA status: `ha-manager status`





---

## PART 17 — Monitoring

> *JD specifically names: SCOM, Centreon, SolarWinds, Grafana.*


###Monitoring Fundamentals

### Monitoring Concepts
| Category | What to Monitor |
|----------|----------------|
| **Availability** | Is it up? Uptime, response time |
| **CPU** | Usage, ready, steal, queue |
| **Memory** | Usage, free, page faults, swap |
| **Disk** | Latency, IOPS, throughput, free space |
| **Network** | Bandwidth, errors, latency, packet loss |
| **Service** | Windows service, process, application availability |
| **Event** | Critical/error events in Event Viewer |

### Thresholds & Alerts
| Concept | Description |
|---------|-------------|
| **Threshold** | Value that triggers an alert (CPU > 90%, disk < 10% free) |
| **Alert** | Notification when threshold breached |
| **Monitoring** | Continuous measurement against thresholds |
| **Alert suppression** | Prevent duplicate alerts |
| **Escalation** | Escalate to higher level if not acknowledged |
| **Dependency** | If A depends on B, alert on B might cascade to A |
| **Synthetic monitoring** | Proactive checks from external location |
| **Capacity monitoring** | Trend analysis for future planning |

### Products (High-Level)
| Product | Type | Key Feature |
|---------|------|------------|
| **SCOM** (System Center Operations Manager) | Microsoft agent-based | Deep Windows/AD/Exchange/SQL monitoring; management packs |
| **Centreon** | Open-source | Host/service checks via plugins; SNMP, NRPE, WMI |
| **SolarWinds** | Comprehensive | Network, server, application, storage; Orion platform |
| **Grafana** | Dashboard/visualization | Time-series dashboards; feeds from Prometheus, InfluxDB, etc. |

### SCOM (Key Concepts)
- **Management Pack**: Pre-built monitoring for Microsoft products (AD, Exchange, SQL, IIS)
- **Agent**: Installed on monitored server
- **Gateway**: Agent relay (for non-domain joined servers)
- **Root Management Server (RMS)**: Central management server (historical; now all management servers)
- **Management Server**: Processing, collection, database
- **Operations Console**: Web console for monitoring
- **Authoring Console**: Custom rules/monitors
- **Dashboards**: Custom views with state, performance, alerts
- **Performance rules**: Collect performance data
- **Threshold-based alerting**: Alert when value exceeds threshold
- **Monitoring**: Watch state (healthy/unhealthy/unknown)

### Centreon (Key Concepts)
- Plugin-based architecture (check commands)
- NRPE (NRPE server), NSClient++, WMI, SNMP for data collection
- Host check: ping/HTTP/HTTPS check
- Service check: specific services (CPU, disk, memory, etc.)
- **Notification**: Email, SMS, Slack, etc.
- **Escalation**: Time-based escalation
- **Dependency**: Host/service dependencies
- **Host groups**: Organize by function/location
- **Time periods**: Define availability hours
- Poller (central or distributed)

### SolarWinds (Key Concepts)
- **Orion Platform**: Core monitoring platform
- **Server & Application Monitor (SAM)**: Application monitoring
- **Network Performance Monitor (NPM)**: Network device monitoring
- **Storage Resource Monitor (SRM)**: Storage monitoring
- **Virtualization Manager**: VM monitoring
- **IP Address Manager (IPAM)**: IP address tracking
- **Syslog**: Collect syslog messages
- **WMI/SSH/SNMP** for data collection
- **Polling interval**: Default 1-5 minutes
- **Alert actions**: Email, SNMP trap, run script, service restart

### Grafana (Key Concepts)
- **Dashboard**: Visual panel with graphs, gauges, alerts
- **Data source**: Prometheus, InfluxDB, Elasticsearch, SQL, etc.
- **Panel types**: Graph, Stat, Gauge, Table, Alert list
- **Variables**: Template variables (dropdowns for server selection)
- **Alert**: Grafana built-in alerting (threshold-based)
- **Annotations**: Mark events on graph
- **Time range**: Relative (last 15 minutes) or absolute (date range)
- **Refresh**: Auto-refresh interval




###Alert Troubleshooting

### Troubleshooting Alerts
| Step | Action |
|------|--------|
| **1. Validate alert** | Is the alert real? Check current status |
| **2. False positive** | Check if alert is legitimate or misconfigured threshold |
| **3. Dependency issue** | Does another service down cause this alert? |
| **4. Suppression** | Is alert suppressed? Check suppression rules |
| **5. Escalation** | Has alert been escalated? Check escalation policy |
| **6. Root cause** | What caused the trigger? |

### Common Alert Scenarios
| Alert | Root Cause |
|-------|-----------|
| **High CPU** | Process consuming resources; DDoS; malware; undersized VM |
| **Memory low** | Memory leak; overcommit; swap thrashing |
| **Disk full** | Logs; temp files; database growth; snapshots |
| **Disk high latency** | Storage overloaded; HBA issue; multipathing |
| **Service stopped** | Crashed; dependency failed; scheduled restart; bug |
| **Event ID XYZ** | Application-specific; check vendor documentation |
| **Ping failure** | Network issue; ICMP blocked; host down; interface down |
| **DNS resolution failure** | DNS server down; DNS record missing; network issue |





---

## PART 18 — Windows Performance Troubleshooting

> *Should be one of your strongest interview sections.*


###CPU Performance

### Key Concepts
| Term | Description |
|------|-------------|
| **High CPU** | CPU at 100% or near; process identification needed |
| **Process identification** | Which process is consuming CPU |
| **CPU queue** | Number of threads waiting for CPU (run queue) |
| **Context switching** | Switching between processes/threads (overhead) |
| **% Ready** | Time VM waited for CPU (VMware) |

### Diagnosis
```powershell
# CPU information
Get-ComputerInfo | Select-Object CsProcessors, OsName, OsVersion

# Process CPU usage
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10 Name, CPU, WorkingSet, Id

# Performance Monitor
perfmon /res                   (Resource Monitor)
perfmon /sys                    (System Monitor)

# Performance Monitor counter
% Processor Time      (overall CPU usage)
% User Time           (user-mode CPU)
% Privileged Time     (kernel-mode CPU)
% Interrupt Time      (hardware interrupts)
% DPC Time            (deferred procedure calls)
% Idle Time           (CPU idle)

# Typical thresholds
% Processor Time: >80% sustained = high
% Idle Time: <20% = high usage
```

### Troubleshooting Flow
1. Identify process (Task Manager / `Get-Process`)
2. Check if process is expected
3. Check process details (file path, digital signature, command line)
4. Check if process has known issue (update/patch)
5. Kill process if appropriate (with approval/documentation)
6. If system-wide: check for malware, DDoS, resource exhaustion
7. High interrupt time: NIC/Driver issue
8. High DPC time: Driver issue
9. High DPC/ISR time: Check `DPC Latency Checker`




###Memory Performance

### Key Concepts
| Term | Description |
|------|-------------|
| **Available memory** | RAM ready to use (free + standby + free) |
| **Paging** | Moving memory pages to page file (disk) |
| **Working set** | Memory currently in RAM for a process |
| **Commit** | Virtual memory (RAM + page file) committed |
| **Page file** | Swap file on disk |
| **Ballooning** | Hypervisor reclaims memory (VMware) |
| **Swapping** | Memory paged to disk |
| **Hard faults/sec** | Page faults where data NOT in RAM (must read from disk) — critical metric |

### Windows Memory Counters
| Counter | Warning | Critical |
|---------|---------|----------|
| **Available MBytes** | < 100-200 MB | < 50 MB |
| **Pages/sec** | > 20 | > 50 |
| **Pages Input/sec** | > 5 | > 10 |
| **% Committed Bytes in Use** | > 80% | > 90% |
| **Pool Paged Bytes** | Growing steadily | Any growth = issue |

### Page File Management
```powershell
# Check page file config
Get-CimInstance -ClassName Win32_ComputerSystem | Select-Object AutomaticManagedPagefile
Get-CimInstance -ClassName Win32_PageFileSetting | Select-Object Name, InitialSize, MaximumSize

# Set page file
$cs = Get-CimInstance -ClassName Win32_ComputerSystem
$pageFile = Get-CimInstance -ClassName Win32_PageFileSetting -Filter "Name like '%pagefile.sys'"
Set-CimInstance -InputObject $pageFile -Property @{InitialSize=4096; MaximumSize=8192}

# Or via systempropertiesperformance
systempropertiesperformance  # Opens Performance Options dialog
```

### Troubleshooting Flow
1. Check available memory (Task Manager / `Get-Counter`)
2. Identify memory-hungry process (Task Manager / `Get-Process`)
3. Check working set size (Process properties)
4. Check commit size (Virtual Memory in Task Manager)
5. Monitor hard faults/sec (Resource Monitor → Memory)
6. Check page file size and configuration
7. Check if memory leak (process working set grows over time)
8. Consider adding RAM or right-sizing VMs
9. Memory compression: Enable (Windows 10/11 Server 2016+)




###Disk Performance

### Key Concepts
| Term | Description |
|------|-------------|
| **Disk queue** | Number of I/O requests waiting |
| **IOPS** | Input/Output Operations Per Second |
| **Latency** | Time for single I/O operation (ms) |
| **Throughput** | Data transfer rate (MB/s) |
| **Free space** | Available disk space |
| **Avg. Disk sec/Read** | Average latency for read (ms) |
| **Avg. Disk sec/Write** | Average latency for write (ms) |

### Performance Counters
| Counter | Good | Warning | Critical |
|---------|------|---------|----------|
| **Avg. Disk sec/Read** | < 5ms | 5-10ms | > 10ms |
| **Avg. Disk sec/Write** | < 5ms | 5-10ms | > 10ms |
| **Avg. Disk Queue Length** | < 2 per disk | 2-3 | > 3 |
| **Disk Bytes/sec** | Depends on disk type | Increasing | Saturated |
| **% Disk Time** | < 50% | 50-80% | > 80% |
| **Current Disk Queue** | < 2 | 2-3 | > 3 |

### Disk Troubleshooting
```powershell
# Disk information
Get-PhysicalDisk | Select-Object FriendlyName, MediaType, Size, HealthStatus, OperationalStatus
Get-Disk | Select-Object Number, FriendlyName, Size, Healthy, OperationalStatus
Get-Partition | Where-Object {$_.DriveLetter -eq "C"} | Select-Object DriveLetter, Size, Type

# Disk usage
Get-Counter "\PhysicalDisk(0)\Avg. Disk sec/Read"
Get-Counter "\PhysicalDisk(0)\Avg. Disk sec/Write"

# Resource Monitor
resmon