# MODULE 6 — ENTRA CONNECT / HYBRID IDENTITY — DEEP DIVE

**Covers Part 7 (Entra Connect / Hybrid Identity)**

---

## 6.1 CONCEPT

Hybrid identity is the bridge between **on-premises Active Directory Domain Services (AD DS)** and **Microsoft Entra ID (cloud directory)**. In enterprise environments, most organizations still have on-prem AD DS for:
- Domain-joined workstations and servers
- On-prem applications (ERP, CRM, line-of-business)
- Legacy authentication (Kerberos, NTLM)
- Group Policy management
- File servers, print servers, on-prem databases

Microsoft Entra ID provides:
- Cloud authentication
- Conditional Access
- Multi-factor authentication
- Cloud application access (Microsoft 365, SaaS apps)
- Modern management (Intune, Endpoint Manager)
- Passwordless and FIDO2 authentication

**Hybrid identity solutions connect these two worlds**, ensuring:
- Users have the same identity in both directories
- Password changes on-prem sync to the cloud (and sometimes vice versa)
- Devices can be managed from both sides
- Authentication can happen in either directory with appropriate controls

**The two current approaches:**

| Approach | Tool | Architecture | When to Use |
|----------|------|-------------|-------------|
| **Synchronization-based** | Entra Cloud Sync (agent-based, lightweight) | Agent on-prem reads AD DS → Syncs to Entra ID cloud | Most new deployments. Preferred by Microsoft. Lighter footprint, more flexible. |
| **Federation-based** | Entra Connect (full, traditional) | On-prem AD DS as authoritative identity → Federation (AD FS or PTA) for authentication | Environments requiring password writeback, seamless SSO, or legacy federation. Being gradually replaced by Cloud Sync + PTA/PHS. |

**Critical L3 distinction:**
```
Synchronization ≠ Authentication

Synchronization = Directory data flows (user attributes, group membership, device info)
  → One-way or two-way
  → Keeps directory data consistent

Authentication = How a user proves their identity
  → Can happen on-prem (AD DS) or in cloud (Entra ID)
  → Synchronization does NOT determine authentication — separate mechanisms do (PHS, PTA, Federation)
```

**Example:**
```
Entra Cloud Sync: Syncs user attributes (name, email, department) from AD DS to Entra ID
  → User's profile is updated in Entra ID
  → User's password is NOT automatically synced (that's a separate mechanism)

Password Hash Sync: Hashes of on-prem passwords are synced to Entra ID
  → User signs in to cloud with same password as on-prem
  → Authentication happens in Entra ID using synced hash

Pass-through Authentication: Entra ID sends password to on-prem agent for validation
  → User signs in to cloud, agent validates against AD DS
  → Password writeback: changes are written back to on-prem AD DS

Federation (AD FS): On-prem AD FS handles cloud authentication
  → User signs in to cloud, request goes to AD FS
  → AD FS validates against AD DS
  → SAML/WS-Fed response returned to Entra ID
```

---

## 6.2 ARCHITECTURE

### 6.2.1 ENTRA CLOUD SYNC ARCHITECTURE (Modern/Preferred)

```
┌─────────────────────────────┐
│      ON-PREM ACTIVE DIRECTORY │
│      (AD DS)                  │
│                              │
│  ┌──────────┐  ┌──────────┐  │
│  │ Domain   │  │ Domain   │  │
│  │ Controllers │  │ Controllers │  │
│  │ (DC1, DC2) │  │ (DC3, DC4) │  │
│  └──────────┘  └──────────┘  │
│       ↑           ↑            │
│       └───────────┘            │
│         AD DS Database          │
└───────────────────────────────┘
           │
           │ (Lightweight Agent — Entra Connect Cloud Sync)
           │ Agent runs on: DC, member server, or separate machine
           │ Uses: AD LDS (Active Directory Lightweight Directory Services)
           │        to read from AD DCs without being a DC itself
           ▼
┌───────────────────────────────┐
│     ENTRA CLOUD SYNC ENGINE    │
│     (in Azure, managed service)│
│                              │
│  ┌─────────────────────────┐  │
│  │  Synchronization Engine  │  │
│  │  - Connectors (AD LDS)   │  │
│  │  - Scope/filtering rules │  │
│  │  - Sync rules (attribute)│  │
│  │  - Dependency resolution  │  │
│  │  - Staging mode support  │  │
│  └─────────────────────────┘  │
│                              │
│  ┌─────────────────────────┐  │
│  │  Connector Server(s)     │  │
│  │  (virtual machines in    │  │
│  │   Azure, managed by      │  │
│  │   Microsoft or self-host)│  │
│  └─────────────────────────┘  │
└───────────────────────────────┘
           │
           │ (Secure TLS connection)
           ▼
┌───────────────────────────────┐
│     MICROSOFT ENTRA ID        │
│     (Cloud Directory)          │
│                              │
│  ┌─────────────────────────┐  │
│  │  Directory Database      │  │
│  │  - Users (synced)        │  │
│  │  - Groups (synced)       │  │
│  │  - Devices (synced)      │  │
│  │  - Attributes (synced)   │  │
│  └─────────────────────────┘  │
│                              │
│  ┌─────────────────────────┐  │
│  │  Authentication Engine   │  │
│  │  (separate from sync)    │  │
│  │  - PHS (password hash)   │  │
│  │  - PTA (pass-through)    │  │
│  │  - Federation            │  │
│  │  - Passwordless          │  │
│  └─────────────────────────┘  │
└───────────────────────────────┘
```

### 6.2.2 ENTRA CONNECT (FULL/TRADITIONAL) ARCHITECTURE

```
┌──────────────────────────────┐
│    ON-PREM AD DS              │
│  ┌─────────────────────────┐  │
│  │ Domain Controllers       │  │
│  │ (AD DS, FSMO roles)      │  │
│  └─────────────────────────┘  │
└──────────────────────────────┘
           │
           ▼
┌──────────────────────────────┐
│    ENTRA CONNECT SERVER       │
│    (Windows Server VM)        │
│                              │
│  ┌─────────────────────────┐  │
│  │  Entra Connect Service   │  │
│  │  - Sync Engine           │  │
│  │  - AD DS Connector       │  │
│  │  - Entra ID Connector    │  │
│  │  - Directory Extension   │  │
│  └─────────────────────────┘  │
│                              │
│  ┌─────────────────────────┐  │
│  │  Optional: AD FS         │  │
│  │  (if using Federation)   │  │
│  └─────────────────────────┘  │
│                              │
│  ┌─────────────────────────┐  │
│  │  Optional: PTA/PHS       │  │
│  │  (Authentication Methods)│  │
│  └─────────────────────────┘  │
└──────────────────────────────┘
           │
           │ (Secure TLS to Entra ID)
           ▼
┌──────────────────────────────┐
│    MICROSOFT ENTRA ID         │
│    (Cloud Directory)           │
└──────────────────────────────┘
```

---

## 6.3 COMPONENTS — DETAILED

### 6.3.1 ENTRA CLOUD SYNC vs ENTRA CONNECT SYNC — CRITICAL DISTINCTION

This is one of the most important distinctions in modern Azure identity.

| Aspect | Entra Cloud Sync | Entra Connect (Full) |
|--------|-----------------|---------------------|
| **Architecture** | Lightweight agent (AD LDS-based) on-prem → Cloud-managed sync service | Full Entra Connect server (Windows VM) on-prem |
| **Server footprint** | No server to manage (Microsoft managed in Azure) | Windows Server VM that you manage |
| **Sync scope** | Per-OU selection, per-attribute filtering (granular) | Full directory or filtered (less granular historically, improved) |
| **Authentication** | Separate (PHS, PTA, or Federation configured independently) | Can include PTA/PHS/Federation as part of installation |
| **Password writeback** | NOT included (by design — use PTA for this) | Supports Password Writeback |
| **Staging mode** | Supported (test sync before going live) | Supported (pre-production configuration) |
| **High availability** | Managed by Microsoft (multi-tenant, redundant) | You manage HA (multiple sync connectors, load-balanced) |
| **Device sync** | Supported (Hybrid Join, Entra registered devices) | Supported (through Device Configuration) |
| **Microsoft recommendation** | **Preferred for new deployments** | For environments with specific requirements (password writeback, legacy federation) |
| **Management** | Cloud-managed, minimal on-prem infrastructure | Requires server management, patching, monitoring |
| **Scalability** | Auto-scales | Limited by server specs, requires planning |
| **Upgrade path** | Automatic by Microsoft | Manual updates (Windows + Entra Connect updates) |

