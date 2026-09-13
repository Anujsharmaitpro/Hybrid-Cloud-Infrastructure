# MODULE 7 — RBAC (Azure RBAC, Directory Roles, Role Assignments, Custom Roles, ABAC) — L3 DEPTH

---

## 1. CONCEPT

**Azure Role-Based Access Control (Azure RBAC)** is Azure's **authorization system** — it determines **what a security principal can do** on Azure resources.

> **Critical distinction from Entra ID/Conditional Access:**
> - Entra ID authentication answers: "Who are you?"
> - Conditional Access answers: "Under what conditions can you sign in?"
> - **Azure RBAC answers: "What can you do on Azure resources?"**

RBAC is **not** about authentication. It is purely about **authorization** — after a user has been authenticated, RBAC determines what actions they are allowed to perform on which resources, at what scope.

**Why RBAC exists:**
In any enterprise Azure environment, hundreds or thousands of people need access to Azure resources. You cannot give everyone the same access. RBAC enables:
- **Segregation of duties** — Helpdesk can reset passwords but cannot delete VMs
- **Least privilege** — Developer can read storage but cannot delete it
- **Scalable management** — Assign roles to groups, not individuals
- **Auditability** — Every action is tied to a role assignment

**The three pillars of RBAC:**
```
Security Principal (WHO) + Role Definition (WHAT) + Scope (WHERE) = Access (RESULT)
```

---

## 2. ARCHITECTURE

```
┌───────────────────────────────────────────────────────────────┐
│                     AZURE TENANT                              │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              MANAGEMENT GROUP(S)                        │  │
│  │  (Optional — for governing multiple subscriptions)       │  │
│  │                                                          │  │
│  │  ┌─────────────────────────────────────────────────┐    │  │
│  │  │              SUBSCRIPTION                        │  │
│  │  │                                                   │  │
│  │  │  ┌───────────────────────────────────────────┐  │    │  │
│  │  │  │            RESOURCE GROUP                  │  │    │  │
│  │  │  │                                           │  │    │  │
│  │  │  │  ┌──────────┐  ┌──────────┐  ┌────────┐  │  │    │  │
│  │  │  │  │ Resource │  │ Resource │  │Resource│  │  │    │  │
│  │  │  │  │   A      │  │   B      │  │   C    │  │  │    │  │
│  │  │  │  │ (VM)     │  │ (Storage)│  │ (App)  │  │  │    │  │
│  │  │  │  └──────────┘  └──────────┘  └────────┘  │  │    │  │
│  │  │  │                                           │  │    │  │
│  │  │  │  RBAC Role Assignments live HERE          │  │    │  │
│  │  │  │  (at Resource Group scope, inherited by   │  │    │  │
│  │  │  │   all resources inside)                   │  │    │  │
│  │  │  └───────────────────────────────────────────┘  │    │  │
│  │  └─────────────────────────────────────────────────┘    │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              DIRECTORY / ENTRA ID                       │  │
│  │  (Holds all identities: users, groups, SPs, MI)         │  │
│  │  (Also holds Directory Roles — separate from RBAC!)     │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

**Key architectural relationships:**
- Every Azure subscription is associated with **exactly one** Entra tenant
- RBAC role assignments live in the **Azure Resource Manager** (control plane)
- Identities (principals) are stored in **Entra ID** (directory)
- RBAC and Directory Roles are **two completely separate systems** (covered in Section 6)
- Role assignments flow **down** the scope hierarchy (subscription → RG → resource)

---

## 3. COMPONENTS — DETAILED

### 3.1 SECURITY PRINCIPAL (WHO)

A **security principal** is any identity that can be assigned a role. There are four types:

| Type | Description | Example |
|------|-------------|---------|
| **User** | Individual person with an Entra ID account | `john@contoso.com` |
| **Group** | Security group in Entra ID (membership can be users, groups, or SPs) | "Engineering-Developers" security group |
| **Service Principal** | Non-human identity for an app/service | App Registration's SP in the tenant |
| **Managed Identity** | Automatically managed SP tied to an Azure resource | System-assigned MI on a VM |

> **L3 key point:** When you assign a role, you assign it to a **security principal**. The most common mistake is assigning to a user individually instead of via a group. In production, **always assign RBAC to groups**, not individual users.

**Group-based assignments:**
```
When a role is assigned to a security group:
  → All members of that group inherit the role
  → Changes to group membership propagate (may take up to 15 minutes)
  → Both users and other groups can be members (nested)
  → Nested group membership is evaluated recursively

