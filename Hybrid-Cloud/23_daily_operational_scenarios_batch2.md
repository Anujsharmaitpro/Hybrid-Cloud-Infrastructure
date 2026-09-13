# 23. DAY-TO-DAY OPERATIONAL SCENARIOS — BATCH 2 (Additional Scenarios)
### Format: Scenario → Short Answer (interview-ready, no duplicates from file 21)

---

# ACTIVE DIRECTORY / DNS / DHCP / GPO

**Q: A user's account keeps locking out every few minutes for no obvious reason.**
A: Check Event ID 4740 on the DC that processed the lockout to find the source computer/process, then check for a cached credential on that device (mapped drive, scheduled task, mobile device with old password) still retrying with the old password.

**Q: A newly promoted server to Domain Controller isn't showing up in AD Sites and Services under the right site.**
A: Check the subnet-to-site mapping in AD Sites and Services — if the DC's IP subnet isn't associated with the intended site object, it gets placed in Default-First-Site-Name instead.

**Q: You need to safely decommission an old Domain Controller.**
A: Run `dcpromo`/`Uninstall-ADDSDomainController` properly (never just power it off), verify FSMO roles aren't held by it first, confirm DNS/AD cleanup afterward (`repadmin /removelingeringobjects` if needed), remove stale DNS records.

**Q: A GPO deployment needs to roll out a new security setting to 5,000 machines safely.**
A: Deploy to a small pilot OU first, monitor via `gpresult`/Group Policy Operational log for a few days, then progressively widen the scope — never link a new GPO directly to the top-level domain OU for an untested setting.

**Q: DHCP scope shows plenty of free addresses, but users still can't get an IP.**
A: Check DHCP server service status itself, check if the scope is actually activated (not just configured), and check for a rogue/unauthorized DHCP server on the network responding first with bad leases.

**Q: Someone asks why AD doesn't just use flat DNS instead of SRV records.**
A: SRV records let clients discover role-specific services (which servers are DCs, GCs, or PDC emulators) and their relative site cost/priority — a simple A record can't convey "prefer this DC over that one based on site."

**Q: A remote office on a slow WAN link has terrible logon times specifically after a new large software-deployment GPO was added.**
A: Slow-link detection should skip Software Installation CSE automatically on detected slow connections — check if slow-link detection is disabled/misconfigured, or increase policy processing timeout rather than forcing full deployment over a poor link.

---

# HYBRID IDENTITY

