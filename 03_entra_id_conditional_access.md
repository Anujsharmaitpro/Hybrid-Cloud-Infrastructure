# 3. MICROSOFT ENTRA ID — INCLUDING CONDITIONAL ACCESS (PRIORITY — this was your weakest area)

## 3.1 Tenants, Users, Groups — quick foundation

**Tenant:** A dedicated, isolated instance of Entra ID for an organization — the top-level security boundary. Every subscription (Azure, M365) is associated with exactly one tenant (though a tenant can have many subscriptions).

**Users:** Cloud-only, synced-from-AD, or guest (B2B) — the type matters because synced users can't have their password changed in the cloud portal, and guests have different Conditional Access considerations.

**Groups:** Security groups (access control) vs Microsoft 365 groups (collaboration, Teams/SharePoint backing) vs Dynamic groups (rule-based membership, e.g., `user.department -eq "Sales"`). Dynamic groups are commonly used to auto-scope Conditional Access and Intune policies.

## 3.2 Roles (RBAC in Entra ID vs Azure RBAC — commonly confused, common interview trap)

**Entra ID roles** (Global Admin, User Admin, etc.) control management of identity/tenant-level resources (users, licenses, tenant settings).
**Azure RBAC roles** (Owner, Contributor, Reader) control access to Azure resources (VMs, storage, resource groups) — completely separate permission model, assigned at Management Group/Subscription/RG/Resource scope.

**Interview trap:** "If I make someone a Global Admin, can they manage all Azure VMs?" — **No**, not by default. Global Admin can elevate themselves to Azure "User Access Administrator" at root scope via a specific toggle, but the two RBAC systems are otherwise separate.

## 3.3 Authentication Methods & MFA

**What is it?** Multiple methods to prove identity beyond password: Microsoft Authenticator (push/passwordless), OATH hardware tokens, SMS, phone call, FIDO2 security keys, Windows Hello for Business.

**Why used:** Password alone is the #1 compromise vector (phishing, breach reuse). MFA blocks the vast majority of automated account-takeover attempts.

**How it works:** Primary auth (password/PHS/PTA) succeeds → Entra ID evaluates Conditional Access → if MFA required → secondary challenge via registered method → token issued only after both succeed.

**Dependency:** User must have registered a method (via SSPR/MFA registration campaign) — if not registered and CA blocks access without registration, user is locked out entirely (a very real onboarding trap).

**Failure scenario:** User loses phone with Authenticator registered, no backup method — locked out. Admin must use **temporary access pass (TAP)** or re-register another method after identity verification (help desk process — this is a common interview scenario question).

## 3.4 SSPR (Self-Service Password Reset)

**What:** Lets users reset their own password using registered methods without a help desk ticket. Requires **Combined Registration** with MFA (same registration portal now for both).
**Dependency:** For hybrid users, requires **password writeback** enabled in Entra Connect so the reset actually flows back to on-prem AD.
**Failure scenario:** Writeback misconfigured → cloud password resets but on-prem doesn't match → user can log into M365 but not domain-joined PC — classic hybrid gotcha, good scenario question.

## 3.5 CONDITIONAL ACCESS — full breakdown (this is where you need the most depth)

**What is it?**
Simple: "If-this-then-that" rules engine for sign-ins — e.g., "if user is external AND app is SharePoint, then require MFA."
Technical: A policy engine evaluated after primary authentication succeeds but before token issuance, combining **assignments** (who/what/where) with **access controls** (grant/block/session).

**Why used:** Enforces Zero Trust — access decisions based on real-time signals (user, device, location, risk) rather than network location alone (the old VPN-perimeter model).

**How it works — evaluation flow:**
User attempts sign-in → primary auth (password/PHS/PTA/Federation) → **CA policy evaluation** (all applicable policies evaluated together, most restrictive wins) → grant controls checked (MFA satisfied? Compliant device? App protection policy?) → session controls applied (sign-in frequency, app-enforced restrictions) → token issued or blocked.

### 3.5.1 Assignments (the "who/what/when")