When a role is assigned to a user directly:
  → Only that user has the role
  → No inheritance through groups (unless they're also in a group with the role)

When a role is assigned to a service principal:
  → The app/service principal has the role
  → Any authenticated identity using that SP inherits the role
  → This is the primary model for automation and apps

When a role is assigned to a managed identity:
  → The MI has the role
  → The Azure resource using that MI inherits the role
  → Common pattern: VM MI → role on Storage Account
```

---

### 3.2 ROLE DEFINITION (WHAT)

A **role definition** is a collection of permissions. It lists the **actions** that can be performed.

**Structure:**
```json
{
  "Name": "Virtual Machine Contributor",
  "Id": "9980e02f-c71b-4a1a-b9c7-7f7f7f7f7f7f",
  "IsCustom": false,
  "Description": "Lets you manage virtual machines, but not access them.",
  "Actions": [
    "Microsoft.Compute/virtualMachines/*",
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/deallocate/action"
  ],
  "NotActions": [
    "Microsoft.Compute/virtualMachines/delete"
  ],
  "DataActions": [],
  "NotDataActions": []
}
```

**Permission types in role definitions:**

| Type | Description | Example |
|------|-------------|---------|
| **Actions** | Control plane operations (management) | `Microsoft.Compute/virtualMachines/read`, `/write`, `/delete` |
| **NotActions** | Exclusions from Actions | Excluding `/delete` from a broad Action set |
| **DataActions** | Data plane operations (data access) | `Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read` |
| **NotDataActions** | Exclusions from DataActions | Excluding specific data actions |

**Action format:**
```
Microsoft.{Provider}/{ResourceType}/{Operation}

Examples:
  Microsoft.Compute/virtualMachines/read           → Read VM properties
  Microsoft.Compute/virtualMachines/write           → Create/update VM
  Microsoft.Compute/virtualMachines/delete          → Delete VM
  Microsoft.Compute/virtualMachines/start/action    → Start a VM
  Microsoft.Network/virtualNetworks/subnets/join/action → Add NIC to subnet
  Microsoft.Storage/storageAccounts/listkeys/action → List storage keys
  Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read → Read blob data
```

**Wildcard notation:**
```
Microsoft.Compute/virtualMachines/*  → ALL operations on virtual machines
Microsoft.Storage/*/read             → Read operations on ALL Storage resources
Microsoft.Network/*                  → ALL operations on Network resources
```

**Built-in roles are NOT modifiable.** You cannot change the actions in a built-in role. If you need different permissions, you must create a **custom role**.

---

### 3.3 ROLE ASSIGNMENT (WHO + WHAT + WHERE)

A **role assignment** is the binding of a role definition to a security principal at a specific scope.

**It is the actual GRANT of access.** Without a role assignment, even if a role definition exists, no one has that permission.

**Three dimensions of every role assignment:**
```
Principal:     Who gets the role? (user, group, SP, MI)
Role:          What permissions? (which built-in or custom role)
Scope:         Where does it apply? (subscription, RG, resource, MG)
```

**Creating role assignments:**

**Azure Portal:**
Navigate to resource → Access Control (IAM) → Add role assignment → Select role → Select principal → Select scope

**Important:** The portal experience can be misleading — it always shows "Select a scope" with the current resource pre-selected. You must manually change the scope if you want subscription or management group level.

---

### 3.4 SCOPE (WHERE)

Scope defines **where** a role assignment applies. Scope flows **down** the hierarchy.

```
Scope hierarchy (from broadest to narrowest):

/{Tenant Root}/                          ← Root scope (everything in tenant)
  /providers/Microsoft.Management/
    managementGroups/{MGName}/           ← Management Group scope
      /subscriptions/
        {subscriptionId}/                ← Subscription scope
          /providers/Microsoft.ResourceManager/
            resourceGroups/{RGName}/     ← Resource Group scope
              /providers/Microsoft.Compute/
                virtualMachines/{vmName} ← Resource scope
```

| Scope Level | Example | Typical Use |
|-------------|---------|-------------|
| **Root** | `/` | Extremely rare — would give access to everything in tenant |
| **Management Group** | `/providers/Microsoft.Management/managementGroups/prod-mg` | Governing multiple subscriptions (e.g., all prod subscriptions share a policy) |
| **Subscription** | `/subscriptions/abc123` | Team that manages an entire subscription |
| **Resource Group** | `/subscriptions/abc123/resourceGroups/prod-rg` | Team that manages a specific application's resources |
| **Resource** | VM-specific ID | Granting access to a single resource |

**Inheritance rules:**
```
1. A role assignment at Subscription scope is inherited by ALL resource groups and resources in that subscription.
2. A role assignment at Resource Group scope is inherited by ALL resources in that RG.
3. A role assignment at Resource scope does NOT propagate up or down.
4. More specific (narrower) scope assignments do NOT override broader scope assignments — they ADD permissions.
5. You CANNOT scope access UP. (If assigned at RG scope, you cannot remove that permission at resource scope.)

In practice:
  If User A has Reader at Subscription scope AND Contributor at Resource Group scope:
  → User A has Reader on ALL resources in the subscription
  → User A has Contributor on resources in that specific RG
  → Combined: User A has Contributor on RG resources (Contributor > Reader) and Reader on everything else
```

**Effective permissions:**
```
A user's effective permissions are the UNION of ALL applicable role assignments.
This means:
  - Multiple role assignments at different scopes combine
  - The most permissive scope wins for any given resource
  - There is NO way to "subtract" permissions via a narrower scope
  - ABAC conditions (Section 8) can filter/limit but cannot explicitly deny
```

---

### 3.5 MANAGEMENT GROUPS (Scope Level)

**Management Groups** provide a level of scope above subscriptions. They organize subscriptions into a hierarchy for governance.

```
Tenant Root
  ├── MG-Prod
  │   ├── Sub-Production-West
  │   ├── Sub-Production-East
  │   └── MG-Shared-Services
  │       ├── Sub-Shared-Networking
  │       └── Sub-Shared-Identity
  └── MG-Dev
      ├── Sub-Dev-Project-A
      └── Sub-Dev-Project-B
```

**RBAC at Management Group scope:**
- Roles assigned at MG scope propagate down to all subscriptions and resources within
- Useful for: "Give the Security team Reader access across ALL production subscriptions"
- Cannot assign at Resource Group scope and expect it to propagate up to MG
- Policy assignments also work at MG scope (governing)

---

### 3.6 RESOURCE ID — CRITICAL CONCEPT

Every Azure resource has a unique **Resource ID**:

```
/subscriptions/{subscriptionId}/resourceGroups/{rgName}/providers/{resourceProvider}/{resourceType}/{resourceName}
```

**Example:**
```
/subscriptions/abc123-def4/resourceGroups/prod-rg/providers/Microsoft.Compute/virtualMachines/prod-vm01
```

**Why Resource IDs matter for RBAC:**
- Role assignments reference scope using resource IDs
- RBAC role assignment list shows the scope (a resource ID or parent)
- When troubleshooting access, you trace the role assignment to its scope
- Effective permissions depend on which resource IDs fall within the scope

**Resource Provider:**
The namespace that registers a resource type in Azure. Examples:
- `Microsoft.Compute` → VMs, disks, images
- `Microsoft.Storage` → Storage accounts, blob services
- `Microsoft.Network` → VNets, NICs, NSGs
- `Microsoft.Authorization` → RBAC role definitions and assignments
- `Microsoft.Web` → App Services

---

## 4. BUILT-IN Azure ROLES

Azure provides ~600+ built-in roles. The most commonly used are:

### Owner
| Property | Value |
|----------|-------|
| **Permissions** | Full access to all resources, including the ability to delegate access (assign roles) |
| **Can delete** | Yes |
| **Can assign roles** | Yes (`Microsoft.Authorization/*/write`) |
| **Scope** | All scopes |
| **Use case** | Full admin ownership of a subscription or RG |

### Contributor
| Property | Value |
|----------|-------|
| **Permissions** | Full access to create and manage all resources |
| **Can delete** | Yes |
| **Can assign roles** | **NO** (cannot grant access to others) |
| **Scope** | All scopes |
| **Use case** | Operational team that manages resources but should not delegate |

### Reader
| Property | Value |
|----------|-------|
| **Permissions** | Read-only access to all resources |
| **Can view** | Resources, properties, configurations |
| **Can modify** | No |
| **Can assign roles** | No |
| **Use case** | Auditors, observers, stakeholders who need visibility |

### User Access Administrator
| Property | Value |
|----------|-------|
| **Permissions** | Manage role assignments (RBAC) for Azure resources |
| **Can delegate** | Can assign roles to others (subject to their own permissions) |
| **Scope** | All scopes |
| **Use case** | Dedicated identity/team for managing access |

### Key Distinction: Owner vs Contributor
```
Owner = Contributor + Can assign roles

If you have Contributor, you CANNOT grant access to others.
If you have Owner, you CAN grant access to others.

This is critical: A compromised Owner account can escalate access
by assigning roles to itself or others across the entire subscription.
```

### Other Important Built-in Roles

| Role | Category | Use Case |
|------|----------|----------|
| **Key Vault Administrator** | Data plane | Manage Key Vault access policies and keys |
| **Storage Blob Data Contributor** | Data plane | Read/write blob data |
| **Storage Blob Data Reader** | Data plane | Read blob data |
| **Storage Account Contributor** | Control plane | Manage storage accounts (not data) |
| **Network Contributor** | Control plane | Manage all Network resources |
| **Virtual Machine Contributor** | Control plane | Manage VMs (cannot manage access or delete RG) |
| **Virtual Machine Operator** | Control plane | Monitor and restart VMs |
| **Managed Identity Contributor** | Control plane | Create and manage managed identities |
| **Monitoring Reader** | Monitoring | Read monitoring data |
| **Log Analytics Contributor** | Monitoring | Manage Log Analytics workspace |
| **DNS Contributor** | Network | Manage DNS zones and records |
| **Private DNS Contributor** | Network | Manage private DNS zones |
| **Security Admin** | Security | Manage Defender for Cloud and security policies |
| **Security Reader** | Security | Read security information |

> **L3 guidance:** Built-in roles are intentionally broad. In production, prefer **custom roles** for operational teams and **least privilege** principle. Use built-in roles for broad administrative functions and then refine with ABAC conditions.

---

## 5. ROLE ASSIGNMENTS — L3 DEPTH

### 5.1 Creating Role Assignments

**Via Portal:**
```
1. Navigate to the target resource/resource group/subscription
2. Go to "Access Control (IAM)"
3. Click "Add" → "Add role assignment"
4. Select Role (built-in or custom)
5. Select Principal (user, group, SP, or MI)
6. Review scope (change if needed)
7. Review + Assign
```

**Important portal gotcha:** The IAM blade on a resource shows assignments at that resource AND inherited assignments from parent scopes. Use the "Include inherited" checkbox to distinguish.

### 5.2 Scope and Inheritance in Practice

```
Scenario:
  - User "jane@contoso.com" is assigned "Contributor" at Subscription scope
  - User "jane@contoso.com" is assigned "Reader" at Resource Group scope for "prod-rg"
  - Resource "prod-vm01" is in "prod-rg"

Result:
  - prod-vm01 (in prod-rg): Jane has BOTH Contributor (inherited from sub) AND Reader (direct on RG)
  - Effective permission: Contributor (more permissive)
  - All other resources in the subscription: Jane has only Reader

Key takeaway: Role assignments ADD UP. There is no "subtract" mechanism in RBAC.
If you want someone to have LESS access, you must remove the broader assignment.
```

### 5.3 Time-Based Role Assignments

Azure supports time-bound role assignments (preview):
- Role assignments can have a start and end time
- After expiration, the assignment automatically becomes inactive
- Useful for contractor access, emergency access, project-based access
- The assignment still exists in the system but does not grant access after expiry

### 5.4 Role Assignment Propagation Delay

```
When a role assignment is created:
  → It may take UP TO 15 minutes to fully propagate
  → During this window, the assignment may not be effective
  → This is NOT a bug — it's eventual consistency in Azure's distributed system

L3 troubleshooting:
  "I just assigned the role but the user still cannot access the resource"
  → Wait 15 minutes and re-check
  → Verify the assignment exists in the IAM blade
  → Check if the user is in a group and group propagation adds additional delay
  → Check the sign-in log for authorization errors
```

### 5.5 Deny Assignments

**What is a Deny Assignment?**
A Deny Assignment is a special type of role assignment that **explicitly DENIES** a specific action, overriding any Allow (RBAC) grant.

```
RBAC (Allow):    "User can do X" → ALLOW
Deny Assignment: "User CANNOT do Y" → DENY (overrides the Allow)
```

**Key properties:**
| Property | Description |
|----------|-------------|
| **Scope** | Deny assignments work at subscription and resource scope (not resource group) |
| **Effect** | The denied action returns "Access Denied" even if the user has it via RBAC |
| **Inheritance** | Deny at subscription applies to all RGs and resources; Deny at resource applies only to that resource |
| **Management** | Managed through the Deny Assignment API (not the standard RBAC portal) |
| **Use case** | Preventing specific destructive actions even for owners (e.g., no one can delete a critical resource) |

**Example:**
```
Scenario: You want all users to be able to create VMs, but NO ONE should delete VMs in a specific RG.

Solution:
  1. Grant "Contributor" at RG scope (allows all VM operations)
  2. Create a Deny Assignment at RG scope: Deny "Microsoft.Compute/virtualMachines/delete"

Result:
  → All users can create, start, restart, modify VMs
  → All users are BLOCKED from deleting VMs
  → Even Owners are blocked from deleting (Deny overrides Allow)
```

**Deny Assignment vs Azure Policy "Deny":**
| Aspect | Deny Assignment | Azure Policy Deny |
|--------|----------------|-------------------|
| **Scope** | Resource or Subscription | Resource Group or higher |
| **What it denies** | Specific actions (e.g., delete VM) | Resource creation or configuration |
| **Who can create** | User with `Microsoft.Authorization/denyAssignments/write` | Policy assignment contributor |
| **Inherited** | Yes (down hierarchy) | Yes (down hierarchy) |
| **Use case** | Prevent destructive actions | Enforce governance rules |

> **L3 note:** Deny Assignments are often overlooked. If a user has Owner but CANNOT delete a resource, check for Deny Assignments BEFORE assuming it's a Policy issue.

---

## 6. MICROSOFT ENTRA DIRECTORY ROLES vs AZURE RBAC — CRITICAL DISTINCTION

This is the **#1 most confused concept** in Azure identity and access management.

**They are TWO COMPLETELY SEPARATE systems:**

```
┌───────────────────────────────────────────────────────────┐
│                    TWO AUTHORIZATION SYSTEMS               │
│                                                            │
│  ┌─────────────────────┐    ┌─────────────────────────┐   │
│  │  AZURE RBAC         │    │  ENTRA ID DIRECTORY     │   │
│  │  (Azure RBAC)       │    │  ROLES                  │   │
│  │                      │    │                         │   │
│  │  Controls:           │    │  Controls:              │   │
│  │  "What you can do   │    │  "What you can admin-   │   │
│  │   on Azure           │    │   ister in Entra ID"    │   │
│  │  resources"          │    │                         │   │
│  │                      │    │  Examples:              │   │
│  │  Owner, Contributor, │    │  Global Admin,          │   │
│  │  Reader, etc.        │    │  Conditional Access     │   │
│  │                      │    │  Admin, Security Admin, │   │
│  │  Scope:              │    │  etc.                   │   │
│  │  Resources, RG, Sub  │    │                         │   │
│  │                      │    │  Scope: ENTIRE TENANT   │   │
│  │  Managed by:         │    │                         │   │
│  │  Azure Resource Mgr  │    │  Managed by:            │   │
│  │  (IAM blade)         │    │  Entra ID (Azure AD)    │   │
│  └─────────────────────┘    └─────────────────────────┘   │
│                                                            │
│  THESE ARE INDEPENDENT:                                    │
│  - A user can have Global Admin but NO RBAC roles         │
│  - A user can have Contributor but NO Directory roles      │
│  - A user can have BOTH simultaneously                    │
│  - They do NOT inherit from each other                    │
│  - They are configured in DIFFERENT portals               │
└───────────────────────────────────────────────────────────┘
```

**What this means for troubleshooting:**

```
Scenario: "I am Global Administrator but cannot create a VM"

WRONG answer: "Global Admin should have access"
CORRECT answer: Global Admin is a DIRECTORY ROLE — it allows managing Entra ID
                settings, NOT Azure resources. To create a VM, the user needs an
                Azure RBAC role (Owner or Contributor) at the subscription or
                resource group scope.

Scenario: "I have Owner on the subscription but cannot reset passwords"
WRONG answer: "Owner should allow everything"
CORRECT answer: Owner is an Azure RBAC role — it allows managing Azure resources.
                Password reset is an Entra ID directory operation requiring a
                Directory Role (Privileged Role Administrator or Password Administrator).

Scenario: "I have Contributor on the RG but cannot create Conditional Access policies"
CORRECT answer: Conditional Access is an Entra ID feature, not an Azure resource.
                Contributor RBAC does not grant directory administration. You need
                the Conditional Access Administrator directory role.
```

**Complete Directory Roles list (major ones):**

| Directory Role | What It Administers |
|---------------|-------------------|
| Global Administrator | ALL Entra ID settings, can grant any role |
| Global Reader | Read-only access to all Entra ID settings |
| Privileged Role Administrator | Can assign and manage directory roles (via PIM or direct) |
| Privileged Role Administrator | Manage PIM eligible/active assignments |
| User Administrator | Create/delete users, reset passwords, manage user properties |
| Application Administrator | Create/manage app registrations, enterprise apps, OAuth settings |
| Cloud App Administrator | Manage enterprise app settings, admin consent |
| Conditional Access Administrator | Create/manage Conditional Access policies |
| Security Administrator | Security-related configurations |
| Domain Admin | Domain-specific settings |
| Device Administrator | Device settings |
| Password Administrator | Reset passwords for other users |
| Helpdesk Administrator | Reset passwords, manage user accounts (limited scope) |

**Key insight for L3 interviews:**
```
The confusion between Directory Roles and Azure RBAC causes production incidents:

1. Engineer creates a new admin account with Global Admin (Directory role)
   but forgets to assign Contributor RBAC → Admin cannot access Azure Portal resources

2. Engineer assigns RBAC roles to a user but expects them to manage
   Conditional Access or Entra ID settings → Not possible without Directory roles

3. Service Principal has RBAC access but no Directory role → Cannot authenticate
   via certain endpoints or manage directory resources

Rule of thumb:
  → Need to manage Azure resources? → Check Azure RBAC
  → Need to manage Entra ID settings? → Check Directory Roles
  → Need to manage BOTH? → You need both systems configured correctly
```

---

## 7. CUSTOM ROLES — L3 DEPTH

### 7.1 Why Custom Roles

Built-in roles are intentionally **broad**. In production, you need **precision**.

**Scenario:** You want a developer to be able to:
- Read VMs
- Start/stop VMs
- But NOT delete VMs
- And NOT access storage accounts

No built-in role provides exactly this. You need a custom role.

### 7.2 Custom Role Structure

```json
{
  "Name": "Dev-VM-Operator",
  "IsCustom": true,
  "Description": "Can read, start, stop, and restart VMs only.",
  "Actions": [
    "Microsoft.Compute/virtualMachines/read",
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/powerOff/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/runCommand/action"
  ],
  "NotActions": [
    "Microsoft.Compute/virtualMachines/delete"
  ],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": [
    "/subscriptions/abc123",
    "/subscriptions/abc123/resourceGroups/prod-rg"
  ]
}
```

**Key properties:**

| Property | Description |
|----------|-------------|
| **`Actions`** | Permissions granted (positive allow list) |
| **`NotActions`** | Permissions explicitly excluded from `Actions` |
| **`DataActions`** | Data plane permissions (e.g., reading blob data) |
| **`NotDataActions`** | Data actions to exclude |
| **`AssignableScopes`** | Where this role CAN be assigned (must be within subscription or RG) |
| **`IsCustom`** | Must be `true` for custom roles |

### 7.3 Custom Role Rules

| Rule | Details |
|------|---------|
| **Cannot modify built-in** | Built-in roles are read-only |
| **Cannot modify via Portal** | Custom roles can only be edited via CLI, PowerShell, REST API, or ARM templates (Portal shows them but editing is limited) |
| **Requires permission to create** | `Microsoft.Authorization/roleDefinitions/write` — typically Owner or User Access Administrator |
| **AssignableScopes limitation** | A custom role can only be assigned at scopes listed in its `AssignableScopes` array |
| **No Data Actions for certain resources** | Some resource types don't support data actions in custom roles |
| **5,000 custom roles** | Per tenant limit |
| **Shared across subscriptions** | If trusted by same tenant, available in all subscriptions in tenant |

### 7.4 Creating Custom Roles

**Azure CLI:**
```bash
# Create from JSON file
az role definition create --role-definition custom-role.json

