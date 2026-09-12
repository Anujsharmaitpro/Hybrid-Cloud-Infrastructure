# 🖥️ VMWARE COMPLETE MASTER PREPARATION GUIDE

> **Context:** This continues your L3 Windows & Virtualization Administrator interview prep. VMware vSphere is the hypervisor layer that underpins the Windows Server workloads (AD DS, DNS, GPO, Exchange, SQL, etc.) covered in earlier sections. Understanding VMware at L3 means understanding how your Windows workloads are hosted, protected, migrated, and optimized.

---

# 📌 SECTION A — DEEP REFERENCE: EVERY VMWARE COMPONENT

---

## 1. ESXi

### What
ESXi is VMware's **bare-metal (Type-1) hypervisor** — a purpose-built OS that installs directly on server hardware and creates a virtualization layer. It abstracts physical CPU, memory, storage, and networking into virtual resources for VMs. A single ESXi host can run hundreds of VMs.

### Why
- **Hardware abstraction**: VMs don't see physical hardware; they see virtual hardware
- **Resource efficiency**: Multiple workloads on one server (10:1 to 20:1 consolidation ratio typical)
- **Workload isolation**: VM crash doesn't affect other VMs on same host
- **Snapshot/backup support**: VM-level protection
- **vMotion/HA/DRS foundation**: Requires ESXi as the compute layer
- **Management**: Unified management via vCenter

### Architecture
```
┌──────────────────────────────────────────────────────────┐
│                     VMKERNEL                             │
│  (ESXi OS — microkernel)                                │
│                                                          │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐            │
│  │vmkernel      │ │Port Group  │ │Management  │            │
│  │Networking    │ │(VM traffic)│ │Network     │            │
│  │(vmk0, vmk1) │ │            │ │(vmk0 for   │            │
│  │              │ │            │ │ management)│            │
│  └────────────┘ └────────────┘ └────────────┘            │
│                                                          │
│  ┌──────────────────────────────────────────────────┐     │
│  │   VM EXECUTION ENGINE                            │     │
│  │   VMkernel processes:                            │     │
│  │   • VMW (VM Worker Process per VM)               │     │
│  │   • VMM (Virtual Machine Monitor)                │     │
│  │   • Schedulers: CPU, Memory, Disk, Network       │     │
│  └──────────────────────────────────────────────────┘     │
│                                                          │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐            │
│  │ VM1        │ │ VM2        │ │ VM3        │            │
│  │ vCPU, vRAM │ │ vCPU, vRAM │ │ vCPU, vRAM │            │
│  │ vNIC, vDisk│ │ vNIC, vDisk│ │ vNIC, vDisk│            │
│  └────────────┘ └────────────┘ └────────────┘            │
│                                                          │
│  ┌──────────────────────────────────────────────────┐     │
│  │  HARDWARE: CPU (VT-x/AMD-V), RAM, NIC, HBA/     │     │
│  │  Storage (local SSD/SAS/NVMe, SAN/NAS LUNs)      │     │
│  └──────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────┘
```

### Dependencies
```
ESXi depends on:
├── Hardware compatibility (HCL — Hardware Compatibility List)
├── CPU with virtualization extensions (Intel VT-x / AMD-V)
├── NX/XD bit support
├── Enough RAM for VM loads + ESXi overhead (~6.4 GB minimum)
├── Boot device (SSD recommended, USB/SD unsupported for production)
├── Network connectivity (management network on vmk0)
├── Shared storage for vMotion/HA/DRS (NFS/iSCSI/FC/NVMe-oF)
├── vCenter (optional but strongly recommended for multi-host)
└── VMware Tools (inside VMs for enhanced performance)

ESXi is depended upon by:
├── All VMs running on it
├── vCenter (manages ESXi hosts)
├── HA/DRS/vMotion/Storage vMotion
├── Resource Pools (resource allocation hierarchy)
└── Monitoring systems (vRealize, Zabbix, etc.)
```

### Failure
| Failure | Impact |
|---------|--------|
| ESXi host BSOD/reboot | All VMs on host go down (unless HA available) |
| Management network (vmk0) down | Cannot access host via vCenter/vSphere Client |
| VMkernel network down | VM traffic fails (if single vmk used) |
| Storage path failure | VMs on that datastore become inaccessible |
| Host enters disconnected state | Cannot manage via vCenter; HA may trigger |
| Lockdown mode enabled | CLI access blocked (security feature) |
| Custom key EVC mode | VMs cannot vMotion if CPU mismatch |
| Full boot device | Host cannot boot; use relief media |

### Symptoms
- VM power-off / inaccessible (host down)
- Cannot connect to vSphere Client
- VMs unreachable (storage path)
- Slow VM performance (resource exhaustion)
- Management network lost

### Troubleshooting
```powershell
# ESXi CLI (SSH/DCUI):
# Check host health:
esxcli hardware health get
# Check ESXi version:
vmware -vl
# Check services:
/etc/init.d/ntfystatus
# Check management network:
esxcli network ip interface list
esxcli network ip interface ipv4 get -i vmk0
# Check vMotion network:
esxcli network ip interface ipv4 get -i vmk1
# Check VM list:
vim-cmd vmsvc/getallvms
# Check VM power state:
vim-cmd vmsvc/power.getstate <VMID>
# Check host in vCenter:
vim-cmd hostsvc/hostcheck_agent
# Check storage:
esxcli storage filesystem list
esxcli storage core path list
# Check network:
esxcli network vswitch standard list -n vSwitch0
esxcli network vswitch distributed list

# From vCenter:
# Host > Monitor > Overview → Check health, alarms
# Host > Manage > Networking → Check VMkernel adapters, vSwitches
# Host > Manage > Storage > PSA (Path Selection Array) → Check multipathing
# Host > Performance → CPU, Memory, Disk, Network charts
```

### Recovery
- Host BSOD → physical reboot, check hardware (RAM, disk, NIC), update drivers/firmware
- Management network down → DCUI (Direct Console UI) or SSH to fix vmk0 IP settings
- Disconnected host → Check management network, firewall, vCenter SSL certificate validity
- Storage path failure → Rescan storage, reconfigure multipathing, fail over paths
- Lockdown mode → DCUI → Troubleshooting Options → Enable ESXi Shell

### Interview Answer
> "ESXi is the bare-metal hypervisor that runs directly on hardware. It abstracts CPU, memory, storage, and networking into virtual resources. All my Windows workloads (DCs, file servers, Exchange, SQL) run as VMs on ESXi hosts. ESXi depends on hardware with VT-x/AMD-V extensions, shared storage for vMotion/HA, and VMware Tools for optimized performance. When an ESXi host goes down, my first check is physical connectivity, then I check if HA is configured to restart VMs on another host. I monitor via vCenter and use esxcli for direct troubleshooting from the host."

---

## 2. vCenter (vCenter Server / vCenter Server Appliance - VCSA)

### What
vCenter is VMware's **centralized management platform** for vSphere environments. It provides a single pane of glass for managing ESXi hosts, VMs, clusters, datastores, networking, and security. It consists of:
- **vCenter Server** (Windows or Linux-based, or VCSA — vCenter Server Appliance, which is now the default)
- **vSphere Client** (HTML5 web interface)
- **vSphere APIs** (programmatic access)

### Why
- **Centralized management**: Single console for multiple ESXi hosts
- **Permissions & roles**: RBAC (Role-Based Access Control) across all objects
- **HA/DRS/vMotion**: Cluster-level features require vCenter
- **Alarms & monitoring**: Proactive alerting
- **Tasks & events**: Audit trail of all actions
- **Customization**: Host profiles, DRS rules, resource pools
- **Backup/restore**: VCSA backup and restore

### Architecture
```
┌────────────────────────────────────────────────────────────┐
│                    vCenter Server                          │
│  (VCSA — Ubuntu-based Linux VM, or Windows Server)        │
│                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐   │
│  │ Management   │  │ Inventory    │  │ vSphere Client │   │
│  │ Agent        │  │ Service      │  │ (HTML5)        │   │
│  │              │  │ (PostgreSQL  │  │ Frontend       │   │
│  │  vpxd        │  │  inside VCSA)│  │                │   │
│  │  (Core       │  │              │  │  SSO/Identity  │   │
│  │  Service)    │  │  vPostgres   │  │  Provider      │   │
│  └──────────────┘  └──────────────┘  └────────────────┘   │
│                                                            │
│  ┌──────────────────────────────────────────────────┐      │
│  │  Services:                                      │      │
│  │  • vpxd (core service, all operations)          │      │
│  │  • vpxa (agent on ESXi hosts)                  │      │
│  │  • Inventory/Query Service                      │      │
│  │  • Alarming Service                             │      │
│  │  • Event Service                                │      │
│  │  • Task Service                                 │      │
│  │  • SSO/LDAP (identity)                          │      │
│  │  • Web Client Service (HTML5 frontend)          │      │
│  │  • Content Library Service                      │      │
│  │  • Certificate Service                          │      │
│  │  • License Service                              │      │
│  │  • Syslog Service                               │      │
│  │  • vRealize Operations Manager (optional)       │      │
│  └──────────────────────────────────────────────────┘      │
│                                                            │
│  ┌──────────────────────────────────────────────────┐      │
│  │  Database: vPostgres (embedded) or external SQL  │      │
│  │  (Oracle, SQL Server — for large environments)   │      │
│  └──────────────────────────────────────────────────┘      │
│                                                            │
│  ┌──────────────────────────────────────────────────┐      │
│  │  Communication Ports:                             │      │
│  │  443 (HTTPS/Client), 902 (vpxa/ESXi agent),      │      │
│  │  10443 (API), 9443 (vPostgres), 5480 (VCSA web)  │      │
│  └──────────────────────────────────────────────────┘      │
└────────────────────────────────────────────────────────────┘
            │          │          │
            ▼          ▼          ▼
      ┌──────┐   ┌──────┐   ┌──────┐
      │ESXi01│   │ESXi02│   │ESXi03│
      └──────┘   └──────┘   └──────┘
```

### Dependencies
```
vCenter depends on:
├── ESXi hosts (managed objects)
├── DNS (FQDN for vCenter, reverse lookup for ESXi)
├── NTP (time sync — critical for certificates, events)
├── SSO/LDAP/AD (identity provider for authentication)
├── Certificate (SSL certificate for vCenter, auto-generated or custom)
├── Database (vPostgres embedded or external Oracle/SQL Server)
├── Network connectivity to all ESXi hosts (port 902 — vpxa agent)
├── Shared storage (for vMotion/HA)
├── Content Library (optional — for templates, ISOs)
└── vRealize Operations / vRealize Log Insight (optional monitoring)

vCenter is depended upon by:
├── HA/DRS/vMotion/Storage vMotion (cluster management features)
├── RBAC (roles and permissions)
├── Alarming and monitoring
├── Resource Pool management
├── DRS rules (VM/Host, VM/VM affinity)
├── Host Profiles (configuration compliance)
├── vRealize Automation (cloud automation)
└── All VM lifecycle operations (deploy, clone, migrate)
```

### Failure
| Failure | Impact |
|---------|--------|
| vCenter down | Cannot manage VMs/hosts via UI; HA/DRS still work; vMotion still works (can use ESXi CLI/host client) |
| vCenter database full/corrupt | Tasks fail, alarms stop, inventory issues |
| vCenter SSO failure | Cannot log in; authentication broken for all users |
| vCenter SSL certificate expired | Browser refuses connection |
| vpxa agent on ESXi fails | Host disconnected from vCenter (host itself still running VMs) |
| vCenter network isolation | Cannot reach from client machines |
| vCenter service crash | Specific function unavailable (e.g., Alarming service down → no alerts) |
| VCSA disk full | Services crash, backups fail, tasks stall |
| vCenter not in HA | Single point of failure for management |

### Symptoms
- "Connect to vCenter Server" error in vSphere Client
- Hosts show "disconnected" in inventory
- Tasks/alarms not working
- Cannot login (SSO issue)
- Performance charts unavailable (data collection stopped)
- "An error occurred during vSphere HA..." messages

### Troubleshooting
```bash
# From VCSA (SSH):
# Check vCenter services:
service-control --status --all
# or:
/etc/init.d/vmware-vpxa status
# Restart services:
service-control --restart vpxd
service-control --restart vpxa
service-control --restart alma
service-control --restart vmon
service-control --restart vcdb (database)

# Check VCSA health:
vcsa-health-check
# or via VAMI (https://vcenter.fqdn:5480)

# Check vCenter time:
date
ntpq -p
# Sync time:
timedatectl set-ntp true

# Check vCenter logs:
/var/log/vmware/vpxd/vpxd.log   # Core service log
/var/log/vmware/vpxd/vpxa.log   # Agent log
/var/log/vmware/vsphere-client/  # Web client logs
/var/log/vmware/sso/            # SSO logs
/var/log/vmware/db/postgresql/  # Database logs

# From ESXi (check vpxa agent):
esxcli software vib list | grep -i vpxa
esxcli network firewall ruleset list | grep -i vpxa
# Port 902 check:
nc -zv <vCenter_IP> 902

# Check vCenter database size:
# Via VCSA VAMI (port 5480) → Database tab
# Or: psql -U postgres -c "SELECT pg_database.datname, pg_size_pretty(pg_database_size(pg_database.datname)) FROM pg_database;"

# Check certificates:
vami-get-certificate
vami-softcerts

# Check events/tasks in vCenter (via API or PowerCLI):
# PowerCLI:
Connect-VIServer vcenter.contoso.com
Get-Event -MaxCount 50 | Select-Object CreatedTime, UserName, FullFormattedMessage
Get-Task -MaxCount 50 | Select-Object Name, State, CreatedTime

# Check for VCSA issues:
vcsa-remote-access-cli   # Remote access shell
```

### Recovery
- vCenter down → restart services (service-control --restart vpxd)
- VCSA disk full → SSH, clean logs, expand disk (vcsa-root), or deploy larger VCSA
- SSO failure → restart SSO service, check AD/LDAP connectivity, reset SSO admin password (vcsa-cli)
- Database full → reclaim space (vacuum, delete old events/tasks), or increase DB size
- SSL cert expired → regenerate (vami), or deploy new custom cert (vmware-cmd)
- Network issue → check DNS, firewall rules, vCenter VM network
- vCenter in HA → check HA status; if primary lost, standby takes over (VCSA HA feature)

### Interview Answer
> "vCenter is the centralized management platform for vSphere. It runs as a VM (VCSA, default now) and provides the web UI, APIs, and core services (vpxd, vpxa agent, SSO, alarming, events). It's critical for HA/DRS/vMotion to function at scale. When vCenter goes down, I first check if it's a service issue (restart vpxd), database issue (check disk/DB size), SSO issue (check AD connectivity), or network issue (DNS, firewall). Important: HA and DRS continue to work because they run on ESXi hosts themselves, but I lose management capability. For production, I deploy VCSA in HA mode with vPostgres in embedded mode for small environments or external DB for large."

---

## 3. Clusters

