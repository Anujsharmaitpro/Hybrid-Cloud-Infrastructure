# MODULE 3 — MICROSOFT ENTRA ID — TENANT, USERS, GROUPS, APPLICATIONS

---

## 3.1 CONCEPT

Microsoft Entra ID (formerly Azure Active Directory) is Azure's **identity and access management platform**. It is the **single most critical component** in any Azure environment because:

1. **Every authentication** flows through it — every user, service principal, managed identity, and workload identity.
2. **Every authorization decision** starts here — RBAC role assignments, Conditional Access policies, and API permissions are all evaluated based on Entra ID constructs.
3. **Every API call** to Azure (and Microsoft 365, and thousands of SaaS apps) begins with a token issued by Entra ID.
4. **Every resource** in Azure has an identity relationship with the Entra tenant — whether it's a user accessing it, a service principal managing it, or a managed identity authenticating to it.

If Entra ID goes down, nothing in Azure works. Even if all your VMs are running perfectly, no one can sign in, no service principals can authenticate, and no managed identities can get tokens.

**Three concepts that must never be confused:**
- **Tenant** = The organizational identity boundary (a GUID, a trust relationship)
- **Directory** = The data store holding users, groups, apps, and devices (historically called "Active Directory" in cloud context)
- **Domain** = The DNS name used for sign-in (e.g., `contoso.com`)

These are related but distinct. A tenant has one or more directories, which have one or more domains.

---

## 3.2 ARCHITECTURE

```
┌───────────────────────────────────────────────────────────────┐
│                    MICROSOFT ENTRA TENANT                     │
│                    (Identity Boundary)                         │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              DIRECTORY (Microsoft Entra ID)              │  │
│  │                                                           │  │
│  │  ┌──────────┐  ┌──────────┐  ┌────────────────────────┐ │  │
│  │  │  USERS   │  │  GROUPS  │  │   APPLICATIONS         │ │  │
│  │  │          │  │          │  │   (App Registrations   │ │  │
│  │  │ Member   │  │ Security │  │    + Enterprise Apps)  │ │  │
│  │  │ Guest    │  │ M365     │  │                        │ │  │
│  │  │ Service  │  │ Dynamic  │  │ App Registrations      │ │  │
│  │  │ Principals│ │ Assigned │  │ Enterprise Apps        │ │  │
│  │  │ Managed    │  │ Dynamic  │  │   → Service Principals│ │  │
│  │  │ Identities│  │          │  │   → Managed Identities│ │  │
│  │  └──────────┘  └──────────┘  └────────────────────────┘ │  │
│  │                                                           │  │
│  │  ┌──────────────┐  ┌───────────────┐  ┌──────────────┐  │  │
│  │  │  DOMAINS     │  │ TENANT ID     │  │ TENANT SETTINGS│ │  │
│  │  │ (contoso.com │  │ (GUID)        │  │ - Auth Methods│ │  │
│  │  │  contoso.onm │  │               │  │ - CA Policies │ │  │
│  │  │  .com)       │  │               │  │ - SSPR        │ │  │
│  │  └──────────────┘  └───────────────┘  └──────────────┘  │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              DIRECTORY ROLES                             │  │
│  │  Global Admin | Privileged Role Admin | Security Admin  │  │
│  │  User Admin | Application Admin | Cloud App Admin       │  │
│  │  (and more — see Section 3.3.5)                         │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              AZURE RBAC (Directory-scoped)               │  │
│  │  Directory Roles ≠ Azure RBAC — see Section 3.8        │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

---

## 3.3 COMPONENTS — DETAILED

### 3.3.1 TENANT

**What it is:**
A Microsoft Entra tenant is a **trust relationship** between your organization and Microsoft. It represents your organization's dedicated instance of Microsoft Entra ID. It is the absolute root of the identity hierarchy.

**Key properties:**
| Property | Description |
|----------|-------------|
| **Tenant ID** | Globally unique GUID (e.g., `abcdef12-3456-7890-abcd-ef1234567890`). This is the primary identifier for the tenant in all API calls, tokens, and configurations. |
| **Default domain** | `{tenant-name}.onmicrosoft.com` — created automatically when the tenant is provisioned. Cannot be deleted. |
| **Tenant name** | Display name shown in Portal (e.g., "Contoso Ltd"). |
| **Tenant type** | Single-tenant (default) or multi-tenant (for SaaS apps that accept external users). |
| **Microsoft Entra ID SKU** | Free, Standard (P1), Premium (P2) — determines feature availability (Conditional Access, PIM, Protection, etc.). |

**What a tenant depends on:**
- Microsoft's global identity infrastructure
- DNS (for custom domain verification)
- Azure AD B2C? No — B2C is a separate tenant type.

**What depends on the tenant:**
- Every Azure subscription (each subscription is associated with exactly one Entra tenant)
- Every Microsoft 365 service
- Every SaaS application that uses Entra ID for authentication
- Every Azure resource (for identity and RBAC)

**Critical L3 concept: One tenant, multiple directories?**
In practice, a tenant has **one** Microsoft Entra ID directory. The terms "tenant" and "directory" are often used interchangeably, but historically there was a distinction:
- A tenant = the trust boundary
- A directory = the data store

Today in the Azure portal, you see "Microsoft Entra ID" as the directory name, and it lives within a tenant. They are tightly coupled but technically distinct concepts.

**Multi-tenant vs single-tenant:**
- **Single-tenant:** Your tenant only contains your organization's users, groups, and apps. (Default.)
- **Multi-tenant:** Your tenant can be "consumed" by other organizations' apps (e.g., a SaaS app in your tenant that external users can sign into). Other organizations' users can be granted guest access in your tenant.

---

### 3.3.2 DIRECTORY

**What it is:**
The Microsoft Entra ID directory is the **data store** within the tenant that holds all identity objects: users, groups, service principals, managed identities, devices, and application registrations.

**Key properties:**
- Contains all identity objects
- Has a default domain
- Supports multiple domains (verified)
- Supports multiple SKU tiers (Free, P1, P2)
- Directory-specific features: Password Sync, Password Writeback, SSPR, PTA, etc.

**Directory vs Tenant — L3 clarity:**
```
Tenant = The trust relationship (the "contract" with Microsoft)
Directory = The identity database (where objects live)

In most cases, one tenant = one directory.
But conceptually, the tenant is the boundary, the directory is the data.

When Microsoft documentation says "tenant ID," it usually means the Directory ID (which is the same GUID).
In the portal, "Microsoft Entra ID" is the directory, living inside a tenant.
```

**What the directory contains:**
| Object Type | Description |
|-------------|-------------|
| Users | Member users, guest users, service principals |
| Groups | Security groups, M365 groups |
| Applications | App registrations, enterprise applications |
| Devices | Hybrid joined, Entra registered devices |
| Organizational Units (AOUs) | For scheduling |
| Policies | Conditional Access, authentication methods, etc. |

---

### 3.3.3 DOMAINS

**What they are:**
Domains are DNS names used for user sign-in and resource naming. Each domain maps to your DNS and is verified by Microsoft (you prove you own the domain by adding a TXT/MX record).

**Types:**
| Domain Type | Example | Notes |
|-------------|---------|-------|
| **Default domain** | `contoso.onmicrosoft.com` | Created automatically. Cannot be deleted. |
| **Custom domain** | `contoso.com` | Must be verified via DNS. Replaces default for user sign-in and UPN. |
| **Federation domain** | `contoso.com` (federated with AD FS or Entra Connect) | Used for password writeback, pass-through auth, or federation. |

**Verification process:**
1. You add a DNS TXT or MX record proving ownership.
2. Microsoft checks the record.
3. Domain status = "Verified."
4. Users can now sign in using `user@contoso.com` instead of `user@contoso.onmicrosoft.com`.

**UPN (User Principal Name):**
- Each user has a UPN (e.g., `john@contoso.com`).
- The UPN must match a verified domain in the tenant.
- The UPN is also the primary sign-in identifier (in most configurations).
- The default UPN suffix is `{tenant}.onmicrosoft.com`.
- You can change the UPN suffix to a verified custom domain.

**L3 operational impact:**
- If a user's UPN doesn't match a verified domain → they cannot sign in.
- Custom domain change is a significant operation — it affects all users' UPNs and service principal names.
- Federation domains are critical for hybrid environments (Entra Connect).

---

### 3.3.4 TENANT ID vs OBJECT ID

| Identifier | What It Identifies | Format | Immutable? |
|------------|-------------------|--------|-----------|
| **Tenant ID** (a.k.a. Directory ID) | The tenant/directory itself | GUID | Yes — never changes |
| **Object ID** | A specific object within the directory (user, group, app, service principal, device, etc.) | GUID | Yes — assigned at creation, never changes |

**Why both?**
- Tenant ID tells the system **which tenant** to look in.
- Object ID tells the system **which specific object** within that tenant.

**Examples in practice:**
- **Token claims:** An access token contains `tid` (tenant ID) and `oid` (object ID of the user or service principal).
- **RBAC role assignments:** Stored as `<object ID of principal>` assigned to `<scope>` with `<role definition ID>`.
- **Audit logs:** All entries include the tenant ID and the object ID of the affected entity.

**L3 troubleshooting:**
When Activity Log or Entra ID logs reference an object, they show Object IDs. You use Object IDs to:
- Identify a specific user, group, or service principal
- Cross-reference RBAC role assignments
- Find the principal in API calls

---

### 3.3.5 DIRECTORY ROLES vs AZURE RBAC ROLES — CRITICAL DISTINCTION

This is one of the most commonly confused topics. **Directory Roles are NOT the same as Azure RBAC roles.**

**Directory Roles (Entra ID roles):**
These are **tenant-level roles** that control who can administer Entra ID itself. They apply to the ENTIRE tenant.

| Role | Permissions |
|------|------------|
| **Global Administrator** | Full admin access to Entra ID, Azure subscriptions, and Microsoft 365. (Legacy — being phased out in favor of PIM and granular roles.) |
| **Privileged Role Administrator** | Manages role assignments, PIM, and eligible assignments. |
| **Password Administrator** | Resets passwords for other users. |
| **Conditional Access Administrator** | Creates/manages Conditional Access policies. |
| **Cloud App Administrator** | Manages enterprise applications and their settings. |
| **Application Administrator** | Manages app registrations, enterprise applications, and their OAuth settings. |
| **Domain Admin** | Manages domain-related settings for a specific verified domain. |
| **Device Administrator** | Manages devices (register, delete, compliance). |
| **User Administrator** | Manages users (create, delete, reset passwords, manage licenses). |
| **Security Administrator** | Manages security-related configurations (Defender, Security Center integration, security alerts). |
| **Global Reader** | Read-only access to all Entra ID settings. |
| **Privileged Role Administrator** | Can assign and manage roles. |

**Azure RBAC roles:**
These are **resource-level roles** that control who can do what to Azure resources. They apply at Management Group, Subscription, Resource Group, or Resource scope.

| Role | Permissions |
|------|------------|
| **Owner** | Full access to resources + can assign RBAC |
| **Contributor** | Can create/manage all resources, but cannot assign RBAC |
| **Reader** | Read-only access to resources |
| **User Access Administrator** | Can manage RBAC role assignments |
| **Network Contributor** | Can manage network resources |
| **Storage Contributor** | Can manage storage resources |
| (and hundreds more) | |

**The distinction:**
```
Directory Role = "You can administer ENTRA ID" (tenant-level identity management)
Azure RBAC Role = "You can administer AZURE RESOURCES" (resource-level access)

