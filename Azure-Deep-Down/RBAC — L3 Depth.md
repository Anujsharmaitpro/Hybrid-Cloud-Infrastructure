Understood. Starting from **Module 7 — RBAC — L3 Depth**. Modules 1–6 were covered in the previous thread, so I'll briefly note the bridge and then go deep into RBAC.

---

## ROADMAP PROGRESS

| Module | Status |
|--------|--------|
| 1–6 | ✅ Covered in previous thread (Azure Architecture → ARM → Entra ID Deep → Authentication/Tokens → Conditional Access → Entra Connect/Hybrid Identity) |
| **7** | **🔴 RBAC — L3 Depth — STARTING NOW** |
| 8–73 | 🔜 Pending |

### Bridge from Modules 1–6 → Module 7

> Modules 1–6 established **who you are** (Entra ID tenant), **how you authenticate** (tokens, Conditional Access), **what you manage** (resources, ARM), and **how hybrid identity connects** on-prem to cloud.
>
> Module 7 answers the fundamental question: **"What can you DO with all of that?"**
>
> RBAC is the intersection of **identity (Modules 3–5)** and **resources (Modules 1–2)**. Every permission question in Azure ultimately traces back to RBAC. If you get this module wrong, every troubleshooting scenario downstream — from "user cannot create a VM" to "deployment is denied" — will be misdiagnosed.

---

# MODULE 7 — RBAC — L3 DEPTH

---

## 1. CONCEPT

**Azure Role-Based Access Control (RBAC)** is Azure's native authorization system. It determines **what actions a user, group, service principal, or managed identity can perform** on Azure resources and at what scope.

### What RBAC Does

- Grants or denies access to Azure resources
- Defines permissions through **roles** (collections of actions)
- Assigns roles at **scopes** (management group → subscription → resource group → resource)
- Inherits permissions downward through the hierarchy
- Works alongside **Azure Policy** (which governs resource configuration, not user permissions)

### Core Principle for L3 Engineers

> **RBAC answers: "Can this identity do this action on this resource?"**
>
> Azure Policy answers: "Is this resource configured in an allowed/compliant way?"

These two are frequently confused. An L3 engineer must distinguish them instantly:

| Control | Governs | Example |
|---------|---------|---------|
| **RBAC** | What a user/service can DO | "Can this user create a VM?" |
| **Azure Policy** | What resources are ALLOWED/ENFORCED | "Are all VMs required to use Managed Disks?" |

---

## 2. ARCHITECTURE

```
Microsoft Entra Tenant
│
├── Directory Roles (Entra ID roles — global, conditional access admin, etc.)
│   └── These are ENTA roles, NOT Azure RBAC roles
│
├── Azure RBAC (authorization for Azure resources)
│   │
│   ├── Built-in Roles (e.g., Contributor, Reader, Owner, User Access Administrator)
│   ├── Custom Roles (defined by the organization)
│   │
│   ├── Role Assignments (the bridge: who + what role + at what scope)
│   │   │
│   │   ├── Management Group Scope (flows down to all subscriptions/RGs/resources below)
│   │   ├── Subscription Scope (flows down to all RGs/resources below)
│   │   ├── Resource Group Scope (flows down to all resources below)
│   │   └── Resource Scope (applies to a single resource only)
│   │
│   ├── Deny Assignments (always override — deny wins over allow)
│   │
│   └── Role Assignment Conditions (ABAC — conditional permissions on resource properties)
│
└── Resources
    ├── Inherited permissions from parent scopes
    ├── Explicit permissions from direct role assignments
    └── Deny assignments that block even if a role allows
```

### Architecture Principles

1. **RBAC is an authorization layer** — it sits between an identity and the resource's control plane (ARM). It does NOT touch the data plane.
2. **Role assignments are the only way permissions are granted** — having a role definition without an assignment grants nothing.
3. **Scope is everything** — a role assignment at Resource scope cannot grant access at Subscription scope.
4. **Inheritance flows DOWN only** — a child resource cannot grant permissions to a parent.
5. **Deny assignments ALWAYS win** — even if a role allows an action, a deny assignment at any applicable scope blocks it.
6. **Azure RBAC and Entra Directory Roles are separate systems** — an engineer with the "Global Administrator" Entra role does not automatically have Azure RBAC permissions, and vice versa.

---

## 3. COMPONENTS

### 3.1 Roles

#### 3.1.1 Built-in Roles

Azure ships with **100+ built-in roles**. These fall into categories:

| Category | Examples | Scope of Use |
|----------|---------|-------------|
| **Owner** | Owner | Full access + can assign roles. Can delete resources. Most powerful. |
| **Contributor** | Contributor | Can create/manage all resources but cannot assign roles. Cannot grant access to others. |
| **Reader** | Reader | Read-only. Cannot see secrets in Key Vault even if they have access. |
| **User Access Administrator** | User Access Administrator | Can manage RBAC role assignments only. |
| **Key Vault Administrator** | Key Vault Administrator | Manage Key Vault and secrets. |
| **Network Contributor** | Network Contributor | Manage networking resources. |
| **Storage Blob Data Contributor** | Storage Blob Data Contributor | Read/write blob data. |
| **Monitoring Contributor** | Monitoring Contributor | Manage monitoring. |
| **Security Admin** | Security Admin | Manage Defender for Cloud. |

**L3 Critical Built-in Roles — You Must Know These Cold:**

| Role | Key Permission | Common Mistake |
|------|---------------|----------------|
| **Owner** | Full access + role assignment + delete | Engineers assume "Owner = can do everything" — but Owner still respects Deny Assignments and Azure Policy |
| **Contributor** | Manage all resources EXCEPT role assignments | Most common role given to engineers. "I have Contributor but I can't grant access to others" — because Contributor cannot assign roles |
| **Reader** | Read everything | Readers CANNOT read secrets in Key Vault — this surprises many engineers. Reader sees the Key Vault exists but cannot retrieve secrets |
| **User Access Administrator** | ONLY manage RBAC assignments | This role alone cannot manage any resources — only who has what role |
| **Custom roles named "e.g., Key Vault Secrets Officer"** | Read secrets but not manage the vault | Sometimes engineers confuse these with full access |

> **Interview Trap Question:** "A user has the Owner role on a resource group. They report 'Access Denied' when trying to delete a resource. Why?"
>
> ✅ Possible answer: A **Deny Assignment** is in place, OR **Azure Policy** is denying the action, OR a **resource lock** (CanNotDelete) is set, OR the resource is **locked** at subscription/RG level.

