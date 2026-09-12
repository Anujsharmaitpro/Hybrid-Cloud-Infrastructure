# 8. AZURE ADMINISTRATION (your resume-documented strength — condensed but complete)

## 8.1 Compute — VM Troubleshooting Deep Dive (fixes gap from your Q1 answer)

**Full L3 sequence for "VM shows Running but unreachable via RDP":**
1. **Boot diagnostics** (Azure Portal > VM > Boot diagnostics) — screenshot of actual console output; instantly tells you if the OS is stuck at a boot screen, showing a bugcheck/BSOD, or actually at a login prompt (meaning it's a network issue, not an OS issue).
2. **Serial console** (Azure Portal > VM > Serial console) — gives you a live text console into the VM even without network connectivity — can run commands directly if the OS is responsive but network stack is broken.
3. **NSG rules** — check both Network Interface-level and Subnet-level NSGs; use **Effective Security Rules** view to see the actual merged rule set, not just what's configured per-NSG.
4. **Effective Routes** — check for a UDR (User Defined Route) accidentally black-holing traffic (e.g., a misconfigured route to a firewall appliance that's down).
5. **VM extension health** — check if the Network Watcher/Guest Agent extension is in a failed state (common after an in-guest update breaks the agent).
6. **In-guest checks via Serial Console/Boot diagnostics:** OS disk full (halts many services silently), Windows Firewall blocking RDP (1st thing to check once you have console access), RDP service (TermService) stopped, NIC has lost its DHCP lease inside the guest (Azure network is fine, guest OS networking isn't).
7. **Network Watcher tools** — Connection Troubleshoot, IP Flow Verify — programmatically test whether specific traffic (e.g., port 3389 from your IP) would be allowed or denied, without needing an actual connection attempt.

## 8.2 Storage
Blob (Hot/Cool/Cold/Archive tiers — lifecycle policies to auto-transition), Managed Disks (Standard HDD/SSD, Premium SSD, Ultra Disk — IOPS/throughput differ), redundancy options (LRS/ZRS/GRS/GZRS — trade-off between cost and durability/availability across zones/regions).

## 8.3 Networking
VNets, Subnets, NSGs (stateful, evaluated by priority number, lowest wins), UDRs, VNet Peering (non-transitive by default — A↔B and B↔C doesn't mean A↔C), Azure Firewall / NVAs, Private Endpoints (brings PaaS services into your VNet's private IP space, avoiding public internet exposure), ExpressRoute/VPN Gateway for hybrid connectivity.

**Interview trap:** "If VNet A peers with B, and B peers with C, can A reach C?" — No, peering is non-transitive — this is a very common L3 architecture question.

## 8.4 Governance — Azure Policy, RBAC, Management Groups, Landing Zones

**Azure Policy:** Enforces/audits configuration compliance (e.g., "deny VM creation without a specific tag," "audit storage accounts without encryption"). Effects: Deny, Audit, Append, DeployIfNotExists.
**RBAC scope hierarchy:** Management Group → Subscription → Resource Group → Resource — permissions inherit downward.
**Landing Zone concept:** A pre-architected, policy-governed subscription/environment structure (hub-spoke networking, standard RBAC, standard policies) that new workloads land into — ensures consistency instead of ad-hoc subscription sprawl.

## 8.5 Monitoring — Azure Monitor / Log Analytics
Metrics (near-real-time numeric data) vs Logs (queryable via KQL in Log Analytics) — different use cases: Metrics for dashboards/alerting thresholds, Logs for deep investigation/correlation.
**Diagnostic settings** must be explicitly configured per-resource to send logs to a Log Analytics workspace — a very common gap (a resource "not logging" usually just has no diagnostic setting configured, not a broken monitoring pipeline).

## 8.6 Backup / DR
Azure Backup (Recovery Services Vault) — VM snapshot-based backup with configurable retention (daily/weekly/monthly/yearly). Azure Site Recovery (ASR) — for full DR replication/failover between regions, different tool from Backup (Backup = point-in-time recovery; ASR = continuous replication + orchestrated failover).

## COMMANDS
```
Test-NetConnection -ComputerName <ip> -Port 3389
Get-AzNetworkWatcherEffectiveRoute
Get-AzEffectiveNetworkSecurityGroup
Get-AzVM -Status                       → check power/provisioning state
Get-AzDiagnosticSetting
```

## INTERVIEW QUESTIONS
🔴 VM shows Running but unreachable via RDP — full L3 diagnostic sequence.
🔴 Explain effective routes and effective NSG rules — why check both instead of just configured rules?
🟠 Is VNet peering transitive? Why does this matter architecturally?
🟠 Difference between Azure Backup and Azure Site Recovery?
🟡 What's the difference between Metrics and Logs in Azure Monitor?
🔴 Scenario: a resource has "no logs" in Log Analytics — what's the first thing you check?
🟠 Architecture: design a landing zone structure for a company onboarding 5 new subscriptions.

## L3 ANSWER
"For an unreachable-but-running VM, I don't jump straight to NSGs — I start with Boot Diagnostics because it tells me in ten seconds whether this is even a network problem versus the OS being stuck at boot. If the console shows a login screen, I know the OS is healthy and it's purely a network path issue — then I check Effective Security Rules and Effective Routes specifically, not just the configured NSG, because a UDR silently routing traffic through a downed firewall appliance produces the exact same symptom as a bad NSG rule but requires a completely different fix. If I still need to get inside the guest, Serial Console lets me check Windows Firewall and the RDP service state without needing the network path to already work."
