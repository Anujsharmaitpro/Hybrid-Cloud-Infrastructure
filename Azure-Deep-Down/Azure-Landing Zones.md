# MODULE 9 — LANDING ZONES (Platform, Application, Management, Identity, Connectivity, Security, Governance) — L3 DEPTH

---

## 1. CONCEPT

**Azure Landing Zones** are a **structured architecture pattern** for organizing Azure resources into a governed, scalable, and secure multi-terabyte cloud environment. They define how subscriptions, resource groups, networking, security, monitoring, and identity are organized to support enterprise workloads.

> **The single most important idea:** A Landing Zone is not a single resource or service — it is an **entire architectural framework** that answers the question: "How do we structure our Azure estate so that hundreds of workloads can be deployed securely, governably, and operably across multiple subscriptions?"

**Why Landing Zones exist:**

```
WITHOUT LANDING ZONES:
  → Resources deployed haphazardly across subscriptions
  → No consistent networking model (each team builds their own VNet)
  → No centralized monitoring or logging
  → Security policies inconsistent across workloads
  → Shared services duplicated by each team
  → No clear ownership model (who owns what subscription?)
  → Cost tracking is impossible across 500+ resources
  → Onboarding new teams takes weeks (they have to "figure it out")
  → Compliance audits require checking every resource individually

WITH LANDING ZONES:
  → Standardized subscription structure with clear ownership
  → Consistent hub-and-spoke networking model
  → Centralized monitoring, logging, and alerting
  → Security controls applied uniformly (NSGs, Firewall, Defender)
  → Shared services in dedicated hub VNet (no duplication)
  → Clear governance: Policy, RBAC, Blueprints applied at MG level
  → Cost tracking via tags and MG hierarchy
  → New teams deploy in <1 day (they use the established pattern)
```

**The core metaphor:**
```
Landing Zone = The "airport" for Azure workloads

  Landing Zone (Airport):
    ├── Runway = Connectivity (how traffic enters/exits)
    ├── Control Tower = Management (monitoring, logging, governance)
    ├── Terminal = Platform (shared services, DNS, firewall)
    ├── Gates = Subscriptions (each subscription is a gate to a workload)
    ├── Hangars = Resource Groups (organized by function)
    ├── Aircraft = Applications (workloads deployed by teams)
    ├── Security = Security services (Defender, NSG, Firewall)
    └── Identity = Who can access what (Entra ID, Directory roles)
```

---

## 2. ARCHITECTURE

```
┌───────────────────────────────────────────────────────────────────────────┐
│                         TENANT / LANDING ZONE                             │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                    MANAGEMENT GROUP: CORP-ROOT                      │  │
│  │                                                                       │  │
│  │  ┌─────────────────────────────────────────────────────────────┐    │  │
│  │  │        MG-SERVICES (Subscription for shared services)        │    │  │
│  │  │                                                                │    │  │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │    │  │
│  │  │  │ Hub VNet    │  │ Log Analytics│  │ Azure Bastion       │ │    │  │
│  │  │  │ (Shared Net)│  │ Workspace   │  │ (Jumpbox)           │ │    │  │
│  │  │  │             │  │ (Centralized│  │                     │ │    │  │
│  │  │  │ ┌─────────┐ │  │  Logs)      │  │ ┌─────────────────┐ │ │    │  │
│  │  │  │ │ Firewall│ │  │             │  │ │ Azure Bastion   │ │ │    │  │
│  │  │  │ │ Azure   │ │  └─────────────┘  │ │                 │ │ │    │  │
│  │  │  │ │ Firewall│ │                    │ └─────────────────┘ │ │    │  │
│  │  │  │ │ Private │ │  ┌─────────────┐  │ ┌─────────────────┐ │ │    │  │
│  │  │  │ │ DNS     │ │  │ Azure AD DS │  │ │ Managed Identity│ │ │    │  │
│  │  │  │ │ (Private│ │  │ (if needed) │  │ │ Vault           │ │ │    │  │
│  │  │  │ │ Zone)   │ │  └─────────────┘  │ │ (Secrets/Keys)  │ │ │    │  │
│  │  │  │ └─────────┘ │                    │ └─────────────────┘ │ │    │  │
│  │  │  └─────────────┘  └─────────────┘  └─────────────────────┘ │    │  │
│  │  │                                                                │    │  │
│  │  └─────────────────────────────────────────────────────────────┘    │  │
│  │                                                                       │  │
│  │  ┌─────────────────────────────────────────────────────────────┐    │  │
│  │  │        MG-PRODUCTION (Subscription: Sub-Prod-West)           │    │  │
│  │  │                                                                │    │  │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │    │  │
│  │  │  │ Spoke VNet  │  │ Spoke VNet  │  │ Spoke VNet          │ │    │  │
│  │  │  │ (App1-Prod) │  │ (App2-Prod) │  │ (Data-Prod)         │ │    │  │
│  │  │  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────────────┐ │ │    │  │
│  │  │  │ │ VMs     │ │  │ │ VMs     │ │  │ │ SQL/Storage      │ │ │    │  │
│  │  │  │ │ App GW  │ │  │ │ App GW  │ │  │ │ VMs              │ │ │    │  │
│  │  │  │ │ NSGs    │ │  │ │ NSGs    │ │  │ │ NSGs             │ │ │    │  │
│  │  │  │ └─────────┘ │  │ └─────────┘ │  │ └─────────────────┘ │ │    │  │
│  │  │  └─────────────┘  └─────────────┘  └─────────────────────┘ │    │  │
│  │  │                                                                │    │  │
│  │  │  Resource Groups: rg-app1-prod, rg-app2-prod, rg-data-prod    │    │  │
│  │  │  RBAC: Team-App1-Contributor, Team-App2-Contributor, etc.     │    │  │
│  │  │  Policy: Security Baseline, Required Tags, Allowed SKUs        │    │  │
│  │  │  Diagnostics: All resources → Log Analytics in MG-Services     │    │  │
│  │  └─────────────────────────────────────────────────────────────┘    │  │
│  │                                                                       │  │
│  │  ┌─────────────────────────────────────────────────────────────┐    │  │
│  │  │        MG-DEVELOPMENT (Subscription: Sub-Dev-A)              │    │  │
│  │  │  (Same structure as Production but relaxed policies)         │    │  │
│  │  └─────────────────────────────────────────────────────────────┘    │  │
│  │                                                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  CONNECTIVITY LAYER                                                  │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐     │  │
│  │  │ ExpressRoute │  │  VPN Gateway │  │  Virtual WAN (Hub)    │     │  │
│  │  │ (Dedicated)  │  │  (Fallback)  │  │  (Transit/Peering)   │     │  │
│  │  └──────────────┘  └──────────────┘  └──────────────────────┘     │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────┘
```

**The seven landing zones within a single architecture:**

```
┌───────────────────────────────────────────────────────────────────┐
│                                                                  │
│  ┌─ PLATFORM LANDING ZONE ─┐                                      │
│  │  Shared infrastructure:   │                                    │
│  │  Hub VNet, Firewall, DNS │                                    │
│  │  Azure Bastion, Vault    │                                    │
│  └──────────────────────────┘                                      │
│                                                                  │
│  ── CONNECTS TO ──                                                 │
│                                                                  │
│  ┌─ CONNECTIVITY LANDING ZONE ─┐                                  │
│  │  ExpressRoute, VPN, vWAN     │                                  │
│  │  (How the estate connects   │                                  │
│  │   to on-prem and internet)  │                                  │
│  └─────────────────────────────┘                                  │
│                                                                  │
│  ┌─ SECURITY LANDING ZONE ─┐                                      │
│  │  Defender, NSG, Firewall │                                     │
│  │  Key Vault, Managed ID   │                                     │
│  │  Security policies       │                                     │
│  └──────────────────────────┘                                      │
│                                                                  │
│  ┌─ IDENTITY LANDING ZONE ─┐                                      │
│  │  Entra ID, Directory     │                                     │
│  │  Roles, PIM, Conditional │                                     │
│  │  Access, Groups          │                                     │
│  └──────────────────────────┘                                      │
│                                                                  │
│  ┌─ MANAGEMENT LANDING ZONE ─┐                                    │
│  │  Log Analytics, Alerts,    │                                   │
│  │  Monitoring, Automation,   │                                   │
│  │  Cost Management           │                                   │
│  └────────────────────────────┘                                   │
│                                                                  │
│  ┌─ GOVERNANCE LANDING ZONE ─┐                                   │
│  │  Policy, RBAC, Blueprints, │                                  │
│  │  Management Groups, Tags   │                                  │
│  └─────────────────────────────┘                                  │
│                                                                  │
│  ┌─ APPLICATION LANDING ZONE ─┐                                  │
│  │  Workload VMs, Apps,       │                                   │
│  │  Databases, Storage        │                                   │
│  │  (In Spoke VNets)          │                                   │
│  └────────────────────────────┘                                  │
│                                                                  │
└───────────────────────────────────────────────────────────────────┘
```

---

## 3. COMPONENTS — DETAILED

### 3.1 PLATFORM LANDING ZONE

The **Platform Landing Zone** houses all **shared infrastructure services** that multiple workloads depend on. It is typically deployed in a **Hub VNet** within a dedicated subscription (often called "Shared Services" or "Platform").

**Shared services typically in the Platform Landing Zone:**

| Service | Purpose |
|---------|---------|
| **Azure Firewall** | Centralized egress/ingress traffic filtering for all VNets |
| **Azure DNS (Private)** | Private DNS resolution for internal services |
| **Azure Bastion** | Secure RDP/SSH access to VMs (no public IPs needed) |
| **Azure Key Vault** | Centralized secrets, keys, certificates |
| **Log Analytics Workspace** | Centralized log collection |
| **Azure AD DS** (if needed) | Domain services for legacy applications |
| **Managed Identity Vault** | Certificate/secret management for MIs |
| **Private Link Endpoints** | Private access to PaaS services |

**Hub-and-Spoke Architecture:**

```
Hub VNet (Platform Landing Zone):
  ├── Azure Firewall (subnet: AzureFirewallSubnet)
  ├── Gateway Subnet (for VPN/ExpressRoute Gateway)
  ├── AzureBastionSubnet (for Bastion)
  ├── AzureFirewallDns (subnet for DNS)
  └── User-defined Routes (UDRs) pointing traffic through Firewall

Spoke VNets (Application Landing Zone):
  ├── App1 VNet
  ├── App2 VNet
  ├── Data VNet
  └── Each Spoke has peerings to Hub VNet

Traffic Flow:
  Spoke → UDR → Hub VNet → Firewall → Internet/on-prem
  Spoke ↔ Spoke via Hub VNet (transit through peering)
  Spoke → Private endpoint → PaaS (bypass Firewall via Private Link)
```

**Key Hub VNet Subnets:**

| Subnet | Purpose |
|--------|---------|
| `AzureFirewallSubnet` | Azure Firewall (must be this exact name) |
| `GatewaySubnet` | VPN/ExpressRoute Gateway |
| `AzureBastionSubnet` | Azure Bastion (must be at least /27) |
| `AzureFirewallDns` | Firewall DNS (if using Private DNS) |
| `ManagedIdentitySubnet` | Managed Identity (if used for Firewall) |
| `UserDefinedRoutes` | Routes to next hop (Firewall) |