### What
A **vSphere Cluster** is a logical grouping of **two or more ESXi hosts** that share resources for:
- **High Availability (HA)**: Automatic VM restart on host failure
- **Distributed Resource Scheduler (DRS)**: Automatic VM placement/migration
- **vMotion**: Live migration of VMs between hosts
- **Storage vMotion**: Live migration of VM disk files between datastores

### Why
- **High Availability**: No single point of failure; if a host fails, VMs restart on surviving hosts
- **Load Balancing**: DRS distributes VMs across hosts based on resource utilization
- **Maintenance**: Hosts can be taken offline for patching while VMs continue running (via vMotion)
- **Capacity Management**: Cluster-level capacity planning
- **Shared Resources**: Shared storage across all hosts enables vMotion

### Architecture
```
┌──────────────────────────────────────────────────────────────┐
│                        vSphere Cluster                       │
│  (e.g., "Prod-Cluster-01")                                  │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   ESXi 01    │  │   ESXi 02    │  │   ESXi 03    │      │
│  │              │  │              │  │              │       │
│  │ VM01 (DC01)  │  │ VM02 (DC02)  │  │ VM03 (DC03)  │      │
│  │ VM04 (SQL01) │  │ VM05 (Web01) │  │ VM06 (App01) │      │
│  │ VM07         │  │ VM08         │  │              │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Cluster Shared Resources                 │   │
│  │                                                      │   │
│  │  ┌──────────────────────────────────────────┐        │   │
│  │  │  Shared Storage (Datastores):            │        │   │
│  │  │  datastore01 (VMFS, 5TB)                 │        │   │
│  │  │  datastore02 (VMFS, 5TB)                 │        │   │
│  │  │  datastore03 (NFS, 10TB)                 │        │   │
│  │  │                                          │        │   │
│  │  │  All hosts can access ALL datastores     │        │   │
│  │  │  (this is required for vMotion)          │        │   │
│  │  └──────────────────────────────────────────┘        │   │
│  │                                                      │   │
│  │  ┌──────────────────────────────────────────┐        │   │
│  │  │  vMotion Network: vmk1 on each host     │        │   │
│  │  │  VMkernel adapter for vMotion traffic   │        │   │
│  │  │  10Gbps recommended, dedicated NICs     │        │   │
│  │  └──────────────────────────────────────────┘        │   │
│  │                                                      │   │
│  │  ┌──────────────────────────────────────────┐        │   │
│  │  │  HA Agent: runs on every host in cluster │        │   │
│  │  │  DRS Resource Manager: runs on every host│        │   │
│  │  │  Master/Slave election for DRS           │        │   │
│  │  └──────────────────────────────────────────┘        │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  Cluster Settings:                                           │
│  ├── HA: enabled, admission control, VM restart priority    │
│  ├── DRS: fully automated / manual / semi-automated         │
│  ├── vMotion: enabled, network configured                   │
│  ├── EVC: enabled (CPU compatibility across hosts)          │
│  └── DRS Rules: VM/Host, VM/VM affinity/anti-affinity       │
└──────────────────────────────────────────────────────────────┘
```

### Dependencies
```
Cluster depends on:
├── Multiple ESXi hosts (min 2 for HA, recommended 3+ for DRS)
├── Shared storage accessible from ALL hosts (VMFS/NFS on SAN/NAS)
├── vMotion network (dedicated VMkernel + NICs, 10Gbps preferred)
├── HA network (management network redundancy)
├── DNS (all hosts resolve each other and vCenter)
├── Time sync (NTP — all hosts must be within 5 seconds for HA)
├── vCenter (for cluster-level management)
├── EVC mode (if hosts have different CPU generations)
├── Licensing (vSphere Enterprise Plus for vMotion, HA, DRS)
├── Firewall rules (VMotion port 8000 TCP, inter-host communication)
└── Heartbeating (datastore heartbeat for HA — if required)

Cluster is depended upon by:
├── VMs (run on hosts within cluster)
├── HA (VM restart on failure)
├── DRS (load balancing)
├── vMotion (VM migration between hosts)
├── Storage vMotion (storage migration)
├── Resource Pools (children of cluster or host)
├── DRS Rules (affinity/anti-affinity)
└── Host Profiles (configuration compliance)
```

### Failure
| Failure | Impact |
|---------|--------|
| Single host failure | HA restarts VMs on remaining hosts (if resources available) |
| All hosts in cluster fail | All VMs down; require rebuild from backup |
| Shared storage failure | All VMs on that datastore become inaccessible; vMotion fails |
| vMotion network failure | Cannot migrate VMs; HA may still work (VM restart without migration) |
| Cluster corruption | vCenter loses inventory; re-add hosts |
| HA disabled/misconfigured | No automatic VM restart on host failure |
| DRS not functioning | Load imbalance persists |
| EVC mode incompatible | vMotion between hosts with different CPU features fails |
| Admission control enabled but insufficient resources | Cannot power on VMs or HA restarts fail |
| Heartbeat failure (HA) | HA may wrongly trigger VM restarts (false positive) |

### Symptoms
- VMs unreachable after host failure
- vMotion timeouts or errors
- HA not triggering VM restarts
- DRS not moving VMs
- Cluster shows "incompatible" or "not responding"
- Alarms: "Host connectivity lost", "HA VM restart failed"

### Troubleshooting
```bash
# Cluster checks:
# From vCenter: Cluster > Monitor > Overview
# Check: HA status, DRS status, vMotion status, total capacity vs used

# From ESXi hosts:
# Check HA agent status:
esxcli storage nfs list           # Check NFS mounts
esxcli storage core path list     # Check iSCSI/FC paths

# Check vMotion:
esxcli network ip interface get -i vmk1   # vMotion VMkernel
esxcli network ip interface ipv4 get -i vmk1

# Check cluster membership:
vim-cmd hostsvc/hostcheck_agent  # Check host agent status

# Check HA events:
Get-WinEvent -FilterHashtable @{LogName='System'; ID=1193,1194,1196,1197,1199,1200,1201} 
# (via ESXi Shell or SSH)

# From PowerCLI:
Connect-VIServer vcenter.contoso.com
Get-Cluster -Name "Prod-Cluster-01" | Select-Object *
Get-Cluster -Name "Prod-Cluster-01" | Get-VM | Select-Object Name, VMHost, PowerState
Get-Cluster -Name "Prod-Cluster-01" | Get-VMHost | Select-Object Name, State, ConnectionState
Get-HAEvent -Cluster "Prod-Cluster-01" -EventTypes HAHostIsolatedEvent, HARestartVMEvent, HADeleteEvent

# Check DRS:
Get-DrsRecommendation -Cluster "Prod-Cluster-01"
Get-DrsVMGroup -Cluster "Prod-Cluster-01"
Get-DrsRule -Cluster "Prod-Cluster-01"

# Check vMotion network:
Test-vMotionConnection -Server ESXi01.contoso.com -Destination ESXi02.contoso.com
# Or use esxcli:
esxcli network ping -i vmk1 -t ESXi02.contoso.com

# Check datastore accessibility from all hosts:
Get-Datastore -Cluster "Prod-Cluster-01" | 
  Select-Object Name, @{N='AccessibleHosts';E={($_ | Get-VMHost).Name}}

# Check EVC:
Get-Cluster -Name "Prod-Cluster-01" | Select-Object -ExpandProperty EVCMode
# Check each host's CPU:
Get-VMHost | Select-Object Name, CpuType, CpuVersion
```

### Recovery
- Host failure → HA restarts VMs; verify VMs running on remaining hosts
- Cluster/vCenter issue → re-add hosts to cluster if inventory corrupted
- vMotion network failure → check vmk1 config, NIC bonding, firewall rules, IP connectivity
- HA not triggering → check HA configuration, admission control, resources on remaining hosts
- EVC issue → set EVC mode to lowest common denominator, upgrade all hosts to same CPU family

### Interview Answer
> "A vSphere cluster is a logical group of ESXi hosts that enables HA, DRS, and vMotion. HA monitors host health and restarts VMs on surviving hosts when a host fails. DRS balances VM workloads across hosts. vMotion allows live migration of running VMs. Cluster requires shared storage accessible by all hosts, dedicated vMotion network, and vCenter for management. When a host fails, HA measures available resources against VM requirements and restarts VMs on the host with most free capacity. I'd check HA status first, then verify VMs restarted, then investigate the failed host for hardware issues."

---

## 4. Datastores

### What
A **datastore** is a logical storage volume (mounted file system) on which VM disk files (VMDK), VM configuration files (.vmx), snapshots (delta VMDKs), and other VM files are stored. Datastores are presented to ESXi hosts as mount points.

### Types
| Type | Protocol | Description |
|------|----------|-------------|
| **VMFS** (VMware Virtual Machine File System) | Block (FC, iSCSI, NVMe-oF) | Proprietary clustered FS; most common for SAN storage |
| **NFS** (Network File System) | File (NFSv3/v4.1) | Network-mounted file share; simpler, used for NAS |
| **vSAN** | Software-defined (local disks) | Distributed storage across cluster nodes |
| **VSAN (all-flash)** | NVMe/Flash | vSAN optimized for flash |
| **NFS v4.1** | NFS v4.1 | Enhanced NFS with session trunking |

### Why
- Provide storage for VM files
- Enable shared storage for vMotion/HA/DRS (all hosts must access the same VM files)
- Enable snapshot/clone operations
- Enable Storage vMotion (migrate VMs between datastores)
- Enable DRS storage balancing

### Architecture
```
┌──────────────────────────────────────────────────────────┐
│ Physical Storage Layer                                     │
│  ┌────────┐ ┌────────┐ ┌────────┐                         │
│  │ Disk 1 │ │ Disk 2 │ │ Disk 3 │  (Local or SAN/NAS)    │
│  └───┬────┘└───┬────┘└───┬────┘                          │
│      │         │        │                                  │
│  ┌───▼─────────▼─────────▼───┐                            │
│  │ RAID Controller / HBA      │                           │
│  │ (PERC, P400, etc.)        │                           │
│  └───────────┬───────────────┘                            │
│              │                                             │
│  ┌───────────▼───────────────────────┐                    │
│  │ Storage System / LUN             │                     │
│  │  (SAN: EMC VMAX, NetApp, Dell    │                    │
│  │   Compellent, HPE 3PAR, Nimble)   │                    │
│  │  (NAS: NFS export from NAS)       │                     │
│  │  (Local: RAID 1/5/6/10 on host)   │                     │
│  └───────────┬───────────────────────┘                    │
│              │                                             │
│  ┌───────────▼───────────────────────┐                    │
│  │ Datastore (VMFS/NFS on ESXi)     │                     │
│  │  • VMFS: formatted with VMFS6    │                    │
│  │  • NFS: mounted via NFS protocol  │                    │
│  │  • Contains: .vmx, .vmdk,       │                    │
│  │    .vmxf, .vmsd, .vmsn, delta   │                    │
│  │    disks, snapshot files         │                    │
│  └───────────┬───────────────────────┘                    │
│              │                                             │
│  ┌───────────▼───────────────────────┐                    │
│  │ VM Files on Datastore:           │                     │
│  │  VM01/                           │                    │
│  │    VM01.vmx        (config)     │                     │
│  │    VM01.vmdk         (flat disk)│                     │
│  │    VM01-flat.vmdk   (data)      │                     │
│  │    VM01-000001.vmdk (delta, snap)│                    │
│  │    VM01.vmsd         (snapshot desc)│                    │
│  │    VM01.vmsn         (snapshot state)│                  │
│  └──────────────────────────────────┘                       │
└──────────────────────────────────────────────────────────┘
```

### Dependencies
```
Datastores depend on:
├── Physical storage (SAN, NAS, local disks)
├── Storage controller (RAID, HBA, SAN controller)
├── Network (FC, iSCSI, NFS, NVMe-oF)
├── Multipathing (PSA — Path Selection Array)
├── Datastore format (VMFS version, NFS version)
├── ESXi host (mounts the datastore)
├── Shared access (all cluster hosts must access for vMotion)
└── Storage I/O Control (SIOC) — congestion management

Datastores are depended upon by:
├── VMs (disk files, config files)
├── Snapshots (delta disks on datastore)
├── Templates (stored as VMDK)
├── ISOs / Floppy images
├── vMotion (VM files must be accessible from target host)
├── Storage vMotion (migrate between datastores)
├── HA (VM files must be accessible from HA restart host)
├── DRS (storage DRS balances VMs across datastores)
├── Clone/Deploy operations
└── vSAN (creates datastore from local disks)
```

### Failure
| Failure | Impact |
|---------|--------|
| Datastore unmounted | All VMs on datastore unreachable; VMs powered off/read-only |
| VMFS corruption | Datastore inaccessible; VMs at risk |
| Storage array failure | All datastores on that array fail; massive outage |
| NFS server down | NFS datastore unmounted; VMs affected |
| Datastore full | Cannot create snapshots, clone, power on, Storage vMotion; VM writes may fail |
| Datastore offline (transient) | I/O errors on VMs; VMs may become unresponsive |
| Lost heartbeat datastore (HA) | HA may trigger false positive VM restarts |
| LUN masking change | LUN no longer visible to ESXi; datastore disappears |
| Thin provisioning overcommit | Datastore runs out; VMs crash when thin space exhausted |
| Storage controller cache battery dead | Write caching disabled; severe performance degradation |

### Symptoms
- VM I/O errors, "Cannot open disk" messages
- VMs go into "inaccessible" state
- Datastore shows "offline" or "missing" in vCenter
- Alarms: "Datastore has less than free space", "Datastore connectivity restored"
- VMs cannot power on, create snapshots, or migrate
- I/O latency spikes (SIOC events)

### Troubleshooting
```bash
# Check datastore status:
esxcli storage filesystem list
# Shows: mount path, filesystem type (VMFS/NFS), capacity, accessible, reachable

# Check datastore from all hosts:
# PowerCLI:
Get-Datastore -Cluster "Prod-Cluster-01" | 
  Select-Object Name, @{N='VMHosts';E={($_ | Get-VMHost).Name}}, 
    FreeSpaceGB, CapacityGB, Url, Type

# Check specific VM's datastore:
Get-VM -Name VM01 | Select-Object Name, Datastore, PowerState

# Check storage paths (multipathing):
esxcli storage core path list -d <device>
# Shows: state (active/passive), type (active/standby/inactive), adapter, transport

# Check NMP (Native Multipathing Plugin):
esxcli storage core device list -d <device>
esxcli storage nmp device list -d <device>

# Rescan storage:
esxcli storage core adapter rescan -a
esxcli storage filesystem rescan

# Check datastore alarms:
Get-AlarmDefinition -Name "Datastore Disk Space" 
Get-Alarm -Datastore (Get-Datastore datastore01) | Where-Object {$_.State -eq "Red"}

# Check I/O latency:
esxcli storage core device stats -d <device>
# Latency, read/write IOPS, queue depth

# Check SIOC (Storage I/O Control):
Get-Datastore -Name datastore01 | Select-Object Name, IORMEnabled, IOReshaping

# Check thin provisioning:
Get-Datastore -Name datastore01 | Select-Object Name, 
  @{N='ProvisionedSpaceGB';E={($_.ExtensionData.Info | Select-Object -ExpandPropertyVmfs | Select-Object -ExpandProperty Extent).}},
  FreeSpaceGB

# Check all VMs on a datastore:
Get-Datastore datastore01 | Get-VM | Select-Object Name, PowerState

# VMDK file sizes:
Get-VM VM01 | Get-HardDisk | Select-Object DiskType, Filename, CapacityGB, StorageFormat

# Check NFS specific:
esxcli storage nfs list
esxcli storage nfs status -v <volume_name>

# Check VMFS specific:
esxcli storage filesystem list | grep -A 5 "VMFS"
vmkfstools -U /vmfs/volumes/datastore01  # Unmount
vmkfstools -V  # Check VMFS volume info
```

