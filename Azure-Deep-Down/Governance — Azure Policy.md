# MODULE 8 — GOVERNANCE (Azure Policy, Initiatives, Blueprints, Tagging, Resource Locks, Management Group Governance) — L3 DEPTH

---

## 1. CONCEPT

**Azure Governance** is the set of tools, practices, and mechanisms that control, manage, and enforce how resources are created, configured, and operated in Azure.

If **RBAC** answers: "What can this user do?"
Then **Azure Policy** answers: "What resources and configurations are allowed — regardless of who the user is?"

> **The single most important governance concept:**
> RBAC controls WHO can do WHAT.
> Azure Policy controls WHAT resources and configurations are ALLOWED or DENIED.
> A user can have sufficient RBAC to create a resource, but Azure Policy can STILL prevent the deployment.

**Why Governance exists in enterprise Azure:**
- Prevent resources from being deployed in wrong regions
- Enforce naming standards across thousands of resources
- Ensure all resources have cost-tracking tags
- Block public IP exposure
- Enforce encryption requirements
- Control SKU sizes (prevent overspending)
- Ensure diagnostic logging is always enabled
- Enforce network security rules
- Maintain compliance with internal and external regulations

**Governance is the guardrail that prevents "it works but it shouldn't exist."**

```
WITHOUT GOVERNANCE:
  → Anyone can create any resource anywhere
  → No tags → No cost tracking
  → No naming standards → Chaos in 10,000 resources
  → Public IPs exposed → Security incidents
  → No diagnostics → No visibility
  → No compliance → Audit failures

WITH GOVERNANCE:
  → Resources must follow rules
  → Non-compliant resources are flagged or blocked
  → Costs are traceable
  → Security baselines are enforced
  → Compliance is auditable
  → Operations are predictable
```

---

## 2. ARCHITECTURE

```
┌──────────────────────────────────────────────────────────────────┐
│                        AZURE TENANT                              │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    MANAGEMENT GROUP(S)                      │  │
│  │  (Governance at scale — policies, RBAC, initiative apply)   │  │
│  │                                                             │  │
│  │  ┌──────────────────────────────────────────────────────┐  │  │
│  │  │              SUBSCRIPTION                             │  │  │
│  │  │                                                       │  │  │
│  │  │  ┌─────────────────────────────────────────────────┐│  │  │
│  │  │  │           RESOURCE GROUP                         ││  │  │
│  │  │  │                                                  ││  │  │
│  │  │  │  ┌──────────┐  ┌──────────┐  ┌───────────────┐││  │  │
│  │  │  │  │ Resource │  │ Resource │  │    Resource    │││  │  │
│  │  │  │  │   A      │  │   B      │  │      C         │││  │  │
│  │  │  │  │ (VM)     │  │(Storage) │  │    (App GW)    │││  │  │
│  │  │  │  └──────────┘  └──────────┘  └───────────────┘││  │  │
│  │  │  │                                                  ││  │  │
│  │  │  │  Governance applied at each level:               ││  │  │
│  │  │  │  ┌─ Policy Assignments ──┐                       ││  │  │
│  │  │  │  ┌─ RBAC Role Assignments│                       ││  │  │
│  │  │  │  ┌─ Resource Locks       │                       ││  │  │
│  │  │  │  ┌─ Tags (inherit down)  │                       ││  │  │
│  │  │  │  ┌─ Diagnostic Settings  │                       ││  │  │
│  │  │  │  └─ Blueprints (if applied)                  │││  │  │
│  │  │  └─────────────────────────────────────────────────┘││  │  │
│  │  └──────────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │           POLICY ENGINE (Azure Policy)                      │  │
│  │  - Evaluates every resource against assigned policies       │  │
│  │  - Compliance states: Compliant / Non-compliant / N/A       │  │
│  │  - Effects: Audit, Deny, Modify, Append, DeployIfNotExists │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │           BLUEPRINT ENGINE (if applicable)                  │  │
│  │  - Multi-resource, multi-resource-group, multi-subscription │  │
│  │  - Artifact catalog with versioning                         │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

**How governance components relate:**
```
RBAC → Controls WHO can do WHAT (identity-based)
Policy → Controls WHAT resources/configurations are allowed (resource-based)
Blueprints → Define a standardized environment (multi-resource foundation)
Tags → Organize, categorize, and track resources
Locks → Prevent accidental modification/deletion
Diagnostic Settings → Ensure observability
Management Groups → Apply governance at scale across subscriptions
```

---

## 3. COMPONENTS — DETAILED

### 3.1 AZURE POLICY — CORE

**Azure Policy** evaluates resources against rules (policy definitions) and either:
- **Allows** compliant resources
- **Flags non-compliant** resources (Audit mode)
- **Blocks** non-compliant resources (Deny mode)
- **Modifies** non-compliant resources to make them compliant (Modify mode)
- **Deploys** required resources if missing (DeployIfNotExists)

**Key concept — Policy evaluation happens at two times:**
1. **At creation/update** — When a resource is created or modified, Policy evaluates the change
2. **Continuous evaluation** — Background scan periodically re-evaluates all resources (not immediate — may take time)

---

### 3.2 POLICY DEFINITION

A **policy definition** is a rule that defines what condition to evaluate and what effect to apply.

**Structure:**
```json
{
  "properties": {
    "displayName": "Allowed locations",
    "policyType": "BuiltIn",
    "mode": "All",
    "description": "Only East US and West US are allowed.",
    "metadata": {
      "category": "Locations"
    },
    "parameters": {
      "allowedLocations": {
        "type": "StringArray",
        "metadata": {
          "displayName": "Allowed locations",
          "description": "List of allowed locations."
        },
        "allowedValues": [
          "eastus",
          "westus"
        ]
      }
    },
    "policyRule": {
      "if": {
        "not": {
          "field": "location",
          "in": "[parameters('allowedLocations')]"
        }
      },
      "then": {
        "effect": "Deny"
      }
    }
  }
}
```

**Key properties:**

| Property | Description |
|----------|-------------|
| **`displayName`** | Human-readable name |
| **`policyType`** | `BuiltIn` (Microsoft-provided), `Custom` (user-created), `Initiative` (set of policies) |
| **`mode`** | `All` (evaluates all resource types), `Indexed` (only evaluates resources with tags/properties), `Microsoft.KeyVault.Data` (for Key Vault data plane), `Microsoft.Azure.ResourceManager` (for resource management operations) |
| **`metadata.category`** | Category for organization (e.g., "Locations", "Networking", "Security") |
| **`parameters`** | Input parameters for the policy (allowed values, lists, etc.) |
| **`policyRule.if`** | Condition to evaluate (the "what to check") |
| **`policyRule.then`** | Action to take if condition is true (the "what to do") |

**Effects — What happens when a policy matches:**

| Effect | Description | Use Case |
|--------|-------------|----------|
| **Audit** | Resource is flagged as Non-compliant but NOT blocked. Deployment proceeds but is logged. | Initial rollout, monitoring, awareness |
| **Deny** | Resource creation/update is BLOCKED. Error returned to user. | Enforce hard rules (allowed locations, required tags) |
| **Modify** | Resource properties are modified to make compliant (e.g., add tags). Requires `Microsoft.Authorization/roleAssignments/write`. | Auto-tagging, auto-enabling diagnostics |
| **Append** | Adds missing values to resource properties (similar to Modify but for arrays). | Append required tags that are missing |
| **DeployIfNotExists** | Deploys a specified resource if the condition is met and the resource doesn't exist. | Deploy diagnostics settings if missing, deploy NSG if missing |
| **Disabled** | Policy is not evaluated. | Temporarily disabling a policy |
| **DisabledDelete** | Resource is non-compliant but deletion allowed (preview). | Allow deletion of non-compliant resources |
| **AuditIfNotExists** | Flags non-compliant if resource doesn't exist, but doesn't deploy. | Check existence without deploying |

> **L3 critical:** `Modify` and `DeployIfNotExists` effects require the policy assignment to have a **managed identity** with `Microsoft.Authorization/roleAssignments/write` permission. Without this, the policy will fail to enforce modifications/deployments.

---

### 3.3 POLICY INITIATIVE (Policy Set)

An **initiative** is a **collection of policy definitions** grouped together and assigned as a single unit.

**Why initiatives exist:**
Instead of assigning 50 individual policies, you create an initiative containing all 50 and assign the initiative once.

**Structure:**
```json
{
  "properties": {
    "displayName": "Security Baseline - Production",
    "description": "All security policies for production environments",
    "parameters": {
      "tagName": {
        "type": "String",
        "metadata": {
          "displayName": "Tag name",
          "description": "Name of the tag for cost tracking."
        },
        "defaultValue": "Project"
      }
    },
    "policyDefinitions": [
      {
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
        "parameters": {
          "tagName": {
            "value": "[parameters('tagName')]"
          }
        }
      },
      {
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/yyyyyyyy-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
        "parameters": {}
      }
    ]
  }
}
```

**Initiative properties:**

| Property | Description |
|----------|-------------|
| **`displayName`** | Name of the initiative |
| **`policyDefinitions`** | Array of policy definition references |
| **`parameters`** | Parameters that flow to individual policies |
| **`description`** | Human-readable description |
| **`metadata.category`** | Category for organization |

**Initiative vs Policy Definition:**

| Aspect | Policy Definition | Initiative |
|--------|-------------------|------------|
| **What it is** | A single rule | A collection of rules |
| **Assignment** | Can be assigned directly | Can be assigned directly (assigned as a group) |
| **Scope** | Same as any assignment | Same as any assignment |
| **Parameter override** | N/A | Parameters can override individual policy parameters at assignment time |
| **Versioning** | N/A | Initiatives can be versioned |
| **Reuse** | Can be in multiple initiatives | Can contain policies from different categories |

**Built-in initiatives (common):**
| Initiative | Description |
|------------|-------------|
| **Recommended policies** | Microsoft's recommended policy set for a clean setup |
| **Security Benchmark** | CIS Microsoft Azure security baseline |
| **Regulatory compliance initiatives** | HIPAA, ISO, SOC, PCI-DSS mapped to policies |
| **Azure Hygiene** | Operational best practices |

---

### 3.4 POLICY ASSIGNMENT

A **policy assignment** is the binding of an initiative/policy to a scope with (optionally) parameter overrides.

**How it works:**
```
Scope: Subscription or Resource Group (or Management Group)
Policy/Initiative: The rule(s) to evaluate
Parameters: Override defaults (e.g., which locations are allowed)
Enforcement mode: Default (evaluate) or Indexed (only tagged resources)
```

**Assignment scope rules:**

| Scope | What it governs |
|-------|-----------------|
| **Management Group** | All subscriptions and RGs beneath it |
| **Subscription** | All RGs and resources in the subscription |
| **Resource Group** | Only resources in that specific RG |

**Inheritance:**
```
Policy assigned at Management Group → propagates to ALL child RGs/subscriptions
Policy assigned at Subscription → propagates to ALL child RGs
Policy assigned at Resource Group → applies ONLY to that RG