**Azure Firewall in Hub VNet:**
```
Firewall acts as the central NVA (Network Virtual Appliance):
  → All egress traffic from Spokes flows through Firewall
  → Firewall applies application rules (FQDN filtering) and network rules (IP/CIDR)
  → Threat intelligence enabled (alert and deny on known malicious IPs)
  → TLS inspection (if enabled via Firewall Manager)
  → DNAT rules for inbound services exposed through Firewall

Without Firewall:
  → Each Spoke would need its own NVA → cost and complexity explosion
  → Central visibility and control is lost
```

**DNS Architecture:**
```
Azure DNS Private Zones (in Hub VNet):
  ├── privatelink.azurewebsites.net (for Private Link)
  ├── privatelink.database.windows.net (for Azure SQL)
  ├── privatelink.blob.core.windows.net (for Storage)
  ├── corp.contoso.com (internal corporate)
  └── app1.contoso.com (application-specific)

DNS Resolution Flow:
  VM in Spoke → queries privatelink.blob.core.windows.net
  → Private DNS zone resolves to Private Endpoint IP
  → Traffic goes directly to Storage via Private Link (bypasses Firewall)

Azure Firewall DNS Proxy (if enabled):
  → All DNS queries from Spokes go to Firewall
  → Firewall forwards to Azure DNS or custom DNS
  → Enables FQDN-based filtering (Firewall rules can filter by domain)

Alternative: Azure Private DNS Resolver (VNet level)
  → DNS resolution at VNet level (not through Firewall)
  → Firewall rules still apply for traffic, but not for DNS
```

**Azure Bastion:**
```
Bastion in Hub VNet provides:
  → Browser-based RDP/SSH to VMs in ANY connected VNet (Hub + Spokes)
  → VMs do NOT need public IPs
  → No NSG rules needed for RDP/SSH (Bastion handles it)
  → All sessions are TLS-encrypted (port 443)
  → Session recording available (Auditing)
  → Generational VMs: RDP and SSH support
  → Session tunneling: connect without opening ports

Without Bastion:
  → VMs need public IPs (security risk)
  → NSGs must allow RDP/SSH from internet (or specific IPs)
  → No centralized access management
  → No session recording
```

---

### 3.2 APPLICATION LANDING ZONE

The **Application Landing Zone** is where **workloads** (VMs, App Services, databases, storage) are deployed. Each application or workload gets its own **Spoke VNet** and dedicated **Resource Group(s)**.

**Workload structure in a Spoke VNet:**

```
Spoke VNet (e.g., App1-Prod):
  ├── Subnet: Web (App VMs, Web Servers)
  │   ├── NSG: Allow 80/443 from Hub (Firewall) and Internet
  │   └── VMs: App1-Web-01, App1-Web-02
  ├── Subnet: App (Application VMs / API)
  │   ├── NSG: Allow 8080 from Web subnet only
  │   └── VMs: App1-API-01
  ├── Subnet: Data (Database VMs)
  │   ├── NSG: Allow 1433 from App subnet only (no internet)
  │   └── VMs: App1-DB-01 (SQL Server)
  ├── Subnet: Management (Bastion, NSG management)
  │   └── Azure Bastion (if per-spoke, or use Hub Bastion)
  └── Subnet: AzureBastionSubnet (if per-spoke)
```

**NSG Flow Log:**
```
NSGs in each Spoke produce flow logs:
  → Sent to Log Analytics workspace (centralized in Platform)
  → Or sent to Storage Account
  → Used for:
    - Troubleshooting connectivity
    - Security analysis (unusual traffic patterns)
    - Compliance verification
    - Network visualization

Azure Firewall logs also flow to central Log Analytics:
  → All traffic decisions logged
  → Threat intelligence alerts
  → FQDN filtering results
  → TLS inspection logs (if enabled)
```

**Application Landing Zone patterns:**

| Pattern | Description |
|---------|-------------|
| **Single workload per Spoke** | One app = one Spoke VNet (cleanest isolation) |
| **Tier-based spokes** | Web tier, App tier, Data tier as separate spokes |
| **Environment per workload** | App1-Prod, App1-Staging, App1-Dev as separate spokes |
| **Microservice spokes** | Each microservice in its own VNet with strict NSGs |

**L3 guidance:**
```
BEST PRACTICE: One Spoke VNet per workload per environment.

Why not put everything in one VNet?
  → NSGs become complex and error-prone
  → No workload isolation (one compromised VM can reach all others)
  → No clear ownership model
  → Scaling limits (VNet scale limits)
  → Troubleshooting is harder (too many sources/destinations)

Why separate spokes?
  → Clean NSG rules (fewer, more specific)
  → Workload isolation (compromise contained)
  → Clear ownership (team owns their Spoke)
  → Independent scaling
  → Easier troubleshooting (fewer connections to analyze)
```

---

### 3.3 MANAGEMENT LANDING ZONE

The **Management Landing Zone** provides **centralized monitoring, logging, alerting, and operational management** for the entire Azure estate.

**Components:**

| Component | Purpose |
|-----------|---------|
| **Log Analytics Workspace** | Central log collection from all resources, NSGs, Firewall, Activity Log |
| **Azure Monitor** | Metrics, alerts, dashboards |
| **Azure Alerts** | Proactive notifications on anomalies (cost, security, performance) |
| **Azure Service Health** | Platform issue notifications |
| **Resource Graph** | Query resources across subscriptions for governance checks |
| **Azure Automation** | Runbooks for automated remediation |
| **Azure Policy compliance** | Compliance dashboard for all assignments |
| **Cost Management** | Budgets, alerts, chargeback reports |
| **Azure Advisor** | Recommendations (cost, performance, security, reliability) |
| **Application Insights** | Application performance monitoring |
| **Azure Monitor Workbooks** | Custom visualization and reporting |

**Centralized Log Architecture:**
```
All resources → Diagnostic Settings → Log Analytics Workspace (Central)

Sources:
  ├── VM OS logs (Windows Event, Linux syslog)
  ├── VM performance counters
  ├── NSG Flow Logs
  ├── Azure Firewall logs
  ├── Azure Activity Log (control plane operations)
  ├── Azure Policy compliance logs
  ├── Azure Security Center / Defender logs
  ├── Application logs (App Service, etc.)
  └── Custom logs (application-specific)

All logs → Single Log Analytics Workspace (in Platform/Shared Services sub)

Queries across all resources:
  → Security: KQL queries for threat detection
  → Operations: Performance correlation
  → Compliance: Policy evaluation tracking
  → Cost: Resource usage analysis
```

**Diagnostic Settings — Where to route:**

| Source | Destination | Purpose |
|--------|------------|---------|
| **Activity Log** | Log Analytics, Storage, Event Hub | Control plane audit trail |
| **NSG Flow Logs** | Storage, Log Analytics | Network traffic analysis |
| **Azure Firewall logs** | Log Analytics, Storage | Security monitoring |
| **VM diagnostics** | Log Analytics, Storage | OS and application monitoring |
| **Azure SQL** | Log Analytics, Storage | Database audit and threat detection |
| **Storage** | Log Analytics, Storage | Data access auditing |
| **Key Vault** | Log Analytics, Storage | Access auditing |
| **App Service** | Log Analytics | Application logging |
| **Azure AD** | Log Analytics, Storage | Sign-in and audit logs |
| **DNS** | Log Analytics, Storage | DNS query logging |

> **L3 critical:** The Management Landing Zone is the "glass cockpit" of your Azure environment. Without it, you are operating blind. Diagnostic settings for ALL resources should be configured as a Policy (DeployIfNotExists) to ensure no resource operates without logging.

---

### 3.4 IDENTITY LANDING ZONE

The **Identity Landing Zone** is the **Entra ID tenant** structure and how directory roles, groups, and Conditional Access are organized for the Landing Zone.

**Structure:**

```
Entra ID Tenant: contoso.onmicrosoft.com (P2)
  │
  ├── Security Groups (RBAC assignment targets):
  │   ├── "Team-App1-Contributor" → Contributor on Sub-App1-Prod
  │   ├── "Team-App1-Reader" → Reader on Sub-App1-Prod
  │   ├── "Team-App2-Contributor" → Contributor on Sub-App2-Prod
  │   ├── "Platform-Admin" → Owner on MG-Services
  │   ├── "Security-Team" → Security Admin + Conditional Access Admin
  │   ├── "Helpdesk" → User Administrator + Password Administrator
  │   └── "Network-Team" → Network Contributor on MG-Production
  │
  ├── Directory Roles (PIM-managed):
  │   ├── Global Administrator → 4 named admins (PIM eligible)
  │   ├── Conditional Access Administrator → 2 security team members
  │   ├── Application Administrator → 3 platform team members
  │   ├── User Administrator → 5 helpdesk members
  │   ├── Privileged Role Administrator → 3 security members
  │   └── Security Administrator → 4 security team members
  │
  ├── Conditional Access Policies:
  │   ├── Require MFA for Global Admins
  │   ├── Block legacy authentication
  │   ├── Require compliant device for corporate resources
  │   ├── Location-based conditions (block from high-risk countries)
  │   ├── Risk-based conditional access (Adaptive Protection)
  │   └── Session controls (Sign-in frequency, app control)
  │
  ├── Custom Security Attributes (for ABAC):
  │   ├── Department
  │   ├── Team
  │   ├── Project
  │   ├── ClearanceLevel
  │   └── CostCenter
  │
  └── Privileged Identity Management (PIM):
      ├── All Global Admin → PIM eligible (not permanently active)
      ├── Time-bound activation (8-hour windows)
      ├── Approval required for activation
      ├── Eligible assignments for operational roles
      └── Permanent assignments only for service accounts (with justification)
```

**Identity Landing Zone best practices:**

```
1. ALL Global Admins must be PIM-eligible (not permanent)
2. P2 license required for PIM
3. Conditional Access enforced BEFORE RBAC grants (CA evaluates at sign-in)
4. All service accounts documented with expiration dates
5. RBAC assigned to groups (not users) — Identity Landing Zone groups
6. Custom security attributes populated for ABAC conditions
7. Break-glass accounts documented and secured (separate RG, heavily monitored)
8. Named individuals for admin roles (no "admin@contoso.com" generic accounts)
```

---

### 3.5 CONNECTIVITY LANDING ZONE

The **Connectivity Landing Zone** defines how the Azure estate connects to on-premises networks, the internet, and between Azure regions.

**Components:**

| Component | Purpose | When to Use |
|-----------|---------|-------------|
| **ExpressRoute** | Private, dedicated, SLA-backed connection to on-prem | Enterprise workloads, low-latency requirements, high bandwidth |
| **VPN Gateway** | Encrypted tunnel over internet to on-prem | Small-medium deployments, cost-sensitive, temporary |
| **Virtual WAN** | Hub-and-spoke connectivity across regions | Multi-region, branch office connectivity, large scale |
| **Load Balancer** | Internal/external traffic distribution | Multi-instance workloads, high availability |
| **Application Gateway** | Layer 7 load balancing, WAF | Web applications, API management |
| **Front Door** | Global HTTP/S load balancing, WAF, CDN | Multi-region web applications |
| **Public IP** | Internet-accessible resources (minimized) | Only where necessary (avoid where Bastion/Private Link works) |

**Connectivity Architecture:**