A user can have:
- Global Administrator (Entra ID directory role) — can manage ALL identity settings
- Contributor on an RG (Azure RBAC) — can manage resources in that RG
- Both simultaneously

A user can have NO directory roles but still have Azure RBAC roles
(e.g., a service principal or managed identity with RBAC but no directory role)

A user can have Global Administrator but NO Azure RBAC roles
(they can manage Entra ID but might not be able to create VMs)
```

**L3 troubleshooting relevance:**
- "I can't create a VM" → Check Azure RBAC (not directory roles).
- "I can't reset passwords" → Check Entra ID directory roles (need Privileged Role Admin or Password Admin).
- "I can't create Conditional Access policies" → Check Conditional Access Administrator directory role.
- "I can't create app registrations" → Check Application Administrator directory role.

---

### 3.3.6 AZURE RBAC (Entra ID-scoped)

Azure RBAC can also be applied **at the directory scope** (not just resource/resource group/subscription/management group). This allows directory-wide permissions like:
- `Azure AD Connect DC Administrator` — allows joining computers to the directory
- `Azure AD Connect Cloud Administrator` — allows cloud-only configuration
- Directory-specific roles can also be assigned at the directory scope for read-only access to all Entra ID objects.

---

## 3.4 USERS — COMPLETE DEEP DIVE

### 3.4.1 User Types

| User Type | Description | Use Case |
|-----------|-------------|----------|
| **Member user** | A user who belongs to your organization. Full access to organizational resources (subject to policies). | Employees, contractors with Entra ID accounts |
| **Guest user** | External users invited to your tenant. Access is limited by Conditional Access and guest user permissions. | Partners, vendors, customers, B2B collaborators |
| **Service principal** | Non-human identity used by apps/services. (Covered in Applications section.) | Apps, daemons, automation, service accounts |

---

### 3.4.2 Member User Properties

| Property | Description | Immutable? |
|----------|-------------|-----------|
| **Display Name** | User's name (e.g., "John Smith") | Editable |
| **User Principal Name (UPN)** | Sign-in ID (e.g., `john@contoso.com`) | Editable (must match verified domain) |
| **Mail Nickname** | Legacy Exchange attribute (e.g., `john`) | Editable |
| **First Name / Last Name** | Given name, surname | Editable |
| **Password** | Credential (can be password, federated, or passwordless) | Managed by auth system |
| **Object ID** | Unique identifier | Immutable |
| **External User?** | Whether this is a member or guest | Set at creation |
| **UserType** | "Member" or "Guest" | Editable (can convert member to guest and vice versa) |
| **Identity Provider** | `common` (Entra ID local), `Google`, `Facebook`, `Microsoft Account`, or federated | Set at creation |
| **UsageLocation** | Required for licensing (determines license eligibility based on country) | Must be set for licensing |
| **Employee ID** | Optional custom attribute | Editable |
| **Department / Job Title** | Optional attributes | Editable |

---

### 3.4.3 Guest User Properties

Guest users have additional properties:
| Property | Description |
|----------|-------------|
| **Invited By** | Which user or admin sent the invitation |
| **Invitation Email** | Email used to accept invitation |
| **Redeem URL** | Link to redeem the invitation |
| **Invitation Status** | Pending, Accepted, Revoked, etc. |
| **Tenant ID** | Shows in Directory Objects as cross-tenant |
| **Internal Identifiers** | `invitedUserExternalTenantId`, `inviterTenantId` |

**L3 troubleshooting:**
Guest access issues are among the most common identity problems in enterprise environments. Always check:
- Has the guest accepted the invitation?
- Is the guest's home tenant allowed by Conditional Access (Cross-Tenant Access settings)?
- Does the guest have the necessary RBAC roles in the inviting tenant?
- Is the guest's UPN verified in the inviting tenant?

---

### 3.4.4 User Lifecycle

```
1. Creation
   └─ Member: Admin creates, or self-service sign-up (if enabled)
   └─ Guest: Existing user sends invitation → Guest receives email → Redeems

2. Sign-in
   └─ Authentication via Entra ID (password, MFA, federation, etc.)

3. Active management
   └─ Admin manages properties, licenses, group memberships, roles

4. Password management
   └─ User changes password (SSPR or admin-assisted)
   └─ Admin resets password
   └─ Federated: Password changed at on-prem AD, synced via Entra Connect

5. Disable / Enable
   └─ Admin can block user sign-in without deleting the account
   └─ Blocked sign-in = user cannot authenticate (account still exists, all data preserved)

6. Deletion
   └─ Soft delete (30-day recovery window)
   └─ Hard delete (after 30 days or purged earlier)
   └─ Requires Global Admin or Privileged Role Admin

7. Recovery (from soft delete)
   └─ Must be done within 30 days
   └─ Restores all properties, group memberships, licenses
   └─ Requires Global Admin, Privileged Role Admin, or User Administrator

8. Permanent deletion
   └─ After recovery window expires (30 days default, max 365 days with notification)