### Recovery
- Datastore offline → rescan storage, re-mount VMFS/NFS, check physical connectivity
- Datastore full → delete old snapshots (carefully!), VMArchive, Storage vMotion to larger datastore, expand datastore, clean up old VMs/templates/ISOs
- VMFS corruption → attempt mount with vmkfstools; if unrecoverable, restore from backup/replica
- NFS server down → check NFS server, export, network, mount options; remount
- Thin overcommit → expand datastore, migrate VMs off, convert thin to eager zeroed thick
- Lost heartbeat datastore → configure additional heartbeat datastore (HA settings)

### Interview Answer
> "Datastores are the storage volumes where VM files live — VMDKs, VMX configs, snapshots. We use VMFS for SAN storage and NFS for NAS. Datastores must be shared across all ESXi hosts in a cluster for vMotion and HA to work. When a datastore fails, VMs become inaccessible. My troubleshooting is: check datastore mount status, check physical storage paths, rescan storage, check multipathing state, and if full, clean up or expand. For HA, we configure multiple heartbeat datastores to prevent false positives."

---

## 5. Virtual Machines (VMs)

### What
A **Virtual Machine (VM)** is a software-emulated computer that runs its own operating system (guest OS) on top of the ESXi hypervisor. Each VM has virtual hardware (vCPU, vRAM, vNIC, vDisk, etc.) that is independent of the physical hardware.

### Why
- **Isolation**: Each VM operates independently; crash/failure is isolated
- **Portability**: VMs can be moved (vMotion), cloned, templated, backed up
- **Multiple OS**: Run Windows, Linux, etc. on same physical hardware
- **Snapshot**: Save and restore VM state
- **Scalability**: Adjust resources (vCPU, vRAM) without hardware changes
- **Disaster Recovery**: Replicate or backup entire VM state

### Architecture
```
┌──────────────────────────────────────────────────────────┐
│                    VM: VM01 (Windows Server 2022)        │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │ Guest OS (Windows Server 2022)                     │ │
│  │ ┌─────────┐ ┌─────────┐ ┌──────────┐ ┌─────────┐ │ │
│  │ │ Explorer│ │ IIS     │ │ SQL Server│ │ AD DS   │ │ │
│  │ └─────────┘ └─────────┘ └──────────┘ └─────────┘ │ │
│  │ ┌──────────────────────────────────────────────┐  │ │
│  │ │ Virtual Hardware (VM hardware abstraction)    │  │ │
│  │ │ Virtual NIC (vmxnet3) → vSwitch → Physical NIC│  │ │
│  │ │ Virtual Disk (VMDK) → Datastore → Physical disk│ │ │
│  │ │ Virtual SCSI Controller → Virtual disks       │ │ │
│  │ │ Virtual USB, Virtual Serial, etc.             │ │ │
│  │ └──────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │ VMX File (VM configuration):                       │ │
│  │  .vmx → Hardware config, power state, CPU, RAM,   │ │
│  │          Network, disk, CD/DVD, etc.               │ │
│  │  .vmxf → Additional metadata                      │ │
│  │  .nvram → UEFI NVRAM                              │ │
│  │  .vmcfx → VM config (vSphere 6.5+)                │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │ VMDK Files (Virtual Disk):                         │ │
│  │  VM01.vmdk → Descriptor (metadata about disk)     │ │
│  │  VM01-flat.vmdk → Actual data (flat format)       │ │
│  │  VM01-000001.vmdk → Snapshot delta disk           │ │
│  │  VM01-flat.vmdk is raw data; .vmdk is descriptor  │ │
│  └────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

### VM Hardware Versions
| Version | Notes |
|---------|-------|
| v1 | Original VM hardware |
| v4 | 6.x era; most compatible |
| v7 | 5.1 era; supports more devices |
| v8 | 5.5 era |
| v9 | 6.0 era |
| v10 | 6.5 era; EFI boot, better SATA |
| v11 | 6.7 era; supports more CPUs |
| v12 | 7.0 era; ESXi 7.0 |
| v13 | 7.0 U2; supports more features |
| v14 | ESXi 8.0 |
| v15 | Latest (ESXi 8.0 U2+) |

### Dependencies
```
VM depends on:
├── ESXi host (runs the VM)
├── VM files (.vmx, .vmdk, .vmxf, .nvram, .vmsd) on datastore
├── Datastore (storage for VM files)
├── vNetwork (vSwitch/port group for vNIC)
├── VMware Tools (optimal performance, time sync, heartbeat)
├── CPU (physical cores, EVC mode for compatibility)
├── RAM (physical memory allocation)
├── Network (vSwitch, port group, physical NICs)
├── VM hardware version (determines features)
├── Guest OS (installed and running)
├── Power state (powered on/off/standby)
├── VM Configuration (CPU, RAM, vNIC, disk controller type)
└── Snapshot chain (if present — adds delta disks)

VM is depended upon by:
├── Applications (run inside VM)
├── Users (access VM services)
├── HA (VM restart priority)
├── DRS (VM placement decisions)
├── vMotion (live migration)
├── Backup/Recovery (VADP, snapshots)
├── Resource Pools (resource allocation)
├── Cloning/Templating
└── Monitoring (vRealize, performance charts)
```

### Failure
| Failure | Impact |
|---------|--------|
| VM unreachable (host down) | HA restarts on another host (if configured) |
| VM unresponsive (OS hung) | Cannot RDP/SSH; applications fail; VM monitor may reset |
| VM disk corruption | Data loss; boot failure; snapshot chain may be corrupt |
| VM memory overcommitted | Balloon driver kicks in; OS slowdown; OOM killer (Linux) |
| VM CPU contention | Slow performance; applications unresponsive |
| VM network connectivity lost | VM cannot communicate; service unavailable |
| VM power state stuck | Cannot power on/off/reset; stuck in intermediate state |
| VMDK missing/corrupt | VM cannot boot; "Cannot open disk" error |
| VM too large for host resources | Cannot power on; insufficient resources error |
| VM hardware version outdated | Cannot use latest VMware features |
| Snapshot chain too long | Severe performance degradation |
| VM File System (VMFS) locked | VM cannot access disk files |
| VM stuck in "inaccessible" state | Files missing or corrupted on datastore |
| VM nested virtualization unsupported | Nested hypervisor fails |

### Symptoms
- VM not responding to ping/RDP/SSH
- VM shows "inaccessible" in vCenter
- VM power state stuck (e.g., "powering off")
- Slow VM performance
- VM cannot power on
- "Cannot open disk" errors
- VM Shows "VMware Tools status: Not Running"
- Guest OS shows blue screen / kernel panic

### Troubleshooting
```powershell
# From vCenter:
# VM > Monitor > Overview → Power state, CPU, Memory, Disk, Network utilization
# VM > Monitor > Performance → Charts
# VM > Monitor > Problems → Active alarms
# VM > Console → Interact (if VM console accessible)

# From ESXi CLI (SSH/DCUI):
# List all VMs:
vim-cmd vmsvc/getallvms
# VM ID = first column

# Check VM power state:
vim-cmd vmsvc/power.getstate <VMID>

# Reset/Power off unresponsive VM:
vim-cmd vmsvc/power.off <VMID>    # Force power off
vim-cmd vmsvc/power.reset <VMID>  # Reset (soft)

# Hard reset (if soft reset fails):
vim-cmd vmsvc/power.off <VMID>
# Then remove snapshot delta if needed, then power on

# Check VM process:
ps | grep -i "vmware-vmx\|<VMID>"
# vmx process running? If not → check vmware.log

# Check VM log:
cat /vmfs/volumes/datastore01/VM01/vmware.log | tail -200
# Key entries:
# "Power on" events
# "Powered off" events  
# "Failed to open disk" errors
# "Unable to allocate memory" errors
# "Connection to the peer endpoint was lost" (vMotion issues)

# Check VMDK integrity:
vmkfstools -y /vmfs/volumes/datastore01/VM01/VM01-flat.vmdk
# -y = check and repair

# Check VM snapshot chain:
vim-cmd vmsvc/snapshot.getall <VMID>
# Or via vSphere Client: VM > right-click > Snapshot > Snapshot Manager

# Check VMware Tools status:
vim-cmd vmsvc/get.guestFeaturesEnabled <VMID>
# Or via PowerCLI:
Get-VM VM01 | Select-Object Name, ToolsVersion, ToolsStatus, PowerState

# Check VM file system operations:
vmkfstools -D /vmfs/volumes/datastore01/VM01/   # Info
vmkfstools -i /vmfs/volumes/datastore01/VM01/VM01.vmdk /vmfs/volumes/datastore01/VM01/VM01-clone.vmdk  # Clone

# From PowerCLI:
# Check VM detailed:
Get-VM VM01 | Select-Object Name, PowerState, NumCpu, MemoryGB, 
  @{N='VMHost';E={$_.VMHost.Name}}, Datastore, 
  @{N='ToolsStatus';E={$_.ExtensionData.Guest.ToolsStatus}},
  @{N='ToolsVersion';E={$_.ExtensionData.Guest.ToolsVersion}}

# Check VM notes:
Get-VM VM01 | Select-Object Name, Notes

# Check VM custom config:
Get-VM VM01 | Get-AdvancedSetting | Where-Object {$_.Name -like "config.*"} | 
  Select-Object Name, Value

# Check VM isolation (HA):
Get-VM VM01 | Select-Object Name, 
  @{N='IsHAIsolated';E={$_.ExtensionData.Extras.IsHAIsolated}},
  @{N='IsVMToolsRunning';E={$_.ExtensionData.Extras.ToolsRunning}}

# Check all VMs with issues:
Get-VM | Where-Object {$_.PowerState -eq "PoweredOn" -and $_.ExtensionData.Guest.ToolsStatus -ne "toolsOk"}

# Check VM datastores:
Get-VM VM01 | Get-HardDisk | Select-Object Disktype, Filename, CapacityGB, StorageFormat, Persistence

# Check VM network:
Get-VM VM01 | Get-NetworkAdapter | Select-Object Name, NetworkName, MacAddress, StartConnected

# Check all alarms on VM:
Get-Alarm -VM VM01 | Where-Object {$_.State -ne "Green"} | 
  Select-Object Name, State, Severity, OldStatus, NewStatus
```

### Recovery
- Unresponsive VM → vCenter: Power off/Reset; ESXi: vim-cmd vmsvc/power.off
- VM unreachable → Check host health, HA restart, datastore accessibility, VMDK integrity
- VM disk corrupt → Check VMDK with vmkfstools; restore from backup/snapshot; remove corrupt snapshot
- VM tools not running → Reinstall VMware Tools (vCenter: VM > Install VMware Tools)
- VM power stuck → Force power off from vCenter or ESXi CLI; check .vmx file
- VM inaccessible → Check datastore mount, VMDK files, file permissions on VMFS

### Interview Answer
> "A VM is a software-emulated computer running a guest OS on the ESXi hypervisor. VMs are defined by .vmx config files and .vmdk disk files stored on datastores. They depend on the ESXi host for compute, shared storage for migration and HA, virtual networking for connectivity, and VMware Tools for optimized performance. When a VM becomes inaccessible, I check: host status (is ESXi up?), datastore connectivity (are VMDK files accessible?), VM power state, VMDK integrity (vmkfstools), and VMware Tools status. I use vCenter VM Monitor, ESXi vim-cmd commands, and vmkfstools for deep troubleshooting."

---

## 6. Templates

### What
A **VM template** is a **golden master image** — a VM that has been converted to a read-only reference for deploying new VMs. Templates contain the OS, applications, configurations, and settings that you want to replicate across multiple VMs.

### Why
- **Rapid deployment**: Deploy 100 VMs from a template in minutes instead of building each from scratch
- **Consistency**: Every VM from the same template has identical OS/software/configuration
- **Standardization**: Enforce security baselines, naming conventions, installed applications
- **Efficiency**: Avoid manual configuration for each VM

### Architecture
```
┌──────────────────────────────────────────────┐
│ SOURCE VM (build master)                     │
│ Install OS → Applications → Config → Custom │
│ Sysprep (Windows) / unattend.xml             │
│ Optimize (disable services, enable templates)│
└───────────────┬──────────────────────────────┘
                │ Convert to Template
                ▼
┌──────────────────────────────────────────────┐
│ TEMPLATE (read-only, cannot power on)        │
│ Location: Typically on a dedicated datastore │
│ Properties: Config, Hardware, Customization │
│ Status: Not deployable (read-only copy)      │
└───────────────┬──────────────────────────────┘
                │ Deploy from Template
                ▼
┌──────────────────────────────────────────────┐
│ NEW VM (from template)                       │
│ - Powered off initially                     │
│ - Customization applied:                    │
│   • Name changes                            │
│   • Network (MAC, IP, DNS)                  │
│   • Windows Sysprep (SID, hostname)         │
│   • Linux cloud-init                        │
│ - Then powered on                           │
└──────────────────────────────────────────────┘
```

### Types
| Type | Description |
|------|-------------|
| **Template (VM-based)** | Converted from a VM; portable across datacenters/VCs |
| **OS Installation ISO** | Not a VM; ISO files (rarely used as template) |
| **Content Library Template** | Stored in vCenter Content Library; downloadable |
| **Clone from VM** | Full clone vs. linked clone (linked uses snapshot delta) |

### Dependencies
```
Templates depend on:
├── Source VM (built manually or via automation)
├── Datastore (where template is stored)
├── OS customization (Sysprep for Windows, cloud-init for Linux)
├── vCenter (template management)
├── Resource Pool/Cluster (for deployment)
├── Network configuration (port group, IP)
├── Storage policy (if Storage Policy-Based Management / SPBM)
├── Content Library (if distributed via content library)

Templates are depended upon by:
├── VM deployments (cloning from template)
├── Customization specs (Sysprep profiles)
├── Auto Deploy (stateless ESXi hosts)
├── vRealize Automation (self-service VM provisioning)
└── Application provisioning workflows
```

### Failure
| Failure | Impact |
|---------|--------|
| Template corrupted | Cannot deploy new VMs from it; rebuild template |
| Template storage full | Cannot create new template; datastore full |
| Sysprep fails during customization | New VM gets generic SID, no hostname, no network config |
| Template not compatible (EVC/version) | Deployment fails if cluster requires different hardware version |
| Template not in Content Library | Cannot deploy across vCenters |
| Linked clone source snapshot deleted | Linked clone VM becomes corrupt |
| Template on read-only datastore | Cannot update template |

### Troubleshooting
```powershell
# PowerCLI:
# List all templates:
Get-Template | Select-Object Name, Datastore, @{N='VC';E={$_.ExtensionData.Config.VcInstanceName}}

