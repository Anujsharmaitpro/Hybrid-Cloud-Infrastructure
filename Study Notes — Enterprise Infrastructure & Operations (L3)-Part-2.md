Continuing from Part 19 — here are Parts 19–36 to complete the study notes.

---

## PART 19 — Event Viewer

> *Master these logs and know exactly what each Event ID/area indicates.*


###Master Log Locations

### Windows Event Logs
| Log | Purpose | Key Events |
|-----|---------|-----------|
| **Application** | Application/software errors | App crashes, service failures, EXE errors |
| **System** | OS/driver/service events | Driver issues, service failures, boot, kernel |
| **Security** | Audit trail (login, access) | Logon, logoff, privilege use, object access |
| **Setup** | Windows Setup/upgrade events | Feature updates, role installations |
| **Forwarded Events** | Events collected via WinRM | Centralized logging from multiple servers |

### Access
```powershell
# Open Event Viewer
eventvwr.msc
Get-WinEvent -LogName Application -MaxEvents 10
Get-WinEvent -LogName System -MaxEvents 10
Get-WinEvent -LogName Security -MaxEvents 10

# Filter by level (0=Critical, 1=Error, 2=Warning, 3=Information, 4=Verbose)
Get-WinEvent -LogName System | Where-Object {$_.Level -le 2}

# Filter by Event ID
Get-WinEvent -FilterHashtable @{LogName='System'; Id=4625}

# Custom query (XML)
Get-WinEvent -LogName Application -FilterXml '<EventList><Event ID="1000" /></EventList>'
```

### Key Properties in Every Event
| Property | Description |
|----------|-------------|
| **Event ID** | Unique identifier for the event type |
| **Source** | Component that generated the event (e.g., Service Control Manager, Microsoft-Windows-Kernel-General) |
| **Level** | Critical/Error/Warning/Information/Verbose |
| **Timestamp** | When the event occurred (UTC or local) |
| **Correlation** | Links related events (ActivityID, RelatedActivityID) |
| **Error code** | Numeric/hex error code |
| **Task Category** | Subcategory within source |
| **Opcode** | Operation being performed (Info, Start, Stop, etc.) |




###Critical Event IDs by Area

### Service Failure
| Event ID | Source | Meaning |
|----------|--------|---------|
| **7000** | Service Control Manager | Service failed to start |
| **7001** | Service Control Manager | Service depends on failed service |
| **7009** | Service Control Manager | Service started then stopped |
| **7023** | Service Control Manager | Service terminated with error |
| **7031** | Service Control Manager | Service terminated unexpectedly (Restart attempt) |
| **7034** | Service Control Manager | Service terminated unexpectedly (no restart) |
| **10000/10001** | DistributedCOM | DCOM application error |

### Driver Issues
| Event ID | Source | Meaning |
|----------|--------|---------|
| **1000** | Application/Windows Error Reporting | Application crash (driver .sys may be involved) |
| **41** | Kernel-General | System rebooted without clean shutdown |
| **1521** | Kernel-General | Boot failure (driver issue) |
| **Driver IRQL error** | Various | Driver caused BSOD (bugcheck code varies) |

### Authentication Events (Security Log)
| Event ID | Meaning |
|----------|---------|
| **4624** | Successful logon (type 2=interactive, 3=network, 7=unlock, 10=RDP, 11=cached) |
| **4625** | Failed logon (check sub-status: 0xC0000133=time skew, 0xC000006A=bad password) |
| **4647** | User initiated logoff |
| **4672** | Special privileges assigned (admin login) |
| **4688** | Process creation (enable auditing for process tracking) |
| **4778** | Session reconnected (RDP) |
| **4779** | Session disconnected (RDP) |
| **4800** | Workstation locked |
| **4801** | Workstation unlocked |
| **4768** | Kerberos TGT request (AS-REQ) |
| **4769** | Kerberos service ticket (TGS-REQ) |
| **4771** | Kerberos pre-authentication failed |

### Kerberos-Specific
| Event ID | Meaning |
|----------|---------|
| **4768** | TGT request (success: status 0x0; failure: status code) |
| **4769** | Service ticket request (success/failure) |
| **4771** | Pre-authentication failed (clock skew, wrong password) |
| **4766** | Service ticket retrieved (success) |

### DNS Events
| Event ID | Source | Meaning |
|----------|--------|---------|
| **Event 4013** | DNS Server | DNS server started |
| **Event 4014** | DNS Server | DNS server stopped |
| **Event 65000** | DNS Server | Query failure |

### Disk / NTFS Events
| Event ID | Source | Meaning |
|----------|--------|---------|
| **51** | Kernel-General | I/O device error (disk problem) |
| **55** | Kernel-General | File system corruption detected |
| **57** | Kernel-General | Disk I/O error (sector read/write failure) |
| **1102** | Security | Audit log cleared by admin |
| **50** | Kernel-General | Pagefile creation/resizing |

### Reboot Events
| Event ID | Source | Meaning |
|----------|--------|---------|
| **6005** | EventLog | Event log service started (boot) |
| **6006** | EventLog | Event log service stopped (shutdown) |
| **6008** | EventLog | Unexpected shutdown |
| **41** | Kernel-General | Reboot without clean shutdown |
| **1074** | User32 | Planned shutdown by user/process |
| **1076** | User32 | Planned reboot by user/process |

### Cluster Events
| Event ID | Source | Meaning |
|----------|--------|---------|
| **1135** | ClusSvc | Node left cluster |
| **1136** | ClusSvc | Node rejoined cluster |
| **1177** | ClusSvc | Node recovery |
| **1179** | ClusSvc | Network name enumeration failure |
| **1205** | ClusSvc | Node cannot join cluster |
| **5000+** | ClusSvc | Various cluster events |

### Certificate Events
| Event ID | Source | Meaning |
|----------|--------|---------|
| **36882** | Schannel | Fatal alert (TLS failure) |
| **36887** | Schannel | Fatal alert (handshake failure) |
| **36888** | Schannel | Fatal alert (certificate expired/revoked) |
| **36889** | Schannel | Non-fatal alert |
| **36890** | Schannel | No common cipher |
| **36891** | Schannel | Certificate chain error |
| **36870/36871** | SChannel | Certificate expiry warnings |

### Windows Update Events
| Event ID | Source | Meaning |
|----------|--------|---------|
| **19** | Windows Update Agent | WU agent started |
| **20** | Windows Update Agent | WU agent stopped |
| **21** | Windows Update Agent | Update handler started |
| **24** | Windows Update Agent | Update search complete |
| **25** | Windows Update Agent | Download complete |
| **26** | Windows Update Agent | Install complete |
| **27** | Windows Update Agent | Install started |
| **34** | Windows Update Agent | Update failed to install |
| **100** | Windows Update | AU handler started |
| **101** | Windows Update | AU handler stopped |
| **102** | Windows Update | AU search complete |
| **103** | Windows Update | AU download started |
| **104** | Windows Update | AU download complete |
| **105** | Windows Update | AU install started |
| **106** | Windows Update | AU install complete |
| **107** | Windows Update | AU install failed |
| **117** | Windows Update | AU reboot status check |
| **240003** | Windows Update | Generic failure |




###Event Viewer Troubleshooting Approach

### Method
1. **Identify timeframe** when issue occurred
2. **Filter by level** (Critical/Error/Warning first)
3. **Filter by source** (component that generated event)
4. **Read Event ID** and look up meaning
5. **Check correlation** for related events
6. **Cross-reference** with other logs (multiple logs may show same issue)

### Common Investigation Paths
| Symptom | Logs to Check | Key Event IDs |
|---------|--------------|---------------|
| **Service crash** | Application, System | 1000, 1001 (App crash); 7000, 7031 (Service failure) |
| **BSOD** | System | 41, 65000+ (BugCheck codes in System log) |
| **Login failure** | Security | 4625 (failed logon), 4768/4769/4771 (Kerberos) |
| **Disk failure** | System | 51, 55, 57 (I/O errors) |
| **Network failure** | System | Network-specific events |
| **DNS failure** | DNS Server log (separate) | 4013, 65000+ |
| **Certificate issue** | System, Security | 36882-36891 (Schannel) |
| **Update failure** | Windows Update log | 103-107, 240003 |

### Tips
- Use `Custom View` to filter across multiple logs by Event ID
- Save filters as `.evtx` files or custom views for reuse
- Forward events to central collector for enterprise visibility
- Check **Forwarded Events** log if using Windows Event Collection (WEC)
- Use PowerShell for bulk/event analysis:
  ```powershell
  # Get all errors in last 24 hours across key logs
  Get-WinEvent -FilterHashtable @{LogName='System'; Level=1,2; StartTime=(Get-Date).AddDays(-1)} | 
    Where-Object {$_.Id -in (51,55,57,41,4625,7000,7031,36882)} |
    Select-Object TimeCreated, Id, ProviderName, Message |
    Format-Table -AutoSize
  ```





---

## PART 20 — Windows Authentication

> *Understand every authentication protocol and how to troubleshoot failures.*


###NTLM

### How NTLM Works
1. **Client** sends username (in plaintext) to server
2. **Server** sends challenge (random number)
3. **Client** hashes challenge with NT hash of password → sends response
4. **Server** forwards to KDC (on DC) for validation
5. **KDC** validates → sends success/failure back to server
6. **Server** grants or denies access

### NLM Versions
| Version | Features | Security |
|---------|----------|----------|
| **NTLMv1** | Original | Weak (susceptible to relay attacks) |
| **NTLMv2** | Improved with client challenge | Better (still has relay risk) |
| **NTLMv2 with session security** | Signing/sealing enabled | Best NTLM option |

### NTLM Security Settings (GPO)
```
Computer Configuration → Windows Settings → Security Settings → Local Policies → Security Options:
  - Network security: Restrict NTLM inbound/outbound NTLM traffic
  - NTLM: Enable LM/NTLM compatibility level
  - Audit: Audit NTLM authentication in domain
```

### NTLM Troubleshooting
```powershell
# Check NTLM status
klist tgt                      (shows current TGT - if Kerberos, NTLM was fallback)
whoami /all                    (shows authentication method)
nltest /sc_query:domain.com   (secure channel status)
```

| Issue | Cause | Resolution |
|-------|-------|-----------|
| NTLM relay attack | NTLM v1 enabled, no constraints | Disable NTLMv1, enable NTLM constraints |
| NTLM blocked | GPO restricting NTLM | Check NTLM restriction policies |
| NTLM used instead of Kerberos | Clock skew, SPN missing, DNS issue | Fix root cause for Kerberos |




###Kerberos

### How Kerberos Works
```
1. User logs in → sends credentials to KDC (DC)
2. KDC verifies → sends TGT (Ticket Granting Ticket) encrypted with user's NT hash
3. User presents TGT to KDC to request Service Ticket (TGS-REQ)
4. KDC validates TGT → sends Service Ticket (TGS-REP) for target service
5. User presents Service Ticket to target service (e.g., SQL server)
6. Service validates Service Ticket with KDC → grants access
```

### Key Components
| Term | Description |
|------|-------------|
| **KDC** | Key Distribution Center (runs on DC) |
| **TGT** | Ticket Granting Ticket (proves identity to KDC) |
| **Service Ticket** | Ticket for specific service |
| **TGT lifetime** | Default 10 hours (renewable) |
| **Service ticket lifetime** | Default 8 hours |
| **Clock skew tolerance** | 5 minutes (default) |
| **Preferred KDC** | DC with PDC Emulator role |

### Kerberos Commands
```powershell
# View current tickets
klist                    (all tickets)
klist tgt                (TGT only)
klist tgt -expand        (expanded TGT info)
klist st                   (service tickets)
klist purge                (clear all tickets)
klist -li 0x3e0            (list all tickets including flags)

# Refresh tickets
klist -renew               (renew TGT if renewable)
klist -renew 0x3e0         (renew specific ticket)

# Check Kerberos events
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4768,4769,4771}

# Check time sync
w32tm /query /status
w32tm /stripchart /computer:dc01 /dataonly /samples 5

# Check SPN
setspn -L domain\user         (list SPNs for account)
setspn -Q HTTP/mysite.domain.com   (query specific SPN)

# Check secure channel
Test-ComputerSecureChannel -Verbose
nltest /sc_query:domain.com
nltest /dsgetdc:domain.com
```

### Kerberos Troubleshooting
| Symptom | Cause | Resolution |
|---------|-------|-----------|
| **Clock skew** | Time > 5 min difference | Sync time from PDC Emulator |
| **SPN duplicate** | Two accounts registered same SPN | Remove duplicate: `setspn -D SPN domain\oldAccount` |
| **SPN missing** | No SPN for service account | Register: `setspn -S SPN domain\serviceAccount` |
| **Wrong SPN format** | SPN registered incorrectly | Verify format: `service/host.domain.com:port` |
| **DNS failure** | Can't locate KDC | Fix DNS resolution, SRV records |
| **Trust relationship broken** | Machine account expired/disabled | `Reset-ComputerMachinePassword` |
| **TGT expired** | Re-authenticate user | User logs in again |
| **Delegation failure** | Constrained/unconstrained delegation misconfigured | Configure appropriate delegation |
| **KDC unreachable** | Network issue, DC down | Network connectivity, DC health check |

### Authentication Flow Troubleshooting
```powershell
# Determine auth method used
whoami /all                      (look for "Kerberos" or "NTLM" in authentication ID)

# Force Kerberos (test)
klist purge                       (clear tickets)
tsdiscon                          (disconnect RDP; reconnect to force fresh auth)

# Check which auth was used for RDP
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} | 
  Where-Object {$_.Properties[8].Value -eq 10} |  # Logon type 10 = RDP
  Select-Object TimeCreated, @{N='AuthType';E={$_.Properties[10].Value}}  # Auth package
```




###LDAP / LDAPS

### LDAP
| Feature | Description |
|---------|-------------|
| **Port** | 389 (TCP/UDP) |
| **Encryption** | None (cleartext) |
| **Use** | AD queries, authentication |
| **Binding** | Simple bind (username/password), SASL bind |
| **Search scope** | Base, One Level, Subtree |

### LDAPS
| Feature | Description |
|---------|-------------|
| **Port** | 636 (TCP) |
| **Encryption** | TLS/SSL |
| **Use** | Secure AD queries |
| **Certificate** | DC must have certificate (AD DS auto-registers) |
| **Signing/Sealing** | LDAP signing (GPO), LDAP channel binding (Windows 8+/Server 2012+) |

