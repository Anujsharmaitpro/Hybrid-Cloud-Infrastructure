# MODULE 2 — AZURE RESOURCE MANAGER (ARM) — DEEP DIVE

---

## 2.1 CONCEPT

Azure Resource Manager (ARM) is the **central management layer** of Azure. It is not a tool you "use" — it is the infrastructure that **every** Azure operation passes through. Every portal click, every CLI command, every API call, every Terraform run, every Bicep compilation, every PowerShell `New-AzResource` call — all of them ultimately reach ARM first.

ARM sits between the **consumer** (you, the portal, the CLI, Terraform, an API client) and the **Azure Resource Providers** (Microsoft.Compute, Microsoft.Storage, Microsoft.Network, etc.) that actually create and manage the underlying resources.

ARM's responsibilities:
1. **Authenticate** the request (validate token against Entra ID)
2. **Authorize** the request (check Azure RBAC at the appropriate scope)
3. **Evaluate Policy** (check all applicable Azure Policies before allowing the operation)
4. **Validate** the request (check syntax, dependencies, quotas)
5. **Orchestrate deployment** (create/update/delete resources in correct dependency order)
6. **Route** the request to the correct Resource Provider
7. **Report** the result (return status, update Activity Log, update resource state)
8. **Lock** resources during operations (to prevent concurrent modification conflicts)

ARM is **stateless** for each request but maintains a **global state database** that tracks:
- Every resource's current state (provisioning state: Succeeded, Failed, Running, etc.)
- Every deployment's history
- Every resource's dependencies
- Every role assignment, policy assignment, and lock

---

## 2.2 ARCHITECTURE

```
┌──────────────────────────────────────────────────┐
│                   CONSUMERS                       │
│  Portal | CLI | PowerShell | ARM Templates |     │
│  Bicep | Terraform | REST API | SDKs | KQL      │
└────────────────────────┬─────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────┐
│              AZURE RESOURCE MANAGER               │
│                                                   │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────┐ │
│  │  Authentication │  │  Authorization │  │ Policy │ │
│  │  (Entra ID)    │  │  (Azure RBAC)  │  │ Eval   │ │
│  └──────┬──────┘  └──────┬───────┘  └────┬─────┘ │
│         └────────────────┼───────────────┘        │
│                          ▼                        │
│         ┌──────────────────────────────┐           │
│         │    Deployment Engine          │           │
│         │  - Dependency resolution      │           │
│         │  - Atomic transactions        │           │
│         │  - Rollback on failure        │           │
│         │  - State management           │           │
│         └──────────────┬───────────────┘           │
│                        ▼                           │
│         ┌──────────────────────────────┐           │
│         │    Resource Provider Router   │           │
│         └──────────────┬───────────────┘           │
└────────────────────────┼──────────────────────────┘
                         │
            ┌────────────┼────────────┐
            ▼            ▼            ▼
    ┌──────────────┐ ┌──────────┐ ┌────────────┐
    │ Microsoft.   │ │ Microsoft│ │ Microsoft. │
    │ Compute      │ │ Network  │ │ Storage    │
    │ Microsoft.Web│ │ Microsoft│ │ Microsoft. │
    │ ...          │ │ SQL      │ │ KeyVault   │
    └──────────────┘ └──────────┘ └────────────┘
```

---

## 2.3 COMPONENTS — DETAILED

### 2.3.1 ARM DEPLOYMENT ENGINE

**What it is:**
The core orchestration engine within ARM that processes deployment requests — templates, Bicep, inline JSON, or imperative commands — and translates them into atomic operations against Resource Providers.

**How it works internally:**

```
Deployment Request Received
│
├─ Step 1: Validate
│   ├─ Parse template/Bicep → compile to ARM JSON
│   ├─ Validate syntax and schema
│   ├─ Resolve variables and parameters
│   └─ Check authentication and authorization
│
├─ Step 2: Policy Evaluation
│   ├─ Evaluate all policies applicable at the deployment scope
│   ├─ Any Deny effect → Block deployment immediately
│   ├─ Audit effects → Log but allow
│   └─ Modify/DeployIfNotExists → Plan modifications
│
├─ Step 3: Quota Check
│   ├─ Verify resource creation does not exceed quotas
│   └─ If exceeded → Fail with quota error
│
├─ Step 4: Dependency Analysis
│   ├─ Build dependency graph from dependsOn and resource references
│   ├─ Determine creation order (topological sort)
│   └─ Identify parallelizable operations
│
├─ Step 5: Execute
│   ├─ Lock resources being modified (to prevent conflicts)
│   ├─ Send create/update requests to Resource Providers in order
│   ├─ Each provider returns: Accepted → Running → Succeeded/Failed
│   └─ If any resource fails → initiate rollback for resources already created
│
├─ Step 6: Report
│   ├─ Update resource states in ARM database
│   ├─ Record deployment operations in Activity Log
│   └─ Return deployment result to caller
│
└─ Step 7: Post-Deployment
    ├─ Trigger any post-deployment scripts/extensions if specified
    └─ Update tags, outputs, and resource properties
```

**Critical ARM behavior:**
- ARM **locks resources during deployment** to prevent concurrent modifications from conflicting. This is an internal ARM mechanism — you may see temporary "locking" errors if another operation hits the same resource during deployment.
- ARM **does NOT guarantee ordering across deployments** — if two deployments touch the same resource simultaneously, one will wait or fail.
- ARM deployments are **atomic at the deployment scope level** — either all resources in a deployment succeed, or the deployment rolls back (but with caveats — see Section 2.7).

---

### 2.3.2 RESOURCE PROVIDERS — DEEP DIVE

**What they are:**
Resource Providers are Azure services that expose their resources through ARM. Each provider handles a specific namespace of resource types. When ARM needs to create a VM, it routes the request to `Microsoft.Compute`. When it needs to create a storage account, it routes to `Microsoft.Storage`.

**How they register:**

```
Subscription-level registration (classic):
  Register-AzResourceProvider -ProviderNamespace Microsoft.Compute
  Get-AzResourceProvider -ProviderNamespace Microsoft.Compute

Management Group-level registration (current, preferred for multi-sub):
  Register-AzResourceProvider -ProviderNamespace Microsoft.Compute -Scope /providers/Microsoft.Management/managementGroups/{mgName}
```

**Current state (Sept 2026):**
- **Auto-registration** is now the default for most providers in most subscriptions. Microsoft has been progressively moving providers to auto-registration.
- Legacy/manual registration is still supported but increasingly rare.
- Some providers (especially in sovereign clouds or specific regulatory scenarios) may still require manual registration.
- If a deployment fails with error code `MissingSubscriptionRegistration` or `ResourceProviderNotRegistered`, the provider needs to be registered.

**Provider lifecycle:**
```
Registration → Registration State: Registered
    ↓
Provider registers interest in subscription/region
    ↓
First deployment of that provider's resource type → Provider initializes
    ↓
Provider handles all subsequent CRUD operations for that resource type
```

**Multiple providers per service:**
Some Azure services have multiple resource providers:
- **Network security:** `Microsoft.Network` (NSGs, VNets, ExpressRoute) AND `Microsoft.ClassicNetwork` (deprecated)
- **Compute:** `Microsoft.Compute` (VMs, VMSS, Availability Sets) AND `Microsoft.Automation` (some Runbook-related compute)

**Provider dependency chain example:**
When creating a VM, ARM needs:
1. `Microsoft.Compute` — for the VM itself
2. `Microsoft.Network` — for VNet, Subnet, NIC, NSG, Public IP
3. `Microsoft.Storage` — for OS disk (and potentially data disks)
4. `Microsoft.Authorization` — for role assignments if part of deployment
5. `Microsoft.KeyVault` — if referencing Key Vault for encryption

If ANY of these providers is not registered, the deployment may fail.

---

### 2.3.3 RESOURCE TYPES

**Format:**
Resource types follow the pattern: `{ProviderNamespace}/{ResourceType}`

Examples:
| Resource Type | Provider | Example |
|---------------|----------|---------|
| Virtual Machines | `Microsoft.Compute/virtualMachines` | `myVM` |
| Virtual Networks | `Microsoft.Network/virtualNetworks` | `myVNet` |
| Storage Accounts | `Microsoft.Storage/storageAccounts` | `stmyaccount` |
| Key Vaults | `Microsoft.KeyVault/vaults` | `myKV` |
| NSGs | `Microsoft.Network/networkSecurityGroups` | `myNSG` |
| Resource Groups | `Microsoft.Resources/resourceGroups` | `myRG` |
| Deployments | `Microsoft.Resources/deployments` | `myDeployment` |
| Policy Assignments | `Microsoft.PolicyInsights/policyAssignments` | (global naming) |

