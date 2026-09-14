# MODULE 13 — AZURE COMPUTE: VMs DEEP, VM DISK / STORAGE PERFORMANCE, VM AVAILABILITY, VM SCALE SETS, AZURE APP SERVICES, CONTAINERS (ACI / ACA / AKS / ACR) — L3 DEPTH 🔴 CRITICAL

---

## 1. CONCEPT

**Azure Compute** is the family of services that provide processing power — the resources that run your applications, processes, and workloads in the cloud. Unlike data services (which persist), compute is **ephemeral** — it can be stopped, deallocated, restarted, replaced, or scaled at any moment. This ephemeral nature makes compute design one of the most dynamic and error-prone areas of Azure architecture.

> **The single most important compute concept:** Compute is always a trade-off between performance, cost, availability, and flexibility — and the "right" choice depends entirely on the workload characteristics. A VM is not "better" or "worse" than App Service or AKS; it is *different*. Choosing a VM for a simple web app (when App Service suffices), or App Service for a complex multi-tier application (when VMs are needed), or AKS for a single microservice (when ACI suffices), or ACR for code-driven deployments (when Docker build is simpler) — each is a fundamental architectural error that results in wasted money, reduced agility, or unnecessary complexity. The discipline is: **right-sized compute for the right workload at the right cost with the right availability.**

**Why Compute selection matters:**

```
WITHOUT PROPER COMPUTE SELECTION:
  → Simple web app on VM (paying 24/7 for idle compute)
  → Complex app on App Service (no VM-level control, no custom extensions)
  → Single container on AKS (overkill orchestration for one pod)
  → Multi-app cluster on ACI (no orchestration, no scaling, no networking)
  → VMs without managed disks (unmanaged blob disks = management nightmare)
  → VMs with standard HDD (slow I/O, no caching, poor database performance)
  → Single VM for production (no HA, single point of failure)
  → Manual scaling (over-provisioned = wasted money; under-provisioned = poor performance)
  → No disk type matching workload (random I/O on page blob = terrible performance)
  → No availability set / zone awareness (VM down = app down)

WITH PROPER COMPUTE SELECTION:
  → Simple web app on App Service (auto-scale, managed, cost-efficient)
  → Complex app on VMs (full control, custom extensions, any OS)
  → Single container on ACI (burst, dev/test, simple run)
  → Multi-app microservices on AKS (orchestration, scaling, rolling updates)
  → VMs with managed disks (no blob management, automatic replication)
  → VMs with Premium SSD for databases (high IOPS, low latency, caching)
  → Multiple VMs in Availability Set/Zone (HA, no single point of failure)
  → Auto-scale VMs / App Service (scale out when busy, scale in when idle)
  → Disk type matching workload (Premium SSD for DB, Standard SSD for app, HDD for archive)
  → Zone-redundant / managed disk replication (VM down → VM in another zone takes over)
```

**Compute Services Taxonomy:**

```
┌─ IAAS (Infrastructure as a Service) ─┐
│                                       │
│  Virtual Machines (VMs)               │
│  → Full OS control, custom extensions │
│  → Windows / Linux / GPU / HPC        │
│  → Configurable: vCPU, RAM, disk, NIC│
│                                       │
│  Virtual Machine Scale Sets (VMSS)    │
│  → Auto-scale VMs based on metrics    │
│  → Identical VMs in uniform set       │
│  → Load balanced across instances     │
│                                       │
│  Available Images                     │
│  → Azure Marketplace (pre-built)      │
│  → Custom Images (your own)           │
│  → Shared Images (across tenanted)    │
│                                       │
└───────────────────────────────────────┘

┌─ PAAS (Platform as a Service) ────────┐
│                                       │
│  Azure App Service                    │
│  → Web apps, REST APIs, mobile backends│
│  → Auto-scaling, built-in CI/CD       │
│  → Managed runtime (no OS management) │
│  → Deployment slots (staging/prod)    │
│                                       │
│  Container Instances (ACI)            │
│  → Single container, burst, dev/test  │
│  → No orchestration, no cluster       │
│  → Pay per second of execution        │
│                                       │
│  Container Apps (ACA)                 │
│  → Docker containers, no orchestrator │
│  → KEDA-based auto-scaling (events)   │
│  → Built-in ingress, secrets, jobs    │
│  → Serverless containers              │
│                                       │
│  Azure Kubernetes Service (AKS)       │
│  → Managed Kubernetes (control plane) │
│  → Agent pools (VM node pools)        │
│  → Rolling updates, self-hecovery     │
│  → Service mesh, ingress, RBAC        │
│                                       │
│  Container Registry (ACR)             │
│  → Docker image storage               │
│  → Build, push, pull, scan            │
│  → Geo-replication                    │
│  → Tasks (auto-build on commit)       │
│                                       │
└───────────────────────────────────────┘

┌─ SPECIALIZED COMPUTE ─────────────────┐
│                                       │
│  Batch                              │
│  → Large-scale parallel HPC jobs      │
│  → Auto-pool, task scheduling         │
│                                       │
│  Virtual Desktop (AVD)               │
│  → Managed virtual desktops           │
│  → Session hosts, persistent pools    │
│                                       │
│  Dedicted Hosts                      │
│  → Single-tenant physical servers     │
│  → License mobility (SQL, Windows)    │
│                                       │
└───────────────────────────────────────┘
```

---

## 2. ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          AZURE COMPUTE ARCHITECTURE                             │
│                                                                                  │
│  ── COMPUTE LIFECYCLE ──────────────────────────────────────────────────         │
│                                                                                  │
│  User/App → Load Balancer / Front Door → Compute Layer                          │
│                                            │                                    │
│                                            ▼                                    │
│  ┌─ Web Tier ───────────────────────────────────────────────────────────┐        │
│  │                                                                      │        │
│  │  Option A: App Service (PaaS)                                        │        │
│  │  → Auto-scale: 1-20 instances (based on CPU, memory, HTTP queue)     │        │
│  │  → Deployment slots: dev → staging → production (swap with preview)  │        │
│  │  → Managed runtime: .NET, Java, Node, Python, PHP                   │        │
│  │  → No VM management, no OS patching                                 │        │
│  │                                                                      │        │
│  │  Option B: AKS (Containers)                                          │        │
│  │  → Kubernetes pods on agent pool VMs                                │        │
│  │  → Auto-scale: HPA (CPU/memory), VPA (resource limits), KEDA (event)│        │
│  │  → Rolling updates, self-healing (restart crashed pods)             │        │
│  │  → ACR for image storage, geo-replication                           │        │
│  │                                                                      │        │
│  └──────────────────────────────────────────────────────────────────────┘        │
│                                                                                  │
│  ┌─ App Tier ───────────────────────────────────────────────────────────┐        │
│  │                                                                      │        │
│  │  Option A: VM Scale Set (stateless apps)                            │        │
│  │  → Identical VMs, load balanced                                     │        │
│  │  → Auto-scale: count-based, CPU-based, schedule-based               │        │
│  │  → Unified OS image (all VMs same configuration)                    │        │
│  │  → Orchestration: aligned, across fault/upgrade domains             │        │
│  │                                                                      │        │
│  │  Option B: VMs (stateful apps, custom requirements)                 │        │
│  │  → Single VM or in Availability Set                                 │        │
│  │  → Full OS control, extensions, custom config                       │        │
│  │  → Manual or script-based scaling                                   │        │
│  │                                                                      │        │
│  └──────────────────────────────────────────────────────────────────────┘        │
│                                                                                  │
│  ┌─ Data Tier ──────────────────────────────────────────────────────────┐        │
│  │                                                                      │        │
│  │  Azure SQL / Cosmos DB / Storage / ADLS Gen2                        │        │
│  │  → All accessed via Private Endpoint (no public exposure)           │        │
│  │  → Managed disks for VM data: Premium SSD for SQL/DB, Standard SSD  │        │
│  │    for app, Standard HDD for archive                                │        │
│  │                                                                      │        │
│  └──────────────────────────────────────────────────────────────────────┘        │
│                                                                                  │
│  ┌─ AVAILABILITY LAYER ────────────────────────────────────────────────┐        │
│  │                                                                      │        │
│  │  VM → Availability Set (fault/upgrade domains) or                   │        │
│  │  VM → Availability Zone (physical separation in datacenter)         │        │
│  │  App Service → Multi-region deployment (deployment slots)           │        │
│  │  AKS → Multiple node pools across zones                             │        │
│  │  Scale Set → Across zones (zone-redundant VMSS)                     │        │
│  │                                                                      │        │
│  └──────────────────────────────────────────────────────────────────────┘        │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. COMPONENTS — DETAILED

### 3.1 VIRTUAL MACHINES (VMs) — COMPLETE DEEP DIVE

**What is an Azure Virtual Machine?**
An Azure Virtual Machine (VM) is a fully managed Infrastructure as a Service (IaaS) compute resource that provides a virtualized computer in the cloud. It gives you complete control over the operating system, applications, and runtime — you manage everything from the OS up (OS management responsibility model).

> **Key distinction from App Service / AKS:** VMs give you full control of the OS and host but require full OS management (patching, updates, security hardening). App Service manages the OS/runtime for you. AKS manages the orchestration but you manage the container images and node OS.

**VM Components:**

```
Azure VM:
  ├── Virtual Machine (compute resource)
  │   ├── OS Disk (OS type: Windows/Linux, disk type: Premium SSD/Standard SSD/HDD)
  │   ├── Data Disks (attached disks: up to 64 per VM)
  │   ├── Network Interface (NIC: 1+ per VM, primary + secondary)
  │   ├── Public IP (optional, not recommended for production)
  │   ├── Boot Diagnostics (captures boot logs/screenshots to Storage)
  │   ├── SSH Key / Windows Admin Username+Password (authentication)
  │   └── Extensions (Custom Script, Antimalware, Disk Encryption, etc.)
  │
  ├── Availability Options:
  │   ├── None (single VM, no redundancy)
  │   ├── Availability Set (fault + upgrade domains)
  │   ├── Availability Zone (physically separate datacenter)
  │   ├── Virtual Machine Scale Set (uniform, auto-scaled)
  │   └── No infrastructure redundancy (requires application-level)
  │
  ├── Security:
  │   ├── NSG (network access control)
  │   ├── Azure Bastion (secure RDP/SSH without public IP)
  │   ├── Azure Defender for VMs (threat detection)
  │   ├── Disk Encryption (Azure Disk Encryption / PMK/CMK)
  │   ├── Confirmed VM (Secure VM: TPM, vTPM, Secure Boot, measured boot)
  │   ├── Just-InTime VM Access (reduce attack surface)
  │   └── Azure AD integration (SSO for Linux)
  │
  └── Management:
      ├── Azure Monitor (metrics, logs, alerts)
      ├── Azure Backup (VM backup to Recovery Vault)
      ├── Run Command (run scripts on VM without public access)
      ├── Serial Console (emergency access)
      ├── Custom Images (golden images for rapid deployment)
      └── Automation (runbooks, desired state configuration)
```

**VM Sizes (vCPU / RAM Categories):**

| Category | Series | vCPU:RAM Ratio | Use Case |
|----------|--------|---------------|---------|
| **General Purpose** | B (burstable), Dsv2/Dsv3/DSv5 | 1:1 to 1:4 | Web servers, small databases, dev/test |
| **Compute Optimized** | Fsv2/FA, F-series | 2:1 (high CPU) | Medium traffic web, network appliances, batch |
| **Memory Optimized** | Ev2/Ev3/Ev4/Easv3/Easv4 | 1:4 to 1:8 (high RAM) | Large databases, in-memory caches, analytics |
| **Storage Optimized** | Lsv2, L-series | 1:4 (high disk) | Big data, SQL, NoSQL, data warehouses |
| **GPU** | NCasv3/NC/ND/ND A100 | Varies | ML training, inferencing, visualization |
| **High Performance Compute (HPC)** | HBv2/HBv3/HC | 1:1 (high CPU+mem) | Molecular dynamics, CFD, oil/gas exploration |
| **Optimized for SAP** | Ev4/ESv4 | 1:4 (high RAM) | SAP HANA, S/4HANA |
| **Burstable** | B1s, B2s | 1:2 | Dev/test, low-traffic web (CPU credits) |

**VM Size Details — B-Series (Burstable):**

```
B-Series: Burstable VMs with CPU credits
  → Baseline CPU: Low (e.g., B1s: 1 vCPU, 2 GB, baseline 10%)
  → CPU Credits: Earned when using less than baseline
  → Consume credits when bursting above baseline
  → Credit balance: Max 30 credits (B1s) to 350 credits (B4ms)
  → Credit expiration: 30-day rolling window
  → When credits exhausted: VM throttles to baseline (performance drops)
  → NOT suitable for: Always-on workloads, databases, production apps
  → Ideal for: Dev/test, low-traffic web apps, monitoring, small services

  Pricing advantage: B1s ~$12/month vs D2s v2 ~$70/month (70% savings)
  But: Performance is NOT guaranteed (burst only, not sustained)

  Important: B-series NOT appropriate for:
    → Production workloads requiring consistent performance
    → Databases (SQL, Cosmos, etc.)
    → Any workload requiring sustained CPU > baseline
    → VM Scale Sets (burst credits deplete quickly under load)
```

**VM Authentication Methods:**

```
Authentication Options:
  1. SSH Key (Linux — RECOMMENDED):
     → Public/private key pair
     → No password required
     → More secure than password (brute-force immune)
     → Generated: Azure CLI, PuTTYgen, ssh-keygen
     → Stored: Azure Key Vault (optional), VM configuration
     → Command: ssh -i ~/.ssh/id_rsa azureuser@<VM-IP>

  2. Password (Linux — NOT recommended):
     → Username + password
     → Vulnerable to brute force
     → Azure enforces password complexity
     → Disable password auth after SSH key setup

  3. Windows Admin Username + Password:
     → Required for Windows VMs
     → Combined with Active Directory (domain-joined VMs)
     → Managed Identity for service access (no credentials in code)

  4. Azure AD (SSO for Linux VMs):
     → Entra ID authentication for Linux (preview/GA)
     → Conditional Access policies apply
     → MFA enforcement
     → No SSH keys needed (Entra ID manages auth)
     → User assignment: Entra ID user/group → Linux login

  5. Managed Identity (Service Access):
     → VM has Azure AD identity (system-assigned or user-assigned)
     → VM authenticates to other Azure services (Storage, SQL, etc.)
     → No credentials stored in code/config
     → Token-based: VM requests AAD token → Validates against target service
     → RBAC: Assign role (e.g., Storage Blob Data Reader) to VM MI

  L3 Best Practice: SSH Key for Linux + Azure AD SSO; Password for Windows +
  Managed Identity for service access; NEVER store credentials in code/repos.
```

**VM Extensions — Deep Dive:**

```
VM Extensions are small software packages that deploy post-deployment configuration:

  1. Custom Script Extension (CSE):
     → Runs PowerShell/Bash scripts on VM after deployment
     → Use for: Install software, configure settings, join domain
     → Example: Install IIS, configure web.config, join Azure AD
     → Can reference scripts in Azure Storage or GitHub
     → Executes: Via Azure Platform Access (no public internet needed)

     az vm extension set \
       --name CustomScript \
       --publisher Microsoft.Azure.Extensions \
       --vm-name myVM \
       --resource-group myRG \
       --settings '{"fileUris": ["https://storage.blob.core.windows.net/scripts/setup.sh"], "commandToExecute": "bash setup.sh"}'

  2. Azure Disk Encryption (ADE):
     → Encrypts OS and data disks (BitLocker for Windows, dm-crypt for Linux)
     → Uses Azure Key Vault (Key Encryption Key, KEK)
     → Supports: Platform-managed key (PMK) and Customer-managed key (CMK)
     → CMK with Key Vault (Soft Delete + Purge Protection REQUIRED)
     → Not supported on: Ephemeral OS disks (use OS disk encryption at host instead)

  3. Azure Security Extensions:
     → Microsoft Defender for Cloud (threat detection on VMs)
     → Just-InTime VM Access (reduce NSG rules, time-limited access)
     → Azure Attack Surface Reduction (ASR) rules

  4. Monitoring Extensions:
     → Microsoft Monitoring Agent (MMA) / OMS Agent
     → Deploys: Log Analytics agent for VM monitoring
     → Collects: Performance counters, syslogs, custom logs
     → Enables: Azure Monitor for VMs (automatic baseline, anomaly detection)

  5. Azure VM Serial Console:
     → Emergency access when RDP/SSH fails
     → Boot diagnostics, kernel logs, recovery
     → Enabled: Boot diagnostics + Serial Console (portal)
     → Access: Portal → VM → Serial Console

  6. Azure Run Command:
     → Run commands on VM without public access (uses Azure platform)
     → Alternative to CSE for one-off commands
     → No agent required
     → Use for: Restart services, check status, emergency fixes
     → Security: No public endpoint required

  7. Dependency Agent:
     → Installs on VM for Application Map dependency tracking
     → Shows: VM → SQL DB, VM → Storage, VM → App Service dependencies
     → Part of: Application Insights distributed tracing

  8. Azure AD Login Extension (Linux):
     → Enables Entra ID authentication on Linux VMs
     → Manages Linux users from Entra ID
     → Supports: SSH key management via Entra ID
```

**VM OS Disk Types and Performance:**

```
OS Disk (and Data Disk) Types:

  1. Ultra Disk (NEW — preview):
     → Highest IOPS and throughput (up to 160,000 IOPS, 4 GB/s)
     → Burstable IOPS (configurable)
     → Use: HPC, ML, mission-critical databases
     → Size: 4–65,536 GB
     → Unique: Live resize (no VM restart needed)

  2. Premium SSD v2 (GA):
     → High IOPS (up to 80,000 IOPS), high throughput (up to 2.5 GB/s)
     → Sub-second latency (typical: <1ms)
     → Use: Production databases, IO-intensive workloads
     → Size: 4–65,536 GB
     → Cost: Highest (but lower than Ultra)

  3. Premium SSD v1 (GA):
     → Moderate IOPS (up to 30,000), moderate throughput (up to 250 MB/s)
     → Use: Production workloads, moderate IO
     → Size: 4–512 GB (4–4,096 GB for v1)
     → Cost: Moderate

  4. Standard SSD:
     → Low-moderate IOPS (up to 500), suitable for boot/app
     → Use: Dev/test, boot disks, low-IO web servers
     → Cost: ~50% cheaper than Premium SSD

  5. Standard HDD:
     → Lowest cost, suitable for infrequent access, throughput workloads
     → Use: Backup, archive, non-critical workloads
     → NOT for: Boot disks (survives VM restart unlike other types), databases
     → Cost: ~75% cheaper than Premium SSD

  Disk Caching:
    → None: No caching (disk I/O goes directly to storage)
    → ReadOnly: Cache read operations (write goes directly)
    → ReadWrite: Cache read/write (default for OS disk)
    
    Recommended caching:
      OS Disk: ReadWrite (faster boot, faster reads)
      Data Disk (Database): None (prevents stale cache, data integrity)
      Data Disk (App/Logs): ReadOnly (improves read performance)

  L3 critical: Disk caching for databases should be None. Stale cache can serve
  outdated data. For databases: Always use None + Premium SSD/Ultra Disk.
  For web/app servers: ReadOnly + Standard SSD is sufficient.
```

**Disk Performance — L3 Detail:**