# Deploy from template:
New-VM -Name "NewVM" -Template "Win2022Template" -VMHost ESXi01.contoso.com -Datastore datastore01 -ResourcePool "Production" -Network "Production-Net"

# Clone from template:
Get-Template Win2022Template | New-VM -Name "VM01" -VMHost ESXi01 -Datastore datastore01 -ResourcePool "Prod"

# Convert VM to template:
Set-VM -VM VM01 -ToTemplate -Confirm:$false

# Check template status:
Get-Template | Where-Object { $_.ExtensionData.Config.Template -eq $true } | Select-Object Name

# Check customization spec:
Get-OSCustomizationSpec | Select-Object Name, OSType, Description

# Check Sysprep file:
Get-OSCustomizationNicSetting | Select-Object *

# Check deployment errors:
Get-Task | Where-Object { $_.Name -like "*deploy*" -or $_.Name -like "*clone*" } | 
  Select-Object Name, State, Error, Started, Completed

# Check Content Library:
Get-ContentLibrary | Select-Object Name, DefaultDatastore
Get-ContentLibraryItem -ContentLibrary "LibraryName" | Select-Object Name, Type, Version

# Re-deploy template from source:
# 1. Clone template VM → new VM
# 2. Configure new VM
# 3. Run Sysprep (Windows) or cloud-init (Linux)
# 4. SetVM -ToTemplate again
```

### Recovery
- Template corrupt → Rebuild from source VM
- Deployment failure → Check task error log, Sysprep spec, network settings, resource pool
- Sysprep failure → Check unattend.xml/customization spec for errors
- Linked clone corrupt → Recreate from source template

### Interview Answer
> "Templates are golden master images for rapid VM deployment. I build a source VM, install OS, applications, configure everything, run Sysprep for Windows (to generalize the image), then convert it to a template via Set-VM -ToTemplate. When deploying, I clone from the template and apply customization specs for hostname, network, and domain join. Templates ensure consistency across all VMs. Common issues include corrupt templates (rebuild from source), Sysprep failures (check customization spec), and full datastores (expand or clean up)."

---

## 7. Snapshots

### What
A **snapshot** is a point-in-time copy of a VM's state, capturing the VM's power state, settings, and disk contents. Snapshots work by creating **delta disks** (delta VMDKs) that record only changes made AFTER the snapshot was taken.

### Why
- **Protection**: Quick rollback point before making risky changes
- **Testing**: Test software/updates with ability to revert
- **Backup support**: Some backup tools use snapshots for consistent backup points
- **Dev/Test**: Save state for repeated testing cycles
- **Troubleshooting**: Capture state before/after issue for analysis

### Architecture (Critical Understanding)
```
BEFORE SNAPSHOT:
┌──────────────┐
│ VM01.vmdk    │ (Flat disk — ALL data)
│ (flat.vmdk)  │ Size: 100 GB
└──────────────┘

AFTER TAKING SNAPSHOT 1 (Snap1):
┌──────────────┐    ┌──────────────┐
│ VM01.vmdk    │    │ Snap1_deltav │ (Delta disk — changes since Snap1)
│ (flat.vmdk)  │ +  │ (000001.vmdk)│ Size: 0 GB initially, grows over time
└──────────────┘    └──────────────┘

AFTER TAKING SNAPSHOT 2 (Snap2):
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ VM01.vmdk    │    │ Snap1_delta  │    │ Snap2_delta  │
│ (flat.vmdk)  │    │ (000001.vmdk)│ +  │ (000002.vmdk)│ Size: grows with changes
└──────────────┘    └──────────────┘    └──────────────┘

LINEAR CHAIN (recommended for simplicity):
VM01-flat.vmdk → 000001-delta.vmdk → 000002-delta.vmdk

BRANCHED CHAIN (from vCenter, NOT ESXi CLI):
VM01-flat.vmdk → 000001-delta.vmdk → branches to → 000003-delta.vmdk
```

### How Snapshots Work
```
1. User takes snapshot at time T1
2. VM continues running
3. Delta disk created: 000001-delta.vmdk (initially 0 KB)
4. As VM writes data, OLD data stays in flat.vmdk, NEW data goes to delta
5. User takes another snapshot at T2
6. New delta disk: 000002-delta.vmdk
7. Chain: flat.vmdk → 000001-delta → 000002-delta

REVERT to T1 snapshot:
- flat.vmdk is restored (delta discarded)
- VM state is as of T1

DELETE ALL snapshots:
- All delta disks merged back into flat.vmdk
- Flat vmdk now contains all changes from all snapshots
- This merge can take significant time and I/O
```

### Dependencies
```
Snapshots depend on:
├── Datastore (where delta disks are stored)
├── VM configuration (.vmx, .vmdk)
├── Datastore free space (must have 1.5x the VM disk size free for snapshots — CRITICAL)
├── Snapshots cannot cross datastore boundaries (all delta disks on same datastore)
├── Time (merge/delete operations take time proportional to delta size)
├── I/O performance (snapshot creation/deletion causes I/O)
├── VM hardware version (supports max snapshots)
├── Backup software (if using snapshot for backup)
└── vCenter (if branched snapshots via vCenter API)

Snapshots are depended upon by:
├── Backup tools (VADP, VDDK — use snapshots for consistent backup)
├── vMotion (with snapshots — can be complex)
├── Storage vMotion (with snapshots — special considerations)
├── Cloning (linked clones use snapshots)
├── VM revert operations (user-initiated)
└── Skipped snapshots (recommendation)
```

### Failure
| Failure | Impact |
|---------|--------|
| Datastore full (delta disks growing) | Cannot take more snapshots; VM may become unresponsive |
| Snapshot merge stuck | VM stays in "reverting" state; I/O stalled |
| Snapshot chain too long (>2-3 generally recommended, max 32) | Severe performance degradation; revert operations very slow |
| Delta disk missing/corrupt | Snapshot cannot be reverted; VM may be inaccessible |
| Snapshot during vMotion | Complex; may fail or require special handling |
| Long-running snapshot (days/weeks) | Delta disk grows extremely large; impact on storage |
| Branching (via vCenter only) | ESXi CLI cannot branch; only creates linear chains |
| Backup fails to delete snapshot | Snapshot left behind; consumes space |
| 3rd-party backup not supported | Snapshots accumulate; known issue with some backup tools |

### Symptoms
- VM slow (multiple delta disks causing extra I/O)
- Datastore full alarms
- "Swap file usage" high (VM memory swapped due to snapshot I/O)
- "Reverting" stuck status
- Delta disk growing rapidly (high write I/O)
- vMotion fails with "Snapshot operation failure"
- Backup warnings about snapshot age
- Event: "VMware VM snapshot operation"

### Troubleshooting
```powershell
# Check snapshot chain:
# vCenter: VM > Snapshot > Snapshot Manager
# PowerCLI:
Get-Snapshot -VM VM01 | Select-Object Name, Id, Created, VM, 
  @{N='SizeMB';E={$_.SizeMB}}, Description | Format-Table

# Detailed snapshot info:
Get-VM VM01 | Get-Snapshot | Select-Object *

# Check snapshot files on datastore:
Get-ChildItem /vmfs/volumes/datastore01/VM01/ -Filter "*.vmdk" | 
  Select-Object Name, Length, @{N='SizeGB';E={$_.Length/1GB}}

# VM file listing:
Get-VM VM01 | Get-HardDisk | Select-Object Name, CapacityGB, Filename, StorageFormat, Persistence, ControllerType

# Check delta disk chain:
vim-cmd vmsvc/get.disk <VMID>
# Or:
vim-cmd vmsvc/device.diskinfo <VMID>

# Delete snapshots (power off VM for safety):
Remove-Snapshot -VM VM01 -Name "SnapshotName" -Confirm:$false
# Remove ALL snapshots (consolidate):
Get-VM VM01 | Get-Snapshot | Remove-Snapshot -Confirm:$false
# Consolidate (merge):
Get-VM VM01 | Get-Snapshot | Remove-Snapshot -Merge:$true -Confirm:$false
# Or use ESXi:
vim-cmd vmsvc/snapshot.removeall <VMID>

# Check if snapshot consolidation is in progress:
vim-cmd vmsvc/snapshot.getall <VMID>

# Check VM snapshot info:
Get-VM VM01 | Select-Object Name, @{N='HasSnapshots';E={$_.ExtensionData.Guest.ConsolidationNeeded}},
  @{N='SnapshotCount';E={($_.ExtensionData.Guest.CheckpointList | Measure-Object).Count}}

# Check delta disk sizes and growth:
# Via vCenter: VM > Monitor > Snapshot
# Via datastore: df -h /vmfs/volumes/datastore01/VM01/

# Force consolidation from ESXi if stuck:
# Use vCenter: Right-click VM > Snapshot > Consolidate
# Or via PowerCLI:
Get-VM VM01 | Get-Snapshot | Remove-Snapshot -Merge:$true -Confirm:$false -RunAsync

# Check datastore space:
Get-Datastore datastore01 | Select-Object Name, FreeSpaceGB, CapacityGB, 
  @{N='UsedSpaceGB';E={$_.CapacityGB - $_.FreeSpaceGB}}

# Check all VMs with snapshots on a datastore:
Get-Datastore datastore01 | Get-VM | Where-Object { (Get-Snapshot $_).Count -gt 0 } | 
  Select-Object Name, @{N='SnapshotCount';E={(Get-Snapshot $_).Count}}

# Check snapshot task progress:
Get-Task | Where-Object { $_.Name -like "*consolidat*" -or $_.Name -like "*snapshot*" } | 
  Select-Object Name, State, Progress, CreatedTime
```

### Recovery
- Datastore full from snapshots → Delete old/unneeded snapshots (consolidate); if VMs are critical, power off and delete; expand datastore
- Consolidation stuck → Check task status; may need vCenter restart; check datastore space; try again
- Long snapshot chain → Consolidate all in stages (one at a time if needed)
- Delta disk corrupt → If possible, consolidate from backup; if not, remove delta chain (data loss risk)

### Interview Answer
> "Snapshots capture VM state at a point in time using delta disks. I NEVER keep snapshots long-term — they're meant for short-term protection, ideally less than 24-72 hours. Each delta disk grows with writes to the original disk. Too many snapshots cause performance issues and can fill the datastore. I recommend max 2-3 snapshots per VM. When done, I consolidate them (delete/merge back into flat disk). For backup, I use proper backup software (Veeam, Commvault) rather than relying on manual snapshots. The most common issue I see is datastore full from forgotten snapshots — I monitor delta disk growth and enforce snapshot policies."

---

## 8. High Availability (HA)

### What
**HA** (High Availability) is a **cluster-level feature** that automatically restarts VMs on different ESXi hosts in the cluster when a host fails or a VM becomes unresponsive. HA monitors host health via:
- **VM heartbeat** (VMware Tools heartbeat)
- **Host isolation** (ping check — host loses network)
- **Storage heartbeat** (check if storage is still accessible)

### Why
- **Minimize downtime**: VMs automatically restart on surviving hosts
- **No manual intervention**: No need for an admin to discover and restart VMs
- **Protect critical workloads**: DCs, SQL, Exchange automatically restart
- **Cost effective**: HA provides failover without dedicated failover hardware

### Architecture
```
┌──────────────────────────────────────────────────────────────┐
│                        Cluster with HA                        │
│  HA Agent runs on EVERY host in the cluster                    │
│                                                              │
│  ┌──────────┐                                              │
│  │ ESXi 01  │ ← HA Agent running                            │
│  │ (Primary)│ ← Master (elected for DRS)                    │
│  └──────────┘     Monitors:                                  │
│       │           │   • Host isolation (ping)                │
│       │           │   • VM tools heartbeat (vmtoolsd)        │
│       │           │   • Storage heartbeat (datastore HB)     │
│       │           │   • Network connectivity to vCenter      │
│       │           │                                            │
│  ┌──────────┐     Heartbeat signals exchanged between hosts  │
│  │ ESXi 02  │ ← HA Agent running                            │
│  │          │ ← Monitors ESXi 01 (and ESXi 03)              │
│  └──────────┘                                              │
│       │                                                    │
│  ┌──────────┐                                              │
│  │ ESXi 03  │ ← HA Agent running                            │
│  │          │                                              │
│  └──────────┘                                              │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  HA Failure Detection:                               │   │
│  │  1. Host Isolation (management network lost)          │   │
│  │  2. Host Unresponsive (HA agent lost contact)         │   │
│  │  3. VM Unresponsive (VM tools heartbeat lost)         │   │
│  │  4. VM Tools Disabled (VM tools not running)          │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  HA Restart Process:                                         │
│  1. Detect failure                                          │
│  2. Determine which hosts are isolated/unresponsive           │
│  3. Calculate available resources on remaining hosts          │
│  4. Check admission control:                                │
│     a) Failover capacity (% reserved)                        │
│     b. Host can accommodate VM?                              │
│  5. Select host with most free resources                     │
│  6. Power on VM on selected host                            │
│  7. Maintain VM priority order (High/Medium/Low)             │
│  8. Repeat for all failed VMs                                │
└──────────────────────────────────────────────────────────────┘
```

### HA Admission Control
```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  Admission Control Method:                                │
│  1. Cluster Resource % — Reserve X% of total resources     │
│     - Example: 25% → 25% of CPU/RAM always free           │
│     - HA can use this reserved capacity for failover      │
│  2. Tolerate N Host Failures — Maintain N host worth       │
│     of capacity (e.g., 2 hosts × CPU/RAM = reserved)      │
│  3. Disable — No admission control (VMs may not restart)  │
│                                                          │
│  IMPORTANT: Even with admission control DISABLED,          │
│  HA will try to restart VMs, but ONLY if there are        │
│  enough resources on remaining hosts.                     │
│  If resources insufficient → VMs won't restart!           │
└──────────────────────────────────────────────────────────┘
```

### Dependencies
```
HA depends on:
├── Cluster (min 2 hosts, ideally 3+)
├── Shared storage (for VM files) OR datastore heartbeating
├── Management network (redundant for isolation detection)
├── VMware Tools (for VM heartbeat monitoring)
├── vCenter (management, though HA continues if vCenter is down)
├── VM restart priority (High/Medium/Low — HA restarts High first)
├── VM protection (VM Restart Priority, VM Restart Action)
├── DRS host availability (DRS VM/Host rules respected during HA)
├── NIC isolation detection (management network heartbeat)
├── Datastore heartbeat (if shared storage unavailable)
└── Network redundancy (multiple NICs, bonded, multiple p switches)

