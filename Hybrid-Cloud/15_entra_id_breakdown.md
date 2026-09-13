# 15. MICROSOFT ENTRA ID — Point-by-Point Breakdown
### Format for every item: **What is it? → What is it used for? → What issues can it cause (if broken/misconfigured)?**

---

## * Tenant

**What is it?**
Simple: Your organization's own dedicated, isolated "instance" of Entra ID — nobody outside your org shares it.
Technical: A dedicated Entra ID instance tied to a unique tenant ID (GUID) and one or more verified domains, representing the top-level security and administrative boundary for your organization.

**What is it used for?**
Every Azure subscription and M365 license is tied to exactly one tenant — it's the container for all your users, groups, apps, and policies. Multi-tenant orgs (post-acquisition, or intentionally separated business units) each have completely separate identity boundaries unless explicitly federated/configured for B2B.

**What issues can it cause?**
- **Multiple tenants from mergers/acquisitions** — creates identity fragmentation; users can't access resources across tenants without B2B guest invitations or a tenant consolidation project (a very real, very painful enterprise problem).
- **Wrong tenant selected during sign-in** (common with personal + work accounts, or consultants working across multiple client tenants) — user sees "access denied" and assumes it's a permissions issue when it's actually a tenant-context issue.
- **Tenant-level settings misapplied** (e.g., external collaboration settings) affect EVERY user/app in that tenant — a single wrong toggle has organization-wide blast radius.

---

## * Users

**What is it?**
An identity object in Entra ID representing a person (or in some cases a service). Three types matter for troubleshooting: **cloud-only** (created directly in Entra ID), **synced** (from on-prem AD via Entra Connect — most attributes not editable in cloud portal), **guest/B2B** (external user invited in, different default permissions).

**What is it used for?**
The core object that authentication, licensing, group membership, and access decisions are built around.

**What issues can it cause?**
- **Trying to edit a synced user's attribute directly in the cloud portal** — fails or gets silently overwritten on next sync, because on-prem AD is the source of truth for synced attributes — a very common admin mistake.
- **Guest users retaining access longer than intended** — if there's no access review process, external guests can accumulate long after a project ends (a common audit finding).
- **Licensing not assigned** — user exists and can sign in, but has no mailbox/Teams/OneDrive because no license SKU is assigned — looks like "broken account" but is a licensing gap.

---

## * Groups

**What is it?**
Security groups (access control), Microsoft 365 groups (collaboration — backs Teams/SharePoint/Planner), and **Dynamic groups** (membership rule-based, e.g., `user.department -eq "Sales"`, evaluated automatically).

**What is it used for?**
Scoping access, licenses, Conditional Access policies, and Intune/app assignments — groups are the primary unit of "who gets what" at scale instead of assigning things per-user.

**What issues can it cause?**
- **Dynamic group processing delay** — rule changes aren't instant; large tenants can see membership updates lag by minutes to hours, which matters if you're troubleshooting "why hasn't this new hire gotten access yet."
- **Nested group depth/complexity** — Conditional Access and Intune generally DO evaluate nested groups, but excessive nesting makes troubleshooting "why does this user have/not have access" much harder to trace.
- **Group-based licensing conflicts** — if a user is in two groups assigning conflicting/duplicate licenses, or a group runs out of available licenses, some users silently don't get provisioned — visible only by checking the "Licenses" > "Processing errors" blade for that group.

---

## * Administrative Units (AUs)

**What is it?**
A way to delegate admin permissions over a SUBSET of the tenant (e.g., "the Sales department's users and groups only") rather than tenant-wide — restricts scope of a Helpdesk Admin or User Admin role to just that AU.

**What is it used for?**
Decentralized IT administration in large orgs (e.g., a school district letting each school manage its own users) without giving anyone tenant-wide rights.

**What issues can it cause?**
- **A user/object not added to the correct AU** — a delegated admin can't manage them even though the delegated admin's role would normally allow it — looks like a permissions bug but is actually a scoping gap.
- **Overlapping AUs and confusion about which admin has authority over a given object** in complex org structures.

---

## * Roles (Entra ID RBAC — distinct from Azure RBAC)

**What is it?**
Built-in or custom roles (Global Administrator, User Administrator, Helpdesk Administrator, Application Administrator, etc.) governing what a person can DO to tenant-level/identity resources.

**What is it used for?**
Least-privilege delegation — e.g., a helpdesk team gets "Helpdesk Administrator" (can reset non-admin passwords) rather than Global Admin, reducing blast radius if that account is compromised.