More specific (lower) assignments can ADD restrictions but cannot REMOVE those from parent assignments.
A resource can be non-compliant due to MULTIPLE policy assignments (cumulative).
```

**Enforcement mode:**
| Mode | Description |
|------|-------------|
| **Default** | Evaluates all resources at scope (including untagged) |
| **Indexed** | Only evaluates resources that have at least one tag (reduces evaluation cost and scope) |

---

### 3.5 POLICY EXEMPTIONS

**Exemptions** allow specific resources or resource groups to be excluded from a policy assignment.

**Why they exist:**
Sometimes a resource needs to be non-compliant for valid reasons (migration, legacy, break-glass). Instead of disabling the entire policy, you exempt specific resources.

**Properties:**
| Property | Description |
|----------|-------------|
| **Scope** | The resource or RG being exempted |
| **Assignment** | Which policy assignment to exempt from |
| **ExpireOn** | Optional expiration date (best practice — exemptions should be temporary) |

**L3 guidance:**
```
Exemptions are a necessary tool but a sign of governance debt.
RULE: Every exemption should have an expiration date.
RULE: Review exemptions regularly — expired exemptions that are still in place indicate process failure.
RULE: Never create permanent exemptions "just in case."
```

---

### 3.6 REMEDIATION

**Remediation** takes action on resources that are non-compliant with a policy.

| Policy Effect | Remediation Action |
|--------------|-------------------|
| **Audit** | Mark compliant (no automatic fix) or deploy required resources |
| **Deny** | Cannot remediate (resource was blocked from being created) |
| **Modify** | Apply the modification to make compliant |
| **DeployIfNotExists** | Deploy the missing resource |

**Remediation tasks:**
- Deploy required resources (e.g., diagnostics settings) across multiple resources
- Modify existing resources (e.g., add tags, enable encryption)
- Batch remediation across thousands of resources
- Track remediation progress

---

## 4. RBAC vs AZURE POLICY — THE CRITICAL DISTINCTION

> **This is the #1 governance concept that causes L3 interview failures.**

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  RBAC:   "User X has Contributor on RG-Prod"                    │
│          → User CAN create resources in RG-Prod                 │
│                                                                 │
│  Policy: "Only East US is allowed in RG-Prod"                   │
│          → User CANNOT create resources in West US              │
│                                                                 │
│  RESULT: User X has Contributor (RBAC allows)                   │
│          BUT tries to create VM in West US (Policy denies)      │
│          → VM creation FAILS                                    │
│          → Error: "Policy execution failed: Policy 'Allowed     │
│            locations' denied action 'Microsoft.Compute/...'"   │
│                                                                 │
│  KEY INSIGHT: RBAC and Policy are INDEPENDENT systems.          │
│  Both must permit the action for it to succeed.                 │
│  RBAC allows AND Policy allows = SUCCESS                        │
│  RBAC allows AND Policy denies = DENIED                         │
│  RBAC denies AND Policy allows = DENIED                         │
│  RBAC denies AND Policy denies = DENIED                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Real production example:**

```
Engineer has "Contributor" on Production Subscription (RBAC).
Engineer tries to deploy a VM in "westus2" (West US 2).
Azure Policy "Allowed locations" is assigned at Subscription scope:
  → Allowed: "eastus", "westus" (East US, West US)
  → Effect: Deny
  → westus2 is NOT in the allowed list

Result: Deployment FAILS.
Error: "The resource 'Microsoft.Compute/virtualMachines/prod-vm01' 
        is not in an allowed location. Location 'westus2' is not 
        in the allowed list."

The Engineer has full RBAC permission (Contributor) but 
Azure Policy BLOCKED the deployment.
```

**Another example:**
```
Engineer has "Owner" on Resource Group (RBAC).
Engineer tries to create a Storage Account with "Standard_LRS" redundancy.
Azure Policy "Allowed SKUs" is assigned at RG scope:
  → Allowed: "Premium_LRS", "Standard_ZRS"
  → Effect: Deny
  → "Standard_LRS" is NOT in the allowed list

Result: Creation FAILS.
Even Owner RBAC does not override Policy.
```

**L3 troubleshooting checklist for "RBAC sufficient but deployment fails":**
```
1. Check Azure Policy assignments at scope (sub, RG, MG)
2. Check for Deny effect policies
3. Check resource-specific Policy evaluation (is the resource type in scope?)
4. Check Policy parameters (what values are allowed?)
5. Check Resource Locks (CanNotDelete/ReadOnly)
6. Check Deny Assignments (RBAC-level deny)
7. Check Resource Provider registration
8. Check quotas