---

#### 3.1.2 Custom Roles

Custom roles let you create **precise, least-privilege permissions** not covered by built-in roles.

| Property | Detail |
|----------|--------|
| Definition location | Can be defined in JSON (ARM template/Bicep/portal) |
| Actions list | `"Microsoft.Compute/virtualMachines/start/action"`, `"*"` (wildcard) |
| NotActions list | Explicitly excludes actions even if inherited from a parent role |
| Data Actions | For data-plane operations (e.g., `Microsoft.Storage/storageAccounts/listKeys/action`) |
| Data Actions NotActions | Exclude specific data-plane actions |
| Assignable scopes | Where the custom role CAN be assigned (subscription, RG, management group) |
| Permissions | Sum of Actions minus NotActions intersected with the scope |
| Minimum permissions principle | Best practice: start empty, add only what's needed |

**Custom Role JSON structure (conceptual):**

```json
{
  "Name": "VM Operator Custom",
  "Description": "Can start/stop/restart VMs only",
  "Actions": [
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/deallocate/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/read"
  ],
  "NotActions": [],
  "DataActions": [],
  "DataActionsNotActions": [],
  "AssignableScopes": [
    "/subscriptions/xxxx-subscription-id"
  ]
}
```

**L3 Operational Impact:** In enterprise environments, engineers almost always need custom roles because built-in roles are too broad. "Contributor" is too powerful for day-to-day operations; a custom "VM Operator" role that only allows start/stop/restart/read is more appropriate.

---

#### 3.1.3 App Registration vs Enterprise Application vs Service Principal — REVISITED FOR RBAC CONTEXT

Since this builds on Module 3, here's the critical RBAC-specific understanding:

| Concept | What It Is | RBAC Relevance |
|---------|-----------|---------------|
| **App Registration** | An identity definition in Entra ID (app identifier, redirect URIs, certificates/secrets) | The App Registration itself has NO permissions to Azure resources. It is NOT a role assignment. |
| **Service Principal** | The actual identity instance created when an App Registration is instantiated in a tenant. It is the object that receives role assignments. | Service principals receive RBAC role assignments just like users. A Service Principal is the RBAC identity, the App Registration is its definition. |
| **Enterprise Application** | A representation of the service principal in the Enterprise Applications blade. Controls what the app can access. | Enterprise Applications are managed for consent, SSO, and provisioning — but RBAC assignments are done on the Service Principal, not the Enterprise Application object directly. |

> **Critical L3 distinction:** When you assign RBAC to an app, you are assigning to the **Service Principal**, not the App Registration. The Enterprise Application blade is where you manage enterprise SSO/consent. Do NOT confuse them.

---

### 3.2 Role Assignments

A **role assignment** binds:
- **Who** (principal: user, group, service principal, managed identity)
- **What role** (built-in or custom role definition)
- **At what scope** (management group, subscription, resource group, or resource)

Each role assignment is unique — the same principal can have different roles at different scopes.

**Role Assignment Properties:**

| Property | Detail |
|----------|--------|
| Principal ID | The object ID of the user/group/service principal |
| Principal Type | User, Group, ServicePrincipal, Device |
| Role Definition ID | The GUID of the role being assigned |
| Scope | The resource ID of the scope (MG, Sub, RG, or Resource) |
| Delegation | Whether the assignment can be delegated to a resource provider (for custom roles) |
| Condition | ABAC condition (if applicable) |
| Condition version | `2.0` (latest) or `1.0` |

---

### 3.3 Scope — THE MOST CRITICAL RBAC CONCEPT

Scope defines **where a role assignment applies**. Scope is hierarchical:

```
Management Group Scope (broadest)
  ↓ inherited to all below
Subscription Scope
  ↓ inherited to all below
Resource Group Scope
  ↓ inherited to all below
Resource Scope (narrowest)
```

#### Scope Examples

| Scope Level | Example Scope ID | What It Grants |
|------------|-----------------|----------------|
| **Management Group** | `/providers/Microsoft.Management/managementGroups/MG-Prod` | Role applies to ALL subscriptions, RGs, and resources in that MG and its children |
| **Subscription** | `/subscriptions/abc123` | Role applies to ALL RGs and resources in that subscription |
| **Resource Group** | `/subscriptions/abc123/resourceGroups/RG-Prod-West` | Role applies to ALL resources in that RG |
| **Resource** | `/subscriptions/abc123/resourceGroups/RG-Prod-West/providers/Microsoft.Compute/virtualMachines/VM-01` | Role applies to ONLY that single VM |

#### Scope Inheritance Rules

1. A principal has the **union** of all role assignments that apply to it — from all levels of the hierarchy
2. A role assignment at a **parent scope** automatically applies to all children
3. A role assignment at a **child scope** does NOT apply to parents or siblings
4. **Deny assignments** at ANY level block the action regardless of allowing role assignments at other levels
5. Multiple role assignments can stack — a user can have Reader at Sub scope and Contributor at RG scope — they effectively have both (but deny still wins)

> **L3 Interview Critical Answer:**
>
> "Why does User A have access to Resource X?"
>
> **Correct investigation path:**
> 1. Check if User A has any role assignments at Resource scope for X
> 2. Check RG scope for X
> 3. Check Subscription scope
> 4. Check Management Group scope
> 5. Check if any Deny Assignment is blocking
> 6. Check if the user is in a group that has a role assignment
> 7. Check if Entra Directory Role is involved (not RBAC)

---

### 3.4 Inheritance

**RBAC inheritance flows DOWN through the scope hierarchy only.**

```
Subscription: Contributor assigned
  ├── RG-1: No explicit role assignment → INHERITS Contributor from Subscription
  │   ├── VM-1: No explicit assignment → INHERITS Contributor
  │   └── VM-2: No explicit assignment → INHERITS Contributor
  └── RG-2: Contributor explicitly removed (no role assignment here)
      └── VM-3: No assignment → Does NOT inherit from Subscription? 
```

Wait — that last point is critical:

> **If a role assignment is NOT made at RG-2, and the user is not explicitly assigned at VM-3's level, does the user still inherit from the subscription?**

**YES.** Inheritance flows down. If there is NO explicit role assignment at a child level, the parent's role assignments still apply. There is no mechanism to "block" inheritance in RBAC (unlike Group Policy inheritance blocking in Active Directory). The ONLY way to restrict is through **Deny Assignments**.

