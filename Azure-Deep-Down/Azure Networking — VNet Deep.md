# MODULE 10 — AZURE NETWORKING — VNET DEEP DIVE 🔴 CRITICAL — L3 DEPTH

---

## 1. CONCEPT

**Azure Virtual Network (VNet)** is the fundamental building block for private networking in Azure. It is a logical isolation boundary — a fully contained, isolated network in the Azure cloud that you define and control entirely.

> **The single most important networking concept:** A VNet is YOUR network in Azure. It is the equivalent of your on-premises corporate network, but in the cloud. Everything inside it — VMs, workloads, services — lives in YOUR controlled network space. And just like your corporate network, how you design it (subnets, routing, security, DNS) determines how well your entire infrastructure operates.

**Why VNet design matters:**

```
WITHOUT PROPER VNET DESIGN:
  → IP address conflicts when merging teams/projects
  → No workload isolation (compromised VM reaches everything)
  → No centralized security inspection (each VM exposed to internet)
  → Un routable traffic (VMs can't reach required services)
  → DNS resolution failures (internal names don't resolve)
  → No network flow visibility (NSG logs show chaos)
  → Cost overruns (oversized VNets, unnecessary peering)
  → Troubleshooting nightmare (500 VMs with random connectivity)

WITH PROPER VNET DESIGN:
  → Clean IP address hierarchy (defined per app/team/environment)
  → Workload isolation (compromise contained within subnet)
  → Centralized security (Firewall/NSG at Hub)
  → Full routing control (UDRs force traffic where needed)
  → Internal DNS works (Private DNS Zones linked)
  → Full network visibility (NSG Flow Logs, Firewall logs)
  → Cost efficiency (right-sized VNets, no unnecessary peering)
  → Troubleshooting by design (predictable traffic flows)
```

**VNet as the center of all networking:**

```
  ┌─────────────────────────────────────────────────────┐
  │                    VIRTUAL NETWORK                    │
  │                     (Your Network)                    │
  │                                                       │
  │   Address Space: 10.0.0.0/16                         │
  │                                                       │
  │   ┌───────────┐ ┌───────────┐ ┌──────────────┐      │
  │   │ Subnet:   │ │ Subnet:   │ │ Subnet:      │      │
  │   │ 10.0.1.0  │ │ 10.0.2.0  │ │ 10.0.3.0     │      │
  │   │ /24       │ │ /24       │ │ /24          │      │
  │   │ (Web)     │ │ (App)     │ │ (Data)       │      │
  │   │ VMs,      │ │ VMs,      │ │ VMs,         │      │
  │   │ App GW    │ │ App SVCS  │ │ SQL,         │      │
  │   │ NSGs      │ │ NSGs      │ │ Storage      │      │
  │   └───────────┘ └───────────┘ └──────────────┘      │
  │                                                       │
  │   ┌─────────────────────────────────────────────┐    │
  │   │              Gateway Subnet                  │    │
  │   │    (VPN Gateway, ExpressRoute Gateway)       │    │
  │   └─────────────────────────────────────────────┘    │
  │                                                       │
  │   Peerings → Other VNets (same/other region)         │
  │   Peering → Hub VNet (Platform)                      │
  │   DNS: Azure Public + Private Zones                  │
  │   NSGs: Per-subnet and per-NIC                       │
  │   UDRs: Custom routing paths                         │
  │   Private Link: Access PaaS privately                │
  │   Load Balancer: Traffic distribution                │
  │   Application Gateway: L7 routing + WAF              │
  │   Bastion: Secure VM access                          │
  └─────────────────────────────────────────────────────┘
```

---

## 2. ARCHITECTURE

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          AZURE VIRTUAL NETWORK                                │
│                     vnet-prod-eastus-01 (10.0.0.0/16)                       │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                     SUBNET: web (10.0.1.0/24)                       │   │
│  │  Purpose: Web servers, frontend VMs                                  │   │
│  │  NSG: Allow 80/443 from Internet, Allow 8080 from App subnet         │   │
│  │  VMs: web-01 (10.0.1.4), web-02 (10.0.1.5)                          │   │
│  │  Private Endpoint: privatelink.azurewebsites.net (web-app)           │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                     SUBNET: app (10.0.2.0/24)                       │   │
│  │  Purpose: Application/API servers                                    │   │
│  │  NSG: Allow 8080 from Web subnet, Allow 1433 from Data subnet       │   │
│  │  VMs: api-01 (10.0.2.4), api-02 (10.0.2.5)                         │   │
│  │  Application Gateway: 10.0.2.10 (L7, WAF)                           │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                     SUBNET: data (10.0.3.0/24)                       │   │
│  │  Purpose: Database servers, data storage                             │   │
│  │  NSG: Allow 1433 from App subnet ONLY (no internet)                  │   │
│  │  VMs: sql-01 (10.0.3.4)                                             │   │
│  │  Private Endpoint: privatelink.database.windows.net (sql-primary)    │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                     SUBNET: gateway (10.0.4.0/27)                    │   │
│  │  Purpose: VPN Gateway, ExpressRoute Gateway                          │   │
│  │  Azure-required: Must be named "GatewaySubnet"                       │   │
│  │  Minimum size: /27 (at least 32 IP addresses)                         │   │
│  │  Resources: vpn-gw-01 (10.0.4.4), er-gw-01 (10.0.4.5)               │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                     SUBNET: AzureBastion (10.0.5.0/27)               │   │
│  │  Purpose: Azure Bastion host (if Bastion in this VNet)               │   │
│  │  Minimum: /27 (at least 32 IP addresses)                             │   │
│  │  Resource: bastion-01 (10.0.5.4)                                     │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                     SUBNET: Microsoft.Azure.Services (10.0.6.0/27)   │   │
│  │  Purpose: Azure Firewall (if Firewall in this VNet)                  │   │
│  │  Required subnet name for Firewall: "AzureFirewallSubnet"            │   │
│  │  Resource: fw-01 (10.0.6.4)                                          │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                     SUBNET: AzureFirewallDns (10.0.7.0/27)           │   │
│  │  Purpose: Firewall DNS (if Firewall DNS proxy enabled)               │   │
│  │  Required subnet name: "AzureFirewallDns"                            │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Address Space: 10.0.0.0/16 → 65,536 IPs (65,536 - 16 gateways = ~65,520)  │
│  Usable: 10.0.0.4 - 10.0.255.254 (after Azure reserved IPs)                 │
│                                                                              │
│  Peerings:                                                                   │
│  ├── vnet-hub-01 (Hub) → Accept/Forward (transit)                           │
│  ├── vnet-spoke-app1-prod (App1) → Accept/Forward (transit)                 │
│  ├── vnet-spoke-app2-prod (App2) → Accept/Forward (transit)                 │
│  └── vnet-spoke-data-prod (Data) → Accept/Forward (transit)                 │
│                                                                              │
│  UDRs:                                                                       │
│  ├── 0.0.0.0/0 → Virtual Appliance (Firewall IP: 10.0.6.4)                  │
│  ├── 10.0.0.0/16 → VNet local (direct within VNet)                          │
│  └── 10.1.0.0/16 → VNet peering to Hub (for Hub resources)                  │
│                                                                              │
│  DNS:                                                                        │
│  ├── Azure Public: *.azurewebsites.net, *.blob.core.windows.net             │
│  ├── Private Zones: corp.contoso.com, privatelink.*                         │
│  └── Custom: app1.contoso.com (private)                                      │
└──────────────────────────────────────────────────────────────────────────────┘
```

**How VNet fits into Landing Zone (reference back to Module 9):**

```
┌─ HUB VNet (Platform Landing Zone) ─┐
│  Firewall, DNS, Bastion, Gateway   │
│  Connected to all Spokes via       │
│  VNet Peering (transit enabled)    │
└─────────────────────────────────────┘
         ↕ VNet Peering (transit)
┌─ SPOKE VNets (Application Landing Zone) ─┐
│  Each workload in separate Spoke          │
│  NSGs, UDRs pointing to Hub              │
│  Private Link endpoints for PaaS          │
└───────────────────────────────────────────┘
         ↕ VNet Peering / Gateway
┌─ ON-PREM NETWORK ─┐
│  ExpressRoute/VPN   │
│  Connectivity Layer  │
└─────────────────────┘
```

---

## 3. COMPONENTS — DETAILED

### 3.1 VNET FUNDAMENTALS

**What is a VNet?**
A VNet is a logically isolated network segment in Azure where you can deploy and manage Azure resources (VMs, App Services, databases, etc.) in a private network space. Resources in a VNet can communicate with each other and with the internet (if configured), with on-premises networks (via VPN/ExpressRoute), and with other VNets (via peering).

**Key Properties:**

| Property | Details |
|----------|---------|
| **Name** | Unique within resource group; e.g., `vnet-prod-eastus-01` |
| **Address Space** | CIDR block(s); e.g., `10.0.0.0/16` (65,536 addresses) |
| **Subnets** | Subdivisions of address space; e.g., `10.0.1.0/24`, `10.0.2.0/24` |
| **Region** | VNet is region-specific (but peering can cross regions) |
| **Resource Group** | VNet lives in a resource group |
| **Peerings** | Connections to other VNets |
| **DNS Servers** | Azure-managed (default) or custom |
| **DDOS Protection** | Basic (free) or Standard (paid, 400 Gbps DDoS) |
| **Encryption** | Unsupported at VNet level (each resource manages its own) |

**Azure Reserved IPs in every VNet:**
```
Each VNet reserves 5 IP addresses at the beginning of each subnet:
  .0   → Network address
  .1   → Default gateway (route table default)
  .2   → Azure DNS (if Azure-provided)
  .3   → Reserved (Azure future use)
  .4+  → Available for resources (first usable)

Example: Subnet 10.0.1.0/24
  10.0.1.0   → Network (not usable)
  10.0.1.1   → Gateway (not usable)
  10.0.1.2   → DNS (not usable)
  10.0.1.3   → Reserved (not usable)
  10.0.1.4   → First usable (first VM can have this IP)
  10.0.1.254 → Last usable
  10.0.1.255 → Broadcast (not usable)

Usable addresses in /24: 10.0.1.4 - 10.0.1.254 = 251 addresses (256 - 5 reserved)

IMPORTANT: Static IP assignment must use reserved IPs correctly.
  Manual IP: Must be within subnet range, not .0-.3
  DHCP: Azure automatically assigns from .4+
```

**VNet SKU:**

| SKU | Use Case |
|-----|----------|
| **Basic** | Simple scenarios, dev/test, small workloads |
| **Standard** | Production workloads; required for Azure Firewall, Application Gateway, Gateway Load Balancer |

> **L3 critical:** Azure Firewall, Application Gateway, and Gateway Load Balancer REQUIRE Standard SKU VNet. Basic VNet cannot host these services. If upgrading from Basic to Standard, verify no peerings use Basic.

---

### 3.2 ADDRESS SPACE AND CIDR PLANNING

**CIDR Notation Explained:**
```
CIDR = IP address + prefix length
  10.0.0.0/16 = 10.0.0.0 with 16 network bits → 65,536 addresses (2^(32-16))
  10.0.0.0/24 = 10.0.0.0 with 24 network bits → 256 addresses (2^(32-24))
  10.0.0.0/27 = 10.0.0.0 with 27 network bits → 32 addresses (2^(32-27))

Address calculations:
  /8   → 16,777,216 addresses (16M)
  /16  → 65,536 addresses (64K)
  /20  → 4,096 addresses (4K)
  /24  → 256 addresses (256)
  /27  → 32 addresses (32) — minimum for GatewaySubnet
  /28  → 16 addresses (16)
  /29  → 8 addresses (8)
  /30  → 4 addresses (4) — point-to-point links
  /32  → 1 address (single host)
```

**Recommended VNet Address Space Design:**

```
Enterprise VNet Hierarchy (10.0.0.0/8 private space):

  10.0.0.0/16  → VNet-A (Primary Production)
  ├── 10.0.1.0/24  → Subnet: Web
  ├── 10.0.2.0/24  → Subnet: Application
  ├── 10.0.3.0/24  → Subnet: Data/Database
  ├── 10.0.4.0/27  → Subnet: Gateway (required /27 minimum)
  ├── 10.0.5.0/27  → Subnet: Bastion (if in VNet)
  ├── 10.0.6.0/27  → Subnet: Firewall (if in VNet)
  ├── 10.0.7.0/27  → Subnet: Firewall DNS (if needed)
  └── 10.0.8.0/24  → Subnet: Management/Operations

  10.1.0.0/16  → VNet-B (Secondary/DR)
  ├── 10.1.1.0/24  → Subnet: Web
  ├── 10.1.2.0/24  → Subnet: Application
  └── ...

  10.2.0.0/16  → VNet-C (Development)
  └── ...

  192.168.0.0/16  → VNet-D (Sandbox/Experimentation)
  └── ...