```

---

### 3.4.5 Password Management

| Method | Description | Managed By |
|--------|-------------|------------|
| **Local password** | User has a password stored in Entra ID | User changes it themselves (or admin resets) |
| **Federated password** | Password stored on-premises (AD FS or Entra Connect) | User changes at on-prem, synced via Entra Connect |
| **Passwordless** | FIDO2 key, Microsoft Authenticator, TOTP, Windows Hello | User authenticates via method, no password stored |
| **Temporary Access Pass (TAP)** | Time-limited, one-time-use password for onboarding/break-glass | Admin generates, sets expiry |

**Password policies:**
- Enforced at tenant level via Authentication Methods and Conditional Access
- Complexity, length, expiry, history — configurable in Entra ID settings
- SSPR: Self-Service Password Reset allows users to reset their own passwords
- PTA: Password Writeback allows on-prem password changes to sync back to the cloud

**L3 operational impact:**
- Federated password changes do NOT trigger Entra ID password reset events — they happen on-premises and sync.
- If PTA is configured but Entra Connect is down → passwords can't be written back to on-prem.
- If SSPR is misconfigured → users can't reset passwords when locked out → helpdesk load increases.

---

### 3.4.6 Authentication Methods

Authentication methods are configured at the tenant level (Authentication Methods policy) and per-user (Authentication Methods settings in My Profile).

| Method | Description | Admin Control |
|--------|-------------|---------------|
| **Microsoft Authenticator** | Push notification, OTP, passwordless, FIDO2 | Enable/disable per tenant, set as required |
| **FIDO2 Security Key** | Hardware key (YubiKey, etc.) | Enable/disable per tenant, set as required |
| **TOTP** | Time-based OTP (via app) | Enable/disable |
| **SMS** | Text message OTP | Enable/disable, deprecated for many scenarios |
| **Voice call** | Phone call OTP | Enable/disable, deprecated for many scenarios |
| **Email OTP** | One-time code sent to email | Enable/disable |
| **Windows Hello for Business** | Biometric/PIN on Windows devices | Enable/disable, requires Hybrid Join or Entra Join |
| **Certificate-based auth** | Client certificate | Enable/disable |

**Passwordless authentication flow:**
```
User goes to sign-in page
→ Enters username (UPN)
→ User is prompted for passwordless method (FIDO2, Authenticator, etc.)
→ User presents credential (touch FIDO2 key, use Windows Hello, etc.)
→ Credential verified by Entra ID
→ Token issued
```

---

### 3.4.7 SSPR (Self-Service Password Reset)

SSPR allows users to reset their own passwords without admin help.

**Configuration:**
- **Authentication methods:** Which methods users can use (Authenticator, email, SMS, security questions)
- **Who can reset:** All users, or specific groups/directory roles
- **Registration:** Whether users must register authentication methods before they're allowed to reset

**Flow:**
```
User locks out / forgets password
→ User navigates to password reset (or is prompted during sign-in)
→ User selects registered authentication method
→ Code sent via registered method (push notification, email, SMS, etc.)
→ User verifies code
→ User sets new password
→ Password updated in Entra ID (or written back to on-prem if PTA configured)
→ User can now sign in
```

**L3 troubleshooting:**
- "User cannot reset password" → Check SSPR configuration, authentication methods registered, user is in eligible group, Conditional Access not blocking, registration required not forcing pre-registration.
- "User registered for SSPR but gets no push notification" → Check device registration, Authenticator app configuration, Conditional Access.

---

### 3.4.8 Account Disable / Enable

**Block sign-in (disable):**
- User cannot authenticate
- Account, group memberships, licenses, and data are ALL preserved
- Resource access continues for already-running sessions (depending on token lifetime)
- Managed identity sessions may continue using existing tokens until they expire

**Enable (unblock):**
- User can sign in again
- New tokens issued upon next sign-in

**Difference from deletion:**
- Disabled: Account exists, just cannot sign in. Quick reversal.
- Deleted: Account removed (soft delete → hard delete). Recovery possible within 30 days.

**L3 best practice:**
- Use "block sign-in" instead of deletion when a user leaves or is compromised.
- Deletion is a security risk if done immediately (no recovery window, no audit trail of actions performed after deletion).
- Always disable first, investigate, then decide on deletion later.

---

### 3.4.9 Deleted Users

**Soft delete:**
- User is "soft-deleted" — they cannot sign in, but all properties, group memberships, and licenses are preserved.
- Default recovery window: 30 days (configurable up to 365 days with notification).
- During soft delete, the user is in a "Deleted" state.

**Hard delete:**
- After recovery window (or if admin purges early), the user is permanently removed.
- Group memberships, licenses, and audit history are still partially preserved in audit logs.

**Recovery process:**
```
1. Global Admin / Privileged Role Admin / User Administrator
2. Portal: Entra ID → Users → All users → Show: Deleted users
3. Select user → Restore
4. User returns with all original properties (name, UPN, group memberships, etc.)
5. User can sign in again
6. If the user's UPN was reassigned during deletion, contact Microsoft Support
```

**L3 operational impact:**
- If you accidentally delete an admin account with no other Global Admins → contact Microsoft Support (RA ID can restore after hard delete).
- Always have break-glass accounts for emergencies.

---

### 3.4.10 User Properties — L3 Relevance

| Property | Why L3 Should Care |
|----------|-------------------|
| **UsageLocation** | Required for license assignment. Without it, user cannot be assigned a license. |
| **UserType (Member/Guest)** | Affects Conditional Access behavior, resource access, and RBAC eligibility. |
| **Identity Provider** | Determines where authentication occurs (Entra ID local, Google, federation). |
| **Account Enabled** | Determines if user can sign in. Different from deletion. |
| **Password Policies** | Determines password requirements. Affected by SSPR/PTA/PTA configuration. |
| **Assigned Licenses** | Determines which services the user can use (M365, Entra P2, Defender, etc.). |
| **Refresh Token Valid From** | Determines when existing refresh tokens become invalid (security trigger). |
| **Assigned Plans** | Service-specific plans (e.g., EMS, Office 365). |
| **Mutable Properties** | Properties that can be updated via Graph API, Portal, or CLI. |

---

## 3.5 GROUPS — COMPLETE DEEP DIVE

### 3.5.1 Group Types

| Group Type | Description | Use Case |
|------------|-------------|----------|
| **Security group** | Can be assigned RBAC roles and used in Conditional Access | Access control, licensing, policy targeting |
| **Microsoft 365 group** | Has a mailbox, calendar, SharePoint site, Teams connection (also acts as security group in Entra) | Collaboration, team resources |
| **Device group** | (Deprecated/limited) Used for device-based Conditional Access | Legacy scenarios |

**L3 focus: Security groups and Microsoft 365 groups are the two main types.**

---

### 3.5.2 Membership Types

| Membership Type | Description | Use Case |
|-----------------|-------------|----------|
| **Assigned** | Admin manually adds/removes members | Static teams, specific access requirements |
| **Dynamic User** | Membership based on user attributes (e.g., department = "Engineering") via Entra ID rules | Auto-populated groups for RBAC, licensing |
| **Dynamic Device** | Membership based on device attributes | Device-based Conditional Access, licensing |

**Dynamic membership rules:**
```
Example: All users where Department equals "Engineering"
Rule: [User.Department -eq "Engineering"]
```

Dynamic groups are extremely powerful for:
- Auto-assigning RBAC roles (e.g., give all Engineering group members Contributor on a specific RG)
- Auto-assigning licenses (e.g., all Engineering users get M365 E5)
- Targeting Conditional Access policies

**Important:** Dynamic membership requires Entra ID Premium (P1 or P2) SKU.

---

### 3.5.3 Group Ownership

Every group has **owners** (1 or more) and **members**.

- **Owners:** Manage group membership, settings, and can delete the group. Default creator is the owner.
- **Members:** Are part of the group. Can be added/removed by owners.
- **Ownership can be delegated:** Multiple owners, or specific users.

**L3 relevance:**
- If a group has no owner and the original creator leaves → group becomes unmanageable.
- Group ownership is critical for security — a compromised group owner can add malicious members.
- Monitor group ownership changes in audit logs.

---

### 3.5.4 Group-Based Licensing

**What it is:**
Licenses are assigned based on group membership instead of assigning licenses individually to users.

**How it works:**
```
Admin creates a security group (e.g., "Engineering-Licenses")
Admin assigns a license (e.g., Enterprise Mobility + Security E5) to the group
All members of the group automatically receive the license
Members added later also receive the license
Members removed later lose the license (with a grace period)
```

**Benefits:**
- No need to individually license hundreds/thousands of users.
- Easy to manage licenses at scale.
- Dynamic groups can auto-assign licenses based on attributes.

**L3 troubleshooting:**
- "User doesn't have a license" → Check if they're in the licensed group, check group membership (dynamic rule might not match), check license availability (sufficient seats remaining), check Conditional Access (if license requires compliant device, user might not access).

---

### 3.5.5 Nested Group Limitations

**What are nested groups?**
A group within a group. E.g., "Engineering" group is a member of "All-Staff" group.

**Important behaviors:**
| Feature | Behavior |
|---------|----------|
| **Dynamic groups** | Cannot contain nested dynamic groups as members. Dynamic membership rules don't cascade. |
| **Assigned groups** | Can contain nested assigned groups. Membership cascades. |
| **License assignment** | Licenses assigned to a parent group are effective for members of nested groups too. |
| **RBAC assignment** | If a group is assigned a role, and that group is nested in another group, RBAC propagates (if scope allows). |
| **Conditional Access** | Conditional Access evaluates membership at the principal level — nested groups ARE evaluated. |
| **Microsoft 365 groups** | Cannot be nested inside security groups (and vice versa in some configurations). |
| **Max nesting depth** | Microsoft does not enforce a hard nesting limit, but deeply nested groups cause performance issues in policy evaluation. |

**L3 best practice:**
- Avoid deep nesting (more than 3-4 levels) — it causes:
  - Slower Conditional Access evaluation
  - Harder troubleshooting (effective membership is hard to determine)
  - Policy conflicts
- Use dynamic groups at leaf level, assigned groups at high level if nesting is needed.

---

### 3.5.6 Group-Driven Authorization Flow

```
User signs in
→ Entra ID issues token
→ Token includes group membership (Object IDs of groups user belongs to)
→ Group claims are embedded in the token (if configured)
→ When user accesses a resource:
  1. Resource checks token's group claims
  2. If user is in required group → Access granted (subject to RBAC/CA)
  3. If user is not in required group → Access denied
→ For Conditional Access:
  1. CA evaluates user's group membership (from token or live directory)
  2. If condition matches user's group → Apply grant controls
  3. If condition doesn't match → Continue to next condition or grant default
```

**Token group claims:**
- By default, tokens include only up to 200 groups (to keep token size reasonable).
- For users in more than 200 groups, group claims are replaced with a "strong claim" — a coded indicator, and the application must query Microsoft Graph API for actual group membership.
- **L3 troubleshooting:** If a user is in a group but doesn't get access, check if the group claim is in the token, or if it exceeds the 200-group limit.

---

## 3.6 APPLICATIONS — COMPLETE DEEP DIVE

This section covers **App Registrations**, **Enterprise Applications**, and **Service Principals** — and the critical distinction between them.

---

### 3.6.1 App Registration vs Enterprise Application vs Service Principal — THE CRITICAL DISTINCTION

These three terms are frequently confused. They are **three different objects** that work together.

| Concept | What It Is | Where It Lives | Purpose |
|---------|-----------|---------------|---------|
| **App Registration** | A **definition** of an application (identity, permissions, capabilities) | Per-tenant (one registration per tenant that needs to use the app) | Defines "What this app CAN do and what ID it uses" |
| **Enterprise Application** | An **instance** of an app registration within your tenant (how it's used in your org) | Per-tenant (one EE per tenant for each app) | Represents "How this app is used in your organization" |
| **Service Principal** | The **identity** of the app registration at runtime (used for authentication) | Created automatically when app registration is created | The actual "account" the app uses to sign in |

**Analogy:**
```
App Registration = Blueprint of a house (architectural design)
Enterprise Application = The house as built in your neighborhood (instance)
Service Principal = The person who lives in the house (identity)
```

**Real-world example — Microsoft 365 app:**
```
App Registration: "Microsoft 365" in your tenant
  → Defines: API permissions (Mail.Read, Files.Read), OAuth endpoints, redirect URIs
