# 13. DEEP-DIVE ROUND 2 — Preferred Skills + Advanced Follow-Up Drilling

This file covers the JD's **Preferred Skills** (not yet built out) plus harder, second-level follow-up questions on topics you've already studied — the kind a panel asks once your first answer holds up and they want to see how deep it actually goes.

---

## A. AZURE GOVERNANCE & LANDING ZONE CONCEPTS (Preferred Skill — not yet covered)

**What is it?**
Simple: A pre-built, policy-governed "template environment" that every new Azure workload lands into, so you're not reinventing network design, RBAC, and policy from scratch each time.
Technical: A landing zone is a defined subscription/management-group structure (commonly hub-spoke networking) with baseline Azure Policy, RBAC assignments, logging, and network topology already applied via Microsoft's Cloud Adoption Framework (CAF) reference architecture.

**How it works:**
Management Group hierarchy (Root → Platform / Landing Zones / Sandbox) → **Platform** management group holds shared services (hub VNet, identity, management/logging subscriptions) → **Landing Zone** management groups (Corp, Online) hold actual workload subscriptions that peer back to the hub → Azure Policy assigned at management group level cascades down automatically to any new subscription placed underneath.

**Why enterprises use it:** Prevents "subscription sprawl" — every team spinning up ad-hoc subscriptions with inconsistent security, tagging, and network design. New workloads inherit governance automatically instead of requiring manual setup each time.

**Failure/gap scenario:** A new subscription is provisioned OUTSIDE the landing zone management group structure (e.g., created directly under Root by mistake) — it won't inherit the baseline policies, meaning no mandatory tagging enforcement, no default deny-public-IP policy, etc. This is a very real, very common governance gap in growing Azure estates.

**Interview answer:** "A landing zone means a new workload doesn't start from zero — it lands into a management group that already has baseline policy, RBAC, and hub connectivity applied, so security and cost governance are consistent by default rather than something each team has to remember to configure."

---

## B. AZURE VIRTUAL DESKTOP (AVD) (Preferred Skill — not yet covered)

**What is it?** Microsoft's own VDI/DaaS platform — conceptually similar to Citrix VAD but Azure-native, using Azure-hosted session hosts and a Microsoft-managed control plane (no Delivery Controller to manage yourself, unlike Citrix).

**Key components:** Host Pool (pooled or personal), Session Hosts (Azure VMs running the AVD agent), Workspace (groups app groups for user-facing access), App Groups (RemoteApp or full Desktop), FSLogix (same profile container technology used in Citrix — this is a big overlap point worth mentioning).

**Key difference from Citrix (a common interview comparison question):**
| | AVD | Citrix VAD |
|---|---|---|
| Control plane | Microsoft-managed (no DDC to run) | Self-managed Delivery Controllers |
| Licensing | Included with certain M365/Windows licenses | Separate Citrix licensing |
| Multi-session Windows 10/11 | AVD-exclusive capability | Not available (Citrix uses Server OS for multi-session) |
| Advanced features (HDX enhancements, app layering) | Fewer native, but growing | More mature ecosystem |

**Interview trap:** "Is AVD 'better' than Citrix?" — Depends on the org: AVD reduces control-plane management overhead and is Azure-native, but Citrix has a more mature feature set (advanced HDX, broader OS support) and many enterprises run Citrix ON TOP of AVD session hosts to get both — a genuinely advanced, high-value answer if you can explain this hybrid model.

**Failure scenario:** Host Pool shows "Available" session hosts but users can't connect — check the AVD agent status on the session host (`Get-Service RDAgent, RDAgentBootLoader`), check FSLogix profile share connectivity (identical failure mode to Citrix profile issues), check Entra ID/RBAC role assignment on the App Group (a very common AVD-specific gotcha — users need BOTH the Application Group RBAC role AND to be in the assigned Entra ID group).

---

## C. MICROSOFT DEFENDER SUITE (Preferred Skill — not yet covered)

**Key products to know by name and function:**
- **Defender for Endpoint** — EDR (Endpoint Detection & Response) for Windows/macOS/Linux devices, integrates with Intune for device risk signals feeding into Conditional Access.
- **Defender for Office 365** — Safe Links (time-of-click URL rewriting) and Safe Attachments (sandbox detonation) — already referenced in the M365 mail-delay scenario.
- **Defender for Identity** — monitors on-prem AD for suspicious activity (e.g., Pass-the-Hash, DCSync attacks) by analyzing DC traffic via a sensor installed on DCs.
- **Defender for Cloud** (formerly Security Center) — Azure resource security posture management (CSPM) + workload protection (CWPP) across VMs, storage, databases.
- **Defender XDR** — unified portal correlating signals across all of the above for cross-domain incident investigation.

