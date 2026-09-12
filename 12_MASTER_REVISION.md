# MASTER REVISION — Hybrid Cloud & Workplace Infrastructure Lead (L3)

## 1. MASTER ARCHITECTURE (how everything connects in a typical enterprise)

```
On-Prem AD (DC, DNS, DHCP, GPO)
        ↓
   Entra Connect (PHS / PTA / Federation)
        ↓
     Entra ID (Tenant, Users, Groups, Roles)
        ↓
  Conditional Access (MFA, Device Compliance, Named Locations, Risk)
        ↓
   ┌────────────┬─────────────┬──────────────┐
   ↓            ↓             ↓              ↓
Microsoft 365   Intune/       Citrix VAD    Azure IaaS/PaaS
(Exchange,      Autopilot     (session      (VMs, Storage,
Teams,          (device       brokering,     Networking,
SharePoint)     mgmt, ESP)    HDX, profile)  FinOps, Backup)
   ↓            ↓             ↓              ↓
PKI/Certificates underpin: TLS for all of the above, client cert auth,
S/MIME, internal web apps, 802.1x
   ↓
ITSM (ServiceNow) + Incident/Problem/Change Management wraps around all of it
```

**How to narrate this in an interview:** "Identity is the foundation — on-prem AD syncs to Entra ID via Entra Connect, and Conditional Access is the control plane that decides who gets into everything downstream: M365, Citrix, Azure resources, and Intune-managed endpoints. PKI underpins trust for nearly every one of those connections. And all of it operates inside an ITIL-based incident/change process so changes don't become outages."

## 2. MASTER TROUBLESHOOTING FRAMEWORK (use this structure for ANY scenario question — it works as a script)

**Understand → Scope → Dependencies → Configuration → Connectivity → Authentication → Logs → Test → Root Cause → Fix → Validate → Prevent**

