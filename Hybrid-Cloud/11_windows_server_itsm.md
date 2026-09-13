# 11. WINDOWS SERVER ADMINISTRATION + ITSM/INCIDENT MANAGEMENT

## 11.1 Windows Server 2016/2019/2022 — Key Differences (interviewers may ask "what's new")

- **2016 → 2019:** Introduced Windows Admin Center (modern browser-based management replacing many MMC consoles), Storage Migration Service, improved Storage Spaces Direct.
- **2019 → 2022:** Secured-core server support (hardware-rooted security), TLS 1.3 support, Azure Arc integration for hybrid management of on-prem servers from Azure Portal, SMB compression.
- **Interview angle:** Know that Azure Arc lets you manage on-prem Windows Servers with Azure-native tools (Azure Policy, Update Management, Monitor) — directly relevant to a "hybrid cloud" role and a good thing to mention proactively.

## 11.2 Patch Management
WSUS (on-prem, traditional) vs Azure Update Management/Update Manager (cloud-native, works across hybrid via Azure Arc). Key L3 concern: **patch rings/rollout waves** (pilot group → broader rollout) to avoid a bad patch causing a fleet-wide outage — never patch 100% of production simultaneously.

## 11.3 OS Hardening / Security Baselines
Microsoft Security Baselines (via Security Compliance Toolkit or Intune Security Baselines) — pre-built GPO/Intune templates aligned to CIS/NIST-style hardening. Common checks: disable SMBv1, enforce LSA protection, restrict NTLM, enable Credential Guard where hardware supports it.

## 11.4 Incident / Problem / Change Management (ITIL Framework)

**Incident Management:** Restore service ASAP — doesn't require root cause first, just mitigation (e.g., failover, restart, rollback).
**Problem Management:** Finds and fixes the ROOT CAUSE, often after the incident is already resolved — prevents recurrence. A "Known Error" is a documented problem with an identified workaround, tracked until permanently fixed.
**Change Management:** Controlled process for making changes (Standard = pre-approved/low-risk, Normal = requires CAB approval, Emergency = expedited approval for urgent fixes) — exists specifically to prevent unplanned outages from unreviewed changes.

**Major Incident process (this is exactly your Q10 scenario, structured properly):**
1. **Detect** — monitoring alert or user reports.
2. **Declare** — classify severity (Sev1 = full outage/major business impact) and open a bridge call.
3. **Communicate** — regular status updates to stakeholders at a fixed cadence (e.g., every 30 min for Sev1), even if the update is "still investigating" — silence is worse than a "no update yet" message.
4. **Mitigate** — restore service first (this might mean a workaround, not the permanent fix) — e.g., failing over to a healthy DC, restarting a service, rolling back a change.
5. **Resolve** — confirm service restored, validate with affected users/monitoring.
6. **RCA (Root Cause Analysis)** — post-incident, blameless review: what happened, why, what were the contributing factors, timeline reconstruction.
7. **Preventive action** — concrete follow-up items (monitoring gap closed, a change process tightened, a redundancy added) with owners and deadlines — an RCA without assigned preventive actions is incomplete.

**This directly upgrades your Q10 answer** — you had the technical DC-recovery steps roughly right but the communication/process structure was underdeveloped. Mention severity classification, fixed-cadence updates, and a formal RCA with assigned action items — that's what an L3/lead-level answer needs versus an engineer-level answer.

## 11.5 ServiceNow / ITSM Platform Concepts
Incident, Problem, Change, and CMDB (Configuration Management Database — the inventory of assets/relationships that ties incidents to affected services). Understanding CMDB relationships is what lets you assess "blast radius" quickly during a major incident (which other services depend on the failing component).

## INTERVIEW QUESTIONS
🔴 Walk through a major incident from detection to RCA and prevention, including stakeholder communication.
🔴 Difference between Incident, Problem, and Change management?
🟠 What's a Known Error and how does it relate to Problem Management?
🟠 Difference between Standard, Normal, and Emergency change types?
🟡 What is Azure Arc and why does it matter for hybrid server management?
🔴 Scenario: DCs at one datacenter stop responding to auth — full incident response including comms cadence.
🟠 Scenario: a routine patch causes an unexpected outage — what does your patch management process look like to prevent recurrence?

## L3 ANSWER (Major Incident)
"When DCs in one datacenter stop authenticating, my first priority is service restoration, not root cause — so I'm checking connectivity, DC service health, and DNS to see if this is recoverable quickly, or if I need to fail over authentication load to another site's DCs while I work the problem. I'd classify this as a Sev1 given the business impact and get a bridge call going immediately, with a fixed communication cadence to stakeholders — even a 'still investigating, next update in 30 minutes' message, because silence during an outage erodes trust faster than the outage itself. Once service is restored — whether that's the original DCs recovering or traffic failing over to healthy ones — I'd move into a blameless RCA: what actually happened, what the contributing factors were, and critically, what specific preventive actions come out of it with an owner and a deadline, not just a write-up that sits in a folder. In general, this specific failure mode shouldn't cause a full site outage in the first place if there are at least two DCs per site — so part of the RCA might well be confirming that redundancy design, not just fixing the immediate DCs."