**Nested resource types:**
Some resources have child types:
```
Microsoft.Compute/virtualMachines/extensions         (extension on VM)
Microsoft.Network/virtualNetworks/subnets             (subnet in VNet)
Microsoft.Network/virtualNetworks/providers/...       (nested network resources)
Microsoft.KeyVault/vaults/secrets                     (secret in Key Vault)
Microsoft.Storage/storageAccounts/blobServices/containers  (container in storage)
```

**Why this matters for L3:**
- RBAC role definitions reference resource types (e.g., `Microsoft.Compute/virtualMachines/write`)
- Policy policyDefinition references resource types (e.g., "if type is Microsoft.Storage/storageAccounts")
- ARM template resources reference resource types in the `apiVersion` and `type` fields
- Troubleshooting permission errors requires knowing the exact resource type namespace

---

### 2.3.4 RESOURCE IDs — COMPLETE DEEP DIVE

**Format:**
```
/subscriptions/{subscription-id}/resourceGroups/{rg-name}/providers/{provider-namespace}/{resource-type}/{resource-name}
```

**Example:**
```
/subscriptions/abc123/resourceGroups/prod-rg/providers/Microsoft.Compute/virtualMachines/web-server-01
```

**For nested resources:**
```
/subscriptions/abc123/resourceGroups/prod-rg/providers/Microsoft.Network/virtualNetworks/vnet-app/subnets/subnet-web
```

**For resources with resource providers (e.g., databases inside a server):**
```
/subscriptions/abc123/resourceGroups/prod-rg/providers/Microsoft.Sql/servers/sql-server-01/databases/mydb
```

**Critical properties:**
| Property | Immutable? | Notes |
|----------|------------|-------|
| Subscription ID | Yes | Changing subscription requires resource move |
| Resource Group name | Yes | Moving between RGs is supported for most resources |
| Provider namespace | Yes | Cannot change the Azure service |
| Resource type | Yes | Cannot change VM to Storage |
| Resource name | Yes* | *Most types; some support rename |
| Location | No (for most) | Can be changed for some resource types |
| Tags | No | Can be updated anytime |
| Properties | No | Service-specific, configurable |
| Identity | No | Can add/remove managed identities |
| Dependencies (dependsOn) | No | Changes during redeployment |

**How ARM uses Resource IDs:**
1. Every ARM operation references a target resource by ID
2. Role assignments reference the scope (which includes the resource ID or higher-level container ID)
3. Policy assignments reference a scope ID
4. ARM templates reference existing resources by ID (for `reference()` and `listXxx()` functions)
5. Activity Log entries record the resource ID of every operation

**L3 troubleshooting:**
When you see an error referencing a resource ID, you can immediately determine:
- Which subscription it's in
- Which Resource Group
- Which provider/service
- Which specific resource
- What type of resource

---

### 2.3.5 CONTROL PLANE — ARM'S ROLE

ARM is THE control plane for Azure. It handles:

| Control Plane Operation | What ARM Does |
|------------------------|---------------|
| Resource creation | Validates → checks RBAC → checks Policy → routes to provider → creates → reports state |
| Resource read | Authenticates → checks RBAC (Reader/Owner) → retrieves from provider → returns |
| Resource update | Same as create, but modifies existing resource |
| Resource delete | Same as create, but tears down resource → waits for provider confirmation → marks deleted |
| Resource movement | Validates move compatibility → updates containment → updates provider references |
| Deployment | Orchestrates multi-resource creation/update/delete as a unit |
| Lock management | Creates/reads/deletes locks on resources |
| Role assignment | Creates/reads/deletes RBAC role assignments |
| Policy assignment | Creates/reads/deletes policy assignments |

**ARM API endpoint:**
`https://management.azure.com/` — All management operations go through this endpoint (or regional ARM endpoints for some operations).

---

### 2.3.6 DATA PLANE — THE SERVICE ITSELF

The **data plane** is what happens AFTER ARM has done its job. Once a resource exists, the data plane is the resource provider's own operational path.

**Key distinction examples:**

| Scenario | Control Plane (ARM) | Data Plane (Resource Provider) |
|----------|---------------------|-------------------------------|
| VM | Creating the VM, changing size, restarting via Portal | VM booting, OS running, application processing |
| Storage | Creating storage account, configuring firewall | Reading/writing blobs, listing containers |
| Key Vault | Creating vault, setting access policies | Retrieving/adding secrets, keys, certificates |
| NSG | Creating NSG rule | Evaluating network traffic against rules |
| App Service | Creating App Service, configuring scaling | Running the app, handling HTTP requests |

**L3 troubleshooting framework based on this distinction:**

```
"I cannot configure the resource"
→ Control Plane problem
→ Check: RBAC, Policy, ARM state, Resource Provider registration, API throttling

"The resource is configured, but it doesn't work as expected"
→ Data Plane problem
→ Check: Resource-specific configuration, networking, dependencies, resource health, service issues
```

**Real example — Storage Account:**
```
Control Plane: I try to create a storage account → Fails with "AuthorizationFailed"
→ Cause: RBAC permission missing to write to Microsoft.Storage/storageAccounts

Control Plane: I try to create a storage account → Fails with "PolicyViolation"
→ Cause: Azure Policy denies storage accounts in this region

Data Plane: Storage account exists, but I cannot read blobs → Fails with "403 Forbidden"
→ Cause: RBAC storage permission missing, OR SAS invalid, OR firewall blocks, OR Private Endpoint/DNS issue
```

---

## 2.4 SUB-COMPONENTS — ARM INTERNALS

### 2.4.1 ARM DATABASE (Resource Manager Database)

ARM maintains a **global distributed database** (internally called the "Resource Manager Database" or "ARM DB") that stores the **desired state** and **current state** of every Azure resource.

**What it stores:**
- Resource metadata (name, type, location, tags, identity, properties)
- Provisioning state (Succeeded, Failed, Accepted, Running, Deleted)
- Resource dependencies
- Deployment history
- Role assignments
- Policy assignments
- Locks
- Resource references and dependencies

**Why this matters for L3:**
- When you query a resource via Portal/CLI, you're often reading from this database, NOT directly from the resource provider.
- A resource may show as "Succeeded" in ARM DB but actually have issues (data plane problem).
- Deployment operations update this database before and after provider execution.
- **This is why a resource can show as "Running" in Portal but not be actually reachable** — ARM DB says it succeeded, but the data plane has a problem.

**Important L3 concept:**
```
ARM DB state ≠ Resource Health ≠ Application functionality

ARM DB: "VM is Succeeded"          → Resource was created OK
Resource Health: "OK"              → No platform issue affecting VM
Application: "Unreachable"         → Network/DNS/OS/App issue
```

These three are independent and must be checked separately.

### 2.4.2 ARM Deployment History

Every deployment is recorded with:
- Deployment name
- Scope (Resource Group, Subscription, etc.)
- Template used (hash of template content)
- Parameters
- Start/end time
- Status (Succeeded/Failed/Canceled)
- Operation results per resource
- Error messages for failures
- Correlation ID (for support/escalation)
- Who triggered it (identity)

**Where to find it:**
- Portal: Resource Group → Deployments
- CLI: `Get-AzDeployment -ResourceGroupName myRG`
- API: `GET /subscriptions/{sub}/resourcegroups/{rg}/deployments`

### 2.4.3 ARM Correlation ID

Every ARM operation gets a **Correlation ID** — a GUID that ties together all operations for a single request across all resource providers.

**Why it matters:**
- When a deployment fails, the Correlation ID links ALL related operations (across multiple providers) into one trace.
- When you open a support ticket with Microsoft, you provide the Correlation ID and Microsoft can trace exactly what happened internally across all services.
- Activity Log entries include the Correlation ID.

**How to use it:**
```
1. Deployment fails → Note the Correlation ID from error or Activity Log
2. Activity Log → Filter by Correlation ID → See ALL related operations
3. Each operation entry shows:
   - Resource affected
   - Operation name
   - Status (Success/Failure)
   - Timestamp
   - Error details (if failed)
4. Pass Correlation ID to Microsoft Support for deep investigation
```

### 2.4.4 ARM API Versions

Every resource type has an associated **API version**. This determines:
- Which properties are available
- Which behaviors are supported
- Which schema the ARM template must use

**How to find the current API version:**
- ARM Template reference documentation (e.g., `Microsoft.Compute/virtualMachines`)
- Azure CLI auto-resolves the latest stable version when you create resources
- Each resource type's documentation lists supported API versions

**Important L3 point:**
- Using an outdated API version may mean you miss newer features.
- Using a too-new API version may not be GA.
- ARM will reject operations with unsupported or invalid API versions.

**Current (Sept 2026) best practice:**
- Use the latest GA API version for new deployments.
- Pin API versions in production templates for stability.
- Be aware that API version changes can change behavior (e.g., new default values, different validation).

---

## 2.5 DEPENDENCIES — COMPLETE DEEP DIVE

### 2.5.1 Explicit Dependencies (`dependsOn`)

In ARM templates, you can specify explicit dependencies:

```json
{
  "type": "Microsoft.Compute/virtualMachines",
  "name": "myVM",
  "dependsOn": [
    "[resourceId('Microsoft.Network/networkInterfaces', 'myNIC')]",
    "[resourceId('Microsoft.Compute/virtualMachineExtensions', 'myExtension')]"
  ]
}
```

**What this does:**
- ARM will NOT create `myVM` until `myNIC` and `myExtension` resources are successfully provisioned.
- This creates a strict ordering in the deployment.

**When to use:**
- When a resource REQUIRES another to exist first (VM requires NIC).
- When you want to ensure ordering even if ARM could figure it out on its own (for clarity/control).

### 2.5.2 Implicit Dependencies (Resource References)

ARM automatically detects dependencies when one resource **references** another using `reference()` or `listXxx()` functions:

```json
{
  "type": "Microsoft.Compute/virtualMachines",
  "name": "myVM",
  "properties": {
    "networkProfile": {
      "networkInterfaceId": "[resourceId('Microsoft.Network/networkInterfaces', 'myNIC')]"
    }
  }
}
```

Here, the VM references the NIC via `resourceId()`. ARM automatically adds an implicit dependency: VM creation waits for NIC creation.

**Important:** The `reference()` function is a runtime function — it is evaluated AFTER the referenced resource exists. ARM uses the `resourceId()` call (which is compile-time) to detect the dependency, then uses `reference()` at deployment time to get properties.

### 2.5.3 Dependency Resolution Algorithm

```
1. Parse all resources in the template
2. For each resource, identify:
   a. Explicit dependsOn references
   b. Implicit references via resourceId() / reference() / listXxx() calls
3. Build a directed acyclic graph (DAG)
4. Perform topological sort
5. Group resources with no dependencies → deploy in parallel
6. After each group succeeds → deploy next group
7. If any resource fails → stop, initiate rollback for completed resources
```

**Parallelization:**
ARM deploys independent resources **in parallel** to speed up deployments. A large deployment with 50 resources may have 20 resources deploy in parallel in the first batch.

**Circular dependencies:**
If ARM detects a circular dependency (A depends on B, B depends on A), the deployment fails immediately with an error like "CycleDetected."

### 2.5.4 Cross-Deployment Dependencies

Resources can reference resources from **other deployments** using `reference()` with a resource ID:

```json
{
  "type": "Microsoft.Compute/virtualMachines",
  "name": "myVM",
  "properties": {
    "storageProfile": {
      "osDisk": {
        "managedDisk": {
          "id": "[resourceId('subId', 'rgName', 'Microsoft.Compute/disks', 'myDisk')]"
        }
      }
    }
  }
}
```

This creates a dependency across deployments — ARM ensures the referenced resource exists before creating this one.

### 2.5.5 Common Dependency Patterns

**VM creation dependency chain:**
```
Virtual Network
  → Subnet
    → Network Security Group (optional, can be separate)
      → Public IP (optional)
        → Network Interface (NIC)
          → Virtual Machine
            → VM Extensions (after VM is created)
```

**Storage dependency chain:**
```
Storage Account
  → Blob Service (implicit)
    → Container (implicit)
      → Blob (created via data plane, not ARM directly)
```

**Key Vault + VM + Disk encryption:**
```
Key Vault (must exist first)
  → VM
    → Disk (OS disk)
      → Disk Encryption Set (references Key Vault)
        → VM Extension (Azure Disk Encryption)
```

---

## 2.6 ARM TEMPLATES — DEEP DIVE

### 2.6.1 What They Are

ARM Templates are **JSON documents** that declare the desired state of Azure infrastructure. They are the foundational Infrastructure as Code (IaC) format for Azure.

### 2.6.2 Structure

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": { ... },
  "variables": { ... },
  "functions": [ ... ],
  "resources": [ ... ],
  "outputs": { ... }
}
```

**Each section:**

| Section | Purpose | Required? |
|---------|---------|-----------|
| `$schema` | Points to schema definition for validation | Yes |
| `contentVersion` | Version of the template | Yes |
| `parameters` | Input values passed at deployment time | No (but recommended) |
| `variables` | Computed values used within template | No |
| `functions` | Custom user-defined functions | No |
| `resources` | Array of resources to deploy | Yes (at least one) |
| `outputs` | Values returned after deployment | No |

### 2.6.3 Parameters — Deep Dive

```json
{
  "parameters": {
    "vmName": {
      "type": "string",
      "defaultValue": "myVM",
      "metadata": { "description": "Name of the VM" },
      "allowedValues": ["myVM", "myVM2"],
      "minLength": 1,
      "maxLength": 64
    },
    "vmSize": {
      "type": "string",
      "defaultValue": "Standard_D2s_v5",
      "metadata": { "description": "VM SKU" }
    },
    "adminPassword": {
      "type": "securestring",
      "metadata": { "description": "Admin password" }
    },
    "tags": {
      "type": "object",
      "defaultValue": { "environment": "prod", "department": "engineering" },
      "metadata": { "description": "Tags to apply" }
    }
  }
}
```

**Parameter types:** `string`, `int`, `bool`, `securestring`, `array`, `object`

**Deployment time parameter passing:**
- Via Portal: Form-based entry
- Via CLI: `--parameters vmName=myVM vmSize=Standard_D4s_v5`
- Via PowerShell: `-TemplateParameterObject @{vmName="myVM"; vmSize="Standard_D4s_v5"}`
- Via JSON parameter file: `--parameters parameters.json`

### 2.6.4 Variables

```json
{
  "variables": {
    "vnetName": "[concat('vnet', parameters('envName'))]",
    "subnetAddressPrefix": "[concat(variables('vnetAddressPrefix'), '/24')]",
    "vmSize": "[parameters('vmSize')]",
    "location": "[resourceGroup().location]"
  }
}
```

**Key difference from parameters:**
- Parameters are input from outside the template.
- Variables are computed inside the template and can reference parameters, other variables, and built-in functions.

### 2.6.5 ARM Template Functions (Key Built-in Functions)

| Function | Purpose | Example |
|----------|---------|---------|
| `resourceId()` | Constructs full resource ID | `resourceId('Microsoft.Network/virtualNetworks', 'myVNet')` |
| `resourceGroup()` | Returns current RG properties | `resourceGroup().id`, `resourceGroup().location`, `resourceGroup().name` |
| `subscription()` | Returns subscription properties | `subscription().subscriptionId`, `subscription().tenantId` |
| `reference()` | Gets a resource's runtime properties | `reference('myVNet').properties.addressSpace.addressPrefixes` |
| `listXxx()` | Gets a list operation result | `listKeys('myStorage', '2021-04-01')` |
| `concat()` | Concatenates strings | `concat('prefix-', parameters('env'), '-suffix')` |
| `format()` | String formatting | `format('{0}-{1}', 'app', 'prod')` |
| `uniqueString()` | Hash-based deterministic string | `uniqueString(resourceGroup().id)` (used for random naming) |
| `guid()` | Generates GUID | `guid('myString')` |
| `parameters()` | Access parameter value | `parameters('vmName')` |
| `variables()` | Access variable value | `variables('vnetName')` |
| `resource()` | Gets another resource's declared properties (template-time) | `resource('myVNet', 'Microsoft.Network/virtualNetworks', '2021-04-01')` |
| `extensionResourceId()` | Constructs ID for extension resource | `extensionResourceId(...)` |
| `listAccountId()` | Gets managed identity account ID | `listAccountId('myKeyVault', '2021-04-01')` |

**Critical runtime vs compile-time distinction:**
- **Compile-time functions** evaluated when template is deployed: `resourceId()`, `concat()`, `parameters()`, `variables()`, `uniqueString()`, `guid()`
- **Runtime functions** evaluated after resources are created: `reference()`, `listXxx()`
- This matters because `reference()` can ONLY reference resources that are in the same template (or cross-referenced with explicit resourceId), while compile-time functions can reference anything.

---

## 2.7 DEPLOYMENT SCENARIOS — DEEP DIVE

### 2.7.1 Deployment Scopes

| Scope | Target | Example |
|-------|--------|---------|
| **Resource** | Single resource | Updates to a specific resource via its resource ID |
| **Resource Group** | All resources in one RG | Most common deployment target |
| **Subscription** | Resources across RGs in one subscription | Subscription-level role assignments, policy, some resources (e.g., `Microsoft.Subscription` |
| **Management Group** | Resources across subscriptions | Cross-subscription deployments, policy initiatives |
| **Tenant** | Tenant-wide | Tenant-level policy, Entra ID resources (via ARM), global configurations |

**Scope hierarchy for inheritance:**
```
Tenant deployment → applies to entire tenant
Management Group deployment → applies to all child subscriptions
Subscription deployment → applies to all RGs in the subscription
RG deployment → applies to resources within the RG
Resource deployment → applies to one resource
```

**Important:** A deployment at a parent scope can create/manage resources in child scopes, but child scope deployments CANNOT affect parent scopes.

### 2.7.2 Deployment Modes

| Mode | Behavior | Use Case | Risk |
|------|----------|----------|------|
| **Incremental** (default) | Creates new, updates existing, leaves untouched | Normal production deployments | Lower — no unexpected deletions |
| **Complete** | Creates/updates, AND **deletes** resources that exist but are NOT in the template | Full infrastructure sync, cleanup | Higher — accidental deletions if template is outdated |
| **What-If** | Shows a change analysis WITHOUT executing any changes | Pre-deployment validation | None — read-only |

**Complete mode danger:**
In Complete mode, if your template doesn't include a resource that exists in the RG, ARM will **delete** it. This has caused production outages when engineers forgot that a resource was managed outside the template (manually created, created by another template, or from a different deployment scope).

**L3 best practice:**
- Always use Incremental for production.
- Use What-If before any deployment to validate.
- If Complete mode is necessary, first run What-If and carefully review all actions.

### 2.7.3 What-If (Change Analysis)

```bash
az deployment group what-if \
  --resource-group myRG \
  --template main.bicep \
  --parameters params.json