Enterprise Application: "Microsoft 365" instance in your tenant
  → Shows: User consent status, provisioning status, assigned users/groups, configuration
Service Principal: The Microsoft 365 identity in your tenant
  → Used: To authenticate when your app accesses Microsoft 365 APIs
```

**Relationship diagram:**
```
App Registration (definition)
  │
  ├── Creates → Service Principal (identity, in the same tenant)
  │
  └── Creates → Enterprise Application (instance, in the same tenant)
       │
       ├── User/Group Consent (who has agreed to use this app)
       ├── Provisioning (SCIM, user assignment)
       ├── Conditional Access (how CA evaluates this app)
       └── API Permissions (admin-assigned vs user-assigned)
```

**One-to-one relationship:**
- Each App Registration creates exactly ONE Service Principal in the same tenant.
- Each App Registration creates exactly ONE Enterprise Application in the same tenant.
- BUT: A single App Registration (from a multi-tenant app) can create Enterprise Applications in MULTIPLE tenants (each tenant that grants consent). Each of those Enterprise Applications has its own Service Principal in that tenant.

---

### 3.6.2 App Registration — Deep Dive

**What it is:**
An App Registration is a **definition** of an application's identity in Entra ID. It tells Entra ID:
- "This app exists"
- "Here's how to authenticate it" (redirect URIs, token configurations)
- "Here's what it can access" (API permissions)
- "Here's its identity" (client ID, secret/certificate)

**Key properties:**
| Property | Description |
|----------|-------------|
| **Application (Client) ID** | A GUID that uniquely identifies this app registration. Immutable. Used in authentication requests. |
| **Object ID** | A GUID that uniquely identifies the app registration object in Entra ID. Immutable. Used in administrative operations. |
| **Redirect URIs** | Where tokens are sent after authentication (web apps, mobile apps, SPA) |
| **Supported account types** | Who can sign in (single tenant, multi-tenant, personal Microsoft accounts) |
| **API permissions** | Permissions the app requests (delegated + application) |
| **Certificates & Secrets** | Credentials used for authentication |
| **Token configuration** | Custom claims added to tokens |
| **Description** | Human-readable name and description |

**Where to find:** Portal → Entra ID → App Registrations

---

### 3.6.3 Service Principal — Deep Dive

**What it is:**
A Service Principal is the **runtime identity** of an app registration. It is what actually authenticates and gets tokens when the app runs.

**When it's created:**
- Automatically when an App Registration is created in the same tenant.
- It has the same Application ID but a different Object ID.

**Key properties:**
| Property | Description |
|----------|-------------|
| **Application ID** | Same as the app registration's Application ID |
| **Object ID** | Unique Entra ID object ID for the Service Principal |
| **Tenant ID** | The tenant where this Service Principal lives |
| **Account Enabled** | Whether the Service Principal can authenticate |
| **Credential Expiry** | When secrets/certificates expire |
| **Assigned RBAC Roles** | What the SP can do in Azure |
| **Managed Identity type** | System-assigned or user-assigned (if applicable) |

**Key distinction:**
- App Registration = the app's "registration document" (what the app is and can do)
- Service Principal = the app's "account" (what the app uses to log in)

**L3 troubleshooting:**
- "App registration exists but authentication fails" → Check Service Principal: is it disabled? Are credentials expired?
- "Service principal cannot access storage" → Check RBAC role assignment on Storage Account (Service Principal needs specific role, not just App Registration).
- "Service principal is in our tenant but app registration is from another tenant" → This is normal for multi-tenant apps. Each tenant has its own SP.

---

### 3.6.4 Enterprise Application — Deep Dive

**What it is:**
An Enterprise Application is an **instance** of an app registration within your tenant. It shows how the app is used in your organization.

**Where to find:** Portal → Entra ID → Enterprise Applications

**Key sections:**
| Section | What It Shows |
|---------|---------------|
| **Overview** | App registration source, provisioning status, user count |
| **Users and groups** | Who is assigned to this app (user/group assignments) |
| **Consent** | Who has consented (admin vs user consent) |
| **Properties** | Configuration settings for how the app behaves in your tenant |
| **Security** | Conditional Access, risk detection, audit logs for this app |
| **Single sign-on** | SSO method configured for this app instance |
| **Provisioning** | SCIM-based auto-provisioning to SaaS apps |
| **Automated provisioning** | Whether SCIM provisioning is enabled |

**L3 troubleshooting:**
- "User cannot access SaaS app" → Check Enterprise Application: is the user assigned? Is provisioning in progress? Is consent given?
- "App shows as 'Not Provisioned'" → User hasn't been assigned yet, or provisioning hasn't completed.
- "Enterprise Application is missing" → The app registration exists but no user has consented/been assigned.

---

### 3.6.5 Certificates & Client Secrets

**Two types of credentials for App Registrations / Service Principals:**

| Credential Type | Description | Use Case |
|-----------------|-------------|----------|
| **Client Secret** | A string (like a password) stored in Entra ID | Simple authentication, daemon apps, CI/CD pipelines |
| **Certificate** | A public/private key pair (X.509 certificate) | Higher security, automated systems, avoid secret exposure |

**Client Secret properties:**
- Value: Generated once, displayed only at creation (must be saved!)
- Expiry: configurable (e.g., 1 year, 2 years, custom)
- Description: for identification

**Certificate properties:**
- Public certificate: uploaded to App Registration (used to verify the app)
- Private key: stays on the application server (used to sign authentication requests)
- Expiry: certificate expiry
- Thumbprint: unique identifier for the certificate

**L3 operational concerns:**
- Secrets MUST be rotated before expiry. Expired secret = authentication failure.
- Use Key Vault to store secrets/certificates, not code or config files.
- Certificate-based auth is more secure but more complex to manage.
- Monitor expiry via automated alerts (e.g., Azure Policy, custom monitoring).

---

### 3.6.6 OAuth 2.0 / OpenID Connect — Deep Dive

**OAuth 2.0:**
Authorization framework. Allows an app to access resources on behalf of a user (with consent) or on its own (as a service principal).

**OpenID Connect (OIDC):**
An authentication layer ON TOP of OAuth 2.0. Adds identity information (ID token) to OAuth flows.

**Common OAuth flows (grant types):**
| Flow | Use Case |
|------|----------|
| **Authorization Code (with PKCE)** | Web apps, SPAs, mobile apps — user signs in via browser |
| **Authorization Code (confidential client)** | Server-side web apps — secret used alongside code |
| **Client Credentials** | Daemon/service apps — no user involved, Service Principal authenticates directly |
| **Implicit** (deprecated) | Legacy SPAs — replaced by Auth Code + PKCE |
| **Resource Owner Password Credentials (ROPC)** (deprecated) | Legacy — disabled by default in modern Entra ID |

**Token types issued:**
| Token Type | Contains | Audience | Lifetime |
|------------|----------|----------|----------|
| **ID Token** | Identity info (oid, upn, name, email, groups) | Application's Client ID | Typically 1 hour |
| **Access Token** | Permissions (scopes), resource-specific claims | Resource's App ID URI (e.g., `https://graph.microsoft.com`) | Typically 1 hour |
| **Refresh Token** | Long-lived credential to get new access tokens | Varies | Varies (session-based, configurable up to 90 days or 180 days in new model) |

**L3 troubleshooting — Token issues:**
- "App gets 401 Unauthorized" → Check token expiry, token audience (is it for the right resource?), client secret/cert validity, app registration permissions.
- "Token doesn't contain expected group claims" → Check token version (v1.0 vs v2.0), group claim configuration (200-group limit), whether to use Graph API instead.
- "Consent required" → User hasn't consented, or admin consent required (configured in API permissions).
- "Invalid grant" → Client secret expired, certificate expired, or authorization expired.

---

### 3.6.7 SAML (Where Relevant)

SAML (Security Assertion Markup Language) is an older authentication protocol. While OAuth/OIDC is now dominant, SAML is still used by many legacy enterprise apps (Salesforce, some ERP systems, etc.).

**SAML flow:**
```
User → Browser → Service Provider (app) → Redirect to Entra ID
→ User signs in at Entra ID → Entra ID issues SAML assertion
→ Assertion sent back to Service Provider → User authenticated
```

**Entra ID SAML support:**
- Entra ID acts as the SAML Identity Provider (IdP).
- Apps consume SAML tokens as Service Providers (SP).
- SAML is in maintenance mode — new apps should use OIDC/OAuth.