# Update
az role definition update --role-definition updated-role.json

# List custom roles
az role definition list --custom-role-only

# Delete (must remove all assignments first)
az role definition delete --name "Custom Role Name"
```

**Azure PowerShell:**
```powershell
# Create
$roleDef = Get-AzRoleDefinition -Name "Custom Role Name"
New-AzRoleDefinition -Role $roleDef

# Update
$roleDef = Get-AzRoleDefinition -Name "Custom Role Name"
$roleDef.Actions.Add("Microsoft.Compute/virtualMachines/read")
Set-AzRoleDefinition -Role $roleDef

# Delete
Remove-AzRoleDefinition -Name "Custom Role Name"
```

### 7.5 Common Custom Role Patterns

| Pattern | Description |
|---------|-------------|
| **Reader + specific actions** | Can read everything but perform only specific actions (start VM, read storage) |
| **Resource-specific operator** | Can manage only VMs in a specific RG, nothing else |
| **Monitoring-only** | Can read metrics, logs, and alerts but cannot modify anything |
| **Deployment-only** | Can deploy specific resource types but cannot delete existing resources |
| **Network reader** | Can read network config but cannot modify |

### 7.6 Modifying Custom Roles

```
When you modify a custom role:
  → Changes take effect immediately for NEW role assignments
  → Existing assignments are NOT updated retroactively
  → Users already holding the role get the NEW permissions on next authentication/authorization check