```

Output shows for each resource:
- **Modify** — Property changes (before → after)
- **Create** — New resource
- **Delete** — Will be deleted (in Complete mode)
- **Ignore** — No changes
- **Deploy** — Will be created/updated

**Color coding:**
- 🟢 Green = No change / Will be created
- 🟡 Yellow = Will be modified
- 🔴 Red = Will be deleted

**L3 usage:**
- Always run What-If before production deployments.
- Check specifically for **Delete** actions (red) — these are the dangerous ones.
- Verify that no unexpected resources will be deleted.
- Confirm that property changes are expected.

---

## 2.8 DEPLOYMENT FAILURES — COMPLETE DEEP DIVE

### 2.8.1 Types of Deployment Failures

**Category 1: Pre-deployment failures (before ARM touches providers)**
| Failure | Cause | Resolution |
|---------|-------|------------|
| AuthorizationFailed | RBAC permission insufficient | Assign appropriate role |
| PolicyViolation | Azure Policy denies operation | Check policy, comply, or request exemption |
| TemplateValidationFailed | JSON syntax error, missing required property, schema violation | Fix template, validate with linter |
| QuotaExceeded | Resource exceeds subscription/region quota | Request quota increase |
| LockedEntity | Resource is locked | Remove lock |
| ProviderNotRegistered | Resource provider not registered | Register provider |

**Category 2: Runtime failures (during provider execution)**
| Failure | Cause | Resolution |
|---------|-------|------------|
| ResourceSpecificDeploymentFailed | Provider-specific error (e.g., VM allocation failure, invalid disk configuration) | Check error details, resolve provider-specific issue |
| AllocationFailed | Insufficient capacity in region for specific SKU | Change SKU, change region, wait for capacity |
| DuplicateVMName | VM name already exists in region | Use unique name |
| DiskIOError | Storage backend issue | Retry, contact support |
| NetworkSecurityGroupAssociationFailed | NSG cannot be associated | Check subnet/NSG configuration |

**Category 3: Partial failures (some resources succeed, some fail)**
- In Incremental mode, resources already successfully created are NOT rolled back (unless the deployment is explicitly canceled).
- ARM returns a failure status for the overall deployment but individual resources that succeeded remain in Succeeded state.
- **This is a critical L3 concept:** A deployment can have status "Failed" but most resources are "Succeeded."

### 2.8.2 Failed Deployment Handling

**What ARM does on failure:**
```
1. ARM detects failure (from provider response or timeout)
2. ARM marks the failing resource as "Failed"
3. ARM stops processing remaining resources in the deployment
4. For Incremental mode: Resources already created remain as-is (NOT rolled back automatically)
5. For Complete mode: ARM attempts to rollback resources created in this deployment that are NOT in the final state
6. All completed operations are logged in Activity Log
7. Deployment status = Failed
8. Error details include Correlation ID
```

**Critical behavior:** ARM does NOT always automatically rollback. In most cases, resources already successfully created remain in place. You must either manually clean up or fix the issue and redeploy.

### 2.8.3 Troubleshooting Failed Deployments

```
Step 1: Check deployment status
  → Portal: Deployments → Select failed deployment → Overview
  → CLI: Get-AzDeployment -ResourceGroupName myRG -Name myDeploy

Step 2: Check error message
  → ErrorCode (e.g., AuthorizationFailed, PolicyViolation, AllocationFailed)
  → ErrorMessage (human-readable)
  → TargetResource (which resource failed)
  → Correlation ID

Step 3: Check deployment operations
  → Portal: Deployments → Failed deployment → Operations
  → Shows per-resource status: Succeeded, Failed, Skipped, NotStarted

Step 4: Identify the first failure
  → Resources are processed in dependency order
  → The first failure is the root cause; subsequent failures are usually cascading

Step 5: Check Activity Log
  → Filter by Correlation ID
  → See all related operations across providers

Step 6: Check resource-specific logs
  → Diagnostic settings for the resource type
  → Resource provider's own operation logs

Step 7: Resolve and redeploy
  → Fix the root cause
  → Redeploy the same template (idempotent for Incremental mode)
  → Use What-If first to validate
```

---

## 2.9 ARM TEMPLATE ROLLBACK

### 2.9.1 How Rollback Works

**In Incremental mode:**
- If deployment fails, ARM does NOT automatically rollback already-created resources.
- You must manually delete or update those resources.
- However, you CAN redeploy a corrected template, which will update/create resources to match the desired state.

**In Complete mode:**
- ARM attempts to delete resources that were created during the deployment but are no longer in the template (i.e., rollback to pre-deployment state).
- This is inherently risky.

**In What-If mode:**
- No rollback needed — nothing was executed.

### 2.9.2 Manual Rollback Process

```bash
# Option 1: Re-deploy corrected template
az deployment group create --resource-group myRG --template corrected.bicep --parameters params.json

# Option 2: Delete resources created by the failed deployment
# Identify from Activity Log / Deployment Operations
az resource delete --id /subscriptions/.../resourceGroups/.../providers/Microsoft.Compute/virtualMachines/failedVM

# Option 3: Use Deployment script resource (for complex rollbacks)
# Include a deployment script in your template that handles cleanup
```

### 2.9.3 L3 Best Practice for Rollback

1. Always test in a non-production environment first.
2. Use What-If before every production deployment.
3. Use Incremental mode (never Complete for production).
4. Maintain versioned templates in a repository.
5. Tag deployments with correlation to change tickets.
6. Have a documented rollback procedure per deployment type.
7. Use CI/CD pipelines with automated What-If validation gates.

---

## 2.10 BICEP — DEEP DIVE

### 2.10.1 What It Is

Bicep is Azure's **domain-specific language (DSL)** for ARM template authoring. It compiles to standard ARM JSON. Bicep is NOT a separate execution engine — it produces ARM JSON, which ARM then executes.

**Why it exists:**
- ARM JSON is verbose and difficult to read/write.
- Bicep provides a cleaner syntax with the same capabilities.
- Bicep compiles to ARM JSON, so all ARM features are available.

### 2.10.2 Bicep vs ARM JSON

| Aspect | ARM JSON | Bicep |
|--------|----------|-------|
| Readability | Verbose, deeply nested | Clean, indented, concise |
| Variables | `"variables": { ... }` block | `var` keyword |
| Loop | `"copy": { ... }` | `for` loop |
| Condition | `"condition": true` | `if` expression |
| Resource references | `resourceId(...)` function everywhere | Direct reference by name |
| Modules | `"modules": [...]` | `module` keyword with separate files |
| Parameters | `"parameters": { ... }` | `param` keyword |
| Type safety | No type checking | Strong typing |
| Schema validation | JSON schema | Bicep type system |
| Learning curve | Must learn ARM JSON patterns | More intuitive for developers |

### 2.10.3 Bicep Compilation

```bash
# Compile Bicep to ARM JSON
az bicep build --file main.bicep --outfile main.json

# Watch mode (auto-compile on change)
az bicep build --file main.bicep --outfile main.json --watch

# Check syntax
az bicep build --file main.bicep --stdout > /dev/null
```

**Compilation output:**
- Produces valid ARM JSON
- All `param`, `var`, resource references are resolved
- `resourceId()` calls are auto-generated from Bicep references
- The output JSON is identical to what you would write manually

### 2.10.4 Key Bicep Features for L3

**Parameter syntax:**
```bicep
param vmName string = 'myVM'
param vmSize string = 'Standard_D2s_v5'
param tags object = {
  environment: 'prod'
  department: 'engineering'
}
```

**Variable syntax:**
```bicep
var vnetName = 'vnet-${envName}'
var subnetPrefix = cidrNETwork(vnetAddressPrefix, 1, 0) // not actual function name, just example pattern
```

**Resource reference (no resourceId() needed!):**
```bicep
resource myVNet 'Microsoft.Network/virtualNetworks@2023-04-01' = {
  name: 'vnet-app'
  location: resourceGroup().location
  properties: {
    addressSpace: {
      addressPrefixes: ['10.0.0.0/16']
    }
  }
}