```
Disk Performance Matrix:

  Disk Type       | Max IOPS   | Max Throughput | Burst (IOPS) | Latency   | Max Size
  ────────────────|────────────|────────────────|──────────────|──────────|─────────
  Ultra Disk      | 160,000    | 4 GB/s         | No (fixed)   | <1 ms     | 65,536 GB
  Premium SSD v2  | 80,000     | 2.5 GB/s       | Yes (config) | <1 ms     | 65,536 GB
  Premium SSD v1  | 30,000     | 250 MB/s       | Yes          | <1 ms     | 512 GB
  Standard SSD    | 500        | 60 MB/s        | No           | <1 ms     | 4,096 GB
  Standard HDD    | 100        | 25 MB/s        | No           | <1 ms*    | 64 TB *

  * Standard HDD burst: Up to 10,000 IOPS for 30 minutes
  * Standard HDD latency: Typically <1 ms for burst, can degrade under sustained load

  IOPS Calculation for VM:
    → Total IOPS = sum of all attached disk IOPS (OS + data disks)
    → VM size limits maximum IOPS (check VM documentation)
    → Example: VM E4s_v3 has max 12,800 IOPS limit
      Disk: 2× Premium SSD v2 (32,000 IOPS each) = 64,000 IOPS
      But VM cap: 12,800 IOPS → actual: 12,800 IOPS (VM is bottleneck)
    → Fix: Use larger VM (E8s_v3: 25,600 IOPS) or fewer disks

  Throughput Calculation:
    → Throughput = (IOPS × I/O size in bytes) / 1024
    → Typical I/O size: 4 KB (random), 256 KB (sequential)
    → Example: 30,000 IOPS × 4 KB = 117,187 KB/s ≈ 114 MB/s (random)
    → Sequential: 30,000 IOPS × 256 KB = 7,500,000 KB/s ≈ 7.3 GB/s

  Storage Space:
    → Max data disks per VM: 64 (depends on VM size — some allow up to 32 or 64)
    → Max data disk storage: Depends on VM size (e.g., D4s v3: 32 TB)
    → Use: Combine multiple smaller disks for larger volumes (RAID-0/striping)
    
  Disk Striping (RAID-0 across multiple Premium SSDs):
    → Why: Combine IOPS/throughput of multiple disks
    → Example: 4× Premium SSD v2 (32,000 IOPS each) = 128,000 IOPS striped
    → But: VM IOPS cap still applies (need larger VM to utilize all)
    → Risk: No redundancy (single disk failure = data loss)
    → Use: When performance > reliability (app-level replication)
    → Tools: mdadm (Linux), Storage Spaces (Windows), disk partitions

  L3 Critical: Know your VM's disk IOPS limit. A D4s v3 VM with 4× P40 disks
  (30,000 IOPS each = 120,000 total) is capped at 12,800 IOPS by the VM.
  You pay for 4 disks but only use 11% of their capacity — waste of money.
  Either downgrade disks OR upgrade VM size.
```

---

### 3.2 VM AVAILABILITY — COMPLETE DEEP DIVE

**What is VM Availability?**
VM Availability ensures that your VMs remain accessible during planned maintenance (OS updates, hardware upgrades) and unplanned events (hardware failures, datacenter issues). Azure provides multiple mechanisms to ensure VM availability, from single-point redundancy to geographic distribution.

**Availability Options:**

```
VM Availability Options (from simplest to most resilient):

  1. None:
     → Single VM, no redundancy
     → VM down = app down (single point of failure)
     → Use: Dev/test only

  2. Availability Set:
     → Groups VMs into fault domains + upgrade domains
     → Physical separation within ONE datacenter
     → No cross-region protection
     → Use: HA within single region for VMs that need same-datacenter

  3. Availability Zone:
     → Physically separate datacenters in ONE region (3 zones per region)
     → Highest resilience (survives entire datacenter failure)
     → Costs more (zone-specific VMs + zone-specific disks)
     → Use: HA within region with maximum resilience

  4. Virtual Machine Scale Set (VMSS):
     → Multiple identical VMs, load balanced, auto-scaled
     → Can be zone-redundant (across zones)
     → Use: Web/stateless apps that scale horizontally

  5. No infrastructure + app-level HA:
     → Application handles failover (e.g., SQL Always On, Cosmos DB multi-region)
     → Use: Stateful apps with built-in replication

  IMPORTANT: Availability Set + Zone = use EITHER but not both (conflict)
  → If using zones: No Availability Set needed (zones provide domain separation)
  → If using Availability Set: Single region only (no zones)
```

**Availability Set — Deep Dive:**

```
Availability Set Architecture:

  Availability Set: "myAVSet"
    ├── Fault Domain 0:
    │   ├── VM-1 (rack 1, power 1, network 1)
    │   └── VM-3 (rack 2, power 1, network 2)
    │
    ├── Fault Domain 1:
    │   ├── VM-2 (rack 1, power 2, network 1)
    │   └── VM-4 (rack 2, power 2, network 2)
    │
    ├── Upgrade Domain 0: (updated first during maintenance)
    │   ├── VM-1, VM-3
    │
    ├── Upgrade Domain 1: (updated second)
    │   ├── VM-2, VM-4
    │
    └── Upgrade Domain 2: (updated last)
        ├── (empty, for future expansion)

  Fault Domains:
    → Maximum physical separation within datacenter
    → Each fault domain: ~2 VMs in same rack/power/network
    → Guarantee: No two VMs in same fault domain share same power supply + network switch
    → Datacenter has 3 fault domains (by default)
    → Maximum VMs per fault domain: Varies (typically 100s)
    → Max VMs per availability set: 1,000 (across all fault domains)

  Upgrade Domains:
    → Groups VMs for sequential maintenance (one UD at a time)
    → During Azure platform maintenance:
      1. UD 0: VMs rebooted and updated (VMs in UD 0 are temporarily unavailable)
      2. Wait for UD 0 VMs to be healthy
      3. UD 1: Same process
      4. UD 2: Same process
    → During VMSS managed upgrade: All VMs updated simultaneously (different behavior)
    → Max upgrade domains: 20 per availability set
    → Default: 3 (if not specified)

  Key Guarantee:
    → At least 2 VMs per fault domain (if 2+ VMs in set)
    → VM-1 (UD 0, FD 0) AND VM-2 (UD 1, FD 1) are in different fault AND upgrade domains
    → During maintenance: One VM down at most (alternating UD updates)
    → During failure: If FD 0 goes down, VMs in FD 1 continue (VM-2, VM-4 survive)

  For HA: Need minimum 2 VMs across 2 fault domains (FD0 + FD1)
  → Both VMs in same FD: No HA (if rack fails, both down)
  → 2 VMs in FD0, 2 VMs in FD1: HA within datacenter (survives rack failure)
  → BUT: Single datacenter failure = ALL fault domains down = all VMs down
  → For cross-datacenter: Use Availability Zones instead
```

**Availability Zone — Deep Dive:**

```
Availability Zone Architecture:

  Region: East US 2 (3 zones available)
    ├── Zone 1: Datacenter A ( physically separate building)
    │   ├── VM-1 (Zone 1) + OS Disk (Zone 1 - LRS/ZRS)
    │   └── VM-2 (Zone 1) + OS Disk (Zone 1 - LRS/Adaptive)
    │
    ├── Zone 2: Datacenter B (physically separate building, 1+ mile away)
    │   ├── VM-3 (Zone 2) + OS Disk (Zone 2 - LRS/Adaptive)
    │   └── VM-4 (Zone 2) + OS Disk (Zone 2 - LRS/Adaptive)
    │
    └── Zone 3: Datacenter C (physically separate building, 1+ mile away)
        ├── VM-5 (Zone 3) + OS Disk (Zone 3 - LRS/Adaptive)
        └── VM-6 (Zone 3) + OS Disk (Zone 3 - LRS/Adaptive)

  Key Concepts:
    → Zones are physically separate datacenters within ONE region
    → Each zone has independent power, cooling, networking
    → Zone resiliency: Survives entire datacenter failure
    → Distance: Typically 1+ miles between zones (minimum 30 minutes apart for natural disasters)
    → Cost: Zone-specific disks cost more (replicate across zones or within single zone)

  Disk Replication and Zones:
    LRS (Locally Redundant):
      → Disk replicated 3× within SINGLE datacenter (zone)
      → If zone goes down: Data on LRS disks LOST (even if VM image is preserved)
      → VM survive (in different zone): But OS disk gone = VM unusable
      → VM can be rebuilt from image, but takes time
    
    ZRS (Zone-Redundant):
      → Disk replicated 3× across zones in same region
      → If zone goes down: Data survives (replicas in other zones)
      → VM can be rebuilt quickly from ZRS disk
    
    Adaptive (newer):
      → Same zone as VM by default
      → Replicates to another zone on explicit request (manual control)
      → Cost savings when not needing cross-zone replication
      → Best for: VMs in single zone where cross-zone disk replication not needed

  Best Practice for High Availability:
    → VM-1 (Zone 1) + ZRS disk (Zone 1,2,3)
    → VM-2 (Zone 2) + ZRS disk (Zone 1,2,3)
    → Load balancer: Frontend IP across all zones
    → Application Gateway: Zone-redundant (deployed in zones 1,2,3)
    → Result: Survives single datacenter failure with minimal RTO

  Zone-Resilient Services (already multi-zone):
    → Azure Load Balancer: Frontend IP can be zone-resilient (single IP for all zones)
    → Application Gateway: V2 zone-redundant (deployed in all 3 zones)
    → Azure Firewall: Zone-redundant (by default, 3 instances in 3 zones)
    → Azure Bastion: Zone-redundant (recommended)
    → Virtual WAN Hub: Scale units across zones (HA within region)

  When to use Availability Zone vs Availability Set:
    → Availability Set: HA within ONE datacenter (cheaper, adequate for most scenarios)
    → Availability Zone: HA across MULTIPLE datacenters (expensive, mission-critical)
    → Combined: NOT SUPPORTED (mutually exclusive)
    → Recommendation: Start with Availability Set; only use Zones for Tier-1 apps
```

---

### 3.3 VM SCALE SETS (VMSS) — COMPLETE DEEP DIVE

**What is Virtual Machine Scale Set?**
A Virtual Machine Scale Set (VMSS) is a set of identical, auto-scaled VMs that are load balanced across instances. VMSS simplifies management of large numbers of identical VMs and enables true horizontal scaling for stateless workloads.

**VMSS Architecture:**

```
Virtual Machine Scale Set: "webapp-vmss"
  ├── Load Balancer (Backend Pool: vmss instances)
  │   ├── Frontend IP: Public/Internal
  │   ├── Backend Pool: vmss-nic (network interface of instances)
  │   ├── Health Probe: HTTP /health, TCP 8080, etc.
  │   └── Load Balancing Rule: Frontend:80 → Backend:8080 (1000 instances)
  │
  ├── Application Gateway (Optional — for WAF, L7 routing):
  │   ├── Backend Pool: vmss instances
  │   ├── Health Probe: HTTP
  │   └── WAF enabled
  │
  ├── VMSS Instances (identical VMs):
  │   ├── Instances: 2–1,000 per scale set (default)
  │   │   → Min: 2 (HA)
  │   │   → Max: 1,000 (1,000 per VMSS; multiple VMSS for more)
  │   ├── VM Size: Standard_D2s_v3 (or any supported size)
  │   ├── OS Disk: Premium SSD (read-write cache)
  │   ├── Data Disks: Standard SSD (if needed)
  │   ├── NIC: 1 per VM (primary NIC)
  │   └── Orchestration: Uniform (all identical) or Flexible (mix of models)
  │
  ├── Auto-Scale Rules:
  │   ├── Metric: Average CPU > 70%
  │   ├── Action: Add 1 instance (scale out)
  │   ├── Cooldown: 5 minutes (wait before next scale action)
  │   ├── Schedule: Scale to 10 instances Mon-Fri 8 AM–6 PM
  │   └── Metric: Memory > 80% → Add 2 instances
  │
  ├── Upgrade Policy:
  │   ├── Manual: You control when instances are upgraded
  │   ├── Automatic: Instances upgraded one-by-one during rolling upgrade
  │   └── Rolling Update: VMs updated in batches (default: 20% at a time)
  │
  ├── Platform Fault Domain:
  │   → Distributed across fault domains (if not zone-based)
  │   → Zone-redundant: Distributed across 3 zones
  │
  └── Extensions:
      → Custom Script (install app on all instances)
      → MMA (monitoring agent)
      → DSC (Desired State Configuration)
```

**VMSS Instance Limits:**

```
Maximum Instances Per VMSS:
  → Default limit: 1,000 instances per VMSS
  → Can be increased (by contacting Microsoft Support) up to 10,000
  → BUT: Each instance = 1 VM = billing cost
  → For 10,000+ instances: Use multiple VMSS (each up to 1,000) behind Load Balancer

Region-Specific Limits:
  → Each region has VMSS limits (check documentation)
  → Typical: 1 VMSS per region, 1,000 instances, 1,000 instances × VM size limits
  → Total vCores per region: Subject to Azure subscription limits (can request increase)

Load Balancer Connection Limits:
  → Standard Load Balancer: 1,000 instances in backend pool (max)
  → Basic Load Balancer: 1,000 instances (but not recommended for production)
  → If > 1,000 instances: Use multiple LBs or Application Gateway (supports more)

VMSS vs Multiple Single VMs:
  → VMSS: Identical VMs, uniform scaling, one configuration, one update domain
  → Single VMs: Different configs, individual management, no auto-scale (manual)
  → Use VMSS when: Multiple identical VMs, auto-scale needed, stateless workload
  → Use single VMs when: Different OS/config per VM, stateful workload, specific customization
```

**VMSS Scaling Mechanisms:**

```
1. Manual Scaling:
   → Set desired capacity (instance count) explicitly
   → Use: predictable load patterns (business hours, known events)
   → Command: az vmss scale --new-capacity 10

2. Auto-Scale Based on Metrics:
   → Metric: CPU, memory, disk I/O, network, custom metric
   → Rules:
      IF avg CPU > 70% THEN add 1 instance (scale out)
      IF avg CPU < 20% THEN remove 1 instance (scale in)
   → Cooldown: Minimum time between scale actions (default: 5 minutes)
   → Monitor: Scale set instance count, scaling activities

3. Schedule-Based Scaling:
   → Predefined time-based scaling (business hours, weekends, etc.)
   → Example:
      Mon-Fri 8 AM: 10 instances
      Mon-Fri 6 PM: 2 instances  
      Weekends: 1 instance
   → Use: Known traffic patterns (not for unpredictable spikes)

4. Predictive Autoscale (ML-based):
   → Uses historical traffic patterns to predict future load
   → Pre-scales (before traffic spike hits)
   → Requires: At least 1 week of historical data
   → Combine with reactive rules for best results

5. VMSS Unified Mode vs Flexible Mode:
   Unified (default):
     → All instances have same OS image, extensions, configuration
     → Rolling upgrades update all instances together
     → Best for: Stateless web/API apps

   Flexible:
     → VMSS manages instances but can have different models/OS
     → Individual instance management (add/remove/modify)
     → Mixed orchestration: VMSS + individual VMs in same LB backend
     → Best for: Gradual migration (from VMs to VMSS), testing new configurations
     → More complex to manage

6. VMSS and Zones:
   → Zone-redundant VMSS: Instances distributed across 3 zones
   → Each instance has OS disk in the same zone as VM
   → Load Balancer frontend IP: Zone-redundant (single IP across all zones)
   → Use: Maximum HA for stateless workloads
   → Cost: Zone-specific VMs + zone-specific disks (more expensive)
   → NOT zone: Instances in single zone only
```

**VMSS Rolling Upgrade Behavior:**

```
Rolling Update Process:

  Batch Size: 20% of instances (default) = 100 instances (for 500 VMSS)
  
  Step 1: Upgrade Batch 1 (100 instances)
    → Deallocate VM-1 to VM-100 (temporarily unavailable)
    → Deploy new image/OS/extensions
    → Boot, run health checks
    → If healthy: Add back to Load Balancer backend pool
    → If unhealthy: Auto-rollback (revert to previous version)
    → Wait for Batch 1 fully healthy

  Step 2: Upgrade Batch 2 (next 100 instances)
    → Same process as Batch 1
    → Batches run sequentially (not parallel)

  Step 3: Repeat until all instances upgraded

  Key Points:
    → Always minimum instances running (20% in example: 100 instances always up)
    → Health probe: Determines if instance is ready for traffic
    → Automatic rollback: If health check fails during upgrade
    → Manual upgrade: Pause between batches for validation
    → Update Domain: Aligns with VMSS upgrade policy (UD mapping)

  VMSS Max Surge (preview/feature):
    → Allows adding instances during upgrade (before removing old)
    → Example: 100 instances → upgrade to 120 (add 20 new, remove 20 old)
    → Benefit: Zero-downtime upgrades (always 100+ instances running)
    → Risk: Higher cost during upgrade (extra instances)

  VMSS Configuration Update (in-place):
    → Update OS disk image → triggers automatic rolling upgrade
    → Update extensions → triggers rolling upgrade
    → Can specify: Unhealthy instance exit (auto-remediation)
```

---

### 3.4 AZURE APP SERVICE — COMPLETE DEEP DIVE

**What is Azure App Service?**
Azure App Service is a fully managed platform for building, deploying, and scaling web applications and REST APIs. It provides a self-patching runtime (OS and framework managed by Azure), built-in CI/CD, auto-scaling, and deployment slots.

**App Service Components:**

```
Azure App Service:
  ├── App Service Plan (hosting plan / compute budget):
  │   ├── SKU: Defines compute size (F1=free, B1=basic, P1V2=Premium, etc.)
  │   ├── Instances: Number of VMs running apps (1+ per plan, auto-scale)
  │   ├── OS: Windows or Linux (chosen per plan)
  │   ├── Workers: Total compute = instances × SKU size
  │   ├── Isolated (Preview/GA): VNet-integrated (no shared infrastructure)
  │   └── Elastic (Premium): Auto-scaling within plan (PremiumV2/V3)
  │
  ├── Web App / API / Function:
  │   → One or more apps hosted per App Service Plan
  │   → Runtime: .NET, Java, Node.js, Python, PHP, Ruby
  │   → Auto-deploy from: Git, GitHub, Azure DevOps, Container Registry
  │   → Deployment slots: Dev, Staging, Production (with auto-swap)
  │   → Always On: Keep app loaded (disable idle timeout — requires Paid/Isolated)
  │
  ├── Scaling:
  │   ├── Manual: Scale instances (1–20 per SKU)
  │   ├── Auto-scale: CPU, memory, HTTP queue, custom metric
  │   ├── Scale up: Change SKU (P1 → P2V2 = more compute per instance)
  │   └── Scale out: Add instances (1 → 5 instances = more parallelism)
  │
  ├── Networking:
  │   ├── VNet Integration (Regional): Route to VNet (regional, private)
  │   ├── VNet Integration (AzurePrivateLinkService): Private endpoint access to VNet
  │   ├── Outbound IP: One or more outbound IPs (SNAT for outbound)
  │   ├── Inbound IP: Single IP for app (no SNAT)
  │   └── Hybrid Connection: Connect to on-prem (without VPN)
  │
  ├── Security:
  │   ├── Authentication / Authorization (Easy Auth)
  │   │   → Entra ID, Google, Facebook, Twitter, Anonymous
  │   │   → App Service manages auth flow (no code needed)
  │   │   → Token validation handled by App Service
  │   │
  │   ├── Managed Identity (system-assigned or user-assigned)
  │   │   → App accesses other Azure services (Storage, SQL, Key Vault)
  │   │   → No credentials in code
  │   │
  │   ├── IP Restrictions (Access Restrictions)
  │   │   → Allow/deny specific IPs or subnets
  │   │   → Use: Restrict to VPN, specific IPs only
  │   │
  │   ├── Private Endpoint (for inbound private access)
  │   │   → Private IP in VNet for App Service
  │   │
  │   ├── SCM (Deployment) credentials (separate from app credentials)
  │   │   → Kudu/SCM site: scm.<app>.azurewebsites.net
  │   │   → Separate auth from main app
  │   │
  │   └── HTTPS Only: Redirect HTTP → HTTPS (Enforce)
  │
  └── Deployment Slots:
      ├── Production: Live app serving traffic
      ├── Staging: Pre-production (same SKU as production)
      ├── Dev: Development (smaller SKU)
      ├── Swap: Production ↔ Staging (with preview: test before switching)
      └── Auto-swap: Auto swap after health check passes
```