**L3 operational impact:**
- New deployments should use **Cloud Sync**, not full Entra Connect.
- Existing Entra Connect deployments can continue but should evaluate migration to Cloud Sync.
- Password Writeback is NOT available with Cloud Sync → if you need writeback, use Entra Connect (full) or PTA.
- Cloud Sync gives more granular control over what's synced (OU-level, attribute-level filtering).

---

### 6.3.2 SYNC ARCHITECTURE — HOW DATA FLOWS

**Cloud Sync data flow:**
```
1. Lightweight Agent installed on-prem (AD LDS instance)
   → Agent connects to AD DS Domain Controllers
   → Agent uses AD LDS as a "shadow" directory
   → Agent reads changes from AD DS via delta queries

2. Agent forwards changes to Cloud Sync Connector Server
   → Connector Server is a VM in Azure (Microsoft managed or self-hosted)
   → Communication over TLS (encrypted)
   → Agent authenticates to Connector Server

3. Connector Server processes changes
   → Connector Server has Connector to Entra ID
   → Applies sync rules
   → Handles dependencies (user before group membership)
   → Staging mode: applies changes in a "preview" state (no actual changes)

4. Changes applied to Entra ID
   → Microsoft-managed sync engine evaluates:
      - Which objects changed?
      - What attributes changed?
      - Do sync rules allow these changes?
      - Are there conflicts (same object in both directories)?
      - Are dependencies met? (e.g., user must exist before group membership)
   → Changes applied to Entra ID directory

5. Entra ID directory updated
   → Users, groups, devices, attributes updated
   → Token claims updated (next token issuance)
   → Conditional Access evaluations use updated data
   → RBAC (group-based) uses updated membership
```

---

### 6.3.3 SYNC ENGINE — DEEP DIVE

The sync engine is the core component that determines what data flows and how.

**Key capabilities:**
| Capability | Description |
|-----------|-------------|
| **Delta sync** | Only reads changed objects/attributes (not full dump every time) |
| **Dependency resolution** | Resolves creation order (parent before child) |
| **Conflict resolution** | Determines winner when same object exists in both directories |
| **Staging mode** | Tests sync without applying changes (preview) |
| **Filtering** | Scope by OU, group, attribute |
| **Sync rules** | Define attribute mapping and transformation |
| **Error handling** | Handles errors with retry, quarantine, logging |
| **High availability** | Redundant connectors, automatic failover (Cloud Sync) |

**Sync rule types:**
| Rule Type | Description | Example |
|-----------|-------------|---------|
| **Inbound** | From AD DS → Entra ID | Map AD DS `givenName` → Entra ID `givenName` |
| **Outbound** | From Entra ID → AD DS | Password Writeback: Entra ID password → AD DS |
| **Join** | Links objects between directories | Match on-prem user to cloud user via Source Anchor |
| **Projection** | Creates/updates object in target | If on-prem user exists → update Entra ID user |
| **Exclusion** | Filters out objects | Exclude OU "Terminated Employees" from sync |

**Sync rule conflict resolution:**
- When two sync rules target the same attribute, the rule with higher precedence wins.
- Precedence is determined by rule order (configured in the connector).
- `in_` prefix rules generally have higher precedence than `out_` prefix rules.

---

### 6.3.4 FILTERING — COMPLETE DEEP DIVE

Filtering controls WHAT gets synced from on-prem to Entra ID.

**Cloud Sync filtering:**
| Filter Type | Description | Configuration |
|------------|-------------|---------------|
| **OU-based** | Sync specific OUs only | Select OUs during configuration. Only objects in selected OUs are synced. |
| **Group-based** | Sync members of specific groups | Objects that are members of specified groups are synced. |
| **Attribute-based** | Sync objects matching attribute criteria | Custom rules using LDAP queries. |

**Entra Connect filtering:**
| Filter Type | Description | Configuration |
|------------|-------------|---------------|
| **OU-based** | Sync specific OUs | Configured in Connector scope during setup. |
| **Domain-based** | Sync specific domains | For multi-domain environments. |
| **Group-based** | Sync members of groups | Less common in Connect, more in Cloud Sync. |
| **Custom** | Custom LDAP queries | Advanced configurations. |

**L3 troubleshooting — "Users not syncing":**
```
Step 1: Check if users are in the synced scope
  → Are the users' OU in the sync scope?
  → Are the users' groups in the sync scope (if group-based filtering)?
  → Are there any exclusion filters?

Step 2: Check Connector health
  → Cloud Sync: Connector Server status (healthy, error, last sync time)
  → Entra Connect: Sync service status, connector server health

Step 3: Check user attributes
  → Does the user have required attributes for Entra ID?
  → UserPrincipalName: Must be set and match a verified domain
  → UsageLocation: Must be set (required for licensing)
  → ObjectGUID: Must be present (used as Source Anchor)
  → Is the user enabled in AD DS? (userAccountControl)

Step 4: Check for sync errors
  → Connector logs (Cloud Sync: Portal → Entra Connect → Connector health)
  → Entra Connect: Synchronization Service Manager → Errors
  → Common errors: Attribute conflicts, duplicate objects, connector failures

Step 5: Check Source Anchor
  → Is the Source Anchor (ObjectGUID) unique and consistent?
  → Is it being correctly matched between on-prem and cloud?

Step 6: Force sync
  → Force a delta sync and check for errors
```

---

### 6.3.5 SOURCE ANCHOR / IMMUTABLE ID — CRITICAL CONCEPT

**What is Source Anchor?**
Source Anchor (previously called Immutable ID) is the **unique identifier that links an on-prem AD DS object to its Entra ID counterpart**. It's the "match key" that tells the sync engine: "This on-prem user IS THIS cloud user."

**Current best practice (as of Sept 2026):**
Microsoft now recommends using `sourceAnchor` attribute = `objectGUID` of the on-prem AD DS object. The `sourceAnchor` attribute is a newer, more flexible attribute that replaced the older `immutableId` (which was a Base64-encoded `objectGUID`).

**How it works:**
```
On-prem AD DS user:
  ObjectGUID: 12345678-abcd-1234-efgh-123456789012 (binary, 16 bytes)

Sync engine:
  → Reads ObjectGUID from AD DS
  → Uses it as sourceAnchor
  → Matches against Entra ID user's sourceAnchor attribute

Entra ID user:
  sourceAnchor: 12345678-abcd-1234-efgh-123456789012 (same value)
  → Match found! This on-prem user maps to this Entra ID user.
```

**Why Source Anchor matters:**
- Without it, the sync engine cannot link on-prem and cloud objects.
- If Source Anchor changes or is corrupted → objects become "disconnected" → sync fails.
- Cloud Sync uses `sourceAnchor` (new attribute).
- Entra Connect (legacy) used `immutableId` (Base64-encoded ObjectGUID).
- You cannot have the same Source Anchor for two different objects (uniqueness constraint).

**L3 troubleshooting — "Source Anchor issues":**
| Scenario | Cause | Resolution |
|----------|-------|------------|
| User in AD DS but no matching user in Entra ID | Source Anchor not set on either side, or not synced | Check sourceAnchor attribute on both sides, force sync |
| User synced but matching wrong to another user | Duplicate Source Anchor values | Ensure Source Anchor uniqueness, fix and force re-sync |
| "Duplicate sourceAnchor value" error | Two on-prem objects have same Source Anchor (rare but possible if ObjectGUID conflict or migration issue) | Fix on-prem, resolve duplicate, re-sync |
| User was synced, then stopped | Source Anchor was removed or corrupted | Check Source Anchor on both sides, force sync, investigate changes |
| Cloud Sync shows "No match found" for new users | Source Anchor not being read from AD DS | Check connector configuration, attribute mapping, ensure objectGUID is populated |

---

### 6.3.6 AUTHENTICATION METHODS — DEEP DIVE

These are the three primary methods for how users authenticate in a hybrid environment. **They are completely separate from synchronization.** Syncing user attributes to the cloud does NOT mean authentication happens in the cloud by default.

---

#### 6.3.6.1 PASSWORD HASH SYNC (PHS)

**What it is:**
A hash of each on-prem AD DS user's password is synced to Entra ID. Entra ID uses this hash to validate cloud authentication requests.