resource myVM 'Microsoft.Compute/virtualMachines@2023-04-01' = {
  name: vmName
  location: resourceGroup().location
  dependsOn: [
    myNIC
  ]
  properties: {
    networkProfile: {
      networkInterfaces: [
        {
          id: myNIC.id  // ← Direct reference, Bicep auto-generates resourceId()
        }
      ]
    }
  }
}
```

**Modules (reusable components):**
```bicep
module vmModule 'vm.bicep' = {
  name: 'deployVM'
  params: {
    vmName: 'myVM'
    vmSize: 'Standard_D2s_v5'
    vnetId: myVNet.id
    subnetId: mySubnet.id
  }
}
```

**Loops:**
```bicep
resource vmArray 'Microsoft.Compute/virtualMachines@2023-04-01' = [for i in range(0, 3): {
  name: 'vm-${i}'
  ...
}]
```

**Conditions:**
```bicep
resource publicIP 'Microsoft.Network/publicIPAddresses@2023-04-01' = if (enablePublicIP) {
  name: 'pip-vm'
  ...
}
```

### 2.10.5 Bicep Current Status (Sept 2026)

- Bicep is **GA** and Microsoft's recommended IaC tool for Azure-native deployments.
- Integrated into Visual Studio, VS Code, and Azure Portal.
- Supports all ARM template features.
- Has its own CLI, VS Code extension, and language server.
- Community and Microsoft actively maintain the Bicep registry (modules, examples).
- Bicep files are NOT deployable directly — they must be compiled to ARM JSON first (though tools like `az deployment group create --template-file main.bicep` auto-compile on the fly).

---

## 2.11 DEPLOYMENT SECURITY

### 2.11.1 Security Concerns with ARM Deployments

| Concern | Risk | Mitigation |
|---------|------|------------|
| **Over-privileged deployment identity** | Deployment can create any resource in scope | Use least-privilege RBAC for deployment identity |
| **Template injection** | Malicious template can create hidden resources | Review templates before deploying, use What-If |
| **Secret in template** | Parameters may contain secrets in plain text | Use securestring, Key Vault reference, Azure Key Vault connections |
| **Complete mode** | Unexpected deletions | Use Incremental, What-If validation |
| **Unverified templates** | Templates from untrusted sources | Use only from trusted repositories |
| **Deployment history** | Past deployments contain sensitive info | Secure Activity Log, access controls |
| **Service principal expiry** | Automated deployments fail when SP secret expires | Monitor expiry, automate rotation |

### 2.11.2 Secure Parameter Patterns

**Bad (avoid):**
```json
{
  "parameters": {
    "adminPassword": {
      "type": "string",
      "defaultValue": "P@ssw0rd123"
    }
  }
}
```

**Better (avoid plaintext):**
```json
{
  "parameters": {
    "adminPassword": {
      "type": "securestring"
    }
  }
}
```

**Best (Key Vault reference — no secret in template):**
```bicep
param keyVaultName string
param secretName string

resource kv 'Microsoft.KeyVault/vaults@2023-04-01' existing = {
  name: keyVaultName
}

// In the VM resource:
{
  "osProfile": {
    "adminPassword": "getSecret(${kv.id}, '${secretName}', '')"
  }
}
```

With Bicep, the syntax is even cleaner using the `@description` and Key Vault object reference:
```bicep
param adminPassword object = {
  reference: {
    keyVault: keyVaultName
    secret: secretName
  }
}
```

---

## 2.12 ARM & RBAC INTERACTION

### 2.12.1 How RBAC Affects ARM Operations

Every ARM operation follows this sequence:
```
Request arrives at ARM
→ ARM authenticates identity (Entra ID token)
→ ARM checks RBAC for the specific operation
  → Has the identity been assigned a role that includes this action?
  → At what scope? (resource, RG, subscription, MG)
  → Is inheritance in play?
→ If allowed: proceed
→ If denied: return AuthorizationFailed, STOP. Never reaches Resource Provider.
```

**RBAC roles and ARM operations:**
| Role | ARM Operations Allowed |
|------|----------------------|
| Owner | All operations + delete role assignments |
| Contributor | All operations except role assignment and lock management |
| Reader | Read operations only |
| User Access Administrator | Manage role assignments |
| Managed Identity | Depends on assigned role |

### 2.12.2 RBAC Scoping for Deployments

**To deploy resources, you need:**
- Write permission on the resource types you're creating (e.g., `Microsoft.Compute/virtualMachines/write`)
- At a scope that includes the deployment target (Resource Group or higher for RG deployment)
- **Additionally:** `Microsoft.Resources/deployments/write` for deployment operations themselves

**Common deployment failure:**
- User has `Contributor` on Resource Group but attempts deployment at Subscription scope → Fails (scope too broad).
- User has `Reader` on Subscription but attempts deployment → Fails (no write permission).
- User has `Contributor` on a Resource Group but deployment targets a different Resource Group → Fails (scope doesn't cover target).

### 2.12.3 RBAC and Resource Ownership in Deployments

After deployment, the **deployer** does NOT automatically become the resource owner. Resources inherit access from their container (Resource Group/subscription). The deployment identity only needs permissions during the deployment — afterwards, RBAC on the resources is determined by separate assignments.

---

## 2.13 ARM & AZURE POLICY INTERACTION

### 2.13.1 Policy Evaluation During Deployment

```
ARM receives deployment request
→ ARM authenticates & authorizes (RBAC check)
→ ARM evaluates ALL applicable Azure Policies at the deployment scope
  → For each policy:
    - Is the deployment target a resource covered by the policy?
    - Does the policy's condition match?
    - What is the policy's effect?
      • Deny → BLOCK deployment immediately, return denial error
      • Audit → Log violation, ALLOW deployment
      • AuditIfNotExists → Log if resource doesn't exist, ALLOW
      • Modify → Apply modification, then ALLOW
      • DeployIfNotExists → Deploy the required resource if missing, then ALLOW
      • Disabled → Ignore this policy
→ If all passed: proceed to provider execution
```

### 2.13.2 Real Example: Policy Blocking Deployment

**Scenario:** An enterprise has this Azure Policy:
- **Policy:** "Allowed locations"
- **Effect:** Deny
- **Allowed locations:** westus, westeurope
- **Scope:** Management Group (Corp-Prod)

Engineer in Portal tries to create a VM in `eastus`:
```
Result: Deployment fails with PolicyViolation
Message: "The resource 'myVM' is not allowed in location 'eastus'. Allowed locations: westus, westeurope."
```

**Resolution:**
1. Compliant option: Create VM in westus or westeurope
2. Policy exception: Request exemption for this subscription/resource (documented justification)
3. Policy change: Update policy to include eastus (if approved)

### 2.13.3 Policy and RBAC: The Critical Distinction

**RBAC:** "Can this user perform this action?" → Yes/No
**Policy:** "Is this action's result compliant?" → Allowed/Denied/Audited

**A user with Owner (RBAC) can attempt the action, but Policy may still deny it.**

This is the #1 misconception among engineers:
```
❌ "I have Owner, so I should be able to create anything."
✅ "I have Owner, so I can attempt the creation. Azure Policy may still block it."
```

---

## 2.14 ARM & LOCK INTERACTION

### 2.14.1 How Locks Affect Deployments

| Lock Level | Effect on Deployment |
|------------|---------------------|
| Resource-level `CanNotDelete` | Can still create/update the resource; just can't delete it |
| Resource-level `ReadOnly` | Cannot update the resource; deployment of changes to that resource fails |
| RG-level `CanNotDelete` | Cannot delete any resource in the RG; other operations OK |
| RG-level `ReadOnly` | Cannot create/update/delete any resource in the RG |
| Subscription-level locks | Apply to all resources in subscription |
| Higher-level locks | Inherited downward; strongest lock wins |

### 2.14.2 Lock Conflicts During Deployment

**Scenario:** Resource has `ReadOnly` lock
```
Deployment tries to update the resource → ARM checks locks → Lock exists → Deployment fails
Error: "The resource is locked and cannot be modified."
```

**Scenario:** Resource Group has `CanNotDelete` lock
```
Deployment includes a resource deletion step (Complete mode or explicit delete) → Lock prevents deletion → Deployment may partially fail
```

### 2.14.3 Troubleshooting Lock Issues

```bash
# Check locks on a resource
Get-AzResourceLock -ResourceGroupName myRG -ResourceName myResource -ResourceType Microsoft.Compute/virtualMachines

# Check locks on a Resource Group
Get-AzResourceLock -ResourceGroupName myRG

# Check locks at subscription level
Get-AzResourceLock -DefaultProfile $context

