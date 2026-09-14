# SAS / ACCESS / SECURITY, STORAGE DATA PROTECTION, MANAGED IDENTITIES — DEEP, AZURE KEY VAULT, APP REGISTRATIONS, SERVICE PRINCIPALS & ENTERPRISE APPLICATIONS, ENTRA PROXY APPLICATION — L3 DEPTH 🔴 CRITICAL

---

## 1. CONCEPT

**Azure Identity, Access, and Security** is the discipline of controlling *who* can access *what*, *how* they authenticate, *what* they can do, and *how* data and secrets are protected. This is the foundation of Azure security — without correct identity and access management, every other security control is bypassed or ineffective.

> **The single most important identity concept:** Microsoft Entra ID (formerly Azure Active Directory) is the identity backbone of Azure. Every resource access decision flows through Entra ID: Who are you? (Authentication) → What can you do? (Authorization via RBAC) → What secrets do you use? (Key Vault / Managed Identity) → How do you access securely? (SAS tokens, Private Link, etc.). If identity is compromised, security is compromised — which is why phishing remains the #1 attack vector against Azure environments.

**Why Identity and Access Management matters:**

```
WITHOUT PROPER IDENTITY AND ACCESS MANAGEMENT:
  → Shared admin accounts (no individual accountability)
  → Over-privileged users (User Access Administrator on everything)
  → Secrets in code/repos (connection strings, API keys in plaintext)
  → No key rotation (certificates and keys never expire)
  → Service-to-service auth via username/password (credential leak risk)
  → Storage account accessible via SAS with full privileges
  → No MFA enforcement (phishing succeeds easily)
  → Stale service principals (old apps still running with expired creds)
  → Key Vault not configured (secrets in app settings, Key Vault references missing)
  → No audit trail of who accessed what secret, when
  → Permissive RBAC (Owner on all resources, no scope control)
  → Conditional Access disabled (any device, any location, no MFA)
  → Global admins everywhere (50+ global admins = catastrophic risk)
  → App registrations orphaned (100+ stale app registrations with secrets)

WITH PROPER IDENTITY AND ACCESS MANAGEMENT:
  → Individual accounts with MFA (every human user)
  → Least privilege RBAC (exact roles at exact scopes)
  → Secrets in Key Vault (never in code, CMK + soft delete + purge protection)
  → Key/cert rotation (auto-rotation policies, lifecycle management)
  → Service-to-service via Managed Identity (no credentials in code)
  → Storage access via SAS (scoped, time-limited, limited privileges)
  → MFA enforced globally (Conditional Access: all users, all apps)
  → Active service principals (review quarterly, remove stale)
  → Key Vault configured (all secrets, keys, certs stored centrally)
  → Full audit trail (Key Vault access logs, Entra ID audit logs)
  → Precise RBAC (built-in + custom roles at subscription/resource group/resource)
  → Conditional Access: compliant devices + MFA + location + risk
  → Zero global admins (break-glass only, just-in-time elevation)
  → Clean app registrations (active apps only, secrets rotated, old ones deleted)
```

**Identity and Access Architecture:**

```
┌─ IDENTITY LAYER (Microsoft Entra ID) ──────────────────────────────┐
│                                                                    │
│  Users (Human)                                                     │
│  → Entra ID user accounts (cloud-only or synced from on-prem AD)  │
│  → MFA enforcement (Conditional Access)                            │
│  → Self-Service Password Reset (SSPR)                              │
│  → Privileged Identity Management (PIM — just-in-time elevation)   │
│                                                                    │
│  Service Principals (Non-Human)                                    │
│  → App Registrations (app definitions — what app can do)           │
│  → Service Principals (app instances — who the app IS)             │
│  → Managed Identities (Azure resource identity — no credentials)    │
│                                                                    │
│  Enterprise Applications                                           │
│  → SaaS apps (Salesforce, GitHub, etc.) integrated via Entra ID   │
│  → SCIM provisioning (auto-create/disable users in SaaS)           │
│  → Federation (SAML/OIDC — single sign-on to SaaS)                 │
│                                                                    │
│  Entra Proxy Application                                           │
│  → Application Proxy: On-prem apps published via Entra ID          │
│  → Seamless SSO for on-prem apps via Entra ID                     │
│  → Connector group: On-prem proxy for remote access                │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘

┌─ ACCESS LAYER (Authorization & Secrets) ────────────────────────────┐
│                                                                    │
│  RBAC (Role-Based Access Control)                                  │
│  → Built-in roles (Owner, Contributor, Reader, etc.)               │
│  → Custom roles (granular permission definition)                    │
│  → Scope: Subscription / Resource Group / Individual Resource       │
│  → Conditional Access: Context-aware access (device, location, etc.)│
│                                                                    │
│  Azure Key Vault                                                   │
│  → Secrets, Keys, Certificates storage                              │
│  → Access Policies (v1) / RBAC (v2) for authorization               │
│  → Soft Delete + Purge Protection (data protection)                 │
│  → Managed Identity integration (app → KV without credentials)      │
│  → Certificate authority (CA) integration (auto-renew)              │
│                                                                    │
│  Shared Access Signature (SAS)                                     │
│  → Delegated access to Azure Storage resources                     │
│  → Scoped (container/blob level), time-limited, permission-limited │
│  → Token-based (no account key exposure)                            │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘

┌─ DATA PROTECTION LAYER ────────────────────────────────────────────┐
│                                                                    │
│  Storage Data Protection                                          │
│  → Encryption: SSE (service-managed), CMK (Key Vault), CSE        │
│  → Soft Delete (recover deleted blobs)                             │
│  → Immutable Blob Storage (WORM — write-once, read-many)           │
│  → Legal Hold (retain data until legal release)                    │
│  → Versioning (keep multiple versions of blob)                     │
│  → Object Lifecycle Management (tier/transition/expire)            │
│  → Azure Storage Firewall + Private Endpoint                     │
│  → SAS tokens (scoped, time-limited access)                       │
│  → Azure Defender for Storage (threat detection)                   │
│                                                                    │
│  Managed Identities (Deep)                                         │
│  → System-Assigned: Tied to resource lifecycle                    │
│  → User-Assigned: Shared identity, reusable across resources      │
│  → Identity Flows: How pods/VMs get tokens for Azure services     │
│  → Cross-tenant identities: Multi-tenant app scenarios             │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 2. ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                  AZURE IDENTITY, ACCESS & DATA PROTECTION ARCHITECTURE           │
│                                                                                   │
│  ── IDENTITY & AUTHENTICATION ──────────────────────────────────────             │
│                                                                                   │
│  Microsoft Entra ID (tenant: globalcontoso.onmicrosoft.com)                       │
│  ├── Users (human): john@contoso.com (MFA enabled, Conditional Access applied)   │
│  ├── Groups: SQL-Admins, DevOps-Team, Finance-Users (used in RBAC assignments)  │
│  ├── Service Principals: app-prod-api, app-prod-worker, ci-cd-bot                │
│  ├── Managed Identities: vmss-webapp-MI, aks-node-MI, app-service-prod-MSI       │
│  ├── App Registrations: api-contoso (definition), web-contoso (definition)       │
│  ├── Enterprise Apps: Salesforce, GitHub, ServiceNow (federated)                 │
│  └── PIM (Privileged Identity Mgmt): Global Admin — break-glass only             │
│                                                                                   │
│  ── AUTHORIZATION ───────────────────────────────────────────────────             │
│                                                                                   │
│  RBAC (Azure Resource Manager)                                                   │
│  ├── Built-in Roles: Owner, Contributor, Reader, Key Vault Secrets Officer, etc. │
│  ├── Custom Roles: "Storage Blob Deleter" (specific: storage blob delete only)  │
│  ├── Scope: Subscription → Resource Group → Resource (most restrictive wins)    │
│  └── Conditional Access: MFA + compliant device + allowed locations              │
│                                                                                   │
│  ── SECRETS & KEYS ──────────────────────────────────────────────────             │
│                                                                                   │
│  Azure Key Vault (vault-contoso-prod-eastus)                                      │
│  ├── Secrets: Connection strings, API keys, passwords (soft delete enabled)     │
│  ├── Keys: Encryption keys for CMK/TDE (key rotation policy enabled)            │
│  ├── Certificates: SSL/TLS certs (auto-renew via CA integration)                │
│  ├── Access Policies: 500+ access policies (v1) / RBAC (v2)                     │
│  ├── Network: Firewall + Private Endpoint (no public access)                     │
│  └── Audit: All access logged to Log Analytics (who accessed what, when)         │
│                                                                                   │
│  ── DATA PROTECTION ──────────────────────────────────────────────────             │
│                                                                                   │
│  Storage Account: storagecontoso (LRS/ZRS/GRS/RA-GRS)                            │
│  ├── Encryption: CMK (Key Vault) + Managed Identity (no account keys exposed)    │
│  ├── Soft Delete: 7-day (blobs recoverable after accidental delete)             │
│  ├── Immutable: Time-based (7 days) + Legal Hold (permanent until released)     │
│  ├── Versioning: Enabled (up to 50 versions per blob)                            │
│  ├── SAS: Container-scoped, read-only, 7-day expiry (delegated access)          │
│  ├── Firewall: Selected networks only (10.0.0.0/8, Virtual Network)             │
│  └── Defender for Storage: Threat detection enabled                                │
│                                                                                   │
│  ── CONNECTIVITY ───────────────────────────────────────────────────             │
│                                                                                   │
│  Entra Application Proxy: On-prem apps published via Entra ID                    │
│  ├── Connector Group: On-prem servers running Application Proxy connector        │
│  ├── Published App: on-prem-contoso (internal app accessible via Entra ID SSO)   │
│  └── Seamless SSO: Users auto-authenticated (no password prompt)                  │
│                                                                                   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. COMPONENTS — DETAILED

### 3.1 SAS (SHARED ACCESS SIGNATURES) / ACCESS / SECURITY — COMPLETE DEEP DIVE

**What is Shared Access Signature (SAS)?**
A Shared Access Signature is a secure, limited token that grants delegated access to Azure Storage resources without exposing account keys. SAS tokens can be scoped to specific containers/blobs, restricted to specific operations, and time-limited.

**SAS Types:**

```
Types of SAS:

  1. Service SAS (Recommended):
     → Access: Single storage account (blob, queue, table, file)
     → Scope: Container-level or Blob-level
     → Permissions: Read, Write, Delete, List, Tag, Move, Execute
     → Time-limited: Start time + Expiry time
     → IP restriction: Optional (source IP filter)
     → Protocol: HTTPS only (recommended)
     → Triggered by: Stored access policy or ad-hoc permissions
     → Most secure: Stored access policy (revocable)

  2. Account SAS:
     → Access: Entire storage account
     → Scope: Service-level (blob/queue/table/file) or Object-level
     → Permissions: Limited set (read, add, create, write, delete, list, tag, move, execute)
     → Time-limited: Start time + Expiry time
     → IP restriction: Optional
     → Triggered by: Stored access policy only (NOT ad-hoc for production)
     → Less common: Use Service SAS instead for specific access

  3. User Delegation SAS (NEW — Key Vault based):
     → Uses Azure AD (Entra ID) authentication (NOT account keys)
     → Requires: Storage account OAuth authorization (Microsoft.Authorization/storages)
     → Scope: Service-level or Object-level
     → Permissions: Read, Write, Delete, List, Tag, Move, Execute
     → Time-limited: Start time + Expiry time (no stored policy required)
     → IP restriction: Optional
     → Most secure: Revocable via Azure AD (revoke user access → SAS invalid)
     → Requires: Azure AD user/service principal with Storage RBAC role
     → Best practice: Use User Delegation SAS over Account SAS (Azure AD based)

  4. CORS SAS (Special):
     → Cross-Origin Resource Sharing for browser-based access
     → Used for: Browser apps accessing Blob Storage directly
     → Configured: CORS rules on storage service
     → Limited: GET, PUT, HEAD, POST only (no DELETE)
```

**Service SAS — Deep Dive:**

```
Service SAS Token Structure:

  https://storageaccount.blob.core.windows.net/container/blob?
    sas=
    sv=2022-11-02&                       → SAS version (storage API version)
    ss=b&srt=s&                          → Signed services (b=blob, s=service, c=container)
    sp=rl&                               → Signed permissions (r=read, l=list, w=write, d=delete, etc.)
    se=2024-12-31T23%3A59%3A59Z&        → Expiry time (ISO 8601, URL-encoded)
    st=2024-01-01T00%3A00%3A00Z&        → Start time (ISO 8601, URL-encoded)
    spr=https&                           → Allowed protocol (https only)
    sv=2022-11-02&                       → API version
    sr=c&                                → Signed resource type (c=container, b=blob, s=service)
    sig=SIGATUREHASH                     → Signature (HMAC-SHA256 hash of the SAS string)

  Components:
    → sv (version): Storage API version
    → ss (services): Which services the SAS applies to
    → srt (resource types): Service, Container, Object
    → sp (permissions): r=read, l=list, a=add, w=write, d=delete, c=create, t=tag, m=move, x=execute
    → se (expiry): When SAS expires (mandatory unless stored access policy)
    → st (start): When SAS becomes valid (optional)
    → spr (protocol): https or https,https (recommended: https only)
    → sr (resource): c=container, b=blob, s=service
    → sig (signature): HMAC-SHA256 hash (authentication)
    → ic (IP range): Source IP filter (optional)
    → rscc (cache control): Optional header
    → rscd (content disposition): Optional header
    → rsct (content type): Optional header
    → skoid, sktid, skt, skv, sks, ske (derived from stored access policy)

  Stored Access Policy (Recommended):
    → Defined ON the container (not in the SAS token)
    → Can be modified/deleted AFTER SAS token is issued
    → Revocation: Delete stored access policy → SAS token immediately invalid
    → Ad-hoc SAS: No stored policy → Cannot be revoked (must wait for expiry)
    → Best practice: ALWAYS use stored access policy for production SAS

  Stored Access Policy Example:
    → Container: /blob/mycontainer
    → Policy name: readpolicy
    → Permissions: read, list
    → Start: 2024-01-01
    → Expiry: 2024-06-30
    → IP Range: 10.0.0.0-10.0.0.255
    → Policy: Signed in Azure Portal or via SDK
    → SAS generated with policy: Includes skoid, sktid, skt, skv, sks, ske (no sig needed)
    → To revoke: Delete policy "readpolicy" → All SAS tokens using it become invalid
