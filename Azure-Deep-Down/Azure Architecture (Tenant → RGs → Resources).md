# # COMPLETE AZURE L3 MASTER ROADMAP

**Date context: September 2026 — Current Azure platform focused. Legacy/deprecated services explicitly flagged.**

---

## ROADMAP — ALL 79 PARTS ORGANIZED AS MODULES

| Module | Covers | Priority |
|--------|--------|----------|
| **MODULE 1** | Azure Architecture (Tenant → RGs → Resources) | 🔴 Critical |
| **MODULE 2** | Azure Resource Manager (ARM) | 🔴 Critical |
| **MODULE 3** | Microsoft Entra ID — Tenant, Users, Groups, Apps | 🔴 Critical |
| **MODULE 4** | Entra ID Authentication & Tokens | 🔴 Critical |
| **MODULE 5** | Conditional Access — L3 Depth | 🔴 Critical |
| **MODULE 6** | Entra Connect / Hybrid Identity | 🔴 Critical |
| **MODULE 7** | RBAC — L3 Depth | 🔴 Critical |
| **MODULE 8** | Governance — Azure Policy | 🔴 Critical |
| **MODULE 9** | Landing Zones | 🟡 High |
| **MODULE 10** | Azure Networking — VNet Deep | 🔴 Critical |
| **MODULE 11** | Network Security (NSG/Firewall/WAF) | 🔴 Critical |
| **MODULE 12** | Routing — L3 Depth | 🔴 Critical |
| **MODULE 13** | Azure DNS | 🟡 High |
| **MODULE 14** | Private Endpoint / Private Link | 🔴 Critical |
| **MODULE 15** | VPN Gateway | 🔴 Critical |
| **MODULE 16** | ExpressRoute | 🔴 Critical |
| **MODULE 17** | Virtual WAN | 🟡 High |
| **MODULE 18** | Azure Virtual Network Manager | 🔵 Awareness |
| **MODULE 19** | Azure Compute — VMs Deep | 🔴 Critical |
| **MODULE 20** | VM Disk / Storage Performance | 🔴 Critical |
| **MODULE 21** | VM Availability | 🔴 Critical |
| **MODULE 22** | VM Scale Sets | 🟡 High |
| **MODULE 23** | Azure App Services | 🟡 High |
| **MODULE 24** | Containers (ACI/ACA/AKS/ACR) | 🔵 Awareness |
| **MODULE 25** | Azure Storage — Accounts Deep | 🔴 Critical |
| **MODULE 26** | Blob Storage | 🔴 Critical |
| **MODULE 27** | Azure Files | 🟡 High |
| **MODULE 28** | SAS / Access / Security | 🟡 High |
| **MODULE 29** | Storage Data Protection | 🔴 Critical |
| **MODULE 30** | Managed Identities — Deep | 🔴 Critical |
| **MODULE 31** | Azure Key Vault | 🔴 Critical |
| **MODULE 32** | Azure Monitor — Deep | 🔴 Critical |
| **MODULE 33** | Log Analytics | 🟡 High |
| **MODULE 34** | Alerting | 🟡 High |
| **MODULE 35** | Azure Update Management | 🟡 High |
| **MODULE 36** | Azure Backup | 🔴 Critical |
| **MODULE 37** | Azure Site Recovery | 🔴 Critical |
| **MODULE 38** | Azure Migrate | 🟡 High |
| **MODULE 39** | Hybrid Identity / Infrastructure | 🔴 Critical |
| **MODULE 40** | Azure Arc | 🔴 Critical |
| **MODULE 41** | Azure Security / Defender for Cloud | 🔴 Critical |
| **MODULE 42** | Azure Firewall / WAF | 🔴 Critical |
| **MODULE 43** | Load Balancing | 🔴 Critical |
| **MODULE 44** | Front Door / Traffic Manager | 🟡 High |
| **MODULE 45** | Cost Management / Advisor | 🟡 High |
| **MODULE 46** | Availability / Resiliency | 🔴 Critical |
| **MODULE 47** | Quotas / Limits / Capacity | 🔴 Critical |
| **MODULE 48** | Deployment / IaC | 🟡 High |
| **MODULE 49** | VM Extensions | 🔴 Critical |
| **MODULE 50** | VM Boot / Guest Troubleshooting | 🔴 Critical |
| **MODULE 51** | Network Watcher | 🔴 Critical |
| **MODULE 52** | Resource Locks | 🟡 High |
| **MODULE 53** | Diagnostic Settings | 🔴 Critical |
| **MODULE 54** | Service Health / Resource Health | 🔴 Critical |
| **MODULE 55** | Production Troubleshooting Framework | 🔴 Critical |
| **MODULE 56** | Network Troubleshooting Scenarios | 🔴 Critical |
| **MODULE 57** | Identity Troubleshooting Scenarios | 🔴 Critical |
| **MODULE 58** | Compute Troubleshooting Scenarios | 🔴 Critical |
| **MODULE 59** | Storage Troubleshooting Scenarios | 🔴 Critical |
| **MODULE 60** | Monitoring Troubleshooting | 🟡 High |
| **MODULE 61** | Backup / DR Failure Scenarios | 🔴 Critical |
| **MODULE 62** | Migration Scenarios | 🟡 High |
| **MODULE 63** | Hybrid VMware + Azure | 🔴 Critical |
| **MODULE 64** | Azure VMware Solution Awareness | 🔵 Awareness |
| **MODULE 65** | Azure Security Incidents | 🔴 Critical |
| **MODULE 66** | Control Plane vs Data Plane | 🔴 Critical |
| **MODULE 67** | Azure Service Dependency Matrix | 🔴 Critical |
| **MODULE 68** | Interview Questions (All Modules) | 🔴 Critical |
| **MODULE 69** | "What Happens Internally?" Questions | 🔴 Critical |
| **MODULE 70** | L1/L2/L3/Architect Differentiation | 🟡 High |
| **MODULE 71** | Real Enterprise Operations | 🟡 High |
| **MODULE 72** | Production Realism Check | 🔵 Meta |
| **MODULE 73** | Azure L3 Gap Analysis | 🔵 Meta |