ORDER: Policy is the #1 most common cause. Check it FIRST.
```

---

## 5. TAGGING — COMPLETE DEEP DIVE

**Tags** are key-value pairs attached to Azure resources for organization and tracking.

### 5.1 Why Tags Matter

| Purpose | Description |
|---------|-------------|
| **Cost allocation** | Track costs by department, project, environment |
| **Governance** | Enforce tagging policies (required tags) |
| **Organization** | Filter, group, and search resources |
| **Compliance** | Prove resources are categorized correctly |
| **Automation** | Script based on tag values |
| **Showback/Chargeback** | Allocate costs to business units |

### 5.2 Tag Properties

| Property | Details |
|----------|---------|
| **Format** | Key-value pair (e.g., `Environment: Production`, `Project: Contoso`) |
| **Max tags per resource** | 50 |
| **Max key length** | 512 characters |
| **Max value length** | 256 characters |
| **Case sensitivity** | Keys are NOT case-sensitive (but casing matters in display and some queries) |
| **Inheritance** | Tags on resource groups are inherited by resources inside (but resource tags can override) |
| **Scope** | Resources, Resource Groups, Subscriptions, Management Groups |

### 5.3 Tag Inheritance

```
Subscription: Environment=Production, CostCenter=12345
  ↓ (inherited)
  Resource Group: Project=App1, Owner=Team-A
    ↓ (inherited + own tags — resource's own tags OVERRIDE group's for same key)
    Resource: Environment=Production (inherited from sub),
               Project=App1 (inherited from RG),
               Owner=Team-B (overrides RG's Team-A)

Key rule: Resource's own tag for a key > Resource Group's tag > Subscription's tag
```

### 5.4 Required Tags (Policy)

**Azure Policy can enforce tagging:**

```json
{
  "policyRule": {
    "if": {
      "field": "tags['Environment']",
      "equals": "null"
    },
    "then": {
      "effect": "Deny",
      "details": {
        "message": "Environment tag is required on all resources."
      }
    }
  }
}
```

**Modify effect for auto-tagging:**
```json
{
  "policyRule": {
    "if": {
      "field": "tags['CostCenter']",
      "equals": "null"
    },
    "then": {
      "effect": "Modify",
      "details": {
        "roleAssignments": [
          {
            "roleDefinitionId": "/providers/Microsoft.Authorization/roleDefinitions/...",
            "principalId": "..."
          }
        ],
        "operations": [
          {
            "operation": "addOrReplace",
            "field": "tags['CostCenter']",
            "value": "[parameters('defaultCostCenter')]"
          }
        ]
      }
    }
  }
}
```

> **Modify effect requires:** A managed identity on the policy assignment with `Microsoft.Authorization/roleAssignments/write` and the appropriate RBAC role (e.g., Contributor) at the target scope.

### 5.5 Common Tag Standards

| Tag Key | Typical Values | Purpose |
|---------|---------------|---------|
| `Environment` | Production, Staging, Development, Test, UAT | Operational classification |
| `Project` | App1, App2, Contoso, Fabrikam | Application/project identification |
| `CostCenter` | 12345, 67890 | Financial tracking |
| `Owner` | Team-A, Team-B, John@contoso.com | Responsibility |
| `Department` | Engineering, Finance, HR | Organizational |
| `Criticality` | High, Medium, Low | Business impact |
| `DataClassification` | Public, Internal, Confidential, Restricted | Security/compliance |
| `CreatedBy` | automation, manual, migration | Provenance |
| `AutoShutdown` | true, false | Operational management |

---

## 6. RESOURCE LOCKS

**Resource Locks** prevent accidental modification or deletion of resources.

### 6.1 Lock Types

| Lock | Effect |
|------|--------|
| **CanNotDelete** | Resource CANNOT be deleted. Read and modify still allowed. |
| **ReadOnly** | Resource is completely read-only. Cannot be read either in some cases (actually: read is still allowed, but modify/delete is blocked). Wait — let me correct: ReadOnly prevents all write operations (modify, delete). Read operations are still allowed. |

> Correction from the specification:
> **CanNotDelete**: Prevents deletion. Modify still works.
> **ReadOnly**: Prevents modification AND deletion. Read still works.

### 6.2 Lock Scope and Inheritance

```
Lock applied at Subscription scope → ALL resources in subscription are locked
Lock applied at RG scope → ALL resources in that RG are locked
Lock applied at Resource scope → ONLY that resource is locked

Locks at parent scope DO propagate to child scopes.
More specific locks at child scope ADD restrictions but do NOT remove parent locks.

Multiple locks are cumulative.

Example:
  Subscription: CanNotDelete
  RG: ReadOnly
  
  Result: All resources are ReadOnly (which includes CanNotDelete)
  Any single lock would be sufficient to prevent deletion,
  but both combine for full protection.
```

### 6.3 Who Can Create/Remove Locks

| Action | Required Permission |
|--------|-------------------|
| Create lock | `Microsoft.Authorization/locks/write` (typically Owner, Contributor, or User Access Administrator) |
| Remove lock | `Microsoft.Authorization/locks/delete` |
| Read locks | `Microsoft.Authorization/locks/read` (built into most roles) |

> **L3 critical:** Locks do NOT inherit from RBAC. Even a resource owner CANNOT delete a resource with CanNotDelete lock unless the lock is removed first. This is a safeguard against accidental deletion and against compromised accounts.

### 6.4 Common Lock Scenarios

| Scenario | Lock Type | Scope |
|----------|-----------|-------|
| Production database | CanNotDelete | RG or resource |
| Core networking infrastructure | ReadOnly | RG or resource |
| Critical VIP application | CanNotDelete | Subscription (rare) |
| Resource during migration | CanNotDelete | Resource |
| Test/development environment | None | (intentionally unlocked) |

---

## 7. AZURE BLUEPRINTS

**Azure Blueprints** define a repeatable set of Azure resources that follow organizational standards, patterns, and requirements.

### 7.1 What Blueprints Do

A blueprint is a **package of artifacts** deployed together to create a standardized environment. Unlike a single template, a blueprint can include:
- Multiple resource groups
- Multiple resource types
- Role assignments
- Policy assignments
- ARM templates
- Existing resources

### 7.2 Blueprint Components

| Component | Description |
|-----------|-------------|
| **Blueprint definition** | The published blueprint (versioned) |
| **Artifact** | A component within the blueprint (ARM template, role assignment, policy, existing resource) |
| **Parameter** | Input values for artifacts |
| **Version** | Blueprints are versioned (V1, V2, etc.) |
| **Blueprint assignment** | An instance of a blueprint deployed to a subscription/RG |

### 7.3 Artifact Types

| Artifact Type | Description |
|--------------|-------------|
| **ARM template** | Deploy resources (VMs, Storage, Networking, etc.) |
| **Resource group** | Create a new resource group as part of the blueprint |
| **Existing resource** | Reference an already-existing resource |
| **Role assignment** | Assign RBAC roles as part of the blueprint |
| **Policy assignment** | Assign policies as part of the blueprint |
| **User-defined artifact** | External ARM template reference |

### 7.4 Blueprint vs ARM Template

| Aspect | ARM Template | Blueprint |
|--------|-------------|-----------|
| **Scope** | Single deployment (one or more resources in a scope) | Multi-resource, multi-RG, multi-subscription package |
| **Includes RBAC** | Can deploy role assignments within template | Dedicated artifact type for role assignments |
| **Includes Policy** | Can deploy policy assignments within template | Dedicated artifact type for policy assignments |
| **Versioning** | No built-in versioning | Versioned definitions |
| **Reusability** | Deployed repeatedly but no version tracking | Published with version history |
| **Parameterization** | Full parameter support | Parameters flow to all artifacts |
| **Use case** | Single deployment | Standardized environment setup |

### 7.5 Blueprint vs Policy

| Aspect | Policy | Blueprint |
|--------|--------|-----------|
| **Purpose** | Enforce rules (audit/deny/modify) | Create and configure resources |
| **Creates resources** | No (except DeployIfNotExists) | Yes (ARM templates) |
| **Governs resources** | Yes (evaluates existing resources) | No (creates new resources) |
| **Includes RBAC** | No | Yes |
| **Deployment** | Policy evaluation | Blueprint assignment |

---

## 8. MANAGEMENT GROUP GOVERNANCE

**Management Groups** are the primary mechanism for applying governance at scale across multiple subscriptions.

### 8.1 Governance at Management Group Scope

| Governance Mechanism | Scope Level |
|---------------------|-------------|
| **Policy assignment** | Management Group → applies to all child subscriptions and RGs |
| **RBAC role assignment** | Management Group → applies to all child subscriptions and RGs |
| **Initiative assignment** | Management Group → same as policy |
| **Blueprints** | Subscription or Management Group |
| **Diagnostic settings** | Resource, RG, Subscription, MG (for Activity Log) |
| **Resource Locks** | Subscription, RG, Resource (NOT at MG) |
| **Tags** | Subscription, MG (for inheritance) |

### 8.2 Typical Governance Architecture

```
Tenant Root
  │
  ├── MG-Corporate-Policy (Policy assignment: required tags, allowed locations)
  │   │
  │   ├── MG-Production
  │   │   │   (RBAC: Prod-Admins group, Security policies)
  │   │   │
  │   │   ├── Sub-Prod-West
  │   │   │   ├── RG-App1-Prod
  │   │   │   └── RG-App2-Prod
  │   │   │
  │   │   └── Sub-Prod-East
  │   │       └── RG-App3-Prod
  │   │
  │   └── MG-Shared-Services
  │       │   (Identity, Networking, Monitoring shared resources)
  │       ├── Sub-Shared-Identity
  │       ├── Sub-Shared-Networking
  │       └── Sub-Shared-Monitoring
  │
  └── MG-Development
      │   (Dev policies, relaxed guardrails)
      │
      ├── Sub-Dev-Project-A
      │   └── RG-Dev-Project-A
      │
      └── Sub-Dev-Project-B
          └── RG-Dev-Project-B
```

**L3 guidance:**
```
Best practice governance hierarchy:

1. Tenant Root → Baseline policies (required tags, allowed locations)
2. MG-Corporate-Policy → Security baselines (Deny public IP, require encryption)
3. MG-Production → Stricter policies (required diagnostics, approved SKUs)
4. MG-Development → Relaxed policies (audit-only for some rules)
5. Subscriptions → Team-specific RBAC
6. Resource Groups → Application-specific access

Policies cascade DOWN. RBAC cascades DOWN. Locks cascade DOWN.
More specific (child) scopes ADD restrictions.
```

---

## 9. DIAGNOSTIC SETTINGS (GOVERNANCE COMPONENT)

**Required diagnostic settings** ensure resources produce logs and metrics for monitoring, security, and compliance.

### 9.1 What Diagnostic Settings Do

```
Resource → Diagnostic Settings → Routes resource logs and platform metrics to:
  - Log Analytics workspace (most common)
  - Storage Account
  - Event Hub (for SIEM integration)
  - Another resource (via partner solutions)
```

### 9.2 Types of Data Captured

| Data Type | Description | Examples |
|-----------|-------------|---------|
| **Resource logs** | Service-specific logs from the resource | Storage access logs, NSG flow logs, App Service logs, VM boot diagnostics |
| **Platform metrics** | Azure platform metrics for the resource | CPU, memory, disk I/O, network throughput |
| **Activity logs** | Azure control plane operations (auto-captured at subscription level) | Resource creation, deletion, role assignments, policy changes |

### 9.3 Required Diagnostic Settings Policy

Azure Policy has built-in policies to enforce diagnostic settings:

| Policy | Description |
|--------|-------------|
| **Audit Diagnostic Settings for Azure Services** | Audit if diagnostic settings are missing |
| **Deploy diagnostic settings for Azure Services** | Deploy diagnostic settings if missing (DeployIfNotExists) |
| **Audit categories of diagnostic settings** | Audit specific log categories |
| **Deploy specific categories of diagnostic settings** | Deploy specific log categories if missing |

**L3 critical:**
```
"Resource is working but no logs are visible" → Diagnostic settings not configured.
This is the #1 cause of monitoring gaps in production.
```

---

## 10. SECURITY BASELINES

**Security baselines** are pre-built policy initiatives from Microsoft that map to security standards.

### 10.1 What Security Baselines Do

| Baseline | Standard |
|----------|----------|
| **Azure Security Benchmark** | CIS Microsoft Azure security baseline |
| **Regulatory compliance policies** | HIPAA, PCI-DSS, ISO 27001, SOC 2, NIST |
| **Defender for Cloud recommendations** | Maps Defender recommendations to policies |

### 10.2 Security Baseline Structure

```
Security Baseline (Initiative)
  ├── Control 1: Network security
  │   ├── Policy: NSG should block all inbound traffic by default
  │   └── Policy: Public IP should not be assigned to VMs
  ├── Control 2: Identity and access
  │   ├── Policy: MFA should be required for owners
  │   └── Policy: Legacy authentication should be blocked
  ├── Control 3: Data protection
  │   ├── Policy: Storage should use encrypted connections
  │   └── Policy: Key Vault should have purge protection
  └── Control 4: Monitoring
      ├── Policy: Diagnostic settings should be enabled
      └── Policy: Alerts should be configured
```

---

## 11. ADMINISTRATION — GOVERNANCE

### Key Administrative Operations

| Operation | Tool | Scope |
|-----------|------|-------|
| Create policy definition | Portal, CLI, REST API | Tenant |
| Create initiative | Portal, CLI, REST API | Tenant |
| Assign policy/initiative | Portal, CLI, REST API | Subscription, RG, MG |
| Create exemption | Portal, CLI | Per assignment |
| Create remediation | Portal, CLI | Per assignment |
| Create blueprint | Portal, CLI, REST API | Tenant/subscription |
| Assign blueprint | Portal, CLI | Subscription |
| Create resource lock | Portal, CLI, PowerShell | RG, subscription, resource |
| Configure diagnostic settings | Portal, CLI, REST API | Resource level |
| Configure required tags policy | Portal, CLI | Subscription/RG/MG |
| Review compliance | Portal → Policy | Per assignment |
| Create/modify tag at scope | Portal, CLI | Subscription, MG |

---

## 12. SECURITY CONSIDERATIONS

| Concern | Risk | Mitigation |
|---------|------|------------|
| **Policy not enforced** | Resources deployed non-compliant, no alerts | Audit mode → Deny mode for critical policies |
| **Permanent exemptions** | Resources always non-compliant without oversight | Time-bound exemptions, regular review |
| **Tags not applied** | No cost visibility, governance gaps | Required tags policy (Deny or Modify) |
| **Diagnostic settings missing** | No monitoring/auditing data | Deploy diagnostic settings policy |
| **Too many policies** | Over-restriction, developer frustration | Use initiatives for grouping, audit mode initially |
| **Policy conflicts** | Conflicting policies create confusing non-compliance | Review policy assignments, use exemptions |
| **Locks blocking operations** | Legitimate changes blocked by CanNotDelete/ReadOnly | Document locks, use break-glass procedures |
| **Blueprint version drift** | Old blueprint version deployed, missing latest standards | Version management, periodic blueprint redeployment |
| **Modify effect misconfiguration** | Policy changes resources unexpectedly | Test in audit mode first, review modifications |
| **Insufficient RBAC for Modify** | Policy with Modify effect fails silently | Grant appropriate role to policy's managed identity |
| **Resource Provider not registered** | Policy can't evaluate certain resource types | Register required providers |
| **Policy scope too broad** | Policies affect test/dev environments inappropriately | Use different policies per environment tier |

---

## 13. MONITORING — GOVERNANCE

### Key Monitoring Targets

| Metric/Log | What It Shows | Alert Trigger |
|------------|---------------|---------------|
| **Policy compliance** | Percentage of compliant/non-compliant resources per assignment | Non-compliance spike, critical policy non-compliance |
| **Non-compliant resources** | List of specific resources failing policy evaluation | New non-compliance on critical policies |
| **Policy remediation progress** | Percentage of remediated resources | Remediation stuck, failures |
| **Diagnostic settings coverage** | Resources without diagnostic settings | Any resource missing diagnostics |
| **Exemptions count** | Number and age of exemptions | Expired exemptions still active |
| **Resource locks** | Locks at various scopes | Locks in unexpected locations |
| **Tag coverage** | Resources without required tags | Resources missing required tags |
| **Activity log** | Policy assignment changes, policy definition modifications | Unauthorized policy changes |

---

## 14. PRODUCTION EXAMPLE

**Scenario: Enterprise governance for 500+ resources across 5 subscriptions.**

```
Tenant: contoso.onmicrosoft.com (P2)

Management Groups:
  MG-Production (Sub-Prod-West, Sub-Prod-East)
  MG-Development (Sub-Dev-A, Sub-Dev-B)

Governance at MG level:
  Policy Assignment: "Required tags" (Deny)
    Required: Environment, Project, CostCenter, Owner
    Scope: Tenant Root (all subscriptions)

  Policy Assignment: "Allowed locations" (Deny)
    Allowed: eastus, westus, centralus
    Scope: MG-Production (production only)

  Policy Assignment: "Security Baseline" (Audit)
    CIS benchmark mapped
    Scope: MG-Production

  Policy Assignment: "Deploy diagnostic settings" (DeployIfNotExists)
    Target: All resources in MG-Production
    Destination: Central Log Analytics workspace
    Scope: MG-Production

  Policy Assignment: "Allowed SKUs" (Deny)
    VMs: Only D-series, E-series allowed
    Storage: Only Premium_LRS, Standard_ZRS allowed
    Scope: MG-Production

RBAC at MG level:
  "Mgmt-Prod-Admins" group → Contributor on MG-Production subscriptions
  "Mgmt-Dev-Admins" group → Contributor on MG-Development subscriptions

Subscription level (Sub-Prod-West):
  Policy Assignment: "Criticality tagging" (Audit)
    High/Critical/Medium tags enforced for production resources

  Resource Lock: CanNotDelete on RG-Core-Infra
    Protects: Load balancers, gateway, DNS

Resource Group level (RG-App1-Prod):
  Policy Assignment: "NSG audit" (Audit)
    Checks: All VMs must have NSG

Blueprint: "Production Environment v3"
  Artifacts:
    - ARM template: VNets, Subnets, NSGs
    - Role assignment: Contributor on RG
    - Policy assignment: Required tags
    - Existing resource: Central Log Analytics

Tags at Subscription level:
  Environment=Production
  CostCenter=12345
  (Inherited by all RGs and resources)

Result:
  500+ resources across 5 subscriptions
  All resources tagged (Deny policy)
  All resources in allowed locations (Deny policy)
  All resources have diagnostics (DeployIfNotExists)
  Security baseline continuously audited
  Locks on critical infrastructure
  Blueprints ensure consistent environment creation
  All governance applied at MG level with subscription overrides
```

---

## 15. FAILURE SCENARIOS

| Scenario | Root Cause | Resolution | Evidence |
|----------|-----------|------------|----------|
| **VM deployment fails with "Policy execution failed"** | Azure Policy Deny effect blocks the deployment (wrong location, missing tag, disallowed SKU, etc.) | Check Activity Log for specific policy name; check Policy assignments at subscription/RG scope; check policy parameters | Activity Log: AuthorizationFailed with policy name; Policy: Compliance state; Error: specific policy message |
| **"I have Contributor but cannot deploy"** | RBAC is sufficient but Policy, Lock, or Quota is blocking | Follow chain: Policy → Lock → Quota → Resource Provider → Network | Activity Log: AuthorizationFailed details; Policy: compliance; Lock: resource locks; Quota: current usage |
| **Resources deployed but no diagnostic logs** | Diagnostic settings not configured on resources; Policy to deploy them may be in audit mode or not assigned | Check diagnostic settings on each resource type; ensure policy with DeployIfNotExists effect is assigned with proper managed identity | Diagnostic settings blade shows no settings; Log Analytics: no data for resource |
| **"Required tags" policy in Audit mode but resources are still untagged** | Policy is in Audit (not Deny) mode → untagged resources are flagged but not blocked; OR policy was recently assigned and propagation hasn't completed | If enforcement is desired: change effect to Deny or Modify; wait for propagation (15 min); re-evaluate compliance | Policy compliance dashboard: untagged resources flagged; Effect: Audit vs Deny |
| **Exempted resource still shows non-compliant** | Exemption not properly scoped; OR exemption expired but enforcement mode re-evaluated; OR policy has multiple conditions and resource fails on different condition | Check exemption scope matches the resource; check if resource fails on a different policy; check exemption status | Policy exemption blade: scope and assignment; Compliance: specific failing policy |
| **Remediation not completing** | Remediation task stuck due to Permission errors (Modify/DeployIfNotExists); OR target resources are locked; OR target resources don't match policy scope | Check remediation task details for errors; verify managed identity has correct RBAC; check for resource locks; check policy scope | Remediation: task status and error count; Activity Log: modification failures |
| **"Resource is working but compliance shows non-compliant"** | Continuous evaluation hasn't completed; OR policy was recently assigned; OR resource state changed (tag removed, config changed) | Wait for continuous evaluation cycle; manually trigger compliance re-evaluation; check when policy was assigned | Policy compliance timestamp; Resource properties: compare to policy requirements |
| **Blueprint deployment fails** | Artifact fails (ARM template error); RBAC permission for blueprint managed identity is insufficient; OR parameter values are incorrect | Check blueprint deployment operations; review each artifact individually; verify managed identity permissions | Blueprint: deployment status per artifact; Activity Log: per-artifact results |
| **"Modify" policy didn't add tags** | Policy managed identity lacks `Microsoft.Authorization/roleAssignments/write`; OR Modify effect not configured correctly; OR resource is read-only (Lock); OR target scope doesn't include resource | Check policy assignment's managed identity and RBAC; check policy rule syntax; check for locks; verify resource is in scope | Policy: assignment details, managed identity; RBAC: role assignments for MI; Activity Log: policy execution failures |
| **Tags inherited from RG but showing wrong on resource** | Resource has its own tag with same key (resource tag overrides); OR tag propagation delay | Check resource-level tags explicitly; check if resource has same key tag | Portal: Resource tags (not RG tags); API: GET resource tags |

---

## 16. TROUBLESHOOTING METHODOLOGY — GOVERNANCE

```
STEP 1: Identify the governance issue
  → What happened? (Deployment blocked? Compliance failure? Missing logs?)
  → What resource(s) are affected?
  → What scope (subscription, RG, resource)?

STEP 2: Check Azure Service Health
  → Is there a platform issue affecting Policy/Blueprint/Policy engine?

STEP 3: Check Policy Assignments
  → What policies are assigned at the target scope?
  → What is their effect? (Deny, Audit, Modify, DeployIfNotExists)
  → Are there higher-priority assignments from parent scopes?
  → Is the policy in Compliance mode or Disabled?

STEP 4: Check Compliance State
  → Is the specific resource non-compliant?
  → Which policy is it failing against?
  → What is the specific failure reason?

STEP 5: Check Parameters
  → Are the policy parameters configured correctly?
  → Are parameter values at assignment time overriding defaults?
  → Are allowed values correct for the region/resource type?

STEP 6: Check RBAC
  → Does the policy's managed identity have sufficient RBAC? (for Modify/DeployIfNotExists)
  → Does the user have sufficient RBAC for their intended action?
  → Is there a Deny Assignment?

STEP 7: Check Resource Locks
  → Is the resource or parent RG/subscription locked?
  → Lock type? (CanNotDelete, ReadOnly)

STEP 8: Check Diagnostic Settings
  → Are diagnostics configured?
  → Is the policy to deploy diagnostics assigned and working?
  → Is the destination (Log Analytics, Storage) healthy?

STEP 9: Check Exemptions
  → Is there an exemption covering the resource?
  → Is the exemption expired?
  → Is the exemption properly scoped?

STEP 10: Check Blueprint
  → Was a blueprint assigned to this subscription/RG?
  → Is the blueprint version current?
  → Are blueprint artifacts deploying correctly?

STEP 11: Fix and validate
  → Address root cause
  → Test deployment/configuration
  → Check compliance re-evaluation
  → Document
```

---

## 17. LOGS / EVIDENCE

| Evidence Source | What It Shows | Access Method |
|---------------|---------------|---------------|
| **Policy compliance dashboard** | Compliance state per assignment, per policy, per resource | Portal → Policy → Compliance |
| **Policy evaluation details** | Specific reason for non-compliance per resource | Portal → Policy → Compliance → Resource → Evaluation details |
| **Activity Log** | Policy assignment changes, policy evaluations, deployment authorization failures | Portal → Subscription/RG → Activity Log |
| **Remediation task status** | Progress, successes, failures per resource | Portal → Policy → Remediation |
| **Exemption list** | All active exemptions with scope and expiration | Portal → Policy → Exemptions |
| **Diagnostic settings** | Per-resource diagnostic configuration | Portal → Resource → Diagnostic settings |
| **Blueprint deployment status** | Per-artifact deployment status | Portal → Blueprints → Assignments → Deployment details |
| **Resource locks** | All locks at resource/RG/subscription level | Portal → Resource → Locks; or API |
| **Log Analytics workspace** | Captured diagnostic data, policy evaluation logs | Log Analytics → Tables |
| **Resource Graph** | Query resources across subscriptions with filters | Portal → Resource Graph; CLI: `az graph query` |

---

## 18. VERSION/CURRENT SERVICE CONSIDERATIONS

| Aspect | Current State (Sept 2026) | L3 Impact |
|--------|--------------------------|-----------|
| **Policy "Modify" effect** | GA. Requires managed identity with appropriate RBAC. | Powerful but must be configured carefully. Test in Audit mode first. |
| **Policy "DeployIfNotExists"** | GA. Deploys missing resources automatically. | Excellent for diagnostics, NSG, etc. Requires managed identity permissions. |
| **Policy "DisabledDelete"** | Preview. Allows deletion of non-compliant resources. | Useful for migration scenarios. |
| **Policy exemptions** | GA. Time-bound exemptions supported. | Exemptions are temporary by design — enforce expiration dates. |
| **Resource Graph** | GA. Query millions of resources across subscriptions. | Essential for enterprise-scale governance checks. |
| **Blueprints** | GA. Versioned, multi-artifact. | Use for standardized environment provisioning. |
| **Security baselines** | Updated regularly. Maps to CIS, NIST, etc. | Use Microsoft's baselines as starting point for security policies. |
| **Regulatory compliance initiatives** | Expanded to cover more standards. | Maps regulatory requirements to policies for audit evidence. |
| **Required tags policy** | GA. Deny, Audit, or Modify. | Critical for cost management. Use Modify to auto-tag if strict enforcement is premature. |
| **Diagnostic settings policy** | GA. DeployIfNotExists for diagnostics. | Should be deployed in every subscription — eliminates monitoring gaps. |
| **Cross-tenant policy** | Supported for multi-tenant governance. | Governs resources across trusted tenants. |
| **Policy mode "All" vs "Indexed"** | "Indexed" mode reduces evaluation scope to tagged resources. | Use Indexed for less critical policies to reduce evaluation cost. |
| **Azure Policy profiler (preview)** | Provides per-resource policy evaluation insights. | Helps debug why a resource is non-compliant. |
| **Resource MAP (cloud enforceability)** | Maps on-prem resources to Azure governance. | Useful for hybrid governance. |
| **Blueprints vs Policies** | Blueprints are preferred for creating standardized environments; Policies are preferred for enforcing existing resources. | Use Blueprints for new environment creation; Policies for ongoing governance. |

---

## 19. L3 INTERVIEW QUESTIONS

### Basic
**Q: What is Azure Policy?**
A: Azure Policy is a service that evaluates Azure resources against rules (policy definitions) and applies effects (Audit, Deny, Modify, DeployIfNotExists, etc.) based on whether resources are compliant. It enforces organizational standards for resource configurations, locations, SKUs, tagging, and more.

### Intermediate
**Q: What is the difference between RBAC and Azure Policy?**
A: RBAC controls WHO can do WHAT on Azure resources (identity-based authorization). Azure Policy controls WHAT resources and configurations are ALLOWED regardless of who the user is (resource-based governance). A user with Contributor RBAC can be blocked by a Policy Deny — they are completely independent systems. Both must permit an action for it to succeed.

### L3
**Q: "A user has Owner on the subscription and tries to create a VM in West US 2 but gets denied. What are all possible causes?"**
A:
1. **Azure Policy "Allowed locations"** — West US 2 is not in the allowed list (most common)
2. **Resource Lock** — CanNotDelete or ReadOnly lock on target RG or subscription
3. **Quota** — User has exceeded VM quota for West US 2 in the subscription
4. **Deny Assignment** — Explicit deny for the create VM action at subscription or resource scope
5. **Azure Policy "Allowed SKUs"** — The VM size is not in the approved list
6. **Resource Provider not registered** — Microsoft.Compute not registered
7. **Subscription benefit/Quota** — Specific subscription type has regional capacity limits
8. **Blueprint** — A blueprint assignment may have constrained the deployment

### Senior L3
**Q: Explain how a Policy with "Modify" effect actually works internally. What can go wrong?**
A: When a resource is created/updated that doesn't meet the policy condition (e.g., missing required tag):
1. Policy engine detects non-compliance
2. Policy's managed identity (assigned at the policy assignment level) attempts to modify the resource
3. The managed identity needs `Microsoft.Authorization/roleAssignments/write` to assign a role
4. The assigned role (e.g., Contributor) needs permission to modify the resource type
5. If all permissions are in place, the resource is modified (e.g., tag is added)
6. The resource is then marked compliant

What can go wrong:
1. Managed identity lacks `Microsoft.Authorization/roleAssignments/write` → modification fails silently (resource stays non-compliant)
2. Resource is read-only (Lock) → modification fails
3. Managed identity's RBAC is at wrong scope → cannot modify the resource
4. Policy is in Audit mode → no modification attempted
5. Target resource is in different scope than policy assignment → not evaluated

### Expert
**Q: What happens internally when a resource is created and Policy evaluates it?**
A: When a deployment request arrives at ARM:
1. ARM validates the request (RBAC check: does the caller have write permission at the scope?)
2. If RBAC passes, ARM queues the resource creation
3. Before creating the resource, ARM invokes the Policy engine
4. Policy engine evaluates ALL policies at the scope (subscription, RG, and inherited from parent scopes)
5. For each policy:
   a. Policy rule IF condition evaluated against resource properties
   b. If condition matches → THEN effect applied
   c. If effect is Deny → ARM blocks the creation, returns error
   d. If effect is Audit → creation proceeds, resource marked non-compliant for tracking
   e. If effect is Modify → creation proceeds, but policy's managed identity modifies the resource post-creation
   f. If effect is DeployIfNotExists → creation proceeds, but managed identity deploys required companion resources
6. If any policy returns Deny → entire creation fails
7. Resource is created (if no Deny) and Compliance state is set
8. Continuous evaluation runs later to catch drift

### Scenario
**Q: "A deployment was successful yesterday but now the same deployment fails with a Policy Deny error. What changed?"**
A: Possible causes:
1. **New Policy assignment** — A Deny policy was assigned at the subscription/RG/MG scope since yesterday
2. **Policy parameter changed** — Allowed values modified (e.g., region list shortened, SKU list changed)
3. **Resource moved** — Resource moved to a different RG that has stricter policies
4. **New Policy definition** — A new policy was added to an existing initiative
5. **Resource properties changed** — Resource now has different properties that trigger a previously non-applicable policy
6. **Scope change** — Policy assignment was added to a parent scope (MG/subscription)

Check: Activity Log for policy assignment changes; Policy compliance dashboard for recent changes; compare policy assignments from yesterday to today.

### Tricky
**Q: "I have a Policy in Audit mode that says 'Deploy diagnostic settings.' Why are diagnostics still not being deployed?"**
A: Audit mode means the policy only FLAGS non-compliance — it does NOT take action. To actually deploy diagnostic settings, you must change the effect from Audit to **DeployIfNotExists** (or "Deploy" if available). The policy in Audit mode correctly identifies which resources need diagnostics but does not remediate them. This is a common misunderstanding: Audit mode is for monitoring and awareness; DeployIfNotExists is for enforcement.

### Tricky 2
**Q: "A resource is tagged correctly, but Policy still says it's non-compliant for 'Required tags.' Why?"**
A: Possible reasons:
1. **The policy checks a different tag key** than what's on the resource (e.g., policy requires `Environment` but resource has `env` or `environment` — case sensitivity matters)
2. **The resource is in a different scope** than the policy assignment
3. **The policy is checking a parent resource** (e.g., policy on VMs checks a property, not a tag)
4. **Continuous evaluation hasn't completed** — compliance may be stale
5. **There are multiple policy assignments** — another assignment at a parent scope requires different tags
6. **The tag exists but value is empty or null** — policy may require a specific value, not just the key's existence

### Tricky 3
**Q: "If I have CanNotDelete lock on a resource, can an Owner still modify it?"**
A: Yes. CanNotDelete only prevents DELETION. An Owner can still:
- Read the resource
- Modify/update the resource
- Add/update tags (if no lock on tags specifically)
- Create new resources alongside it

The Owner CANNOT:
- Delete the resource
- Delete the resource group (if the RG has CanNotDelete)
- Delete any resource inside an RG with CanNotDelete

Common mistake: Assuming CanNotDelete makes the resource fully protected. It doesn't — only deletion is blocked. For full protection, you need ReadOnly lock.

---

## 20. SCENARIO-BASED QUESTIONS

### Scenario 1: "Production deployment blocked — policy denied"
**Architecture:** ARM deployment → Policy evaluation → Deny effect → Deployment failure.
**Dependencies:** Policy assignments, parameters, Resource Provider, RBAC, Locks.
**Checks:**
1. Activity Log: exact policy name and error message
2. Policy compliance: which policy blocked?
3. Policy parameters: what values are allowed?
4. Scope: which policy assignment triggered?
5. Resource properties: does the resource match the denied condition?
6. RBAC: does user have permission at scope?
7. Locks: any CanNotDelete/ReadOnly blocking?
**Root Cause: Resource is in a non-allowed location, SKU, or missing required tag per Policy.**
**Fix:** Change deployment parameters to comply, update policy, or request exemption.
**Validation:** Deployment succeeds; Compliance dashboard shows compliant.

### Scenario 2: "Resources not tagged after creation despite tagging policy"
**Architecture:** Resource creation → Policy evaluation → Modify/Append effect → Tag applied (or not).
**Dependencies:** Policy effect, managed identity RBAC, resource scope, Lock status.
**Checks:**
1. Is policy effect Modify/Append (not Audit)?
2. Does policy managed identity have RBAC to modify resources?
3. Is resource in policy scope?
4. Is there a Resource Lock preventing modification?
5. Are tags being applied correctly? (check key and value)
6. Continuous evaluation: has it run?
**Root Cause (common):** Policy in Audit mode (not Modify), or managed identity lacks RBAC.
**Fix:** Change policy effect to Modify/Append; grant proper RBAC to managed identity.
**Validation:** New resources are tagged automatically; compliance shows compliant.

### Scenario 3: "Diagnostic settings missing on 200 VMs"
**Architecture:** VMs created → Diagnostic settings not configured → Policy detects non-compliance → DeployIfNotExists remediates (if configured).
**Dependencies:** Diagnostic settings policy, managed identity, VM scope, Log Analytics workspace.
**Checks:**
1. Is there a policy with DeployIfNotExists for diagnostics?
2. Does policy managed identity have RBAC on VMs and Log Analytics?
3. Is the Log Analytics workspace accessible and provisioned?
4. Are VMs in the policy scope?
5. Is the policy in enabled (not Disabled) mode?
6. Has remediation been triggered?
**Root Cause: No policy assigned, policy in Audit mode, or managed identity lacks permissions.**
**Fix:** Deploy diagnostic settings policy with DeployIfNotExists effect; verify managed identity permissions; run remediation.
**Validation:** All VMs have diagnostic settings; logs flowing to Log Analytics.

### Scenario 4: "Developer team frustrated — all their deployments are being denied"
**Architecture:** Developer RBAC → Policy Deny → All deployments blocked.
**Dependencies:** RBAC, Policy, Scope, Environment.
**Checks:**
1. Which policies are in Deny mode at their scope?
2. Are developers inheriting policies from parent scopes (MG/Sub)?
3. Are policies intended for production also applied to development?
4. Can developers deploy in a different scope/RG?
5. Should policies be in Audit mode for dev environments?
**Root Cause: Production governance policies (Deny effect) applied to development subscriptions/RGs without adjustment. Best practice: Audit mode for dev, Deny for prod.**
**Fix:** Create environment-specific policy assignments: Audit for Dev, Deny for Prod. Or create separate MGs for dev and prod with different policy assignments.
**Validation:** Developers can deploy in Dev; Production remains governed.

### Scenario 5: "Compliance dropped from 95% to 40% overnight"
**Architecture:** Continuous evaluation → Policy re-evaluation → Compliance state change.
**Dependencies:** Policy assignments, resource changes, new policies, resource additions.
**Checks:**
1. Check Policy compliance dashboard timeline — what changed?
2. Activity Log — were new policies assigned or existing policies modified?
3. Resource Graph — were many resources created or modified?
4. Resource Graph — were tags removed from resources?
5. Check for new policy assignments at parent scopes
6. Check for new policy definitions added to existing initiatives
7. Were resources moved between scopes?
**Root Cause (common):** A new strict policy was assigned at subscription/MG level, or policy parameters were changed, causing mass non-compliance.
**Fix:** Review and adjust the new policy (stricter conditions), update parameters, or add exemptions for legacy resources with a timeline.
**Validation:** Compliance returns to acceptable level; no critical policies remain non-compliant.

### Scenario 6: "Blueprint deployed but resources are non-compliant with policy"
**Architecture:** Blueprint assignment → Artifact deployment → Policy evaluation → Non-compliant.
**Dependencies:** Blueprint artifacts, Policy assignments, Resource properties.
**Checks:**
1. Was the policy assigned BEFORE or AFTER the blueprint deployment?
2. Do blueprint artifacts include the required tags/configuration?
3. Are blueprint parameters matching policy requirements?
4. Is the policy in Deny or Audit mode? (Audit wouldn't block, just flag)
5. Continuous evaluation: has it run since blueprint deployment?
**Root Cause: Blueprint templates don't include the resources/properties required by policy (e.g., blueprint doesn't tag resources, but policy requires tags).**
**Fix:** Update blueprint artifacts to include required tags/configuration; run remediation on existing blueprint resources.
**Validation:** Blueprint resources are compliant; Compliance dashboard shows improved state.

### Scenario 7: "Resource moved between resource groups and now non-compliant"
**Architecture:** Resource move → Scope change → Different policy assignments apply → New non-compliance detected.
**Dependencies:** Policy assignments at new scope, continuous evaluation, resource properties.
**Checks:**
1. What policies are assigned at the new RG vs old RG?
2. Are the new policies stricter?
3. Does the resource meet all new policy requirements?
4. Continuous evaluation: has it re-evaluated?
5. Were exemptions in the old RG not transferred?
**Root Cause: New RG has stricter policy assignments; resource doesn't comply with new requirements.**
**Fix:** Update resource to comply; request exemption if justified; or move back if policy mismatch is unintended.
**Validation:** Resource shows compliant in new RG.

---

## 21. KNOWLEDGE TEST

1. **What is the difference between RBAC and Azure Policy?**
   RBAC controls WHO can do WHAT on resources (identity-based authorization). Policy controls WHAT resources/configurations are allowed (resource-based governance). Both must permit an action — they are independent systems.

2. **What are the Azure Policy effects?**
   Audit (flag but allow), Deny (block), Modify (change resource to comply), Append (add missing values), DeployIfNotExists (deploy companion resource), Disabled (no evaluation), DisabledDelete.

3. **What is an initiative?**
   A collection of policy definitions grouped and assigned as a single unit. Allows managing multiple policies together.

4. **What is a policy assignment?**
   The binding of a policy/initiative to a specific scope with parameter overrides.

5. **What scope levels can policies be assigned to?**
   Management Group, Subscription, Resource Group. (NOT individual resource level.)

6. **Can a user with Owner RBAC bypass a Policy Deny?**
   No. Azure Policy Deny overrides RBAC Owner/Contributor permissions. Policy and RBAC are independent — both must permit.

7. **What is a resource lock? What types exist?**
   A lock prevents accidental modification or deletion. Types: CanNotDelete (prevents deletion only), ReadOnly (prevents all writes).

8. **What is the difference between CanNotDelete and ReadOnly locks?**
   CanNotDelete prevents resource deletion but allows modification. ReadOnly prevents all write operations (modify, delete) but allows reads.

9. **What is ABAC and how does it relate to RBAC?**
   Attribute-Based Access Control adds conditions to RBAC role assignments, filtering permissions by resource attributes, user attributes, and environment. ABAC extends RBAC but does not replace it.

10. **What is a Blueprint?**
    A versioned package of artifacts (ARM templates, role assignments, policies, existing resources) deployed together to create a standardized environment across resource groups and subscriptions.

11. **What is the difference between a Blueprint and an ARM Template?**
    ARM templates deploy resources in a single deployment. Blueprints are multi-artifact, multi-resource-group, versioned packages that can include RBAC and Policy assignments alongside resource deployment.

12. **Why is a "Modify" policy effect powerful but dangerous?**
    It automatically changes resources to make them compliant, which is convenient at scale. But it requires a managed identity with RBAC permissions, and changes may be unexpected if the modification logic is incorrect. Always test in Audit mode first.

13. **What is a policy exemption? When should it be used?**
    An exclusion of a specific resource or RG from a policy assignment. Used for valid non-compliance (migration, legacy). Should always have an expiration date and be reviewed regularly.

14. **What is continuous evaluation in Azure Policy?**
    Background scanning that periodically re-evaluates all resources against policy assignments. Not immediate — there can be a delay between resource creation and compliance evaluation.

15. **What are Security Baselines?**
    Pre-built policy initiatives from Microsoft mapping to security standards (CIS, NIST, HIPAA, etc.). Provide a starting point for security governance.

16. **What is the difference between Policy and Blueprints?**
    Policy governs existing resources (audit, deny, modify). Blueprints create new standardized environments (deploy resources, assign RBAC, apply policies as a package).

17. **What happens when a Policy in Deny mode is assigned to a scope where the user has Owner RBAC?**
    The user CANNOT create/update resources that violate the policy, even though they have Owner. Policy Deny overrides RBAC.

18. **How do tags inherit across scope levels?**
    Tags flow DOWN: Subscription tags → inherited by RG → inherited by Resource. Resource's own tag for the same key OVERRIDES the inherited value.

19. **Why might a Policy in Audit mode not be enough for compliance?**
    Audit only flags non-compliance — it doesn't block or fix. Resources can still be created non-compliant. For strict enforcement, use Deny; for auto-remediation, use Modify/DeployIfNotExists.

20. **What permissions does a "Modify" policy effect require?**
    The policy assignment's managed identity needs `Microsoft.Authorization/roleAssignments/write` and an appropriate RBAC role (e.g., Contributor) at the target scope.

21. **What is the Resource Graph and why is it important?**
    Azure Resource Graph enables querying millions of resources across subscriptions using KQL-like syntax. Essential for enterprise governance: finding non-compliant resources, tracking tag coverage, identifying idle resources.

22. **What happens if a resource is created outside a policy's scope?**
    The policy is NOT evaluated for that resource. Policy scope determines which resources are evaluated. Resources outside scope can be non-compliant but won't be flagged.

23. **Can a Resource Lock override RBAC?**
    Yes. Even Owners cannot delete a resource with a CanNotDelete lock. Locks operate independently of RBAC.

24. **What's the difference between Policy "Audit" and Policy "AuditIfNotExists"?**
    Audit evaluates existing resources against conditions. AuditIfNotExists specifically flags when a related resource DOESN'T EXIST (e.g., no NSG on a VM). DeployIfNotExists acts on the absence.

25. **What happens to existing non-compliant resources when a new Deny policy is assigned?**
    Existing resources remain until they are modified or deleted (they become "Non-compliant" but are grandfathered). Any CREATE/UPDATE operation that violates the policy will be blocked.

---

## 22. L3 GAP CHECK

| Topic | Status |
|-------|--------|
| Azure Policy core concepts (definition, assignment, effect) | ✅ Covered |
| Policy effects (Audit, Deny, Modify, Append, DeployIfNotExists, Disabled) | ✅ Covered |
| Policy vs RBAC distinction (critical) | ✅ Covered (Critical) |
| Policy initiatives / policy sets | ✅ Covered |
| Policy parameters and inheritance | ✅ Covered |
| Policy scope (MG, Sub, RG) and evaluation | ✅ Covered |
| Policy exemptions (with expiration) | ✅ Covered |
| Remediation | ✅ Covered |
| Continuous evaluation | ✅ Covered |
| Resource locks (CanNotDelete, ReadOnly, inheritance) | ✅ Covered |
| Tagging strategies and tag inheritance | ✅ Covered |
| Required tags policy (Deny/Modify) | ✅ Covered |
| Required diagnostic settings policy | ✅ Covered |
| Security baselines | ✅ Covered |
| Blueprints (definition, artifacts, versioning) | ✅ Covered |
| Blueprints vs ARM Templates vs Policy | ✅ Covered |
| Management Group governance | ✅ Covered |
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

After Module 8, you should be able to confidently explain:

1. **The fundamental difference between RBAC and Azure Policy** — RBAC is identity-based (WHO can do WHAT). Policy is resource-based (WHAT is allowed). Both are independent authorization systems that must BOTH permit an action. An Owner with full RBAC CANNOT create a resource that violates a Deny policy. This single concept prevents the most common governance misunderstanding in Azure.

2. **How the four governance mechanisms work together** — RBAC controls user actions, Policy controls resource configurations, Blueprints create standardized environments, Tags organize and track resources. In a well-governed environment, all four are configured: RBAC gives the right people access, Policy enforces standards, Blueprints ensures consistent creation, and Tags enable cost tracking.

3. **Every Azure Policy effect and when to use it** — Audit for awareness and initial rollout, Deny for hard enforcement (allowed locations, required tags, approved SKUs), Modify for auto-correcting (add missing tags, enable diagnostics), DeployIfNotExists for deploying companion resources (NSG, diagnostics config). Each has specific use cases and governance implications.

4. **How Policy assignments cascade through the scope hierarchy** — Policies at Management Group apply to all child subscriptions and RGs. Policies at Subscription apply to all RGs. Policies at RG apply only to that RG. More specific (child) assignments ADD restrictions. Non-compliance can be caused by multiple assignments at different levels.

5. **Why Tags matter beyond organization** — Tags enable cost allocation (showback/chargeback), policy enforcement (required tags), resource filtering (Resource Graph queries), and operational automation (auto-shutdown based on tags). Tag governance (required tags policy) is essential for cost management.

6. **How Resource Locks work independently of RBAC** — Locks override RBAC. Even Owners cannot delete resources with CanNotDelete locks. Locks are a safeguard against accidental operations and malicious account compromise. They cascade down scope (sub → RG → resource).

7. **The difference between Blueprints and ARM Templates** — ARM templates are deployment artifacts for a single operation. Blueprints are versioned, multi-artifact environment definitions that include RBAC assignments, Policy assignments, and ARM templates — designed for repeatable, governed environment creation.

8. **The Modify effect power and danger** — Modify can automatically fix thousands of non-compliant resources, but requires a managed identity with proper RBAC, and changes resources without user interaction. Always test in Audit mode first and verify managed identity permissions.

9. **The role of Management Groups in governance** — Management Groups are the primary mechanism for applying governance at scale. Policy, RBAC, and Blueprints can all be assigned at MG scope, propagating to all child subscriptions. They enable tiered governance (corporate baseline at root, production stricter, development relaxed).

10. **How to troubleshoot a blocked deployment** — The chain: RBAC check → Policy evaluation (Deny blocks here) → Resource Lock check → Quota check → Resource Provider registration → Networking. Policy is the #1 cause of "I have permission but deployment fails" incidents.

11. **The critical importance of Diagnostic Settings policy** — Without it, resources work but produce no logs for monitoring, security, and compliance. The DeployIfNotExists effect for diagnostics should be deployed in every production subscription. This eliminates the "resource is working but no logs are visible" scenario.

12. **How Policy Exemptions work and why they must be temporary** — Exemptions are necessary but represent governance debt. Every exemption should have an expiration date and be regularly reviewed. Permanent exemptions indicate a governance process failure.

---

# Ready for Module 9 — Landing Zones (Platform, Application, Management, Identity, Connectivity, Security, Governance)?

It covers:
- Landing Zone architecture and rationale
- Platform landing zone (hub/spoke, VPN, ExpressRoute, Firewall, DNS)
- Application landing zone (workload VMs, apps)
- Management landing zone (monitoring, management services)
- Identity landing zone (Entra ID, Directory roles)
- Connectivity land zone (Virtual WAN, ExpressRoute, VPN)
- Security landing zone (Defender, NSG, Firewall)
- Governance landing zone (Policy, RBAC, Blueprints)
- Subscription structure and Management Group hierarchy
- Centralized logging and monitoring
- Shared services
- And all required module components

Say **"Next module"** to continue.