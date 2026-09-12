# 20. AZURE — COMPUTE / STORAGE / NETWORKING / GOVERNANCE / FINOPS / BACKUP — Point-by-Point Breakdown
### Format for every item: **What is it? → What is it used for? → What issues can it cause (if broken/misconfigured)?**

---

# SECTION A — COMPUTE

## * Azure Virtual Machines (IaaS)

**What is it?**
Simple: A full virtual server hosted in Microsoft's datacenters, but you manage the OS, patching, and everything inside it, same as on-prem.
Technical: An IaaS compute instance backed by a chosen VM size/family (determines vCPU, RAM, disk throughput, network bandwidth caps), running on Azure's underlying hypervisor fabric.

**What is it used for?**
Lift-and-shift migrations, workloads needing full OS control, or applications not yet re-architected for PaaS/serverless.

**What issues can it cause?**
- **Wrong VM size chosen relative to workload** — under-provisioned causes performance issues; over-provisioned wastes cost (directly ties to the FinOps rightsizing conversation).
- **VM shows "Running" but is unreachable** — could be OS-level (stuck boot, BSOD), network-level (NSG/UDR/firewall), or agent-level (VM extension failure) — Boot Diagnostics and Serial Console are the fastest way to distinguish OS vs network cause.
- **Availability Set/Zone not configured** — a single hardware/rack failure or planned Azure maintenance can take down the VM with no redundancy, when a properly configured Availability Set or Zone would have protected against exactly that failure domain.

---

## * VM Extensions

**What is it?**
Small agents/add-ons that run on a VM to provide additional functionality beyond the base OS — e.g., Guest Agent (needed for most management features), Network Watcher Agent, Custom Script Extension, Azure Monitor Agent, BGInfo.

**What is it used for?**
Extends what Azure can do TO and WITH the VM — monitoring, diagnostics, automated configuration — without requiring manual in-guest setup for each capability.

**What issues can it cause?**
- **Extension in a "Failed" or "Provisioning" stuck state** — can silently break dependent functionality (e.g., a broken Guest Agent means Boot Diagnostics/Serial Console access or certain monitoring stops working) without any obvious symptom pointing directly at "check your extensions."
- **Extension conflicts** — certain combinations (e.g., two different monitoring agents both trying to manage similar OS hooks) can cause resource contention or one silently failing to report data.
- **In-guest update breaking an extension** — a Windows Update or manual in-guest change can corrupt an extension's dependencies, requiring a reinstall of that specific extension rather than a VM-level fix.

---

## * Boot Diagnostics & Serial Console

**What is it?**
**Boot Diagnostics** captures a screenshot + boot logs of the VM's console output, viewable from the portal. **Serial Console** gives a live, interactive text-based console connection to the VM — works even without a functioning network stack inside the guest.