---

# MODULE 1 — AZURE ARCHITECTURE

## 1.1 CONCEPT

Azure is a **multi-tenant cloud platform** organized in a strict hierarchical containment model. Every resource you create exists inside a nested chain of scoping containers. Understanding this hierarchy is the foundation of every architectural, administrative, troubleshooting, and governance decision you will make as an L3 engineer.

The hierarchy is not just organizational — it is the **enforcement boundary** for:
- **Authentication** (Microsoft Entra ID tenant)
- **Authorization** (Azure RBAC scope)
- **Policy** (assignment scope)
- **Networking** (VNet boundaries, peering, connectivity)
- **Billing** (subscription/management group)
- **Availability** (region, zone, pair)
- **Resource identity** (resource ID)

If you do not understand what container something lives in, you cannot troubleshoot permissions, networking, policy, or billing. This is the single most important architectural concept in Azure.

---

## 1.2 ARCHITECTURE

```
Tenant (Microsoft Entra)
 └── Management Groups (optional, multi-subscribe governance)
      └── Subscriptions
           └── Resource Groups
                └── Resources (VMs, Storage, VNets, etc.)
```

**The chain is strictly one-directional downward.** A resource can only access things inside its own containment chain or things explicitly exposed to it (peering, trust relationships, etc.).

---

## 1.3 COMPONENTS — DETAILED

### 1.3.1 TENANT (Microsoft Entra Tenant)

**What it is:**
A Microsoft Entra tenant (previously Azure Active Directory tenant) is a **trusted identity boundary**. It represents an organization's instance of Microsoft Entra ID. Every Azure subscription belongs to exactly one Entra tenant.

**Why it exists:**
- It is the authentication authority. Every sign-in, token issuance, and authorization decision starts here.
- It defines the directory of users, groups, applications, and service principals.
- It enforces tenant-wide policies like Conditional Access, authentication methods, and domain verification.

**Key properties:**
| Property | Description |
|----------|-------------|
| Tenant ID | Globally unique GUID that identifies the tenant in Microsoft Entra ID |
| Default domain | `tenantname.onmicrosoft.com` — created automatically |
| Custom domains | Verified DNS domains added for user sign-in (e.g., `contoso.com`) |
| Directory roles | Global Admin, Privileged Role Admin, Security Admin, etc. |

**What it depends on:**
- Microsoft's global authentication infrastructure
- DNS (for custom domain verification)
- No other Azure resource — it is the root identity anchor

**What depends on it:**
- Every single Azure resource (for identity and RBAC)
- Every single authentication event
- Every single API call (tokens are validated against the tenant)

**L3 operational impact:**
- If the tenant is compromised, everything is compromised.
- Tenant-level settings (authentication methods, Conditional Access, tenant restrictions) affect every workload.
- Cross-tenant scenarios (B2B guest access, multi-tenant apps) require explicit trust configuration.

---

### 1.3.2 MANAGEMENT GROUPS

**What it is:**
A Management Group is a **governance container** above subscriptions. It provides a level of scope for applying Azure policies and RBAC across multiple subscriptions that share common requirements.

**Why it exists:**
- Enterprise environments have hundreds or thousands of subscriptions.
- You need to apply governance (Policy, RBAC) at a level higher than subscription but below tenant.
- It enables hierarchical governance: apply once at Management Group, inherited by all child subscriptions.

**Key properties:**
- Can contain other Management Groups (nested) and Subscriptions
- Has a unique identifier
- RBAC and Policy assignments at this scope propagate downward

**Dependencies:**
- Requires Azure subscription(s) as children
- Requires RBAC assignments (e.g., Owner at Management Group level to manage it)

**What depends on it:**
- All subscriptions and resources within its hierarchy inherit governance
- Budgets, policy compliance, and cost reporting can roll up

**L3 operational impact:**
- A Policy assignment at the Management Group level is the most efficient way to enforce standards across hundreds of subscriptions.
- A misconfigured deny assignment at this level can block deployments across the entire organization.
- Understanding the Management Group tree is essential before troubleshooting "why can't this subscription deploy X?"

---

### 1.3.3 SUBSCRIPTION

**What it is:**
A subscription is an **isolation boundary** and **billing unit** within an Azure tenant. It maps to a trust contract with Microsoft.

**Why it exists:**
- Isolates resources, spending, and access control.
- Provides a deployment scope for ARM.
- Maps to a specific agreement (Enterprise Agreement, Pay-As-You-Go, etc.).

**Key properties:**
- Subscription ID (GUID)
- Subscription name
- Tenant association (always one tenant)
- Management Group parent(s)
- Spending limit (for Pay-As-You-Go)

**What it depends on:**
- The Entra tenant
- Azure Resource Manager
- Location/region availability

**What depends on it:**
- Resource Groups and Resources inside it
- RBAC assignments at subscription scope
- Policy assignments at subscription scope
- Cost tracking