This means:
  If you REMOVE an action from a custom role, users may still have access
  until their next token refresh (typically 60-90 minutes for access tokens)
```

---

## 8. ATTRIBUTE-BASED ACCESS CONTROL (ABAC) — L3 DEPTH

### 8.1 What is ABAC

**Azure ABAC** extends RBAC by adding **conditions** to role assignments. Conditions are expressions that filter the permissions granted by the role.

```
RBAC alone:       "User has Contributor on this RG" → Can do EVERYTHING on everything in the RG
RBAC + ABAC:      "User has Contributor on this RG, BUT ONLY where tag Project=Contoso"
                  → Can do everything ONLY on resources tagged Project=Contoso
```

> **ABAC does NOT explicitly deny.** It filters/limits. If a condition is not met, the user simply does not have access to that resource — but it's not a "Deny" per se.

### 8.2 Why ABAC

| Problem | Without ABAC | With ABAC |
|---------|-------------|-----------|
| 100 developers need access to their own projects | Create 100 role assignments | 10 role assignments with conditions |
| Different teams need different access to same resources | Separate RGs or custom roles | Conditions filter by team attribute |
| Contractors need temporary access | Time-based assignments + separate roles | Conditions + time-based assignments |

### 8.3 ABAC Condition Syntax

```
(
  !(ActionMatches{'Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read'} 
    AND NOT SubOperationMatches{'Blob.List'})
  OR
  (@Resource[Microsoft.Storage/storageAccounts/blobServices/containers/blobs/tags:Project<key_case_sensitive$>] 
   StringEquals 'Contoso')
)
```

**Key elements:**
| Element | Meaning |
|---------|---------|
| `ActionMatches{...}` | Checks the action being performed |
| `SubOperationMatches{...}` | Checks sub-operations (e.g., List vs Get) |
| `@Resource[...]` | Checks resource attributes (tags, name, type, location) |
| `@Principal[...]` | Checks principal attributes (user attributes, group membership, custom security attributes) |
| `StringEquals` | Condition operator (others: StringNotEquals, StringContains, StringStartsWith, etc.) |
| `StringCaseInsensitive` | Case-insensitive comparison |

### 8.4 ABAC Sources

**Resource attributes:**
```
@Resource[Microsoft.Storage/storageAccounts/blobServices/containers/blobs/tags:Project<key_case_sensitive$>]
@Resource[Microsoft.Storage/storageAccounts/blobServices/containers/blobs/Name]
@Resource[Microsoft.Network/virtualNetworks/Name]
@Resource[Microsoft.Compute/virtualMachines/Location]
```

**Principal attributes:**
```
@Principal[Microsoft.Directory/CustomSecurityAttributes/Id:Department]
@Principal[Microsoft.Directory/CustomSecurityAttributes/Id:Project]
@Principal[Microsoft.Directory/CustomSecurityAttributes/Id:Team]
```

**Important:** Principal attributes come from **Entra ID custom security attributes**. These must be configured in Entra ID (Enterprise Applications → Microsoft Entra ID → Attribute Management) before they can be used in ABAC.

### 8.5 ABAC Real-World Example

**Scenario:** Multiple teams work in the same resource group. Each team should only access resources tagged with their project name.

```
Resource Group: "shared-rg"
Resources tagged: 
  - VM-01: Project=Contoso
  - VM-02: Project=Fabrikam
  - Storage-01: Project=Contoso

Role Assignment:
  - "Contoso-Team" group → Contributor on "shared-rg" 
    WITH condition: @Resource[.../tags:Project<key_case_sensitive$>] StringEquals "Contoso"
  - "Fabrikam-Team" group → Contributor on "shared-rg"
    WITH condition: @Resource[.../tags:Project<key_case_sensitive$>] StringEquals "Fabrikam"

Result:
  Contoso-Team can manage VM-01 and Storage-01 (both tagged Contoso)
  Contoso-Team CANNOT manage VM-02 (tagged Fabrikam)
  Fabrikam-Team can manage VM-02 only
