 I'll proceed by building the most comprehensive, enterprise-grade Technology Dependency Map possible for a **Lead Windows & Virtualization Administrator** role, covering all four listed topics (Windows Server, Active Directory, Group Policy, DNS) and extending into the full scope of technologies that such a role inherently encompasses (Hyper-V, Clustering, PowerShell, etc.).

If you share the actual JD text, I can refine this to match it exactly. Let's dive in.

---

# 🏗️ PART 1 — COMPLETE TECHNOLOGY DEPENDENCY MAP

## The Enterprise Dependency Hierarchy (Top-Down)

```
┌─────────────────────────────────────────────────────────────────┐
│                    PHYSICAL / CLOUD LAYER                        │
│  Power → Networking → Storage → Hypervisor / Cloud             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   HYPER-V / VIRTUALIZATION LAYER                 │
│  Hyper-V Server → Virtual Machines → VM Configuration / Replication│
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    OPERATING SYSTEM LAYER                        │
│  Windows Server (OS, Drivers, Services, File System, Registry) │
└────────────────────────┬────────────────────────────────────────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
┌──────────────┐ ┌───────────┐ ┌──────────┐
│ ACTIVE       │ │ DNS       │ │ GROUP    │
│ DIRECTORY    │ │ (AD DS    │ │ POLICY   │
│ (AD DS,      │ │  Servers, │ │ (GPO)    │
│  ADCS, ADCS, │ │  DNS/Zones│ │          │
│  ADFS, etc.) │ │  Clients) │ │          │
└──────┬───────┘ └─────┬─────┘ └────┬─────┘
       │               │             │
       └───────┬───────┘─────────────┘
               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SERVICES & AUTOMATION LAYER                   │
│  IIS, DHCP, FSMO Roles, PowerShell, SCCM/Intune, WSUS, DFS    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    APPLICATIONS & USERS                          │
│  Line-of-Business Apps, User Profiles, Sessions, GPO Effects   │
└─────────────────────────────────────────────────────────────────┘
```

---

# 🔷 SECTION 1 — WINDOWS SERVER

## 1.1 What Is Windows Server?

Windows Server is Microsoft's enterprise-class operating system designed to provide **high availability, security, manageability, and scalability** for running services that serve hundreds to thousands of users and devices. Unlike Windows 10/11 (client OS), it is built around:

- **Server roles** (what the server *does*)
- **Features** (capabilities installed on top of roles)
- **Services** (background processes)
- **The Registry** (centralized configuration store)
- **NTFS/ReFS** (file systems with advanced permissions)

## 1.2 Why Is It Used?

| Reason | Explanation |
|--------|-------------|
| **Service Hosting** | Runs AD DS, DNS, DHCP, IIS, File Server, Print Server, etc. |
| **Centralization** | Single pane of management for users, computers, resources |
| **Security** | Group Policy, BitLocker, Windows Defender, AppLocker, Credential Guard |
| **Scalability** | Supports up to 24 TB RAM (Server 2022), 64+ processors |
| **High Availability** | Failover Clustering, Live Migration, Storage Spaces Direct |
| **Automation** | PowerShell remoting, Desired State Configuration (DSC), WinRM |
| **Hybrid** | Azure Arc, Azure AD Join, Hyper-V Replica to cloud |

## 1.3 How Does It Work? (Architecture)

```
┌─────────────────────────────────────────────┐
│              USER MODE                        │
│  ┌─────────┐  ┌─────────┐  ┌─────────────┐ │
│  │ Services │  │ Apps    │  │ PowerShell  │ │
│  │ (svchost)│  │ (GUI/CLI)│ │ (WinRM/SSH)│ │
│  └────┬─────┘  └────┬─────┘  └──────┬─────┘ │
├───────┼──────────────┼───────────────┼───────┤
│       ▼     NT Kernel & Executive       ▼     │
│  ┌──────────────────────────────────────────┐ │
│  │  HAL  │  Process Manager │ Memory Manager│ │
│  │  File System │  Security Reference │ I/O │ │
│  └──────────────────────────────────────────┘ │
├───────────────────────────────────────────────┤
│              KERNEL MODE                      │
│  Hardware Abstraction Layer (HAL)             │
│  Registry (SYSTEM, SOFTWARE, SAM, SECURITY)   │
└───────────────────────────────────────────────┘
```

### Key Architectural Components:

1. **NT Kernel** — Core of the OS, handles process/thread scheduling, memory management, I/O
2. **HAL (Hardware Abstraction Layer)** — Abstracts hardware differences; allows multiple HALs (ACPI, etc.)
3. **Registry** — Hierarchical database (`HKLM`, `HKCU`, `HKCR`, `HKU`, `HKPD`) storing OS/config data
4. **Services Control Manager (SCM)** — Manages service lifecycle (start, stop, pause)
5. **svchost.exe** — Generic host process for DLL-based services (each svchost group is isolated via service hardening)
6. **Session 0 Isolation** — Services run in Session 0 (no UI); interactive users in Session 1+ (prevents Shatter attacks)
7. **Windows Subsystem** — Win32 subsystem (legacy), WSL2 (Linux compatibility layer)
8. **Driver Model** — WDM/WDF drivers, signed driver enforcement (CI/CI policies)

## 1.4 Components Involved

### Core Components:

1. **Kernel (`ntoskrnl.exe`)** — Core operating system kernel
2. **HAL (`hal.dll`)** — Hardware abstraction layer
3. **Registry** — `C:\Windows\System32\config\` (SYSTEM, SOFTWARE, SAM, SECURITY, DEFAULT)
4. **File Systems** — NTFS (primary), ReFS (Resilient File System for Storage Spaces/S4H)
5. **Windows Defender / Microsoft Defender Antivirus** — Integrated AV/EDR
6. **Windows Update (WSUS/SCUP/SUS)** — Patch management infrastructure
7. **IIS (Internet Information Services)** — Web server role
8. **WMI (Windows Management Instrumentation)** — Infrastructure for management data
9. **PowerShell** — Task automation and configuration management framework
10. **Windows Event Log** — System, Application, Security, Setup logs
11. **Component-Based Servicing (CBS)** — Manages Windows features and updates (`C:\Windows\Logs\CBS\`)
12. **Driver Store** — `C:\Windows\System32\DriverStore\` — Driver repository
13. **Certificate Store** — PKI integration, TLS/SSL certificates
14. **Task Scheduler** — Scheduled tasks and jobs
15. **Windows Error Reporting (WER)** — Crash dump collection

## 1.5 What Depends on Windows Server?

```
Windows Server (OS Layer)
│
├── Active Directory Domain Services (AD DS)
├── DNS Server Role
├── DHCP Server Role
├── Active Directory Certificate Services (AD CS)
├── Active Directory Federation Services (AD FS)
├── Active Directory Lightweight Directory Services (AD LDS)
├── IIS (Web Applications, APIs, intranet portals)
├── File Server (SMB/NFS) → DFS Namespaces, DFS Replication
├── Print Server
├── Windows Failover Clustering
├── Hyper-V (if standalone or host)
├── Windows Deployment Services (WDS) / MDT / SCCM
├── Remote Desktop Services (RDS / RDSH)
├── WSUS / SCCM / Intune (patch management)
├── PowerShell Remoting (WinRM/SSH)
├── VPN / RRAS
├── SharePoint / SQL Server (applications on the server)
└── All LOB applications running on the server
```

## 1.6 What Happens If Windows Server Fails?

| Failure Type | Impact |
|---|---|
| **Boot Failure** | No services run; users can't log in; dependent services offline |
| **BSOD (Blue Screen)** | Server crash; all workloads on that host go down; VMs may auto-restart or be lost |
| **Service Crash (e.g., svchost group)** | Specific functionality lost (e.g., DNS stops answering, AD can't replicate) |
| **Disk Failure (OS drive)** | Server unbootable; require restore from backup/RAID |
| **Memory Leak / Exhaustion** | Server slowdown; service instability; OOM kills |
| **Corrupt Registry** | Services may fail to start; OS may not boot; restore from \Registry\Backups |
| **Time Drift** | Kerberos authentication breaks (5-min skew tolerance); AD replication failures |
| **Network Driver Failure** | Server unreachable; VMs lose connectivity if on physical host |

## 1.7 How to Identify Failure

### Diagnostic Tools & Commands:

```powershell
# 1. System health overview
systeminfo
winmsd.exe
msinfo32.exe

# 2. Event log queries
Get-EventLog -LogName System -Newest 50
Get-WinEvent -LogName System,Application,Security -MaxEvents 100
Get-WinEvent -FilterHashtable @{LogName='System'; Level=1,2} -MaxEvents 50

# 3. Service status
Get-Service | Where-Object {$_.Status -ne 'Running'}
sc query state= all

# 4. Resource usage
Get-Process | Sort-Object WorkingSet64 -Descending | Select-Object -First 20 Name, Id, WorkingSet64
Get-Counter '\Processor(_Total)\% Processor Time'
Get-Counter '\Memory\Available MBytes'
Get-Counter '\PhysicalDisk(_Total)\% Disk Time'

# 5. Network
Test-Connection -ComputerName DC01 -Count 4
Get-NetIPConfiguration
Resolve-DnsName DC01
netstat -anob

# 6. Time check (critical for AD)
w32tm /query /status
w32tm /query /peers

# 7. BSOD analysis
Get-WinEvent -Path C:\Windows\Minidump\*.dmp  # or use WinDbg
!analyze -v  (in WinDbg)

# 8. Driver verification
driverquery /v
Get-WindowsDriver -Online

# 9. Certificate health
Get-ChildItem Cert:\LocalMachine\My
certutil -verifyCTL

# 10. Component-based servicing (update health)
Get-WindowsPackage -Online | Where-Object {$_.InstallState -eq 'Pending'}
dism /online /get-packages
```

### Key Logs to Check:

| Log | Path / Source | What It Tells You |
|-----|--------------|-------------------|
| **System** | Event Viewer → Windows Logs → System | Service failures, driver issues, boot errors, WMI events |
| **Application** | Event Viewer → Windows Logs → Application | App crashes, IIS errors, software-specific events |
| **Security** | Event Viewer → Windows Logs → Security | Logon attempts, privilege escalation, object access, audit failures |
| **Setup** | Event Viewer → Windows Logs → Setup | Windows Update / Feature Update status |
| **CBS** | `C:\Windows\Logs\CBS\CBS.log` | Component/Feature/Update installation details |
| **DISM** | `C:\Windows\Logs\DISM\dism.log` | Deployment Image Servicing and Management operations |
| **Windows Update** | `C:\Windows\WindowsUpdate.log` (2004+) / WSUS logs | Update download/installation status |
| **Directory Service** | Event Viewer → Applications and Services Logs → Directory Service → NTDS | AD-specific events on DCs |
| **DNS Server** | Event Viewer → Applications and Services Logs → DNS Server | DNS-specific events |
| **Forwarded Events** | Event Viewer → Windows Logs → Forwarded Events | Collected events from other servers (if WinRM/Event Collectors configured) |
| **Minidump / Crash dumps** | `C:\Windows\Minidump\` | BSOD analysis |

## 1.8 Step-by-Step Troubleshooting Methodology

```
STEP 1: GATHER INFORMATION
  ├── What is the symptom? (Error message, user report, monitoring alert)
  ├── When did it start? (Correlation with changes, updates, deployments)
  ├── What is the impact? (Single server? All users? Specific application?)
  └── Who is affected? (Specific OUs, groups, sites, departments?)

STEP 2: CHECK IMMEDIATE HEALTH
  ├── Ping / network connectivity
  ├── Event Viewer → System & Application (errors/warnings since incident)
  ├── Service status (critical services running?)
  ├── Resource check (CPU, RAM, Disk, Network utilization)
  └── Time synchronization status

STEP 3: ISOLATE THE PROBLEM
  ├── Scope: Single server or multiple?
  ├── Reproduce: Can you consistently reproduce the issue?
  ├── Correlate: Do all affected servers share a common dependency?
  ├── Roll back: Was there a recent change (GPO, update, config)?
  └── Test: Can you access the service from another machine?

STEP 4: DEEP ANALYSIS
  ├── Trace the dependency chain (what does this server depend on?)
  ├── Analyze specific logs for the failing service/component
  ├── Use Process Monitor (ProcMon) for file/registry access issues
  ├── Use Process Explorer for handle/DLL analysis
  ├── Performance Monitor (PerfMon) for bottlenecks
  └── Network Monitor / Wireshark if network-level issues

STEP 5: REMEDIATE
  ├── Apply fix (hotfix, config change, service restart, restore)
  ├── Document the root cause and fix
  ├── Verify the fix (monitor for recurrence)
  └── Communicate to stakeholders