HA is depended upon by:
├── VMs (automatic restart on host failure)
├── DRS (HA takes precedence during failures)
├── vMotion (HA restarts VMs where vMotion couldn't save)
├── Cluster availability
└── Business continuity
```

### Failure
| Failure | Impact |
|---------|--------|
| HA disabled | No automatic VM restart; manual intervention needed |
| Insufficient resources for HA restart | VMs don't restart even though HA tried |
| HA false positive (host isolated but actually alive) | VMs restart on another host; original host still running (split-brain potential) |
| VM not restarting (VM restart disabled) | VM stays down despite HA |
| All hosts lose isolation simultaneously | HA cannot restart VMs (no surviving hosts) |
| HA agent crash | No monitoring; VMs won't auto-restart |
| Network isolation detection failure | HA triggers wrong actions |
| Datastore only heartbeat path lost | HA may trigger VM restarts (false positive) |
| VMware Tools not running | VM heartbeat fails; HA may mark VM as unresponsive |
| Admission control blocks HA restart | VMs don't restart due to resource constraints |

### Symptoms
- VMs not restarted after host failure
- "HA VM Restart Failed" alarm
- VMs restarting repeatedly (HA loop)
- HA not responding to host failures
- "Host isolated" alarms
- VMs stuck in "VM restart in progress" state

### Troubleshooting
```powershell
# PowerCLI:
# Check HA status:
Get-Cluster "Prod-Cluster-01" | Select-Object Name, 
  @{N='HAEnabled';E={$_.ExtensionData.Configuration.HaConfig.ClusterConfigInfo.HaEnabled}},
  @{N='AdmissionControlEnabled';E={$_.ExtensionData.Configuration.HaConfig.ClusterConfigInfo.AdmissionControlEnabled}},
  @{N='RestartPriority';E={$_.ExtensionData.Configuration.HaConfig.ClusterConfigInfo.RestartPriority}}

# Check HA VM restart specifications:
Get-VM VM01 | Select-Object Name, 
  @{N='HA Restart Priority';E={$_.ExtensionData.HAEvent.HAConfig.RestartPriority}},
  @{N='HA Restart Action';E={$_.ExtensionData.HAEvent.HAConfig.RestartAction}},
  @{N='HA Isolate Response';E={$_.ExtensionData.HAEvent.HAConfig.IsolateResponse}}

# Check HA events:
Get-Cluster "Prod-Cluster-01" | Get-VIEvent -HA | Select-Object CreatedTime, Message | Select-Object -First 50
# Or:
Get-HAEvent -Cluster "Prod-Cluster-01" -EventTypes HAIsolatedEvent, HARestartVMEvent, HADeleteEvent | 
  Select-Object CreatedTime, FullFormattedMessage

# Check HA host monitoring:
Get-Cluster "Prod-Cluster-01" | Get-VMHost | Select-Object Name, 
  @{N='State';E={$_.State}},
  @{N='ConnectionState';E={$_.ConnectionState}},
  @{N='HANotResponding';E={$_.ExtensionData.HAEvent.HAConfig.HANotResponding}}

# Check vCenter alarms:
Get-Alarm -Cluster "Prod-Cluster-01" | Where-Object {$_.State -ne "Green"} | 
  Select-Object Name, State, OldStatus, NewStatus, MonitoringObject

# Check all HA events across cluster:
Get-Event -Cluster "Prod-Cluster-01" -MaxEvents 200 | 
  Where-Object {$_.FullName -like "*HA*" -or $_.FullName -like "*isolate*"} | 
  Select-Object CreatedTime, FullFormattedMessage | Format-Table -Wrap

# From ESXi:
# Check HA agent:
esxcli software vib list | grep -i ha
# Check host agent:
esxcli system ha running get
esxcli system ha cluster get

# Check HA network:
esxcli network ip interface list | grep vmk0
# Verify management network reachable from other hosts

# Check isolation address:
esxcli system ha cluster get | grep -i "isolation"

# Check isolation response:
cat /etc/ha/ha.conf | grep -i isolate

# Check VM restart attempts:
vim-cmd vmsvc/power.getstate <VMID>  # Check VM state
esxcli system ha vm list  # List VMs managed by HA

# Check log:
tail -f /var/log/ha.log
cat /var/log/fdm.log   # Fault Domain Manager log
cat /var/log/vmkernel  # Look for HA-related messages

# Check host connectivity (from another host):
ping <isolated-host-ip>
ping <vCenter-ip>  # vCenter is not required for HA, but if isolated depends on management network
```

### Recovery
- HA not restarting VMs → Check admission control, resources on remaining hosts, VM restart priority
- HA false positive → Check isolation response settings (default: power off + restart; options: power off, do nothing, restart)
- HA agent not running → Restart: `esxcli system ha cluster restart`
- All hosts isolated → Physical network recovery needed; cannot resolve at VMware level
- VMs not restarting after HA → Check VM configuration: ensure restart action is "Restart" not "Power Off"

### Interview Answer
> "HA is automatic VM restart on host failure. HA monitors hosts via network heartbeat (isolation detection) and VM tools heartbeat. When a host is detected as failed, HA calculates available resources on remaining hosts, checks admission control, and restarts VMs in priority order (High first) on the host with most free capacity. Common issues: insufficient resources for HA restart, HA disabled, VMware Tools not running (VM heartbeat lost), false positive isolation (network flapping). I ensure HA is configured with admission control (25-50% failover capacity), multiple isolation addresses, datastore heartbeating, and proper NIC teaming on management network."

---

## 9. DRS (Distributed Resource Scheduler)

### What
**DRS** is a cluster-level feature that **automatically balances** VM compute workloads across ESXi hosts in a cluster. It monitors CPU and memory utilization on each host and, when imbalance is detected, uses **vMotion** to migrate VMs to more appropriate hosts.

### Why
- **Load balancing**: Spread VMs evenly across hosts
- **Resource optimization**: Prevent hosts from being overloaded while others are idle
- **Maintenance**: Evacuate a host for maintenance (DRS can move all VMs off)
- **Proactive**: Predicts future load and moves VMs before problems occur
- **Business continuity**: Automatically handles resource spikes

### DRS Modes
| Mode | Behavior |
|------|----------|
| **Fully Automated** | DRS makes recommendations AND executes them automatically |
| **Semi-Automatic** | DRS makes recommendations; admin must approve and execute |
| **Manual** | DRS only shows recommendations; admin executes manually |

### DRS Tiers (Automation Levels per VM)
| Tier | Meaning |
|------|---------|
| **Fully Automated** | DRS can vMotion this VM automatically |
| **Manual** | DRS recommends but doesn't execute |
| **Disabled** | DRS ignores this VM for balancing |

### Architecture
```
┌──────────────────────────────────────────────────────────────┐
│                    Cluster with DRS                          │
│  DRS runs on EVERY host (Master election among hosts)        │
│  DRS Resource Manager checks every 5 minutes (default)      │
│                                                              │
│  ┌──────────┐         ┌──────────┐         ┌──────────┐     │
│  │ ESXi 01  │         │ ESXi 02  │         │ ESXi 03  │     │
│  │ CPU: 80% │         │ CPU: 20% │         │ CPU: 40% │     │
│  │ RAM: 85% │         │ RAM: 30% │         │ RAM: 50% │     │
│  └──────────┘         └──────────┘         └──────────┘     │
│       ↑                     ↑                     ↑         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  DRS Evaluation Cycle (every 5 min):                 │   │
│  │  1. Collect CPU/RAM metrics from all hosts            │   │
│  │  2. Calculate imbalance score                         │   │
│  │  3. If imbalance > threshold (default 5%):            │   │
│  │     a. Find best VM to vMotion                       │   │
│  │     b. Calculate target host (most free)              │   │
│  │     c. Check DRS rules (affinity/anti-affinity)       │   │
│  │     d. Generate recommendation                       │   │
│  │     e. Execute (if fully automated)                  │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  DRS Rules:                                                  │
│  • VM/Host Group: Group VMs and hosts                        │
│  • VM/VM Affinity: Keep VMs together (e.g., cluster nodes)   │
│  • VM/VM Anti-Affinity: Separate VMs (e.g., DCs in different hosts) │
│  • VM/Host Affinity: VM must run on specific host group      │
│  • VM/Host Must Run: VM must run on specific host            │
└──────────────────────────────────────────────────────────────┘
```

### DRS Score Calculation
DRS uses a **balancing score** (0-100, 100 = perfectly balanced):
- Checks CPU and memory utilization across all hosts
- Calculates standard deviation of utilization
- If deviation > threshold (default 5% for CPU, 5% for memory) → generates recommendation
- Also considers: affinity rules, VM overrides, migration cost (network, storage access, etc.)

### Dependencies
```
DRS depends on:
├── Cluster (min 2 hosts)
├── vMotion (to migrate VMs between hosts)
├── Shared storage (VM files accessible from all hosts — REQUIRED)
├── vCenter (DRS runs through vCenter initially, but DRS agent runs on hosts too)
├── Network connectivity between hosts (for vMotion)
├── Resources on target host (enough CPU/RAM for VM being migrated)
├── DRS rules (affinity/anti-affinity may block migrations)
├── HA/DRS interactions (DRS must respect VM/Host rules)
├── EVC mode (CPU compatibility across hosts)
└── VM restart priority / HA restrictions

DRS is depended upon by:
├── Cluster resource efficiency
├── VM performance (balanced across hosts)
├── Maintenance mode (DRS evacuates host)
├── Capacity planning
└── vRealize Operations (capacity planning integration)
```

### Failure
| Failure | Impact |
|---------|--------|
| DRS disabled | VMs unevenly distributed; manual balancing only |
| DRS recommendations not executing | VMs stay on overloaded hosts (semi-auto mode; admin hasn't approved) |
| vMotion blocked by DRS rules | Affinity/anti-affinity rules prevent optimal placement |
| Insufficient resources on target host | DRS cannot migrate; VMs stay on overloaded host |
| DRS migration cost too high | VM won't migrate (storage access, etc.) |
| DRS agent crash | No balancing; uneven distribution persists |
| DRS and HA conflict | VM restarted by HA, then DRS may move it again |
| VM retention policy (VM/Host must run) | VM stays on specific host regardless of load |
| Cross-vCenter DRS not configured | VMs only balanced within single vCenter |

### Symptoms
- VMs concentrated on few hosts; others idle
- Host performance degraded (high CPU/RAM on specific hosts)
- "DRS should migrate" recommendation not executed
- vMotion blocked by DRS rules
- DRS event log shows "insufficient resources"

### Troubleshooting
```powershell
# PowerCLI:
# Check DRS enabled:
Get-Cluster "Prod-Cluster-01" | Select-Object Name, 
  @{N='DRSEnabled';E={$_.ExtensionData.Configuration.DrsConfigInfo.DrsEnabled}},
  @{N='DRSMode';E={$_.ExtensionData.Configuration.DrsConfigInfo.Level}}  # FullyAutomated, etc.

# Get DRS recommendations:
Get-DrsRecommendation -Cluster "Prod-Cluster-01" | 
  Select-Object Name, Reason, @{N='Actions';E={$_.Actions}}

# Get DRS VM/Host rules:
Get-DrsRule -Cluster "Prod-Cluster-01" | Select-Object Name, RuleType, Enabled

# Check which host each VM is on:
Get-Cluster "Prod-Cluster-01" | Get-VM | 
  Select-Object Name, @{N='VMHost';E={$_.VMHost.Name}}, 
    @{N='CPUUtilization';E={$_.ExtensionData.ResourceConfigOverrides.HostCpuUsage}},
    @{N='MemoryUtilization';E={$_.ExtensionData.ResourceConfigOverrides.HostMemoryUsage}} |
  Format-Table -AutoSize

# Check host utilization:
Get-VMHost | Select-Object Name, 
  @{N='CPUUsage%';E={[math]::Round(($_.ExtensionData.Host.Hardware.CpuUsage/$_.ExtensionData.Host.Hardware.CpuCount)*100,2)}},
  @{N='MemoryUsage%';E={[math]::Round(($_.ExtensionData.Host.MemoryUsage/1GB)/($_.ExtensionData.Host.Hardware.MemoryCapacity/1GB)*100,2)}},
  @{N='NumVMs';E={($_ | Get-VM).Count}} |
  Format-Table -AutoSize

# Force DRS to rebalance:
Get-Cluster "Prod-Cluster-01" | Set-Cluster -DrsEnabled:$true -DrsAutomationLevel FullyAutomated

# Check DRS event log:
Get-Cluster "Prod-Cluster-01" | Get-VIEvent | 
  Where-Object {$_.FullName -like "*DRS*" -or $_.FullName -like "*drs*"} |
  Select-Object CreatedTime, FullFormattedMessage | Format-Table -Wrap

# Check migration history:
Get-DrsRecommendation -Cluster "Prod-Cluster-01" -ShowDetails | 
  Select-Object Name, Reason, @{N='VMsToMove';E={$_.Actions | Where-Object {$_.Type -eq 'Vmmigrate'}}}

# From vCenter:
# Cluster > Monitor > DRS (shows current balancing, recommendations)
# Cluster > Monitor > vMotion (shows migration history)
```

### Recovery
- DRS not balancing → Enable DRS, set to Fully Automated, check vMotion network, check resources
- DRS recommendations not executing → Check automation level, check DRS rules blocking migration, approve recommendations
- DRS conflicts with HA → Check VM restart priority; DRS respects HA decisions initially
- Insufficient resources → Add hosts, increase capacity, reduce VM resource allocation

### Interview Answer
> "DRS automatically balances VMs across hosts using vMotion. It checks CPU/RAM every 5 minutes and when imbalance exceeds threshold, it recommends or executes migrations. Key dependencies: vMotion, shared storage, vCenter. Common issues: DRS disabled, semi-auto mode needing manual approval, DRS rules blocking migrations, insufficient resources on target hosts. I monitor via vCenter DRS dashboard and PowerCLI Get-DrsRecommendation. I ensure DRS is fully automated with proper resource thresholds."

---

## 10. vMotion

### What
**vMotion** is the live migration of a running VM from one ESXi host to another with **zero downtime** — the VM continues operating with no perceptible interruption to users or applications.

### Why
- **Live maintenance**: Patch/reboot hosts without VM downtime
- **Load balancing**: Move VMs to hosts with more resources (DRS uses vMotion)
- **Hardware failure recovery**: Move VMs off failing hosts
- **Datacenter operations**: Physical relocation of VMs between chassis

### Architecture (Technical Detail)
```
Step-by-step vMotion Process:

PHASE 1: PRE-COPY (Memory pages)
├── Source host sends VM config to target host
├── Source host starts copying ALL memory pages to target host
├── While copying, VM continues running on source
├── Changes to memory are tracked (bitmap/dirty pages)

PHASE 2: CONTINUOUS COPY
├── Source host continues running VM
├── Additional memory changes are copied to target
├── Iterative process: copy changes since last copy
├── Network traffic flows to source host (until cutover)
├── Storage: VM files accessed from shared storage (same datastore)
│   (Or: vMotion WITH Storage vMotion: copy storage to target datastore too)

PHASE 3: CUTOVER (Brief pause, <1-2 seconds)
├── Source host pauses VM execution
├── Remaining dirty pages copied to target (should be very few)
├── VM state transferred to target host
├── Network traffic re-routed to target host (MAC address move, virtual NIC migration)
├── Target host resumes VM execution
└── Source host resumes VM as stopped/cleared state

Overall downtime: < 1-2 seconds (user may see brief network disconnect)

NETWORK REQUIREMENTS:
├── vMotion VMkernel adapter (dedicated vmk1 or similar)
├── vMotion TCP port: 8000 (default), configurable
├── Source and target hosts must be able to reach each other on port 8000
├── 10Gbps recommended (not required but strongly advised)
├── NIC teaming / redundancy for vMotion network
├── MTU: Jumbo frames recommended for large VMs
├── vMotion requires TCP/IP networking on all hosts
├── Cross-vCenter vMotion: requires vCenter cross-vCenter SSO trust
└── Storage vMotion: requires both source and target datastores accessible simultaneously

STORAGE REQUIREMENTS:
├── For vMotion (compute only): Shared storage accessible from both hosts (same datastore)
├── For Storage vMotion: Target datastore must be accessible from target host
├── For Storage vMotion: Can move between VMFS and NFS, different datastores
```

### Dependencies
```
vMotion depends on:
├── Two ESXi hosts (source and target)
├── vMotion VMkernel adapter (vmk1 or dedicated)
├── vMotion network (TCP port 8000 between hosts)
├── vMotion license (Enterprise Plus or higher)
├── Compatible CPU (EVC mode or same CPU vendor)
├── Shared storage (same datastore accessible by both hosts) — for compute-only vMotion
├── For Storage vMotion: target datastore accessible from target host
├── VM compatibility (hardware version 4+; specific vMotion versions)
├── Network: MAC address migration (virtual switch must support MAC move)
├── NIC: Dedicated vMotion NICs (recommended)
├── Bandwidth: Sufficient for memory size (10Gbps ideal for large VMs)
├── vCenter (orchestration, though vMotion API can be used directly)
├── SSL certificates (VMhost-to-VMhost SSL trust)
└── MTU consistency (if using jumbo frames)

vMotion is depended upon by:
├── DRS (automated balancing)
├── HA (VM restart on different host — vMotion can preserve state better)
├── Cluster maintenance (evacuate host)
├── Storage vMotion (storage migration)
└── Cross-site DR (Long-distance vMotion)
```

### Failure
| Failure | Impact |
|---------|--------|
| vMotion network unreachable | Migration times out; VM stays on source host |
| CPU incompatibility | vMotion fails; EVC mode resolution needed |
| Insufficient memory on target | VM cannot be migrated |
| Shared storage not accessible from target | vMotion fails (target can't access VM files) |
| MAC address conflict | Network interruption during migration |
| VM compatibility version too low | vMotion not allowed |
| SSL certificate error | vMotion connection refused between hosts |
| Network partitioning during vMotion | VM may be accessible on both or neither hosts |
| Large VM slow vMotion | Extended migration time; user impact during cutover |
| Port 8000 blocked | vMotion cannot establish TCP connection |
| VM with local (non-shared) disk | Compute vMotion fails; must use Storage vMotion |
| Snapshot during vMotion | Complex; may need to consolidate first |
| Host lockdown mode | vMotion API access blocked |

### Symptoms
- vMotion times out
- "A general system error occurred: Host is not in the same vMotion network"
- "vMotion was cancelled" 
- "Host does not support the feature"
- "Migration failed: Cannot access the designated datastore"
- Slow migration (large VMs over slow network)
- VM temporarily inaccessible during migration

### Troubleshooting
```powershell
# PowerCLI:
# Test vMotion connection:
Test-vMotionConnection -Server ESXi01.contoso.com -Destination ESXi02.contoso.com

# Check vMotion VMkernel:
Get-VMHostNetworkAdapter -VMHost ESXi01 | Where-Object {$_.VMKernel -eq $true} | 
  Select-Object VMHost, Name, IP, SubnetMask, @{N='MTU';E={$_.Mtu}}

# Check vMotion enabled on host:
Get-VMHostService -VMHost ESXi01 | Where-Object {$_.Key -eq 'vmotion'} | Select-Object Key, Running, Policy

# Start/stop vMotion service:
Start-VMHostService -HostService (Get-VMHostService -VMHost ESXi01 | Where-Object {$_.Key -eq 'vmotion'})

# Check vMotion network port:
Test-NetConnection ESXi02.contoso.com -Port 8000

# Check EVC/CPU compatibility:
Get-VMHost ESXi01, ESXi02 | Select-Object Name, CpuType, CpuVersion, CpuMHz
Get-Cluster "Prod-Cluster-01" | Select-Object -ExpandProperty EVCMode

# Check VM compatibility for vMotion:
Get-VM VM01 | Select-Object Name, 
  @{N='CompatibleWithvMotion';E={$_.ExtensionData.Summary.Config.VirtioFwInfo.VmotionSupported}}

# Check available datastores from both hosts:
Get-Datastore | Where-Object { (Get-VMHost -Id $_.ExtensionData.Info.UniqueId) -contains ESXi01 } | Select-Object Name
Get-Datastore | Where-Object { (Get-VMHost -Id $_.ExtensionData.Info.UniqueId) -contains ESXi02 } | Select-Object Name

# Check vMotion network throughput:
esxcli network ip interface get -i vmk1
# Use esxtop/dvFilter stats for real-time vMotion throughput

# From ESXi:
# Check vMotion network config:
esxcli network ip interface ipv4 get -i vmk1
esxcli network ip interface ipv4 set -i vmk1 --ipv4=10.1.1.10/24

# Check vMotion port:
netstat -an | grep 8000
ss -tlnp | grep 8000

# Check vMotion logs:
/var/log/vmware/vmotion/vmkernel.log   # on ESXi
var/log/vmkernel.log | grep -i motion

# Check if vMotion is using specific port:
esxcli network firewall ruleset list | grep -i motion
esxcli network firewall ruleset allowedip list -r vmotion

# From vCenter:
# Recent vMotion:
Get-Cluster "Prod-Cluster-01" | Get-VM | 
  Select-Object Name, @{N='LastVMotion';E={$_.ExtensionData.RecentTaskSummary}}, 
    @{N='MigrateHistory';E={$_.ExtensionData.RecentTaskSummary}}

# Check migration history:
Get-Task | Where-Object { $_.Name -like "*vMotion*" -or $_.Name -like "*Migrate*" } | 
  Select-Object Name, State, CreatedTime | Sort-Object CreatedTime -Descending | Select-Object -First 20

# Check for failed vMotion:
Get-Task | Where-Object { $_.Name -like "*vMotion*" -and $_.State -eq "Error" } | 
  Select-Object Name, Error, CreatedTime | Format-Table -Wrap

# Manual vMotion:
Move-VM -VM VM01 -Destination ESXi02 -Datastore datastore01 -DiskStorageFormat Thin
Move-VM -VM VM01 -Destination "Prod-Cluster-01" -Datastore datastore02
```

### Recovery
- vMotion fails → Check vMotion network connectivity, VMkernel adapter, port 8000, CPU compatibility, datastores accessible from both hosts
- CPU incompatibility → Enable EVC mode on cluster matching both hosts' CPU features
- Network issue → Check physical NIC, cable, switch config, firewall, routing
- Storage inaccessible → Remount datastore on target host, check multipathing
- Timeouts on large VMs → Upgrade network to 10Gbps, use jumbo frames, schedule during maintenance window

### Interview Answer
> "vMotion live-migrates running VMs between hosts with sub-second downtime. It works by copying memory pages iteratively, then doing a brief cutover (1-2 seconds). It requires: dedicated vMotion VMkernel adapter, port 8000 TCP, shared storage accessible by both hosts, and compatible CPUs (EVC mode). Common failures: network issues, CPU incompatibility, insufficient target resources, firewall blocking port 8000. I test vMotion connectivity with Test-vMotionConnection and verify using esxcli. For large VMs over WAN, I use Long-Distance vMotion or Hyper-V Replica (if mixed environment)."

---

## 11. Storage vMotion

### What
**Storage vMotion** is the live migration of a VM's disk files (VMDKs) from one datastore to another while the VM continues running. Unlike vMotion (which moves compute), Storage vMotion moves storage.

### Why
- **Storage migration**: Move VMs from one storage tier to another (e.g., high-performance to archive)
- **Datastore maintenance**: Migrate VMs off a datastore for maintenance without VM downtime
- **Storage rebalancing**: Distribute VMs across datastores evenly
- **Storage type migration**: VMFS ↔ NFS migration
- **Thin/Thick conversion**: Change disk format during migration
- **Delete snapshots**: Move to a new datastore and consolidate

### Architecture
```
Storage vMotion Process:

VM running on ESXi Host A, accessing Datastore 1:
┌──────────┐                     ┌──────────────┐
│ ESXi 01  │                     │  Datastore 1 │
│ VM01     │ ──── Reads/Writes ──│  (VM files)  │
│ Running  │                     │  VM01.vmdk   │
└──────────┘                     │  VM01-flat.vmdk │
                                 └──────────────┘

PHASE 1: Copy phase
┌──────────┐                     ┌──────────────┐
│ ESXi 01  │ ── Copy ──────────→│  Datastore 2 │
│ VM01     │    (background)     │  VM01.vmdk   │
│ Still    │                     │  VM01-flat.vmdk (COPY)│
│ reading  │                     └──────────────┘
│ from DS1 │
└──────────┘

PHASE 2: Switch phase (brief pause)
┌──────────┐                     ┌──────────────┐
│ ESXi 01  │    PAUSE I/O        │  Datastore 1 │  (still has old files)
│ VM01     │                     └──────────────┘
│          │ ── Redirect ──────────────────→ ┌──────────────┐
│          │                                  │  Datastore 2 │
│          │                                  │  VM01.vmdk   │  (now new active)
└──────────┘                     (Old files on DS1 can be deleted)

PHASE 3: Resume
┌──────────┐                     ┌──────────────┐
│ ESXi 01  │ ← Resumes I/O ──────│  Datastore 2 │
│ VM01     │                     │  (all files) │
│ Running  │                     └──────────────┘
└──────────┘

Key: VM never stops! I/O is redirected during switch phase.
Switch phase duration: typically < 1 second for most VMs.
```

### Dependencies
```
Storage vMotion depends on:
├── Two datastores (source and target) accessible from the SAME host (or via vMotion across hosts)
├── Host with vMotion capability (storage vMotion uses vMotion infrastructure)
├── License (Enterprise Plus or higher)
├── Source VM running
├── Shared storage OR a host that can see both datastores
├── vMotion VMkernel network (for host-to-host communication if source/target datastores on different hosts)
├── Datastore types compatible (VMFS to VMFS, VMFS to NFS, NFS to NFS, NFS to VMFS)
├── Space on target datastore (target must have space for all VM files including delta disks)
├── Firewall: port 8000 (vMotion) between hosts if cross-host
├── VM compatibility (hardware version)
├── Snapshot consideration: VMs with snapshots can use Storage vMotion
└── Multipathing on both datastores
```

### Failure
| Failure | Impact |
|---------|--------|
| Insufficient space on target | Storage vMotion fails; VM stays on source |
| Target datastore incompatible | VMFS ↔ NFS specific checks; incompatible formats fail |
| Datastore inaccessible on target | Cannot create VMDK files on target |
| vMotion network timeout | Migration fails mid-copy; VM stays on source (usually) |
| VM with RDM (Raw Device Mapping) | RDM+LUN mapping complicates migration; physical mode RDMs are more complex |
| Thick disk on full datastore | Cannot copy thick disk to target |
| Active snapshots large | Large delta disks need to be copied too; long migration time |
| NFS server unresponsive | NFS-based datastore migration may stall |
| Storage I/O contention | Source and target storage systems under load; slow migration |
| Cross-host storage vMotion | Complex; requires vMotion networking and both hosts seeing both datastores |

### Symptoms
- Storage vMotion task stuck or timed out
- "Target datastore has insufficient space"
- "Cannot migrate VM: datastore incompatible"
- VM latency spike during migration (brief I/O pause during switch)
- Storage vMotion fails; VM still on source datastore

### Troubleshooting
```powershell
# PowerCLI:
# Initiate Storage vMotion:
Move-VM -VM VM01 -Datastore datastore02 -DiskStorageFormat Thin -Confirm:$false

# Check storage vMotion tasks:
Get-Task | Where-Object { $_.Name -like "*Storage vMotion*" -or $_.Name -like "*StorageM*" } | 
  Select-Object Name, State, CreatedTime, TargetObject

# Check datastore free space:
Get-Datastore datastore01, datastore02 | 
  Select-Object Name, FreeSpaceGB, CapacityGB, 
    @{N='Used%';E={[math]::Round(($_.CapacityGB - $_.FreeSpaceGB)/$_.CapacityGB*100,1)}}

# Check target datastore accessible from host:
Get-VMHost ESXi01 | Get-Datastore datastore02 | Select-Object Name, CimStoreType, @{N='Mounted';E={$_.ExtensionData.Info.Mounted}}

# Check if datastore is NFS:
Get-Datastore | Where-Object { $_.Type -eq "NFS" } | Select-Object Name, Url

# Check VM disk format:
Get-VM VM01 | Get-HardDisk | Select-Object Name, Filename, StorageFormat, DiskType, CapacityGB

# Check if VM has snapshots (will need to copy delta disks too):
Get-VM VM01 | Get-Snapshot | Select-Object Name, Created, VM

# Check migration from vCenter:
# Cluster > Monitor > Storage vMotion (if applicable)

# From ESXi:
# Check storage vmotion on host:
esxcli storage vmotion get
# Check if vmotion network is configured:
esxcli network ip interface get -i vmk1

# Check for errors:
/var/log/vmware/vmotion/vmkernel.log
grep -i "storage.*migration" /var/log/vmkernel.log

# Check NFS specific:
esxcli storage nfs list
# Ensure NFS mount is active on both hosts

# Check iSCSI/FC paths to target datastore:
esxcli storage core path list
esxcli storage core device list -d <device>

# Check multipathing state:
esxcli storage nmp device list -d <device>
```

### Recovery
- Insufficient space → Clean up target datastore, delete old VMs/snapshots, expand datastore, use thin format
- Incompatible datastore → Convert disk format during migration (Move-VM with -DiskStorageFormat option), or convert source disk first
- Network timeout → Check vMotion network, retry migration during off-peak
- NFS issue → Remount NFS on target host, check NFS server health
- Migration stuck → Cancel task, check VM state, retry

### Interview Answer
> "Storage vMotion live-migrates VM disk files between datastores without VM downtime. It copies VMDK files to the target datastore while the VM keeps reading/writing from the source, then briefly pauses I/O (sub-second) to switch the VM to the new datastore. It requires both datastores accessible from the host, vMotion license, and sufficient space on target. Common issues: insufficient space, incompatible datastore types, NFS server down, thick disks on limited storage. I use Move-VM -Datastore and verify with Get-Datastore free space checks. I prefer thin format during migration to reduce data transfer."

---

## 12. Resource Pools

### What
**Resource Pools** are logical partitions of CPU and memory resources within an ESXi host or cluster. They create a hierarchy for allocating and controlling how resources are distributed among VMs.

### Why
- **Guarantee resources**: Ensure critical VMs get minimum CPU/RAM
- **Limit resources**: Prevent VMs from consuming all resources
- **Share resources**: Define how VMs share available resources
- **Hierarchical allocation**: Parent pool → child pools → VMs (multi-level hierarchy)

### Architecture
```
Cluster "Prod-Cluster-01" (Parent)
│ Total CPU: 120 cores, Total RAM: 1.5 TB
│
├── Resource Pool "Production"
│   │ CPU: 60 cores (75%), RAM: 1 TB (75%)
│   │ Shares: High
│   │
│   ├── Resource Pool "Database"
│   │   │ CPU: 30 cores (50% of Production)
│   │   │ RAM: 500 GB (50% of Production)
│   │   │ Shares: High
│   │   │
│   │   ├── VM01 (SQL01): CPU 8 cores, RAM 128 GB
│   │   ├── VM02 (SQL02): CPU 8 cores, RAM 128 GB
│   │   └── VM03 (DB01): CPU 4 cores, RAM 64 GB
│   │
│   └── Resource Pool "Web"
│       │ CPU: 30 cores, RAM: 500 GB
│       │ Shares: Normal
│       │
│       ├── VM04 (Web01): CPU 4 cores, RAM 32 GB
│       └── VM05 (Web02): CPU 4 cores, RAM 32 GB
│
└── Resource Pool "Development"
    │ CPU: 15 cores (25%), RAM: 375 GB (25%)
    │ Shares: Low
    │
    ├── VM06 (Dev01): CPU 4 cores, RAM 32 GB
    └── VM07 (Dev02): CPU 4 cores, RAM 32 GB
```

### Key Settings
| Setting | What | Behavior |
|---------|------|----------|
| **Reservation** | Guaranteed minimum | VM/Pool ALWAYS gets this amount, even if host is overloaded |
| **Limit** | Maximum allowed | VM/Pool CANNOT exceed this, even if host has free resources |
| **Shares** | Relative priority | When resources are contested, shares determine who gets more |

### Shares Levels vs. Values
| Level | CPU Shares (per vCPU) | Memory Shares (per MB) |
|-------|-----------------------|----------------------|
| **Low** | 500 | 512 |
| **Normal** | 1000 | 1024 |
| **High** | 2000 | 2048 |

### Shares Calculation Example:
- Pool A: 3 VMs, each Normal (1000 shares) → 3000 total shares
- Pool B: 2 VMs, each High (2000 shares) → 4000 total shares
- If only 1000 CPU MHz available:
  - Pool A gets: 1000 * (3000/7000) = 429 MHz
  - Pool B gets: 1000 * (4000/7000) = 571 MHz

### Dependencies
```
Resource Pools depend on:
├── ESXi host or cluster
├── vCenter (create/manage pools via vCenter or PowerCLI)
├── Physical CPU and memory resources
├── VM assignments (VMs must be assigned to pools)
├── Other pools (hierarchy)
├── Admission control (if reservation cannot be met, VM cannot power on)
├── DRS (recognizes resource pools for placement decisions)
└── HA (recognizes resource pool reservations for failover)
```

### Interview Answer
> "Resource Pools are logical partitions of CPU and memory. They use three mechanisms: Reservation (guaranteed minimum), Limit (maximum ceiling), and Shares (relative priority when contested). I use pools to guarantee critical VMs (databases) have minimum resources, limit noisy neighbors (development), and set priority (high for production, low for dev). Key gotcha: reservations are honored even during host overload, so setting too high can block VM power-ons. I verify resource pool health via vCenter: Cluster > Resources, and PowerCLI: Get-ResourcePool."

---

## 13. CPU

### Key Concepts

| Concept | What | Why It Matters |
|---------|------|---------------|
| **vCPU** | Virtual CPU (1 vCPU = 1 pCPU core from guest perspective; can be overcommitted) | Guest OS sees dedicated CPU |
| **pCPU** | Physical CPU core (or thread with Hyper-Threading) | Real compute resource |
| **Overcommitment** | vCPUs > pCPUs (common, e.g., 2:1 or 4:1 ratio) | More VMs per host; risk of CPU contention |
| **CPU Ready Time** | Time VM is ready to run but waiting for pCPU | High CPU ready = CPU contention; >5-10% is concerning |
| **CPU Co-stopping** | Time VM ready but scheduler paused it for another VM | Similar to CPU ready; indicates contention |
| **EVC Mode** | Equalize CPU features across hosts | Enables vMotion between hosts with different CPU generations |
| **CPU Affinity** | Bind VM to specific pCPU/cores | Prevents NUMA issues; rarely needed |
| **CPU Workload Priority** | Normal/High/Low/Custom | Scheduling priority for CPU cycles |

### Troubleshooting High CPU
```powershell
# vCenter: VM > Monitor > Performance > CPU > Chart
# Check: CPU Usage %, CPU Ready Time %, CPU Co-stopping %

# PowerCLI:
Get-VM VM01 | Select-Object Name, 
  @{N='CPUUsageMHz';E={$_.ExtensionData.Summary.Runtime.HostCpuUsage}},
  @{N='CPULimitMHz';E={$_.ExtensionData.Summary.Runtime.MaxCpuUsage}},
  @{N='NumCPUs';E={$_.NumCpu}},
  @{N='CPUReady';E={$_.ExtensionData.PerfStats.CpuReady}},
  @{N='CPUcoStop';E={$_.ExtensionData.PerfStats.CpuCoStop}}

# Check CPU on host:
Get-VMHost ESXi01 | Select-Object Name, 
  @{N='CPUUsage%';E={[math]::Round($_.ExtensionData.Host.Hardware.CpuUsage/$_.ExtensionData.Host.Hardware.CpuCount*100,2)}},
  @{N='NumVMs';E={($_ | Get-VM).Count}},
  @{N='OverheadCPU';E={$_.ExtensionData.Host.CpuOverhead}}

# High CPU ready across cluster:
Get-Cluster "Prod-Cluster-01" | Get-VM | 
  Where-Object { $_.ExtensionData.PerfStats.CpuReady -gt 10 } | 
  Select-Object Name, @{N='CPUReady%';E={$_.ExtensionData.PerfStats.CpuReady}}

# Check for CPU limit set:
Get-VM | Where-Object { $_.ExtensionData.Config.Hardware.CpuLimit -ne -1 } | 
  Select-Object Name, @{N='CPULimit';E={$_.ExtensionData.Config.Hardware.CpuLimit}}

# Check CPU reservation:
Get-VM | Where-Object { $_.ExtensionData.Config.Hardware.CpuReservation -gt 0 } | 
  Select-Object Name, @{N='CPUReservationMHz';E={$_.ExtensionData.Config.Hardware.CpuReservation}}

# ESXi top:
esxtop  # Press 'c' for CPU view, 'V' for VM view
# Look for %RDY (ready time) and %CST (co-stop time)

# Check if VM has CPU affinity set:
Get-VM VM01 | Select-Object Name, NumCpu, @{N='CpuAffinity';E={$_.ExtensionData.Config.Hardware.CpuAffinity}}

# Check EVC mode:
Get-Cluster "Prod-Cluster-01" | Select-Object -ExpandProperty EVCMode

# Reduce vCPUs if overcommitted:
Set-VM -VM VM01 -NumCpu 2 -Confirm:$false
# (Reduce from 4 to 2 if guest only uses 2 cores)
```

### Recovery
- High CPU ready → Reduce vCPUs (sometimes more vCPUs = more contention), migrate via DRS to less loaded host, check for CPU limit/reset configured
- CPU co-stopping → Check if host has CPU overcommitment too high, reduce overcommit ratio, check for CPU affinity issues
- VM not getting enough CPU → Check CPU reservation, increase reservation or shares, reduce other VM allocations

---

## 14. Memory

### Key Concepts

| Concept | What | Why It Matters |
|---------|------|---------------|
| **vRAM** | Virtual RAM (VM sees) | Guest OS memory allocation |
| **pRAM** | Physical RAM | Real resource |
| **Memory Overcommitment** | vRAM > pRAM (using ballooning, swapping, compression) | More VMs than physical RAM |
| **Ballooning (vm Balloon Driver)** | VMware Tools driver reclaims memory from guest OS | Guest OS unaware; balloons inside VM |
| **Swapping** | ESXi swaps VM memory to disk (last resort) | Severe performance degradation |
| **Memory Compression** | ESXi compresses memory pages before swapping | Better than swap; reduces I/O |
| **Transparent Page Sharing (TPS)** | Shares identical memory pages between VMs | Memory deduplication (deprecated in newer versions due to security) |
| **Memory Reservation** | Guaranteed minimum physical RAM | VM always has this RAM, no ballooning |
| **Memory Limit** | Maximum RAM VM can use | VM cannot exceed this |
| **Memory Shares** | Priority when contested | Relative allocation when memory is scarce |
| **MMU (Memory Management Unit)** | Hardware-assisted paging | Virtual-to-physical address translation |

### ESXi Memory Management (Hierarchical):
```
Memory Request from VM
│
├── 1. If reservation met → Allocate dedicated memory
│
├── 2. If memory available on host → Allocate from free pool
│
├── 3. If no free memory:
│   ├── 3a. Try TPS (transparent page sharing — deduplicate identical pages)
│   ├── 3b. Try Ballooning (vm balloon driver inside guest reclaims memory)
│   │   └── Guest OS decides what to page out (files, cache, etc.)
│   ├── 3c. Try Memory Compression (compress infrequently used pages)
│   └── 3d. Swap to disk (VM swap file .vswp on datastore — LAST RESORT)
│
└── Result: VM memory satisfied (though possibly with degradation)

.vswp file:
- Created when VM powers on
- Size = (vRAM - memory reservation)
- If reservation = vRAM → no .vswp file
- Swap file location: Datastore where VM is registered
```

### Troubleshooting High Memory Usage
```powershell
# vCenter: VM > Monitor > Performance > Memory
# Check: Memory Usage %, Balloon Driver %, Swapping %, Compressed %

# PowerCLI:
Get-VM VM01 | Select-Object Name, 
  @{N='MemoryUsageGB';E={[math]::Round($_.ExtensionData.Summary.Runtime.HostMemoryUsage/1GB,2)}},
  @{N='MemoryBalloonPct';E={$_.ExtensionData.Summary.Memory.Balloon}},
  @{N='MemorySwappedPct';E={$_.ExtensionData.Summary.Memory.Swap}},
  @{N='MemoryCompressedPct';E={$_.ExtensionData.Summary.Memory.Compression}},
  @{N='MemoryOverheadGB';E={[math]::Round($_.ExtensionData.Summary.Memory.Overhead/1GB,2)}}

# Check host memory:
Get-VMHost ESXi01 | Select-Object Name, 
  @{N='MemoryUsage%';E={[math]::Round($_.ExtensionData.Host.MemoryUsage/1GB/($_.ExtensionData.Host.Hardware.MemoryCapacity/1GB)*100,1)}},
  @{N='OverheadMB';E={$_.ExtensionData.Host.MemoryOverhead}},
  @{N swappedMB';E={$_.ExtensionData.Host.MemorySwapUsed}},
  @{N='BalloonedMB';E={$_.ExtensionData.Host.MemoryBallooned}}

# Check VMs causing most memory usage:
Get-VM | 
  Sort-Object -Property @{E={$_.ExtensionData.Summary.Runtime.HostMemoryUsage};D='Descending'} | 
  Select-Object Name, @{N='MemoryGB';E={[math]::Round($_.ExtensionData.Summary.Runtime.HostMemoryUsage/1GB,2)}}, 
    @{N='Balloon%';E={$_.ExtensionData.Summary.Memory.Balloon}},
    @{N='Swap%';E={$_.ExtensionData.Summary.Memory.Swap}} |
  Select-Object -First 20 | Format-Table -AutoSize

# Check .vswp file sizes:
Get-VM | Select-Object Name, @{N='vswpSizeGB';E={
  ($_.ExtensionData.Summary.Config.Hardware.MemoryMB - $_.ExtensionData.Config.Hardware.MemoryReservationMB)/1024
}} | Sort-Object vswpSizeGB -Descending | Format-Table -AutoSize

# ESXi top:
esxtop  # Press 'm' for memory view
# Look for: SWAP/PIG (swap rate), BAL (balloon), CMP (compress)

# Check memory reservation:
Get-VM | Where-Object { $_.ExtensionData.Config.Hardware.MemoryReservation -gt 0 } | 
  Select-Object Name, @{N='MemReservationMB';E={$_.ExtensionData.Config.Hardware.MemoryReservation}}

# Check if memory limit set:
Get-VM | Where-Object { $_.ExtensionData.Config.Hardware.MemoryLimit -ne -1 } | 
  Select-Object Name, @{N='MemLimitMB';E={$_.ExtensionData.Config.Hardware.MemoryLimit}}

# Reduce vRAM if overallocated:
Set-VM -VM VM01 -MemoryGB 8 -Confirm:$false

# Check host memory overhead (VM overhead can be significant):
Get-VMHost | Select-Object Name, @{N='OverheadMB';E={$_.ExtensionData.Host.MemoryOverhead}}
```

### Recovery
- High balloon % → Ballooning is reclaiming memory; consider adding more RAM, reducing VM memory, or increasing reservation
- High swap % → CRITICAL: Add physical RAM, reduce vRAM overcommitment, increase memory reservations for critical VMs
- High compression % → Better than swapping, but still indicates pressure; add RAM
- VM cannot power on → Check host memory, reduce reservations on other VMs, increase host RAM

---

## 15. VMware Tools

### What
**VMware Tools** is a suite of utilities and drivers installed inside the guest OS that enhances VM performance, improves guest OS management, and enables VMware features.

### Why
- **Time synchronization**: Guest clock sync with ESXi
- **Heartbeat**: HA monitors VM via VMware Tools heartbeat
- **Memory ballooning**: Balloon driver for memory reclamation
- **Network performance**: vmxnet3 driver (vs. emulated e1000)
- **Storage performance**: paravirtualized SCSI driver (pvscsi)
- **Mouse integration**: No need for Ctrl+Alt+Delete to release cursor
- **DPI scaling**: Proper display scaling in console
- **File copy/paste**: Between host and guest (drag-and-drop)
- **Auto-resize**: Resize VM window to match guest resolution
- **Guest OS customization**: Sysprep alternative features

### Components
```
VMware Tools Suite:
├── pvscsi (Paravirtualized SCSI driver) — highest storage performance
├── vmxnet3 (Paravirtualized NIC driver) — highest network performance
├── vmw_balloon (Balloon driver) — Memory reclamation
├── vmw_time (Time sync driver) — Clock synchronization
├── vmw_hgfs (Host-Guest File System) — Shared folders
├── vmw_xlibs (Video driver) — Display driver
├── vmware-user (User agent) — Copy/paste, drag/drop, auto-resize
├── vmw_storflsh (Storage flush driver) — Proper flush semantics
└── vmw_font (Font driver) — Font rendering improvements
```

### Dependencies
```
VMware Tools depends on:
├── Guest OS (compatible OS version, service pack level)
├── VMware Tools installer (ISO or offline bundle)
├── VM hardware version (compatible with Tools version)
├── vCenter (for auto-upgrade via vCenter)
├── Network (for guest IP visibility, heartbeat)
├── Administrative access to guest OS (for installation/upgrade)
└── VMware Tools service (vmtoolsd in guest OS)
```

### Failure
| Failure | Impact |
|---------|--------|
| VMware Tools not installed | No balloon driver (memory issues), no heartbeat (HA can't monitor), time drift, poor network/storage performance |
| VMware Tools not running | HA may mark VM as unresponsive, no memory ballooning, clock drift |
| VMware Tools outdated | Compatibility issues, known bugs, missing features |
| VMware Tools service crashed | Heartbeat stops, balloon driver stops, time sync stops |
| VMware Tools incompatible with guest OS | Blue screen, driver conflicts, features not available |
| Drag/drop/copy-paste not working | VMware Tools user agent issue |
| VMware Tools upgrade failed | VM may become inaccessible; partially installed Tools |

### Troubleshooting
```powershell
# Check VMware Tools status from vCenter/PowerCLI:
Get-VM VM01 | Select-Object Name, ToolsVersion, ToolsStatus, PowerState, 
  @{N='ToolsRunning';E={$_.ExtensionData.Guest.ToolsRunning}}

# ToolsStatus values:
# toolsOk — running and current
# toolsOld — running but outdated
# toolsNotInstalled — not installed
# toolsNotRunning — not running
# toolsStartup — starting up
# toolsUpgradeFailed — upgrade failed
# toolsReconfiguring — being upgraded

# From guest OS:
# Windows:
# Check service: sc query vmtools
# Check process: tasklist | findstr vmtoolsd
# Check version: reg query "HKLM\SOFTWARE\VMware, Inc.\VMware Tools" /v ProductVersion

# Linux:
# Check service: systemctl status open-vm-tools
# Check process: ps aux | grep vmtoolsd
# Check version: vmware-toolbox-cmd -v

# Check VMware Tools events:
Get-VM VM01 | Select-Object Name, 
  @{N='ToolsEvents';E={$_.ExtensionData.Guest.Events}}

# Check if Tools heartbeat is working:
Get-VM VM01 | Select-Object Name, 
  @{N='ToolsHeartbeat';E={$_.ExtensionData.Guest.ToolsHeartbeatEnabled}},
  @{N='HAIsolated';E={$_.ExtensionData.HAEvent.HAConfig.HAIsolated}}

# Reinstall VMware Tools:
# From vCenter: VM > Install VMware Tools (mounts Tools ISO to guest)
# Or PowerCLI:
Install-Tools -VM VM01
# (After mounting ISO, run installer inside guest OS)

# Check upgrade status:
Get-VM | Where-Object { $_.ExtensionData.Guest.ToolsStatus -ne "toolsOk" } | 
  Select-Object Name, ToolsStatus, @{N='Version';E={$_.ExtensionData.Guest.ToolsVersion}}

# Upgrade VMware Tools via PowerCLI:
Update-Tools -VM VM01
# Or:
Get-VM VM01 | Install-Tools -Type latest

# Check if Guest OS Customization (Linux) needs reboot:
# VMware Tools after kernel update: need restart service

# Check VMware Tools version compatibility:
# VMware Tools compatibility matrix:
# vSphere 8.0 → VMware Tools 12.x
# vSphere 7.0 → VMware Tools 11.x

# Check heartbeat:
vim-cmd vmsvc/get.guestFeaturesEnabled <VMID>

# Check if Tools auto-upgrade is enabled:
Get-AdvancedSetting -Entity (Get-Cluster "Prod-Cluster-01") | 
  Where-Object { $_.Name -like "*tools*" }
```

### Recovery
- Tools not running → Restart vmtoolsd service inside guest OS (or reboot VM)
- Tools not installed → Install via vCenter (VM > Install VMware Tools)
- Tools outdated → Upgrade via vCenter or PowerCLI (Update-Tools)
- Tools upgrade failed → Manually uninstall, reboot, reinstall latest version
- Guest service crashed → Restart service inside guest OS

### Interview Answer
> "VMware Tools is critical for VM performance and management. It includes the balloon driver for memory management, heartbeat for HA monitoring, time sync, and paravirtualized drivers (vmxnet3, pvscsi) for superior network and storage performance. Without Tools, VMs lose ballooning capability, HA can't monitor VMs, and performance degrades. I monitor Tools status via vCenter and PowerCLI (ToolsStatus property). I enable auto-upgrade in vCenter and proactively upgrade Tools before or after major vSphere updates."

---

## 16. Virtual Networking (vSwitch, Distributed Switch, Port Groups, NIC Teaming)

### What
Virtual networking in vSphere provides software-defined networking for VMs and the management/VMkernel interfaces on ESXi hosts.

### Types
| Type | Description | Best For |
|------|-------------|----------|
| **vSwitch (Standard)** | Per-host virtual switch; configured individually on each host | Small environments; simple configurations |
| **Distributed Switch (VDS/dVS)** | Centralized switch managed by vCenter; spans multiple hosts | Large environments; consistent configuration; advanced features |
| **Port Group** | Logical grouping of ports on a vSwitch; defines VLAN, security policies | Network segmentation; VM network isolation |

### vSwitch Architecture
```
Standard vSwitch (vSwitch0):
┌──────────────────────────────────────────────────────────┐
│                      vSwitch0                            │
│                                                          │
│  ┌──────────────────┐  ┌──────────────────┐              │
│  │   VM Network     │  │   vMotion        │              │
│  │   Port Group     │  │   Port Group     │              │
│  │   VLAN 10        │  │   (no VLAN)      │              │
│  │   Security:      │  │   MTU: 9000      │              │
│  │   - Promiscuous: │  │   NIC Teaming:   │              │
│  │     Reject       │  │   - Active: vmnic0│             │
│  │   - Forged:      │  │   - Standby:      │             │
│  │     Accept       │  │     vmnic1       │              │
│  │   - Mac Change:  │  │   - Active:       │             │
│  │     Accept       │  │     vmnic2       │              │
│  │   + Teaming & Failover              │  │     vmnic3       │              │
│  └──────────────────┘  └──────────────────┘              │
│                                                          │
│  Physical NICs (vmnic0-3): Connected to top-of-rack switches │
│  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐                    │
│  │vmnic0│  │vmnic1│  │vmnic2│  │vmnic3│                   │
│  │ Uplink│ │Uplink│ │Uplink│ │Uplink│                   │
│  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘                 │
│     │         │         │         │                       │
└─────┴─────────┴─────────┴─────────┴──────────────────────┘

Distributed vSwitch (dVS):
┌──────────────────────────────────────────────────────────┐
│               Distributed vSwitch (dvs01)                │
│  Managed centrally by vCenter                            │
│  Spans: ESXi01, ESXi02, ESXi03 (all hosts)             │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Port Group   │  │ Port Group   │  │ Port Group   │  │
│  │ "Prod-Net"   │  │ "vMotion"    │  │ "Mgmt"       │  │
│  │ VLAN 10      │  │ (no VLAN)    │  │ VLAN 0       │  │
│  │ Uplinks:     │  │ Uplinks:     │  │ Uplinks:     │  │
│  │  vmnic0/1    │  │  vmnic2/3    │  │  vmnic0/1    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                          │
│  Each ESXi host gets a "virtual switch" instance of dVS  │
│  with the same configuration (pNic, port groups, etc.)   │
│  Changes made on vCenter propagate to all hosts          │
└──────────────────────────────────────────────────────────┘

NIC Teaming (Load Balancing):
┌─────────────────────────────────────────────┐
│ NIC Teaming Policies:                        │
│                                              │
│ 1. Route based on originating virtual port   │
│    (default for vSwitch)                     │
│    — Hash of VM's MAC/port → specific NIC    │
│    — Simple, but may not balance evenly      │
│                                              │
│ 2. Route based on IP hash (LACP compatible)  │
│    — Hash of source/dest IP → specific NIC   │
│    — Better distribution                     │
│    — Requires LACP on physical switch        │
│                                              │
│ 3. Route based on physical NIC load          │
│    — ESXi monitors NIC utilization           │
│    — Dynamic load balancing                  │
│    — Best distribution (recommended)         │
│    — Requires vSphere Enterprise Plus        │
│                                              │
│ 4. Explicit failover order                   │
│    — Primary NIC active; failover to standby │
│    — No load balancing, just redundancy      │
│                                              │
│ Beacon Probing (failover detection):         │
│ — ESXi sends beacon frames on standby NICs  │
│ — If beacon response received → NIC is up   │
│ — More reliable than link-status only       │
└─────────────────────────────────────────────┘
```

### Port Group Security
| Setting | Options | Default | Recommendation |
|---------|---------|---------|---------------|
| **Promiscuous Mode** | Accept / Reject | Reject | Reject (security risk) |
| **MAC Address Changes** | Accept / Reject | Reject | Reject |
| **Forged Transmits** | Accept / Reject | Reject | Reject |

### Distributed Switch Features (not on Standard vSwitch)
- **Port Mirroring** (mirror VM traffic for analysis)
- **NetFlow** (traffic analysis)
- **QoS** (Network I/O Control — bandwidth guarantees)
- **Private VLANs**
- **Security Port Groups** (MAC address change, forged transmits, promiscuous mode)
- **Uplink Health Verification** (checks link speed, duplex, beacon)
- **Network I/O Control (NIOC)** — Bandwidth allocation
- **RHEL/SLES SR-IOV**
- **DXGI 1.2 RemoteFX vGPU**

### Dependencies
```
Virtual Networking depends on:
├── Physical NICs (vmnic0, vmnic1, etc.)
├── Physical switches (uplink connections)
├── VMkernel adapters (vmk0 management, vmk1 vMotion, etc.)
├── Port groups (VLAN configuration)
├── VLANs (802.1Q trunking between switches and ESXi)
├── Distributed Switch (if used) — requires vCenter
├── NIC teaming policy (failover, load balancing)
├── Network I/O Control (if used)
├── MTU (Jumbo frames if >1500 bytes)
├── Firewall rules (for inter-VM traffic if needed)
└── Physical network infrastructure (switches, routers, ACLs)

Virtual Networking is depended upon by:
├── VMs (network connectivity)
├── vMotion (VMkernel traffic for migration)
├── Management (vCenter, SSH, DCUI access)
├── Storage traffic (iSCSI, NFS via VMkernel)
├── HA (management network heartbeat)
├── DRS (vMotion network)
├── vSAN (vmkernel for vSAN traffic)
└── Monitoring (SNMP, syslog, NetFlow)
```

### Failure
| Failure | Impact |
|---------|--------|
| NIC failure (single physical NIC) | If teamed and failover configured, failover to standby; if not, network down |
| Switch port down | Physical NIC loses link; all VMs on that NIC lose network |
| VLAN misconfiguration | VMs cannot communicate across VLAN boundaries |
| Distributed Switch not updated | Inconsistent config across hosts; network issues on some hosts |
| vMotion VMkernel misconfigured | vMotion fails; DRS cannot balance VMs |
| Promiscuous mode enabled | Security risk — VM can sniff other VM traffic |
| MTU mismatch | Jumbo frame traffic fails; fragmentation/packet loss |
| Physical cable failure | Specific NIC down; failover if teamed |
| Switch trunk port down | All VLANs on that port down |
| Network I/O Control misconfigured | Bandwidth starvation for some traffic types |
| Port group not created on target host | VM cannot connect to network after vMotion (if dVS not used) |

### Symptoms
- VM loses network connectivity
- Ping fails between VMs or between VM and external network
- Slow network throughput
- vMotion fails with "network" error
- ESXi management network down
- vCenter shows "Network connectivity" warnings
- VM network adapter shows "Disconnect"
- NIC team shows a NIC as "standalone" or "dead"

### Troubleshooting
```powershell
# PowerCLI:
# Check VM network adapter:
Get-VM VM01 | Get-NetworkAdapter | Select-Object Name, NetworkName, MacAddress, StartConnected, Type

# Check VM network connectivity from vCenter:
Test-NetworkConnectivity -Entity VM01 -ServerOrIs "ping" -Protocol ICMP -Target "8.8.8.8" -Port 443

# Check vSwitch configuration:
Get-VirtualSwitch -VMHost ESXi01 | Select-Object Name, NumPorts, MTU, @{N='NicTeam';E={$_.ExtensionData.Config.UplinkPortGroup}}
Get-VirtualSwitch -VMHost ESXi01 | Get-NicTeam | Select-Object Name, NicTeamPolicy, @{N='ActiveNic';E={$_.ExtensionData.Config.ActiveNic}} | Format-Table

# Check port groups:
Get-VirtualPortGroup -VMHost ESXi01 | Select-Object Name, VlanId, VlanType, VMHost

# Check physical NICs:
Get-VMHostNetworkAdapter -VMHost ESXi01 | Where-Object {$_.Physical -eq $true} | 
  Select-Object Name, MacAddress, Speed, Duplex, Driver, Status

# Check VMkernel adapters:
Get-VMHostNetworkAdapter -VMHost ESXi01 | Where-Object {$_.VMKernel -eq $true} | 
  Select-Object Name, IP, SubnetMask, MacAddress, PortGroup, Connected

# Check NIC teaming:
Get-VMHostVirtualNicSwitch -VMHost ESXi01 -Name vSwitch0 | 
  Select-Object -ExpandProperty NicTeamingPolicy | Select-Object *

# Check specific NIC load (for load balancing):
Get-VMHost ESXi01 | Get-VMHostNetworkAdapter -Physical | 
  Select-Object Name, Status, Speed, Duplex

# Check network connectivity (from ESXi):
esxcli network ping -i vmk0 -H 8.8.8.8
esxcli network ping -H ESXi02.contoso.com
esxcli network ping -H vcenter.contoso.com

# Check network failure (from ESXi):
esxcli network diag ip load -i vmk0

# Check Distributed vSwitch:
Get-VDSwitch | Select-Object Name, UplinkPortgroupName, MTU, NumUplinks
Get-VDSwitch -Name "dvs01" | Get-VDPortgroup | Select-Object Name, VlanId
Get-VDPortgroup -Name "Prod-Net" | Get-VDPort -VM VM01 | Select-Object VM, PortId, Active, Connecting

# Check physical NIC connection:
esxcli network nic get -n vmnic0
ethtool vmnic0    (via SSH, Linux-style)

# Check if VM is connected to network:
vim-cmd vmsvc/network.getavailableadapters <VMID> 0

# Check for promiscuous mode / security issues:
Get-VDPortgroup -Name "Prod-Net" | Get-VDPort | 
  Select-Object -ExpandProperty Config | Where-Object { $_.Security -ne $null } | 
  Select-Object Security

# Check physical switch port:
# On switch side: show interface status, show mac address table, show spanning-tree

# On ESXi (check physical layer):
esxcli network nic elec get -n vmnic0    # LED status
esxcli network nic link get -n vmnic0    # Link status

# Check Network I/O Control:
Get-NetworkPool | Select-Object Name, AllocationPolicy
Get-VMHostNetworkResourcePool -VMHost ESXi01 | Select-Object Name, 
  @{N='Shares';E={$_.Shares}}, @{N='ReservationMBps';E={$_.ReservationMBps}}, 
  @{N='LimitMBps';E={$_.LimitMBps}}

# Check VM network throughput:
esxtop  # Press 'n' for network
vim-top    # or vscivTop

# Check VLAN configuration