How to apply it out loud in an interview (this is your answer template for almost every scenario question you'll get):
1. "First I'd clarify the actual symptom and when it started." (Understand)
2. "Then I'd check if it's one user, one site, or everywhere." (Scope)
3. "I'd check the hard dependencies first — DNS, network, time sync, licensing." (Dependencies)
4. "Then the relevant configuration — policy, GPO, CA policy, NSG, whatever applies." (Configuration)
5. "I'd verify actual connectivity/reachability between the components involved." (Connectivity)
6. "I'd check whether authentication itself is succeeding or failing." (Authentication)
7. "I'd pull the specific logs/tools for this technology." (Logs)
8. "I'd test in isolation — one affected account/device/resource — before assuming it's fixed everywhere." (Test)
9. "Once I've narrowed it to one clear cause, not just a plausible one." (Root Cause)
10. "I'd apply the fix at the source, not just the symptom." (Fix)
11. "I'd validate with a real test, not just 'looks fine.'" (Validate)
12. "And I'd close the loop with monitoring/alerting or a process change so it doesn't recur." (Prevent)

**Use this out loud, in this order, for every scenario question — it demonstrates structured L3 thinking even under pressure, and it's genuinely how experienced engineers work.**

## 3. TOP 40 HIGH-VALUE INTERVIEW QUESTIONS (curated across all topics, ranked)

🔴 Critical (expect these):
1. VM shows Running but unreachable via RDP — full diagnostic sequence.
2. Password changes on-prem not reflecting in M365 for some accounts — diagnose.
3. Design a Conditional Access policy for contractors vs employees with different MFA requirements.
4. Certificate renewal causes trust errors for some users only — full diagnosis.
5. 200 Autopilot devices stuck at ESP — triage in order.
6. Reduce Azure spend 20% without impacting SLA — prioritized approach.
7. Walk through a real PowerShell automation with error handling.
8. DCs in one datacenter stop authenticating — full incident response with comms.
9. Difference between PHS, PTA, and Federation, including resilience trade-offs.
10. Slow Citrix logons in one Delivery Group — isolate root cause using Director.
11. Walk through Kerberos authentication end-to-end.
12. Explain Conditional Access evaluation order and why break-glass exclusion matters.
13. A VIP's mail is intermittently delayed — full diagnosis beyond Message Trace.
14. Difference between Configuration Profiles and Compliance Policies in Intune.
15. Difference between Reserved Instances and Savings Plans.

🟠 High:
16. What are FSMO roles and what breaks if each is unavailable?
17. How do you troubleshoot GPO not applying?
18. What is staging mode in Entra Connect and why is it dangerous?
19. What's the difference between CRL and OCSP?
20. Difference between MCS and PVS provisioning in Citrix?
21. What logs would you check for a failed Win32 app deployment via Intune?
22. Is VNet peering transitive?
23. What is Azure Advisor and what does it check for cost?
24. Difference between Incident, Problem, and Change management?
25. What replaced the deprecated MSOnline/AzureAD PowerShell modules?
26. What is Continuous Access Evaluation?
27. External sharing works at tenant level but not for a specific site — why?
28. What is Report-only mode in Conditional Access and why use it?
29. Difference between Azure Backup and Azure Site Recovery?
30. What is a Known Error in ITIL?

🟡 Medium:
31. What is an RODC and when would you use one?
32. What is Seamless SSO and what does it depend on?
33. What is authentication strength vs standard MFA requirement?
34. What is a Named Location's actual security limitation?
35. Difference between Terraform and PowerShell for infra tasks?
36. What is Azure Arc used for in hybrid server management?
37. What's the difference between Metrics and Logs in Azure Monitor?
38. What is a Landing Zone?
39. What is DHCP failover — load balance vs hot standby?
40. What is FSLogix and why does profile share placement matter for Citrix logon speed?

## 4. TOP 25 "WHAT HAPPENS IF...?" — QUICK-FIRE ANSWERS

1. **DC fails (multi-DC site)?** → Transparent failover to another DC; if it's the only DC, total site auth outage.
2. **DNS fails?** → AD can't locate DCs — auth breaks even though AD service itself may be healthy.
3. **Entra Connect stops?** → No new syncs; PHS-cached passwords still work in cloud until changed on-prem, then a gap appears.
4. **PHS sync stops?** → Cloud sign-in still works with last-synced password; new on-prem changes won't reflect until sync resumes.
5. **All PTA agents go down?** → Total cloud sign-in failure — no fallback, unlike PHS.
6. **AD FS farm goes down?** → Total federated sign-in failure — worst blast radius of the three hybrid auth methods.
7. **Conditional Access blocks all admins (no break-glass excluded)?** → Complete lockout — must use break-glass or emergency Microsoft support process to recover.
8. **A CA policy is deployed without Report-only testing?** → Risk of unexpected lockouts for legitimate users matching the new condition.
9. **An intermediate CA cert isn't distributed to some clients?** → Those clients show "not trusted" even though the cert itself is valid.
10. **CRL/OCSP is unreachable?** → Depending on client's hard-fail/soft-fail setting, connections either fail entirely or proceed with a warning.
11. **Intune enrollment can't reach Graph endpoints (proxy blocks it)?** → Autopilot/Intune enrollment fails or hangs at ESP.
12. **A Win32 app's detection rule is wrong?** → Intune reports failure even if install succeeded — blocks ESP if marked required.
13. **FSLogix profile share has high latency?** → Slow Citrix/RDS logons specifically at the "Profile Load" phase.
14. **DHCP scope is exhausted?** → New devices can't get an IP; existing leases unaffected until renewal.
15. **SYSVOL/DFS-R replication fails?** → GPOs stop applying consistently across the domain.
16. **Time sync breaks (>5 min skew)?** → Kerberos authentication fails silently — looks like a random auth outage.
17. **VNet peering exists A-B and B-C?** → A cannot reach C — peering is non-transitive.
18. **A UDR misroutes traffic to a downed firewall appliance?** → VM appears "Running" but is completely unreachable over network.
19. **No diagnostic setting configured on an Azure resource?** → No logs appear in Log Analytics even though monitoring "should" work.
20. **A CA (Conditional Access) policy conflicts with another (one blocks, one grants)?** → Block always wins.
21. **RID pool is exhausted?** → Can't create new AD objects domain-wide from that DC.
22. **A site-level SharePoint sharing setting is more restrictive than tenant default?** → Site-level always wins (can't be more permissive than the site allows).
23. **A patch is deployed to 100% of production simultaneously?** → No pilot/canary group to catch issues — highest blast-radius risk in patch management.
24. **RCA has no assigned preventive actions?** → Incomplete RCA — root cause identified but recurrence isn't actually prevented.
25. **A disabled AD user isn't removed from cloud-synced groups?** → Stale access persists in M365/Azure resources tied to that group even after AD disablement — common audit finding.

## 5. FINAL NIGHT-BEFORE CHEAT SHEET (60-90 min revision)

**Identity chain:** AD → Entra Connect (PHS/PTA/Fed) → Entra ID → Conditional Access → downstream apps.
Resilience ranking if on-prem goes down: **PHS > PTA > Federation** (PHS has cloud-side fallback, others don't).

**Conditional Access — the one thing to get right:** Named Location = signal, not proof. Always exclude break-glass. Always Report-only first. Grant vs Session controls are different things.

**PKI — the one thing to get right:** Trust errors after renewal on SOME users = check for a changed intermediate CA not yet distributed to those clients. Compare Certification Path between working/failing client.

**Intune/Autopilot — the one thing to get right:** ESP stuck → check Intune "Device install status" FIRST to see which app/policy is the blocker, before touching logs.

**Citrix — the one thing to get right:** Director's logon phase breakdown (Broker/Boot/HDX/Profile/GPO) tells you which layer is actually slow — don't guess.

**Azure VM unreachable — the one thing to get right:** Boot Diagnostics + Serial Console FIRST (tells you OS vs network problem) before diving into NSGs.

**FinOps — the one thing to get right:** Free wins first (Advisor, zombie cleanup) → data-driven rightsizing → RI/Savings Plans for stable workloads → schedule non-prod → tier storage → govern with tags/budgets going forward.

**AD — the one thing to get right:** DNS → Time → Network → Replication → FSMO, checked in that order, for almost any "random" auth issue.

**Major incident — the one thing to get right:** Restore service first, root-cause after; fixed-cadence stakeholder updates even when there's no news; RCA needs assigned owners/deadlines, not just a write-up.

**Universal troubleshooting script (say this structure out loud for any scenario question):**
Understand → Scope → Dependencies → Configuration → Connectivity → Authentication → Logs → Test → Root Cause → Fix → Validate → Prevent

**Terminology currency check:** Graph PowerShell SDK (`Get-MgUser`) replaced deprecated MSOnline/AzureAD modules — mention this if PowerShell for M365/Entra ID comes up, it signals you're current.

---

*Files in this set: 01_active_directory.md, 02_hybrid_identity.md, 03_entra_id_conditional_access.md, 04_pki_certificates.md, 05_intune_autopilot.md, 06_citrix.md, 07_microsoft_365.md, 08_azure_administration.md, 09_azure_finops.md, 10_powershell_automation.md, 11_windows_server_itsm.md, 12_MASTER_REVISION.md (this file).*

*Recommended revision order given your interview performance: read 03, 04, 05, 06 first (your weakest areas), then 12 (master revision) the morning of the interview, then skim 01/02/07/08/09/10/11 as refreshers on areas you're already stronger in.*