**App Service Plan SKU Comparison:**

| SKU | Tier | Instances | VNet Integration | Use Case | Monthly Cost (approx) |
|-----|------|-----------|-----------------|---------|----------------------|
| F1 | Free | 1 (shared) | No | Dev/test only | Free |
| D1 | Shared | 1 (shared) | No | Dev/test | ~$10 |
| B1 | Basic | 1-3 | No (Isolated only) | Small production | ~$13 |
| B2 | Basic | 1-3 | No | Small production | ~$27 |
| B3 | Basic | 1-3 | No | Small production | ~$55 |
| P1V2 | Premium | 1-10 | Regional | Production web apps | ~$150 |
| P2V2 | Premium | 1-10 | Regional | Production web apps | ~$300 |
| P3V2 | Premium | 1-10 | Regional | Enterprise apps | ~$550 |
| P1V3 | Premium (Isolated) | 1-10 | Internal (VNet only) | Compliance apps | ~$370 |
| P2V3 | Premium (Isolated) | 1-10 | Internal (VNet only) | Enterprise apps | ~$740 |
| P3V3 | Premium (Isolated) | 1-10 | Internal (VNet only) | Mission-critical | ~$1,475 |
| I1 | Isolated v1 | 1-20 | Internal (VNet only) | Enterprise apps | ~$2,585 |
| I2 | Isolated v1 | 1-20 | Internal (VNet only) | Mission-critical | ~$5,170 |
| I3 | Isolated v1 | 1-20 | Internal (VNet only) | Largest apps | ~$10,340 |
| I1v2 | Isolated v2 | 1-20 | Internal (VNet only) | Enterprise apps | ~$2,585 |
| I2v2 | Isolated v2 | 1-20 | Internal (VNet only) | Mission-critical | ~$5,170 |

> **L3 critical:** Premium v2/v3 plans can auto-scale (elastic): instances within plan can auto-scale between min and max based on CPU/memory/HTTP queue. Isolated plans: Fixed instances (no auto-scale within plan); scale up/down manually. Always On: Only available in paid tiers (B+). Free/shared tiers idle apps after 20 minutes of inactivity.

**App Service Auto-Scale:**

```
App Service Auto-Scale (within single plan):
  → Maximum instances per SKU:
    F1/D1: 1 (no scaling)
    B1/B2/B3: 3 max
    P1V2/P1V3: 10 max
    P2V2/P2V3: 10 max
    P3V2/P3V3: 10 max
    I1/I2/I3: 20 max (Isolated)

  Auto-Scale Rules:
    CPU > 80% (avg across instances): Scale out 1 instance
    CPU < 30% (avg across instances): Scale in 1 instance
    HTTP queue length > 20: Scale out 2 instances
    Memory > 80%: Scale out 1 instance (P2V2+ only)
    Schedule: Scale to max during business hours, scale down at night
    Custom metric: Any Azure Monitor metric (e.g., request count)

  Scale Out Behavior:
    → New instances start in seconds (pre-warmed via Always On)
    → Load Balancer routes traffic to new instances automatically
    → Health check: New instances must pass (HTTP 200 on /) before routing
    → Scale In: Removes instances slowly (graceful shutdown, drain connections)

  Scale Up Behavior:
    → Change SKU (P1 → P2): App restarts (brief downtime, 5-10 minutes)
    → Schedule-based scale up: Off-peak, planned
    → For zero-downtime scaling: Scale out instead of up

  Limitation: Max instances per SKU. For 100 instances:
    → Use multiple App Service Plans (each up to max SKU limit)
    → Use Azure Kubernetes Service (AKS) instead
    → Use VM Scale Set (up to 1,000 instances per VMSS)
```

**App Service Deployment Slots:**

```
Deployment Slots:

  Production Slot:
    → Live application serving all traffic
    → Always running (production traffic = primary route)
    → Cannot be deleted

  Staging Slot:
    → Identical app running in same plan (same compute)
    → Swappable with Production (instant swap)
    → Swap types:
      → Auto-swap: After deployment → auto-swap to production
      → Manual swap: You trigger swap when ready
      → Preview swap: Test in production slot before committing
    → Health check: HTTP endpoint must return 200 before swap completes

  Swap Process:
    1. Staging slot receives new deployment
    2. Testing on staging URL: <app>-staging.azurewebsites.net
    3. Swap triggered:
       a. Production name → Staging (URL changes)
       b. Staging app moves to Production
       c. Old Production app moves to Staging (old version)
    4. Health check: New production responds → swap confirmed
    5. If health check fails: Rollback (swap back automatically)

  Zero-Downtime Deployment:
    1. Deploy to staging slot
    2. Test on staging URL
    3. Swap to production (instant, no downtime — DNS/hostname swapped)
    4. Old app (previous version) runs in staging slot
    5. If issue: Swap back (rollback to previous version)
    6. Monitoring: Application Insights tracks health post-swap

  Database Migrations During Swap:
    → Problem: Database schema changes may break old app version (in staging slot after swap)
    → Solution: "Do not drop" database migrations (run scripts in both slots)
    → Best practice: Backward-compatible DB changes only
    → Run migration BEFORE swap (on production slot), then swap

  Configuration:
    → Each slot has own app settings (can override production values)
    → Staging settings: Different connection strings, feature flags, etc.
    → "Sticky" settings: Scale-set, deployment-related settings (stick to slot)
    → Non-sticky: App settings, connection strings (follow during swap)
    → Manage via: Azure Portal or CLI (az webapp config appsettings set)
```

**App Service Networking — VNet Integration:**

```
VNet Integration (Regional) — Premium/Isolated SKU only:

  Without VNet Integration:
    App Service → Internet → Target (SQL, Storage, etc.)
    → Traffic goes over internet (even to Azure services)
    → Outbound IPs: 5 (SNAT-based, shared with other App Service plans)
    → Inbound: Public IP only

  With VNet Integration (Regional):
    App Service → VNet (gateway) → Target in VNet (private)
    → Traffic stays on Azure backbone
    → Access: VNet resources (VMs, Private Endpoints, etc.)
    → Cannot access: Internet (requires Gateway or NAT Gateway for outbound internet)
    → For outbound internet + VNet access: Use VNet Integration + NAT Gateway

  Architecture:
    App Service Plan (P2V2) → Regional VNet Integration → Gateway Subnet
    → VNet Integration Gateway: 4 instances (auto-scaled)
    → Points to: VNet subnets with private endpoints
    → Access: VMs in VNet, Private Endpoints, Azure SQL Private, Storage Private

  VNet Integration Limitations:
    → Regional only (same region as App Service)
    → Subnet size: /27 minimum (100+ instances needs larger)
    → Gateway: Auto-created and managed (4 instances, auto-scale)
    → Cannot use: Port 80/443 from VNet to App Service (SNAT port exhaustion)
    → Use Regional VNet Integration only for: Access to VNet resources, not internet

  Hybrid Connection (alternative for on-prem access):
    → Connect App Service to on-prem resources (without VPN)
    → Requires: Hybrid Connection Manager (HCM) installed on-prem
    → Flow: App Service → Hybrid Connection → HCM (on-prem) → On-prem resource
    → Use: Access on-prem databases, file shares, APIs from App Service
    → NOT for: Full VNet integration (no VNet route, no private connectivity)

  Outbound IP Addresses:
    App Service has multiple outbound IPs (SNAT - Source Network Address Translation)
    → App uses random outbound IP for each request (load distribution)
    → Allowlist on target: Must allow ALL outbound IPs (not just one)
    → View: Portal → App Service → Properties → Outbound IP Addresses
    → "Outbound IP Addresses" and "Possible Outbound IP Addresses"
    → Always include all "possible" IPs in allowlists (they can be used)
    → White-listing only one IP = intermittent failures (SNAT selects different IP)

  VNet Integration Best Practice:
    → For Azure services access: Use Private Endpoints (App Service can have Private Endpoint too!)
    → For VNet resources: Use VNet Integration (regional)
    → For on-prem: Hybrid Connection OR VNet Integration + Gateway (if VPN exists)
    → For internet: NAT Gateway (with VNet Integration)
```

**App Service Authentication and Authorization:**

```
Easy Auth (Authentication / Authorization):
  → Built-in authentication module (pre-app code)
  → Intercepts all requests → Validates token → Passes user info to app
  → Identity Providers:
    → Entra ID (Azure AD) — most common
    → Google, Facebook, Twitter — social login
    → Anonymous — no auth required (public access)
  
  → Flow:
    1. User requests App Service
    2. Easy Auth intercepts
    3. If not authenticated → redirect to login page (provider)
    4. User authenticates with provider
    5. Token returned to Easy Auth
    6. Easy Auth validates token
    7. Request passed to app with user claims (X-MS-CLIENT-PRINCIPAL header)
    8. App reads user identity from header (no auth code needed)

  → Configuration:
    → Authentication: On (or Off)
    → Action: Login with provider / Allow anonymous
    → Token Store: Enabled (persist tokens for API calls)
    → Allowed Tokens: 24 hours (default)

  → App Service does NOT validate tokens in app code;
    it handles all validation before the request reaches your code.
    This is the key difference from manual authentication.

  → Managed Identity + Easy Auth:
    → System-assigned managed identity for app-to-app authentication
    → App A (with MI) → calls App B → App B validates caller MI
    → No user interaction needed (service-to-service)
    → Configure: App B Authentication → Allowed Token Audiences → App A's MI ID
```

**App Service Limitations (L3 Critical):**

```
What App Service CAN do:
  → Host web apps, REST APIs, mobile backends
  → Auto-scale (within SKU limits)
  → Deployment slots (dev/staging/prod)
  → Managed runtime (.NET, Java, Node, Python, PHP)
  → VNet Integration (regional, Premium+)
  → Private Endpoint
  → Managed Identity, Easy Auth
  → CI/CD (GitHub, Azure DevOps, Container)
  → Logging, Application Insights integration
  → Custom domains, SSL certificates (managed)

What App Service CANNOT do:
  → Full VM-level control (no RDP/SSH to underlying OS)
  → Custom OS configuration (registry, services, daemons)
  → Install Windows Services or Linux daemons
  → Long-running background processes (>20 minutes idle timeout unless Always On)
  → Shared across multiple plans (1 app per plan... wait, multiple apps per plan YES)
  → Direct access to compute layer (VM underneath)
  → Run Docker containers natively (use Container Instances/AKS/Registry instead)
  → Scale beyond SKU instance limits (use VMSS/AKS for 100+ instances)
  → Access within 100% VNet (VNet Integration has limitations)

When NOT to use App Service:
  → Need OS-level control → Use VMs
  → Need to run Windows Services → Use VMs
  → Need 100+ instances → Use AKS or VMSS
  → Need containers → Use ACI/AKS
  → Need long-running background jobs (hours) → Use Functions (consumption) or VMs
  → Need stateful connections (WebSocket limited) → Use VMs
```

---

### 3.5 CONTAINERS — COMPLETE DEEP DIVE

#### 3.5.1 Azure Container Instances (ACI)

**What is Azure Container Instances?**
ACI is the simplest way to run a container in Azure without managing any VMs or orchestration. It's serverless containers — you provide the image, ACI runs it, you pay per second of execution.

**ACI Architecture:**

```
Azure Container Instance:
  ├── Container Group:
  │   → One or more containers (same or different images)
  │   → Shared resources: IP, DNS, volumes, network
  │   → OS: Linux or Windows
  │   → Restart policy: Always, OnFailure, Never
  │
  ├── Container:
  │   → Image: ACR/ECR/DockerHub/public registry
  │   → CPU: 0.25–8 vCPUs
  │   → Memory: 0.5–30 GB
  │   → Ports: Exposed ports (80, 443, etc.)
  │   → Environment variables
  │   → Volume mounts (azurefile, azureblob, emptyDir)
  │
  ├── IP Address:
  │   → Public IP (optional, for internet access)
  │   → DNS name label (e.g., myapp.eastus.azurecontainer.io)
  │   → Internal (no public IP, VNet-only)
  │
  ├── OS Disk:
  │   → Ephemeral (default, fast, no persistent storage)
  │   → Managed disk (persistent, survives restarts)
  │
  ├── Scaling: Manual only (no auto-scale in ACI)
  │
  └── Billing: Per-second (minimum 1 second, rounded to nearest second)

Use Cases:
  → Dev/test (quick spin-up, tear-down)
  → Batch jobs (run job, stop, pay per second)
  → Single microservice (simple, no orchestration needed)
  → CI/CD build agents (run build, then stop)
  → Event-driven processing (Azure Functions + ACI for heavy processing)
  → Migration testing (run old app in container temporarily)

NOT for production:
  → No orchestration (no scaling, no self-healing, no rolling updates)
  → No service discovery
  → No built-in monitoring (Application Insights manual)
  → For production: Use AKS or App Service (Containers) instead
```

**ACI Networking:**

```
ACI Networking Options:

  Public IP (default):
    → Container gets public IP
    → Accessible from internet
    → DNS label: <name>.<region>.azurecontainer.io
    → Use: Dev/test, public-facing containers

  Internal (VNet Integration — Preview):
    → Container gets private IP in VNet
    → Not accessible from internet (access via VNet only)
    → Can access: Other VNet resources, Private Endpoints, Azure Private services
    → Use: Secure containers, backend services, data processing
    → Configuration: Subnet + IP address (must be in subnet range)
    → VNet must be in same region

  No IP (run in VNet):
    → Container runs in VNet context
    → Access via DNS resolution from VNet resources
    → Use: Background processing, event-driven workloads

ACI + Virtual Network:
  → ACI can join existing VNet (Preview/GA features expanding)
  → Container IP: Private (within VNet subnet)
  → Access: From VMs, App Service (VNet Integration), other ACI containers in same group
  → Cannot: Access ACI from internet directly (use public IP or VPN/ExpressRoute)

ACI + Azure Private Link:
  → ACI can access Private Link endpoints in same VNet
  → ACI cannot have its own Private Endpoint (yet)
  → Workaround: VNet integration + DNS resolution

ACI Limitations:
  → No auto-scaling (manual scale = deploy more ACI)
  → No rolling updates (deploy new, stop old)
  → No health probes
  → Max CPU: 8 vCPUs, Max Memory: 30 GB per container
  → Container groups: Max 10 containers per group (for sidecar pattern)
  → Ephemeral OS disk: Lost on restart (unless managed disk specified)
  → No GPU support (currently in preview)
```

---

#### 3.5.2 Azure Container Apps (ACA)

**What is Azure Container Apps?**
Azure Container Apps is a serverless container platform that runs Docker containers without requiring you to manage infrastructure, orchestrators, or VMs. It uses KEDA (Kubernetes Event-Driven Autoscaling) for scaling and provides built-in ingress, service discovery, secrets, and jobs.

**ACA Architecture:**

```
Azure Container App: "my-container-app"
  ├── Environment:
  │   → Links: Log Analytics, Azure Container Registry (optional)
  │   → Dapr: Service mesh (mRNA architecture)
  │   → Infrastructure: Managed by ACA (no user management)
  │
  ├── Container App:
  │   → Ingress: Disabled (no external traffic), Internal (VNet), or External (public)
  │   → Scale Rules:
  │       → CPU/Memory (HPA-like)
  │       → HTTP requests (KEDA HTTP trigger)
  │       → Azure Service Bus (KEDA Service Bus trigger)
  │       → Azure Storage Queue (KEDA Queue trigger)
  │       → Kafka, Redis, Event Hubs, etc. (event-driven scaling)
  │       → Min replicas: 0 (scale to zero — true serverless)
  │       → Max replicas: 20 (or more)
  │   → Replicas: Current running instances (auto-scaled based on rules)
  │   → Terminate: Grace period for shutdown (drain connections)
  │
  ├── Container:
  │   → Image: ACR/ECR/DockerHub
  │   → CPU/Memory: Per container
  │   → Environment Variables
  │   → Azure Storage Volumes (Blob, File)
  │   → Secrets (Key Vault reference)
  │
  ├── Job:
  │   → Parallel jobs (scale based on workload)
  │   → Scheduled jobs (cron expression)
  │   → Event-driven jobs (KEDA triggers)
  │
  └── Service Discovery:
      → Other Container Apps in same environment can connect via service name
      → Dapr: mTLS, service invocation, state management, pub/sub

Key Features:
  → Min replicas: 0 (scale to zero — no charge when idle)
  → Event-driven scaling: KEDA-based (Kafka, Service Bus, Queue, HTTP)
  → No Kubernetes: Serverless, no node management
  → Built-in ingress: HTTP/TCP with headers-based routing
  → Secrets: Key Vault references (no plain text secrets)
  → Dapr integration: Service mesh for microservices
  → Jobs: Run-to-completion (batch, scheduled)
  → Volume mounts: Azure Storage (Blob, File)
  → Outbound: VNet Integration (Preview/GA)

ACA vs AKS vs ACI Comparison:
  ┌─ ACA ──────────┐ vs ┌─ AKS ───────────┐ vs ┌─ ACI ────────────┐
  → No k8s         │    → Kubernetes     │    → No orchestrator  │
  → Scale to zero  │    → No scale to zero│    → Scale to zero   │
  → Simple         │    → Complex (k8s)   │    → Simple           │
  → Event-driven   │    → HPA/VPA/KEDA   │    → Manual only      │
  → Serverless     │    → PaaS (managed)  │    → Serverless       │
  → No VNet (yet)  │    → VNet native    │    → VNet (Preview)  │
  → Limited traffic  │    → Full traffic │    → Single          │
    handling          │    → Full traffic      │    container/group    │

When to use ACA:
  → Event-driven microservices (scale to zero, event triggers)
  → Simple containerized apps (no k8s complexity)
  → Jobs (scheduled/parallel/batch)
  → Dev/test (quick deployment)

When NOT to use ACA:
  → Need k8s features (RBAC, Helm, operators, network policies)
  → Existing k8s investment (use AKS)
  → Stateful apps with persistent storage (AKS has more options)
  → Need VNet integration (use AKS or ACI VNet Preview)
  → Complex service mesh (use AKS + Istio/Linkerd)
```

**ACA Scaling Deep Dive:**

```
ACA Scale Rules (KEDA-based):

  1. HTTP Scale:
     → Trigger: HTTP requests per second
     → Min replicas: 0 (scale to zero when no traffic)
     → Max replicas: 20
     → Cooldown: 300 seconds (5 minutes)
     → Target: 100 concurrent requests per replica
     → Behavior: Scale up when requests spike, scale down when idle
     → Scale to zero: No replicas running when no requests (zero cost)

  2. CPU/Memory Scale:
     → Trigger: Average CPU > 70% across replicas
     → Scale out: Add replica when threshold exceeded
     → Scale in: Remove replica when below threshold
     → Min replicas: 1 (always running) — or 0 (scale to zero)

  3. Service Bus Queue Scale:
     → Trigger: Messages in queue (per partition/topic)
     → Scale: Max(1, messages / (messages-per-replica))
     → Example: 100 messages, 10 per replica = 10 replicas
     → When queue empty: Scale to 0 (or min replicas)

  4. Azure Blob Scale:
     → Trigger: Blob events (new blob in container)
     → Process each blob: 1 replica per blob (or batch)
     → Scale: Based on pending blobs (unprocessed)
     → Use: Image processing, file conversion, data ingestion

  5. Kafka Scale:
     → Trigger: Kafka topic lag
     → Scale: Based on lag (messages behind)
     → Each replica processes: N messages
     → Use: Stream processing, event consumers

  6. Cron/Job Scale:
     → Scheduled: Cron expression → Scale to N replicas at scheduled time → Scale back down
     → Parallel: Each job = 1 replica (max = job count)

Scale to Zero Behavior:
  → Min replicas = 0: ACA terminates all replicas when no activity
  → When event arrives: Scale from 0 → 1 replica (cold start)
  → Cold start: ~30 seconds (image pull + startup — depends on image size)
  → Image size impact: Large images (2 GB+) = longer cold start
  → Mitigation: Keep 1 min replica (not 0) for latency-sensitive apps
  → Cost: No compute charges during scale-to-zero period

  Important: Scale to zero works for ACA but NOT for AKS (unless Virtual Node/KEDA on AKS)
```