**L3 relevance:**
- SAML-based apps may have different group claim behaviors than OIDC apps.
- SAML tokens have different lifetime and refresh characteristics.
- Mixed OAuth/SAML environments require careful Conditional Access configuration.

---

### 3.6.8 Consent — Deep Dive

**What is consent?**
Consent is the user's (or admin's) agreement to let an application access specific resources/data.

| Consent Type | Who | Scope |
|-------------|-----|-------|
| **User consent** | Individual user | Limited to what the user can access; requires user's permission |
| **Admin consent** | Tenant admin (Global, Cloud App Admin, or Application Admin) | Grants for ALL users in the tenant; admin authorizes on behalf of everyone |

**API Permissions — Application vs Delegated:**
| Permission Type | What It Means | When Used |
|-----------------|--------------|-----------|
| **Application permission** | App accesses resource as itself (Service Principal), no user involved | Background services, daemons, scheduled tasks |
| **Delegated permission** | App accesses resource on behalf of a signed-in user | User-interactive apps, web apps with user context |

**Important:**
- Application permissions require **admin consent**.
- Delegated permissions can be consented by users (if configured) or require admin consent.
- An app with both types may use either depending on the authentication flow.

**L3 troubleshooting:**
- "App fails with 'Consent required'" → Admin consent needs to be granted for the API permission.
- "App works with admin consent but fails for individual users" → User consent is blocked by Conditional Access or policy.
- "API permission changed but app still fails" → Admin consent may need to be re-granted, or token needs to be refreshed (new token includes updated permissions).

---

### 3.6.9 Managed Identities — Deep Dive

**What it is:**
A Managed Identity is a Service Principal that is **automatically managed by Azure**. It is tied to a specific Azure resource and does not require you to manage credentials (secrets, certificates).

**Types:**
| Type | Description | Lifecycle |
|------|-------------|-----------|
| **System-assigned** | Created automatically when the resource is created. Tied to the resource lifecycle — if the resource is deleted, the identity is deleted. | Same as the resource |
| **User-assigned** | Created as a standalone Entra ID object. Can be assigned to multiple resources. Managed independently of resources. | Independent — must be explicitly deleted |

**Key properties:**
| Property | System-Assigned | User-Assigned |
|----------|----------------|---------------|
| **Identity ID** | Same as resource ID (conceptually) | Separate Object ID in Entra ID |
| **Principal ID** | Unique, assigned at creation | Unique, assigned at creation |
| **Automatic creation** | Yes (with resource) | No (created separately) |
| **Automatic deletion** | Yes (when resource deleted) | No (must be deleted explicitly) |
| **Can be assigned to multiple resources** | No (tied to one resource) | Yes |
| **RBAC role assignments** | On the resource itself or scope | Same |

**L3 operational concerns:**
- **System-assigned:** If you recreate a resource (delete + create), the system-assigned identity is NEW. All old role assignments are LOST. You must reassign roles.
- **User-assigned:** More portable — can be moved between resources, survive resource recreation.
- **Monitoring:** Track credential expiry (system-assigned managed identity tokens are rotated automatically; but external token audiences may have expiry considerations).
- **Cross-subscription:** User-assigned identities CAN be assigned across subscriptions; system-assigned CANNOT.

---

## 3.7 COMMUNICATION FLOW — AUTHENTICATION & AUTHORIZATION

### User Signs In → Entra ID → Token → Resource Access

```
1. USER ACTION
   User opens app → Redirected to Entra ID sign-in page
   User enters UPN + password (+ MFA if configured)

2. ENTRA ID AUTHENTICATION
   Entra ID validates:
   - User exists in directory? (Object ID lookup)
   - User is enabled? (AccountEnabled = true)
   - Password valid? (or federated check to on-prem)
   - MFA required? (Conditional Access evaluation)
   - Account has no risks? (Identity Protection)
   - Sign-in risk acceptable?
   → Authentication successful

3. TOKEN ISSUANCE
   Entra ID issues:
   - ID Token (identity info, groups)
   - Access Token (for target resource, with scopes/permissions)
   - Refresh Token (for session persistence)
   All tokens signed with Microsoft keys, include:
   - aud (audience = resource app ID URI)
   - iss (issuer = Entra ID tenant)
   - oid (object ID of user)
   - tid (tenant ID)
   - iat (issued at)
   - exp (expiry)
   - groups (group memberships, up to 200 or encoded)

4. RESOURCE ACCESS
   User/app presents Access Token to target resource
   Resource validates:
   - Token signature (via JWKS endpoint)
   - Token issuer (must be trusted)
   - Token audience (must be for this resource)
   - Token expiry (not expired)
   - User/SP has permission via RBAC or resource-level access

5. ACCESS GRANTED / DENIED
   - If all checks pass → Resource returns data
   - If any check fails → 401/403 error

6. TOKEN REFRESH (when access token expires)
   App uses Refresh Token to get new Access Token
   Entra ID validates Refresh Token:
   - Still valid (not revoked, not expired)
   - Session still active (Conditional Access still allows)
   → Issues new Access Token (and possibly new Refresh Token)
```

---

## 3.8 ENTERPRISE APPLICATION CONSUMPTION — COMPLETE FLOW

### When an app (App Registration) is used in your tenant:

```
1. APP REGISTRATION exists (defined in your tenant)
   - Has: Application (Client) ID, Redirect URIs, API Permissions

2. ENTERPRISE APPLICATION is the instance in your tenant
   - Has: User/Group assignments, Provisioning, Consent

3. SERVICE PRINCIPAL is the identity created for this app
   - Used: For authentication when app accesses resources

4. USER SIGN-IN → App uses authentication flow → Gets tokens from Entra ID
   - For delegated permissions: User's identity + permissions
   - For application permissions: Service Principal's identity + permissions

5. APP USES ACCESS TOKEN to call Microsoft Graph or other APIs
   - Token includes user's or SP's permissions
   - Resource validates token → Grants or denies access
```

---

## 3.9 ADMINISTRATION

### Key Administrative Operations

| Operation | Tool | Scope |
|-----------|------|-------|
| Create/manage users | Portal, CLI, Graph API, Entra Connect/Cloud Sync | Tenant |
| Create/manage groups | Portal, CLI, Graph API, Entra Connect/Cloud Sync | Tenant |
| Create/manage app registrations | Portal, CLI, Graph API, API | Tenant |
| Assign API permissions | Portal, CLI, Graph API | Per app registration |
| Grant admin consent | Portal (admin consent workflow), Graph API | Tenant-wide |
| Assign RBAC roles | Portal, CLI, PowerShell, Graph API | Any scope |
| Assign groups to roles | Portal, CLI | Directory-scoped roles |
| Manage Conditional Access | Portal, CLI | Tenant (or subset via groups) |
| Manage authentication methods | Portal, CLI | Tenant |
| Manage SSPR | Portal, CLI | Tenant |
| Manage licenses | Portal, CLI, Graph API | Subscription/directory |
| Group-based licensing | Portal, CLI | Directory |
| Manage guest invitations | Portal, CLI, Graph API | Tenant |
| Manage Dynamic membership rules | Portal, CLI, Graph API | Per group |
| Group ownership management | Portal, CLI | Per group |
| Password reset (admin) | Portal, CLI, Graph API | Per user |
| Block sign-in / Delete user | Portal, CLI, Graph API | Per user |
| Directory role assignment | Portal, CLI, Graph API | Tenant |
| B2B collaboration (guest) | Portal, CLI, Graph API | Tenant |
| Provisioning (SCIM) | Portal, CLI, API | Per Enterprise App |
| Audit logs | Portal, CLI, Graph API | Tenant |

---

## 3.10 SECURITY CONSIDERATIONS

| Concern | Risk | Mitigation |
|---------|------|------------|
| **Global Admin sprawl** | Too many Global Admins = high risk of compromise | PIM (just-in-time elevation), reduce to 2-4 named admins, use Azure AD Connect/DC Admin roles for hybrid identity only |
| **Stale service principals** | Unused SPs with valid secrets are security risks | Regular access reviews, audit logs, PIM for SPs |
| **Over-privileged apps** | Apps with excessive API permissions | Review API permissions regularly, use least privilege |
| **Consent phishing** | Users consenting to malicious apps | Block user consent, require admin consent, Conditional Access app controls |
| **Legacy authentication** | Basic auth protocols bypass CA | Disable legacy auth in Conditional Access (Monitor → Legacy Authentication) |
| **Guest access abuse** | External users with excessive access | Limit guest roles, Conditional Access for guests, Cross-Tenant Access settings |
| **Token theft** | Stolen tokens grant access | Short token lifetimes, Conditional Access session controls, FIDO2 (phishing-resistant) |
| **SSPR bypass** | Users bypassing security via weak SSPR methods | Enforce strong SSPR methods (Authenticator, FIDO2) |
| **Hardcoded secrets** | App secrets in source code | Key Vault, managed identities, secret rotation automation |
| **Group-based licensing mismatch** | Wrong licenses assigned dynamically | Careful dynamic group rules, regular audit |
| **App registration proliferation** | Too many app registrations = hard to govern | App governance policies, naming standards, regular cleanup |

---

## 3.11 MONITORING

### What to Monitor for Entra ID