**Q: You're asked to migrate from AD FS federation to Password Hash Sync with minimal disruption.**
A: Run PHS in parallel (staged rollout using Entra Connect's staged rollout feature) against a pilot group first, validate Conditional Access/MFA still function correctly, then cut over fully and decommission the AD FS farm once confidence is established.

**Q: A merger means two companies' AD forests need users to access shared resources.**
A: Options: Entra ID B2B guest invitations (lightest touch), cross-forest trust (heavier, more traditional), or full tenant/identity consolidation (most disruptive, most complete) — the right choice depends on the timeline and how permanent the merger integration is.

**Q: Seamless SSO stopped working after a domain rename.**
A: The AZUREADSSOACC computer object may be orphaned/misconfigured after the rename — needs to be reset/reconfigured, since it's tied to the specific domain's Kerberos configuration.

---

# ENTRA ID / CONDITIONAL ACCESS

**Q: You need to enforce phishing-resistant MFA for all IT admins specifically.**
A: Use Authentication Strength in Conditional Access scoped to admin roles, requiring FIDO2/Windows Hello for Business specifically rather than "any MFA method" — regular MFA (SMS/push) doesn't satisfy the phishing-resistant requirement.

**Q: A dynamic group meant to auto-add all Sales department users to a CA policy isn't updating fast enough for new hires.**
A: This is expected — dynamic group membership evaluation isn't instant, it can lag; for time-sensitive onboarding, consider a hybrid approach (assigned group for immediate access, dynamic group for longer-term maintenance).

**Q: Leadership wants guest/external users to have stricter access than employees automatically, without manual per-guest configuration.**
A: Build a Conditional Access policy scoped to "Guest or external users" (a built-in user type condition) rather than manually managing a guest group — applies uniformly to any B2B guest regardless of when they were invited.

**Q: You're asked to justify why Security Defaults isn't sufficient for the organization anymore.**
A: Security Defaults is an all-or-nothing, one-size-fits-all MFA enforcement with no granularity — no ability to scope by app, location, device compliance, or risk; Conditional Access (P1+) is required for anything beyond the most basic universal MFA requirement.

---

# PKI / CERTIFICATES

**Q: You need to plan a Root CA key rollover (the Root cert itself is approaching expiry in 2 years).**
A: This needs long lead time — plan the new Root CA hierarchy, cross-sign or dual-issue during a transition period, and ensure the new Root is distributed to ALL clients (via GPO/Intune) well before the old Root actually expires, not after.

**Q: A new internal application needs a certificate that supports both server and client authentication.**
A: Create/use a Certificate Template with BOTH Server Authentication and Client Authentication in the Extended Key Usage field — a template built for only one EKU will fail validation for the other purpose even if issued successfully.

**Q: Security wants to know if your PKI is vulnerable to a known AD CS privilege escalation issue (ESC1-ESC8 style).**
A: Would need to audit certificate template permissions specifically for templates allowing low-privileged users to specify a Subject Alternative Name (a classic ESC1-style vulnerability) — this is a real, named category of AD CS misconfiguration worth being aware of even at a summary level.

---

# INTUNE / AUTOPILOT

**Q: You need to migrate 500 existing (already-deployed, not new) devices into Autopilot management.**
A: Use "Convert to Autopilot device" in Intune (registers the hardware hash from an already-enrolled device) rather than requiring a full wipe/reimage — lets existing devices get Autopilot profile benefits going forward (e.g., for future resets) without disrupting current users.

**Q: A specific app needs to be force-reinstalled on all devices after a bad update went out via Intune.**
A: Update the app's version/detection rule so Intune's detection logic sees the old version as non-compliant, triggering reinstall; or use "Uninstall and reinstall" assignment intent if supported for that app type.

**Q: Company wants BYOD phones to access email but NOT be fully MDM-enrolled.**
A: Use Intune App Protection Policies (MAM without enrollment) — applies data protection (prevent copy/paste out of managed apps, require PIN) to the Outlook/Teams app itself without requiring full device enrollment.

---

# CITRIX

**Q: You need to patch the OS on all VDAs in a Delivery Group without disrupting active users.**
A: Use Citrix's drain mode on a subset of machines (stops new sessions from brokering to them while existing sessions finish), patch the drained machines, bring them back, repeat for the rest — avoids a hard cutover affecting active users.

**Q: A specific published app crashes only when launched by a subset of users.**
A: Check if those users' FSLogix profile has app-specific corrupted settings/cache; also check if those users are hitting resource limits (per-user CPU/memory throttling policies) that others aren't.

**Q: Leadership wants to move some Citrix workloads to Azure Virtual Desktop instead.**
A: A valid hybrid approach many enterprises use is running Citrix's broker/management layer ON TOP of AVD session hosts — gets Azure-native infrastructure benefits while keeping Citrix's more mature HDX/feature set; full migration to native AVD is also possible but requires re-evaluating anything Citrix-specific (like advanced HDX policies) that AVD may not fully replicate yet.

---

# MICROSOFT 365 / EXCHANGE / MAIL SECURITY

**Q: Legal needs to place a specific user's mailbox on hold for an investigation without the user knowing.**
A: Use Litigation Hold or a Microsoft Purview eDiscovery hold — preserves mailbox content (including deleted items) silently without any visible indication to the user.

**Q: A shared mailbox is being used to send phishing internally — who's actually responsible?**
A: Check delegated "Send As"/"Send on Behalf" permissions on the shared mailbox and audit sign-in logs — since shared mailboxes typically don't have their own credentials, this usually traces back to a compromised individual account with delegated access, not the shared mailbox itself.

**Q: You need to block all legacy authentication (POP/IMAP/basic auth) tenant-wide.**
A: Build a Conditional Access policy targeting "legacy authentication clients" as a client app condition with a Block grant control — doesn't affect modern-auth Outlook/Teams/mobile apps, only the older protocols.

**Q: A user says they never received a calendar invite, but the sender says it was sent.**
A: Check Message Trace for the specific invite; also check if a mail flow rule or an Outlook rule on the recipient's side is redirecting/deleting calendar items, and confirm the invite wasn't auto-declined by a conflicting calendar rule.

---

# AZURE

**Q: A dev team wants production-like data in their test subscription — what's the risk and how do you handle it?**
A: Flag data governance/compliance risk (test environments often have weaker security controls) — recommend masked/synthetic data instead, or if real data is required, ensure the test subscription has equivalent security controls (encryption, access restrictions, Azure Policy) as production.

**Q: A resource group needs to be deleted, but deletion keeps failing.**
A: Check for resource locks (`Get-AzResourceLock`) — a Delete or ReadOnly lock on any resource within the group blocks the entire group's deletion until removed.

**Q: You need to give a third-party vendor temporary access to a specific resource group only, then revoke it automatically.**
A: Use Azure PIM (Privileged Identity Management) for time-bound, just-in-time role assignment scoped to that resource group — avoids leaving standing access that has to be manually remembered and revoked later.

**Q: An Azure Policy is supposed to enforce tagging but existing resources still show as non-compliant.**
A: Policy assignment alone doesn't retroactively fix existing resources — need to trigger a Remediation task (for DeployIfNotExists/Modify effect policies) to actually apply the fix to already-existing non-compliant resources.

**Q: A storage account's public access needs to be disabled, but an application breaks immediately after.**
A: The application was likely using the public endpoint directly without a Private Endpoint or service endpoint configured — before disabling public access, ensure Private Endpoint + correct DNS resolution is in place and tested first.

---

# WINDOWS SERVER / PATCHING / ITSM

**Q: A patch Tuesday update breaks a critical application on a subset of servers.**
A: This is exactly why patch rings exist — the pilot ring should have caught it before full rollout; immediate action is to pause the rollout to remaining rings, roll back the pilot group if possible, and open a Problem record to determine why pilot testing didn't catch the conflict.

**Q: You're asked to justify moving from WSUS to Azure Update Manager for a hybrid server fleet.**
A: Azure Update Manager works across both on-prem (via Azure Arc) and cloud servers from one pane, integrates with Azure Policy for compliance reporting, and doesn't require maintaining a separate WSUS server — a genuine hybrid management consolidation benefit relevant to this exact role.

**Q: A Standard (pre-approved) change causes an unexpected outage.**
A: This should trigger a Problem record — a Standard change is supposed to be low-risk and repeatable by definition; if it caused an outage, the change's risk classification itself needs review, not just the immediate incident.

**Q: ServiceNow shows a recurring incident category spiking month over month.**
A: This is a Problem Management trigger — a recurring incident pattern (not a one-off) should generate a formal Problem record to find the common root cause, rather than continuing to resolve each occurrence as an isolated incident.

---

# SECURITY / DEFENDER SUITE

**Q: Defender for Identity flags "suspicious DCSync activity" — what does that mean and what do you do?**
A: DCSync is a technique where an attacker impersonates a DC to request password hash data via replication APIs — immediately investigate the source account/machine, check if it legitimately needs replication rights (only DCs and specific service accounts like Entra Connect should), and treat as a likely credential-theft attempt if not.

**Q: A device shows a Defender for Endpoint alert for "impossible process execution" but the user says they didn't do anything unusual.**
A: Investigate via Defender's device timeline — check for a legitimate but unusual admin/IT script execution (false positive) vs genuine malware/living-off-the-land technique; don't dismiss without checking the actual process tree and parent process.

---

## HOW TO USE THIS FILE
This batch intentionally avoids repeating file 21's scenarios — together the two files give you roughly 80 realistic, operational scenario/answer pairs. Read both back-to-back a day or two before the interview; the goal is fluency, not memorization of exact wording.