---

#### 3.5.3 Azure Kubernetes Service (AKS) — COMPLETE DEEP DIVE

**What is Azure Kubernetes Service?**
AKS is a managed Kubernetes service that simplifies deploying, managing, and scaling containerized applications using Kubernetes. Microsoft manages the control plane (API server, etcd, scheduler, controller manager), and you manage the data plane (nodes/pods/containers).

> **Key distinction from ACA:** AKS gives you full Kubernetes (all features, complexity included). ACA gives you serverless containers (simplified, no Kubernetes). AKS for enterprise microservices; ACA for simple event-driven containers.

**AKS Architecture:**

```
AKS Cluster:
  ├── Control Plane (MANAGED by Microsoft — you don't manage):
  │   ├── API Server: REST API (kubectl, Helm, etc.)
  │   ├── etcd: Distributed key-value store (cluster state)
  │   ├── Scheduler: Assigns pods to nodes
  │   ├── Controller Manager: Replication, endpoints, namespace
  │   └── Managed by Azure (HA across 3 availability zones)
  │
  ├── Agent Pools (Node Pools — YOU manage):
  │   ├── System Node Pool (default):
  │   │   → Runs system pods (kubelet, kube-proxy, DNS, metrics-server)
  │   │   → Minimum 1 node (can be scaled)
  │   │   → SKU: VM size (Standard_D2s_v3 etc.)
  │   │
  │   ├── User Node Pool(s):
  │   │   → Runs your application pods
  │   │   → Can be multiple pools (GPU, CPU-optimized, memory-optimized)
  │   │   → Each pool: Independent scaling, OS, labels, taints
  │   │
  │   └── Spot Node Pool (optional):
  │       → Low-priority VMs (Azure Spot)
  │       → Cost: 60-90% cheaper than regular VMs
  │       → Risk: Eviction when Azure needs capacity (230-second warning)
  │       → Use: Stateless, interruptible workloads (batch, CI/CD)
  │       → Not for: Production stateful apps
  │
  ├── Pods (smallest deployable unit in k8s):
  │   → One or more containers (tightly coupled, shared namespace)
  │   → Shared storage (volumes), shared network (IP)
  │   → Lifecycle: Pending → Running → Succeeded/Failed
  │   → Restart policy: Always (default), OnFailure, Never
  │
  ├── Services (abstraction for pod access):
  │   ├── ClusterIP (default): Internal only (cluster-internal)
  │   ├── NodePort: Exposed on each node's static port
  │   ├── LoadBalancer: Cloud Load Balancer (external) — auto-created by AKS
  │   └── ExternalName: DNS alias (CNAME)
  │
  ├── Ingress (HTTP/L7 routing):
  │   ├── Application Gateway Ingress Controller (AGIC):
  │   │   → Application Gateway v2 as Ingress
  │   │   → WAF, SSL, routing based on host/path
  │   │   → Most common for production AKS
  │   │
  │   ├── NGINX Ingress Controller:
  │   │   → Lightweight, open-source
  │   │   → Configured via Ingress resources
  │   │
  │   └── Traefik, Istio, Contour: Alternatives
  │
  ├── Add-ons:
  │   ├── Azure CNI (Kubernetes Network Plugin):
  │   │   → Each pod gets VNet IP (direct VNet access)
  │   │   → More IPs per pod (VNet IP space)
  │   │   → Requires VNet integration (AKS-native)
  │   │
  │   ├── Kubenet (default for small clusters):
  │   │   → Pods use NAT (masquerade) to access VNet
  │   │   → Fewer IPs needed (shared node IP)
  │   │   → Limitation: No direct pod-to-VNet access (NAT only)
  │   │
  │   ├── HTTP Application Routing (Harbor):
  │   │   → Auto-creates Ingress rules based on labels
  │   │
  │   ├── Azure Policy (Integrate):
  │   │   → Enforce policies on AKS (e.g., no privileged containers)
  │   │
  │   ├── OPA Gatekeeper (Integrate):
  │   │   → Policy enforcement (e.g., no root containers, required labels)
  │   │
  │   └── Azure RBAC for Kubernetes Authorization:
  │       → RBAC (kubectl create/delete pods) via Azure roles
  │       → Namespace-level or cluster-level permissions
  │       → Entra ID groups → k8s roles
  │
  ├── Monitoring:
  │   ├── Azure Monitor for Containers:
  │   │   → Container logs, metrics, events
  │   │   → Container health, resource utilization
  │   │
  │   ├── Container Insights:
  │   │   → Centralized monitoring, cluster-level dashboards
  │   │   → Performance, capacity planning
  │   │
  │   └── Log Analytics:
  │       → All k8s logs, events, resource metrics
  │
  └── Identity:
      ├── Cluster Identity (Managed Identity):
      │   → AKS control plane identity
      │   → Used for AKS API calls (create/delete/modify)
      │
      └── Node Identity (Managed Identity):
          → VMs in node pools have Managed Identity
          → Pods use node MI (or assigned MI per pod — Pod Identity)
          → Access to Azure services from pods
```

**AKS Node Pool Types:**

```
1. System Node Pool:
   → Purpose: Runs Kubernetes system components
   → Required: Yes (at least 1)
   → Auto-scaling: Enabled (can scale system nodes)
   → Example: Default node pool (replaces System pool in newer AKS)
   → Min: 1 node (recommended 2 for HA)

2. User Node Pools:
   → Purpose: Run application workloads
   → Multiple pools: CPU, GPU, Memory, etc.
   → Can have different OS (Windows + Linux in same cluster)
   → Labels/taints: For pod scheduling affinity
   → Auto-scaling: Min/max nodes per pool

3. Virtual Node (ACI Connector):
   → Runs ACI pods in AKS cluster
   → No VMs for burst workloads (ACI on demand)
   → Kubernetes runs pod → If no node available → Use ACI
   → Cost: Only ACI charges (no VM cost for burst)
   → Limitation: Single node concept (no HA for virtual nodes)
   → Use: Event-driven, burst, not always-on production

4. GPU Node Pool:
   → VM size: NCas/Turing/ND (GPU series)
   → Use: ML training, inferencing, visualization
   → Taints: Prevent non-GPU pods from scheduling on GPU nodes
   → Driver: NVIDIA device plugin installed

5. Spot Node Pool:
   → VMs: Azure Spot (low-priority)
   → Cost: 60-90% cheaper
   → Risk: Eviction (230-second warning)
   → Tolerate: Interrupted jobs, CI/CD, batch processing
   → Configuration: Node labels, eviction policy
   → Combine: Spot + On-demand in same pool (not supported; use separate pools)

6. Ambient (secure workloads):
   → Node pool with ambient security (In-tree vs Out-of-tree)
   → Limitations: GPU not supported, specific SKUs only
   → Use: Enhanced security workloads

AKS Node Pool Sizing:
  → Min nodes: 0 (for user pools) → Scale to zero during idle
  → Max nodes: 1,000 per pool (10,000 across cluster)
  → Recommended: Start with auto-scaling (min 2, max appropriate for workload)

AKS Upgrade Strategies:
  → Rolling Update (default): Nodes updated one at a time (Cordon → Drain → Upgrade → Uncordon)
  → Force Update: Replace VMs (not OS upgrade)
  → Node Image Update: Update OS image without replacing node ( Hotfixes)
```

**AKS Scaling and Auto-Scale:**

```
AKS Scaling Mechanisms:

  1. Cluster Auto-Scaler (CAVS):
     → Runs as part of K8s cluster
     → Monitors: Pods that can't be scheduled (insufficient resources)
     → Action: Add nodes to scale set
     → Scale In: Remove unused nodes (after 10-minute idle — configurable)
     → Min/Max: Per node pool configuration
     → Default: Enabled on user node pools
     → Trigger: Unschedulable pod → Add node → Reschedule pod
     → Scale down criteria:
       a. Node has no pods that need rescheduling (all pods can move)
       b. Node has been underutilized for 10 minutes
       c. Pod PDB (Pod Disruption Budget) respected

  2. Horizontal Pod Autoscaler (HPA):
     → Pod-level scaling (NOT node-level)
     → Monitors: CPU/memory usage per pod OR custom metrics
     → Action: Increase pod replicas (not VM nodes)
     → Example:
       → Current: 3 pods, avg CPU = 80%
       → Target: 50% CPU per pod
       → Scale: 3 pods → 5 pods (to reduce CPU per pod to 50%)
     → Min/Max: Per deployment (e.g., min 2, max 20)
     → Cooldown: 15 seconds (scale up), 5 minutes (scale down)

  3. Vertical Pod Autoscaler (VPA):
     → Adjusts pod resource requests (CPU/memory) automatically
     → Action: Increase/decrease pod CPU/memory based on usage
     → Modes:
       → Off: Only recommendation (no changes)
       → Initial: Sets requests based on historical usage (at pod start)
       → Auto: Adjusts resources at runtime (restart pod)
     → Use: Optimize resource allocation (don't over/under-provision pods)

  4. KEDA (Kubernetes Event-driven Autoscaling):
     → External metric-driven scaling
     → Triggers: Kafka, Service Bus, Storage Queue, Redis, HTTP, Cron, etc.
     → Scales pods (not nodes) based on external metric
     → Use: Event-driven microservices (scale from 0 based on queue depth)
     → Integration: ACA uses KEDA; AKS can also use KEDA

  5. Manual Scaling:
     → Set min/max for cluster autoscaler
     → Scale node pool (VM size: change VM SKU in node pool)
     → Scale pod replicas (kubectl scale deployment — replicas count)
     → kubectl: kubectl scale deployment/myapp --replicas=10

AKS + Application Gateway Ingress Controller (AGIC):
  → AGIC: Ingress controller that creates Application Gateway rules from Kubernetes Ingress resources
  → Flow: Developer creates K8s Ingress → AGIC watches Ingress events → Updates Application Gateway routes → Traffic routes to correct pods
  → Benefits: WAF, SSL, path/host-based routing, auto-scaling backend pools
  → Setup:
    → AKS + VNet integration + Application Gateway v2 + AGIC add-on
    → Application Gateway: Private (VNet only) or Public
    → AGIC: Installs as pod in AKS cluster

  AGIC Workflow:
    1. Create K8s Ingress resource:
       apiVersion: networking.k8s.io/v1
       kind: Ingress
       metadata:
         name: myapp-ingress
         annotations:
           kubernetes.io/ingress.class: azure/application-gateway
           appgw.ingress.kubernetes.io/use-private-ip: "true"
       spec:
         rules:
         - host: myapp.contoso.com
           http:
             paths:
             - path: /
               pathType: Prefix
               backend:
                 service:
                   name: myapp-service
                   port:
                     number: 80

    2. AGIC watches for new Ingress events
    3. AGIC creates Application Gateway rules
    4. User requests myapp.contoso.com → AGIC routes to K8s pods

  AGIC + WAF:
    → Application Gateway v2 with WAF enabled
    → OWASP managed rules (same as standalone WAF)
    → Custom rules for specific applications (K8s namespace-based)

  AGIC + Private Link:
    → Application Gateway: Private (internal only)
    → No public exposure for backend services
    → Frontend: Private IP in VNet
    → Backend pools: AKS pods (via AGIC)
```

**AKS Security:**

```
1. Azure RBAC for Kubernetes Authorization:
   → RBAC (cluster-level): Who can create/delete clusters (Azure RBAC)
   → RBAC (k8s-level): Who can create/delete pods/deployments (k8s RBAC)
   → Entra ID groups → k8s Roles (cluster-admin, admin, edit, view)
   → Namespace-scoped or cluster-scoped permissions
   → Configuration: Azure portal or `az aks op assign-rbac`

2. Pod Security Standards (PSS):
   → Restricted (most restrictive): No privilege escalation, no root, no host network
   → Baseline: Minimal privileges (deny privilege escalation, read-only root FS)
   → Privileged: Full access (NOT recommended, used by system components)
   → Enforced via: OPA Gatekeeper / Kyverno policies
   → AKS: Default = baseline for user pools, restricted for system pools

3. Azure Policy for AKS:
   → Enforce: Allowed VM SKUs, no privileged containers, required labels, no host network, etc.
   → Audit mode: Review compliance
   → Deny mode: Prevent non-compliant resources
   → Built-in policies: AKS-specific

4. Network Policies:
   → Calico/Cilium (CNI plugins): Control pod-to-pod network access
   → Default: All pod traffic allowed (no network policies)
   → After enabling: Only explicitly allowed traffic flows between pods
   → Use: Isolate frontend/backend pods (frontend can talk to backend only)

5. Secrets Management:
   → Kubernetes Secrets: Base64 encoded (NOT encrypted by default)
   → Azure Key Vault integration:
     → CSI driver (AzureKeyVaultSecretsProvider): Mount Key Vault secrets as volumes
     → Secrets Store CSI Driver: Sync secrets from KV to K8s secrets
     → AKS: Add-on = "azure-keyvault-secrets-provider"
   → Best practice: NEVER store secrets in K8s secrets; use Key Vault CSI driver

6. Private Cluster:
   → Control plane endpoint: Private IP only (no public)
   → API server: Accessible only from authorized VNet/VPN
   → Node communication: Through VNet only
   → Use: Compliance, security-sensitive workloads
   → Setup: Enable "Private cluster" during AKS creation

7. Azure Defender for Containers:
   → Threat detection for AKS (runtime threat alerts)
   → Detects: Cryptocurrency mining, privilege escalation, unusual network activity
   → Integration: Microsoft Sentinel for alert correlation
   → Agent: Microsoft Defender for Containers agent on nodes

8. Managed Identity for AKS:
   → Cluster Identity (MSI): For AKS API operations
   → Node Identity (MSI): For pod access to Azure services
   → Pod Identity (Azure Pod Identity / Workload Identity):
     → Workload Identity (newer): Microsoft Entra ID tokens for pods
     → Uses: Azure AD pod identity (proposed) — replaces older Azure Pod Identity
     → Flow: Pod requests token from AKS OIDC issuer → Validates with Entra ID → Access Azure service
```

**AKS Monitoring:**

```
Azure Monitor for Containers:
  Metrics (container-level):
    → Container CPU usage, memory usage, network bytes
    → Pod restart count (high restarts = problem)
    → Container disk usage (ephemeral storage)

  Logs:
    → Container logs (stdout/stderr from each container)
    → Kubernetes events (pod scheduling, errors, warnings)
    → System logs from nodes (syslog equivalent)

  Dashboards:
    → Container Insights: Cluster summary, node health, pod status
    → Namespace view: Pods, services, deployments within namespace
    → Pod view: Individual pod metrics, logs, events

  Alerts:
    → Pod restart count > 1 in 5 minutes → Investigate application
    → Node CPU > 90% for 10 minutes → Scale out
    → Container memory > 90% of limits → Scale up or restart
    → Pod pending > 5 minutes → Resource constraints or scheduling issue

  Integration with Application Insights:
    → Per-pod Application Insights instance
    → Distributed tracing across microservices
    → Dependency tracking (pod → SQL DB, pod → Storage)
    → Custom events, metrics, logs
    → Live Metrics Stream (real-time)
```

---

#### 3.5.4 Azure Container Registry (ACR)

**What is Azure Container Registry?**
ACR is a managed, private Docker container registry. It stores Docker images (and OCI artifacts) for container deployment, with geo-replication, build capabilities, and integration with other Azure services.

**ACR Architecture:**

```
Azure Container Registry: "acrcontoso"
  ├── Repositories:
  │   → Each repository: Contains tagged images
  │   → Example: acrcontoso.azurecr.io/myapp:v1.0.0
  │   → Tags: version, environment (v1.0.0, v2.0.0, latest, staging)
  │   → Manifest: Image layers + configuration (OCI/Docker format)
  │   → Layers: Reused across images (same base image, different apps)
  │
  ├── Geo-Replication:
  │   → Enabled: Replicate images to multiple ACR regions
  │   → Benefit: Push once → Replicate automatically to all regions
  │   → Pull locally: AKS/VMs pull from nearest region (faster, cheaper)
  │   → Configuration: Per ACR (enable for regions where you have services)
  │
  ├── Build:
  │   → ACR Tasks (build on demand, scheduled, or on commit)
  │   → Source: GitHub, Azure DevOps, ACR repo
  │   → Dockerfile: Reference in ACR task
  │   → Context: Build context (source code + Dockerfile)
  │   → Output: Image pushed to ACR automatically
  │
  ├── ACR Tasks (build automation):
  │   → Task: Pre-built script (Dockerfile in repo)
  │   → Step: Run, Set, File, External (GitHub/DevOps)
  │   → Trigger: On push to branch, scheduled (cron), manual
  │   → Build context: Automatically pulls from source
  │
  ├── Quota and Limits:
  │   → SKU-based: Basic, Standard, Premium (storage and throughput)
  │   → Geo-replication: SKU-dependent (Premium allows more)
  │   → Build: SKU-dependent (compute limits per build)
  │
  ├── Data Protection:
  │   → Admin user: Enabled/Disabled (recommended: Disable)
  │   → Managed Identity: ACR access via MI (recommended)
  │   → Repository permissions: Reader, Writer, Owner, Admin (per repo)
  │   → AAD authentication: Entra ID-based access
  │
  └── Integration:
      → AKS: ACR pull image for deployments
      → ACI: ACR pull image for container instances
      → ACR Tasks: Auto-build from ACR
      → Azure DevOps / GitHub Actions: CI/CD pipeline build
```

**ACR SKU Comparison:**

| SKU | Storage | Geo-Replication | Build | Use Case | Monthly Cost (approx) |
|-----|---------|----------------|-------|---------|----------------------|
| **Basic** | 10 GB | 1 region (primary only) | Limited (1 build at a time) | Dev/test | ~$50 |
| **Standard** | 100 GB | 5 regions (configurable) | Standard (build compute) | Production | ~$325 |
| **Premium** | 500 GB | 10+ regions | Enhanced (faster builds) | Enterprise, global | ~$875 |

**ACR Pull Authentication:**

```
Authentication for AKS/ACI pulling images from ACR:

  1. Managed Identity (RECOMMENDED for AKS):
     → AKS node pool: Assign Identity → ACR pull role
     → AKS: acrpull role (pull images from ACR)
     → Node MI: Pull images without credentials
     → Configuration:
        az role assignment create \
          --assignee <msi-id> \
          --role acrpull \
          --scope /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.ContainerRegistry/registries/acrcontoso

  2. ACR Pull Role on AKS (cluster-wide or per-node-pool):
     → Grant acrpull to AKS cluster identity (for all node pools)
     → Or per node pool (recommended for least privilege)

  3. Service Principal (legacy):
     → SP with acrpull role
     → AKS: Store credentials in K8s secrets
     → Problem: Credentials in K8s secrets (not ideal)
     → Use: Migrate to MI-based authentication

  4. Admin User (NOT recommended):
     → ACR admin credentials (username/password)
     → Disable admin user after setup
     → Problem: Shared credentials, no granular access

  5. Anonymous Pull (public ACR):
     → Only for public images (Docker Hub)
     → ACR private registry: Always requires authentication
     → ACR public tier (preview): Allow anonymous pulls for specific repos
```