**How it works:**
```
1. Entra Connect / Cloud Sync installation includes PHS option
2. On-prem agent reads password hash from AD DS (NTLM hash)
3. Hash is encrypted and synced to Entra ID
4. When user signs in to cloud:
   a. User enters password
   b. Entra ID computes hash of entered password
   c. Compares with synced hash
   d. Match → Authentication successful
   e. No match → Authentication failed
5. Cloud tokens issued upon success
```

**PHS properties:**
| Property | Description |
|----------|-------------|
| **Hash type** | NTLM hash (same as AD DS) |
| **Encryption** | Encrypted before syncing to cloud |
| **Transparency** | Microsoft never sees actual passwords (only hashes) |
| **Password change** | User changes on-prem → hash syncs to cloud (typically within minutes) |
| **Cloud password change** | By default, cloud password changes are written back to on-prem (if PTA or writeback enabled). With PHS alone, cloud changes do NOT affect on-prem (unless Password Writeback is separately configured). |
| **Requirements** | Agent on-prem (part of Entra Connect or separate Cloud Sync agent) |

**L3 operational impact:**
- PHS is the most common authentication method for hybrid environments.
- Simple to configure, no additional infrastructure (no AD FS required).
- Password changes on-prem propagate to cloud within minutes (typical sync frequency).
- **PHS + Password Writeback** = On-prem password changes sync to cloud AND cloud password changes write back to on-prem. This provides a unified password experience.

---

#### 6.3.6.2 PASS-THROUGH AUTHENTICATION (PTA)

**What it is:**
An on-prem agent validates passwords in real-time. When a user signs in to the cloud, Entra ID sends the password to the on-prem agent, which validates it against AD DS and returns a pass/fail.

**How it works:**
```
1. PTA agent installed on-prem (on one or more servers)
2. Agent has a secure connection to Entra ID
3. When user signs in to cloud:
   a. User enters username and password
   b. Entra ID sends password (encrypted) to PTA agent
   c. PTA agent validates password against AD DS (via secure channel to DC)
   d. PTA agent returns: Valid / Invalid / Account Locked / etc.
   e. Entra ID makes authentication decision based on agent response
4. Tokens issued if valid
```

**PTA properties:**
| Property | Description |
|----------|-------------|
| **Real-time validation** | Password validated at sign-in time (no hash stored in cloud) |
| **No password hash in cloud** | More secure (Microsoft/cloud never has password hash) |
| **Password writeback** | If enabled, cloud-initiated password changes are written back to on-prem AD DS |
| **High availability** | Requires at least 2 PTA agents for HA (load balanced, one primary/one failover) |
| **Latency** | Add ~200-500ms to authentication (network round-trip to on-prem agent) |
| **Requirements** | PTA agent on-prem, network connectivity between Entra ID and agent, firewall rules allow traffic on port 443 |

**L3 operational impact:**
- PTA requires on-prem network connectivity for EVERY cloud authentication.
- If PTA agent is down → authentication fails (unless PHS is also configured as fallback).
- PTA agents need to be highly available (minimum 2, behind load balancer or with automatic failover).
- PTA is Microsoft's recommended authentication method for environments that don't want to sync password hashes to the cloud.

**PHS vs PTA — Key decision factors:**
| Factor | PHS | PTA |
|--------|-----|-----|
| On-prem dependency for auth | No (hash in cloud) | Yes (agent must be reachable) |
| Network latency | Low (cloud-only auth) | Higher (round-trip to on-prem) |
| Password hash in cloud | Yes (encrypted) | No |
| Password writeback | Yes (with separate writeback) | Yes (built-in) |
| Simplicity | Simpler | More infrastructure |
| HA | Inherent (cloud) | Requires agent HA |
| Security preference | Some orgs don't want hash in cloud | Preferred for security-conscious orgs |

**L3 best practice:** Deploy both PHS and PTA. PHS as primary (works when on-prem is unreachable), PTA for day-to-day (no hash in cloud), with PTA writeback for password changes.

---

#### 6.3.6.3 FEDERATION (AD FS or SAML-based)

**What it is:**
An on-prem federation server (typically AD FS) handles authentication requests from Entra ID. The user's password is validated on-prem, and a SAML/WS-Fed response is returned to Entra ID.

**How it works:**
```
1. AD FS server(s) on-prem (or other SAML IdP)
2. Entra Connect configures trust between Entra ID and AD FS
3. When user signs in to cloud:
   a. Entra ID redirects authentication request to AD FS (via federation endpoint)
   b. User enters credentials (or is already authenticated to AD FS via browser cookie)
   c. AD FS validates credentials against AD DS
   d. AD FS issues SAML token (or WS-Fed response)
   e. Entra ID validates SAML token
   f. Entra ID issues its own tokens for cloud resources
4. User accesses cloud resources with Entra ID tokens
```

**Federation properties:**
| Property | Description |
|----------|-------------|
| **Authentication** | Happens on-prem (AD FS validates against AD DS) |
| **Password storage** | No password hash in cloud |
| **SSO** | Seamless SSO between cloud and on-prem (via AD FS cookies) |
| **Protocol** | SAML 2.0, WS-Fed ( Passive), OAuth2/OIDC (Active) |
| **Infrastructure** | Requires AD FS servers (at least 2 for HA), certificates |
| **Complexity** | Highest complexity, most infrastructure |
| **Common use** | Organizations with existing AD FS, or specific requirements (custom auth, claim transformation) |

**L3 operational impact:**
- Federation requires ALL authentication to go through AD FS → AD DS dependency for EVERY cloud sign-in.
- AD FS server outage → All cloud authentication fails (unless PHS is configured as fallback).
- Federation is being gradually replaced by PHS/PTA for most scenarios. Microsoft recommends PHS or PTA unless you have specific federation requirements.

---

### 6.3.7 PASSWORD WRITEBACK

**What it is:**
When a user changes their password in the cloud (via SSPR or my profile), the change is written back to on-prem AD DS.