**Why this matters for THIS role specifically:** Defender for Identity directly protects the AD/DC infrastructure you're responsible for; Defender for Endpoint integrates with Intune compliance which feeds Conditional Access — this ties every domain in the JD together, and naming these connections shows systems-level thinking, not siloed tool knowledge.

**Failure scenario:** Defender for Identity sensor stops reporting from a DC — creates a blind spot for credential-attack detection without any visible symptom to end users (silent security gap) — should be monitored via the Defender portal's sensor health page specifically, not assumed healthy.

---

## D. AZURE BACKUP AND RECOVERY SERVICES (Preferred Skill — expand beyond what's in file 08)

**Recovery Services Vault deep dive:** Backup policies define frequency (daily/weekly) and retention (short-term daily, long-term monthly/yearly via GFS - Grandfather-Father-Son retention). **Soft delete** is enabled by default — deleted backup data is retained 14 days, protecting against accidental or malicious deletion (ransomware scenario relevance — a strong point to raise proactively).

**Restore options:** Full VM restore, Restore disks only (then attach to existing VM — useful when only OS is corrupted but data disks are fine), File-level recovery (mount a recovery point without a full VM restore — much faster for "I just need one file back").

**Cross-region restore:** Requires GRS/GZRS vault redundancy — lets you restore in a secondary region if the primary region has a full outage — critical DR conversation point.

**Failure scenario:** Backup jobs show "Completed with warnings" repeatedly — often means VSS (Volume Shadow Copy Service) issues inside the guest (app-consistent snapshot failing, falling back to crash-consistent) — check VSS writer status inside the VM (`vssadmin list writers`) rather than assuming the backup infrastructure itself is broken.

**Interview answer:** "Azure Backup gives point-in-time recovery with configurable retention, and soft delete means even a compromised admin account deleting backups doesn't immediately destroy recovery points — that's a meaningful ransomware-resilience point I'd raise proactively with a security team, not just a backup feature."

---

## E. INFRASTRUCTURE AS CODE AWARENESS (Preferred Skill — expand on Terraform from file 10)

**Core IaC concepts to articulate clearly:**
- **Declarative vs imperative** — Terraform/ARM/Bicep declare desired END STATE; a script imperatively lists STEPS. IaC tools reconcile actual vs desired state automatically.
- **State file** (Terraform-specific) — tracks what Terraform believes exists; a mismatch between actual Azure resources and the state file ("drift") is a very real operational problem — e.g., someone manually changes a resource in the portal, and the next `terraform apply` may try to revert it unexpectedly.
- **Idempotency** — running the same IaC repeatedly produces the same result, doesn't duplicate resources — a core reason IaC is safer than manual click-ops or ad-hoc scripts for repeatable provisioning.
- **Bicep vs ARM Templates vs Terraform** — Bicep is Microsoft's newer, cleaner DSL that compiles to ARM JSON (Azure-only); Terraform is multi-cloud and uses HCL, has a larger ecosystem/community modules; ARM JSON is the original, verbose, still underlies Bicep.

**Failure scenario — "drift":** A production NSG rule was manually added directly in the Azure Portal (hotfix during an incident) but never reflected back into the Terraform config — the next `terraform apply` from another engineer could silently remove that rule, reintroducing the original problem. **This is a real, common, dangerous IaC pitfall** — a strong thing to mention proactively as something you actively guard against (e.g., via `terraform plan` review discipline before any apply, and importing manual changes back into state promptly).

**Interview answer:** "IaC's value isn't just automation — it's that infrastructure state becomes reviewable and versioned like code. The risk I actively manage is drift: if someone makes a manual change directly in the portal during an incident, I make sure that gets reconciled back into the Terraform state quickly, because otherwise the next apply can silently undo a legitimate emergency fix."

---

## F. CLOUD MIGRATION AND MODERNIZATION EXPERIENCE (Preferred Skill — you have strong real experience here, sharpen the narrative)