**What is it used for?**
The FIRST diagnostic step for any "VM unreachable" scenario — instantly tells you whether the problem is the OS itself (stuck at boot, BSOD, disk error visible on screen) versus a purely network-layer problem (OS shows a normal login screen, meaning it's healthy and the issue is elsewhere).

**What issues can it cause?**
Neither tool itself "causes" issues, but **skipping this step and jumping straight to NSG/networking troubleshooting** wastes significant time when the actual problem is OS-level (e.g., a full OS disk halting services, or a driver failure causing a boot loop).

---

# SECTION B — STORAGE

## * Blob Storage Tiers (Hot / Cool / Cold / Archive)

**What is it?**
Different pricing/performance tiers for the SAME blob storage service, trading off storage cost against access cost and retrieval latency — Hot (frequent access, higher storage cost/lower access cost), Cool (infrequent access), Cold (rarely accessed, cheaper still), Archive (lowest storage cost, but retrieval can take hours, not instant).

**What is it used for?**
Cost optimization for data with predictable access patterns — e.g., recent logs in Hot, older compliance-retention logs in Archive, since you're paying for storage cost 24/7 but access cost only when actually retrieved.

**What issues can it cause?**
- **Data placed in Archive tier accessed unexpectedly urgently** — retrieval isn't instant (can take hours depending on priority setting), causing a real operational delay if someone assumed "it's still in the cloud, I can grab it anytime."
- **Lifecycle policy misconfigured** (wrong age threshold for auto-tiering) — data moves to a colder tier before it should still be frequently accessed, causing unexpected access charges when repeatedly retrieved from a cheaper-but-costlier-per-access tier.
- **No lifecycle policy configured at all** — data sits in Hot tier indefinitely even once access patterns drop off, representing ongoing unnecessary cost — directly ties into the FinOps storage-tiering lever.

---

## * Managed Disks (Standard HDD / Standard SSD / Premium SSD / Ultra Disk)

**What is it?**
Different disk performance tiers for VM-attached storage — increasing IOPS/throughput/consistency (and cost) from Standard HDD up through Ultra Disk, which offers configurable IOPS/throughput independent of disk size.

**What is it used for?**
Matching disk performance to actual workload I/O requirements — a database server needs Premium SSD or Ultra Disk; a low-activity file server may be fine on Standard SSD/HDD.

**What issues can it cause?**
- **Disk tier mismatched to workload** — Standard HDD under a high-IOPS database workload causes severe performance degradation (often the actual root cause of a "slow VM" complaint that looks like a compute sizing issue but is actually disk-tier related).
- **Disk-level throttling** — even Premium SSDs have IOPS/throughput ceilings per disk size; a workload exceeding that ceiling gets throttled regardless of VM size — a nuanced point showing you understand storage isn't just "attached to the VM," it has its own independent performance ceiling.
- **Orphaned/unattached managed disks** — disks left behind after a VM deletion continue incurring storage cost indefinitely if not cleaned up — a classic "zombie resource" FinOps finding.

---

## * Redundancy Options (LRS / ZRS / GRS / GZRS)

**What is it?**
**LRS** (Locally Redundant Storage — 3 copies in one datacenter), **ZRS** (Zone Redundant — copies across availability zones in one region), **GRS** (Geo-Redundant — copies replicated to a secondary paired region), **GZRS** (combines zone + geo redundancy).

**What is it used for?**
Matching durability/availability requirements to actual business risk tolerance — a dev/test storage account may only need LRS; a critical production dataset needing regional-outage protection needs GRS or GZRS.

**What issues can it cause?**
- **Choosing LRS for genuinely critical data** — no protection against a full datacenter/region-level outage; that data is unavailable (or lost, in a true disaster) until the affected facility itself recovers.
- **Assuming GRS provides automatic failover** — it doesn't by default; GRS keeps a geo-replicated copy, but activating it (RA-GRS for read-access, or a manual failover for GRS) requires explicit action or configuration — a very common misunderstanding worth clarifying in an interview.
- **Cost surprise** — GRS/GZRS costs meaningfully more than LRS; applying it broadly "just to be safe" without workload-by-workload justification is a real FinOps governance gap.

---

# SECTION C — NETWORKING

## * Virtual Networks (VNets) & Subnets

**What is it?**
A VNet is an isolated private network space within Azure; subnets divide it into smaller segments, similar in concept to on-prem network segmentation.

**What is it used for?**
Logical isolation and organization of resources, applying different security/routing policies to different tiers (e.g., web tier subnet vs database tier subnet with tighter NSG rules).

**What issues can it cause?**
- **Subnet address space too small** — running out of available IPs within a subnet blocks new resource deployment into it, even though the broader VNet has capacity — a planning oversight that surfaces later as an unexpected deployment failure.
- **Overlapping address spaces** between VNets that later need to peer or connect via VPN/ExpressRoute — makes peering/routing impossible without a costly re-addressing project — a critical design-time consideration.

---

## * Network Security Groups (NSGs)

**What is it?**
A stateful firewall rule set (allow/deny by port/protocol/source/destination) applied at the subnet level and/or individual network interface level, evaluated by priority number (lowest number = highest priority, first match wins).

**What is it used for?**
Segmenting and controlling traffic flow within and into a VNet — the primary network-layer access control mechanism in Azure IaaS.

**What issues can it cause?**
- **Configured rules vs Effective rules mismatch** — NSGs exist at BOTH subnet and NIC level; checking only one misses rules that are actually being applied — this is exactly why "Effective Security Rules" (the merged, actual result) must be checked, not just what's configured on a single NSG.
- **Rule priority conflict** — a lower-priority-number (higher precedence) DENY rule can silently override an intended ALLOW rule further down the list — a common source of "I added the rule but it's still blocked" confusion.
- **Default rules forgotten** — Azure has implicit default rules (e.g., deny all inbound from internet by default) that engineers sometimes forget exist when troubleshooting "why can't I reach this from outside."

---

## * User Defined Routes (UDRs) & Effective Routes

**What is it?**
UDRs override Azure's default system routing (e.g., to force traffic through a network virtual appliance/firewall instead of directly to its destination). Effective Routes show the actual MERGED routing table Azure applies (system routes + UDRs + peering/gateway routes combined).

**What is it used for?**
Forcing specific traffic paths for security/inspection purposes (e.g., all outbound traffic must go through a central firewall appliance) — common in hub-spoke network architectures.

**What issues can it cause?**
- **UDR misconfigured or pointing to a downed/unreachable appliance** — traffic is black-holed; the destination VM can show as perfectly healthy and "Running" while being completely unreachable, because traffic never even gets a chance to reach it — a subtle, easy-to-miss root cause behind "VM unreachable" tickets.
- **UDR forgotten after an architecture change** (e.g., firewall appliance decommissioned/replaced but old UDR still references the old IP) — silent routing failure for any traffic still matching that route.
- **Not checking Effective Routes** — troubleshooting only the UDR you know about while missing another UDR or a peering-injected route that's actually the interfering factor.

---

## * VNet Peering

**What is it?**
A direct, low-latency connection between two VNets (in the same or different regions) using Azure's backbone network rather than public internet — NOT the same as a VPN, no gateway/encryption overhead needed.

**What is it used for?**
Connecting hub-spoke architectures, or connecting VNets across subscriptions/regions that need to communicate directly.

**What issues can it cause?**
- **Assuming peering is transitive** — it is NOT: VNet A peered to B, and B peered to C, does NOT mean A can reach C. This is one of the most common Azure networking interview traps and a genuinely common real-world design mistake.
- **Peering configured on only one side** — peering must be established from BOTH VNets; a one-sided configuration doesn't establish connectivity.
- **Overlapping address spaces discovered only at peering time** — peering fails outright if the two VNets have overlapping IP ranges, forcing an unplanned re-addressing effort.

---

## * Private Endpoints

**What is it?**
Brings a PaaS service (Storage Account, SQL Database, Key Vault, etc.) INTO your VNet's private IP address space, rather than that service only being reachable via its public endpoint.

**What is it used for?**
Eliminates public internet exposure for PaaS services while still allowing normal VNet-based access/routing/NSG control over that traffic — a significant security posture improvement over relying solely on service-level firewall rules on the public endpoint.

**What issues can it cause?**
- **DNS not properly configured for Private Link** — without correct Private DNS Zone integration, clients may still resolve the service's PUBLIC endpoint even though a Private Endpoint exists, silently bypassing the intended private routing (and potentially failing if the public endpoint is also locked down) — a very common, subtle Private Endpoint misconfiguration.
- **NSG/firewall blocking traffic to the Private Endpoint's IP** within the VNet — since it now behaves like any other private IP resource, it's subject to the same NSG rules as anything else, which can be an unexpected point of failure if not accounted for.

---

# SECTION D — GOVERNANCE

## * Azure Policy

**What is it?**
A service that evaluates resources against defined rules and can Audit (report only), Deny (block non-compliant deployment outright), Append (auto-add a missing setting), or DeployIfNotExists (auto-remediate by deploying a companion resource) — assignable at Management Group, Subscription, or Resource Group scope.

**What is it used for?**
Enforcing organizational standards automatically at scale (mandatory tagging, disallowed VM SKUs, required encryption settings) rather than relying on manual review/tribal knowledge.

**What issues can it cause?**
- **A Deny policy blocking a LEGITIMATE deployment** unexpectedly — a team's valid deployment fails with a policy-denial error that can be confusing if they don't immediately realize a policy (rather than a permissions or syntax issue) is the actual blocker.
- **Policy assigned at too broad a scope without proper exemptions** — can block legitimate exceptions (e.g., a genuinely justified need for a normally-disallowed VM SKU for a specific workload) without a clear exemption process.
- **Audit-only policies mistaken for enforcement** — a team assumes a policy is actively preventing something when it's only Auditing (reporting), and non-compliant resources continue to be created — a genuine governance gap if the distinction isn't understood.

---

## * RBAC (Role-Based Access Control) — Azure Resources

**What is it?**
Permission assignments (Owner, Contributor, Reader, or custom roles) at Management Group → Subscription → Resource Group → Resource scope, with permissions inheriting DOWNWARD through that hierarchy.

**What is it used for?**
Least-privilege access control for Azure resources — distinct from Entra ID roles (which govern identity/tenant management, not resource access).

**What issues can it cause?**
- **Over-broad role assignment** (e.g., Contributor at Subscription level when only Resource-Group-level access was actually needed) — unnecessarily large blast radius if that account/credential is ever compromised.
- **Role assigned at the wrong scope** — assigning at Resource Group level when the user actually needs cross-resource-group visibility causes repeated "access denied" tickets that look like a bug but are a scoping decision.
- **Custom role definition errors** (wrong action strings, overly broad wildcard permissions) — can accidentally grant far more access than intended, a genuine security risk if not carefully reviewed.

---

## * Management Groups & Landing Zones

**What is it?** A hierarchy above subscriptions (Root → Platform/Landing Zones/Sandbox) that lets policy and RBAC cascade down automatically to new subscriptions placed underneath.
**What is it used for?** Preventing subscription sprawl with inconsistent governance — new workloads inherit baseline security/cost policy by default.
**What issues can it cause?** A subscription created outside the intended management group structure doesn't inherit baseline policies at all — a very real, very common governance gap in growing Azure estates.

---

# SECTION E — FINOPS / COST MANAGEMENT

## * Azure Advisor (Cost Recommendations)

**What is it?**
A free, built-in recommendations engine analyzing actual resource usage and flagging cost-saving opportunities (underutilized VMs, unattached disks, idle public IPs/load balancers) alongside its reliability/security/performance recommendations.

**What is it used for?**
The fastest, lowest-effort, zero-risk starting point for ANY cost-reduction initiative — should always be checked before more involved rightsizing/architecture work.

**What issues can it cause?**
Advisor itself doesn't cause issues, but **ignoring it and jumping straight to manual utilization analysis** wastes effort re-discovering things Advisor already flagged for free.

---

## * Reserved Instances vs Savings Plans

**What is it?**
**Reserved Instances (RI):** Commit to a SPECIFIC VM size/family/region for 1 or 3 years for the highest discount, least flexibility. **Savings Plans:** Commit to a SPENDING amount/hour across eligible compute (VMs, App Service, Functions) regardless of size/region/family — more flexible, slightly lower discount than a perfectly-matched RI.

**What is it used for?**
Reducing cost for predictable, sustained compute usage — the commitment-based discount mechanism, as opposed to pay-as-you-go pricing.

**What issues can it cause?**
- **RI purchased for a workload whose size/family later changes** — the reservation no longer matches actual usage, and the discount benefit is partially or fully lost (RIs have some flexibility for instance size within a family, but changing families/regions breaks the match).
- **Over-committing** to a 3-year RI for a workload whose future isn't actually certain — locks in cost even if the workload is decommissioned early (limited exchange/cancellation options exist but aren't unlimited).
- **Confusing the two options** during planning — choosing an RI for a workload whose SIZE will likely change over time (better suited to a Savings Plan's flexibility) results in a suboptimal, less flexible commitment.

---

## * Budgets & Cost Alerts

**What is it?**
Configurable spend thresholds (e.g., "alert at 80% of $50,000 monthly budget") that trigger notifications — proactive visibility rather than discovering overspend only at invoice time.

**What is it used for?**
Ongoing cost governance — catching a runaway spend trend (e.g., a forgotten test environment left running, or a team's usage growing faster than planned) while there's still time to act.

**What issues can it cause?**
Budgets/alerts don't themselves cause issues, but their ABSENCE is a common root cause of "how did we not notice this spend for 3 months" — a governance gap worth calling out proactively in a FinOps conversation.

---

## * Tagging & Cost Allocation

**What is it?**
Metadata key-value pairs attached to resources (e.g., `CostCenter: Finance`, `Environment: Production`) enabling cost reports to be broken down by team/project/environment rather than a single undifferentiated bill.

**What is it used for?**
Chargeback/showback — making cost visible and attributable to the team actually responsible, which drives accountability and behavior change far more effectively than a central team just occasionally cleaning up waste.

**What issues can it cause?**
- **Inconsistent or missing tagging** — cost reports become unusable for accountability purposes; you can see total spend but not WHO is responsible for which portion, undermining the entire chargeback model.
- **Tagging enforced via policy only after the fact** — existing resources created before the policy don't retroactively get tagged unless a remediation task (DeployIfNotExists policy effect, or a manual/scripted backfill) is specifically run.

---

# SECTION F — BACKUP & RECOVERY

## * Azure Backup (Recovery Services Vault)

**What is it?**
A snapshot-based, point-in-time backup service for VMs (and files, SQL, etc.) with configurable frequency (daily/weekly) and retention (including long-term GFS-style monthly/yearly retention).

**What is it used for?**
Point-in-time recovery — restoring a VM, specific disks, or individual files to a prior known-good state, distinct from disaster recovery (see ASR below) which is about continuous replication and failover, not periodic snapshots.

**What issues can it cause?**
- **Backup jobs "Completed with Warnings" repeatedly** — commonly indicates VSS (Volume Shadow Copy Service) issues INSIDE the guest OS, causing the backup to fall back to crash-consistent rather than app-consistent — worth checking `vssadmin list writers` inside the VM rather than assuming the backup infrastructure itself is broken.
- **Retention policy misconfigured** (too short) — a needed recovery point has already aged out and been deleted by the time someone realizes they need it — a policy-design gap, not a technical failure.
- **Soft delete not understood/relied upon** — Azure Backup enables soft delete by default (deleted backup data retained for a further ~14 days), which is a meaningful ransomware-resilience feature; NOT knowing this exists means missing an opportunity to reassure a security team during a ransomware-scenario discussion.

---

## * Azure Site Recovery (ASR)

**What is it?**
Continuous, near-real-time replication of VMs to a secondary region (or on-prem to Azure), with an orchestrated failover/failback process — a fundamentally different tool from Azure Backup (Backup = periodic snapshots for point-in-time recovery; ASR = continuous replication for full disaster recovery with minimal data loss).

**What is it used for?**
True disaster recovery — if an entire primary region/datacenter is lost, ASR lets you fail over running workloads to a secondary region with a defined RPO/RTO (Recovery Point/Time Objective), rather than restoring from a backup that could be up to 24 hours stale.

**What issues can it cause?**
- **Confusing ASR with Backup** — assuming Backup alone provides adequate DR when the business actually needs a low-RPO failover capability that only ASR provides — a genuine architecture/planning gap if surfaced too late (e.g., during an actual regional outage).
- **Failover never tested** — an ASR configuration that's never been through a test failover may have subtle issues (network config, DNS, dependency ordering) that only surface during a REAL disaster — DR testing discipline matters as much as the underlying replication technology.
- **Replication health degraded/paused without alerting** — if replication silently falls behind or stops, the actual RPO achievable during a real failover is much worse than assumed, and nobody finds out until it's needed.

---

## END-TO-END FLOW: A cost governance + resilience review across an Azure estate

```
Azure Advisor scan -> flags idle/underutilized resources (zero-effort first pass)
   |
Rightsizing based on Azure Monitor utilization trends (weeks of data, not a snapshot)
   |
Reserved Instances / Savings Plans applied to stable, predictable production workloads
   |
Non-prod: scheduled auto-shutdown outside business hours
   |
Storage: lifecycle policies auto-tier Blob data (Hot -> Cool -> Cold -> Archive) based on access age
   |
Tagging enforced via Azure Policy -> cost allocated per team/cost-center (Budgets/Alerts monitor ongoing)
   |
Resilience layer (separate but related): Azure Backup for point-in-time recovery,
ASR for full regional DR with tested failover -- NOT interchangeable with each other
```

## BREAK/FIX MASTER TABLE

| Component | Purpose | If it breaks/misconfigured | Symptom | First Check | Fix |
|---|---|---|---|---|---|
| VM | IaaS compute | Wrong size / no redundancy | Poor performance or single point of failure | Utilization metrics, Availability Set/Zone config | Rightsize, add Availability Zone/Set |
| VM Extensions | Agent-based functionality | Failed/stuck state | Silent loss of monitoring/diagnostic capability | Extension status blade | Reinstall/repair extension |
| Managed Disk | VM storage | Tier mismatched to I/O need | "Slow VM" that's actually disk-bound | Disk metrics (IOPS/throughput vs limit) | Upgrade disk tier |
| NSG | Network access control | Configured vs Effective rules mismatch | "I added the rule but it's still blocked" | Effective Security Rules view | Correct actual merged rule set, check priority order |
| UDR | Custom routing | Points to downed/removed appliance | VM "Running" but totally unreachable | Effective Routes | Fix/remove stale UDR |
| VNet Peering | Direct VNet connectivity | Assumed transitive | A can't reach C via B | Peering config on both sides | Establish direct peering if truly needed (not transitive) |
| Private Endpoint | Private PaaS access | DNS not integrated | Still resolves to public endpoint | Private DNS Zone linkage | Fix DNS zone integration |
| Azure Policy | Governance enforcement | Deny blocks legit deployment | Confusing deployment failure | Policy compliance/denial reason in deployment error | Add exemption or adjust policy scope |
| Azure Backup | Point-in-time recovery | VSS issues in guest | "Completed with warnings" repeatedly | `vssadmin list writers` in guest | Fix VSS writer issue in OS |
| ASR | Full DR failover | Never tested | Real failover fails when actually needed | Last test failover date/result | Schedule and run test failovers regularly |

## TOP 20 INTERVIEW QUESTIONS — THIS CLUSTER

🔴 VM shows "Running" but is unreachable — full diagnostic sequence including Boot Diagnostics/Serial Console.
🔴 Explain the difference between configured NSG rules and Effective Security Rules — why check both?
🔴 Is VNet peering transitive? Walk through why this matters architecturally.
🔴 Reduce Azure spend by 20% without impacting SLA — full prioritized approach.
🔴 Difference between Azure Backup and Azure Site Recovery — when would you use each?
🟠 Difference between Reserved Instances and Savings Plans, including the risk of choosing wrong.
🟠 What is a Private Endpoint and what's the most common misconfiguration with it?
🟠 What are the four Blob storage tiers and what's the risk of using Archive incorrectly?
🟠 Explain LRS vs ZRS vs GRS vs GZRS and how you'd choose between them.
🟡 What is Azure Policy's Deny vs Audit effect, and why does confusing them matter?
🟡 What causes a Managed Disk to throttle even on a powerful VM size?
🟡 What is a Landing Zone and why does subscription placement matter for governance?
🔴 Scenario: a resource has no logs appearing in Log Analytics — what's the first thing you check?
🔴 Scenario: a backup job repeatedly shows "Completed with Warnings" — full diagnosis.
🟠 Scenario: an ASR-protected workload fails over badly during an actual incident — what likely went wrong, and how would you have caught it earlier?
🟠 Scenario: a UDR causes a VM to become unreachable after a firewall appliance was replaced — how do you find this?
🟡 Scenario: two VNets can't peer — what's the most likely reason?
🔴 Architecture: design a landing zone structure for a company onboarding 5 new subscriptions.
🟠 Architecture: design storage tiering and redundancy strategy for a compliance-retention dataset.
🟡 What's the difference between Azure Monitor Metrics and Logs, and when do you use each?

## RED FLAGS / TRICK QUESTIONS
- "If VNet A peers with B, and B peers with C, can A reach C?" — No, peering is non-transitive.
- "Does GRS provide automatic instant failover?" — No, GRS keeps a geo-replicated copy but failing over to it requires explicit action/configuration, not automatic instant redirection.
- "Is Azure Backup the same as Disaster Recovery?" — No — Backup is periodic point-in-time recovery; ASR is continuous replication for full DR with a much better RPO/RTO.
- "Does an Azure Policy set to Audit actually prevent non-compliant resources from being created?" — No, Audit only reports; Deny is the effect that actually blocks creation.

## L3 ANSWER — VM Unreachable (60-90s)
"For a VM that shows Running but is unreachable, I don't start with NSGs — I start with Boot Diagnostics, because in ten seconds it tells me whether this is even a network problem at all. If the screenshot shows a normal login prompt, the OS is healthy and I know the issue is purely in the network path — then I check Effective Security Rules, not just the configured NSG, because a rule can look correct in isolation but be overridden by another rule at a different priority or a different scope entirely. I also check Effective Routes specifically, because a UDR silently routing traffic through a firewall appliance that's since been replaced or is down produces the exact same 'unreachable but running' symptom as a bad NSG rule, but needs a completely different fix. If I need to get inside the guest without network access working yet, Serial Console lets me check things like Windows Firewall state or a stopped RDP service directly."

## MEMORY TRICK
**VM unreachable = Boot Diagnostics FIRST (OS vs network), THEN Effective Rules + Effective Routes (not just configured).**
**Peering = never transitive. GRS = not automatic failover. Backup ≠ DR (ASR is DR).**
**FinOps order = Free wins (Advisor) -> data-driven rightsizing -> commit for stable (RI/Savings Plan) -> schedule non-prod -> tier storage -> govern with tags/budgets.**

## 10-SECOND REVISION
- Boot Diagnostics + Serial Console before NSGs, always, for unreachable VMs
- Effective Rules/Routes ≠ configured rules/routes — always check the merged, actual result
- VNet peering is never transitive
- GRS needs explicit failover action — it isn't automatic
- Azure Backup = point-in-time recovery; ASR = continuous replication DR — genuinely different tools
- Azure Advisor is the free, zero-effort first step in any cost reduction exercise