**What issues can it cause?**
- **Confusing Entra ID roles with Azure RBAC roles** (a very common real-world and interview mistake) — Global Admin does NOT automatically grant control over Azure resources (VMs, storage) unless explicitly elevated via the "Access management for Azure resources" toggle at the tenant root scope.
- **Over-provisioned Global Admin accounts** — a common audit finding and major security risk; best practice is a small number of Global Admins, most delegated admin work done via scoped roles (ideally via Privileged Identity Management/PIM for just-in-time elevation rather than standing access).
- **Role assignment doesn't propagate instantly** in rare cases — cache/token refresh delay can make a just-assigned role appear "not working" for a few minutes.

---

## * Authentication (Methods)

**What is it?**
The mechanisms a user can register and use to prove identity: password, Microsoft Authenticator (push/passwordless), FIDO2 security keys, Windows Hello for Business, OATH hardware tokens, SMS/voice call (being deprecated/discouraged due to SIM-swap risk).

**What is it used for?**
Layered proof of identity — moving away from password-only, and increasingly away from SMS specifically toward phishing-resistant methods (FIDO2, certificate-based auth).

**What issues can it cause?**
- **User has no registered method and CA requires MFA** — complete lockout at first login unless a registration grace period or Temporary Access Pass (TAP) is used for onboarding.
- **SMS/voice methods are vulnerable to SIM-swap attacks** — a known weakness; orgs increasingly disable these as an allowed method for exactly this reason, which can surprise users used to relying on them.
- **Authentication method policy conflicts** — an admin disables a method org-wide (e.g., turns off SMS) without checking who relies on it as their ONLY registered method — instant lockout for that population.

---

## * MFA (Multi-Factor Authentication)

**What is it?**
Requiring a second proof of identity beyond password — can be enforced via Security Defaults (free, all-or-nothing), per-user MFA (legacy, being phased out), or (recommended) Conditional Access-based MFA (flexible, conditional).

**What is it used for?**
Blocks the vast majority of automated credential-based attacks (password spray, credential stuffing, phishing) even if the password itself is compromised.

**What issues can it cause?**
- **MFA fatigue attacks** — attacker with a stolen password spams push notifications hoping the user accidentally approves one — mitigated by number-matching (Authenticator now requires entering a displayed number, not just tap-to-approve) — a good proactive answer to mention.
- **Legacy per-user MFA conflicting with Conditional Access MFA policies** — can create confusing, inconsistent enforcement; Microsoft recommends migrating fully to CA-based MFA and disabling legacy per-user MFA.
- **No backup method registered** — single point of failure if the primary device (phone) is lost.

---

## * SSPR (Self-Service Password Reset)

**What is it?**
Lets users reset their own forgotten password using registered authentication methods, without a helpdesk ticket. Uses the same **Combined Registration** experience as MFA now.

**What is it used for?**
Reduces helpdesk ticket volume (password reset is consistently one of the top helpdesk ticket categories in every enterprise) and speeds up user self-recovery.

**What issues can it cause?**
- **Password Writeback not enabled (hybrid users)** — cloud password resets but the change doesn't flow back to on-prem AD — user can log into M365 with the new password but not their domain-joined PC, a classic and confusing hybrid support ticket.
- **Registration not enforced/completed** — a user who never completed SSPR registration can't self-serve when they actually need it — best practice is a registration campaign/enforcement policy at onboarding, not leaving it optional indefinitely.
- **SSPR policy requiring 2 methods but user only registered 1** — locked out of SSPR itself, ironically needing helpdesk anyway.

---

## * Conditional Access

*(Covered in full depth in file 03 — Named Locations, Grant/Session controls, Report-only mode, break-glass exclusions, What-If tool. This entry is the summary cross-reference.)*

**What is it?** The policy engine deciding grant/block/challenge based on real-time signals after primary authentication.
**What is it used for?** Zero Trust enforcement — replacing "on the network = trusted" with signal-based, continuously evaluated access decisions.
**What issues can it cause?** Lockouts from missing break-glass exclusions, over-reliance on network location as a sole signal, policy conflicts (Block always wins over Grant), and legitimate access blocked when a policy is enforced without Report-only testing first.

---

## * Identity Protection

**What is it?**
An Entra ID P2 feature providing **risk-based** signals — **User risk** (credentials likely compromised — e.g., found in a breach dataset, leaked credentials detected) and **Sign-in risk** (this specific sign-in attempt looks suspicious — impossible travel, anonymous IP/Tor, unfamiliar sign-in properties) — calculated via Microsoft's threat intelligence and ML.

**What is it used for?**
Feeds risk signals into Conditional Access as a condition (e.g., "if sign-in risk is Medium or High, require MFA; if High, block entirely") — dynamic, adaptive security rather than static rules alone.