### LDAP Troubleshooting
```powershell
# LDAP test (from server)
ldp.exe                         (GUI tool)

# LDAP queries
dsquery user -name "jdoe"
dsquery group -name "ITAdmins"
dsget user "CN=John Doe,CN=Users,DC=domain,DC=com" -samid
dsmember "CN=Group,DC=domain,DC=com"    (list members)

# Check LDAP connectivity
Test-NetConnection -ComputerName DC01 -Port 389
Test-NetConnection -ComputerName DC01 -Port 636

# RootDSE query (LDAP)
# Connect to: ldap://DC01.domain.com/RootDSE

# Check LDAP signing policy (GPO)
Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → Security Options:
  - Domain controller: LDAP server signing requirements
  - Domain member: LDAP server signing requirements
  - Domain member: LDAP negotiation signing requirements
```

### LDAP Signing
| Setting | Effect |
|---------|--------|
| **None** | No signing (insecure, deprecated) |
| **Require signing** | LDAP bound connections must be signed |
| **Require channel binding** | Requires TLS + channel binding tokens (most secure) |
| **Negotiate signing** | Attempt signing, fall back if not available |

### Common LDAP Issues
- **LDAP search limits**: Default 1000 results (modify with `Directory Server Administrator`)
- **Referral**: Cross-domain queries need proper GC references
- **Anonymous LDAP bind**: Disabled by default on modern DCs
- **LDAP signing requirement**: Client/server must both support signing
- **LDAPS certificate**: Must be issued by Enterprise CA, stored in Computer certificate store




###SPN / KDC / Tickets / Delegation

### SPN Management
```powershell
# List SPNs
setspn -L domain\account
setspn -Q HTTP/server.domain.com

# Register
setspn -S HTTP/webapp.domain.com domain\serviceAcct
setspn -S MSSQLSvc/db01.domain.com:1433 domain\sqlservice
setspn -S LDAP/DC01.domain.com domain\DC01$

# Remove (de-duplicate)
setspn -D HTTP/oldserver.domain.com domain\serviceAcct

# Verify no duplicates
setspn -Q HTTP/ | Where-Object {$_ -match "HTTP/"}

# Check SPN in AD
Get-ADServiceAccount -Identity "serviceAcct" -Properties ServicePrincipalName
```

### Delegation
| Type | Description | Security |
|------|-------------|----------|
| **Unconstrained** | TGT forwarded to any service | Less secure (TGT exposure) |
| **Constrained (Kerberos)** | TGS only forwarded to specific services | Better (specific services) |
| **Constrained (Protocol Transition + CBT)** | Uses S4U2Self/S4U2Proxy, requires CBT | Most secure (Windows 8+/Server 2012+) |
| **Resource-based (RBCR)** | Target service accepts delegating account | New (Server 2012+); cross-domain |

### Key Authentication Troubleshooting Commands
```powershell
# Comprehensive auth check
whoami /all                    (show auth method, groups, SID)
klist                          (show current Kerberos tickets)
klist tgt                      (show TGT)
klist purge                    (clear tickets, force re-auth)

# Secure channel
Test-ComputerSecureChannel -Verbose
Test-ComputerSecureChannel -Repair
nltest /sc_query:domain.com
nltest /dsgetdc:domain.com
nltest /dsgetdc:domain.com /dsflag      (detailed DC info)

# Time sync check
w32tm /query /status
w32tm /query /configuration
w32tm /stripchart /computer:dc01.domain.com /dataonly /samples 5

# Time set
w32tm /config /manualpeerlist:dc01.domain.com /syncfromflags:manual /reliable:true /update
w32tm /resync /rediscover

# Check if Kerberos was used
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4768} |
  Where-Object {$_.Properties['Status'].Value -eq 0}    (successful TGT requests)

# Check NTLM fallback
# If 4768 fails and 4625 follows → Kerberos failed, NTLM attempted
```

### Delegation Troubleshooting
| Issue | Cause | Resolution |
|-------|-------|-----------|
| **Delegation not working** | SPN not registered correctly | Register correct SPN for service account |
| **Constrained delegation fails** | Target SPN wrong | Verify SPN, check target service configuration |
| **Protocol transition fails** | CBT not enabled | Enable CBT on both client and target |
| **"An unknown security error occurred"** | Commonly resource-based delegation issue | Check RBCR settings on target, verify SPN |
| **Double-hop issue** | Unconstrained not configured or constrained misconfigured | Configure constrained delegation or RBCR |
| **Clock skew** | Time sync failure | Fix time sync (NTP → PDC Emulator) |





---

## PART 21 — Windows Networking Troubleshooting

> *Complete troubleshooting methodology — layered approach.*


###Layered Troubleshooting Model

### OSI/Troubleshooting Layers
| Layer | Focus | Key Checks |
|-------|-------|-----------|
| **L1 — NIC / Physical** | Hardware/link | Cable, NIC link, driver, switch port |
| **L2 — VLAN / MAC / Switch** | L2 connectivity | VLAN assignment, MAC address, switch config, ARP table |
| **L3 — IP / Subnet / Gateway / Routing** | Network layer | IP address, subnet mask, gateway, routing table, firewall rules |
| **L4 — TCP / UDP / Ports** | Transport | Port availability, firewall rules, application listener |
| **L5+ — DNS / Auth / Application** | Application | DNS resolution, authentication, application-specific checks |

### Key Command Reference by Layer
| Layer | Commands |
|-------|----------|
| **L1** | `ipconfig /all`, `Get-NetIPConfiguration`, `Get-NetAdapter`, `ethtool` (Linux) |
| **L2** | `arp -a`, `Get-NetNeighbor`, `switch show mac address-table` |
| **L3** | `ipconfig`, `route print`, `ping`, `Test-NetConnection`, `tracert`, `pathping`, `Get-NetRoute` |
| **L4** | `netstat -ano`, `Test-NetConnection -Port X`, `Get-NetTCPConnection`, `telnet host port`, `nmap` |
| **L5+** | `nslookup`, `Resolve-DnsName`, `ipconfig /flushdns`, `klist`, `whoami /all`, `gpresult` |

### Network Adapter Troubleshooting
```powershell
# Check adapter status
Get-NetAdapter | Select-Object Name, InterfaceDescription, Status, LinkSpeed, MediaType
Get-NetIPConfiguration | Where-Object {$_.NetAdapter.Status -eq 'Disconnected'}

# Disable/Enable adapter
Disable-NetAdapter -Name "Ethernet" -Confirm:$false
Enable-NetAdapter -Name "Ethernet"

# Reset adapter
Reset-NetAdapter -Name "Ethernet" -Confirm:$false

# Check adapter driver
Get-WmiObject Win32_NetworkAdapter | Select-Object Name, DriverVersion, DeviceID

# Check for driver issues
Get-WinEvent -FilterHashtable @{LogName='System'; Id=10000,10001,10003,10004}  # NIC-related events

# Reset TCP/IP stack
netsh int ip reset
netsh winsock reset

# Check NIC teaming
Get-NetLbfoTeam
Get-NetLbfoTeamMember
Get-NetLbfoTeamNic
```

### VLAN Troubleshooting
```powershell
# Check VLAN config on Windows
Get-NetAdapter | Where-Object {$_.VlanID -ne 0}
Get-NetAdapterVlan -Name "Ethernet"

# Trunk vs Access
# Access: VLAN ID = specific VLAN (e.g., 10)
# Trunk: VLAN ID = 0 (passthrough/untagged, or configured with allowed list)
```

### Firewall Troubleshooting
```powershell
# Check firewall status
Get-NetFirewallProfile | Select-Object Name, Enabled, DefaultInboundAction

# Check specific rule
Get-NetFirewallRule -DisplayName "RDP*" | Get-NetFirewallPortFilter

# Test if firewall is blocking
Test-NetConnection -ComputerName 192.168.10.10 -Port 3389

# Check if port is listening
netstat -ano | findstr ":3389"

# Windows Firewall with Advanced Security (wf.msc)
# Check inbound/outbound rules, connection security rules

# Network Security Policy (NLA) for RDP
Get-NetConnectionSecurityRule
```




###Example: Server Cannot Connect to SQL Server

### Step-by-Step (Do NOT immediately restart)

```
SYMPTOM: Server cannot connect to SQL Server on 192.168.10.50

┌─────────────────────────────────────────────────┐
│  Step 1: DNS                                   │
│  → Can server resolve "SQL01.domain.com"?      │
│  → nslookup / Resolve-DnsName SQL01            │
│  → If fails: DNS issue → fix DNS               │
│  ↓ OK                                          │
├─────────────────────────────────────────────────┤
│  Step 2: IP Connectivity                       │
│  → ping 192.168.10.50                          │
│  → If fails: L3 issue (IP, subnet, gateway)    │
│  ↓ OK                                          │
├─────────────────────────────────────────────────┤
│  Step 3: Port Check                            │
│  → Test-NetConnection 192.168.10.50 -Port 1433 │
│  → If fails: Port blocked or SQL not listening │
│  ↓ OK                                          │
├─────────────────────────────────────────────────┤
│  Step 4: Firewall                              │
│  → Check Windows Firewall on both sides        │
│  → Check network firewall (L3/L2)              │
│  → Check SQL Browser service (UDP 1434)        │
│  ↓ OK                                          │
├─────────────────────────────────────────────────┤
│  Step 5: SQL Listener                          │
│  → Check SQL Server service is running         │
│  → Check SQL is listening on correct IP/port   │
│  → SQL Server Configuration Manager            │
│  → Check TCP/IP protocol enabled               │
│  ↓ OK                                          │
├─────────────────────────────────────────────────┤
│  Step 6: Authentication                        │
│  → Check auth mode (Windows/ Mixed)            │
│  → Check user has SQL login / permission       │
│  → Check domain trust for Windows auth         │
│  → Check "Login failed" in SQL error log       │
│  ↓ OK                                          │
├─────────────────────────────────────────────────┤
│  Step 7: Application                           │
│  → Check application connection string         │
│  → Check SQL error log for connection errors   │
│  → Check max connection limit                  │
│  → Check SQL Agent / jobs                      │
└─────────────────────────────────────────────────┘
```

### Commands
```powershell
# Step 1: DNS
Resolve-DnsName SQL01.domain.com
nslookup SQL01.domain.com

# Step 2: IP connectivity
ping SQL01.domain.com
Test-NetConnection SQL01.domain.com -InformationLevel Detailed

# Step 3: Port
Test-NetConnection SQL01.domain.com -Port 1433
tcping SQL01.domain.com 1433

# Step 4: Firewall
Get-NetFirewallRule -Direction Inbound -Enabled True | 
  Where-Object {$_.Action -eq 'Block'} | Select-Object DisplayName

# Step 5: Check SQL is listening (run on SQL server)
Get-Service -Name MSSQLSERVER
netstat -ano | findstr ":1433"

# Step 6: Test auth (run from client)
sqlcmd -S SQL01.domain.com -E          (Windows auth)
sqlcmd -S SQL01.domain.com -U sa -P xxx  (Mixed auth)

# Step 7: SQL logs
# SQL Server Management Studio → Management → SQL Server Logs
```





---

## PART 22 — PowerShell

> *Explicitly a must-have in the JD.*


###Fundamentals

### Core Concepts
| Concept | Description |
|---------|-------------|
| **Cmdlets** | Built-in commands (Verbs-Nouns pattern: `Get-Service`, `Set-Item`) |
| **Parameters** | Modify cmdlet behavior (`-Name`, `-ComputerName`, `-Force`) |
| **Variables** | `$variable`, `$PSItem`, `$True`, `$False` |
| **Objects** | Everything is an object (.NET type with properties/methods) |
| **Pipeline** | `|` passes output as objects to next command (not text!) |
| **Properties** | Object attributes (e.g., `Name`, `Status`) |
| **Methods** | Object actions (e.g., `.Restart()`, `.Stop()`) |
| **Functions** | Reusable script blocks |
| **Modules** | Package of related cmdlets (`Import-Module ActiveDirectory`) |
| **Error handling** | `try/catch/finally`, `$ErrorActionPreference`, `-ErrorAction` |

### Error Handling
```powershell
# ErrorAction options
Get-Service -Name "Spooler" -ErrorAction SilentlyContinue
Get-Service -Name "Spooler" -ErrorAction Stop
Get-Service -Name "Spooler" -ErrorAction Continue

# try/catch
try {
    Get-ADUser -Identity "NonExistent" -ErrorAction Stop
} catch {
    Write-Error "Failed: $_"
} finally {
    Write-Host "Cleanup"
}

# $ErrorActionPreference
$ErrorActionPreference = "Stop"

# Check $error variable
$error[0]           # Last error
$error.Count        # Number of errors
$error.Clear()      # Clear errors

# Write-Error vs Throw
Write-Error "Non-terminating error"
Throw "Terminating error"
```

### Object Pipeline
```powershell
# Pipeline passes objects, not text
Get-Process | Where-Object {$_.CPU -gt 1000} | Sort-Object CPU -Descending | Select-Object -First 5 Name, CPU

# Property access
Get-Service | Select-Object Name, Status, StartType

# Method usage
$svc = Get-Service -Name Spooler
$svc.Restart()
$svc | Format-List *

# Where-Object vs Filter
Get-Process | Where-Object {$_.WorkingSet -gt 100MB}     # Pipeline filter
Get-Process -ErrorAction SilentlyContinue
```




###Essential Commands

### System
```powershell
# General system info
Get-ComputerInfo                    # Full system info
Get-CimInstance -ClassName Win32_OperatingSystem
Get-CimInstance -ClassName Win32_Processor
Get-CimInstance -ClassName Win32_PhysicalMemory
Get-CimInstance -ClassName Win32_LogicalDisk

# Hotfixes
Get-HotFix | Sort-Object InstalledOn -Descending
Get-HotFix -Id KB5005565

# Time
Get-Date
Get-WinEvent -MaxEvents 1 -LogName System | Where-Object {$_.Id -eq 6005}
```