STEP 6: PREVENT RECURRENCE
  ├── Update runbooks/documentation
  ├── Add monitoring/alerting for early detection
  ├── Implement change control to prevent recurrence
  └── Consider architecture improvements (redundancy, HA)
```

## 1.9 How Does It Affect Other Infrastructure Components?

```
If a Windows Server fails:
├── Running VMs on this host → All workloads on it lose compute
│   └── If it's a Hyper-V host: VMs may auto-start on another node (if cluster)
├── AD DS on this server → If DC, authentication breaks, replication halts, FSMO issues
├── DNS on this server → Resolution fails for clients using this DNS server
├── File Server → SMB shares unreachable; DFS replication stops
├── IIS → Web applications down; API endpoints unreachable
├── DHCP → No IP leases; new clients can't join network (existing leases still work until T+scope)
├── WSUS → No patch approval/deployment (but clients still receive updates from Microsoft if direct)
├── Print Server → All shared printers offline
├── SQL Server (if hosted) → Databases inaccessible
└── Monitoring/Alerting → Server goes from "healthy" to "down" in monitoring systems
```

## 1.10 L3-Level Interview Answers — Sample Questions

**Q: What would you do during a production outage where a Windows Server hosting critical applications is unresponsive?**

```
1. IMMEDIATE (First 5 minutes):
   - Confirm the outage (ping, monitoring dashboard, team confirmation)
   - Check if it's isolated or affecting multiple servers/services
   - Initiate war room / communication to stakeholders
   - Determine if it's a Hyper-V host (VM impact) or standalone server

2. DIAGNOSE:
   - If VM: Check Hyper-V host health, VM state (running/paused/crasaned)
   - If physical/bare-metal: Check console access (iLO/iDRAC/IPMI/IASM)
   - Check Event Viewer remotely (if WinRM accessible):
       Get-WinEvent -ComputerName $Server -FilterHashtable @{LogName='System'; Level=1,2}
   - Check resource exhaustion remotely:
       Enter-PSSession → perfmon / tasklist / disk checks
   - Check if it's network isolated (ping, traceroute, ARP tables, firewall rules)

3. REMEDIATE:
   - If VM on Hyper-V cluster: Check if host failed over (Live Migration/VMHA)
   - If hung but running: Remote debug → if unresponsive → force restart (last resort)
   - If crashed/BSOD: Collect crash dump (if possible) → restart → analyze dump
   - If disk failure: Restore from backup or fail over to replica/DR site

4. VERIFY:
   - Service checks after restart
   - Application smoke tests
   - Monitoring confirmation
   - Root cause documentation

5. FOLLOW-UP:
   - RCA (Root Cause Analysis) report
   - Update runbooks
   - Propose infrastructure improvements
```

---

# 🔷 SECTION 2 — ACTIVE DIRECTORY (AD DS)

## 2.1 What Is Active Directory?

Active Directory Domain Services (AD DS) is a **directory service** developed by Microsoft that provides:
- **Authentication** (who are you?)
- **Authorization** (what can you do?)
- **Directory** (a hierarchical database of objects — users, computers, groups, OUs, services)
- **Policy** (Group Policy delivery)
- **Replication** (multi-master replication between domain controllers)

AD DS is built on **LDAP (Lightweight Directory Access Protocol)**, **Kerberos**, and **DNS** for locating services.

## 2.2 Why Is It Used?

| Reason | Explanation |
|--------|-------------|
| **Centralized Authentication** | Single set of credentials for all domain-joined resources |
| **Authorization** | ACLs on objects; group-based access control |
| **Scalable Directory** | Supports millions of objects across multiple domains/sites |
| **Group Policy** | Centralized configuration push to users/computers |
| **Security** | Password policies, account lockout, audit, LAPS, Credential Guard |
| **Single Sign-On (SSO)** | Kerberos tickets enable seamless access across resources |
| **Service Location** | DNS-integrated service records (SRV records) locate DCs, KDCs, etc. |
| **Hierarchy & Delegation** | OUs allow delegated administration (helpdesk resets passwords in specific OUs) |
| **Integration** | Almost all enterprise apps integrate with AD for auth (Exchange, SQL, IIS, SharePoint, etc.) |

## 2.3 How Does It Work? (Deep Architecture)

```
┌──────────────────────────────────────────────────────────────┐
│                    ACTIVE DIRECTORY FOREST                    │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                    DOMAIN (e.g., contoso.com)          │  │
│  │  ┌──────────────────────────────────────────────────┐  │  │
│  │  │              TREE (Partitions)                    │  │  │
│  │  │  ┌──────────────────────────────────────────┐    │  │  │
│  │  │  │         DOMAIN CONTROLLERS                │    │  │  │
│  │  │  │  (DC01, DC02, DC03 - multiple sites)     │    │  │  │
│  │  │  │                                          │    │  │  │
│  │  │  │  NTDS.DIT (Directory Database)            │    │  │  │
│  │  │  │  ├── Schema (object classes & attributes) │    │  │  │
│  │  │  │  ├── Configuration (sites, services)      │    │  │  │
│  │  │  │  ├── Domain (objects in this domain)      │    │  │  │
│  │  │  │  └── Applications (app-specific partitions)│    │  │  │
│  │  │  └──────────────────────────────────────────┘    │  │  │
│  │  └──────────────────────────────────────────────────┘  │  │
│  │  ┌──────────────────────────────────────────────────┐  │  │
│  │  │  Other Domains in Forest (if multi-domain)       │  │  │
│  │  └──────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### Core Mechanisms:

1. **NTDS.DIT** — The actual database file stored at `C:\Windows\NTDS\` on each DC. Contains all directory objects.
2. **LDAP (TCP 389 / 636 for LDAPS)** — Protocol for querying and modifying the directory.
3. **Kerberos (TCP/UDP 88)** — Authentication protocol. AD is a KDC (Key Distribution Center).
4. **Replication** — Multi-master between DCs using **Knowledge of Consistency (KCC)** and **Incoming/Outgoing connection objects**.
5. **Schema** — Defines every object class and attribute that can exist in the directory. Extensible (but restricted).
6. **Global Catalog (GC)** — Partial replica of every domain in the forest, stored on GC servers. Contains a searchable subset of every object.
7. **SYSVOL** — Folder replicated via FRS (deprecated) or **DFSR** (current) containing:
   - Logon scripts
   - Group Policy Templates (GPT)
   - SYSVOL replication data
8. **KCC (Knowledge Consistency Checker)** — Built-in process that builds the replication topology.
9. **FRS / DFSR** — File Replication Service (FRS) was used for SYSVOL until Windows 2008 R2; DFSR is the current engine.

## 2.4 Components Involved

### FSMO Roles (Flexible Single Master Operations):

There are **5 FSMO roles** split across **2 scopes**:

**Forest-wide (1 per forest):**
1. **Schema Master** — Controls all updates to the AD schema
2. **Domain Naming Master** — Controls adding/removing domains in the forest

**Domain-wide (1 per domain):**
3. **RID Master** — Allocates pools of RIDs (Relative Identifiers) to each DC; every security object has a unique SID = Domain SID + RID
4. **PDC Emulator** — Time synchronization source for the domain; password change replication target; processes account lockouts; controls GPO edits in mixed-mode domains
5. **Infrastructure Master** — Updates SID-to-name and DN-to-SID references; should NOT be a GC if multi-domain forest

### Key Objects in AD:

6. **User Objects** — `user`, `inetOrgPerson`, etc.
7. **Computer Objects** — Represent domain-joined machines
8. **Group Objects** — Security groups (can have ACLs) and Distribution Groups (no ACLs)
9. **Organizational Units (OUs)** — Container objects for organizing objects and applying GPOs
10. **Contacts** — Mail-enabled objects (no security principal)
11. **Service Principal Names (SPNs)** — Unique names for service instances (used by Kerberos)
12. **Trusted Domains** — AD trusts (parent-child, forest, external, shortcut, forest trust)
13. **Sites & Subnets** — Define physical network topology for replication and logon optimization
14. **Domain Controllers** — Servers holding the AD DS role
15. **Partition Replication Scope** — Domain partition, Configuration partition, Schema partition, Application partition

### Replication Topology Components:

16. **Knowledge Consistency Checker (KCC)** — Automates replication topology
17. **Connection Objects** — Point-to-point replication paths between DCs
18. **NTDS Settings** — Object on each DC containing replication config
19. **Site Link Bridges** — Connect site links for inter-site replication
20. **Inter-Site Transports** — IP (RPC over TCP) and SMTP (for cross-forest)

### AD Database Components:

21. **NTDS.DIT** — Main directory database
22. **Schema** — ObjectClass and attributeSchema partitions
23. **Configuration Partition** — Stores site, subnet, and service topology info
24. **Application Partition** — Application-specific data (e.g., DNS zones if AD-integrated)
25. **Linked Value Replication** — Optimized replication for multi-valued attributes (e.g., group membership)

## 2.5 What Depends on Active Directory?

```
Active Directory DS (NTDS.DIT, LDAP, Kerberos, Replication)
│
├── Group Policy (GPOs require AD for storage and targeting)
├── DNS (AD-integrated zones stored in AD; AD relies on DNS for DC location)
├── Kerberos Authentication (all domain auth uses Kerberos by default)
├── LDAP Queries (apps, management tools, Exchange all query AD)
├── DHCP (Access-based DHCP enrollment uses AD computer objects)
├── Certificate Services (AD CS queries AD for enrollment policies)
├── WSUS (uses AD groups for computer targeting)
├── SCCM / Intune (AD groups for device/user collections)
├── Exchange Server (uses AD for recipients, mailboxes, permissions)
├── SQL Server (Windows Auth, contained users, SSMS connectivity)
├── IIS (Windows Authentication, App Pool identities)
├── File Server (SMB uses AD for auth; ACLs reference AD objects)
├── Print Server (AD for printer publishing and location)
├── Remote Desktop Services (Connection Broker, licensing)
├── SharePoint / LOB Apps (AD for authentication, authorization)
├── PowerShell Remoting (WinRM uses Kerberos/Negotiate auth)
├── Microsoft 365 Hybrid (Azure AD Connect syncs AD to Azure AD)
├── Azure AD Connect / Pass-through Authentication / Password Hash Sync
├── LAPS (Local Administrator Password Solution — stores passwords in AD)
├── AdminSDBox (Protected Groups → ACLs applied automatically)
└── All Windows authentication (logon, network access, DFS, etc.)
```

## 2.6 What Happens If Active Directory Fails?

| Failure | Impact |
|---------|--------|
| **Single DC goes down** | Redundancy should handle it; but if last DC in a site → site-wide auth issues |
| **PDC Emulator fails** | Password changes not replicated immediately; time drift; GPO editing issues |
| **RID Master fails** | No new RIDs allocated → can't create new security objects (users/computers/groups) |
| **Schema Master fails** | Schema extensions (e.g., Exchange, Lync installs) fail; not a daily operations issue |
| **Domain Naming Master fails** | Can't add/remove domains; not a daily operations issue |
| **All DCs in a site fail** | Users in that site can't authenticate; can still access resources if cached credentials exist (cached logon) |
| **NTDS.DIT corruption** | DC may fail to boot/start AD DS; restore from System State backup or authoritative/non-authoritative restore |
| **Replication breaks** | DCs have stale/inconsistent data; GPOs may not apply consistently; trust issues |
| **Kerberos failure** | Authentication completely fails (NTLM fallback depends on NTLM being enabled; if disabled → total outage) |
| **DNS break (AD-integrated)** | DC location fails; clients can't find DCs; Kerberos fails; name resolution fails |
| **AD forest compromise** | Attacker has Domain Admin → can do anything; bloodhound attacks; Golden/Silver Ticket |
| **Schema corruption** | Can only restore from backup; very difficult to fix live |
| **DoS / Flooding** | LDAP queries saturate DC; responses slow; authentication delays |

## 2.7 How to Identify AD Failure

### Commands & Tools:

```powershell
# 1. Basic AD health
dcdiag /v                    # Comprehensive DC diagnostics
dcdiag /e                     # Test all DCs in enterprise
repadmin /replsummary         # Replication summary
repadmin /showrepl            # Per-DC replication status
repadmin /prq                 # Replication queue (inbound/outbound)
repadmin /from <DC> /to <DC>  # Specific replication path
repadmin /options             # Check DC options (isGlobalCatalog, etc.)

# 2. FSMO role holders
netdom query fsmo             # All 5 FSMO role holders

# 3. AD replication status
Get-ADReplicationPartnerMetadata -Target * -ErrorAction SilentlyContinue
Get-ADReplicationStatus -Target DC01 -Verbose