**L3 operational impact:**
- Moving a resource between subscriptions requires careful planning (depends, identity, networking).
- Quota limits are subscription-regional specific.
- A subscription hit its spending limit → new resources cannot be created.

---

### 1.3.4 RESOURCE GROUP

**What it is:**
A Resource Group is a **logical container** for related resources within a subscription. It is the most granular scope for many administrative operations.

**Key properties:**
- Name (unique within the subscription)
- Location (region — this matters for many operations)
- Subscription association
- Tags (can be inherited)
- Locks (can be applied here)

**What it depends on:**
- Subscription
- ARM

**What depends on it:**
- All resources inside it
- Deployments targeting this RG
- Locks, RBAC assignments at RG scope
- Resource movement (resources can move between RGs in the same subscription/region)

**L3 operational impact:**
- Resources in different RGs cannot peer VNets unless done explicitly (VNet peer configuration is per-VNet, not per-RG, but the RGs of the peering VNets matter for cross-RG peering).
- Deleting a Resource Group deletes all resources inside it (unless locked).
- Resource Group location ≠ resource locations. The RG has a "location" property but resources inside can be in different regions.

**Common misconception:**
Many engineers think a Resource Group's location determines where resources are deployed. It does not. The resource's own location property determines its region. The RG location is primarily a metadata/organizational property and determines where certain management operations occur.

---

### 1.3.5 AZURE RESOURCES

**What it is:**
An Azure resource is any deployable, manageable item in Azure: VM, Storage Account, VNet, NIC, Disk, Key Vault, etc.

**Key properties:**
- **Resource ID** (globally unique): `/subscriptions/{sub}/resourceGroups/{rg}/providers/{provider}/{type}/{name}`
- **Resource name** (unique within its resource group and type)
- **Resource type** (e.g., `Microsoft.Compute/virtualMachines`)
- **Resource provider** (e.g., `Microsoft.Compute`)
- **Location** (region where the resource physically resides)
- **Kind** (additional metadata for some types)
- **Tags** (key-value pairs)
- **Identity** (system-assigned or user-assigned managed identity)
- **Dependencies** (references to other resources, usually via `dependsOn` in templates)

**Resource ID — Deep Dive:**

Every resource has a unique immutable ID:
```
/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myVM
```

This ID is used for:
- RBAC role assignments
- Policy targeting
- ARM template references
- API calls
- Logging and monitoring
- Cross-subscription references

---

### 1.3.6 RESOURCE PROVIDER

**What it is:**
A Resource Provider is a service that publishes resource types into Azure. Each Azure service has one or more resource providers.

**Examples:**
| Service | Resource Provider |
|---------|-------------------|
| Virtual Machines | `Microsoft.Compute` |
| Storage Accounts | `Microsoft.Storage` |
| Virtual Networks | `Microsoft.Network` |
| Key Vaults | `Microsoft.KeyVault` |
| App Services | `Microsoft.Web` |
| VMs (availability sets) | `Microsoft.Compute` |
| DNS | `Microsoft.Network` (for Azure DNS) or `Microsoft.Sql` (for SQL DB) |

**Why it matters:**
- Before creating resources from a provider, it typically needs to be **registered** in the subscription: `Register-AzResourceProvider -ProviderNamespace Microsoft.Compute`
- If a provider is not registered, deployment of resources from that provider may fail.
- Resource providers handle the actual CRUD operations for their resource types.

**L3 troubleshooting relevance:**
When a deployment fails with "Resource type not found" or "Provider not registered," this is the first thing to check. ARM can't talk to a provider that isn't registered.

**Current status (Sept 2026):**
Most providers auto-register in modern Azure subscriptions. Legacy/manual registration is now rare but still relevant when working with management groups or subscriptions where auto-registration was disabled.

---

### 1.3.7 AZURE RESOURCE MANAGER (ARM)

**What it is:**
Azure Resource Manager is the **management layer** that sits between the Azure portal/CLI/API and the underlying services. It is the control plane for all Azure resources.

**What it does:**
1. **Authentication** — Validates the identity making the request
2. **Authorization** — Checks RBAC permissions via Azure RBAC
3. **Policy evaluation** — Evaluates Azure Policy before allowing the operation
4. **Deployment orchestration** — Creates/updates/deletes resources in dependency order
5. **Routing** — Directs requests to the correct resource provider
6. **Auditing** — Records all operations in Activity Log

**Control Plane vs Data Plane (critical distinction):**

| Plane | Responsible For | Examples |
|-------|-----------------|----------|
| **Control Plane** | Managing resources (create, read, update, delete, configure) | ARM, Portal, CLI, API calls — "Create a VM," "Change NSG rule," "Modify storage tier" |
| **Data Plane** | Using the resource for its intended purpose | VM boot and OS execution, Storage blob read/write, Key Vault secret retrieval, Network traffic flow |

**Real example:**
```
Control Plane: You call ARM to create a VM → ARM checks RBAC → checks Policy → tells Microsoft.Compute to create the VM → returns success
Data Plane: Your application connects to a blob in Storage → Storage service handles the read/write → data flows
```

**Why this matters for troubleshooting:**
- "I cannot create a VM" → Control Plane issue (RBAC, Policy, Provider, Quota)
- "The VM is running but my app cannot access the storage account" → Data Plane issue (NSG, DNS, Private Endpoint, SAS, Network path)

If you confuse these planes, you waste hours troubleshooting the wrong layer.

---

### 1.3.8 REGIONS

**What it is:**
A region is a geographic area where Azure operates data centers. Each region has one or more data centers.