### Services & Processes
```powershell
Get-Service                         # All services
Get-Service | Where-Object {$_.Status -eq 'Running'}
Get-Service -Name "Spooler"
Restart-Service -Name "Spooler" -Force
Set-Service -Name "Spooler" -StartupType Automatic

Get-Process | Sort-Object CPU -Descending | Select-Object -First 10
Get-Process notepad
Stop-Process -Id 1234 -Force
Get-Process | Where-Object {$_.Path -like "*temp*"}
```

### Event Logs
```powershell
Get-EventLog -LogName System -Newest 50
Get-WinEvent -LogName Application -MaxEvents 20
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} -MaxEvents 100
Get-WinEvent -LogName 'Applications and Services Logs\Microsoft\Windows\GroupPolicy\Operational'
```

### Computer Info (Short Reference)
```powershell
Get-CimInstance Win32_OperatingSystem    # OS version, last boot
Get-CimInstance Win32_ComputerSystem     # RAM, domain
Get-CimInstance Win32_Processor          # CPU info
Get-CimInstance Win32_LogicalDisk        # Drives
Get-CimInstance Win32_NetworkAdapterConfiguration -Filter "IPEnabled=True"  # Network
Get-CimInstance Win32_Product            # Installed software (slow)
```

### AD Commands
```powershell
Import-Module ActiveDirectory

# Users
Get-ADUser -Identity "jdoe"
Get-ADUser -Filter "Department -eq 'IT'" -Properties Department, DisplayName
Get-ADUser -Filter * -Properties LastLogonDate, PasswordLastSet | 
  Where-Object {$_.LastLogonDate -lt (Get-Date).AddDays(-90)}

# Computers
Get-ADComputer -Filter "OperatingSystem -like '*Windows 10*'" -Properties OperatingSystem
Get-ADComputer -Identity "PC01" -Properties OperatingSystem, LastLogonDate

# Groups
Get-ADGroup -Filter "Name -like '*IT*'"
Get-ADGroupMember -Identity "IT-Admins" -Recursive

# Domain
Get-ADDomain
Get-ADDomainController -Filter *
Get-ADDomainController -Identity "DC01.domain.com"

# OU
Get-ADOrganizationalUnit -Filter *

# Search base examples
Get-ADUser -Filter * -SearchBase "OU=Users,DC=domain,DC=com"
```

### DNS Commands
```powershell
Import-Module DnsServer

Get-DnsServerZone                                         # List zones
Get-DnsServerResourceRecord -ZoneName "domain.com" -Name "SRV01"
Add-DnsServerResourceRecordA -ZoneName "domain.com" -Name "SRV02" -IPv4Address "192.168.10.20"
Remove-DnsServerResourceRecord -ZoneName "domain.com" -Name "SRV02" -FQDN "SRV02.domain.com"
Get-DnsServerDiagnostics                                  # DNS server stats
```

### VMware Commands
```powershell
Import-Module VMware.VimAutomation.Core

Connect-VIServer -Server vcenter.domain.com

Get-VM                  # List all VMs
Get-VM -Name "MyVM"
Get-VMHost              # List ESXi hosts
Get-VMHost -Name "ESXi01" | Select-Object Name, State, Version, Build
Get-Datastore           # List datastores
Get-Datastore -Name "Datastore1" | Select-Object Name, FreeSpaceMB, CapacityMB

Get-VM "MyVM" | Get-NetworkAdapter | Select-Object NetworkName
Get-VM "MyVM" | Get-HardDisk | Select-Object CapacityGB, DiskType
```




###Automation Scripts

### Bulk User Creation
```powershell
# From CSV
Import-CSV "C:\new_users.csv" | ForEach-Object {
    $params = @{
        Name = $_.Name
        SamAccountName = $_.Sam
        UserPrincipalName = $_.UPN
        Path = $_.OU
        AccountPassword = (ConvertTo-SecureString $_.Password -AsPlainText -Force)
        Enabled = $true
        ChangePasswordAtLogon = $true
    }
    New-ADUser @params
}
```

### AD Reporting
```powershell
# Stale computer accounts
Get-ADComputer -Filter * -Properties LastLogonDate | 
  Where-Object {$_.LastLogonDate -lt (Get-Date).AddDays(-90)} |
  Select-Object Name, LastLogonDate | Sort-Object LastLogonDate

# Users who haven't logged in for 90 days
Search-ADAccount -UsersOnly -Inactive 90 | Select-Object Name, SamAccountName, LastLogonDate

# Locked out accounts
Search-ADAccount -LockedOut | Select-Object Name, SamAccountName, LockedOutTime

# Expiring passwords
Search-ADAccount -PasswordExpired | Select-Object Name, SamAccountName, PasswordLastSet

# All DCs status
Get-ADDomainController -Filter * | Select-Object Name, IPv4Address, OperatingSystem, IsGlobalCatalog
```

### Health Checks
```powershell
# Disk space
Get-PSDrive C, D, E | Select-Object Name, Used, Free, @{N='Free%';E={[math]::Round($_.Free/$_.Used*100,2)}}

# Services not running
Get-Service | Where-Object {$_.Status -ne 'Running' -and $_.StartType -eq 'Automatic'} | Select-Object Name, Status, StartType

# Last reboot time
(Get-CimInstance -ClassName Win32_OperatingSystem).LastBootUpTime

# Network connectivity test
$servers = @("SRV01", "SRV02", "DC01")
foreach ($srv in $servers) {
    $test = Test-Connection -ComputerName $srv -Count 1 -Quiet
    [PSCustomObject]@{
        Server = $srv
        Ping = $test
    }
}
```

### Service Restart
```powershell
# Restart services that match pattern
Get-Service | Where-Object {$_.Name -like "W3SVC*" -or $_.Name -like "Spooler"} | 
  Restart-Service -Force

# Check and restart
Get-Service -Name Spooler | 
  If ($_.Status -ne 'Running') { Start-Service -Name Spooler }
```

### Patch Validation
```powershell
# Check if specific KB installed
Get-HotFix -Id KB5005565
Get-HotFix | Where-Object {$_.HotFixID -like "*5005*"} | 
  Select-Object HotFixID, InstalledOn, Description

# Check pending reboot
Get-CimInstance -ClassName Win32_QuickFixEngineering | Select-Object HotFixID, InstalledOn
# Or: (Use PSWindowsUpdate module)
Get-WURebootStatus
```

### Log Collection
```powershell
# Export events
Get-WinEvent -FilterHashtable @{LogName='System'; StartTime=(Get-Date).AddHours(-24)} |
  Export-Csv "C:\logs\system_events.csv" -NoTypeInformation

# Forward events (Windows Event Collector)
Wecutil qc                                          (configure WEC)
Wevtutil es <channel> <filename>                    (export channel)

# Create custom event query
$xml = '<QueryList><Query Id="0" Path="Application"><Select Path="Application">*[System[Level=1 or Level=2]]</Select></Query></QueryList>'
Get-WinEvent -FilterXml $xml | Export-Csv "C:\logs\errors.csv" -NoTypeInformation
```

### Compliance Reporting
```powershell
# Check security policy
auditpol /get /category:*

# Check NTLM restriction policy
Get-CimInstance -ClassName Win32_NetworkClient | Select-Object EnableLMCompatibility

# Check TLS settings
Get-TlsCipherSuite | Select-Object Name

# Check RDP settings
Get-CimInstance -ClassName Win32_TSPermissionsSetting -Namespace root\cimv2\terminalservice | 
  Select-Object *
```

### Remediation Example
```powershell
# Restart service on multiple servers
$servers = @("SRV01", "SRV02", "SRV03")
foreach ($srv in $servers) {
    Invoke-Command -ComputerName $srv -ScriptBlock {
        Restart-Service -Name "Spooler" -Force
    } -ErrorAction SilentlyContinue
}

# Fix registry across servers
$servers = @("SRV01", "SRV02")
foreach ($srv in $servers) {
    Invoke-Command -ComputerName $srv -ScriptBlock {
        Set-ItemProperty -Path "HKLM:\SOFTWARE\MyApp" -Name "Version" -Value "2.0" -Force
    }
}
```





---

## PART 23 — Patch Management

> *Enterprise patch management: WSUS, SCCM/MECM, and the full lifecycle.*


###Patch Fundamentals

### Patch Types
| Type | Description | Scope |
|------|-------------|-------|
| **Cumulative Update (CU)** | Includes ALL previous patches for that month + new fixes | Monthly, comprehensive |
| **Security Update** | Specific security fix | Critical/Important |
| **Servicing Stack Update (SSU)** | Updates Windows Update engine itself | Before CU (mandatory) |
| **Feature Update** | Major version upgrade (e.g., 21H2 → 22H2) | Semi-annual/annual |
| **Quality Update** | Non-security fixes (productivity, runtime fixes) | Monthly (separate from security in newer models) |

### Patch Tuesday
- Second Tuesday of each month (Monthly Rollup for Windows, Security-only for Office)
- Extended Security Updates (ESU) for legacy OS versions
- Out-of-band patches for zero-day vulnerabilities (separate schedule)

### SCCM/MECM Concepts
| Component | Description |
|-----------|-------------|
| **SCCM (ConfigMgr)** | System Center Configuration Manager |
| **MECM (Microsoft Endpoint Configuration Manager)** | Modern name for SCCM |
| **Site Server** | Primary site server |
| **Site System** | Role-bearing server (DP, MP, etc.) |
| **Distribution Point (DP)** | Distributes packages/updates |
| **Management Point (MP)** | Client communication (policy, inventory) |
| **Software Update Point (SUP)** | WSUS integration for patch approval |
| **Update Baseline** | Set of updates for compliance |
| **Deployment** | Approved update + schedule + target collection |
| **Package** | Software to deploy (legacy) |
| **Application** | Modern app deployment model |

### Enterprise Patch Process
```
Assess
 ↓       (identify missing patches, evaluate impact)
Test
 ↓       (test in isolated environment)
Approve
 ↓       (change request / CAB approval)
Deploy
 ↓       (deploy to production in rings/maintenance window)
Validate
 ↓       (verify patches installed, service running)
Report
 ↓       (patch compliance report)
Remediate (fix failed patches, retry deployment)
```

### WSUS (Windows Server Update Services)
| Feature | Description |
|---------|-------------|
| **Role** | Windows Server role for patch management |
| **Automatic Approvals** | Rules to auto-approve updates (Security Updates, Critical Updates) |
| **Target Groups** | Server groups for approval scope |
| **Automatic Updates** | Configure via GPO → Windows Updates for Business or classic policy |
| **Update Root** | `C:\WSUS` (default) |
| **Synchronization** | Pull from Microsoft Update or upstream WSUS |

```powershell
# WSUS PowerShell (SUS) module
Import-Module UpdateServices
$wsus = Get-WsusServer -Name "WSUS01" -PortNumber 8530 -UseSsl

# Get updates
$updates = $wsus.GetUpdates()
$updates | Where-Object {$_.Approval -eq "Unapproved"} | Select-Object Title, UpdateId, MsrcSeverity

# Approve updates
$update = $wsus.GetUpdate($updateId)
$update.Approve("Install", $targetGroup)

# Get compliance
$wsus.GetComputerTargetGroups() | Select-Object Name, TargetCount
$wsus.GetComputerTargets() | Select-Object FullDomainName, LastReportedStatus, LastReportedTime
```

### GPO Windows Update Settings
```
Computer Configuration → Administrative Templates → Windows Components → Windows Update:
  - Configure Automatic Updates
  - Set deadlines for auto-update restarts
  - No auto-restart with logged on users
  - Specify intranet Microsoft update service location (WSUS)
```

### Patch Troubleshooting
| Issue | Diagnosis | Resolution |
|-------|-----------|-----------|
| **Patch failed** | Check CBS log (`C:\Windows\Logs\CBS\CBS.log`) | DISM repair, SFC, manual install |
| **Windows Update errors** | WUErr log, Event Viewer Windows Update log | WU reset tool, clear SoftwareDistribution |
| **WSUS connectivity** | Client settings, GPO applied, port 8530/8531 | Test: `C:\Program Files\Update Services\Tools\wsusutil.exe ping` |
| **Disk space** | C:\Windows\Temp, C:\Windows\SoftwareDistribution | Clean up, expand disk |
| **Pending reboot** | Many updates require reboot before/after | Check reg keys: `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending` |
| **Component store** | CBS store corruption | `DISM /Online /Cleanup-Image /RestoreHealth`, then `sfc /scannow` |
| **Servicing errors** | CBS log error codes | Check specific CBS error (0x800F081E common) |

### Clear Windows Update Cache
```powershell
# Stop Windows Update services
Stop-Service -Name wuauserv
Stop-Service -Name bits
Stop-Service -Name cryptSvc
Stop-Service -Name msiserver

# Clear cache
Remove-Item -Path C:\Windows\SoftwareDistribution\* -Recurse -Force

# Start services
Start-Service -Name wuauserv
Start-Service -Name bits
Start-Service -Name cryptSvc
Start-Service -Name msiserver

# Reset Windows Update (advanced)
# Use Microsoft "System Update Readiness Tool" or:
net stop wuauserv
net stop cryptSvc
rd /s /q C:\Windows\SoftwareDistribution
net start wuauserv
net start cryptSvc
```

### Check Pending Reboot
```powershell
# Multiple methods
Get-CimInstance -ClassName Win32_QuickFixEngineering | Select-Object HotFixID, InstalledOn
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing" -Name RebootPending -ErrorAction SilentlyContinue
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired" -ErrorAction SilentlyContinue
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Updates\UpdateExeVolatile" -ErrorAction SilentlyContinue
```




---

## PART 24 — Hardening & Security

> *Windows Security baseline, CIS concepts, vulnerability management.*


###Windows Security