# 4. LDAP connectivity
ldp.exe                       # GUI tool to test LDAP bind/search
dsquery * -filter "(objectClass=user)" -limit 10

# 5. Kerberos
klist                         # List current tickets
klist tickets
klist purge                   # Clear tickets (force re-auth)
ktpass                        # Key tab creation
SetSPN -L <account>           # Check SPN registration
SetSPN -S <SPN> <account>     # Register SPN

# 6. DNS (critical for AD)
Resolve-DnsName _kerberos._tcp.dc._msdcs.<domain>
Resolve-DnsName gc._msdcs.<domain>
nslookup <domain>

# 7. Event logs
Get-WinEvent -FilterHashtable @{LogName='Directory Service'; Level=1,2} -MaxEvents 50
Get-WinEvent -FilterHashtable @{LogName='Active Directory Web Services'; Level=1,2}

# 8. AD site awareness
nltest /dsgetdc:<domain> /force  # Which DC a client is talking to
nltest /sc_query:<domain>        # Secure channel status

# 9. Secure channel (computer/domain join)
Test-ComputerSecureChannel -Server DC01 -Credential (Get-Credential)
Test-ADServiceAccount -Identity <gMSA>

# 10. Performance counters for AD
Get-Counter '\LDAP\Connections'
Get-Counter '\LDAP\Search Results'
Get-Counter '\NTDS\$Groups\DSC Initializations'

# 11. Check AD Recycle Bin status
Get-ADOptionalFeature -Identity 'Recycle Bin Feature'

# 12. Check AD health comprehensively
Get-ADDomainController | Select-Object Name, OperatingSystem, IPv4Address, IsGlobalCatalog, Site
Get-ADForest | Select-Object Name, ForestMode
Get-ADDomain | Select-Object Name, DomainMode, PDCEmulator, RIDMaster, InfrastructureMaster
```

### Key Logs:

| Log | Event IDs |
|-----|-----------|
| **Directory Service (NTDS)** | 1644 (replication failure), 2080 (DS role started), 2087, 2088 (replication) |
| **Active Directory Web Services** | 2000, 2001, 36886 (TLS issues) |
| **KDC (Kerberos)** | 4 (TGT failure), 7 (bad password), 9 (password change), 11 (APREQ failure) |
| **NDFS (DFSR)** | 13516, 4612, 4614 (SYSVOL/DFSR) |
| **DNS Server** | 4000, 4004, 4006, 4007, 4008, 4013 |
| **NTFRS** (if still using FRS) | 13516, 13568, 13598 |

## 2.8 Step-by-Step Troubleshooting Methodology for AD

```
SCENARIO: Users report they can't log in / "There is no network path to this domain"

STEP 1: IDENTIFY SYMPTOM SCOPE
  ├── All users or specific OU/department?
  ├── All sites or one site?
  ├── Can users still access network resources (cached creds)?
  └── Check: Are ALL DCs down, or just some?

STEP 2: CHECK DNS FIRST (ALWAYS DNS FIRST)
  ├── Client can't find DC → DNS issue
  ├── Resolve: _ldap._tcp.dc._msdcs.<domain>
  ├── Resolve: _kerberos._tcp.dc._msdcs.<domain>
  ├── Check DC's DNS server settings (points to itself? to correct DNS server?)
  └── Test: nslookup, Resolve-DnsName, dcdiag /test:dns

STEP 3: CHECK DC AVAILABILITY
  ├── Ping DCs
  ├── Can you RDP to DC? (if not, check network/firewall/VM health)
  ├── dcdiag /v on each DC
  ├── Check DC event logs (Directory Service log)
  ├── Check if Netlogon service is running
  └── Check if NTDS.DIT is corrupt (event 1644, 1987, 1988)

STEP 4: CHECK REPLICATION
  ├── repadmin /replsummary
  ├── If failures → check specific DC pairs
  ├── Check event 2087 (last-successful replication) and 2088 (last-failed)
  ├── Verify sites/subnets are configured correctly
  └── Check if KCC has errors (event 2006 - KCC failed to create topology)

STEP 5: CHECK FSMO ROLES
  ├── netdom query fsmo
  ├── If a FSMO role holder is down → seize (dcpromo /forceremoval or Move-ADDirectoryServerOperationMasterRole)
  └── Never assume; verify which role and whether to seize or transfer

STEP 6: CHECK KERBEROS
  ├── klist on client → see TGT?
  ├── klist get krbtgt/<domain> → request TGT manually
  ├── Check KDC event log (events 4, 7, 9)
  ├── Verify time sync (5-minute skew): w32tm /query /status
  └── Check SPN registration (duplicate or missing SPNs cause auth failures)