```

### 8.6 ABAC and PIM

ABAC conditions can be applied to **Privileged Identity Management (PIM) eligible assignments**, enabling:
- Time-bound access with attribute filtering
- Approval workflows with attribute conditions
- Audit trail for conditional access

---

## 9. KEY DIFFERENCES SUMMARY

| Aspect | Azure RBAC | Azure ABAC | Directory Roles |
|--------|-----------|-----------|-----------------|
| **Purpose** | Manage access to Azure resources | Filter/limit RBAC by attributes | Manage Entra ID administration |
| **Scope** | Resources, RG, Sub, MG | Applied ON TOP of RBAC assignments | Entire tenant |
| **Configured in** | Azure Portal (IAM) or API | Portal (conditions on role assignment) | Entra ID portal (Roles & Admins) |
| **Identity types** | User, Group, SP, MI | Same + Principal attributes | User, Group, SP |
| **Granularity** | Role-level (broad permissions) | Attribute-level (fine-grained filter) | Role-level (tenant-wide admin) |
| **Deny capability** | No (allow-only) | No (allow-only, filtering) | No (allow-only) |
| **Inheritance** | Down hierarchy | N/A (conditions are per-assignment) | No inheritance (flat tenant) |
| **Dependency** | Entra ID for principals | Entra ID for principal attributes | N/A |

---

## 10. BEST PRACTICES

### 10.1 Least Privilege

```
START with Reader.
GRANT only what's needed.
NEVER use Owner as a default.
USE groups for assignments.
VALIDATE periodically.
```

### 10.2 Group-Based Assignments

```
RULE: Never assign RBAC directly to individual users.

BENEFITS:
  → Centralized management (modify group membership → permissions change)
  → Auditing easier (one assignment per group, not per user)
  → Onboarding/offboarding easier (add/remove from group)
  → Temporary access easier (add to group, remove when done)

PATTERN:
  Create security groups per role:
    "RG-Prod-Contributor" → Contributor on prod-rg
    "RG-Prod-Reader" → Reader on prod-rg
    "Sub-Dev-Owner" → Owner on dev-sub

  Assign users to these groups.
  NEVER assign roles to users directly in production.
```

### 10.3 Scope Down

```
BAD:    Assign "Contributor" at Subscription scope to 50 developers
GOOD:   Assign "Contributor" at Resource Group scope to specific RGs per team
BETTER: Create custom role with specific actions + RG scope

RULE: Always use the NARROWEST scope that meets the requirement.
```

### 10.4 Custom Roles over Broad Built-in

```
BAD:    Give "Owner" to a developer who needs to restart VMs
GOOD:   Create custom role: "VM-Restart-Operator" with only start/restart/read actions

RULE: Built-in roles are for administrators. Operational roles should be custom.
```

### 10.5 ABAC to Reduce Assignment Count

```
Without ABAC:
  100 developers × 5 resource groups = 500 role assignments

With ABAC:
  5 teams × 1 role assignment per team (with conditions) = 5 assignments

ABAC dramatically reduces management overhead.
```

---

## 11. PRODUCTION EXAMPLE

**Scenario: Enterprise with 200 developers across 5 teams managing resources in 3 subscriptions.**

```
Tenant: contoso.onmicrosoft.com (P2)
Management Groups:
  MG-Production (Sub-Prod-West, Sub-Prod-East)
  MG-Development (Sub-Dev-A, Sub-Dev-B)

Identity Management:
  Teams organized as Entra ID security groups:
    "Team-Contoso-Developers" (40 users)
    "Team-Fabrikam-Developers" (35 users)
    "Team-Contoso-Operations" (25 users)
    "Team-Fabrikam-Operations" (30 users)
    "Team-Shared-Services" (70 users)

RBAC Structure (ABAC-enabled):

  Subscription-level:
    "Team-Contoso-Operations" → Contributor on Sub-Prod-West
    "Team-Fabrikam-Operations" → Contributor on Sub-Prod-East
    (with ABAC condition: tag Team = respective team name)

  Resource Group-level:
    "Team-Contoso-Developers" → Contributor on RG-Contoso-Prod
    "Team-Fabrikam-Developers" → Contributor on RG-Fabrikam-Prod
    (with ABAC condition: @Resource[.../tags:Project] StringEquals "Contoso")

  Custom Roles created:
    "Custom-VMC-Operator" (restart/start/read VMs only)
    "Custom-Storage-Reader" (read storage data only)
    "Custom-Network-Contributor" (manage networks except firewall rules)

  Directory Roles:
    Global Admin: 4 named admins (PIM eligible)
    Conditional Access Admin: 2 security team members
    Application Admin: 3 platform team members
    User Administrator: 5 helpdesk team members

  PIM:
    All Global Admin assignments are PIM-eligible
    Time-bound: 8-hour activation windows
    Approval required for activation
    Audit trail for every activation

  Deny Assignments:
    Sub-Prod-West: Deny "Microsoft.Compute/virtualMachines/delete" for all non-operations teams
    Sub-Prod-East: Deny "Microsoft.Storage/storageAccounts/delete" for all non-operations teams

  Result:
    500+ developers managed through 5 groups + ABAC conditions
    No individual RBAC assignments
    Custom roles for operational precision
    PIM for privileged access
    Deny assignments for destructive action prevention
```

---

## 12. FAILURE SCENARIOS

| Scenario | Root Cause | Resolution | Evidence |
|----------|-----------|------------|----------|
| **"User has Contributor but cannot create VM"** | Azure Policy Deny assignment conflicts; or subscription quota exceeded; or Resource Provider not registered; OR the user is assigned RBAC but group propagation hasn't completed (15 min) | Check: (1) Azure Policy in the subscription/RG, (2) Activity Log for denial reason, (3) Quotas, (4) Resource Provider registration, (5) Role assignment propagation delay | Activity Log: "Deployment failed" with error code; Policy: Deny assignment details; Quota: current usage vs limit |
| **"User was added to group with Owner but still can't do anything"** | Group membership propagation delay (up to 15 minutes); OR the user has conflicting Deny Assignment; OR the group is in a different tenant; OR user is excluded from the scope by an ABAC condition | Wait 15 minutes, check group membership sync, check for Deny Assignments, verify group trust relationship | Sign-in log: authorization failure; RBAC: inherited assignments; Deny Assignment: scope check |
| **"Contractor was removed from group but still has access"** | Token caching — existing access tokens still valid until expiry; OR role assignment directly on user (not just via group); OR PIM active assignment still active | Wait for token expiry; check direct role assignments on the user; check PIM active assignments; revoke tokens via Conditional Access if needed | Token decode: upn and groups claim; RBAC: all assignments for that user; PIM: active assignments |
| **"Custom role assignment not working"** | Custom role's `AssignableScopes` doesn't include the scope where it's being assigned; OR the role definition is malformed; OR the custom role hasn't been saved/registered properly | Verify `AssignableScopes` in the role definition matches the intended scope; re-create the role if corrupted; check for syntax errors in JSON | `az role definition list --custom-role-only` shows the role; check AssignableScopes field |
| **"Owner cannot delete a resource"** | Deny Assignment exists blocking the delete action; OR Resource Lock (CanNotDelete) on the resource or RG; OR Azure Policy Deny effect on resource deletion; OR the resource is part of a resource group with locks | Check Deny Assignments at subscription and resource scope; check resource locks on the RG and resource; check Azure Policy for deny effects on deletion | Resource locks: blade shows locks; Deny Assignment: API shows assignments; Policy: assignment shows deny effect |
| **"Service principal cannot access Storage despite RBAC assignment"** | SP is a different object from the App Registration; RBAC assigned to App Registration but authentication uses different SP; OR SP is at wrong scope; OR authentication token hasn't refreshed with new role assignment; OR Private Endpoint / Firewall blocking | Verify RBAC assigned to the SP that's actually authenticating (check Object ID); check token audience; verify scope; force token refresh; check network access | SP sign-in log: which SP authenticated; Token decode: oid claim matches the SP with RBAC |
| **"ABAC condition not filtering as expected"** | ABAC feature not registered (needs feature flag enablement); OR condition syntax error; OR resource tag missing/mismatched; OR principal attribute not configured in Entra ID; OR condition version mismatch | Register ABAC feature; validate condition syntax; verify resource tags match condition exactly; verify custom security attributes are populated; check condition version is 2.0 | Diagnostic: check if condition evaluation logs show the condition; Resource tags: verify exact key/value; Principal attributes: verify Entra ID attributes are populated |
| **"New custom role assignment takes time to work"** | RBAC propagation delay (up to 15 minutes); OR the user's existing token was issued before the assignment and doesn't include new group membership; OR the custom role's `AssignableScopes` doesn't include the target scope | Wait 15 minutes; force user sign-out/sign-in or wait for token refresh; verify `AssignableScopes` | Sign-in log: authorization check timestamp; Token decode: groups claim updated |
| **"Directory Admin cannot access Azure resources"** | Global Admin (or other Directory Admin) has NO Azure RBAC role assignments. Directory Admin can manage Entra ID but has no access to Azure resources by default | Assign Azure RBAC role (e.g., Owner or Contributor) at subscription/RG scope to the Directory Admin account or a group they belong to | Azure Portal: IAM blade shows no RBAC assignments for that user; Entra ID: Directory role shows they are admin |
| **"Deny Assignment blocking expected access"** | Deny Assignment created for a different purpose is overlapping with the user's RBAC; OR the user is in a group that's included in the Deny Assignment scope | Check all Deny Assignments at subscription and resource scope; identify which specific action is denied; evaluate if the Deny Assignment should be modified or if the user should be excluded | Deny Assignment API: `GET https://management.azure.com/{scope}/providers/Microsoft.Authorization/denyAssignments` |