```
ON-PREM DATA CENTER
       │
       ├── ExpressRoute Circuit 1 (Primary)
       │       │
       │       └── ExpressRoute Gateway
       │               │
       │               └── Connected to: Hub VNet Gateway Subnet
       │
       ├── ExpressRoute Circuit 2 (Secondary/Redundant)
       │       │
       │       └── ExpressRoute Gateway
       │               │
       │               └── Connected to: Hub VNet Gateway Subnet
       │
       └── VPN (Fallback/Backup)
               │
               └── VPN Gateway (in Hub VNet)

INTERNET
       │
       ├── Azure Front Door (web traffic)
       │       │
       │       └── Application Gateway (WAF)
       │               │
       │               └── Spoke VNet (Web tier)
       │
       └── Direct internet access (minimized, only public endpoints)

AZURE LANDING ZONE (Hub-and-Spoke via Virtual WAN):
       │
       ├── Virtual WAN Hub (Primary Region)
       │       │
       │       ├── Spoke VNet (Region 1)
       │       ├── Spoke VNet (Region 2)
       │       └── Hub VNet (Platform Services)
       │
       └── Virtual WAN Hub (Secondary Region)
               │
               └── Spoke VNet (DR Region)
```

**ExpressRoute vs VPN:**

| Aspect | ExpressRoute | VPN |
|--------|-------------|-----|
| **Connection** | Dedicated circuit through connectivity provider | IPsec tunnel over public internet |
| **Latency** | Consistent, low | Variable (depends on internet) |
| **Bandwidth** | Up to 100 Gbps | Up to 10 Gbps (VpnGw1_AZ) |
| **SLA** | Yes (99.9% for primary, 99.99% for redundant) | 99.9% (gateway SLA only) |
| **Security** | Private (not on internet) | Encrypted (on internet) |
| **Cost** | Higher (circuit + provisioned throughput) | Lower |
| **Use case** | Production workloads, sensitive data | Dev/test, small workloads, temporary |
| **Redundancy** | Two circuits (primary + secondary) | Active-active VPN gateways |

**Virtual WAN:**
```
Virtual WAN provides:
  → Hub-and-spoke connectivity across regions
  → Transitive peering (spoke A → hub → spoke B)
  → Integration with ExpressRoute and VPN
  → Automated branch-to-Azure connectivity
  → Network Virtual Appliance (NVA) integration (Firewall, IDS/IPS)
  → Traffic flow rules (security segmentation)

Without Virtual WAN:
  → Manual VNet peerings (complex at scale)
  → No transitive peering
  → Each region requires its own gateway
  → No centralized traffic management
```

---

### 3.6 SECURITY LANDING ZONE

The **Security Landing Zone** defines how security is enforced across the Azure estate — from network perimeters to data protection to threat detection.

**Components:**

| Component | Purpose | Layer |
|-----------|---------|-------|
| **Azure Defender (Sentinel/MD)** | Unified threat detection and response | Detection |
| **Network Security Groups (NSGs)** | VM-level traffic filtering | Network |
| **Azure Firewall** | Centralized egress/ingress filtering | Network |
| **Web Application Firewall (WAF)** | Application-layer protection | Application |
| **Azure Key Vault** | Secret and key management | Data |
| **Azure Information Protection** | Data classification and labeling | Data |
| **Microsoft Purview** | Data governance, DLP, sensitivity | Data |
| **Microsoft Defender for Cloud** | CSPM, CWPP, regulatory compliance | Cloud |
| **Azure Security Center** (legacy) → Defender for Cloud | Unified security posture management | Cloud |
| **Managed Identity** | Credential-free authentication | Identity |
| **Privileged Identity Management** | Time-bound, approved admin access | Identity |

**Security Architecture:**

```
DEFENSE IN DEPTH:

Layer 1: Perimeter
  ├── Azure Front Door (DDoS, WAF, CDN)
  ├── ExpressRoute (private, not internet)
  └── DNS policies (block malicious domains)

Layer 2: Network
  ├── NSGs (VM-level, micro-segmentation)
  ├── Azure Firewall (egress filtering, FQDN rules)
  ├── User Defined Routes (force traffic through Firewall)
  └── Private Link (no public access to PaaS)

Layer 3: Compute
  ├── Defender for Cloud (VM vulnerability assessment, extension)
  ├── OS hardening (AutoPatch, Image policies)
  ├── Just-in-Time VM access (JIT — reduce open RDP/SSH ports)
  ├── Azure Bastion (no public IPs on VMs)
  └── Adaptive network security (Adaptive NSG rules)

Layer 4: Data
  ├── Key Vault (all secrets/certificates/keys)
  ├── Encryption at rest (Azure-managed or CMK)
  ├── Encryption in transit (TLS 1.2+)
  ├── Microsoft Purview (sensitivity labels, DLP)
  └── Backup and recovery (soft delete, immutable backup)

Layer 5: Identity
  ├── Conditional Access (MFA, device compliance, location)
  ├── PIM (time-bound admin access)
  ├── Defender for Identity (on-prem AD monitoring)
  └── Risk-based Conditional Access (Adaptive Protection)

Layer 6: Monitoring
  ├── Microsoft Sentinel (SIEM/SOAR)
  ├── Log Analytics (centralized logs)
  ├── NSG Flow Logs (network visibility)
  ├── Firewall logs (traffic visibility)
  └── Defender for Cloud alerts (threat alerts)
```

**Azure Firewall with Firewall Manager:**
```
Firewall Manager provides:
  → Central security policy management across multiple Firewalls
  → Firewall policy (reusable, versioned)
  → Threat intelligence feeding (alert/deny)
  → TLS inspection (beta/preview at scale)
  → DNS security and filtering
  → Security stacking (automatic Firewall deployment in spoke VNets)

Without Firewall Manager:
  → Each Firewall must be configured individually
  → Policy drift across multiple Firewalls
  → No centralized visibility
  → Manual threat intelligence updates per Firewall
```

---

### 3.7 GOVERNANCE LANDING ZONE

The **Governance Landing Zone** is the umbrella of all governance mechanisms applied to the Landing Zone structure. This is where the policies, RBAC, Blueprints, and management structure from earlier modules come together.

**Governance applied at each level:**

| Level | Governance Mechanism | Examples |
|-------|---------------------|----------|
| **Tenant Root** | Baseline policy | Allowed locations, required tags (at minimum) |
| **MG-Corporate-Policy** | Security baseline | Security Benchmark, allowed SKUs, required diagnostics |
| **MG-Production** | Operational governance | NSG standards, monitoring requirements, naming conventions |
| **MG-Development** | Relaxed governance | Audit mode for some policies, broader locations allowed |
| **Subscription** | RBAC, Blueprint, Policy | Team-specific RBAC, Blueprints for deployment |
| **Resource Group** | RBAC, Policy, Locks | Application-specific access, resource locks on critical resources |

**How governance lands in the Landing Zone:**

```
Blueprint assignment on Subscription → Creates standardized environment
  → ARM templates for resource structure
  → RBAC role assignments (who can do what)
  → Policy assignments (what is allowed)
  → Diagnostic settings (monitoring enabled)
  → Tags (inherited from subscription)

Policy assignments at MG level → Enforce standards across subscriptions
  → Required tags (Deny/Modify)
  → Allowed locations (Deny)
  → Deploy diagnostic settings (DeployIfNotExists)
  → Security Baseline (Audit)

Resource locks at RG level → Protect critical resources
  → CanNotDelete on production databases
  → ReadOnly on core networking

RBAC at every level → Who can access what
  → Subscription: Team owners
  → RG: Application teams
  → Resource: Individual resource access (minimal)
```

---

## 4. SUBSCRIPTION STRUCTURE — COMPLETE

### 4.1 Subscription Types

| Subscription Type | Purpose | Governance Tier |
|-------------------|---------|-----------------|
| **Production** | Live workloads serving customers | Strictest policies, PIM for all admins |
| **Staging/Pre-prod** | Testing before production release | Moderate policies, some relaxed |
| **Development** | Active development and testing | Audit-mode policies, broader access |
| **Shared Services** | Platform infrastructure (Hub, monitoring, identity) | Owner access limited to platform team |
| **Sandbox** | Experimentation and learning | Minimal governance, broadest access |
| **Audit/Compliance** | Regulatory and security audit logging | Dedicated subscription for audit logs |

### 4.2 Subscription Structure Pattern

```
SUBSCRIPTION STRATEGY:

  Per-team subscriptions (not per-user):
    Sub-App1-Production
    Sub-App2-Production
    Sub-App1-Development
    Sub-App2-Development

  Per-function subscriptions:
    Sub-Shared-Platform (Hub, Firewall, DNS, Bastion, Logs)
    Sub-Shared-Identity (Entra ID resources, Conditional Access)
    Sub-Shared-Monitoring (Log Analytics, Alerts, Dashboards)
    Sub-Shared-Networking (VPN, ExpressRoute, vWAN)

  Per-environment subscriptions:
    Sub-Contoso-Production (all apps in prod)
    Sub-Contoso-Development (all apps in dev)

  Per-region subscriptions:
    Sub-Contoso-East (East US resources)
    Sub-Contoso-West (West US resources)
```

### 4.3 Subscription Limits and Considerations

| Limit | Value |
|-------|-------|
| Max subscriptions per tenant | 2,000 (varies by offer type) |
| Max RBAC role assignments per subscription | 20,000 |
| Max resource groups per subscription | 1,000 (or more depending on API version) |
| Max resources per subscription | Unlimited (practical limits depend on management) |
| Cost tracking | Per subscription, per resource group, per tag |

> **L3 guidance:** Choose the subscription structure that best fits your organization. The most common pattern is per-application-per-environment. Avoid creating too many subscriptions (management overhead) or too few (governance gaps). A typical enterprise with 20-50 teams might have 50-100 subscriptions.

---

## 5. MANAGEMENT GROUP HIERARCHY — COMPLETE

### 5.1 Hierarchy Design

```
Tenant Root
  │
  ├── MG-Enterprise-Baseline
  │   ├── Policy: Required tags (Deny)
  │   ├── Policy: Allowed locations (Audit)
  │   └── RBAC: Tenant-level read access
  │
  ├── MG-Security
  │   ├── Policy: Security Baseline (Audit → eventually Deny)
  │   ├── Policy: Allowed SKUs (Deny)
  │   ├── Policy: Deploy diagnostics (DeployIfNotExists)
  │   ├── RBAC: Security team → Security Admin + Conditional Access Admin
  │   ├── MG-Production
  │   │   ├── Sub-Prod-West
  │   │   ├── Sub-Prod-East
  │   │   └── RBAC: Production team → Contributor
  │   └── MG-Development
  │       ├── Sub-Dev-A
  │       ├── Sub-Dev-B
  │       └── RBAC: Dev team → Contributor (wider scope)
  │
  └── MG-Shared-Services
      ├── Sub-Platform (Hub VNet, Firewall, DNS, Bastion)
      ├── Sub-Monitoring (Log Analytics, Alerts, Dashboards)
      ├── Sub-Identity (Conditional Access, PIM configuration)
      └── RBAC: Platform team → Owner/Contributor
```

### 5.2 Management Group Design Principles

| Principle | Description |
|-----------|-------------|
| **Mirror your organization** | MG structure should map to teams/business units |
| **Consistent naming** | Standard naming convention (e.g., MG-{Function}-{Scope}) |
| **Minimal depth** | Keep MG hierarchy shallow (3-4 levels max) |
| **Policy at highest scope** | Apply baseline policies at highest practical scope |
| **RBAC separation** | Platform team manages shared services; App teams manage their own workloads |
| **Avoid deep nesting** | Complex nesting makes troubleshooting harder |
| **Document the hierarchy** | Maintain documentation of MG structure and its purpose |

---

## 6. CENTRALIZED LOGGING — COMPLETE DEEP DIVE

### 6.1 Log Analytics Workspace Architecture

