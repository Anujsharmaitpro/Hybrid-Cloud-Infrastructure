# MODULE 12 — NETWORK SECURITY (NSG / FIREWALL / WAF), ROUTING, PRIVATE ENDPOINT / PRIVATE LINK, VPN GATEWAY, EXPRESSROUTE, VIRTUAL WAN — L3 DEPTH 🔴 CRITICAL

---

## 1. CONCEPT

**Azure Network Security and Connectivity** is the discipline of controlling who can reach what, how traffic flows between networks, and how to securely connect on-premises networks to Azure. This module covers the complete stack: from the most granular security layer (NSG at the NIC level) to the most strategic connectivity layer (Virtual WAN spanning global hubs).

> **The single most important network security concept:** Defense in depth — no single security layer is sufficient. NSG is the first filter (subnet/NIC level), Firewall is the centralized inspection layer (hub level), WAF is the application-layer protector (web tier), Private Endpoint is the access layer (no public exposure), VPN/ExpressRoute is the connectivity layer (on-prem bridge), and Virtual WAN is the global orchestration layer (multi-region connectivity). Each layer addresses a different threat vector; together they form an impenetrable fortress.

**Why Network Security design matters:**

```
WITHOUT PROPER NETWORK SECURITY DESIGN:
  → VMs exposed to internet (no NSG = everything open)
  → No centralized traffic inspection (each VM is its own firewall)
  → Web applications vulnerable to OWASP attacks (no WAF)
  → PaaS services accessible via public internet (no Private Link)
  → On-prem connectivity fragile (single VPN, no redundancy)
  → No routing control (traffic takes default paths, no NVA inspection)
  → Global connectivity chaotic (no hub-and-spoke, random peering)
  → Security rules scattered (100 NSGs with 500 rules each = chaos)
  → Compliance failures (no audit trail of network access)
  → Cost overruns (oversized gateways, unnecessary peering)
  → Troubleshooting nightmare (500 VMs, 1000 NSG rules, no flow logs)

WITH PROPER NETWORK SECURITY DESIGN:
  → VMs isolated in subnets with NSGs (tier-based security)
  → Centralized inspection via Firewall (all traffic through NVA)
  → WAF protects web apps (OWASP Top 10 blocked automatically)
  → PaaS accessed via Private Endpoint (no public exposure)
  → On-prem connected via ExpressRoute + VPN (active/active DR)
  → UDRs force traffic through Firewall (predictable routing)
  → Virtual WAN orchestrates global connectivity (hub-and-spoke)
  → NSG Flow Logs provide full audit trail
  → Compliance proven (network segmentation, access controls)
  → Cost efficient (right-sized gateways, no unnecessary peering)
  → Troubleshooting by design (flow logs, effective rules, connection monitor)
```

**Network Security Architecture:**

```
┌─ INTERNET ─┐
│             │
│   User → → → Front Door / Application Gateway (WAF)
│              ↓
│         NSG (Web Subnet): Allow 443 from Internet
│              ↓
│         App Tier (App Subnet): NSG allows only from Web
│              ↓
│         Firewall (AzureFirewallSubnet): ALL traffic inspected
│              ↓
│         Data Tier (Data Subnet): NSG allows only from App
│              ↓
│         PaaS (SQL, Storage): Private Endpoint only
│              ↓
│         On-Prem: VPN Gateway (backup) / ExpressRoute (primary)
│              ↓
│         Other Azure Regions: Virtual WAN / VNet Peering
└─────────────┘

Security Layers (Defense in Depth):
  Layer 1: NSG (SubNet + NIC level)          → First filter, granular
  Layer 2: Azure Firewall (NVA in Hub)        → Centralized inspection
  Layer 3: Application Gateway WAF            → L7 web protection
  Layer 4: Private Endpoint (no public IP)     → No public exposure
  Layer 5: DDoS Protection (Basic/Standard)    → Volumetric/protocol attacks
  Layer 6: Azure Front Door (global WAF)       → Global L7 protection
  Layer 7: DNS Security (Resolver, Firewall)   → DNS-based threat blocking
  Layer 8: Micro-segmentation (NSG per VM)    → VM-level isolation
```

---

## 2. ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        NETWORK SECURITY & CONNECTIVITY ARCHITECTURE              │
│                                                                                  │
│  ┌─ NSG LAYER (Granular Filtering) ──────────────────────────────────┐         │
│  │                                                                     │         │
│  │  Subnet NSGs: Tier-based allow/deny rules                          │         │
│  │  NIC NSGs: Per-VM exceptions                                       │         │
│  │  Default Rules: VNet-inbound, VNet-outbound, Internet deny          │         │
│  │  Priority: 100-4095 (user) + 65500-65550 (Azure default)           │         │
│  │  First match wins, lowest priority number first                     │         │
│  │  NSG Flow Logs: IP-level audit of all allow/deny decisions          │         │
│  │                                                                     │         │
│  └─────────────────────────────────────────────────────────────────────┘         │
│                                                                                  │
│  ┌─ FIREWALL LAYER (Centralized NVA) ────────────────────────────────┐         │
│  │                                                                     │         │
│  │  Azure Firewall (Standard/Premium SKU in AzureFirewallSubnet)      │         │
│  │  → SNAT (outbound internet via public IP)                          │         │
│  │  → DNAT (inbound from internet to VMs)                            │         │
│  │  → Threat Intelligence (alert/deny known bad IPs)                  │         │
│  │  → FQDN-based rules (HTTP/HTTPS domain filtering)                  │         │
│  │  → TLS inspection (premium)                                        │         │
│  │  → IDPS (premium — signature-based attack detection)               │         │
│  │  → Application Rules (FQDN + port)                                 │         │
│  │  → Network Rules (IP/CIDR + port/protocol)                         │         │
│  │  Firewall Manager: Central policy across multiple firewalls        │         │
│  │  Security Stacking: Distributed firewall in spoke subnets          │         │
│  │                                                                     │         │
│  └─────────────────────────────────────────────────────────────────────┘         │
│                                                                                  │
│  ┌─ WAF LAYER (L7 Web Protection) ───────────────────────────────────┐         │
│  │                                                                     │         │
│  │  Application Gateway v2 (WAF endpoint)                             │         │
│  │  → OWASP Top 10 protection (SQL injection, XSS, etc.)              │         │
│  │  → Custom rules + managed rule sets (MISE, CRS)                    │         │
│  │  → Rate limiting (DDoS protection at app layer)                    │         │
│  │  → SSL/TLS termination + redirection                                │         │
│  │  → Bot management (preview)                                        │         │
│  │  → Multi-site hosting (host header routing)                         │         │
│  │  → Auto-scaling (v2 with autoscale)                                │         │
│  │                                                                     │         │
│  └─────────────────────────────────────────────────────────────────────┘         │
│                                                                                  │
│  ┌─ PRIVATE ENDPOINT / PRIVATE LINK (No Public Access) ──────────────┐         │
│  │                                                                     │         │
│  │  Private Endpoint: NIC in VNet with private IP                     │         │
│  │  Private DNS Zone: Resolves service names to private IPs           │         │
│  │  Private Link Service: Azure PaaS services (SQL, Storage, etc.)    │         │
│  │  No public endpoint needed; traffic stays on Azure backbone        │         │
│  │  Cross-tenant access supported                                     │         │
│  │  Connection approval model (endpoint owner approves access)        │         │
│  │                                                                     │         │
│  └─────────────────────────────────────────────────────────────────────┘         │
│                                                                                  │
│  ┌─ VPN GATEWAY (Site-to-Site / P2S) ────────────────────────────────┐         │
│  │                                                                     │         │
│  │  Site-to-Site VPN: VNet ↔ On-prem (IPsec/IKE, UDP 500/4500)      │         │
│  │  Point-to-Site VPN: VM/Client ↔ VNet (openvpn, IKEv2)             │         │
│  │  SKUs: Basic, VpnGw1-5, HighPerformanceVPNGW                       │         │
│  │  BGP support (dynamic routing)                                     │         │
│  │  Zones: Zone-redundant (HA across AZs)                             │         │
│  │  Active/Active + Active/Passive configurations                     │         │
│  │  Breakout scenarios: VNet-to-VNet VPN (transit)                    │         │
│  │                                                                     │         │
│  └─────────────────────────────────────────────────────────────────────┘         │
│                                                                                  │
│  ┌─ EXPRESSROUTE (Dedicated Private Connectivity) ───────────────────┐         │
│  │                                                                     │         │
│  │  Dedicated circuit: On-prem ↔ Azure (not over internet)            │         │
│  │  SKUs: Standard (10G), ExpressRoute 10/40/100G                    │         │
│  │  Redundancy: Fiber-level + provider-level + geo-redundant          │         │
│  │  BGP (dynamic routing) — required for production                   │         │
│  │  Gateways: ER Gateway (ARM-based, recommended) / Classic ER GW     │         │
│  │  Microsoft Peering: Azure services, Microsoft 365, DAC              │         │
│  │  Public Peering: Azure public services                             │         │
│  │  Private Peering: Azure private services (SQL, Storage, etc.)      │         │
│  │  Global Reach: ER circuits connected across locations              │         │
│  │  FastPath: Bypass gateway for ultra-low latency                    │         │
│  │                                                                     │         │
│  └─────────────────────────────────────────────────────────────────────┘         │
│                                                                                  │
│  ┌─ VIRTUAL WAN (Global Orchestration) ───────────────────────────────┐         │
│  │                                                                     │         │
│  │  Virtual WAN Hub: Central routing node (ARM-based, recommended)     │         │
│  │  → VNet attachment (spoke VNets connect to hub)                    │         │
│  │  → VPN Gateway attachment (VPN connects to hub)                   │         │
│  │  → ExpressRoute Gateway attachment (ER circuit connects to hub)   │         │
│  │  → User VPN P2S connection (direct client connection to hub)      │         │
│  │  → Branch (erDC appliance) — software NVA in hub                  │         │
│  │  → Site-to-Site connections (IPsec between branches and hub)       │         │
│  │  → Transit: Spoke-to-spoke via hub (no direct peering needed)      │         │
│  │  → BYO Hub: Bring Your Own (use existing VNet as Virtual WAN hub) │         │
│  │  → QoS policies for traffic prioritization                        │         │
│  │  → Security partners: Automated firewall/security appliance attach │         │
│  │                                                                     │         │
│  └─────────────────────────────────────────────────────────────────────┘         │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. COMPONENTS — DETAILED

### 3.1 NETWORK SECURITY GROUPS (NSG) — COMPLETE DEEP DIVE

> **Note:** NSG is covered in Module 10 (VNet Deep Dive) at full L3 depth. This section provides the Network Security perspective with emphasis on NSG as the first layer of defense in the network security stack.

**NSG as Defense Layer 1:**

```
NSG Position in Security Stack:

  Internet → NSG (SubNet/NIC) → Firewall (NVA) → Target Resource
              ↓
         First filter: decides what traffic enters/exits
              ↓
         If denied: traffic stopped (NSG Flow Log records deny)
              ↓
         If allowed: traffic forwarded to Firewall (via UDR)
              ↓
         Firewall inspects: decides what traffic passes to resource
              ↓
         If denied: traffic stopped (Firewall log records deny)
              ↓
         If allowed: traffic reaches target resource

NSG is the FIRST line of defense. If NSG blocks traffic, it never reaches Firewall.
This means NSG rules should be LEAST RESTRICTIVE (allow what NSG-level can permit).
Firewall should be MORE RESTRICTIVE (block what you don't explicitly want through).

Common mistake: NSG allows everything (0.0.0.0/0 Allow *), then Firewall does all filtering.
Best practice: NSG blocks obvious bad traffic (Internet → Data subnet), Firewall does detailed filtering.
```

**NSG Flow Logs — CRITICAL FOR SECURITY AUDITING:**

```
NSG Flow Logs Capture:
  → All IP traffic allowed/denied by NSGs
  → Source IP, Destination IP, Source Port, Dest Port
  → Protocol, Action (Allow/Deny), Rule that matched
  → Flow start/end timestamps, packet count, byte count
  → Interface direction (Ingress/Egress)
  → WARNING: Does NOT capture traffic that bypasses NSG (e.g., same-VNet traffic if "Allow VNet Access" is default)

Flow Logs for Security Operations:
  1. Threat hunting: Unusual access patterns (e.g., VM accessing unexpected IPs)
  2. Compliance proof: Demonstrate network segmentation (PCI, HIPAA, etc.)
  3. Incident response: Trace attacker lateral movement (which VMs communicated)
  4. Firewall validation: Verify Firewall is seeing all traffic (no bypass)
  5. Network forensics: Reconstruct traffic flows during incident

Without Flow Logs: "Someone accessed my database from an unknown IP" becomes untraceable.
With Flow Logs: "SQL DB (10.0.3.4) received 1433 connections from 203.0.113.50 (unknown) at 3 AM, denied by NSG Rule #500"
```

---

### 3.2 AZURE FIREWALL — COMPLETE DEEP DIVE

**What is Azure Firewall?**
Azure Firewall is a fully stateful cloud-native network security service that protects your Virtual Network resources. It is a managed, scalable, and highly available service with built-in high availability and unlimited scale.

**Firewall SKU:**

| SKU | Use Case | Features |
|-----|----------|----------|
| **Basic** | Dev/test, low-throughput, simple filtering | SNAT, DNAT, Threat Intelligence (alert mode only), FQDN rules (limited), 250 rule limits |
| **Standard** | Production, centralized network security | Full SNAT/DNAT, Threat Intelligence (Alert+Deny), FQDN rules, TLS inspection (preview), IDPS (preview), 1000+ rule limits |
| **Premium** | Maximum security, compliance-critical environments | All Standard features + TLS 1.3 inspection, IDPS (full), SSL/TLS decryption, 5000+ rule limits, Bot protection |

