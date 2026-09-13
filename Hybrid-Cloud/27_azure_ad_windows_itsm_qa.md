# 27. FRESH Q&A BATCH — Azure / AD-DNS-DHCP-GPO / Windows Server & ITSM

---

# AZURE (10 Questions)

**Q1. What's the difference between Azure Metrics and Azure Monitor Logs, and when would you use each?**
A: Metrics are near-real-time numeric time-series data (CPU%, disk IOPS) best for dashboards and threshold-based alerting. Logs are queryable (via KQL in Log Analytics) event/record data best for deep investigation and correlation across multiple resources. Use Metrics to know something's wrong fast; use Logs to figure out why.

**Q2. A resource has a Diagnostic Setting configured, but no logs are showing up in Log Analytics — what do you check?**
A: Confirm the Diagnostic Setting is actually sending to the CORRECT workspace (easy to point it at the wrong one), check that the specific log category needed is enabled (not all categories are on by default), and verify the resource is actually generating that type of event to begin with.

**Q3. What's the risk of applying GRS (Geo-Redundant Storage) broadly "just to be safe" across all storage accounts?**
A: GRS costs meaningfully more than LRS, and it doesn't even provide automatic failover — activating the secondary copy requires explicit action. Applying it without workload-by-workload justification is a real FinOps governance gap; not every workload's business risk justifies the added cost.

**Q4. What's the difference between a Resource Lock (Delete/ReadOnly) and RBAC permissions?**
A: RBAC controls WHO can perform actions based on their role assignment. A Resource Lock is a separate, role-independent safeguard that blocks a specific ACTION (delete or any modification) regardless of who's attempting it — even an Owner can't delete a resource with a Delete lock without first removing the lock itself.

**Q5. A team says their Azure Policy-required tag isn't showing up on resources they created last month — why?**
A: Policy assignment doesn't retroactively apply to existing resources. A Remediation task needs to be triggered (for DeployIfNotExists/Modify effect policies) to actually backfill compliance on resources created before the policy existed.

**Q6. What's the actual difference between Azure Backup and Azure Site Recovery, in one sentence each?**
A: Azure Backup provides periodic, point-in-time snapshot recovery (restore to a prior state). Azure Site Recovery provides continuous, near-real-time replication with orchestrated failover for full disaster recovery with a much lower RPO/RTO.

**Q7. Why might a Managed Disk throttle even on a powerful VM size with plenty of CPU/RAM headroom?**
A: Each managed disk tier has its own independent IOPS/throughput ceiling based on disk size, separate from the VM's own limits — a workload can exceed the disk's ceiling and get throttled even if the VM itself has capacity to spare.

**Q8. What's the actual mechanism by which VNet peering being "non-transitive" causes a real design problem?**
A: In a hub-spoke model, if Spoke A peers to Hub, and Spoke B peers to Hub, Spoke A cannot automatically reach Spoke B through the Hub via peering alone — that requires either a Network Virtual Appliance/Azure Firewall routing traffic between spokes via UDRs, or Virtual WAN, which handles this transitivity for you.

**Q9. What does "Effective Routes" show you that just looking at your own UDR doesn't?**
A: It shows the actual MERGED routing table Azure applies — combining system default routes, your UDRs, AND any routes injected by VNet peering or gateway connections. Checking only your own UDR can miss an interfering route from another source entirely.

**Q10. Why is Azure Advisor considered the "free" first step in any cost optimization exercise?**
A: It automatically analyzes actual usage and flags concrete cost-saving opportunities (idle resources, oversized VMs, unattached disks) at zero effort and zero risk — checking it first avoids spending manual analysis time rediscovering things it already surfaces for free.

---

# ACTIVE DIRECTORY / DNS / DHCP / GPO (10 Questions)

**Q1. What's the actual difference between a "soft" replication failure and a "hard" one in AD?**
A: A soft failure is transient/intermittent (temporary network blip, brief unavailability) that typically self-resolves on the next replication cycle. A hard failure is persistent — the same error appears cycle after cycle (e.g., broken RPC connectivity, deleted/orphaned replication link) and requires manual intervention to fix.

**Q2. Why does DHCP scope exhaustion affect only NEW devices, not existing ones?**
A: Existing devices already hold an active lease and continue using it until it needs renewal — DHCP doesn't revoke active leases just because the pool is now full. Only devices requesting a NEW lease (new device, or an existing one whose lease expired) are affected by exhaustion.

**Q3. What's the difference between DNS Scavenging and simply deleting old DNS records manually?**
A: Scavenging is an automated, age-based process that removes records that haven't been refreshed within a configured aging period — ongoing and hands-off. Manual deletion is a one-time cleanup with no ongoing mechanism, meaning the same staleness problem reaccumulates unless scavenging (or a recurring manual process) is actually enabled.

**Q4. A GPO is linked at the Domain level and Enforced. An OU below it has Block Inheritance enabled and a conflicting setting in its own GPO. Which setting wins?**
A: The Domain-level Enforced GPO wins. Enforced always overrides Block Inheritance — this is one of the most commonly misunderstood GPO interactions.

**Q5. What is the actual mechanism behind "USN Rollback," and why is it dangerous?**
A: Each DC tracks Update Sequence Numbers (USNs) representing what it believes has been replicated where. If a DC is restored from an old, non-AD-aware snapshot/backup, its USN counter resets to an earlier point, but OTHER DCs still believe that DC is at a LATER USN — causing replication inconsistency and, in severe cases, missed or duplicated changes that corrupt directory consistency.