---

## 13. TROUBLESHOOTING METHODOLOGY — RBAC

```
STEP 1: Identify the problem
  → What action is the user trying to perform?
  → What resource?
  → What error message exactly?

STEP 2: Check the error message
  → "AuthorizationFailed" → RBAC issue
  → "Forbidden" → Could be RBAC, Policy, or NSG
  → "403" → Authorization denied
  → "401" → Authentication failed (NOT RBAC)

STEP 3: Check effective permissions
  → Azure Portal: User's profile → check roles
  → CLI: Check role assignments for the user's Object ID
  → Portal: Navigate to resource → Access Control (IAM) → check assignments
  → Include inherited assignments

STEP 4: Verify the role assignment
  → Does the user have a role assignment for the target resource?
  → Is the assignment direct or via group membership?
  → If via group → has group membership propagated (15 min)?
  → Is the scope correct (sub/RG/resource)?
  → Is the role definition correct (has the required action)?

STEP 5: Check for Deny Assignments
  → Azure Resource Manager → Deny Assignments
  → At subscription scope and resource scope
  → Does any Deny Assignment block the action?

STEP 6: Check for Resource Locks
  → Is the resource or RG locked?
  → CanNotDelete lock? ReadOnly lock?

STEP 7: Check for Azure Policy
  → Could a Policy Deny the action?
  → Check Policy compliance for the user's attempted action
  → Policy is NOT RBAC — it can block even with Owner/Contributor

STEP 8: Check ABAC conditions
  → If ABAC is enabled, do conditions filter the user's access?
  → Resource tags match conditions?
  → Principal attributes populated?

STEP 9: Check Resource Provider
  → Is the Resource Provider registered? (e.g., Microsoft.Compute, Microsoft.Storage)
  → Some actions fail if the provider is not registered in the subscription

STEP 10: Check scope and inheritance
  → Is the role assignment at a parent scope that doesn't cover this resource?
  → Is the user in the right subscription/RG context?

STEP 11: Validate and document
  → Fix identified issue
  → Test the action
  → Document root cause in incident log
```

---

## 14. LOGS / EVIDENCE

| Evidence Source | What It Shows | Access Method |
|---------------|---------------|---------------|
| **Activity Log** | All Azure control plane operations, including role assignments, with identity context | Portal → Subscription/RG → Activity Log |
| **RBAC Assignment list** | All role assignments at a given scope (including inherited) | Portal → Access Control (IAM) → Check "Include inherited" |
| **Effective permissions** | What a specific user can actually do (combines all assignments) | Some Azure services show "Effective permissions" (e.g., Storage) |
| **Deny Assignment list** | All deny assignments at a scope | API: `GET /{scope}/providers/Microsoft.Authorization/denyAssignments` |
| **Resource Locks** | All locks on a resource/RG/subscription | Portal → Resource → Locks; or `Get-AzResourceLock` |
| **Policy compliance** | Whether a resource is compliant or non-compliant with policy | Portal → Policy → Compliance |
| **Sign-in logs (Entra ID)** | Authentication events, including authorization failures | Portal → Entra ID → Monitoring → Sign-in logs |
| **Audit logs (Entra ID)** | Administrative changes to directory settings | Portal → Entra ID → Monitoring → Audit logs |

---

## 15. VERSION/CURRENT SERVICE CONSIDERATIONS

| Aspect | Current State (Sept 2026) | L3 Impact |
|--------|--------------------------|-----------|
| **Classic Administrators** | Fully retired as of May 2026. Classic Administrators tab removed from portal. | If you still have legacy Classic Admin roles, they are no longer active. Must migrate to Azure RBAC. |
| **Custom roles** | Up to 5,000 per tenant. All creation/management via API/CLI/PowerShell (Portal support is limited for editing). | Plan custom role strategy. Use naming conventions. Document all custom roles. |
| **ABAC** | In preview/public preview. Requires feature registration in Azure Resource Manager. | ABAC is powerful but still evolving. Test in non-production first. Validate condition syntax. |
| **Deny Assignments** | GA. Managed via API. Can be at subscription or resource scope. | Often overlooked in troubleshooting. Always check for Deny Assignments when RBAC seems sufficient but access is denied. |
| **Time-based assignments** | Preview. Allows automatic expiration of role assignments. | Useful for contractor access and emergency access. Reduces standing privilege. |
| **PIM integration** | PIM can add conditions and time bounds to RBAC role assignments. | PIM + RBAC + ABAC is the gold standard for enterprise access management. |
| **Role assignment propagation** | Up to 15 minutes for full propagation. | Always wait 15 minutes before concluding an RBAC assignment failed. |
| **Built-in roles expansion** | Microsoft continues adding new built-in roles for new Azure services. | Check for updated built-in roles when deploying new services. |
| **RBAC API** | Full management via REST API and ARM templates. | Programmatic RBAC management enables automation and governance. |

---

## 16. L3 INTERVIEW QUESTIONS

### Basic
**Q: What is Azure RBAC?**
A: Azure Role-Based Access Control is Azure's authorization system that determines what actions a security principal (user, group, service principal, or managed identity) can perform on Azure resources, at what scope. It consists of three components: security principals, role definitions (permissions), and role assignments (binding principals to roles at scopes).

### Intermediate
**Q: What is the difference between Owner and Contributor?**
A: Both have full access to manage all Azure resources. The key difference: Owner can assign roles to others (delegate access), while Contributor cannot. Contributor can create and manage resources but cannot grant access to others.

### L3
**Q: A user has the "Owner" role on a subscription but cannot create a VM. What are all possible causes?**
A:
1. **Azure Policy:** A "Deny" policy blocks VM creation (e.g., disallowed location, disallowed SKU, required tags missing)
2. **Resource Lock:** CanNotDelete or ReadOnly lock on the target RG (CanNotDelete would block creation of new resources in some interpretations, or lock prevents operations)
3. **Quota:** User has exceeded vCPU quota for the region
4. **Resource Provider:** `Microsoft.Compute` provider not registered in the subscription
5. **Deny Assignment:** A Deny Assignment blocks the specific VM creation action
6. **ABAC condition:** ABAC condition filters out the resource (if enabled)
7. **Propagation delay:** Role assignment was just made and hasn't propagated (up to 15 minutes)
8. **Network connectivity:** If creating from Portal, network issues could prevent the operation
9. **Sku availability:** The specific VM size may not be available in the region