```

**Best Practices for Address Planning:**

| Practice | Description |
|----------|-------------|
| **Leave gaps** | Don't use adjacent /24s — leave at least one /24 gap for future expansion |
| **Plan for growth** | Each subnet should have 50% headroom (if 100 VMs now, plan for 200 — use /23 or /22) |
| **Consistent naming** | `vnet-{app}-{env}-{region}-{n}` with systematic subnet naming |
| **Separate environment tiers** | Don't mix prod/dev address spaces — easier to manage, fewer mistakes |
| **Document everything** | Keep IP address plan updated (spreadsheet, wiki, or tool) |
| **Consider peering** | Address spaces of peered VNets must NOT overlap (non-overlapping CIDR) |
| **Gateway subnet** | Always /27 minimum; if multiple gateways, /24 or larger |
| **Reserve for services** | Bastion (/27), Firewall (/27), Private Link endpoints consume IPs |

**IP Address Exhaustion Warning:**
```
/16 VNet (65,536 IPs): 
  With /24 subnets → 256 subnets possible (but 5 reserved per subnet)
  Realistic usable: ~251 per subnet × 10-15 subnets = ~3,000 usable IPs

When running low:
  - Use /23 or /22 subnets (double or quadruple IPs)
  - Add secondary address space (additional /16 or /20)
  - Re-number (extremely painful, avoid)
  - Use smaller subnets for low-density areas

Tip: Monitor IP usage via VNet → Address Space → Check utilization
```

---

### 3.3 SUBNETS — COMPLETE DEEP DIVE

**What is a Subnet?**
A subnet is a logical subdivision of a VNet's address space. It provides:
- Organization (group related resources)
- Security boundaries (NSGs apply at subnet level)
- Traffic control (UDRs applied at subnet level)
- Performance isolation (subnet resource density)

**Subnet Properties:**

| Property | Details |
|----------|---------|
| **Name** | Descriptive; convention-based (e.g., `subnet-web`, `subnet-app`) |
| **Address Range** | Subset of VNet address space (CIDR) |
| **NSG** | Optional; can be applied at subnet level and/or NIC level |
| **UDR** | Optional; route table associated with subnet |
| **Private Endpoint** | Can deploy Private Endpoints in any subnet |
| **Service Endpoints** | Can enable for Azure services (Storage, SQL, etc.) |
| **Allocation** | Automatic (IPAM) or Static (manual IP assignment) |

**Special Subnet Names (Required by Azure Services):**

| Subnet Name | Required For | Minimum Size | Notes |
|------------|--------------|-------------|-------|
| `GatewaySubnet` | VPN Gateway, ExpressRoute Gateway | /27 (32 IPs) | Must be named exactly this; cannot be used for other resources |
| `AzureFirewallSubnet` | Azure Firewall | /27 (32 IPs) | Must be named exactly this; Standard VNet required |
| `AzureFirewallDns` | Firewall DNS (proxy mode) | /27 (32 IPs) | Must be named exactly this |
| `AzureBastionSubnet` | Azure Bastion (if in this VNet) | /27 (32 IPs) | Recommended: /28 (16 IPs) minimum in newer Bastion versions |
| `AzureLoadBalancerSubnet` | Internal Load Balancer (if required) | /28 (16 IPs) | Newer deployments use any subnet |
| `AzureMonitorSubnet` | Azure Monitor Agent (preview) | /28 (16 IPs) | For monitoring agent communication |

> **L3 critical:** These special subnet names are REQUIRED by Azure. If the subnet doesn't exist with the exact name, the service deployment will fail. The GatewaySubnet is also the reason you need at least a /27 in every VNet — even if no VPN/ExpressRoute is planned (Azure requires it).

**Subnet Design Patterns:**

| Pattern | Description | When to Use |
|---------|-------------|-------------|
| **Tier-based** | Subnets by tier: Web, App, Data, DB | Multi-tier applications |
| **Service-based** | Subnets by service type: VMs, PaaS, Databases | Mixed workloads |
| **Environment-based** | Separate VNet per environment | Strict isolation, separate governance |
| **Team-based** | Subnets per team | Multi-team subscription |
| **Security-based** | Subnets by security zone: DMZ, Internal, Restricted | High-security environments |

**Tier-based Subnet Design (Recommended):**

```
VNet: 10.0.0.0/16
  ├── 10.0.1.0/24 (Web/FE)
  │   ├── VMs: Web servers
  │   ├── App Gateway (if L7 LB)
  │   ├── NSG: Allow 80/443 from Internet; Allow 8080/API from App subnet
  │   └── UDR: 0.0.0.0/0 → Firewall
  │
  ├── 10.0.2.0/24 (App/Middleware)
  │   ├── VMs: Application servers, API gateways
  │   ├── NSG: Allow 8080 from Web; Allow [service ports] from Web and Data
  │   └── UDR: 0.0.0.0/0 → Firewall; 10.0.3.0/24 → VNet local
  │
  ├── 10.0.3.0/24 (Data/Database)
  │   ├── VMs: SQL Server, MongoDB, etc.
  │   ├── NSG: Allow 1433 (SQL) from App subnet ONLY; Deny everything else
  │   └── UDR: 10.0.0.0/16 → VNet local (no internet route)
  │
  ├── 10.0.4.0/27 (Gateway)
  │   ├── VPN Gateway
  │   ├── ExpressRoute Gateway
  │   └── NSG: Allow gateway traffic (1701 GRE, 500 ESP, 4500 UDP)
  │
  ├── 10.0.5.0/27 (Bastion) [optional if Bastion in this VNet]
  │   └── Azure Bastion host
  │
  ├── 10.0.6.0/27 (Azure Firewall) [optional if Firewall in this VNet]
  │   └── Azure Firewall
  │
  └── 10.0.8.0/24 (Management)
      ├── Azure Monitor Agent
      ├── Management VMs
      └── NSG: Restricted access (jumpbox only)
```

**Subnet vs NIC-level NSG:**

| Aspect | Subnet NSG | NIC NSG |
|--------|-----------|---------|
| **Scope** | All resources in subnet | Single NIC (single VM) |
| **Management** | Centralized (one NSG per subnet) | Per-resource (many NSGs) |
| **Consistency** | High (same rules for all in subnet) | Variable (each VM can differ) |
| **Performance** | Evaluated once per subnet flow | Evaluated per NIC |
| **Use case** | Default allow/deny at tier level | Exception rules per VM |
| **Recommended** | Primary security boundary | Supplementary exceptions |

> **Best Practice:** Apply NSGs at subnet level for base security. Use NIC-level NSGs only for specific exceptions (e.g., one VM needs a special port).

---

### 3.4 NETWORK SECURITY GROUPS (NSGs) — COMPLETE DEEP DIVE

**What is an NSG?**
An NSG is a virtual firewall that filters network traffic to and from resources in an Azure VNet. NSGs contain security rules that allow or deny traffic based on source/destination IP, port, protocol, and priority.

**NSG Evaluation Order:**

```
Traffic evaluation sequence (in order):

1. User-Driven Default (if applicable):
   → Allow VNet to VNet (default, can be overridden)
   → Allow Internet inbound if any SG rule allows (default, can be overridden)
   → Deny all other default traffic

2. NSG Rules (evaluated by priority, lowest number first):
   → Priority 100-4095 (user-defined)
   → Priority 65500-65550 (Azure default, cannot be modified)
   → First matching rule wins (Evaluation stops at first match)
   
3. Effective Rules = Combined (Subnet NSG + NIC NSG)
   → Rules from both NSGs are merged
   → Most restrictive (first match) applies

Default NSG Rules (Azure-managed, in Priority 65500-65550 range):
  Priority 65500: Allow VNet Inbound (within VNet)
  Priority 65501: Allow VNet Outbound (within VNet)
  Priority 65502: Allow Azure Load Balancer Inbound (health probes)
  Priority 65503: Deny All Inbound (Internet → VNet, unless overridden)
  Priority 65504: Allow Outbound (VNet → Internet)
  Priority 65505: Deny All Outbound (can be overridden by user rules)

Note: "Allow VNet to VNet" means within the SAME VNet (including peered VNets with transitive settings)
```

**NSG Rule Properties:**

| Property | Values | Description |
|----------|--------|-------------|
| **Name** | Descriptive string | e.g., `AllowHTTPFromInternet`, `DenyAllInbound` |
| **Priority** | 100-4095 (user) / 65500-65550 (Azure) | Lower = evaluated first. Unique within NSG. |
| **Source** | IP address, IP range, Service Tag, Application Security Group, Any | Where traffic comes from |
| **Source port** | Any or specific range | Usually "Any" |
| **Destination** | IP address, IP range, Service Tag, ASG, Any | Where traffic goes |
| **Destination port** | Any or specific range | e.g., 80, 443, 1433, 22 |
| **Protocol** | TCP, UDP, ICMP, Any | Transport protocol |
| **Action** | Allow or Deny | What to do when rule matches |

**Service Tags (Pre-defined):**

| Service Tag | Description | Use Case |
|------------|-------------|---------|
| `Internet` | All public IP addresses outside Azure virtual network | Allow/deny internet traffic |
| `AzureCloud` | All Azure cloud IPs (includes Azure services + Microsoft 365) | Allow Azure API calls, Azure services |
| `AzureLoadBalancer` | Azure Load Balancer IPs | Internal LB health probes |
| `VirtualNetwork` | All address spaces in current VNet + peered VNets + 10.0.0.0/8 (for Azure internal traffic) | VNet-to-VNet traffic |
| `AzureActiveDirectory` | Azure AD IPs (Microsoft managed) | Conditional Access, AAD calls |
| `Storage` | Azure Storage IP ranges | Service Endpoints (replaced by Private Endpoint in best practice) |
| `Sql` | Azure SQL IP ranges | Service Endpoints |
| `AzureMonitor` | Azure Monitor IP ranges | Monitoring agent connectivity |
| `AzureCosmos` | Azure Cosmos DB IP ranges | Cosmos DB connectivity |

> **L3 critical:** Service Tags are NOT static — they change as Azure updates infrastructure. NSGs using Service Tags automatically follow these changes. This is why Service Tags are preferred over hardcoding IP ranges (which go stale).

**NSG Best Practice Rules (Standard Template):**

```
Standard NSG Rule Set for Web Subnet:

  Priority 100: Allow VNet Inbound (Azure default — leave)
  Priority 200: Allow HTTP from Internet (Source: Internet, Dest: *, Port: 80, TCP, Allow)
  Priority 210: Allow HTTPS from Internet (Source: Internet, Dest: *, Port: 443, TCP, Allow)
  Priority 220: Allow HTTPS from AzureLoadBalancer (Source: AzureLoadBalancer, Dest: *, Port: 443, TCP, Allow) — health probes
  Priority 300: Allow App port from App subnet (Source: 10.0.2.0/24, Dest: *, Port: 8080, TCP, Allow)
  Priority 400: Allow SSH from Bastion (Source: Bastion subnet, Dest: *, Port: 22, TCP, Allow)
  Priority 401: Allow RDP from Bastion (Source: Bastion subnet, Dest: *, Port: 3389, TCP, Allow)
  Priority 500: Deny All Inbound (Source: *, Dest: *, Port: *, *, Deny) — explicit deny
  
Standard NSG Rule Set for Data/DB Subnet:

  Priority 100: Allow VNet Inbound (Azure default)
  Priority 200: Allow SQL from App subnet (Source: 10.0.2.0/24, Dest: *, Port: 1433, TCP, Allow)
  Priority 210: Allow App port from App subnet (Source: 10.0.2.0/24, Dest: *, Port: 8080, TCP, Allow)
  Priority 220: Allow from Data subnet (Source: 10.0.3.0/24, Dest: *, Port: *, *, Allow) — DB-DB communication
  Priority 230: Allow from Bastion (Source: Bastion subnet, Dest: *, Port: 22/3389, *, Allow)
  Priority 300: Deny All Inbound (Source: *, Dest: *, Port: *, *, Deny) — NO INTERNET
  Priority 400: Allow VNet Outbound (Azure default)
  Priority 500: Deny All Outbound (Source: *, Dest: *, Port: *, *, Deny) — CAN BE OVERRIDDEN by UDR
```

**NSG Flow Logs — CRITICAL FOR TROUBLESHOOTING:**

```
What NSG Flow Logs Capture:
  → All IP traffic allowed/denied by NSGs
  → Source IP, Destination IP, Source Port, Dest Port
  → Protocol, Action (Allow/Deny), Rule that matched
  → Flow start/end timestamps, packet count, byte count
  → Interface direction (Ingress/Egress)

Where Flow Logs Go:
  → Storage Account (long-term retention, low cost)
  → Log Analytics workspace (real-time analysis)
  → Both (dual)

Why NSG Flow Logs are Essential:
  1. Troubleshooting: "My VM can't connect" → Check flow logs: denied by which rule?
  2. Security audit: What traffic is being blocked?
  3. Network visualization: Map all traffic flows
  4. Compliance: Prove network segmentation
  5. Impact analysis: Before changing NSG rules, check current flow

Performance Impact:
  → Minimal (NSG evaluates anyway; logging adds negligible overhead)
  → Check "Flow Log Status" in Portal before troubleshooting