> **L3 critical:** Basic SKU Firewall cannot be deployed in a Standard VNet (it's the reverse — Standard SKU REQUIRES Standard VNet). Basic SKU has limited rule capacity (250 rules). For production, ALWAYS use Standard or Premium.

**Firewall Architecture:**

```
Azure Firewall in AzureFirewallSubnet (10.0.6.0/27 — Standard VNet required):

  ┌─ Firewall Instance 1 (10.0.6.4) ──┐
  │  → Public IP: 20.1.1.1 (for SNAT/DNAT)  │
  │  → Management IP: 10.0.6.4 (for admin)   │
  │  → HA: Active/Standby or Active/Active  │
  │  → UDR: 0.0.0.0/0 → Firewall (all egress) │
  │  → SNAT: 10.0.6.4-10.0.6.10 (SNAT pool)  │
  └──────────────────────────────────────┘
  ┌─ Firewall Instance 2 (10.0.6.5) ──┘
  │  → HA pair with Instance 1
  │  → Active/Active or Active/Standby
  │  → Separate public IPs for each (if Active/Active)
  └──────────────────────────────────────┘

Traffic Flow — Egress (VM → Internet):
  VM (10.0.1.4) → NSG allows → UDR: 0.0.0.0/0 → Firewall (10.0.6.4)
  → Firewall performs SNAT (source IP becomes 20.1.1.1)
  → Firewall evaluates Application Rules (FQDN-based)
  → Firewall evaluates Network Rules (IP/CIDR-based)
  → Firewall evaluates Threat Intelligence (blocked IPs)
  → Traffic forwarded to internet
  → Return traffic: Firewall performs stateful reverse DNAT
  → NSG on VM allows return traffic (ephemeral ports)

Traffic Flow — Ingress (Internet → VM via DNAT):
  Internet → Public IP:20.1.1.1:443 → Firewall
  → Firewall DNAT rule: 20.1.1.1:443 → 10.0.1.4:443
  → Firewall Application Rule: Allow FQDN "app.contoso.com"
  → Firewall Network Rule: Allow port 443
  → UDR on Web Subnet: 10.0.0.0/16 → VNet local (direct to VM)
  → NSG on VM: Allow 443 from Internet (or from Firewall subnet)
  → VM receives traffic from Firewall (not directly from Internet)

Key Insight: DNAT + UDR combination is REQUIRED for inbound traffic.
  DNAT alone doesn't work because return traffic doesn't come back through Firewall.
  UDR ensures return traffic goes through Firewall for stateful return.
```

**Firewall Rules:**

| Rule Type | Scope | Description |
|-----------|-------|-------------|
| **Application Rules** | FQDN-based | Allow/deny HTTP/HTTPS traffic based on fully qualified domain names. E.g., Allow "*.contoso.com" on port 443. Evaluated before Network Rules. |
| **Network Rules** | IP/CIDR-based | Allow/deny traffic based on source/destination IP, port, protocol. E.g., Allow 203.0.113.0/24 to VM on port 22. |
| **Threat Intelligence** | IP/URL-based | Alert or Deny traffic to/from known malicious IPs/domains (Microsoft threat intelligence feeds). |
| **NAT Rules** | IP/port-based | DNAT rules for inbound traffic (map public IP:port → private IP:port). |

**Application Rule Details:**

```
Application Rule Example:
  Rule Name: Allow-Office365
  Source: 10.0.1.0/24 (Web Subnet)
  Protocol: TCP:443
  Target FQDN: *.office.com, *.microsoft.com
  Action: Allow
  Priority: 100

  Rule Name: Block-SocialMedia
  Source: *
  Protocol: TCP:443
  Target FQDN: *.facebook.com, *.twitter.com
  Action: Deny
  Priority: 200

  Rule Name: Allow-AllHTTP
  Source: 10.0.1.0/24
  Protocol: TCP:80
  Target FQDN: *
  Action: Allow
  Priority: 300

Important Notes:
  → FQDN rules only work for HTTP/HTTPS (port 80/443)
  → Wildcard FQDNs: *.contoso.com matches a.contoso.com but NOT a.b.contoso.com (different match level)
  → FQDN resolution: Firewall resolves FQDN to IP every 5 minutes (configurable); if IP changes, rule breaks until next resolution
  → DNS proxy: If Firewall is DNS proxy (Firewall + AzureFirewallDns subnet), FQDN resolution is more accurate
  → Intended FQDN: For Azure PaaS (e.g., *.database.windows.net) — resolves to stable Azure IPs
  → Fully qualified FQDN: For internet domains (e.g., www.contoso.com) — resolves to current IP
```

**Firewall DNS Proxy:**

```
Azure Firewall DNS Proxy (requires AzureFirewallDns subnet):

  Without DNS Proxy:
    VM DNS setting: Custom DNS server (e.g., 10.0.0.4)
    → Firewall cannot inspect DNS queries
    → FQDN-based Application Rules may fail (no DNS visibility)

  With DNS Proxy:
    VM DNS setting: 10.0.6.4 (Firewall IP)
    → All DNS queries go through Firewall
    → Firewall resolves FQDNs for Application Rules
    → Firewall can also forward DNS to custom server (Conditional Forwarding)

  Architecture:
    VM (10.0.1.4) DNS query → 10.0.6.4 (Firewall DNS Proxy)
    → Firewall resolves: app.contoso.com → 13.107.42.14
    → Application Rule: Allow app.contoso.com on 443 → Match (13.107.42.14:443 allowed)
    → VM connects to 13.107.42.14:443 (through Firewall)

  DNS Proxy + Conditional Forwarding:
    VM DNS → Firewall (10.0.6.4)
    → Firewall forwards: *.contoso.com → 10.0.0.4 (on-prem DNS)
    → Firewall forwards: *.azure.com → 168.63.129.16 (Azure DNS)
    → Firewall resolves everything for FQDN rules

  Required Subnet: AzureFirewallDns (10.0.7.0/27)
    → Minimum /27 (32 addresses)
    → Must be named exactly "AzureFirewallDns"
    → Only used for DNS proxy functionality
```

**Firewall Manager:**

```
Azure Firewall Manager centralizes security policy management:

  Components:
    1. Firewall Policy: Centralized rule collection
       → Application Rules (shared across all firewalls)
       → Network Rules (shared across all firewalls)
       → Threat Intelligence (shared across all firewalls)
       → SSL/TLS settings (shared)
       → IDS/IPS settings (shared)

    2. Security Stance: How firewalls interact (recommended/required)
       → Recommended: Use Firewall policy as guideline; local rules can supplement
       → Required: Only Firewall policy rules apply (strict central control)

    3. Firewall Automation: Auto-deploy firewall in spokes
       → Auto-create UDR, NSG, and Firewall in spoke subnets
       → Security stacking: Each spoke gets its own Firewall for distributed inspection

    4. Threat Intelligence: Centralized feed management
       → Alert mode: Log only
       → Deny mode: Block
       → Auto-update: Microsoft updates threat feeds automatically

  Benefits:
    → Single policy for 100+ firewalls (consistent security)
    → Reduce rule duplication (DRY across firewalls)
    → Central monitoring (all firewall logs in one place)
    → Rapid deployment (new spoke gets firewall policy automatically)
    → Audit trail (who changed what, when)
```

**Firewall Threat Intelligence:**

```
Threat Intelligence Lists:
  → Sources: Microsoft Threat Intelligence (Global Threat Analysis and Geopolitical Threat lists)
  → IP Lists: Known malicious IPs (C2 servers, botnets, phishing)
  → URL Lists: Known malicious domains and URLs
  → Geo-IP Lists: Traffic from high-risk countries (optional)

Modes:
  → Alert Mode: Log when traffic matches threat intel, but ALLOW traffic through
    Use when: Just learning about threats, no disruption
  → Deny Mode: BLOCK traffic that matches threat intel
    Use when: Confident in threat intelligence, want strict blocking

  Transition Strategy:
    Phase 1: Alert mode (1-2 weeks) — understand what's being blocked
    Phase 2: Analyze logs — identify false positives
    Phase 3: Exclude false positives (allow list)
    Phase 4: Switch to Deny mode

Important: Threat Intelligence works alongside (not instead of) Application/Network rules.
  If a rule says Allow but Threat Intel says Deny, DENy wins (lower priority = deny).
  Threat Intel priority: 100-200 (evaluated before user rules in most configurations).
```

---

### 3.3 APPLICATION GATEWAY — WAF — COMPLETE DEEP DIVE

**What is Application Gateway?**
Application Gateway is a web traffic load balancer that enables you to manage traffic to web applications. The WAF (Web Application Firewall) tier provides OWASP Top 10 protection.

**Application Gateway SKUs:**

| SKU | Use Case |
|-----|----------|
| **WAF_v2** | Production web apps with WAF protection, autoscale, zone-redundant |
| **WAF_v1** | Legacy, not recommended for new deployments |
| **Standard_v2** | L7 load balancing WITHOUT WAF |
| **Standard_v1** | Legacy L7 load balancing |

> **L3 critical:** Only WAF_v2 supports autoscale, private link, wildcard hostnames, and advanced features. Always use WAF_v2 for production.

**WAF Rule Sets:**

| Rule Set | Description |
|----------|-------------|
| **OWASP 3.2** (Default) | Core rules covering SQL injection, XSS, RFI, LFI, session fixation, HTTP protocol violations, etc. |
| **OWASP 3.4** (Newer) | Updated rules, more signatures, better false-positive handling |
| **Managed Rule Set (MRS)** | Microsoft-managed rules (includes OWASP + Microsoft-specific threats) |
| **Custom Rules** | User-defined rules for specific scenarios (allowlist/blocklist) |
| **Rate Limit Rules** | Limit requests per IP/time period (DDoS mitigation at L7) |

**WAF Operation Modes:**

```
Detection Mode (Audit):
  → WAF logs threats but ALLOWS traffic through
  → Use when: Rolling out new rules, tuning WAF to reduce false positives
  → No impact on legitimate traffic

Prevention Mode (Block):
  → WAF blocks threats AND logs them
  → Use when: WAF rules are tuned and production protection needed
  → May impact traffic if false positives exist (test first!)

Recommendation:
  Phase 1: Detection mode for 1-2 weeks (understand what WAF would block)
  Phase 2: Analyze logs, adjust exclusions (disable rules causing false positives)
  Phase 3: Switch to Prevention mode
  Phase 4: Monitor continuously
```

**Application Gateway Architecture:**

```
Application Gateway v2 (WAF):

  Frontend IP Configuration:
    → Public IP: 20.2.2.10 (internet-facing)
    → Private IP: 10.0.1.20 (internal-facing, if private endpoint)
    → Port: 80, 443 (HTTP/HTTPS listeners)

  Listener:
    → Protocol: HTTPS (TLS 1.2)
    → Host: app.contoso.com
    → SSL Certificate: app.contoso.com.pfx (uploaded to Key Vault or GW)

  Rule:
    → Type: Based on host header
    → Host: app.contoso.com
    → Action: Forward to Backend Pool (api-servers)

  Backend Pool:
    → Targets: App VMs (10.0.2.4, 10.0.2.5, 10.0.2.6)
    → Health Probe: HTTP:8080, /health (every 30 sec, 3 failures = unhealthy)
    → Load Balancing: Cookie-based affinity or 5-tuple

  WAF Configuration:
    → Rule Set: OWASP 3.2 (or Managed Rule Set)
    → Mode: Prevention
    → Custom Rules:
       → Allow: 203.0.113.0/24 on all paths (corporate IP allowlist)
       → Block: SQL injection patterns on /api/* (custom)
       → Rate Limit: 100 requests/minute per IP on /login (prevent brute force)

  Autoscale:
    → Min: 2 instances (HA minimum)
    → Max: 10 instances (handle traffic spikes)
    → Metric: Capacity units (CU) — 250 CU per instance
    → Scale trigger: CU > 70% for 5 minutes

  Private Link Integration:
    → Application Gateway supports Private Endpoint
    → Private IP: 10.0.1.20 in VNet
    → DNS: privatelink.azurewebsites.net → private IP
    → Access: VNet resources can reach App Gateway privately

  Integration with Front Door (Global WAF):
    → Front Door sits in front of Application Gateway (global entry point)
    → Front Door: Global L7 WAF, DDoS protection, CDN, SSL offloading
    → Application Gateway: Regional WAF, backend routing, health probes
    → Traffic: User → Front Door (global WAF) → App Gateway (regional WAF) → Backend VMs
```

**WAF Custom Rules Example:**

```
WAF Custom Rules (OWASP 3.2 Managed + Custom):

  Priority 100: Block SQL Injection (managed rule — SQL Injection (RS 942100))
    → Match: Request URI or Body contains SQL patterns
    → Action: Block
    → Log: Yes

  Priority 200: Block Cross-Site Scripting (managed rule — XSS (RS 941310))
    → Match: Request headers/body contain XSS patterns
    → Action: Block
    → Log: Yes

  Priority 300: Block Remote File Inclusion (custom)
    → Match: URI contains ../ or file= in query string
    → Action: Block
    → Log: Yes

  Priority 400: Allow corporate VPN IPs (custom — whitelist)
    → Match: Source IP 203.0.113.0/24
    → Action: Allow
    → Log: Yes

  Priority 500: Rate Limit login endpoint (custom)
    → Match: URI /api/login, 100 requests per 5 minutes per IP
    → Action: Block
    → Log: Yes

  Priority 1000: Allow all other traffic (managed — default)
    → Match: *
    → Action: Allow
    → Log: Yes
```

---

### 3.4 AZURE DDoS PROTECTION — COMPLETE DEEP DIVE

**What is Azure DDoS Protection?**
Azure DDoS Protection provides enhanced DDoS mitigation capabilities to Azure applications. It's automatically enabled for all Azure resources (Basic is free, Standard requires SKU).

| SKU | Protection Level | Cost | Use Case |
|-----|-----------------|------|---------|
| **Basic** | Always-on, 5 common attack types (connection, transport, resource) | Free (included) | Dev/test, simple apps |
| **Standard** | Enhanced, all Basic + custom alerts, 400 Gbps mitigation, WAF integration | ~$29/day per region | Production, compliance, financial apps |

**DDoS Protection Tiers:**

```
Always-On Protection (Basic — Free):
  → Automatic protection against:
    - SYN flood (transport layer)
    - UDP/ICMP flood (transport layer)
    - HTTP GET/POST flood (application layer)
    - Connection exhaustion (transport layer)
    - Resource exhaustion (application layer)
  → No configuration required
  → No alerting (just mitigates attacks automatically)

Standard (Paid):
  → All Basic protection +:
    - Adaptive tuning (learns traffic patterns, adjusts thresholds)
    - Custom alert rules (notify when attack detected)
    - Integration with Azure Monitor/Action Groups
    - WAF integration (Application Gateway)
    - Threat intelligence-based filtering
    - Higher mitigation capacity (up to 400 Gbps)
    - Detailed attack analytics
    - DDoS rapid response team engagement
```

**DDoS + WAF Combined Architecture:**

```
DDoS Protection (Standard) + WAF (Application Gateway v2):

  Internet → Front Door (global WAF + CDN) → DDoS Protection (Standard)
    → Application Gateway v2 (regional WAF) → Backend VMs

  Defense in Depth:
    Layer 1: Front Door — Global L7 WAF + DDoS mitigation at edge
    Layer 2: DDoS Standard — Enhanced volumetric and protocol attack mitigation
    Layer 3: Application Gateway WAF — L7 application protection
    Layer 4: NSG — L3/L4 filtering at subnet level
    Layer 5: Backend VMs — OS-level firewall, application security

  Alert Configuration:
    → Metric: DDoS Protection — Mitigated (Kbps)
    → Alert when: Mitigated traffic > 100 Mbps for 5 minutes
    → Action: Email, SMS, Teams notification, auto-trigger runbook

  Without DDoS Standard: Only Basic (free, always-on, no alerts, limited visibility)
  With DDoS Standard: Adaptive protection, custom alerts, detailed analytics, WAF integration
```

---

### 3.5 PRIVATE ENDPOINT / PRIVATE LINK — COMPLETE DEEP DIVE

> **Covered in Module 10 (VNet Deep Dive) at full L3 depth.** This section provides the Network Security perspective with emphasis on Private Endpoint as a security layer.

**Private Endpoint as Defense Layer 4 (No Public Access):**

```
Security Principle: PaaS services should NEVER have public access in production.

WITHOUT Private Endpoint:
  VM → Internet (public IP) → PaaS Service (public endpoint)
  → Traffic over internet (security risk)
  → Service publicly accessible (anyone can connect)
  → NSG/Firewall cannot inspect traffic (it's public)
  → Requires firewall rules on PaaS service (IP-based, fragile)

WITH Private Endpoint:
  VM → VNet (private route) → Private Endpoint (10.0.3.10)
  → Azure Private Link backbone → PaaS Service (internal)
  → NEVER touches internet
  → Service NOT publicly accessible (only via Private Endpoint)
  → NSG on Private Endpoint subnet provides access control
  → Firewall doesn't need to inspect (no public traffic)
  → DNS resolves to private IP (not public)

Security Benefits:
  1. Attack surface reduction (PaaS service invisible to internet)
  2. Data sovereignty (traffic stays on Azure backbone)
  3. Access control (who can connect = NSG + Private Endpoint approval)
  4. Compliance (no public endpoints = meets strict security policies)
  5. DNS poisoning prevention (Private DNS Zone maps to private IP only)
```

**Private Endpoint Security Checklist:**

| Check | Description | Verification |
|-------|-------------|-------------|
| Public access disabled | Storage/SQL/etc. account has "Deny public access" enabled | Portal: Storage Account → Configuration → Public access |
| Private DNS Zone linked | Private DNS Zone linked to correct VNet(s) | Portal: Private DNS Zone → VNet links |
| NSG on PE subnet | Private Endpoint subnet has NSG restricting access | Portal: Subnet → NSG association |
| Only needed VNs have access | Private Endpoint connected to necessary VNet(s) only | Portal: Private Endpoint → properties |
| Connection approved | Private Link connection status = Approved | Portal: Private Endpoint → Private Link connection |
| Private IP correct | Private IP within subnet range, not conflicting | Portal: Private Endpoint → Private IP |
| DNS resolution test | VM resolves service name to private IP | nslookup from VM |
| No public endpoint | PaaS service has no public endpoint | Portal: PaaS service → Networking → Public access |

---

### 3.6 VPN GATEWAY — COMPLETE DEEP DIVE

**What is Azure VPN Gateway?**
VPN Gateway is a virtual network gateway that sends encrypted traffic between Azure virtual networks and on-premises locations over the internet using a secure IPsec/IKE tunnel.

**VPN Gateway Types:**

| Type | Description |
|------|-------------|
| **Site-to-Site (S2S)** | VNet ↔ On-premises network (IPsec/IKE tunnel) |
| **Point-to-Site (P2S)** | Individual client (VM/PC) ↔ VNet (OpenVPN or IKEv2) |
| **VNet-to-VNet (V2V)** | VNet A ↔ VNet B (IPsec/IKE tunnel between gateways) |

**VPN Gateway SKUs:**

| SKU | Basis | Throughput | Use Case |
|-----|-------|-----------|---------|
| **Basic** | DTU | 100 Mbps | Dev/test, low-throughput |
| **VpnGw1** | vCPU | 650 Mbps | Production (single AZ) |
| **VpnGw2** | vCPU | 1 Gbps | Production HA (AZ-redundant) |
| **VpnGw3** | vCPU | 1.25 Gbps | High-throughput production |
| **VpnGw4** | vCPU | 2 Gbps | Enterprise |
| **VpnGw5** | vCPU | 2.5 Gbps | Enterprise (max) |
| **HighPerformanceVPNGW1-5** | vCPU | 10-50 Gbps | High-bandwidth scenarios |

> **L3 critical:** For production, use VpnGw2 or higher (zone-redundant). Basic SKU is NOT zone-redundant (single point of failure). For > 1.25 Gbps, consider ExpressRoute instead (VPN Gateway has throughput limits).

**Site-to-Site VPN Configuration:**

```
S2S VPN Architecture:

  VNet: vnet-prod-eastus-01 (10.0.0.0/16)
    ├── GatewaySubnet (10.0.4.0/27)
    │   └── VPN Gateway: vpn-gw-prod (VpnGw2, Zone-redundant)
    │       ├── Public IP: 20.3.1.1 (active)
    │       ├── Public IP: 20.3.2.1 (standby — auto for HA)
    │       ├── SKU: VpnGw2
    │       ├── BGP: Enabled (ASN 65515)
    │       │   → Local BGP IP: 10.0.4.1 (VPN GW)
    │       │   → Peer BGP IP: 10.10.1.1 (on-prem router)
    │       │   → Advertised prefixes: 10.0.0.0/16 (VNet)
    │       └── Active/Active mode
    │
    └── Connections:
        ├── Connection to On-Prem (IPsec/IKEv2):
        │   → Shared Key: (strong PSK, 30+ chars)
        │   → Peer IP: 13.100.1.1 (on-prem VPN device)
        │   → BGP: Enabled (learns on-prem routes)
        │   → IKEv2: Phase 1 (Encryption: AES256, Hash: SHA256, DH: 14)
        │   → IKEv2: Phase 2 (Encryption: AES256, Hash: SHA256, PFS: 14)
        │   → Dead Peer Detection: 30 sec (detect tunnel failures)
        │
        └── Connection to VNet (V2V):
            → Peer VNet: vnet-spoke-app1-eastus-01 (10.1.0.0/16)
            → Shared Key: (strong PSK)
            → V2V over internet (not over S2S)

On-Premises Configuration:
  → VPN Device: Must support IPsec/IKEv2 (most enterprise routers do)
  → BGP ASN: Configure on both VPN GW and on-prem router
  → Routing: On-prem router advertises on-prem prefixes to VPN GW
  → VNet routes: VPN GW advertises 10.0.0.0/16 to on-prem router

Traffic Flow:
  VM in VNet (10.0.1.4) → Traffic to on-prem (10.20.1.100)
  → VNet routes: 10.20.0.0/16 → Virtual Network Gateway (UDR)
  → VPN Gateway: Encrypts traffic (IPsec), sends through tunnel
  → Internet: Encrypted tunnel (UDP 500 ISAKMP, UDP 4500 NAT-T)
  → On-prem router: Decrypts, routes to 10.20.1.100
  → Return: Reverse path

Multiple Site Connections (Active/Active S2S):
  → VPN GW supports up to 30 site-to-site connections
  → Multiple connections = multiple tunnels (each to different peer)
  → For redundancy: Two VPN devices on-prem (active/active)
  → Both connect to same VPN GW (two connections)
  → BGP handles path selection (prefer lower ASN/metric)

Important Considerations:
  → GatewaySubnet is REQUIRED (minimum /27)
  → VPN GW takes 15-45 minutes to deploy
  → Public IP is REQUIRED (one per GW, two for HA)
  → Changing SKU requires restart (45+ minutes)
  → VM sizes on-prem: Check throughput compatibility with VPN GW SKU
  → VPN GW throughput: Shared across all connections (1 Gbps total for VpnGw2)
  → If using BGP: Advertise more-specific routes for better control
```

**Point-to-Site (P2S) VPN Configuration:**

```
P2S VPN Architecture:

  VNet: vnet-prod-eastus-01
    └── VPN Gateway: vpn-gw-prod (VpnGw2, with P2S enabled)
        ├── P2S Configuration:
        │   ├── Type: IKEv2 (recommended) or OpenVPN
        │   ├── Root Certificate: (public cert for client authentication)
        │   │   → Upload .cer file to VPN GW
        │   ├── Client Address Pool: 172.16.0.0/24 (IP pool for VPN clients)
        │   │   → Must not overlap with VNet or on-prem address spaces
        │   ├── RADIUS Server: (optional, for MFA integration)
        │   └── Authentication: EAP-MSCHAPv2 (username/password + MFA via RADIUS)
        │
        └── Client Connection:
            → User installs VPN client (Windows/macOS/iOS/Android)
            → Authenticates with username/password + MFA
            → Gets IP from 172.16.0.0/24 pool
            → Can access VNet resources (and on-prem if configured)

  IKEv2 vs OpenVPN:
    IKEv2:
      → Built-in OS support (Windows 10/11, macOS natively)
      → Faster connection (less overhead)
      → Better mobility (survives network changes — reconnects automatically)
      → Requires: EAP certificate or EAP-MSCHAPv2

    OpenVPN:
      → Cross-platform (requires OpenVPN client)
      → More flexible (TLS-based, easier to configure)
      → Supports TCP (port 443 — useful when UDP blocked)
      → Better for: Non-Windows platforms, restrictive networks

  P2S + MFA:
    → Integrate with Azure AD for conditional access
    → RADIUS server (NPS, FreeRADIUS) for MFA (MSCHAPv2 → Azure AD MFA)
    → Certificate-based auth: Strongest (cert + MFA via conditional access)
    → User-based: Username/password + MFA via RADIUS

  VNet-to-VNet (V2V) via VPN:
    → Both VNets have VPN Gateways
    → Each connects to the other (two connections, each direction)
    → NOT recommended: VNet Peering is preferred for VNet-to-VNet
    → Use V2V only when: Peering is not possible (different tenants, different subscriptions without peering)
```

---

### 3.7 EXPRESSROUTE — COMPLETE DEEP DIVE

**What is ExpressRoute?**
ExpressRoute is a dedicated private network connection between on-premises infrastructure and Azure. Unlike VPN, traffic does NOT go over the public internet — it goes through a dedicated circuit provided by a connectivity provider.

**ExpressRoute Circuit SKUs:**

| SKU | Bandwidth | Failover | Use Case |
|-----|----------|----------|---------|
| **Standard (QoS)** | 10 Gbps | Standard (static, primary/backup) | Enterprise, production |
| **Standard (Metered)** | 10 Gbps | Standard | Cost-sensitive, metered bandwidth |
| **ExpressRoute 10G** | 10 Gbps | Premium (no downtime during failover) | Mission-critical |
| **ExpressRoute 40G** | 40 Gbps | Premium | High-bandwidth workloads |
| **ExpressRoute 100G** | 100 Gbps | Premium | Largest workloads (databases, HPC) |

> **L3 critical:** For production, use Premium failover (no downtime). Standard failover can cause 2-3 seconds of downtime during failover. For financial/transactional systems, ALWAYS use Premium.

**ExpressRoute Architecture:**

```
ExpressRoute Circuit:

  Provider Network (Connectivity Provider):
    └── Circuit: 10 Gbps (SKU: ExpressRoute 10G Premium)
        ├── Primary Connection: Provider PoP in East US 2
        ├── Secondary Connection: Provider PoP in West US 2 (redundancy)
        └── Bandwidth: 10 Gbps (metered or unlimited depending on plan)

  Azure Side:
    ├── Region: East US 2 (primary)
    ├── ExpressRoute Gateway: er-gw-prod-eastus (Standard SKU)
    │   ├── Public IP: 20.4.1.1
    │   ├── SKU: Standard (can be upgraded to Premium for faster throughput)
    │   └── Associated Circuit: Circuit in East US 2
    ├── ExpressRoute Gateway: er-gw-prod-westus (Standard SKU) — DR
    │   └── Associated Circuit: Circuit in West US 2
    └── VNet Association:
        → VNet: vnet-prod-eastus-01 (10.0.0.0/16)
        → Gateway: er-gw-prod-eastus
        → Circuit: ExpressRoute Circuit (East US 2)

  Global Reach (Connect two ER circuits):
    → Circuit A (East US 2, 10G) ←→ Circuit B (West US 2, 10G)
    → Creates: Private connection between regions (not over internet)
    → Use for: Cross-region DR, provider redundancy, geographic diversity

Peering Types:
  ┌─ Microsoft Peering:
  │   → Routes to Azure services and Microsoft services
  │   → Services: Azure Virtual Machines, Azure SQL, Azure Storage, Azure Active Directory, Microsoft 365
  │   → Most common: Used for 90% of ExpressRoute traffic
  │   → Routing: BGP ( Advertise on-prem prefixes to Azure, advertise Azure prefixes to on-prem)
  │
  ├─ Public Peering:
  │   → Routes to Azure public services only (compute, storage, etc.)
  │   → Does NOT include Microsoft 365 or Azure AD
  │   → Less common, use Microsoft Peering instead
  │
  └─ Private Peering:
      → Routes to Azure private services (Azure SQL Private, Storage Private, etc.)
      → Only via private endpoints
      → Required for accessing PaaS services over ExpressRoute privately

Circuit Configuration:
  → Primary (East US 2):
    BGP Session:
      → Local BGP IP: 169.254.1.1 (ExpressRoute GW)
      → Peer BGP IP: 169.254.1.2 (on-prem router)
      → ASN: 65515 (Azure) / 65000 (on-prem)
      → Advertised prefixes: 10.0.0.0/16 (Azure VNet)
      → Received prefixes: 10.20.0.0/16 (on-prem), 10.30.0.0/16 (other on-prem)

  → Secondary (West US 2):
    Same configuration as Primary (for DR)
    Global Reach: Connects to Primary circuit

  → Maximum prefixes per circuit: 4000 (per IPv4 prefix)
  → Route filter: Use Route Filter to limit advertised/received prefixes

ExpressRoute vs VPN:
  ┌─ ExpressRoute ─────┐ vs ┌─ VPN Gateway ──────────┐
  │ Dedicated circuit    │    │ Shared internet        │
  │ Not over internet    │    │ Over internet (encrypted)│
  │ <1ms latency         │    │ 10-50ms latency         │
  │ 99.95% SLA           │    │ 99.9% SLA               │
  │ 10-100 Gbps          │    │ Up to 2.5 Gbps          │
  │ BGP required         │    │ BGP optional             │
  │ Expensive ($200+/mo) │    │ Cheap (~$5/mo)           │
  │ Primary connection   │    │ Backup connection        │
  │ No encryption (phys. │    │ IPsec encryption         │
  │   private link)      │    │ (not needed, private)    │
  │ Provider dependency  │    │ No provider dependency    │
  └──────────────────────┘    └─────────────────────────┘

Best Practice: ExpressRoute (primary) + VPN (backup)
  → ExpressRoute for normal operations (high performance, low latency)
  → VPN for DR (automatically activates if ExpressRoute fails)
  → Auto-failover: Configure VPN as backup, BGP prepends or longer routes for VPN
```

**ExpressRoute Redundancy Models:**

```
Redundancy Model 1: Single Circuit (NOT recommended for production)
  → One circuit, no redundancy
  → Single point of failure (provider circuit failure = no connectivity)
  → Use: Dev/test only

Redundancy Model 2: Dual Circuit — Same Provider, Different PoPs
  → Circuit A: Provider PoP East US
  → Circuit B: Provider PoP West US
  → BGP: Both advertise same prefixes (auto-failover via BGP path selection)
  → Use: Production with provider-level redundancy

Redundancy Model 3: Dual Circuit — Different Providers
  → Circuit A: Provider X (East US)
  → Circuit B: Provider Y (West US)
  → Maximum redundancy (survives provider failure + region failure)
  → Use: Mission-critical, financial, healthcare

Redundancy Model 4: ExpressRoute + VPN (Primary + Backup)
  → ExpressRoute: 10-100 Gbps (primary, high performance)
  → VPN: 1-2.5 Gbps (backup, automatic failover)
  → BGP: VPN routes have longer AS-path prepended (preferred = ExpressRoute)
  → If ExpressRoute fails: VPN routes become preferred (auto-failover in 2-3 seconds)
  → Use: Most enterprise production environments
```

---

### 3.8 VIRTUAL WAN — COMPLETE DEEP DIVE

**What is Virtual WAN?**
Virtual WAN is a Microsoft-managed wide area network service that provides hub-and-spoke connectivity across multiple regions. It replaces the need for manual VNet peering, VPN gateway configuration, and ExpressRoute gateway configuration by centralizing all connectivity in a Virtual WAN hub.

**Virtual WAN vs Traditional Architecture:**

```
Traditional (manual configuration):
  VNet-Hub: VPN Gateway + VNet Peerings (to each Spoke) + ER Gateway
  VNet-Spoke1: VNet Peer to Hub, UDR to Hub, NSGs, etc.
  VNet-Spoke2: VNet Peer to Hub, UDR to Hub, NSGs, etc.
  → Every peering needs configuration
  → UDRs need manual updates
  → Error-prone at scale

Virtual WAN (managed):
  Virtual WAN: Hub (auto-configured)
  VNet-Spoke1: Attach to Virtual WAN Hub (one click)
  VNet-Spoke2: Attach to Virtual WAN Hub (one click)
  → Hub automatically configures routing
  → Attachments auto-create UDRs, VNet peerings
  → Scales to 1000s of attachments
```

**Virtual WAN Components:**

```
Virtual WAN Hub (the central node):
  ├── VNet: vnet-hub-vwan-eastus (automatically created or BYO)
  ├── Address Space: 10.0.0.0/16 (default, customizable)
  ├── Subnets:
  │   ├── AzureFirewallSubnet (if Firewall attached)
  │   ├── GatewaySubnet (if VPN/ER attached)
  │   └── Default subnet (for Hub resources)
  ├── VPN Gateway (optional — attached to Hub):
  │   → Site-to-Site connections
  │   → Point-to-Site connections
  ├── ExpressRoute Gateway (optional — attached to Hub):
  │   → ExpressRoute circuit connections
  ├── Security Partner (optional — NVA attached to Hub):
  │   → Firewall, IDS/IPS, etc.
  │   → Traffic flows through security appliance automatically
  ├── VNet Attachments:
  │   → Spoke VNet 1 (10.1.0.0/16)
  │   → Spoke VNet 2 (10.2.0.0/16)
  │   → Spoke VNet 3 (10.3.0.0/16)
  │   → Each attachment = 1 VNet + 1 virtual Hub
  └── Hub-Sponsor VNet (if BYO Hub)

Virtual WAN Features:
  → Transitive routing: Spoke-to-Spoke via Hub (automatic, no peering needed)
  → BYO Hub: Use existing VNet as Virtual WAN Hub (bring your own)
  → QoS Policies: Prioritize different traffic types (VoIP, critical apps)
  → Security Partners: Attach NVAs to Hub for centralized inspection
  → Scale Units: Minimum 2 (HA), maximum 125 (large-scale)
  → Multi-region: Deploy Virtual WAN in multiple regions (global hub)
  → VNet route propagation: Spoke VNets learn routes from Hub and vice versa
```

**Virtual WAN Hub Modes:**

| Mode | Description | Use Case |
|------|-------------|---------|
| **Standard** | ARM-based hub (newer, recommended) | All new deployments |
| **Legacy** | Classic-based hub (deprecated) | Migrate to Standard |
| **BYO Hub** | Use existing VNet as Virtual WAN Hub | Already have existing VNet infrastructure |

**Virtual WAN Traffic Flow:**

```
Traffic Flow: Spoke VM → Virtual WAN Hub → Target

  VM in Spoke-1 (10.1.1.4) → UDR: 0.0.0.0/0 → Virtual WAN Hub (10.0.0.1)
  → Virtual WAN Hub evaluates:
    → Is traffic destined for another Spoke? → Route to Spoke-2 (10.2.1.4)
    → Is traffic destined for Internet? → Route to Firewall (if attached) or Gateway
    → Is traffic destined for on-prem? → Route to VPN/ER Gateway (if attached)
    → Is traffic destined for another Region? → Route to regional Virtual WAN Hub

  Multiple regions:
    Virtual WAN East US (Hub-1): Spoke-1, Spoke-2 (East US VMs)
    Virtual WAN West US (Hub-2): Spoke-3, Spoke-4 (West US VMs)
    → Inter-region traffic: Hub-1 → Hub-2 (via Microsoft backbone)
    → Route propagation: Each Hub knows routes from all attached Spoke VNets

Security Partner Flow:
  VM → Spoke-1 → Virtual WAN Hub → Security Partner (Firewall/IDS) → Target
  → All traffic through Security Partner automatically
  → No manual UDR configuration needed for NVA

BYO Hub Flow:
  Existing VNet: vnet-hub-existing (10.0.0.0/16)
  → Enable Virtual WAN on existing VNet
  → Attach: VPN Gateway, ExpressRoute, Security Partner
  → Spoke VNets attach to existing VNet (BYO Hub)
  → UDRs automatically configured (0.0.0.0/0 → Virtual WAN Hub)
```

**Virtual WAN Configuration Steps:**

```
Step 1: Create Virtual WAN
  → Virtual WAN name: vwan-prod-eastus
  → Region: East US 2
  → Hub type: Standard
  → Address space: 10.0.0.0/16 (or BYO existing VNet)

Step 2: Create Virtual Hub
  → Virtual Hub: vhub-hub-prod-eastus
  → Virtual WAN: vwan-prod-eastus
  → Address space: 10.0.0.0/16
  → Scale units: 2 (HA)
  → Subnet: HubSubnet (10.0.0.0/24)

Step 3: Attach VNet Spokes
  → Spoke-1: vnet-spoke-app1-eastus (10.1.0.0/16)
    → Virtual Hub: vhub-hub-prod-eastus
    → Allow VNet to VNet traffic: Enabled (transitive)
    → Allow Gateway Transit: Enabled (if VPN/ER on Hub)

  → Spoke-2: vnet-spoke-app2-eastus (10.2.0.0/16)
    → Same Virtual Hub
    → Transitive routing: Spoke-1 ↔ Spoke-2 via Hub (automatic)

Step 4: Attach VPN Gateway (optional)
  → VPN Gateway: vpn-gw-vwan-eastus (VpnGw2)
  → Virtual Hub: vhub-hub-prod-eastus
  → S2S connections: On-prem (multiple sites)
  → P2S connections: Client access

Step 5: Attach ExpressRoute Gateway (optional)
  → ER Gateway: er-gw-vwan-eastus
  → Virtual Hub: vhub-hub-prod-eastus
  → Circuit: ExpressRoute Circuit (East US 2)

Step 6: Attach Security Partner (optional)
  → Firewall: fw-vwan-eastus
  → Virtual Hub: vhub-hub-prod-eastus
  → Subnet: AzureFirewallSubnet
  → All Spoke traffic routes through Firewall (security stacking)

Step 7: Configure QoS (optional)
  → Policy: VoIP traffic — Priority 1 (highest)
  → Policy: Database traffic — Priority 2
  → Policy: Web traffic — Priority 3
  → Policy: Bulk data — Priority 4 (lowest)

Step 8: Validate
  → Connection Monitor: Test from Spoke VM to:
    - Other Spoke VMs
    - On-prem (via VPN/ER)
    - Internet (via Gateway)
    - Other region VMs (via Global Virtual WAN)
```

**Virtual WAN — What Works / What Doesn't:**

```
✅ Supported:
  → VNet-to-VNet transit via Hub (transitive)
  → VNet attachment (any VNet in supported region)
  → VPN Gateway attachment (S2S and P2S)
  → ExpressRoute Gateway attachment
  → Security Partner (NVA) attachment
  → BYO Hub (existing VNet as Hub)
  → Global Virtual WAN (multi-region hubs)
  → QoS policies
  → User-Defined Routes (with propagation)
  → Private Endpoints on Hub VNet

❌ NOT Supported (limitations):
  → BGP route propagation in BYO Hub mode (static routes only)
  → VNet peering with Virtual WAN (direct peering and Virtual WAN coexist but cause conflicts)
  → Gateway Transit via peering AND Virtual WAN simultaneously (pick one)
  → VNet Gateway (VPN/ER) + Virtual WAN VPN/ER on same VNet (pick one — Virtual WAN replaces)
  → Some legacy features (VNet Gateway Manager, classic gateways)
  → Multiple address spaces per VNet (use primary address space)
  → VMs in Hub VNet with multiple interfaces (limited support)
  → AS Path Prepend for controlling VPN/ER path selection (use different methods)

Best Practice:
  → New deployments: Use Virtual WAN from the start
  → Existing deployments: Migrate gradually (BYO Hub mode)
  → Do NOT mix Virtual WAN and direct peering (use one or the other)
  → Keep Hub VNet minimal (only essential Hub resources)
```

---

## 4. NETWORK SECURITY DESIGN PATTERNS

### 4.1 Hub-and-Spoke with Firewall (Standard)

```
Virtual WAN Hub: vhub-prod-eastus (10.0.0.0/16)
  ├── Firewall: fw-prod (Standard SKU, AzureFirewallSubnet)
  ├── DNS: Azure DNS Private Resolver (Hub VNet link)
  ├── Bastion: bastion-prod (AzureBastionSubnet)
  └── Attached VNets (Spokes):
      ├── Spoke-Web: vnet-spoke-web-eastus (10.1.0.0/16)
      ├── Spoke-App: vnet-spoke-app-eastus (10.2.0.0/16)
      ├── Spoke-Data: vnet-spoke-data-eastus (10.3.0.0/16)
      └── Spoke-Mgmt: vnet-spoke-mgmt-eastus (10.4.0.0/16)

  UDR on all Spokes:
    → 0.0.0.0/0 → Virtual WAN Hub (propagated automatically)
    → All internet-bound traffic → Firewall
    → All Spoke-to-Spoke traffic → Hub (transitive, via Firewall)

  NSG on Hub Firewall subnet:
    → Allow HubSubnet → VirtualNetwork (outbound for firewall traffic)
    → Allow HubSubnet → Internet (outbound for SNAT, DNS)

  Result: ALL inter-Spoke and ALL internet traffic flows through Firewall
```

### 4.2 Distributed Security with Security Stacking

```
Virtual WAN Hub: vhub-prod-eastus
  ├── Central Firewall: fw-central (gateway-level inspection)
  │   → All Spoke-to-Internet traffic (default)
  ├── Spoke Firewalls (Security Stacking):
  │   ├── fw-spoke-app1 (in Spoke-1 subnet)
  │   ├── fw-spoke-app2 (in Spoke-2 subnet)
  │   └── Each inspected by central policy (from Firewall Manager)
  │
  → Security Stacking: Each Spoke gets its own Firewall
  → Central policy: Distributed to all Spoke Firewalls
  → Local rules: Each Spoke Firewall can have additional rules
  → Traffic inspection: Distributed (at Spoke level, not Hub)
  → Benefits: Lower latency (less hairpinning through Hub), scalable, per-tenant isolation
```

### 4.3 Hub-and-Spoke with ExpressRoute + VPN

```
Virtual WAN Hub: vhub-prod-eastus
  ├── ExpressRoute Gateway: er-gw-prod (ExpressRoute 10G, Premium)
  │   → Circuit A: Provider X (East US 2) — Primary
  │   → Global Reach: Circuit B (West US 2) — DR
  ├── VPN Gateway: vpn-gw-prod (VpnGw2, Zone-redundant)
  │   → S2S: On-prem (Primary Backup — auto-failover)
  │   → P2S: Client access
  ├── Spoke-1: vnet-spoke-app1 (10.1.0.0/16)
  │   → UDR: 0.0.0.0/0 → Virtual WAN Hub
  │   → On-prem routes: propagated via ER/VPN
  ├── Spoke-2: vnet-spoke-app2 (10.2.0.0/16)
  │   → Same configuration
  └── ...

  Redundancy:
    Primary: ExpressRoute (10G, Premium failover)
    Backup: VPN Gateway (auto-failover via BGP route preference)
    On-prem: Two routers (active/active) connecting to ER + VPN
```

---

## 5. PRODUCTION EXAMPLE

**Scenario: Global enterprise with 5 regions, 20 applications, 1000+ VMs, ExpressRoute + VPN, Virtual WAN global hub.**

```
TENANT: globalcontoso.onmicrosoft.com

VIRTUAL WAN: vwan-global-contoso (Global)
  Hub-EastUS2: vhub-hub-prod-eastus2 (10.0.0.0/16, Scale Units: 4)
    ├── Firewall: fw-hub-prod-eastus2 (Standard, HA)
    ├── DNS: Private DNS Resolver (Hub VNet link)
    ├── Bastion: bastion-hub-prod-eastus2 (AzureBastionSubnet /27)
    ├── ExpressRoute Gateway: er-hub-prod-eastus2
    │   ├── Circuit A: ER-Circ-Eastus2 (Provider X, 10G, Premium) — Primary
    │   ├── Circuit B: ER-Circ-Westus2 (Provider Y, 10G, Premium) — DR
    │   └── Global Reach: Circuits A ↔ B (cross-region private link)
    ├── VPN Gateway: vpn-hub-prod-eastus2 (VpnGw3, Zone-redundant)
    │   ├── S2S-Primary: On-prem East US (2 tunnels, BGP)
    │   ├── S2S-Backup: On-prem West US (2 tunnels, BGP)
    │   └── P2S: Client VPN (IKEv2, 500 concurrent)
    └── VNet Attachments (Spokes):
        ├── Spoke-1: vnet-spoke-app1-eus (10.1.0.0/16)
        ├── Spoke-2: vnet-spoke-app2-eus (10.2.0.0/16)
        ├── Spoke-3: vnet-spoke-data-eus (10.3.0.0/16)
        └── Spoke-Mon: vnet-spoke-mon-eus (10.4.0.0/16)

  Hub-WestUS2: vhub-hub-prod-westus2 (10.10.0.0/16, Scale Units: 4)
    ├── Firewall: fw-hub-prod-westus2 (Standard, HA — independent policy)
    ├── ExpressRoute Gateway: er-hub-prod-westus2
    │   └── Circuit C: ER-Circ-Westus2 (Provider Y, 10G, Premium) — Primary (West US)
    ├── VPN Gateway: vpn-hub-prod-westus2 (VpnGw3) — DR
    └── VNet Attachments:
        ├── Spoke-1: vnet-spoke-app1-wus (10.11.0.0/16)
        ├── Spoke-2: vnet-spoke-app2-wus (10.12.0.0/16)
        └── Spoke-Data: vnet-spoke-data-wus (10.13.0.0/16)

  → Global WAN: Hub-EastUS2 ↔ Hub-WestUS2 (Microsoft backbone, transitive)
  → Spoke East US ↔ Spoke West US: Via Global WAN (automatic, no peerings)

GLOBAL SECURITY ARCHITECTURE:
  All Spokes: UDR → Virtual WAN Hub → Firewall (per region)
  All Spokes: Private Endpoints (no public access for any PaaS)
  All Spokes: NSGs (tier-based, deny-internet-by-default)
  All PaaS: Private Endpoint + Private DNS Zone linked to Spoke VNet
  All VMs: No public IPs (Bastion for access)
  All VNet traffic: Audited via NSG Flow Logs + Firewall logs

VPN + EXPRESSROUTE REDUNDANCY (per region):
  Primary: ExpressRoute 10G Premium (provider-redundant, dual circuits)
  Backup: VPN Gateway VpnGw3 (auto-failover via BGP route prepend)
  On-prem: Two routers (active/active), each connecting to both ER circuits and VPN
  Failover: ER primary fails → BGP converges to VPN backup in <3 seconds

WAF + DDoS (per application):
  All public-facing apps: Application Gateway v2 (WAF) + DDoS Protection Standard
  → WAF: Prevention mode (after tuning phase)
  → WAF Rules: OWASP 3.2 Managed + Custom Rules (allowlist, rate limit)
  → DDoS: Standard with custom alert rules (>100 Mbps mitigated)

DNS:
  Azure Public DNS: Default (Azure services resolution)
  Private DNS Zones:
    → corp.contoso.com → Linked to ALL Spoke VNets
    → privatelink.* (all service-specific) → Linked to ALL Spoke VNets
  Private DNS Resolver (Hub):
    → Inbound Endpoint: VNet Hub (10.0.0.4)
    → Forwarding Rules: corp.contoso.com → on-prem DNS (10.20.1.1)
    → Forwarding Rules: azurefdns.azure.com → Azure DNS (168.63.129.16)

MONITORING:
  NSG Flow Logs: ALL subnets in ALL Spokes → Log Analytics per region
  Firewall Logs: ALL Firewalls (Hub) → Log Analytics per region
  Application Gateway Logs: All WAF instances → Log Analytics
  VPN Gateway Logs: All VPN Gateways → Log Analytics
  ER Logs: All ER Gateways → Log Analytics
  Connection Monitor: Per-application, cross-region tests
  Virtual WAN metrics: Hub utilization, attachment status, QoS policy compliance

BACKUP:
  VMs: Azure Backup (daily, 30-day retention, 7-year LTR)
  SQL DBs: PITR (35 days), LTR (10 years), Geo-restore
  Cosmos DB: Continuous backup + scheduled export to Blob Storage (LTR)
  Storage: Soft Delete (90 days), Immutable (legal hold for compliance data)
  ADLS: Backup to separate backup storage account (cool tier)

ONBOARDING:
  New app team:
    1. Request VNet Spoke through IT catalog
    2. Virtual WAN attachment created (2-5 minutes)
    3. NSGs, UDRs auto-provisioned (via blueprint)
    4. Private DNS linked automatically
    5. Firewall policy applied (via Firewall Manager)
    6. Bastion access available immediately
    Time to operational: < 2 hours (vs. 2-4 days manual)

COST MANAGEMENT:
  ExpressRoute: ~$2000-$5000/month per 10G circuit
  VPN Gateway: ~$140/month per VpnGw2
  Firewall: ~$2600/month per Standard HA pair
  Application Gateway v2: ~$200/month + per-hour compute
  Bastion: ~$0.30/VM-hour + SKU cost
  DDoS Standard: ~$29/day per region
  Total network security: ~$15,000-$25,000/month for 5 regions, 20 apps

DRILL ANNUALLY:
  → ER circuit failure: Verify VPN auto-failover
  → Firewall failure: Verify HA pair takes over
  → Bastion access: Verify all VMs accessible via Bastion
  → Private Endpoint: Verify all PaaS accessible privately
  → WAF: Verify OWASP rules block test attack
  → VPN: Verify P2S connectivity from test client
```

---

## 6. FAILURE SCENARIOS

| Scenario | Root Cause | Resolution | Evidence |
|-----------|-----------|------------|----------|
| **NSG blocks production traffic after deployment** | Someone added deny rule at high priority (100-200); misconfigured service tag | Check effective NSG rules for NIC; NSG Flow Logs show deny by which rule; Roll back/adjust rule priority. Most common: auto-generated NSG during deployment has deny-all-inbound at priority 100. | Portal: NIC → Effective NSG Rules; Activity Log: NSG changes; NSG Flow Logs: deny details |
| **Firewall not seeing traffic (traffic bypass)** | UDR on subnet points to Internet instead of Firewall; or GatewaySubnet has UDR; or same-VNet traffic doesn't go through Firewall by design | Check UDR on ALL subnets (0.0.0.0/0 → Firewall); verify no subnet has direct internet UDR; check GatewaySubnet (should have NO UDR). Most common: someone added UDR 0.0.0.0/0 → Internet for convenience. | Portal: Subnet → Effective Routes; UDR: 0.0.0.0/0 next hop; Firewall logs: traffic source |
| **WAF blocking legitimate traffic (false positive)** | OWASP managed rule blocks valid request (e.g., SQL-like text in URL parameter); rate limit exceeded | WAF in Detection mode: identify what's being blocked; Add exclusion rule (skip specific path/header/parameter); Whitelist specific IP; Disable specific rule for specific path. Most common: OWASP CRS rule blocking URL-encoded characters in form data. | WAF Logs: matched rule ID, rule name; Application Gateway: access logs showing 403/502; Browser: blocked by WAF |
| **Private Endpoint for SQL DB not resolving** | Private DNS Zone "privatelink.database.windows.net" not linked to VNet; VM DNS set to Azure public DNS (168.63.129.16) which doesn't resolve private zones not linked; VM in different VNet from PE. Most common: Private DNS Zone not linked to the VNet where VM resides. | Check: Private DNS Zone → VNet links (is VM's VNet linked?); nslookup from VM: privatelink.database.windows.net → should return private IP; VM DNS settings; Link Private DNS Zone to VM's VNet; Reconfigure VM DNS to use linked resolver. | nslookup: privatelink.database.windows.net from VM; Private DNS Zone: linked VNets; Portal: Private Endpoint status |
| **VPN Gateway not connecting (S2S tunnel down)** | PSK mismatch; on-prem router misconfigured; BGP not learning routes; Gateway SKU too small for throughput; Dead Peer Detection timeout; Certificate expired (if cert-based). Most common: PSK mismatch (someone changed on-prem side but not Azure side, or vice versa). | Check: PSK on both sides (must match); BGP status: both sides learning routes; Gateway health: Running; Tunnel status: Connected; Activity Log: configuration changes; On-prem router logs: tunnel establishment. Use: Connection Monitor for diagnostics. | Portal: VPN Gateway → Connections → Status; BGP: peer status; Activity Log: recent config; Tunnel logs |
| **ExpressRoute circuit down (primary)** | Provider circuit failure (fiber cut, PoP down); BGP session dropped; router misconfiguration; circuit de-provisioned. Most common: provider-side failure (not Azure-side — Azure marks circuit as "Succeeded" but provider-side is down). | Verify: ExpressRoute circuit status in Portal (Succeeded/Failed); Provider portal: circuit status; BGP session: established; ER Gateway health; Global Reach: secondary circuit active; If primary truly down: BGP auto-fails to VPN backup (3 seconds). If Global Reach: secondary takes over. | Portal: ER Circuit status; ER Gateway: connections; BGP: BGP peer status; Provider: circuit status page |
| **Virtual WAN Spoke-to-Spoke traffic not working** | Spoke VNet not attached to Virtual WAN Hub; "Allow VNet to VNet traffic" disabled on attachment; UDR not propagated; address space overlap; "Do not use VNet peering with Virtual WAN" conflict. Most common: Spoke VNet attached but VNet-to-VNet traffic checkbox not enabled (default = disabled). | Check: Spoke attachment status; VNet-to-VNet traffic checkbox: enabled; UDR: 0.0.0.0/0 → Virtual WAN Hub; Address spaces non-overlapping; Check for conflicting VNet peerings. Most common: VNet-to-VNet traffic disabled on attachment (must be enabled for transit). | Portal: Virtual WAN → Attachments; Spoke VNet: UDR effective routes; Test: ping from Spoke to Spoke; Virtual WAN: connection monitor |
| **Firewall HA pair failing (one node down)** | Firewall instance down; managed identity issue; subnet IP conflict; firewall policy not propagated; Premium features not supported on Basic SKU. Most common: one firewall in HA pair stops (VM size exhausted, disk full, managed identity expired). | Check: Firewall health in Portal; Both instances should be running; Check: Firewall logs for errors; If one down: other takes over (active/standby); Verify: SNAT pool exhausted; Check: Threat Intel feed updates; Check: Firewall policy deployment status. | Portal: Firewall → Health; Log Analytics: firewall errors; Activity Log: firewall events |
| **WAF autoscale not scaling up** | Scale threshold not configured; Metric threshold not reached (low traffic); SKU limitations (V1 doesn't autoscale); WAF policy not attached; Capacity units maxed. Most common: Autoscale not enabled on Application Gateway v2 (must be explicitly configured with min/max capacity). | Check: Application Gateway → Autoscale settings; Min: 2, Max: 10+; Scale metric: Capacity units; Current CU usage; Gateway status: Running; Verify: WAF policy attached; Check: Gateway SKU (must be v2 for autoscale). | Portal: App Gateway → Metrics (CU usage); Autoscale settings; Activity Log: scale operations |
| **DDoS attack causing application slowdown** | Volumetric attack exceeding Basic protection limits; L7 attack (many requests from botnets) not mitigated by Basic; WAF not configured; App Gateway capacity exhausted. Most common: L7 attack that Basic DDoS handles but Application Gateway can't keep up (insufficient capacity or WAF not enabled). | Enable: DDoS Protection Standard (adaptive, mitigates volumetric); Enable: WAF on Application Gateway; Enable: Rate limiting on WAF; Scale up: Application Gateway instances; Monitor: DDoS metrics; Enable: alerts. Most common: DDoS Basic protection works but App Gateway WAF capacity insufficient for volumetric L7. | DDoS: Mitigation metrics; App Gateway: request rate, CU usage; WAF: blocked requests; Alert rules: triggered alerts |
| **NSG Flow Logs not capturing traffic** | Flow Logs not enabled on NSG; Storage Account not configured (or deleted); Log Analytics workspace not linked; Flow Log status: Disabled; VM traffic same-VNet (not captured if VNet-to-VNet is default allow). Most common: Flow Logs were never enabled (must be explicitly enabled per NSG). | Check: NSG → Flow Logs → Status; Storage Account: exists and accessible; Log Analytics: linked and receiving data; Check: Flow log version (version 2 recommended); Enable on all production NSGs. | Portal: NSG → Flow Logs → Status; Log Analytics: flow log entries; Storage: flow log files |
| **Bastion cannot connect to VM (session fails)** | Bastion in wrong subnet; VNet peering not configured (Bastion in Hub, VM in Spoke without transit); NSG on Bastion subnet blocking traffic; VM has no network connectivity; Browser cache; Bastion SKU limit reached. Most common: NSG on Bastion subnet missing rule "Allow AzureBastionSubnet → VirtualNetwork" or "Allow AzureBastionSubnet → Internet (443)". | Check: Bastion: status (Running); Bastion subnet NSG: rules (Allow VirtualNetwork + Allow Internet 443); VM: running, NSG allows Bastion subnet (RDP/SSH); Check: Bastion in correct subnet (AzureBastionSubnet); VM DNS: resolves Bastion service; Check: VM has NSG allow from Bastion subnet. | Bastion: connection error; NSG Bastion subnet: rules; VM: NSG effective rules; Activity Log: Bastion session logs |
| **ExpressRoute circuit shows "Succeeded" but traffic not flowing** | BGP session not established on one side; Route filter not configured; Private peering not configured for service; Circuit provisioning still in progress (under hood); MAC/IP mismatch on provider side. Most common: BGP sessions not yet up (circuit provisioned on Azure side, but provider-side BGP not configured yet — takes hours after approval). | Check: ER Circuit status: Succeeded AND BGP Sessions: Established; Provider portal: circuit active; Check: BGP status in Portal (Connections → BGP peers); Routes learned? If no BGP: configure on both sides; Check: Route filters; Check: Provider peering VLAN/MAC. | Portal: ER Circuit → BGP peers status; Activity Log: provisioning events; Provider portal: circuit status; BGP: routes received |
| **VPN Gateway throughput saturated (slow connections)** | Gateway SKU too small for traffic; Too many simultaneous connections; All connections through single gateway (no load distribution); BGP not aggregating routes (too many /32 routes); Encryption overhead high. Most common: VPN Gateway VpnGw1 (650 Mbps) serving too many connections or high-throughput apps. | Upgrade: VPN Gateway SKU (VpnGw2, VpnGw3); Add: Second VPN Gateway (active/active); Distribute: connections across multiple gateways; BGP aggregation: summarize routes; Check: Gateway throughput metrics; Monitor: VPN Gateway bandwidth utilization. | Portal: VPN Gateway → Metrics (bandwidth); Connections: per-tunnel metrics; VPN Gateway: connection count |
| **ExpressRoute + Virtual WAN — routes not propagating** | BYO Hub mode without BGP; Route filter not associated with ER Gateway; BGP session not established; Route Propagation not enabled on Spoke VNet; VNet address space not advertised. Most common: Route Filter not associated with ER Gateway (must link route filter to gateway for route advertisement). | Check: ER Gateway → Route Filter: associated?; Route Filter: rules (which prefixes to advertise); BGP: routes received and advertised; VNet attachment: Route propagation enabled; Check: Provider side: BGP advertisements; Check: Virtual WAN: attachments and route propagation settings. | Portal: ER Gateway → Routes received; Virtual WAN: VNet attachments; VNet: UDR effective routes; BGP: advertised prefixes |
| **Application Gateway returning 502 Backend Healthy but no traffic** | Backend VMs NSG blocking App Gateway subnet traffic; Backend health "Healthy" but VMs not listening on expected port; Backend pool missing VMs; App Gateway in different subnet/VNet than VMs; App Gateway private endpoint not accessible. Most common: Backend VM NSG doesn't allow traffic from App Gateway subnet (must allow from App GW subnet, not just "Internet"). | Check: VM NSG: Allow from App Gateway subnet on VM port; App Gateway: Backend health = Healthy; Backend pool: VMs listed; Verify: VMs listening on expected port (netstat); Verify: App Gateway can reach VMs (test from App GW subnet); Check: UDR/NSG between App GW and VMs. | VM NSG: Effective NSG rules; Backend health: App Gateway; Test: curl from App GW subnet VM; Backend pool: members |
| **WAF blocking all traffic (everything denied)** | WAF policy in Prevention mode with deny-all default; OWASP managed rules too aggressive; Custom deny-all rule at low priority (e.g., priority 1000 → deny); WAF enabled on Gateway but not needed for backend-only traffic. Most common: Someone accidentally set default action to Deny instead of Allow, or added overly aggressive custom rule. | Check: WAF policy: default action (should be Allow); WAF rules: review priorities (deny rules should be after allow rules); Switch to Detection mode temporarily; Review logs: which rule is denying everything; Fix: Adjust default action to Allow; add specific Deny rules after Allow rules. | WAF: policy rules and default action; WAF logs: denial counts by rule; App Gateway: HTTP 403/502 responses; WAF policy: audit mode test |

---

## 7. MONITORING — NETWORK SECURITY

| Metric/Log | What It Shows | Alert Trigger |
|------------|---------------|---------------|
| **NSG Flow Logs** | Allowed/denied traffic per NSG rule | Spike in denies, unusual patterns |
| **Firewall Logs (AzureFirewall)** | All traffic through Firewall (allowed/denied by rule) | Threat intel matches, unusual destinations |
| **Firewall Logs (AzureFirewallApplicationRule)** | HTTP/HTTPS traffic by FQDN | Banned domains accessed, unusual FQDNs |
| **Firewall Logs (AzureFirewallNetworkRule)** | IP/CIDR-based traffic | Unusual IP destinations |
| **Application Gateway Access Logs** | All HTTP/HTTPS requests, WAF blocks, backend responses | 4xx/5xx spikes, WAF blocks, slow responses |
| **Application Gateway WAF Logs** | WAF rule matches, blocked requests | SQL injection attempts, XSS attempts, OWASP blocks |
| **DDoS Metrics** | Mitigated traffic, attack type, source IPs | High mitigation volumes, sustained attacks |
| **VPN Gateway Metrics** | Bandwidth, tunnel status, BGP routes | Tunnel down, bandwidth spikes, BGP flap |
| **ExpressRoute Metrics** | BGP sessions, bits transferred, route changes | BGP down, circuit failover, route churn |
| **ExpressRoute Circuit Status** | Succeeded/Failed, BGP session state | Circuit provisioning failure, BGP session down |
| **Virtual WAN Metrics** | Hub capacity, attachment count, route count | Hub near capacity, attachment failures |
| **Bastion Session Metrics** | Active sessions, connection failures, authentication | Unusual session volumes, auth failures |
| **Effective NSG Rules** | Computed rules for NIC/subnet (read-only, real-time) | Unexpected denies |
| **Effective Routes** | Computed routes for subnet/NIC | Route changes, unexpected paths |
| **Connection Monitor** | End-to-end connectivity, latency, packet loss | Connectivity failures, latency spikes |
| **IP Flow Verify** | Whether specific IP flow is allowed/denied by NSG | Quick troubleshooting |
| **Network Watcher Next Hop** | Where traffic will go for specific source/dest | Unexpected next hops (traffic bypass) |
| **Private Endpoint Status** | Provisioning, connection status, private IP | Endpoints failing, connections rejected |
| **Private DNS Resolution** | DNS resolution success/failure, resolution time | DNS failures, wrong IP resolution |
| **Application Gateway Backend Health** | Backend VM health (Healthy/Unhealthy/Draining) | Backend unhealthy, health probe failures |
| **Gateway Load Balancer Metrics** | NVA health, traffic distribution | NVA unhealthy, traffic imbalance |
| **Alert: Route Change** | UDR or system route changes | Unauthorized route changes |
| **Alert: NSG Rule Change** | NSG rule added/modified/deleted | Unauthorized changes |
| **Alert: Gateway Down** | VPN/ExpressRoute Gateway stopped | Gateway unavailable |
| **Alert: Firewall Threat Detected** | Threat Intel alerts, malware C2 traffic | Threat intelligence match |

---

## 8. SECURITY CONSIDERATIONS

| Concern | Risk | Mitigation |
|---------|------|------------|
| **VMs with public IPs** | Directly accessible from internet; RDP/SSH exposed | No public IPs; Bastion for access; NSG deny-inbound-from-internet |
| **Flat NSG rules (Allow * *)** | All traffic allowed, NSG is useless | Least privilege; specific source/dest/port/protocol; regular audit |
| **No Firewall (no centralized NVA)** | No inspection of east-west or north-south traffic | Deploy Azure Firewall in Hub; all traffic via UDR to Firewall |
| **WAF in Detection mode indefinitely** | Threats not blocked, only logged | Transition to Prevention after tuning (1-2 weeks max) |
| **PaaS public access** | Storage/SQL accessible from internet; data exposure | Disable public access; Private Endpoints; firewall rules |
| **ExpressRoute single circuit** | Single point of failure (provider failure = no on-prem) | Dual circuits (same or different providers); VPN backup |
| **VPN Gateway Basic SKU** | Single point of failure (no AZ); limited throughput | VpnGw2+ (zone-redundant); ExpressRoute for production |
| **Missing NSG Flow Logs** | No visibility into what traffic is blocked | Enable Flow Logs on ALL NSGs → Storage + Log Analytics |
| **Firewall SKU mismatch** | Basic SKU in Standard VNet; or Basic features expected | Standard/Premium SKU for production; verify VNet SKU |
| **Default NSG allows everything** | Default rules "Allow VNet Inbound/Outbound" override user rules | Explicit deny rules at higher priority; deny-by-default approach |
| **Spoke-to-Spoke via internet** | VPN tunnel or peering break → traffic routes via internet | Virtual WAN Hub (transitive, no internet); no direct peering without Hub |
| **Shared NSG across many VNets** | One rule change affects all VNets (accidental disruption) | Per-subnet NSGs; NSG standardization via Firewall policy |
| **Private DNS Zone in wrong VNet** | VM can't resolve service name → connection fails | Verify Private DNS Zone links to correct VNet(s); test from VM |
| **UDR pointing to non-existent Firewall** | Traffic black-holed; all internet access lost | UDR 0.0.0.0/0 → Firewall IP must be valid; Firewall HA required; monitor Firewall health |
| **BGP flap (VPN/ER unstable)** | Routes constantly changing; traffic interrupted | BGP multi-hop TTL; stable on-prem routers; BFD (Bidirectional Forwarding Detection) |
| **DDoS protection not enabled** | Application can be overwhelmed by volumetric attacks | DDoS Standard (production); alerts for mitigation events |
| **Application Gateway without WAF** | Web apps vulnerable to OWASP Top 10 | Always use WAF_v2 SKU with OWASP managed rules |
| **VPN Gateway no Dead Peer Detection** | Dead connections hold resources; tunnel stuck | Enable DPD (30 sec interval on both sides) |
| **Firewall Management not enabled** | Inconsistent rules across multiple firewalls | Enable Firewall Manager; deploy security policy centrally |
| **No DNS logging** | Malicious DNS queries go undetected | Enable DNS query logs; Private DNS Resolver; Defender for DNS |
| **TLS 1.0/1.1 still enabled** | Weak encryption; compliance failures | Enforce TLS 1.2 minimum; Application Gateway: TLS 1.2 only; SQL: TLS 1.2 enforced |
| **WAF custom rules misconfigured** | Blocking legitimate traffic or allowing malicious traffic | Test in Detection mode; review regularly; use specific paths and conditions |
| **ExpressRoute + no backup (VPN)** | ExpressRoute failure = no on-prem connectivity | VPN Gateway backup (auto-failover via BGP); Global Reach for circuit redundancy |
| **Virtual WAN route leaks** | Spoke-to-Spoke traffic that shouldn't transit via Hub | Security group tags; enforce routing policies; use Virtual WAN routing policies |
| **No network segmentation** | Compromised VM can reach all VMs and services | Tier-based subnets; NSGs between tiers; Firewall between zones |
| **Firewall SNAT exhaustion** | Outbound traffic fails (no available SNAT IPs) | Multiple public IPs; Premium SKU (more SNAT IPs); reduce connections; scale up |

---

## 9. DIAGNOSTIC SETTINGS — NETWORK SECURITY

| Resource | Log Category | Destination |
|----------|-------------|------------|
| **NSG** | All (Flow Logs v1/v2) | Storage Account + Log Analytics |
| **Azure Firewall** | All (AzureFirewall, AppRule, NetRule, TLS, IDPS) | Log Analytics |
| **Firewall Manager** | Policy change logs, deployment logs | Log Analytics |
| **Application Gateway** | Access, WAF, Performance, SSL, Health Probe | Log Analytics + Storage |
| **DDoS Protection** | Mitigation, alerts, attack metrics | Log Analytics + Azure Monitor Alerts |
| **VPN Gateway** | Connect, IKE, P2S, BGP | Log Analytics + Storage |
| **ExpressRoute Gateway** | Connect, BGP, ER Health | Log Analytics + Storage |
| **ExpressRoute Circuit** | Provider status, BGP events | Log Analytics + Service Health |
| **Virtual WAN Hub** | Connection metrics, route changes, attachment status | Log Analytics + Azure Monitor |
| **Bastion** | Session logs, connection logs, error logs | Log Analytics + Storage |
| **Private Endpoint** | Provisioning, connection, DNS logs | Log Analytics |
| **Private DNS Resolver** | Query logs, resolution success/failure | Log Analytics |
| **Connection Monitor** | Test results, latency, packet loss | Log Analytics |
| **Network Watcher** | All diagnostics (IP Flow Verify, Next Hop, etc.) | Log Analytics |
| **Activity Log** | All network resource configuration changes | Log Analytics + Storage |
| **Alert (NSG)** | Rule changes, deny spikes | Azure Monitor → Action Groups |
| **Alert (Firewall)** | Threat detected, throughput, health | Azure Monitor → Action Groups |
| **Alert (VPN)** | Tunnel down, BGP flap | Azure Monitor → Action Groups |
| **Alert (ER)** | Circuit down, BGP down | Azure Monitor → Action Groups |

---

## 10. DIAGNOSTIC TOOLS

| Tool | Purpose | Access |
|------|---------|--------|
| **NSG Flow Logs** | IP-level allow/deny audit | Storage Account + Log Analytics |
| **Effective NSG Rules** | Computed rules for specific NIC | Portal → NIC → Effective NSG Rules |
| **Effective Routes** | Computed routes for specific subnet/NIC | Portal → Subnet → Effective Routes |
| **Connection Monitor** | End-to-end connectivity, latency | Portal → Network Watcher → Connection Monitor |
| **IP Flow Verify** | Is specific IP flow allowed by NSG? | Portal → NIC → IP Flow Verify |
| **Next Hop** | Where will traffic go from this source/dest? | Portal → NIC → Next Hop |
| **Network Watcher VNet View** | Visual topology of VNet/connections | Portal → VNet → Network Watcher |
| **Log Analytics (KQL)** | Advanced log analysis across all network services | Portal → Log Analytics → Logs |
| **Microsoft Sentinel** | SIEM/SOAR (threat detection, automated response) | Portal → Sentinel → Investigations |
| **Azure Monitor Metrics** | Real-time metrics (bandwidth, packets, connections) | Portal → Resource → Metrics |
| **Activity Log** | Configuration changes (who changed what, when) | Portal → Activity Log |
| **Resource Graph** | Query resources across subscriptions | Portal → Resource Graph |
| **ExpressRoute Explorer** | ER circuit details, BGP routes, provider status | Portal → ExpressRoute → Explorer |
| **Virtual WAN Dashboard** | Hub status, attachment health, connections | Portal → Virtual WAN → Dashboard |
| **Private Endpoint Network Interface** | PE IP, status, group IDs | Portal → Private Endpoint → Properties |
| **Storage Explorer** | Blob/File management | Portal → Storage Account → Storage Explorer |
| **Test Connection (AFD)** | Front Door health/connectivity tests | Portal → Front Door → Test Connection |

---

## 11. L3 INTERVIEW QUESTIONS

### Basic
**Q: What is an NSG?**
A: A Network Security Group is a virtual firewall that filters network traffic to/from resources in an Azure VNet. Rules are evaluated by priority (lowest first), first match wins. Can be applied at subnet or NIC level.

### Basic
**Q: What is the difference between Allow VNet Access and Deny All Inbound in NSG default rules?**
A: Allow VNet Access (priority 65500) allows all traffic within the same VNet (including peered VNets). Deny All Inbound (priority 65503) blocks all traffic from internet to VNet unless an explicit Allow rule exists. The higher priority number means lower priority — so VNet Access is evaluated first.

### Intermediate
**Q: What is the difference between Azure Firewall and NSG?**
A: NSG operates at L3/L4 (IP/port/protocol), applied per subnet or NIC. Firewall operates at L3-L7 (can do FQDN-based rules, threat intelligence), is centralized (deployed once in Hub), and provides SNAT/DNAT. NSG is the first filter; Firewall is the deep inspection engine.

### Intermediate
**Q: What is Private Endpoint and why is it important?**
A: Private Endpoint creates a private network interface in your VNet with a private IP, allowing you to connect to Azure PaaS services (SQL, Storage, etc.) privately over Azure's backbone. No public endpoint needed, no internet traffic, reduced attack surface.

### Intermediate
**Q: What is the difference between VPN Gateway and ExpressRoute?**
A: VPN Gateway sends encrypted traffic over the internet (IPsec tunnel), cheaper but higher latency (10-50ms) and lower throughput (up to 2.5 Gbps). ExpressRoute is a dedicated private circuit (not over internet), more expensive, ultra-low latency (<1ms), high throughput (up to 100 Gbps). Best practice: ExpressRoute primary + VPN backup.

### Intermediate
**Q: What are the five Cosmos DB APIs?**
A: SQL (JSON documents), MongoDB (BSON documents), Gremlin (graph), Cassandra (wide-column), Table (key-value).

### L3
**Q: Why does VPN Gateway need a GatewaySubnet?**
A: GatewaySubnet is a dedicated subnet (minimum /27) where VPN Gateway resources are deployed. Azure requires this specific subnet name and minimum size to deploy the gateway VMs, public IPs, and routing infrastructure. Without it, VPN Gateway deployment will fail.

### L3
**Q: Explain how Application Gateway WAF works with Front Door for global protection.**
A: User → Front Door (global entry point, global WAF, CDN, SSL) → Application Gateway v2 (regional WAF, L7 routing, backend health probes) → Backend VMs. Front Door protects globally (DDoS, bot management); Application Gateway provides regional L7 protection, URL-based routing, and backend health monitoring. Defense in depth — both layers must pass.

### L3
**Q: A user reports they can't access a PaaS service (Azure SQL) from their VM. The VM can access the internet. What are ALL possible causes?**
A:
1. Private DNS Zone not linked to VM's VNet (SQL name resolves to public IP, not private)
2. Public access on SQL Server disabled AND VM not connected to private endpoint
3. NSG on VM subnet blocking outbound 1433
4. UDR not pointing traffic correctly (or pointing to Firewall that blocks SQL)
5. Private Endpoint not provisioned or in failed state
6. SQL Firewall rules not including VM IP (if public access)
7. VM DNS pointing to wrong DNS server
8. Private Link connection not approved (waiting for approval)
Most common: DNS resolution — VM resolves SQL name to public IP, but public access is disabled.

### L3
**Q: Explain the concept of "Allow Forwarded Traffic" in VNet peering and why it's critical for hub-and-spoke.**
A: Allow Forwarded Traffic enables traffic from a remote VNet to transit through the local VNet. In hub-and-spoke: Spoke-1 → Hub (transit) → Spoke-2. Without Allow Forwarded Traffic on Hub's peering to Spoke-1, traffic from Spoke-2 passing through Hub would be dropped. It's REQUIRED on the Hub side for every Spoke peering. Most common cause of "VNet peering works one way but not the other" is missing this setting.

### Senior L3
**Q: Design a secure data platform network architecture for a healthcare company (HIPAA compliance) with multi-region, ExpressRoute, and all PaaS services via Private Endpoint.**
A:
  Architecture:
    Virtual WAN: Global hub (East US + West US)
    → Each region: Virtual WAN Hub (Standard, Scale Units: 4)
    → Spoke VNets: Per application (isolated)
    → Firewall: Standard HA pair (per region)
    → Bastion: AzureBastionSubnet (per region)

  Security:
    → All traffic via Firewall (UDR: 0.0.0.0/0 → Firewall)
    → All PaaS: Private Endpoint (no public access)
    → WAF: Application Gateway v2 (OWASP 3.2, Prevention)
    → DDoS: Standard (custom alerts)
    → NSG: Deny-internet-by-default on all subnets
    → NSG Flow Logs: All NSGs → Log Analytics (HIPAA retention)

  Connectivity:
    → ExpressRoute: Dual circuits (different providers, Premium failover)
    → VPN: Backup (VpnGw3, auto-failover)
    → On-prem: Two routers (active/active)
    → Private DNS: Linked to all Spoke VNets
    → DNS forwarding to on-prem DNS for HIPAA domain resolution

  Compliance:
    → All data services: CMK encryption
    → Private endpoints only (no public endpoints)
    → Audit logs: All network access logged (NSG, Firewall, Bastion)
    → Access: Entra ID + MFA + Conditional Access (compliant devices only)
    → Backup: Encrypted, geo-redundant, 7-year retention

### Expert
**Q: You have 50 VNets across 5 regions connected via Virtual WAN. A specific Spoke VM can suddenly no longer reach the internet. All other VMs work. Trace the troubleshooting methodology.**
A:
1. VM status: Running? Network connected? (Azure status check)
2. NSG: Effective NSG Rules on VM NIC — anything denying outbound 80/443?
3. NSG Flow Logs: Is outbound traffic being logged? Denied by which rule?
4. UDR: Effective Routes on VM subnet — 0.0.0.0/0 → Virtual WAN Hub?
5. Virtual WAN: Is Spoke attachment healthy? (Portal → Virtual WAN → Attachments)
6. Firewall: Healthy? HA pair active? Throughput within limits? Rule denying this VM/IP?
7. DNS: Can VM resolve internet hostnames? (nslookup google.com → must resolve)
8. Firewall DNS Proxy: If configured, is Firewall resolving DNS correctly?
9. Private Link: Is there a Private Endpoint for the target? (if accessing specific service)
10. Compare: Effective NSG Rules on VM NIC vs. a working VM (differences?)
11. Compare: Effective Routes on VM subnet vs. a working VM subnet (differences?)
12. Recent changes: Activity Log — what changed in the last hour?
Most common for single VM: NSG on VM NIC (NIC-level NSG) blocking outbound traffic, or DNS resolution failure.

### Tricky
**Q: "Azure Firewall is in active/active mode, but all egress traffic goes through only one firewall. Why?"**
A:
Possible causes:
1. SNAT distribution: Firewall uses hash-based SNAT distribution — if connections are few, they all hash to one firewall (needs enough connections for distribution)
2. UDR: Only points to one Firewall IP (check: UDR next hop should have both Firewall IPs for active/active)
3. HA pairs: Active/Active requires both firewalls in same Availability Zone; if one is in different AZ, it won't receive traffic
4. Distribution: Two firewalls in active/active don't mean equal traffic distribution — it means both can serve traffic; load is hash-based
5. Connection count: Low connection count = poor hash distribution (all connections stick to one firewall)
6. Firewall policy: If both have different policies, traffic goes to the one with matching policy
7. Scaling: One firewall might be overwhelmed if connection count exceeds its capacity
Fix: Check UDR (must point to both firewalls for distribution), check HA pair configuration, increase connection count for better distribution, monitor per-firewall throughput.

### Tricky 2
**Q: "I have VNet Peering between Spoke-1 and Spoke-2 (direct). I also have a Virtual WAN Hub with both attached. Traffic between them goes through Virtual WAN instead of direct peering. Why?"**
A:
This happens because:
1. UDR on Spoke-1: 0.0.0.0/0 → Virtual WAN Hub (transitive via Hub)
2. UDR on Spoke-2: 0.0.0.0/0 → Virtual WAN Hub (transitive via Hub)
3. Even though Spoke-1 ↔ Spoke-2 peering exists, the UDR takes precedence for 0.0.0.0/0 (more specific than direct peering for any destination in the Hub's address space)
4. Most UDRs are configured with 0.0.0.0/0 → Virtual WAN Hub for centralized Firewall inspection
5. Traffic to 10.2.0.0/16 (Spoke-2's address space) from Spoke-1: Goes through Hub (via UDR) because UDR covers 0.0.0.0/0 which includes Spoke-2's address space
6. Direct peering between Spoke-1 and Spoke-2: Only works for traffic that goes through the VNet peering directly (which doesn't happen if UDR overrides)
Fix: If you want direct peering traffic (spoke-to-spoke without Hub):
- Specific UDR: 10.2.0.0/16 → VNet local (direct, not via Hub)
- But: This means Spoke-1 → Spoke-2 traffic bypasses Firewall (no inspection)
- Trade-off: Performance vs. security (direct peering = faster but no inspection)

### Tricky 3
**Q: "If I enable Private Endpoint for Azure SQL DB, will my Firewall rules still work?"**
A:
It depends on the configuration:
1. WITH Private Endpoint AND "Allow Azure services and resources to access this server" enabled:
   → Traffic from Azure services goes through Firewall rules
   → Traffic from your VNet (via Private Endpoint) bypasses Firewall entirely (direct private route)
   → Your VMs connect via private IP, never through Firewall
   → Firewall is not in the path for Private Endpoint traffic

2. WITH Private Endpoint AND "Allow Azure services" DISABLED:
   → Only Private Endpoint connections work
   → No public access (Firewall cannot inspect because it's not in path)
   → Firewall rules are irrelevant for VMs (they use Private Endpoint)

3. WITHOUT Private Endpoint (only Firewall):
   → VMs route 0.0.0.0/0 → Firewall → SNAT → Internet → SQL DB (public endpoint)
   → Firewall DNAT or direct: VM → Firewall → SQL DB on port 1433
   → Firewall evaluates Application Rules (FQDN: *.database.windows.net) and Network Rules (port 1433)

Key insight: Private Endpoint traffic BYPASSES Firewall by design. If you want Firewall to inspect SQL traffic, you must NOT use Private Endpoint and instead route through Firewall (which adds latency and exposes SQL publicly unless Firewall has explicit rules). Best practice: Private Endpoint for access; Firewall for internet-bound traffic; no need to inspect Private Link traffic (it's private by definition).

---

## 12. SCENARIO-BASED QUESTIONS

### Scenario 1: "VM can't reach the internet, but other VMs can"
**Architecture:** VM → NSG → UDR → Firewall → Internet.
**Dependencies:** NSG, UDR, Firewall health, DNS, VM NIC.
**Checks:**
1. VM has network connectivity (Azure status check)
2. NSG on VM subnet/NIC: allow outbound 80/443?
3. UDR: 0.0.0.0/0 → Firewall? (Check effective routes)
4. Firewall: Healthy? HA pair active? Throughput OK?
5. DNS: Can VM resolve hostnames?
6. VM-specific NSG (NIC level): Blocking?
**Root Cause:** VM-specific NIC NSG blocking outbound traffic while subnet NSG allows.
**Fix:** Remove or update NIC NSG on VM.
**Validation:** nslookup from VM; Test-Connection to internet IP; Browser test.

### Scenario 2: "Application Gateway returning 502 but backend health is Healthy"
**Architecture:** User → Front Door → App Gateway (WAF) → Backend VMs.
**Dependencies:** WAF rules, backend VM app, App Gateway config, DNS.
**Checks:**
1. Backend health: Healthy (confirmed)?
2. WAF: Is a rule blocking? (Check WAF logs)
3. Backend VMs: App listening on expected port?
4. App Gateway: Listener/rule configuration correct?
5. Backend pool: VMs listed?
6. NSG between App GW and VMs: Allow from App GW subnet?
**Root Cause: WAF blocking specific request pattern (OWASP rule triggered by legitimate request).**
**Fix: WAF in Detection mode → identify blocked pattern → add exclusion rule → switch to Prevention.**
**Validation: Backend health: Healthy; WAF logs: no blocks for test requests; Response: 200 OK.**

### Scenario 3: "ExpressRoute primary down, VPN backup didn't failover"
**Architecture:** ER (primary) + VPN (backup) → On-prem.
**Dependencies:** ER circuit health, VPN Gateway health, BGP routing, ExpressRoute Gateway.
**Checks:**
1. ER Circuit status: Succeeded/Failed? (Portal: ExpressRoute → Circuit)
2. VPN Gateway: Running? Connections: Connected?
3. BGP: On-prem router advertising VPN routes with longer AS-path? (VPN routes should be less preferred)
4. ER Gateway: Healthy?
5. BGP: ER routes withdrawn (circuit down)? VPN routes taking over?
6. BGP convergence time: Expected 2-5 seconds for failover.
7. If using Global Reach: Secondary circuit active?
**Root Cause: BGP not configured for auto-failover (VPN routes not longer AS-path or not advertised).**
**Fix: Configure BGP route prepend for VPN routes (make them less preferred); Configure AS-path prepend on VPN (add extra ASN hops); Verify BGP session on both sides.**
**Validation: ER down (simulate); BGP converges to VPN routes within 5 seconds; Traffic flows via VPN.**

### Scenario 4: "Private Endpoint DNS not resolving from VNet"
**Architecture:** VM → DNS → Private DNS Zone → Private IP → Private Endpoint → PaaS.
**Dependencies:** Private DNS Zone, VNet link, VM DNS settings, Private Endpoint status.
**Checks:**
1. Private DNS Zone: Linked to VM's VNet?
2. VM DNS: Configured correctly (not overriding with custom DNS)?
3. nslookup: VM resolves privatelink.* to private IP?
4. Private Endpoint: Provisioned (Succeeded)?
5. NSG: Allows DNS traffic (port 53)?
6. Private DNS Resolver: If used, rules configured?
**Root Cause: Private DNS Zone not linked to VM's VNet (most common).**
**Fix: Link Private DNS Zone to correct VNet; nslookup from VM → private IP.**
**Validation: nslookup: Correct private IP; VM → PaaS connection works privately.**

### Scenario 5: "VPN Gateway connection dropped after Virtual WAN Hub was created"
**Architecture:** VPN Gateway was standalone → Virtual WAN Hub created → VPN Gateway attached to Hub.
**Dependencies:** Virtual WAN attachment, VPN Gateway config, on-prem router, BGP.
**Checks:**
1. Virtual WAN: VPN Gateway attachment status: Succeeded?
2. VPN Gateway: Still running? Configuration preserved?
3. UDR: Updated automatically by Virtual WAN? (Check: Subnet → Effective Routes)
4. On-prem router: BGP session re-established?
5. BGP: Routes redistributed via Virtual WAN?
6. Connection: VPN tunnel: Established? (Portal: VPN Gateway → Connections)
7. NSG: New rules from Virtual WAN? Check: Subnet NSG additions.
**Root Cause: Virtual WAN attachment changed routing (VNet-to-VNet traffic not enabled on attachment)?**
**Fix: Enable "Allow VNet to VNet traffic" on Virtual WAN attachment; Verify BGP session; Check UDR.**
**Validation: VPN tunnel: Connected; BGP: Routes learned; Test: Connection from on-prem to VM.**

### Scenario 6: "All VMs in Subnet lose internet after Firewall update"
**Architecture:** All VMs → UDR 0.0.0.0/0 → Firewall → Internet.
**Dependencies:** UDR, Firewall health, NSG, DNS.
**Checks:**
1. Firewall: Running? Both instances healthy? (Portal: Firewall → Health)
2. Firewall rules: Updated? Old rules missing?
3. Firewall throughput: Near limit? (Firewall metrics)
4. UDR: Still 0.0.0.0/0 → Firewall?
5. DNS: Can VMs resolve? (Firewall DNS proxy affected?)
6. SNAT: Exhausted? (Check: Firewall SNAT usage)
7. NSG: Changed during Firewall update?
8. Recent Activity Log: What was changed during Firewall update?
**Root Cause: Firewall policy update removed default rules or added Deny-all at higher priority. Or Firewall HA failover caused brief outage.**
**Fix: Review Firewall rules and policy; restore missing rules; check HA pair status; restart if needed.**
**Validation: VM → Internet; Firewall logs: traffic flowing; Effective routes: 0.0.0.0/0 → Firewall.**

### Scenario 7: "Front Door → Application Gateway → Backend returning 403 Forbidden"
**Architecture:** User → Front Door (global WAF) → App Gateway (regional WAF) → VMs.
**Dependencies:** Front Door WAF, App Gateway WAF, backend health, DNS.
**Checks:**
1. Front Door WAF: Blocking? (Check Front Door logs)
2. App Gateway WAF: Blocking? (Check WAF logs)
3. Backend health: Healthy?
4. Front Door routing rule: Targets correct App Gateway endpoint?
5. App Gateway listener: Host header matches Front Door hostname?
6. Backend VM: App responding? (curl directly to VM IP)
7. Custom domains: CNAME correct? (Front Door hostname → App Gateway)
8. SSL certificates: Valid for hostname?
**Root Cause: Front Door WAF OWASP rule blocking legitimate request; OR host header mismatch; OR CNAME not configured.**
**Fix: Review Front Door WAF logs; add exclusion; verify host header; configure CNAME.**
**Validation: Front Door → 200 OK; WAF logs: no blocks for test request.**

### Scenario 8: "Cosmos DB Private Endpoint works from VM, but from on-prem it doesn't"
**Architecture:** VM (VNet) → Private Endpoint → Cosmos DB. On-prem → VPN → VNet → Private Endpoint → Cosmos DB.
**Dependencies:** VPN, DNS routing, NSG, Private Endpoint, on-prem network.
**Checks:**
1. VPN: Connected? (Tunnel established)
2. BGP: On-prem routes advertised?
3. DNS: On-prem resolves Cosmos DB private endpoint hostname → private IP? (Must route DNS through VNet DNS or Private DNS Resolver)
4. NSG: VPN Gateway subnet allows traffic to Cosmos DB VNet?
5. UDR on VPN Gateway subnet: Routes to Cosmos DB VNet?
6. Cosmos DB: Private Endpoint connection approved?
7. On-prem firewall: Allows traffic to Cosmos DB endpoint IP?
8. On-prem DNS: Conditional forwarder to Azure DNS?
**Root Cause: DNS from on-prem doesn't resolve Cosmos DB hostname to private IP (resolves to public IP instead, which Cosmos DB denies due to "Deny public access").**
**Fix: Configure DNS conditional forwarding from on-prem DNS → Azure Private DNS Resolver → Cosmos DB resolves to private IP.**
**Validation: nslookup from on-prem PC: Cosmos DB hostname → private IP; Connection: Works from on-prem.**

### Scenario 9: "Virtual WAN Hub VNet address space conflicts with attached Spoke VNet"
**Architecture:** Hub: 10.0.0.0/16; Spoke: 10.0.1.0/16 (overlapping).
**Dependencies:** Address spaces, Virtual WAN attachment, VNet peering.
**Checks:**
1. Hub VNet address space: 10.0.0.0/16
2. Spoke VNet address space: 10.0.0.0/16? (overlapping!)
3. Virtual WAN: Attachment fails? (Portal: Virtual WAN → Attachments)
4. Routes: Conflicting routes? (Portal: Subnet → Effective Routes)
5. Traffic: Intermittent, unpredictable routing.
**Root Cause: Hub and Spoke VNets have overlapping address spaces (both 10.0.0.0/16). Azure cannot route between them.**
**Fix: Change Spoke VNet address space (requires downtime, re-IP everything) OR change Hub address space. Best: Use non-overlapping CIDRs from design phase.**
**Validation: Attachment: Succeeded; VM in Spoke: Can reach all resources; No routing anomalies.**

### Scenario 10: "DDoS attack saturates Application Gateway (WAF_v2 autoscale maxed)"
**Architecture:** User → Front Door → App Gateway v2 (WAF, autoscale min:2, max:10) → Backend.
**Dependencies:** DDoS protection, autoscale, Front Door WAF, App Gateway capacity.
**Checks:**
1. App Gateway: CU at 100%? (Autoscale maxed at 10 instances)
2. Front Door WAF: Blocking? (L7 protection)
3. DDoS: Standard? Mitigating?
4. Front Door: Traffic from botnet hitting Front Door?
5. App Gateway: New connections refused when at max?
6. Backend: Healthy but not receiving all traffic?
**Root Cause: L7 DDoS attack exceeding Front Door WAF rate limits and App Gateway max autoscale.**
**Fix: Increase App Gateway max autoscale (to 20+); Enable Front Door rate limiting; Contact Front Door support for larger attacks; Enable DDoS Standard + WAF custom rate limit rules; Consider geo-filtering (block attack source regions).**
**Validation: Front Door: Traffic filtered; App Gateway: Not saturated; Backend: Healthy and responding.**

---

## 13. KNOWLEDGE TEST

1. **What is the primary purpose of an NSG?**
   A Network Security Group acts as a virtual firewall that filters network traffic to and from Azure resources based on source/destination IP, port, protocol, and action (allow/deny). Rules are evaluated by priority, first match wins.

2. **What is the difference between Subnet NSG and NIC NSG?**
   Subnet NSG applies to all resources in the subnet (centralized). NIC NSG applies to a single VM NIC (granular). Both are evaluated together; most restrictive (lowest priority) rule wins.

3. **What is Azure Firewall and what tier requires Standard VNet?**
   Azure Firewall is a managed, stateful network security service (NVA) that provides centralized SNAT, DNAT, threat intelligence, and FQDN-based filtering. Standard and Premium SKUs require Standard SKU VNet (not Basic).

4. **What is WAF and where is it deployed?**
   Web Application Firewall protects web applications from OWASP Top 10 attacks (SQL injection, XSS, etc.). Deployed on Application Gateway v2 (WAF SKU). Can also be used with Azure Front Door for global WAF.

5. **What is Private Endpoint?**
   A Private Endpoint is a network interface in your VNet with a static private IP that connects privately to Azure PaaS services via Azure Private Link. No public endpoint needed; traffic stays on Azure backbone.

6. **What is the minimum size for GatewaySubnet?**
   /27 (32 addresses). This is an Azure requirement for VPN and ExpressRoute gateway deployment.

7. **What is the difference between VPN Gateway and ExpressRoute?**
   VPN Gateway: Encrypted traffic over public internet (IPsec), up to 2.5 Gbps, ~$5/month gateway cost. ExpressRoute: Dedicated private circuit (not over internet), up to 100 Gbps, ~$2000+/month circuit cost. VPN: higher latency, lower bandwidth; ER: lower latency, higher bandwidth.

8. **What is ExpressRoute FastPath?**
   FastPath bypasses the ExpressRoute Gateway for ultra-low latency (sub-millisecond). Bypasses gateway → direct BGP session between circuit and VNet. Requires: GatewaySubnet with UDR to Gateway bypassed, BGP configured on VM/subnet directly. NOT recommended for most use cases (no gateway-level features, more complex).

9. **What is Virtual WAN?**
   Virtual WAN is a Microsoft-managed service that provides centralized hub-and-spoke routing across Azure regions. Replaces manual VNet peering, VPN gateway, and ExpressRoute gateway configuration with centralized routing. Supports VNet attachments, VPN, ExpressRoute, Security Partners, and P2S connections.

10. **What is BGP and why is it important for VPN/ExpressRoute?**
    BGP (Border Gateway Protocol) is dynamic routing protocol that automatically learns and advertises routes between Azure and on-premises. BGP enables: automatic failover (route withdrawal/advertisement), multiple path selection, and dynamic route propagation. Essential for ExpressRoute (required); recommended for VPN (optional but strongly recommended).

11. **What is the difference between ExpressRoute Gateway and VPN Gateway?**
    ExpressRoute Gateway: Connects ExpressRoute circuits to VNets (handles dedicated private connectivity). Cannot be used for VPN connections. VPN Gateway: Connects VPN tunnels (S2S and P2S) to VNets. Cannot be used for ExpressRoute. Both are deployed in GatewaySubnet.

12. **What is SNAT in Azure Firewall?**
    SNAT (Source Network Address Translation) translates private source IPs of VM traffic to Firewall's public IP addresses when accessing the internet. All outbound internet traffic from VMs appears to come from Firewall's public IPs, not from VM IPs. SNAT pool: range of IPs on Firewall for translation.

13. **What is DNAT in Azure Firewall?**
    DNAT (Destination Network Address Translation) translates inbound traffic from Firewall's public IP:port to a private VM IP:port. Enables: Internet → Public IP:Port → VM Private IP:Port (with Firewall inspection in between). Requires: UDR on VM subnet pointing back to Firewall for return traffic.

14. **What is the OWASP Top 10?**
    OWASP Top 10 is a standard list of the most critical web application security vulnerabilities. Includes: Injection, Broken Authentication, Sensitive Data Exposure, XML External Entities, Broken Access Control, Security Misconfiguration, Cross-Site Scripting, Insecure Deserialization, Known Vulnerabilities, Insufficient Logging. WAF (Application Gateway) protects against these.

15. **What is the difference between DDoS Basic and Standard protection?**
    Basic: Free, always-on, 5 common attack types, no alerting, no integration. Standard: Paid (~$29/day/region), adaptive tuning, custom alerts, WAF integration, 400 Gbps mitigation, detailed analytics. Standard required for production.

16. **What is a Front Door and how does it relate to WAF?**
    Azure Front Door is a global entry point for web applications with built-in WAF, CDN, load balancing, and SSL offloading. Front Door WAF: Global L7 protection (OWASP rules, rate limiting, bot management). Application Gateway WAF: Regional L7 protection (routing, backend health, cookies). Combined: Global (Front Door) + Regional (App GW) = defense in depth.

17. **What is the IPsec protocol used for VPN Gateway?**
    IPsec (Internet Protocol Security) is the protocol suite used for VPN Gateway tunnels. Components: ESP (Encapsulating Security Payload — encryption), AH (Authentication Header — authentication), IKE (Internet Key Exchange — key negotiation). Ports: UDP 500 (IKE/NAT-T discovery), UDP 4500 (NAT-T traversal), Protocol 50 (ESP), Protocol 51 (AH).

18. **What are the IKEv2 phases?**
    IKE Phase 1: Establishes secure management channel between VPN devices (encryption, authentication, key exchange). IKE Phase 2: Establishes IPsec SAs (data encryption parameters). Phase 1 uses ISAKMP (UDP 500/4500). Phase 2 uses ESP for data encryption.

19. **What is Security Partner in Virtual WAN?**
    Security Partner is an NVA (Network Virtual Appliance) deployed in Virtual WAN Hub for centralized traffic inspection. Firewall, IDS/IPS, or other NVAs can be attached as Security Partners. Traffic from all attached Spoke VNets routes through the Security Partner automatically (no manual UDR required).

20. **What is QoS in Virtual WAN?**
    Quality of Service policies in Virtual WAN prioritize different traffic types (e.g., VoIP = high priority, bulk data = low priority). QoS policies: Associate with Hub attachments, set priority levels (1-6), define bandwidth limits. Important for: Real-time applications, ensuring critical traffic isn't delayed by bulk transfers.

21. **What is ExpressRoute Global Reach?**
    Global Reach connects two ExpressRoute circuits (in different regions) privately, allowing traffic to flow between them without going through the internet or Azure. Use: Cross-region DR, provider redundancy, geographic circuit diversity. Configuration: Link two circuits in Portal, configure BGP (advertise each other's prefixes).

22. **What is FastPath for ExpressRoute?**
    FastPath bypasses the ExpressRoute Gateway for ultra-low latency (sub-millisecond) connectivity. Instead of traffic going through Gateway (with gateway processing), it goes directly from circuit to VNet via BGP. Requirements: VM configured with BGP (not common), GatewaySubnet with UDR bypassing Gateway, supported SKU. Use: Ultra-low latency scenarios (trading, real-time).

23. **What is BYO Hub in Virtual WAN?**
    Bring Your Own Hub: Use an existing VNet as Virtual WAN Hub instead of auto-creating one. Benefits: Preserve existing VNet configuration, attach VPN/ER/security partners to existing infrastructure, migrate gradually from traditional to Virtual WAN. Limitations: BGP route propagation not supported (static routes only), some features unavailable.

24. **What is Security Stacking in Virtual WAN?**
    Security Stacking: Deploy NVAs (firewalls, IDS/IPS) in each Spoke VNet (not just Hub) for distributed inspection. Virtual WAN automates deployment (via Firewall Manager). Each Spoke gets its own NVA, but all follow central policy (from Firewall Manager). Benefits: Lower latency (no hairpinning through Hub), per-Spoke isolation, scalable.

25. **What is the maximum number of VNet attachments to a Virtual WAN Hub?**
    1000 VNet attachments per Virtual WAN (across all hubs in the Virtual WAN). Hub scale units: Minimum 2 (HA), maximum 125. Each attachment associates one VNet with one Virtual Hub. VNet cannot be attached to multiple hubs simultaneously.

26. **What happens if you create both VNet Peering and Virtual WAN between the same VNets?**
    Both can coexist but will cause routing conflicts. VNet Peering provides direct (non-transitive) routing. Virtual WAN provides transitive routing via Hub. Traffic will follow UDRs (which override VNet peering for overlapping prefixes). Best practice: Choose ONE approach (Virtual WAN preferred; do NOT use VNet peering with Virtual WAN).

27. **What is the recommended VPN Gateway SKU for production with ExpressRoute backup?**
    VpnGw2 or higher (zone-redundant). VpnGw2: 1 Gbps throughput, zone-redundant, supports BGP, P2S (IKEv2 + OpenVPN). For > 1.25 Gbps VPN: Consider HighPerformanceVPNGW (up to 50 Gbps) or use multiple VPN Gateways.

28. **What is the difference between P2S with IKEv2 and OpenVPN?**
    IKEv2: Native OS support (Windows/macOS), faster, better network mobility (survives network changes), requires EAP certificate or EAP-MSCHAPv2. OpenVPN: Cross-platform (requires client), flexible (supports TCP port 443 — useful when UDP blocked), easier configuration. Both support MFA.

29. **What is the role of Private DNS in Private Endpoint connectivity?**
    Private DNS Zone maps service names (e.g., privatelink.database.windows.net) to private IPs. Linked to VNet(s) where Private Endpoints reside. VM DNS query → Private DNS Zone → private IP → Private Endpoint → PaaS service. Without Private DNS Zone link, VM resolves service name to public IP (may fail if public access disabled).

30. **What is the attack surface reduction from using Private Endpoints?**
    Without Private Endpoint: PaaS service has public endpoint → accessible from internet → attack surface = entire internet. With Private Endpoint: PaaS service has NO public endpoint (public access disabled) → accessible ONLY via Private Endpoint in your VNet → attack surface = your VNet. Reduction from "internet-sized" to "your-VNet-sized."

31. **What is a WAF custom rule?**
    A user-defined rule in Application Gateway WAF that allows/denies/blocks traffic based on specific conditions (match variables: IP, headers, URI, body, etc.). Used for: Allowlisting trusted IPs, blocking specific attack patterns, rate limiting, geo-blocking. Evaluated before managed rules (lower priority number = first evaluated).

32. **What is managed rule set in WAF?**
    Predefined rule sets managed by Microsoft (OWASP CRS, MISE, etc.) that protect against common web attacks. OWASP 3.2: Core rules covering SQL injection, XSS, RFI, LFI, etc. MISE (Managed Intrusion Set Evaluation): Rules for known attack types. Automatic updates from Microsoft. Less maintenance than custom rules but may produce false positives.

33. **What is the difference between ExpressRoute Standard and Premium failover?**
    Standard failover: When primary circuit fails, secondary circuit takes over with 2-3 seconds downtime (BGP reconvergence). Premium failover: Near-zero downtime (seamless failover, no BGP reconvergence delay). For mission-critical: Always use Premium.

34. **What is the difference between ExpressRoute metered and unlimited (QoS) bandwidth?**
    Metered: Pay per GB of data transferred (e.g., $0.0288/GB). Suitable for variable traffic patterns with low volume. Unlimited (QoS): Fixed monthly fee regardless of data volume. Suitable for high-volume, consistent traffic. Pricing difference: Unlimited more expensive per month but cheaper per GB for high volume.

35. **What is the significance of the AS path in BGP for VPN/ExpressRoute failover?**
    AS (Autonomous System) path length determines BGP route preference — shorter AS path = preferred route. For VPN backup: Configure on-prem router to add extra AS hops for VPN routes (AS path prepend), making ExpressRoute routes preferred (shorter AS path). When ExpressRoute fails: VPN routes become preferred (longer AS path now shortest). BGP auto-converges within 2-3 seconds.

36. **What is DDoS rapid response?**
    With DDoS Standard, Microsoft's DDoS response team can be activated during large-scale attacks. Triggered via: Alert → Action Group → Manual activation or auto-trigger (configurable). Response team investigates, tunes mitigations, and coordinates with connectivity providers if needed. Available for Standard customers only.

37. **What is the difference between Application Gateway v1 and v2 SKUs?**
    v1: Legacy, no autoscale, no Private Link, limited WAF features, no autoscaling, single instance (unless manual). v2: Autoscale (min/max instances), Private Link support, wildcard hostnames, autoscaling, zone-redundant, autoscale based on CU usage. Always use v2 for new deployments.

38. **What is the recommended approach for P2S VPN certificate management?**
    Generate root CA certificate → Issue client certificates from CA → Upload root CA (.cer) to VPN Gateway → Distribute client certificates to users → Validate client certificate on VPN connection. Renewal: Generate new client cert before expiration → Distribute to users → VPN Gateway auto-validates (no portal update needed for client cert renewal). Revocation: Use CRL or OCSP endpoint configured on VPN Gateway.

39. **What is a Hub-Sponsored VNet vs a Self-Sponsored VNet in Virtual WAN?**
    Self-Sponsored: Virtual WAN auto-creates a Hub VNet for you (simpler, no existing VNet). Hub-Sponsored (BYO Hub): You provide an existing VNet as the Hub (preserve existing config, attach to existing infrastructure). Choice depends on: New deployment → Self-Sponsored; Existing VNet with VPN/ER already attached → BYO Hub.

40. **What happens when Firewall in Security Stacking mode receives traffic from a Spoke?**
    In Security Stacking: Each Spoke VNet has its own Firewall (attached via Virtual WAN + Firewall Manager). Traffic from Spoke → Virtual WAN Hub → Spoke's local Firewall (not Hub Firewall) → Target. Firewall Manager ensures all Spoke Firewalls have same central policy. Local rules: Each Spoke Firewall can add supplementary rules. Result: Distributed inspection, lower latency (no hairpinning), consistent policy.

41. **What is the difference between ExpressRoute Private Peering and Microsoft Peering?**
    Microsoft Peering: Routes to Azure services (compute, storage) and Microsoft services (M365, Azure AD). Most common, default configuration. Private Peering: Routes to Azure private services (Private Link endpoints, SQL Private, Storage Private). Used when: Accessing PaaS via private endpoints over ExpressRoute. Both configured on the same circuit but with different VLANs and BGP sessions.

42. **What is the recommended retention for NSG Flow Logs for compliance?**
    Minimum 90 days in Log Analytics (operational); 1 year recommended for security analysis; Transfer to Storage Account for long-term retention (7+ years) for compliance. Retention in Log Analytics is configurable (90 days default). Storage Account: Immutable for compliance. Backup retention aligned with compliance requirements (HIPAA: 6 years; PCI: 1 year; SOX: 7 years).

43. **What is the role of the Microsoft backbone in Virtual WAN and ExpressRoute Global Reach?**
    Microsoft backbone = private Azure network infrastructure (not internet). Virtual WAN: Inter-region traffic flows through Microsoft backbone (not over internet), providing low latency and security. ExpressRoute Global Reach: Two circuits in different regions connected via Microsoft backbone (private), allowing cross-region traffic without internet routing. Both use the same underlying Microsoft network fabric.

44. **How does Firewall Manager handle conflicting rules between Spoke Firewalls?**
    Firewall Manager enforces "Security Stance":
    Required: Only Firewall policy rules apply; local rules are REMOVED (strict central control). No conflict possible.
    Recommended: Firewall policy rules + local rules can coexist. Policy rules take precedence over local rules (lower priority). Local rules supplement policy rules for specific Spoke needs. Conflicts: Lower priority (policy) wins over higher priority (local). Best practice: Use Required mode for strict compliance; Recommended for flexibility.

45. **What is the max number of S2S VPN connections per VPN Gateway?**
    Up to 30 site-to-site VPN connections per VPN Gateway (VpnGw1-VpnGw5). Combined total bandwidth shared across all connections (e.g., VpnGw2: 1 Gbps total across 30 tunnels). Individual tunnel throughput limited by slowest link. For more connections: Deploy additional VPN Gateways (load balance with BGP prepend).

---

## 14. DIAGNOSTIC SETTINGS — NETWORK SECURITY

| Resource | Log Category | Destination |
|----------|-------------|------------|
| **NSG** | Flow Logs (v1/v2) | Storage Account + Log Analytics |
| **Azure Firewall** | All logs (AzureFirewall, AppRule, NetRule, TLS, IDPS) | Log Analytics |
| **Firewall Manager** | Policy deployment, change logs | Log Analytics |
| **Application Gateway** | Access, WAF, Performance, SSL, Health | Log Analytics + Storage |
| **Front Door (WAF)** | Access, WAF, DDoS, Front Door health | Log Analytics + Storage |
| **DDoS Protection** | Mitigation metrics, alert logs | Log Analytics + Alerts |
| **VPN Gateway** | Connect, IKE, P2S, BGP | Log Analytics + Storage |
| **ExpressRoute Gateway** | Connect, BGP, health | Log Analytics + Storage |
| **Virtual WAN** | Hub metrics, attachment logs, route changes | Log Analytics + Monitor |
| **Bastion** | Session logs, connection logs | Log Analytics + Storage |
| **Private Endpoint** | Provisioning, connection, access logs | Log Analytics |
| **Private DNS Resolver** | Query logs, resolution logs | Log Analytics |
| **Connection Monitor** | Test results, connectivity data | Log Analytics |
| **Load Balancer** | Health probes, SNAT, flow logs | Log Analytics + Storage |
| **Activity Log** | All configuration changes | Log Analytics + Storage |
| **Alerts (All)** | Rule triggers, thresholds exceeded | Azure Monitor → Action Groups (Email, SMS, Teams, runbooks) |

---

## 15. L3 GAP CHECK

| Topic | Status |
|-------|--------|
| NSG (concepts, rules, priorities, evaluation, Flow Logs) | ✅ Covered |
| NSG as Defense Layer 1 in security stack | ✅ Covered |
| NSG Flow Logs for security auditing | ✅ Covered |
| Azure Firewall (SKUs, architecture, SNAT, DNAT) | ✅ Covered |
| Firewall Rules (Application, Network, Threat Intel) | ✅ Covered |
| Firewall DNS Proxy (AzureFirewallDns subnet) | ✅ Covered |
| Firewall Manager (policies, security stacking) | ✅ Covered |
| Firewall Threat Intelligence (Alert/Deny modes) | ✅ Covered |
| Application Gateway v2 (WAF, autoscale, architecture) | ✅ Covered