### Senior L3
**Q: Explain the critical difference between Directory Roles and Azure RBAC. Give a real scenario where confusion between them causes an incident.**
A: Directory Roles manage who can administer Entra ID (identity management). Azure RBAC manages who can administer Azure resources. They are independent systems. Real scenario: A "Global Administrator" in Entra ID has NO access to create VMs in Azure unless they also have an Azure RBAC role (like Owner or Contributor) on a subscription or resource group. Many engineers assume Global Admin = full access to everything, but Global Admin is purely for the identity/management plane. This causes incidents when new admins are given Directory Roles but not RBAC roles, and they cannot access Azure resources. Conversely, a user with "Contributor" on a subscription can manage all Azure resources but CANNOT reset passwords or create Conditional Access policies because those are Directory Role operations.

### Expert
**Q: What happens internally when a role assignment is created?**
A: When a role assignment is created:
1. Azure Resource Manager receives the request with the principal ID, role definition ID, and scope
2. ARM validates: Does the caller have `Microsoft.Authorization/roleAssignments/write` permission at the scope?
3. ARM validates: Is the principal a valid security principal in Entra ID?
4. ARM validates: Is the role definition valid and does its `AssignableScopes` include the target scope?
5. ARM creates the role assignment object in its internal store
6. ARM propagates the assignment to all relevant control plane components
7. The assignment is now effective (up to 15 minutes for full propagation across all subsystems)
8. Subsequent authorization checks for the principal at the scope (and child scopes) will include this role's permissions

### Scenario
**Q: "A developer was added to the 'Dev-Contributor' group last night, but this morning they still cannot deploy resources. What do you check?"**
A:
1. Confirm group membership — is the user actually in the group?
2. Check the role assignment — is "Dev-Contributor" group assigned to Contributor at the right scope?
3. Check propagation delay — was the group assignment made recently? Wait 15 minutes.
4. Check for Deny Assignments — could a Deny Assignment block deployments?
5. Check Azure Policy — could a Policy deny resource creation?
6. Check quotas — is the user at their vCPU limit?
7. Check Resource Provider — is `Microsoft.ResourceManager` (or the specific provider) registered?
8. Have the user sign out and sign back in (token refresh with new group membership).

### Tricky
**Q: "I have 'Contributor' at subscription scope but cannot delete a resource. Why?"**
A: Multiple possible reasons:
1. **Resource Lock:** A "CanNotDelete" lock on the resource or its RG. Locks override RBAC.
2. **Deny Assignment:** A Deny Assignment explicitly denies the delete action (overrides Contributor).
3. **Azure Policy:** A Policy "Deny" effect blocks the deletion (Policy can deny even with Owner/Contributor).
4. **Resource dependency:** Another resource depends on it (e.g., a NIC attached to a VM blocks VM deletion).

The common mistake is answering "Contributor should be able to delete." Contributor is allow-based and can be overridden by Locks, Deny Assignments, and Policy.

### Tricky 2
**Q: "If I assign 'Reader' at the subscription scope and 'Contributor' at the resource group scope, what effective permissions does the user have?"**
A: The user has:
- Reader on ALL resources in the subscription (from the subscription-level Reader assignment)
- Contributor on ALL resources in the specific RG (from the RG-level Contributor assignment)
- Combined: Contributor on resources within that RG, Reader on everything else in the subscription
- Both assignments are ADDING permissions, not subtracting. The user cannot be MORE permissive than the union of all their assignments.

### Tricky 3
**Q: "A user was removed from a security group that had a 'Contributor' RBAC assignment, but they can still create resources. Why?"**
A:
1. **Token caching:** The user's existing access token still contains the group membership claim. Access tokens are valid for 60-90 minutes (or longer depending on session controls). The user's existing token still has the group claim, so they still appear to have Contributor.
2. **Direct assignment:** The user might also have a direct Contributor assignment (not just via group).
3. **PIM active assignment:** The user might have an active PIM assignment that was activated before group removal.
4. **Propagation delay:** Group membership changes take time to propagate.
Resolution: Wait for token expiry, check for direct assignments, check PIM active assignments, force sign-out/sign-in.

---

## 17. SCENARIO-BASED QUESTIONS

### Scenario 1: "VM deployment fails with AuthorizationFailed"
**Architecture:** ARM deployment → RBAC evaluation → Resource Provider → Resource creation.
**Dependencies:** RBAC role, Policy, Locks, Quotas, Resource Provider.
**Checks:**
1. Activity Log: What is the specific error message? (AuthorizationFailed details)
2. Does the user have the required RBAC role at the target scope?
3. Is there a Deny Assignment blocking?
4. Is there an Azure Policy Deny that blocks this resource type/location/SKU?
5. Is there a Resource Lock?
6. Is the user within quota limits?
7. Is the Resource Provider registered?
**Root Cause (common):** Azure Policy Deny, or missing Contributor role at target scope, or unregistered Resource Provider.
**Fix:** Adjust Policy, assign RBAC role, register Resource Provider.
**Validation:** Deployment succeeds; Activity Log shows success.

### Scenario 2: "Service principal was working, now returns 403"
**Architecture:** App authentication → Token issuance with SP identity → RBAC check → Resource access.
**Dependencies:** SP status, RBAC role assignment, Token validity, Policy, Network.
**Checks:**
1. Was the SP recently reassigned? (New SP Object ID after app registration refresh?)
2. Are RBAC role assignments still in place for this SP's Object ID?
3. Is the token being issued by the correct SP? (Verify `oid` claim in token)
4. Is there a Deny Assignment?
5. Is there a Policy change?
6. Has the SP been disabled?
**Root Cause: RBAC assignment was on the old SP (Object ID A), but the app was regenerated and now uses a new SP (Object ID B).**
**Fix:** Reassign RBAC roles to the new SP's Object ID.
**Validation:** SP authenticates successfully, access granted.

### Scenario 3: "New employee cannot access any Azure resources despite being in the correct security group"
**Architecture:** Entra ID group membership → RBAC inheritance → Resource access.
**Dependencies:** Group membership, RBAC assignment, Token generation, Propagation.
**Checks:**
1. Is the user confirmed in the group? (Entra ID → Groups → Members)
2. Does the group have a role assignment? (Check at subscription/RG scope)
3. Has the role assignment propagated? (Wait 15 minutes if just created)
4. Has the user signed in since being added to the group? (Token needs to include group claim)
5. Is the group in the correct tenant? (B2B guest vs member issue)
6. Are there Deny Assignments or Policies that exclude this user?
**Root Cause (most common):** User was added to group but hasn't signed in since. Group membership claim is in a NEW token that must be generated. Or role assignment was just created and propagation delay applies.
**Fix:** Have user sign out/sign in; wait 15 minutes if new assignment.
**Validation:** User can access resources; sign-in log shows successful authentication.

### Scenario 4: "Contractor was given temporary Contributor access via group, but access persists after removal"
**Architecture:** Group-based RBAC → Token generation → Access validation.
**Dependencies:** Group membership, RBAC, Token lifetime, PIM.
**Checks:**
1. Is the user confirmed removed from the group?
2. Does the user have a direct RBAC assignment (not just via group)?
3. Check PIM: Is there an active PIM assignment?
4. Is the user's token still valid? (Tokens can be valid for hours after group removal)
5. Can tokens be revoked via Conditional Access?
**Root Cause: Active token still contains group membership claim. The user's access token is still valid (up to 60-90 minutes). Additionally, check PIM — if the assignment was PIM-activated, it may still be active.**
**Fix:** Revoke tokens via Conditional Access, deactivate PIM assignments, force sign-out.
**Validation:** User can no longer perform Contributor actions; sign-in log confirms access denied.