```
CENTRAL LOG ANALYTICS WORKSPACE:
  Located in: MG-Shared-Services or Sub-Monitoring

  Tables:
    ├── AzureActivity (control plane operations)
    ├── AzureDiagnostics (resource diagnostic logs)
    ├── NSGFlow (network security group flow logs)
    ├── AzureFirewall (firewall logs)
    ├── AzureVMSyslogCommon (VM OS logs)
    ├── AzureVMWindowsEvent (Windows events)
    ├── AzureSQLSecurityAuditLog (SQL audit)
    ├── AzureResourceAction (resource operations)
    ├── ResourceContainer (resource inventory)
    ├── SecurityEvent (security events from Defender)
    ├── SentinelEvents (SIEM events from Microsoft Sentinel)
    └── ... (hundreds more)

  Standard Queries:
    ├── Failed login attempts
    ├── Resource creation/deletion events
    ├── Network traffic anomalies
    ├── Policy compliance changes
    ├── Cost analysis by tag
    └── Security incidents
```

### 6.2 Diagnostic Settings — Resource Level

Every resource should have diagnostic settings configured. The settings define:

| Setting | Options |
|---------|---------|
| **Category** | Depends on resource type (e.g., "AllLogs", "AzureActivity", "StorageRead", "StorageWrite") |
| **Destination** | Log Analytics workspace, Storage Account, Event Hub, or another resource |
| **Enabled** | Yes/No per category |

**Centralized pattern:**
```
All resources → Diagnostic Settings → Log Analytics Workspace (central)
  
For redundancy/cost:
  All Activity Logs → Storage Account (long-term retention, archive)
  All critical resource logs → Log Analytics (real-time analysis)
  All NSG/Firewall logs → Storage Account + Log Analytics (dual)
  All security events → Log Analytics + Sentinel (SIEM)
```

### 6.3 Log Analytics Workspace per Region vs Centralized

| Approach | Description | Trade-off |
|----------|-------------|-----------|
| **Per-region workspace** | Each region has its own Log Analytics | Regional isolation, but management overhead |
| **Central workspace** | All logs flow to one workspace | Single pane of glass, but latency and size |
| **Hub workspace (recommended)** | Central workspace with link workspaces per region | Best of both: regional collection + central analysis |

---

## 7. SHARING AND COST MANAGEMENT

### 7.1 Shared Services Pattern

```
SHARED SERVICES (in Platform Landing Zone):
  No individual team "owns" a shared service.
  All teams consume shared services.

  Benefits:
    → No duplication (one Firewall for all, not one per team)
    → Centralized management (security team manages Firewall)
    → Consistent enforcement (same Firewall rules for all)
    → Cost efficiency (one resource vs. many copies)
    → Clear ownership (Platform team is responsible)

  Shared Services examples:
    ├── Firewall (network security)
    ├── DNS (Private zones)
    ├── Log Analytics (centralized logging)
    ├── Bastion (VM access)
    ├── Key Vault (secrets/certificates)
    ├── Azure AD tenant identity management
    └── Application Gateway / Front Door (web traffic)
```

### 7.2 Cost Management

```
COST ALLOCATION STRATEGY:

1. Tag every resource (required by policy)
   → Environment, Project, CostCenter, Owner, Department

2. Cost allocation by:
   → Subscription (subscription-level cost)
   → Resource Group (RG-level cost)
   → Tag (project, department, environment)
   → Service (compute, storage, network separately)

3. Budget alerts:
   → Monthly budget per subscription
   → Email notifications at 80%, 100%
   → Action groups (auto-shutdown, email, webhook)

4. Cost analysis:
   → Cost Management + Billing
   → Analyze by tag, service, resource group, subscription
   → Showback/chargeback reports per team/department

5. Optimization:
   → Advisor recommendations (reserved instances, idle resources)
   → Auto-shutdown for dev/test (tag-based)
   → Right-sizing recommendations
   → Storage tier optimization
```

---

## 8. SECURITY CONSIDERATIONS

| Concern | Risk | Mitigation |
|---------|------|------------|
| **Hub VNet compromise** | Attacker in Hub can access all Spokes | NSGs on Hub subnets, Firewall in path, minimal Hub NSG rules, NSG flow logs |
| **Spoke-to-Spoke direct peering** | Bypasses Firewall inspection | No direct Spoke-to-Spoke peering; all transit through Hub |
| **Public IP exposure** | VMs with public IPs are internet-facing | Azure Bastion (no public IPs), Private Link for PaaS, NSGs block inbound from internet |
| **Centralized logging compromise** | Attacker deletes logs to cover tracks | Storage account with immutable audit logs, separate monitoring subscription, restricted access to Log Analytics |
| **Shared service dependency** | If Platform goes down, all workloads affected | Redundancy (multi-region Hub), separate monitoring for Platform services |
| **Privileged access** | Global Admins can bypass governance | PIM for all admin roles, break-glass procedure, audit logging for all admin actions |
| **Firewall single point of failure** | If Firewall is down, egress is blocked | Active/Standby Firewall (HA pair), or firewall redundancy across availability zones, fallback VPN for egress (if configured) |
| **DNS poisoning** | Private DNS records can be spoofed | DNS policies, Private Link, private endpoints instead of public names |
| **Management plane exposure** | Activity logs, resource graph, cost data accessible to unauthorized users | RBAC at MG level, monitor access to governance resources |
| **Resource drift** | Resources deviate from baseline (tags removed, diagnostics off) | Continuous evaluation, Resource Graph queries, regular compliance audits |
| **Data exfiltration** | Data copied to unauthorized storage | Defender for Cloud alerts, NSG flow log analysis, DLP policies in Purview |

---

## 9. MONITORING — LANDING ZONES

### Key Monitoring Targets

| Metric/Log | What It Shows | Alert Trigger |
|------------|---------------|---------------|
| **Policy compliance %** | Percentage of compliant resources | Drop below threshold (e.g., 95%) |
| **Non-compliant resources count** | Number of resources failing policies | Increase in non-compliance |
| **Firewall logs** | Traffic patterns, blocked connections | Unusual traffic, threat intel alerts |
| **NSG flow logs** | Network traffic to/from VMs | Unusual patterns, unauthorized access |
| **Diagnostic settings coverage** | Resources without diagnostics | Resources missing diagnostics |
| **Activity log anomalies** | Unusual admin actions | Resource deletion in production, role assignment changes |
| **Cost anomalies** | Unexpected cost spikes | Budget threshold exceeded |
| **Log Analytics capacity** | Ingestion rate, storage usage | Near capacity limits |
| **Bastion session activity** | Who accessed which VM, when | Unusual access patterns |
| **Key Vault access** | Secret/key access patterns | Access from unusual IP, unusual time |

---

## 10. PRODUCTION EXAMPLE

**Scenario: Global enterprise with 5,000+ resources across 12 subscriptions in 3 regions.**

```
Tenant: contoso.onmicrosoft.com (P2)

TENANT ROOT
  │
  ├── MG-ENTERPRISE-BASELINE
  │   │  Policy: Required Tags (Deny) — Environment, Project, CostCenter, Owner
  │   │  Policy: Allowed Locations (Audit) — All Azure regions
  │   │  RBAC: Tenant Reader (all authenticated users)
  │   │
  │   ├── MG-CORPORATE-SECURITY
  │   │   │  Policy: Security Baseline (Audit → 6-month → Deny rollout)
  │   │   │  Policy: Allowed SKUs (Deny — approved VM/storage SKUs)
  │   │   │  Policy: Deploy Diagnostics (DeployIfNotExists → central Log Analytics)
  │   │   │  Policy: Required Tags (Deny — stricter: also DataClassification)
  │   │   │
  │   │   │  RBAC:
  │   │   │    "Platform-Admin" → Owner on MG-SHARED-SERVICES
  │   │   │    "Security-Team" → Security Admin + Conditional Access Admin
  │   │   │    "Network-Team" → Network Contributor + Firewall Operator
  │   │   │    "Helpdesk" → User Administrator + Password Administrator
  │   │   │
  │   │   ├── MG-PRODUCTION
  │   │   │   │
  │   │   │   ├── Sub-Prod-East-US (App VMs, databases)
  │   │   │   │   ├── rg-app1-prod, rg-app2-prod
  │   │   │   │   ├── rg-data-prod
  │   │   │   │   ├── RBAC: "Team-App1-Prod" → Contributor on rg-app1-prod
  │   │   │   │   ├── RBAC: "Team-App2-Prod" → Contributor on rg-app2-prod
  │   │   │   │   ├── RBAC: "Team-Data-Prod" → Contributor on rg-data-prod
  │   │   │   │   ├── Blueprints: Production Environment v3 (assigned)
  │   │   │   │   └── Lock: CanNotDelete on rg-data-prod
  │   │   │   │
  │   │   │   ├── Sub-Prod-West-US (DR/Secondary)
  │   │   │   │   ├── rg-app1-dr, rg-app2-dr
  │   │   │   │   └── Same RBAC groups (regional prefix)
  │   │   │   │
  │   │   │   └── Policy: Subscription override
  │   │   │       └── Additional policy: NSG audit (all VMs must have NSG)
  │   │   │
  │   │   └── MG-DEVELOPMENT
  │   │       │
  │   │       ├── Sub-Dev-A (Application development)
  │   │       │   ├── rg-app1-dev, rg-app2-dev
  │   │       │   ├── RBAC: "Team-App1-Dev" → Contributor on rg-app1-dev
  │   │       │   └── Policy: Relaxed SKU policy, Audit for tags
  │   │       │
  │   │       ├── Sub-Dev-B (Experimentation)
  │   │       │   └── rg-experiments
  │   │       │   └── Policy: Audit mode for most policies
  │   │       │
  │   │       └── Policy: Subscription override
  │   │           └── Relaxed allowed locations (all regions allowed)
  │   │
  │   └── MG-SHARED-SERVICES (Platform Landing Zone)
  │       │
  │       ├── Sub-Platform-Network (Hub VNet, Firewall, DNS, Bastion)
  │       │   ├── rg-hub-network
  │       │   ├── Hub VNet: 10.0.0.0/16
  │       │   ├── Subnet: AzureFirewallSubnet, GatewaySubnet, BastionSubnet
  │       │   ├── Azure Firewall (HA pair across AZs)
  │       │   ├── Azure Bastion (BG v2)
  │       │   ├── Azure DNS Private Resolver
  │       │   ├── UDRs: all Spokes → Firewall
  │       │   └── RBAC: "Platform-Network-Admin" → Owner on this sub
  │       │
  │       ├── Sub-Platform-Identity (Entra ID configuration)
  │       │   ├── Conditional Access policies
  │       │   ├── PIM configuration
  │       │   ├── Security attribute management
  │       │   └── RBAC: "Platform-Identity-Admin" → Conditional Access Admin
  │       │
  │       ├── Sub-Platform-Monitoring (Centralized Logging)
  │       │   ├── rg-monitoring
  │       │   ├── Log Analytics Workspace (Central — all logs here)
  │       │   ├── Alert Rules (cost, compliance, security)
  │       │   ├── Dashboards (executive, operational, security)
  │       │   ├── Microsoft Sentinel workspace (if SIEM)
  │       │   └── RBAC: "Platform-Monitoring" → Monitoring Reader + Log Analytics Contributor
  │       │
  │       └── Sub-Platform-Security (Security services)
  │           ├── rg-security
  │           ├── Defender for Cloud (all subscriptions enrolled)
  │           ├── Key Vault (centralized secrets for Platform)
  │           ├── Azure AD DS (if needed)
  │           └── RBAC: "Platform-Security-Admin" → Security Admin
  │
  └── MG-EXPERIMENTATION (Sandbox)
      │  Minimal governance
      │  Audit mode for all policies
      │  Broad RBAC (Reader/Contributor)
      └── Sub-Sandbox-01
```