### Microsoft Security Baseline
- **Microsoft Security Baseline (MSB)**: Previously called "Security Compliance Toolkit" (SCT)
- Published by Microsoft, covers Windows 10/11 and Windows Server
- Includes: OS settings, Microsoft Defender, Microsoft Edge, IE11, TLS config
- Available at: [Microsoft MSB GitHub](https://github.com/MicrosoftDocs/security-compliance-toolkit-docs)
- **Application Compliance** and **Security Baseline** are separate documents

### CIS Benchmarks
| Component | Description |
|-----------|-------------|
| **CIS (Center for Internet Security)** | Provides configuration benchmarks |
| **CIS Benchmark** | Step-by-step hardening guide for specific OS/application |
| **CIS-CAT** | Automated assessment tool (Pro, free for community) |
| **Level 1** | Baseline, generally compatible with software |
| **Level 2** | More secure, may affect legacy software |
| **CIS Hardened Images** | Pre-hardened VM images (Azure, AWS) |

### Principle of Least Privilege
| Concept | Implementation |
|---------|----------------|
| **Local admin** | Remove from users; use LAPS for random local admin passwords |
| **Group membership** | Add users to groups rather than direct permissions |
| **Service accounts** | Minimum privileges needed for service function |
| **User rights** | Restrict "Allow log on locally", "Access this computer from network" |
| **Rights delegation** | Delegate specific admin tasks rather than giving full admin |
| **Admin Tier Model** | Tier 0 (DCs, Tier 0 infrastructure) → Tier 1 (servers) → Tier 2 (workstations) |

### LAPS (Local Administrator Password Solution)
- Installs via GPO
- Random password per computer, stored in AD (attribute on computer object)
- Regularly rotated
- Access via ADUC or PowerShell

```powershell
# Check LAPS password (requires AD permissions)
Get-ADComputer -Identity "PC01" -Properties ms-Mcs-AdmPwd, ms-Mcs-AdmPwdExpirationTime |
  Select-Object Name, ms-Mcs-AdmPwd, ms-Mcs-AdmPwdExpirationTime
```

### Local Administrator Control
| Method | Description |
|--------|-------------|
| **LAPS** | Random password per device, stored in AD |
| **Local Administrator Password Solution** | Microsoft tool (predecessor to LAPS) |
| **Admin SD template** | Disables local admin group if >15 members |
| **Restrict local admin** | GPO: "Deny log on locally" for non-admin |
| **PAM/PIM** | Privileged Identity Management (Azure AD) |
| **JIT access** | Just-In-Time elevation (via PIM) |

### Windows Firewall
```powershell
# Ensure firewall enabled (all profiles)
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True

# Block all inbound by default (advanced)
Set-NetFirewallProfile -Profile Public -DefaultInboundAction Block
Set-NetFirewallProfile -Profile Private -DefaultInboundAction Block
Set-NetFirewallProfile -Profile Domain -DefaultInboundAction Block

# Audit mode
Set-NetFirewallProfile -Profile Domain,Public,Private -DefaultInboundAction Block -Enabled True
# Use audit mode to log without blocking, then create specific allow rules
```




###SMB Security / NTLLM Reduction / LDAP Security

### SMB Security
| Setting | Implementation |
|---------|---------------|
| **Disable SMBv1** | `Disable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol` |
| **Require SMB signing** | GPO: Computer Config → Windows Settings → Security Settings → Local Policies → Security Options → "Microsoft network client/server: Digitally sign communications" |
| **SMB encryption** | `Set-SmbServerConfiguration -EncryptData $true` |
| **Block legacy auth** | Disable NTLMv1, LM |
| **Audit SMB access** | Enable Object Access auditing on file shares |

```powershell
# Disable SMBv1 (all methods)
Disable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol-Server
Disable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol-Client

# Or via registry (also prevents loading)
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters" -Name "SMB1" -Value 0 -Type DWord
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\mrxsmb10" -Name "Start" -Value 4 -Type DWord

# Check SMB server config
Get-SmbServerConfiguration | Select-Object EnableSMB1Protocol, EnableSMB2Protocol, EncryptData, RequireSecuritySignature

# Require SMB signing (GPO)
# Computer Configuration → Administrative Templates → Network → Lanman Workstation → "Client/server: Digitally sign communications (always)" → Enabled
```

### NTLM Reduction
| Strategy | Implementation |
|----------|---------------|
| **Audit NTLM** | GPO: "Audit NTLM authentication in domain" (Security Option) |
| **Block NTLMv1** | LMCompatibilityLevel = 5 |
| **NTLM restrictions** | NTLM Authentication Restrictions policy |
| **TLS protection** | Enable TLS 1.2+, disable TLS 1.0/1.1 |
| **LDAP signing** | Require LDAP signing on servers and clients |

```
GPO: Computer Configuration → Windows Settings → Security Settings → Local Policies → Security Options:
  Network security: Restrict NTLM inbound from remote servers → Audit
  Network security: Restrict NTLM outbound from this computer to remote servers → Audit
  Network security: LAN Manager authentication level → Send NTLMv2 response only
```

### LDAP Security
| Setting | Value |
|---------|-------|
| **LDAP signing** | Require signing on DCs and clients |
| **LDAP channel binding** | Require (Windows Server 2012+) |
| **LDAP channel binding (GPO)** | "Domain controller: LDAP server signing requirements" = Require signing |
| **Disable simple bind** | LDAP simple bind disabled by default on modern DCs |
| **LDAP referral** | Secure referrals to LDAPS |

### TLS Configuration
```powershell
# Enable TLS 1.2, disable TLS 1.0/1.1
# Via registry
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Server" -Name "Enabled" -Value 0 -Type DWord -Force
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.1\Server" -Name "Enabled" -Value 0 -Type DWord -Force
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Server" -Name "Enabled" -Value 1 -Type DWord -Force
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Server" -Name "DisabledByDefault" -Value 0 -Type DWord -Force

# Check TLS cipher suites
Get-TlsCipherSuite | Select-Object Name

# Enable strong cryptography
Set-TlsCipherSuite -Name "TLS_AES_256_GCM_SHA384" -TLSVersion 1.3

# Schannel settings
# https://learn.microsoft.com/en-us/windows-server/security/tls/tls-cipher-suites-in-windows-server-2019
```

### Audit Policy
```powershell
# Advanced Audit Policy Configuration
# Computer Configuration → Windows Settings → Security Settings → Advanced Audit Policy Configuration
# Key auditing categories:
#   Logon/Logoff: Audit, Success, Failure
#   Access of Object: Audit, Success, Failure (for sensitive objects)
#   Privilege Use: Audit Failure (privilege escalation)
#   System: Audit Failure (crash/reboot)
#   Detailed Tracking: Audit Failure (process tracking)

# Apply via command
auditpol /set /category:"Logon/Logoff" /success:enable /failure:enable
auditpol /set /category:"Access of Object" /success:enable /failure:enable
```




###Vulnerability Management

### Key Concepts
| Term | Description |
|------|-------------|
| **CVE** | Common Vulnerabilities and Exposures (unique ID per vulnerability) |
| **CVSS** | Common Vulnerability Scoring System (score 0.0-10.0; severity: Low/Med/High/Critical) |
| **Vulnerability scanning** | Tools to detect known vulnerabilities |
| **Remediation** | Fixing the vulnerability (patch, config change) |
| **Compensating control** | Alternative control when direct fix not possible (e.g., firewall rule when patch unavailable) |
| **Exception** | Documented risk acceptance for specific system |
| **Risk acceptance** | Formal decision to accept vulnerability risk |

### Vulnerability Scanning Tools
| Tool | Type | Description |
|------|------|-------------|
| **Qualys** | Cloud/agent | Enterprise vulnerability management |
| **Nessus** | Agent | Popular vulnerability scanner |
| **Rapid7 InsightVM** | Agent | Vulnerability management + IR |
| **Microsoft Defender Vulnerability Mgmt** | Cloud | Integrated with Defender |
| **OpenVAS** | Open-source | Greenbone vulnerability scanner |
| **Wazuh** | Open-source | SIEM + vulnerability detection |

### CVSS Score Reference
| Score | Severity | Example |
|-------|----------|---------|
| **0.0-3.9** | Low | Minor information disclosure |
| **4.0-6.9** | Medium | Privilege escalation (limited) |
| **7.0-8.9** | High | Remote code execution (non-critical system) |
| **9.0-10.0** | Critical | Domain admin compromise, wormable |

### Remediation Workflow
```
1. Detect vulnerability (scan results)
      ↓
2. Assess risk (CVSS, asset value, exposure)
      ↓
3. Plan remediation (patch, configure, replace, or accept risk)
      ↓
4. Implement fix (deploy patch, change config)
      ↓
5. Validate fix (rescan, test functionality)
      ↓
6. Document (vulnerability ID, fix applied, date, responsible party)
```

### Compensating Controls
| Scenario | Compensating Control |
|----------|---------------------|
| Can't patch legacy application | Isolate in network segment, restrict access |
| Cannot disable legacy protocol | Restrict via firewall rules, audit usage |
| Cannot upgrade OS | Virtualize, isolate, add external monitoring |
| Cannot remove legacy application | Lock down access, log usage, monitor |

### Risk Acceptance
- Requires documented approval (change advisory board, CISO, etc.)
- Time-bound (expires after X months)
- Must include compensating controls
- Must be monitored
- Must include review date

### Vulnerability Scenarios
| Scenario | Typical Remediation |
|----------|-------------------|
| **Critical CVE on DC** | Emergency patch (out-of-band) or isolate if patch unavailable |
| **High CVE on file server** | Patch in next maintenance window |
| **Medium CVE on workstation** | Include in next Patch Tuesday cycle |
| **Low CVE** | Include in regular patch cycle or accept risk |





---

## PART 25 — Backup

> *JD requires coordinating backup and recovery and verifying recoverability.*


###Backup Fundamentals

### Backup Types
| Type | Description | Pros | Cons |
|------|-------------|------|------|
| **Full** | Complete data copy | Fast restore, standalone | Slowest backup, most storage |
| **Incremental** | Only changes since last backup (any type) | Fastest backup, least storage | Slow restore (chain dependent) |
| **Differential** | Changes since last full | Moderate backup/restore | Growing differential over time |
| **Synthetic Full** | Periodic full from incrementals | Fast backup, full restore available | Requires processing |

### Backup Chain Example
```
Monday: Full (100GB)
Tuesday: Incremental (5GB)
Wednesday: Incremental (7GB)
Thursday: Incremental (6GB)
Friday: Incremental (8GB)

Restore: Monday Full → Tuesday Incr → Wednesday Incr → Thursday Incr → Friday Incr
(Each depends on previous; chain break = data loss)
```

### Backup Technologies
| Term | Description |
|------|-------------|
| **Image-level backup** | Full VM/servers snapshot (bare-metal restore capability) |
| **File-level backup** | Individual files/folders |
| **Application-aware backup** | Application-aware (VSS integration for SQL, AD, Exchange) |
| **Application consistency** | Ensures app transactions committed before backup |
| **Crash consistency** | Backup at point in time; app may not have committed all transactions |
| **Changed Block Tracking (CBT)** | Tracks changed blocks since last backup (VMware) |
| **Backup repository** | Location where backups are stored |
| **Backup retention** | Policy for how long backups are kept |

### VMware Backup
| Concept | Description |
|---------|-------------|
| **Snapshot-based** | VMware snapshot created → backup from snapshot → snapshot deleted |
| **CBT** | VMware Change Block Tracking; more efficient than snapshot method |
| **VADP (vStorage APIs for Data Protection)** | VMware backup framework |
| **Hot-add** | Backup server accesses VM disk files directly |
| **NBD** | Network block device (backup via network) |
| **NBDSSL** | NBD over SSL (encrypted) |

### SQL/AD-Aware Backup
| System | Consideration |
|--------|--------------|
| **SQL Server** | Use VSS writer; backup transaction logs; checkpoint before backup |
| **Active Directory** | Use VSS writer; System State backup includes AD database, SYSVOL, registry, boot files |
| **Exchange** | Use VSS writer; truncate logs after backup |
| **Oracle/DB2** | Application-specific backup agents |

### Backup Best Practices
| Practice | Description |
|----------|-------------|
| **3-2-1 Rule** | 3 copies, 2 different media, 1 offsite |
| **3-2-1-1-0 Rule** | +1 immutable, 0 errors verified |
| **Test restore** | Regular restore testing (at least monthly) |
| **Backup validation** | Verify backup integrity (checksums, test boots) |
| **Immutable backups** | Cannot be modified/deleted (ransomware protection) |
| **Backup encryption** | Encrypt at rest and in transit |
| **Document retention** | Document backup policies and schedules |

### Backup Retention Examples
| Policy | Description |
|--------|-------------|
| **GFS** | Grandfather-Father-Son (daily, weekly, monthly, yearly) |
| **Incremental-forever** | Initial full + incrementals forever |
| **Synthetic full weekly** | Weekly synthetic full + daily incrementals |
| **Retention** | 30 days local, 90 days offsite, 1 year archived |




###Backup Verification & Restore Testing

### Backup Verification Checklist
- [ ] Backup job completed successfully (no errors in logs)
- [ ] Backup size is reasonable (alert if suddenly smaller/larger)
- [ ] Checksum validation passed
- [ ] Backup repository has available space
- [ ] Retention policy applied
- [ ] Last successful backup within SLA

### Restore Testing
```
Scenario: Restore VM from backup
───────────────────────────────────
1. Identify backup date/time
2. Select restore type (full/incremental/differential)
3. Restore to isolated/test environment (PITR)
4. Boot VM and validate
5. Test application functionality
6. Verify data consistency
7. Document test results
8. Delete test restore (if not retained)
```

### Backup Troubleshooting
| Issue | Cause | Resolution |
|-------|-------|-----------|
| Backup fails | VSS writer issue, disk space, network | Check VSS writers, free space, network connectivity |
| Backup size small | No changes since last backup (normal for incremental) | Normal; check full backup schedule |
| Backup size unexpectedly large | Snapshot not consolidated, large change rate | Consolidate snapshots, investigate change rate |
| Backup job stuck | Network issue, storage latency, backup server overloaded | Check job status, restart service, retry |
| Backup corruption | Storage issue, interrupted backup | Restore test, rerun backup |
| Backup window exceeded | Too much data, slow storage/network | Adjust window, add backup proxy, optimize |

### VSS (Volume Shadow Copy Service)
```powershell
# Check VSS writers
vssadmin list writers
# All writers in "Stable" state = healthy
# "Failed" or "Timed out" = issue (restart service, reboot)

# Check VSS providers
vssadmin list providers

# Common VSS issues
# Event ID 8236, 8237: VSS writer errors
# Event ID 8193: VSS errors in System log
```




---

## PART 26 — Disaster Recovery

> *RPO, RTO, BCP, DR — fundamental business continuity concepts.*


###DR Fundamentals

### Key Metrics
| Metric | Definition | Example |
|--------|-----------|---------|
| **RPO (Recovery Point Objective)** | Maximum acceptable data loss (measured in time) | "We can tolerate 1 hour of data loss" = RPO = 1 hour |
| **RTO (Recovery Time Objective)** | Maximum acceptable downtime (measured in time) | "We need to be back up in 4 hours" = RTO = 4 hours |
| **MTPD (Maximum Tolerable Period of Disruption)** | Absolute maximum time business can function without service | Usually RTO + buffer |
| **MTD (Maximum Tolerable Downtime)** | Synonym for MTPD |
| **RTO vs RPO** | RPO = data loss, RTO = time to recover |

### Related Concepts
| Term | Description |
|------|-------------|
| **BCP (Business Continuity Plan)** | Plan for keeping business running during disruption |
| **DR (Disaster Recovery)** | Technical plan for restoring IT services after disaster |
| **HA (High Availability)** | Minimize downtime via redundancy (single site) |
| **Backup** | Data copy for restore; doesn't guarantee availability |
| **Recovery** | Actual restoration process |

### Relationship Diagram
```
Business Continuity Plan (BCP)
    ├── Business Impact Analysis (BIA)
    │       ↓
    │   Identify critical systems, RTO, RPO
    │       ↓
    └── Disaster Recovery Plan (DRP)
            ├── Technical recovery procedures
            ├── HA strategies (minimize RTO)
            ├── Backup strategies (minimize RPO)
            └── DR testing (verify recoverability)
```

### DR Strategies by Scope
| DR Type | Description | RPO/RTO |
|---------|-------------|---------|
| **Backup & Restore** | Restore from backup to rebuilt infrastructure | RPO: hours, RTO: hours/days |
| **VM Replication** | VMs replicated to secondary site (asynchronous) | RPO: minutes, RTO: minutes |
| **Storage Replication** | Storage-level async/sync replication | RPO: seconds-minutes, RTO: hours |
| **Pilot Light** | Minimal infrastructure running; deploy VMs on demand | RPO: near-zero, RTO: hours |
| **Warm Standby** | Running but scaled-down environment | RPO: seconds-minutes, RTO: minutes-hours |
| **Hot Standby** | Full duplicate environment running | RPO: near-zero, RTO: near-zero |
| **Multi-site Active-Active** | Both sites serving traffic simultaneously | RPO: zero, RTO: zero (failover) |




###DR Scenarios

| Scenario | Response |
|----------|----------|
| **Host failure** | HA restarts VMs on other hosts; vMotion for live migration |
| **VM failure** | Restart VM; restore from backup if corrupted |
| **Storage failure** | Failover to replicated storage; restore from backup |
| **Datacenter failure** | DR failover to secondary site; DNS reroute |
| **Domain controller failure** | DCDiag; restore from AD Recycle Bin or System State; seize FSMO if needed |
| **Network failure** | Check DNS, routing, firewall, physical connectivity |

### DR Testing Plan
```
DR Test Procedure
─────────────────
1. Plan
   → Define scope (which systems, what scenario)
   → Schedule (maintenance window)
   → Stakeholder communication
      ↓
2. Failover
   → Stop primary VMs
   → Failover to secondary site (VMware: SRM; Hyper-V: Replica)
   → Verify VMs running on secondary
      ↓
3. Application Validation
   → Test application functionality
   → Check service connectivity
   → Verify database access
      ↓
4. Data Validation
   → Check last replicated data
   → Compare data with primary (pre-failover snapshot)
   → Verify RPO compliance
      ↓
5. Business Validation
   → End-user acceptance testing
   → Business process verification
      ↓
6. Failback
   → Reverse replication (secondary → primary)
   → Stop secondary VMs
   → Restore primary VMs from replication
   → Validate primary site
      ↓
7. Lessons Learned
   → Document issues encountered
   → Update DR plan
   → Schedule next test
```

### Hyper-V DR
- **Hyper-V Replica**: Asynchronous VM replication between Hyper-V servers/sites
- **Replication frequency**: 30 seconds, 5 minutes, 15 minutes, 30 minutes, 1 hour
- **Test failover**: Create test VM (isolated) without affecting replication
- **Automatic failover**: Configured for planned failover

### VMware DR
- **SRM (Site Recovery Manager)**: Automated DR orchestration
- **vSphere Replication**: VM replication at hypervisor level (not storage-level)
- **Storage Replication**: VSAN/NSX/Storage-level async replication

### Key DR Commands
```powershell
# Check DR readiness (VMware)
Get-DrsVM -Cluster "DRCluster" | Select-Object Name, HAState

# Test failover (Hyper-V)
Test-VMReplication -VMName "MyVM"

# Check replication status (Hyper-V)
Get-VMReplication

# Start planned failover (Hyper-V)
Start-VMFailover -VMName "MyVM" -ComputerName "ReplicaServer"
```





---

## PART 27 — Azure IaaS

> *JD specifically asks for Azure IaaS and Azure Arc.*


###Azure VM

### Core Concepts
| Term | Description |
|------|-------------|
| **VM** | Virtual machine in Azure |
| **VM Size** | CPU/RAM configuration (e.g., B2s, D4s_v5) |
| **OS Disk** | Boot disk (OS installed) |
| **Data Disk** | Additional disks for data |
| **NIC** | Network interface (1+ per VM) |
| **NSG** | Network Security Group (cloud firewall) |
| **VNet** | Virtual Network (isolated network) |
| **Subnet** | Subnet within VNet |
| **Public IP** | Internet-routable IP |
| **Private IP** | Internal VNet IP |

### VM Deployment
```powershell
# Azure PowerShell
Connect-AzAccount
New-AzResourceGroup -Name "MyRG" -Location "East US"

New-AzVM -Name "MyVM" -ResourceGroupName "MyRG" -Location "East US" `
  -Image "Win2022Datato1234" `
  -VmSize "Standard_B2s" `
  -AdminUsername "adminuser" `
  -AdminPassword (ConvertTo-SecureString "P@ssw0rd" -AsPlainText -Force) `
  -SubnetName "default" `
  -SecurityGroupName "MyNSG" `
  -PublicIpAddressName "MyPublicIP" `
  -OpenPorts 3389

# Alternative: ARM template / Terraform / Azure CLI
az vm create -g MyRG -n MyVM --image Win2022Datato1234 --size Standard_B2s --admin-username adminuser --generate-ssh-keys
```

### VM Sizes
| Series | Purpose | Example |
|--------|---------|---------|
| **B-series** | Burstable (dev/test) | B2s (2 vCPU, 4GB) |
| **D-series** | General purpose | D4s_v5 (4 vCPU, 16GB) |
| **E-series** | Memory optimized | E8s_v5 (8 vCPU, 64GB) |
| **F-series** | Compute optimized | F8s_v2 (8 vCPU, 16GB) |
| **G-series** | GPU | NC24ads_A100_v4 (GPU) |
| **L-series** | Storage optimized | L8s_v3 (high disk throughput) |
| **M-series** | High memory | M128ms_v2 (128 vCPU, 3.8TB) |

### Managed Disk Types
| Type | Description | Max IOPS | Use Case |
|------|-------------|----------|----------|
| **Standard HDD** | HDD-based | 500 | Backup, infrequently accessed |
| **Standard SSD** | SSD-based | 1,000 | Dev/test, low-latency |
| **Premium SSD** | Premium SSD | 30,000+ | Production workloads |
| **Ultra Disk** | Ultra (newest) | 160,000+ | SQL Server, Oracle, high IOPS |

```powershell
# Disk operations
Get-AzDisk -ResourceGroupName "MyRG" | Select-Object Name, DiskSizeGB, DiskType, ProvisioningState
New-AzDisk -ResourceGroupName "MyRG" -DiskName "DataDisk1" -SkuName "Premium_LRS" -DiskSizeGB 128 -Location "East US"

# Attach disk to VM
$vm = Get-AzVM -Name "MyVM" -ResourceGroupName "MyRG"
$disk = Get-AzDisk -ResourceGroupName "MyRG" -Name "DataDisk1"
$vm = Add-AzVMDataDisk -VM $vm -Name "DataDisk1" -CreateOption "Attach" -ManagedDiskId $disk.Id -Lun 0
Update-AzVM -VM $vm -ResourceGroupName "MyRG"
```

### NSG (Network Security Group)
| Rule Property | Description |
|---------------|-------------|
| **Source** | IP/CIDR, Service Tag, Application Security Group |
| **Source Port** | * or specific range |
| **Destination** | IP/CIDR, Service Tag, ASG |
| **Destination Port** | * or specific range |
| **Protocol** | TCP/UDP/Any |
| **Action** | Allow/Deny |
| **Priority** | 100-4096 (lower = higher priority) |
| **Name** | Rule name |

```powershell
# NSG rules
Get-AzNetworkSecurityGroup -Name "MyNSG" -ResourceGroupName "MyRG" | 
  Select-Object -ExpandProperty SecurityRules | 
  Select-Object Name, Direction, Action, Protocol, SourcePortRange, DestinationPortRange, Priority

# Create NSG rule
New-AzNetworkSecurityRuleConfig -Name "Allow-RDP" -Direction Inbound -Access Allow -Protocol Tcp `
  -SourceAddressPrefix * -SourcePortRange * -DestinationAddressPrefix * -DestinationPortRange 3389 -Priority 1000

# NSG default rules (always exist)
# Priority 65500: AllowAllInternalInBound
# Priority 65501: AllowAllInternalOutBound
# Priority 65502: AllowAzureLoadBalancerInBound
# Priority 65503: AllowAzureLoadBalancerOutBound
# Priority 65504: DenyAll
```

### Availability Set vs Availability Zone
| Feature | Availability Set | Availability Zone |
|---------|-----------------|-------------------|
| **Concept** | Group of VMs across fault/update domains | Physically separate datacenter within region |
| **Protection** | Platform failure (hardware, host) | Datacenter-level failure |
| **SLA** | 99.95% (with 2+ VMs) | 99.99% per zone |
| **Zone types** | Fault Domain, Update Domain | Zone 1, Zone 2, Zone 3 (region-specific) |
| **Cost** | Free | VMs in zones incur cross-zone traffic costs |

### VNet/Subnet
```powershell
# VNet creation
New-AzVirtualNetwork -Name "MyVNet" -ResourceGroupName "MyRG" -Location "East US" `
  -AddressPrefix "10.0.0.0/16"

# Subnet
Add-AzVirtualNetworkSubnetConfig -Name "default" -VNet $vnet -AddressPrefix "10.0.1.0/24"
Set-AzVirtualNetwork -VirtualNetwork $vnet

# Peering (connect VNets)
New-AzVirtualNetworkPeering -Name "VNet1-to-VNet2" -RemoteVirtualNetworkId $vnet2.Id -VirtualNetwork $vnet1
```

### Route Table / UDR
| Concept | Description |
|---------|-------------|
| **Route Table** | Set of routing rules |
| **UDR (User Defined Route)** | Custom route overriding default Azure routes |
| **Next Hop Type** | Virtual Appliance, Virtual Network Gateway, Internet, VNet Local, None |
| **Association** | Route table can be associated with one or more subnets |

### Azure Firewall
- Managed, cloud-native network security service
- Fully stateful firewall
- Built-in high availability
- Threat intelligence (domain/IP/URL filtering)
- Hub-and-spoke architecture (typically in hub VNet)

### Private Endpoint
| Feature | Description |
|---------|-------------|
| **Purpose** | Connect to Azure PaaS (Storage, SQL, etc.) over private endpoint |
| **Network** | Uses Azure Private Link |
| **IP** | Private IP from VNet subnet |
| **DNS** | Requires DNS private zone integration |




###Azure DNS

### Azure DNS
- Hosted DNS service in Azure
- **Public DNS**: Publicly resolvable zones
- **Private DNS**: Private zones within VNet (resolvable only within VNet/peered VNets)
- Integration: VNet delegation for private DNS resolution

```powershell
# Azure DNS
New-AzDnsZone -Name "contoso.com" -ResourceGroupName "MyRG"
New-AzDnsRecordSet -Name "server1" -RecordType A -ZoneName "contoso.com" -ResourceGroupName "MyRG" -Ttl 3600 -DnsRecords (New-AzDnsRecordConfig -IPv4Address "10.0.1.4")
Get-AzDnsRecordSet -ZoneName "contoso.com" -ResourceGroupName "MyRG" -Name "server1"
```

### DNS Troubleshooting (Azure)
```powershell
# Check DNS resolution in Azure
Resolve-DnsName server1.contoso.com
nslookup server1.contoso.com

# Check VNet DNS settings
Get-AzVirtualNetwork -Name "MyVNet" -ResourceGroupName "MyRG" | Select-Object -ExpandProperty DhcpOptions | Select-Object DnsServers

# Enable Azure DNS Private Resolver (for on-prem connectivity)
New-AzDnsResolver -Name "MyResolver" -ResourceGroupName "MyRG" -Location "East US" -InboundEndpoint "/subscriptions/.../resourceGroups/MyRG/providers/Microsoft.Network/dnsResolvers/inboundEndpoints/MyInbound" -OutboundEndpoint "/subscriptions/.../resourceGroups/MyRG/providers/Microsoft.Network/dnsResolvers/outboundEndpoints/MyOutbound"
```





---

## PART 28 — Azure + On-Prem Hybrid


###Azure Arc

### Azure Arc
- Unified control plane for managing Windows/Linux servers, Kubernetes clusters, and Azure resources across hybrid environments
- Enables Azure services (Update Management, Security, Monitoring) on non-Azure machines
- Part of Azure Stack Edge / Azure services for on-premises

### Hybrid Servers
- On-premises servers registered with Azure Arc
- Appear in Azure Portal under "Arc" resource group
- Managed like Azure VMs (in many respects)
- Run Azure extensions (IaaSDiagnostics, AzureMonitor, etc.)

### Arc-Enabled Server
```powershell
# Azure Arc registration (via Azure PowerShell)
Connect-AzAccount
New-AzConnectedMachine -Name "MyServer" -ResourceGroupName "ArcRG" -Location "East US" `
  -IdentityType "SystemAssigned" `
  -GatewayResourceId "/subscriptions/.../resourceGroups/ArcRG/providers/Microsoft.HybridCompute/gateways/MyGateway" `
  -Proxy "http://proxy:8080"    (if using proxy)

# Or via Azure CLI
az connectedmachine create -g ArcRG -n MyServer -l East US --identity-system-assigned
```

### Arc Components
| Component | Description |
|-----------|-------------|
| **Agent** | Software installed on server (Microsoft.Azure.RecoveryAgent / HybridConnection) |
| **Azure Connected Machine** | The registered server in Azure |
| **Extensions** | Azure services that run on the server (Monitoring, Update Management, etc.) |
| **Identity** | System-assigned or user-assigned managed identity |
| **Policy** | Azure Policy for compliance |
| **Inventory** | Hardware/software inventory |
| **Update Management** | Patch management (WUA-based) |
| **Monitoring** | Azure Monitor agent for logs/metrics |
| **Governance** | Azure Policy compliance |
| **Tags** | Organize and group resources |




###Hybrid Services

### Azure Monitor + Log Analytics
| Feature | Description |
|---------|-------------|
| **Log Analytics Workspace** | Central repository for logs |
| **Azure Monitor Agent (AMA)** | New agent (replaces MMA) for collecting telemetry |
| **Microsoft Monitoring Agent (MMA)** | Legacy agent |
| **Custom logs** | Ingest custom log files |
| **KQL (Kusto Query Language)** | Query language for log data |
| **Azure Monitor for VMs** | Pre-built VM monitoring solutions |
| **Azure Monitor for SQL** | SQL Server monitoring |

```powershell
# On-premises monitoring with Log Analytics
# MMA extension configuration via Azure Portal
# Or via KQL queries:

# Sample KQL queries
Heartbeat | where TimeGenerated > ago(1h) | summarize LastHeartbeat = max(TimeGenerated) by Computer
SecurityEvent | where EventID == 4625 | summarize count() by bin(TimeGenerated, 5m)
Performance | where CounterName == "% Free Space" | summarize avg(CounterValue) by bin(TimeGenerated, 1h), Computer

# Log Analytics Workspace
New-AzLogAnalyticsWorkspace -ResourceGroupName "MonitorRG" -Name "LogAnalyticsRG" -Location "East US" -Sku PerGB2018
```

### Azure Update Manager (Azure Arc)
| Feature | Description |
|---------|-------------|
| **Agent-based** | Uses WUA (Windows Update Agent) on Windows; yum/apt on Linux |
| **Update classifications** | Critical, Security, UpdateRollup, FeaturePack, Definition, Updates |
| **Maintenance windows** | Schedule deployments |
| **Update baselines** | Scope of updates to deploy |
| **Deployment** | Per-machine or per-group |
| **Reboot control** | Configure reboot behavior |

### Defender for Cloud
| Feature | Description |
|---------|-------------|
| **Unified security** | Multi-cloud security management |
| **VM scanning** | Vulnerability assessment for VMs/Arc servers |
| **Just-In-Time (JIT)** | Restrict RDP/SSH access to approved IP/time |
| **Adaptive network hardening** | Alert on suspicious network traffic |
| **File integrity monitoring** | Track changes to critical files |
| **Attack path analysis** | Identify potential attack paths |
| **Defender for Servers** | Per VM or Arc-enabled machine |
| **Defender for Kubernetes** | Container security |
| **Defender for IoT** | IoT security |

### Azure Backup
| Feature | Description |
|---------|-------------|
| **Azure Backup** | Native Azure backup service |
| **Backup vault** | Container for backup items |
| **Backup policy** | Schedule + retention |
| **Recovery Services vault** | For Azure IaaS VM backup |
| **MAB (Microsoft Azure Backup)** | Backup agent for on-premises (for DPM/Log Analytics) |
| **Arc-enabled servers** | Backup via Azure Backup |

### Azure Site Recovery (ASR)
| Feature | Description |
|---------|-------------|
| **Purpose** | DR orchestration for Azure + on-premises |
| **Replicates** | Hyper-V VMs, VMware VMs, physical servers |
| **Target** | Azure or secondary on-premises site |
| **RPO** | Typically 5 minutes (VMware), near-zero (Hyper-V) |
| **Failover** | Planned and unplanned |
| **Components** | Master target server, Process Server, Mobility Service |

### Hybrid Identity
| Concept | Description |
|---------|-------------|
| **Azure AD (Entra ID)** | Cloud identity provider |
| **Azure AD Connect** | Sync on-prem AD with Entra ID |
| **Password Hash Sync** | Syncs password hashes |
| **Pass-through Authentication** | Validates passwords on-premises |
| **Seamless SSO** | Users sign in with on-prem credentials in cloud |
| **Federation (ADFS)** | Active Directory Federation Services |
| **Hybrid Join** | Device registered in both Azure AD and on-prem AD |
| **Azure AD Join** | Device registered only in Azure AD |

### VPN & ExpressRoute
| Feature | VPN Gateway | ExpressRoute |
|---------|------------|--------------|
| **Type** | Site-to-site or point-to-site | Dedicated private connection |
| **Protocol** | IPsec/IKEv2 | MPLS (provider circuits) |
| **Encryption** | AES-256 | Private (not encrypted by default) |
| **Cost** | Lower (gateway + bandwidth) | Higher (circuit + gateway) |
| **Reliability** | Internet-dependent (ISP) | Provider SLA |
| **Use case** | Site-to-site backup, low traffic | Production workloads, high bandwidth |
| **BGP** | Supported | Supported |




---

## PART 29 — Azure Arc (Deep Dive)

> *Understand Arc-specific troubleshooting and operations.*


###Azure Arc Deep Dive

### Arc-Enabled Server Components
```
┌─────────────────┐     ┌──────────────────┐
│ Arc-enabled VM  │     │ Arc-enabled      │
│ (Azure VM)      │     │ on-prem server   │
│ Agent:          │────→│ Agent:           │
│ Azure Monitor  │     │ Hybrid           │
│ Agent (AMA)     │     │ Connected Machine│
│                 │     │ Agent (MCACore)  │
└────────┬────────┘     └────────┬─────────┘
         │                       │
         ↓                       ↓
    ┌──────────────────────────────────┐
    │ Azure Arc (control plane)        │
    │ Arc Resource Graph               │
    │ Arc Policy                       │
    │ Arc Inventory                    │
    │ Arc Update Management            │
    │ Arc Governance                   │
    └──────────────────────────────────┘
```

### Agent & Connectivity
| Component | Description |
|-----------|-------------|
| **Connected Machine Agent (MCACore.exe)** | Arc agent on Windows |
| **Connected Machine Agent (.rpm/.deb)** | Arc agent on Linux |
| **Azure Monitor Agent (AMA)** | Telemetry collection agent |
| **Azure Policy for Arc** | Compliance evaluation agent |
| **Azure Update Management Agent** | WUA-based update agent (Windows); yum/apt (Linux) |
| **Communication** | HTTPS (443) to Azure public endpoint or Azure China/Gov endpoints |
| **Proxy support** | Arc agent supports proxy configuration |

### Extensions
| Extension | Purpose |
|-----------|---------|
| **IaasMonitor** | Infrastructure monitoring |
| **AzureMonitorLinuxAgent / AzureMonitorWindowsAgent** | Azure Monitor agent |
| **AzureHotfixManagement** | Hotfix management |
| **AzureUpdateManagerWindows** | Windows Update management |
| **AzureUpdateManagerLinux** | Linux Update management |
| **AzurePolicyForArc** | Policy compliance |
| **Microsoft.Arc.Rde.Runtime** | RDE (Run Command) |
| **OSConfigurationWindows** | DSC/OS config |

### Troubleshooting
| Issue | Diagnosis | Resolution |
|-------|-----------|-----------|
| **Agent disconnected** | Check connectivity, agent version, Azure connectivity | Re-register, check connectivity, update agent |
| **Connectivity issues** | Test connection to Azure endpoints | Check proxy, firewall, DNS, network path |
| **Proxy issues** | Arc agent may need proxy config | Configure proxy in `C:\Program Files\Microsoft Azure Arc\connectedmachineagent\config` |
| **Certificate issues** | Arc uses Azure certificates | Check certificate expiry, renew/re-register |
| **Identity issues** | Check managed identity | Verify identity exists, has correct permissions |
| **Extension failure** | Check extension status in Portal | Restart agent, re-register, update extension |

### Arc Troubleshooting Commands
```powershell
# Check Arc machine status
Get-AzConnectedMachine -ResourceGroupName "ArcRG" -Name "MyServer" | 
  Select-Object Name, Status, OperatingSystem, Extensions

# Check agent connectivity (on-prem machine)
Get-EventLog -LogName "Application" -Source "Azure Arc" -Newest 50
Get-EventLog -LogName "Application" -Source "Microsoft Azure" -Newest 50

# Common Arc agent paths
# Windows: C:\Program Files\Microsoft Azure Arc\
# Linux: /opt/microsoft/arc/

# Check Arc agent logs
# Windows: C:\ProgramData\Microsoft\AzureArc\Logs\
# Linux: /var/opt/microsoft/arc/logs/

# Check Hybrid Connection relay (if used)
Get-AzHybridConnection -ResourceGroupName "ArcRG" | Select-Object Name, Status, SentBytes, ReceivedBytes
```

### Arc Registration/Re-registration
```powershell
# Unregister (cleanup)
Get-AzConnectedMachine -Name "MyServer" -ResourceGroupName "ArcRG" | Remove-AzConnectedMachine

# Re-register
New-AzConnectedMachine -Name "MyServer" -ResourceGroupName "ArcRG" -Location "East US" -IdentityType SystemAssigned

# Update agent
# Download latest agent from:
# https://docs.microsoft.com/en-us/azure/azure-arc/agents-windows
# Or use Azure portal: Connected Machine → Extensions → Update
```

### Arc Inventory
- Hardware inventory: CPU, memory, disks, network adapters
- Software inventory: Installed programs, Windows updates
- OS information: OS version, build, registration state
- Network adapters: IP addresses, MAC addresses
- Storage: Disk layout, file systems
- Azure Arc inventory → Azure Portal → Connected Machine → Inventory

### Arc Policy (Governance)
```powershell
# Check compliance
Get-AzPolicyState -Scope "/subscriptions/.../resourceGroups/ArcRG/providers/Microsoft.HybridCompute/machines/MyServer"

# Assign policy
New-AzPolicyAssignment -Name "AuditArcVersion" -PolicyDefinition "/subscriptions/.../policyDefinitions/..." -Scope "/subscriptions/.../resourceGroups/ArcRG"
```

### Arc Update Management
```powershell
# Check update status
Get-AzVM -Name "MyVM" -ResourceGroupName "MyRG" | Select-Object -ExpandProperty Extensions | Where-Object {$_.ExtensionType -eq "AzureUpdateManagerWindows"}

# Or via Arc portal: Connected Machine → Update Management → Updates

# Update classifications (Windows)
Get-WindowsUpdate (PSWindowsUpdate module)
Get-HotFix (all installed)

# Or via Arc (Azure Portal)
# Connected Machine → Update Management → Scan → Deploy
```





---

## PART 30 — Terraform / Ansible / CI/CD

> *Secondary but useful for Lead level.*


###Terraform

### Core Concepts
| Term | Description |
|------|-------------|
| **Provider** | Plugin that Terraform uses to manage resources (azurerm, aws, vmware, kubernetes) |
| **Resource** | Infrastructure component (azurerm_virtual_machine, aws_instance) |
| **Variable** | Input parameter (passed via tfvars, env var, or CLI) |
| **Output** | Value returned after apply (IP, hostname, etc.) |
| **Module** | Reusable Terraform code (local or remote) |
| **State** | Maps infrastructure to resources (terraform.tfstate) |
| **Plan** | Preview of changes |
| **Apply** | Execute planned changes |
| **Destroy** | Remove all infrastructure |
| **for_each** | Iterate over a map/set |
| **count** | Iterate over a list (numeric index) |

### Basic Terraform
```hcl
# main.tf
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

variable "rg_name" {
  type    = string
  default = "MyRG"
}

variable "location" {
  type    = string
  default = "East US"
}

resource "azurerm_resource_group" "main" {
  name     = var.rg_name
  location = var.location
}

resource "azurerm_virtual_network" "main" {
  name                = "MyVNet"
  address_space       = ["10.0.0.0/16"]
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_subnet" "internal" {
  name                 = "internal"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}

# Output
output "vnet_id" {
  value = azurerm_virtual_network.main.id
}
```

### Terraform Commands
```bash
# Initialize (download providers, modules)
terraform init
terraform init -backend-config="key=terraform.tfstate"

# Format
terraform fmt

# Validate
terraform validate

# Plan (preview changes)
terraform plan -out=tfplan
terraform plan -var-file="dev.tfvars"

# Apply
terraform apply
terraform apply tfplan

# Destroy
terraform destroy

# Show state
terraform show
terraform state list
terraform state show azurerm_resource_group.main

# Import existing resource
terraform import azurerm_resource_group.main/MyRG
```

### Iteration
```hcl
# for_each
variable "vm_names" {
  type = map(string)
  default = {
    web1  = "Web1"
    web2  = "Web2"
    db1   = "DB1"
  }
}

resource "azurerm_virtual_machine" "example" {
  for_each = var.vm_names
  name     = each.value
  # ...
}

# count
variable "instance_count" {
  type = number
  default = 3
}

resource "azurerm_virtual_machine" "example" {
  count = var.instance_count
  name  = "VM-${count.index}"
  # ...
}
```

### Terraform for VMware
```hcl
# VMware provider
terraform {
  required_providers {
    vmware = {
      source  = "hashicorp/vmware"
      version = "~> 2.0"
    }
  }
}

provider "vmware" {
  vsphere_server = "vcenter.domain.com"
  user           = "administrator@vsphere.local"
  password       = var.vcenter_password
  allow_unverified_ssl = true
}

resource "vmware_compute_cluster" "cluster" {
  name          = "MyCluster"
  datacenter_id = data.vmware_compute_datacenter.dc.id
}

resource "vmware_virtual_machine" "vm" {
  name          = "MyVM"
  resource_pool = vmware_compute_cluster.cluster.resource_pool_id
  datacenter_id = data.vmware_compute_datacenter.dc.id
  # ...
}
```




###Ansible

### Core Concepts
| Term | Description |
|------|-------------|
| **Inventory** | List of hosts/groups (hosts file or dynamic inventory) |
| **Playbook** | YAML file defining tasks to run on hosts |
| **Module** | Reusable unit of work (yum, apt, service, copy, template, command, shell, win_* ) |
| **Role** | Reusable collection of tasks, handlers, templates, variables |
| **Idempotency** | Running playbook multiple times produces same result |
| **YAML** | Ansible playbook format (key-value, indentation-sensitive) |

### Basic Playbook
```yaml
---
- name: Configure Web Servers
  hosts: webservers
  become: yes    # Run as root/admin
  
  vars:
    http_port: 80
    
  tasks:
    - name: Ensure IIS is installed
      win_feature:
        name: Web-Server
        state: present
      
    - name: Ensure IIS service is running
      win_service:
        name: W3SVC
        start_mode: auto
        state: started
      
    - name: Configure firewall rule
      win_firewall_rule:
        name: "Allow HTTP"
        display_name: "Allow HTTP"
        direction: In
        action: Allow
        protocol: TCP
        local_port: "{{ http_port }}"
        enable: yes
      
    - name: Create website directory
      file:
        path: C:\inetpub\wwwroot
        state: directory
        
    - name: Copy website files
      copy:
        src: /ansible/files/index.html
        dest: C:\inetpub\wwwroot\
        
  handlers:
    - name: Restart IIS
      win_service:
        name: W3SVC
        state: restarted
```

### Ansible Commands
```bash
# Check syntax
ansible-playbook site.yml --syntax-check

# Dry run
ansible-playbook site.yml --check

# Run with verbose
ansible-playbook site.yml -vvv

# Limit to specific hosts
ansible-playbook site.yml --limit "web*"

# Run specific tags
ansible-playbook site.yml --tags "install,configure"

# Ad-hoc command
ansible webservers -m win_ping
ansible webservers -m win_command -a "ipconfig"

# List inventory
ansible-inventory --list

# Ping all hosts
ansible all -m win_ping
```

### Inventory Examples
```ini
# /etc/ansible/hosts
[webservers]
web1.example.com
web2.example.com

[dbservers]
db1.example.com

[all:vars]
ansible_user=admin
ansible_ssh_private_key_file=~/.ssh/id_rsa

[webservers:vars]
http_port=80
```

### Roles Structure
```
roles/
  common/
    tasks/
      main.yml
    handlers/
      main.yml
    templates/
      nginx.conf.j2
    files/
      index.html
    vars/
      main.yml
    defaults/
      main.yml
    meta/
      main.yml
```

### Windows Ansible Modules
| Module | Purpose |
|--------|---------|
| **win_feature** | Install Windows features |
| **win_service** | Manage Windows services |
| **win_firewall_rule** | Manage firewall rules |
| **win_package** | Install software (MSI/EXE) |
| **win_copy** | Copy files (Windows) |
| **win_template** | Template file with variables |
| **win_command/shell** | Execute commands |
| **win_ping** | Test connectivity |
| **win_user** | Manage users |
| **win_regedit** | Manage registry |
| **win_chocolatey** | Package management |
| **win_execution_policy** | Set execution policy |
| **win_domain** | Domain join |

### Idempotency
- Running playbook multiple times produces same result
- Most Ansible modules are idempotent by design
- **win_command** and **win_shell** are NOT idempotent
- Use specific modules (win_package, win_service) for idempotent operations
- Use `changed_when` and `failed_when` to define custom triggers




###CI/CD (Git, Pipelines)

### Core Concepts
| Term | Description |
|------|-------------|
| **Git** | Distributed version control system |
| **Repository** | Project code/configuration storage |
| **Branch** | Line of development |
| **Pull Request (PR)** | Request to merge changes from one branch to another |
| **Pipeline** | Automated build/test/deploy process |
| **Validation** | Pre-merge checks (linting, tests, security scans) |
| **Deployment** | Actual release to environment |

### Git Workflow
```
main (stable)
  ↑
  ├── feature/TICKET-123 (feature branch)
  │     ↓
  │   Push → PR → Review → Validation → Merge to main
  │
  └── hotfix/TICKET-456 (hotfix branch)
        ↓
      Push → PR → Review → Merge to main
```

### Git Commands
```bash
# Clone
git clone https://github.com/org/repo.git
cd repo

# Branch operations
git checkout -b feature/TICKET-123
git checkout main
git merge feature/TICKET-123

# Commit & push
git add .
git commit -m "TICKET-123: Add Terraform config"
git push origin feature/TICKET-123

# Pull request
# Use GitHub/GitLab/Azure DevOps UI

# Check status
git status
git log --oneline
git diff main..feature/TICKET-123

# Pull latest changes
git pull origin main
git rebase main    (rebase current branch onto main)
```

### Pipeline Concepts (Azure DevOps / GitHub Actions / Jenkins)
```yaml
# Azure DevOps Pipeline (azure-pipelines.yml)
trigger:
  branches:
    include:
      - main
      - feature/*

stages:
  - stage: Validate
    jobs:
      - job: TerraformValidation
        steps:
          - task: TerraformInstaller@1
            inputs:
              terraformVersion: '1.0.0'
          - script: terraform init
          - script: terraform validate
          - script: terraform plan -out=tfplan
  
  - stage: DeployDev
    dependsOn: Validate
    jobs:
      - deployment: DeployTerraform
        environment: 'dev'
        strategy:
          runOnce:
            deploy:
              steps:
                - script: terraform apply tfplan
  
  - stage: DeployProd
    dependsOn: DeployDev
    # Manual approval required
```

### Pipeline Stages
| Stage | Purpose |
|-------|---------|
| **Source** | Get code from repository |
| **Build/Validate** | Compile, lint, validate (terraform validate, ansible-lint) |
| **Test** | Run unit/integration tests |
| **Security** | Vulnerability scan, secret scan |
| **Deploy to Dev/Test** | Deploy to non-production |
| **Validate** | Automated tests, manual checks |
| **Deploy to Staging** | Production-like environment |
| **Approve** | Manual approval gate |
| **Deploy to Prod** | Production deployment |
| **Post-deploy** | Smoke tests, health checks |

### DevOps Tools Comparison
| Tool | Type | Key Feature |
|------|------|------------|
| **Azure DevOps** | MS platform | Full pipeline, repos, boards, artifacts |
| **GitHub Actions** | MS/Git platform | YAML-based, GitHub integrated |
| **Jenkins** | Open-source | Highly extensible, self-hosted |
| **GitLab CI** | Git platform | Integrated with GitLab |
| **Ansible Tower/AWX** | Ansible | Ansible pipeline management |





---

## PART 31 — Incident Management

> *L3 support and major-incident leadership.*


###Incident Management Concepts

### Definitions
| Term | Description |
|------|-------------|
| **Incident** | Unplanned interruption or degradation of IT service |
| **Major Incident** | Significant business impact, requires dedicated management |
| **Problem** | Root cause of one or more incidents |
| **Request** | User request for information, access, or standard change |
| **SLA** | Service Level Agreement (defined uptime, response times) |

### Priority vs Severity
| Priority | Description | Example |
|----------|-------------|---------|
| **P1 - Critical** | Complete business service down, multiple users affected, revenue impact | DC down, all AD auth failing |
| **P2 - High** | Significant degradation, workarounds available | Single DC down, non-priority server down |
| **P3 - Medium** | Non-critical service impacted, workaround available | Non-essential server slow |
| **P4 - Low** | Minor issue, no work disruption | Cosmetic issue, informational alert |

| Severity | Description |
|----------|-------------|
| **SEV 1** | Business critical, all services unavailable, no workaround |
| **SEV 2** | Major service degraded, limited workaround |
| **SEV 3** | Service impacted, workaround available, non-critical |
| **SEV 4** | Minor issue, information request |

### Priority/Severity Matrix
| | SEV 1 | SEV 2 | SEV 3 | SEV 4 |
|---|---|---|---|---|
| **P1** | Max resources, immediate | Max resources | Escalate | Escalate |
| **P2** | Rapid response | Escalate quickly | Standard | Standard |
| **P3** | Standard | Standard | Standard | Standard |
| **P4** | Low priority | Low priority | Low priority | Resolve when convenient |

### Incident Response Roles
| Role | Responsibility |
|------|---------------|
| **Incident Commander** | Overall incident management, decisions, escalation |
| **Technical Lead** | Technical investigation, resolution direction |
| **Communication Lead** | Stakeholder updates, status reports |
| **Support Staff** | Execute technical tasks, gather information |

### SLA Targets (Common Examples)
| Priority | Response Time | Resolution Time |
|----------|--------------|-----------------|
| **P1** | 15 minutes | 4 hours |
| **P2** | 1 hour | 8 hours |
| **P3** | 4 hours | 24 hours |
| **P4** | 8 hours | 72 hours |

### Escalation Path
```
L1 → L2 → L3 → Vendor/Architecture Team
        ↓
     Major Incident (P1/P2)
        ↓
     Incident Commander + Technical Lead + Communication Lead
        ↓
     Stakeholder Updates (every 30-60 min for P1)
```

### Bridge Call
- Conference call for major incidents
- Participants: Incident Commander, Tech Lead, support staff, stakeholders
- Regular updates (every 30 min for P1, hourly for P2)
- Document all actions, decisions, and communications
- End with clear next steps and owner for each action

### Stakeholder Communication
| Audience | Content | Frequency |
|----------|---------|-----------|
| **Technical team** | Technical details, action items, logs | Continuous |
| **Management** | Impact, resolution ETA, status | Every 30-60 min (P1) / hourly (P2) |
| **End users** | Service impact, workarounds, status | As needed / status page |
| **Vendors** | Technical details, logs, escalation | As needed |




---

## PART 32 — RCA / Problem Management

> *Root cause analysis — for complex enterprise issues.*


###RCA Structure

### RCA Template
```
1. Incident
   → What incident? (ID, date/time, service affected)

2. Impact
   → Business impact, users affected, duration, SLA breach?

3. Timeline
   → When did it start? When was it detected? 
   → When was work started? When was it resolved?

4. Symptoms
   → Observable symptoms (alerts, errors, user reports)

5. Investigation
   → Steps taken to investigate
   → Logs reviewed, commands run

6. Root Cause
   → Primary cause (single root cause preferred)

7. Contributing Factors
   → Conditions that enabled the incident (secondary factors)

8. Resolution
   → How was it fixed? (immediate fix)

9. Corrective Action
   → How to prevent recurrence (short-term)

10. Preventive Action
    → How to prevent related issues (long-term)
```

### Example RCA Document
```
INCIDENT: PRB-2024-001 — AD authentication failure
DATE: 2024-01-15 08:30 UTC
IMPACT: All authentication failures; 500 users unable to log in; P1; SLA breached

TIMELINE:
  08:30 - Users report login failures
  08:45 - L2 escalates to L3
  09:00 - Investigation started
  09:15 - Found: DC01 DNS resolution failing
  09:30 - Found: DNS server crashed due to memory pressure
  10:00 - Restarted DNS Server service
  10:15 - Authentication restored
  10:30 - RCA document started

SYMPTOMS:
  - Event ID 4625 in Security log (failed logons)
  - DNS resolution timeout for domain controllers
  - NTLM fallback in authentication logs
  - Kerberos failures: clock skew event 4771

INVESTSTIGATION:
  1. Checked AD authentication logs → Multiple 4625 events
  2. Checked Kerberos events → 4771 (pre-auth failures)
  3. Checked DNS → Slow/unavailable resolution
  4. Checked DNS server → DNS Server service stopped
  5. Checked DNS server memory → Low available memory (1GB of 32GB)
  6. Checked DNS server memory pattern → Steady growth over 30 days

ROOT CAUSE:
  DNS Server service stopped due to memory exhaustion on DC01
  (Memory leak in DNS Server due to record bloat in DNS cache)

CONTRIBUTING FACTORS:
  - DNS scavenging not configured (stale records accumulated)
  - DNS cache size unlimited (default)
  - No memory alerting threshold
  - DNS server running on DC (combined role)

RESOLUTION:
  Restarted DNS Server service; cleared DNS cache; verified DCs authenticate

CORRECTIVE ACTION:
  - Configure DNS scavenging on all zones
  - Set DNS cache size limit
  - Configure memory alerts at 75% threshold
  - Move DNS Server role to dedicated server if possible

PREVENTIVE ACTION:
  - Implement monitoring for DNS server memory usage
  - Add DNS scavenging cleanup automation
  - Review all DCs for combined roles
  - DNS server health dashboard in monitoring
```

### RCA Techniques
| Technique | Description | When to Use |
|-----------|-------------|-------------|
| **5 Whys** | Ask "why?" 5 times to reach root cause | Simple causal chains |
| **Fishbone (Ishikawa)** | Categorize causes (Man, Machine, Method, Material, Environment, Measurement) | Complex, multi-factor |
| **Timeline Analysis** | Build timeline of events, identify when issue started | Incidents with clear start |
| **Dependency Analysis** | Map dependencies between components | Distributed systems |
| **Change Correlation** | Check recent changes near time of incident | Post-change failures |

### 5 Whys Example
```
Problem: Users cannot log in to domain
  Why 1: Why? → Kerberos authentication failing (Event ID 4771)
  Why 2: Why? → Clock skew between workstation and DC (>5 min)
  Why 3: Why? → Time service not synchronizing
  Why 4: Why? → w32time service not running
  Why 5: Why? → Service stopped after Windows Update
  ROOT CAUSE: Windows Update changed time service config; service didn't restart properly
  CORRECTIVE: Patch validation should include service state verification
```

### Fishbone (Ishikawa) Categories
- **Man**: Human error, training gap, procedure gap
- **Machine**: Hardware failure, driver issue, firmware bug
- **Method**: Wrong procedure, undocumented step, missing process
- **Material**: Bad data, corrupt file, bad config
- **Environment**: Power issue, temperature, network disruption
- **Measurement**: Monitoring gap, alert threshold wrong, log missing




---

## PART 33 — Change Management

> *The JD specifically calls for risk assessment, testing, rollback, and stakeholder communication.*


###Change Management Concepts

### Change Types
| Type | Description | Approval | Examples |
|------|-------------|----------|---------|
| **Normal Change** | Requires full CAB approval | Pre-authorized | New server deployment, major software update |
| **Standard Change** | Pre-authorized, low risk | Auto-approved | Password reset, add monitoring agent, printer install |
| **Emergency Change** | Emergency, implemented immediately | Post-implementation review | Emergency patch, critical fix |

### Change Process
```
Pre-check
  ↓
Risk Assessment
  ↓
Impact Assessment
  ↓
Implementation Plan
  ↓
Test/Validate (if time permits)
  ↓
CAB Review / Approval
  ↓
Communication (stakeholders)
  ↓
Implementation
  ↓
Post-check (validation)
  ↓
Rollback (if failed)
  ↓
Communication (results)
```

### Risk Assessment Template
| Factor | Assessment | Mitigation |
|--------|-----------|-----------|
| **Impact if change fails** | High/Medium/Low | Rollback plan, backup, test in isolated environment |
| **Probability of failure** | High/Medium/Low | Test thoroughly, have experienced staff |
| **Scope** | How many systems affected | Phased rollout, ring deployment |
| **Dependencies** | What other systems/services are affected | Identify all dependencies, test each |
| **Time window** | How long implementation takes | Schedule during maintenance window |
| **Rollback complexity** | Easy/Medium/Hard | Document rollback steps in advance |

### Impact Assessment
```
Identify affected systems:
  ↓
Determine blast radius:
  ↓
   How many users affected?
  ↓
   What services impacted?
  ↓
   What is the recovery plan?
  ↓
   What is the rollback plan?
```

### Pre-Check
| Item | Check |
|------|-------|
| **Documentation** | Current state documented |
| **Backup** | Current configuration/data backed up |
| **Dependencies** | All dependencies identified |
| **Permissions** | Staff have appropriate access |
| **Maintenance window** | Approved, communicated |
| **Resources** | Required personnel, tools, access available |
| **Testing** | Change tested in isolated environment |

### Post-Check
| Item | Check |
|------|-------|
| **Change successful** | All objectives met |
| **System functioning** | Services operational, users functional |
| **Monitoring** | Alerts configured and no unexpected alerts |
| **Documentation** | Updated (runbook, diagram, CMDB) |
| **Backout plan** | Validated or no longer needed |

### Rollback Planning
```
IF change fails:
  1. Stop implementation
  2. Assess impact
  3. Execute rollback steps:
     a. Restore backup
     b. Revert configuration
     c. Restart services
  4. Validate system functionality
  5. Notify stakeholders
  6. Investigate root cause of failure
  7. Update change record

Rollback time estimate = same as implementation time (plan accordingly)
```

### Communication
| Audience | Content | Timing |
|----------|---------|--------|
| **Stakeholders** | Impact, timing, risks | Before change, during, after |
| **IT Team** | Technical details, implementation steps, rollback steps | Before change |
| **End Users** | Service impact, workarounds, expected duration | Before change, during |
| **Management** | Change summary, risks, SLA impact | Before, during (major), after |
| **Vendors** | Technical details, escalation path | Before change (if involved) |

### CAB (Change Advisory Board)
| Member | Role |
|--------|------|
| **IT Operations** | Operational feasibility, resource availability |
| **Security Team** | Security review, compliance |
| **Network Team** | Network impact, connectivity |
| **Application Team** | Application impact, compatibility |
| **Business Representative** | Business impact, priority |
| **Change Manager** | Process owner, documentation |
| **Incident Manager** | Impact on current incidents |




---

## PART 34 — Cross-Team Troubleshooting

> *Demonstrate that you know where the problem actually belongs.*


###Cross-Team Troubleshooting Map

### The Troubleshooting Web
```
Windows
   ↕
AD
   ↕
DNS
   ↕
Network
   ↕
Firewall
   ↕
Storage
   ↕
VMware
   ↕
Backup
   ↕
Azure
   ↕
Application
```

### Example Scenario: "VM is Slow"
**Key Decision Point: Where does the problem belong?**

```
VM is slow
  ↓
Is it slow INSIDE the VM (OS-level)?
├── YES → Application issue OR Windows issue
│         ↓
│    Check inside VM:
│      - CPU (Task Manager / Get-Process)
│      - Memory (Task Manager / Get-Counter)
│      - Disk inside VM (Resource Monitor)
│      - Network inside VM (netstat)
│      - Application logs
│
└── NO (VM hardware performing well) → VM/ESXi issue
      ↓
   Check VM metrics:
      - CPU Ready (>5%?) → ESXi CPU contention
      - Memory Ballooning (high?) → Memory contention
      - Disk latency (high?) → Storage issue
      - Network latency (high?) → Network issue
      ↓
   Check ESXi Host:
      - Host CPU/memory/disk/network
      - Other VMs on same host also slow?
      - ESXi version/patches
      ↓
   Check Storage:
      - Datastore latency
      - Storage IOPS limits
      - Multipathing
      - APD/PDL events
      ↓
   Check Network:
      - VMkernel network
      - vSwitch configuration
      - Physical NIC status
      - Storage network
```

### Root Cause Decision Tree
```
Problem: Application slow

Q1: Is application issue reproducible?
  YES → Application code / configuration (contact app team)
  NO → Continue ↓

Q2: Is all application on this server affected or just one tenant?
  ALL → OS/VM/Storage issue
  ONE → Application configuration / tenant-specific issue

Q3: Did recent changes happen?
  YES → Change correlation (review change log)
  NO → Continue ↓

Q4: Is Windows event log showing errors?
  YES → Follow Event Viewer troubleshooting
  NO → Continue ↓

Q5: Check network connectivity
  YES → DNS/Network/Application issue
  NO → Network/Firewall issue

Q6: Storage latency check
  HIGH → Storage issue
  NORMAL → VM/ESXi/Application issue
```

### Common Misattributions
| Mistake | Correct Approach |
|---------|-----------------|
| **"VM is slow, so VMware problem"** | Check Application → Windows first; VM may be fine, application inside VM may be causing it |
| **"Network is slow, so switch problem"** | Check DNS first, then network, then application; DNS failure can appear as network issue |
| **"AD login failed, so AD problem"** | Check DNS, then time sync, then secure channel, then AD (often not AD itself) |
| **"Backup failed, so storage problem"** | Check backup service/agent, then network, then storage (often software/agent issue) |
| **"Azure VM unreachable, so Azure problem"** | Check NSG, DNS, application, then Azure (often configuration, not platform) |




---

## PART 35 — L3 Troubleshooting Scenarios (Selected Key Scenarios)

> *Step-by-step troubleshooting for at least 30+ scenarios.*


###Windows Scenarios (Selected)

### Server Won't Boot
```
1. Observe behavior (BSOD, black screen, stuck at logo)
2. Check Event ID 41 in System log (unexpected shutdown)
3. Boot into Safe Mode or DSRM
   - DSRM: Press F8 at boot → Directory Services Restore Mode
   - Safe Mode: F8 → Safe Mode
4. In DSRM:
   - Test-ComputerSecureChannel -Repair
   - Check disk: chkdsk /f
   - Restore from backup if needed
5. Check disk space on system drive (C:\)
6. Boot with Last Known Good Configuration
7. Check BSOD error code
   - Use WinDbg / !analyze -v
   - Check C:\Windows\Minidump\*.dmp
   - Common causes: driver (.sys files in dump), corrupt system files
8. System File Checker: sfc /scannow
9. DISM repair: DISM /Online /Cleanup-Image /RestoreHealth
10. Boot to recovery environment → Startup Repair
```

### Server Extremely Slow
```
1. Check Resource Monitor / Task Manager inside VM
   - CPU, Memory, Disk, Network tabs
2. Get-Process | Sort-Object CPU -Descending | Select-Object -First 10
3. Get-Counter '\Processor(_Total)\%Processor Time', '\Memory\Available MBytes', '\PhysicalDisk(0)\Avg. Disk sec/Read'
4. Check if VSS writers are stuck (vssadmin list writers)
5. Check disk space (C:\)
6. Check for running backups (if backup running, this is normal)
7. Check Windows Update status (WaaSMedicSvc, usocheck)
8. Check Windows Defender scanning
9. Check for malware (Windows Defender scan)
10. Check for NTLM/LDAP timeout (network issue causing delays)
```

### High CPU
```
1. Identify process (Get-Process, Task Manager)
2. If system process (100%): typically DPC/ISR (driver issue)
   - Check: KeBugCheck, DPC Latency Checker
3. If specific process: analyze further
   - Check if process is legitimate (path, signature)
   - Check if recently installed/updated
   - Kill process (with approval): Stop-Process -Id PID -Force
4. Check CPU queue length
5. If VM: Check CPU Ready (VMware) / Co-stop (VMware)
6. If ESXi host: Check DRS balance, EVC mode, host load
7. Check for DDoS/brute force (many connections)
8. Check for scheduled tasks running at time of high CPU
```

### Disk 100% (100% Disk Utilization)
```
1. Check Resource Monitor → Disk tab
2. Identify process/response time for each disk activity
3. Check for:
   - Antivirus scanning (Windows Defender)
   - Backup running
   - Windows Update/WSUS processing
   - Indexing (Windows Search)
   - FRS/DFSR (AD replication)
   - WsusCtrl processing
4. Check disk queue length and latency
5. Check free space
6. If VM: Check VMware disk latency (esxtop)
7. If ESXi: Check storage latency, multipathing, APD/PDL
8. Add disk or move workload to faster storage
9. Disable unnecessary services (search indexing, etc.)
```

### RDP Unavailable
```
1. Check RDP service: Get-Service RDP-Tcp
2. Check firewall: Get-NetFirewallRule -DisplayName "*Remote Desktop*"
3. Test port: Test-NetConnection -ComputerName SERVER -Port 3389
4. Check RDP enabled in System Properties (SystemPropertiesRemote)
5. Check network level authentication (NLA)
6. Check max sessions: Win32_TerminalServiceSetting
7. Check certificate: RD Gateway certificate
8. Check if RDP-Tcp listener is running: Query ProcessName=TermService
9. Check Event Viewer: RDP-ClientActiveX-related events
10. Check if Restricted Groups GPO is overriding admin group
```

### Blue Screen (BSOD)
```
1. Note bugcheck code from BSOD screen
2. Check minidumps: C:\Windows\Minidump\*.dmp
3. Analyze with WinDbg: !analyze -v
4. Common drivers in dumps:
   - ndis.sys, nbt.sys, ntfs.sys, storport.sys, dxgkrnl.sys
5. If driver crash:
   - Update/rollback driver
   - Check driver signing (unsigned driver?)
   - Check for known driver issues
6. If memory related:
   - Run Windows Memory Diagnostic (mdsched.exe)
   - Check RAM modules
7. If storage related:
   - Check disk health (chkdsk, SMART)
   - Check storage controller/driver
8. Event ID 41 = unexpected shutdown (not always BSOD)
```

### Service Keeps Stopping
```
1. Check service status: Get-Service -Name SVCNAME
2. Check service recovery options: Services GUI → Recovery tab
   - Configure: First/Second/Subsequent failure → Restart/Restart Service/Run Program/Restart Computer
3. Check Event Viewer: Service Control Manager events (7000, 7023, 7031, 7034)
4. Check service dependencies: sc qc SVCNAME
5. Run service manually: net start SVCNAME (check error message)
6. Check application event log for service-specific errors
7. Check service account password (if expired)
8. Check disk space (service needs room to write logs/temp)
9. Check service configuration files/registry
10. Check if service is conflicting with other service (port conflict, resource conflict)
```

### Disk Full
```
1. Identify what's consuming space:
   Get-ChildItem -Path C:\ -Recurse -Force -ErrorAction SilentlyContinue | 
     Sort-Object Length -Descending | 
     Select-Object -First 20 FullName, @{N="SizeMB";E={[math]::Round($_.Length/1MB,2)}}
2. Common space consumers:
   - C:\Windows\Temp
   - C:\Windows\Logs (especially CBS, WindowsUpdate, DIAGNOSTIC)
   - C:\Windows\SoftwareDistribution\Download
   - C:\Windows\System32\LogFiles
   - C:\inetpub\logs (IIS logs)
   - Application logs (SVChost, SQL, etc.)
   - User profile (Desktop, Downloads, AppData)
   - Pagefile.sys
   - Hibernation file (hiberfil.sys)
3. Clean:
   - Disk Cleanup (cleanmgr.exe /scdiskcleanup)
   - Delete temp files
   - Rotate/delete old logs
   - Clear Windows Update cache
4. Expand disk (VM: expand VHD, then extend partition in Windows)
5. Set up log rotation / retention policies
```

---

### AD Scenarios

### User Cannot Log In
```
1. Verify username spelling
2. Check if account is disabled: Get-ADUser -Identity username -Properties Enabled
3. Check if account is locked out: Search-ADAccount -LockedOut
4. Check password expiration: Get-ADUser -Identity username -Properties PasswordLastSet
5. Check password correct (try reset with approval)
6. Check time sync (client syncs with DC)
   - w32tm /query /status on client
   - If time >5 min skew: kerberos fails, NTLM also may fail
7. Check secure channel: Test-ComputerSecureChannel -Verbose
8. Check if DC is reachable: ping DC; nslookup DC; Test-NetConnection DC -Port 389
9. Check if user has correct OUs/GPOs applied
10. Check if "Deny log on locally" or similar GPO applied to user
11. Check Event 4625 (Security log) → sub-status:
    - 0xC0000133 = time skew (fix time sync)
    - 0xC000006A = bad password
    - 0xC0000193 = account expired
    - 0xC0000072 = account disabled
    - 0xC0000234 = account locked out
```

### Account Repeatedly Locks
```
1. Identify which DC locked it (Event 4740 on each DC)
2. Identify source IP (Event 4625 sub-status: Workstation Name)
3. Check Event 4625 on ALL DCs (find source IP)
4. Common causes:
   - Mobile device syncing with wrong password
   - Service account with hardcoded password
   - Scheduled task running with old credentials
   - RDP session cached credentials
   - Browser saving credentials for domain resources
   - Mapping network drives with cached credentials
5. Check source machine:
   - Credential Manager (Windows Credential Manager, saved RDP connections)
   - Browser saved passwords
   - Service accounts (check scheduled tasks, services)
6. Unlock: Unlock-ADAccount username
7. To prevent recurrence:
   - Find source and fix
   - Increase lockout threshold (e.g., 10 attempts)
   - Reduce lockout duration (