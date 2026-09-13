# 9. AZURE FINOPS / COST MANAGEMENT (your strongest demonstrated area — polish, add missing pieces)

## 9.1 Core Levers (full list — your answer covered ~60% of this)

**What you already covered well:** rightsizing based on real utilization, auto-shutdown scheduling for non-prod, reserved instances for stable long-running workloads, zombie/orphaned resource cleanup, storage lifecycle tiering.

**What was missing — add these:**
- **Azure Advisor** — built-in, free recommendations engine specifically for cost (also reliability/security/performance) — flags underutilized VMs, unattached disks, idle App Gateways/Load Balancers automatically. Always mention this as your starting point in a real answer — it's the fastest, lowest-effort win.
- **Savings Plans vs Reserved Instances** — commonly confused, a real interview trap:
  - **Reserved Instances (RI):** Commit to a SPECIFIC VM size/family/region for 1 or 3 years — highest discount, but least flexible.
  - **Savings Plans:** Commit to a SPENDING amount/hour across compute (VMs, App Service, Functions, etc.) regardless of size/region/family — more flexible, slightly less discount than a perfectly-matched RI.
- **Budgets and Cost Alerts** — proactive threshold alerting (e.g., "alert at 80% of monthly budget") rather than discovering overspend after the invoice.
- **Tagging and cost allocation** — enables chargeback/showback to specific teams, which drives accountability (teams optimize their own spend when they can see it attributed to them).
- **Spot VMs** — for interruptible, fault-tolerant workloads (batch processing, dev/test), can be 60-90% cheaper than pay-as-you-go, but Azure can reclaim capacity with short notice.
- **Right architecture over right-sizing alone** — e.g., moving from VM-based workloads to PaaS/serverless (App Service, Functions) where usage-based billing avoids paying for idle capacity entirely — a more advanced, "architect-level" answer point.

## 9.2 FinOps as ongoing governance, not a one-time project (important framing point)

The FinOps Foundation's model (Inform → Optimize → Operate) is worth knowing by name — shows you understand FinOps as a continuous discipline, not a quarterly cleanup exercise. **Inform** = visibility (tagging, showback dashboards). **Optimize** = the levers above. **Operate** = ongoing governance (policies preventing waste from recurring — e.g., Azure Policy requiring a cost-center tag on all new resources, or auto-shutdown enforced by policy rather than manually per-VM).

## 9.3 Full Prioritized Approach for "reduce spend 20% without impacting SLA"

1. **Run Azure Advisor + Cost Management reports first** — free, fast, zero-risk wins (unattached disks, idle public IPs, oversized VMs already flagged).
2. **Rightsize based on actual utilization** (Azure Monitor metrics, minimum 2-4 weeks of data, not a snapshot) — biggest lever, lowest risk if data-driven.
3. **Reserved Instances / Savings Plans for stable, predictable workloads** — production baseline compute that isn't scaling down.
4. **Auto-shutdown/scale non-prod outside business hours** — near-zero risk to production SLA since it's explicitly non-prod.
5. **Storage lifecycle tiering** — move cold/archival data down tiers automatically.
6. **Decommission zombie resources** — orphaned disks, unattached NICs/public IPs, old snapshots — genuinely free money, no SLA risk.
7. **Tag and allocate cost** to drive team-level accountability going forward (not a direct savings lever itself, but sustains the other five).
8. **Evaluate Spot VMs for eligible non-critical workloads** — only if genuinely fault-tolerant, since SLA impact risk is real here if misapplied.

**Why this order:** always exhaust the zero-risk/zero-effort wins (Advisor, zombie cleanup) before touching anything that could affect running production workloads, and always base rightsizing on real trend data, not a single day's snapshot which can be misleading (e.g., don't rightsize down a VM just because it was idle during a holiday week).

## INTERVIEW QUESTIONS
🔴 Reduce Azure spend by 20% without impacting SLA — full prioritized approach.
🔴 Difference between Reserved Instances and Savings Plans?
🟠 What does Azure Advisor check for cost specifically?
🟠 How do budgets/cost alerts fit into a FinOps practice?
🟡 What's the FinOps lifecycle (Inform/Optimize/Operate) and why does it matter that it's continuous?
🟡 When would you use Spot VMs, and what's the risk?
🔴 Scenario: a team's dev/test spend has crept up 300% over 6 months with no one noticing — what governance would you put in place?

## L3 ANSWER
"I'd start with Azure Advisor and Cost Management — that's free visibility into unattached disks, idle resources, and already-flagged oversized VMs, so it's the fastest win with zero production risk. Then I'd rightsize based on actual utilization trends over several weeks, not a snapshot, since a single slow day can make a VM look over-provisioned when it isn't. For predictable, stable production workloads I'd layer in Reserved Instances or Savings Plans — Savings Plans if the workload mix changes over time, RIs if it's a fixed VM family we're confident won't change for the term. Non-prod gets auto-shutdown outside business hours since there's no SLA risk there at all. Beyond the one-time cleanup, I'd put tagging and budget alerts in place so cost visibility becomes ongoing governance rather than something we only look at when someone asks for a 20% cut — that's really the FinOps principle: Inform, Optimize, Operate, as a continuous loop, not a project with an end date."

## MEMORY TRICK
**Free first (Advisor, zombies) → Data-driven rightsizing → Commit for stable (RI/Savings Plan) → Schedule for non-prod → Tier storage → Govern going forward (tags/budgets).**