**Supported with:**
- PTA (built-in)
- Entra Connect (full) with Password Writeback feature
- NOT available with Cloud Sync alone (Cloud Sync doesn't do writeback by design)

**How it works (PTA):**
```
1. User changes password via SSPR or my profile in cloud
2. Entra ID sends new password to PTA agent
3. PTA agent validates connectivity to AD DS
4. PTA agent changes password on AD DS domain controller
5. Change propagated via AD DS replication to other DCs
6. User can now sign in to both cloud and on-prem with new password
```

**L3 troubleshooting — "Password changed in cloud but on-prem password didn't change":**
1. Is PTA configured and enabled?
2. Are PTA agents running and healthy?
3. Do PTA agents have permission to reset passwords on AD DS?
4. Is the user's account in the correct scope?
5. Check PTA agent logs for errors.
6. Check AD DS for password last set time.
7. Check for password policy conflicts (minimum password age, complexity).

---

### 6.3.8 DEVICE SYNCHRONIZATION

**What it is:**
Device information from on-prem AD DS is synced to Entra ID, enabling device-based Conditional Access policies and hybrid device management.

**What gets synced:**
| Device Type | Sync Behavior |
|------------|---------------|
| **Hybrid Joined devices** | Device object synced from AD DS to Entra ID. Device state visible in CA. |
| **Entra Registered devices** | Synced to Entra ID (from mobile devices, etc.) |
| **Server devices** | Server objects synced (for server-based CA conditions) |

**L3 operational impact:**
- Without device sync, Conditional Access cannot evaluate device state (compliant, hybrid joined, etc.) for on-prem devices.
- Device sync is required for Zero Trust scenarios that include device compliance.
- Device sync enables: "Require hybrid joined device" CA condition.

---

### 6.3.9 STAGING MODE

**What it is:**
Staging mode allows you to test Entra Connect/Cloud Sync configuration BEFORE applying it to production.

**In Staging mode:**
- Sync engine reads on-prem data
- Sync engine evaluates sync rules
- Sync engine shows WHAT changes would be made
- NO actual changes are applied to Entra ID
- After validation, you switch from Staging to Production mode

**L3 best practice:**
- Always configure in staging mode first.
- Run at least one full sync cycle in staging.
- Review what would be synced (no surprises in production).
- Switch to production only after validation.

---

### 6.3.10 HIGH AVAILABILITY

**Cloud Sync HA:**
- Microsoft-managed infrastructure (automatically HA).
- Multiple connector servers in the background (customer doesn't manage).
- Automatic failover.
- Agent (AD LDS) on-prem should be on multiple domain controllers for redundancy (agent can reach any DC).

**Entra Connect (Full) HA:**
- Deploy at least 2 Entra Connect servers (or connector servers for sync).
- Load balanced (or primary/standby).
- If using AD FS: Minimum 2-3 AD FS servers in a farm, behind a load balancer.
- PTA: Minimum 2 agents (automatic failover between them).
- PHS: No additional HA required (cloud-based).

---

### 6.3.11 HYBRID JOIN — SOFT MATCH vs HARD MATCH

**What is Hybrid Join?**
A device that is both:
- Joined to on-prem AD DS (domain-joined)
- Registered with Entra ID (cloud identity)

This gives the device:
- On-prem management (Group Policy, AD DS)
- Cloud management (Intune/Endpoint Manager, Entra ID)
- Conditional Access device state: "Hybrid Joined"

**How Hybrid Join works with Entra Connect/Cloud Sync:**

**Soft Match (modern, preferred):**
```
1. Device is domain-joined (on-prem AD DS)
2. User signs in with on-prem credentials
3. During sign-in, Entra Connect/Cloud Sync:
   a. Reads device information from AD DS
   b. Checks if a matching device already exists in Entra ID
   c. "Soft match" = Find device in Entra ID by matching:
      - Device ID (if previously registered)
      - Or device attributes (serial number, etc.)
   d. If match found → Link on-prem device to existing Entra device
   e. If no match → Create new Entra device object, link to on-prem device
4. Device now shows as "Hybrid Joined" in Entra ID
5. Device sync: Device attributes flow from AD DS to Entra ID
```

**Hard Match (legacy, Entra Connect full):**
```
1. Device is domain-joined
2. Entra Connect "device configuration" creates device object in Entra ID
3. Links on-prem device to Entra device via hard match
4. Same result as soft match but through Connect's device management
5. Hard match is being replaced by soft match in Cloud Sync
```

**Soft Match vs Hard Match — Key Differences:**
| Aspect | Soft Match | Hard Match |
|--------|-----------|------------|
| **Managed by** | Cloud Sync (recommended) | Entra Connect (legacy) |
| **How matching works** | Device attributes compared to find existing Entra device | Direct linking via Connect |
| **Device registration** | User-driven (device registered via Settings or Intune) | Connect-driven |
| **Azure AD Join vs Hybrid Join** | Azure AD Join (cloud-only) vs Hybrid Join (both) are distinct | Azure AD Join vs Hybrid Join were less clearly separated |
| **Recommended** | Yes (Microsoft's current guidance) | No (legacy) |

**L3 troubleshooting — "Device shows as not Hybrid Joined despite being domain-joined":**
1. Is Entra Connect/Cloud Sync running and healthy?
2. Is device sync enabled in the configuration?
3. Is the device actually domain-joined? (check System Properties → Computer Name)
4. Has the device sync completed? (check Entra ID → Devices for the device)
5. Check if there's a device object in Entra ID but it's not linked to the on-prem device.
6. Check for sync errors related to device objects.
7. Verify the user who signed in has the right to register/link the device.
8. Soft match may need device to be Entra registered first (via Settings → Accounts → Access work or school).

---

### 6.3.12 DUPLICATE IDENTITIES

**What is a duplicate identity?**
A situation where the same user exists as both a member user and a guest user (or has multiple objects) in Entra ID.

**Common causes:**
- User invited as guest before being synced as member
- User created manually in cloud, then synced from on-prem
- Different UPNs for same person (e.g., `john@contoso.com` and `john@contoso.onmicrosoft.com`)
- Migration issue: User synced under one Source Anchor, then created with another

**How Entra ID handles it:**
- Entra ID tries to **consolidate** duplicate objects.
- If the same sourceAnchor maps to two objects, one becomes "Pending" and waits for resolution.
- If the same UPN maps to two objects, the objects are flagged for consolidation.

**L3 troubleshooting:**
```
1. Check for duplicates:
   - Portal: Entra ID → Users → filter by "Guest" and "Member"
   - Look for same person with both Member and Guest user types
   - Check Source Anchor consistency

2. Consolidation:
   - Entra ID may auto-consolidate when it detects duplicates
   - If stuck in "Pending" → Admin needs to consolidate manually
   - Delete the guest object (if member is the primary identity)

3. Prevention:
   - Ensure all users are synced from AD DS (no manual cloud creation)
   - Use consistent UPNs
   - Source Anchor uniqueness across all objects
   - Clean up guest invitations before sync
```

---

### 6.3.13 ENTRA CONNECT / CLOUD SYNC DIAGNOSTIC TOOLS

| Tool | Purpose | Type |
|------|---------|------|
| **Synchronization Service Manager** | Monitor sync operations, view errors, force sync | Entra Connect (full) |
| **Entra Connect Health** | Monitoring dashboard, sync alerts, AD DS health checks | Entra Connect (full) / Cloud Sync |
| **Cloud Sync connector health** | Portal-based monitoring of Cloud Sync connectors | Cloud Sync |
| **AD DS connectivity test** | Verify connector can reach DCs | Both |
| **Connector log files** | Detailed sync logs | Both |
| **Event logs** | Windows Event Log on Connect server | Entra Connect (full) |
| **Staging mode preview** | See what would sync without applying | Both |
| **Microsoft Support tools** | RPV (Request Password Verification), SPT (Sync Performance Testing) | Both (Microsoft support-assisted) |

---

## 6.4 COMMUNICATION FLOW — HYBRID IDENTITY

### Complete Hybrid Authentication Flow

```
SCENARIO: User with on-prem AD DS account signs in to Azure Portal

STEP 1: USER OPENS BROWSER → NAVIGATES TO AZURE PORTAL
  → Browser goes to portal.azure.com

STEP 2: REDIRECT TO ENTRA ID SIGN-IN PAGE
  → Portal redirects to: https://login.microsoftonline.com/{tenant-id}/oauth2/authorize

STEP 3: TENANT LOOKUP
  → Entra ID identifies tenant from URL or user's UPN
  → Tenant settings loaded (authentication methods, CA policies, auth method)

STEP 4: USER ENTERS UPN
  → Entra ID searches directory for user
  → User found in synced directory (Cloud Sync or Entra Connect)

STEP 5: AUTHENTICATION METHOD DETERMINATION
  → Entra ID checks: How is this user's password managed?
  → Three possible paths:

  PATH A: PASSWORD HASH SYNC (PHS)
  ────────────────────────────────────
  5A.1: User enters password
  5A.2: Entra ID computes hash of entered password
  5A.3: Compares with synced hash from on-prem
  5A.4: Match → Authentication success
  5A.5: Tokens issued

  PATH B: PASS-THROUGH AUTHENTICATION (PTA)
  ──────────────────────────────────────────
  5B.1: User enters password
  5B.2: Entra ID sends password to PTA agent (encrypted, over secure channel)
  5B.3: PTA agent validates against AD DS (secure channel to DC)
  5B.4: PTA agent returns: Valid / Invalid
  5B.5: Valid → Authentication success, tokens issued
  5B.6: Invalid → Authentication failed

  PATH C: FEDERATION (AD FS)
  ──────────────────────────
  5C.1: Entra ID redirects authentication request to AD FS endpoint
  5C.2: User is redirected to AD FS sign-in page (or has existing AD FS session)
  5C.3: User enters credentials (or AD FS session is still active)
  5C.4: AD FS validates against AD DS
  5C.5: AD FS issues SAML/WS-Fed response to Entra ID
  5C.6: Entra ID validates response → Authentication success
  5C.7: Entra ID issues tokens → User accesses portal

STEP 6: CONDITIONAL ACCESS EVALUATION (regardless of auth path)
  → After authentication, CA policies evaluated:
    - Is user in scope? (include/exclude)
    - Location trusted? (IP geolocation, Named Locations)
    - Device compliant? (Intune compliance or device state from sync)
    - Device hybrid joined? (from device sync)
    - Client app modern? (browser, app type)
    - Authentication strength sufficient?
    - Risk level acceptable?
  → Grant controls applied if required

STEP 7: TOKEN ISSUANCE
  → ID Token, Access Token, Refresh Token issued
  → Token includes CA claims (if session controls applied)
  → Token includes group memberships (from synced directory)

STEP 8: RESOURCE ACCESS
  → User accesses Azure resources
  → RBAC evaluated (from tokens and group claims)
  → Resource-specific authorization
  → Access granted/denied
```

---

## 6.5 ADMINISTRATION — HYBRID IDENTITY

### Key Administrative Operations

| Operation | Tool | Scope |
|-----------|------|-------|
| Deploy Entra Cloud Sync | Portal (Entra ID → Settings → Sync settings) | Tenant |
| Deploy Entra Connect (full) | Entra Connect installer (Windows Server VM) | Tenant |
| Configure authentication method | Portal → Entra ID → Authentication Methods | Tenant |
| Configure PTA | Portal → Entra ID → Authentication Methods → PTA | Tenant |
| Configure PHS | Portal → Entra ID → Authentication Methods → Password Hash Sync | Tenant |
| Configure AD FS | Entra Connect installer or manual AD FS deployment | Tenant |
| Manage sync scope | Portal (Cloud Sync) or Synchronization Service Manager (Connect) | Per connector |
| Force delta sync | Portal or CLI | Per connector |
| Staging mode toggle | Portal | Per connector |
| Password Writeback enable/disable | Portal | Tenant |
| View sync errors | Portal, Synchronization Service Manager | Per connector |
| Device sync management | Portal → Devices | Tenant |
| Monitor health | Entra Connect Health / Cloud Sync health dashboard | Tenant |
| Manage duplicate identities | Portal → Entra ID → Users → Consolidate | Tenant |
| Manage Hybrid Join | Intune, Entra ID → Devices | Tenant |

---

## 6.6 SECURITY CONSIDERATIONS

| Concern | Risk | Mitigation |
|---------|------|------------|
| **On-prem AD DS compromise** | If on-prem AD DS is compromised, attacker can use PTA to validate passwords for cloud access | Secure AD DS (least privilege, LAPS, Defender for Identity, monitor DCs) |
| **PTA agent compromise** | If PTA agent compromised, attacker could validate any password | Secure PTA servers, monitor agent health, limit PTA agent network access |
| **PHS hash extraction** | If password hash is extracted from Entra ID (theoretical) | Microsoft encrypts hashes; monitor for unusual hash extraction; use PTA for sensitive accounts |
| **Sync over、过宽 scope** | Syncing all on-prem objects including terminated employees | Use OU/Group filtering to limit scope |
| **Source Anchor collision** | Two objects with same Source Anchor → sync conflicts | Ensure uniqueness, use `sourceAnchor` (objectGUID) consistently |
| **Stale devices** | Old domain-joined devices still synced as hybrid joined | Device cleanup, decommission process, remove stale device objects |
| **Guest invitation before sync** | User created as guest, then synced as member → duplicate | Sync before inviting, consolidate duplicates |
| **Federated authentication dependency** | AD FS outage = cloud authentication failure | Deploy AD FS HA, consider PHS/PTA as fallback, or dual authentication methods |
| **PTA agent failure** | All PTA agents down → authentication failure | Deploy 2+ PTA agents, enable PHS as fallback, monitor agent health |
| **Domain Controller unreachable** | PTA cannot validate, PHS already in cloud (works), Federation cannot validate | PHS is most resilient; ensure PTA agent has multiple DCs to connect to |
| **Password spray from cloud** | Attacker tries many passwords via cloud → triggers account lockout on-prem | Ensure account lockout policy is consistent, monitor for spray patterns |
| **Sync tampering** | Agent compromised to manipulate sync data | Secure connector servers, monitor sync logs, integrity checks |

---

## 6.7 MONITORING — HYBRID IDENTITY

### Key Monitoring Targets

| Metric/Log | What It Shows | Alert Trigger |
|------------|---------------|---------------|
| **Sync health** | Connector status, last sync time, error count | Sync stopped, errors increasing |
| **Sync latency** | Time between on-prem change and cloud sync | High latency (>15 min for delta) |
| **Sync errors** | Specific errors per object/sync cycle | Any sync error for critical objects (users, groups) |
| **Authentication method usage** | PHS vs PTA vs Federation sign-in counts | Unexpected auth method changes |
| **PTA agent health** | Agent status, response time, primary/secondary | Agent down, response time increase |
| **AD DS connectivity** | Connector's connection to DCs | DC unreachable, replication issues |
| **Device sync** | Devices synced, pending, error | Sudden drop in synced devices |
| **Password writeback** | Writeback success/failure | Writeback failures, password inconsistency |
| **Duplicate identities** | Pending consolidation objects | Unresolved duplicates |
| **Staging mode** | Whether staging is still enabled | Staging left enabled in production |
| **Entra Connect Health** | Overall health dashboard, AD DS connector health | Health score degraded |
| **Cloud Sync connector health** | Connector server status, sync throughput | Connector unhealthy |

---

## 6.8 PRODUCTION EXAMPLE

**Scenario: Enterprise with 10,000 users, 3 data centers, hybrid Azure AD Join.**

```
Tenant: contoso.onmicrosoft.com (Premium P2)
AD DS: Forest contoso.com, 3 domains (contoso.com, corp.contoso.com, dev.contoso.com)
       12 Domain Controllers across 3 sites

Authentication:
  Primary: Password Hash Sync (PHS) — works even if on-prem unreachable
  Secondary: Pass-through Authentication (PTA) — 4 PTA agents (2 per site, automatic failover)
  Fallback: Entra Connect full installed as backup for PTA/PHS if both fail
  Password Writeback: Enabled (via PTA)
  Federation: None (deprecated, no AD FS)

Sync: Entra Cloud Sync
  Configuration:
    - Staging mode → validated → Production
    - OU Filtering:
      Synced: Users in "Corp-Users", "Corp-Computers", "Corp-Groups" OUs
      Excluded: "Terminated-Employees" OU, "Service-Accounts-Decommissioned" OU
    - Attribute Filtering:
      Excluded: employeeType (internal only), costCenter (privacy)
    - Device Sync: Enabled (Hybrid Join for all corporate devices)
    - Source Anchor: sourceAnchor = objectGUID (Microsoft recommended)

Connector Servers (Azure VMs, managed by Microsoft):
  - 3 Connector servers across 3 Azure regions for HA
  - Automatic failover between them

Authentication Flow:
  All 10,000 users: PHS primary, PTA secondary
  - If on-prem reachable: PTA validates (no hash in cloud)
  - If on-prem unreachable: PHS validates (hash in cloud)
  - Password changes: Written back via PTA to on-prem AD DS

Devices:
  15,000 devices: All corporate devices Hybrid Joined
  - Device sync from AD DS to Entra ID via Cloud Sync
  - Conditional Access: Require hybrid joined device for corporate apps

Conditional Access:
  Priority 1: Block legacy authentication
  Priority 5: Require MFA from untrusted locations
  Priority 10: Require compliant device for all cloud apps
  - Device state from device sync + Intune compliance

Monitoring:
  - Entra Connect Health dashboard → monitored 24/7
  - Cloud Sync connector health → automated alerts
  - PTA agent health → automated alerts
  - Sign-in logs → monitored for auth failures, CA blocks
  - Sync errors → automated alerts for critical objects
```

---

## 6.9 FAILURE SCENARIOS — HYBRID IDENTITY

| Scenario | Root Cause | Resolution | Evidence |
|----------|-----------|------------|----------|
| **Users not syncing from on-prem** | Connector down, scope filtering excludes users, user UPN not set, Source Anchor issue, AD DS connectivity failure | Check connector health, sync scope, user properties, Source Anchor, AD DS connectivity | Connector logs, sync operations, error codes |
| **Password changed on-prem but cloud login fails** | PHS sync not yet completed (typical delay: 1-5 min), PTA agent down if PTA is primary, or PHS not configured | Wait for sync, check sync health, verify PHS/PTA configured | Sign-in log: "Invalid password"; Cloud Sync: last sync time; PHS/PTA status |
| **Password changed in cloud but on-prem password unchanged** | PTA not configured/agents down, Password Writeback not enabled, PTA agent permission issue | Check PTA agents, Password Writeback settings, agent logs | PTA agent logs, AD DS password last set timestamp |
| **User authenticates from cloud but gets "Account disabled"** | User was disabled in on-prem AD DS, sync propagated to Entra ID (AccountEnabled = false), or user was disabled in Entra ID separately | Check user status in both directories, check sync direction | Entra ID user → AccountEnabled; AD DS user → userAccountControl |
| **Conditional Access shows "Device not compliant" for domain-joined device** | Device sync not complete, device not Entra registered, Intune compliance not configured, device state sync delay | Check device sync status, device registration, Intune compliance, CA device state | Entra ID → Devices → device status; Portal → device details |
| **PTA agent returns timeout during authentication** | PTA agent cannot reach DC, network issue between Entra ID and PTA agent, DC overloaded, firewall blocking | Check PTA agent health, network connectivity, DC health, firewall rules | PTA agent logs, Event Viewer on PTA server |
| **Duplicate identity detected (member + guest)** | User invited as guest before sync, or manually created in cloud then synced | Consolidate identities (delete guest or remove member), ensure consistent Source Anchor | Entra ID → Users → duplicates warning |
| **Sync stopped after configuration change** | Bad sync rule, OU filtering change excluded all users, connector authentication failure, staging mode left on | Check staging mode, sync rules, connector health, scope configuration | Connector logs, sync engine errors |
| **Device shows as "Hybrid Joined" in AD DS but not in Entra ID** | Device sync failed, device object not created in Entra ID, soft match not completed | Check device sync status, force sync, check for sync errors on device objects | Entra ID → Devices → search device; Cloud Sync → device sync results |
| **Service principal cannot authenticate after on-prem password change** | If SP uses PHS-based credential, password change didn't sync yet; if SP uses PTA, PTA agent down | Wait for sync, verify authentication method for SP, check PTA/PHS health | SP sign-in log, authentication method status |

---

## 6.10 TROUBLESHOOTING METHODOLOGY — HYBRID IDENTITY

```
STEP 1: Identify the symptom
  → "User cannot sign in to cloud"
  → "Users not syncing from on-prem"
  → "Password not updating between on-prem and cloud"
  → "Device not showing as Hybrid Joined"

STEP 2: Determine scope
  → All users or specific users?
  → All OUs or specific OUs?
  → All devices or specific devices?
  → Recent change (sync config, auth method, network)?

STEP 3: Check Connector/Agent health (FIRST for sync issues)
  → Cloud Sync: Portal → Entra ID → Overview → Entra Connect → Connector health
  → Entra Connect: Synchronization Service Manager
  → Is connector running? When was last sync? Any errors?

STEP 4: Check Authentication method
  → PHS enabled? PTA enabled? Federation configured?
  → Is the user's password managed by on-prem or cloud?
  → Is the authentication method healthy? (PTA agents running, PHS syncing)

STEP 5: Check user object properties
  → User exists in both directories?
  → Source Anchor present and matching? (sourceAnchor or immutableId)
  → UserPrincipalName correct and in verified domain?
  → UsageLocation set? (required for licensing)
  → AccountEnabled = true?
  → User in synced OU?

STEP 6: Check sync scope
  → Is user's OU in the sync scope?
  → Are there exclusions that would exclude this user?
  → Is filtering configured correctly?

STEP 7: Check sync errors
  → Review connector error logs
  → Look for specific error codes
  → Force delta sync and observe

STEP 8: Check device-specific (for device issues)
  → Is device domain-joined?
  → Is device Entra registered?
  → Device sync enabled?
  → Device object exists in Entra ID?
  → Is device linked via soft match?

STEP 9: Network connectivity
  → PTA agents can reach Entra ID? (port 443)
  → Connector agents can reach AD DS DCs?
  → Any VPN changes, firewall changes, network outages?
  → AD DS replication healthy?

STEP 10: Force and validate
  → Force delta sync
  → Check for errors
  → Test authentication
  → Monitor for recurrence
  → Document root cause
```

---

## 6.11 LOGS / EVIDENCE

| Evidence Source | What It Shows | Access Method |
|----------------|---------------|---------------|
| **Sync logs (Cloud Sync)** | Object sync results, attribute changes, errors, timestamps, connector info | Portal → Entra ID → Overview → Entra Connect → Cloud Sync |
| **Synchronization Service Manager** | Detailed sync operations, errors, warnings, per-object sync status (Entra Connect full) | Synchronization Service Manager tool (on Entra Connect server) |
| **Connector health** | Connector server status, last sync, error count, timestamp | Portal (Cloud Sync) or tool (Entra Connect) |
| **PTA agent logs** | PTA agent status, validation results, errors, timestamps | PTA server Event Viewer, Portal → Authentication Methods → PTA status |
| **AD DS connectivity logs** | Connector's connection to DCs, authentication to AD DS | Connector server logs, Event Viewer |
| **Password Writeback logs** | Password change events, writeback success/failure | PTA agent logs, Entra ID audit logs |
| **Device sync logs** | Device object creation, updates, linking, errors | Portal → Devices, Cloud Sync logs |
| **Entra Connect Health** | Overall hybrid identity health dashboard | Portal → Entra ID → Overview → Entra Connect Health |
| **Event logs (Entra Connect server)** | Windows events from Entra Connect installation | Event Viewer on Connect server |
| **Activity Log (Azure)** | Administrative changes to Entra Connect, authentication methods | Portal → Subscription → Activity Log |

---

## 6.12 VERSION/CURRENT SERVICE CONSIDERATIONS (Sept 2026)

| Aspect | Current State | L3 Impact |
|--------|--------------|-----------|
| **Cloud Sync vs Entra Connect** | Cloud Sync is Microsoft's preferred tool. Entra Connect (full) still supported for specific scenarios (Password Writeback via PTA, legacy federation). | New deployments: Cloud Sync. Existing deployments: evaluate migration. |
| **Source Anchor** | `sourceAnchor` attribute (using `objectGUID`) is now the standard. Legacy `immutableId` (Base64-encoded ObjectGUID) still works but is being phased out. | New deployments use `sourceAnchor`. Existing deployments should migrate if using `immutableId`. |
| **Password Hash Sync** | Fully supported, recommended for most scenarios. No longer considered a security risk by Microsoft (encrypted in transit and at rest). | PHS is the default for most configurations. |
| **Pass-through Authentication** | Fully supported, recommended alongside PHS. | PTA agents: minimum 2 for HA. PTA is the preferred method for environments that don't want password hash in cloud. |
| **AD FS** | Still supported but Microsoft is actively moving customers to PHS/PTA. AD FS is NOT recommended for new deployments unless specific requirements exist. | Evaluate migration from AD FS to PHS/PTA. |
| **Password Writeback** | Available via PTA (not available with Cloud Sync alone). | If using Cloud Sync and need writeback: deploy PTA alongside, or use Entra Connect (full). |
| **Device sync** | Cloud Sync device sync is the modern approach. Replaces legacy Entra Connect device management. | Ensure device sync is enabled for Conditional Access device-based policies. |
| **Hybrid Join** | Soft Match is the current method. Hard Match is legacy. | Use soft match for new Hybrid Join deployments. |
| **Legacy authentication** | Being deprecated. CA can block it. | Eliminate legacy auth protocols from your environment. |
| **Federated authentication** | Supported but declining. Microsoft pushing PHS/PTA. | Only use federation if you have specific requirements (e.g., custom identity provider, AD FS). |
| **Source Anchor migration** | Tools available to migrate from `immutableId` to `sourceAnchor` without re-syncing all objects (Microsoft support-assisted). | Plan and execute migration for older deployments. |
| **Entra Connect Health** | Available and recommended. Monitors connector health, sync status, AD DS health. | Deploy and configure Entra Connect Health for monitoring. |

---

## 6.13 L3 INTERVIEW QUESTIONS

### Basic
**Q: What is the difference between Entra Connect and Entra Cloud Sync?**
A: Entra Connect (full) is the traditional on-premises sync server that requires a Windows Server VM and manages sync, authentication (PTA/PHS), and device management from a single on-prem server. Entra Cloud Sync is the modern, lightweight agent-based approach where the connector infrastructure is managed by Microsoft in Azure, and a lightweight agent on-prem reads from AD DS. Cloud Sync is preferred for new deployments.

### Intermediate
**Q: What is the difference between synchronization and authentication?**
A: Synchronization is about keeping directory data consistent between on-prem AD DS and Entra ID (user attributes, group membership, device info). Authentication is about how a user proves their identity when signing in (password hash, pass-through validation, federation). You can have sync without authentication being in the cloud (e.g., data syncs but auth still happens on-prem via federation). They are independent mechanisms.

### L3
**Q: A user changed their on-prem password, but they cannot sign in to the cloud for 15 minutes. Why?**
A: The password hash (PHS) or password validation data (PTA) hasn't synced yet. PHS typically syncs within 1-5 minutes, but can take longer during high load or if the connector is experiencing issues. Check: (1) Connector health and last sync time, (2) Whether PHS or PTA is being used, (3) Connector error logs. If PTA is the primary method and PTA agent is down, the user cannot authenticate at all until PHS fallback is available (if configured) or PTA is restored.

### Senior L3
**Q: Explain why PTA requires high availability and what happens if all PTA agents go down.**
A: PTA works by sending the user's password to an on-prem agent for validation against AD DS. If there's only one PTA agent and it goes down (server restart, crash, network issue), ALL cloud authentication via PTA fails. If PHS is also configured, users can still authenticate via PHS (password hash in cloud). If only PTA is configured, ALL users are locked out from cloud access. Best practice: minimum 2 PTA agents for automatic failover, plus PHS as a resilient fallback.

### Expert
**Q: What happens internally when a synced user signs in to the cloud with PTA?**
A: User navigates to Azure Portal → Redirected to Entra ID → Enters UPN → User found in directory → Entra ID determines password is managed by PTA → User enters password → Entra ID encrypts password and sends to PTA agent over secure TLS channel → PTA agent validates password against AD DS (via secure channel to DC, typically using a service account with password reset permissions) → PTA agent returns "Valid" or "Invalid" → Entra ID makes authentication decision → If valid, CA policies evaluated → If CA passes, tokens issued (ID, Access, Refresh) → User accesses cloud resources. Throughout this flow, if any PTA agent is unavailable, the request fails over to another PTA agent automatically.

### Scenario
**Q: "Entra Connect shows sync errors for all user objects. What do you check first?"**
A: First, check the connector health (is it running?). Then check: (1) AD DS connectivity — can the connector reach domain controllers? (2) Account used by connector — does it have appropriate permissions to read AD DS? (3) Scope filtering — was filtering accidentally changed to exclude all users? (4) Source Anchor — is it configured correctly? (5) Staging mode — is it still in staging mode (changes not applied)? (6) Event logs on the connector server for errors. (7) Force a delta sync and observe specific errors.

### Tricky
**Q: "A user's on-prem password was changed, and they can sign in to the cloud. But their cloud password is still the old one." Is this a problem?**
A: Not necessarily. If PHS + PTA is configured with Password Writeback:
- On-prem password change → synced to cloud via PHS → cloud password is now the same as on-prem.
- If PTA without Password Writeback → cloud password didn't change (cloud uses on-prem hash).
- If user changed password via cloud (SSPR) → Password Writeback writes it back to on-prem → both passwords updated.
The question is: which direction did the password change happen? If on-prem → cloud: should be synced. If cloud → on-prem: needs Password Writeback. The tricky part is that many engineers assume "password change in one place = both places" without checking the synchronization direction and writeback configuration.

### Tricky 2
**Q: "Cloud Sync is being used, but Password Writeback is needed. How?"**
A: Cloud Sync itself does NOT support Password Writeback. The options are:
1. Deploy PTA alongside Cloud Sync (PTA handles authentication and Password Writeback).
2. Deploy Entra Connect (full) alongside Cloud Sync for PTA/Password Writeback functionality.
3. Switch to Entra Connect (full) if Password Writeback is a primary requirement.
A common misconception is that Cloud Sync should do everything. It's designed specifically for synchronization only — authentication and writeback are separate mechanisms configured independently.

### Tricky 3
**Q: "A device is domain-joined and shows as 'Hybrid Joined' in AD DS, but Entra ID shows it as 'Registered' (not 'Hybrid Joined'). Why?"**
A: Device sync may not have completed, or the device was only Entra registered (not linked to on-prem device via soft match). Soft match requires the device to be linked based on matching attributes. If soft match didn't occur (e.g., device was registered via Intune/Settings but not recognized as the same as the AD DS device), it shows as "Registered" instead of "Hybrid Joined." Check device sync status, force sync, and verify device attributes match between on-prem and cloud.

---

## 6.14 SCENARIO-BASED QUESTIONS

### Scenario 1: "All users not syncing after Entra Connect installation"
**Architecture:** Connector → AD DS connector → Sync engine → Entra ID connector → Entra ID
**Dependencies:** AD DS connectivity, connector account permissions, scope configuration, Source Anchor
**Checks:**
- Connector running?
- AD DS connectivity (can reach DCs)?
- Connector account has appropriate permissions?
- Scope includes correct OUs?
- Staging mode left on?
- Source Anchor configured?
- Force delta sync → check specific errors
**Root Cause (common):** Staging mode left enabled, or connector account lacks permissions, or scope doesn't include user OUs.
**Fix:** Disable staging, correct permissions, adjust scope, force sync.
**Validation:** Users appear in Entra ID, subsequent changes sync automatically.

### Scenario 2: "Users synced but passwords don't match between on-prem and cloud"
**Architecture:** PHS/PTA → Password sync → Authentication
**Dependencies:** Authentication method configuration, connector health, Password Writeback configuration
**Checks:**
- Which auth method? (PHS or PTA or Federation?)
- Is Password Writeback enabled?
- Are PTA agents healthy?
- When was last sync?
- Did password change happen on-prem or in cloud?
**Root Cause: (common):** PTA agents down, or Password Writeback not configured, or sync delay.
**Fix:** Restore PTA agents, configure Password Writeback, wait for sync, or manually reset password in both places.
**Validation:** Password works in both on-prem and cloud.

### Scenario 3: "Domain-joined laptop shows as 'Azure AD Registered' instead of 'Hybrid Joined'"
**Architecture:** Device sync → Soft match → Device registration
**Dependencies:** Device sync configuration, soft match algorithm, Entra Connect/Cloud Sync health
**Checks:**
- Is device sync enabled?
- Did device sync complete?
- Does device match via soft match criteria?
- Is the device object in Entra ID linked to on-prem device?
- Check device attributes in both directories
**Root Cause: (common):** Device sync didn't happen, or soft match criteria not met (different device IDs, or device was re-registered).
**Fix:** Force device sync, verify device identity, manually link if needed.
**Validation:** Device shows "Hybrid Joined" in Entra ID, CA device state = Hybrid Joined.

### Scenario 4: "Password writeback fails with 'insufficient access rights'"
**Architecture:** PTA → Password Writeback → AD DS password change
**Dependencies:** PTA agent permissions, AD DS permissions, password policy
**Checks:**
- PTA agent service account permissions (must have "Reset Password" and "Write" on AD DS objects)
- Is the OU the user is in have correct ACL?
- Password policy: minimum password age preventing immediate change?
- PTA agent can reach DC?
**Root Cause: (common):** PTA service account doesn't have password reset permission on the user's OU or container.
**Fix:** Grant password reset permission to PTA service account on the appropriate OU.
**Validation:** Password change via SSPR writes back to on-prem successfully.

### Scenario 5: "After enabling Cloud Sync, existing users are duplicated as guests"
**Architecture:** B2B invitation vs sync → Duplicate identity consolidation
**Dependencies:** Invitation timing, Source Anchor, sync configuration
**Checks:**
- Were users invited as guests before sync?
- Are there duplicate UPNs in Entra ID?
- Source Anchor consistency check
- Duplicate identity report in Entra ID
**Root Cause: (common):** Users were invited as guests (B2B) before Cloud Sync was configured, then synced as members → Entra ID detects duplicates.
**Fix:** Consolidate identities (delete guest objects, retain member objects), verify Source Anchor uniqueness.
**Validation:** No duplicate identities, users have correct user type, licenses applied correctly.

---

## 6.15 KNOWLEDGE TEST

1. **What is the difference between Entra Cloud Sync and Entra Connect (full)?**
   Cloud Sync uses a lightweight agent and cloud-managed connector infrastructure (preferred for new deployments). Entra Connect (full) uses a Windows Server VM that you manage, supports Password Writeback natively, and is for environments with specific legacy requirements.

2. **What is the difference between synchronization and authentication?**
   Synchronization = directory data flows (user attributes, group membership). Authentication = how users prove their identity (PHS, PTA, Federation). They are independent — you can sync without authenticating in the cloud.

3. **What is Source Anchor?**
   The unique identifier (currently `sourceAnchor` = `objectGUID`) that links an on-prem AD DS object to its Entra ID counterpart. Used by the sync engine to match objects between directories.

4. **What is the difference between `sourceAnchor` and `immutableId`?**
   `sourceAnchor` is the newer attribute (binary `objectGUID`). `immutableId` is the older attribute (Base64-encoded `objectGUID`). Microsoft recommends `sourceAnchor` for new deployments.

5. **What is the difference between PHS, PTA, and Federation?**
   PHS: Password hash synced to cloud, authentication in cloud. PTA: Password validated on-prem in real-time, agent validates against AD DS. Federation: Authentication handled by on-prem AD FS, SAML/WS-Fed response returned.

6. **Why is PTA high availability critical?**
   If all PTA agents are down, users cannot authenticate via PTA. If PHS is also configured, users can still authenticate via PHS. If only PTA is configured, users are completely locked out.

7. **Can Cloud Sync do Password Writeback?**
   No. Cloud Sync is synchronization-only. Password Writeback requires PTA (configured separately) or Entra Connect (full).

8. **What is Soft Match vs Hard Match for Hybrid Join?**
   Soft Match: Cloud Sync automatically links on-prem device to existing Entra device based on matching attributes (modern, recommended). Hard Match: Entra Connect manually links devices (legacy).

9. **What happens if all PTA agents go down and PHS is not configured?**
   Users cannot authenticate to cloud services at all. All cloud authentication fails because PTA cannot validate passwords and there's no PHS hash to validate against.

10. **What is Staging Mode?**
    A testing mode where sync configuration is validated without actually applying changes to Entra ID. After validation, switch to production mode.

11. **What are the two types of duplicate identities in Entra ID?**
    (1) Same user as both Member and Guest. (2) Same Source Anchor / UPN on two different objects. Both are flagged for consolidation.

12. **Why is device sync important for Conditional Access?**
    Device sync provides device state information (Hybrid Joined, Registered, Compliant) to Conditional Access. Without it, device-based CA conditions cannot be evaluated for on-prem devices.

13. **What is Password Writeback?**
    When a user changes their password in the cloud (SSPR, my profile), the change is written back to on-prem AD DS. Requires PTA or Entra Connect with Password Writeback.

14. **How does PHS handle password changes from both sides?**
    With PTA/Password Writeback: on-prem password change → hash syncs to cloud. Cloud password change → PTA writes back to on-prem. Without writeback: only on-prem → cloud (PHS one-way).

15. **What is the recommended authentication configuration for a hybrid enterprise?**
    PHS + PTA (both configured). PHS as resilient fallback (works without on-prem connectivity), PTA for day-to-day (no password hash in cloud). Password Writeback enabled via PTA.

16. **What is the difference between "Marked as compliant" and device registration?**
    "Marked as compliant" is a property in the directory (set by Intune/MDM or CA). Device registration is the device being registered with Entra ID (creating an Entra device object). Hybrid Joined = both domain-joined AND Entra registered.

17. **What is the connector account and what permissions does it need?**
    The account used by Entra Connect/Cloud Sync to read from AD DS. Needs: read access to selected OUs, read on all user/group/device attributes being synced, ability to read computer objects. Does NOT need write access to AD DS.

18. **What is the default sync frequency for Cloud Sync?**
    Delta sync occurs automatically every 2-5 minutes (configurable). Full sync occurs every 6-8 hours. Can be forced manually.

---

## 6.16 L3 GAP CHECK

| Topic | Status |
|-------|--------|
| Entra Connect vs Cloud Sync distinction | ✅ Covered |
| Sync architecture (agent, connector, engine) | ✅ Covered |
| Source Anchor / Immutable ID | ✅ Covered |
| sourceAnchor vs immutableId | ✅ Covered |
| Synchronization Engine (delta, dependency, conflict) | ✅ Covered |
| Sync rules (inbound, outbound, join, projection, exclusion) | ✅ Covered |
| Filtering (OU, group, attribute) | ✅ Covered |
| Filtering troubleshooting | ✅ Covered |
| Staging mode | ✅ Covered |
| High availability | ✅ Covered |
| PHS (deep) | ✅ Covered |
| PTA (deep) | ✅ Covered |
| Federation (deep) | ✅ Covered |
| Password Writeback | ✅ Covered |
| PHS vs PTA decision | ✅ Covered |
| Device synchronization | ✅ Covered |
| Device Sync / Hybrid Join | ✅ Covered |
| Soft Match vs Hard Match | ✅ Covered |
| Duplicate identities | ✅ Covered |
| Complete authentication flow (hybrid) | ✅ Covered |
| Synchronization and Authentication distinction | ✅ Covered (Critical) |
| Connector health and logs | ✅ Covered |
| AD DS connectivity | ✅ Covered |
| Security considerations | ✅ Covered |
| Monitoring targets | ✅ Covered |
| Production example | ✅ Covered |
| Failure scenarios | ✅ Covered |
| Troubleshooting methodology | ✅ Covered |
| Evidence sources | ✅ Covered |
| Current service considerations | ✅ Covered |
| L3 interview questions (all levels) | ✅ Covered |
| Scenario-based questions | ✅ Covered |
| Knowledge test | ✅ Covered |
| L3 gap check | ✅ Covered |

---

## 6.17 WHAT AN EXPERIENCED AZURE L3 ENGINEER SHOULD NOW BE ABLE TO EXPLAIN CONFIDENTLY

After Module 6, you should be able to confidently explain:

1. **The critical distinction between synchronization and authentication** — Syncing user attributes to the cloud does NOT mean authentication happens in the cloud. Authentication is managed separately via PHS, PTA, or Federation. This is the #1 conceptual confusion in hybrid identity.

2. **Why Cloud Sync is preferred over Entra Connect (full) for new deployments** — Cloud Sync has no on-prem server to manage, supports granular OU and attribute filtering, is automatically HA, and is Microsoft's current recommendation. Entra Connect (full) is for environments needing PTA/Password Writeback or legacy federation.

3. **The three authentication methods (PHS, PTA, Federation) and when each is appropriate** — PHS is simplest and most resilient (works without on-prem connectivity). PTA is most secure (no password hash in cloud) but requires on-prem connectivity. Federation is most complex (AD FS infrastructure) and declining.

4. **Why PTA requires high availability** — All PTA agents going down means cloud authentication fails (if PHS isn't configured as fallback). Minimum 2 PTA agents for automatic failover.

5. **Why Cloud Sync cannot do Password Writeback** — Cloud Sync is synchronization-only by design. Password Writeback requires PTA (configured separately) or Entra Connect (full). This is a common misconception.

6. **How Source Anchor links on-prem and cloud objects** — `sourceAnchor` (typically `objectGUID`) is the match key. If it's wrong, missing, or duplicated, objects won't sync or will be linked incorrectly.

7. **How Soft Match works for Hybrid Join** — Cloud Sync automatically links on-prem domain-joined devices to Entra ID device objects by matching device attributes, making the device show as "Hybrid Joined" rather than just "Registered."

8. **How to troubleshoot "users not syncing"** — Connector health → AD DS connectivity → User properties (UPN, UsageLocation, ObjectGUID) → Sync scope (OU filtering) → Sync errors → Force sync → Observe specific errors.

9. **How duplicate identities are created and resolved** — Usually from inviting users as guests before syncing them as members. Resolution is consolidation (delete one, keep the other), with Source Anchor as the verification.

10. **How hybrid authentication flows end-to-end** — From user sign-in, through tenant lookup, authentication method determination (PHS/PTA/Federation), CA evaluation, token issuance, to resource access. Understanding that each authentication path has different failure modes and troubleshooting approaches.

11. **Why the on-prem AD DS is a critical dependency** — For PTA, Federation, and device sync, on-prem AD DS must be healthy. A DC outage can affect cloud authentication (PTA, Federation) or device sync, even though the cloud directory is separate.

12. **The importance of monitoring both directories** — Hybrid identity requires monitoring on-prem (AD DS health, connector health, PTA agents) AND cloud (Entra ID sync status, authentication health, CA compliance). A problem in either directory affects the other.

---

# Ready for Module 7 — RBAC (Azure RBAC, Directory Roles, Role Assignments, Custom Roles, ABAC)?

It covers:
- Azure RBAC deep dive (built-in roles, custom roles, role assignments, scope, inheritance)
- Directory Roles vs Azure RBAC (critical distinction)
- Deny assignments
- Role Assignment Conditions / ABAC
- Least privilege principles
- Managed identity permissions
- Troubleshooting common RBAC issues
- And all 19 required module components

Say **"Next module"** to continue.