- **Users and groups** — include/exclude (always exclude at least one break-glass account from ANY CA policy — critical safety practice, huge interview point)
- **Cloud apps** — scope to specific apps (e.g., SharePoint Online) or "All cloud apps"
- **Conditions:**
  - **Sign-in risk / User risk** (requires Entra ID Protection / P2) — real-time risk scoring from ML (impossible travel, leaked credentials, anonymous IP)
  - **Device platforms** (iOS, Android, Windows, etc.)
  - **Locations** — **Named/Trusted Locations** (IP ranges or countries) — THIS is the piece you got wrong in the mock interview
  - **Client apps** — browser vs mobile apps vs legacy auth (blocking legacy auth here is a very common, very important real-world policy)

### 3.5.2 Named/Trusted Locations — full correction of your earlier answer

**What it actually is:** A named IP range or set of countries you define in Entra ID (Protection > Named Locations). You can mark one as "Trusted" specifically to represent your corporate network egress IPs.

**How you'd actually design your Q3 scenario correctly:**
- Create a Named Location = your corporate office public IP range(s), mark it Trusted.
- Policy 1: **Assign to** Employee group, **Exclude** Contractor group → Condition: Location = "Not in" Trusted Location → Grant: Require MFA. (So employees off-network need MFA; on-network, this specific policy doesn't fire.)
- Policy 2: **Assign to** Contractor group, **all locations** (no location exclusion) → Grant: Require MFA always, regardless of network.
- Optionally, layer in **device compliance** (via Intune) as an additional grant control for employees even on the trusted network, since IP-based trust alone is weak.

**Real risks (what I should have said instead of what I said in the mock):**
- IP-based trust doesn't verify the actual device or user session — a compromised device or credential-stuffed session **on** the trusted network still gets in without MFA under a network-only policy.
- Corporate NAT/VPN split-tunneling can make an attacker's traffic appear to originate from the trusted range.
- This is why Microsoft's own best practice is to treat named locations as ONE signal, not a sufficient control alone — pair with device compliance or sign-in risk for real defense-in-depth.

### 3.5.3 Grant Controls
Require MFA, Require compliant device (Intune), Require hybrid Azure AD joined device, Require approved client app, Require app protection policy, **Block access** (highest severity), Terms of Use acceptance. Multiple controls can be combined with AND/OR logic (require ALL selected controls vs require ONE of selected controls).

### 3.5.4 Session Controls
- **Sign-in frequency** — force re-auth after N hours (useful for high-risk apps)
- **Persistent browser session** — allow/prevent "stay signed in"
- **App-enforced restrictions** — e.g., SharePoint browser-only limited access for unmanaged devices (view-only, no download)
- **Continuous Access Evaluation (CAE)** — near real-time token revocation when a critical event occurs (user disabled, password changed, location risk) — newer, high-value L3 answer, shows you're current.

### 3.5.5 Authentication Strength (newer, replaces some older MFA-method-specific policies)
Lets you require specific *strength* of MFA method (e.g., phishing-resistant methods like FIDO2/Windows Hello only) rather than "any MFA method" — used for high-privilege or high-risk access scenarios.

### 3.5.6 Report-only mode
**Critical practice:** Every new CA policy should be deployed in **Report-only** mode first — lets you see who WOULD be blocked/challenged in the sign-in logs, without actually enforcing, before going live. This is a strong interview answer for "how do you safely roll out a new CA policy without breaking access."

## FAILURE SCENARIOS — Conditional Access

**Scenario 1:** New CA policy locks out the whole org, including admins. **Root cause:** No break-glass account excluded; misconfigured "All users, All apps, Block." **Prevention:** Always exclude at least 2 break-glass (emergency access) accounts with strong, monitored, non-MFA-required-but-heavily-audited credentials, stored securely, from EVERY CA policy.

**Scenario 2:** External contractor can access SharePoint without MFA despite policy. **Root cause:** Contractor group not correctly scoped (nested group not evaluated, or a conflicting policy has a higher-priority exclude).

**Scenario 3:** Legitimate users suddenly blocked after a "trusted location" IP range change (ISP re-assigned corporate public IP) — Named Location wasn't updated.

## TROUBLESHOOTING METHODOLOGY

1. Understand: is it a specific user, app, or location that's affected?
2. Use **Sign-in logs** in Entra ID — filter by user/app/date — check the "Conditional Access" tab on the specific sign-in event to see EXACTLY which policies applied and whether each was Success/Failure/Not Applied.
3. Use the built-in **"What If" tool** (Entra ID > Conditional Access > What If) — simulate a specific user/app/condition combination to see which policies would apply, without waiting for a real sign-in.
4. Check for **policy conflicts** — multiple policies with overlapping scope, one might Block while another Grants (Block always wins in CA evaluation).
5. Check group membership/nesting — CA does evaluate nested group membership, but propagation can lag slightly after group changes.
6. Validate fix using the What-If tool AND a real test sign-in from a test account matching the affected profile.

**Key commands/tools:**
```
Entra admin center → Sign-in logs → filter → Conditional Access tab
Entra admin center → Conditional Access → What If tool
Get-MgIdentityConditionalAccessPolicy   (Graph PowerShell)
```

## INTERVIEW QUESTIONS

🔴 Walk me through exactly how Conditional Access evaluates when a user signs in.
🔴 What's the difference between Grant controls and Session controls?
🔴 Why is a break-glass account exclusion mandatory on every CA policy?
🔴 What are Named/Trusted Locations and what are their real limitations?
🟠 What is Report-only mode and why use it before enforcing?
🟠 What is Continuous Access Evaluation and why does it matter?
🟠 How do you troubleshoot why a specific user was blocked by CA?
🟡 What's the difference between requiring MFA vs requiring Authentication Strength?
🟡 How does risk-based CA (Identity Protection) differ from static condition-based CA?
🔴 Scenario: New CA policy accidentally locks out all admins — how do you recover, and how do you prevent it next time?
🔴 Scenario: Contractors are getting into SharePoint without MFA — diagnose.
🟠 Scenario: employees on the "trusted" office network still get MFA-prompted unexpectedly — why might that happen?

## RED FLAGS / TRICK QUESTIONS
- "Does Conditional Access work if MFA isn't licensed?" — CA is a P1/P2 feature; MFA itself (Security Defaults) is free tier, but full CA rules engine requires Entra ID P1 minimum.
- "Is a Named Location a security control by itself?" — No — it's a signal, not proof of identity or device trust; should be combined with other controls.
- "Does blocking legacy auth break modern Outlook/Teams?" — No, modern auth clients aren't affected; it blocks POP/IMAP/older basic-auth protocols, which is exactly why blocking it is a standard, low-risk-high-value CA policy.

## IDEAL ANSWER (30-60s)
"Conditional Access is Entra ID's policy engine that evaluates real-time signals — user, group, app, device, location, risk — after primary authentication, and decides whether to grant, block, or challenge the sign-in with something like MFA or a compliant-device requirement. It's the core Zero Trust control in a modern tenant, replacing the old assumption that being on the corporate network is itself proof of trust."

## L3 ANSWER (60-120s)
"When I design Conditional Access, my starting principle is that no single signal — including network location — should be the sole determinant of access. For something like scoping MFA differently for employees vs contractors, I'd build a Named Location for the corporate egress IPs, then run two policies: one for the employee group excluding that trusted location so off-network access needs MFA, and one for contractors requiring MFA unconditionally regardless of location. But I'd flag to the business that IP-based trust alone doesn't verify the device or session — split-tunnel VPNs or a compromised device on that network would bypass the location signal — so wherever possible I pair it with device compliance via Intune as a second grant control. Operationally, I never deploy a new policy live first — always Report-only mode initially, checked against the sign-in logs, and I always make sure at least two break-glass accounts are excluded from every policy before it goes live, because a misconfigured 'block all' policy locking out your own admins is one of the most common real-world CA incidents."

## MEMORY TRICK
**CA = WHO + WHAT + WHERE → GRANT/BLOCK + SESSION**
Location = one signal, not a lock. Always exclude break-glass. Always test in Report-only first.

## 10-SECOND REVISION
- CA evaluates AFTER primary auth, before token issuance
- Named location = signal, not proof — pair with device compliance
- ALWAYS exclude break-glass accounts from every policy
- Sign-in logs → Conditional Access tab = your #1 troubleshooting tool
- Report-only mode before going live, always

## RELATED TECHNOLOGIES
Conditional Access → Entra ID → MFA → Device Compliance (Intune) → Identity Protection (risk signals) → Authentication Strength → Named Locations → Continuous Access Evaluation