**Migration methodology stages (know this framework by name — Microsoft's Cloud Adoption Framework migration phases):**
Assess → Migrate → Optimize → Secure/Manage.

**Migration approaches (the "6 Rs" — a very commonly asked framework, make sure you can name them):**
1. **Rehost** ("lift and shift") — move as-is, fastest, least risk, least benefit (your resume's 1,000+ VM migration is likely primarily this).
2. **Replatform** — minor optimization during move (e.g., move DB to managed PaaS instance without app rewrite).
3. **Refactor/Re-architect** — redesign for cloud-native (microservices, serverless).
4. **Rebuild** — discard and rewrite from scratch.
5. **Replace** — swap for SaaS (e.g., move a self-hosted CRM to Dynamics 365/Salesforce).
6. **Retire** — decommission if no longer needed (you mentioned zombie resource cleanup — same principle at migration-assessment time).

**Why naming this matters:** Your resume's 1,000+ VM migration reads as a Rehost — that's legitimate and valuable at scale, but an L3/architect-level interviewer may probe whether you assessed candidates for Replatform (e.g., "did you consider moving any SQL Servers to Azure SQL Managed Instance instead of lifting the VM as-is?") — be ready to discuss why Rehost was the right choice for THAT migration (speed, risk tolerance, zero-downtime requirement) rather than defaulting to "we just lifted everything."

**Failure scenario during cutover:** Application breaks post-migration due to hardcoded IP addresses or DNS names that assumed on-prem network topology — classic "lift and shift" gotcha; mitigated by pre-migration dependency mapping (which services talk to which, by IP/hostname) before cutover, not discovered during it.

**Real interview-ready answer using YOUR actual experience:** "For the 1,000+ VM migration, we used a Rehost approach with block-level replication for near-zero downtime cutover — the right call given the SLA constraints and timeline, but before each cutover wave we did dependency mapping to catch hardcoded network references, which is where lift-and-shift migrations most commonly break in production even when the migration itself technically succeeds."

---

## G. ADVANCED FOLLOW-UP DRILLING — Second-Level Questions on Already-Covered Topics

*(These are the "you answered that well, now let me push further" questions a strong panel asks. Format: first-level question → drill-down → what a strong answer includes.)*

**On Conditional Access:**
- Q1: "You said Named Locations are just a signal — what would you actually pair it with for a genuinely strong policy?"
- Strong answer: Device compliance (Intune) as an AND condition, or sign-in risk (Identity Protection, P2) — meaning even on the trusted network, an unmanaged or risky device still gets challenged.

**On PKI:**
- Q2: "If the intermediate CA distribution via GPO doesn't reach a laptop that's rarely on the corporate network (remote worker), what's your alternative?"
- Strong answer: Intune-deployed trusted certificate profile (Configuration Profile → Trusted Certificate) pushes the same intermediate cert over the internet via MDM channel, independent of GPO/on-prem network reachability — ties Intune and PKI together, a good cross-domain answer.

**On Hybrid Identity:**
- Q3: "You said PHS is more resilient — does that mean PHS has no security trade-off compared to PTA/Federation?"
- Strong answer: No — PHS does store password hash material (double-hashed) in the cloud, which some highly regulated orgs' compliance policies explicitly disallow regardless of the practical resilience benefit; that's a real, legitimate reason some enterprises still choose PTA or Federation despite the availability trade-off. Don't present PHS as strictly superior — it's a trade-off between resilience and a specific compliance posture.

**On Autopilot/Intune:**
- Q4: "If ESP is set to non-blocking as a permanent fix rather than a temporary mitigation, what's the risk?"
- Strong answer: Users reach the desktop before required security policies/compliance settings are fully applied — a device could be in active use, potentially non-compliant, before Conditional Access "require compliant device" checks would even have current data — a real security gap if left permanent rather than temporary.

**On Citrix:**
- Q5: "If Director shows Profile Load is slow only for a subset of users, not the whole Delivery Group, what does that change about your diagnosis?"
- Strong answer: Points away from a shared infrastructure issue (which would hit everyone) toward per-user profile bloat (e.g., large Outlook OST cached in the profile, or corrupted FSLogix container) rather than a storage/network-wide problem — narrows root cause to individual profile hygiene, not infrastructure.

**On Azure FinOps:**
- Q6: "You said rightsize based on utilization trends over weeks — what if the workload is genuinely seasonal (e.g., retail, tax season)?"
- Strong answer: Don't rightsize down purely off average utilization in that case — use autoscale (VM Scale Sets) or a scheduled scale-up/down aligned to the known seasonal pattern instead of a single static "right size," since a fixed size that's efficient in the off-season will bottleneck during the peak.

**On Major Incident/RCA:**
- Q7: "Your RCA identifies redundant DCs as the fix — who owns implementing that, and how do you make sure it actually happens and isn't just documented?"
- Strong answer: RCA action items need a named owner, a deadline, and should be tracked in the same ITSM problem record until closed — not just a bullet point in a report; some orgs formally review open RCA action items in a recurring problem-management meeting specifically to prevent this exact "documented but never done" failure mode.

---

## H. HOW TO USE THIS FILE IN PREP

Read section G last — those are the "gotcha" follow-ups most likely to expose shallow prep even when your first-level answer sounds confident. If you can answer both the original question AND its listed follow-up without hesitation, that's a genuine signal you're ready for that topic at panel level, not just mock-interview level.