# Remove a lock
Remove-AzResourceLock -LockName myLock -ResourceGroupName myRG -ResourceName myResource -ResourceType Microsoft.Compute/virtualMachines -Force
```

---

## 2.15 ARM DEPLOYMENT EXECUTION — STEP BY STEP

### Complete Internal Walkthrough: Deploying a VM via Bicep/ARM Template

```
1. USER ACTION
   Engineer runs: az deployment group create --resource-group prod-rg --template main.bicep --parameters params.json

2. BICEP COMPILATION (if Bicep file)
   ARM compiles main.bicep → main.json (ARM JSON)
   - Variables resolved
   - Parameters bound
   - Resource references converted to resourceId() calls
   - Dependencies established

3. AUTHENTICATION
   ARM validates the user's/service principal's token against Entra ID tenant
   - Token valid? → Proceed
   - Token expired/invalid? → FAIL with AuthenticationFailed

4. AUTHORIZATION
   ARM checks RBAC:
   - Does identity have Microsoft.Resources/deployments/write?
   - Does identity have write permission on resources in the template?
   - At what scope? Resource Group prod-rg or higher?
   - Result:
     ✓ Allowed → Proceed
     ✗ Denied → FAIL with AuthorizationFailed

5. POLICY EVALUATION
   ARM evaluates all applicable policies:
   - Allowed locations: VM region in allowed list? ✓
   - Required tags: Template includes required tags? ✓
   - Allowed SKUs: VM size in allowed list? ✓
   - Results:
     ✓ All passed / Audited → Proceed
     ✗ Deny effect → FAIL with PolicyViolation

6. QUOTA CHECK
   ARM checks subscription quotas for region:
   - vCPU quota sufficient for VM size? ✓
   - Public IP quota available? ✓
   - Results:
     ✓ Pass → Proceed
     ✗ Fail → FAIL with QuotaExceeded

7. RESOURCE PROVIDER CHECK
   ARM verifies providers are registered:
   - Microsoft.Compute: Registered ✓
   - Microsoft.Network: Registered ✓
   - Microsoft.Storage: Registered ✓
   - Results:
     ✓ All registered → Proceed
     ✗ Not registered → FAIL with MissingSubscriptionRegistration

8. LOCK CHECK
   ARM checks locks on target resources:
   - Can the deployment write to the target scope?
   - Any CanNotDelete or ReadOnly locks on target resources?
   - Results:
     ✓ No blocking locks → Proceed
     ✗ Lock blocks → FAIL with LockedEntity

9. DEPENDENCY RESOLUTION
   ARM builds dependency graph:
   - VNet (no dependencies) → Deploy first
   - Subnet (depends on VNet) → Deploy second
   - NIC (depends on Subnet, Public IP) → Deploy third
   - VM (depends on NIC) → Deploy fourth
   - Extensions (depend on VM) → Deploy fifth
   - Parallel where possible (e.g., NSG can deploy independently)

10. DEPLOYMENT EXECUTION
    ARM locks each resource during creation:
    → Sends create requests to Resource Providers:
      - Microsoft.Network: Create VNet → Succeeded
      - Microsoft.Network: Create Subnet → Succeeded
      - Microsoft.Network: Create NIC → Succeeded
      - Microsoft.Compute: Create VM → Succeeded
      - Microsoft.Compute: Create Extension → Succeeded

11. STATE UPDATE
    ARM updates its internal database:
    - All resources now in "Succeeded" provisioning state
    - Resource properties updated (ID, state, outputs)

12. LOGGING
    ARM records in Activity Log:
    - Deployment started (correlation ID)
    - Each resource created (per resource)
    - Deployment completed (status, duration, correlation ID)

13. OUTPUT RETURN
    ARM returns deployment result to caller:
    - Deployment status: Succeeded
    - Resource IDs of all created resources
    - Outputs from template (if defined)
```

---

## 2.16 ADMINISTRATION

### Key Administrative Operations

| Operation | Tool | Scope |
|-----------|------|-------|
| Create/manage deployments | Portal, CLI, PowerShell, REST API | RG, Subscription, MG, Tenant |
| View deployment history | Portal (Deployments), CLI (`Get-AzDeployment`) | RG-level default |
| Cancel running deployment | Portal, CLI (`Stop-AzDeployment`) | Deployment scope |
| List deployment operations | Portal (Operations tab), CLI (`Get-AzDeploymentOperation`) | Per deployment |
| Export existing resources as template | Portal (Export Template), CLI | RG-level |
| Validate template | Portal, CLI (`What-If`), Bicep build | Template level |
| Manage template specs | Portal, CLI | RG, Subscription, MG |
| Configure deployment defaults | CLI (`az configure`) | User level |
| Manage deployment scripts | Portal, CLI | RG level |
| Cross-subscription deployment | CLI with `--subscription` flag | Multi-scope |

---

## 2.17 SECURITY CONSIDERATIONS — ARM-SPECIFIC

| Concern | Detail | Mitigation |
|---------|--------|------------|
| **Deployment identity theft** | Compromised deployment SP can create malicious resources | Least privilege, PIM (just-in-time), monitoring |
| **Template tampering** | Modified templates may include hidden resources | Git-based template management, code review |
| **Secret exposure** | Parameters/logs may contain secrets | securestring, Key Vault references, no plaintext secrets |
| **Scope creep** | Deployments at too-broad scopes increase blast radius | Deploy at RG level, not subscription, unless necessary |
| **Orphaned deployments** | Incomplete deployments leave partially created resources | Monitor deployments, automate cleanup |
| **API throttling** | Rapid deployments can hit ARM rate limits | Batch deployments, implement retry logic |
| **Cross-subscription attacks** | Compromised SP with cross-scope access | Strict scope boundaries, PIM |
| **Deployment replay** | Old deployment templates may be reused maliciously | Secure deployment history, access controls |

---

## 2.18 MONITORING

### What to Monitor for ARM Operations

| Metric/Log | What It Shows | Alert Trigger |
|------------|---------------|---------------|
| **Activity Log — Deployment** | All deployment operations with status | Failed deployments |
| **Activity Log — Write/Delete** | All resource modifications | Unexpected changes |
| **Deployment duration** | How long deployments take | Slow deployments (infrastructure issue) |
| **ARM API latency** | Time to process ARM requests | High latency (throttling, platform issue) |
| **Deployment frequency** | How often deployments occur | Unusual frequency (compromised SP?) |
| **Quota utilization** | vCPU, IP, storage usage | Approaching limits |
| **Resource provider registration state** | Whether providers are registered | Provider unregistered |
| **Resource provisioning state** | Succeeded/Failed/Running per resource | Resources in Failed state |

### Diagnostic Settings for ARM

ARM itself does not have diagnostic settings — but:
- **Activity Log** is enabled by default (cannot be disabled).
- **Resource-specific** diagnostic settings capture resource-level logs.
- **Azure Policy** compliance state changes can be monitored.
- **ARM API calls** can be logged via Activity Log or by routing Activity Log to Log Analytics/Storage/Event Hub.

---

## 2.19 PRODUCTION EXAMPLE

**Scenario: Enterprise CI/CD pipeline deploys infrastructure changes.**

```
Pipeline:
1. Developer pushes Bicep template to Git repository
2. CI pipeline compiles Bicep → ARM JSON
3. CI runs What-If deployment → reviews changes
4. If What-If shows no deletes and changes are expected:
   a. CD pipeline deploys to DEV (Incremental mode)
   b. Automated tests run against DEV resources
   c. Approval gate for PROD (manual or automated)
   d. CD pipeline deploys to PROD (Incremental mode)
   e. Post-deployment validation tests
5. Results logged to Log Analytics workspace
6. Slack/Teams notification of deployment status