| Metric/Log | What It Shows | Alert Trigger |
|------------|---------------|---------------|
| **Sign-in logs** | All authentication events (success/failure), user, app, CA result, risk level, device | Failed sign-ins, suspicious sign-ins, legacy auth usage |
| **Audit logs** | Administrative changes (role assignments, user/group changes, policy changes) | Unauthorized role changes, bulk user operations |
| **Non-interactive sign-in logs** | Service principal / managed identity authentication events | Unusual SP sign-in patterns |
| **Conditional Access compliance** | CA policy evaluation results | Policies not applying, exclusions being exploited |
| **Risk detections (Identity Protection)** | Sign-in risk, user risk, risky users, risky sign-ins | Elevated risk levels |
| **Token issuance** | Token issuance rates, types, audiences | Abnormal token volumes (potential compromise) |
| **App consent** | User vs admin consent events | User consent for high-permission apps |
| **Password reset events** | SSPR usage, admin resets | Excessive resets (helpdesk burden), suspicious resets |
| **Directory role assignments** | New role assignments, changes | Unexpected Global Admin assignments |
| **Guest invitations** | Guest creation, acceptance, revocation | Unusual guest activity |
| **SSPR configuration health** | Registration status, method availability | Users unable to register SSPR methods |

---

## 3.12 PRODUCTION EXAMPLE

**Scenario: Enterprise with 5000 employees, 300 contractors (guests), and 15 line-of-business apps.**

```
Tenant: contoso.onmicrosoft.com (Premium P2 SKU)
Domains: contoso.com (verified), contoso.corp (verified, internal)

Directory Groups:
  - "All-Employees" (Dynamic: Department -ne null)
  - "All-Contractors" (Dynamic: UserType = Guest)
  - "Engineering-Users" (Dynamic: Department = "Engineering")
  - "Engineering-Licenses" (Dynamic: Department = "Engineering")
  - "Finance-Users" (Dynamic: Department = "Finance")
  - "Global-Readers" (Assigned: External auditors)
  - "Break-Glass-Admins" (Assigned: Emergency access, 4 members)
  - "LOB-App-Readers" (Assigned: Per-app requirement)

App Registrations:
  15 LOB apps (each has: App Registration, Enterprise App, Service Principal)
  - Each has: API permissions (admin-consented), redirect URIs, certificates
  - Each Enterprise App: Provisioned, user/group assignments configured

RBAC:
  - "All-Employees" → Reader on Production Resource Group (read-only access)
  - "Engineering-Users" → Contributor on Engineering RG
  - "Finance-Users" → Reader on Finance RG
  - "Global-Readers" → Reader on subscription (for audit)
  - Break-glass → Global Admin (PIM-eligible, time-limited)

Conditional Access:
  - All users: MFA required
  - All users: FIDO2 preferred (Authenticator fallback)
  - Contractors: Cannot access Production RG
  - Legacy authentication: Blocked (Monitored mode initially, then Enforced)
  - Risky users: Block sign-in until resolved
  - Session: 12-hour timeout

SSPR:
  - All employees: Enabled, Authenticator + FIDO2 required
  - Contractors: Disabled (helpdesk resets)

Licensing:
  - All-Employees → Entra P2, Defender for Cloud P1, Office 365 E5
  - All-Contractors → Entra Free (B2B), no Defender license

Identity Protection:
  - Sign-in risk: Block at medium+
  - User risk: Block at medium+
  - Risky sign-ins: Alert + Block
```

---

## 3.13 FAILURE SCENARIOS — IDENTITY

| Scenario | Root Cause | Resolution | Evidence |
|----------|-----------|------------|----------|
| **User cannot sign in** | Wrong password, account disabled, MFA failure, CA policy blocking, tenant lockdown, risk-based block | Check sign-in log, account status, CA evaluation result, risk level, MFA registration | Sign-in log: error code + CA failure reason |
| **Guest user cannot redeem invitation** | Invitation expired, guest tenant blocked by Cross-Tenant Access, guest cannot authenticate (no license) | Check invitation status, Cross-Tenant Access settings, guest sign-in log | Invitation status: Redeemed/Expired/Revoked |
| **Service principal cannot authenticate** | Secret expired, certificate expired, account disabled, SP not assigned required RBAC role, wrong tenant | Check SP status, credential expiry, RBAC assignments, sign-in log | SP sign-in log: error code |
| **Dynamic group has no members** | Dynamic rule doesn't match any users, or users' properties don't match rule | Check rule syntax, user properties, refresh status | Entra ID → Groups → Members count |
| **Group-based licensing doesn't work** | No license available, group not assigned license, dynamic group rule issue, license assignment not propagated | Check license availability, group assignment, member count | License assignment audit log |
| **Enterprise App shows "Not Provisioned"** | User not assigned, provisioning still in progress, SCIM connector down | Check assignments, provisioning status, connector health | Enterprise App → Provisioning logs |
| **SSPR fails for user** | User not registered, no authentication methods enrolled, Conditional Access blocking, SSPR not enabled | Check SSPR config, user's registration, MFA method availability | SSPR audit log |
| **Token validation fails at resource** | Token expired, wrong audience, SP secret expired, certificate expired, certificate not rotated | Check token claims (jwt.ms), SP credentials, resource validation | Token decode (jwt.ms), error message |
| **Admin consent not applying** | Admin consent was not granted, policy change invalidated consent, permission scope changed | Grant admin consent via Portal or Graph API | API permissions: "Consent for all users" status |
| **Nested group membership not recognized** | Deep nesting, dynamic nested group limitation, 200 group claim limit, token stale | Check membership flow, claim configuration, token claims | Token decode → groups claim |

---

## 3.14 TROUBLESHOOTING METHODOLOGY — IDENTITY ISSUES

```
Step 1: Identify the scope
  → Is it one user or many?
  → Is it a member, guest, or service principal?
  → Is it a specific application or all apps?

Step 2: Check sign-in logs (most important for authentication issues)
  → Portal: Entra ID → Monitoring → Sign-in logs
  → Filter by user, app, status, failure reason
  → Look for: errorCode, conditionalAccessStatus, appliedConditionalAccess, riskDetail

Step 3: Check user/SP status
  → AccountEnabled? (not blocked)
  → UserType? (member/guest — affects access)
  → UsageLocation set? (needed for licensing)
  → Credential expiry? (secret/cert not expired)

Step 4: Check Conditional Access evaluation
  → Sign-in log → Applied Conditional Access policies
  → Was the user included? Excluded?
  → What grant controls were required? Were they satisfied?
  → Was it a "Report Only" (not enforced) policy?

Step 5: Check RBAC
  → Does the user/SP have the required role at the correct scope?
  → Is the role assignment propagating (newly assigned = can take up to 15 minutes)?
  → Is there a Deny assignment blocking?

Step 6: Check Azure Policy
  → Could a Policy be blocking (not RBAC)?
  → This is a common source of confusion.

Step 7: Check directory health
  → Entra Connect/Cloud Sync running?
  → Sync errors?
  → Duplicate objects?

Step 8: Check service-specific issues
  → App Registration: permissions, secrets, certificates
  → Enterprise App: provisioning, consent, assignments
  → Managed Identity: role assignments, status

Step 9: Validate
  → Fix identified issue
  → Test sign-in/access again
  → Check sign-in log for success
  → Monitor for recurrence
```

---

## 3.15 LOGS / EVIDENCE — IDENTITY

| Evidence Source | What It Shows | Access Method |
|----------------|---------------|---------------|
| **Sign-in logs** | All authentication events: success/failure, user, app, device, IP, CA evaluation, risk level, tenant ID, user agent, conditionalAccessStatus | Portal → Entra ID → Monitoring → Sign-in logs; Graph API; Log Analytics (via diagnostic settings) |
| **Audit logs** | Administrative actions: role assignments, user/group changes, policy changes, app changes | Portal → Entra ID → Monitoring → Audit logs; Graph API |
| **Non-interactive sign-in logs** | Service principal / SP authentication events | Sign-in logs filtered by Authentication Method = "NonInteractive" |
| **Provisioning logs** | Enterprise App provisioning events (SCIM) | Portal → Enterprise App → Provisioning → History |
| **Risk detections** | Identity Protection risk detections (user risk, sign-in risk, risky users, risky sign-ins) | Portal → Entra ID → Identity Protection → Risk detections |
| **Token validation** | Decode tokens via jwt.ms or similar | Copy access token → paste at jwt.ms |
| **Activity Log (Azure)** | Azure Resource Manager operations with identity context | Portal → Subscription/Resource Group → Activity Log |
| **Graph API** | Programmatic access to all Entra ID data | `https://graph.microsoft.com/v1.0` (requires appropriate permissions) |

---

## 3.16 VERSION/CURRENT SERVICE CONSIDERATIONS (Sept 2026)