**Resource Tags at every level:**
```
Subscription level: Environment=Production, CostCenter=12345, Department=Engineering
RG level: Project=App1, Owner=Team-App1
Resource level: (adds Criticality=High, DataClassification=Confidential)
```

**Onboarding a new team:**
```
1. Request subscription/RG access through IT service catalog
2. IT creates RBAC role assignments (via group-based assignment)
3. New team deploys workload using Production Blueprint
4. Blueprint creates: VNets, NSGs, VMs, RGs, RBAC, Diagnostics, Tags
5. Policy enforces: tags, locations, SKUs, diagnostics
6. Security: Firewall, NSG rules, Defender alerts
7. Monitoring: logs flowing to central Log Analytics
8. Cost: tracking via tags from day 1

Time to operational: 1-2 days.
Without Landing Zone: 2-4 weeks.
```

---

## 11. FAILURE SCENARIOS

| Scenario | Root Cause | Resolution | Evidence |
|----------|-----------|------------|----------|
| **Spoke VM cannot reach internet** | UDR not configured or Firewall in path; NSG blocking outbound; Firewall rules blocking destination; DNS resolution failing; Private DNS interfering; Firewall overloaded | Check: (1) UDR in Spoke points to Firewall (or Internet?), (2) NSG allows outbound, (3) Firewall rules allow destination, (4) DNS resolves correctly, (5) Firewall health (CPU, throughput). Most common: Firewall not in path OR Firewall rules block destination. | NSG flow logs: traffic flow, UDR: next hop, Firewall: health and rules, DNS: resolution test |
| **Hub Firewall overloaded with traffic** | All Spoke traffic flows through single Firewall; Firewall throughput exceeded; All traffic including trusted internal routed through Firewall | Check: Firewall throughput and CPU (vs. SKU limits); Check if internal Spoke-to-Spoke traffic should bypass Firewall (Traffic flow rules); Consider Firewall Manager security stacking; Scale up Firewall SKU. Most common: internal traffic should bypass Firewall. | Firewall metrics: throughput, CPU, packet drops; Firewall Manager: traffic flow rules |
| **"I can access the VM but no logs are flowing"** | Diagnostic settings not configured on the VM; Log Analytics workspace in another region; VM is in a subscription not linked to Log Analytics; Agent not installed (for Windows/Linux logs) | Check: Diagnostic settings blade on VM; check Log Analytics workspace; check VM Agent status; check workspace linked subscription. Most common: no diagnostic settings on the VM. | Diagnostic settings: enabled/disabled; Log Analytics: incoming data; VM Extension: AzureMonitorAgent installed |
| **"New VM deployed but isn't compliant with required tags"** | Blueprint not including tags; Resource created outside Blueprint; Tag policy in Audit mode (not Modify/Deny); Tag policy evaluation hasn't completed; Tag inheritance from RG/subscription | Check: policy effect (Audit vs Modify vs Deny); resource tags vs policy requirements; Blueprint template includes tags; continuous evaluation completed. Most common: Modify policy failed (managed identity RBAC issue). | Policy compliance: specific failing policy; Modify: managed identity RBAC; Blueprint: artifact includes tags |
| **"Bastion can't connect to VM in Spoke VNet"** | Bastion in Hub VNet not peered to Spoke VNet; NSG blocking Bastion traffic; VM doesn't have the right NSG; Bastion SKU too old; VM is in different subscription; No User-Assigned MI for Bastion (in some configurations) | Check: VNet peering between Spoke and Hub; NSG rules for Bastion (Allow AzureBastionSubnet → VirtualNetwork, Allow AzureBastionSubnet → Internet for certificate download); VM running; Bastion version. Most common: peering missing or NSG blocking Bastion traffic. | Portal: Bastion connection log; Network: ping/traceroute from Bastion subnet; NSG: effective rules for Bastion subnet |
| **"Audit logs for Subscription are not in central Log Analytics"** | Activity Log diagnostic settings not configured at subscription level; Activity Log diagnostic settings were never configured (default in Azure is to NOT store Activity Logs beyond 90 days in portal); Log Analytics workspace in different subscription; Subscription not linked | Check: Subscription → Diagnostic Settings → Activity Log; check destination; check Log Analytics workspace. Most common: Activity Log diagnostic never configured. | Diagnostic Settings: Activity Log → Enabled? Destination? Activity Log: 90-day limit in portal without diagnostic |
| **"New subscription doesn't inherit policies from MG"** | Subscription is in wrong MG; Policy at MG level is Disabled; Policy assignment at MG has a scope override; Policy enforcement mode is Disabled; Subscription was manually added to MG but policy assignment hasn't propagated (up to 15 min) | Check: Subscription → Subscriptions in MG; Policy: assignments at MG scope; Policy: compliance (green vs red); Policy: enforcement mode. Most common: subscription not in correct MG. | Policy: Compliance overview shows red for new subscription; MG: subscription membership |
| **"DNS resolution fails for Private Link endpoint in Spoke VNet"** | Private DNS zone not linked to Spoke VNet; DNS Private Resolver not configured; Firewall DNS proxy intercepting DNS but not forwarding to Azure DNS; Private endpoint not created; Private Link not in allowed location | Check: Private DNS zone links (VNet associations); DNS resolution from VM (nslookup); Firewall DNS proxy settings; Private endpoint status. Most common: Private DNS zone not linked to the Spoke VNet. | DNS: nslookup/test; Private DNS Zone: VNet link status; Private Endpoint: Provisioning State |
| **"Firewall is not logging traffic"** | Firewall diagnostic settings not configured; Firewall SKU doesn't support logging (Basic doesn't support Azure Firewall logs? Actually Basic does for some); Log Analytics workspace not accessible; Firewall manager not configured (if using FM) | Check: Firewall → Diagnostic Settings; Log Analytics: data flowing; Firewall SKU; Firewall Manager. Most common: Diagnostic settings not configured on Firewall. | Diagnostic Settings: Firewall → Enabled? Log Analytics: Firewall table has data |
| **"Production resources can be deleted by someone who shouldn't be able to"** | No CanNotDelete lock on production RGs; User has Contributor/Owner RBAC; Deny Assignment for delete not present; RBAC assignment was changed recently (backdating possible in API); User is in a group that was recently added to a role | Check: Resource locks on production RGs; RBAC assignments (who has Contributor/Owner?); Deny assignments; recent RBAC changes in Activity Log. Most common: no resource lock on critical production resource group. | Locks: CanNotDelete on production RGs? RBAC: who has delete permission? Activity Log: recent role assignment changes |
| **"Cost is 3x higher than expected this month"** | Dev/Test resources running 24/7 (no auto-shutdown); Oversized VMs; Unattached disks (unused); Storage not in cool/archive tier; Unexpected traffic (exfiltration or misconfigured service); New resources deployed without team knowledge; Auto-shutdown policy not applied | Check: Cost Analysis by service and tag; Advisor recommendations; Unattached disks query; VM size vs. utilization; check new resources via Resource Graph. Most common: dev/test running 24/7 + unattached disks. | Cost: by service and tag; Advisor: cost recommendations; Resource Graph: unattached disks, running VMs outside business hours |
| **"Resource deployed successfully but immediately shows as non-compliant"** | Continuous evaluation hasn't run yet; Policy was just assigned (propagation delay); Resource properties don't match policy; Diagnostic settings for Policy logging not configured; Non-compliance may be stale evaluation | Check: Policy compliance timestamp; Policy assignment time; resource properties vs policy conditions; wait for continuous evaluation (minutes to hours). Most common: policy just assigned and hasn't evaluated yet. | Policy: compliance state timestamp; Resource properties: compare to policy conditions |

---

## 12. TROUBLESHOOTING METHODOLOGY — LANDING ZONES

```
STEP 1: Identify the symptom
  → What is the specific issue? (No connectivity? No logs? Non-compliance? Cost spike?)
  → Which resource(s) are affected?
  → Which Landing Zone component is involved? (Platform, App, Network, Security?)

STEP 2: Determine the Landing Zone layer
  → Is this a Platform issue? (Hub services not working)
  → Is this an Application issue? (Workload VM problem)
  → Is this a Connectivity issue? (Network routing problem)
  → Is this a Security issue? (NSG/Firewall/Defender)
  → Is this a Governance issue? (Policy/RBAC/Blueprint)
  → Is this a Management issue? (Monitoring/logging/alerting)

STEP 3: Apply layer-specific troubleshooting

  → PLATFORM LAYER (Hub services):
    Check: Hub VNet health, Firewall health/subscription, DNS resolution, Bastion connectivity
    Check: UDRs on Spokes, NSGs on Hub subnets
    Check: Private Link endpoint status

  → APPLICATION LAYER (Workload):
    Check: VM running status, OS health, application logs
    Check: NSG rules for the specific VM
    Check: Resource diagnostics (is the VM even healthy?)
    Check: Application-specific logs in Log Analytics

  → CONNECTIVITY LAYER:
    Check: ExpressRoute/VPN status (connection health)
    Check: Virtual WAN hub status
    Check: VNet peerings (are spokes properly peered to Hub?)
    Check: UDRs (next hop correct?)
    Check: Public IP accessibility (if applicable)

  → SECURITY LAYER:
    Check: NSG effective rules (per NSG, per NIC)
    Check: Firewall rules (application/network rules)
    Check: Defender for Cloud alerts
    Check: NSG flow logs for traffic analysis
    Check: JIT access status

  → GOVERNANCE LAYER:
    Check: Policy compliance (which policy? which effect?)
    Check: RBAC assignments (effective permissions)
    Check: Resource locks (CanNotDelete/ReadOnly)
    Check: Blueprint deployment status
    Check: ABAC conditions (if enabled)

  → MANAGEMENT LAYER:
    Check: Diagnostic settings on resource
    Check: Log Analytics data flowing
    Check: Alerts configured and firing
    Check: Dashboard configuration

STEP 4: Cross-layer analysis
  → Many issues span multiple layers
  → Example: VM can't reach internet
    → Application: VM is running (check)
    → Connectivity: UDR exists pointing to Firewall (check)
    → Security: NSG allows outbound (check)
    → Security: Firewall has rule allowing destination (check)
    → Platform: Firewall is healthy and has throughput (check)
    → Connectivity: DNS resolves the destination (check)
    → Found: Firewall has deny rule for the destination — Security layer fix

STEP 5: Fix and validate
  → Apply fix at identified layer
  → Test the specific action
  → Check broader impact (does fix affect other Landing Zone components?)
  → Document root cause
  → Update Landing Zone documentation
```

---

## 13. LOGS / EVIDENCE