Key ARM considerations in this flow:
- Deployment identity has Contributor on DEV and PROD resource groups (minimum scope)
- What-If runs before every deployment
- Deployment names are versioned (e.g., infra-v1.2.3)
- Correlation IDs from each deployment logged for support tracing
- Template specs used for standardized deployments
- PIM (Privileged Identity Management) used for elevated deployment permissions
- Deployment history retained for audit compliance
```

---

## 2.20 FAILURE SCENARIOS — ARM-SPECIFIC

| Scenario | Symptoms | Root Cause | Resolution | Evidence |
|----------|----------|------------|------------|----------|
| **Deployment stuck in Running** | Deployment never completes, stays in "Running" state for hours | Resource Provider timeout, hung operation | Cancel deployment, retry, check provider health | Activity Log shows "Accepted" but no "Succeeded" |
| **Partial deployment failure** | Deployment status = Failed, but some resources exist and work | Resource failure in middle of dependency chain | Identify first failed resource, fix, redeploy corrected template | Deployment Operations tab shows mix of Succeeded/Failed |
| **Rollback failure** | Failed deployment leaves orphaned resources | Incremental mode doesn't auto-rollback | Manually delete orphaned resources, redeploy fixed template | Portal shows resources created despite overall failure |
| **Concurrent deployment conflict** | Two deployments race, one fails with conflict errors | Both deployments try to modify same resource | Sequence deployments, use deployment locks or runbooks | Activity Log shows concurrent operations |
| **"Deployment failed due to quantity exceeded"** | All resources fail to deploy | vCPU or resource quota exhausted | Request quota increase, use different SKU, change region | Error message specifies quota type and current/max values |
| **Template validation error** | Deployment never starts, immediate failure | Syntax error, missing required property, schema violation | Check error details, use Bicep linter, validate JSON | CLI/Portal error shows specific line and property |
| **"Resource is locked"** | Specific resource cannot be deployed/updated | CanNotDelete or ReadOnly lock on resource | Remove lock, then redeploy | Lock listed on resource or RG |
| **Policy violation mid-deployment** | ARM blocks deployment before provider execution | Deny policy applies to resource type/location/size | Fix template to comply, or request policy exemption | Policy compliance dashboard shows violation |
| **Provider timeout** | Deployment hangs on specific provider | Provider backend overloaded, or resource creation genuinely slow | Wait, retry, or escalate to Microsoft support | Correlation ID helps Microsoft trace provider-side |
| **What-If shows unexpected deletes** | Complete mode or outdated template | Template doesn't include existing resources | Switch to Incremental mode, update template | What-If output lists Delete actions |

---

## 2.21 TROUBLESHOOTING METHODOLOGY — ARM ISSUES

### When a Deployment Fails

```
Step 1: Check deployment status
  Portal: Deployments → Find deployment → Status = Failed/Succeeded
  CLI: az deployment group show --name myDeploy --resource-group myRG

Step 2: Read error message
  → ErrorCode, ErrorMessage, TargetResource
  → Correlation ID (critical for Microsoft Support)

Step 3: Check deployment operations
  → Per-resource status
  → First failure is the root cause

Step 4: Categorize the failure
  → Control Plane (RBAC/Policy/Quota/Lock/Provider) vs Template error
  → If template error: fix template, redeploy
  → If control plane: fix identity/policy/quota/lock, redeploy

Step 5: Check Activity Log for correlation
  → Filter by Correlation ID
  → See all related operations and timestamps

Step 6: Check resource-specific issues
  → If the resource was created but is failing, check Resource Health
  → Check provider-specific logs

Step 7: Resolve
  → Fix root cause
  → What-If to validate
  → Redeploy

Step 8: Validate
  → Verify resource exists and is in Succeeded state
  → Check resource functionality (data plane)
  → Monitor for any post-deployment issues