**Key concepts:**
| Concept | Description |
|---------|-------------|
| Region | Geographical area (e.g., `eastus`, `westeurope`, `southeastasia`) |
| Availability Zone | Physically separate data centers within a region (typically 3) |
| Region pair | Two regions within 300 miles, designed for DR (e.g., eastus ↔ eastus2) |
| Geo | Broader geographic area (continent-level, e.g., US, Europe) |

**Region properties:**
- Each region has a unique name (used in URLs, resource locations)
- Not all services are available in all regions
- Regions have quota limits (vCPU, IP addresses, etc.)
- Some regions are "paired" for replication purposes

**L3 operational impact:**
- Choosing the wrong region affects latency, compliance, availability, pricing, and service availability.
- Resources in different regions require explicit connectivity (VNet Peering Global, VPN, ExpressRoute, Private Link).
- Regional outages affect all resources in that region — check Service Health first.

---

### 1.3.9 AVAILABILITY ZONES

**What it is:**
Availability Zones are physically separate data centers within a single Azure region. Each zone has independent power, cooling, and networking.

**Why they exist:**
- Protect against data center-level failures (fire, power failure, network outage in one building).
- Provide zone-level redundancy for compute, storage, and networking.

**Key points:**
- Not all regions have 3 zones; some have 1, 2, or more.
- Not all services support zone redundancy.
- You can specify a zone (1, 2, or 3) when creating a resource, or choose "zone-redundant" redundancy.
- Zone IDs are 1, 2, 3 within a region, but the physical mapping can vary.

**L3 troubleshooting:**
- A VM in Zone 1 and a VM in Zone 2 are in different physical buildings. If Zone 1 has an outage, Zone 2 VMs continue running.
- A VM deployed to a specific zone cannot "fail over" automatically to another zone — that requires load balancing, availability sets, or VMSS with zone balancing.

---

### 1.3.10 REGION PAIRS

**What it is:**
Every Azure region is paired with another region in the same geography (typically within 300 miles / 600 km).

**Why they exist:**
- For disaster recovery: replicate data across the pair.
- For planned maintenance: Azure can update one region without affecting the pair.
- For compliance: data stays within a geographic boundary.

**Examples:**
- `(eastus, eastus2)`
- `(westeurope, northeurope)`
- `(southeastasia, eastasia)`

**Key behavior:**
- Data replication (GRS, Azure Backup, Site Recovery) typically targets the paired region.
- If a region goes down, the paired region is the failover target.

---

### 1.3.11 GLOBAL VS REGIONAL SERVICES

**What it means:**
- **Global services** exist outside any region — they are accessible from anywhere and have no region-specific deployment. Examples: Entra ID, Azure Active Directory (global endpoints), Azure Policy (management plane), Cost Management (global).
- **Regional services** must be deployed in a region. Examples: VMs, Storage Accounts, VNets, NSGs.

**Some services are both:**
- The **control plane** may be global (ARM, API endpoints) but the **data plane** is regional (the actual VM, storage data).

**L3 impact:**
- When you see an error referencing a global service, it affects your entire tenant/subscription hierarchy.
- Regional errors usually affect only the specific region.

---

### 1.3.12 RESOURCE LOCATIONS VS METADATA LOCATIONS

**Resource Location:**
- The physical region where the resource's data and compute reside.
- This is the `location` property on the resource.
- Determines latency, compliance boundary, cost, and availability.

**Metadata Location:**
- Where the resource's metadata (ARM representation, tags, properties) is managed.
- For most resources, this is the same as the resource location.
- Some resources (like certain global services) have metadata managed globally.

**Key distinction:**
A VM's location is where it runs. The ARM API endpoint that manages it might be global (`management.azure.com`), but the VM itself is in `eastus`.

---

### 1.3.13 TAGS

**What they are:**
Key-value pairs attached to resources for organization and tracking.