| Aspect | Current State | L3 Impact |
|--------|--------------|-----------|
| **Azure AD → Entra ID** | Fully rebranded. All references use "Microsoft Entra ID." Legacy "Azure AD" terminology still common. Portal experience redirects from old Azure AD URLs to Entra. | Scripts and documentation may still reference "Azure AD." The underlying GUID-based identifiers are identical. |
| **Global Admin reduction** | Microsoft is actively pushing PIM and granular role assignment. Fewer Global Admins recommended. Microsoft plans to further restrict Global Admin usage. | Audit Global Admin assignments regularly. PIM-enable them. |
| **Passwordless authentication** | FIDO2 and Microsoft Authenticator passwordless are GA and encouraged. SMS/voice for SSPR are being deprecated. SMS OTP for CA is still supported but strongly discouraged. | Migrate to FIDO2/Authenticator. Disable SMS/voice where possible. |
| **Conditional Access** | GA and core feature in P1+. Granular controls for legacy auth, authentication strengths, session controls, named locations. | Conditional Access is mandatory for L3 environments. Audit regularly. |
| **Cloud Sync vs Entra Connect** | Cloud Sync (Entra Cloud Sync) is the newer lightweight approach. Entra Connect (full) still supported for hybrid identity. Cloud Sync is preferred for most new deployments. | Understand both for hybrid identity troubleshooting. |
| **Cross-Tenant Access** | Configuration portal updated. Settings for external collaboration, B2B direct connect, and inbound/outbound policies. | Critical for guest access scenarios. |
| **Identity Protection** | Risk-based Conditional Access integration. Real-time risk detection. Risk-based enforcement (block, require MFA, alert only). | Enable and configure risk-based policies. |
| **SSPR** | Self-Service Password Reset GA in all SKUs. Registration required before reset is recommended. | SSPR is critical for reducing helpdesk burden. |
| **Named Locations** | GA. Define trusted locations (IP ranges, countries) for Conditional Access. | Use for break-glass and trusted network scenarios. |
| **Authentication Strengths** | GA. Bundles of authentication methods with confidence levels. | Use for high-assurance scenarios (privilele access). |
| **Tenant Restrictions** | Conditional Access: "Tenants" condition for CA. Block external tenant access. | Important for B2B security. |
| **Microsoft Graph API** | Primary management API for Entra ID. Portal is built on top of Graph API. | Graph API is essential for advanced administration, monitoring, automation. |
| **App Consent Policies** | Admin consent workflows and app consent policies are configurable. | Restrict user consent for security. |
| **Privileged Identity Management (PIM)** | GA, P2 feature. Just-in-time elevation for directory roles and Azure RBAC. | PIM is essential for enterprise security. |

---

## 3.17 L3 INTERVIEW QUESTIONS

### Basic
**Q: What is Microsoft Entra ID?**
A: Microsoft Entra ID is Azure's identity and access management platform. It is the directory and authentication authority for all Azure resources, Microsoft 365, and integrated SaaS applications. It manages users, groups, applications, and service principals, and handles authentication, authorization, and policy enforcement.

### Intermediate
**Q: What is the difference between a directory role and an Azure RBAC role?**
A: Directory roles (e.g., Global Administrator, Conditional Access Administrator) are tenant-level roles that control who can administer Entra ID itself. Azure RBAC roles (e.g., Owner, Contributor, Reader) are resource-level roles that control who can create, manage, and delete Azure resources. A user can have directory roles and Azure RBAC roles simultaneously, and they operate independently.

### L3
**Q: A user can sign in but cannot create a VM. What are all the possible causes?**
A:
1. **RBAC:** User lacks `Microsoft.Compute/virtualMachines/write` at RG/subscription scope.
2. **Azure Policy:** Deny policy blocks VM creation (e.g., disallowed location, disallowed SKU).
3. **Quota:** User has exceeded vCPU quota for the region.
4. **Resource Provider:** `Microsoft.Compute` provider not registered in the subscription.
5. **Resource Group lock:** `ReadOnly` lock on the target RG.
6. **Location availability:** VM size not available in the selected region.
7. **Permission propagation delay:** RBAC assignment was made recently and hasn't propagated (up to 15 minutes for some changes).

### Senior L3
**Q: Explain the difference between App Registration, Enterprise Application, and Service Principal. When would each be relevant for troubleshooting?**
A:
- **App Registration:** The definition (blueprint) of the app in your tenant — its identity (Client ID), permissions, redirect URIs. Relevant when: API permissions need changing, authentication endpoints need configuration, or certificates/secrets are being managed.
- **Enterprise Application:** The instance of the app in your tenant — who uses it, provisioning status, consent. Relevant when: users can't access the app, provisioning is stuck, or admin consent needs granting.
- **Service Principal:** The identity (account) used at runtime. Relevant when: authentication fails, RBAC permissions are wrong, or tokens are invalid.
The common misconception is treating them as one object. They are three distinct objects that must be checked independently during troubleshooting.

### Expert
**Q: What happens internally when a user signs in to Entra ID?**
A: (Describe complete flow covering: Azure portal request → Entra ID endpoint → tenant lookup → user object retrieval → password/federation validation → MFA/CA evaluation → risk assessment → token generation (ID + Access + Refresh) → token signing and return → user presents token to resource → resource validates token → access granted/denied.) This should be covered in detail in a dedicated section (Module 69 "What Happens Internally").

### Scenario
**Q: "A service principal was working yesterday, but today it cannot authenticate. What changed?"**
A: Check: (1) Client secret expired — this is the #1 cause. Check credential expiry on the App Registration. (2) Service Principal account disabled. (3) Certificate expired. (4) RBAC role assignment removed. (5) Conditional Access policy changed to block this app. (6) Resource Provider or Azure service outage (check Service Health). (7) Password was changed on-prem and PTA failed (for hybrid scenarios).

### Tricky
**Q: "My user has Global Administrator role but still gets Access Denied when creating a resource. How?"**
A: Global Administrator is a DIRECTORY role (Entra ID), not an Azure RBAC role. Global Admins CAN assign themselves Azure RBAC roles, but if they haven't been assigned a resource-level role (Owner/Contributor at the relevant scope), they still cannot create resources in that scope. Many engineers confuse directory roles with RBAC roles. However, in practice, Global Admins often have Owner at subscription scope in many environments, so this is an environment-specific issue.

### Tricky 2
**Q: "A Conditional Access policy shows as 'Applied' in the sign-in log but access was still allowed without MFA. Why?"**
A: The policy might be in **Report Only** mode (not enforced). Check the policy's Grant controls section — if it says "Report only: Would block/require MFA," the policy is logging but not enforcing. Another possibility: the user is **excluded** from the policy (include/exclude settings), or a more permissive policy with a "grant" control that allows access is taking precedence (policy precedence).

### Tricky 3
**Q: "I added a user to a security group, they can see the RBAC-allowed resource, but they still get Access Denied. Why?"**
A: RBAC role assignments via group membership can take **up to 15 minutes** to propagate. If the user just joined the group, they need to wait. Also, if the user has more than 200 groups, group claims in tokens may be truncated, and the application must query Graph API. The application might not be handling the group claim encoding correctly.

---

## 3.18 SCENARIO-BASED QUESTIONS

### Scenario 1: "User cannot sign in to portal"
**Symptoms:** Sign-in fails with "Account disabled" or "Invalid credentials" or "Conditional Access policy blocks access."
**Architecture:** Entra ID authentication pipeline → CA evaluation → Token issuance.
**Dependencies:** User account state, password, MFA device, CA policies, device compliance, risk level.
**Checks:**
- Sign-in log: What specific error? (errorCode, conditionalAccessStatus)
- Is user account enabled? (AccountEnabled = true)
- Is user blocked by CA? (Applied policies in sign-in log)
- Is user in risk list? (Identity Protection)
- Is password expired/locked out?
- Is MFA device registered?
- Is the user's UPN verified in tenant?
- Is Conditional Access evaluating correctly? (Check include/exclude, precedence, authentication strength)
**Evidence:** Sign-in log entry with failure reason, applied CA policy details, risk detail.
**Root Cause (common):** Account disabled, CA blocks access, password expired, risk level too high, MFA device unregistered.
**Fix:** Enable account, CA exception or policy adjustment, password reset, risk remediation, MFA registration.
**Validation:** User signs in successfully, MFA triggered as expected, resource access works.

### Scenario 2: "Service principal cannot access Azure resources after secret rotation"
**Symptoms:** App returns 401/403. SP sign-in shows "Invalid credentials."
**Architecture:** App → Authentication request → Entra ID → SP validation → Token issuance → Resource access.
**Dependencies:** App Registration credentials, SP status, RBAC assignment.
**Checks:**
- Was the old secret retired but new secret not uploaded?
- Was the new secret uploaded but app still using old secret?
- Is the certificate (if used) correctly uploaded to App Registration and available on the app server?
- Is SP account enabled?
- Is RBAC still assigned to the SP?
**Evidence:** App Registration → Certificates & Secrets → expiry date; SP sign-in log: "Invalid client secret."
**Root Cause:** Secret/cert mismatch between app and App Registration after rotation.
**Fix:** Upload correct credential, update app configuration, restart app, test authentication.
**Validation:** App authenticates successfully, accesses resources.

### Scenario 3: "Dynamic group has zero members despite matching criteria"
**Symptoms:** Group shows 0 members, but users with matching properties exist.
**Architecture:** Dynamic membership → Entra ID rule evaluation → Member assignment.
**Dependencies:** Rule syntax, user properties, Entra ID SKU (P1+ required for dynamic).
**Checks:**
- Is group membership type "Dynamic User"?
- Is the rule syntax correct? (e.g., `[User.Department -eq "Engineering"]`)
- Are users' properties matching the rule? (Check actual property values, not assumed values)
- Is Entra ID P1 or P2 license assigned? (Dynamic groups require this)
- Has the directory synced recently? (Property changes may take time to reflect)
- Are users' UsageLocation set?
- Does the rule reference a property that exists? (e.g., Department must be populated)
**Evidence:** Group → Membership type, Rule details, User properties (Department value), License check.
**Root Cause:** Rule syntax error, property mismatch, or missing license.
**Fix:** Correct rule syntax, update user properties, verify license.
**Validation:** Group member count increases, CA/RBAC assignments take effect.