```

**SAS Permissions Matrix:**

| Permission | Blob | Container | Service | Description |
|-----------|------|-----------|---------|-------------|
| **r (Read)** | Read blob content/list blob | List container (blobs) | List service | Read access |
| **a (Add)** | Create blob (not applicable) | Create blob | Create queue/table | Add/create operation |
| **c (Create)** | Create blob | Create container | Create service | Create operation |
| **w (Write)** | Write blob content | Update container metadata | Update service | Write access |
| **d (Delete)** | Delete blob | Delete blob (soft delete if enabled) | Delete service | Delete access |
| **t (Tag)** | Update blob tags | N/A | N/A | Tag management |
| **m (Move)** | Move blob within account | Move blob between containers | N/A | Move operation |
| **x (Execute)** | Execute blob (ISO image) | N/A | N/A | Execute (for ISO VHD) |
| **l (List)** | List blobs in container | List containers | List service | List operation |

**User Delegation SAS (Azure AD-based — Best Practice):**

```
User Delegation SAS:

  Authentication: Azure AD (Entra ID) — NOT storage account keys
  → User/service principal authenticates with Entra ID
  → Gets token from Entra ID (OAuth 2.0)
  → Uses token to generate SAS token from Storage API
  → SAS token: Bound to Azure AD identity (not account key)

  Benefits over Account SAS:
    1. No account keys exposed (never shared)
    2. Revocable (remove Azure AD permission → SAS invalid)
    3. Auditable (Azure AD audit logs track SAS generation)
    4. Conditional Access applies (MFA, device compliance)
    5. No stored access policy needed (simpler)
    6. Supports all permissions (including move/delete)

  Requirements:
    → Azure AD user/service principal with Storage RBAC role
      (e.g., Storage Blob Data Reader, Storage Blob Delegator)
    → Storage account: "Allow Azure AD-based access" enabled
    → API version: 2022-11-02 or later
    → OAuth 2.0 token: https://storage.azure.com/.default scope

  Flow:
    1. App authenticates with Entra ID (interactive/service principal)
    2. Gets OAuth 2.0 token (scope: https://storage.azure.com/.default)
    3. Calls: POST https://<account>.blob.core.windows.net/?restype=service&comp=sas
    4. Request body: Permissions, expiry, protocol, etc.
    5. Storage API validates: Entra ID token → RBAC role → Authorization
    6. Returns: User delegation SAS token
    7. App uses SAS token to access storage (limited to permissions)

  RBAC Roles for User Delegation SAS:
    → Storage Blob Data Reader: Read/list blobs
    → Storage Blob Data Contributor: Read/write/delete blobs
    → Storage Blob Delegator: Generate SAS tokens (required)
    → Storage Queue Data Reader: Read/list queues
    → Storage Queue Data Message Processor: Send/receive messages
    → Storage File Data Reader: Read/list files
    → Storage File Data Contributor: Read/write/delete files

  L3 Critical: User Delegation SAS requires Azure AD authentication.
  If using Azure AD auth for storage: Remove account keys from storage account
  (set to disabled — recommended security practice).
```

**SAS Security Best Practices:**

```
1. NEVER use account keys in application code
   → Always use SAS tokens or User Delegation SAS
   → Account keys = full access to entire storage account
   → SAS = limited scope, time-limited, specific permissions

2. Use Service SAS over Account SAS
   → Service SAS: Container/blob level scope
   → Account SAS: Entire storage account (broader)
   → Principle: Least privilege (smallest scope possible)

3. ALWAYS use stored access policy (when possible)
   → Enables revocation (delete policy → SAS invalid)
   → Ad-hoc SAS: Cannot be revoked (must wait for expiry)
   → Exception: User Delegation SAS doesn't use stored policies (Azure AD revocable)

4. ALWAYS set expiry time (short expiry)
   → Production SAS: Hours/days (not years)
   → Dev/test SAS: Minutes/hours
   → If no expiry: SAS never expires (security risk)

5. ALWAYS restrict IP range (when possible)
   → Only allow specific IPs/subnets
   → Blocks: Access from unexpected locations
   → Especially important for: Internet-facing SAS URLs

6. ALWAYS use HTTPS protocol
   → spr=https (not http)
   → Prevents SAS token interception over network
   → Prevents: MITM, token theft

7. NEVER grant delete permission in SAS for production
   → Read-only SAS for public access
   → Write-only SAS for upload scenarios
   → Delete SAS: Only when explicitly required (with short expiry)

8. ALWAYS log SAS usage
   → Storage Analytics: Track SAS token usage
   → Azure Monitor: Alert on unusual SAS activity
   → Key Vault logging: Track SAS generation events

9. For cross-tenant access: Use Azure AD B2B + SAS
   → External user invited via Entra ID B2B
   → SAS token scoped to specific container
   → Time-limited (hours/days)
   → No account key sharing across tenants

10. For Service Bus SAS (different from Storage SAS):
    → Service Bus has its own SAS tokens (namespace/queue/topic level)
    → Managed identity preferred over SAS for Service Bus
    → SAS policies: RootManageSharedAccessPolicy (avoid using directly)
    → Create custom policies: Send, Listen, Manage (separate)
```

---

### 3.2 STORAGE DATA PROTECTION — COMPLETE DEEP DIVE

**What is Storage Data Protection?**
Storage Data Protection encompasses all mechanisms to ensure Azure Storage data is encrypted, recoverable, immutable when required, and access is controlled — protecting against accidental deletion, malicious attacks, ransomware, and compliance violations.

**Storage Encryption:**

```
Encryption at Rest:

  1. Service-Managed Encryption (SME / SSE):
     → Enabled by default (no action needed)
     → Azure encrypts data using 256-bit AES
     → Encryption keys managed by Microsoft
     → Keys: Microsoft-managed (automatic)
     → Scope: All storage accounts (Blob, File, Queue, Table)
     → Cost: No additional cost (included)

  2. Customer-Managed Key (CMK):
     → Customer provides encryption key from Azure Key Vault
     → Azure Storage uses KV key to encrypt/decrypt data
     → Key stored in: Key Vault (Soft Delete + Purge Protection REQUIRED)
     → Identity: Storage Managed Identity (system-assigned) has Key Vault access
     → Rotation: Customer rotates Key Vault key (Storage auto-uses new version)
     → Use: Compliance requirements, BYOK (Bring Your Own Key)
     → Cost: Key Vault pricing applies

  3. Customer-Managed Key + HSM (Double Encryption):
     → Key Vault with HSM-backed keys (FIPS 140-2 Level 3)
     → Additional layer: Microsoft-managed key (platform encryption)
     → Double encryption: Data encrypted with CMK + platform key
     → Required for: Highest compliance (DoD, PCI, etc.)
     → Cost: HSM Key Vault pricing + Storage Premium

  Encryption in Transit:
     → HTTPS only (TLS 1.2 minimum)
     → All storage API calls encrypted via TLS
     → SMB: AES-256 encryption (Azure Files)
     → NFS: TLS encryption (Azure Files NFS)
     → Enforce: "Secure transfer required" = Enabled (denies HTTP requests)

  Important: "Secure transfer required" = Enabled DENIES any HTTP (unencrypted)
  request to Storage. This is REQUIRED for compliance (PCI, HIPAA).
```

**Soft Delete:**

```
Blob Soft Delete:
  → Enables: Recovery of soft-deleted blobs for specified retention period
  → Retention: 1-365 days (default: 7 days)
  → How it works: Deleted blob → Marked as deleted (not actually removed)
  → Recovery: Undelete operation within retention period
  → After retention: Blob permanently deleted (IRREVERSIBLE)
  → Important: Soft Delete protects against ACCIDENTAL deletion
  → Does NOT protect against: Malicious deletion (someone with delete permission)
    → Use: Immutable Blob Storage for ransomware/malicious protection

  Blob Container Soft Delete:
  → Enables: Recovery of soft-deleted containers
  → Retention: 7-365 days
  → Protects: Entire container (all blobs within) from accidental deletion

  Versioning + Soft Delete:
  → Versioning: Keeps previous versions of blobs
  → Soft Delete: Recovers deleted blobs
  → Combined: Protection against both modification and deletion
  → Configuration: Both enabled on storage account (independently)

  Table/Queue Soft Delete:
  → Enables: Recovery of soft-deleted tables/queues
  → Retention: 7-365 days
  → Note: Different from blob soft delete (separate setting)
```

**Immutable Blob Storage (WORM — Write Once, Read Many):**

```
Immutable Blob Storage:
  → Data cannot be modified or deleted for specified duration
  → Two modes: Time-based and Legal Hold

  1. Time-Based Immutability:
     → Blob locked for: 1 day to 365 days
     → After period: Blob can be modified/deleted
     → Configuration: Container-level (applies to all blobs in container)
     → Cannot be disabled: Once set, period is locked (even by admin)
     → Use: Compliance (financial records, healthcare data, audit logs)
     → Note: Policy can be shortened (but NOT extended during lock period)

  2. Legal Hold:
     → Blob locked indefinitely (no expiry)
     → Until explicitly removed (by authorized user)
     → No time limit (manual release required)
     → Use: Active legal matters (litigation hold)
     → Requires: Legal department to initiate hold
     → Release: Remove legal hold (authorized personnel) → Blob modifiable

  Immutability + Versioning:
     → Versions are also immutable (if policy is immutable)
     → Previous versions: Protected during immutability period
     → After period expires: Previous versions modifiable/deletable

  Immutability + Soft Delete:
     → Soft delete still works (for recovery after immutability expires)
     → During immutability: Cannot delete (even soft delete blocked)
     → After immutability: Soft delete protects recovered blobs

  Configuration:
     → Storage Account → Data Protection → Immutability Policy
     → Container → Immutability → Time-based (1-365 days) or Legal Hold
     → Enforcement: Cannot be disabled (can only be shortened/released)
     → Audit: All attempts to modify/delete immutable blobs logged

  Important: Immutability policy CANNOT be disabled once set.
  → Time-based: Policy duration can be shortened (but NOT extended)
  → Legal Hold: Cannot be removed until legal team releases
  → This is by design: Prevents an attacker (even admin) from bypassing immutability
```

**Versioning:**

```
Blob Versioning:
  → Enables: Automatic versioning of blob when modified
  → How it works: Each modification creates new version (immutable)
  → Example: blob.txt v1 (upload) → v2 (modify) → v3 (modify)
  → Current version: v3 (what users see by default)
  → Previous versions: v1, v2 (accessible if version ID specified)
  → Configuration: Blob Service → Data Protection → Versioning = Enabled

  Benefits:
    → Accidental overwrite recovery (restore previous version)
    → Data lineage (track changes over time)
    → Combined with Soft Delete: Double protection
    → Compliance: Complete audit trail of blob changes

  Considerations:
    → Storage cost: Each version = additional storage (costs money!)
    → Lifecycle: Auto-tier/expire old versions (save cost)
    → Limit: Up to 50 versions per blob (configurable, unlimited with lifecycle)
    → Performance: Minimal impact (async version creation)

  Version Lifecycle Management:
    → Policy: Move versions older than 30 days → Cool tier
    → Policy: Delete versions older than 90 days
    → Policy: Keep last 5 versions always
    → Policy: Move current version to Archive after 365 days of no change
```

**Object Lifecycle Management (OLM):**

```
OLM Policies:
  → Automatically transition blob tiers based on age or conditions
  → Actions: Tier change, Delete, Copy, Archive
  → Scope: Container-level or prefix-level (by blob name prefix)

  Common Policies:
    → Hot → Cool after 30 days (infrequently accessed)
    → Cool → Archive after 90 days (rarely accessed)
    → Archive → Delete after 365 days (compliance expired)
    → Cool → Delete after 365 days (temporary data)

  Policy Configuration:
    → Storage Account → Data Management → Lifecycle Management
    → Define: Rules (filter + actions)
    → Enable: Time-based or modified-date-based

  Cost Impact:
    → Tier transition: Small per-GB cost (tier change fee)
    → Archive retrieval: Significant cost (rehydrate from Archive)
    → Delete: Free (no cost)
    → Best practice: Analyze access patterns before setting policies
    → Mistake: Archive tier + frequent access = High retrieval costs

  OLM + Immutability:
    → OLM policies respect immutability (cannot delete/modify during lock)
    → After immutability expires: OLM transitions/deletes blobs
    → Combined: Compliance protection + Cost optimization
```

**Storage Firewall + Private Endpoint:**

```
Storage Firewall:
  → Controls: Which networks can access storage account
  → Options:
    1. Public access: All networks (NOT recommended for production)
    2. Selected networks: Virtual Network + IP whitelist only
    3. Private endpoint only: No public access, only Private Endpoint

  Configuration:
    → Storage Account → Networking → Firewalls and virtual networks
    → Options:
      → Trust trusted Microsoft services: Bypass firewall for Azure services
      → Virtual Networks: Add VNet + Subnet (or VNet with private endpoints)
      → IP Addresses: Add client/public IP whitelist

  IP Whitelist:
    → Client IP: Your corporate/public IP
    → Service endpoints: Azure services (SQL, etc.)
    → Limitation: Max 200 IP addresses per storage account

  Private Endpoint:
    → Creates: Private IP in VNet for storage account
    → Access: VNet resources connect via private IP (no internet)
    → DNS: Private DNS Zone → Resolves storage name to private IP
    → NOT on firewall: Private Endpoint bypasses firewall (direct access)

  Bypass:
    → Bypass "AzureServices" (trusted Microsoft services)
    → Allows: Azure SQL, Azure Data Factory, etc. to access storage
    → WITHOUT bypass: Azure SQL → Public endpoint (firewall blocks) → Fails
    → WITH bypass: Azure SQL → Internal (trusted) → Works

  Important: For production, use "Selected networks" + Private Endpoint
  + Bypass "AzureServices" (for PaaS services to access storage privately)
```

**Azure Defender for Storage:**

```
Defender for Storage (part of Microsoft Defender for Cloud):
  → Threat detection for storage accounts
  → Detects: Unusual access patterns, potential data exfiltration
  → Alerts: Anomalies in authentication, suspicious IP activity
  → Protection: RAID-6 (8 copies of data in LRS/ZRS storage)
  → Cost: ~$0.10/GB/month per storage account (Defender)
  → Integration: Microsoft Sentinel (SIEM/SOAR)

  Threat Detection Scenarios:
    → Unusual principal: Different IP/user accessing storage
    → Anomalous download: Large data download from unexpected IP
    → Failed auth spike: Many auth failures (brute force)
    → Suspicious activity: Deleting many blobs rapidly (ransomware-like)

  Defender + Private Endpoint:
    → Combined: Maximum protection (no public access + threat detection)
    → Defender also protects: Blob, Queue, Table (all storage services)
```

**Storage Data Protection Checklist:**

| Check | Setting | Verification |
|-------|---------|-------------|
| Encryption at rest | Enabled (SME or CMK) | Portal: Storage Account → Encryption |
| Secure transfer required | Enabled (HTTPS only) | Portal: Storage Account → Configuration |
| CMK (Customer Managed Key) | Key Vault with Soft Delete + Purge | Portal: Storage Account → Encryption → Customer-managed key |
| Soft Delete (Blob) | Enabled (1-365 days) | Portal: Storage Account → Data Protection → Blob Soft Delete |
| Soft Delete (Container) | Enabled | Portal: Storage Account → Data Protection → Container Soft Delete |
| Soft Delete (Table/Queue) | Enabled | Portal: Storage Account → Data Protection |
| Immutable Blob Storage | Time-based or Legal Hold (where required) | Portal: Container → Immutability Policy |
| Versioning | Enabled | Portal: Storage Account → Data Protection → Blob Versioning |
| Firewall | Selected Networks (not Public) | Portal: Storage Account → Networking |
| Private Endpoint | Configured (no public access) | Portal: Private Endpoint → Status |
| Azure Defender | Enabled | Portal: Defender for Storage → On |
| SAS tokens | Scoped, time-limited, HTTPS only | Audit: SAS usage in Storage Analytics |
| Lifecycle Management | Policies for tiering/deletion | Portal: Storage Account → Data Management |
| Access Control | RBAC (not account keys) | Portal: IAM → Check for key-based access |
| Audit Logs | All access logged | Portal: Diagnostic Settings → Log Analytics |

---

### 3.3 MANAGED IDENTITIES — DEEP DIVE

**What is a Managed Identity?**
A Managed Identity is an Azure resource that provides an identity for Azure services to authenticate to other Azure services (like Key Vault, Storage, SQL, etc.) WITHOUT requiring credentials in code. The identity is managed by Azure (no secrets to rotate, no passwords to store).

**Types of Managed Identities:**

```
1. System-Assigned Managed Identity:
   → Tied to: Specific Azure resource (VM, App Service, Function, etc.)
   → Lifecycle: Created with resource, DELETED when resource is deleted
   → Uniqueness: One per Azure resource (cannot share)
   → Use: Single resource accessing specific services (VM → Key Vault)
   → Automatic: No need to register in Entra ID (created automatically)
   → Identity in Entra ID:
      → Tenant ID: Same as resource tenant
      → Principal ID: Unique ID (like a user/SP in Entra ID)
      → Object ID: Same as Principal ID

   Lifecycle:
     VM created → System MI created (automatically)
     VM deleted → System MI deleted (automatically, with resource)
     MI re-created: If VM redeployed, new MI created (different Principal ID)

   Authentication Flow:
     App on VM → Requests token from IMDS (169.254.169.254)
     → IMDS returns: Access token for target resource
     → App uses token: Authorization: Bearer <token>
     → Target service: Validates token with Entra ID
     → Access granted: Based on RBAC role assigned to MI

   IMDS (Instance Metadata Service):
     → URL: http://169.254.169.254/metadata/identity/oauth2/token
     → Parameters: api-version=2018-02-01, resource=https://storage.azure.com
     → Returns: Access token (scoped to specified resource)
     → Important: IMDS is for Azure resources (VMs, VMs in scale sets)
     → IMDS is NOT accessible from outside Azure (only from within Azure)
     → Security: NSG rules can block IMDS (169.254.169.254)

2. User-Assigned Managed Identity:
   → Tied to: Standalone Azure resource (not tied to specific resource)
   → Lifecycle: Independent (exists without any other resource)
   → Reusability: Can be assigned to MULTIPLE resources (1-1000)
   → Use: Multiple resources sharing same identity (100 VMs → same MI)
   → Cost: Separate Azure resource (billed independently)
   → Creation: Create MI first, then assign to resources

   Key Differences from System-Assigned:
     → Not deleted when resource is deleted (independent)
     → Can be assigned to multiple resources
     → Can be granted roles independently
     → Can be used across resource groups/subscriptions (with proper RBAC)
     → Pre-provisioned: Create before deployment, attach during deployment
     → Faster: No waiting for MI creation during resource deployment

   Identity in Entra ID:
     → Has: Application ID (Client ID), Principal ID, Tenant ID
     → Managed as: Entra ID application (App Registration)
     → Visible in: Entra ID → App Registrations

   Authentication Flow:
     App on VM → Requests token from IMDS (169.254.169.254)
     → IMDS returns: Access token
     → Same as System-Assigned (uses IMDS)

3. Identity Flows for Different Azure Services:

   VM Scale Set / VM:
     → IMDS: http://169.254.169.254 (Instance Metadata Service)
     → Token scope: https://storage.azure.com, https://vault.azure.net, etc.
     → Configuration: VM → Identity → System Assigned or User Assigned
     → Access: Azure portal → VM → Identity → Get Token

   App Service / Function:
     → Managed Identity endpoint:
       https://<app-name>.azurewebsites.net/.auth/me (token endpoint)
     → Also: IMDS (169.254.169.254) available
     → Configuration: App Service → Identity → Managed Identity
     → Token acquisition:
       → SDK: Azure SDK uses Managed Identity credentials automatically
       → REST: Call IMDS endpoint from code
       → Token cache: SDK caches tokens (refreshes automatically)

   AKS Pod (Workload Identity):
     → Azure Pod Identity (deprecated) → Workload Identity (current)
     → Flow: Pod requests token from AKS OIDC issuer
     → AKS OIDC issuer: https://oidc.azure.com/<cluster-id>
     → Token: Signed by AKS, validated by Entra ID
     → Service: Uses Entra ID federation (Workload Identity)
     → RBAC: Namespace-level or cluster-level (Azure RBAC for K8s)

   Azure Functions:
     → Same as App Service (Managed Identity endpoint)
     → System-Assigned: Created with Function App
     → User-Assigned: Pre-created, attached to Function App
     → Configuration: Function App → Identity → User Assigned

   Azure Data Factory / Synapse Pipeline:
     → System MI or User MI for pipeline execution
     → MI authenticates to: Linked services (Storage, SQL, etc.)
     → No connection strings with passwords (uses MI RBAC)

   Azure SDK (Automatic):
     → Azure SDK libraries: Auto-detect Managed Identity
     → No code changes needed for MI vs Key Vault secret
     → Example (Python):
       from azure.storage.blob import BlobServiceClient
       # No credentials in code — SDK auto-uses Managed Identity
       client = BlobServiceClient(account_url="https://storage.blob.core.windows.net")
       # SDK: Tries MI → Gets token from IMDS → Uses token for auth

     Example (C#):
       var client = new BlobServiceClient(
           new Uri("https://storage.blob.core.windows.net"),
           new ManagedIdentityCredential());
       // Explicitly use Managed Identity credential

     Example (Azure CLI):
       az storage blob list --account-name mystorage --auth-mode login
       // --auth-mode login: Uses Azure AD (MI or user) instead of account key
```

**Managed Identity — Cross-Tenant Identity:**

```
Cross-Tenant Identity Scenarios:

  1. Multi-Tenant SaaS Application:
     → Tenant A: Users authenticate via App Registration (A)
     → Tenant B: Users authenticate via App Registration (A) or B2B
     → App Registration: Multi-tenant (accessTokenAcceptedForTenantIds = all)
     → MI: Single identity accessing resources in Tenant A
     → Cross-tenant access: B2B invitation (different mechanism)

  2. Identity Provider Federation:
     → Entra ID: Primary identity provider
     → On-prem AD: Federated (ADFS or Entra Connect)
     → Users: Sync from on-prem → Entra ID (cloud)
     → MI: Same identity in cloud (sync'd user = MI credential)

  3. Service-to-Service Cross-Tenant:
     → Service A (Tenant A) → Needs to call Service B (Tenant B)
     → Options:
       a. App Registration: Service A uses MI with cross-tenant access
       b. B2B: Service A invites Service B (via Entra ID B2B)
       c. SAS token: Service B generates SAS for Service A
     → Best: User Delegation SAS or Azure AD cross-tenant app registration

  4. MI in Different Subscriptions:
     → User MI: Can be assigned to resources in different subscriptions
     → System MI: Tied to resource (can't be in different subscription)
     → RBAC: Grant MI access to target resource (in different subscription)
     → Process:
       1. Create User MI in Subscription A
       2. Assign MI role in Subscription B (on target resource)
       3. VM in Subscription A uses MI → Access to resource in Subscription B

  5. Container Registry Access (Cross-Tenant):
     → ACR in Tenant A
     → AKS in Tenant B
     → Options:
       a. ACR in Tenant B (replicate images)
       b. Entra ID B2B (B2B access to ACR)
       c. SAS token (ACR generates SAS for cross-tenant pull)
     → Best practice: ACR in same tenant (replicate for cross-tenant)

Important: System MI and User MI both use IMDS (169.254.169.254) for token
acquisition from Azure VMs. The difference is identity lifecycle and reusability,
not the authentication mechanism.
```

---

### 3.4 AZURE KEY VAULT — COMPLETE DEEP DIVE

**What is Azure Key Vault?**
Azure Key Vault is a cloud-based key management service that stores encryption keys, secrets (passwords, connection strings, API keys), and certificates. It provides secure storage, access control, auditing, and lifecycle management for all cryptographic assets.

**Key Vault Components:**

```
Azure Key Vault: "kv-contoso-prod-eastus"
  ├── Vault URI: https://kv-contoso-prod-eastus.vault.azure.net
  ├── Resource Group: rg-contoso-prod-eastus
  ├── Subscription: SUB-ID-12345
  ├── Tenant: globalcontoso.onmicrosoft.com
  │
  ├── Secrets:
  │   → storage-connection-string: "DefaultEndpointsProtocol=https;..."
  │   → sql-admin-password: "P@ssw0rd123!"
  │   → api-key: "abc123xyz789"
  │   → Each secret: Value up to 25 KB
  │   → Versioning: Each set creates new version (previous preserved)
  │   → Version count: Unlimited (practical limit: storage)
  │   → Activation: Dates, Email addresses
  │
  ├── Keys:
  │   → rsa-key-1: RSA 2048/4096 bit
  │   → rsa-key-2: RSA (Key Vault KMS — used for CMK)
  │   → ec-key-1: ECC (P-256, P-384, P-521)
  │   → Key types: RSA, EC, octagonal (symmetric)
  │   → Operations: Encrypt, Decrypt, Sign, Verify, Wrap, Unwrap
  │   → Key rotation: Configurable (auto-rotate with policy)
  │   → Key rotation notification: Event Grid (when key rotates)
  │
  ├── Certificates:
  │   → app-cert-contoso: SSL/TLS certificate
  │   → ca-cert-contoso: Certificate Authority (not signed)
  │   → Generated: Self-signed or from CA (DigiCert, GlobalSign, etc.)
  │   → Auto-renewal: Configurable (90 days before expiry)
  │   → Auto-rotation: Configurable (with CA integration)
  │   → Download: Not available (only for CA-issued certs)
  │   → Policy: Renewal + Rotation (key attestation)
  │
  ├── Storage Accounts (HMS - Hardware Security Module):
  │   → Storage account + Identity = storage account identity
  │   → Used: HSM (BYOK - Bring Your Own Key) for 2023+
  │   → RBAC: Replaces access policies (v2)
  │
  ├── Access Policies (v1) / RBAC (v2):
  │   → v1: Access Policies (deprecated)
  │     - Up to 1024 access policies
  │     - Per principal: Secret/Key/Certificate permissions
  │     - Template: Secret Management, Key Management, Certificate Management
  │     - Replacement: Move to RBAC (v2)
  │
  │   → v2: RBAC (recommended)
  │     - Built-in roles: Key Vault Administrator, Key Vault Secrets Officer,
  │       Key Vault Certificates Officer, Key Vault Crypto Officer,
  │       Key Vault Crypto User, Key Vault Reader
  │     - Custom roles: Granular (specific operations like "Microsoft.KeyVault/vaults/secrets/read")
  │     - Scope: Vault-level, Resource Group-level, Subscription-level
  │     - No access policies needed (RBAC only)
  │
  ├── Network:
  │   → Firewall + Private Endpoint (no public access for production)
  │   → Selected Networks: VNet + IP whitelist
  │   → Bypass: AzureServices (for trusted Microsoft services)
  │   → Private Endpoint: VNet integration (private IP)
  │   → DNS: Private DNS Zone (privatelink.vaultcore.azure.net)
  │
  └── Soft Delete + Purge Protection:
      → Soft Delete: 7-90 days (recover deleted KV)
      → Purge Protection: Indefinite (cannot purge until soft delete period ends)
      → Recovery: Vault + Resources recoverable during soft delete period
      → NOT recoverable: After soft delete + purge period → DATA LOST FOREVER
```

**Key Vault — RBAC vs Access Policies:**

```
Access Policies (v1) — Legacy:
  → Per principal (User/App): Secret/Key/Cert permissions
  → 1024 policy limit per vault
  → Template-based (Secret Management, Key Management, etc.)
  → Cannot use RBAC with access policies simultaneously
  → Deprecation: New vaults default to v2 (RBAC only)
  → Existing: Can still use, but Microsoft recommends migration to v2

RBAC (v2) — Recommended:
  → Azure RBAC roles for Key Vault:
    - Key Vault Administrator: Full control (create, delete, update, get, list, set policies)
    - Key Vault Secrets Officer: Secret operations (get, list, set, delete, backup, restore)
    - Key Vault Secrets User: Secret read (get, list, get-attributes)
    - Key Vault Certificates Officer: Certificate operations
    - Key Vault Certificates User: Certificate read
    - Key Vault Crypto Officer: Encryption/decryption operations
    - Key Vault Crypto User: Encryption/decryption (keys only)
    - Key Vault Reader: Read-only (get/list all properties)
    - Key Vault Network Contributor: Network settings management
    - Key Vault Purge Protection Contributor: Purge protection management
    - Key Vault Soft Delete Contributor: Soft delete management

  → Advantages over v1:
    - No principal limit (unlimited)
    - RBAC standard (consistent with rest of Azure)
    - Supports groups, roles, and conditional access
    - No template-based (full permissions per role)
    - Can be combined with Access Policies (transition period)

  → Assignment Scope:
    - Vault-level: All operations on this vault
    - Resource Group-level: All vaults in RG
    - Subscription-level: All vaults in subscription

Migration (v1 → v2):
  → Azure Portal: Key Vault → Configuration → Configuration
  → Switch: Access Policies to RBAC
  → Before switching: Grant RBAC roles to all principals that had access policies
  → After switching: Access policies removed (RBAC only)
  → Warning: Test in non-production first (break access if misconfigured)
```

**Key Vault — Network Security Deep Dive:**

```
Key Vault Network Configuration:

  1. Public Access (Default):
     → Accessible from internet
     → URL: https://kv-contoso.vault.azure.net
     → Risk: Anyone with token can access (if RBAC permits)
     → NOT recommended for production

  2. Selected Networks:
     → Virtual Networks:
       - Add: VNet + Subnet (for Private Endpoint)
       - OR: VNet (all subnets in VNet)
       - For App Service/AKS: Regional VNet Integration
     → IP Addresses:
       - Client IPs (your corporate/public IP)
       - Service endpoints: Azure Services (SQL, etc.)
     → Bypass AzureServices: Allow trusted Microsoft services
     → Combined: VNet + IP + Bypass + Private Endpoint

  3. Private Endpoint Only:
     → No public access (firewall blocks all)
     → Only Private Endpoint connections allowed
     → DNS: Must resolve to private IP (Private DNS Zone)
     → Most secure: Completely isolated from internet

  Key Vault DNS Resolution:
     → Public DNS: kv-contoso.vault.azure.net → Public IP
     → Private DNS: privatelink.vaultcore.azure.net → Private IP
     → Private DNS Zone: privatelink.vaultcore.azure.net linked to VNet
     → Resolution: VM in VNet → DNS query → Private DNS Zone → Private IP
     → Access: VM → Private IP → Key Vault (no internet)

  Virtual Network for Key Vault:
     → Regional VNet Integration (Preview/GA):
       - App Service, AKS, Function: Access KV via VNet
       - No Private Endpoint needed (VNet integration handles it)
       - Requires: Regional VNet Link (key vault → VNet)
     → For VMs: Private Endpoint or IP whitelist
     → For on-prem: VPN/ExpressRoute + Private Endpoint

  Implementation:
     Key Vault → Networking → Firewalls and Virtual Networks:
       → Selected Networks:
         - VNet: vnet-hub-prod (all subnets)
         - IP Addresses: 203.0.113.0/24 (corporate VPN range)
         - Bypass: AzureServices
       → Private Endpoint:
         - VNet: vnet-spoke-data (10.0.3.0/24)
         - Private IP: 10.0.3.4
         - Private DNS Zone: privatelink.vaultcore.azure.net (linked)
```

**Key Vault — Certificate Auto-Renewal:**

```
Certificate Auto-Renewal:
  → CA-issued certificates: Auto-renewal supported
  → Self-signed certificates: Auto-renewal supported (Key Vault generates new)
  → CA integration: DigiCert, GlobalSign, Entrust, etc.
  → How it works:
    1. Certificate created with policy (expiry: 1 year)
    2. Key Vault contacts CA 90 days before expiry
    3. CA issues new certificate
    4. Key Vault stores new version (old version remains)
    5. Event Grid notification: Certificate renewed
    6. Downstream services: Get latest version automatically

  Rotation Policy:
    → Configurable: 30-90 days before expiry
    → Event Grid: Trigger on certificate renewal
    → Automation: Event Grid → Function → Deploy new certificate
    → Key Vault auto-rotation: Key Vault rotates key automatically

  Certificate Versions:
    → Multiple versions: Each renewal creates new version
    → Current version: Enabled = true (latest)
    → Previous versions: Disabled (still retrievable if needed)
    → auto-apply-policy: Set which version is active

  Certificate Operations:
    → Create: Self-signed, Import (existing), Generate via CA
    → Export: Not available (security; cannot export private key)
    → Download: Available for CA-issued cert (only public cert)
    → Delete: Permanent delete (irreversible if purge protection enabled)
    → Re-import: Import new version of same certificate
    → Contacts: Email contacts for renewal (CA sends email)
```

**Key Vault — Event Grid Integration:**

```
Key Vault Event Grid Integration:
  → Events: Certificate created, renewed, expired, deleted; Key rotated; Secret changed
  → Event Grid: Triggers on Vault events (not individual key/secret events)
  → Integration: Key Vault → Event Grid → Subscription (Topic)
  → Consumer: Azure Function, Logic Apps, Automation Runbook
  → Use cases:
    → Certificate renewal → Deploy to App Service automatically
    → Key rotation → Update CMK reference in Storage/ADLS
    → Secret change → Restart App Service (re-read secret)
    → Deletion alert → Security team notified

  Event Types:
    → Microsoft.KeyVault.CertificateNewVersionCreated
    → Microsoft.KeyVault.CertificateApproachingExpiration
    → Microsoft.KeyVault.CertificateExpired
    → Microsoft.KeyVault.KeyRotationCompleted
    → Microsoft.KeyVault.SecretChanged
    → Microsoft.KeyVault.VaultUpdated
```

**Key Vault — Access Audit:**

```
Key Vault Audit Logging:
  → Diagnostic Settings → Log Analytics:
    → AuditEvent: All operations on Key Vault (who, what, when, where)
    → PrivateLinkMatch: Private Link matching
    → PrivateLinkStatus: Private Link status changes
    → HTTPServerLogs: HTTP request logs (status codes, latency)

  Example Audit Log:
    → OperationName: Get Secret
    → Caller: app-prod-api (Managed Identity)
    → Principal ID: <MI-ID>
    → Secret Name: sql-connection-string
    → Vault Name: kv-contoso-prod-eastus
    → Timestamp: 2024-10-15T09:30:00Z
    → Status: 200 (Success)
    → Correlation ID: <UUID>
    → Request ID: <UUID>
    → Client IP: 169.254.169.254 (IMDS — from VM)

  Alerting:
    → Alert 1: Failed access attempts (audit logs with 401/403 status)
    → Alert 2: Secret read outside business hours (off-hours access)
    → Alert 3: High volume of secret reads (data exfiltration indicator)
    → Alert 4: Access from unexpected IP (if IP-restricted KV)
    → Alert 5: Admin operations (change access policy, delete vault)
```

**Key Vault — Key Rotation with Storage:**

```
CMK Rotation with Azure Storage:
  1. Storage Account: Encryption = Customer Managed Key (CMK)
  2. Key Vault: Contains encryption key (RSA or EC)
  3. Rotation:
     a. Key Vault: Create new key version (rotation policy triggers)
     b. Storage: Auto-detects new key version (uses latest)
     c. Decryption: Storage tries all key versions (old data still decryptable)
     d. Old key version: Retired (kept but not used for new encryption)
     e. Data encrypted with old version: Still decryptable (multi-key support)

  4. Rotation Policy:
     → Expire key: 1 year
     → Rotate before: 90 days (configure in Key Vault)
     → Rotate: Auto-generates new version (same key, new version)
     → Notification: Event Grid (key rotation completed)
     → Storage: Automatically uses new version (no config change needed)

  5. Key Rotation Best Practices:
     → Rotation: Every 90 days (or per compliance requirement)
     → Backup: Export public key (not private key!) for disaster recovery
     → HSM: Use HSM-backed keys for highest security
     → Multi-key: Support for data encrypted with previous keys
     → Re-wrap: Decrypt with old key + re-encrypt with new key (optional)
```

---

### 3.5 APP REGISTRATIONS — COMPLETE DEEP DIVE

**What is an App Registration?**
An App Registration (Application Object) in Entra ID defines an application's identity — its permissions, authentication methods, and which Azure AD tenants can access it. It is the "blueprint" or "definition" of an application.

**App Registration vs Service Principal:**

```
App Registration (Application Object):
  → What it IS: Application definition (blueprint)
  → Where: Entra ID → App Registrations
  → Contains:
    - Application ID (Client ID) — unique identifier
    - Supported account types (Single tenant vs Multi-tenant)
    - Redirect URIs (where auth responses are sent)
    - API permissions (what the app requests)
    - Manifest (detailed configuration)
    - Certificates & secrets (authentication credentials)

  Service Principal (Service Principal Object):
  → What it IS: Instance of an app (who the app IS in a specific tenant)
  → Where: Entra ID → Enterprise Applications (per tenant)
  → Contains:
    - Service Principal ID (Object ID) — different from App ID
    - Account enabled/disabled
    - Assigned RBAC roles
    - Federated credentials
    - App role assignments

  Relationship:
    1 App Registration → Multiple Service Principals (one per tenant)
    → Single-tenant app: 1 App Registration → 1 Service Principal (same tenant)
    → Multi-tenant app: 1 App Registration → N Service Principals (one per tenant)

  Example:
    App Registration: "Contoso Payroll App"
      Application ID: 12345-ABC-DEF
      → Tenant A (globalcontoso.onmicrosoft.com):
        Service Principal: Object ID = sp-a1b2c3 (Enabled, RBAC assigned)
      → Tenant B (client-contoso.onmicrosoft.com):
        Service Principal: Object ID = sp-d4e5f6 (Enabled, RBAC assigned)
      → Tenant C (disabled tenant):
        Service Principal: None (user consent required for B2B)

  L3 Critical: People often confuse App Registration with Service Principal.
  When you create an app in Entra ID, you create BOTH:
    1. App Registration (definition, in App Registrations blade)
    2. Service Principal (instance, in Enterprise Applications blade)
  They are different objects with different IDs (Application ID vs Object ID).

  When you delete App Registration:
    → Service Principals across all tenants are cleaned up
    → If SP is stuck (orphaned): Enterprise Applications → Delete manually
    → Orphaned SP: Not visible in App Registrations but visible in Enterprise Apps
```

**App Registration — Authentication:**

```
Authentication Methods:

  1. Client Credentials (Service to Service):
     → Type: Client Secret or Certificate
     → Flow:
       1. App calls Entra ID: POST /token (grant_type=client_credentials)
       2. Authenticates with: Client ID + Secret/Certificate
       3. Returns: Access token (for target resource)
       4. App uses token: Authorization: Bearer <token>
     → Use: Daemon apps, background services, CI/CD bots
     → NOT suitable: User-interactive apps

  2. Authorization Code (Web App):
     → Type: Interactive (user login)
     → Flow:
       1. User redirected to Entra ID login
       2. User authenticates (MFA if Conditional Access)
       3. Entra ID returns authorization code to redirect URI
       4. App exchanges code for access token
       5. App uses token to call API (on behalf of user)
     → Use: Web apps with user authentication
     → Requires: Redirect URI (registered in App Registration)

  3. Implicit Grant (Deprecated):
     → Returns: Token directly (no code exchange)
     → SECURITY ISSUE: Token in URL (leak risk via browser history, referer)
     → Deprecated: Microsoft recommends Auth Code + PKCE instead

  4. Device Code Flow (Smart TVs, CLI tools):
     → Flow:
       1. App calls Device Code endpoint
       2. User gets code (displayed on device)
       3. User goes to https://microsoft.com/devicelogin on phone/PC
       4. User enters code
       5. App polls for token (until user authenticates)
     → Use: Smart TVs, SSH tools, CLI applications
     → User-friendly: No typing on constrained device

  5. Integrated Windows Authentication (IWA):
     → For: On-prem Windows apps accessing cloud
     → Flow: Kerberos/NTLM → Entra ID token (via Entra Connect sync)
     → Seamless: No login prompt (uses current Windows credentials)
     → Requires: Entra Connect sync (on-prem AD → Entra ID)

  6. OAuth 2.0 On-Behalf-Of (OBO):
     → For: Middle-tier service calling downstream API as user
     → Flow:
       1. User → Web App (auth code)
       2. Web App → API (client credentials + token)
       3. API → Entra ID (OBO: exchange token for user token)
       4. API → Downstream API (user token, delegated permissions)
     → Use: Multi-tier apps (web app → API → another API)

  7. PKCE (Proof Key for Code Exchange):
     → For: Public clients (mobile, SPA — cannot keep secret)
     → Enhances: Authorization Code flow with code verifier/challenge
     → Prevents: Authorization code interception attacks
     → Use: Mobile apps, SPA (Single Page Applications), desktop apps
     → Security: Code challenge = SHA-256(code verifier)

  8. Certificate (X.509):
     → For: Service-to-service auth (more secure than client secret)
     → Flow:
       1. App signs JWT assertion with certificate private key
       2. App sends assertion to Entra ID /token endpoint
       3. Entra ID validates certificate (thumbprint match)
       4. Returns: Access token
     → Use: Production S2S (cert more secure than secret)
     → Rotation: Auto-renewal via Key Vault certificate policy

  9. Federated Credentials (New — No Secret/Cert):
     → For: Workload identity (GitHub Actions, Kubernetes, CI/CD)
     → Flow:
       1. External identity provider (GitHub, Kubernetes, etc.)
       2. Issues: Subject token (JWT signed by GitHub/K8s)
       3. App sends: Subject token + Audience (Entra ID app ID)
       4. Entra ID: Validates with issuer (GitHub/K8s OIDC)
       5. Returns: Access token
     → Use: CI/CD without secrets, Kubernetes workloads
     → Providers: GitHub Actions, Kubernetes, Docker Hub, etc.
     → Security: No secrets stored in app registration
```

**App Registration — API Permissions:**

```
API Permissions:
  → Define: What Azure/Microsoft APIs the app needs access to
  → Types:
    1. Microsoft Graph (most common):
       → Delegated: On behalf of signed-in user
         - User.Read: Read user profile
         - Mail.Read: Read user's mail
         - Files.Read: Read user's files
       → Application: App runs without user (daemon/service)
         - User.Read.All: Read all users
         - Mail.Read: Read all mail
         - Directory.Read.All: Read all directory data

    2. Azure Resources:
       → Storage: user_impersonation (delegated), AzureStorage (application)
       → Key Vault: user_impersonation, https://vault.azure.net/.default
       → SQL: user_impersonation, AzureSQLDataManagement
       → Service Bus: user_impersonation, AzureServiceBus

    3. Other APIs (custom):
       → scope-based (OAuth 2.0 scopes)
       → OpenID Connect: openid, profile, email
       → Resource-specific: API-level scopes

  Permission States:
    → Granted: Consent given (user or admin)
    → Not Granted: Requires consent
    → Privileged: Requires admin consent (high-privilege permissions)

  Consent Types:
    → User Consent: User grants for their own data
      - Limited: Only delegated permissions (not application)
      - Risk: Users can grant broad access (if not restricted)
      → Admin Consent: Tenant admin grants for everyone
      - Broad: Can grant application permissions
      - Security: Use Conditional Access to restrict
      - Best: Require admin consent (prevent user over-granting)

  Consent Framework (newer — additional security):
    → Requires admin approval for sensitive permissions
    → Request: App requests permission → Admin reviews → Approve/Deny
    → Restrictions: Per-app, per-tenant (admin configurable)
    → Grant: Admin consent via portal or Graph API
    → Grant (build-up): Progressive consent (build-up over time)

  Refresh Tokens:
    → Delegated permissions: Refresh token (long-lived) for access token renewal
    → Application permissions: No refresh token (client_credentials, no session)
    → Session timeouts: Refresh token expires (configurable: 1-90 days)
    → Security: Revoke refresh tokens (sign-in/logytics)
```

**App Registration — Manifest:**

```
App Manifest (JSON):
  → Accessible: Portal → App Registration → Manifest
  → Key Properties:
    - appId: Application ID (Client ID)
    - appRoles: App-specific roles (authorization)
    - oauth2RequiredPostResponse: Post-response required (OAuth 2.0)
    - requiredResourceAccess: API permissions (deprecated, use API Permissions UI)
    - keyCredentials: Certificates (public keys)
    - passwordCredentials: Client secrets
    - identifierUris: Custom URIs (e.g., api://contoso.com/app)
    - knownClientApplications: Related app IDs (OAuth 2.0)
    - publicClient: SPA/mobile/desktop app (no secret)
    - web: Web app config (redirect URIs, implicit grant)
    - spa: Single Page App config
    - api: API exposure (scopes, sets)
    - oauth2AllowIdTokenImplicitFlow: Implicit grant
    - oauth2AllowImplicitFlow: Allow implicit flow
    - accessTokenAcceptedForTenantIds: Multi-tenant config
    - IsDomainVerified: Domain verified for API exposure

  Roles (App Roles):
    → Define: Roles within the app (Authorization, not Authentication)
    → Example:
      - roleId: UUID
        allowedMemberTypes: [User, Application]
        displayName: Admin
        description: Full access
        value: admin
      - roleId: UUID
        allowedMemberTypes: [User]
        displayName: User
        description: Limited access
        value: user
    → Assignment: App Role Assignment (Enterprise Applications → Users/Groups)
    → Usage: In code, check user's role → Different access based on role

  Example Manifest:
    {
      "appId": "12345-abc-def",
      "appRoles": [
        {
          "allowedMemberTypes": ["User", "Application"],
          "displayName": "App.Admin",
          "id": "uuid-1234",
          "isEnabled": true,
          "description": "Full admin access",
          "value": "App.Admin"
        }
      ],
      "publicClient": false,
      "web": {
        "redirectUris": ["https://app.contoso.com/auth/callback"],
        "implicitGrant": {
          "accessTokenIssuanceEnabled": false,
          "idTokenIssuanceEnabled": true
        }
      },
      "requiredResourceAccess": []
    }
```

---

### 3.6 SERVICE PRINCIPALS — COMPLETE DEEP DIVE

**What is a Service Principal?**
A Service Principal is an identity created for use with applications, connected services, and automation tools to access specific Azure resources. It is an instance of an App Registration within a specific tenant (identity in a specific Azure AD/Entra ID tenant).

**Service Principal Lifecycle:**

```
Service Principal Lifecycle:

  1. Creation:
     → Automatic: When App Registration is created (SP created in same tenant)
     → Manual: Create SP in different tenant (B2B, cross-tenant)
     → From Key Vault: Key Vault MI (system-assigned creates SP automatically)

  2. Authentication:
     → Client Secret: Generated in App Registration (expires periodically)
     → Certificate: Uploaded to App Registration (X.509 cert)
     → Federated: External IdP (GitHub, Kubernetes) — no secret needed
     → Managed Identity: Uses IMDS (no auth required in app code)

  3. Authorization:
     → RBAC: Role assignments on subscription/resource group/resource
     → App Role: App-specific roles (assigned in Enterprise Applications)
     → Consent: User consent or Admin consent (API permissions)

  4. Usage:
     → Application uses SP to authenticate (client credentials flow)
     → Gets Access Token: POST /token (grant_type=client_credentials)
     → Token contains: App ID, roles, tenant ID, issuer
     → Calls Azure APIs: Authorization: Bearer <token>

  5. Maintenance:
     → Secret rotation: Rotate client secrets (every 6 months)
     → Certificate renewal: Update cert (annual or as needed)
     → Role review: Verify permissions are still appropriate (quarterly)
     → Activity review: Check SP activity logs (monthly)

  6. Decommissioning:
     → Disable: Set SP to Account Disabled (soft disable)
     → Delete: Remove SP (immediate disable, no recovery)
     → App Registration Delete: SP across all tenants deleted
     → Best practice: Disable (not delete) for incident response

  7. Stale SP Detection:
     → Check: Last activity > 90 days → Stale SP (potential security risk)
     → Check: SP with no role assignment → Orphaned SP (delete)
     → Check: SP with expired secrets → Inactive SP (delete or renew)
     → Check: SP with Owner role → Over-privileged (downgrade)
```

**Service Principal — Secret/Certificate Management:**

```
Client Secret Rotation:
  1. Generate new secret in App Registration (Portal → Certificates & Secrets)
  2. Update application: Use new secret (deploy updated app)
  3. Verify: Application works with new secret (test)
  4. Delete old secret (remove from App Registration)
  5. Verify: Application doesn't work with old secret (should fail)
  6. Audit: Check Activity Log for SP auth events (old secret refused)
  → Time gap: Minimal (minutes between old/new secret valid)

  Best Practice: Two-window rotation
    → Day 1: Add new secret (both old and new valid)
    → Day 2-3: Deploy app with new secret (test in production)
    → Day 4: Delete old secret (after confirming new works)
    → Schedule: Every 6 months (or per policy)

Certificate Rotation:
  1. Upload new certificate (public key) to App Registration
  2. Application: Load new certificate (private key from Key Vault or secure store)
  3. Verify: Application authenticates with new certificate
  4. Remove old certificate from App Registration
  5. Key Vault: Certificate auto-rotation (if CA integration)
  6. Best Practice: Use Key Vault + auto-rotation (no manual work)

  Thumbprint: Certificate identity (SHA-1 hash of public cert)
  → App Registration: Adds thumbprint (not full certificate)
  → Application: Uses private key for signing JWT assertion
  → Thumbprint change: Must update App Registration when rotating cert

Federated Credentials (NEW — No Secret/Cert):
  → Replace: Client secrets and certificates (for certain scenarios)
  → Use: GitHub Actions, Kubernetes, external IdP (OIDC)
  → How: External IdP signs JWT → Entra ID validates signature (thumbprint/URL)
  → No credential storage: No secret in code, no cert in Key Vault
  → Best Practice: Use for CI/CD (GitHub Actions → AKS deployment)
  → Configuration: App Registration → Federated Credentials
    - Issuer URL: https://token.actions.githubusercontent.com
    - Subject identifier: repo:org/repo-name:ref:refs/heads/main
    - Name: GitHub-AKS-Deploy
    - Audience: api://AzureADTokenExchange

  Federated Credential Scenarios:
    → GitHub Actions → Entra ID SP: Deploy to Azure
    → Kubernetes Pod → Entra ID SP: Access Azure APIs (Workload Identity)
    → Docker Hub → Entra ID SP: Pull images
    → Terraform Cloud → Entra ID SP: Deploy Azure resources
```

**Service Principal — Best Practices:**

```
1. Use Managed Identity instead of SP where possible
   → Managed Identity: No credentials to manage
   → SP: Secrets/certs to rotate, audit trail more complex
   → Use MI for: Azure resources (VMs, App Service, AKS, etc.)
   → Use SP for: External applications, cross-tenant access

2. Least privilege RBAC
   → Don't assign: Owner or Contributor (too broad)
   → Assign: Specific role (Reader, Key Vault Secrets User, etc.)
   → Scope: Resource Group or individual resource (not subscription)
   → Review: Quarterly (remove unused permissions)

3. Disable/Delete stale SPs
   → Stale SP: No activity > 90 days
   → Check: Entra ID → Sign-in logs → SP authentication events
   → Action: Disable stale SPs (not delete — keep for recovery)
   → Delete: Only after confirming no usage (90+ days disabled)

4. Never use SP Owner role
   → Owner = Full control over resource + RBAC + access management
   → Compromised SP with Owner = Full infrastructure takeover
   → Use: Contributor (cannot manage RBAC) or custom role
   → Break-glass: Owner role reserved for emergency (not for SPs)

5. Secret/cert expiration monitoring
   → Alert: Secret expires in 30 days
   → Alert: Certificate expires in 30 days
   → Action: Auto-rotation (Key Vault CA integration)
   → Manual: Calendar reminders for secret rotation
   → Test: Monthly test of secret rotation process

6. Unique SP per application
   → Don't share: SP across multiple applications
   → Do: Unique SP per app (clear audit trail)
   → Benefit: Compromised SP → only affects one app

7. Document SPs
   → Registration: App Registration → Description
   → Owner: Assign email owner (not just name)
   → Tagging: Add tags (Environment=Prod, App=Payroll)
   → Documentation: Wiki/GitHub with SP details and purpose

8. SP security monitoring
   → Alert: SP authenticated from unusual IP
   → Alert: SP authenticated outside business hours
   → Alert: SP API calls with high volume (data exfiltration)
   → Monitor: Microsoft Defender for Cloud (SP risk detection)
   → Integration: Microsoft Sentinel (SP anomaly detection)

9. SP naming convention
   → Format: app-{name}-{env}-{purpose}
   → Example: app-payroll-prod-api, ci-cd-bot-dev
   → Tags: Application, Environment, Owner, CostCenter
   → Discovery: Search by name, tag, or application ID

10. SP cross-tenant access
    → Multi-tenant app: SP created in each tenant (B2B)
    → Tenant restrictions: Limit which tenants can access
    → External access: Disable multi-tenant if only single-tenant needed
    → B2B: Use Entra ID B2B (not SP sharing across tenants)
```

---

### 3.7 ENTERPRISE APPLICATIONS — COMPLETE DEEP DIVE

**What are Enterprise Applications?**
Enterprise Applications in Entra ID represent SaaS applications (Salesforce, Salesforce, GitHub, ServiceNow, etc.) that are integrated with Entra ID for single sign-on (SSO), provisioning, and conditional access. When users access these SaaS apps via Entra ID, Entra ID acts as the identity provider.

**Enterprise Application Types:**

```
1. SaaS Applications:
   → Examples: Salesforce, GitHub, ServiceNow, Dropbox, Zoom, Okta
   → Managed by: SaaS vendor (not Entra ID)
   → Integration: SSO via SAML or OIDC (federated)
   → Provisioning: SCIM (auto-create/disable users via Entra ID)

2. Application Proxy (On-Prem Apps Published):
   → Example: Internal web app (HR portal, ERP) published via Entra ID
   → Connector: On-prem server running Application Proxy connector
   → Access: Users access via Entra ID URL (no VPN needed)
   → Seamless SSO: Auto-login (no password prompt)

3. Azure AD Gallery Applications:
   → Microsoft Marketplace apps (verified by Microsoft)
   → Examples: Power BI, Dynamics 365, Azure DevOps, etc.
   → Integration: Native Entra ID integration (no custom config)
   → Provisioning: SCIM (often automated)

4. Web APIs / Custom APIs:
   → Examples: Custom REST API exposed as enterprise app
   → Integration: OAuth 2.0 / OIDC
   → Use: App-to-app authentication (not user-facing)

5. Other (Deprecated/Advanced):
   → Facebook, Google, Amazon, etc. (deprecated)
   → Core Microsoft services (Office 365, Azure)
```

**SaaS Application — SSO Integration Deep Dive:**

```
SAML SSO (Security Assertion Markup Language):
  → Flow:
    1. User accesses SaaS app (e.g., Salesforce)
    2. SaaS app redirects to Entra ID login page
    3. User authenticates (username + MFA)
    4. Entra ID generates SAML assertion (user identity, roles, groups)
    5. SAML assertion sent to SaaS app (via HTTP POST)
    6. SaaS app validates assertion (digital signature)
    7. User logged in (no separate SaaS password)

  → Configuration:
    1. Enterprise Application: Salesforce (gallery app)
    2. SSO method: SAML
    3. Entra ID metadata: Federation metadata XML (contains SP entity ID, signing cert)
    4. SaaS app metadata: Upload Salesforce metadata XML (contains SP entity ID, signing cert)
    5. Attribute mapping: Entra ID attributes → SaaS attributes
       - Name → Email
       - given_name → First Name
       - surname → Last Name
       - group → Role (Salesforce profile)

  → Attribute Mapping:
    → Basic: Name, Email, First Name, Last Name
    → Custom: Departments, Job Title, Groups (for role mapping)
    → Groups: Entra ID groups → SaaS app roles (auto-provision)
    → Programmatic: SCIM (for user lifecycle management)

OIDC SSO (OpenID Connect):
  → Flow:
    1. User accesses SaaS app
    2. SaaS app redirects to Entra ID (Authorization Code flow)
    3. User authenticates
    4. Entra ID returns: ID Token (JWT) + Access Token
    5. SaaS app validates ID Token (signature, issuer, audience)
    6. User logged in

  → Benefits over SAML:
    - JSON-based (easier to parse)
    - Built on OAuth 2.0 (familiar)
    - Better for mobile/SPA apps
    - Simpler implementation

  → Configuration:
    1. Enterprise Application: App (gallery)
    2. SSO method: OIDC
    3. Configure: Client ID, Issuer, JWKS endpoint
    4. Scopes: openid, profile, email (standard OIDC scopes)
    5. Claims: Map Entra ID claims → SaaS claims

SCIM Provisioning (User Lifecycle Management):
  → SCIM (System for Cross-domain Identity Management)
  → Standard: REST API for user provisioning
  → Flow:
    1. Entra ID SCIM endpoint: SaaS app configured with SCIM
    2. User created in Entra ID → SCIM push to SaaS → User provisioned in SaaS
    3. User disabled in Entra ID → SCIM push → User disabled in SaaS
    4. User deleted in Entra ID → SCIM push → User deleted in SaaS

  → Configuration:
    1. Enterprise Application → Provisioning → Manual
    2. SCIM URL: SaaS app SCIM endpoint (e.g., https://salesforce.com/scim)
    3. Auth Token: Bearer token (from SaaS admin)
    4. Tenant: SaaS tenant identifier
    5. Attribute Mapping: Entra ID attributes → SCIM attributes
       - userName → Email
       - name.givenName → First Name
       - name.familyName → Last Name
       - active → Status (true/false)
       - roles → Role (custom)
    6. Provisioning mode: Automatic (real-time) or Manual (on-demand)

  → Use Cases:
    → Onboarding: User added to Entra ID group → Auto-provisioned in SaaS
    → Offboarding: User removed from group → Disabled/deleted in SaaS
    → Role change: User moved between groups → Roles updated in SaaS

  → Best Practice: Always enable SCIM provisioning for critical SaaS apps
    → Prevents: Orphaned SaaS accounts (user leaves company but still has access)
    → Automates: User lifecycle (no manual SaaS admin work)
    → Audits: Full provisioning history (who was provisioned, when)

Enterprise Application — Conditional Access:
  → Apply Conditional Access to SaaS apps (via Entra ID)
  → Policies:
    - Require MFA for all SaaS apps
    - Block access from untrusted locations
    - Require compliant device for SaaS access
    - Block legacy authentication (basic auth) for SaaS
    - Require app-resilient MFA (phishing-resistant)
    - Rate-limit sign-ins for SaaS apps (prevent brute force)

  → Conditional Access for SaaS:
    → Create: Policy targeting specific SaaS app (or all)
    → Conditions: Users, Groups, Risk, Device, Location, App, Sign-in
    → Access Controls: Grant (MFA, compliant device, app-enforced restrictions)
    → Session: Sign-in frequency (re-auth every 1-12 hours)
    → App-resilient MFA: Apps with FIDO2/PTA (phishing-resistant)

  → App-Enforced Restrictions:
    → Block download: Prevent data download (for sensitive SaaS)
    → Block copy/paste: Prevent data exfiltration
    → Block print: Prevent hard-copy exfiltration
    → Use: Sensitive data apps (SharePoint, OneDrive, Salesforce)

  → Sign-In Frequency:
    → Default: Every 12 hours (re-authenticate)
    → Configurable: 1 minute to 90 days
    → Use: Short frequency for sensitive apps (every 1-4 hours)
    → Note: Conditional Access re-evaluates at each sign-in (not session-based)
```

---

### 3.8 ENTRA PROXY APPLICATION (APPLICATION PROXY) — COMPLETE DEEP DIVE

**What is Entra Application Proxy?**
Application Proxy is a service in Entra ID that enables on-premises applications to be accessed securely from the cloud (via Entra ID), without VPN. It uses a connector on-premises to proxy requests from the cloud to the on-prem app, providing SSO and conditional access.

**Application Proxy Architecture:**

```
┌─ Cloud Side (Entra ID) ─────────────────────────────────────────────┐
│                                                                      │
│  Entra ID (globalcontoso.onmicrosoft.com)                            │
│  ├── Application Proxy (service)                                     │
│  │   → Published App: on-prem-contoso.com                            │
│  │   → URL: https://onprem.contoso.com (Entra ID-accessible)         │
│  │   → Backend URL: http://hr-app.internal.contoso.com:8080          │
│  │   → Connector Group: On-prem-connector-group (West US)            │
│  │   → Authentication: Seamless SSO, Integrated Windows Auth         │
│  │   → Pre-Authentication: Entra ID (user must authenticate)         │
│  │   → Session: Single-connection (or persistent)                    │
│  │                                                                   │
│  ├── Connector Server (proxy):                                       │
│  │   → Installed on: On-prem server (Windows Server)                 │
│  │   → Connector Group: On-prem-connector-group                      │
│  │   → Outbound: HTTPS (443) to Entra ID (no inbound firewall open)  │
│  │   → Health: Connected (active)                                    │
│  │                                                                   │
│  └── Seamless SSO: Auto-login (no password prompt)                   │
│      → Kerberos authentication (on-prem AD)                          │
│      → SSO: Users auto-authenticated (using on-prem AD credentials)  │
│      → Requires: Entra Connect sync (on-prem AD → Entra ID)         │
│                                                                   │
└──────────────────────────────────────────────────────────────────────┘

┌─ On-Premises Side ──────────────────────────────────────────────────┐
│                                                                      │
│  On-Prem Network                                                   │
│  ├── HR App: http://hr-app.internal.contoso.com:8080                │
│  │   → Accessible: Internal network only (no internet exposure)      │
│  │   → Authentication: Integrated Windows Auth (Kerberos)           │
│  │   → No public endpoint (never exposed to internet)                │
│  │                                                                   │
│  ├── Connector Server:                                              │
│  │   → App Proxy Connector: v4.x                                   │
│  │   → Runs as: NT SERVICE\AADNLBProxyService (low-privilege)       │
│  │   → Communicates: HTTPS (443) outbound to Entra ID               │
│  │   → No inbound connections (reverse proxy pattern)                │
│  │   → Connector Group: On-prem-connector-group (West US)           │
│  │                                                                   │
│  ├── DNS: onprem.contoso.com → Internal IP (or not even needed)     │
│  │   (Application Proxy doesn't require DNS on-prem)                 │
│  │                                                                   │
│  └── AD: hr-app.internal.contoso.com (domain-joined app server)      │
│      → Seamless SSO: Entra ID authenticates via Kerberos              │
│      → No password required (auto-login via Integrated Windows Auth) │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘

User Flow:
  1. User: Browser → https://onprem.contoso.com
  2. Entra ID: Cloud-side (URL resolves to Entra ID endpoint)
  3. Entra ID: Prompts user to authenticate (if not already)
  4. User: Authenticates with Entra ID credentials (MFA enforced)
  5. Entra ID: Seamless SSO (auto-login if on-domain, no MFA challenge)
  6. Entra ID: Application Proxy → Routes request to Connector Group
  7. Connector: On-prem server → Reverse proxy → HR App (http://hr-app:8080)
  8. HR App: Returns response → Connector → Entra ID → User
  9. User sees: On-prem app in browser (no VPN, no RDP)

Connector Details:
  → Connector VM: Small Windows VM (or container) in on-prem datacenter
  → Connector OS: Windows Server 2016+
  → RAM: 2 GB minimum, 4 GB recommended
  → CPU: 2 cores minimum
  → Network: Outbound HTTPS (443) to Entra ID only (no inbound required)
  → Installation: Download from Entra ID portal, run installer, register
  → Registration: Connector → Entra ID (authenticates via Entra ID token)
  → High availability: Multiple connectors in same group (load balanced)
  → Multiple groups: Different on-prem locations (West US, East US, etc.)
  → Auto-update: Connector updates automatically (latest version)
  → Health monitoring: Entra ID checks connector health (every 5 min)
  → Removal: Connector removed → App Proxy stops routing through it
  → Security: Connector doesn't store user credentials (token-based auth)

Application Proxy vs VPN:
  ┌─ Application Proxy ──────────┐ vs ┌─ VPN ──────────────────────┐
  → No VPN client needed          │    → VPN client required        │
  → Accessed via Entra ID URL     │    → Accessed via VPN network    │
  → SSO via Entra ID             │    → VPN authentication          │
  → Conditional Access enforced   │    → Limited Conditional Access  │
  → Reverse proxy (cloud → on-prem)│   → Direct network access       │
  → No on-prem firewall changes   │    → May need firewall changes   │
  → Specific apps published       │    → Full network access         │
  → Cost: Entra ID P1 (included)  │    → Cost: VPN Gateway SKU      │

Application Proxy vs Azure Bastion:
  ┌─ Application Proxy ──────┐ vs ┌─ Bastion ──────────────────┐
  → Web application proxy     │    → VM remote access (RDP/SSH) │
  → App-level access          │    → Server-level access         │
  → No VM access (just app)   │    → Full VM access             │
  → SSO via Entra ID          │    → Entra ID + Certificate     │
  → Published URL             │    → Bastion host URL            │

Application Proxy Limitations:
  → No File Stream Access: (Windows file server not supported via proxy)
  → No Windows ODBC: Cannot use ODBC through Application Proxy
  → Session persistence: Required for multi-window operations (SPNs)
  → Web app only: Must be HTTP/HTTPS (no raw TCP)
  → No custom headers: Cannot pass custom headers to backend
  → No WebSocket: Not supported (use SignalR or web sockets via app gateway)
  → SPN conflicts: Session cookies might conflict with backend auth (Kerberos)
  → Pre-auth: Must be Entra ID (no anonymous access through proxy)
  → Publish settings: Download via PowerShell (not portal)

Session Types:
  → Single Connection (default):
    → One connection at a time
    → Fast (low overhead)
    → Use: Simple web apps

  → Persistent (recommended for complex apps):
    → Multiple connections per session
    → Required for: Multi-tab apps, apps with sessions, NTLM/Kerberos
    → Feature: Supports Kerberos authentication for backend
    → Important: Enable for apps that require Kerberos (SPN conflict)

 Kerberos Through Application Proxy:
    → Backend app: Kerberos authenticated (Windows Integrated Auth)
    → Application Proxy: Forwards Kerberos ticket (delegation)
    → User: Seamless SSO (auto-Kerberos, no prompt)
    → Pre-auth: Seamless SSO via Kerberos Constrained Delegation (KCD)
    → Configuration:
      - Azure AD Connect: Enable Seamless SSO (KCD)
      - Connector: In same domain (Kerberos works)
      - Backend app: SPN configured (HTTP/fqdn)
    → Limitation: Requires Azure AD Connect Seamless SSO + KCD

Application Proxy — Publishing a Web App:
  Step 1: On-Prem Connector Installation
    → Download: Portal → Entra ID → Application Proxy → Download Connector
    → Install: Run on Windows Server (in same domain as target app)
    → Register: Sign in with Entra ID admin credentials
    → Verify: Connector status: "Connected" (Portal → Application Proxy → Connectors)
    → Connector Group: Add to group (e.g., "OnPrem-WestUS")

  Step 2: Publish Web App
    → Portal: Entra ID → Application Proxy → + Publish
    → Display Name: HR Portal (on-prem)
    → Internal URL: http://hr-app.internal.contoso.com:8080
    → External URL: https://onprem.contoso.com (auto-generated or custom)
    → Authentication: Seamless SSO + Integrated Windows Auth
    → Connector Group: OnPrem-WestUS
    → Session: Persistent (for Kerberos/SPN)
    → Pre-Auth: Entra ID (verified)
    → Tags: Environment=Production, App=HR
    → Translate: URL translation mode (default)

  Step 3: Assign Access
    → Application Proxy → Users and groups
    → Assign: Entra ID groups (e.g., "HR Users")
    → Access: Only assigned users can access published app
    → Conditional Access: Enforced (if policy configured)

  Step 4: Test Access
    → External: User (on-domain, corporate network) → onprem.contoso.com
    → Expected: Seamless SSO (no prompt) → HR app loaded
    → External: User (off-network) → onprem.contoso.com
    → Expected: Entra ID login (MFA) → HR app loaded (via VPN/Wi-Fi)
    → Unauthorized: User (not in HR Users group) → Access denied
```

---

## 4. DESIGN PATTERNS

### 4.1 Zero Trust Identity Architecture

```
Zero Trust: Never trust, always verify

  Layer 1 — Identity Verification:
    → All users: MFA (Entra ID Conditional Access)
    → All apps: Entra ID authentication (no anonymous)
    → All SPs: Certificate/secrets (never in code)
    → New users: Just-in-time (PIM) for privileged access
    → Risk-based: Sign-in risk → Additional verification

  Layer 2 — Device Verification:
    → Conditional Access: Compliant devices only
    → Device registration: Entra ID device registration
    → Intune: Device compliance policy
    → BYOD: Conditional Access for personal devices (limited)
    → Result: Non-compliant device → Block or limited access

  Layer 3 — Network Verification:
    → No public endpoints: All private (Private Endpoint)
    → NSG: Default deny inbound internet
    → Firewall: All traffic inspected (UDR)
    → Bastion: VM access (no public IPs)

  Layer 4 — App Verification:
    → App Service: Easy Auth (Entra ID only)
    → AKS: Namespace-level RBAC
    → Key Vault: RBAC (not access policies)
    → Storage: SAS tokens (not account keys)

  Layer 5 — Data Verification:
    → Key Vault: All secrets encrypted (CMK)
    → Storage: Immutable backups (soft delete + immutable)
    → Data classification: Labels on all data (Confidential, Public, etc.)
    → DLP: Data loss prevention (restrict data sharing)

  Implementation:
    1. Conditional Access: All users, all apps, MFA required
    2. Remove: Global admin accounts (break-glass only)
    3. Replace: All admin accounts with PIM (just-in-time elevation)
    4. Deploy: Private Endpoints for all PaaS services
    5. Deploy: Key Vault (all secrets, certs, keys)
    6. Deploy: Managed Identity (all Azure resources)
    7. Enable: Soft Delete + Purge Protection on all storage
    8. Enable: Defender for Cloud (all resources)
    9. Enable: Monitoring (all resources → Sentinel)
    10. Review: Quarterly (access review, stale SPs, over-privilege)
```

### 4.2 Secure Secret Management Pattern

```
Secret Management Architecture:

  Application → Key Vault (all secrets stored here)
    → No secrets in code, config, or environment variables
    → Access: Managed Identity (system-assigned or user-assigned)
    → RBAC: App MI → Key Vault Secrets Officer role

  Key Vault:
    → Soft Delete: 90 days (recover deleted secrets)
    → Purge Protection: Enabled (cannot purge until soft delete ends)
    → Network: Private Endpoint only (no public access)
    → RBAC: App MI (secrets read), Admin MI (secrets full access)
    → Access Policy: Key Vault Administrator (admin only)
    → Audit: All access logged (who, what, when)
    → Rotation: Auto-rotation for certificates (CA integration)

  VM/App:
    → System MI: VM/App Service has Entra ID identity
    → Token: IMDS → Key Vault token
    → RBAC: App MI → Key Vault role (Secrets User/Officer)
    → Access: App calls Key Vault → MI token validated → Secret returned
    → No credentials in code or config

  CI/CD:
    → Service Connection: Entra ID SP → Key Vault access
    → Pipeline: Uses SP token → Key Vault → Secrets → Deployment
    → No secrets in pipeline variables
    → Managed Identity: AKS/Pipeline MI → Key Vault

  Database:
    → SQL DB: Entra ID admin (not SQL auth)
    → App MI: SQL DB RBAC (db_datareader, db_datawriter)
    → Connection: Token-based (no password in connection string)
    → Token: Acquired via MI → SQL DB → Validates MI identity

  Key Vault Best Practices:
    1. All secrets → Key Vault (never in config/app settings)
    2. Access → RBAC (not access policies where possible)
    3. Network → Private Endpoint (no public access)
    4. Soft Delete + Purge Protection → Enabled (all vaults)
    5. Logging → All access logged to Log Analytics/Sentinel
    6. Rotation → Certificates auto-rotate (Key Vault + CA integration)
    7. Secrets → Short lifespan (rotate regularly)
    8. RBAC review → Quarterly (who can access KV)
    9. Auditing → All KV access monitored (alerts for anomalies)
    10. Backup → KV backup to Storage (encrypted, geo-redundant)
```

---

## 5. PRODUCTION EXAMPLE

**Scenario: Global enterprise (50K employees, 50 SaaS apps, 100 on-prem apps, 5 regions, compliance-driven).**

```
IDENTITY ARCHITECTURE:

  Microsoft Entra ID Tenant: globalcontoso.onmicrosoft.com
  → Tier: P2 (Conditional Access, PIM, Identity Protection, Governance)
  → Users: 50,000 (synced from on-prem AD via Entra Connect)
  → Groups: 500 (department, role-based, app-specific)
  → Dynamic groups: Auto-populated based on attributes

  ── CONDITIONAL ACCESS POLICIES ──
  Policy 1: "All Users - MFA Required"
    → Users: All (except break-glass)
    → Conditions: All cloud apps
    → Grant: Require MFA (or FIDO2 for admins)
    → Session: Sign-in frequency 12 hours

  Policy 2: "Block Legacy Authentication"
    → Users: All
    → Client apps: Exclude modern auth clients
    → Grant: Block access
    → Exclusions: Service SPs with MFA restriction (declare supported)

  Policy 3: "Admins - Break-Glass"
    → Users: Global Admin + PIM roles
    → Conditions: Risky sign-in, unfamiliar locations
    → Grant: Require MFA, compliant device
    → PIM: All admin roles activated via PIM (4-hour max)

  Policy 4: "Risk-Based Access"
    → Users: All
    → Conditions: Sign-in risk (medium/high)
    → Grant: Require password change + MFA
    → Risk levels: Medium (require MFA), High (block + alert)

  Policy 5: "Device Compliance"
    → Users: All
    → Conditions: Device state (compliant/not compliant)
    → Grant: Compliant → Full access; Not compliant → Limited access
    → Limitation: Block access for unmanaged devices

  ── PRIVILEGED IDENTITY MANAGEMENT (PIM) ──
  → Global Admin: Activated via PIM (max 4 hours, approval required)
  → SQL Admin: Activated via PIM (max 8 hours, approval required)
  → Network Admin: Activated via PIM (max 8 hours, approval required)
  → Break-glass accounts: Global Admin (standalone, local, not in Entra ID)
    → Emergency: Physical break-glass procedure (not Entra ID based)

  ── KEY VAULT INFRASTRUCTURE ──
  Per Region: 1 Key Vault (kv-contoso-{region})
    → Soft Delete: 90 days
    → Purge Protection: Enabled
    → Network: Private Endpoint + Firewall (Selected Networks)
    → RBAC: Vault Access (vs Access Policies)
    → Certificates: App certs (auto-renew via CA)
    → Secrets: Connection strings, API keys, passwords
    → Keys: Encryption keys (CMK for Storage, SQL)
    → Audit: All access logged
    → Backup: Weekly (encrypted, geo-redundant)

  ── MANAGED IDENTITY DEPLOYMENT ──
  All VMs: System-assigned MI (no local admin credentials)
  All App Services: System-assigned MI
  All AKS Node Pools: System-assigned MI
  All ACI: User-assigned MI (for cross-resource access)
  All Data Factory: System-assigned MI
  All Function Apps: System-assigned MI

  RBAC Assignments (MI → Resources):
    VM → Key Vault: Secrets Officer (read secrets)
    App Service → Storage: Blob Data Reader
    AKS → SQL: SQL DB Data Reader/Writer
    Function → Service Bus: Message Sender/Receiver

  ── SERVICE PRINCIPALS ──
  CI/CD: ci-cd-bot-{env}
    → Auth: Federated credential (GitHub Actions → Entra ID)
    → RBAC: Contributor on Resource Group (per environment)
    → Scope: Not subscription (granular: RG level)
    → Lifetime: Certificate rotation (annual, Key Vault managed)

  Applications: app-{name}-{env}-{purpose}
    → Auth: Certificate (Key Vault)
    → RBAC: Specific roles on required resources
    → Monitoring: Activity Log monitoring for SP auth
    → Review: Quarterly (disable stale SPs)

  ── APP REGISTRATIONS ──
  Per Application: 1 App Registration
    → Multi-tenant: No (single-tenant)
    → Auth: Client credentials (certificate)
    → API permissions: Specific to required APIs (admin consent)
    → RBAC: Assigned to Service Principal (Enterprise Applications)

  ── ENTERPRISE APPLICATIONS (SaaS) ──
  Per SaaS App: 1 Enterprise Application
    → SSO: SAML or OIDC (per app requirements)
    → Provisioning: SCIM automatic
    → Conditional Access: Per-app policies (sensitive apps)
    → Assignment: Entra ID groups → SaaS roles
    → Access Review: Quarterly (who has access to which SaaS)

  Salesforce: Entra ID SSO (OIDC) + SCIM provisioning
  GitHub: Entra ID SSO (OIDC) + SCIM provisioning (enterprise plan)
  ServiceNow: Entra ID SSO (SAML) + SCIM provisioning
  Zoom: Entra ID SSO (OIDC) + SCIM provisioning
  Dropbox: Entra ID SSO (SAML) + SCIM provisioning

  ── APPLICATION PROXY ──
  Per On-Prem App: 1 Published App
    Connector Group: Per datacenter (West US, East US, etc.)
    Published Apps: 100+ on-prem apps published
    Seamless SSO: Enabled for all (Kerberos delegation)
    Conditional Access: Enforced on all published apps
    Access: Entra ID groups assigned per app

  ── STORAGE DATA PROTECTION ──
  Per Storage Account:
    → Encryption: CMK (Key Vault)
    → Firewall: Selected Networks + Private Endpoint
    → Soft Delete: Enabled (90 days for Blob, 14 days for Queue/Table)
    → Immutable: Time-based for compliance data (7 days)
    → Versioning: Enabled
    → OLM: Hot → Cool after 30 days → Archive after 90 days
    → Defender: Enabled for all storage accounts
    → SAS: None (all access via MI/RBAC)
    → Account Keys: Disabled (use MI only, no SAS)

  ── STORAGE ACCESS ──
  No SAS tokens in production:
    → All access via Managed Identity (RBAC)
    → No SAS tokens (no storage account keys exposed)
    → Alternatives: SAS only for dev/test or third-party sharing (scoped, time-limited)
    → External sharing: Entra ID B2B (not SAS)
    → Third-party: User Delegation SAS (Azure AD-based)

  ── MONITORING ──
  Entra ID → Audit Logs: All sign-ins, changes, PIM events
  Key Vault → Audit Logs: All secret/key/cert operations
  Microsoft Defender: Identity Protection, Risk detections
  Microsoft Sentinel: Correlation across all logs
  Alerts: Failed access attempts, stale SP detection, over-privilege

  ── COST ──
  Entra ID P2: ~$6/user/month × 50,000 = $3.6M/year (identity platform)
  Key Vault: ~$0.03/vault/month + ~$0.03/10,000 transactions × 10 regions = minimal
  Application Proxy: Entra ID P1 included (connectors on-prem VM cost)
  PIM: Entra ID P2 included
  Managed Identity: No additional cost (included in Azure resources)
```

---

## 6. FAILURE SCENARIOS

| Scenario | Root Cause | Resolution | Evidence |
|-----------|-----------|------------|----------|
| **App cannot access Key Vault (403 Forbidden)** | MI not assigned to app; RBAC role not assigned to MI; Wrong key vault; Network firewall blocking; Private DNS not resolving. Most common: RBAC role not assigned to MI on Key Vault (app has MI but no Key Vault role). | Check: App → Identity → MI assigned; Check: KV → Access Control (RBAC) → App MI role assigned (Secrets Officer/User); Check: KV → Networking → Firewall (is IP/MI bypassed?); Check: Private DNS: privatelink.vaultcore.azure.net linked to VNet; Fix: Assign RBAC role to MI on KV; Test: az rest --uri https://kv.vaultcore.azure.net/secrets?api-version=7.3. | Portal: App → Identity; KV → Access Control; KV → Networking; Log Analytics: AuditEvent (403 errors); Activity Log: RBAC assignments |
| **User cannot sign in (Conditional Access blocks)** | MFA required but not enrolled; Device non-compliant; Location blocked; Risk-based policy blocks; Legacy auth used. Most common: User not enrolled in MFA (or MFA registered but token expired). | Check: Sign-in logs → Conditional Access → Why blocked; Check: User MFA status (Portal → Users → MFA); Check: Device compliance (Intune); Check: Location (is user in allowed location?); Fix: User enroll MFA, device compliance, join allowed location; Or: Admin grant bypass (PIM, temporary). | Portal: Sign-in logs → Conditional Access → Status; User: MFA registration; Device: Compliance status; Location: Geo-coordinates |
| **Storage account accessible via public endpoint (leak)** | Firewall not configured (Public access); Private Endpoint not created; SAS token leaked; Account keys exposed. Most common: Firewall set to "All networks" (public access enabled by default in older storage accounts). | Check: Storage Account → Networking → Firewalls and Virtual Networks → Selected Networks (must be enabled); Check: Private Endpoint: Exists and Approved; Check: SAS tokens: None in production; Check: Account keys: Disabled (use MI only); Fix: Enable firewall + Private Endpoint + Disable account keys; Rotate: Any leaked credentials; Monitor: Defender for Storage (alerts). | Portal: Storage → Networking (firewall status); Private Endpoint: Status; SAS: Storage Analytics; Log Analytics: Storage access logs; Defender for Storage: Alerts |
| **SAS token exploited (unauthorized access)** | SAS token had excessive permissions (full access); No expiry (never expires); IP restriction not set; HTTPS not enforced; Shared account key used (not SAS). Most common: SAS token with write+delete permissions + no expiry → Attacker deletes data. | Check: Storage Analytics: SAS token usage; Check: SAS permissions: Restricted (read-only, time-limited); Check: SAS expiry: Far future (should be short); Check: HTTPS: Enforced; Fix: Create new SAS (scoped, short expiry, HTTPS); Remove: Old SAS (if compromised); Enable: Defender for Storage; Audit: All SAS usage. | Storage Analytics: SAS authentication logs (percentage of requests via SAS); Defender for Storage: Alerts; Storage Access Logs: IP/timestamp/operations |
| **Managed Identity cannot authenticate to SQL DB** | MI not assigned to app; Entra ID admin not set on SQL; RBAC not assigned (db_datareader); SQL DB firewall blocks; Connection string uses password (not MI). Most common: SQL DB not configured with Entra ID admin (MI cannot authenticate without Entra ID admin on SQL). | Check: SQL Server → Security → Administrators → Entra ID admin (must be set); Check: App → Identity → MI assigned; Check: SQL DB → Security → RBAC → MI assigned (db_datareader, etc.); Check: SQL DB → Networking → Firewall (allow Azure services); Fix: Set Entra ID admin; Assign RBAC role to MI; Use Token-based connection (no password). | Portal: SQL Server → Security → Administrators; App → Identity; SQL DB → RBAC assignments; Application: Token-based connection string |
| **Service Principal secret expired → App fails** | SP secret expired → App cannot authenticate; No secret rotation process; App not updated with new secret. Most common: Secret expired after 1-2 years (default) and no rotation process in place. | Check: SP → Certificates & Secrets → Expiry date; Check: Activity Log: SP authentication failures; Check: App: Error "Invalid client secret"; Fix: Generate new secret → Update app → Verify → Delete old secret; Implement: Secret rotation process (Key Vault + auto-rotation or 90-day manual process). | Portal: SP → Secrets (expiry); Activity Log: Sign-in failures (401); Application logs: Auth failures; Key Vault: Secret expiry notifications |
| **Certificate expired → App cannot authenticate** | X.509 cert expired → App signs JWT with expired cert → Entra ID rejects; No renewal process; App not updated with new cert thumbprint. Most common: Manual cert management (no Key Vault CA integration → manual renewal → process missed). | Check: App Registration → Certificates & Secrets → Expiry; Check: Key Vault: Certificate auto-renewal status; Check: Activity Log: SP auth failures; Fix: Upload new cert to App Registration + Key Vault → Verify app works → Delete old cert; Implement: Key Vault CA integration (auto-renewal). | Portal: App Registration → Certificates; Key Vault: Cert status; Activity Log: SP auth events; Application: Token acquisition failures |
| **Private Endpoint for Key Vault in Pending state** | Private Endpoint not approved (manual approval required); Private DNS Zone not linked; Subnet conflict; VNet mismatch; DNS resolution failing. Most common: Private Endpoint → Provisioning State: Pending → Needs Manual Approval (if private endpoint policy requires). | Check: Private Endpoint → Provisioning State → Pending (manual) or Succeeded (auto); Check: Private DNS Zone linked to VNet; Check: Subnet: Not used by other Private Endpoints; Check: DNS resolution: nslookup from VM; Fix: Approve Private Endpoint (Portal → Private Endpoint → Approve); Link Private DNS Zone; Configure DNS; Verify: nslookup returns private IP. | Portal: Private Endpoint → Status; DNS: nslookup; Private DNS Zone: VNet links; Log Analytics: PrivateLinkMatch events |
| **Application Proxy connector offline** | Connector VM stopped/restarted; Network disconnected (outbound 443 blocked); Certificate expired (connector auth token); Connector version outdated; Connector group mismatched. Most common: Connector VM stopped (maintenance, restart, or network failure). | Check: Portal → Application Proxy → Connectors → Status (Connected/Disconnected); Check: Connector VM: Running (Azure portal → VM → Status); Check: Network: Outbound HTTPS (443) to Entra ID available; Check: Connector version: Latest; Fix: Restart VM; Restore network; Update connector; Check: Connector health every 5 min (automatic). | Portal: Connectors status; Connector VM: Status; Log Analytics: Application Proxy logs; Published app: Access failures |
| **Seamless SSO not working (password prompt)** | Entra Connect not configured; KDC not responding; Azure AD Connect service stopped; SPN not configured; Device not domain-joined; User not on-prem AD; Seamless SSO disabled. Most common: Entra Connect Seamless SSO not enabled or service stopped. | Check: Entra ID → Seamless SSO → Status (Enabled/Disabled); Check: Entra Connect: Service running; Check: KDC: Reachable (kerberos); Check: Device: Domain-joined; Check: User: Synced from on-prem; Fix: Enable Seamless SSO; Restart Entra Connect service; Verify: Auto-login works (no prompt). | Portal: Seamless SSO; Entra Connect: Service status; Device: Domain join; Sign-in logs: Seamless SSO events |
| **Multi-tenant SaaS app rejected (user from Tenant B)** | App Registration → Supported account types: Single tenant (not multi); B2B invitation not sent; Tenant consent not granted; User not in Tenant B; App not published for Tenant B. Most common: App Registration set to Single tenant (must be Multi-tenant for cross-tenant access). | Check: App Registration → Supported account types → Multi-tenant; Check: Enterprise Application → User assignments (Tenant B users added); Check: Tenant consent: Admin consent for Tenant B; Fix: Change to Multi-tenant; Add users from Tenant B; Configure: Tenant restrictions (if needed); Verify: User from Tenant B can authenticate. | App Registration: Supported account types; Enterprise Application: Assignments; Sign-in logs: Multi-tenant events; Tenant consent: Admin consent settings |
| **Enterprise Application provisioning stuck (SCIM)** | SCIM endpoint not configured; Auth token expired; Network connectivity (SCIM URL blocked); Attribute mapping incorrect; User not in assigned Entra ID group; SCIM disabled. Most common: Auth token expired (SCIM token needs periodic rotation) or user not in assigned group. | Check: Enterprise Application → Provisioning → Status (Started/Not Started); Check: SCIM URL: Accessible; Check: Auth Token: Valid (not expired); Check: Attribute Mapping: Correct; Check: User group assignment: Correct; Check: Provisioning mode: Automatic; Fix: Update auth token; Verify SCIM URL; Fix attribute mapping; Check user group membership; Enable automatic provisioning. | Enterprise App: Provisioning status; SCIM logs: User provisioning events; Provisioning errors: Log Analytics; User: Group assignment (Enterprise App → Users and Groups) |
| **SSO for SaaS app not working (SAML assertion invalid)** | SAML signing cert expired (or incorrect); Entra ID metadata not updated on SaaS app; SaaS app metadata not updated in Entra ID; Attribute mapping wrong (email mismatch); User not assigned to app. Most common: SAML signing certificate expired on Entra ID side (auto-renew not configured). | Check: Enterprise App → SSO → Certificate expiry; Check: SaaS app: SAML configuration (matches Entra ID metadata); Check: Attribute mapping: Entra ID attributes → SaaS attributes (correct); Check: User assignment: User added to Enterprise App; Fix: Update cert (re-upload or auto-renew); Sync metadata; Fix attribute mapping; Add user assignment. | Enterprise App: SSO settings; Sign-in logs: SAML failures; SaaS app: Error message (invalid assertion); User: Assignment status |
| **Federated credential not working (GitHub Actions → Azure)** | Federated credential not configured correctly; Issuer URL mismatch; Subject identifier wrong; OIDC discovery not accessible; App Registration → Issuer URI mismatch; GitHub OIDC not enabled. Most common: Subject identifier wrong (not matching GitHub repository format). | Check: App Registration → Federated Credentials → Issuer URL: https://token.actions.githubusercontent.com; Subject: repo:org/repo-name:ref:refs/heads/main; Check: GitHub: OIDC provider enabled; Check: Repository: Correct (org/repo-name); Check: Branch: Correct (main); Check: Audience: api://AzureADTokenExchange; Fix: Correct federated credential configuration; Verify: GitHub Actions → Azure auth works; Check: Entra ID sign-in logs for federated auth events. | App Registration: Federated Credentials; GitHub: Repository OIDC; Sign-in logs: Federated credential events; Azure Actions: Service connection status |
| **App Registration client secret expired → CI/CD fails** | CI/CD service principal secret expired → Pipeline cannot authenticate to Azure; Secret rotation process not in place; Secret in pipeline config not updated. Most common: SP secret expired (1-2 years after creation) and CI/CD config not updated. | Check: App Registration → Certificates & Secrets → Secret expiry; Check: CI/CD: Pipeline logs (Auth failure); Check: Service connection: Azure RM in DevOps (last updated); Fix: Generate new secret → Update pipeline service connection → Run test → Delete old secret; Implement: Secret rotation process (quarterly). | App Registration: Secret expiry; CI/CD logs: Auth failures; DevOps: Service connection last updated; Activity Log: SP sign-in events |
| **Storage Account Key still in use (despite MI recommended)** | Account keys not disabled; Some apps still using account keys (legacy); Account keys not rotated; Monitoring not enabled for account key usage. Most common: Legacy app using account keys instead of MI (not yet migrated). | Check: Storage Account → Access Keys → Key1/Key2 status (active); Check: Storage Analytics: Authentication type (AccountKey vs MI vs SAS); Check: All apps: Migrated to MI; Check: Account keys: Rotate (if still in use); Fix: Disable account keys; Migrate all apps to MI; Enable: Defender for Storage; Monitor: Storage Analytics auth logs. | Storage Account: Access Keys status; Storage Analytics: Auth type distribution; Defender for Storage: Alerts; Log Analytics: Storage auth type logs |

---

## 7. MONITORING

| Metric/Log | Service | What It Shows | Alert Trigger |
|-----------|---------|---------------|---------------|
| **Sign-in Failures** | Entra ID | Failed authentication attempts (per user, per app, per location) | >10 failures/user/hour → Investigate (brute force) |
| **Risk Detections (Sign-in)** | Entra ID Protection | Sign-in risk level (low/medium/high) per sign-in | Medium/High risk → Block + require MFA/password change |
| **Risk Detections (User)** | Entra ID Protection | User risk level (per user) | High user risk → Block + require password reset |
| **PIM Elevations** | PIM | Active role activations (who, when, duration) | Unusual activation time or off-hours → Investigate |
| **Key Vault Access** | Key Vault | All operations (get, set, list, delete) per secret/key/cert | Sensitive secret access outside business hours → Investigate |
| **Failed Key Vault Access** | Key Vault | 401/403 errors per principal | >5 failures/principal/hour → Investigate (credential leak) |
| **SAS Token Usage** | Storage Analytics | Requests authenticated via SAS vs MI vs keys | Unexpected SAS usage (should be 0 in production) |
| **Account Key Usage** | Storage Analytics | Requests using account keys (should be 0 with MI) | Any account key usage → Migrate to MI |
| **Storage Access Patterns** | Storage Analytics | Read/write/delete operations per storage account | Anomalous access pattern → Defender for Storage alert |
| **Conditional Access Decisions** | Entra ID | Allow/Deny/Modify per policy per user | Deny spike → Investigate; Blocked MFA → Investigate |
| **SCIM Provisioning** | Entra ID | User provisioned/disabled/deleted per SaaS app | Provisioning failure → SaaS account orphaned |
| **Enterprise App Sign-ins** | Entra ID | Sign-in events per SaaS app | Sign-in spike → Possible attack; Zero sign-ins → Stale app |
| **Connector Health** | Application Proxy | Connector online/offline status | Connector offline → On-prem apps inaccessible |
| **Published App Access** | Application Proxy | Access count per published on-prem app | Zero access → App issue; Spike → Possible attack |
| **Service Principal Auth** | Entra ID | SP authentication events (success/fail) | Failed auth after secret/cert rotation → Credential issue |
| **App Registration Auth** | Entra ID | App auth events per App Registration | Failed auth → Secret/cert issue; Unusual pattern → Investigate |
| **Managed Identity Token** | IMDS | MI token acquisition (success/fail) | Token failures → MI configuration issue |
| **Federated Credential Auth** | Entra ID | Federated SSO auth events | Failures → Configuration issue (issuer, subject, etc.) |
| **Privileged Role Assignments** | Entra ID | RBAC role assignments (Owner, Contributor) | New Owner assignment → Investigate (over-privilege) |
| **Global Admin Count** | Entra ID | Number of Global Admin users | Any Global Admin (should be 0 + break-glass) |
| **Password Expiry** | Entra ID | Password expiry within 30 days | User password expiring → Notification (or disable password expiry for MI) |
| **MFA Registration** | Entra ID | Users without MFA enrolled | Users without MFA → Enforce Conditional Access |
| **Key Vault Certificate Expiry** | Key Vault | Certificates expiring within 30/60 days | Certificate expiring → Auto-renew or manual renewal |
| **SP Secret/Cert Expiry** | App Registration | Secrets/certs expiring within 30 days | Expiry approaching → Rotate secret/cert |
| **DDoS Protection (Storage)** | DDoS | Mitigated traffic volume on storage | Sustained mitigation → Investigate (DDoS attack) |
| **Defender for Storage** | Defender | Threat alerts for storage accounts | Ransomware-like behavior → Investigate immediately |
| **Cost per Service** | Cost Management | Compute/storage/identity cost per service | Cost spike → Analyze usage pattern |

---

## 8. DIAGNOSTIC SETTINGS

| Resource | Log Category | Destination |
|----------|-------------|------------|
| **Entra ID (Sign-ins)** | All sign-in logs (success, failure, conditional access) | Log Analytics + Entra ID Logs |
| **Entra ID (Audit)** | All configuration changes, role assignments, group membership | Log Analytics + Entra ID Logs |
| **Entra ID (Provisioning)** | SCIM provisioning events (user added/removed/updated) | Log Analytics + Entra ID Logs |
| **Key Vault (Audit)** | All operations on secrets, keys, certificates | Log Analytics + Storage |
| **Key Vault (HTTPS)** | HTTP request logs (status, latency, operations) | Log Analytics |
| **Managed Identity (IMDS)** | Token acquisition events (success, failure) | Log Analytics (via VM diagnostics) |
| **Application Proxy (Connector)** | Connector health events, authentication logs | Log Analytics + Event Logs |
| **Application Proxy (Published App)** | Access logs, authentication logs | Log Analytics + Event Logs |
| **Storage Analytics** | Authentication type, SAS usage, account key usage | Storage Analytics (log blob) |
| **Storage Access Logs** | All storage API requests (read, write, delete) | Storage Analytics (log blob) |
| **DDoS Protection** | Mitigation events, attack detection | Log Analytics + DDoS Metrics |
| **Defender for Storage** | Threat detection alerts, recommendations | Log Analytics + Defender alerts |
| **Defender for Identity** | Sign-in risk, user risk, privilege escalation | Log Analytics + Defender alerts |
| **PIM** | Role activation, assignment, eligibility events | Log Analytics + PIM Logs |
| **Conditional Access** | Policy evaluation results (allow, deny, modify) | Log Analytics + Entra ID Logs |
| **Activity Log (Entra ID)** | Entra ID configuration changes | Log Analytics + Storage |
| **SCIM Provisioning** | Provisioning events per SaaS app | Enterprise Application logs |
| **Application Gateway** | Access, WAF, performance, SSL | Log Analytics + Storage |
| **Load Balancer** | Health probes, SNAT, flow logs | Log Analytics + Storage |
| **NSG (Flow Logs)** | IP-level allow/deny decisions | Storage + Log Analytics |
| **Azure Monitor** | All metrics, alerts, autoscale events | Log Analytics + Alerts |
| **Cost Management** | Resource-level cost alerts, budget alerts | Cost Management + Email/Teams |

---

## 9. SECURITY CONSIDERATIONS

| Concern | Risk | Mitigation |
|---------|------|------------|
| **Shared admin accounts** | No individual accountability; credential sharing; impossible to trace actions | Individual accounts for all admins; PIM for privilege elevation; No sharing |
| **Global Admin accounts (50+)** | Catastrophic risk; any compromised GA = full tenant takeover | Zero GA (break-glass only); PIM; Just-in-time elevation; Emergency accounts only |
| **No MFA enforced** | Phishing succeeds; credentials compromised easily | Conditional Access: All users, all apps, MFA required (or FIDO2 for admins) |
| **Secrets in source code** | Credential leak; GitHub scanning finds secrets; Public repo exposure | All secrets in Key Vault; Pre-commit hooks; GitHub Secret Scanning; Secret scanning in CI/CD |
| **No key rotation** | Stale keys/certs; Exploitable indefinitely | Key Vault auto-rotation; Certificate CA integration; 90-day secret rotation; Alert on expiry |
| **Storage account keys in code** | Full storage access; If leaked: Complete data compromise | Use Managed Identity; Disable account keys; SAS (scoped, time-limited) for third-party |
| **App Registration with client secret** | Secret leak; Hard to rotate; No audit trail of usage | Use Managed Identity where possible; Certificate (not secret) for S2S; Federated credentials for CI/CD |
| **SAS token with full permissions** | Over-privileged; Can delete all data; Account-wide access | Use Service SAS (container/blob scope); Permission: read-only; Time-limited (hours/days) |
| **Key Vault public access** | Anyone with token can access secrets (no network isolation) | Private Endpoint; Firewall (Selected Networks); No public access in production |
| **Key Vault no soft delete** | Deleted secrets gone forever (no recovery) | Soft Delete: 90 days; Purge Protection: Enabled |
| **Stale service principals** | Old SPs with expired creds; Orphaned SPs; Privilege creep | Quarterly review; Disable after 90 days inactivity; Delete orphaned SPs |
| **Over-privileged service principals** | SP with Owner role; Full subscription control | Least privilege RBAC; Contributor (not Owner); Resource group scope (not subscription) |
| **No conditional access** | All devices, all locations, no MFA → High risk | Conditional Access: All users, MFA, compliant device, allowed locations |
| **Privileged users without PIM** | Permanent elevated access; Privilege creep; Compromised accounts | PIM: All privileged roles; Just-in-time elevation (4-hour max); Approval required |
| **Conditional Access not enforced on SaaS apps** | SaaS apps accessible without MFA; Risk of data breach | Apply Conditional Access to ALL enterprise applications (not just Azure) |
| **SCIM provisioning disabled** | Offboarded users retain SaaS access; Orphaned accounts | Enable SCIM on all critical SaaS apps; Auto-disable/offboard users |
| **Application Proxy without Conditional Access** | On-prem apps accessible without MFA; Risk from internet | Conditional Access enforced on all published apps (via Enterprise App policies) |
| **Identity provider (Entra ID) compromised** | Full identity takeover; All resources accessible | Password strength enforcement; Risk-based access; Anomaly detection; MFA |
| **Certificate-based auth (expired cert)** | Apps cannot authenticate; Downtime; Security risk (stale certs) | Auto-rotation (Key Vault CA); Alert on 30-day expiry; Quarterly cert inventory |
| **Legacy authentication protocols** | IMAP, POP3, SMTP basic auth bypass MFA; Insecure | Disable legacy auth (Conditional Access policy: Block); Modern auth only |
| **B2B guest users without restrictions** | External users with full access; Data leak risk | External Access settings: Restrict domains; Conditional Access for guests; Scope access |
| **App Proxy connector in unsecured network** | Connector compromised → On-prem app access via cloud | Connector in isolated network segment; Windows hardening; Regular patching; Monitoring |
| **Enterprise app with admin consent** | One-click consent: User grants broad permissions | Require admin consent for sensitive permissions; Restrict user consent (Azure policy) |
| **No access reviews** | Users accumulate permissions over time; Stale access | Periodic access reviews (Entra ID → Access Reviews); Quarterly for privileged; Semi-annual for general |
| **Federated credential misconfiguration** | External IdP trusts not verified; Token forgery possible | Verify issuer URL, subject format; Use HTTPS only; Monitor auth events |
| **Key Vault hsm key from unsupported SKU** | Key not backed by HSM; Weaker encryption | Use Premium SKU (HSM-backed keys); Standard SKU = software-backed |

---

## 10. L3 INTERVIEW QUESTIONS

### Basic
**Q: What is a Shared Access Signature (SAS)?**
A: A SAS is a secure token that grants delegated access to Azure Storage resources without exposing account keys. SAS tokens are scoped (container/blob level), time-limited, and permission-restricted. They provide fine-grained access control.

### Basic
**Q: What is the difference between account SAS and service SAS?**
A: Account SAS grants access to the entire storage account (broader). Service SAS grants access to specific containers or blobs within a storage account (more targeted). Best practice: Use service SAS (least privilege).

### Basic
**Q: What is Azure Key Vault?**
A: Azure Key Vault is a cloud-based service for securely storing encryption keys, secrets (passwords, API keys, connection strings), and certificates. It provides access control, auditing, and lifecycle management for cryptographic assets.

### Basic
**Q: What is a Managed Identity?**
A: A Managed Identity is an Azure resource identity that allows Azure services to authenticate to other Azure services without credentials in code. Azure manages the identity lifecycle. Types: System-Assigned (tied to resource) and User-Assigned (standalone, reusable).

### Basic
**Q: What is the difference between System-Assigned and User-Assigned Managed Identity?**
A: System-Assigned: Created with resource, deleted when resource deleted, unique per resource. User-Assigned: Standalone resource, reusable across multiple resources, persists independently. System-Assigned: Simpler setup. User-Assigned: More flexible, shared identity.

### Basic
**Q: What is an App Registration?**
A: App Registration (Application Object) in Entra ID defines an application's identity — its Application ID (Client ID), authentication methods, permissions (API permissions), and supported account types. It's the blueprint of an application.

### Basic
**Q: What is a Service Principal?**
A: Service Principal is the identity instance of an App Registration within a specific Entra ID tenant. It's used by applications to authenticate (via client credentials flow) to access Azure resources. App Registration = definition, Service Principal = instance.

### Basic
**Q: What is Conditional Access?**
A: Conditional Access is an Entra ID feature that enforces context-aware access decisions based on conditions like user risk, sign-in risk, device compliance, location, app, and client app. Controls: Require MFA, block access, require compliant device.

### Basic
**Q: What is Entra ID Privileged Identity Management (PIM)?**
A: PIM enables just-in-time, time-limited, approval-based activation of privileged roles (e.g., Global Admin). Instead of permanent elevated access, admins request activation (max 4-8 hours), which requires approval. Reduces permanent privilege exposure.

### Basic
**Q: What is Application Proxy?**
A: Application Proxy publishes on-premises applications via Entra ID, enabling secure remote access without VPN. Uses a connector (on-prem) and supports Seamless SSO. Users access on-prem apps via Entra ID URL (no public IP needed on-prem).

### Intermediate
**Q: What is the difference between App Registration and Service Principal?**
A: App Registration (Application Object): The definition of an application (Client ID, permissions, auth methods). Service Principal (Service Principal Object): The instance of that app in a specific tenant (Object ID, RBAC roles). One App Registration → multiple Service Principals (one per tenant).

### Intermediate
**Q: What is User Delegation SAS and why is it preferred over Account SAS?**
A: User Delegation SAS uses Azure AD (Entra ID) authentication (not storage account keys) to generate SAS tokens. Preferred because: No account keys exposed, revocable via Azure AD, auditable, Conditional Access applies, no stored access policy needed. Account SAS: Uses account keys, not revocable, not audited.

### Intermediate
**Q: What is the difference between Managed Identity and Service Principal?**
A: Managed Identity: Created/managed by Azure, lifecycle tied to resource (system) or standalone (user), no credential management needed, uses IMDS for token. Service Principal: Created manually, lifecycle independent, requires credential management (secret/cert), uses client credentials flow. Best practice: Prefer Managed Identity where possible.

### Intermediate
**Q: Explain the difference between Single-tenant and Multi-tenant App Registration.**
A: Single-tenant: App works only in one Entra ID tenant (one service principal). Multi-tenant: App works across multiple tenants (multiple service principals, one per tenant that consents). Multi-tenant: Other tenants must consent (admin or user) before using the app.

### Intermediate
**Q: What are the authentication methods supported by App Registrations?**
A: Client Credentials (secret/certificate), Authorization Code (interactive), PKCE (public clients), Device Code Flow, Integrated Windows Authentication, OAuth 2.0 On-Behalf-Of, and Federated Credentials (GitHub/Kubernetes/CI-CD).

### Intermediate
**Q: What is SCIM provisioning in Enterprise Applications?**
A: SCIM (System for Cross-domain Identity Management) is a standard for automated user provisioning/deprovisioning to SaaS apps via Entra ID. Flow: User created in Entra ID → Auto-provisioned in SaaS; User disabled → Auto-disabled in SaaS; User deleted → Auto-deleted in SaaS. Uses REST API with bearer token.

### L3
**Q: Explain the difference between soft delete, immutable blob storage, and legal hold in Azure Storage.**
A: Soft Delete: Recovers deleted blobs within retention period (e.g., 7 days) — protects against accidental deletion. Immutable Blob Storage: WORM (Write Once, Read Many) — data cannot be modified/deleted for specified duration — protects against malicious/ransomware deletion. Legal Hold: Indefinite hold (no expiry) — data cannot be deleted until explicitly released — protects against active legal matters. Key difference: Soft delete has time limit; Immutable and Legal Hold are more restrictive. Immutability: Cannot be disabled once set.

### L3
**Q: Explain how Key Vault soft delete and purge protection work together.**
A: Soft Delete: When Key Vault is deleted, it enters soft-deleted state (recoverable for 7-90 days). The vault is inaccessible but data is preserved. Purge Protection: Prevents purging/deleting data even after soft delete period expires. Without purge protection: After soft delete period, vault can be purged → Data lost forever. With purge protection: Data cannot be purged → Recoverable at any time. L3 Best Practice: Both enabled (soft delete + purge protection) on all production Key Vaults.

### L3
**Q: Explain how Managed Identity authentication works via IMDS.**
A: App on Azure VM → Calls IMDS endpoint (http://169.254.169.254/metadata/identity/oauth2/token) with parameters: api-version, resource (e.g., https://storage.azure.com). IMDS returns: Access token for specified resource (scoped). App uses token: Authorization: Bearer <token> → Calls target Azure service (Storage, SQL, etc.) → Service validates token with Entra ID → Access granted based on RBAC role assigned to MI. No credentials in code; Token is temporary (1-24 hours); Automatically refreshed.

**Q: Explain the authentication flow for a federated credential in App Registration.**

A: Federated credential enables passwordless, secretless authentication from external identity providers (IdPs) to Entra ID. Complete flow:

```
Federated Credential Authentication Flow:

  Step 1: External IdP Issues Token
    → GitHub Actions: Workflow run triggers
    → GitHub OIDC: Issues JWT subject token
      {
        "iss": "https://token.actions.githubusercontent.com",
        "sub": "repo:org/repo-name:ref:refs/heads/main",
        "aud": "api://AzureADTokenExchange",
        "repository": "org/repo-name",
        "ref": "refs/heads/main",
        "job_workflow_ref": "org/repo-name/.github/workflows/deploy.yml@main"
      }
    → Token signed by GitHub (private key); GitHub's public key available via JWKs endpoint

  Step 2: App Registration — Federated Credential Configuration
    → Portal: App Registration → Federated Credentials → Add
    → Configuration:
      - Issuer URL: https://token.actions.githubusercontent.com
        (or: https://token.actions.githubusercontent.com/org/repo-name
         for repo-scoped trust)
      - Subject Identifier: repo:org/repo-name:ref:refs/heads/main
        (or: repo:org/repo-name:* for branch wildcard)
      - Name: GitHub-AKS-Deploy
      - Audience: api://AzureADTokenExchange
        (or: api://<app-id> for app-specific)

  Step 3: Application Requests Token
    → App (in GitHub Actions): Calls Entra ID /token endpoint
    → Request:
      POST https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token
      Content-Type: application/x-www-form-urlencoded

      grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
      &client_id=<app-registration-app-id>
      &assertion=<GitHub-JWT-subject-token>
      &scope=https://management.azure.com/.default

    → No client_secret needed (federated = no credentials)

  Step 4: Entra ID Validates Token
    → Entra ID: Checks issuer URL (matches federated credential config)
    → Entra ID: Checks subject identifier (matches federated credential config)
    → Entra ID: Fetches GitHub JWKS endpoint (public keys)
    → Entra ID: Validates JWT signature (GitHub signature = valid)
    → Entra ID: Validates claims (audience, expiry, not-before)
    → Entra ID: Maps: Subject → Service Principal → App Registration

  Step 5: Entra ID Issues Access Token
    → Returns: Access token (for Azure Resource Manager or target resource)
      {
        "access_token": "<JWT>",
        "token_type": "Bearer",
        "expires_in": 3600
      }

  Step 6: App Uses Access Token
    → App: Authorization: Bearer <access_token>
    → Calls: Azure Resource Manager (deploy resources, etc.)
    → ARM validates: Entra ID token → SP has RBAC role → Access granted

  Trust Chain:
    GitHub (issues JWT) ←→ Entra ID (validates JWT) ←→ Azure (accepts token)
    App Registration (federated credential config) ←→ Defines trust between GitHub and Entra ID

  Key Points:
    → No client secret stored in app registration
    → No certificate uploaded to app registration
    → Trust is based on: Issuer URL + Subject Identifier + JWT signature
    → Granular: Can restrict by repo, branch, workflow
    → Secure: No credential rotation needed (GitHub handles JWT signing)
    → Auditable: All auth events in Entra ID sign-in logs
    → Revocable: Delete federated credential → Immediate auth failure

  Common Issues:
    → Subject identifier mismatch: GitHub repo/branch doesn't match config
    → Issuer URL wrong: Using wrong GitHub OIDC issuer
    → Audience mismatch: Token audience ≠ configured audience
    → JWKs fetch failure: Entra ID cannot download GitHub public keys
    → Permission: SP doesn't have RBAC role on target resources

  Debug Steps:
    1. Check: Sign-in logs → Entra ID (federated credential auth events)
    2. Check: Error message (token validation failure reason)
    3. Verify: Issuer URL matches GitHub OIDC issuer exactly
    4. Verify: Subject identifier matches GitHub workflow trigger
    5. Verify: Audience matches expected value
    6. Check: SP RBAC role on target resources
    7. Test: Manually decode JWT (jwt.ms) → Verify claims
```

---

### L3 Additional Questions:

**Q: How do you secure service-to-service communication in Azure without credentials?**

A: Use Managed Identity (MI) for all Azure resource-to-resource communication:

```
Secured S2S Without Credentials:

  VM (MI) → Key Vault: Token via IMDS → RBAC role "Key Vault Secrets User"
  App Service (MI) → Storage: Token via IMDS → RBAC role "Storage Blob Data Reader"
  AKS Pod (MI) → SQL DB: Token via Workload Identity → RBAC role "db_datareader"
  Function (MI) → Service Bus: Token via IMDS → RBAC role "Service Bus Sender"
  Logic App (MI) → Cosmos DB: Token via MI → RBAC role "Cosmos DB Contributor"

  Connection Strings:
    ❌ Bad: "Server=sql;Database=db;User=sa;Password=secret"
    ✅ Good: "Server=sql;Database=db;Authentication=Active Directory Managed Identity"
    → Token-based auth (no password in connection string)

  Key Points:
    → MI: No credential management (auto-lifecycle)
    → IMDS: Automatic token acquisition (SDK handles)
    → RBAC: Granular permissions (least privilege)
    → Audit: All MI auth logged (Entra ID sign-in logs)
    → No secrets: Zero credentials in code, config, or pipeline
    → Rotation: MI is auto-managed (no rotation needed)
```

---

**Q: Explain the difference between Key Vault access policies (v1) and RBAC (v2).**

A: Key Vault transitioned from access policies (v1) to RBAC (v2) for authorization. Key differences:

```
Access Policies (v1) vs RBAC (v2):

  Feature               v1 (Access Policies)     v2 (RBAC)
  ─────────────────────────────────────────────────────────────
  Principal Limit       1,024 per vault          Unlimited
  Scope                 Vault only               Vault / RG / Subscription
  Roles                 Templates (Secret Mgmt,  Built-in roles (Key Vault
                          Key Mgmt, Cert Mgmt)     Admin, Secrets Officer, etc.)
  Groups                Not supported            Supported
  Conditional Access    Not supported            Supported
  Audit Trail           Key Vault logs           Azure Activity Log + RBAC
  Standard              Proprietary              Azure-standard RBAC
  Deprecation           Legacy (still works)     Recommended (default for new vaults)
  Cross-Vault           Per-vault policy         RG/Subscription scope assignments
  Custom Roles          Not supported            Supported

  Migration Path:
    1. Assign RBAC roles to all principals (with v1 access)
    2. Verify: All access works with RBAC
    3. Switch: Configuration → Configuration → v2 (RBAC)
    4. Test: All access verified
    5. Note: Cannot revert to v1 after switching

  Key Vault RBAC Roles:
    → Key Vault Administrator: Full control
    → Key Vault Secrets Officer: Secret CRUD + backup/restore
    → Key Vault Secrets User: Secret read (get, list)
    → Key Vault Certificates Officer: Certificate CRUD
    → Key Vault Certificates User: Certificate read
    → Key Vault Crypto Officer: Encrypt/Decrypt/Sign/Verify
    → Key Vault Crypto User: Encrypt/Decrypt (keys only)
    → Key Vault Reader: Read-only (properties)
    → Key Vault Network Contributor: Network settings
    → Key Vault Purge Protection Contributor: Purge management
    → Key Vault Soft Delete Contributor: Soft delete management

  Best Practice: Use RBAC (v2) exclusively
    → No access policies (v1) on new vaults
    → Migrate existing vaults to v2
    → Use built-in roles where possible; custom roles for granular needs
```

---

**Q: How do you implement cross-tenant access using Entra ID?**

A: Cross-tenant access uses Entra ID B2B (Business-to-Business) for secure external collaboration:

```
Cross-Tenant Access Patterns:

  1. B2B Direct Collaboration:
     → Tenant A: User invites user from Tenant B
     → Tenant B: User accepts invitation
     → Result: Tenant B user has Guest role in Tenant A
     → Access: Can access resources (RBAC Guest role = limited by default)
     → Auth: Tenant B user authenticates with their own credentials
     → Conditional Access: Applied per policy (can restrict guest access)

  2. Multi-Tenant SaaS Application:
     → App Registration: Supported account types = "Multitenant"
     → Tenant A: User consents (admin or user-level)
     → Result: Service Principal created in Tenant A (for app)
     → Tenant B: User consents → SP created in Tenant B
     → Each tenant: Independent SP with independent RBAC
     → App: Uses tenant-specific tenant ID for auth

  3. B2B Collaboration with Groups:
     → Tenant A: Group "Collaborators" contains B2B guests from Tenant B
     → RBAC: Assign role to Group (not individual users)
     → Result: All guests in group inherit RBAC role
     → Management: Group-based access control (simpler than individual)

  4. Cross-Tenant MI (Advanced):
     → User-Assigned MI in Tenant A
     → RBAC: Assign MI role in Tenant B (target resource)
     → App (Tenant A): Uses MI → Authenticates to Tenant B resource
     → Process:
       1. Create UAI in Tenant A (app-prod-mi)
       2. Grant: UAI "Storage Blob Reader" on resource in Tenant B
       3. VM (Tenant A): Uses UAI → Token → Storage (Tenant B) → Access granted
     → Cross-tenant: Requires RBAC in target tenant (not same SP)

  5. Tenant Restrictions (Conditional Access):
     → Restrict which external tenants can access: Enterprise App → Settings
     → Include: Specific domains (contoso.com, fabrikam.com)
     → Exclude: All other domains
     → Effect: Only users from included domains can authenticate
     → Block: Unknown/untrusted external tenants

  6. B2B Collaboration Settings:
     → Azure AD → Settings → External Access → Collaboration settings
     → Allow: B2B collaboration (ON/OFF)
     → Allow: B2B direct connect (for Teams, etc.)
     → Block: Unnamed/unknown external users
     → Scope: Specific domains or all

  Security Considerations:
    → Guest access: Limited by default (RBAC Guest role)
    → Elevated Guest: Can be assigned higher roles (but risky)
    → Conditional Access: Apply to guests (MFA, compliant device, location)
    → Monitoring: Sign-in logs track guest authentication
    → Tenant restrictions: Limit which external tenants can access
    → PIM: Do NOT assign privileged roles to guests (risk of tenant takeover)
```

---

**Q: How do you implement a complete secret rotation strategy?**

A: Secret rotation ensures no credential is valid indefinitely:

```
Secret Rotation Strategy:

  1. Secrets (Connection Strings, Passwords, API Keys):
     → Storage: Azure Key Vault (all secrets centralized)
     → Rotation: Manual or automated (90-day cycle)
     → Process:
       Day 1: Generate new secret → Add to Key Vault (version 2)
       Day 1-3: Update app to use new secret (test in parallel)
       Day 3-7: Confirm app works with new secret (monitoring)
       Day 7: Delete old secret version from Key Vault
       → Automation: Key Vault event → Function → Auto-rotate + deploy

     Azure Key Vault Secret Rotation:
     → Key Vault: Secret with rotation policy
     → Policy: Every 90 days
     → Event: Secret rotation triggers Event Grid → Azure Function
     → Function: Creates new secret version, updates app config
     → Notification: Email to app owner (rotation completed)

  2. Certificates (TLS/SSL):
     → Storage: Azure Key Vault (certificates)
     → Rotation: Auto-renewal (CA integration)
     → Process:
       Day -90: Key Vault contacts CA (auto-renewal)
       Day -7: CA issues new certificate
       Day 0: New certificate published (new version in Key Vault)
       Event Grid: Notifies downstream (app restart, deployment)
       → App: Automatically uses latest certificate version
     → No manual intervention needed (if CA integration configured)

  3. Client Secrets (App Registration):
     → Storage: App Registration → Certificates & Secrets
     → Rotation: Quarterly (6-month maximum lifetime)
     → Process:
       Day 1: Generate new secret → Add to App Registration
       Day 1-3: Deploy app with new secret (CI/CD pipeline update)
       Day 3-7: Verify app works with new secret
       Day 7: Delete old secret from App Registration
       → Alert: 30 days before expiry (email + Teams notification)

  4. Account Keys (Storage Account — NOT recommended):
     → Strategy: Disable account keys entirely (use Managed Identity)
     → If account keys still required: Rotate key1 → Swap key1/key2 → Rotate key2 → Swap
     → Process:
       Step 1: Regenerate key1 → Apps updated with new key1
       Step 2: Verify: All apps use key1
       Step 3: Regenerate key2 (key1 no longer valid)
       Step 4: Verify: All apps use key2
       Step 5: Disable account keys (switch to MI entirely)

  5. Kubernetes Secrets (if applicable):
     → External Secrets Operator (ESO): Syncs K8s secrets from Key Vault
     → Rotation: Key Vault secret rotated → ESO syncs to K8s → Pod restarts
     → Process:
       Key Vault secret rotation → ESO detects change → Updates K8s secret → Pod refresh

  Rotation Monitoring:
    → Alert: Secret expires in 30 days (Key Vault: certificate expiry event)
    → Alert: SP secret expires in 30 days (App Registration: expiry notification)
    → Alert: Certificate expiry in 30 days (CA integration auto-renewal)
    → Dashboard: All secrets/certs expiring within 90 days (consolidated view)
    → Compliance: Audit that rotation occurred (log evidence)

  Automation Tools:
    → Key Vault rotation policy + Event Grid + Azure Function
    → GitHub Actions: Scheduled workflow (quarterly secret rotation)
    → Terraform: Manage secret lifecycle (version = terraform state)
    → Azure Policy: Enforce secret rotation (max age 90 days)
    → Defender for Secrets: Detect secrets in code (prevent manual management)
```

---

**Q: Explain the difference between Single Sign-On (SSO), Seamless SSO, and Password Hash Synchronization (PHS).**

A: These are three different SSO mechanisms:

```
SSO Comparison:

  1. SSO (Cloud-based, Standard Entra ID SSO):
     → User: Signs in once → Access multiple cloud apps
     → Mechanism: Entra ID issues tokens (OIDC/SAML) to all apps
     → Auth: User enters password once (per session)
     → Apps: SaaS apps (Salesforce, GitHub, etc.)
     → Requirement: Entra ID P1+ (SSO built-in)
     → Use: Cloud-native apps

  2. Seamless SSO (On-Prem SSO via Kerberos):
     → User: On-domain device → Access on-prem apps → No password prompt
     → Mechanism: Kerberos ticket → Entra ID validates → Auto-login
     → Auth: Uses on-prem AD Kerberos (no password sent to cloud)
     → Apps: On-prem web apps (via Application Proxy) + Azure AD-connected apps
     → Requirement: Entra Connect + Seamless SSO enabled + KCD
     → Use: Hybrid environment (on-prem + cloud)
     → Key: User never types password (auto-login via Kerberos delegation)
     → Security: Password never leaves on-prem network

  3. Password Hash Synchronization (PHS):
     → User: Same password for on-prem AD and Entra ID
     → Mechanism: Entra Connect syncs password hash → Entra ID
     → Auth: User signs in to Entra ID with on-prem password (hash match)
     → Apps: All Entra ID apps (cloud + on-prem via Application Proxy)
     → Requirement: Entra Connect with PHS enabled
     → Use: Simple hybrid (single password, no separate auth)
     → Security: Hash synced (not plaintext password)
     → Note: Alternative to federation (ADFS) or Seamless SSO

  Comparison Table:
    Feature           PHS              Seamless SSO        Federation (ADFS)
    ─────────────────────────────────────────────────────────────────
    Password          Same (synced)    Same (synced)       Separate (ADFS)
    On-prem Auth      Hash sync        Kerberos            ADFS
    Cloud Auth        Hash match       Token (SSO)         Token (ADFS)
    MFA               Entra ID CA      Entra ID CA         ADFS + MFA
    Complexity        Low              Medium              High
    Internet req      Yes (cloud auth) Yes (cloud auth)    No (on-prem ADFS)
    Disaster Rec      Entra ID (cloud) Entra ID (cloud)    ADFS (on-prem)
    Use Case          Simple hybrid    Seamless auto-login Enterprise federation

  Best Practice:
    → PHS + Seamless SSO: Combined (most common in hybrid environments)
    → Seamless SSO: Auto-login on-domain devices
    → PHS: Password sync for off-domain/mobile devices
    → Neither: Use PHS + Seamless SSO (simpler than ADFS federation)
    → ADFS: Only for specific federation requirements (not recommended for new deployments)
```

---

**Q: How do you implement a comprehensive monitoring strategy for identity and access security?**

A: Comprehensive monitoring requires correlation across multiple data sources:

```
Identity & Access Monitoring Strategy:

  Data Sources:
    1. Entra ID Sign-in Logs: All authentication events (success/failure)
    2. Entra ID Audit Logs: All configuration changes
    3. Key Vault Audit Logs: All secret/key/cert operations
    4. Activity Logs: All Azure resource operations
    5. Storage Analytics: SAS/key/auth-type usage
    6. Defender for Identity: Risk detections (sign-in/user risk)
    7. Microsoft Sentinel: SIEM/SOAR (correlation, analytics)
    8. Application Proxy Logs: Published app access
    9. SCIM Provisioning Logs: SaaS user lifecycle events

  Dashboards (Consolidated):
    → Real-time: Sign-in volume (per minute)
    → Real-time: Failed auth attempts (per user, per app)
    → Real-time: Conditional Access decisions (allow/deny/block)
    → Real-time: Key Vault access (secret reads, writes, deletes)
    → Real-time: Storage auth types (MI vs SAS vs Key — alert if Key used)
    → Trend: Sign-in patterns (baseline vs anomaly)
    → Trend: PIM activations (count, duration, justification)
    → Trend: SCIM provisioning (users provisioned per day)
    → Alert: Risk detections (high-risk sign-ins)
    → Alert: Stale SP detection (no activity > 90 days)

  Analytics (Microsoft Sentinel):
    → Analytics Rule 1: Impossible travel (user signs in from two distant locations within short time)
    → Analytics Rule 2: Brute force (many failed attempts from same IP)
    → Analytics Rule 3: Unusual authentication (user never used before, from new device/IP)
    → Analytics Rule 4: Excessive privileges (new Owner/Contributor assignment)
    → Analytics Rule 5: Secret leak (secret/cert expiry approaching without rotation)
    → Analytics Rule 6: Storage account key usage (should be 0 in production)
    → Analytics Rule 7: Global Admin sign-in (should be extremely rare)
    → Analytics Rule 8: Off-hours privileged access (admin access at 2 AM)
    → Analytics Rule 9: SAS token generation (should be 0 in production)
    → Analytics Rule 10: Failed MI token acquisition (MI not working)

  Playbooks (Microsoft Sentinel SOAR):
    → Playbook 1: High-risk sign-in → Block user → Email security team → Create incident
    → Playbook 2: Brute force detected → Block source IP → Alert user → Require MFA reset
    → Playbook 3: Secret leak detected → Rotate secret automatically → Notify app owner
    → Playbook 4: Stale SP detected → Disable SP → Notify owner → Create review ticket
    → Playbook 5: Excessive privilege → Revert to least privilege → Notify manager
    → Playbook 6: Storage key usage → Alert → Investigate → Migrate to MI

  Reporting (Weekly/Monthly):
    → Weekly: Sign-in summary, failed auth summary, PIM activations
    → Monthly: Access review completion, stale SP report, permission changes
    → Quarterly: Compliance report (MFA coverage, Conditional Access, PIM)
    → Annual: Security posture assessment (identity security score)
```

---

**Q: Explain how to design a multi-region identity architecture.**

A: Multi-region identity ensures high availability and compliance with data residency:

```
Multi-Region Identity Architecture:

  Entra ID (Global Service):
    → Entra ID is global (not region-specific)
    → Authentication: Served from nearest Azure region (low latency)
    → Tenant: Single tenant, global (all users worldwide)
    → Data: User data replicated across regions (not stored in specific region)
    → Compliance: Entra ID complies with regional data residency requirements

  Key Vault (Regional):
    → Per Region: Separate Key Vault (kv-contoso-{region})
    → Replication: Not replicated (regional resource)
    → Access: Each region's resources access their regional KV
    → Cross-region: VNet peering + Private Link (or replicate via Backup)

  Managed Identity (Regional):
    → MI is regional (attached to regional resources)
    → VM in East US: System MI → Authenticates to East US resources
    → VM in West US: System MI → Authenticates to West US resources
    → Cross-region: User-Assigned MI can be assigned cross-region

  Storage (Regional):
    → Per Region: Separate Storage Account (storage-contoso-{region})
    → Replication: GRS (Geo-Redundant Storage) for DR
    → Access: Regional resources access their regional storage (via MI)

  Application Proxy (Regional Connector Groups):
    → Per Datacenter: Separate Connector Group
    → Connectors: On-prem servers in each region
    → Routing: User → Nearest connector (based on connector health)

  Design Pattern:
    ┌─ Primary Region (East US) ───────────────────────────────┐
    │  App Service (East US) → Key Vault (East US)              │
    │  → MI: System-assigned → RBAC: KV Secrets Officer (East)  │
    │  → Storage: East US → MI: Blob Data Reader                │
    │  → SQL DB: East US → Entra ID Admin                       │
    └───────────────────────────────────────────────────────────┘

    ┌─ Secondary Region (West US) ────────────────────────────┐
    │  App Service (West US) → Key Vault (West US)              │
    │  → MI: System-assigned → RBAC: KV Secrets Officer (West)  │
    │  → Storage: West US → MI: Blob Data Reader                │
    │  → SQL DB: West US → Entra ID Admin (auto-failover)       │
    └───────────────────────────────────────────────────────────┘

    ┌─ DR Region (North Central US) ──────────────────────────┐
    │  Storage: GRS (replicated from East US)                   │
    │  Key Vault: Backup (weekly, encrypted)                    │
    │  SQL DB: Auto-failover group                              │
    │  → Not active until failover event                        │
    └───────────────────────────────────────────────────────────┘

  Key Considerations:
    → Entra ID: Single tenant (global) — no regional split
    → Key Vault: Regional — ensure each app accesses regional KV
    → Storage: Regional primary + GRS for DR
    → MI: Regional (system-assigned) or cross-region (user-assigned)
    → App Proxy: Connector groups per region
    → Conditional Access: Global (applies to all regions)
    → Monitoring: Per-region dashboards (regional auth patterns)
    → Latency: Nearest region for auth (Entra ID serves globally)
    → Compliance: Data stored in specific region (KV, Storage)

  Disaster Recovery:
    → Entra ID: Regional failover (automatic, Microsoft-managed)
    → Key Vault: Backup/restore (weekly backup to geo-redundant storage)
    → Storage: GRS/RA-GRS (automatic cross-region replication)
    → SQL DB: Auto-failover groups (automatic DR)
    → App Proxy: Connector health monitoring → Auto-route to healthy connector
```

---

**Q: How do you implement Zero Trust for a hybrid environment (on-prem + cloud)?**

A: Zero Trust in hybrid requires consistent policies across on-prem and cloud:

```
Zero Trust Hybrid Architecture:

  Identity Layer (Unified):
    → Entra Connect: Sync on-prem AD → Entra ID (single identity source)
    → Conditional Access: Apply to cloud AND on-prem apps
    → MFA: Required for all users (cloud and on-prem)
    → PIM: All privileged roles (on-prem admin and cloud admin)
    → Seamless SSO: On-prem auto-login (via Kerberos delegation)

  Device Layer (Unified):
    → Intune: Device compliance for cloud apps (conditional access)
    → On-prem: Device compliance checked via Entra ID (if hybrid)
    → BYOD: Conditional Access (limited access, managed MDM)
    → Corporate: Full access (compliant + MFA)
    → Personal: Blocked (or limited access)

  Network Layer (Unified):
    → Cloud: Private Endpoints (no public access)
    → On-prem: No inbound internet (firewall rules)
    → Application Proxy: On-prem apps published via Entra ID (no VPN)
    → VPN: Not required (Application Proxy replaces VPN for apps)
    → Bastion: VM access via Entra ID (no public IPs on VMs)
    → ExpressRoute/VPN: Private connectivity (on-prem ↔ Azure)

  Application Layer:
    → Cloud apps: Entra ID authentication (no anonymous)
    → On-prem apps: Application Proxy + Entra ID auth
    → All apps: Conditional Access enforced
    → API apps: Entra ID auth (client credentials or user token)

  Data Layer:
    → Cloud data: Encrypted (CMK), access via MI, soft delete enabled
    → On-prem data: Encrypted (BitLocker), access via RBAC
    → Shared data: Replicated via Entra ID + Storage (GRS)
    → Classification: Sensitivity labels applied (cloud + on-prem)

  Monitoring Layer:
    → Entra ID: Sign-in logs (cloud + on-prem via Application Proxy)
    → On-prem: Events forwarded to Sentinel (via Log Analytics agent)
    → Key Vault: All access logged (regional)
    → Sentinel: Unified correlation (cloud + on-prem events)

  Implementation Steps:
    1. Entra Connect: Sync on-prem AD → Entra ID (seamless SSO)
    2. Conditional Access: All users, all apps, MFA required
    3. Application Proxy: Publish critical on-prem apps
    4. Private Endpoints: All cloud PaaS (no public access)
    5. Managed Identity: All Azure resources (no credentials)
    6. Key Vault: All secrets centralized (soft delete + purge)
    7. PIM: All privileged roles (just-in-time)
    8. Sentinel: All logs correlated (cloud + on-prem)
    9. Device compliance: Intune + Conditional Access
    10. Review: Quarterly access reviews (cloud + on-prem)

  Key Difference from Cloud-Only:
    → On-prem apps need Application Proxy (to expose via Entra ID)
    → Seamless SSO required (auto-login for on-prem users)
    → Device compliance: May need on-prem MDM + Intune hybrid
    → Network: On-prem connectivity to cloud (VPN/ExpressRoute)
    → Identity: Entra Connect sync (single source of truth)
```

---

## 11. PRACTICAL SCENARIO WALKTHROUGHS

### Scenario A: Migrating from Account Keys to Managed Identity for Storage Access

```
Migration Plan (Account Keys → Managed Identity):

  Current State:
    → App: Web App (app-prod) accessing Storage Account (storage-contoso)
    → Auth: Account Key (in App Settings: "StorageKey")
    → Risk: Key exposed in config, no rotation, full access

  Migration Steps:
    Step 1: Create System MI
      → Portal: Web App → Identity → System-Assigned → On
      → Wait: Provisioning (2-5 minutes)
      → Verify: Identity → Principal ID (note for RBAC)

    Step 2: Grant RBAC Role
      → Portal: Storage Account → Access Control (IAM) → Add → Add Role Assignment
      → Role: Storage Blob Data Reader (or Contributor for full access)
      → Assign access to: Managed Identity → Select web-app-prod
      → Scope: Storage Account (or Container for finer control)
      → Save: Wait for propagation (5-10 minutes)

    Step 3: Update App Code
      → Remove: Storage account key from App Settings
      → Update: Connection string (remove AccountKey)
      → Use: Azure SDK with MI credential (no key needed)
        Python:
          from azure.storage.blob import BlobServiceClient
          client = BlobServiceClient(
              account_url="https://storage.blob.core.windows.net",
              credential=DefaultAzureCredential()  # Auto-uses MI
          )
        C#:
          var client = new BlobServiceClient(
              new Uri("https://storage.blob.core.windows.net"),
              new ManagedIdentityCredential()
          );

    Step 4: Test
      → Deploy: Updated app
      → Verify: App can access Storage (read/write)
      → Verify: App Settings → No Storage key present
      → Verify: Log Analytics → MI token acquisition events

    Step 5: Disable Account Keys
      → Portal: Storage Account → Access Keys → Select key → Disable
      → Both keys: Disabled (not deleted — keep for emergency)
      → Monitor: Storage Analytics → Ensure no key auth (should be 0%)
      → After 2 weeks: Delete keys (if no usage)

    Step 6: Monitor
      → Defender for Storage: No key auth alerts
      → Storage Analytics: Auth type = MI (100%)
      → Key Vault: If CMK also configured → Double security

  Result:
    → No credentials in code/config
    → No key rotation needed (MI is auto-managed)
    → Full audit trail (MI auth logged in Entra ID)
    → Automatic token refresh (no session management)
    → Least privilege (RBAC role scoped to specific storage account)
```

### Scenario B: Publishing On-Prem App via Application Proxy with Conditional Access

```
Scenario: HR Portal (on-prem) published via Application Proxy

  Current State:
    → App: HR Portal (http://hr-app.internal.contoso.com:8080)
    → Access: VPN required, no Conditional Access, no SSO
    → Auth: Windows Integrated Auth (Kerberos)

  Target State:
    → App: Published via Application Proxy
    → Access: Internet (no VPN), Conditional Access enforced, SSO

  Implementation:
    Step 1: Install Connector
      → Download: Portal → Entra ID → Application Proxy → Download Connector
      → Install: On Windows Server (in same domain as HR app)
      → Register: Sign in with global admin
      → Verify: Status = Connected

    Step 2: Publish App
      → Display Name: HR Portal (Published via App Proxy)
      → Internal URL: http://hr-app.internal.contoso.com:8080
      → External URL: https://hr.contoso.com (custom domain + SSL)
      → Authentication: Seamless SSO + Integrated Windows Auth
      → Connector Group: OnPrem-Connectors
      → Session: Persistent (for Kerberos/SPN)
      → Pre-Auth: Entra ID (verified)

    Step 3: Assign Access
      → Users and Groups: HR-Users group (Entra ID group)
      → Conditional Access: Apply policy (MFA required for HR app)

    Step 4: Configure Conditional Access
      → Policy: "HR Portal Access"
        Users: HR-Users group
        Apps: HR Portal (published app)
        Grant: Require MFA, compliant device
        Session: Sign-in frequency 4 hours
        Condition: Block legacy authentication

    Step 5: Test Access
      → On-domain, corporate network: User → hr.contoso.com → Seamless SSO → No prompt → HR Portal loaded
      → On-domain, VPN: User → hr.contoso.com → Seamless SSO → No prompt → HR Portal loaded
      → Off-network, MFA enrolled: User → hr.contoso.com → MFA challenge → HR Portal loaded
      → Non-compliant device: User → hr.contoso.com → Access denied (compliant device required)
      → Unauthorized user: User → hr.contoso.com → Access denied (not in HR-Users group)

  Result:
    → No VPN required for HR Portal access
    → Conditional Access enforced (MFA + compliant device)
    → Seamless SSO (no password prompt for domain-joined users)
    → Application Proxy connector (outbound only — no inbound firewall changes)
    → Full audit trail (Application Proxy logs + Entra ID sign-in logs)
```

### Scenario C: Setting Up User Delegation SAS for Third-Party Sharing

```
Scenario: Share read-only blob container with external partner

  Requirement:
    → External company (partner-contoso.com) needs read-only access to specific container
    → Time-limited: 7 days
    → No account keys shared

  Solution: User Delegation SAS (Azure AD-based)

  Step 1: Configure Storage Account
    → Storage Account → Configuration → "Allow Azure AD-based access" = Enabled
    → Storage Account → Networking → Selected Networks (no public access)
    → Remove: Account keys (disable — use MI only)

  Step 2: Create Entra ID User/Group for Partner
    → Create: Entra ID user (partner-user@contoso.com) — or use existing
    → OR: Create: Entra ID group (storage-readers-partner)

  Step 3: Grant RBAC Role
    → Storage Account → Access Control (IAM) → Add Role Assignment
    → Role: Storage Blob Data Reader
    → Assign to: partner-user (or group)
    → Scope: Storage Account (or specific container)
    → Save: Wait for propagation

  Step 4: Generate User Delegation SAS
    → Using Azure CLI:
      az storage container generate-sas \
        --account-name storagecontoso \
        --name partner-container \
        --auth-mode login \
        --permissions rl \
        --expiry 2024-10-22T00:00:00Z \
        --output tsv
      → --auth-mode login: Uses Azure AD auth (not account key)
      → --permissions rl: Read + List only
      → --expiry: 7 days from now

    → Using Azure Portal:
      → Storage Account → Shared Access Signature
      → Permissions: Read, List
      → Expiry: 7 days
      → Generate SAS token

  Step 5: Share SAS URL
    → SAS URL: https://storagecontoso.blob.core.windows.net/partner-container?sv=2022-11-02&ss=b&srt=sco&sp=rl&se=2024-10-22T00:00:00Z&sr=c&sig=...
    → Share: With partner (via secure channel — email encrypted, Teams, etc.)
    → Partner: Uses SAS URL (read-only, no write/delete)

  Step 6: Monitor & Revoke
    → Storage Analytics: Monitor SAS usage (who accessed what)
    → Revoke: Remove RBAC role from partner-user → SAS becomes invalid
    → Expiry: After 7 days, SAS auto-expires (no action needed)
    → Audit: All access logged (Storage Analytics + Defender)

  Result:
    → No account keys shared
    → Time-limited (7 days)
    → Permission-restricted (read-only)
    → Revocable (remove RBAC role)
    → Auditable (all access logged)
    → No long-term commitment
```

### Scenario D: Complete Application Proxy with Seamless SSO and Kerberos Delegation

```
Scenario: Internal HR Portal (on-prem) with Kerberos auth published via Application Proxy

  Environment:
    → Domain: contoso.com (on-prem AD)
    → Entra ID: globalcontoso.onmicrosoft.com
    → Entra Connect: Syncing users (contoso.com → Entra ID)
    → App: HR Portal (http://hr-app.internal.contoso.com:8080)
    → Auth: Kerberos (SPN: HTTP/hr-app.internal.contoso.com)

  Step 1: Configure Entra Connect Seamless SSO
    → Entra Connect: Enable Seamless SSO
    → Service: Entra Connect service (on DC)
    → KCD: Kerberos-based, uses computer account (ENTRA CONNECT$)
    → DNS: Entra Connect DNS (internal DNS)
    → Verify: Seamless SSO: Enabled (Portal → Entra ID → Seamless SSO)

  Step 2: Install Application Proxy Connector
    → Server: On-prem Windows Server (in contoso.com domain)
    → Download: Portal → App Proxy → Download Connector
    → Install: Run installer, sign in with global admin
    → Verify: Connector status = Connected (Portal → App Proxy → Connectors)
    → Connector Group: Add to "OnPrem-HR" group

  Step 3: Configure Kerberos for Application Proxy
    → Application Proxy → Connector settings:
      → Enable Kerberos constrained delegation (KCD)
      → Connector uses: CONTOSO\Entra Connect$ computer account
      → Delegation: SPN for HR Portal (HTTP/hr-app.internal.contoso.com)
    → Active Directory:
      → Computer account: CONTOSO\Entra Connect$
      → Delegation: Trust for delegation to HTTP/hr-app.internal.contoso.com
      → Protocol: Kerberos (constrained)

  Step 4: Publish HR Portal
    → Display Name: HR Portal (Published)
    → Internal URL: http://hr-app.internal.contoso.com:8080
    → External URL: https://hr.contoso.com (custom SSL cert)
    → Authentication: Seamless SSO + Integrated Windows Auth
    → Connector Group: OnPrem-HR
    → Session: Persistent (required for Kerberos)
    → Pre-Auth: Entra ID

  Step 5: Test Kerberos Flow
    → User (on-domain): Opens browser → hr.contoso.com
    → Entra ID: Seamless SSO check → Kerberos ticket available
    → Entra ID: Validates Kerberos ticket → User authenticated (no password prompt)
    → Application Proxy: Routes request → Connector (OnPrem-HR)
    → Connector: Kerberos delegation → HR Portal (http://hr-app:8080)
    → HR Portal: Kerberos ticket forwarded (KCD) → User authenticated (no re-auth)
    → HR Portal: Returns data → Connector → Entra ID → User
    → User experience: Zero prompts (fully seamless)

  Step 6: Conditional Access
    → Policy: "HR Portal"
      → Users: All HR users
      → Apps: HR Portal (published app)
      → Grant: MFA (for external users), Seamless SSO (for on-domain)
      → Session: 4 hours

  Result:
    → Zero prompt for on-domain users (seamless SSO + KCD)
    → Conditional Access enforced (MFA for external/unmanaged)
    → Kerberos delegation through Application Proxy (KCD)
    → No password sent to cloud (Kerberos stays on-prem)
    → Full audit trail (Application Proxy + Entra ID logs)
```

---

## 12. ARCHITECTURE DECISION RECORDS (ADRS)

| Decision | Rationale | Alternatives Considered | Impact |
|----------|-----------|------------------------|--------|
| **Use RBAC (v2) for Key Vault** | v1 (Access Policies) limited to 1,024 principals; RBAC supports unlimited; consistent with Azure RBAC standard | Keep v1 (legacy); Hybrid (v1 + v2 during transition) | All Key Vault access uses RBAC; No access policy limit; Consistent permission model |
| **Use Managed Identity over Service Principal for Azure resources** | No credential management; Auto-lifecycle; IMDS-based; Full audit trail; Zero secret rotation | Service Principal with secret rotation; Client Certificate in Key Vault; Account Keys | No secrets in code; Automatic token refresh; Lower operational overhead |
| **Use User-Assigned MI for shared identity across resources** | Single MI for 100+ VMs (cost efficiency); Reusable across resource groups; Pre-provisioned (faster deployment) | System-Assigned MI per VM (100+ MIs, higher cost); Service Principal (manual management) | Shared identity; Centralized RBAC; Separate billing for MI resource |
| **Use User Delegation SAS instead of Account SAS** | Azure AD-based (not key-based); Revocable via RBAC; Auditable; Conditional Access applies | Account SAS (key-based); Shared Access Policy (non-Azure); Account Keys (insecure) | No account key exposure; SAS revocation via RBAC; Azure AD audit trail |
| **Use Application Proxy instead of VPN for on-prem access** | No VPN client; Conditional Access enforced; Seamless SSO; No on-prem firewall changes; Centralized access control | VPN (full network access); Bastion (VM only); Reverse proxy (load balancer); Direct web exposure (insecure) | On-prem apps accessible via Entra ID; MFA enforced; No VPN infrastructure |
| **Use SCIM provisioning for SaaS apps** | Automated user lifecycle; No orphaned accounts; Real-time provisioning/deprovisioning; Reduced admin workload | Manual provisioning (error-prone); Scheduled sync (delayed); No provisioning (security risk) | Zero orphaned accounts; Instant offboarding; Reduced admin effort |
| **Use Conditional Access for all SaaS apps** | Unified security policy; MFA enforced; Device compliance; Location-based restrictions; Consistent with cloud security | Per-app Conditional Access (complex); No Conditional Access (insecure); Per-user MFA (inconsistent) | All SaaS access protected; Consistent security posture; Reduced data breach risk |
| **Use Federated Credentials for CI/CD** | No secret in code; No cert in Key Vault; External IdP validates (GitHub/K8s); Automatic rotation (IdP managed) | Client Secret (secret in DevOps); Certificate in Key Vault (rotation needed); IP-based restrictions (insecure) | Zero credentials in code; Automatic rotation; Auditable; Revocable |
| **Disable Storage Account Keys** | Forces MI-based auth (most secure); Eliminates key leak risk; Full audit trail; No key rotation | Keep keys disabled (recommended); Keep keys active (risk); Use SAS only (partial security) | 100% MI-based auth; Zero key leak risk; Complete audit trail |
| **Use PIM for all privileged roles** | Just-in-time elevation; Time-limited (4-hour max); Approval required; Full audit trail; Reduces permanent privilege exposure | Permanent admin (risk); Break-glass only (operational overhead); No PIM (compliance failure) | Zero permanent privileged access; Full audit trail; Compliance with security standards |

---

## 13. REFERENCE ARCHITECTURE DIAGRAMS

### Complete Identity Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                             COMPLETE IDENTITY ARCHITECTURE                           │
│                                                                                     │
│  ┌─ IDENTITY PROVIDER ──────────────────────────────────────────────────────────┐   │
│  │                                                                               │   │
│  │  Microsoft Entra ID (globalcontoso.onmicrosoft.com)                            │   │
│  │  ├── Users (50K) — MFA, Conditional Access, PIM                                │   │
│  │  ├── Groups (500) — RBAC assignments                                          │   │
│  │  ├── App Registrations (200) — App definitions                                 │   │
│  │  ├── Service Principals (200) — App instances                                  │   │
│  │  ├── Enterprise Applications (50 SaaS) — SSO + SCIM                           │   │
│  │  ├── Application Proxy (100 published apps) — On-prem access                   │   │
│  │  └── PIM — Privileged role management                                          │   │
│  │                                                                               │   │
│  └───────────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                                │
│                                    ▼                                                │
│  ┌─ AUTHORIZATION LAYER ──────────────────────────────────────────────────────┐    │
│  │                                                                             │    │
│  │  RBAC (Resource Manager)                                                  │    │
│  │  ├── Built-in Roles (Owner, Contributor, Reader, etc.)                     │    │
│  │  ├── Custom Roles (granular permissions)                                   │    │
│  │  ├── Scope: Subscription / RG / Resource                                   │    │
│  │  └── Conditional Access: MFA + Device + Location + Risk                    │    │
│  │                                                                             │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│           │                    │                    │                               │
│           ▼                    ▼                    ▼                               │
│  ┌─ SECRETS LAYER ──┐  ┌─ COMPUTE ──┐  ┌─ DATA LAYER ──┐                        │
│  │                  │  │            │  │                │                         │
│  │ Key Vault (×5)   │  │ VMs (100)  │  │ Storage (50)   │                         │
│  │ System MI        │──│ MI auth    │──│ MI auth        │                         │
│  │ Secrets, Keys    │  │ App Service│  │ Soft Delete    │                         │
│  │ Certificates     │  │ AKS (5)    │  │ Immutable      │                         │
│  │ ├─ Soft Delete   │  │ Function   │  │ Versioning     │                         │
│  │ ├─ Purge Prot.   │  │ Logic App  │  │ Private Endpt  │                         │
│  │ ├─ Private Endpt │  │ ACI (20)   │  │ Defender       │                         │
│  │ └─ RBAC (v2)     │  │ All MI     │  │ SAS (0 in prod)│                         │
│  │                  │  │            │  │ Account Key:Disabled│                     │
│  └──────────────────┘  └────────────┘  └────────────────┘                         │
│                                                                                     │
│  ┌─ ON-PREM ACCESS ──────────────────────────────────────────────────────────┐    │
│  │                                                                           │    │
│  │  Application Proxy (100+ published apps)                                  │    │
│  │  ├── Connectors (per datacenter): OnPrem-WestUS, OnPrem-EastUS            │    │
│  │  ├── Seamless SSO (Kerberos constrained delegation)                       │    │
│  │  ├── Conditional Access: MFA + compliant device                            │    │
│  │  └── Published apps: HR, ERP, CRM, etc.                                   │    │
│  │                                                                           │    │
│  │  Entra Connect: On-prem AD → Entra ID sync                                │    │
│  │  ├── Password Hash Sync (PHS)                                              │    │
│  │  ├── Seamless SSO (Kerberos)                                               │    │
│  │  └── Device sync (if Intune hybrid)                                        │    │
│  │                                                                           │    │
│  └───────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
│  ┌─ MONITORING ──────────────────────────────────────────────────────────────┐    │
│  │                                                                           │    │
│  │  Microsoft Sentinel (SIEM/SOAR)                                            │    │
│  │  ├── Sign-in logs (Entra ID) — All auth events                             │    │
│  │  ├── Key Vault logs — All secret/key/cert operations                       │    │
│  │  ├── Activity logs — All Azure resource operations                         │    │
│  │  ├── Storage Analytics — SAS/key/auth-type usage                           │    │
│  │  ├── Defender for Identity — Risk detections                               │    │
│  │  ├── Application Proxy logs — Published app access                         │    │
│  │  └── Playbooks (SOAR) — Automated response                                 │    │
│  │                                                                           │    │
│  │  Alerts:                                                                    │    │
│  │  → High-risk sign-in → Block + notify                                     │    │
│  │  → Key Vault 403s → Investigate (credential leak)                         │    │
│  │  → Storage key usage → Alert (should be 0%)                               │    │
│  │  → Stale SP → Disable + review                                            │    │
│  │  → Global Admin sign-in → Alert (should be 0)                             │    │
│  │  → Failed MI auth → Investigate (permission issue)                        │    │
│  │                                                                           │    │
│  └───────────────────────────────────────────────────────────────────────────┘    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 14. CLOSING SUMMARY

```
MODULE 14 KEY TAKEAWAYS:

  1. IDENTITY IS THE PERIMETER: In cloud-native security, identity (Entra ID) replaces
     network perimeter as the primary security boundary. Protecting identity = protecting
     everything. MFA + Conditional Access + PIM = foundational controls.

  2. MANAGED IDENTITY > SERVICE PRINCIPAL: Always prefer Managed Identity over Service
     Principals for Azure resources. No credentials to manage, no rotation, full audit
     trail, automatic lifecycle. Use SPs only for external applications.

  3. KEY VAULT = SINGLE SOURCE OF TRUTH: All secrets, keys, and certificates in Key Vault
     with soft delete, purge protection, private endpoint, and RBAC (v2). Never in code,
     config, or environment variables.

  4. SAS TOKENS ARE TEMPORARY BRIDGES: Use User Delegation SAS (Azure AD-based) when SAS
     is required. Never use account keys. Prefer MI-based auth over any SAS type.

  5. STORAGE DATA PROTECTION IS LAYERED: CMK + Soft Delete + Immutable + Versioning +
     Firewall + Private Endpoint + Defender + lifecycle management. Each layer addresses
     different threats (accidental, malicious, ransomware, compliance).

  6. APP REGISTRATIONS ≠ SERVICE PRINCIPALS: App Registration = blueprint. Service
     Principal = instance. Understanding this distinction is critical for troubleshooting
     and security management.

  7. ENTERPRISE APPLICATIONS = SaaS GATEWAYS: SSO + SCIM + Conditional Access =
     secure SaaS access management. Always enable SCIM for user lifecycle.

  8. APPLICATION PROXY = SECURE ON-PREM BRIDGE: No VPN needed. Conditional Access
     enforced. Seamless SSO (Kerberos). Reverse proxy pattern (outbound only).

  9. ZERO TRUST = EVERY LAYER VERIFIED: Identity → Device → Network → App → Data.
     Each layer independently verified. No implicit trust from any layer.

  10. MONITOR EVERYTHING: Entra ID logs, Key Vault logs, Storage logs, Defender,
      Sentinel. Correlation is key. Alerts enable response. Playbooks enable automation.

  MEMORY AID (SAS-S):
    S — Secrets in Key Vault (never in code)
    A — Access via RBAC (least privilege)
    S — Storage protected (CMK + Soft Delete + Immutable)
    — — — — — — — — — — — —
    S — System MI (preferred over SP)
    A — App Proxy (on-prem via Entra ID)
    S — SCIM (SaaS user lifecycle)
    S — Seamless SSO (hybrid auth)
```

This concludes the comprehensive coverage of Module 14: SAS/Access/Security, Storage Data Protection, Managed Identities, Azure Key Vault, App Registrations, Service Principals, Enterprise Applications, and Entra Proxy Application at L3 depth.
</answer>