**What issues can it cause?**
- **False positives from legitimate but unusual travel** — a genuinely traveling executive gets flagged as "impossible travel" and challenged/blocked, generating support tickets and user frustration if not tuned or explained.
- **Risk remediation not automated** — a user flagged as "High risk" stays blocked/limited until an admin or self-service password reset clears the risk state; if nobody monitors the risky users report, accounts can remain in a degraded state unnoticed.
- **Requires P2 licensing** — orgs on P1 don't have this at all and may not realize their MFA/CA setup lacks the risk-based adaptive layer entirely (a licensing-driven capability gap worth knowing to mention in a governance discussion).

---

## END-TO-END FLOW: New employee onboarding through Entra ID

```
HR system creates record → provisioning (manual or automated via Entra ID Governance/HR-driven provisioning)
   ↓
User object created (cloud-only or synced from on-prem AD)
   ↓
Added to Dynamic or Assigned Groups (department, location, role-based)
   ↓
License assigned (directly or via group-based licensing)
   ↓
MFA/SSPR registration required at first sign-in (Combined Registration)
   ↓
Conditional Access evaluates sign-in (device, location, risk)
   ↓
Access granted to M365, Azure resources, apps per group/role assignment
```

## BREAK/FIX MASTER TABLE

| Component | Purpose | If it breaks/misconfigured | Symptom | First Check | Fix |
|---|---|---|---|---|---|
| Tenant settings | Org-wide identity boundary | Wrong external collaboration setting | Guests can't be invited / too permissive | Entra ID > External Identities settings | Correct tenant-wide policy |
| Dynamic Groups | Auto membership | Rule error or processing delay | New hire missing access | Check group's "Dynamic membership rules" + processing status | Fix rule syntax, wait for processing or force refresh |
| Roles (Entra ID) | Delegated admin scope | Confused with Azure RBAC | Admin can't manage Azure resources despite Global Admin | Check "Access management for Azure resources" toggle | Elevate explicitly if intended, or assign correct Azure RBAC role |
| MFA | Second-factor proof | No registered method + CA requires MFA | Total lockout at first login | Check user's registered methods | Issue Temporary Access Pass (TAP) |
| SSPR | Self-service reset | Password writeback disabled | Cloud password works, on-prem doesn't | Check Entra Connect writeback setting | Enable writeback, re-sync |
| Identity Protection | Risk signals | False positive on legitimate travel | User blocked/challenged unexpectedly | Check Risky sign-ins report | Dismiss risk after verification, tune policy |

## TOP INTERVIEW QUESTIONS

🔴 What's the difference between Entra ID roles and Azure RBAC roles — walk through a scenario where confusing them causes a problem.
🔴 A new hire isn't getting expected group-based access — how do you troubleshoot?
🔴 What is Identity Protection and how does risk feed into Conditional Access?
🟠 What is Password Writeback and what breaks without it in a hybrid environment?
🟠 What's the difference between Security Defaults, per-user MFA, and Conditional Access MFA — which would you recommend and why?
🟠 What is an Administrative Unit and when would you use one?
🟡 What is Temporary Access Pass used for?
🟡 Why is number-matching in Authenticator an improvement over simple push-approve?
🔴 Scenario: a Global Admin says they can't manage Azure VMs — explain why and how to fix it.
🟠 Scenario: an executive is repeatedly flagged as "impossible travel" — how do you handle it operationally?

## L3 ANSWER — Entra ID Roles vs Azure RBAC (60-90s)
"This is a distinction I make sure stakeholders understand early, because it's a common source of confusion. Entra ID roles like Global Administrator control identity and tenant-level management — users, licenses, tenant settings. Azure RBAC — Owner, Contributor, Reader — is a completely separate permission model that controls access to actual Azure resources like VMs and storage, assigned at management group, subscription, resource group, or resource scope. Being a Global Admin doesn't automatically give you rights over Azure resources unless someone has explicitly elevated Global Admin access to Azure resources at the tenant root scope — which is itself a privileged, auditable action I'd expect to see logged and justified, not left on by default."

## MEMORY TRICK
**Entra ID roles = manage IDENTITY. Azure RBAC = manage RESOURCES. Two separate systems, don't assume one grants the other.**
**MFA fatigue defense = number-matching, not just push-approve.**
**Hybrid SSPR gotcha = Password Writeback — cloud reset without it never reaches on-prem AD.**

## 10-SECOND REVISION
- Tenant = your isolated identity boundary; one tenant per org identity domain
- Entra ID roles ≠ Azure RBAC roles — commonly confused, know the distinction cold
- Dynamic groups have processing delay — not instant
- SSPR needs Password Writeback for hybrid users, or cloud/on-prem passwords diverge
- Identity Protection (P2) = risk-based signals feeding Conditional Access — requires proper licensing