STEP 7: CHECK SECURE CHANNEL
  ├── Test-ComputerSecureChannel (on machine that can't auth)
  ├── Reset: Reset-ComputerMachinePassword -Server DC01 -Credential (Get-Credential)
  └── nltest /sc_reset:<domain>

STEP 8: REMEDIATE
  ├── Restart services (Netlogon, NTDS, DNS, KDC)
  ├── If DC is corrupt → authoritative restore (ntdsutil) or non-authoritative + reseed
  ├── If DNS corrupt → rebuild DNS role, re-seed AD-integrated zones
  ├── If FSMO role holder dead → seize role to healthy DC
  └── If entire DC needs rebuild → dcpromo to demote, rebuild, promote again

STEP 9: VERIFY & DOCUMENT
  ├── User logins work
  ├── Replication healthy (repadmin /replsummary)
  ├── Kerberos tickets issued (klist)
  ├── Document root cause and resolution
  └── Update runbook
```

## 2.9 How Does AD Affect Other Infrastructure Components?

```
If AD goes completely down:
├── Authentication → NO domain logins (cached logon still works until offline)
├── Group Policy → No new GPOs applied; existing GPO still cached locally until refresh
├── DNS (AD-integrated zones) → AD DNS zone replication stops; but cached DNS may still resolve
├── Kerberos → All Kerberos auth fails; NTLM fallback kicks in (if enabled)
├── LDAP → Apps using LDAP auth (Exchange, LOB apps) fail to authenticate
├── DHCP → DHCP service itself doesn't need AD, but DHCP-DCL authorization fails
├── File Server → ACL evaluation needs AD; if DC unreachable, existing sessions may stay
├── Exchange → Cannot expand groups, GAL access fails, recipient policies stop
├── Remote Desktop Services → Connection Broker (if AD-dependent) fails
├── WSUS → Computer group targeting (if AD-based) breaks
├── LAPS → Password reads from AD fail; admin resets may not work
├── New computer joins to domain → Fails (cannot locate DC, cannot authenticate join)
├── New user creation → Fails (no DC available)
└── PowerShell remoting (using Kerberos) → Fails
```

## 2.10 Difference Between Related Technologies

| Technology | What It Is | Key Difference |
|-----------|-----------|---------------|
| **AD DS** | Directory service (domain) | Authentication, authorization, directory |
| **AD LDS** (ADAM) | Lightweight directory | No domain, no GPOs, app-specific directory (port 50000+) |
| **AD CS** | Certificate services | PKI, certificates, enrollment |
| **AD FS** | Federation services | SSO across trust boundaries (external partners, cloud) |
| **AD RMS** | Rights management | Document/email-level information protection |
| **AD CR** | Certificate Registration | Auto-enrollment for certificates |
| **Azure AD** (Entra ID) | Cloud directory | Not a domain controller; cloud-based identity; no GPOs; uses MDM/MAM |
| **Workgroup** | Non-domain | Local SAM database only; no centralized auth |
| **NTLM** | Auth protocol | Challenge-response; less secure than Kerberos; fallback only |

## 2.11 L3-Level Interview Answer: Production Outage Scenario

**Scenario: "After a scheduled Windows Update reboot, one of your three DCs (DC02 in Site B) came back up but users in Site B report slow logons and some are failing."**

```
1. TRIAGE (5 min):
   - Confirm: Which DCs are up? DC01 (Site A) ✓, DC02 (Site B) ???, DC03 (Site A) ✓
   - Ping DC02 from Site B client; resolve DNS for DC02
   - Check nltest /dsgetdc:<domain> from affected client — which DC is it targeting?

2. DNS CHECK:
   - Is DC02's DNS server role running? Get-Service DNS on DC02
   - Is DNS resolving correctly from Site B? Resolve-DnsName DC02
   - Check DNS event logs for zone loading errors

3. DC CHECK (DC02):
   - RDP to DC02 (if possible) or use PSEXEC/WinRM
   - Event Viewer: Directory Service log (errors since boot)
   - dcdiag /v /s:DC02
   - repadmin /showrepl — is DC02 replicating?
   - Check NTDS.DIT size and disk space on C: drive of DC02
   - Is Netlogon service running? (critical for secure channel)

4. LDAP/KERBEROS CHECK:
   - ldp.exe connect to DC02 — can you bind?
   - klist on affected client — do you have a TGT?
   - Check time: w32tm /query /status (DC02 is it syncing time?)

5. COMMON ROOT CAUSES AFTER UPDATE REBOOT:
   ├── AD DS service didn't start (event 2087, service account issue)
   ├── DNS role didn't start (DNS servers list empty, misconfigured)
   ├── Netlogon service not running (secure channel broken)
   ├── DC02 lost AD DS role somehow (shouldn't happen, but check dcdiag)
   ├── IP/subnet/site membership mismatch after IP change
   └── Replication broke between DC02 and others

6. REMEDIATE:
   - If DNS didn't start → start DNS service; check DNS server list in adapter settings
   - If Netlogon not running → start it; reset secure channel
   - If AD DS service crashed → restart; check why (disk space? corrupt NTDS.DIT?)
   - If replication broke → repadmin /syncall /A (force sync)
   - If clients pointing to dead DC → flush DNS (ipconfig /flushdns); netdom resetpwd

7. VERIFY:
   - Users in Site B can log in (monitor)
   - repadmin /replsummary is clean
   - nltest /dsgetdc:<domain> from Site B returns healthy DC
   - Check event logs for recurrence

8. PREVENTION:
   - After Windows Update reboots, verify DC services auto-start
   - Ensure DC DNS is configured with internal DNS server (not ISP/external)
   - Review update process: update DCs during maintenance window, verify before leaving
   - Add monitoring alert: DC service down, DNS zone load failure
   - Consider adding a DC at Site B if redundancy is insufficient
```

---

# 🔷 SECTION 3 — GROUP POLICY (GPO)

## 3.1 What Is Group Policy?

Group Policy is a **hierarchical mechanism** within Windows that allows administrators to:
- **Configure** settings for users and computers (security, software installation, drive mappings, scripts, etc.)
- **Enforce** rules (password policies, account lockout, audit policies)
- **Deploy** software and updates
- **Map drives and printers**
- **Run scripts** (logon/logoff, startup/shutdown)
- **Apply security baselines** (AppLocker, BitLocker, Windows Defender policies)

GPOs are stored in Active Directory and applied to **Active Directory objects** (users, computers) within **Organizational Units (OUs)** or at domain/site level.

## 3.2 Why Is It Used?

| Reason | Explanation |
|--------|-------------|
| **Centralized Configuration** | One GPO can configure thousands of computers simultaneously |
| **Security Baseline** | Enforce password complexity, account lockout, audit policies |
| **Consistency** | All domain-joined machines have identical base configuration |
| **Scalability** | Apply settings to OUs, entire domains, or entire forests |
| **Software Deployment** | Push MSI/MSIX/EXE packages to users or computers |
| **Drive Mapping** | User-based drive maps based on group membership |
| **Printer Deployment** | Auto-connect printers based on user/computer location |
| **Scripting** | Logon scripts, startup scripts, scheduled tasks via GPO |
| **Regulatory Compliance** | Enforce settings required by standards (CIS, NIST, SOX) |

## 3.3 How Does GPO Work? (Processing & Application)

### GPO Processing Order (LSDOU):

```
┌──────────────────────────────────────────────────────┐
│         GPO Processing Order: LSDOU                  │
│                                                      │
│  L = Local       (Local Group Policy on the computer)│
│  S = Site        (Site-linked GPOs - processed first)│
│  D = Domain      (Domain-linked GPOs)               │
│  OU = Organizational Unit (OU-linked GPOs, nearest to object processed last)│
│                                                      │
│  LAST ONE WINS for non-security settings (overrides) │
│  FIRST WINS for security settings (deny overrides)   │
└──────────────────────────────────────────────────────┘
```

### GPO Processing Steps (gpresult /r to see):

1. **Computer Startup Phase:**
   - Computer identifies its site (based on AD Site/Subnet)
   - Applies **computer-side** GPO settings in order:
     - Site policies → Domain policies → OU policies
   - Processes: Startup scripts, software installations (per-machine), security policies
   - After processing, creates **GPC (Group Policy Container)** in AD and **GPT (Group Policy Template)** in SYSVOL

2. **User Logon Phase:**
   - User authenticates via Kerberos
   - User identifies their site
   - Applies **user-side** GPO settings in order:
     - Site policies → Domain policies → OU policies
   - Processes: Logon scripts, drive mappings, printer connections, user-side software

3. **Background Refresh:**
   - Every 90 minutes (random offset 0-30 min) for computers
   - Every 60 minutes for user
   - At logon, full reapplication

### GPO Architecture:

```
GPO in AD
├── GPC (Group Policy Container) — AD object
│   ├── gPCFileSysPath (path to GPT)
│   ├── gPCMachineExtensionNames (CSEs for computer)
│   ├── gPCUserExtensionNames (CSEs for user)
│   ├── gPCWQLQuery (WMI filter)
│   ├── gPCOptions (GPO status: enabled/disabled/enforced/blocked)
│   └── Version number (incremented on changes)
│
└── GPT (Group Policy Template) — File system (SYSVOL)
    ├── GPT.INI (version info, description)
    ├── Machine\
    │   ├── Registry.pol (Registry settings)
    │   └── Scripts (Startup/Shutdown scripts)
    ├── User\
    │   ├── Registry.pol (Registry settings)
    │   └── Scripts (Logon/Logoff scripts)
    ├── Modified Access Based Enforcement (ABE) lists
    └── PowerPoint templates, etc. (if any)
```

### Group Policy Client Side Extensions (CSEs):

CSEs are the handlers that actually *do something* with GPO settings:
- **Registry CSE** → Applies registry.pol settings
- **Security Options CSE** → Enforces security policies
- **Software Installation CSE** → Installs MSI/XAP packages
- **Drive Maps CSE** → Maps network drives
- **Printers CSE** → Connects printers
- **Scripts CSE** → Runs scripts
- **Folder Redirection CSE** → Redirects folders (Desktop, Documents, etc.)
- **RSOP CSE** → Resultant Set of Policy
- **WMI Filter CSE** → Filters GPO application based on WMI query
- **AppLocker CSE** → Enforces application allow/deny rules
- **BitLocker CSE** → Applies BitLocker policies
- **Windows Defender CSE** → AV/EDR policies
- **PowerShell CSE** → Execution policy, module lists

## 3.4 Components Involved

1. **GPO (Group Policy Object)** — The container containing settings
2. **GPC (Group Policy Container)** — AD representation of GPO
3. **GPT (Group Policy Template)** — File system representation in SYSVOL
4. **Registry.pol** — Binary file containing registry settings
5. **WMI Filter** — Query to determine if GPO applies (e.g., "is this a laptop?")
6. **Security Groups** — GPO targeting via group membership (Access-Based Enumeration)
7. **Security Filtering** — Read permission on GPO for specific groups
8. **WMI Filter** — Boolean query against WMI data on client
9. **APPLC (Application Compatibility Layer)** — Compatibility for legacy settings
10. **CSE (Client Side Extension)** — Handler for specific GPO settings
11. **GPMC (Group Policy Management Console)** — GUI tool for managing GPOs
12. **GPResult** — Command-line tool to view applied GPOs
13. **gpupdate /force** — Force immediate GPO reapplication
14. **Sysvol** — File share containing GPT files (DFSR-replicated)
15. **AGDLP (Account → Global → Domain Local → Permission)** — Best practice for GPO permissions

## 3.5 What Depends on GPO?

```
Group Policy (GPO processing, CSEs, SYSVOL, WMI filters)
│
├── Security Policies (Password, Account Lockout, Audit)
├── Software Deployment (MSI/XAP installations)
├── Drive Mappings (user environment)
├── Printer Connections
├── Folder Redirection (Profile management)
├── Scripts (Logon, Startup, Logoff, Shutdown)
├── AppLocker (Application control)
├── BitLocker (Device encryption)
├── Windows Defender / Security Policies
├── Network Settings (proxy, mapped drives)
├── Scheduled Tasks (via GPO)
├── Disable/Enable features (Control Panel, USB, etc.)
├── Administrative Templates (all registry-based settings)
├── Search Options, Desktop Icons, Start Menu
├── Internet Explorer/Edge settings
├── DHCP Classification (based on GPO)
├── WSUS Computer Targeting (if AD-based)
├── RDP settings (allow/block remote desktop)
├── Power Settings (laptop battery management)
├── Mapped printers via Print Management
├── Credential Delegation (Constrained Delegation policies)
├── LAPS (GPO configures password rotation)
├── Login Scripts and Environment Variables
└── All user/computer configuration in enterprise
```

## 3.6 What Happens If GPO Fails?

| Failure Type | Impact |
|-------------|--------|
| **GPO not applying** | Config changes missing; inconsistent settings across machines |
| **SYSVOL not replicating** | GPOs can't be downloaded; no settings applied (DFSR failure) |
| **GPO processing error** | Event 1058/1030 — GPO skipped due to slow links or errors |
| **WMI filter failure** | GPO may apply to wrong machines (or not apply when it should) |
| **Software install failure** | Apps not deployed; installation failures in Application log |
| **Security policy not enforced** | Password policy, audit policy don't apply (huge security risk) |
| **Drive mapping fails** | Users can't access network resources; helpdesk tickets spike |
| **Folder redirection fails** | User profile stays local; data not on file server; profile corruption |
| **AppLocker misconfigured** | Block legitimate apps → productivity halts |
| **BitLocker GPO failure** | Encryption not applied; non-compliant systems |
| **DNS records fail (LAPS)** | LAPS password not written to AD attribute |
| **GPO corruption** | Import/export issues; backup restoration needed |
| **Geneva / AGP issues** | Centralized policy for non-domain systems (cloud/GPO agility) |
| **Slow GPO processing** | Slow logons (many GPOs, large policies, scripts) |
| **Loopback processing failure** | Computer-based policies don't apply to users as expected |
| **Permission misconfiguration** | GPO security filtering prevents application; ERS (no prior policies) |
| **AD Replication delays** | GPO changes not propagated to all DCs; inconsistent application |

## 3.7 How to Identify GPO Failure

### Commands & Tools:

```powershell
# 1. What GPOs are applied?
gpresult /r                           # Show applied GPOs (current user)
gpresult /r /user <username>          # For a specific user
gpresult /h C:\gpreport.html          # HTML report
gpresult /x C:\gpreport.xml           # XML report
gpresult /scope Computer              # Computer settings only
gpresult /scope User                 # User settings only
gpresult /v                           # Detailed (CSEs, extension modes)

# 2. Force GPO update
gpupdate /force                       # Force reapplication
gpupdate /target:<ComputerName>       # For a specific computer (Win10+)

# 3. Check GPO status (on DC or client)
Get-GPResultantSetOfPolicy -Report Xml -Path C:\GPO.xml -User <User> -Computer <Computer>

# 4. Check WMI filter
Get-GPResultantSetOfPolicy -Report Xml -Path C:\GPO.xml | Select-String -Pattern "WMI"

# 5. Verify GPO links
Get-GPInheritance -Target "OU=Sales,DC=contoso,DC=com"
Get-GPLink -Target <GPOName>

# 6. Check WMI filter evaluation
Get-WmiObject -Class Win32_ComputerSystem | Select-Object Manufacturer, Model, Name

# 7. Check for slow GPO processing
Get-EventLog -LogName "Application" -Source "GroupPolicy" -After (Get-Date).AddMinutes(-30)
# Event ID 4006 (slow processing > 15 min)
# Event ID 4016 (slow user profile)
# Event ID 1058 (slow link processing)

# 8. Check GPO processing errors
Get-EventLog -LogName Application -Source GroupPolicy
# Event 1058: GPO skipped due to slow link
# Event 1030: GPO not applied (also includes WMI filter failure)
# Event 1085: Application installation failed
# Event 1086: Extension failed (specific to CSE)
# Event 1098: User policy has no user version
# Event 1129: Drive map failed

# 9. Check SYSVOL health (DFSR)
Get-DfsrMembership -ComputerName DC01
Get-DfsrContentionPath -ComputerName DC01
dfsrdiag.exe dumpset                       # Detailed DFSR status on DC

# 10. Check GPO version and content
Get-GPO -All | Select DisplayName, GPOStatus, Description
Get-GPO -Name <GPOName> | Select-Object Id, DisplayName, Status
Get-GPRegistryValue -Name <GPOName> -Key "HKLM\..."

# 11. Debug GPO application
logman gpconsum -start -o C:\GPLog.etl -p "Microsoft-Windows-GroupPolicy" 0x1000
# Or use: Group Policy Operational log
Get-WinEvent -LogName "Microsoft-Windows-GroupPolicy/Operational"

# 12. Check specific registry settings applied
reg query "HKLM\Software\Policies" /s
reg query "HKCU\Software\Policies" /s

# 13. Group Policy logs
C:\Windows\debug\gpresults\ — old location
Use Event Viewer → Applications and Services Logs → Microsoft → Windows → GroupPolicy

# 14. PowerShell GP result
Get-GPResultantSetOfPolicy -Report Html -Path C:\GPOReport.html -Computer $env:COMPUTERNAME -User $env:USERNAME
```

### Key Logs and Event IDs:

| Event Source | Event ID | Meaning |
|-------------|----------|---------|
| **GroupPolicy/Operational** | 4001 | GPO processing started |
| **GroupPolicy/Operational** | 4002 | GPO processing succeeded |
| **GroupPolicy/Operational** | 4003 | GPO processing failed |
| **GroupPolicy/Operational** | 4004 | User policy applied |
| **GroupPolicy/Operational** | 4005 | Computer policy applied |
| **GroupPolicy/Operational** | 4006 | Slow processing (warning) |
| **GroupPolicy/Operational** | 4007 | Slow user processing (warning) |
| **GroupPolicy/Operational** | 4008 | GPO processing in background |
| **GroupPolicy/Operational** | 4010 | Error reading GPO |
| **GroupPolicy/Operational** | 4011 | GPOWMI evaluation |
| **GroupPolicy/Operational** | 4012 | GPO pre-check failed |
| **Application** | 1058 | Slow link (GPO processing skipped) |
| **Application** | 1030 | GPO not applied (filtering) |
| **Application** | 1085 | Software install failure |
| **Application** | 4098 | Security policy settings applied |
| **Application** | 4099 | Security settings processing error |

## 3.8 Step-by-Step GPO Troubleshooting Methodology

```
SCENARIO: A new GPO has been linked to an OU, but computers/users in that OU are not getting the settings.

STEP 1: VERIFY GPO EXISTS AND IS LINKED
  ├── Open GPMC (gpmc.msc)
  ├── Navigate to the linked OU
  ├── Confirm GPO link is present and ENFORCED (if needed)
  ├── Confirm GPO is not DISABLED (GPO Status = "Enabled")
  └── Check link order (lower = processed first within same level)

STEP 2: CHECK GPO SCOPE
  ├── Security Filtering: Does the Authenticated Users / specific group have Read + Apply Group?
  ├── WMI Filter: Is the WMI query correct? Does it evaluate to TRUE on the client?
  ├── Item-Level Targeting (ILT): If used, does target match current user/computer?
  ├── Loopback Processing: Is it configured (if applicable)?
  └── Security Groups: Is user/computer a member of the required security group?

STEP 3: CHECK GPO PROCESSING
  ├── On client: gpresult /h report.html
  ├── Check if GPO appears in the report
  ├── If GPO not listed → It's NOT being applied (go back to Step 1/2)
  ├── If GPO listed but settings not applied → CSE error (check Application log)
  └── Check Event Viewer → Group Policy Operational log for detailed events

STEP 4: CHECK SYSVOL REPLICATION
  ├── Is SYSVOL DFSR-replicating between DCs?
  ├── DFSR management: Get-DfsrMembership (on DCs)
  ├── Sysvol share accessible? \\<DC>\SYSVOL\<domain>\Policies\<GUID>\
  ├── GPT.INI present and version updated?
  ├── Check DFSR event logs (event 13516, 13598)
  └── FRS pre-W2008R2: check NTFRS (mstsmmgmt.msc)

STEP 5: CHECK FOR CONFLICTS
  ├── GPO processing order (LSDOU) — is another GPO overriding?
  ├── Enforced links (Enforced = override block)
  ├── Block Inheritance on OU? (blocks GPOs from above)
  ├── Security filtering (deny > allow principle)
  └── Use gpresult to compare expected vs. actual

STEP 6: APPLY AND VERIFY
  ├── gpupdate /force (on client)
  ├── Check again: gpresult /r
  ├── Verify specific settings (registry, group, etc.)
  ├── Check Event Viewer for errors
  └── Logoff/logon test if user-side policy

STEP 7: REMEDIATE
  ├── Fix permissions (Security Filtering → Authenticated Users: Read + Apply)
  ├── Remove/fix WMI filter
  ├── Fix WMI query syntax
  ├── Fix broken SYSVOL replication
  ├── Remove conflict (another GPO overriding this one)
  └── Enforce GPO if necessary (Enforced checkbox)

STEP 8: VERIFY & DOCUMENT
  ├── Confirmed settings applied across all target clients
  ├── gpresult report shows GPO applied with all settings
  ├── No Event ID 1058/1030 errors
  └── Document findings
```

## 3.9 How Does GPO Affect Other Infrastructure Components?

```
If GPO fails or is misconfigured:
├── Security Policies → Password complexity, lockout not enforced → Security risk
├── AppLocker → Applications may or may not run; inconsistent security
├── BitLocker → Devices not encrypted → Non-compliance
├── Drive Mappings → Users can't access resources → Productivity loss
├── Printer Connections → Users can't print → Helpdesk tickets
├── Folder Redirection → Data stays on local machine → Data loss risk, profile corruption
├── Software Deployment → Apps not installed → User complaints, version inconsistencies
├── Scripts → Logon scripts don't run → Environment variables, mappings missing
├── Power Settings → Laptop battery management inconsistent → Hardware wear
├── Windows Updates → GPO for WSUS targeting fails → Patch compliance gaps
├── Windows Defender → AV definitions/policies not applied → Security risk
├── Credential Delegation → Constrained delegation fails → Auth failures for double-hop
├── LAPS → Admin passwords not rotated/read → Security/operational impact
├── RDP Access → GPO may block RDP → Remote management impossible
├── Blocked Controls → Users may access Control Panel → Security risk
├── Time Zone → Inconsistent time zones → Kerberos/AD issues
├── Drive Access → USB restriction fails → Data exfiltration risk
└── All Kerberos delegation (Constrained/Unconstrained) → Trust chain issues
```

## 3.10 Key Differences

| Feature | Computer Configuration | User Configuration |
|---------|----------------------|-------------------|
| **Applies at** | Computer startup | User logon |
| **Processing phase** | Before user logs in | After user authenticates |
| **Examples** | Security policies, software install (per-machine), scripts (startup), Windows Update | Drive maps, printer connections, folder redirection, scripts (logon), Start menu |
| **GPO linked to** | Site, Domain, OU | Same (but user-side settings) |
| **Background refresh** | Every 90 min | Every 60 min |

## 3.11 L3-Level Interview Answer: GPO Outage Scenario

**Scenario: "A security GPO was deployed to lock down USB drives via AppLocker. After gpupdate, users in Sales report they can't open Excel and their mapped drives disappeared. What do you do?"**

```
1. IMMEDIATE Triage:
   - Confirm it's GPO-related (timing aligns with GPO deployment)
   - Determine scope: all Sales? Some? Test user vs. production user?

2. CHECK GPRESULT:
   - On affected user: gpresult /h C:\gpreport.html
   - Check if the security GPO is listed and what's in it
   - Identify what's new vs. before

3. CHECK APPLOCKER:
   - Open Event Viewer → Applications and Services Logs → AppLocker
   - Event 8001 (rule collection not loaded)
   - Event 8002 (DLL/SFC check failed)
   - Event 8026 (executable blocked) — Excel blocked
   - Is AppLocker service running? (AppIDSvc)

4. CHECK MAPPED DRIVES:
   - GPO drive maps likely failed
   - Check gpresult — is the drive map GPO still listed?
   - Check Application log: Event 1129, 1130 (drive map failures)
   - Network path accessible? (fileshare DNS resolution, connectivity)

5. DETERMINE ROOT CAUSE:
   ├── AppLocker GPO misconfigured → blocked Excel (because AppLocker rules are too restrictive)
   ├── Drive map GPO changed → network path no longer exists or DNS broke
   ├── GPO processing order changed → another GPO overrode
   ├── DFSR issue → SYSVOL not updated → old GPO being applied
   └── Permissions change → user no longer has access to file share

6. IMMEDIATE REMEDIATION:
   ├── Roll back the GPO (unlink it temporarily, or set to Block Inheritance while fixing)
   ├── Or fix the specific setting in the GPO (add Excel exception to AppLocker rules)
   ├── Run gpupdate /force after fix
   ├── Verify users can access Excel and drives
   └── If all else fails: revert from GPO backup

7. FORENSIC WORK:
   ├── Compare GPO backup before/after change
   ├── Check if AppLocker rules were exported from wrong GPO
   ├── Check for conflicts: "Enforce" on a different GPO blocking Excel
   └── Check SYSVOL content for the GPO

8. PREVENTION:
   ├── Test GPOs in a pilot OU first
   ├── Use GPO staging (linking WMI filter or security group for pilot)
   ├── Maintain GPO backups (scripted backup: Backup-GPO)
   ├── Add AppLocker exception list for Office apps in every GPO
   ├── Document changes with Change Request (CR)
   └── Monitor: Set alert on Event ID 8026 (AppLocker blocking)
```

---

# 🔷 SECTION 4 — DNS

## 4.1 What Is DNS?

DNS (Domain Name System) is a **distributed, hierarchical naming system** that translates human-readable domain names (e.g., `contoso.com`) into IP addresses (e.g., `192.168.1.10`). It is one of the **most critical infrastructure components** — without DNS, virtually nothing works: no web browsing, no email, no authentication (Kerberos relies on DNS for KDC location), no service discovery.

## 4.2 Why Is It Used?

| Reason | Explanation |
|--------|-------------|
| **Name Resolution** | Human-friendly names → IP addresses |
| **Service Discovery** | Locate AD DS, Kerberos, LDAP, Global Catalog, etc. via SRV records |
| **Load Balancing** | Multiple A/AAAA records for round-robin distribution |
| **High Availability** | Multiple DNS servers for redundancy |
| **Email Routing** | MX records route email to correct mail server |
| **Security** | DNSSEC (signed zones), DNS sinkholing for malware domains |
| **Internal Namespace** | Internal-only zones (e.g., corp.contoso.com) for internal resources |
| **Split-Brain DNS** | Different answers for internal vs. external queries |

## 4.3 How Does DNS Work? (Resolution Process)

### The DNS Query Flow:

```
CLIENT: "I need the IP for server01.contoso.com"
│
▼
┌─────────────────────────────────────────────────────────┐
│ 1. CLIENT CHECKS LOCAL CACHE                            │
│    (DNS cache: ipconfig /displaydns)                    │
│    Found? → Return IP. Done.                            │
│    Not found ↓                                          │
└─────────────┬───────────────────────────────────────────┘
              ▼
┌─────────────────────────────────────────────────────────┐
│ 2. CLIENT CHECKS HOSTS FILE                             │
│    C:\Windows\System32\drivers\etc\hosts                │
│    Found? → Return IP. Done.                            │
│    Not found ↓                                          │
└─────────────┬───────────────────────────────────────────┘
              ▼
┌─────────────────────────────────────────────────────────┐
│ 3. CLIENT QUERIES CONFIGURED DNS SERVER                 │
│    (Default gateway DNS, or specific DNS server)         │
│    This is the RECURSIVE resolver                       │
└─────────────┬───────────────────────────────────────────┘
              ▼
┌─────────────────────────────────────────────────────────┐
│ 4. RECURSIVE DNS SERVER CHECKS ITS CACHE                │
│    Found (and TTL not expired)? → Return IP.            │
│    Not found ↓                                          │
└─────────────┬───────────────────────────────────────────┘
              ▼
┌─────────────────────────────────────────────────────────┐
│ 5. RECURSIVE SERVER QUERIES ROOT SERVERS (.)            │
│    ("Who handles .com?")                                │
│    Root servers refer to TLD (.com) servers             │
└─────────────┬───────────────────────────────────────────┘
              ▼
┌─────────────────────────────────────────────────────────┐
│ 6. TLD (.com) SERVERS RETURN:                           │
│    "contoso.com's authoritative servers are ns1, ns2"   │
└─────────────┬───────────────────────────────────────────┘
              ▼
┌─────────────────────────────────────────────────────────┐
│ 7. AUTHORITATIVE DNS SERVER FOR CONTOSO.COM             │
│    Returns the actual A/AAAA record: 192.168.1.10      │
└─────────────┬───────────────────────────────────────────┘
              ▼
┌─────────────────────────────────────────────────────────┐
│ 8. RECURSIVE SERVER CACHES & RETURNS ANSWER             │
│    to client. Client caches result (with TTL).          │
└─────────────────────────────────────────────────────────┘
```

### DNS Query Types:

1. **Recursive Query** — Client asks server to do all the work (returns answer or error)
2. **Iterative Query** — Server returns best referral (tells client who to ask next)
3. **Non-Recursive Query** — Server has answer in cache (returns immediately)

### DNS Record Types:

| Record Type | Purpose | Example |
|------------|---------|---------|
| **A** | IPv4 address | server1.contoso.com → 192.168.1.10 |
| **AAAA** | IPv6 address | server1.contoso.com → 2001:db8::1 |
| **CNAME** | Canonical name (alias) | www.contoso.com → webapp.contoso.com |
| **MX** | Mail exchanger | contoso.com → mailserver.contoso.com (priority 10) |
| **NS** | Name server | contoso.com → ns1.contoso.com |
| **PTR** | Reverse lookup (IP → name) | 192.168.1.10 → server1.contoso.com |
| **SRV** | Service location (used by AD!) | _ldap._tcp.dc._msdcs.contoso.com → DC01:389 |
| **SOA** | Start of Authority (zone info) | contoso.com → ns1.contoso.com, serial, refresh, retry, expire, TTL |
| **TXT** | Text (verification, SPF, DKIM, etc.) | Verification codes, email security |
| **DNAME** | DNAME redirect | Redirects a subtree |
| **AAAA** | IPv6 | |

## 4.4 Components Involved

1. **DNS Server Service** — The DNS Server role on Windows Server (DNS Server service, `dns.exe`)
2. **DNS Client Service** — The DNS Client service on Windows (manages client cache, registers A/PTR records)
3. **DNS Zones** — Authoritative storage containers:
   - **Primary Zone** — Read/write copy
   - **Secondary Zone** — Read-only copy (zone transfer from primary)
   - **Stub Zone** — Contains only NS, SOA, glue A records
   - **Forward Lookup Zone** — Name → IP
   - **Reverse Lookup Zone** — IP → Name
4. **Forwarders** — DNS servers to which queries are forwarded (e.g., ISP DNS, Google 8.8.8.8)
5. **Conditional Forwarders** — Forward specific domains to specific servers (e.g., partner domain)
6. **Root Hints** — List of root DNS servers (if not using forwarders)
7. **DNS Scavenging** — Automatic removal of stale records
8. **DNSSEC** — Cryptographic signing of DNS records
9. **DNS Policies** — Query resolution based on policies (QoS, filtering, routing)
10. **DNS Server Zones File** — Zone data file (e.g., `C:\Windows\System32\dns\<zone>.dns`)
11. **Event Logs** — DNS Server log, DNS Client log
12. **DnsCache** — DNS Client service cache
13. **SRV Records** — Critical for AD (DC location, Kerberos, GC)
14. **DHCP Integration** — DHCP option 006 (DNS servers), option 015 (domain name)
15. **AD-Integrated Zones** — Zones stored in AD, replicated via AD replication (multi-master)

## 4.5 What Depends on DNS?

```
DNS (Recursive & Authoritative Resolution)
│
├── Active Directory (AD DS) — EVERYTHING depends on DNS:
│   ├── DC Location (SRV records: _ldap._tcp.dc._msdcs, _kerberos._tcp.dc._msdcs)
│   ├── GC Location (gc._msdcs.<domain>)
│   ├── KDC Location (Kerberos authentication)
│   ├── AD-Integrated DNS zones (replication via AD)
│   └── AD DS installation wizard requires DNS
├── DNS (recursive) → Forwarders, root hints, conditional forwarders
├── DHCP → Option 006 (DNS server list), Option 015 (domain name)
├── Internet Access → Without DNS, URLs don't resolve
├── Email → MX records route email
├── Web Applications → URL resolution for IIS, web apps
├── PowerShell Remoting (WinRM) → Server name resolution
├── All Application Connections → App connects to "db01.contoso.com" — DNS resolves
├── Network Access (VPN, DirectAccess, Always On VPN) — VPN server name resolution
├── Monitoring & Management → Server names used in all monitoring tools
├── Load Balancers (NLB, HAProxy, F5) — Front-end names resolve via DNS
├── SSL/TLS Certificates → CN/SAN matches DNS name
├── SCCM/MECM → Site system names, distribution points
├── SCOM/SCSM/Monitoring → Server names for monitoring targets
├── Cross-Domain Trusts → DNS conditional forwarders for inter-domain resolution
├── DNSSEC Validation → Chain of trust depends on DNS resolution
├── Remote Desktop Gateway → RD Gateway server name resolution
└── All Service Discovery → WS-Discovery, mDNS rely on DNS in enterprise
```

## 4.6 What Happens If DNS Fails?

| Failure Type | Impact |
|-------------|--------|
| **DNS Server service stops** | Queries to that server fail; clients can't resolve (if they have other servers, failover) |
| **DNS zones missing/corrupt** | Authoritative answers fail; AD can't locate DCs; email breaks; web apps fail |
| **Forwarders unreachable** | External queries fail (internet, email sending/receiving); internal may still work |
| **Conditional forwarder broken** | Cross-domain trust lookups fail; partner domain resources unreachable |
| **Stub zone not updating** | Stale data; incorrect referrals |
| **AD-integrated DNS zone not replicating** | DCs have different DNS data; DC location inconsistent; AD replication breaks |
| **Root hints missing/broken** | If no forwarders, external resolution completely fails |
| **DNS scavenging misconfigured** | Stale records clutter zones; old servers appear active |
| **DNSSEC failure** | Signed zones fail validation; clients may reject answers |
| **DNS Client cache poison** | Wrong IP for services; authentication goes to wrong server |
| **DNS servers not in DHCP option** | Clients don't know which DNS to use → no resolution |
| **Split-brain DNS misconfigured** | Internal queries get external answers (or vice versa) |
| **All DNS servers fail** | COMPLETE INFRASTRUCTURE FAILURE — nothing works |

## 4.7 How to Identify DNS Failure

### Commands & Tools:

```powershell
# 1. Basic name resolution
Resolve-DnsName server01.contoso.com
nslookup server01.contoso.com
nslookup server01.contoso.com 8.8.8.8   # Query specific DNS server

# 2. Check DNS server (is it running?)
Get-Service DNS
Test-NetConnection -ComputerName DC01 -Port 53    # Is port 53 open?

# 3. Check DNS client configuration
Get-DnsClientServerAddress
Get-DnsClientGlobalSetting
Get-DnsClientCache                       # Show client's DNS cache
Clear-DnsClientCache                   # Flush DNS cache

# 4. Check specific record types
Resolve-DnsName _ldap._tcp.dc._msdcs.contoso.com -Type SRV
Resolve-DnsName _kerberos._tcp.dc._msdcs.contoso.com -Type SRV
Resolve-DnsName gc._msdcs.contoso.com -Type SRV
Resolve-DnsName contoso.com -Type SOA
Resolve-DnsName contoso.com -Type NS
Resolve-DnsName server01.contoso.com -Type A
Resolve-DnsName 1.2.168.192.in-addr.arpa -Type PTR   # Reverse lookup

# 5. Check DNS server zones
Get-DnsServerZone -ComputerName DC01
Get-DnsServerResourceRecord -ZoneName "contoso.com" -ComputerName DC01

# 6. Check DNS forwarding
Get-DnsServerForwarder -ComputerName DC01

# 7. Check DNS server logs
Get-EventLog -LogName "DNS Server" -Newest 50
Get-WinEvent -FilterHashtable @{LogName='DNS Server'; Level=1,2}

# 8. Check DNS client logs
Get-EventLog -LogName "DNS Client" -Newest 50

# 9. Test DNS resolution performance
Resolve-DnsName -Name www.google.com -Server 8.8.8.8
Measure-Command { Resolve-DnsName www.google.com }

# 10. Check DNS debug logging (on DNS server)
Set-DnsServerDiagnostics -All $true    # Enable debug logging
# Logs at: C:\Windows\System32\dns\Dns.log

# 11. Network-level DNS troubleshooting
tcpdump / Wireshark filter: udp port 53
Check firewall rules: Get-NetFirewallRule | Where-Object {$_.DisplayName -like "*DNS*"}

# 12. Check DNS integrity on DCs (AD-specific)
dcdiag /test:dns /v
dnslint /d contoso.com                # Microsoft DNS lint tool

# 13. Check DNS Server statistics
Get-DnsServerStatistics -ComputerName DC01

# 14. Check for DNS record duplicates
Get-DnsServerResourceRecord -ZoneName "contoso.com" | Group-Object -Property RecordType, Name | Where-Object {$_.Count -gt 1}

# 15. Check scavenging status
Get-DnsServerZone -ComputerName DC01 | Select-Object ZoneName, ScavengeServers, IsAutoScavenging
```

### Key DNS Logs and Event IDs:

| Log | Event ID | Meaning |
|-----|----------|---------|
| **DNS Server** | 4000 | DNS server failed to start |
| **DNS Server** | 4001 | DNS server started |
| **DNS Server** | 4002 | DNS server stopped |
| **DNS Server** | 4004 | Zone load failure |
| **DNS Server** | 4005 | Zone unload |
| **DNS Server** | 4006 | Zone is not loading (permissions, corrupt) |
| **DNS Server** | 4007 | Network connection failed (TCP/DNS connectivity) |
| **DNS Server** | 4008 | TCP listen failure |
| **DNS Server** | 4013 | Dynamic update failure |
| **DNS Server** | 4015 | Authentication failure |
| **DNS Server** | 5501, 5502 | DNSSEC validation failure |
| **DNS Client** | 1, 5000+ | Various client events |

## 4.8 Step-by-Step DNS Troubleshooting Methodology

```
SCENARIO: Users report "No internet access" and "can't open websites". Also, domain logins are slow.

STEP 1: VERIFY CLIENT DNS CONFIGURATION
  ├── ipconfig /all → Are DNS servers configured? What are they?
  ├── Check DHCP server options → Option 006 (DNS servers), Option 015 (domain name)
  ├── Check for manually configured DNS (static) on client
  └── Confirm: Client's DNS server is reachable?

STEP 2: TEST DNS RESOLUTION ON CLIENT
  ├── nslookup google.com → Does it resolve?
  ├── nslookup server01.contoso.com → Does it resolve?
  ├── nslookup contoso.com → Does SOA/NS resolve?
  ├── If no → DNS server not responding or not configured
  ├── If partial → DNS server returning wrong/incomplete data
  └── Check: ping 8.8.8.8 (bypasses DNS — if internet works, it's DNS issue)

STEP 3: CHECK DNS SERVER
  ├── Ping DNS server (DC01) — is it reachable?
  ├── Test-NetConnection DC01 -Port 53 — is port 53 open?
  ├── Check DNS service is running: Get-Service DNS
  ├── If DNS service stopped → Start it: Start-Service DNS
  ├── Check DNS Server log for errors (event 4000, 4004, etc.)
  └── RDP to DNS server if possible → check DNS MMC console

STEP 4: CHECK DNS ZONES
  ├── Get-DnsServerZone — are zones loaded?
  ├── For AD-integrated zones: Check replication status (repadmin /replsummary)
  ├── For file-backed zones: Check zone file exists and permissions
  ├── Event 4004, 4006 → Zone load failure (corrupt? permissions?)
  └── Restart DNS server after zone repair: Restart-Service DNS -Force

STEP 5: CHECK FORWARDERS
  ├── Get-DnsServerForwarder → What forwarders are configured?
  ├── Can forwarders be reached? (ping, Test-NetConnection port 53)
  ├── If forwarders down → Internet/external DNS fails
  ├── If forwarders respond slowly → Use root hints as alternative or fix network
  └── Consider: DNS policies, DoH, DNS firewalls (Cisco Umbrella, etc.)

STEP 6: CHECK AD-INTEGRATED DNS SPECIFIC
  ├── On DC: Is DNS zone replicating?
  ├── DNS zones in AD: check with ntdsutil or dsmgmt
  ├── Event ID 4006 (zone not loading) on DC → Check DSRM, AD DS health
  ├── Is DNS server also a DC? Check DNS DS integration
  └── dcdiag /test:dns /v — test all DNS records

STEP 7: CHECK DNS CLIENT CACHE
  ├── ipconfig /displaydns → Check for stale/incorrect entries
  ├── ipconfig /flushdns → Clear cache and re-resolve
  ├── Check for DNS cache poisoning (unlikely but possible)
  └── Windows: Check hosts file (C:\Windows\System32\drivers\etc\hosts)

STEP 8: CHECK NETWORK PATH
  ├── Firewall rules blocking UDP/TCP 53?
  ├── Network ACLs between client and DNS server?
  ├── Split-brain: Wrong DNS server being used (internal vs. external)
  └── Check: nslookup server01.contoso.com 192.168.1.10 (force specific DNS server)

STEP 9: REMEDIATE
  ├── Start DNS service if stopped
  ├── Reload/recreate zone if corrupt
  ├── Fix forwarders
  ├── Fix DHCP options
  ├── Clear client DNS cache
  ├── Fix firewall rules
  ├── If root cause is dead server → failover to secondary DNS server
  └── If DNS replication broken → repadmin /replall /A

STEP 10: VERIFY & DOCUMENT
  ├── All clients can resolve names
  ├── AD logins fast (no more slow DC location)
  ├── Internet access restored
  ├── Document root cause and fix
  └── Add monitoring for DNS server health, zone load, response time
```

## 4.9 How Does DNS Affect Other Infrastructure Components?

```
If DNS goes completely down:
├── Internet Access → All URLs fail (no resolution)
├── Email → MX records fail; email sending/receiving breaks
├── Active Directory → DC location fails (SRV records); Kerberos fails; LDAP fails
├── Authentication → No DC found; "No network path to domain" errors
├── Web Applications → All web apps fail (URL can't resolve)
├── File Shares → UNC path resolution fails (though netbios may still work in some cases)
├── Powershell Remoting → Server name resolution fails
├── VPN → VPN server name can't resolve; VPN connection fails
├── Microsoft 365 → Hybrid sync issues; Azure AD Connect can't reach endpoints
├── Monitoring → All monitoring relying on hostnames fails (false alerts)
├── Print Services → Printer discovery may fail
├── DHCP → DHCP relay may fail (IP helper relies on DNS sometimes)
├── SCCM/MECM → Distribution point names can't resolve
├── NTP/Time → Some NTP uses names; time sync may fail
├── Secure Channels → Computer domain trusts may break (using FQDNs)
└── EVERYTHING else → Anything connecting by hostname instead of IP fails
```

## 4.10 Key DNS Differences

| Feature | Authoritative DNS | Recursive DNS |
|---------|-------------------|---------------|
| **Role** | Holds definitive zone data | Resolves queries on behalf of clients |
| **Answer type** | Final answer OR referral (NS records) | Cached answer OR referral |
| **Zone type** | Primary/Secondary/Stub | Not zone-specific |
| **DNS recursion** | No (by default) | Yes (if enabled) |
| **Port** | 53 (TCP/UDP) | 53 (TCP/UDP) |
| **Example in AD** | DC01 serving contoso.com zone | DC01 answering client queries recursively |

## 4.11 L3-Level Interview Answer: DNS Outage Scenario

**Scenario: "After a DNS server reboot for patching, internal users report they can't reach internal apps, but external websites work. Domain logins are failing on one floor."**

```
1. TRIAGE (2 min):
   - "External websites work" → Internet DNS (forwarders/cloud DNS) is working
   - "Internal apps can't reach" → Internal DNS is broken
   - "Domain logins failing on ONE FLOOR" → Not all DNS servers down; floor-specific issue

2. IMMEDIATE CHECKS:
   ├── Which DNS server is the affected floor pointing to?
   ├── Check DHCP scope for that floor: Option 006 (DNS server list)
   ├── Did the DHCP server get patched? Are leases still valid?
   ├── nslookup server01.contoso.com from affected floor (which DNS responds?)
   └── Compare with floor that works: nslookup server01.contoso.com (different DNS?)

3. IDENTIFY DNS SERVER IN USE BY AFFECTED CLIENTS:
   ├── ipconfig /all on affected client → see DNS server IP
   ├── Is that DNS server (e.g., DC03 on that floor) reachable?
   ├── ping DC03 → if no → network/PowerLoss/Hyper-V issue
   ├── Test-NetConnection DC03 -Port 53 → if no → firewall/service down
   └── If ping works but port 53 doesn't → DNS service may have crashed

4. CHECK DNS SERVER (DC03):
   ├── Can you access it? RDP/console?
   ├── Get-Service DNS → is DNS Server service running?
   ├── If stopped → Start-Service DNS
   ├── Get-DnsServerZone → are zones loaded?
   ├── If zone failed (Event 4004/4006):
       │   Check zone file permissions
       │   Check zone corruption: dnscmd /zonereset <zone> /file <file>
       │   For AD-integrated: Check replication (repadmin /replsummary)
       └── If zone OK: check forwarders

5. CHECK FOR DHCP ISSUE:
   ├── Are there enough IP addresses in DHCP scope for that floor?
   ├── Did DHCP server get patched/rebooted?
   ├── Check DHCP lease times: are leases expired?
   ├── Is DHCP service running? Get-Service DHCPServer
   └── DHCP fails to update DNS records? Check: "DNS-DHCPR Audit Log" event

6. REMEDIATE:
   ├── If DNS service down → Start-Service DNS
   ├── If zone corrupt → Reload zone or restore from backup
   ├── If DHCP lease issue → Expand scope / extend lease time
   ├── If DHCP not giving correct DNS → Fix Option 006 in DHCP server
   ├── If firewall blocking port 53 → Check Windows Firewall / network ACL
   └── If DNS service crashed on specific server → Investigate why (memory leak? corrupt zone?)

7. VERIFY:
   ├── ipconfig /flushdns on affected clients
   ├── nslookup server01.contoso.com returns correct IP from floor DNS server
   ├── Users can reach internal apps
   ├── Check AD: dcdiag /s DC03
   ├── No Event ID 4006 (zone load failure)
   └── User logins successful on affected floor

8. ROOT CAUSE ANALYSIS:
   ├── Why did DNS service crash? (Memory? Corrupt zone? Bug?)
   ├── Why is only one floor affected? (DHCP option 006 pointing to wrong DNS?)
   ├── Why did it happen after patching? (Update broke DNS service? DHCP lease expiry?)
   └── What if DNS service crashed again? Consider reinstalling DNS role or rebuilding server

9. PREVENTION:
   ├── DNS server should NOT be the only DNS for a subnet (min 2 DNS servers per site)
   ├── DHCP Option 006 should point to ≥2 DNS servers (primary + secondary)
   ├── DNS scavenging configured properly to avoid stale records
   ├── Monitor: DNS service status, zone load success, query response time
   ├── DNS server memory/CPU monitoring (alert on high usage)
   ├── Test DNS failover: what if one DC/DNS goes down?
   └── After patching: Always verify DNS service is running and zones are loaded
```

---

# 🔷 SECTION 5 — CROSS-CUTTING DEPENDENCY MATRIX

## The Complete Dependency Map

```
                         ┌─────────────┐
                         │   USERS     │
                         └──────┬──────┘
                                │ (Logon)
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      ACTIVE DIRECTORY (AD DS)                       │
│  Authentication │ Authorization │ GPO │ Kerberos │ LDAP │ Replication │
│  All DCs replicate via multi-master KCC-managed topology           │
│  DNS is required for DC location (SRV records)                      │
└───┬──────────┬──────────┬──────────┬──────────────┬─────────────────┘
    │          │          │          │              │
    ▼          ▼          ▼          ▼              ▼
┌───────┐ ┌───────┐ ┌────────┐ ┌────────┐ ┌───────────────────┐
│DNS    │ │GPO    │ │AD CS   │ │AD FS   │ │ Other Apps/Infra  │
│       │ │       │ │(PKI)  │ │(SLO)  │ │                   │
│SRV    │ │Registry│ │Certs  │ │Trusts │ │ IIS, SQL, File,   │
│Records│ │Policies│ │Enroll │ │SAML   │ │ Print, RDS, etc.  │
│A/Zones│ │Software│ │OCSP   │ │OAuth  │ │                   │
│Reverse│ │Scripts │ │CRL    │ │SAML   │ │                   │
└───┬───┘ └───┬───┘ └───┬────┘ └───┬────┘ └────────┬──────────┘
    │         │          │           │                │
    │         │          │           │                ▼
    ▼         │          │           │        ┌───────────────┐
┌────────┐   │    ┌─────┴──────┐   │        │ WINDOWS SERVER  │
│DNS INTE│   │    │ SYSVOL/    │   │        │ (OS, Hyper-V,  │
│ GRATED │   │    │ DFT/DFS    │   │        │  Services)     │
│ ZONES  │   │    │ Replication│   │        └────────────────┘
│ (AD)   │   │    │ (GPO files)│   │
└───┬────┘   │    └────────────┘   │
    │        │                     │
    ▼        │                     │
DHCP Option  │    ALL OF THE ABOVE REQUIRE:
006/015      │    ┌─────────────────┐
    │        │    │  NETWORK        │
    ▼        │    │  STORAGE        │
DHCP Leases  │    │  COMPUTE        │
    │        │    │  (CPU/RAM)      │
Clients      │    └─────────────────┘
get IPs      │
             │
             ▼
    ┌────────────────┐
    │ TIME SERVICE   │
    │ (w32time)      │
    │ (Critical:     │
    │  5-min skew    │
    │  for Kerberos) │
    └────────────────┘
```

## The #1 Rule for L3: "Check DNS First"

```
┌───────────────────────────────────────────────────────────────────┐
│  "When in doubt, check DNS."                                     │
│                                                                   │
│  AD needs DNS for DC location                                    │
│  Kerberos needs DNS for KDC location                             │
│  LDAP needs DNS for server location                              │
│  GPO needs DNS for SYSVOL location                               │
│  DHCP needs DNS for FQDN of domain                               │
│  Everything needs DNS for service discovery                      │
│  DNS failure cascades to EVERYTHING                              │
│                                                                   │
│  #1 Diagnostic Step:                                            │
│  nslookup <anything> → If it fails, FIX DNS FIRST.               │
└───────────────────────────────────────────────────────────────────┘
```

---

# 🔷 SECTION 6 — BONUS: HYPER-V / VIRTUALIZATION

Since this is a "Windows & Virtualization Administrator" role, this section is essential.

## 6.1 What Is Hyper-V?

Hyper-V is Microsoft's **Type-1 (bare-metal) hypervisor** that enables running multiple virtual machines on a single physical server. It allows server consolidation, workload isolation, live migration, high availability, and disaster recovery.

## 6.2 Why Is It Used?

| Reason | Explanation |
|--------|-------------|
| **Server Consolidation** | Run 10-50 VMs on one physical server |
| **Isolation** | Each VM is isolated; one crash doesn't affect others |
| **High Availability** | Failover Clustering + VMs can auto-migrate |
| **Disaster Recovery** | Hyper-V Replica (asynchronous VM replication) |
| **Live Migration** | Move running VMs between hosts with zero downtime |
| **Rapid Deployment** | Clone/Template VMs for fast provisioning |
| **Snapshot/Checkpoint** | Save VM state for testing/backup |
| **Resource Allocation** | Dynamic memory, virtual processors, VHDX |
| **Cloud Integration** | Azure Migrate, Hyper-V Replica to Azure |

## 6.3 How Does It Work? (Architecture)

```
┌──────────────────────────────────────────────────────────────────┐
│                    HYPER-V HYPERVISOR                            │
│                    (Type-1 / Bare-Metal)                         │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                 │
│  │   VM 1     │  │   VM 2     │  │   VM 3     │                │
│  │ (Guest OS) │  │ (Guest OS) │  │ (Guest OS) │                │
│  │ vCPU       │  │ vCPU       │  │ vCPU       │                 │
│  │ vRAM       │  │ vRAM       │  │ vRAM       │                 │
│  │ vNIC       │  │ vNIC       │  │ vNIC       │                 │
│  │ vDisk      │  │ vDisk      │  │ vDisk      │                 │
│  └────────────┘  └────────────┘  └────────────┘                 │
│                                                                  │
│  ┌──────────────────────────────────────────────────────┐        │
│  │            VMBUS (Virtualization Bus)                  │       │
│  │   Enlightened devices communicate via VMBus           │       │
│  └──────────────────────────────────────────────────────┘        │
│                                                                  │
│  ┌──────────────────────────────────────────────────────┐        │
│  │         VMMS (Virtual Machine Management Service)     │       │
│  │         VMWP (Virtual Machine Worker Process)         │       │
│  └──────────────────────────────────────────────────────┘        │
│                                                                  │
│  ┌──────────────────────────────────────────────────────┐        │
│  │     PHYSICAL HARDWARE (CPU/Mem/Storage/Network)      │       │
│  └──────────────────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────────────────┘
```

### Key Components:

1. **vmwp.exe (VM Worker Process)** — One per running VM; owns VM's virtual devices; executes emulated devices
2. **vmms.exe (Virtual Machine Management Service)** — Core Hyper-V service; manages VM lifecycle
3. **Hypervisor** — Sits below OS; manages hardware partitioning; VMs run at ring -1 (below ring 0)
4. **VMBus** — High-performance communication channel between parent partition (host OS) and child partitions (VMs)
5. **VMIC (VM Interface Driver)** — Enlightened driver for network/storage in VMs
6. **Virtual Switch (vSwitch)** — Connects VMs to physical NICs; types: External, Internal, Private
7. **VM Configuration** — XML-based; VM settings, CPU, RAM, devices
8. **VHDX** — Virtual Hard Disk format (up to 64 TB); resilient, supports 4KB sectors
9. **VHD** — Older format (up to 2 TB); still supported
10. **VMMS Database** — `C:\ProgramData\Microsoft\Windows\Hyper-V\Virtual Machines\`
11. **Snapshots/Checkpoints** — VM state saved to AVHDX (differencing disk)
12. **VM Guest Service Interface** — Allows Hyper-V Integration Services in VMs
13. **Host Guardian Service (HGS)** — For shielded VMs (BitLocker, TPM attestation)
14. **PowerShell Direct** — Manage VMs from inside host via VM management OS
15. **Live Migration** — Moves running VMs between nodes via TCP or SMB direct

## 6.4 Components Involved

1. **Hyper-V Role** — Installed on Windows Server
2. **VMMS Service** — VM Management Service (vmms.exe)
3. **VM Worker Process** — vmwp.exe (per VM)
4. **Virtual Switch** — External, Internal, Private
5. **vNIC (Virtual NIC)** — Network adapter for VM
6. **Virtual Disk (VHDX)** — Virtual hard drive
7. **Virtual Machine Configuration File** — XML (e.g., .vmcx in Server 2016+)
8. **Integration Services** — Enhanced session, time sync, heartbeat, VSS
9. **Live Migration** — TCP or SMB-based
10. **Quick Migration** — Short downtime migration (memory pre-copy)
11. **Hyper-V Replica** — Asynchronous VM replication (15-min intervals default)
12. **Checkpoints** — VM state snapshots
13. **Virtual Machine Manager (VMM / SCVMM)** — Central management (if used)
14. **Failover Clustering** — HA for VMs via Hyper-V cluster
15. **Storage Spaces Direct (S2D)** — Software-defined storage for Hyper-V clusters
16. **SMB 3.0 Multi-Channel** — For live migration and CSV (Cluster Shared Volumes)
17. **PowerShell Hyper-V Module** — All management via PowerShell
18. **WMI/VMI** — Windows Management Instrumentation for Hyper-V
19. **Shielded VM** — Encrypted VMs with BitLocker and TPM attestation
20. **GPU-P / Discrete Device Assignment** — GPU passthrough to VMs

## 6.5 What Depends on Hyper-V?

```
Hyper-V (Hypervisor + VM Lifecycle Management)
│
├── Virtual Machines (all workloads running as VMs)
│   ├── Windows Server VMs (DCs, File Servers, SQL, IIS, etc.)
│   ├── Windows 10/11 VMs (workstation VMs)
│   ├── Linux VMs (if running on Hyper-V)
│   └── Other OS VMs (via generation 2 / UEFI)
├── Failover Cluster (VM HA)
│   ├── Live Migration
│   ├── VM Restart Priority
│   ├── VM Initialization
│   └── Cluster Shared Volumes (CSV)
├── Hyper-V Replica (DR)
├── Virtual Networking (vSwitch, vNIC, VLAN, SDN)
├── Virtual Storage (VHDX, CSV, SMB 3.0 Storage)
├── PowerShell Hyper-V Module (management)
├── System Center Virtual Machine Manager (SCVMM)
├── Azure Stack HCI / Azure Arc (if applicable)
├── Container Hosting (Windows Containers on Hyper-V isolation)
├── Shielded VM (HGS + BitLocker)
├── Production Checkpoints (VSS integration)
├── Storage Spaces Direct (S2D)
├── Network Controller (SDN)
├── Nested Virtualization (VM running Hyper-V inside VM)
└── GPU-P (GPU Passthrough for VMs)
```

## 6.6 What Happens If Hyper-V Fails?

| Failure Type | Impact |
|-------------|--------|
| **Hyper-V role fails** | All VMs on that host become inaccessible |
| **VMMS service crashes** | VM management stops; VMs may continue running but can't be managed |
| **VMWP crashes** | Individual VM crashes; guest OS loses power; may auto-restart |
| **vmwp.exe process crashes** — common when running out of VM worker process quota (16384 max) |
| **Hyper-V cluster failure** — node down → VMs fail over to other nodes (if configured) |
| **Live Migration failure** — VM goes to maintenance mode, or migrating VMs during network/storage issues |
| **Storage failure** | VM disk unreachable; VM can't boot or crashes; data loss if no redundancy |
| **Virtual switch failure** | VMs lose network connectivity |
| **vNIC driver failure** | VM loses network; "Network cable unplugged" state |
| **VHDX corruption** | VM data inaccessible; requires restore from backup/checkpoint |
| **Checkpoint/AVHDX chain corruption** | VM can't revert; chain broken |
| **Hyper-V Replica failure** — DR site not in sync |
| **Snapshots consuming excessive disk space** | VM performance degrades; datastore fills up |
| **Nested virtualization issues** — L1 hypervisor inside VM L2 can conflict |

## 6.7 How to Identify Hyper-V Failure

```powershell
# 1. Check Hyper-V role status
Get-WindowsFeature Hyper-V
Get-Service vmms
Get-Service vmcompute   # Hyper-V compute service (Windows 10/11)

# 2. List all VMs and their states
Get-VM
Get-VM | Select-Object Name, State, CPUUsage, MemoryAssigned, Uptime
Get-VM | Where-Object {$_.State -ne 'Running'}

# 3. Check VM worker process
Get-Process vmwp | Select-Object Id, ProcessName, WorkingSet64
# Or: Get-VM | Select-Object Name, VMId | ForEach-Object { Get-Process vmwp | Where-Object { $_.CommandLine -match $_.VMId } }

# 4. Check Hyper-V events
Get-WinEvent -LogName "Microsoft-Windows-Hyper-V-VMMS-Admin" -MaxEvents 20
Get-WinEvent -LogName "Microsoft-Windows-Hyper-V-Worker" -MaxEvents 20
Get-WinEvent -LogName "Microsoft-Windows-Hyper-V-VMMS-Operational" -MaxEvents 20

# 5. Check VMMS service health
Get-Service vmms | Select-Object Name, Status, StartType
# If not running:
Start-Service vmms

# 6. Check virtual switches
Get-VMSwitch
Get-VMSwitch | Select-Object Name, Type, NetAdapterInterfaceDescription

# 7. Check VM network
Get-VMNetworkAdapter -VMName <VMName>
Get-VMNetworkAdapter -VMName <VMName> | Set-VMNetworkAdapter -PortMapping ...

# 8. Check virtual disks
Get-VHD -Path "C:\VMs\VM01\VM01.vhdx"
Get-VHD | Where-Object {$_.FileState -ne 'OK'}

# 9. Check replication health
Get-VMReplication
Get-VMReplication | Where-Object {$_.ReplicationHealth -ne 'Health'}
Get-VMReplicationReport -VMName <VMName>

# 10. Check live migration
Get-VMMigrationNetwork
Get-VM | Where-Object {$_.MigrationState -eq 'Migrating'}

# 11. Check Hyper-V cluster (if clustered)
Get-Cluster
Get-ClusterNode
Get-ClusterGroup | Where-Object {$_.State -ne 'Online'}
Get-ClusterResource | Where-Object {$_.State -ne 'Online'}

# 12. Check VM configuration
Get-VM -VMName <VMName> | Select-Object *
Get-VMConfiguration -VMName <VMName>

# 13. Check checkpoints
Get-VMCheckpoint -VMName <VMName>
Get-VMCheckpoint -VMName <VMName> | Measure-Object -Property Size -Sum

# 14. Performance counters
Get-Counter '\Hyper-V Hypervisor Physical(*)\% Total Physical CPU'
Get-Counter '\Hyper-V Dynamic Memory Balancer(*)\Physical Memory (MB)'
Get-Counter '\Hyper-V VM (\VMName)\Guest Visible (Highest) CPU'

# 15. VM settings check
Get-VMKeyProtection -VMName <VMName>
Get-VMFirmware -VMName <VMName>
Get-VMProcessor -VMName <VMName>
Get-VMMemory -VMName <VMName>

# 16. Hyper-V logs
C:\ProgramData\Microsoft\Windows\Hyper-V\Logs\
C:\Windows\Temp\Hyper-V live migration logs\
```

## 6.8 L3-Level Interview Answer: Hyper-V Production Outage

**Scenario: "A Hyper-V host (HVHost01) just rebooted unexpectedly (BSOD on host). You have 15 VMs on it, 3 are domain controllers. What do you do?"**

```
1. IMMEDIATE (First 5 minutes):
   ├── Confirm host BSOD: check monitoring/IPMI/iLO
   ├── Check if Hyper-V Cluster: did VMs auto-failover?
   │   └── YES (Cluster): Check cluster: Get-ClusterGroup | Where-Object State -ne Online
   │   └── NO (Standalone): VMs are OFF. Plan recovery.
   ├── Identify DCs (3 of them) → These affect AD/DNS/GPO
   ├── Identify which DCs hold FSMO roles
   └── Start communication to stakeholders

2. HOST RECOVERY:
   ├── Diagnose BSOD: Check minidump (C:\Windows\Minidump\)
   ├── WinDbg analysis: !analyze -v
   ├── Check: Was it a storage driver issue? Memory? Update?
   ├── Fix root cause before bringing host back
   └── Common: NIC driver, storage firmware, Windows Update

3. IF CLUSTERED (Failover occurred):
   ├── VMs now running on other hosts (or VMHA restarted)
   ├── Verify: Are all 15 VMs running on secondary hosts?
   ├── Verify: Are DCs (3) running? Check AD health:
   │   └── dcdiag /e
   │   └── repadmin /replsummary
   │   └── Check if FSMO roles moved with VM (live migration preserves FSMO)
   ├── Verify: Are VMs with FSMO roles still able to service requests?
   └── Monitor VMs on secondary hosts (resource capacity)

4. IF STANDALONE (VMs stopped):
   ├── Restart DCs first (to restore AD/DNS/GPO)
   ├── Start VM workers (or use Start-VM -Name * for all VMs)
   ├── Monitor boot order of VMs
   └── Check: VM checkpoints? Use them if previous state needed

5. PREVENT RECURRENCE:
   ├── Enable Hyper-V Cluster with VMHA (if not already)
   ├── Configure Live Migration
   ├── Configure Hyper-V Replica for DR
   ├── Install host monitoring (SCOM, Nagios, Zabbix)
   ├── Schedule maintenance windows for host patching (rolling)
   └── Add root cause to knowledge base
```

---

# 🔷 SECTION 7 — COMPLETE INTERVIEW SCENARIO BANK

## Scenario 1: "Users in Building 3 can't log into the domain."

```
1. DNS Check (FIRST): Can clients in Building 3 resolve DC names?
   └── If no → Check DHCP Option 006 (DNS servers) for Building 3 scope
   └── If yes → Check AD replication, DC status

2. DC Check: Are all DCs up? Specifically, is there a DC in Building 3?
   └── No DC in Building 3 → Users relying on cached creds only (if any)
   └── DC in Building 3 is down → Why? Check service, disk, replication

3. Replication: repadmin /replsummary — Is DC in Building 3 replicating?

4. Kerberos: w32tm /query /status — Time skew? (5-min tolerance)

5. Secure Channel: Test-ComputerSecureChannel on client machines

6. Event Logs: Check Directory Service, DNS, System logs on DC in Building 3

7. Remediate: Start services, fix DNS, force replication, reset secure channel
```

## Scenario 2: "GPO changes are not applying to users in Sales OU."

```
1. GPResult: gpresult /h report.html on affected user's computer
2. Check GPO link in Sales OU (is it linked? Enabled? Enforced?)
3. Check Security Filtering (Authenticated Users has Read+Apply?)
4. Check WMI filter (if any — does it evaluate TRUE?)
5. Check if Sales OU has Block Inheritance
6. Check SYSVOL DFSR health (GPT files accessible?)
7. Check for another GPO in parent OU with Enforced that overrides
8. Check Event Viewer Application log (1058/1030?)
9. Force: gpupdate /force and re-check
10. If issue persists: Restore GPO from backup
```

## Scenario 3: "DNS resolution is slow (5-10 seconds per request)."

```
1. Compare: nslookup from fast vs. slow DNS servers
2. Check: Are forwarders configured? Test without forwarders (root hints)
3. Check: DNS server overloaded? Get-DnsServerStatistics (queries/sec)
4. Check: Too many DNS zones? Performance impact?
5. Check: Reverse lookup zones missing → timeouts on PTR lookups
6. Check: DNS client cache poisoning or corrupt cache
7. Check: Network issues (latency, firewall inspection of port 53)
8. Check: DNSSEC validation slow? (Disable DNSSEC for test)
9. Remediate: Add forwarders, clear cache, add more DNS servers, optimize zones
```

## Scenario 4: "After a Windows Update reboot, DC02 is no longer responding to LDAP queries."

```
1. Ping DC02 → Reachable?
2. Test-NetConnection DC02 -Port 389 (LDAP) → Open?
3. Check AD DS service: Get-Service NTDS
4. Check DNS (LDAP needs DNS for other DC location): Resolve-DnsName _ldap._tcp.dc._msdcs.domain
5. Check Event Logs on DC02: Directory Service log
6. dcdiag /v /s:DC02
7. Check replication: repadmin /showrepl DC02
8. Check if NTDS.DIT is corrupt (event ID 1644, 1987)
9. If corrupt: Dsmgmt → Activate "Non-Authoritative Restore" → Restore System State backup
10. If service won't start: Check dependencies (DNS, Netlogon, MSADCS, SRS)
11. Reboot → Check again → Seize FSMO roles if DC02 is permanently offline
```

---

# 🔷 SECTION 8 — QUICK REFERENCE: EMERGENCY COMMAND CHEAT SHEET

## "I don't know what's broken" — Universal Diagnostic Flow

```powershell
# EVERYthing starts with these:
systeminfo
Get-EventLog -LogName System -Newest 50
Get-Service | Where-Object {$_.Status -ne 'Running'}
Test-Connection -ComputerName localhost -Count 4
Get-NetIPConfiguration

# Name resolution:
Resolve-DnsName localhost
nslookup localhost

# Time:
w32tm /query /status
w32tm /query /peers

# AD (if DC):
dcdiag /v
repadmin /replsummary
netdom query fsmo

# GPO:
gpresult /r
gpupdate /force

# DNS:
Get-DnsServerZone
Resolve-DnsName <name>

# Hyper-V (if host):
Get-VM
Get-Service vmms

# Everything (unified):
Get-WinEvent -FilterHashtable @{LogName='System'; Level=1,2} -MaxEvents 100
```

---

# Summary Table: Complete Technology Map

| Technology | Purpose | Key Ports | Key Dependency | Key Failure Impact |
|-----------|---------|-----------|----------------|-------------------|
| **Windows Server** | OS for all services | Various (per role) | Hardware, Networking, Hyper-V | All roles on it go down |
| **AD DS** | Auth, Directory, GPO | 389 (LDAP), 636, 88 (Kerberos), 445, 53 (DNS) | DNS, Time, DNS-integrated zones | No auth, no GPO, no service location |
| **Group Policy** | Configuration, Security | 445 (SYSVOL), 135 (RPC), 139, 445 | AD DS, SYSVOL/DFSR, DNS | Inconsistent security, no software deploy, no drive maps |
| **DNS** | Name Resolution | 53 (TCP/UDP) | Active Directory (SRV records), AD-integrated zones | Total infrastructure failure (everything breaks) |
| **Hyper-V** | Virtualization | 21794 (VMIC), 49152-65535 (Live Migration) | Hardware (VT-x/AMD-V), Storage, Network, Failover Cluster | All VMs on host go down |
| **Time Service (w32time)** | Time Synchronization | 123 (UDP NTP) | All (Kerberos needs 5-min skew) | Kerberos fails, AD breaks |

---

This document covers the complete enterprise infrastructure architecture as requested. When you share the actual JD, I can tailor every section specifically to the technologies and responsibilities mentioned there. Would you like me to go deeper into any specific technology, build out more scenario-based practice questions, or create a hands-on lab exercise for any of these areas?