### Scenario 4: "Guest user was invited but never received invitation email"
**Symptoms:** Invitation created but guest reports no email; invitation status = "Pending."
**Architecture:** Invitation → Email delivery → Guest redemption.
**Dependencies:** Cross-tenant access settings, email delivery, guest tenant policies, spam filters.
**Checks:**
- Cross-Tenant Access settings: Are external collaborations enabled?
- Is the guest's email domain restricted?
- Is the invitation redirect URL configured correctly?
- Was the email marked as spam?
- Is the guest's email address correct?
- Is the guest's home tenant blocked? (Tenant restrictions in CA)
**Evidence:** Invitation status (Pending), email logs, Cross-Tenant Access settings.
**Root Cause:** Cross-Tenant Access restricted, email blocked, or incorrect email address.
**Fix:** Adjust Cross-Tenant Access settings, verify email, resend invitation, check spam folders.
**Validation:** Guest receives and redeems invitation.

### Scenario 5: "User was removed from a licensed group but still has license for 30 days"
**Symptoms:** User no longer in group, but still shows as licensed.
**Architecture:** Group-based licensing → Entra ID license propagation → License removal process.
**Dependencies:** Group-based licensing process, propagation timing, license availability.
**Checks:**
- Is the user still in the group? (Verify group membership)
- How long since removal? (License removal takes time — up to 30 days in some cases, but usually 24-48 hours)
- Is the license still showing in user's profile?
- Is there a separate direct license assignment?
**Evidence:** User license info, Group membership timeline, Audit log for license changes.
**Root Cause:** Propagation delay, or separate license assignment.
**Fix:** Wait for propagation, or check for direct license assignment (remove if unintended).
**Validation:** License removed from user, cost analysis reflects change.

---

## 3.19 KNOWLEDGE TEST

1. **What is the difference between a tenant, a directory, and a domain?**
   Tenant = trust boundary with Microsoft (GUID). Directory = data store holding identity objects. Domain = verified DNS name for sign-in. In practice, one tenant has one directory and one or more domains.

2. **What is the difference between App Registration, Enterprise Application, and Service Principal?**
   App Registration = definition of app (Client ID, permissions). Enterprise Application = instance of the app in your tenant (usage, consent). Service Principal = runtime identity of the app (used for authentication). Three distinct objects.

3. **What is the difference between Application and Delegated API permissions?**
   Application permissions: app acts as itself (Service Principal), no user involved. Delegated permissions: app acts on behalf of a signed-in user. Application permissions require admin consent.

4. **Why would a user with Global Administrator still not be able to create a VM?**
   Global Admin is a directory role (Entra ID), not an Azure RBAC role. Unless they've been assigned Owner/Contributor at subscription/RG scope, they cannot create Azure resources.

5. **What happens to RBAC permissions when a user is removed from a group?**
   They are removed — but propagation can take up to 15 minutes. During that time, the user may still have access.

6. **What is the 200 group claim limit?**
   Tokens include group Object IDs, but only up to 200. If a user has more groups, claims are replaced with an encoded "strong claim," and the application must query Microsoft Graph API for the actual membership.

7. **Why is Dynamic Group membership requiring Entra ID P1/P2?**
   Dynamic membership rules and group-based licensing are premium features requiring a higher SKU.

8. **What is the difference between blocking sign-in and deleting a user?**
   Blocked sign-in: account exists, cannot authenticate, data preserved, easily reversible. Deleted: account removed (soft delete → hard delete), recovery window 30 days, more destructive.

9. **What is consent in the context of app permissions?**
   Consent is the user's or admin's agreement to allow an app to access specific data/APIs. Without consent, the app cannot use the requested permission.

10. **What is a Service Principal?**
    A Service Principal is the identity (account) created for an App Registration within a tenant, used for authentication when the app runs. It has its own Object ID and is a type of security principal in Entra ID.

11. **What is a Managed Identity?**
    A Managed Identity is a Service Principal that Azure automatically manages. It's tied to an Azure resource and requires no credential management. Two types: System-assigned (resource lifecycle) and User-assigned (independent).

12. **What are the two types of credentials for an App Registration?**
    Client secrets (strings, like passwords) and certificates (X.509 key pairs). Both can expire and must be rotated.

13. **What is SSPR?**
    Self-Service Password Reset — allows users to reset their own passwords using registered authentication methods, without admin intervention.

14. **What is the difference between PTA and Password Hash Sync?**
    PTA (Pass-through Authentication): On-prem AD validates passwords in real-time, Entra ID writes back password changes. PHS (Password Hash Sync): Entra ID syncs a hash of the on-prem password, allows cloud-only password changes. Both are methods for hybrid identity password management.

15. **What is the significance of the Tenant ID?**
    It uniquely identifies the Entra tenant in all API calls, tokens (as the `tid` claim), and configurations. It's a GUID that never changes.

---

## 3.20 L3 GAP CHECK

| Topic | Status |
|-------|--------|
| Tenant concept and properties | ✅ Covered |
| Directory concept and properties | ✅ Covered |
| Domains and UPN | ✅ Covered |
| Tenant ID vs Object ID | ✅ Covered |
| Directory Roles vs Azure RBAC | ✅ Covered (Critical distinction) |
| Member users | ✅ Covered |
| Guest users | ✅ Covered |
| User properties and lifecycle | ✅ Covered |
| Password management methods | ✅ Covered |
| Authentication Methods | ✅ Covered |
| SSPR | ✅ Covered |
| Account disable/enable | ✅ Covered |
| Deleted users and recovery | ✅ Covered |
| Groups (Security, M365) | ✅ Covered |
| Dynamic membership | ✅ Covered |
| Group-based licensing | ✅ Covered |
| Nested group limitations | ✅ Covered |
| Group ownership | ✅ Covered |
| App Registration | ✅ Covered |
| Enterprise Application | ✅ Covered |
| Service Principal | ✅ Covered |
| App Registration vs EE vs SP (Critical) | ✅ Covered (Deep) |
| Certificates & Secrets | ✅ Covered |
| OAuth 2.0 / OpenID Connect | ✅ Covered |
| SAML (where relevant) | ✅ Covered |
| Consent (User vs Admin) | ✅ Covered |
| Application vs Delegated permissions | ✅ Covered |
| Managed Identities (System/User-assigned) | ✅ Covered |
| Token lifecycle | ✅ Covered |
| Sign-in flows | ✅ Covered |
| Entra ID administration | ✅ Covered |
| Security considerations | ✅ Covered |
| Monitoring (Sign-in logs, Audit logs, Risk) | ✅ Covered |
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

## 3.21 WHAT AN EXPERIENCED AZURE L3 ENGINEER SHOULD NOW BE ABLE TO EXPLAIN CONFIDENTLY

After Module 3, you should be able to confidently explain:

1. **The complete Azure identity hierarchy** — Tenant → Directory → Users/Groups/Apps → Service Principals → Managed Identities — and how each layer relates to authentication, authorization, and resource access.

2. **The critical distinction between Directory Roles and Azure RBAC roles** — Directory Roles (e.g., Global Admin, Conditional Access Administrator) govern Entra ID administration; Azure RBAC roles (Owner, Contributor, Reader) govern resource management. They are independent systems.

3. **The three-way distinction between App Registration, Enterprise Application, and Service Principal** — and why each must be checked independently during troubleshooting. They are NOT the same thing.

4. **How a sign-in actually flows** — from the user entering credentials, through Entra ID authentication, Conditional Access evaluation, token issuance, and resource access. Every step can fail independently.

5. **The difference between Application and Delegated permissions** — and why Application permissions require admin consent while Delegated permissions may or may not.

6. **How managed identities work** — System-assigned (tied to resource lifecycle) vs User-assigned (independent), and why System-assigned identities lose their role assignments when resources are recreated.

7. **How group-based licensing works** — licenses assigned to groups, dynamic membership for auto-assignment, propagation delays, and why a user might retain a license after group removal.

8. **The 200 group claim limit in tokens** — and why users with many groups may lose group-based access in applications that don't query Graph API.

9. **Why Conditional Access may not apply even when it looks correct** — Report Only mode, exclusions (including service principals and break-glass accounts), policy precedence, authentication strength mismatch, guest/user-type exclusions, legacy authentication.

10. **How to read a sign-in log** — extracting the actual failure reason, applied CA policies, risk level, authentication method, and conditionalAccessStatus field to determine root cause.

11. **How user lifecycle (disable vs delete vs recover) affects running applications** — disabled users retain running sessions (in many cases), deleted users eventually lose access, recovered users regain original permissions.

12. **Why secret/certificate expiry is the #1 cause of Service Principal failures** — and how to proactively monitor and rotate credentials.

---

# Ready for Module 4 — Entra ID Authentication (Tokens, MFA, Conditional Access Pre-Authentication, Sign-in Process)?

It will cover:
- Authentication vs Authorization
- Authentication Methods (MFA, Passwordless, FIDO2, Authenticator, TAP)
- Token concepts (Access, Refresh, ID tokens, lifetimes, renewal)
- Sign-in process (complete step-by-step)
- Conditional Access (users, groups, roles, conditions, grant controls, session controls, exclusions, break-glass, risk-based, legacy auth)
- And all 19 required module components

Say **"Next module"** to continue.