```

---

## 2.22 LOGS / EVIDENCE

| Evidence Source | What It Shows | Access Method |
|----------------|---------------|---------------|
| **Deployment Operations** | Per-resource deployment status, timestamps, error messages | Portal: Deployments → Operations; CLI: `Get-AzDeploymentOperation` |
| **Activity Log** | All ARM operations including deployments, with Correlation ID | Portal: Activity Log; CLI: `Get-AzLog` |
| **Resource Provider logs** | Service-specific operational data | Diagnostic Settings → Log Analytics/Storage/Event Hub |
| **ARM API response** | Full error response with details | CLI output, REST API response |
| **What-If output** | Planned changes (without execution) | `az deployment group what-if` |
| **Correlation ID trace** | All operations across providers for a single request | Activity Log filtered by Correlation ID; Microsoft Support trace |

---

## 2.23 VERSION/CURRENT SERVICE CONSIDERATIONS (Sept 2026)

| Aspect | Current State | Impact |
|--------|--------------|--------|
| **ARM Templates** | Fully GA, still primary deployment mechanism | All existing templates work without changes |
| **Bicep** | GA, Microsoft's recommended authoring tool | New projects should use Bicep; existing JSON templates continue working |
| **What-If** | Fully supported for all deployment scopes | Always use before production deployments |
| **Complete mode** | Still supported but strongly discouraged for production | High risk of accidental resource deletion |
| **Template Specs** | GA | Use for versioned, reusable templates across enterprise |
| **Deployment Scripts** | GA | Run PowerShell/Bash during deployments for complex orchestration |
| **Multi-session transactions** | Supported | Concurrent deployment handling improved |
| **Template linting** | Bicep linter + ARM template linter | Use in CI/CD pipeline as pre-deployment validation |
| **Nested templates** | Still supported but Bicep modules preferred | Bicep modules provide better modularity |
| **Linked templates** | Legacy, Bicep modules are replacement | Migrate linked templates to Bicep modules |
| **API Versions** | Each resource type has regular updates | Monitor deprecations; pin to GA versions in production |
| **Terraform** | Widely used, provider maintained by HashiCorp/Microsoft | Not native ARM but uses ARM underneath; state management is key |

---

## 2.24 L3 INTERVIEW QUESTIONS

### Basic
**Q: What is the purpose of Azure Resource Manager?**
A: ARM is Azure's management layer. Every Azure operation (create, read, update, delete) passes through ARM first. ARM handles authentication, authorization, policy evaluation, deployment orchestration, and state management before routing requests to the appropriate resource provider.

### Intermediate
**Q: What happens when a deployment fails?**
A: ARM stops processing further resources, marks the failed resource as Failed, and returns a failure status with a Correlation ID. In Incremental mode (the default), resources already successfully created are NOT rolled back — they remain in place. The engineer must fix the issue and redeploy or manually clean up partial resources.

### L3
**Q: My deployment failed but most resources were created successfully. What do I do?**
A: First, identify the FIRST resource that failed using Deployment Operations (not just the error message — look at the operation timeline). The first failure is the root cause; subsequent failures are usually cascading. Fix that root cause, then redeploy the full template in Incremental mode — ARM will skip already-created resources and create/update any remaining ones.

### Senior L3
**Q: Explain the difference between Complete mode and Incremental mode, and when you'd use each.**
A: Incremental mode creates new resources and updates existing ones but leaves anything not in the template untouched. This is the safe default for production. Complete mode also DELETES resources that exist but aren't in the template — useful for infrastructure sync scenarios but dangerous in production where resources may be managed outside the template. I always use Incremental for production and What-If to preview changes before any deployment.

### Expert
**Q: What happens internally when ARM processes a deployment?**
A: ARM: (1) compiles Bicep→JSON if needed, (2) authenticates via Entra ID, (3) authorizes via RBAC, (4) evaluates all applicable policies (Deny blocks immediately), (5) checks quotas, (6) verifies resource providers are registered, (7) checks for blocking locks, (8) builds a dependency graph from dependsOn and implicit resource references, (9) topologically sorts the graph and deploys independent resources in parallel, (10) locks each resource during creation, (11) sends requests to resource providers, (12) waits for provider confirmation (Accepted→Running→Succeeded/Failed), (13) updates ARM database state, (14) logs everything to Activity Log with a Correlation ID, (15) returns results to the caller. If any resource fails, ARM stops processing and the deployment is marked Failed.

### Scenario
**Q: "A deployment succeeded, but when I check, only some resources exist. The deployment status shows Failed. What happened?"**
A: The deployment likely failed on a resource that was later in the dependency chain, but resources that were already successfully created before the failure remain in place — they were not rolled back because the deployment was in Incremental mode. Check Deployment Operations to see which resources succeeded and which failed. Fix the root cause and redeploy; the already-created resources won't be affected, and the failed ones will be created on the second attempt.

### Tricky
**Q: "If I have Contributor on the Resource Group, will my deployment always succeed?"**
A: No. Contributor allows write operations, but:
- Azure Policy can still deny the deployment (e.g., Deny on disallowed location).
- Quotas can block creation (e.g., vCPU limit reached).
- Resource locks (CanNotDelete/ReadOnly) can prevent modification.
- Resource providers might not be registered.
- The specific resource type might require additional permissions beyond Contributor.
Contributor is necessary but not sufficient for successful deployments.

---

## 2.25 SCENARIO-BASED QUESTIONS

### Scenario 1: "Deployment fails with AuthorizationFailed"
**Symptoms:** Deployment immediately fails, all resources report as not created.
**Architecture:** ARM deployment pipeline → RBAC evaluation.
**Dependencies:** Identity, role assignment, scope.
**Checks:**
- What identity is being used?
- What role is assigned to that identity?
- At what scope?
- Is the scope at least Resource Group level?
- Does the role include `Microsoft.Resources/deployments/write` and the specific resource types' write permissions?
**Evidence:** Activity Log shows AuthorizationFailed, ARM response includes "Authorization failed for action Microsoft.Resources/deployments/write."
**Root Cause:** Insufficient RBAC permissions.
**Fix:** Assign appropriate role (Contributor or custom with write permissions) at Resource Group scope or higher.
**Validation:** Redeploy What-If, then deployment.

### Scenario 2: "Deployment stuck, never completes"
**Symptoms:** Deployment status remains "Running" for hours.
**Architecture:** ARM deployment engine → Resource provider.
**Dependencies:** Provider health, resource-specific creation time.
**Checks:**
- Check provider status in Portal
- Check Activity Log for the Correlation ID
- Is one resource hanging?
- Is the provider backend overloaded?
**Evidence:** Deployment Operations shows one resource stuck at "Accepted" state for extended time.
**Root Cause:** Provider-side timeout or resource creation hang.
**Fix:** Cancel deployment, retry, or escalate to Microsoft support with Correlation ID.
**Validation:** Retry deployment, monitor completion.

### Scenario 3: "Complete mode deployment deleted production VMs"
**Symptoms:** VMs that existed before deployment are now gone.
**Architecture:** ARM Complete mode → Resource deletion.
**Dependencies:** Template contents, resource grouping.
**Checks:**
- Was deployment in Complete mode?
- Were the existing resources included in the template?
- Were resources in other RGs or with different names?
**Evidence:** Activity Log shows Delete operations on the VMs, correlated with deployment.
**Root Cause:** Complete mode deployment with outdated template.
**Fix:** Never use Complete mode in production. Recover from backup/site recovery. Update template, use Incremental mode.
**Validation:** Infrastructure matches desired state, applications running.

### Scenario 4: "What-If shows my VM size change will trigger a replacement"
**Symptoms:** What-If shows Modify with a replacement indicator (≈ symbol).
**Architecture:** VM properties → VM size change → requires VM deallocation/recreation.
**Dependencies:** VM current state, size compatibility.
**Checks:**
- Some VM size changes require deallocation (which means downtime).
- Check if the new size is available in the region/VM host.
- Check maintenance policy for the VM.
**Evidence:** What-If shows VM replacement (not just property modification).
**Root Cause:** VM size change requires resource recreation (platform behavior).
**Fix:** Schedule maintenance window, use update Domain-aware deployment, or change size in a way that doesn't require replacement if possible.
**Validation:** VM resized, application running, no unexpected downtime.

### Scenario 5: "Deploying the same template twice causes conflicts"
**Symptoms:** Second deployment fails with "AlreadyExists" or resource conflict errors.
**Architecture:** ARM idempotency → resource naming conflicts.
**Dependencies:** Resource names, deployment mode, uniqueness constraints.
**Checks:**
- Are resources using static names that already exist?
- Is the deployment using Incremental mode (should handle existing resources)?
- Are there duplicate resource names in the same region?
**Evidence:** Error "Resource already exists" or "Conflict" for specific resource.
**Root Cause:** Template tries to create resources that already exist without proper handling.
**Fix:** Ensure template is idempotent — use existing resource references or unique naming (e.g., `uniqueString()` for random suffixes).
**Validation:** Deployment completes, both resources coexist or are updated correctly.

---

## 2.26 KNOWLEDGE TEST

1. **What is the difference between ARM and a Resource Provider?**
   ARM is the management layer (control plane). Resource Providers are the services (e.g., Microsoft.Compute) that actually create and manage resources. ARM routes requests to providers.

2. **Why does a deployment sometimes fail but leave resources behind?**
   In Incremental mode (default), ARM does NOT automatically rollback already-created resources when a later resource fails. It stops processing and marks the deployment Failed.

3. **What is the difference between explicit and implicit dependencies?**
   Explicit: Using `dependsOn` in the template. Implicit: ARM detects from `resourceId()`/`reference()` calls automatically.

4. **What is the purpose of What-If?**
   To preview changes without executing them. Shows Create, Modify, Delete, Ignore actions for each resource.

5. **What does a Correlation ID help you do?**
   Trace all related operations across multiple resource providers for a single deployment request — essential for Microsoft Support escalation and Activity Log analysis.

6. **Can Bicep be deployed directly?**
   Not directly — it must be compiled to ARM JSON first. However, tools like `az deployment group create --template-file main.bicep` auto-compile it on the fly.

7. **What is the risk of Complete deployment mode?**
   It deletes resources that exist but are NOT in the template. This can accidentally delete production resources if the template is outdated.

8. **What role is needed to deploy resources?**
   At minimum: Write permission on the target resources (e.g., `Microsoft.Compute/virtualMachines/write`) and `Microsoft.Resources/deployments/write` at the deployment scope. Contributor role covers this.

9. **What happens if a policy has Deny effect and applies to your deployment?**
   ARM blocks the deployment BEFORE it reaches any resource provider. You get a PolicyViolation error. The deployment does not partially execute.

10. **What is the difference between ARM state and Resource Health?**
    ARM state = whether the resource was successfully created/updated (control plane). Resource Health = whether the underlying platform is healthy for that resource (data plane/physical). A resource can have ARM state "Succeeded" but Resource Health "IssuesDetected."

---

## 2.27 L3 GAP CHECK

| Topic | Status |
|-------|--------|
| ARM concept and role | ✅ Covered |
| ARM deployment engine workflow | ✅ Covered |
| Resource Providers deep dive | ✅ Covered |
| Resource types and IDs | ✅ Covered |
| ARM Templates structure | ✅ Covered |
| Parameters, Variables, Functions | ✅ Covered |
| ARM vs Bicep comparison | ✅ Covered |
| Bicep syntax and compilation | ✅ Covered |
| Deployment scopes (5 levels) | ✅ Covered |
| Deployment modes (Incremental/Complete/What-If) | ✅ Covered |
| What-If usage and analysis | ✅ Covered |
| Dependency handling (explicit/implicit) | ✅ Covered |
| Dependency resolution algorithm | ✅ Covered |
| Failed deployment behavior | ✅ Covered |
| Partial deployment handling | ✅ Covered |
| Rollback concepts | ✅ Covered |
| ARM template security | ✅ Covered |
| ARM + RBAC interaction | ✅ Covered |
| ARM + Policy interaction | ✅ Covered |
| ARM + Locks interaction | ✅ Covered |
| Control Plane vs Data Plane (ARM context) | ✅ Covered |
| ARM internal database | ✅ Covered |
| Correlation ID usage | ✅ Covered |
| API versions | ✅ Covered |
| Complete walkthrough of deployment flow | ✅ Covered |
| ARM monitoring and logging | ✅ Covered |
| ARM failure scenarios | ✅ Covered |
| ARM troubleshooting methodology | ✅ Covered |
| Current service considerations | ✅ Covered |

---

## 2.28 WHAT AN EXPERIENCED AZURE L3 ENGINEER SHOULD NOW BE ABLE TO EXPLAIN CONFIDENTLY

After Module 2, you should be able to confidently explain:

1. **The complete ARM deployment lifecycle** — from the moment a deployment request arrives at ARM through authentication, authorization, policy evaluation, dependency resolution, provider execution, state update, and logging.

2. **Why Control Plane and Data Plane failures require completely different troubleshooting approaches** — "I can't create a VM" (control plane: RBAC, Policy, Quota, Provider) vs "The VM is running but my app can't reach Storage" (data plane: NSG, DNS, Private Endpoint, network path).

3. **How ARM handles dependencies** — the difference between explicit `dependsOn` and implicit resource references, how ARM builds a dependency graph, and how it parallelizes independent operations.

4. **Why Complete mode is dangerous** — and why What-If should be mandatory before every production deployment.

5. **Why Bicep is preferred over raw ARM JSON** — cleaner syntax, type safety, direct resource references (no more `resourceId()` soup), and modules for modularity.

6. **How Azure Policy can override your Owner permissions** — RBAC allows you to attempt an action; Policy determines if the result is allowed. These are independent checks in ARM.

7. **What the Correlation ID is and how to use it for support escalation** — it ties ALL related operations across all providers into a single trace.

8. **Why a deployment can show "Failed" but some resources are actually created** — Incremental mode does not rollback successfully created resources.

9. **How ARM locks work during deployments** — preventing concurrent modification conflicts, and how locks can unexpectedly block your deployments.

10. **The difference between ARM state (Succeeded/Failed) and Resource Health (OK/IssuesDetected)** — and why both must be checked independently when troubleshooting.

11. **How to read a resource ID** and extract subscription, RG, provider, resource type, and name from it.

12. **Why resource providers need to be registered** and what happens when they aren't.

---

# Ready for Module 3 — Microsoft Entra ID (Tenant, Users, Groups, Applications)?

It covers:
- Tenant architecture (Tenant, Directory, Domains, Tenant ID, Object ID, Directory Roles, Azure RBAC)
- Users (Member, Guest, External identities, lifecycle, authentication methods, SSPR)
- Groups (Security, M365, assigned, dynamic, group-based licensing)
- Applications (App registrations, Enterprise apps, Service principals, Managed identities, Certificates, OAuth, OIDC, consent, API permissions)
- Deep explanation of **App Registration vs Enterprise Application vs Service Principal**
- And all 19 required module components

Say **"Next module"** to continue, or ask any questions about Module 2.