| Evidence Source | What It Shows | Access Method |
|---------------|---------------|---------------|
| **Policy compliance dashboard** | Compliance state per assignment, per policy | Portal → Policy → Compliance |
| **Activity Log** | All control plane operations across Landing Zone | Portal → Subscription → Activity Log |
| **Log Analytics (central)** | All diagnostic data, network traffic, application logs | Log Analytics → Logs (KQL queries) |
| **NSG flow logs** | Network traffic allowed/denied per NSG rule | Storage Account or Log Analytics |
| **Azure Firewall logs** | Firewall decision logs (allow/deny/delete) | Log Analytics: AzureFirewall table |
| **Log Analytics** | All diagnostic data, network traffic, application logs | Log Analytics → Logs (KQL queries) |
| **Resource Graph** | Resources across subscriptions with filters | Portal → Resource Graph; CLI: `az graph query` |
| **Resource Locks** | All locks at various scopes | Portal → Resource → Locks; API |
| **Diagnostic Settings** | Per-resource diagnostic configuration | Portal → Resource → Diagnostic settings |
| **Blueprint deployment** | Per-artifact deployment status | Portal → Blueprints → Assignments |
| **Microsoft Sentinel/Defender alerts** | Security threats and incidents | Portal → Defender for Cloud → Alerts |
| **Bastion session logs** | Active and past Bastion sessions | Portal → Bastion → Session records |
| **ExpressRoute/VPN status** | Connection health, bandwidth utilization | Portal → ExpressRoute/VPN Gateway |
| **Cost Analysis** | Cost by subscription, RG, tag, service | Portal → Cost Management |
| **Resource Provider operations** | Resource provider health, registration status | Portal → Subscription → Resource Providers |
| **Sign-in logs (Entra ID)** | Authentication events, CA results | Portal → Entra ID → Sign-in logs |
| **Audit logs (Entra ID)** | Administrative directory changes | Portal → Entra ID → Audit logs |

---

## 14. VERSION/CURRENT SERVICE CONSIDERATIONS

| Aspect | Current State (Sept 2026) | L3 Impact |
|--------|--------------------------|-----------|
| **Azure Bastion (BG v2)** | GA — improved performance, faster connections, Session Recording GA | Newer SGv2 SKU with enhanced throughput. Migration from v1 requires redeployment. |
| **Azure Firewall** | Standard and Premium SKUs GA. Firewall Manager GA with TLS inspection (preview) at scale. Threat Intelligence enabled by default (Alert and Deny modes). | Firewall is the cornerstone of Platform Landing Zone. Premium SKU includes IDPS. TLS inspection still in preview but rapidly maturing. |
| **Private DNS Resolver** | GA. Enables DNS resolution at VNet level (alternative to Firewall DNS proxy). | More flexible than Firewall DNS proxy. Allows DNS filtering without forcing all DNS through Firewall. |
| **Virtual WAN** | GA. Supports millions of connections. Integration with ExpressRoute, VPN, and branch office connectivity. | Primary connectivity fabric for enterprise-scale Azure. Transit through hubs is fully supported. |
| **ExpressRoute** | Global Reach, FastPath, ER Gateway SKU v2 (100Gbps). | FastPath provides line-rate forwarding (bypasses Gateway S chain). ER GW Gen2 supports 100Gbps. |
| **Resource Graph** | GA. 500+ million resources queryable. | Essential for enterprise governance. Run policy compliance queries across all subscriptions in seconds. |
| **Azure Policy** | Modify and DeployIfNotExists are GA. Policy profiler in preview. Cross-tenant policy supported. | Policy is the backbone of Governance Landing Zone. Ensure all governance policies are in Deny/Modify mode (not just Audit) for production. |
| **Azure Blueprints** | GA. Versioning, parameterization, and artifact support complete. | Blueprints are the deployment pattern for Application Landing Zones. Use them for standardized environment creation. |
| **Diagnostic Settings** | Policy-driven deployment (DeployIfNotExists) is the recommended pattern. | Every resource should have diagnostics. Policy ensures this automatically. |
| **Log Analytics** | Per-workspace limits increased. Linked workspaces for multi-region. Data Collection Rules (DCR) for granular control. | DCR allows routing different log categories to different workspaces — useful for regulatory or cost reasons. |
| **Microsoft Defender for Cloud** | Defenders for 60+ Azure services. Server Defender is GA. Adaptive network security is GA. | Defender for Cloud is the Security Landing Zone's detection layer. Ensure all subscriptions are enrolled. |
| **Conditional Access** | Session controls (Sign-in frequency, app control) GA. Risk-based policies with Adaptive Protection. Named locations for trusted zones. | Conditional Access is the Identity Landing Zone's enforcement layer. Policies must be tested carefully before enforcement. |
| **Privileged Identity Management** | GA with time-bound, eligible, and approved assignments. | PIM is essential for Identity Landing Zone. All Global Admins should be PIM-eligible (P2 license). |
| **NSG Service Endpoints vs Private Endpoints** | Service Endpoints deprecated in favor of Private Endpoints for most services. | Private Endpoints are the standard for PaaS access in Landing Zones. Service Endpoints are being phased out. |
| **Azure Firewall migration** | Basic → Standard (or Premium) migration supported. | Standard is recommended for most Landing Zones. Premium for high-security (IDPS enabled). |
| **P4P (Private Access)** | Azure Private Access (Private Link + Private Endpoint + DNS) is GA. | Simplifies Private Link deployment in Landing Zones. |
| **Application Gateway v2 (WAF)** | GA with autoscale. | WAF v2 is recommended for web application security in Application Landing Zone. |
| **Front Door Standard/Premium** | GA with WAF, compression, caching. | Front Door is the recommended ingress for multi-region web applications. |

---

## 15. L3 INTERVIEW QUESTIONS

### Basic
**Q: What is an Azure Landing Zone?**
A: An Azure Landing Zone is an architectural framework that organizes Azure resources into a structured, governed, and secure multi-terabyte cloud environment. It defines how subscriptions, resource groups, networking (hub-and-spoke), security, monitoring, identity, and governance are organized to support enterprise workloads at scale.

### Intermediate
**Q: What is the difference between a Hub VNet and a Spoke VNet?**
A: Hub VNet houses shared infrastructure services (Firewall, DNS, Bastion, Gateway) that multiple workloads depend on. Spoke VNets house individual workload resources (VMs, apps, databases). All Spoke traffic flows through the Hub (via UDRs) for centralized security inspection. Spokes are peered to the Hub but typically not directly to each other.

### L3
**Q: Why would a VM in a Spoke VNet lose internet access after deploying Azure Firewall?**
A: Common causes:
1. User Defined Routes (UDRs) not configured on Spoke subnet pointing to Firewall as next hop
2. NSG on Spoke subnet blocking outbound traffic to internet (port 80/443)
3. Firewall not configured with proper NAT rules for outbound traffic
4. DNS resolution failing (Firewall DNS proxy not configured, or Private DNS zone interfering)
5. Firewall SKU doesn't support required throughput (SKU exhausted)
6. Firewall health issues (overloaded, stopped)
Most common: UDRs not configured. Without UDRs, traffic takes default internet path and doesn't go through Firewall.

### Senior L3
**Q: Explain how the seven Landing Zones interact in a production Azure estate.**
A:
1. **Platform Landing Zone (Hub VNet)**: Houses Firewall, DNS, Bastion, Key Vault. All Spokes depend on these shared services.
2. **Application Landing Zone (Spoke VNets)**: Where workloads actually run. Each app gets its own Spoke for isolation.
3. **Connectivity Landing Zone (ExpressRoute/VPN/vWAN)**: Provides connectivity to on-prem and between regions. Connects to Hub Gateway Subnet.
4. **Security Landing Zone (NSG, Firewall, Defender, Key Vault)**: Distributed across Hub (Firewall) and Spokes (NSGs). Centralized in Hub, enforced on Spokes.
5. **Identity Landing Zone (Entra ID, CA, PIM)**: Tenant-wide. Grants access to all Landing Zone resources based on directory roles and RBAC.
6. **Management Landing Zone (Log Analytics, Alerts, Cost Mgmt)**: Central workspace in Platform sub. Collects logs from all resources across all Landing Zones.
7. **Governance Landing Zone (Policy, RBAC, Blueprints, MGs)**: Tenant-wide. Enforces standards, assigns permissions, deploys blueprints across all Landing Zones.

### Expert
**Q: What happens internally when a VM in a Spoke VNet tries to access an Azure PaaS service (e.g., Azure SQL) via Private Link?**
A:
1. VM's DNS resolver queries for `privatelink.database.windows.net`
2. If Azure DNS Private Zone is linked to Spoke VNet → resolves directly to Private Endpoint IP
3. If Firewall DNS proxy is enabled → DNS query goes to Firewall → forwarded to Azure DNS → returns Private Endpoint IP
4. VM creates TCP connection to Private Endpoint IP (in Spoke or Hub VNet)
5. Traffic goes through Private Link infrastructure (not through Firewall)
6. Traffic reaches Azure SQL (in Azure backbone network)
7. Authentication: SQL login or Azure AD authentication via Entra ID
8. Response follows reverse path

Key: Private Link traffic BYPASSES Firewall — this is by design (PaaS services should have direct private access). Firewall is for VM-to-VM and VM-to-internet traffic control.

### Scenario
**Q: "Production deployment was working yesterday, but today all VMs in Spoke-01 have no internet connectivity. Nothing changed in the network." What do you check?**
A:
1. Check VM status (running? stopped/deallocated?)
2. Check NSG effective rules on Spoke-01 (did NSG rules change?)
3. Check UDRs on Spoke-01 subnet (did next hop change? Was it always pointing to Firewall, or was there a default internet route?)
4. Check Firewall health (running? CPU/throughput exhausted? Rules changed?)
5. Check Firewall diagnostics (is it actually processing traffic?)
6. Check DNS resolution (can VM resolve hostnames? Nslookup test)
7. Check ExpressRoute/VPN gateway status (if internet access goes through these)
8. Check Activity Log for network-related changes (even if "nothing changed", rules and configs do change)
9. Check NSG Flow Logs (what traffic is being denied and by which rule?)
10. Check if Firewall has gone down or if there's a planned maintenance event

Most likely: Firewall issue (rules changed, SKU exhausted, health degraded) or DNS issue. Even if "nothing changed in the network," Firewall auto-scaling or rule updates might have occurred.