**Q6. Why does Kerberos specifically (not just "authentication" generally) have a hard 5-minute time skew tolerance?**
A: Kerberos tickets are time-stamped to prevent replay attacks — an attacker capturing a ticket and reusing it later should fail because too much time has passed. The 5-minute window balances practical clock drift tolerance against this replay-attack protection; too much drift and the protocol can't distinguish a legitimate slightly-delayed request from a replayed old one.

**Q7. What's the difference between a Conditional Forwarder and a standard Forwarder in DNS?**
A: A standard Forwarder sends ALL otherwise-unresolvable queries to one upstream DNS server (typically for general internet resolution). A Conditional Forwarder sends queries for ONE SPECIFIC named domain to a specific DNS server, while everything else follows normal resolution — used for targeted cross-domain/cross-forest/partner name resolution.

**Q8. Why would a GPO's setting show correctly in the GPMC console but not actually apply on a client?**
A: The console reads GPO metadata from AD, but the actual policy FILES live in SYSVOL, replicated separately via DFS-R. If DFS-R hasn't caught up yet, the client pulls old policy files even though AD-side metadata already shows the update — a replication timing gap between two technically separate replication mechanisms.

**Q9. What's the real difference between DHCP Failover Load Balance mode and Hot Standby mode in terms of actual client experience?**
A: In Load Balance mode, both servers actively respond to DHCP requests, roughly splitting the load (clients might get a lease from either server under normal conditions). In Hot Standby, one server handles ALL requests actively while the other sits passive and only responds if the primary becomes unreachable — clients normally only ever interact with the primary.

**Q10. Why is restoring a Domain Controller from a hypervisor-level VM snapshot specifically risky, compared to a proper backup tool?**
A: A raw VM snapshot restore doesn't go through AD's own VSS-aware backup/restore process, so the DC's USN state can become inconsistent with what other DCs believe it already replicated — this is exactly the mechanism behind USN Rollback. AD-aware backup tools (Windows Server Backup with System State, or third-party AD-aware tools) properly reset replication state during restore; a generic snapshot does not.

---

# WINDOWS SERVER / ITSM / INCIDENT MANAGEMENT (10 Questions)

**Q1. What's the practical difference between a Known Error and a Workaround in ITIL Problem Management?**
A: A Workaround is a temporary way to restore service or reduce impact without fixing the root cause. A Known Error is the formally DOCUMENTED problem record — including its root cause (once identified) and its associated workaround — tracked until a permanent fix is implemented and the error is formally closed.

**Q2. Why does a Standard Change still need governance even though it's "pre-approved"?**
A: Pre-approval assumes the change is genuinely low-risk and repeatable exactly as documented. If a Standard Change causes an unexpected outage, that's a signal the risk classification itself was wrong — it should trigger a review of whether that change type still qualifies as Standard, not just a one-off incident fix.

**Q3. What is Azure Arc, and why does it matter specifically for a hybrid infrastructure role?**
A: Azure Arc extends Azure-native management (Azure Policy, Update Management, Monitor, RBAC) to on-prem or other-cloud servers, letting you manage them from the same control plane as your actual Azure resources — directly relevant to a hybrid role because it reduces the need for separate on-prem-only tooling (like WSUS) for basic governance/patching.

**Q4. What's the difference between Incident Management and Problem Management in terms of the actual GOAL of each process?**
A: Incident Management's goal is restoring service as fast as possible — it doesn't need root cause first, just mitigation. Problem Management's goal is finding and permanently fixing the ROOT CAUSE, often after the incident is already resolved, specifically to prevent recurrence.

**Q5. Why should patch rollout use rings (pilot → broader → full) instead of a single mass deployment?**
A: A pilot ring limits the blast radius if a patch has an unexpected conflict — catching the problem on a small, monitored subset before it affects the entire production fleet, rather than discovering the issue only after everything is already patched.

**Q6. What does a CMDB actually do, and why does it matter during a major incident specifically?**
A: A CMDB (Configuration Management Database) tracks assets and their RELATIONSHIPS/dependencies. During a major incident, it lets you quickly assess "blast radius" — which other services depend on the failing component — so you can prioritize communication and understand downstream impact instead of discovering dependencies reactively.

**Q7. What's the difference between Emergency Change and Standard Change, beyond just "it's urgent"?**
A: Standard Change is pre-approved and repeatable with known low risk, requiring no case-by-case approval. Emergency Change is for genuinely urgent situations needing expedited approval — but it still requires SOME authorized approval (even verbal/fast-tracked), just not the full normal CAB cycle upfront; it gets formal retrospective review afterward.

**Q8. Why is an RCA considered "incomplete" if it identifies a root cause but has no assigned action items?**
A: Identifying WHY something happened without assigning OWNERSHIP and a DEADLINE for the fix means there's no actual mechanism ensuring the fix happens — the same root cause is very likely to recur, making the RCA a documentation exercise rather than an effective prevention process.

**Q9. What's the real difference between Windows Server 2019 and 2022 that's actually relevant operationally, not just a marketing bullet point?**
A: 2022 adds Secured-core server support (hardware-rooted security features), native TLS 1.3, and stronger Azure Arc integration — the Arc integration specifically matters operationally for hybrid management consistency, while Secured-core matters for security posture on servers handling sensitive workloads.

**Q10. During a Sev1 major incident, why is a "still investigating, next update in 30 minutes" message better than staying silent until you have a real update?**
A: Silence during an outage erodes stakeholder trust faster than the outage itself — people assume nothing is being done. A fixed-cadence update, even with no new information, demonstrates active, structured incident handling and keeps stakeholders from escalating unnecessarily or making their own uninformed assumptions about status.