---

## 4. DESIGN PATTERNS

### 4.1 Web Application Architecture (App Service)

```
Front Door (global WAF, SSL, CDN)
  ↓
Application Gateway v2 (regional WAF) in VNet
  ↓
App Service (PremiumV2/PremiumV3) — Auto-scaled (P1V2 → P2V2 based on CPU)
  ↓
VNet Integration (regional) → Access VNet resources privately
  ↓
Azure SQL DB (Private Endpoint) — Business Critical tier
  ↓
Azure Storage (Private Endpoint) — Blob Storage for static content
  ↓
Azure Cache for Redis (VNet) — Session caching

Security:
  → App Service: Managed Identity → SQL DB (no connection string credentials)
  → App Service: Easy Auth (Entra ID SSO + MFA)
  → App Service: IP Restrictions (VPN only for admin)
  → Deployment Slots: Dev → Staging → Production (swap with preview)

Scaling:
  → Auto-scale: CPU > 70% → Scale out (add instances)
  → Scale up: Manual (P1V2 → P2V2) during known events
  → Schedule: Scale to max during business hours, scale down at night
```

### 4.2 Microservices Architecture (AKS)

```
Front Door (global entry)
  ↓
Application Gateway (WAF) in AKS VNet (Private)
  ↓
AGIC (Application Gateway Ingress Controller) — Routes to K8s Ingress resources
  ↓
Kubernetes Cluster (AKS)
  ├── System Node Pool (2+ nodes, Standard_D4s_v3)
  │   → System pods (kube-dns, metrics-server, coredns, etc.)
  │
  ├── User Node Pool — Frontend (autoscale 3-10, Standard_D2s_v3)
  │   → Pods: Frontend app (React, Angular, Blazor)
  │   → HPA: CPU > 70% → Scale out (3 → 10 pods)
  │
  ├── User Node Pool — Backend (autoscale 2-20, Standard_D4s_v3)
  │   → Pods: API, Business logic
  │   → HPA: HTTP queue length > 50 → Scale out
  │
  ├── User Node Pool — ML/AI (spot, 2-5, Standard_NC6)
  │   → Pods: ML inference, GPU workloads
  │
  ├── Virtual Node (ACI) — Burst workloads
  │   → When no AKS nodes available → Run on ACI (on-demand)
  │
  ├── Namespaces:
  │   → frontend: Frontend pods
  │   → backend: API pods
  │   → data: Data processing pods
  │   → monitoring: Prometheus, Grafana (optional)
  │
  └── Services:
      → ClusterIP: Internal pod-to-pod communication
      → LoadBalancer: External exposure (via Application Gateway)

  Storage: ACR (image registry, geo-replicated)
  Secrets: Azure Key Vault (mounted via CSI driver)
  Monitoring: Azure Monitor for Containers + Application Insights

Data Tier:
  Azure SQL DB (Private Endpoint, zone-redundant)
  Azure Cosmos DB (Private Endpoint, multi-region)
  Azure Storage (Private Endpoint, Blob Storage)
```

### 4.3 Container Migration Pattern

```
Migration Path: On-Prem VMs → AKS (Container Migration)

  Phase 1: Assessment
    → Identify: Which VMs to migrate (stateless web apps first)
    → Containerize: Create Docker images (Dockerfile, build, test)
    → Store: Push images to ACR (Standard SKU, geo-replicate to production region)

  Phase 2: Pilot
    → Migrate: 1-2 low-risk apps to AKS (test namespace)
    → Validate: Performance, functionality, monitoring
    → Security: Network policies, managed identity, private endpoints

  Phase 3: Production
    → Migrate: Remaining apps to AKS (proper namespaces per app)
    → Scale: Auto-scale configurations, HPA rules
    → HA: Multi-zone node pools, VNet integration

  Phase 4: Decommission
    → Verify: All apps running on AKS
    → Backup: Take final VM snapshots before decommission
    → Remove: VM resources, NSGs, public IPs
    → Monitor: AKS performance, cost comparison
```

---

## 5. COST COMPARISON

| Scenario | Service | Configuration | Monthly Cost (approx) |
|----------|---------|---------------|----------------------|
| Simple web app | App Service B1 | 1 instance, Basic | ~$13 |
| Production web app | App Service P2V2 | 2 instances, Premium (autoscale to 5) | ~$600-1,500 |
| VM for dev/test | VM Standard_B2s | 2 vCPU, 4 GB | ~$25 |
| VM for production web | VM Standard_D2s_v3 | 2 vCPU, 8 GB | ~$70 |
| VM for SQL Server | VM Standard_E8s_v3 | 8 vCPU, 64 GB | ~$550 |
| VM for high-perf DB | VM Standard_L8s_v2 | 8 vCPU, 128 GB, local SSD | ~$860 |
| VM Scale Set (web) | 10× D2s_v3 | 10 VMs, load balanced | ~$700-1,000 |
| VM Scale Set (autoscale) | 2-10× D2s_v3 | Autoscale based on CPU | ~$140-1,000 (variable) |
| Single container (dev) | ACI | 1 vCPU, 2 GB | ~$15-50 (per-second) |
| Microservice (scale to zero) | ACA | 1-10 replicas, HTTP scale | ~$20-200 (variable) |
| Kubernetes cluster | AKS | 2-10 nodes (autoscale) | ~$150-2,000 (VM costs) |
| Container registry | ACR Standard | 100 GB storage | ~$325 |

---

## 6. PRODUCTION EXAMPLE

**Scenario: Global e-commerce platform (10M+ users, 100K concurrent, multi-region, high availability, PCI compliance).**

```
GLOBAL E-COMMERCE PLATFORM

REGIONS: East US 2, West US 2, West Europe, Southeast Asia

EAST US 2 (Primary):
  Virtual WAN Hub: vhub-prod-eastus2 (Scale Units: 6)
  ├── Firewall: fw-prod-eastus2 (Standard HA pair)
  ├── Bastion: bastion-prod-eastus2
  ├── ExpressRoute Gateway: er-prod-eastus2 (Circuit: 10G Premium)
  ├── VPN Gateway: vpn-prod-eastus2 (VpnGw3 — backup)
  │
  ├── Spoke-Web: vnet-spoke-web-eastus2 (10.1.0.0/16)
  │   ├── App Service (PremiumV3, P2V3 × 4 instances, autoscale to 10)
  │   │   → Front Door → App Gateway → App Service
  │   │   → Deployment slots: Dev/Staging/Prod (slot staging, prod active)
  │   │   → Auto-scaling: CPU > 70% → Add instances
  │   │   → Always On: Enabled
  │   │   → VNet Integration: Yes (regional)
  │   │   → Managed Identity: Yes (SQL DB, Storage access)
  │   │   → Easy Auth: Entra ID (SSO)
  │   │
  │   ├── VMSS — Admin Portal (D2s_v3, 2-8 instances, zone-redundant)
  │   │   → Internal only (no public access)
  │   │   → Load Balanced within VNet
  │   │
  │   └── AKS — ML Inference (node pools):
  │       → System: 2× D2s_v3 (zone-redundant)
  │       → GPU: NC6 (2-5, spot + on-demand)
  │       → CPU: D4s_v3 (2-10, autoscale)
  │       → Application Gateway Ingress Controller (WAF enabled)
  │       → ACR: acrprod-eastus2 (Standard, geo-replicated)
  │
  ├── Spoke-Data: vnet-spoke-data-eastus2 (10.2.0.0/16)
  │   ├── Azure SQL DB (Private Endpoint, Business Critical)
  │   │   → Active Geo-Replication: West US 2 (read-only)
  │   │   → Threat Detection: Microsoft Defender for SQL
  │   │   → TDE: CMK (Key Vault)
  │   │   → Auditing: Long-term retention (7 years)
  │   │
  │   ├── Azure Cache for Redis (Private Endpoint)
  │   │   → Session caching, cart data
  │   │
  │   ├── ADLS Gen2 (Private Endpoint, Standard_GRS)
  │   │   → Product images, reports, logs
  │   │   → Lifecycle: Hot 90 days → Cool 365 days → Archive 365 days
  │   │
  │   └── Cosmos DB (Private Endpoint, SQL API)
  │       → Multi-region writes: East US + West US
  │       → Session consistency (default), Strong for orders
  │       → Change Feed → Azure Functions (sync to search index)

WEST US 2 (Secondary/DR):
  ├── Mirror of East US (read-only replica of data)
  ├── AKS: Standby (same configuration, scaled to min)
  ├── App Service: Staging (automatically promoted during failover)
  └── Geo-replicated storage (GRS from East US)

WEST EUROPE (Customer-Facing — EU):
  ├── App Service P2V3 (EU customer-facing, autoscale 3-10)
  ├── Cosmos DB (West Europe — multi-region read)
  ├── ADLS Gen2 (West Europe — regional data)
  └── Application Gateway (WAF) → Front Door

SOUTHEAST ASIA (Customer-Facing — APAC):
  ├── App Service P2V3 (APAC customer-facing, autoscale 3-10)
  ├── Cosmos DB (Southeast Asia — multi-region read)
  ├── ACR: acrprod-sea (Standard, geo-replicated)
  └── Application Gateway (WAF) → Front Door

GLOBAL ENTRY:
  Front Door (Global):
    → Global WAF (OWASP managed rules + custom rules)
    → DDoS Protection Standard (all endpoints)
    → CDN: Static content (product images, CSS, JS)
    → SSL: Global certificate
    → Routing: Latency-based (nearest region)
    → Health probes: Per-region app health (auto-route away from unhealthy region)
    → Geo-filtering: Block high-risk countries (optional)

MONITORING:
  Azure Monitor: All metrics, alerts, dashboards
  Log Analytics: All logs (App Service, AKS, SQL, Cosmos, Firewall)
  Application Insights: Per-app monitoring, dependency tracking, alerts
  Microsoft Sentinel: Security alerts, threat detection
  NSG Flow Logs: All subnets (Log Analytics per region)
  Firewall Logs: Centralized (Log Analytics)

BACKUP:
  Azure Backup Vault: VMs, App Service (if using VNet backup), AKS (etcd backup)
  SQL DB: PITR (35 days) + LTR (7 years)
  Cosmos DB: Continuous backup + scheduled export (7-year LTR)
  Storage: Soft Delete (90 days), Immutable (legal hold for compliance data)

SECURITY:
  All PaaS: Private Endpoint only (no public access)
  All VMs: No public IPs (Bastion for access)
  All data: CMK encryption at rest, TLS 1.2 in transit
  All access: Entra ID + MFA (Conditional Access: compliant devices only)
  All APIs: Managed Identity (no connection string secrets)
  PCI compliance: Network segmentation, audit logging, vulnerability scanning

DISASTER RECOVERY:
  Primary: East US 2
  Secondary: West US 2 (automated failover for Cosmos DB, manual for App Service/SQL)
  RTO: 4 hours (full region failover)
  RPO: 5 seconds (Cosmos DB), 5 minutes (SQL DB), 15 minutes (Storage)
  DR drill: Annually

COST MANAGEMENT:
  Tags: Environment=Production, Project=Ecommerce, CostCenter=Web, Criticality=Tier1, DataClassification=PII
  Budget: $50,000/month per region (alert at 80%)
  Reserved Instances: SQL DB (1-year, 3-year), AKS nodes (1-year), App Service (1-year)
  Cost optimization: App Service scale-down at night (schedule-based), Cosmos DB autoscale, VMSS spot instances
```

---

## 7. FAILURE SCENARIOS