### Tricky
**Q: "Azure Firewall is logging traffic, but NSG flow logs are not. Both are in the same subscription."**
A: Possible reasons:
1. Diagnostic settings: NSG has no diagnostic settings configured (Firewall has them; NSGs don't)
2. NSG diagnostic settings are configured but destination is wrong/down
3. NSG flow logs have a different collection endpoint (separate storage or Log Analytics)
4. NSG was created after the diagnostic settings policy was applied (not yet remediated)
5. NSG is in a different scope than the diagnostic settings policy
Most likely: NSG diagnostic settings simply not configured (common — Firewall gets attention, NSGs are forgotten). Check Diagnostic Settings on each NSG.

### Tricky 2
**Q: "All VMs are in a Spoke VNet. The team wants internet access but no public IPs. We have Bastion and Firewall. How does outbound internet work without public IPs on VMs?"**
A:
1. VMs have no public IP (by design)
2. Outbound internet traffic flow:
   a. VM → UDR → Spoke subnet routes to Hub VNet
   b. Hub VNet → UDR → Azure Firewall (default route 0.0.0.0/0)
   c. Firewall has NAT rules for outbound (source translation: Firewall's public IP)
   d. Firewall → Internet
3. Return traffic: Internet → Firewall (DNAT) → Spoke VM (private IP)
4. Bastion is separate: provides RDP/SSH access TO VMs (not FROM VMs to internet)
5. VMs can reach Azure PaaS via Private Link (bypassing Firewall)

The key insight: VMs use Firewall's public IP for outbound internet via SNAT/DNAT. VMs never need their own public IP.

### Tricky 3
**Q: "A new subscription was created under MG-PRODUCTION but policies from MG aren't applying. The subscription's RG has resources that are non-compliant. What's the issue?"**
A: Possible reasons:
1. Subscription was NOT placed under MG-PRODUCTION (it's directly under Tenant Root or a different MG)
2. Policy assignment at MG has Enforcement Mode = Disabled
3. Policy assignment at MG is in a different MG than where the subscription was placed
4. Policy was recently assigned — propagation delay (up to 15 minutes for continuous evaluation)
5. Resource Provider registration issue (some policies need the provider registered)
6. Policy scope override at subscription level (a policy assignment specifically excludes this subscription)
Most likely: Subscription is NOT in the correct Management Group. Verify subscription placement in the MG hierarchy. This is the #1 cause of "policies not applying" issues.

---

## 16. SCENARIO-BASED QUESTIONS

### Scenario 1: "All VMs lose internet after Firewall deployment"
**Architecture:** VM → UDR → Hub VNet → Firewall → Internet. 
**Dependencies:** UDR, NSG, Firewall rules, DNS, Firewall health.
**Checks:**
1. UDR on Spoke subnet: does 0.0.0.0/0 point to Firewall?
2. NSG on Spoke: does outbound 80/443 to Internet allow?
3. Firewall: has default route and NAT rule configured?
4. Firewall: health and throughput OK?
5. DNS: can VM resolve external hostnames?
6. Firewall: diagnostic logs show traffic being processed?
**Root Cause: UDR not configured pointing to Firewall, OR NSG blocking outbound, OR Firewall misconfigured (no default route rule).**
**Fix:** Configure UDR to point to Firewall; add NSG allow rules; configure Firewall default route and NAT.
**Validation:** VM can reach internet; Firewall logs show traffic.

### Scenario 2: "New VMs deployed via Blueprint don't have diagnostics"
**Architecture:** Blueprint deployment → VM created → Diagnostic settings expected via Policy → No logs.
**Dependencies:** Blueprint template, Policy with DeployIfNotExists, Diagnostic settings, Log Analytics workspace.
**Checks:**
1. Does Blueprint ARM template include diagnostic settings?
2. Is the policy with DeployIfNotExists for diagnostics assigned?
3. Does policy managed identity have RBAC to configure diagnostics?
4. Is Log Analytics workspace provisioned and accessible?
5. Did Blueprint deployment complete before policy evaluation ran?
6. Is the diagnostic settings policy in the correct scope?
**Root Cause: Diagnostic settings policy not assigned at subscription scope, OR Blueprint template doesn't include diagnostic settings and policy hasn't remediated yet.**
**Fix:** Ensure policy assigned at subscription scope; run remediation; if Blueprint includes diagnostic settings, update Blueprint.
**Validation:** VMs have diagnostic settings configured; logs flowing to Log Analytics.

### Scenario 3: "Spoke-to-Spoke traffic should bypass Firewall but doesn't"
**Architecture:** Spoke-1 → Hub → Firewall (forced) → Spoke-2 (should be direct but isn't).
**Dependencies:** VNet peerings, UDRs, NSGs, Firewall Manager traffic flow rules.
**Checks:**
1. UDR on Spoke-1: does it point to Firewall for Spoke-2 address space?
2. UDR on Spoke-2: does it point to Firewall for Spoke-1 address space?
3. VNet peering: is Spoke-1 ↔ Hub enabled? Is Spoke-2 ↔ Hub enabled?
4. NSGs: do they allow traffic between the address spaces?
5. Firewall Manager: traffic flow rules forcing transit through Firewall?
**Root Cause: UDRs or NSGs force traffic through Firewall. By default, VNet peering allows direct Spoke-to-Spoke, but UDRs override the default route.**
**Fix:** Remove UDR forcing traffic through Firewall for inter-Spoke communication. Or, if Firewall inspection is required, accept the architecture.
**Validation:** Spoke-1 can reach Spoke-2 directly (test connectivity). Firewall logs show reduced internal traffic.

### Scenario 4: "Teams can't deploy in their subscription — everything is denied"
**Architecture:** Team has Contributor RBAC → Policy evaluation → Deny → All denied.
**Dependencies:** Policy assignments, parameters, RBAC, scope.
**Checks:**
1. What policies are in Deny mode at the subscription?
2. Are policies from parent MGs inherited?
3. Are policy parameters too restrictive (e.g., only 1 allowed region, 1 allowed SKU)?
4. Are required tags enforced as Deny but team can't add tags?
5. Did the policy recently change (was previously Audit)?
**Root Cause: Policy in Deny mode is blocking deployments because resources don't meet policy conditions (missing tags, wrong region, disallowed SKU).**
**Fix:** Update policy parameters, change to Audit mode for less critical policies, or update deployment to comply with policy.
**Validation:** Teams can deploy; Policy compliance shows compliant for deployed resources.

### Scenario 5: "Spoke VM cannot resolve internal DNS name but can resolve internet domains"
**Architecture:** VM → DNS resolver → Azure Private DNS Zone vs. external DNS.
**Dependencies:** Private DNS zone, VNet peering, DNS resolver, Firewall DNS proxy.
**Checks:**
1. Private DNS zone: Is the zone for the internal domain linked to the Spoke VNet?
2. Private DNS Zone: Is the link in "Registered" or "Automatic" mode?
3. DNS resolution test: nslookup for internal name from VM
4. If Firewall DNS proxy: Is the DNS policy configured in Firewall?
5. Is the Private Endpoint for the service deployed and healthy?
6. NSG: Does Spoke allow DNS (port 53) to Hub/DNS?
**Root Cause: Private DNS zone not linked to Spoke VNet — most common DNS issue in Hub-and-Spoke.**
**Fix:** Link Private DNS zone to Spoke VNet; or use Azure DNS Private Resolver for cross-VNet DNS resolution.
**Validation:** VM can resolve internal DNS name; Private DNS zone shows correct link.

### Scenario 6: "Cost is soaring in Production subscription; unexpected billing"
**Architecture:** VMs deployed → Tags not applied → Cost untrackable; Oversized resources; Unattached disks.
**Dependencies:** Tagging policy, auto-shutdown, VM utilization, disk management.
**Checks:**
1. Cost Analysis: by service, tag, and resource group — where is the cost?
2. New resources: Resource Graph query for recent deployments
3. VM sizes: oversized for actual utilization (Advisor recommendations)
4. Unattached disks: Resource Graph query for disks not attached to any VM
5. Auto-shutdown: is it enabled for dev/test resources?
6. Tags: are resources properly tagged for cost allocation?
7. Reserved instances: should some VMs be reserved instead of pay-as-you-go?
8. Storage tier: are infrequently accessed resources in Premium tier?
**Root Cause (common):** Unattached disks (unused VMs or oversized VMs + dev/test running 24/7 without auto-shutdown).
**Fix:** Resize VMs, delete unattached disks, enable auto-shutdown for non-production, tag resources, purchase reserved instances.
**Validation:** Cost returns to expected range; Advisor no longer shows optimization recommendations.

### Scenario 7: "New workload deployed in Spoke but cannot reach any other service"
**Architecture:** New VM in Spoke → Needs to access: other VMs, PaaS, internet → All blocked.
**Dependencies:** NSGs, UDRs, Firewall rules, DNS, Private Link, Bastion.
**Checks:**
1. NSG on new VM subnet: Are inbound/outbound rules configured?
2. NSG on target subnets: Do they allow traffic from source subnet?
3. UDRs: Does Spoke have correct routes?
4. Firewall: Does it have rules allowing traffic?
5. DNS: Can the new VM resolve all hostnames?
6. Private Link: Are Private Endpoints deployed for required PaaS?
7. Bastion: Can we RDP into VM to troubleshoot?
8. NSG Flow Logs: Where is traffic being blocked?
**Root Cause: NSG on new VM's subnet has overly restrictive rules — typically deployed with default-deny NSG that allows nothing.**
**Fix:** Add NSG rules for required traffic (inbound from Hub/App, outbound to internet/PaaS). Use NSG flow logs to verify.
**Validation:** New VM can reach all required services; NSG flow logs show allowed traffic.

### Scenario 8: "Azure Bastion sessions are extremely slow for VMs in remote Spokes"
**Architecture:** User → Bastion → Spoke VM (cross-region or high latency).
**Dependencies:** Bastion SKU, network latency, Bastion configuration, VM size.
**Checks:**
1. Bastion SKU: Basic vs Standard/Premium (throughput differs)
2. Is Bastion in same region as user?
3. Is Bastion in same region as VM? (Bastion in Hub; VM in remote Spoke → latency)
4. Network latency: express route/VPN to on-prem? Internet latency to Bastion?
5. VM size: high CPU on VM affects session performance
6. Session recording: enabled? (adds overhead)
7. Number of concurrent Bastion sessions: SKU limit reached?
**Root Cause: Bastion in different region from target VM, or Bastion SKU too low for concurrent load, or VM under heavy load.**
**Fix:** Deploy Bastion in same region as VMs, scale up Bastion SKU, optimize VM performance, disable session recording if not needed.
**Validation:** Bastion sessions responsive; performance metrics acceptable.

### Scenario 9: "Blueprint deployed but RBAC assignments don't work"
**Architecture:** Blueprint assignment → Role assignment artifact → User can't access resources.
**Dependencies:** Blueprint role assignment, RBAC, principal identity, scope.
**Checks:**
1. Did Blueprint deployment complete? Check deployment operations for role assignment artifact.
2. Does the role assignment reference a valid principal? (principal still exists in Entra ID?)
3. Is the role assigned at the correct scope?
4. Does the role definition exist? (custom role deleted after Blueprint deployment?)
5. RBAC propagation: has 15 minutes elapsed?
6. Does the assigned principal have the role via direct assignment or group?
**Root Cause: Blueprint role assignment failed — principal doesn't exist, principal ID changed (app registration re-created), or scope incorrect. Blueprint artifacts can fail individually.**
**Fix:** Review Blueprint deployment operations; fix role assignment manually; ensure principal exists with correct Object ID.
**Validation:** Role assignment exists and is effective; principal can access resource.

### Scenario 10: "Audit logs for production subscription stopped flowing to central Log Analytics"
**Architecture:** Subscription → Activity Log → Diagnostic Settings → Log Analytics → Monitoring.
**Dependencies:** Diagnostic settings, Log Analytics workspace, subscription linkage, permissions.
**Checks:**
1. Diagnostic Settings on subscription: Activity Log → Enabled? Destination correct?
2. Log Analytics workspace: still running? Not deleted? In same subscription?
3. Subscription: Diagnostic settings are subscription-level, not RG-level (must be at subscription scope for Activity Log)
4. Permissions: Who configured the diagnostic settings? Was it removed?
5. Activity Log: Are there new operations? (If Activity Log is empty, diagnostic settings are the issue)
6. Log Analytics: Data flow stopped? Workspace at capacity?
**Root Cause: Diagnostic Settings for Activity Log was removed/disabled — often happens when someone changes subscription configuration without understanding dependencies.**
**Fix:** Re-enable Activity Log diagnostic settings; direct to correct Log Analytics workspace; verify data flow.
**Validation:** Activity Log data flowing to Log Analytics; queries return recent entries.

---

## 17. KNOWLEDGE TEST

1. **What is a Landing Zone?**
   An architectural framework for organizing Azure resources into a governed, scalable, secure environment using hub-and-spoke networking, centralized shared services, and standardized governance.

2. **What is the difference between Hub and Spoke VNets?**
   Hub VNet contains shared infrastructure (Firewall, DNS, Bastion, Gateway). Spoke VNets contain workloads. All Spoke traffic routes through the Hub for centralized security inspection.

3. **Why is Azure Firewall placed in the Hub VNet?**
   Centralized egress/ingress filtering for ALL Spoke VNets. One Firewall serves the entire estate — no duplication, consistent enforcement, centralized visibility.

4. **What is User Defined Routing (UDR) and why is it critical?**
   UDRs override default Azure routing. In a Landing Zone, UDRs on Spoke subnets point default traffic (0.0.0.0/0) to the Firewall in the Hub, ensuring all egress traffic is inspected.

5. **What is Azure Bastion and why is it important?**
   Bastion provides browser-based RDP/SSH to VMs without public IPs. VMs in Spokes can be accessed securely through Bastion in the Hub, eliminating the need for public IPs and open NSG rules.

6. **What is the difference between CanNotDelete and ReadOnly locks?**
   CanNotDelete prevents deletion only (modify still allowed). ReadOnly prevents all write operations (modify, delete blocked; read still allowed).

7. **What are the seven Landing Zones?**
   Platform, Application, Management, Identity, Connectivity, Security, and Governance. Each defines a category of infrastructure and services.

8. **Why should all diagnostic settings be configured via Policy?**
   To ensure no resource operates without logging. Manual configuration is error-prone and doesn't scale. Policy with DeployIfNotExists automatically remediates missing settings.

9. **How does cost management work in Landing Zones?**
   Mandatory tagging (enforced by Policy), cost allocation by subscription/RG/tag, budget alerts, Cost Management analysis, showback/chargeback reports.

10. **What is the difference between ExpressRoute and VPN?**
    ExpressRoute is a dedicated, private, SLA-backed connection (higher cost, better performance). VPN is an encrypted tunnel over the internet (lower cost, variable performance).

11. **Why shouldn't Spokes have direct peering to each other?**
    Direct peering bypasses centralized security inspection (Firewall). All traffic should flow through the Hub for consistent security enforcement and visibility.

12. **What role does PIM play in Landing Zones?**
    PIM ensures admin roles (especially Global Admin) are time-bound and approved, not permanently active. This is a critical Identity Landing Zone security control.

13. **What is the Management Landing Zone responsible for?**
    Centralized monitoring, logging, alerting, cost management, and operational management for the entire estate. Typically houses Log Analytics, alerts, dashboards, and Sentinel.

14. **What is Virtual WAN and when should it be used?**
    Virtual WAN provides hub-and-spoke connectivity across Azure regions and integrates with ExpressRoute/VPN/branch offices. Use it for multi-region or large-scale connectivity.

15. **What is the primary purpose of the Governance Landing Zone?**
    Policy enforcement, RBAC management, Blueprint deployment, and Management Group governance applied consistently across all subscriptions and resource groups.

16. **Why is DNS important in the Platform Landing Zone?**
    Private DNS zones resolve internal service names and Private Link endpoint names without exposing them to the public internet. DNS management is critical for secure internal communication.

17. **What is the difference between a Policy Audit and Deny effect in terms of Landing Zones?**
    Audit flags non-compliant resources for awareness but allows them. Deny blocks non-compliant resources entirely. In Production Landing Zones, critical policies should use Deny.

18. **What is Firewall Manager and what does it provide?**
    Centralized management of multiple Azure Firewalls including reusable firewall policies, threat intelligence, TLS inspection, and security stacking.

19. **How does Private Link improve security in Landing Zones?**
    Private Link allows PaaS services to be accessed via private endpoints in VNet IP space, completely bypassing the public internet and eliminating the need for public IPs on PaaS services.

20. **What is the recommended approach for onboarding a new team in a Landing Zone architecture?**
    Assign them to a pre-created Spoke via group-based RBAC, have them deploy through an approved Blueprint, let Policy enforce standards (tags, locations, SKUs), and ensure monitoring is already configured.

21. **What happens if the Hub VNet goes down?**
    All Spoke VNets lose centralized Firewall protection, DNS resolution (if hub-based), Bastion access, and VPN/ExpressRoute connectivity. Shared services become unavailable. Spoke-to-Spoke and Spoke-to-internet traffic fails. This is why Hub redundancy is critical.

22. **What are Private Endpoint and Private Link Zone?**
    Private Endpoint: An IP address in your VNet that connects privately to a PaaS service. Private Link Zone: A DNS zone (e.g., privatelink.blob.core.windows.net) that resolves Private Endpoint IPs.

23. **What is the difference between Front Door and Application Gateway?**
    Front Door is global (multi-region) HTTP/S load balancing with WAF and CDN — best for internet-facing multi-region apps. Application Gateway is regional Layer 7 load balancer with WAF — best for single-region web apps.

24. **Why should you NOT assign RBAC directly to individual users in Landing Zones?**
    Centralized group-based management. When a person changes roles or leaves, group membership changes automatically revoke access. Individual assignments require manual cleanup and create audit complexity.

25. **What is the primary metric to monitor Landing Zone health?**
    Policy compliance percentage, log ingestion volume and freshness, Firewall health, DNS resolution success rate, and ExpressRoute/VPN connection health.

---

## 18. L3 GAP CHECK

| Topic | Status |
|-------|--------|
| Landing Zone concept and rationale | ✅ Covered |
| Platform Landing Zone (Hub/Spoke, Firewall, DNS, Bastion) | ✅ Covered |
| Application Landing Zone (workload VMs, Spoke VNets, NSGs) | ✅ Covered |
| Management Landing Zone (monitoring, Log Analytics, alerts) | ✅ Covered |
| Identity Landing Zone (Entra ID, Directory Roles, PIM, CA) | ✅ Covered |
| Connectivity Landing Zone (ExpressRoute, VPN, Virtual WAN) | ✅ Covered |
| Security Landing Zone (Defender, NSG, Firewall, Key Vault) | ✅ Covered |
| Governance Landing Zone (Policy, RBAC, Blueprints, MGs) | ✅ Covered |
| Subscription structure | ✅ Covered |
| Management Group hierarchy | ✅ Covered |
| Centralized logging | ✅ Covered |
| Shared services pattern | ✅ Covered |
| UDR and traffic flow | ✅ Covered |
| Private Link/Private Endpoint/DNS | ✅ Covered |
| Cost management | ✅ Covered |
| Security considerations | ✅ Covered |
| Monitoring targets | ✅ Covered |
| Production example | ✅ Covered |
| Failure scenarios (10 scenarios) | ✅ Covered |
| Troubleshooting methodology | ✅ Covered |
| Logs/evidence | ✅ Covered |
| Current service considerations | ✅ Covered |
| L3 interview questions (all levels) | ✅ Covered |
| Scenario-based questions | ✅ Covered |
| Knowledge test (25 questions) | ✅ Covered |
| L3 gap check | ✅ Covered |

---

## WHAT AN EXPERIENCED AZURE L3 ENGINEER SHOULD NOW BE ABLE TO EXPLAIN CONFIDENTLY

After Module 9, you should be able to confidently explain:

1. **The seven Landing Zones and how they form a complete Azure estate** — Platform (shared infrastructure), Application (workloads), Management (monitoring), Identity (who accesses what), Connectivity (network connections), Security (threat protection), and Governance (standards enforcement). Each Landing Zone is a pillar; together they form the complete architecture. Any single pillar being missing or weak compromises the entire estate.

2. **The hub-and-spoke model in detail** — Hub VNet in a Platform subscription houses Firewall, DNS, Bastion, and Gateways. Spoke VNets in Application subscriptions house workloads. UDRs force all Spoke traffic through the Hub for centralized inspection. Spokes are NOT directly peered to each other (traffic transits through Hub). This model provides consistent security at scale.

3. **Why Firewall is the cornerstone of the Platform Landing Zone** — It provides centralized egress/ingress filtering, FQDN-based rules, threat intelligence, and traffic logging for the entire estate. Without it, every Spoke would need its own NVA, consistency would be impossible, and visibility would be lost. Firewall Manager adds centralized policy management for multiple Firewalls.

4. **How Private Link and Private DNS work together** — Private Endpoints give PaaS services a VNet IP address. Private DNS Zones resolve that endpoint to the private IP. Together, they eliminate the need for public access to PaaS services. Traffic goes directly to the PaaS service via the Azure private backbone, completely bypassing the public internet and the Firewall.

5. **The critical role of UDRs** — Without UDRs, Azure uses default routing (direct internet access from each Spoke). With UDRs, all traffic is forced through the Firewall in the Hub. UDRs are the mechanism that makes hub-and-spoke work. Misconfigured UDRs are the #1 cause of "VM lost internet" after Firewall deployment.

6. **Management Group hierarchy as governance backbone** — MG-Enterprise-Baseline (tenant-wide policies), MG-Corporate-Security (security and compliance), MG-Production (stricter operational), MG-Development (relaxed for flexibility). Policies cascade down from higher MGs. More specific (child) assignments add restrictions.

7. **How centralized logging works in the Management Landing Zone** — Diagnostic Settings on every resource route logs to a central Log Analytics Workspace (typically in a Platform/Monitoring subscription). This includes NSG flow logs, Firewall logs, Activity Logs, VM logs, and application logs. KQL queries cross-reference data across all resources. Without this, the estate operates blind.

8. **Identity Landing Zone and its relationship to RBAC** — Entra ID (directory) is separate from RBAC (Azure resource access). Directory Roles (Global Admin, Conditional Access Admin, etc.) manage Entra ID. RBAC roles (Owner, Contributor, Reader, etc.) manage Azure resources. Both must be configured independently. PIM makes privileged assignments time-bound and approved.

9. **How Blueprints deploy standard environments** — Blueprints version and deploy a package: ARM templates (infrastructure), role assignments (who can access), policy assignments (what's allowed), diagnostic settings (monitoring), and tags (cost tracking). Teams don't build from scratch — they deploy the Blueprint. This ensures consistency.

10. **Why Private Link bypasses Firewall** — By design, Private Link traffic goes directly from VNet to PaaS service via Azure's private backbone. It doesn't traverse the Firewall. This means FQDN-based Firewall rules don't apply to Private Link traffic (use NSGs on Private Endpoints instead).

11. **How cost management relies on governance** — Policy-enforced mandatory tagging ensures every resource has cost allocation tags. Cost Management analyzes by tag, service, RG, and subscription. Budget alerts proactively notify of spikes. Without governance (no tags, no policy), cost tracking is impossible across 5,000+ resources.

12. **The complete troubleshooting methodology for Landing Zone issues** — Identify the symptom → Determine the Landing Zone layer (Platform, Application, Connectivity, Security, Governance, Management) → Apply layer-specific checks → Cross-reference for multi-layer issues → Fix at identified layer → Validate and document.

---

# Ready for Module 10 — Data (Azure Storage, Azure SQL, Cosmos DB, Backup, Azure Files, Blob Storage, Data Lake, Synapse)?

It covers:
- Azure Storage types (Blob, File, Queue, Table)
- Storage account types and redundancy (LRS, ZRS, GRS, RA-GRS, GZRS)
- Blob storage tiers (Hot, Cool, Cold, Archive)
- Azure SQL (DB, Managed Instance, elastic pools)
- Cosmos DB (APIs, consistency levels, multi-region)
- Azure Backup and Azure Site Recovery
- Azure Files (SMB, NFS)
- Azure Data Lake Storage
- Azure Synapse Analytics
- Data Factory / Synapse Pipelines
- Private endpoints for data services
- Data encryption and security
- Cross-region replication and disaster recovery

Say **"Next module"** to continue.