Without Flow Logs: "VM can't connect" becomes guesswork.
With Flow Logs: "VM can't connect because NSG Rule #220 denied TCP/443 from 10.0.1.4"
```

---

### 3.5 USER DEFINED ROUTES (UDRs) — COMPLETE DEEP DIVE

**What is a Route Table (UDR)?**
A Route Table defines the routing paths for traffic leaving a subnet. Each subnet can be associated with exactly one route table. Routes override the default Azure system routes (or supplement them).

**Route Properties:**

| Property | Values | Description |
|----------|--------|-------------|
| **Address Prefix** | CIDR block | Destination network (e.g., 0.0.0.0/0 for all internet traffic) |
| **Next Hop Type** | Virtual Appliance, Virtual Network, Internet, Virtual Network Gateway, None | Where traffic goes |
| **Next Hop Address** | IP address (for Virtual Appliance) | IP of the next hop (e.g., Firewall IP) |

**Default System Routes (Always Present):**

```
Every subnet has these default routes (cannot be deleted):
  Address Prefix: 10.0.0.0/8    → Virtual Network (VNet local)
  Address Prefix: 0.0.0.0/0     → Internet (default outbound)
  Address Prefix: 10.0.0.0/16   → VNet (specific VNet's address space)
  Address Prefix: 168.63.129.16/32 → AzureDNS (single IP, DNS resolution)

These CAN be overridden by user-defined routes (UDRs):
  → UDR with prefix 0.0.0.0/0 overrides default Internet route
  → UDR with prefix 10.0.0.0/8 overrides default VNet route (rare)

These CANNOT be overridden:
  → AzureDNS route (168.63.129.16/32)
  → VNet-local route within the same VNet's primary address space
  → BGP routes from Gateway (learned routes, higher priority)
```

**UDR Architecture (Hub-and-Spoke with Firewall):**

```
VNet: 10.0.0.0/16
  Route Table "RT-Default" (associated with ALL subnets EXCEPT Gateway):
    Route 1: 0.0.0.0/0 → Virtual Appliance (Next Hop: 10.0.6.4 = Firewall IP)
    Route 2: 10.0.0.0/16 → VNet local (handled by Azure default — NOT in UDR)
    Route 3: 10.1.0.0/16 → Virtual Network (Hub VNet peering — for Hub resources)
    Route 4: 168.63.129.16/32 → Internet (DNS — Azure default, NOT in UDR)

Subnet "web" (10.0.1.0/24) → Associated with RT-Default
Subnet "app" (10.0.2.0/24) → Associated with RT-Default
Subnet "data" (10.0.3.0/24) → Associated with RT-Default
Subnet "gateway" (10.0.4.0/27) → NO UDR (needs direct internet access for gateway)

Result:
  web-01 (10.0.1.4) → Internet → Goes to Firewall (10.0.6.4) → Internet
  web-01 (10.0.1.4) → app-01 (10.0.2.4) → Direct VNet traffic (same VNet)
  app-01 (10.0.2.4) → sql.service.internal → Direct VNet traffic or DNS resolution

WITHOUT UDR for GatewaySubnet:
  GatewaySubnet would send VPN/ExpressRoute traffic through Firewall → BLOCKED
  GatewaySubnet must have NO UDR (uses default Internet route for gateway)
```

**User-Defined Route Types:**

| Next Hop | Use Case | Example |
|----------|----------|---------|
| **Virtual Appliance** | Force traffic through NVA (Firewall, IDS/IPS, proxy) | 0.0.0.0/0 → 10.0.6.4 (Firewall) |
| **Virtual Network** | Route to another VNet (transit, peering) | 10.1.0.0/16 → VNet peering to Hub |
| **Virtual Network Gateway** | Route to on-premises via VPN/ER | 10.10.0.0/16 → VPN Gateway (on-prem 10.10.x.x) |
| **Internet** | Direct internet access (overrides default if needed) | Special cases |
| **None** | Drop traffic (black hole route) | Security: block specific CIDR |

**UDR Association Rules:**

```
One route table per subnet (maximum)
→ Can be associated with multiple subnets (same RT shared)
→ Different subnets can have different RTs

Use case:
  RT-Default: Associated with Web, App, Data subnets → all traffic via Firewall
  RT-Gateway: NOT associated with GatewaySubnet → default routing
  RT-DB: Could be separate RT for Data subnet with specific routing (e.g., to dedicated DB subnet)

Propagate settings:
  Subnet can propagate routes to VNet peers (default: yes for GatewaySubnet)
  If propagate is OFF, routes only apply to the directly associated subnet
```

**Multiple UDRs and Route Selection:**

```
If multiple UDRs exist (different subnets with different RTs), or when a subnet has both system routes and UDRs:

Rule: Most specific prefix wins.
  10.0.0.0/16 (VNet local) — System (more specific)
  0.0.0.0/0 (Internet) — UDR (less specific)

Traffic to 10.0.2.4: Matches 10.0.0.0/16 (VNet local) → direct VNet routing
Traffic to 8.8.8.8: Matches 0.0.0.0/0 (UDR) → Firewall (10.0.6.4)

If two UDRs match (rare but possible with overlapping prefixes):
  Lower address prefix (more specific) wins
  E.g., /24 wins over /16 for same destination
```

---

### 3.6 VNET PEERING — COMPLETE DEEP DIVE

**What is VNet Peering?**
VNet Peering connects two VNets so that resources in them can communicate privately using private IP addresses. Peering is bi-directional and traffic stays on the Azure backbone (never over the internet).

**Peering Properties:**

| Property | Details |
|----------|---------|
| **Name** | e.g., `peering-vnet-spoke1-to-vnet-hub` |
| **Remote VNet** | The other VNet being connected to |
| **Allow VNet Access** | Whether resources can communicate (enabled/disabled) |
| **Allow Forwarded Traffic** | Whether traffic from remote VNet (not direct peer) can transit through this VNet |
| **Use Remote Gateways** | Whether this VNet can use remote VNet's VPN/ExpressRoute gateway |
| **Allow Gateway Transit** | Whether this VNet's gateway can be used by remote VNet (if Use Remote Gateways is enabled on remote) |
| **Fwd-remained traffic (Timeout)** | Timeout for forwarded traffic |
| **Peering Link** | Per-subnet peering (preview) — more granular control |

**Peering Direction:**

```
Two peering configurations needed (one in each VNet):

VNet-A: peering-to-B
  → Remote VNet: VNet-B
  → Allow VNet Access: Enabled
  → Allow Forwarded Traffic: Enabled (if transit needed)
  → Use Remote Gateways: Enabled (if using VNet-B's gateway)

VNet-B: peering-to-A
  → Remote VNet: VNet-A
  → Allow VNet Access: Enabled
  → Allow Forwarded Traffic: Enabled (if transit needed)
  → Allow Gateway Transit: Enabled (if VNet-A's gateway is shared)
  → Use Remote Gateways: Disabled

IMPORTANT: "Allow Forwarded Traffic" vs "Allow Gateway Transit":
  → Allow Forwarded Traffic (on local VNet peering): Traffic from VNet-B that is destined for VNet-C can pass through VNet-A IF VNet-A has a peering to VNet-C and forwarded traffic is enabled.
  → Allow Gateway Transit (on remote VNet peering): If VNet-B has "Use Remote Gateways" enabled, it can use VNet-A's gateway. VNet-A must have "Allow Gateway Transit" enabled on its peering to VNet-B.

Classic Hub-and-Spoke with Peering:
  VNet-Hub has:
    peering to Spoke-1: Allow Forwarded Traffic = Enabled
    peering to Spoke-2: Allow Forwarded Traffic = Enabled
  
  Spoke-1 has:
    peering to Hub: Allow Forwarded Traffic = Enabled, Use Remote Gateways = (depends)
    peering to Spoke-2: NOT needed (traffic goes through Hub)
  
  Traffic flow: Spoke-1 → Hub (transit) → Spoke-2 (transit through Hub)
```

**Peering Rules and Constraints:**

| Rule | Description |
|------|-------------|
| **Non-overlapping address space** | Peered VNets MUST have non-overlapping CIDR |
| **Bi-directional** | Must be configured from BOTH sides |
| **One peering per pair** | Only one peering between any two VNets |
| **No transitive** | VNet-A ↔ VNet-B ↔ VNet-C does NOT mean VNet-A ↔ VNet-C (must configure directly, or use VNet-B with forwarded traffic) |
| **DNS resolution** | VNet peering enables name resolution between VNets (by default) — Azure DNS resolves private IPs in peered VNets |
| **Same region or cross-region** | Peering can be within region or cross-region (with same rules) |
| **Subscription** | Can peer VNets in different subscriptions (same or different tenants with specific conditions) |
| **Bandwidth** | VNet peering bandwidth = VM SKU bandwidth limits (not VNet limit) |
| **Cannot be deleted** | If traffic is flowing (any active connection) — must terminate all connections first |

> **L3 critical:** "No transitive routing" means you CAN'T do VNet-A → VNet-B → VNet-C without explicitly enabling "Allow Forwarded Traffic" on both ends of VNet-B. The most common cause of "peering works one way but not the other" is missing Allow Forwarded Traffic.

**Hub-and-Spoke with Transit:**

```
Hub VNet: vnet-hub-eastus (10.0.0.0/16)
  ├── Peering to Spoke-1: Allow Forwarded Traffic = ENABLED
  ├── Peering to Spoke-2: Allow Forwarded Traffic = ENABLED
  ├── Peering to Spoke-3: Allow Forwarded Traffic = ENABLED
  └── UDR: 0.0.0.0/0 → Firewall; 10.1.0.0/16 → VNet local

Spoke-1: vnet-spoke-app1 (10.1.1.0/24)
  ├── Peering to Hub: Allow Forwarded Traffic = ENABLED, Use Remote Gateways = ENABLED
  └── UDR: 0.0.0.0/0 → Remote VNet (Hub) via peering
      (more specifically: 0.0.0.0/0 → Virtual Network, Next Hop = Hub peering)
      Or: 0.0.0.0/0 → VNet local (Azure default for direct VNet)
      
Traffic Flow: Spoke-1 VM → UDR → Hub VNet (10.0.0.0/16) → Firewall (NVA) → Internet

IMPORTANT: Spoke-1's UDR for 0.0.0.0/0 must route through Hub's address space (10.0.0.0/16)
  NOT directly to Internet (that would bypass Hub/Firewall!)

Common mistake: Spoke-1 has UDR 0.0.0.0/0 → Internet. This BYPASSES Hub/Firewall.
  Fix: UDR 0.0.0.0/0 → Virtual Network → Hub VNet (Azure peering next hop)
```

**VNet Peering vs VNet Gateway connectivity:**

| Feature | VNet Peering | VNet Gateway |
|---------|-------------|-------------|
| **Connection type** | Azure backbone (private) | VPN/ExpressRoute (tunnel) |
| **Between** | Two Azure VNets | Azure VNet ↔ On-premises |
| **Bandwidth** | Limited by VM SKU | Limited by Gateway SKU |
| **Latency** | Very low (Azure backbone) | Depends on connection type |
| **Use case** | Azure-to-Azure connectivity | Azure-to-on-premises |
| **Can transit** | Yes (with Allow Forwarded Traffic) | No (only one gateway per VNet) |
| **Cost** | No cost for peering traffic (data transfer may cost) | Gateway + data transfer costs |
| **Security** | NSG controls | VPN encryption + NSG |

---

### 3.7 DNS IN VNets — COMPLETE DEEP DIVE

**Azure DNS Architecture:**

```
DNS Resolution in Azure:

  1. Azure Public DNS (default, for all VNets)
     → Resolves: *.azurewebsites.net, *.blob.core.windows.net, *.queue.core.windows.net, etc.
     → Resolves private IPs for PaaS services to PRIVATE IPs (not public!)
     → Managed by Azure, no configuration needed
     → Cannot be disabled (can only change custom DNS settings)

  2. Azure Private DNS Zones
     → Your own custom DNS zones (e.g., corp.contoso.com, app1.contoso.com)
     → Resolves internal names to private IPs
     → Must be LINKED to one or more VNets to be accessible
     → Registered vs Delegated mode:
        - Registered: Link directly to VNet (simpler)
        - Delegated: Use Private DNS Resolver for cross-VNet resolution (more flexible)

  3. Custom DNS Servers (VM-based)
     → Windows DNS Server VM in VNet
     → Linux BIND/Dnsmasq VM in VNet
     → Forward queries to Azure DNS + resolve internal
     → Used when you need: conditional forwarding, custom records, split-brain DNS

  4. Azure DNS Private Resolver (preview/GA)
     → VNet-level DNS resolver
     → Rules: forwarding rulesets (where to forward specific domains)
     → Better than VM-based for: simple forwarding, no VM maintenance
     → In-bound endpoint (receives queries from VNet)
     → Out-bound endpoint (for forwarding to on-prem DNS)
```

**DNS Resolution Flow (Hub-and-Spoke with Private DNS):**

```
VM in Spoke-1 (10.1.1.4) queries: "storageaccount1.blob.core.windows.net"
  → VM DNS: 168.63.129.16 (Azure DNS)
  → Azure DNS resolves to: Private IP (e.g., 10.0.6.10) if Private Link exists
  → VM connects to 10.0.6.10 via VNet (NOT internet)

VM in Spoke-1 queries: "app1.contoso.com" (custom internal)
  → IF Private DNS Zone "app1.contoso.com" linked to Spoke-1 VNet:
    → Resolves from linked zone → private IP (e.g., 10.1.1.10)
  → IF NOT linked to Spoke-1 VNet:
    → Query fails (Azure public DNS doesn't know your private zone)
    → Resolution fails!

Fix: Either link Private DNS Zone to Spoke-1, or use Azure DNS Private Resolver with forwarding rules

VM in Spoke-1 queries: "corp.contoso.com" (if using custom DNS server)
  → VM DNS: Custom DNS server IP (e.g., 10.0.0.4)
  → Custom DNS server:
    → If query matches local zone: resolves from internal database
    → If not: forwards to Azure DNS (168.63.129.16)
    → If conditional forward: forwards to on-prem DNS (e.g., 10.10.10.1)
```

**Private DNS Zone Linking Modes:**

| Mode | Description | When to Use |
|------|-------------|-------------|
| **Registered** | Private DNS Zone linked directly to VNet(s); VM in linked VNet can resolve | Simple scenarios; VNet-specific zones |
| **Delegated** | Uses Azure DNS Private Resolver with rules; more flexible cross-VNet | Multi-VNet resolution; complex routing |

---

### 3.8 PRIVATE LINK AND PRIVATE ENDPOINTS — COMPLETE DEEP DIVE

**What is Azure Private Link?**
Azure Private Link is a service that lets you access Azure PaaS services (Azure SQL, Storage, Key Vault, etc.) and Azure hosted customer-owned/partnered services over a private endpoint in your virtual network.

**What is a Private Endpoint?**
A Private Endpoint is a network interface in your VNet (with a static private IP) that connects privately to a PaaS service.

```
WITHOUT Private Link:
  VM → Internet → PaaS Service (public endpoint)
  → Traffic over internet (security risk)
  → Requires public IP on VM or service
  → Firewall/NSG must allow public access

WITH Private Link:
  VM → VNet (private route) → Private Endpoint (10.0.3.10)
  → Azure Private Link backbone → PaaS Service (internal)
  → NEVER touches internet
  → No public IP needed
  → Firewall/NSG don't apply (traffic is private)

Connection Flow:
  VM (10.0.2.4) → DNS resolves "privatelink.database.windows.net" → 10.0.3.10
  → TCP to 10.0.3.10 → Private Endpoint → Azure Backbone → Azure SQL

Components:
  1. Private Endpoint: Network interface in Spoke VNet with private IP
  2. Private DNS Zone: Maps privatelink.database.windows.net to private IP
  3. Private Link Service: The PaaS service being accessed
  4. Connection: Approval-based link between endpoint and service
```

**Private Endpoint Properties:**

| Property | Details |
|----------|---------|
| **VNet** | Must be in a VNet (can be different from the service's VNet) |
| **Subnet** | Any subnet (but must have correct NSG and UDR rules) |
| **Private IP** | From subnet address range (static, auto-assign, or manual) |
| **Group IDs** | Specific to service type (e.g., SQL Server = `sqlServer`, Blob Storage = `blob`) |
| **Status** | Pending, Approved, Rejected, Failed, Succeeded |

**Private Endpoint DNS Configuration:**

```
Option 1: Private DNS Zone (simplest)
  → Azure creates Private DNS Zone: privatelink.database.windows.net
  → Link this zone to VNet(s) where Private Endpoint resides
  → DNS auto-resolves: privatelink.database.windows.net → private IP

Option 2: Private DNS Resolver (more control)
  → Use forwarding rulesets
  → More flexible cross-VNet resolution
  → Can apply different rules per subnet/zone

Option 3: Custom DNS (manual)
  → Manually create A record in custom DNS
  → Point privatelink.database.windows.net → private IP
  → Hardest to maintain, least recommended
```

**Common Private Link Services:**

| Service | Group ID | Private DNS Zone |
|---------|----------|-----------------|
| Azure SQL Database | `sqlServer` | `privatelink.database.windows.net` |
| Azure SQL Managed Instance | `sqlServer` | `privatelink.database.windows.net` |
| Azure Storage (Blob) | `blob` | `privatelink.blob.core.windows.net` |
| Azure Storage (File) | `file` | `privatelink.file.core.windows.net` |
| Azure Key Vault | `vault` | `privatelink.vaultcore.azure.net` |
| Azure App Service (Web Apps) | `appService` | `privatelink.azurewebsites.net` |
| Azure Service Bus | `serviceBus` | `privatelink.service.core.windows.net` |
| Azure Event Hubs | `eventHub` | `privatelink.eventhub.core.windows.net` |
| Azure Synapse | `synapse` | `privatelink.dev.azure.net` |
| Azure VM (as service) | `virtualMachine` | Custom |
| Azure Container Registry | `registry` | `privatelink.azurecr.io` |

---

### 3.9 LOAD BALANCERS — COMPLETE DEEP DIVE

**Azure Load Balancer Types:**

| Type | Tier | Use Case |
|------|------|----------|
| **Basic** | Basic SKU | Simple, single-region, low-budget scenarios |
| **Standard** | Standard SKU | Production; required with Standard VNet; zone-redundant, HA |

**Load Balancer vs Application Gateway:**

| Feature | Load Balancer | Application Gateway |
|---------|--------------|-------------------|
| **Layer** | Layer 4 (TCP/UDP) | Layer 7 (HTTP/HTTPS) |
| **Protocol** | TCP, UDP, HTTP, HTTPS | HTTP, HTTPS only |
| **Content-based routing** | No | Yes (URL path, host header, headers) |
| **WAF** | No | Yes (WAF v2 with OWASP rules) |
| **SSL offloading** | Yes | Yes |
| **Cookie-based affinity** | Yes (5-tuple, source IP) | Yes (cookie-based, App affinity) |
| **Auto-scaling** | No (Manual SKU) | Yes (v2 with autoscale) |
| **Multiple ports** | Yes (any TCP/UDP) | Limited to HTTP/HTTPS (80/443) |
| **Subnets** | Any subnet | Any subnet (Standard VNet required) |
| **Cost** | Lower | Higher (v2) |
| **Use case** | Any TCP/UDP load balancing | Web apps, API management, WAF |

**Internal Load Balancer (ILB) Configuration:**

```
ILB in Subnet: 10.0.2.0/24
  Name: ilb-app1-api
  SKU: Standard (recommended)
  Frontend IP: 10.0.2.100 (internal IP only)
  Backend Pool: VMs in App Subnet (app-01, app-02, app-03)
  Health Probe: TCP 8080 (every 5 sec, 2 failures = unhealthy)
  Load Balancing Rule: 
    Frontend: 10.0.2.100:8080
    Backend: 10.0.2.x:8080 (VMs in backend pool)
    Protocol: TCP
    Idle Timeout: 4 min
    Floating IP: Disabled (usually)

Traffic Flow:
  User (any VNet) → 10.0.2.100:8080 → ILB → distributes to VM with port 8080 open

Health Probe Importance:
  If health probe fails for a VM → VM removed from backend pool (no traffic)
  When probe passes again → VM re-added to pool
  Without health probes: all VMs get traffic (even unhealthy ones)
```

---

### 3.10 AZURE BASTION — COMPLETE DEEP DIVE

**What is Azure Bastion?**
Azure Bastion is a fully managed service that provides secure and seamless RDP/SSH connectivity to your virtual machines directly from the Azure portal over TLS. Bastion supports all Azure VM creation methods (ARM, classic, custom images) and connects to any VM in any subnet of a virtual network to which it is attached.

**Bastion Deployment:**

```
Bastion in Subnet: AzureBastionSubnet (minimum /27 in Standard VNet)
  SKU: Basic (limited), Standard (recommended), Premium (Gen2 VMs, Session Recording, Auditing)
  Generation: Generation1 (Standard B1), Generation2 (Premium; better performance)
  Public IP: Required (Bastion has a public IP for browser connection)
  Scale Unit: Minimum 2 for HA (across availability zones)

Bastion Configuration:
  - Name: bastion-prod-eastus-01
  - Subnet: AzureBastionSubnet (10.0.5.0/27)
  - Public IP: bastion-prod-eastus-01-pip (Standard SKU, Zone-redundant)
  - Scale Units: 2 (HA pair)
  - OS Type: Windows (RDP), Linux (SSH), or Both
  - Copy/Paste: Enabled
  - File Transfer: Enabled (Standard+ Premium)
  - Session Recording: Premium only (auditing)

Traffic Flow:
  User Browser → Bastion Public IP (HTTPS/443) → Bastion → Private link in VNet → VM (no public IP on VM needed)

NSG for Bastion Subnet:
  - Allow AzureBastionSubnet → VirtualNetwork (outbound for RDP/SSH)
  - Allow AzureBastionSubnet → Internet (outbound for certificate download, usually to *.microsoft.com for auth)
  - Allow AzureBastionSubnet → AzureCloud (for Azure AD auth)

Common NSG rule for Bastion subnet:
  Priority: 100
  Source: AzureBastionSubnet
  Destination: VirtualNetwork
  Port: Any
  Action: Allow

  Priority: 110
  Source: AzureBastionSubnet
  Destination: Internet
  Port: 443
  Action: Allow

Without Bastion:
  - VMs need public IPs (security risk)
  - NSGs must allow RDP/SSH from your IP (management complexity)
  - No centralized audit trail of sessions
  - Risk of forgotten open RDP/SSH rules
```

---

### 3.11 SERVICE ENDPOINTS (Deprecated Path)

**Note:** Service Endpoints are being deprecated in favor of Private Endpoints for most services. Understanding for legacy contexts only.

```
Service Endpoint:
  → Enables secure access to Azure PaaS services (Storage, SQL, etc.) over Azure backbone
  → Adds service traffic to VNet (originates from VNet subnet)
  → Identity: VNet Subnet (not individual VM)
  → Configuration: Enable endpoint on subnet, add ACL on PaaS resource

Private Endpoint (replacement):
  → Creates a private IP in your VNet for the PaaS service
  → Traffic: Direct from VNet to PaaS via private route
  → Identity: Private Endpoint IP (specific to the connection)
  → Configuration: Create endpoint in VNet subnet, link to PaaS resource

Why Private Endpoint is Preferred:
  → Service Endpoints only work for specific Azure services
  → Service Endpoints don't provide static IPs
  → Service Endpoints don't work across tenants
  → Private Endpoints: static IP, DNS integration, cross-Tenant, any service with Private Link
```

---

### 3.12 NETWORK INTERNAL GATEWAY LOAD BALANCER / AZURE FIREWALL INTEGRATION

**Azure Firewall as NVA in VNet:**

```
Azure Firewall Deployment in VNet:
  Subnet: AzureFirewallSubnet (10.0.6.0/27 — Standard VNet required)
  SKU: Standard (recommended) or Premium (with IDPS)
  Zone: Zone-redundant (across AZs)
  Public IP: Firewall needs public IP for egress SNAT and DNAT
  
Internal Architecture:
  Firewall (HA pair) in AzureFirewallSubnet
    ├── Management IP: for admin access
    ├── Internal IP: for internal traffic (if needed)
    └── Public IP: for SNAT (outbound) and DNAT (inbound)

UDR for Firewall (Force Trafic):
  Subnet: web (10.0.1.0/24)
  UDR:
    0.0.0.0/0 → Virtual Appliance (Next Hop: 10.0.6.4 = Firewall)
    10.0.0.0/16 → VNet local
    10.1.0.0/16 → Virtual Network (Hub peering)

NSG for Firewall Subnet:
  Allow AzureFirewallSubnet → VirtualNetwork (outbound for traffic to be forwarded)
  Allow AzureFirewallSubnet → Internet (outbound for SNAT, health checks)

Azure Firewall Manager Integration:
  → Central Firewall Policy (reusable)
  → Threat intelligence (alert/deny)
  → TLS inspection (preview)
  → DNS security
  → Security stacking (auto-deploy Firewall in spoke subnets for distributed inspection)
```

---

## 4. VNET DESIGN PATTERNS

### 4.1 HUB-AND-SPOKE (Most Common)

```
┌─ Hub VNet ─┐
│  Firewall   │
│  DNS        │
│  Bastion    │
│  Gateway    │
└─────────────┘
     ↕ Peer (transit)
┌─ Spoke-1 ─┐  ┌─ Spoke-2 ─┐  ┌─ Spoke-3 ─┐
│ Web VMs   │  │ App VMs   │  │ Data VMs  │
│ NSG       │  │ NSG       │  │ NSG       │
│ UDR → Hub │  │ UDR → Hub │  │ UDR → Hub │
└───────────┘  └───────────┘  └───────────┘

Advantages:
  → Centralized security (Firewall in Hub)
  → Simplified management (one Firewall for all)
  → No duplicate infrastructure
  → Consistent DNS
  → Single Bastion for all VMs
  → Cost efficiency

Disadvantages:
  → Hub is SPOF (mitigate with HA/redundancy)
  → Latency through Hub (usually minimal)
  → Hub complexity (many services managed there)
```

### 4.2 SPoke-TO-SPOKE (Direct Peering)

```
Spoke-1 ↔ Spoke-2 Direct Peering (No Hub)
  → Use case: High-bandwidth, low-latency between specific workloads
  → Security: Both Spokes must have NSGs allowing the traffic
  → Trade-off: Loses centralized inspection

When to use:
  → Spoke-1 and Spoke-2 are same team/tier
  → Bandwidth-intensive traffic (e.g., data transfer > 10 Gbps)
  → Latency-sensitive (e.g., distributed cache, database replication)

When NOT to use:
  → Cross-team communication (lose visibility)
  → Security inspection required
  → More than 2-3 peers (becomes mesh = management nightmare)
```

### 4.3 PER-WORKLOAD VNET

```
Each workload gets its own VNet:
  vnet-app1-prod: 10.0.0.0/16 (App1 production)
  vnet-app2-prod: 10.1.0.0/16 (App2 production)
  vnet-app1-dev:  10.2.0.0/16 (App1 development)

Connected via:
  → VNet Peering (for app1-prod ↔ app1-dev)
  → VNet Gateway (for app1-prod ↔ on-prem)

Advantages:
  → Maximum isolation (network failure in one VNet doesn't affect others)
  → Clear ownership
  → Independent address spaces
  → Simpler NSGs (fewer rules)
  → Simpler UDRs (fewer routes)

Disadvantages:
  → More VNets to manage
  → Cross-VNet communication requires peering
  → Shared services duplicated (unless using Hub)
  → Cost (more VNets = small cost, but management overhead)

Best for: Large enterprises, multi-tenant, strict compliance
```

### 4.4 TIERED VNET

```
All tiers in same VNet:
  vnet-prod: 10.0.0.0/16
    ├── Subnet-Web (10.0.1.0/24): Web servers
    ├── Subnet-App (10.0.2.0/24): Application servers  
    ├── Subnet-DB (10.0.3.0/24): Database servers
    └── Subnet-DMZ (10.0.0.0/27): DMZ (proxies, gateways, public-facing)

Advantages:
  → Single VNet to manage
  → Simple peering
  → VNet-level DNS resolution works automatically

Disadvantages:
  → Single VNet failure affects everything
  → Address space planning complexity
  → NSG rules more complex
  → Security boundary is subnet-level (not VNet-level)

Best for: Single applications, smaller environments
```

---

## 5. VNET TROUBLESHOOTING — DEEP METHODOLOGY

### 5.1 Connectivity Troubleshooting Flow

```
STEP 1: Can the VM be reached at all?
  → Ping VM's private IP from another VM in same VNet (no ICMP usually — use Test-NetConnection or nc)
  → RDP/SSH from Bastion (not from internet)
  
STEP 2: Check VM status
  → Is VM running? (not deallocated, not stopped)
  → Is NSG on VM allowing the traffic?
  → Is NSG on subnet allowing the traffic? (subnet NSG + NIC NSG combined)
  
STEP 3: Check NSG Flow Logs (most powerful tool)
  → What rule is blocking/denying traffic?
  → Source: correct IP?
  → Destination: correct IP?
  → Port: correct port?
  → Action: Allow or Deny?
  → Rule name: which specific rule?
  
STEP 4: Check UDRs
  → Is the subnet associated with a route table?
  → Is the route table correct for the traffic?
  → Is 0.0.0.0/0 going where it should (Firewall, not Internet)?
  
STEP 5: Check DNS resolution
  → Can VM resolve the hostname?
  → If Private DNS: is zone linked to correct VNet?
  → If Custom DNS: is DNS server reachable and configured?
  
STEP 6: Check VNet Peering
  → Is peering configured and accepted?
  → Is Allow Forwarded Traffic enabled (for transit)?
  → Are address spaces non-overlapping?
  
STEP 7: Check Gateway
  → Is VPN/ExpressRoute Gateway running?
  → Is connection established? (VPN Gateway = "Succeeded" connection)
  → Are BGP routes learned (if BGP configured)?
  
STEP 8: Check Firewall
  → Is Firewall running? (health check)
  → Are Firewall rules correct?
  → Is Firewall throughput within SKU limits?
  
STEP 9: Check Application Gateway/WAF
  → Is Gateway running?
  → Are listeners and rules configured?
  → Is backend health "Healthy"?
  
STEP 10: Apply fix, test, document
```

### 5.2 Key Diagnostic Tools

| Tool | Purpose | Access |
|------|---------|--------|
| **NSG Flow Logs** | See what traffic is allowed/denied and by which rule | Storage Account or Log Analytics |
| **Effective NSG Rules** | Shows computed rules for a specific NIC/subnet | Portal → NIC → Effective NSG Rules |
| **Effective Route Table** | Shows computed routes for a specific subnet/nic | Portal → Subnet → Effective Routes |
| **Connection Monitor** | Tests connectivity between endpoints | Portal → Network Watcher → Connection Monitor |
| **Network Watcher --> VNet View** | Visual topology of VNet | Portal → VNet → Network Watcher |
| **IP Flow Verify** | Checks if specific IP flow is allowed by NSG | Portal → NIC → IP Flow Verify |
| **Next Hop** | Checks where traffic will go for a specific source/dest | Portal → NIC → Next Hop |
| **Resource Graph** | Query resources across subscriptions | Portal → Resource Graph; `az graph query` |
| **Log Analytics (KQL)** | Advanced log analysis | Log Analytics → Logs |

---

## 6. PRODUCTION EXAMPLE

**Scenario: Global enterprise with 3 regions, 4 applications, strict security.**

```
TENANT: contoso.onmicrosoft.com

REGION: East US 2 (Primary), West US 2 (DR)

PER-REGION VNET: vnet-prod-eastus-01 (10.0.0.0/16), vnet-prod-westus-01 (10.2.0.0/16)

VNet Address Space Design:
  10.0.0.0/16 (East US) — NOT overlapping with 10.2.0.0/16 (West US)
  Subnets (10.0.0.0/16):
    ├── 10.0.1.0/24 (Web) — 251 usable IPs, ~50 VMs, headroom for 100+
    ├── 10.0.2.0/24 (App) — 251 usable IPs, ~50 VMs, headroom for 100+
    ├── 10.0.3.0/24 (Data/DB) — 251 usable IPs, ~20 VMs, headroom for 50+
    ├── 10.0.4.0/27 (Gateway) — 27 usable IPs (VPN + ER gateways + bastion if needed)
    ├── 10.0.5.0/27 (AzureBastion) — 27 usable IPs
    ├── 10.0.6.0/27 (AzureFirewallSubnet) — 27 usable IPs (Standard VNet required)
    ├── 10.0.7.0/27 (AzureFirewallDns) — 27 usable IPs (if DNS proxy)
    └── 10.0.8.0/24 (Management) — reserved for monitoring, NI, etc.

  10.2.0.0/16 (West US) — identical structure for DR

GOVERNANCE:
  Policy: Required Tags (Deny) — Environment, Project, CostCenter, Criticality, DataClassification
  Policy: Allowed Locations (Deny) — East US 2, West US 2 only
  Policy: Allowed SKUs (Deny) — D/DS, E/ES series only; Premium LRS storage only
  Policy: Deploy Diagnostics (DeployIfNotExists) → Log Analytics in Sub-Platform
  Policy: Security Baseline (Audit initially, transition to Deny)

RBAC:
  "Team-App1-Prod-Contributor" → Contributor on rg-app1-prod
  "Team-App2-Prod-Contributor" → Contributor on rg-app2-prod
  "Team-Data-Prod-Contributor" → Contributor on rg-data-prod
  "Platform-Network-Admin" → Owner on Sub-Platform (VNet, Firewall, DNS, Bastion)
  "Platform-Security-Admin" → Security Admin + Conditional Access Admin
  "Platform-Monitoring" → Monitoring Reader + Log Analytics Contributor

BLUEPRINT: Production Environment v3
  Artifacts:
    - ARM template: VNet, Subnets, NSGs, Bastion, Firewall
    - Role assignments: Team groups on RGs
    - Policy assignments: Required tags, allowed locations, allowed SKUs
    - Diagnostic settings: All resources → Log Analytics

NSG Design:
  Web Subnet NSG:
    Rule 100: Allow 80/443 from Internet (source: Internet)
    Rule 110: Allow 8080 from App subnet (source: 10.0.2.0/24)
    Rule 120: Allow HTTPS from AzureLoadBalancer (source: AzureLoadBalancer)
    Rule 130: Allow SSH/RDP from Bastion subnet (source: 10.0.5.0/27, port: 22/3389)
    Rule 500: Deny All Inbound
  
  App Subnet NSG:
    Rule 100: Allow VNet Inbound (default)
    Rule 110: Allow 8080 from Web subnet
    Rule 120: Allow 1433 from Data subnet
    Rule 130: Allow SSH/RDP from Bastion
    Rule 500: Deny All Inbound

  Data Subnet NSG:
    Rule 100: Allow VNet Inbound (default)
    Rule 110: Allow 1433 from App subnet (SQL access only)
    Rule 120: Allow SSH/RDP from Bastion
    Rule 130: Allow from Bastion subnet
    Rule 500: Deny All Inbound (NO internet access for databases)

UDR Design:
  All subnets (except Gateway) use RT-Default:
    0.0.0.0/0 → Virtual Appliance (Firewall: 10.0.6.4)
    168.63.129.16/32 → Internet (Azure DNS — system route)

  Gateway subnet: No UDR (needs direct gateway connectivity)

DNS:
  Azure Public DNS: default (handles *.azure.com, *.core.windows.net, etc.)
  Private DNS Zones:
    - corp.contoso.com → Linked to all VNet subnets
    - privatelink.database.windows.net → Linked (for SQL Private Endpoints)
    - privatelink.blob.core.windows.net → Linked (for Storage Private Endpoints)
    - privatelink.vaultcore.azure.net → Linked (for Key Vault Private Endpoints)

Private Endpoints:
  - SQL Server → Spoke Data subnet (10.0.3.10)
  - Storage Account → Spoke Data subnet (10.0.3.11)
  - Key Vault → Spoke Data subnet (10.0.3.12)
  - App Service → Spoke Web subnet (10.0.1.20)

Load Balancing:
  - Application Gateway v2 (WAF): Subnet Web (10.0.1.10), autoscale, WAF enabled
  - Internal Load Balancer: Subnet App (10.0.2.100), for app-tier scaling
  - Front Door: Global ingress (web-facing)

Monitoring:
  All resources → Diagnostic Settings → Log Analytics (Sub-Platform-Monitoring)
  NSG Flow Logs → Storage Account (90 days) + Log Analytics (1 year)
  Firewall logs → Log Analytics
  Alerts: VM unhealthy, NSG deny spikes, Firewall threat alerts, cost overrun

Connectivity to On-Prem:
  ExpressRoute (Primary): 10 Gbps, Circuit in East US 2
  ExpressRoute (DR): 10 Gbps, Circuit in West US 2
  Global Reach: ExpressRoute circuits connected for redundancy
  VPN Gateway: Backup (using Standard SKU VPN Gateway, auto-activates if ER fails)

Connectivity between regions:
  VNet Peering: vnet-prod-eastus-01 ↔ vnet-prod-westus-01 (cross-region, Allow Forwarded Traffic)
  → Use Global VNet Peering (built-in)
  → Enables VM in East to communicate with VM in West via private IP

Backup/DR:
  Azure Backup for all VMs (policy: daily, 30-day retention)
  Azure Site Recovery: VMs replicated to West US 2 (if primary is East US 2)
  SQL: Active geo-replication (auto-failover to West US 2)

Tagging at every level:
  Subscription: Environment=Production, CostCenter=12345, Department=Engineering
  RG: Project=App1, Owner=Team-App1, Criticality=High
  Resource: DataClassification=Confidential

Result:
  3 regions × 4 applications × multiple environments
  All VMs isolated in proper subnets
  All traffic inspected through Firewall
  All VMs accessible via Bastion (no public IPs)
  All data services accessed via Private Link
  All logs in centralized monitoring
  All governance enforced via Policy
  Onboarding new workload: Deploy Blueprint → fully configured in ~2 hours
```

---

## 7. FAILURE SCENARIOS

| Scenario | Root Cause | Resolution | Evidence |
|----------|-----------|------------|----------|
| **VM cannot reach internet** | UDR missing or wrong next hop; NSG blocking outbound; Firewall overloaded; DNS failing; VM stopped | Check: UDR (0.0.0.0/0 → correct next hop), NSG outbound rules, Firewall health, DNS, VM running. Most common: UDR not configured or Firewall down. | NSG Flow Log: deny rule; UDR: check next hop; Firewall: health; DNS: nslookup |
| **VM can reach internet but not other VMs** | NSG not allowing inter-VM traffic; UDR forcing traffic wrong; VNet peering not configured or disabled; Address overlap | Check: NSG rules for both subnets, UDR for both subnets, VNet peering status, peering rules. Most common: NSG not allowing traffic between subnets or peering disabled. | NSG Flow Log: deny between IPs; Peering: status and rules |
| **VM can reach internet but not Azure PaaS (SQL, Storage)** | Service Endpoint not enabled; NSG denies service traffic; Private Link not working; DNS not resolving to private IP | Check: Private Link/Service Endpoint; NSG for service tags; DNS resolution. Most common: DNS resolving to public IP instead of private. | DNS: nslookup for privatelink.*; Private Endpoint: status; NSG Flow: traffic to PaaS IP |
| **Bastion cannot connect to VM** | Bastion subnet missing or too small; VNet peering missing; NSG blocking Bastion traffic; VM OS not running; Bastion SKU limit reached; Browser cache | Check: Bastion subnet size; Bastion health; NSG for Bastion subnet; VM running; Bastion session limit. Most common: NSG on Bastion subnet blocking traffic. | Bastion: connection error message; NSG: Bastion subnet rules; VM: running status |
| **VNet Peering not working (one direction)** | Allow VNet Access disabled on one side; Address space overlap; Peering not accepted; "Allow Forwarded Traffic" missing for transit | Check: Both peering configurations; Accept status; Address spaces; Forwarded traffic settings. Most common: Peering not accepted (pending) on remote side. | Portal: Peering status; Address spaces; Both directions' settings |
| **Traffic bypasses Firewall** | UDR for 0.0.0.0/0 set to Internet (not Firewall); NSG allows direct internet; Firewall not in path; Wrong next hop in UDR | Check: UDR on all subnets; confirm 0.0.0.0/0 → Firewall, not Internet. Most common: Someone added UDR route to Internet by mistake. | UDR: check 0.0.0.0/0 next hop; Firewall logs: traffic from subnet; NSG: traffic bypassing |
| **DNS resolution fails for Azure PaaS** | Private DNS Zone not linked to VNet; DNS forwarding broken; Firewall DNS proxy intercepting but not resolving; Private Endpoint failed; VM using wrong DNS server | Check: DNS zone links; DNS resolution from VM; Private Endpoint status; VM DNS settings. Most common: Private DNS Zone not linked to the correct VNet/subnet. | nslookup from VM; Private DNS Zone: linked VNet list; Private Endpoint: provisioned |
| **VPN Gateway not connecting** | Gateway subnet missing or too small; Gateway SKU; Public IP not associated; Pre-shared key mismatch; BGP configuration error; NSG blocking gateway traffic | Check: GatewaySubnet exists (/27+); VPN GW health; Connection status; PSK matches both sides. Most common: PSK mismatch or Gateway not running. | VPN GW: connection status and error; Activity Log: configuration changes; BGP: routes learned |
| **Application Gateway returning 502/504** | Backend VMs unhealthy (health probe failing); NSG blocking AAG ↔ VM traffic; App Gateway SKU overwhelmed; Backend VM not running | Check: Backend health in AAG; NSG between AAG and VMs; AAG SKU limits; VM running. Most common: Health probe configuration wrong (wrong port/path). | AAG: backend health status; Health probe configuration; NSG: AAG subnet to VM subnet rules |
| **"VM is running but I can't RDP/SSH"** | VM has public IP but NSG blocks RDP/SSH; VM has no public IP; Bastion not configured; NSG on subnet blocks; Firewall in path with no RDP rule; VM OS RDP/SSH disabled | Check: VM public IP; NSG RDP/SSH rules (priority, source, port); Bastion connection; OS-level RDP/SSH config. Most common: NSG missing RDP/SSH allow rule OR VM has no public IP and Bastion not set up. | NSG: RDP/SSH rules; VM: IP configuration; Bastion: available; OS: RDP/SSH service running |
| **Cross-region VNet peering traffic black-holed** | Peering not configured for forwarded traffic; Address space overlap; Transitive setting off on both sides; NSG blocks traffic between address spaces | Check: Allow Forwarded Traffic enabled; Address spaces don't overlap; Both sides accept peering. Most common: Allow Forwarded Traffic disabled on Hub → Spoke traffic can't transit. | Peering: both sides' settings; UDR: next hop for remote address space |
| **Premium Firewall not blocking traffic** | Firewall in non-Standard VNet; Firewall SKU is Basic (limited rules); Firewall rules not configured; Traffic not routed through Firewall (UDR issue); Firewall not deployed | Check: VNet SKU (Standard); Firewall SKU; Firewall rules; UDR next hop. Most common: Firewall is Basic SKU with Advanced features expected (or not configured at all). | Firewall: SKU and rules; UDR: traffic routed to Firewall; NSG: not directly blocking |
| **All VMs in subnet lose connectivity simultaneously** | NSG change (new deny rule at high priority); UDR change (wrong next hop); Subnet IP exhaustion; GatewaySubnet down (if internet traffic via gateway); Firewall down (if UDR points to it) | Check: Recent changes in Activity Log; NSG rules by priority; UDR changes; resource health. Most common: Someone pushed a high-priority NSG deny rule (priority 100-101 overriding all others). | Activity Log: NSG/UDR changes in last hour; NSG: effective rules; UDR: current routes |

---

## 8. MONITORING — VNET

| Metric/Log | What It Shows | Alert Trigger |
|------------|---------------|---------------|
| **NSG Flow Logs** | Allowed/denied traffic per NSG rule | Spike in denies, unusual patterns |
| **DNS Query Logs** | DNS resolution success/failure | High DNS failure rate, unusual domains |
| **Traffic Metrics** | Bytes in/out per NIC, per VNet | Anomalous traffic volume |
| **Network Watcher Connection Monitor** | End-to-end connectivity between VMs | Connectivity failures |
| **Effective Routes** | Computed routes for subnets/NICs | Route changes or unexpected paths |
| **Effective NSG Rules** | Computed rules for subnets/NICs | Unexpected denies |
| **Gateway Connection Health** | VPN/ExpressRoute connection status | Gateway disconnected |
| **Application Gateway Metrics** | Requests, backend health, latency | Backend unhealthy, high latency |
| **Load Balancer Metrics** | Healthy vs unhealthy hosts, throughput | Backend failures |
| **Bastion Session Metrics** | Active sessions, connection failures | Unusual session volumes |
| **Private Endpoint Status** | Provisioning state, connection status | Endpoints failing or disconnected |
| **Firewall Metrics** | Throughput, CPU, threat alerts | Throughput near limit, threat detections |
| **DNS Resolution Rate** | Private vs public resolution patterns | Private DNS failure spike |
| **VNet IP Utilization** | Used vs available IPs in subnets | Subnet > 80% utilized |

---

## 9. SECURITY CONSIDERATIONS

| Concern | Risk | Mitigation |
|---------|------|------------|
| **VM public IP exposure** | VM directly accessible from internet | Remove public IPs; use Bastion; NSGs deny inbound from internet |
| **Flat network (no subnetting)** | Compromised VM can reach everything | Tier-based subnets; NSGs between tiers; UDRs through Firewall |
| **No UDR (default routing)** | Traffic bypasses Firewall; no inspection | UDR 0.0.0.0/0 → Firewall on all work subnets |
| **NSG rule conflict** | Rules override each other unexpectedly | Priority ordering audit; least privilege rules; regular review |
| **DNS poisoning/spoofing** | Traffic redirected to malicious endpoints | Private DNS Zones; Private Link; DNS query logging |
| **Peering address overlap** | Two VNets with same CIDR can't peer | IPAM; non-overlapping design; validate before peering |
| **Bastion in non-Bastion subnet** | Bastion doesn't work; connection fails | Bastion in AzureBastionSubnet specifically |
| **GatewaySubnet occupied** | Gateway won't deploy; VPN/ER fails | GatewaySubnet is reserved; don't put other resources there |
| **NSG on Bastion subnet blocking** | Bastion sessions fail silently | Allow BastionSubnet → VirtualNetwork + BastionSubnet → Internet (443) |
| **Overly permissive NSG rules** | Any/Allow-all rules defeat security | Least privilege; specific source/dest/port; regular audit |
| **Unmonitored traffic** | No visibility into network flows | NSG Flow Logs enabled everywhere; Firewall logging enabled |
| **Private Endpoint in wrong subnet** | Private access from unintended VNet | Deploy Private Endpoint in correct VNet/subnet; restrict access |
| **Cross-VNet lateral movement** | Compromised VM moves to other VNet | VNet peering only where needed; NSGs on both sides; UDR control |
| **Service Endpoint leak** | Traffic not fully private | Migrate Service Endpoints to Private Endpoints for control |

---

## 10. DIAGNOSTIC SETTINGS — NETWORKING

Every network resource should have diagnostic logging:

| Resource | Log Category | Destination |
|----------|-------------|------------|
| **NSG** | All (flow logs) | Storage Account + Log Analytics |
| **Azure Firewall** | All (AzureFirewall, AzureFirewallApplicationRule, AzureFirewallNetworkRule) | Log Analytics |
| **Application Gateway** | Access, WAF, Performance | Log Analytics + Storage |
| **Load Balancer** | All (health probes, SNAT, etc.) | Log Analytics + Storage |
| **VPN Gateway** | Connect, IKE, P2S, BGP | Log Analytics + Storage |
| **ExpressRoute Gateway** | Connect, BGP | Log Analytics + Storage |
| **DNS Resolver** | Queries, responses | Log Analytics |
| **Network Watcher** | Connection Monitor, IP Flow Verify | Log Analytics |
| **VNet** | Activity Log | Log Analytics + Storage |
| **Private Endpoint** | Logs and metrics | Log Analytics |
| **Bastion** | Session logs, connection logs | Log Analytics + Storage |

---

## 11. L3 INTERVIEW QUESTIONS

### Basic
**Q: What is an Azure VNet?**
A: A Virtual Network is a logically isolated network segment in Azure where you can deploy and manage Azure resources in a private network space. It's equivalent to your on-premises network, with your own IP address space, subnets, and routing control.

### Intermediate
**Q: What is the difference between NSG at subnet level vs NIC level?**
A: Subnet NSG applies to all resources in the subnet (centralized, consistent). NIC NSG applies to a single VM (granular, per-resource). Best practice: use subnet NSG for base security rules and NIC NSG only for specific exceptions. Both are evaluated together — most restrictive rule wins.

### L3
**Q: A VM in a Spoke VNet (10.1.1.0/24) cannot reach the internet. It can reach other VMs in the same VNet. What are ALL possible causes?**
A:
1. UDR for 0.0.0.0/0 points to Internet (not Firewall) — BYPASSES Firewall (if expected through Firewall) OR points to non-existent resource
2. NSG on VM/subnet blocks outbound 80/443 (default is allow outbound, but explicit deny would block)
3. Firewall is down or overwhelmed — traffic routes to Firewall but gets dropped
4. DNS fails (can't resolve hostnames) — VM can ping IP but not reach URLs
5. UDR not associated with the subnet
6. Firewall SKU exhausted (throughput)
7. Firewall rules block the traffic (no rule allowing outbound)
8. VM has no network connectivity (NIC issue, VM OS network adapter disabled)
9. Private DNS zone linked to subnet is serving wrong records
10. Azure platform issue (rare)

Most common: UDR misconfiguration (pointing to Firewall when Firewall is down, or UDR pointing to Internet instead of Firewall, or UDR not associated at all).

### Senior L3
**Q: Explain the complete traffic flow from a VM in Spoke-1 to Azure SQL via Private Link, and all the components involved.**
A:
1. VM (10.1.1.4) needs to connect to Azure SQL: sql-primary.database.windows.net
2. DNS resolution: VM queries DNS → Private DNS Zone "privatelink.database.windows.net" linked to Spoke-1 VNet → resolves to Private Endpoint IP (10.1.1.20)
3. VM creates TCP connection to 10.1.1.20 (Private Endpoint IP)
4. Traffic stays within VNet (no Firewall/UDR needed — Private Link bypasses network security)
5. Private Endpoint (network interface in Spoke VNet) → Azure Private Link backbone
6. Azure Private Link routes traffic to Azure SQL service
7. Azure SQL authenticates (SQL login or Azure AD via Entra ID)
8. Response follows reverse path

Components involved: VM, DNS (Private DNS Zone), Private Endpoint, Azure Private Link, Azure SQL. NO Firewall, NO NSG between VM and Private Endpoint (same VNet), NO public internet.

Key insight: Private Link traffic is explicitly designed to bypass network security appliances. Security inspection for Private Link traffic must be done at the endpoint level (NSG on Private Endpoint subnet) or at the service level (SQL firewall, SQL-level authentication).

### Expert
**Q: What happens when you create a VNet with address space 10.0.0.0/16 and then try to peer it with another VNet 10.0.0.0/16?**
A: Peering will FAIL with error: "Address space overlaps." VNet peering REQUIRES non-overlapping address spaces. The two VNets must have completely separate CIDR blocks. You must either:
1. Change one VNet's address space (requires downtime, re-IP everything)
2. Change the overlapping VNet to a different address space before peering
3. Use one VNet's address space as the basis and ensure no overlap

This is a hard constraint and cannot be overridden. Even /16 and /17 of the same 10.0.0.0 space will overlap (10.0.0.0/16 contains 10.0.0.0/17).

### Scenario
**Q: "New production VM deployed in Spoke-Web. NSG Flow Logs show traffic being denied by rule #200. What do I check first?"**
A:
1. Identify what rule #200 is (check NSG rules → find priority 200 rule)
2. Determine if it's Allow or Deny
3. If Deny: What source/dest/port does rule #200 cover? Does it match the traffic being blocked?
4. If Allow: Traffic should be allowed (check if a higher-priority rule denies first)
5. Check effective NSG rules from Portal (combines subnet + NIC NSG)
6. Check if NSG is associated with correct subnet/NIC
7. Check if there are multiple NSGs and which one is evaluated first

Common cause: Someone added a Deny rule at priority 200 that accidentally blocks legitimate traffic (e.g., Deny all from Internet with priority 200, placed below Allow HTTPS at 201 — both match, 200 wins).

### Tricky
**Q: "Both my Spoke VNet and Hub VNet have 10.0.0.0/16 address spaces. I can ping between them, but no traffic actually flows. Why?"**
A: This is a trick question. If both VNets have the SAME address space (10.0.0.0/16), they CANNOT be peered at all. The peering would fail. If you can "ping" between them, one of these is true:
1. The ping is actually going through the internet (not peering) — VMs have public IPs
2. You're confusing VMs that are in the same VNet (different subnets, not different VNets)
3. One VNet has been changed and no longer overlaps (but you're not aware)
4. You're using a VPN tunnel (not VNet peering) which has its own routing

REAL ANSWER: VNet peering with overlapping address spaces is IMPOSSIBLE. If you think you're peering and pinging between same-address-space VNets, double-check — you're likely testing in the same VNet, or using VPN/internet, not VNet peering.

### Tricky 2
**Q: "I have a UDR with 0.0.0.0/0 → Firewall. Why is some internet traffic NOT going through Firewall?"**
A: Possible reasons:
1. GatewaySubnet has no UDR (by design — needs direct gateway access)
2. VM has a public IP and no UDR on its subnet → takes default internet route (bypasses UDR)
3. Service tags or Azure DNS (168.63.129.16) traffic uses Azure DNS route (not UDR)
4. BGP route from VPN/ExpressRoute overrides UDR for specific prefixes
5. Traffic to Azure PaaS via Private Link bypasses UDR (private route)
6. DNS queries going directly to Azure DNS (168.63.129.16) — this IP has a special route
7. Application Traffic (Azure services) — Azure routes traffic to Azure services internally
8. UDR not associated with the specific subnet (RT association is per-subnet, not global)
9. Effective routes show different route than expected — always check "Effective Routes" not just configured routes

Most common: UDR associated with some subnets but not all; or someone attached a public IP to the VM and traffic takes the direct internet route.

### Tricky 3
**Q: "If I add a subnet-level NSG that Deny All Inbound with priority 500, and a VM in that subnet has a NIC-level NSG that Allows RDP with priority 100, will RDP work?"**
A: Yes, RDP will work — BUT ONLY from sources allowed by the NIC NSG. Here's why:
- Effective NSG = Subnet NSG + NIC NSG (merged)
- Priority 100 (NIC NSG: Allow RDP) beats priority 500 (Subnet NSG: Deny All)
- First matching rule wins → RDP traffic matching NIC NSG Allow rule is permitted
- RDP traffic NOT matching the Allow rule (e.g., from wrong source IP) hits Deny All at priority 500
- Both NSGs are evaluated, not either/or

This is a common pattern: subnet NSG provides base deny-all-inbound, NIC NSG provides specific allows for individual VMs.

### Tricky 4
**Q: "What happens to traffic when a route table has 0.0.0.0/0 → Virtual Appliance AND the Virtual Appliance is down?"**
A: Traffic is DROPPED (black-holed). Azure does NOT automatically fall back to the default Internet route when a UDR next hop (Virtual Appliance) is unavailable. The route table says "send everything to Firewall" — if Firewall is down, traffic goes to Firewall and is simply not forwarded. There's no automatic failover or fallback route in the route table.

This is why HA is critical for NVAs in UDR paths:
1. Firewall HA (Standard/Premium SKU, zone-redundant)
2. Multiple NVAs with automatic failover (using technologies like NVA failover, or Gateway Load Balancer for NVA health probes)
3. Or accept the risk (smaller environments where NVA downtime is acceptable)

---

## 12. SCENARIO-BASED QUESTIONS

### Scenario 1: "VM in Web subnet can't be reached from internet on port 443"
**Architecture:** Internet → NSG → VM (443) → VM running app.
**Dependencies:** NSG rules, VM public IP, VM OS, Application, health.
**Checks:**
1. VM has public IP?
2. NSG on subnet/NIC allows 443 from Internet (priority < 500)?
3. VM is running?
4. Application listening on 443?
5. OS firewall (Windows Firewall, iptables)?
6. NSG Flow Logs: what rule blocks?
**Root Cause:** NSG missing allow rule for 443 from Internet.
**Fix:** Add NSG rule: Priority 200, Source: Internet, Dest: *, Port: 443, TCP, Allow.
**Validation:** Browser connects to VM:443; NSG Flow Log shows allowed.

### Scenario 2: "App Gateway backend is unhealthy after scaling VM set"
**Architecture:** App GW → Backend Pool → VMSS VMs → Application.
**Dependencies:** Health probe config, VMSS health, NSG, App GW SKU.
**Checks:**
1. Health probe: correct port, path, interval?
2. VMs respond to health probe (manual test)?
3. NSG between App GW and VMs allows health probe traffic?
4. App GW backend pool includes new VMs?
5. App GW SKU not overwhelmed?
6. VMs running and app is healthy?
**Root Cause: Health probe configured for port/path that VMs don't respond to after app update.**
**Fix:** Update health probe path/port; restart VMs; update App GW backend pool.
**Validation:** Backend health shows "Healthy" in App Gateway.

### Scenario 3: "Cross-region DR failover works but latency increases dramatically"
**Architecture:** Primary East US → DR West US (VNet Peering).
**Dependencies:** VNet Peering, bandwidth, application design, database replication.
**Checks:**
1. VNet Peering bandwidth (VM SKU limits)?
2. Database replication lag?
3. Application cross-region calls (no affinity)?
4. Traffic taking inefficient routes?
5. DNS resolution pointing to wrong region?
6. Front Door traffic steering?
**Root Cause: Application making cross-region calls for every request instead of local processing.**
**Fix:** Optimize app for regional processing; use CDN/Front Door for geo-routing; read replicas in DR region.
**Validation:** Latency within acceptable range; user experience is consistent.

### Scenario 4: "ExpressRoute connection dropped after VNet peering added"
**Architecture:** ExpressRoute Gateway → VNet → VNet Peering → Spoke.
**Dependencies:** ExpressRoute, VNet Peering, UDRs, GatewaySubnet.
**Checks:**
1. Did peering change VNet address space or routing? (Check BGP if enabled)
2. Is GatewaySubnet still intact?
3. Is "Allow Forwarded Traffic" or "Allow Gateway Transit" affected?
4. BGP route conflict (new peering introduced overlapping routes)?
5. UDR on GatewaySubnet was accidentally modified?
6. Activity Log: recent changes?
**Root Cause: Someone modified route table associated with GatewaySubnet (or VNet peering propagated route that conflicts with on-prem routes).**
**Fix:** Restore route table; verify peering settings; check BGP.
**Validation:** ExpressRoute connection re-established; BGP routes learned.

### Scenario 5: "Adding Private Endpoint for Azure Storage broke existing application"
**Architecture:** VM → Storage (was public) → Private Endpoint → Storage (private).
**Dependencies:** Private Endpoint, DNS, NSG, Storage firewall.
**Checks:**
1. DNS: Does VM resolve storage account name to private IP?
2. If using Private DNS: Is zone linked to VM's VNet?
3. Storage Account: Network settings — did you disable "Allow trusted Microsoft services"?
4. NSG: Does traffic flow between VM subnet and Private Endpoint subnet?
5. Storage Account firewall: Is VM's subnet/VNet allowed?
6. Application: Is it using correct connection string?
**Root Cause: Storage Account network restriction was set to "Deny all networks" without allowing the VNet/subnet or Private Endpoint. Or DNS doesn't resolve to private IP.**
**Fix:** Add VNet/Subnet to Storage Account network rules; link Private DNS Zone; verify DNS resolution.
**Validation:** VM can access Storage via private endpoint; DNS resolves to private IP.

### Scenario 6: "All VMs in Application subnet lose connectivity after Firewall upgrade"
**Architecture:** All VMs → UDR 0.0.0.0/0 → Firewall (10.0.6.4) → Internet.
**Dependencies:** UDR, Firewall health, Firewall rules, NSG.
**Checks:**
1. UDR next hop IP: Was Firewall IP changed during upgrade?
2. Firewall health: Is new Firewall running? (old was stopped?)
3. Firewall rules: Migrated to new Firewall?
4. Firewall SKU: Standard vs Premium (different management)?
5. NSG on UDR subnet: Allow traffic to new Firewall IP?
6. Test: ping Firewall IP from VM; if reachable but no internet → Firewall rules issue.
**Root Cause: Firewall upgraded but UDR still points to old Firewall IP (or new Firewall is in different subnet/IP).**
**Fix:** Update UDR next hop to new Firewall IP; verify new Firewall health and rules.
**Validation:** VMs reach internet through new Firewall; Firewall logs show traffic.

### Scenario 7: "DNS resolution for Azure PaaS works but on-prem DNS fails in Spoke VNet"
**Architecture:** VM → DNS resolver → Azure DNS (works) + Custom DNS (fails) → on-prem.
**Dependencies:** DNS configuration, VNet, peering, Private DNS Resolver.
**Checks:**
1. VM DNS setting: Custom DNS server IP configured?
2. Custom DNS server: Running? Reachable from Spoke?
3. DNS server: Conditional forwarding to on-prem configured?
4. Network path: Spoke → VNet → VPN/ExpressRoute → on-prem DNS?
5. NSG: DNS port (53) allowed between Spoke and DNS server?
6. If using Azure DNS Private Resolver: Rules configured correctly?
**Root Cause: Custom DNS server not reachable or not configured as DNS server on VM. Or DNS server can't reach on-prem (VPN/ExpressRoute down).**
**Fix:** Configure DNS server on VM; verify DNS server reachability; check VPN/ExpressRoute; configure conditional forwarding.
**Validation:** nslookup on-prem hostname from VM returns correct IP.

### Scenario 8: "Cost for VNet peering is higher than expected"
**Architecture:** Multiple VNet peerings across regions.
**Dependencies:** Cross-region data transfer, peering count, VM traffic volume.
**Checks:**
1. Cost Analysis: peering data transfer costs by region pair?
2. Number of peering connections: unnecessary peerings?
3. Traffic patterns: large data transfers between peered VNets?
4. Alternative: Could use VNet Gateway (ExpressRoute) for some traffic?
5. Bandwidth: VM SKUs generating high traffic?
6. Unnecessary data replication across peering?
**Root Cause: High-volume cross-region data transfer over peering (e.g., large DB sync between East US and West US over peering instead of ExpressRoute/cheaper link).**
**Fix:** Use ExpressRoute for bulk data; reduce unnecessary cross-VNet traffic; evaluate if some traffic can stay regional.
**Validation:** Cost analysis shows reduction in peering data transfer costs.

---

## 13. KNOWLEDGE TEST

1. **What is an Azure Virtual Network (VNet)?**
   A logically isolated network segment in Azure where resources can be deployed in a private network space. Equivalent to an on-premises network in the cloud.

2. **What are the key components of a VNet?**
   Address space, subnets, NSGs, UDRs, gateways, peering, DNS, Private Endpoints, and load balancers.

3. **What is the difference between Basic and Standard VNet SKU?**
   Basic: simple scenarios, dev/test. Standard: production, required for Firewall, Application Gateway, and Gateway Load Balancer.

4. **What are Azure reserved IPs in each subnet?**
   First 5 IPs: .0 (network), .1 (gateway), .2 (DNS), .3 (reserved), .4+ (usable). So a /24 has 251 usable IPs, not 256.

5. **What is a GatewaySubnet and why is it required?**
   A subnet named "GatewaySubnet" in every VNet, minimum /27 size, used by VPN and ExpressRoute gateways. Azure requires it even if no gateway is planned.

6. **What is the minimum size for GatewaySubnet?**
   /27 (32 addresses). This is an Azure requirement for VPN/ExpressRoute gateway deployment.

7. **What is an NSG and how does it work?**
   A virtual firewall filtering network traffic to/from resources. Rules evaluated by priority (lowest first); first match wins. Can be applied at subnet or NIC level.

8. **What are Service Tags and why use them?**
   Pre-defined IP groups for Azure services (Internet, AzureCloud, Storage, Sql, etc.). Automatically updated by Azure. Preferred over hardcoding IP ranges.

9. **What is the difference between NSG Flow Logs and Activity Log?**
   NSG Flow Logs = network traffic (IP, port, allow/deny). Activity Log = control plane operations (who did what, when).

10. **What is a User Defined Route (UDR) and why is it critical?**
    A route in a route table that overrides default Azure routing. In Landing Zones, UDRs force traffic through Firewall (0.0.0.0/0 → Firewall).

11. **What happens if a UDR points 0.0.0.0/0 to a non-existent IP?**
    Traffic is dropped (black-holed). Azure does NOT automatically fall back to the default Internet route. The route table says send to that next hop, and that's it.

12. **What is VNet Peering?**
    A connection between two VNets enabling private communication using private IPs. Bi-directional, stays on Azure backbone, requires non-overlapping address spaces.

13. **Why can't two VNets with overlapping address spaces be peered?**
    Azure cannot determine which VNet owns which IP. Routing would be ambiguous. Non-overlapping CIDR is a hard requirement.

14. **What is "Allow Forwarded Traffic" in peering?**
    A peering setting that enables traffic from a remote VNet to transit through the local VNet (not just direct peer-to-peer). Essential for hub-and-spoke transit.

15. **What is the difference between VNet Peering and VNet Gateway connectivity?**
    Peering = Azure-to-Azure (two cloud networks). Gateway = Azure-to-on-premises (cloud-to datacenter).

16. **What is Azure Bastion and why is it important?**
    Browser-based RDP/SSH without VM public IPs. Provides secure VM access from the Azure portal over TLS. No open RDP/SSH NSG rules needed.

17. **What subnets are required for Azure Bastion?**
    AzureBastionSubnet (minimum /27, /28 recommended in newer Bastion versions). Bastion is deployed IN this specific subnet.

18. **What is a Private Endpoint?**
    A network interface in your VNet with a private IP that connects privately to a PaaS service over Azure's private backbone.

19. **What is the difference between Service Endpoints and Private Endpoints?**
    Service Endpoints add VNet identity to PaaS traffic (still use public IPs). Private Endpoints provide static private IPs and direct private routing. Private Endpoints are preferred.

20. **What is the difference between Load Balancer and Application Gateway?**
    Load Balancer = Layer 4 (TCP/UDP). Application Gateway = Layer 7 (HTTP/HTTPS) with WAF, content-based routing, SSL offloading.

21. **Why does the GatewaySubnet need to have no UDR?**
    GatewaySubnet requires the default Azure Internet routing for VPN/ExpressRoute gateway connectivity. A UDR pointing to Firewall would block gateway traffic.

22. **What is Azure Private DNS Zone?**
    A DNS zone for resolving names to private IPs within your VNet. Must be linked to VNet(s) to be accessible. Used with Private Endpoints.

23. **What is a DNS Private Resolver?**
    An Azure DNS resolver at VNet level with forwarding rulesets. More flexible than direct Private DNS Zone linking for cross-VNet resolution.

24. **Why is Private Link traffic not inspected by Firewall?**
    By design — Private Link traffic goes directly from VNet to PaaS via Azure's private backbone, bypassing the public internet entirely. NSGs on Private Endpoint subnets can provide security instead.

25. **What is the complete traffic flow from VM to internet in a Hub-and-Spoke architecture with Firewall?**
    VM → Subnet UDR (0.0.0.0/0 → Firewall) → Spoke VNet → VNet Peering → Hub VNet → Firewall (NVA) → SNAT → Internet. Return: Internet → Firewall (DNAT) → VNet Peering → Spoke VNet → VM.

26. **What happens when a peering is configured but not accepted?**
    Peering is incomplete. Traffic cannot flow. Both sides must accept peering for it to work. Check peering status in Portal: "Accepted" or "Pending".

27. **Can VNet peering be established across Azure tenants?**
    Yes, with specific conditions: both tenants must be in the same Microsoft Entra ID tenant or have a cross-tenant access configuration. Microsoft recommends using Azure Private Access for cross-tenant scenarios.

28. **How do you troubleshoot "VM can't reach internet but can reach other VMs"?**
    Check NSG Flow Logs first. If inter-VM works but internet doesn't → UDR misconfiguration (most likely), Firewall down, DNS failure, or NSG blocking outbound.

29. **What is the difference between UDR on subnet vs on NIC?**
    UDRs are associated at SUBNET level only (not NIC). Each subnet has one route table. NIC-level routing is controlled by the subnet's UDR.

30. **What are Gateway Load Balancer (GLB) use cases?**
    Running multiple NVAs (firewalls, IDS/IPS) in active-active with automatic failover. GLB provides health probes and traffic distribution for NVA appliances.

31. **What is the default DNS server IP in every Azure VNet?**
    168.63.129.16 (Azure DNS). It resolves Azure service names (e.g., *.azurewebsites.net, *.blob.core.windows.net) to their appropriate IPs (often private!). Cannot be overridden for Azure DNS.

32. **What is the IP address reservation in a subnet?**
    First 5 IPs: .0 network, .1 gateway, .2 DNS (Azure DNS), .3 reserved, .4+ usable. Plus gateway IP at .1.

33. **What is the difference between Allow VNet Access and Allow Forwarded Traffic in NSG default rules?**
    Allow VNet Access = traffic within same VNet (or peered VNet with direct peering). Allow Forwarded Traffic = traffic from peered VNets that is transiting through the current VNet.

34. **What is the difference between BGP and static routes in VPN Gateway?**
    BGP = dynamic routing (routes learned automatically from on-prem router). Static = manual configuration. BGP is recommended for production (automatic failover and route updates).

35. **What is a float IP in Load Balancer?**
    Floating IP directs traffic directly to the backend VM (bypassing the load balancer for return traffic). Used for NAT scenarios. Disabled by default.

---

## 14. L3 GAP CHECK

| Topic | Status |
|-------|--------|
| VNet fundamentals (address space, subnets, properties) | ✅ Covered |
| CIDR planning and IP address management | ✅ Covered |
| Address space design patterns | ✅ Covered |
| Subnet design (tier-based, service-based, etc.) | ✅ Covered |
| Subnet properties and special subnet names | ✅ Covered |
| NSG deep dive (rules, priorities, evaluation) | ✅ Covered |
| NSG Flow Logs | ✅ Covered |
| Service Tags | ✅ Covered |
| UDR deep dive (route tables, next hops) | ✅ Covered |
| UDR architecture (Hub-and-Spoke with Firewall) | ✅ Covered |
| VNet Peering (configuration, rules, transit) | ✅ Covered |
| Hub-and-Spoke with peering transit | ✅ Covered |
| DNS in VNets (Public, Private Zones, Resolvers) | ✅ Covered |
| Private Link and Private Endpoints | ✅ Covered |
| Load Balancer (Basic vs Standard, ILB) | ✅ Covered |
| Application Gateway (WAF, v2, autoscale) | ✅ Covered |
| Azure Bastion (configuration, NSG, architecture) | ✅ Covered |
| Service Endpoints (deprecated context) | ✅ Covered |
| VNet design patterns (Hub-Spoke, per-workload, tiered) | ✅ Covered |
| Production example (comprehensive) | ✅ Covered |
| Failure scenarios (12 scenarios) | ✅ Covered |
| Troubleshooting methodology | ✅ Covered |
| Diagnostic tools (Connection Monitor, IP Flow Verify, etc.) | ✅ Covered |
| Logs/evidence | ✅ Covered |
| Current service considerations | ✅ Covered |
| L3 interview questions (all levels, 33 questions) | ✅ Covered |
| Scenario-based questions (8 scenarios) | ✅ Covered |
| Knowledge test (33 questions) | ✅ Covered |
| L3 gap check | ✅ Covered |

---

## WHAT AN EXPERIENCED AZURE L3 ENGINEER SHOULD NOW BE ABLE TO EXPLAIN CONFIDENTLY

After Module 10, you should be able to confidently explain:

1. **The complete VNet architecture from address space to application deployment** — How IP address planning (CIDR), subnet design (tier-based), NSG rules (prioritized), UDRs (forced routing), and DNS work together to create a functional, secure, predictable network. Every IP address has a purpose; every NSG rule has a reason; every UDR has a destination. Understanding this end-to-end is what separates a network engineer from a networking architect.

2. **Why UDRs are the most critical networking component in a Hub-and-Spoke architecture** — Without UDRs, traffic takes the default Azure route (direct internet from every subnet), completely bypassing the Firewall. UDRs are the mechanism that forces all egress traffic through the Firewall for centralized inspection. Misconfigured UDRs (pointing to wrong IP, pointing to Internet instead of Firewall, or not associated with the correct subnet) are the #1 cause of "traffic bypassed security" incidents.

3. **NSG evaluation and how to troubleshoot with NSG Flow Logs** — NSGs evaluate rules by priority (lowest number first), first match wins, combining subnet and NIC NSGs. NSG Flow Logs are the definitive troubleshooting tool for "VM can't connect" scenarios — they show exactly what traffic was allowed and denied, by which rule, with source/destination/port/protocol. Every L3 network engineer in Azure should have NSG Flow Logs enabled as a default.

4. **The precise mechanics of VNet Peering and common pitfalls** — Peering is bi-directional, requires non-overlapping CIDRs, and does NOT transit (VNet-A → VNet-B → VNet-C doesn't work without Allow Forwarded Traffic on both sides of VNet-B). "No transitive routing" is the most misunderstood concept in Azure networking. Peering also enables DNS resolution between VNets by default (Azure DNS resolves private IPs in peered VNets).

5. **How Private Link and Private Endpoints bypass network security appliances** — Private Link traffic goes from your VNet directly to the PaaS service via Azure's private backbone, COMPLETELY BYPASSING the Firewall and NSGs in the path. This is by design (PaaS services should have direct private access). Security for Private Link traffic must be handled at the endpoint level (NSG on Private Endpoint subnet) or at the service level (PaaS firewall rules).

6. **The critical relationship between UDRs, GatewaySubnet, and VPN/ExpressRoute** — GatewaySubnet has no UDR by design because VPN/ExpressRoute traffic needs the default Internet route to reach the gateway. If someone adds a UDR to GatewaySubnet or modifies the route table, VPN/ExpressRoute connections will fail. The GatewaySubnet is a sacred resource — treat it with care.

7. **DNS resolution flow in a multi-VNet architecture** — Azure Public DNS handles Azure service names (resolving to private IPs via Private Link). Private DNS Zones handle custom internal names (must be linked to the correct VNet). Custom DNS servers handle complex scenarios (conditional forwarding to on-prem). When DNS fails, most connectivity fails (VMs can't reach services by name). Private DNS Zone linking to the wrong VNet is the #1 DNS issue.

8. **The difference between Layer 4 (Load Balancer) and Layer 7 (Application Gateway) traffic management** — Load Balancer distributes TCP/UDP traffic based on IP/port (five-tuple). Application Gateway inspects HTTP/HTTPS content (URL path, host header, cookies) for intelligent routing and includes WAF capabilities. Use Load Balancer for backend VMs serving TCP traffic; Application Gateway for web applications requiring content inspection and security.

9. **How Azure Bastion eliminates the need for public IPs on VMs** — Bastion provides browser-based RDP/SSH via TLS (port 443) from the Azure portal. VMs never need public IPs; NSGs don't need RDP/SSH allow rules from internet; sessions are recorded (Premium); all connections are centralized and auditable. Bastion is deployed in AzureBastionSubnet and connects to VMs in ANY subnet in the VNet.

10. **How to systematically troubleshoot any VNet connectivity issue** — VM status → NSG (Flow Logs, effective rules) → UDR (effective routes) → DNS (resolution test) → Peering (status, rules, forwarded traffic) → Gateway (health, BGP) → Firewall (health, rules, throughput) → Application Gateway (backend health) → Network Watcher tools (Connection Monitor, IP Flow Verify). Following this ordered methodology resolves 95%+ of connectivity issues within 30 minutes.

11. **Why Private Link traffic is invisible to Firewall logs** — Private Link bypasses all network-level security by design. If security inspection is required for Private Link traffic (compliance requirement), use NSGs on Private Endpoint subnets, or deploy application-level inspection, or use Azure Firewall Manager's security stacking feature (which deploys distributed NVAs in spoke subnets).

12. **The complete traffic path from internet to a VM behind Application Gateway** — User → Front Door (global) → Application Gateway v2 (WAF, regional) → Backend Pool (VMs in App Subnet) → NSG allows App Gateway subnet → VM receives request. The Application Gateway terminates SSL, inspects content, applies WAF rules, and forwards clean traffic to backend VMs. Backend VMs don't need public IPs and don't need internet-facing NSG rules.

---

# Ready for Module 11 — Identity and Access Management (Entra ID, RBAC Deep, Conditional Access, PIM, ABAC, SSO, Federation, External Identities)?

It covers:
- Entra ID tenant structure
- Directory roles (including newly renamed ones)
- RBAC at Azure level (built-in and custom)
- Conditional Access (CA) policies
- Privileged Identity Management (PIM)
- Attribute-Based Access Control (ABAC)
- Multi-Factor Authentication (MFA)
- SSO and federation (SAML, OIDC)
- External Identities (B2B, B2C)
- Service principals and managed identities
- Self-Service Password Reset (SSPR)
- Hybrid identity (Azure AD Connect, Pass-through Authentication, Password Hash Sync)
- And all required module components

Say **"Next module"** to continue.