| Scenario | Root Cause | Resolution | Evidence |
|-----------|-----------|------------|----------|
| **VM CPU at 100% sustained (performance degradation)** | VM size too small for workload; Memory leak in application; No auto-scale (single VM) | Upgrade VM SKU (D2s_v3 → D4s_v3); Add VMSS with autoscale; Profile application for memory leak; Scale out (multiple VMs). Most common: Single VM handling production load (should use VMSS or App Service for auto-scale). | Portal: VM → Metrics (CPU %); Application: Memory usage trend; VMSS: Instance count; Activity Log: Scaling history |
| **App Service returns "503 Service Unavailable"** | App Service Plan exhausted (max instances reached); App crashed (out of memory); Deployment slot swap failed; SKU too small. Most common: Out of memory crash (app restarting in loop, health probe fails). | Check: App Service logs (failed request, crash); Check: Instances count (max reached?); Scale up (P2V2 → P3V2); Fix: Memory leak in code; Restart: Always On enabled (prevents idle timeout). | Portal: App Service → Health Checks; Logs: Failed requests; Diagnose: Memory dumps; Configuration: Always On |
| **VMSS instances not scaling out (CPU > 80%, no new VMs)** | VMSS autoscale rules not configured; VM size limit reached; Insufficient vCPU quota; VMSS in single fault domain (capacity limit). Most common: Insufficient vCPU quota (Azure quota limit for region). | Check: Activity Log → VMSS scaling → Error message; Check: Subscription quota (vCPUs); Check: Scaling rules (configured? metric? threshold?); Increase quota (Azure Support request); Scale up VM size to reduce vCPU per VM. | Portal: VMSS → Scaling rules: configured? Activity Log: scaling attempts; Metrics: CPU; Log Analytics: scaling operations and errors; Azure: Subscription limits |
| **AKS pods stuck in Pending state** | Insufficient node resources; Node pool at max capacity; Taints on nodes (pod can't schedule); Pod resource requests exceed available; Network policy blocking. Most common: Node pool maxed out (min=max, no room for new pods). | Check: kubectl describe pod → Events (Insufficient CPU/Memory); Check: Node pool auto-scale max; kubectl get nodes (capacity); Add more nodes (scale up node pool); Add node pool (GPU, spot); Increase node pool max; Fix taints. | kubectl: describe pod, get nodes; AKS: Node pool metrics (node count, CPU/Memory); Azure Monitor: Pending pods count |
| **AKS pod crash looping (CrashLoopBackOff)** | Application crashed on startup; Missing environment variables; Volume mount failed; Liveness probe failed (app not ready); OOMKilled (out of memory). Most common: OOMKilled (memory limits too low) or missing config (env var not set). | kubectl logs → Check crash logs; kubectl describe pod → Events (OOMKilled, etc.); Check: Resource limits (memory too low?); Check: Environment variables; Check: Volume mounts; Check: Liveness probe settings; Fix: Increase memory limit, add env vars, adjust probes. | kubectl: logs, describe pod; Events: OOMKilled, Liveness probe; AKS: Container logs; Pod: Restart count |
| **Application Gateway returning 502 (backend unhealthy)** | Backend health probe fails; App Service/VMs not responding; Port mismatch; NSG blocking health probe; Backend pool empty. Most common: Health probe configured on wrong port/path (app doesn't respond on /health on port 80). | Check: Backend health (App Gateway → Backend health: Healthy/Unhealthy); Check: Health probe config (port, path, interval); Test: curl from Gateway subnet to backend port; NSG: Allow probe source; App: Respond to probe endpoint. | App Gateway: Backend health; Health probe config; Manual test: curl from App GW subnet to backend IP:port; Logs: Health probe results |
| **Cosmos DB changed consistency level unexpectedly** | Someone changed default consistency level; Per-request consistency overrides; Client SDK not configured correctly. Most common: Someone changed account-level default consistency from Session to Eventual (causing stale reads). | Check: Cosmos DB account → Settings → Default Consistency Level; Check: Per-request consistency (x-ms-consistency-level header); Verify: Application behavior; Revert: Default to Session consistency. | Portal: Cosmos DB → Settings → Consistency level; SDK: Consistency configuration; Application logs: Stale data detected; Monitor: Read latency |
| **AKS cluster autoscaler not scaling (pods unschedulable but no new nodes)** | Cluster autoscaler not enabled on node pool; Node pool at max nodes; Insufficient vCPU quota; Node pool Taints prevent node scaling; VM Scale Set health issues. Most common: Node pool max nodes reached (max = max capacity, autoscaler can't add more). | Check: Cluster autoscaler enabled; Node pool max: Increase max; Check: vCPU quota; Check: Taints and node pool configuration; VMSS: Health check; Increase cluster capacity. | kubectl: describe unschedulable pods; AKS: Node pool configuration; Activity Log: Scaling events; VMSS: Instance count |
| **AKS node pool upgrade causing downtime** | Rolling upgrade: Pod disruption budget (PDB) not configured; Disrupted pods not rescheduled; Insufficient capacity for rescheduling; Migration failure. Most common: PDB limits pod disruption (too strict — prevents rescheduling). | Check: kubectl get pdb; Increase PDB maxUnavailable (default: 1); Ensure: Sufficient capacity in cluster; Use: Eviction tolerance in PDB; Rolling update: Verify with kubectl rollout status. | kubectl: get pdb, describe node, rollout status; AKS: Upgrade history; Events: Node draining failures |
| **ACI container exits immediately (crash)** | Image not found in ACR; Wrong image tag; Entrypoint/command not found; Environment variables missing; Authentication failure (ACR pull). Most common: Wrong image tag (using old or non-existent tag in deployment config). | Check: ACI events (Error: Image not found); Verify: Image name and tag in ACR; kubectl: describe if applicable; Check: Environment variables; Check: ACR pull authentication. | ACI: Container events; ACR: Image tags; Activity Log: ACI creation events; Container: Logs |
| **ACA container not scaling (0 replicas but traffic incoming)** | Min replicas set to 0 and cold start prevents immediate availability; Scale rule not configured for HTTP or wrong metric; ACA environment not linked to Log Analytics; KEDA scaling not enabled. Most common: Scale rule configured for wrong port/endpoint (health probe failing). | Check: Scale rules in ACA; Min replicas: Set to 1 for always-on (avoid cold start); Health probe: Verify /health endpoint; Scale rule: Match correct port and protocol; Monitor: Scale event logs. | ACA: Scale rules, replica count; Logs: Scaling activities; Application: Request rate; Portal: Scale events |
| **App Service VNet Integration not working (can't reach VNet resources)** | VNet Integration gateway not provisioned (takes time); Subnet too small (not /27 minimum); VNet address space conflict; Backend resources not configured for private access (no Private Endpoint); Firewall blocking. Most common: VNet Integration takes 10+ minutes to provision and queries fail until ready. | Check: VNet Integration → Provisioning state (Succeeded?); Check: Subnet size (/27 minimum); Check: Backend resource private endpoint configuration; VNet: Check address spaces (no overlap); DNS: Resolve private IP; Test: From App Service (outbound IP) to target. | VNet Integration portal: Provisioning state; Subnet: Address space; nslookup: Private endpoint hostname from App Service (via test endpoint); Kudu: Console test |
| **VM Boot Diagnostics shows "No boot data"** | OS disk corruption; Incorrect image reference; OS disk not attached properly; VM configured with no bootable disk; Boot diagnostics storage account issue. Most common: OS disk failed to attach properly during deployment (rare) or custom image has no bootable OS. | Rebuild VM: Redeploy from market image; Check: OS disk attached (Portal → VM → Disks); Check: Boot diagnostics: Storage account accessible; Check: Custom image: Valid OS installed; Use: Serial console for troubleshooting. | Boot Diagnostics: Screenshots/screens; Portal: VM → Disks; VM: Serial console; Activity Log: Disk attachment events |
| **VM extension fails (Custom Script)** | Storage account unreachable; Script not found; Script has syntax errors; Extension timed out; Insufficient permissions on storage account. Most common: Script URL is incorrect or storage account firewall blocking Azure platform access. | Check: Extension status (Azure Portal → VM → Extensions); Script URL: Accessible from Azure platform; Storage account: Firewall allows Azure services; Script: Test locally before deploying; Increase: Timeout if long-running. | Portal: VM → Extensions (status: Failed); Activity Log: Extension events; Storage: Access logs; Script: Test URL accessibility |
| **Application Gateway autoscale hitting max instances** | Traffic spike exceeds max autoscale limit; WAF rules consuming excessive compute; Large response payloads; Too many backend health checks. Most common: Traffic spike exceeds max autoscale instances (e.g., max = 10, but need 15). | Increase: Autoscale max instances (e.g., 10 → 20); Front Door: Add CDN (cache static content); Optimize: WAF rules (reduce complexity); Scale up: Larger SKU (WAF_v2 → WAF_v3); Add: Second Application Gateway in active/active. | App Gateway: Metrics (CU usage, autoscale events); Autoscale: Min/max config; Log Analytics: Scaling activities; WAF: Request rate, blocked requests |
| **Secure VM (Confidential VM) failing attestation** | TPM not supported by VM image; Attestation report not generated; Azure Attestation service unavailable; UEFI settings not configured. Most common: Using Windows image that doesn't support vTPM/Secure Boot (older images). | Check: VM size supports Confidential VMs; Check: OS image supports vTPM; Enable: vTPM and Secure Boot in VM settings; Attestation: Azure Attestation service status; Update: VM image to latest version. | VM: VM settings (TPM, Secure Boot); Attestation: Report generation; Portal: VM → Security (confidential VM status); VM: UEFI settings |

---

## 8. MONITORING — COMPUTE

| Metric/Log | Service | What It Shows | Alert Trigger |
|------------|---------|---------------|---------------|
| **VM CPU %** | Azure Monitor | CPU utilization per VM | CPU > 80% sustained (5 min) → Scale up/out |
| **VM Memory %** | Azure Monitor | Memory utilization per VM | Memory > 85% → Scale up, investigate leak |
| **VM Disk IOPS** | Azure Monitor | Disk read/write IOPS per disk | IOPS > 80% of disk limit → Upgrade disk type |
| **VM Disk Throughput** | Azure Monitor | Disk read/write throughput | Throughput > 80% of limit → Upgrade disk |
| **VM Disk Latency** | Azure Monitor | Disk read/write latency | Latency > 10ms (SSD) → Investigate workload |
| **VM Network In/Out** | Azure Monitor | Network bytes per VM | Sudden spike → Security investigation |
| **VM Status Check** | Azure Monitor | VM overall health | VM unreachable → Investigate restart |
| **VM Availability** | Azure Monitor | VM uptime percentage | < 99.9% → HA investigation |
| **VMSS Instance Count** | Azure Monitor | Current VMSS instances | Unexpected scale down → Check scaling rules |
| **VMSS CPU Average** | Azure Monitor | Average CPU across VMSS instances | CPU > 70% → Auto-scale triggered |
| **App Service CPU %** | Azure Monitor | CPU per App Service instance | CPU > 80% → Scale out |
| **App Service HTTP Queue** | Azure Monitor | Requests waiting in queue | Queue > 20 → Scale out (P2V2+ only) |
| **App Service Response Time** | Azure Monitor | Response time per request | P95 > 2 seconds → Investigate |
| **App Service Instance Count** | Azure Monitor | Running instances | Scale up/down verification |
| **App Service Always On** | Azure Monitor | App loaded status | App unloaded (idle timeout) → Enable Always On |
| **AKS Node CPU %** | Azure Monitor | Per-node CPU utilization | CPU > 80% → Cluster autoscaler trigger |
| **AKS Pod CPU %** | Azure Monitor | Per-pod CPU (per replica) | CPU > 80% → HPA scale out |
| **AKS Pod Memory %** | Azure Monitor | Per-pod memory utilization | Memory > 80% → HPA scale out or VPA adjust |
| **AKS Pod Restart Count** | Azure Monitor | Restarts per pod | Restarts > 0 in 5 min → Investigate crash |
| **AKS Node Count** | Azure Monitor | Current node count | Min/max not met → Check autoscaler |
| **AKS Pending Pods** | Azure Monitor | Pods waiting for scheduling | Pending > 1 min → Resource/quota issue |
| **AKS Node Not Ready** | Azure Monitor | Node health | Node Not Ready → Investigate (kubectl describe) |
| **AKS API Server Latency** | Azure Monitor | API server response time | Latency > 500ms → Contact Microsoft |
| **AKS CronHPA Cron Trigger** | KEDA/HPA | Cron schedule adherence | Cron-triggered scale failed → Check KEDA logs |
| **ACA Replicas** | Azure Monitor | Current running replicas | Min replicas 0, traffic spike → Cold start latency |
| **ACA Scale Event** | Azure Monitor | Scale in/out activities | Scale failed → Check scale rule configuration |
| **ACI Execution Time** | Azure Monitor | Container lifetime | Long-running (unexpected) → Investigate |
| **Front Door Latency** | Azure Front Door | Global response time | Latency > 500ms → Check backend health |
| **Application Gateway WAF Blocks** | Application Gateway | Blocked requests (WAF) | Spike in blocks → Investigate attacks |
| **Application Gateway Backend Health** | Application Gateway | Backend instance health | Unhealthy backends → Investigate |
| **DDoS Mitigation** | DDoS Protection | Mitigated traffic volume | Sustained mitigation → Investigate attack |
| **VPN Gateway Tunnel Status** | Azure Monitor | S2S/P2S tunnel status | Tunnel Down → BGP/PSK investigation |
| **ExpressRoute BGP Status** | Azure Monitor | BGP peer status | BGP Down → Provider/router investigation |
| **ExpressRoute Circuit Bits** | Azure Monitor | Throughput on circuit | Near capacity (10G) → Upgrade circuit |
| **Virtual WAN Hub Throughput** | Azure Monitor | Hub capacity utilization | > 80% → Scale up hub (scale units) |
| **Virtual WAN Attachment Status** | Azure Monitor | VNet/VPN/ER attachment health | Attachment Failed → Troubleshoot |
| **Bastion Session Count** | Azure Monitor | Active Bastion sessions | Unusual session count → Security investigation |

---

## 9. DIAGNOSTIC SETTINGS — COMPUTE

| Resource | Log Category | Destination |
|----------|-------------|------------|
| **VM** | Boot, Metrics, Guest OS (via MMA) | Log Analytics + Storage + Azure Monitor |
| **VMSS** | Metrics, Autoscale events, Instance view | Log Analytics + Azure Monitor |
| **App Service** | App Service logs (request, application, web server, http access), Deployment logs | Log Analytics + Storage |
| **AKS** | Container logs, Container metrics, Cluster events, K8s audit, API server logs, kube logs | Log Analytics + Storage + Azure Monitor |
| **AKS Add-ons** | CNI logs, DNS logs, ingress controller logs | Log Analytics |
| **ACA** | Container logs, Scale logs, Execution logs, Job logs | Log Analytics + Azure Monitor |
| **ACI** | Container logs, Events | Log Analytics + Storage |
| **ACR** | Authentication logs, Pull/Push logs, Task run logs | Log Analytics + Storage |
| **Application Gateway** | Access, WAF, Performance, SSL, Health Probe | Log Analytics + Storage |
| **Front Door** | Access, WAF, DDoS, Front Door health | Log Analytics + Storage |
| **VPN Gateway** | IKE, Connect, P2S, Routes, Diagnostics | Log Analytics + Storage |
| **ExpressRoute Gateway** | Connect, BGP, Route Filters, Health | Log Analytics + Storage |
| **Virtual WAN Hub** | Connection Monitor, Route events, Hub metrics | Log Analytics + Azure Monitor |
| **Bastion** | Session logs, error logs, activity logs | Log Analytics + Storage |
| **Azure Backup** | Backup job logs, Restore logs, Alert logs | Log Analytics |
| **Activity Log** | All resource management operations (deploy, modify, delete) | Log Analytics + Storage |
| **Alert Rules** | All scaling, threshold, and operational alerts | Azure Monitor → Action Groups |

---

## 10. SECURITY CONSIDERATIONS

| Concern | Risk | Mitigation |
|---------|------|------------|
| **VM exposed to internet (public IP + no NSG)** | VM directly accessible; brute-force RDP/SSH; malware | No public IPs; Bastion; NSG restrict; Just-in-Time access |
| **VM with admin username/password** | Brute-force authentication; credential leak | SSH key (Linux); Strong password (Windows + rotation); Entra ID SSO |
| **Unmanaged disks (unmanaged blob)** | Storage account management overhead; URL leak risk | Always use Managed Disks |
| **Standard HDD for database workloads** | Low IOPS; database performance degrades | Premium SSD/Ultra Disk for database workloads |
| **Disk caching misconfigured for databases** | Stale cache data served; data integrity risk | None caching for database disks; ReadOnly/ReadWrite for app |
| **Single VM for production workload** | Single point of failure; VM down = app down | VMSS; Availability Set; Availability Zones |
| **VMSS single-zone only** | Datacenter failure = all VMSS VMs down | Zone-redundant VMSS (across 3 zones) |
| **App Service on Free/Shared tier** | No HA; shared resources; idle timeout; no VNet | Paid tier minimum for production (B1+) |
| **App Service without Always On** | App unloads after 20 min idle (F1/D1 only) | Enable Always On (Paid tier) |
| **App Service IP allowlist incomplete** | SNAT rotation allows unauthorized IPs | Allow all outbound IPs + possible outbound IPs |
| **ACI no orchestration (manual scale)** | No auto-scale; manual instance management | Use ACA or AKS for production scaling |
| **ACA min replicas = 0 for latency-sensitive apps** | Cold start: 30+ seconds delay | Min replicas = 1 (always running) |
| **AKS no network policies** | All pod-to-pod traffic allowed (no isolation) | Enable Calico/Cilium network policies |
| **AKS PSS not enforced (privileged containers)** | Privileged containers = full host access | Enforce Restricted/Baseline PSS via OPA/Kyverno |
| **AKS secrets in plain text** | K8s secrets base64 encoded (not encrypted) | Azure Key Vault CSI driver |
| **AKS public control plane endpoint** | API server accessible from internet | Enable private cluster |
| **ACR admin user enabled** | Shared credentials; broad access | Disable admin; use MI-based auth |
| **Container images from untrusted sources** | Malware in container images | ACR vulnerability scanning; AAD auth for ACR; Private registry only |
| **AKS not using Private Endpoint** | AKS API server publicly accessible | Private cluster; ACR private; private endpoints |
| **App Service Easy Auth disabled** | No authentication on web app | Easy Auth enabled with Entra ID |
| **Scale set without load balancer** | VMs receive traffic unevenly or not at all | Application Gateway or Load Balancer in front of VMSS |
| **VMSS uniform mode with custom config per VM** | Uniform mode = all VMs identical; custom configs don't persist | Use Flexible mode or configure via Custom Script Extension |
| **AKS not using Managed Identity for ACR** | Credentials in K8s secrets (not secure) | MI-based ACR pull (acrpull role) |
| **Private endpoint not approved** | Connection waits for approval; access blocked | Configure auto-approval or approve promptly |
| **AKS cluster autoscaler with spot nodes only** | Spot nodes can be evicted at any time | Mix spot + on-demand; PDB; pod disruption budgets |
| **AKS not backed up (etcd)** | Cluster state lost; all resources deleted | AKS backup (etcd snapshots) to Storage Account |
| **VPN Gateway no Dead Peer Detection** | Dead connections hold resources; tunnel stuck | Enable DPD (30 sec) on both sides |
| **Bastion in VM subnet (not dedicated)** | Bastion requires AzureBastionSubnet | Dedicated AzureBastionSubnet (/27 min) |
| **Application Gateway v1 (no autoscale)** | Manual scaling; bottleneck under load | Upgrade to v2 (autoscale, Private Link, etc.) |
| **DDoS Basic only (no alerts)** | No notification of attacks; limited mitigation | DDoS Standard (alerts, adaptive, WAF integration) |
| **ExpressRoute single circuit (no DR)** | Provider failure = no on-prem connectivity | Dual circuits + VPN backup |
| **No NSG Flow Logs** | No audit trail of network access | Enable on all production NSGs |

---

## 11. L3 INTERVIEW QUESTIONS

### Basic
**Q: What is the difference between IAAS, PAAS, and SAAS?**
A: IaaS: You manage OS, runtime, data (VMs, VMs). PaaS: You manage data and app code (App Service, AKS — platform manages OS/runtime). SaaS: You only use the application (Microsoft 365, Salesforce — Microsoft manages everything).

### Basic
**Q: When would you use a VM vs App Service?**
A: VM: Need OS-level control, custom extensions, any OS configuration, Windows services, background processes. App Service: Simple web apps/REST APIs, no OS control needed, auto-scaling built-in, deployment slots. Key question: Do you need to install Windows services or Linux daemons? If yes → VM; If no → App Service.

### Basic
**Q: What is a VM Scale Set?**
A: A set of identical, auto-scaled VMs that are load balanced. All VMs have the same configuration. Auto-scale based on CPU, memory, or schedule. Use for: Stateless web/API workloads that can scale horizontally.

### Basic
**Q: What is the difference between Standard HDD and Premium SSD?**
A: Standard HDD: Cheapest (~$0.02/GB/month), low IOPS (100 max), suitable for backup/archive. Premium SSD: More expensive (~$0.126/GB/month), high IOPS (up to 30,000), suitable for production databases and applications.

### Basic
**Q: What is a Deployment Slot in App Service?**
A: Deployment slots are separate environments within the same App Service Plan. Typical slots: Dev, Staging, Production. Swapping: Exchange URLs between slots (zero-downtime deployment). Staging receives new code → Swap to Production → Old code moves to Staging.

### Intermediate
**Q: What is the difference between App Service and AKS?**
A: App Service: Fully managed PaaS for web apps; No container orchestration; Auto-scaling within SKU limits; Deployment slots; No Kubernetes complexity. AKS: Managed Kubernetes; Full container orchestration; Horizontal pod scaling; Rolling deployments; Requires knowledge of Kubernetes; More flexible but more complex.

### Intermediate
**Q: What is the difference between ACI, ACA, and AKS?**
A: ACI: Single container, no orchestration, serverless (pay per second), burst/scale to zero manually. ACA: Container-based, serverless orchestration (no Kubernetes), event-driven scaling (KEDA), scale to zero built-in, simplified. AKS: Full Kubernetes, complete orchestration, most flexible, most complex, requires k8s knowledge.

### Intermediate
**Q: What is VMSS and how does it differ from a single VM?**
A: VMSS = multiple identical VMs, load balanced, auto-scaled. Single VM = one VM, no auto-scale, manual management. VMSS provides: horizontal scaling, uniform configuration, automatic instance management, integrated load balancing, deployment templates. Use VMSS for stateless, scalable workloads; single VM for unique, stateful workloads.

### Intermediate
**Q: What is Availability Set vs Availability Zone?**
A: Availability Set: VMs distributed across fault/upgrade domains within ONE datacenter. Cheap, adequate for most scenarios. Availability Zone: VMs in physically separate datacenters within ONE region. More expensive, maximum resilience (survives entire datacenter failure). Both provide HA but at different scales.

### Intermediate
**Q: What is the difference between VM Scale Set and App Service?**
A: VMSS: IaaS (VMs you manage), full OS control, requires patching/updates, auto-scale on VM count, load balanced. App Service: PaaS (no VM management), managed runtime, auto-scaling within SKU (instance count), deployment slots, built-in CI/CD. VMSS for: Custom/legacy/OS-specific apps. App Service for: Modern web/API apps.

### L3
**Q: Explain VM disk caching. When should you use None vs ReadWrite vs ReadOnly?**
A: Disk caching controls how disk I/O is cached in VM memory. None: No caching (I/O goes directly to storage disk) — recommended for database disks (prevents stale data). ReadOnly: Reads are cached, writes go directly — good for read-heavy app data. ReadWrite: Both reads and writes are cached — good for OS disk (fast boot, but write-back cache risks data loss on crash).

### L3
**Q: Explain how Application Gateway WAF and Front Door WAF work together.**
A: User → Front Door (global entry, global WAF, CDN, rate limiting, bot management) → Application Gateway (regional WAF, L7 routing, backend health, SSL termination) → Backend VMs/App Service/AKS. Defense in depth: Front Door blocks global attacks (volumetric, OWASP), Application Gateway provides regional L7 filtering and routing. Both use OWASP managed rules but have independent rule sets.

### L3
**Q: A customer has a VM Scale Set with 10 instances behind Load Balancer. Users report intermittent timeouts. What would you investigate?**
A:
1. VM health: All 10 instances running? (VMSS → Instances)
2. Load balancer health probe: All instances healthy? (Health probe config — port, path, interval)
3. VM metrics: CPU/Memory/Network on all instances? (Unbalanced load or VM under-resourced)
4. Backend health: Application responding on health endpoint on all instances?
5. NSG: Load balancer health probe traffic allowed? (NSG blocks probe → instance marked unhealthy)
6. VMSS auto-scale: Instances scaling in under load? (Scale-in during peak → reduced capacity)
7. Application Gateway/Load Balancer: Connection limits exceeded?
8. Application: Thread pool exhausted, database connection limits, etc.
Most common: Health probe failing intermittently (health probe timeout or application not responding within probe timeout).

### Senior L3
**Q: Design a container-based microservices architecture for a payment processing system (high throughput, low latency, audit logging, compliance).**
A:
  Architecture:
    AKS Cluster:
      → Control Plane: Managed (private endpoint, HA across 3 zones)
      → System Node Pool: 3× D4s_v3 (zone-redundant)
      → User Node Pool — Payment Services: 5-20× D8s_v3 (autoscale, high CPU/memory)
      → User Node Pool — Audit/Logging: 3-10× L8s_v2 (storage optimized, zone-redundant)
      → User Node Pool — Dev/Test: 1-3× B2s (burstable, cost-efficient)
      → Virtual Node (ACI): Burst for transaction spikes

    Ingress:
      → Application Gateway v2 (WAF) + AGIC (routes to K8s Ingress)
      → Front Door (global entry, DDoS, SSL)
      → WAF: OWASP managed + custom (block SQL injection in payment data)

    Container Registry:
      → ACR Premium (geo-replicated, vulnerability scanning)
      → Tasks: Auto-build on commit to main branch

    Secrets:
      → Azure Key Vault (CMK, audit logging, Purge Protection)
      → Key Vault CSI driver (mount secrets as volumes in pods)

    Monitoring:
      → Application Insights: Per-service monitoring (payment, audit, notification)
      → Azure Monitor for Containers: Cluster-level monitoring
      → Log Analytics: Centralized logging with retention (90 days hot, 7 years archive)
      → Microsoft Sentinel: Security threat detection, compliance monitoring

    Data Tier:
      → Azure SQL DB (Business Critical, Private Endpoint, Zone-Redundant)
      → Cosmos DB (SQL API, Private Endpoint, multi-region, Strong consistency for payments)
      → Azure Storage (Blob, Private Endpoint, immutable for audit logs)

    Compliance:
      → Private cluster: API server private only
      → Network policies: Frontend ↔ Backend only (not frontend ↔ frontend)
      → PSS: Restricted (no privileged containers, no root, read-only root FS)
      → RBAC: Entra ID groups for k8s access (admin, developer, read-only)
      → Audit logging: All k8s API calls logged to Log Analytics

    Availability:
      → Multi-region: East US 2 + West US 2 (active-active for read, single for write)
      → Zone-redundant: All critical node pools across 3 zones
      → RPO: 5 seconds (Cosmos DB), 5 minutes (SQL DB)
      → RTO: 4 hours (full region failover)

### Expert
**Q: Explain the Kubernetes Pod lifecycle in AKS. What happens when a pod crashes?**
A:
Kubernetes Pod lifecycle in AKS:
  1. Pending: Pod created, waiting for scheduling (insufficient resources, taints)
  2. Container Creating: Images being pulled from ACR
  3. Running: Container(s) executing
  4. Succeeded: All containers completed and exited (run-to-completion jobs)
  5. Failed: One or more containers terminated with non-zero exit code

When a pod crashes:
  → Restart policy: Always (default) → kubelet restarts container automatically
  → Restart backoff: 10s (first), 20s, 40s, ... max 5 minutes (exponential backoff)
  → CrashLoopBackOff: Pod crashes, restarts repeatedly, enters CrashLoopBackOff state
  → kubectl logs: Use --previous flag to see logs from previous (crashed) container
  → Events: kubectl describe pod → Events section (OOMKilled, error, etc.)
  → Liveness probe: If configured, kubelet executes liveness probe; if fails, pod restarted
  → Readiness probe: If fails, pod removed from service endpoints (no traffic, but not restarted)

Self-healing in AKS:
  → ReplicaSet controller: Maintains desired number of pods (restarts crashed ones)
  → If node goes down: Pods rescheduled to other nodes automatically
  → Cluster autoscaler: If insufficient resources, adds nodes to schedule rescheduled pods
  → Horizontal Pod Autoscaler: If average CPU/memory exceeds threshold, increases pod count

### Tricky
**Q: "My AKS cluster has 10 nodes in a user pool with auto-scale min:3 max:20. Pods are running fine, but cluster autoscaler is never adding nodes. Why?"**
A:
Possible causes:
1. Pod resource requests match available capacity (no unschedulable pods → no trigger)
2. Cluster autoscaler looks at unschedulable pods (if all pods CAN be scheduled, it does nothing)
3. HPA already scaled out pods (pods running on current nodes, no need for more nodes)
4. Cluster autoscaler is disabled (verify: AKS → Properties → Autoscaler → Enabled)
5. Node pool: Max = 20, Current = 10 → Room to scale, but no need
6. System pods consuming significant resources (leaving less room for user pods)
7. Node pool taints: Pods can't be scheduled on certain nodes (but tolerations exist for other nodes)
8. Pod PDB (Pod Disruption Budget): Blocking cluster autoscaler from scaling (can't disrupt more pods)
9. Cluster autoscaler is in "balance" mode: Removing nodes rather than adding (utilization is OK)
10. AKS cluster autoscaler syncs with VMSS; VMSS: Scale set is at desired capacity (check: VMSS autoscale)

Most common: Pods are running fine (all scheduled, all healthy) — cluster autoscaler only scales when there are unschedulable pods. If all pods fit on current nodes, autoscaler doesn't add more. It scales IN when nodes are underutilized. Scale OUT requires HPA (pod count) or new pods requiring resources.

### Tricky 2 
**Q: "I have a VM Scale Set in Zone 1 only. A zone failure occurs. What happens?"**
A:
1. All VMSS VMs in Zone 1 are in the same physical datacenter
2. Zone failure = ALL VMs go down simultaneously (single point of failure)
3. Load Balancer health probes fail on all instances → All marked unhealthy
4. No VMs available → Application completely down
5. VMSS auto-scale: Tries to scale out → Cannot create new VMs in Zone 1 (zone down)
6. VMSS in single-zone: No failover to another zone (not configured)
7. Recovery: Must manually recreate VMSS in another zone (or redeploy with zone-redundant)
8. RTO: Hours (manual intervention required for zone change)
9. RPO: Data on OS disks in Zone 1 may be lost (if LRS disks — not replicated across zones)
10. If ZRS disks: Data survives (replicated to other zones) → VM rebuild faster

**Fix: Zone-redundant VMSS across 3 zones:**
```
VMSS: webapp-vmss (Zone-redundant)
  → Instances distributed: 33% in Zone 1, 33% in Zone 2, 33% in Zone 3
  → If Zone 1 fails: 66% of instances still running (Zones 2+3)
  → Load Balancer: Automatically routes traffic to healthy zones
  → Auto-scale: Rebalances across remaining zones
  → ZRS disks: Data survives zone failure
  → RTO: Minutes (VMSS auto-recovers, no manual intervention)
  → RPO: Zero (ZRS disks + application-level replication)
```

### Tricky 3
**Q: "My Application Service Plan is P2V2 with 5 instances. I want to scale to 10 instances. Will it work?"**
A:
No, not on P2V2. P2V2 (Premium v2) has a maximum of 10 instances per SKU. If already at 5, you CAN scale to 10. But if at 10, you CANNOT scale further on P2V2.

Options for > 10 instances:
1. Scale up: Move to P3V2 (higher per-instance compute, still max 10 instances)
2. Multiple App Service Plans: Create 2× P2V2 plans, each with up to 10 instances (use VNet Integration or Front Door for routing)
3. AKS: For 100+ instances (no SKU instance limit)
4. VMSS: For 1,000+ instances (no per-instance SKU limit)
5. Isolated SKU: P1V3/P2V3/I1/I2/I3 (higher limits but still capped per SKU)

Most common mistake: Assuming App Service can scale indefinitely. Each SKU has hard limits.

### Tricky 4
**Q: "Can I use a single AKS cluster with both Linux and Windows node pools?"**
A:
Yes. AKS supports mixed OS node pools:
1. System Node Pool: Linux (required, runs system pods)
2. User Node Pool — Linux: Linux worker nodes for Linux containers
3. User Node Pool — Windows: Windows worker nodes for Windows containers

Key points:
→ All node pools in same AKS cluster share control plane
→ Each node pool = Separate VM Scale Set (Linux VMs / Windows VMs)
→ Pods with Linux images: Scheduled on Linux node pools (selector/tolerations)
→ Pods with Windows images: Scheduled on Windows node pools
→ Not possible: Linux container on Windows node (and vice versa)
→ Cost: Windows VMs more expensive (Windows licensing included)
→ Best practice: Separate Linux and Windows into different node pools for scaling independence

### Tricky 5
**Q: "My ACA container scales to 0 at night. First request in the morning takes 30 seconds. How do I fix this?"**
A:
This is a cold start issue (scale-to-zero + image pull + startup time).

Fix options:
1. Min replicas: Set to 1 instead of 0 (always running, no scale-to-zero)
   → Cost: ~$25-50/month extra for small container
   → Benefit: Zero latency on first request
2. Image optimization: Reduce image size (< 500 MB recommended)
   → Smaller image = faster pull = shorter cold start
3. Pre-warming: Use cron trigger (KEDA) to scale to 1 at 7 AM daily
   → Schedule: Every morning at 7 AM, scale to 1 for 5 minutes
4. Azure Container Instances: For always-on, use ACI instead (no scale-to-zero)
5. AKS: For production, use AKS (no scale-to-zero, always running pods)

Most common: Set min replicas to 1 for latency-sensitive apps; use 0 only for background/event-driven.

### Tricky 6
**Q: "AKS cluster autoscaler added a node but no new pods were scheduled on it. Why?"**
A:
Possible causes:
1. Cluster autoscaler added node to meet pending pod demand, but pod scheduling failed (node taints, node selectors, affinity rules don't match)
2. Pod was scheduled on another node (not the new one) before autoscaler noticed
3. System pods (kube-dns, etc.) consumed most of node capacity → No room for user pods → Autoscaler adds another node
4. Pod resource requests larger than node capacity → Pod can't fit on any node (including new one)
5. Taints on new node: Node has taint, pod doesn't tolerate → Pod skipped
6. PVC/PV issue: Pod needs persistent volume, no available PV matching claim
7. Image pull: Pod image not available (pulling from ACR, slow on cold node)
8. Pod has node selector pointing to specific node labels (not new node)

Most common: System pods consuming significant resources on small nodes → Each new node fills up with system pods → Autoscaler keeps adding nodes. Fix: Use larger node pool SKU (D4s_v3 instead of D2s_v3) so system pods don't consume proportionally as much.

### Tricky 7
**Q: "VMSS instance keeps being deallocated and recreated. What's happening?"**
A:
Possible causes:
1. Health probe failing: Load Balancer health probe marks instance unhealthy → Instance removed → Auto-scale recreates
2. VMSS auto-scale: Instances crossing scale boundaries (scale in → remove, scale out → add)
3. OS image update: Rolling update triggered (image/template changed)
4. VMSS model changed: Configuration update triggered rolling upgrade
5. Unhealthy VM: Azure platform detected hardware/OS issue → Auto-remediation
6. Manual intervention: Someone scaled down then up
7. Orchestration mode: Flexible mode — VM managed independently
8. Platform fault domain: VM in overloaded fault domain → Azure migrates

Troubleshooting:
→ Check: VMSS → Instances → Status (deallocated, running, etc.)
→ Check: Activity Log: What triggered deallocation
→ Check: Health probe: Is it configured correctly?
→ Check: Auto-scale rules: Are instances crossing min/max boundaries?
→ Check: Upgrade policy: Manual or Automatic (triggering upgrades)?

### Tricky 8
**Q: "My App Service is on B1 (Basic) and I need VNet Integration. Can I do it?"**
A:
No. VNet Integration (regional) requires Premium v2 (P1V2+), Premium v3 (P1V3+), or Isolated (I1+) SKU. B1 (Basic) does NOT support VNet Integration.

Options for B1:
1. Upgrade to P1V2 or higher (enables VNet Integration)
2. Use Hybrid Connection (connects to on-prem without VNet Integration)
3. Use VNet Integration via Private Link service (experimental, limited)
4. Use API for Management to access VNet resources (limited functionality)

Most common: Developers deploy on B1 (cheapest), then realize they need VNet Integration → Must upgrade to P1V2+ → Cost increase (~$13/month → ~$150/month).

### Tricky 9
**Q: "I deployed a container to ACI, but it says 'Image not found.' The image is in ACR. What's wrong?"**
A:
Common causes:
1. Wrong image name format: Must be acrname.azurecr.io/image:tag (fully qualified)
   → Wrong: myapp:v1.0
   → Correct: acrcontoso.azurecr.io/myapp:v1.0
2. ACR authentication: Anonymous pull not allowed (ACR is private)
   → Fix: Use managed identity, service principal, or admin user for authentication
3. ACR in different subscription/tenant: Access not granted (cross-tenant ACR)
   → Fix: Grant pull access (ACR access level: Subscription, Resource Group, or ACR)
4. Image not pushed to ACR: Image exists locally but not in ACR
   → Fix: docker push acrcontoso.azurecr.io/myapp:v1.0
5. Wrong tag: Image pushed as v1.0 but deployed with v2.0
   → Fix: Verify tag in ACR: acrcontoso.azurecr.io/myapp:v1.0 exists
6. ACR firewall: ACR has firewall rules blocking ACI pull
   → Fix: Allow ACI or disable firewall (use MI auth instead)
7. ACR geo-replication: ACR in different region → Image not replicated
   → Fix: Enable geo-replication or push image to correct region's ACR

### Tricky 10
**Q: "AKS cluster is running fine, but I can't connect to API server (kubectl get nodes fails). What's happening?"**
A:
Possible causes:
1. Public API server endpoint disabled (private cluster)
   → If private cluster: Must connect from authorized VNet/VPN/ER
   → Fix: Connect from within VNet or via VPN/ExpressRoute
2. Entra ID expired: Cluster authentication expired
   → Fix: Re-authenticate: az aks get-credentials --refresh
3. RBAC: User doesn't have RBAC permission for cluster
   → Fix: az aks command-invoker add-rbac --user <username>
4. Network: VPN/ER to cluster VNet broken
   → Fix: Check VPN/ER status, routes, NSGs
5. API server down: Managed control plane issue (rare)
   → Fix: Contact Microsoft support, check Service Health
6. Cluster deleted: Cluster resource was deleted
   → Fix: Check cluster existence: az aks list
7. Wrong cluster context: kubectl using old context
   → Fix: az aks get-credentials --overwrite-existing

Most common (for production): Private cluster enabled but user trying to connect from local machine without VPN → Connection refused.

### Senior L3
**Q: "Design a compute architecture for a real-time gaming platform (500K concurrent users, <50ms latency, global, auto-scaling, anti-cheat)."**
A:
  Architecture:
    Global Entry:
      → Azure Front Door (global entry, DDoS Protection Standard, rate limiting, bot management)
      → Front Door Routes: Latency-based routing to nearest region

    Regions (East US 2, West Europe, Southeast Asia, South America):
      → Virtual WAN Hub (per region, Scale Units: 6)
      → Firewall: Standard HA (per region)
      → Bastion: Zone-redundant

      Spoke-Game:
        → AKS Cluster (Private Cluster):
           System Node Pool: 3× D8s_v5 (zone-redundant) — System pods
           User Node Pool — Game Server: 10-100× NC24as_T4_v3 (GPU, spot + on-demand)
              → Game server pods (Unity/Unreal dedicated server containers)
              → HPA: CPU > 70% → Scale out GPU nodes
              → Anti-cheat: Sidecar container in each pod (behavior analysis)
           User Node Pool — Matchmaking: 5-20× D8s_v5 (CPU/memory optimized)
              → Matchmaking service (ELO, latency-based)
              → HPA: HTTP queue > 100 → Scale out
           User Node Pool — Relay: 5-20× D4s_v5 (network optimized)
              → Relay/proxy service (game traffic relay between players)
           Virtual Node (ACI): Burst for tournament spikes

        → Application Gateway (WAF, zone-redundant):
           → WAF: OWASP 3.2 + custom anti-cheat rules
           → Rate limiting: 100 requests/sec per IP (prevent DDoS/cheat)
           → AGIC → Routes to AKS Ingress resources

        → Redis Cache (Premium, zone-redundant):
           → Game state, player sessions, leaderboards
           → Replication: 1 primary + 2 replicas (per zone)

        → Azure SQL DB (Business Critical, zone-redundant):
           → Player data, inventory, transactions
           → Active Geo-Replication: All regions (read-only replicas)

        → Cosmos DB (Private Endpoint):
           → Player profiles, game config, real-time events
           → Multi-region writes, Session consistency

        → Azure Media Services:
           → Game streaming, video recording (replays)
           → CDN integration (Front Door CDN)

        → Private Endpoints: All PaaS services (no public access)

        → Anti-Cheat Architecture:
           → Sidecar container in game server pod
           → Memory scanning: Detect injected code (cheat DLLs)
           → Network monitoring: Detect unusual traffic patterns
           → Behavioral analysis: Detect impossible actions (speed hacks)
           → Reporting: Suspicious activity → Log Analytics → Sentinel alert
           → Action: Auto-kick player, flag account, notify moderators

        → Monitoring:
           → Application Insights: Per-service monitoring
           → Azure Monitor: AKS, VM, network metrics
           → Log Analytics: All logs, 90-day retention
           → Custom metrics: Game latency, player count, FPS
           → Alerts: Latency > 50ms, player disconnect spike, cheat detected

        → Cost Optimization:
           → GPU spot instances: 70% cost savings (interruptible = acceptable for gaming)
           → VMSS autoscale: Scale down overnight (50% fewer players)
           → Reserved instances: Predictable baseline capacity
           → CDN: Cache static assets (reduce Origin load)

  Latency Optimization:
    → Front Door: Nearest region routing
    → Redis: In-memory (sub-ms response)
    → Game servers: Co-located in same datacenter as players
    → Network: Premium SSD (low latency), Accelerated Networking (SR-IOV)
    → Relay: Direct peer-to-peer via AKS relay (not through backend)

  Anti-Cheat:
    → Server-side validation: All game logic on server (client can't be trusted)
    → Random checks: 10% of players validated per game session
    → ML-based: Anomaly detection on player behavior (Microsoft Sentinel)
    → Real-time: Suspicious activity alerts → Automated response (kick + ban)

---

## 12. SCENARIO-BASED QUESTIONS

### Scenario 1: "VM running production database has 100% CPU for 2 hours, application is slow"
**Architecture:** VM → Managed Disk (Premium SSD) → Azure SQL (separate VM).
**Dependencies:** VM size, disk type, application, OS, NSG.
**Checks:**
1. VM CPU: 100% sustained (Azure Monitor: VM CPU %)
2. VM processes: What process consuming CPU (top/htop on Linux, Task Manager on Windows)
3. Disk IOPS: Disk near limit (Azure Monitor: Disk IOPS)
4. SQL queries: Expensive queries running (Query Performance Insight)
5. OS: Memory pressure (swap usage increasing)
6. Network: Incoming traffic spike (DDoS? legitimate traffic increase?)
7. VM size: Currently D4s_v3 → Upgrade to D8s_v3?
8. Auto-scale: VM in VMSS? Check VMSS scaling rules
**Root Cause: SQL query with no index causing CPU-intensive table scans; or application bug causing infinite loop.**
**Fix: Short-term: Scale up VM (D4s_v3 → D8s_v3). Long-term: Fix query (add index), restart process, kill runaway process. Monitor: CPU returns to normal.**
**Validation: Azure Monitor: CPU < 70%. Application: Response time < 2 sec. Database: No expensive queries running.**

### Scenario 2: "App Service on P1V2 auto-scaled to 10 instances, but response time still slow"
**Architecture:** Front Door → App Gateway → App Service (P1V2, 10 instances).
**Dependencies:** App Service, App Gateway, database, Front Door, NSG.
**Checks:**
1. App Service instances: 10 running (max reached for P1V2 — can't scale further)
2. Response time: Per-instance vs total (P1V2 instances might be underpowered)
3. Database: Connection pool exhausted? (App Service has 10 instances = 10× connection pool)
4. CPU/Memory: Per-instance metrics (if each instance > 80% → need scale-up not scale-out)
5. Front Door: CDN cache hit ratio (low cache hit = all traffic to backend)
6. Application Gateway: Backend health (all instances healthy)
7. Code: Memory leak (response time degrades over time since deployment)
8. Scaling: CPU < 70% but still slow → Not CPU bottleneck → Application issue (database, I/O, blocking threads)
**Root Cause: DB connection string uses single connection pool; 10 instances × pool size = connection exhaustion. Or: P1V2 instance size too small (need scale-up to P2V2).**
**Fix: Increase DB connection limit; Use connection pooling (SqlConnection pooling); Scale up to P2V2 (more compute per instance).**
**Validation: App Service: Response time < 2 sec. Database: Active connections < limit. App Service: CPU < 70%.**

### Scenario 3: "AKS pod OOMKilled repeatedly, CrashLoopBackOff"
**Architecture:** AKS → User Node Pool → Pod (Container) → Database.
**Dependencies:** Pod resource limits, memory, application, AKS node size.
**Checks:**
1. kubectl describe pod: Events section (OOMKilled)
2. kubectl top pod: Memory usage (exceeding limits?)
3. Pod resource limits: Memory limit (too low for workload?)
4. Application: Memory leak (gradual increase until limit hit)
5. Node capacity: Insufficient memory on node (HPA scaling not triggered?)
6. Database: Connection pool causing memory growth per pod
7. Container image: Large image consuming memory for runtime
**Root Cause: Pod memory limit set too low (256 MB) for actual workload (512 MB needed). Application processes all requests in memory without streaming.**
**Fix: Increase pod memory limit (256MB → 1GB); Implement streaming (don't load all data into memory); Add HPA rule (memory > 80% → scale out); Restart pod after fix.**
**Validation: kubectl top pod: Memory < 80% of limit. No OOMKilled events. Pod: Running (CrashLoopBackOff resolved).**

### Scenario 4: "VMSS behind Load Balancer — users report intermittent 502 errors"
**Architecture:** Internet → Load Balancer → VMSS (5-20 instances).
**Dependencies:** Load Balancer health probe, VMSS instances, NSG, application.
**Checks:**
1. Backend health: All instances Healthy? (Load Balancer → Backend health)
2. Health probe: Correct port, path, interval? (e.g., HTTP:80 /health every 5 sec)
3. NSG: Health probe port allowed? (NSG must allow probe traffic on health probe port)
4. Application: Responding to health probe on all instances? (App must have /health endpoint)
5. VMSS instances: All running? (Auto-scaling causing churn?)
6. NSG on VMSS subnet: Load Balancer source IP allowed? (AzureLoadBalancer service tag)
7. Backend VM: Application responding within probe timeout (default 5 sec)?
**Root Cause: Health probe failing on 1-2 instances (intermittent — application slow to respond under load). Load Balancer marks instances unhealthy → Traffic redistributed → Users hit remaining instances (overloaded → 502).**
**Fix: Increase health probe timeout (5 → 10 sec); Increase probe interval (5 → 10 sec); Add /health endpoint to application; Ensure NSG allows AzureLoadBalancer traffic on health probe port.**
**Validation: All instances Healthy; No 502 errors; Application monitoring: Response time < 2 sec.**

### Scenario 5: "New AKS deployment — pods stuck in Pending for 15 minutes"
**Architecture:** AKS Cluster → System Node Pool (2 nodes) → User Node Pool (0 nodes, scale: 0-10).
**Dependencies:** AKS node pools, auto-scaler, quotas, image availability.
**Checks:**
1. kubectl get nodes: No user nodes (only system nodes)
2. kubectl describe pod: Events → Insufficient CPU/Memory?
3. User node pool: Min: 0, Max: 10, Current: 0 (not scaled yet)
4. Cluster autoscaler: Enabled? (AKS → Properties → Autoscaler)
5. vCPU quota: Sufficient quota? (Subscription → Quotas)
6. Image pull: Image available in ACR? (Authentication correct?)
7. Taints: User node pool has taints? (Pod doesn't tolerate)
8. System pods: Consuming resources? (kube-dns, metrics-server on 2 nodes)
**Root Cause: Cluster autoscaler not enabled (default for new AKS). Pods can't schedule without user nodes. Scale from 0 → 0 nodes means no capacity.**
**Fix: Enable cluster autoscaler on user node pool (min: 2, max: 10); Wait for autoscaler to add nodes; Or manually scale node pool to 2.**
**Validation: kubectl get nodes: User nodes (2+ running). Pod: Running (not Pending). Metrics: CPU/Memory normal.**

### Scenario 6: "ACA container deployed, traffic spike from 0 to 1000 requests/sec — users getting errors"
**Architecture:** ACA → Ingress → Container (KEDA scale rules) → Backend.
**Dependencies:** ACA scale rules, max replicas, cold start, image size.
**Checks:**
1. ACA replicas: How many replicas running? (Was 0, now scaling)
2. Scale rule: HTTP-based? Max replicas: 20? (If max = 20, can't handle 1000 RPS at 50 RPS/replica)
3. Cold start: First 30 sec of traffic spike → All requests queued (scale from 0)
4. Min replicas: 0 → Scale to zero = initial delay (cold start)
5. HPA: CPU/memory based? Might be too slow to respond
6. Backend: Container responding within acceptable time?
7. Application Gateway in front: Rate limiting? (If App Gateway WAF throttling → 429 errors)
**Root Cause: Scale from 0 (cold start) + max replicas (20) insufficient for sudden 1000 RPS spike. Cold start: 30 sec delay; Scaling: 20 replicas handle 1000 RPS but need time to spin up.**
**Fix: Min replicas: 5 (pre-warmed); Max replicas: 50 (handle spike); Scale rule: Target 20 RPS per replica (not 50); Use: Predictive scaling (if traffic pattern known).**
**Validation: ACA: Replicas scaling up within 30 sec; Error rate < 1%; Response time < 2 sec; All replicas healthy.**

### Scenario 7: "ACI container in VNet can't reach Azure SQL DB (Private Endpoint) but VM in same VNet can"
**Architecture:** ACI (VNet) → Azure SQL DB (Private Endpoint, same VNet).
**Dependencies:** ACI networking, DNS, NSG, Private Endpoint, VNet configuration.
**Checks:**
1. ACI VNet Integration: Is ACI in same VNet as SQL Private Endpoint? (Yes/No)
2. Private DNS Zone: Linked to VNet? (Same VNet as ACI and SQL)
3. ACI DNS resolution: Does ACI resolve sql-server.database.windows.net → Private IP?
4. ACI IP: Is it in VNet subnet? (Private IP within subnet range)
5. NSG: ACI subnet allows traffic to SQL subnet? (NSG rules: Allow 1433 to SQL subnet)
6. SQL Server: Firewall: Allow Azure services enabled? OR Private Endpoint only?
7. SQL DB: Private Endpoint: Provisioned and Approved?
8. ACI: Running as Internal (no public IP)? (Must use Private Endpoint to resolve private DNS)
**Root Cause: ACI VNet Integration not configured (ACI has public IP only, not in VNet). ACI DNS resolves to public IP → SQL denies connection (Deny public access enabled).**
**Fix: Enable ACI VNet Integration (assign ACI to VNet and subnet); Ensure Private DNS Zone linked to VNet; Verify DNS: ACI resolves SQL to private IP.**
**Validation: ACI → nslookup/sqlcmd: Resolves private IP; ACI → SQL: Connection successful; NSG: Traffic allowed.**

### Scenario 8: "VM Scale Set scaled out to 10 instances, but only 5 are receiving traffic"
**Architecture:** Load Balancer → VMSS (10 instances) → Application.
**Dependencies:** Load Balancer, VMSS, NSG, health probe, application.
**Checks:**
1. VMSS: 10 instances running (Azure Portal → VMSS → Instances)
2. Load Balancer: 10 instances in backend pool? (BL → Backend Pool → Members)
3. Load Balancer health probe: All 10 instances Healthy? (Some might be Unhealthy)
4. NSG: VMSS subnet allows Load Balancer traffic (AzureLoadBalancer tag)
5. Application: All instances responding correctly?
6. Load Balancing Rule: Backend port correct? (If wrong port → 5 instances fail)
7. VMSS NIC: All instances have correct IP configuration?
8. Application Gateway (if used): Backend pool has 10 instances?
**Root Cause: Health probe failing on 5 instances → Load Balancer marks them Unhealthy → Traffic only to 5 Healthy instances. Probe failing because application slow to respond under load.**
**Fix: Increase health probe timeout and interval; Add more resources to application (scale up VM size); Fix application performance issue; Add more instances.**
**Validation: Load Balancer: 10/10 Healthy; All instances receiving traffic; Application: Response time normal.**

### Scenario 9: "AKS cluster upgrades failed — nodes stuck 'Upgrading' for 3 hours"
**Architecture:** AKS Cluster → Node Pools → System + User nodes.
**Dependencies:** AKS version, node image, OS upgrade, capacity.
**Checks:**
1. AKS upgrade history: Current upgrade in progress? (Check: AKS → Upgrade)
2. Node status: kubectl get nodes — some 'NotReady'?
3. Events: kubectl get events — upgrade related errors?
4. VMSS: Instance view — upgrade failures?
5. Image: OS image not available for target version? (AKS version incompatible with node image)
6. Capacity: Insufficient vCPU quota to add new nodes (old nodes being drained, new nodes can't be provisioned)?
7. PDB: Pod disruption budget blocking rolling upgrade (too strict)?
8. Node pool: OS disk type not supported for target AKS version?
9. Network: Node can't download new image (Firewall blocking Microsoft download URLs)?
**Root Cause: vCPU quota exceeded. Old nodes being drained → New nodes need provisioned → Insufficient vCPU quota → Upgrade stuck.**
**Fix: Request vCPU quota increase (Azure Support); Pause upgrade; Wait for quota increase; Resume upgrade; Or: Scale down cluster before upgrade (reduce vCPU usage).**
**Validation: AKS: All nodes upgraded; Version: New version running; Pods: All running and healthy.**

### Scenario 10: "ACA container deployment fails — image pull error — despite working locally"
**Architecture:** ACA Environment → Container App → Image (ACR) → Container Registry.
**Dependencies:** ACR, ACA, authentication, image tag, network.
**Checks:**
1. Image name: Correct format? (acr.azurecr.io/image:tag — fully qualified)
2. ACR authentication: Configured for ACA? (ACR pull role or anonymous access)
3. Image tag: Correct tag? (Pushed to ACR as v1.0 but deploying v1.1?)
4. Image push: Image actually pushed to ACR? (docker push acr.../image:v1.0)
5. ACR: Exists in same region as ACA? (Cross-region pulls might be slow)
6. ACR firewall: Blocking ACA pull? (Allow ACA service or use MI auth)
7. ACA: Environment linked to ACR? (Not required but check)
8. Network: ACA can reach ACR (no VNet isolation blocking)?
9. Image: Compatible with ACA runtime (ACA supports Linux containers only)?
**Root Cause: Image pushed to ACR with tag 'latest' but ACA deploying tag 'v1.0' (which doesn't exist in ACR). Locally: docker run myapp:v1.0 (local image exists). ACA: acr.azurecr.io/myapp:v1.0 (doesn't exist in ACR).**
**Fix: Push image to ACR with correct tag (docker push acr.azurecr.io/myapp:v1.0); Verify: acr.azurecr.io/myapp:v1.0 exists in ACR; Redeploy ACA with correct tag.**
**Validation: ACA: Container running; No image pull errors in logs; Image: Correct version deployed.**

### Scenario 11: "VM in VMSS keeps going to 'Deallocated' state after 5 minutes of running"
**Architecture:** VMSS → VM instances (identical VMs) → Load Balancer.
**Dependencies:** VMSS auto-scale, auto-shutdown, OS, extensions, application.
**Checks:**
1. VMSS: Auto-shutdown configured? (Task Scheduler: Scheduled shutdown at specific time)
2. Auto-scale: VMSS scaling in? (Scale-in policy: Remove instance after idle time)
3. OS: Scheduled task (Linux: crontab, Windows: Task Scheduler)
4. Extensions: Custom Script Extension causing shutdown?
5. Application: Application initiates shutdown (clean shutdown command)
6. Platform: Azure platform deallocating (maintenance, error)?
7. VMSS model: Different from existing instances (rolling update triggered)
8. Health: VM unhealthy → Auto-remediation (Azure detects VM unhealthy → Deallocates)
9. Activity Log: What caused deallocation? (Check Activity Log for VMSS instance)
**Root Cause: Auto-shutdown scheduled task (e.g., Linux: crontab "0 10 * * * shutdown -h now" — daily at 10 AM). Or: Auto-scale rule: Scale in from 5 to 3 instances → 2 instances removed.**
**Fix: Remove scheduled shutdown task; Check Auto-scale rules: Scale-in policy not too aggressive; Set min instances to 2 (prevents over-scaling); Investigate: Activity Log for each deallocation event.**
**Validation: VMSS: All instances running (no Deallocated); Activity Log: No deallocation events; Auto-scale: Min instances = 2 (prevents over-scaling).**

### Scenario 12: "Application Gateway with AKS backend — Backend health Unhealthy for 5 minutes, then becomes Healthy"
**Architecture:** Front Door → Application Gateway (WAF) → AKS (Ingress → Pods).
**Dependencies:** App Gateway, AGIC, AKS, pods, health probes, NSG.
**Checks:**
1. Health probe: App Gateway probes Kubernetes service endpoint (not pod directly)
2. Kubernetes service: ClusterIP changes when pods scale → AGIC updates App Gateway
3. AGIC: Watcher — detects Service/Ingress changes → Updates App Gateway backend
4. Time lag: Pod comes up → Kubernetes service updated → AGIC detects → App Gateway updated (60-90 sec delay)
5. During lag: App Gateway probes old endpoint → Unhealthy → Then probes new endpoint → Healthy
6. Pod health: Application responding before pod marked Ready (kubernetes readiness probe)
7. Scaling: HPA scaled up pods (new pods not ready yet)
8. AGIC config: Probe path matching K8s ingress path
**Root Cause: App Gateway health probe running before Kubernetes service and AGIC updated backend pool after scaling event. Time lag between pod creation and App Gateway backend update (60-90 seconds).**
**Fix: Increase health probe interval (30 sec → 60 sec) to reduce flapping; Implement: Pod readiness probe (only mark Ready after app responds); AGIC: Configure faster sync (aggressive sync settings); Use: Connection drain (graceful shutdown of old pods).**
**Validation: App Gateway: All backends Healthy; AKS: Pods Ready; AGIC: Sync events logged; No 502 errors during scaling.**

---

## 13. DIAGNOSTIC SETTINGS — COMPUTE

| Resource | Log Category | Destination |
|----------|-------------|------------|
| **VM** | Boot Diagnostics, Guest OS logs, Metrics, Custom logs | Log Analytics + Storage + Azure Monitor |
| **VMSS** | Metrics, Scaling events, Instance view, OS logs | Log Analytics + Azure Monitor |
| **App Service** | Request, Application, Web Server, HTTP Access, Deployment, Debug Logs | Log Analytics + Storage |
| **App Service** | Autoscale Events, Health Check Failures, Always On Status | Log Analytics + Alerts |
| **AKS Control Plane** | API Server Audit Logs, Authentication Logs, Scheduler Logs, Controller Manager Logs | Log Analytics |
| **AKS Nodes** | Kubelet Logs, Kube-Proxy Logs, Container Logs, System Logs | Log Analytics + Azure Monitor |
| **AKS Pods** | Container Logs, Container Metrics, Container Events, Application Logs | Log Analytics + Application Insights |
| **AKS Events** | Kubernetes Events (Pod scheduled, failed, pulling image, etc.) | Log Analytics |
| **AKS Add-ons** | CNI Logs, DNS Logs, Ingress Controller Logs, AGIC Logs | Log Analytics |
| **ACA** | Container Logs, Scale Events, Job Logs, Execution Logs, Environment Logs | Log Analytics + Azure Monitor |
| **ACI** | Container Logs, Container Events, IP Allocation Logs | Log Analytics + Storage |
| **ACR** | Authentication Logs, Pull/Push Logs, Task Run Logs, Replication Logs | Log Analytics + Storage |
| **Application Gateway** | Access Logs, WAF Logs, Performance Logs, SSL Logs, Health Probe Logs | Log Analytics + Storage |
| **Front Door** | Access Logs, WAF Logs, DDoS Logs, Cache Logs, Health Probe Logs | Log Analytics + Storage |
| **DDoS** | Mitigation Metrics, Attack Logs, Alert Logs | Log Analytics + Azure Monitor |
| **Load Balancer** | Flow Logs, Health Probe Status, Backend Pool Metrics | Log Analytics + Storage |
| **VPN Gateway** | IKE Logs, Connection Logs, P2S Logs, Route Logs, Diagnostics | Log Analytics + Storage |
| **ExpressRoute** | Circuit Logs, BGP Peer Logs, ARP Logs, MAC Logs | Log Analytics + Storage |
| **Virtual WAN** | Connection Monitor, Route Change Logs, Attachment Logs, Hub Metrics | Log Analytics + Azure Monitor |
| **Bastion** | Session Logs, Connection Logs, Error Logs, Activity Logs | Log Analytics + Storage |
| **Azure Backup** | Backup Job Logs, Restore Logs, Alert Logs, Recovery Logs | Log Analytics |
| **Activity Log** | All configuration changes (deploy, modify, delete, autoscale, upgrade) | Log Analytics + Storage |
| **Alerts (All)** | Threshold breach, scaling events, health failures, configuration changes | Azure Monitor → Action Groups |
| **Cost Management** | Resource-level cost, tag-level cost, service-level cost | Cost Management + Alerts |

---

## 14. L3 GAP CHECK

| Topic Area | Sub-Topics Covered | Status |
|-----------|-------------------|--------|
| **VM Deep Dive** | VM components, sizes (B/D/E/L/NC/HB/HC), authentication, extensions, disk types, caching, IOPS, throughput, OS disk, data disk | ✅ |
| **VM Disk/Storage Performance** | Ultra Disk, Premium SSD v1/v2, Standard SSD, HDD, disk caching (None/ReadOnly/ReadWrite), IOPS limits, throughput, VM disk bottlenecks, striping, IOPS calculation, VM size disk limits | ✅ |
| **VM Availability** | None, Availability Set (fault/upgrade domains), Availability Zone (ZRS/adaptive), VMSS, HA patterns, single vs multi-zone | ✅ |
| **VM Scale Sets** | Architecture, instance limits, scaling mechanisms (manual/auto/schedule/predictive), rolling upgrades, surge, unified vs flexible mode, zone-redundant | ✅ |
| **Azure App Service** | Plan SKUs, auto-scale, deployment slots, VNet Integration, Easy Auth, managed identity, IP restrictions, private endpoint, outbound IPs, limitations | ✅ |
| **Containers — ACI** | Architecture, networking (public/internal), limitations, use cases, authentication | ✅ |
| **Containers — ACA** | Architecture, scale rules (KEDA), scale to zero, event-driven scaling, jobs, ingress, Dapr, comparison with AKS/ACI | ✅ |
| **Containers — AKS** | Architecture, control plane, node pools, CNI/Kubenet, HPA/VPA/CAVS, AGIC, security (RBAC/PSS/Policy), secrets (KV CSI), private cluster, Defender, monitoring | ✅ |
| **Containers — ACR** | SKUs, geo-replication, build/tasks, authentication (MI/SP/admin), pull access | ✅ |
| **Design Patterns** | Web app (App Service), microservices (AKS), container migration pattern | ✅ |
| **Production Example** | Global e-commerce (10M+ users, multi-region, PCI compliance) | ✅ |
| **Failure Scenarios** | 12 comprehensive scenarios (VM CPU, App Service 503, VMSS scale, AKS pending, CrashLoop, AGIC 502, ACI crash, ACA scale, VNet Integration, boot diagnostics, extension failure, etc.) | ✅ |
| **Monitoring** | Comprehensive metrics table (VM, VMSS, App Service, AKS, ACA, ACI, ACR, etc.) | ✅ |
| **Diagnostic Settings** | All compute resources — log categories and destinations | ✅ |
| **Security Considerations** | 25+ security concerns with risk/mitigation (public IP, unmanaged disks, caching, single VM, WAF, secrets, PSS, private cluster, etc.) | ✅ |
| **L3 Interview Questions** | 40+ questions (Basic, Intermediate, L3, Senior L3, Expert, Tricky, Tricky 2-10) | ✅ |
| **Scenario-Based Questions** | 12 comprehensive scenarios with architecture, dependencies, checks, root cause, fix, validation | ✅ |

---

### Gap Check Summary

```
Module 1 L3 Gap Check Result:

  Total Topic Areas Checked: 17
  Fully Covered at L3 Depth: 17 / 17 (100%)
  Partially Covered:           0
  Missing:                     0

  ✅ NO GAPS IDENTIFIED — Module 1 is fully covered at L3 depth.
```

---

### Visual Summary

```
Module 1: Azure Compute — L3 Gap Check
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 VM Deep Dive .............................. ✅
 VM Disk/Storage Performance ............... ✅
 VM Availability ........................... ✅
 VM Scale Sets ............................. ✅
 Azure App Services ....................... ✅
 Containers — ACI .......................... ✅
 Containers — ACA .......................... ✅
 Containers — AKS .......................... ✅
 Containers — ACR .......................... ✅
 Design Patterns ........................... ✅
 Production Example ........................ ✅
 Failure Scenarios (12 scenarios) .......... ✅
 Monitoring (all compute metrics) .......... ✅
 Diagnostic Settings ....................... ✅
 Security Considerations ................. ✅
 L3 Interview Questions (40+) .............. ✅
 Scenario-Based Questions (12) ............. ✅

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 STATUS: ALL 17 TOPIC AREAS COVERED AT L3 DEPTH
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```