### Scenario 5: "Owner on subscription cannot modify network settings"
**Architecture:** RBAC Owner → Network resource management → Network service control plane.
**Dependencies:** RBAC, Network-specific permissions, Resource Provider, Locks, Policy.
**Checks:**
1. Does Owner include the specific network actions? (Owner should include ALL)
2. Is there a Deny Assignment for network modifications?
3. Is there a Policy that restricts network changes?
4. Is there a Resource Lock?
5. Is there a Service Limitation (e.g., a specific NIC can't be modified after a certain state)?
**Root Cause (common):** Deny Assignment, Policy, or Lock. Even Owners can be blocked by these mechanisms.
**Fix:** Review and adjust Deny Assignment/Policy/Lock as appropriate.
**Validation:** Modification succeeds; Activity Log shows authorized operation.

---

## 18. KNOWLEDGE TEST

1. **What are the three components of Azure RBAC?**
   Security Principal (who), Role Definition (what permissions), Role Assignment (where/how they're bound together).

2. **What is the difference between Actions and DataActions in a role definition?**
   Actions control management/control plane operations (create, read, update, delete resources). DataActions control data plane operations (reading/writing actual data within a resource, e.g., reading blob content).

3. **What is the scope hierarchy in Azure?**
   Root → Management Group → Subscription → Resource Group → Resource. Role assignments flow DOWN this hierarchy.

4. **What is the difference between Azure RBAC and Directory Roles?**
   Azure RBAC controls what you can do on Azure resources. Directory Roles control what you can administer in Entra ID. They are completely independent systems. A user can have one without the other.

5. **Why can't Contributor assign roles?**
   Contributor does not include `Microsoft.Authorization/*/write` permission. Only Owner and User Access Administrator (and roles with explicit write authorization) can create role assignments.

6. **What is a custom role?**
   A user-defined role with specific permissions tailored to operational needs. Built-in roles cannot be modified; custom roles fill the gap when built-in roles are too broad or too narrow.

7. **What is ABAC?**
   Attribute-Based Access Control. Extends RBAC by adding conditions to role assignments that filter permissions based on resource attributes, principal attributes, and environment conditions.

8. **What is a Deny Assignment?**
   A special authorization that explicitly denies a specific action, overriding any Allow grants. Deny takes precedence over RBAC Allow. Managed separately from standard RBAC.

9. **How long does RBAC role assignment propagation take?**
   Up to 15 minutes for full propagation across all Azure subsystems.

10. **Why assign RBAC to groups instead of users?**
    Centralized management, easier auditing, simpler onboarding/offboarding, and temporary access is just group add/remove.

11. **What permission does Contributor lack that Owner has?**
    Contributor cannot create role assignments (`Microsoft.Authorization/*/write`).

12. **Can Azure Policy override RBAC?**
    Yes. Azure Policy "Deny" effect can block actions even if the user has Owner/Contributor RBAC. This is separate from RBAC.

13. **What is the maximum number of custom roles per tenant?**
    5,000 (2,000 for Azure operated by 21Vianet).

14. **What happens if a Deny Assignment conflicts with an RBAC role?**
    Deny Assignment wins. Even Owners are blocked by Deny Assignments for the denied action.

15. **How do ABAC conditions affect existing tokens?**
    ABAC conditions are evaluated at each authorization check. New tokens reflect condition changes; existing tokens may still grant the old permissions until they expire (60-90 minutes).

16. **What are AssignableScopes in a custom role?**
    The scope levels where the custom role can be assigned. If a role's AssignableScopes doesn't include a scope, you cannot assign the role at that scope.

17. **What is the difference between RBAC scope and Conditional Access scope?**
    RBAC scope = which Azure resources the role applies to (subscription, RG, resource). Conditional Access scope = which users/apps/conditions the CA policy evaluates.

---

## 19. L3 GAP CHECK

| Topic | Status |
|-------|--------|
| Azure RBAC core concepts (Principal, Role, Assignment) | ✅ Covered |
| Scope hierarchy (Root, MG, Sub, RG, Resource) | ✅ Covered |
| Inheritance and propagation | ✅ Covered |
| Built-in roles (Owner, Contributor, Reader, etc.) | ✅ Covered |
| Role assignment creation and management | ✅ Covered |
| Propagation delay (15 minutes) | ✅ Covered |
| Deny Assignments | ✅ Covered |
| Custom roles (structure, creation, rules) | ✅ Covered |
| ABAC (conditions, syntax, sources, examples) | ✅ Covered |
| Directory Roles vs Azure RBAC (critical distinction) | ✅ Covered (Critical) |
| Least privilege and group-based assignments | ✅ Covered |
| Best practices | ✅ Covered |
| Production example | ✅ Covered |
| Failure scenarios (10 scenarios) | ✅ Covered |
| Troubleshooting methodology | ✅ Covered |
| Logs/evidence | ✅ Covered |
| Current service considerations | ✅ Covered |
| L3 interview questions (all levels) | ✅ Covered |
| Scenario-based questions | ✅ Covered |
| Knowledge test | ✅ Covered |
| L3 gap check | ✅ Covered |

---

## WHAT AN EXPERIENCED AZURE L3 ENGINEER SHOULD NOW BE ABLE TO EXPLAIN CONFIDENTLY

After Module 7, you should be able to confidently explain:

1. **The three pillars of RBAC** — Security Principal (WHO), Role Definition (WHAT), and Role Assignment (WHERE). Every access question involves all three, and troubleshooting starts with identifying which component is wrong.

2. **The critical distinction between Directory Roles and Azure RBAC** — Directory Roles (Global Admin, Conditional Access Admin, etc.) manage Entra ID administration. Azure RBAC (Owner, Contributor, Reader, etc.) manages Azure resources. They are completely independent. Global Admin does NOT equal Owner; Contributor does NOT equal Application Admin. This confusion causes more production incidents than any other single concept.

3. **How scope and inheritance work** — Role assignments flow down (sub → RG → resource). Multiple assignments add up (union). You cannot subtract permissions via narrower scope. Narrower scope does not override broader scope — it adds more permissions.

4. **Why Contributor and Owner are different** — Contributor cannot assign roles (no `Microsoft.Authorization/*/write`). Owner can delegate access. This distinction matters for security: a compromised Owner can escalate privileges by assigning roles.

5. **What ABAC does and why it matters** — ABAC adds conditions to role assignments, filtering permissions by resource tags, user attributes, and environment. This dramatically reduces the number of role assignments needed (from 500 to 5 in common scenarios) and enables precise, attribute-driven access control.

6. **How Deny Assignments override RBAC** — Even Owners can be blocked by Deny Assignments. This is critical for troubleshooting: if a user with Owner cannot perform an action, check Deny Assignments before assuming it's a Policy issue.

7. **Why RBAC propagation takes up to 15 minutes** — Azure's distributed system is eventually consistent. New or changed role assignments are not immediately effective. This causes "I just assigned the role but access still doesn't work" scenarios.

8. **How group-based RBAC assignments work** — Assigning to groups (not users) is the production best practice. But it introduces propagation delays, token caching, and PIM complications that must be understood for troubleshooting.

9. **Why tokens cache group membership** — A user removed from a group may still have access for hours because their existing access token contains the old group claim. This is a major security concern and troubleshooting pitfall.

10. **How to troubleshoot "Access Denied" systematically** — RBAC → Deny Assignment → Resource Lock → Azure Policy → ABAC condition → Resource Provider registration → Quota → Propagation delay. This checklist is the foundation of every RBAC-related troubleshooting investigation.

11. **Why custom roles exist** — Built-in roles are too broad for operational use. Custom roles provide precise permission sets (e.g., "can restart VMs but cannot delete them") that align with real job functions.

12. **How the four authorization systems interact** — RBAC (what you can do), Directory Roles (what you can administer), Conditional Access (under what conditions you can sign in), and Azure Policy (what resources/configurations are allowed). All four must be checked independently when a user cannot do something.

---

# Ready for Module 8 — Governance (Azure Policy, Initiatives, Assignments, Blueprints)?

It covers:
- Azure Policy deep dive (definitions, effects, assignments, parameters, remediation)
- Policy initiatives and policy sets
- ABAC integration
- Azure Blueprints
- Tagging strategies and enforcement
- Resource locks
- Management Group governance
- Allowed locations, allowed SKUs, required tags
- Policy exemptions
- Compliance and audit

Say **"Next module"** to continue.