**Key behaviors:**
- Tags can be inherited from Resource Group to Resources (if the resource doesn't override).
- Tags do NOT affect resource behavior — they are purely organizational.
- Tags propagate to cost analysis, budget reports, and Advisor recommendations.
- You can filter resources by tag across subscriptions and management groups.
- **Important:** Tagging a Resource Group does NOT automatically tag all existing resources — it only applies to new resources created in that RG (for inherited tags). Existing resources need manual re-tagging or automation.

**L3 operational impact:**
- Poor tagging = poor cost visibility = no chargeback/showback = inability to identify orphaned resources.
- Some governance policies (Required Tags effect in Azure Policy) can enforce tagging.

---

### 1.3.14 LOCKS

**What they are:**
Azure Resource Locks prevent accidental deletion or modification of critical resources.

| Lock | Effect |
|------|--------|
| `CanNotDelete` | Resource cannot be deleted (but can be modified) |
| `ReadOnly` | Resource cannot be modified or deleted |

**Inheritance:**
- Locks propagate downward in the hierarchy.
- A lock at the Resource Group level locks all resources in it.
- A lock at the Subscription level locks everything.
- More specific locks can be more restrictive, but a more permissive lock at a higher level cannot override a restrictive lock at a lower level.

**L3 troubleshooting:**
- "I cannot delete this storage account" → Check for locks at the storage account, resource group, and subscription levels.
- Locks on Resource Groups can prevent RG deletion — very common issue during cleanup.

---

### 1.3.15 QUOTAS AND LIMITS

**What they are:**
Service-level caps on resource creation within a subscription/region.

**Categories:**
- **vCPU quotas** — Maximum vCPUs per region per subscription
- **IP address limits** — Public IPs per subscription
- **Storage limits** — Storage accounts per subscription, capacity per account
- **Network limits** — VNets, subnets, NSGs, load balancers
- **VM family quotas** — Specific to certain VM series (e.g., DSv2 quota)
- **API rate limits** — ARM API throttling (typically 1,200 requests per subscription per minute for write operations)

**Why they exist:**
- Capacity management in data centers
- Prevent billing surprises
- Ensure fair usage

**How to check:**
Portal → Subscription → Usage + quotas. Or Azure CLI: `az vm list-usage --location eastus`

**How to increase:**
Support request via Azure Portal (typically 1-2 business days for approval).

**L3 troubleshooting relevance:**
One of the most common "mystery failures" — deployment looks correct but fails with quota-related errors.

---

### 1.3.16 ARM DEPLOYMENT MODEL

**What it is:**
All resource creation, update, and deletion in Azure goes through ARM. Even resources created via Portal or CLI are processed by ARM.

**Deployment scopes (from smallest to largest):**
| Scope | Level | Used For |
|-------|-------|----------|
| **Resource** | Single resource | Targeting a specific resource for operations |
| **Resource Group** | Within one RG | Most common deployment target |
| **Management Group** | Multiple subscriptions | Enterprise-wide deployments |
| **Subscription** | Within one subscription | Subscription-scoped deployments |
| **Tenant** | Entire tenant | Tenant-wide policies, global configurations |

**Deployment modes (ARM templates):**
| Mode | Behavior | Use Case |
|------|----------|----------|
| **Incremental** (default) | Creates new resources, updates existing, leaves untouched | Normal deployments |
| **Complete** | Creates/updates, and deletes resources that exist but are not in the template | Full infrastructure replacement |
| **What-If** | Shows what changes would occur without actually deploying | Pre-deployment validation |

---

## 1.4 COMPONENTS — INTERCONNECTION MAP

```
Tenant (Entra ID)
 │
 ├── Management Group(s) [governance scope]
 │    ├── Subscription A
 │    │    ├── Resource Group 1
 │    │    │    ├── Resource: VM (Microsoft.Compute)
 │    │    │    ├── Resource: VNet (Microsoft.Network)
 │    │    │    └── Resource: Storage (Microsoft.Storage)
 │    │    └── Resource Group 2
 │    │         └── Resource: Key Vault (Microsoft.KeyVault)
 │    └── Subscription B
 │         └── Resource Group 3
 │              └── Resource: App Service (Microsoft.Web)
 │
 └── Entra ID Resources (global)
      ├── Users, Groups, Apps
      ├── Conditional Access Policies
      └── Authentication Methods
```

**Connection rules:**
1. **Authentication** always goes to the Entra Tenant (regardless of which subscription/resource)
2. **Authorization** (RBAC) is evaluated at the scope where the resource lives
3. **Policy** is evaluated at the assignment scope (and inherited downward)
4. **Networking** is determined by the resource's location and configuration, NOT by the Management Group/Subscription
5. **Billing** rolls up from Resource → Resource Group → Subscription → Management Group → Tenant

---

## 1.5 DEPENDENCIES

### What Tenant depends on:
- Microsoft global infrastructure
- DNS (for custom domains)
- Entra Connect/Cloud Sync (for hybrid identity, but tenant works standalone)

### What Management Group depends on:
- At least one Subscription as child
- Entra Tenant
- ARM

### What Subscription depends on:
- Entra Tenant (for identity)
- ARM (for deployment/management)
- Region availability (for resource deployment)
- Resource Providers (registered)

### What Resource Group depends on:
- Subscription
- ARM
- Region (its own location property)

### What Resources depend on:
- Resource Group (containment)
- Subscription (billing)
- Location (region for physical presence)
- Entra ID (for identity/auth)
- Networking (VNet, NIC for VMs)
- Other resources (dependencies)
- Resource Providers (registration)

---

## 1.6 COMMUNICATION FLOW

### When a user creates a VM via Portal:
```
User (Authenticated via Entra ID)
 → Portal sends request to ARM (management.azure.com)
 → ARM authenticates token against Entra Tenant
 → ARM checks RBAC: "Does user have Microsoft.Compute/virtualMachines/write at this scope?"
 → ARM evaluates Azure Policy: "Is this deployment allowed?"
 → ARM checks quotas: "Are there enough vCPUs available?"
 → ARM resolves dependencies, validates template/Bicep
 → ARM sends request to Microsoft.Compute Resource Provider
 → Provider creates VM in the specified region
 → ARM records operation in Activity Log
 → ARM returns success to user
```

### When a VM accesses a Storage Account:
```
VM (in VNet, with Managed Identity)
 → Requests token from Entra ID (for MSI)
 → Token presented to Storage API
 → Storage checks:
    1. Token validity (Entra ID validation)
    2. RBAC permissions on Storage Account
    3. Network rules (firewall/private endpoint/VNet)
    4. DNS resolution (Azure-provided or custom)
 → If all checks pass → Data flows from VM to Storage
```

---

## 1.7 INTERNAL WORKFLOW — WHAT HAPPENS WHEN A VM IS CREATED

Step-by-step internal flow:

1. **User action** → Portal/CLI/API sends VM creation request to ARM
2. **Authentication** → ARM validates the user's token against Entra ID tenant
3. **Authorization** → ARM checks: Does the user have `Microsoft.Compute/virtualMachines/write` at the target Resource Group scope (or higher)?
4. **Policy evaluation** → ARM evaluates all applicable Azure Policies:
   - Is the VM size allowed? (Allowed SKUs policy)
   - Are tags required? (Required Tags policy)
   - Is the region allowed? (Allowed Locations policy)
5. **Quota check** → ARM checks subscription regional quota for the VM size
6. **Resource Provider** → ARM contacts `Microsoft.Compute` provider (must be registered)
7. **Dependency resolution** → ARM resolves `dependsOn` (if any):
   - VNet must exist
   - Subnet must exist
   - NIC may need to exist first
8. **Resource creation** → Microsoft.Compute creates:
   - VM object in Azure DB
   - Associated OS disk (if not pre-created)
   - NIC (if not pre-created)
   - Public IP (if requested)
9. **Platform orchestration** → Azure fabric controller:
   - Picks a host within the region/zone
   - Allocates compute resources
   - Attaches disks
   - Configures networking
10. **VM boots** → Hypervisor starts the VM, OS boots
11. **ARM returns** → VM resource ID, properties, status
12. **Activity Log** → Entry recorded

---

## 1.8 ADMINISTRATION

Key administrative operations and their scope:

| Operation | Scope | Tool |
|-----------|-------|------|
| Create/manage subscriptions | Tenant/Management Group | Portal, CLI, API |
| Create/manage Management Groups | Tenant | Portal, CLI |
| Create/manage Resource Groups | Subscription | Portal, CLI, ARM/Bicep |
| Assign RBAC | Any scope | Portal, CLI, ARM |
| Apply Policy | Any scope | Portal, CLI, ARM |
| Apply Locks | Any scope | Portal, CLI |
| Tag resources | Resource/RG/Subscription | Portal, CLI |
| Check quotas | Subscription/Region | Portal, CLI |
| Move resources | Same subscription (same region for most) | Portal, CLI, ARM |
| Deploy resources | Any scope | ARM templates, Bicep, Portal |

---

## 1.9 SECURITY CONSIDERATIONS

| Concern | Mitigation |
|---------|------------|
| Tenant compromise | Privileged Identity Management (PIM), Conditional Access, MFA on Global Admins |
| Subscription hijacking | RBAC (least privilege), MFA, audit logs |
| Resource Group exposure | RBAC at RG scope, locks, diagnostic settings |
| Resource misconfiguration | Azure Policy, Defender for Cloud |
| Over-provisioned permissions | Regular access reviews, PIM, Azure AD Access Reviews |
| Management plane exposure | Activity Log monitoring, alerts on risky operations |

---

## 1.10 MONITORING

**What to monitor at the Architecture level:**
- **Activity Log** — All control plane operations across subscription/management group
- **Resource Health** — Platform problems affecting specific resources
- **Service Health** — Broader Azure platform issues
- **Diagnostic Settings** — Resource-level logs and metrics
- **Cost Analysis** — Spending per RG, subscription, tag, resource type

**Key alerts to configure:**
- Cost threshold alerts per subscription/RG/tag
- Activity Log alerts for sensitive operations (role assignments, firewall changes, NSG changes, diagnostic settings changes)
- Resource Health alerts
- Quota threshold alerts (90% warning)

---

## 1.11 PRODUCTION EXAMPLE

**Scenario: An enterprise has 3 departments (Engineering, Finance, HR), each with their own subscription, under a Management Group "Corp-Prod."**

```
Management Group: Corp-Prod
├── Subscription: Corp-Prod-Eng (500 vCPUs quota)
│   ├── RG: rg-eng-appsvc (Application workloads)
│   │   ├── VM: web-server-01 (East US, Standard_D2s_v5)
│   │   ├── VNet: vnet-eng-app (10.0.0.0/16)
│   │   └── NSG: nsg-eng-app
│   └── RG: rg-eng-data (Data workloads)
│       ├── Storage: stengdata (LRS, Standard_LRS)
│       └── SQL: sqldb-eng (East US)
├── Subscription: Corp-Prod-Fin (200 vCPUs quota)
│   └── RG: rg-fin-apps
│       └── VM: fin-app-01
└── Subscription: Corp-Prod-HR (100 vCPUs quota)
    └── RG: rg-hr-apps
```

**Engineering creates a VM in rg-eng-data:**
- ARM authenticates user via Entra ID
- ARM checks RBAC at rg-eng-data scope (does user have write access?)
- ARM checks Policy (is D2s_v5 in Allowed SKUs? Are required tags present?)
- ARM checks vCPU quota (does Engineering have capacity?)
- ARM creates VM in East US region

**Finance engineer cannot access Engineering's storage:**
- RBAC: Finance user has no role on rg-eng-data
- Result: Access denied — correct behavior

**Quota issue:**
- Engineering tries to create 100 D2s_v5 VMs but only has 500 vCPU quota
- Each D2s_v5 = 2 vCPUs → 100 VMs = 200 vCPUs → still under quota
- But if they try 300 VMs = 600 vCPUs → exceeds quota → deployment fails
- Fix: Request quota increase via Azure Support

---

## 1.12 FAILURE SCENARIOS

| Scenario | Root Cause | Resolution |
|----------|------------|------------|
| Deployment fails, "Subscription not registered" | Resource provider not registered | Register via Portal or CLI |
| VM creation fails, "Not enough cores" | vCPU quota exceeded | Request quota increase |
| User cannot create resource, "Authorization failed" | Missing RBAC role assignment | Assign appropriate role |
| Deployment fails with "Policy violation" | Azure Policy denies operation | Check policy assignments, request exemption if valid |
| Resource Group deletion fails | Resource lock on RG or resource | Remove locks, then retry |
| VM created but shows "VM deallocated" | Subtier/sku issue, or platform issue | Check Resource Health, VM serial logs |
| Cannot access Portal, "Tenant not found" | Wrong tenant, or tenant access issue | Verify tenant ID, check Conditional Access |

---

## 1.13 TROUBLESHOOTING METHODOLOGY — ARCHITECTURE ISSUES

When something fails at the Architecture level, follow this approach:

### Step 1: Identify the failure scope
- Which resource? Which subscription? Which RG? Which region?
- Is it affecting one resource or many?

### Step 2: Check the containment chain
- What subscription/RG/Management Group is the resource in?
- Are there locks, policies, or RBAC at each level?

### Step 3: Check ARM (Control Plane)
- Activity Log: What does it say about the failed operation?
- Check error message: Is it an Authorization, Policy, Quota, or Provider error?

### Step 4: Check Identity
- Is the user/service principal authenticated to the correct tenant?
- Does the identity have the required role at the correct scope?

### Step 5: Check Policy
- Are there deny assignments blocking the operation?
- Is the resource configuration compliant?

### Step 6: Check Quotas
- Is there a regional/compute/storage quota issue?

### Step 7: Check Resource Provider
- Is the relevant provider registered?

---

## 1.14 LOGS / EVIDENCE

| Evidence Source | What It Shows | How to Access |
|----------------|---------------|---------------|
| Activity Log | All control plane operations (create, delete, modify, read) with status, actor, timestamp | Portal → Subscription/Resource Group → Activity Log |
| ARM Deployment Logs | Detailed deployment operations (success/failure per resource) | Portal → Deployment Operations, or ARM API |
| Resource-specific logs | Service-specific operational data | Diagnostic Settings → Log Analytics/Storage/Event Hub |
| Error messages | ARM error codes and messages | Portal/CLI output, ARM API response |

---

## 1.15 VERSION/CURRENT SERVICE CONSIDERATIONS (Sept 2026)

| Aspect | Current State |
|--------|---------------|
| **Azure AD → Entra ID** | Fully rebranded. "Azure Active Directory" portal experience redirects to Entra. All references in ARM/APIs use Microsoft Entra. Legacy "Azure AD" terminology still common in scripts/docs. |
| **Management Groups** | Fully GA, widely adopted. Multi-tenant governance standard. |
| **Resource Manager** | Unified deployment model. Classic deployment model is fully deprecated (removed). |
| **Resource Providers** | Auto-registration is now standard. Legacy manual registration still supported via API. |
| **Regions** | New regions regularly added. Always check current list for specific service availability. |
| **Availability Zones** | Expanding. Most mainline regions have 3 zones now. |
| **Tags** | No major changes. Still critical for governance. |
| **Locks** | No major changes. Still essential for production. |
| **Quotas** | Dynamic quota systems. Some VM series use capacity reservations. Always verify current quota model. |
| **ARM Templates** | Still fully supported and GA. |
| **Bicep** | GA and preferred by Microsoft for new IaC. Compiles to ARM JSON. |
| **Terraform** | Widely used, provider maintained by HashiCorp/Microsoft. State management is key. |

---

## 1.16 L3 INTERVIEW QUESTIONS

### Basic
**Q: What is the hierarchy of Azure resources?**
A: Tenant → Management Groups → Subscriptions → Resource Groups → Resources. Each level is a containment and governance boundary.

### Intermediate
**Q: What is the difference between a Resource Group's location and a resource's location?**
A: The Resource Group's location is metadata/organizational; it determines where management operations and certain default behaviors occur. The resource's location is where the actual resource data and compute physically reside. A Resource Group in "eastus" can contain a Storage Account in "westus" (for most resource types).

### L3
**Q: A user can view a resource but cannot delete it. What are all the possible causes?**
A:
1. **RBAC:** User lacks `Microsoft.Compute/virtualMachines/delete` (or equivalent) at the resource/RG/subscription scope.
2. **Lock:** A `CanNotDelete` lock exists on the resource, RG, or subscription.
3. **Policy:** An Azure Policy may deny delete operations.
4. **Resource provider:** Not applicable for deletion itself, but if provider is unhealthy, deletion may fail.
5. **Resource dependencies:** Other resources may depend on it.
6. **System-initiated:** Azure may lock resources during maintenance.

### Senior L3
**Q: How would you design a multi-department Azure environment with isolated access, consistent governance, and shared services?**
A: Use Management Groups for governance inheritance, separate subscriptions per department for isolation and billing, shared services subscription for centralized resources (Key Vault, DNS, Firewall), Resource Groups per workload type, RBAC with least privilege per subscription, Azure Policy for standardization, and shared services accessed via Private Endpoints.

### Expert
**Q: What happens internally when you create a VM through the Azure portal?**
A: See Section 1.7 above. Brief: Authentication → Authorization → Policy → Quota → Resource Provider → Resource creation → Platform orchestration → Boot → Return → Activity Log.

### Scenario
**Q: "A junior engineer creates a VM, tags the Resource Group, but the VM shows no tags. Why?"**
A: Tag inheritance from Resource Group to Resources only applies to **new resources** created after the tag was applied on the RG. Existing resources are not retroactively tagged. The VM was either created before the RG tag, or the RG tag was applied and the VM was created later but with explicit override. Check the VM's tag inheritance settings.

### Tricky
**Q: "I have Owner role on a subscription. Why can't I delete a resource?"**
A: Two common reasons: (1) A `CanNotDelete` lock exists at the resource, RG, or subscription level — locks override RBAC, even Owner cannot bypass them. (2) Azure Policy is set to Deny on delete operations. Many experienced engineers forget that locks and policies can override Owner permissions.

---

## 1.17 SCENARIO-BASED QUESTIONS

### Scenario 1: "VM deployed successfully but is unreachable"
**Architecture:** VM exists in VNet, has NIC, NSG, IP.
**Dependencies:** NSG rules, route table, public IP, DNS.
**Checks:** NSG rules, effective security rules (Network Watcher), effective routes, ping/RDP from outside.
**Root Cause (common):** NSG denies inbound, no public IP, or no route to internet.
**Fix:** Add NSG allow rule, assign public IP, verify route.
**Validation:** Can reach VM via RDP/SSH.

### Scenario 2: "Subscription has 150 resources but no one knows who owns them"
**Architecture:** Resources spread across RGs, no tagging.
**Dependencies:** Tag policy, RBAC for visibility.
**Checks:** Resource inventory by tag, resource groups, RBAC assignments.
**Root Cause:** No tagging standards enforced, no ownership model.
**Fix:** Enforce Required Tags policy, implement tag ownership convention.
**Validation:** All resources have Owner tag, cost analysis per owner.

### Scenario 3: "Management Group policy is blocking deployments in a specific subscription"
**Architecture:** Policy assigned at Management Group propagates to all child subscriptions.
**Dependencies:** Policy assignment, exemption settings.
**Checks:** Policy compliance, exemptions, policy assignment scope.
**Root Cause:** Policy is inherited from parent scope; either the policy is correct (and subscription must comply) or an exemption is needed.
**Fix:** Add exemption if justified, or adjust resource to comply.
**Validation:** Deployment succeeds, compliance shows "Compliant."

---

## 1.18 KNOWLEDGE TEST

1. **What is the difference between a Management Group and a Subscription?**
   - Management Group is a governance container above subscriptions; Subscription is a billing/isolation boundary. Management Groups can contain other Management Groups; Subscriptions cannot.

2. **Can a Resource Group contain resources from multiple regions?**
   - Yes. The RG location is metadata; individual resources can be in different regions (with some exceptions).

3. **What is the difference between the Control Plane and Data Plane?**
   - Control Plane = management operations (ARM, create/delete/configure). Data Plane = using the resource (traffic, data operations).

4. **What happens if a Resource Provider is not registered?**
   - Resources of that type cannot be created/managed in that subscription until the provider is registered.

5. **Can a lock at the Resource Group level prevent a resource at the Subscription level from being deleted?**
   - No. Locks do not propagate upward. But if the resource is inside the RG, the lock applies.

6. **Where does tagging happen?**
   - At the Resource level. RG-level tags are inherited by new resources created in that RG (unless overridden), but not retroactively.

7. **What scope does RBAC operate on?**
   - Management Group, Subscription, Resource Group, and Resource scopes. Inheritance flows downward.

8. **If a user is assigned "Reader" at the Management Group scope, what can they see?**
   - Read access to all subscriptions, resource groups, and resources within that Management Group hierarchy.

---

## 1.19 L3 GAP CHECK

| Topic | Status |
|-------|--------|
| Tenant / Entra ID hierarchy | ✅ Covered |
| Management Groups | ✅ Covered |
| Subscription | ✅ Covered |
| Resource Group | ✅ Covered |
| Resources & Resource IDs | ✅ Covered |
| Resource Providers | ✅ Covered |
| ARM (Control Plane) | ✅ Covered |
| Control Plane vs Data Plane | ✅ Covered |
| Regions, Zones, Pairs | ✅ Covered |
| Tags | ✅ Covered |
| Locks | ✅ Covered |
| Quotas/Limits | ✅ Covered |
| ARM Deployment Scopes | ✅ Covered |
| Deployment modes | ✅ Covered |
| RG location vs Resource location | ✅ Covered |
| Hierarchy interconnection | ✅ Covered |

---

## 1.20 WHAT AN EXPERIENCED AZURE L3 ENGINEER SHOULD NOW BE ABLE TO EXPLAIN CONFIDENTLY

After Module 1, you should be able to confidently explain:

1. **The complete Azure containment hierarchy** — Tenant → MG → Subscription → RG → Resource — and why each level exists as a governance, security, and billing boundary.

2. **The difference between Control Plane and Data Plane** — and why "I can't create a VM" is fundamentally different from "My app can't read from Storage."

3. **How ARM orchestrates every operation** — authentication, authorization, policy evaluation, deployment, and logging.

4. **Why a Resource Group's location is NOT where resources run** — and why this misconception causes confusion.

5. **How locks, policies, and RBAC interact** — and why having Owner role doesn't always mean you can do everything.

6. **How resource providers work** — and why unregistered providers block deployment.

7. **How availability zones, regions, and region pairs relate** — for disaster recovery and high availability design.

8. **How to read a resource ID** and understand what it tells you about containment, provider, and type.

9. **How quotas can silently block deployments** — and how to investigate and resolve them.

10. **Why tagging is not cosmetic** — it's fundamental to cost management, governance, and operational visibility.

---

# Ready for Module 2 — Azure Resource Manager (ARM) Deep Dive?

Just say **"Next module"** and I'll deliver **MODULE 2 — AZURE RESOURCE MANAGER** with the same depth, covering:
- ARM internals
- Resource providers deep dive
- Resource types and IDs
- ARM templates and Bicep
- Deployment scopes and modes
- Dependency handling
- Failed deployments and partial deployments
- Locks and RBAC interaction
- Policy interaction
- And all 19 other required module components

Let me know when you're ready to continue, or if you'd like to go deeper on any specific topic from Module 1 first.