**Key Rules:**

| Rule | Behavior |
|------|----------|
| Assignment at parent scope | Applies to all children unless overridden by Deny |
| Assignment at child scope | Does NOT apply to parent or siblings |
| No assignment at child | Inherits from parent — no blocking possible via RBAC alone |
| Deny Assignment at any level | Blocks action even if role allows it |
| Multiple allowing assignments | User gets the UNION of all permissions |

---

### 3.5 Deny Assignments

**Deny Assignments are the one mechanism that CAN override role assignments.**

| Property | Detail |
|----------|--------|
| Effect | Always denies the specified actions |
| Precedence | DENY always wins over ALLOW — even Owner role cannot bypass a Deny Assignment |
| Scope | Can be set at Management Group, Subscription, RG, or Resource scope |
| Applies to | All principals within that scope (except those who are explicitly exempted via another scope's allow) |
| Relationship to Policy | Different from Policy "Deny" effect — Deny Assignment is an RBAC mechanism, Policy Deny is a governance mechanism, both block but through different systems |

**Common Deny Assignment Scenarios in Enterprise:**

| Scenario | Why |
|----------|-----|
| Preventing resource creation in subscription West US 2 | Cost control — restrict deployment to specific regions |
| Preventing deletion of production resources | Protection against accidental deletion |
| Blocking public IP creation | Security policy enforcement via Deny |
| Preventing Contributor from modifying network security | Security boundary for network team |

> **L3 Critical:** When investigating "Why can't this user do X even though they have the Owner role?" — the answer is almost always one of:
> 1. Deny Assignment
> 2. Azure Policy (Deny effect)
> 3. Resource Lock (CanNotDelete/ReadOnly)
> 4. The resource is NOT in the scope where the role is assigned

---

### 3.6 Role Assignment Conditions (ABAC — Attribute-Based Access Control)

ABAC allows conditional role assignments based on **resource properties** (tags, location, SKU, etc.).

| Property | Detail |
|----------|--------|
| Availability | Preview/GA — available on select roles |
| Condition type | String operators: `StringEquals`, `StringNotEquals`, `StringLike`, `StringNotLike` |
| Condition key | `tag`, `location`, `sku`, `resourceGroup`, etc. |
| Condition version | `2.0` — latest, supports more operators and keys |
| Effect | If condition evaluates to true, the role assignment applies; if false, it does not |
| Use case | "Allow this user to manage VMs only if tag `environment = production`" |

**Example:** A role assignment with condition:

```
[StringEquals(resource[tags].environment, "production")]
```

This means: "This role assignment is ACTIVE only for resources where the `environment` tag equals `production`."

**L3 Operational Impact:** ABAC enables fine-grained least privilege without creating dozens of custom roles. It's increasingly used in enterprise environments instead of (or in addition to) multiple custom roles.

---

### 3.7 Microsoft Entra Directory Roles vs Azure RBAC — THE CRITICAL DISTINCTION

This is the **#1 source of confusion** for Azure engineers. Let me be absolutely clear:

| Aspect | **Entra Directory Roles** | **Azure RBAC Roles** |
|--------|-------------------------|---------------------|
| **What they govern** | Entra ID operations (identity, users, groups, apps) | Azure resource operations (VMs, Storage, Networking) |
| **Where assigned** | Entra ID blade → Directory Roles | Azure portal → Access Control (IAM) |
| **Scope** | Tenant-wide (entire Entra tenant) | Resource, RG, Subscription, or MG scope |
| **Examples** | Global Administrator, Conditional Access Administrator, Privileged Role Administrator, Security Administrator, Helpdesk Administrator | Owner, Contributor, Reader, User Access Administrator, Key Vault Administrator, Network Contributor |
| **Can they manage VMs?** | NO — Directory Roles do NOT grant any resource-level permissions | YES — if assigned with appropriate role at resource scope |
| **Can they manage Entra ID?** | YES — Global Admin can manage everything in Entra | NO — RBAC has no authority in Entra ID |
| **Can Global Admin assign RBAC?** | NO — Global Admin is an Entra role, not an Azure RBAC role | Only if explicitly assigned the appropriate Azure RBAC role (e.g., Owner, User Access Administrator) |
| **Can RBAC manage Entra ID?** | NO — RBAC has no authority over Entra ID objects | — |

**Real Enterprise Scenario:**

> A Global Administrator cannot create a VM. Period. Global Admin is an Entra ID role — it governs identity operations, not resource operations. To create a VM, the user needs an Azure RBAC role (e.g., Contributor or Owner) assigned at the subscription or resource group scope.

Conversely:

> A user with the Owner role on a subscription can manage all resources in that subscription but CANNOT reset a user's password in Entra ID. That requires the Global Administrator or Privileged Role Administrator Entra role.

**Entra ID Directory Roles That Matter for L3:**

| Directory Role | What It Controls |
|---------------|-----------------|
| Global Administrator | Full control of Entra ID — can reset passwords, manage all settings |
| Privileged Role Administrator | Can assign/remove privileged directory roles |
| Conditional Access Administrator | Can create/manage CA policies |
| Security Administrator | Can manage security defaults and alerts in Entra |
| Helpdesk Administrator | Can reset passwords, read directory audit logs |
| Application Administrator | Can manage app registrations and enterprise apps |
| Cloud Application Administrator | Can manage app registrations, certificates, secrets |
| Domain Administrator | Admin for a domain in the tenant |
| Exchange Administrator (Microsoft 365 overlap) | Exchange-related settings |

---

### 3.8 Group-Based Role Assignments

Roles can be assigned to **Azure Security Groups** (security groups in Entra ID, NOT AD DS security groups).

**How it works:**
1. An Entra ID Security Group is created
2. The role (e.g., Contributor) is assigned to the group at a scope
3. ALL members of the group inherit that role at that scope
4. When a user is added/removed from the group, their permissions change automatically
5. **Nested groups:** Entra ID supports nesting security groups, but for RBAC, the role assignment is on the group object itself, so nested membership resolves

**L3 Operational Impact:**
- Enterprise best practice: assign roles to groups, NOT individual users
- Group membership = permission boundary
- When an engineer says "User X should have access but doesn't" — check if User X is a member of the correct security group
- **Dynamic groups:** Entra ID supports dynamic membership rules based on user attributes (department, jobTitle, etc.). If the rule changes, membership changes, and RBAC changes with it.

---

### 3.9 Managed Identity Permissions

Managed identities are **service principals** in Entra ID that are automatically created and managed by Azure. They receive RBAC role assignments exactly like any other service principal.

| Identity Type | What It Is | RBAC Assignment |
|--------------|-----------|----------------|
| **System-assigned** | Tied to the resource lifecycle (created with VM, deleted with VM) | RBAC role assigned to the resource's system identity |
| **User-assigned** | Standalone Azure resource that can be attached to multiple resources | RBAC role assigned to the user-assigned identity resource itself |

**Important:** When you assign a role to a managed identity, you assign it to the **identity's principal ID** (a GUID), not the resource itself. The resource references the identity, and when the resource makes an API call, it presents the identity's token, which is validated against the RBAC role assignments for that principal ID.

---

## 4. SUBCOMPONENTS — Quick Reference

| Subcomponent | What It Defines | Example |
|-------------|----------------|---------|
| **Role Definition** | A list of allowed/actions (permissions) | `Microsoft.Compute/virtualMachines/start/action` |
| **Role Assignment** | Who + Which Role + At What Scope | User Bob → Contributor → RG-Prod-West |
| **Principal** | The identity receiving the assignment | User, Group, Service Principal, Managed Identity |
| **Scope** | The boundary where the assignment applies | Subscription, RG, Resource, MG |
| **Condition (ABAC)** | Resource property filter | `tag = production` |
| **Deny Assignment** | Always-deny override | Block all write actions at MG scope |
| **NotActions** | Permissions explicitly excluded from a role | Exclude `Microsoft.Compute/virtualMachines/delete` from Contributor |

---

## 5. DEPENDENCIES

| RBAC Depends On | Why |
|----------------|-----|
| **Microsoft Entra ID** | Principals (users, groups, SPs) live in Entra ID. No Entra ID = no RBAC assignments |
| **Azure Resource Manager (ARM)** | RBAC is enforced by ARM at the authorization layer before a request reaches the resource provider |
| **Resource Providers** | The resource provider validates that the authenticated identity has the required RBAC permission for the action being performed (or ARM pre-validates and blocks) |
| **Resource Locks** | Locks operate independently of RBAC — even with Owner role, CanNotDelete lock prevents deletion |
| **Azure Policy** | Policy can deny actions that RBAC would allow — dual enforcement layer |
| **Management Groups** | Provide the top-level scope for RBAC inheritance |

**Dependency Chain:**

```
Entra ID (holds principals)
  ↓
ARM (enforces authorization)
  ↓
Resource Provider (executes action if authorized)
  ↓
But: Locks, Policy, Deny Assignments can block at ANY point
```

---

## 6. COMMUNICATION FLOW

When a user attempts an action on a resource:

```
Step 1: User authenticates → Entra ID issues an access token (covered in Module 4)
         │
Step 2: Token sent to ARM (Azure Resource Manager)
         │
Step 3: ARM checks: "Is this token valid? Is the principal in this token assigned a role that allows this action at this scope?"
         │
Step 4a: DENY ASSIGNMENT exists at any applicable scope? → BLOCKED (403)
         │
Step 4b: Role assignment found that allows this action? → ALLOWED → proceed to Step 5
         │
Step 4c: No role assignment found? → BLOCKED (403)
         │
Step 5: ARM forwards request to the Resource Provider (e.g., Microsoft.Compute)
         │
Step 6: Resource Provider executes the action
         │
Step 7: But wait — did Azure Policy also evaluate? For write operations, Policy runs BEFORE the resource provider:
         Policy says Deny? → BLOCKED (403) with Policy compliance status
         Policy says Audit? → ALLOWED but flagged as non-compliant
         Policy says Modify? → ALLOWED, resource is modified to comply
```

**Authorization Evaluation Order:**

1. **Azure Policy** (for write operations — evaluated first)
2. **Deny Assignments** (RBAC-level deny)
3. **Role Assignments** (is there an allowing role?)
4. **Resource Locks** (CanNotDelete/ReadOnly)
5. **Resource Provider** executes the action

---

## 7. INTERNAL WORKFLOW

### What Happens When a Role Assignment Is Created

1. **Request sent to ARM** — `PUT` to `{scope}/providers/Microsoft.Authorization/roleAssignments/{uuid}`
2. **ARM validates** — Is the caller authorized to create role assignments? (requires `Microsoft.Authorization/roleAssignments/write` permission — usually Owner or User Access Administrator)
3. **Principal validated** — Does the principal ID exist in Entra ID?
4. **Role definition validated** — Does the role definition ID exist?
5. **Scope validated** — Is the scope a valid management group, subscription, RG, or resource?
6. **Assignment created** — Stored in ARM's authorization database
7. **Propagation** — The assignment takes effect immediately (no propagation delay)
8. **Audited** — Activity Log records the role assignment operation

### What Happens When an Action Is Evaluated

1. **Token presented** — The caller's access token contains the object ID (OID) of the principal
2. **ARM receives request** — For any resource operation, ARM first checks authorization
3. **Entra ID call** — ARM calls Entra ID to check Directory Role (for built-in directory roles, not RBAC) — actually this is cached, and RBAC is evaluated from stored assignments
4. **RBAC evaluation** — ARM looks up all role assignments for the principal at the resource scope, then RG scope, then Subscription scope, then MG scope
5. **Deny Assignment check** — At each level, ARM also checks for Deny Assignments
6. **Policy evaluation** — For write/delete operations, Policy evaluates
7. **Decision** — Allow or Deny (HTTP 200 or 403)

### What Happens When a Role Assignment Is Removed

1. **DELETE request to ARM** — role assignment is removed
2. **Immediate effect** — The principal no longer has that role at that scope
3. **No propagation delay** — the effect is instantaneous
4. **Activity Log** — recorded

---

## 8. ADMINISTRATION

### How to View Role Assignments

| Method | How |
|--------|-----|
| **Azure Portal** | Resource → Access Control (IAM) → Roles tab → check assignments |
| **Azure Portal** | Subscription → Access Control (IAM) → see all assignments at Sub scope |
| **Management Group** | In Azure Portal, open the MG → Access Control (IAM) → assignments at MG scope |

### How to Grant a Role

1. Navigate to the scope (MG, Sub, RG, or Resource)
2. Go to **Access Control (IAM)**
3. Click **Add** → **Add role assignment**
4. Select the role (built-in or custom)
5. Select the principal (user, group, service principal, managed identity)
6. (Optional) Set ABAC condition
7. Click **Review + Assign**

### How to Create a Custom Role

1. **Option A (from existing):** Start from a built-in role → remove unnecessary permissions → add required ones → assignable scopes
2. **Option B (from scratch):** Define in JSON → deploy via ARM/Bicep → assign
3. **Validate:** Assign to yourself in a test RG → try the actions → adjust as needed
4. **Deploy via IaC:** Custom roles should be stored in Git and deployed via ARM/Bicep for auditability

### Best Practices for L3

| Practice | Why |
|----------|-----|
| **Least privilege** | Never assign Owner unless absolutely necessary |
| **Use groups for assignments** | Easier to manage than individual user assignments |
| **Custom roles for operational tasks** | "VM Operator" instead of "Contributor" |
| **Audit role assignments quarterly** | Remove stale assignments |
| **Avoid Global Admin in Entra ID** | Use Privileged Role Administrator + just-in-time elevation |
| **Use Deny Assignments for guardrails** | Restrict regions, resource types |
| **Document all assignments** | Maintain a RBAC matrix document |
| **Use ABAC for dynamic scenarios** | Tag-based access instead of multiple custom roles |

---

## 9. SECURITY

### Security Considerations for RBAC

| Concern | Detail | Mitigation |
|---------|--------|-----------|
| **Over-provisioning** | Too many users with Contributor or Owner | Audit monthly, implement least privilege |
| **Stale assignments** | Users who left the org still have roles | Automated deprovisioning tied to HR system, regular audits |
| **Service principal secrets** | SPs with Contributor and long-lived secrets | Rotate secrets, use managed identities where possible |
| **Privilege escalation** | User with role assignment permission grants themselves Owner | Monitor role assignment changes in Activity Log, use Deny Assignments |
| **Orphaned assignments** | When a user is deleted, their role assignments become "stale" | Review Activity Log for stale assignments |
| **Nested group explosion** | Deeply nested groups make permission analysis difficult | Limit nesting, document group hierarchy |
| **Global Admin sprawl** | Multiple Global Admins | Limit to emergency break-glass accounts only |
| **Conditional Access bypass** | Service principals may bypass CA | Use CA for "All applications" and specifically include service principals |

### Break-Glass Accounts

Enterprise pattern:

- **Break-glass accounts** are Global Administrators in Entra ID
- They are NOT assigned any Azure RBAC role by default
- In an emergency, they are temporarily assigned Owner at subscription scope
- **Just-in-time elevation** — elevation is granted via Azure AD Privileged Identity Management (PIM) or manual process with approval
- After emergency resolved, break-glass RBAC role is removed
- Activity Log is monitored for any break-glass usage

---

## 10. MONITORING

### Monitoring RBAC Changes

| What to Monitor | How | Activity Log Category |
|----------------|-----|-----------------------|
| Role assignments created | Activity Log → `Add role assignment` | Audit |
| Role assignments deleted | Activity Log → `Delete role assignment` | Audit |
| Role definitions created | Activity Log → `Add role definition` | Audit |
| Custom role modifications | Activity Log → `Update role definition` | Audit |
| Role assignments for break-glass | Filter Activity Log by role name | Audit |

### Key Activity Log Operations for RBAC

| Operation | Display Name | Significance |
|-----------|-------------|-------------|
| `Add role assignment` | Granted access | Who got what role at what scope? |
| `Delete role assignment` | Revoked access | Who removed what role? |
| `Add role definition` | Created role | New custom role created |
| `Update role definition` | Modified role | Permissions changed |
| `Delete role definition` | Deleted role | Custom role removed |
| `Write roleAssignments` | Bulk changes | PowerShell/CLI bulk assignment changes |

### Alerting Recommendations

- **Alert on** `Add role assignment` where role = Owner or User Access Administrator
- **Alert on** role assignment to unusual users (not in standard groups)
- **Alert on** `Delete role assignment` for critical roles
- **Monthly report** — export all role assignments via `Get-AzureRmRoleAssignment` (PowerShell) or `az role assignment list` (CLI) — review

---

## 11. PRODUCTION EXAMPLE

### Scenario: Enterprise "VM Operations Team" Needs Start/Stop Access

**Requirement:** The "VM Ops" team needs to start, stop, and restart VMs in the Production resource group but should NOT be able to delete, create, or modify any other resources.

**Naive Approach (Wrong):**
- Assign `Contributor` at RG scope → ❌ Too broad — they can delete, create, modify everything

**Better Approach (Better):**
- Assign `Reader` + specific built-in roles like `Virtual Machine Operator` → Gets closer, but `Virtual Machine Operator` includes some management plane access beyond start/stop

**Best Approach (L3):**
- Create a **Custom Role** with only these actions:
  - `Microsoft.Compute/virtualMachines/read`
  - `Microsoft.Compute/virtualMachines/start/action`
  - `Microsoft.Compute/virtualMachines/deallocate/action`
  - `Microsoft.Compute/virtualMachines/restart/action`
  - `Microsoft.Compute/virtualMachines/listAllComponents/action` (for monitoring in some scenarios)
  - `Microsoft.Network/virtualNetworks/subnets/join/action` (for NIC operations if needed)
- Assign this custom role to an Entra Security Group called `SG-VMOps-Prod`
- Assign the role at the `RG-Prod-West` scope
- Team members are added to `SG-VMOps-Prod`

**Result:**
- Team can start/stop/restart VMs
- Team CANNOT delete VMs, create resources, or modify anything else
- When someone leaves the team, remove them from the group → access is instantly revoked
- Audit trail exists in Activity Log

---

## 12. FAILURE SCENARIOS

### Failure 1: "I have Contributor but I cannot create a VM"

**Possible Causes (in order of likelihood):**

| # | Cause | Check |
|---|-------|-------|
| 1 | **Azure Policy Deny** — Policy blocks the VM SKU/region/zone | Check Policy compliance for the resource |
| 2 | **Quota exceeded** — vCPU quota exhausted | Check subscription quotas for the region |
| 3 | **Deny Assignment** — a Deny Assignment blocks VM creation | Check Deny Assignments at subscription/MG scope |
| 4 | **Role not actually assigned** — assignment not propagated or assigned at wrong scope | Check effective permissions (Access Review) |
| 5 | **Resource not in scope** — RG is in different subscription than where role was assigned | Verify RG → subscription mapping |
| 6 | **Resource SKU unavailable** — the specific VM size is not available in that region | Check available SKUs for the region |
| 7 | **Resource lock** — CanNotDelete or ReadOnly lock | Check resource locks on the RG or subscription |

### Failure 2: "User has Owner role but still gets Access Denied"

**Investigation Order:**
1. **Deny Assignment** — MOST LIKELY. Check all Deny Assignments at subscription, MG, and RG scope
2. **Azure Policy Deny** — Check if a Policy denies the action (the 403 response may include policy compliance info)
3. **Resource lock** — Check for CanNotDelete lock
4. **Scope mismatch** — Verify the Owner role is assigned at the correct scope (the resource may be in a different subscription)
5. **Entra ID issue** — Is the user's authentication valid? Token expired? Conditional Access blocking?
6. **Consistency delay** — Extremely rare, but RBAC changes can take up to a few seconds to propagate in rare edge cases

### Failure 3: "Group members are not getting the role"

**Possible Causes:**
- Role assigned to the **group** but the user is not actually a member (check membership)
- Role assigned to the group **directly** but nested group membership not resolving (check nested groups)
- The group is a **Security group in Entra ID** but the role was assigned to a different group with the same name (verify object ID)
- User is a **Guest user** in the tenant and the group is configured to exclude guests

### Failure 4: "Service Principal cannot access Storage Account"

**Investigation:**
1. Is the Service Principal's role assignment at the Storage Account scope? (Resource scope)
2. Is the role the right one? (e.g., `Storage Blob Data Contributor` — not just `Reader`)
3. Is there a **Storage Account firewall** blocking the SP's traffic? (Network restriction — separate from RBAC)
4. Is there a **Private Endpoint** DNS resolution issue?
5. Is the SP's identity correct? (Check the identity used for authentication — system vs user-assigned mismatch)
6. Is the SP's **certificate/secret expired**?

### Failure 5: "Reader can see Key Vault but cannot get secrets"

**Expected Behavior:**
- The `Reader` built-in role does NOT include `Microsoft.KeyVault/vaults/secrets/read` permission
- This is BY DESIGN — Reader is a management-plane role, and Key Vault secrets are data-plane
- To allow reading secrets, you need `Key Vault Secrets Reader` or `Key Vault Administrator` role

> **L3 Interview Must-Know:** "Reader cannot see secrets in Key Vault" — This is a common interview question designed to test whether you understand the difference between control plane and data plane permissions.

---

## 13. TROUBLESHOOTING METHODOLOGY

### "User cannot perform action X on Resource Y"

**Step-by-step RBAC troubleshooting:**

```
1. Verify authentication
   └── Is the user signed in? Is the token valid? Is CA blocking? (Module 5)

2. Verify the identity used
   └── Is it the correct user? Correct Service Principal? Correct Managed Identity?
       (Common mistake: using a different SP than the one assigned the role)

3. Check RBAC role assignments AT RESOURCE Y
   └── Go to Resource Y → Access Control (IAM) → Who has access?

4. Check RBAC role assignments at RESOURCE GROUP level
   └── Go to parent RG → Access Control (IAM)

5. Check RBAC role assignments at SUBSCRIPTION level
   └── Go to subscription → Access Control (IAM)

6. Check RBAC role assignments at MANAGEMENT GROUP level
   └── Go to parent MG → Access Control (IAM)

7. Check group membership
   └── Is the user in a group that has a role assignment? Check the group's object ID, not just its name

8. Check Deny Assignments
   └── At ALL applicable scopes — Deny overrides ALL allows

9. Check Azure Policy
   └── For write/delete operations — Policy can deny even with Owner role

10. Check Resource Locks
    └── CanNotDelete/ReadOnly locks at RG, subscription, or resource level

11. Check Activity Log
    └── Look for "Access denied" entries — they sometimes include the specific RBAC failure reason

12. Check Entra Directory Roles
    └── If the action involves Entra ID operations, the user needs the appropriate Directory role
```

---

## 14. LOGS / EVIDENCE

### Key Logs for RBAC Troubleshooting

| Log Source | What It Shows | How to Access |
|-----------|--------------|---------------|
| **Activity Log** | Role assignment operations (add/delete), resource operations, policy compliance | Azure Portal → Activity Log, or Azure Monitor/Log Analytics |
| **Audit Logs (Entra ID)** | Directory role changes, group membership changes, app consent | Entra ID portal → Audit Log |
| **Azure Policy compliance** | Whether a resource is compliant, what policy denied it | Azure Policy → Compliance dashboard |
| **Resource-specific logs** | Service-specific authorization failures (e.g., Storage logs) | Resource diagnostic settings → Log Analytics |
| **Sign-in logs** | Authentication events, CA evaluation results | Entra ID portal → Sign-in logs |

### Evidence to Collect for an Incident

1. **The exact error message** — usually includes `AuthorizationFailed` or `CallerRole` information
2. **Activity Log** filtered by the user's identity and time window
3. **Effective permissions** screenshot from the Azure portal (if available)
4. **Role assignment list** at all applicable scopes
5. **Deny Assignment list** at all applicable scopes
6. **Policy compliance status** for the resource
7. **Resource locks** on the resource and parent containers
8. **Group membership** verification

---

## 15. VERSION / CURRENT-SERVICE CONSIDERATIONS

### RBAC Changes and Current State (2025-2026)

| Change | Impact |
|--------|--------|
| **ABAC (Role Assignment Conditions) — Generally Available** | Now widely available for supported roles. Use tag-based conditions for least privilege. |
| **Privileged Identity Management (PIM) — GA** | Time-bound and approval-based role assignments. Use PIM for Owner/Contributor at high scopes. |
| **Custom role delegation** | Resource providers can delegate custom roles to their own RBAC. Affects which roles can be assigned at resource scope for specific providers. |
| **Built-in role updates** | Microsoft periodically updates built-in role permissions. Check what's in a built-in role before assuming it's safe — a role named "Contributor" may have had its actions modified over time. |
| **Microsoft Defender for Cloud RBAC** | New Defender roles (Security Contributor, Security Reader, etc.) separate from general Azure RBAC. |
| **"Owner" no longer implies "delete" via Deny** | Even Owner cannot delete if CanNotDelete lock exists. Owner is still the most powerful RBAC role. |

### Deprecated/Changed Items

| Old Approach | Current | Change Reason |
|-------------|---------|--------------|
| **"Co-Admin" (Classic)** | RBAC role assignments | Classic subscription co-admins were replaced by RBAC. Co-Admin still exists in some legacy contexts but new deployments use RBAC. |
| **"Admin" and "User" roles in Classic** | RBAC: Owner/Contributor/Reader | Classic roles are being fully replaced by RBAC. New subscriptions don't have classic admins. |
| **Access Control (ACL) for Storage** | RBAC + Firewalls | ACLs were the old way to control Storage access; now RBAC + Firewalls + Private Endpoints are the standard. |

---

## 16. L3 INTERVIEW QUESTIONS

### Basic

**Q: What is RBAC?**
- **Ideal Answer:** "RBAC is Azure's authorization system that controls what users, groups, service principals, and managed identities can do on Azure resources. It works by assigning roles at scopes. Roles define permissions (actions), and scopes define where those permissions apply."
- **Keywords:** Authorization, roles, scopes, principals, actions
- **Common Mistake:** Confusing RBAC with authentication ("RBAC logs you in") — RBAC is about authorization, not authentication

### Intermediate

**Q: What is the difference between a role and a role assignment?**
- **Ideal Answer:** "A role (role definition) is a list of permissions — a set of actions. A role assignment binds a principal to a role at a scope. Without a role assignment, a role definition has no effect. You can think of the role as 'what permissions exist' and the role assignment as 'who gets those permissions where.'"
- **Keywords:** Role definition, role assignment, principal, scope

**Q: What is the scope of a role assignment?**
- **Ideal Answer:** "Scope defines the boundary where a role assignment applies. It can be at Management Group, Subscription, Resource Group, or Resource level. Inheritance flows DOWN — a role assigned at Subscription scope applies to all RGs and resources in that subscription."

### L3

**Q: A user has the Owner role on a resource group but cannot delete a VM in that RG. Why?**
- **Ideal Answer:** "There are several possible reasons — I would check in this order:
  1. A **CanNotDelete resource lock** at the RG, subscription, or resource level
  2. A **Deny Assignment** blocking delete operations
  3. **Azure Policy** with a Deny effect preventing deletion
  4. The role assignment may be at a different scope than I initially assumed — I'd verify the exact scope
  5. Network/security control blocking (less likely for deletion but worth checking if the action is intercepted by a network appliance)"

**Q: How would you determine exactly what permissions a specific user has on a specific resource?**
- **Ideal Answer:** "I would check at each scope level:
  1. Resource-level RBAC assignments (direct)
  2. RG-level RBAC assignments (inherited)
  3. Subscription-level RBAC assignments (inherited)
  4. MG-level RBAC assignments (inherited)
  5. Check if the user is in any groups that have assignments (expand group membership)
  6. Check Deny Assignments at all levels
  7. The user's effective permissions are the UNION of all allowing assignments minus any Deny assignments.
  In the Azure Portal, I can use 'Access Reviews' or check the 'Effective roles' view."

### Senior L3

**Q: What is the difference between Microsoft Entra Directory Roles and Azure RBAC Roles?**
- **Ideal Answer:** "They are completely separate systems:
  - **Entra Directory Roles** govern identity operations (managing users, groups, Conditional Access, applications) within the Entra tenant. They are tenant-wide. A Global Administrator cannot create a VM.
  - **Azure RBAC** governs resource operations (VMs, Storage, Networking) on Azure resources. Roles are scoped. A Contributor cannot reset a user's password in Entra ID.
  - A user can have both types of roles simultaneously, but neither system grants permissions in the other."

### Expert

**Q: What happens internally when a user tries to delete a resource?**
- **Ideal Answer:** "1. The user authenticates to Entra ID and receives an access token containing their object ID (OID). 2. The request goes to ARM. 3. ARM evaluates authorization: it checks all role assignments for the OID at the resource, RG, subscription, and MG scope, plus all Deny Assignments. 4. For the delete operation, ARM also checks if any Deny Assignment exists that blocks deletion, AND checks for resource locks (CanNotDelete). 5. For write/delete operations, Azure Policy evaluates first — if Policy has a Deny effect, the request is blocked before it reaches ARM's authorization. 6. If all checks pass, ARM forwards the delete request to the resource provider (e.g., Microsoft.Compute). 7. The resource provider deletes the resource. 8. The Activity Log records the operation.
  Note: A resource with a CanNotDelete lock cannot be deleted even by the Owner role. However, a ReadOnly lock prevents Write operations but does NOT prevent Delete — unless explicitly configured to also block deletes (ReadOnly lock does block deletes too for most resources — it prevents both write and delete)."

### Scenario

**Q: "What would you do if a user with the Contributor role cannot create a network interface?"**
- **Ideal Answer:** "I would systematically investigate:
  1. Check if there's an Azure Policy Deny for NIC creation (check policy compliance)
  2. Check for Deny Assignments at subscription/MG scope blocking network write operations
  3. Check if the user is in the correct scope — maybe Contributor is assigned at one subscription and the NIC is in another
  4. Check the subnet for available IP addresses (even with correct RBAC, a subnet with no available IPs will fail)
  5. Check if the RG has a resource lock
  6. Check if there's a quota issue (network interface quota)
  7. Check Activity Log for the specific error
  The most common cause in enterprise environments is Azure Policy or Deny Assignment."

### Tricky

**Q: "A user says 'I have the Owner role, so I should be able to do everything.' Is this correct?"**
- **Ideal Answer:** "NO. Even the Owner role does NOT override:
  1. **Deny Assignments** — always win, even over Owner
  2. **Azure Policy** — a Deny policy blocks even Owner
  3. **Resource Locks** — CanNotDelete lock prevents deletion even for Owner
  4. **Azure service limits and quotas** — Owner cannot exceed regional capacity
  5. **Entra ID controls** — Owner is an Azure RBAC role; it does NOT grant Entra ID permissions
  6. **Conditional Access** — CA can block authentication entirely regardless of RBAC

  The correct mindset is: Owner = maximum Azure RBAC permission, but it operates within guardrails set by Policy, Deny, Locks, and Quota."
- **Common Mistake:** Saying "Owner = full access, period" — this is the most common wrong answer in L3 interviews.

---

## 17. SCENARIO-BASED QUESTIONS

### Scenario A: "Developer accidentally deleted production resource group and it's unrecoverable"

**Interview Question:** "How would you prevent this in the future?"

**Ideal Answer:**
1. **Resource Lock** — Apply `CanNotDelete` lock at the production subscription or MG level (prevents any delete)
2. **Azure Policy** — Deny deletion of resource groups at the MG scope
3. **RBAC** — Remove Owner/Contributor from production subscription; use scoped roles with least privilege
4. **PIM** — Make Owner/Contributor time-bound and require approval for production
5. **Break-glass** — For emergency deletions, use break-glass accounts with explicit approval workflows
6. **Backup** — Azure Backup for critical resources (can restore RG-level backups with Backup Vault)
7. **Monitoring** — Alert on any `Delete resource group` Activity Log event

### Scenario B: "Security team wants to audit all role assignments across all subscriptions"

**Interview Question:** "How would you generate this report?"

**Ideal Answer:**
- Use `Get-AzRoleAssignment` (PowerShell) or `az role assignment list` (CLI) at subscription scope, loop across all subscriptions
- Include: principal name, role definition, scope, condition (if any)
- Cross-reference with group membership to understand inherited permissions
- Filter for Owner and User Access Administrator assignments specifically (high-risk roles)
- Export to CSV/Excel for security review
- Automate this as a scheduled runbook or weekly report

### Scenario C: "Service principal has Contributor but is failing to create a storage account"

**Interview Question:** "Root cause analysis?"

**Ideal Answer:**
1. Verify SP is the correct one (object ID matches the assignment)
2. Check if Contributor is assigned at the correct subscription/RG scope
3. Check Azure Policy — is Storage Account creation denied? (e.g., allowed SKUs, allowed locations)
4. Check for Deny Assignment on storage account operations
5. Check if the subscription has reached storage account quota (250 per region by default)
6. Check if the resource group is in a subscription that allows storage creation
7. **Common root cause:** Azure Policy — many enterprises deny storage accounts in certain regions or with certain SKUs for security/cost reasons.

---

## 18. KNOWLEDGE TEST

Test yourself:

1. **Can a Reader role assignment at subscription scope read secrets in Key Vault?** Why or why not?
   > *Answer: No. Reader is a management-plane role and does not include Key Vault data-plane permissions (`Microsoft.KeyVault/vaults/secrets/read`). Key Vault requires explicit data-plane RBAC roles.*

2. **A user is assigned the Owner role at Management Group scope. Can they delete a resource in a child subscription?** What could block them?
   > *Answer: They can attempt to delete it. But it could be blocked by: CanNotDelete resource lock, Azure Policy Deny effect, or a Deny Assignment at a lower scope. Deny always wins.*

3. **What is the difference between a Role Definition and a Role Assignment?**
   > *Answer: Role Definition = the list of permissions (what actions). Role Assignment = the binding of a principal to a role at a scope (who + what + where).*

4. **True or False: A Global Administrator in Entra ID can create VMs in Azure.**
   > *Answer: FALSE. Global Admin is an Entra ID directory role. It does not grant any Azure RBAC permissions. Creating VMs requires an Azure RBAC role like Contributor or Owner at the appropriate scope.*

5. **You assign Contributor at the Subscription scope. A user is also in a group with Reader at the RG scope. What are their effective permissions?**
   > *Answer: They have Contributor permissions (union of both). Since Contributor allows everything Reader allows plus writes, the effective permission is Contributor-level. BUT any Deny Assignment overrides both.*

6. **What is a Deny Assignment and why does it override Owner?**
   > *Answer: A Deny Assignment explicitly denies specific actions at a scope. It overrides ALL allowing role assignments because the authorization pipeline evaluates Deny first. Even an Owner role cannot bypass a Deny Assignment — this is by design for safety and governance.*

7. **Can a custom role at RG scope be assigned to a user at Subscription scope?**
   > *Answer: Yes, if the custom role's "Assignable Scopes" include the subscription scope. The "Assignable Scopes" property of a custom role controls WHERE the role CAN be assigned.*

---

## 19. L3 GAP CHECK

| Topic | Status |
|-------|--------|
| Azure RBAC concept | ✅ Covered |
| Built-in roles (Owner, Contributor, Reader, UAA, etc.) | ✅ Covered |
| Custom roles | ✅ Covered |
| Role Assignment vs Role Definition | ✅ Covered |
| Scope (MG/Sub/RG/Resource) | ✅ Covered |
| Inheritance | ✅ Covered |
| Deny Assignments | ✅ Covered |
| ABAC / Role Assignment Conditions | ✅ Covered |
| Entra Directory Roles vs Azure RBAC | ✅ Covered — CRITICAL DISTINCTION |
| Group-based assignments | ✅ Covered |
| Managed identity permissions in RBAC | ✅ Covered |
| Service Principal vs App Registration vs Enterprise App in RBAC context | ✅ Covered |
| Control Plane vs Data Plane (RBAC relevance) | ✅ Covered |
| Authorization evaluation order (Policy → Deny → RBAC → Lock → Resource Provider) | ✅ Covered |
| RBAC troubleshooting methodology | ✅ Covered |
| Monitoring (Activity Log for RBAC changes) | ✅ Covered |
| Common failure scenarios | ✅ Covered |
| Interview questions (Basic → Expert) | ✅ Covered |
| Scenario-based questions | ✅ Covered |
| Current service changes (ABAC GA, PIM, built-in role drift) | ✅ Covered |
| Production example (custom VM Operator role) | ✅ Covered |

> **What an experienced Azure L3 engineer should now be able to explain confidently:**
>
> 1. **The difference between Entra Directory Roles and Azure RBAC roles** — and why Global Admin cannot create a VM
> 2. **How role assignments work** — who, what role, at what scope, and how inheritance flows down
> 3. **Why "Owner = everything" is WRONG** — Deny Assignments, Policy, Locks, and Quota all override even Owner
> 4. **How to troubleshoot any permission-denied issue** — systematic scope-by-scope investigation
> 5. **The authorization evaluation order** — Policy → Deny → RBAC → Lock → Resource Provider
> 6. **How to design least-privilege access** — custom roles, ABAC conditions, group-based assignments
> 7. **The difference between control plane and data plane permissions** — and why Reader cannot read Key Vault secrets
> 8. **How to audit and secure RBAC** — Activity Log monitoring, PIM, break-glass patterns, regular reviews

---

### Ready for Module 8 — Governance — Azure Policy?

Module 8 builds directly on Module 7 (RBAC) — Azure Policy is the complementary governance system that, unlike RBAC, governs **resource configuration** rather than **user permissions**. The distinction between "RBAC allows/denies what a user can do" vs "Policy governs what resources are allowed" will be a key focus.

Say the word and I'll deliver Module 8 in the same depth.