# 21. DAY-TO-DAY OPERATIONAL SCENARIOS — Short, Interview-Ready Answers
### Real situations an L3 handles routinely. Format: Scenario → Short Answer (say this in an interview).

---

# ACTIVE DIRECTORY / DNS / DHCP / GPO

**Q: A user calls saying they can't log in, but it's fine for everyone else on their team.**
A: Check if it's account lockout (`Get-ADUser -Identity <user> -Properties LockedOut`), check last bad password attempt via Event ID 4740 on the DC, check if their password expired, and confirm which DC they're authenticating against isn't isolated from replication.

**Q: Password reset on-prem doesn't reflect in Outlook/M365 for one user.**
A: Force a delta sync (`Start-ADSyncSyncCycle -PolicyType Delta`), check Synchronization Service Manager for an object-specific error (usually a duplicate UPN/proxyAddress), confirm the account is in the correct OU scoped for sync.

**Q: A new server can't be created — "RID pool exhausted" type error.**
A: Check FSMO RID Master health (`netdom query fsmo`), verify it's online and reachable, and check the domain's RID pool consumption — may need to raise the RID block size or fix RID Master connectivity.

**Q: GPO shows updated in the console, but clients aren't getting the new setting.**
A: Run `gpresult /h report.html` on an affected client first — check if it's Denied (security filtering), check `dfsrdiag replicationstate` for SYSVOL replication lag between the console's DC and the client's DC.

**Q: A specific building's new laptops can't get an IP address.**
A: Check `Get-DhcpServerv4ScopeStatistics` for that scope's utilization — if exhausted, expand the scope or reclaim stale leases; if the scope has room, check DHCP relay/IP helper config on that building's router.

**Q: Two remote sites both report login failures at the same time.**
A: Check DNS SRV record resolution from both sites first, then `repadmin /replsummary` for replication health, then time sync (`w32tm /query /status`) — don't assume it's the DC hardware until dependencies are ruled out.

**Q: A GPO works for most of an OU but not a specific subset of machines.**
A: Check WMI filtering on the GPO — likely evaluating false for that subset (e.g., an OS version query written incorrectly); confirm with `gpresult /h` on one of the excluded machines.

**Q: A decommissioned server's old IP is now causing intermittent wrong-host resolution.**
A: Classic stale DNS record issue — check if scavenging is enabled/configured correctly on that zone; manually remove the stale record and verify scavenging settings going forward.

---

# HYBRID IDENTITY (ENTRA CONNECT)

**Q: Sync hasn't run in 24 hours — what's your first check?**
A: `Get-ADSyncScheduler` to confirm the scheduler is enabled and check last run time; check if StagingModeEnabled is accidentally true; check Entra Connect Health portal for alerts.

**Q: A user was disabled in AD but still has active M365 sessions.**
A: Disabling AD doesn't instantly kill live sessions/tokens — force sync, then explicitly revoke sessions (`Revoke-MgUserSignInSession`) since token validity can outlast the directory change until natural expiry or CAE kicks in.

**Q: All PTA agents show "Inactive" in the portal.**
A: This is a hard outage for PTA-based sign-in with no cloud fallback (unlike PHS) — check agent servers are online, check outbound 443 connectivity to Microsoft endpoints, restart the Azure AD Application Proxy Connector service if needed.

---

# ENTRA ID / CONDITIONAL ACCESS

**Q: A new CA policy just went live and admins are locked out.**
A: Use a break-glass account (should always be excluded from every CA policy) to sign in and disable/fix the offending policy immediately; this is exactly why break-glass exclusion is mandatory practice, not optional.

**Q: An external contractor got into SharePoint without MFA.**
A: Check Sign-in logs → Conditional Access tab for that specific sign-in to see which policy applied/didn't; likely the contractor's group wasn't correctly scoped, or a broader policy's exclusion accidentally covers them.

**Q: A traveling executive keeps getting flagged as "impossible travel" and blocked.**
A: Check Identity Protection's Risky sign-ins report, confirm it's a false positive, dismiss the risk, and consider a travel-aware named location or an authentication-strength exception rather than disabling risk detection entirely.

**Q: You need to roll out a new CA policy without risking a lockout.**
A: Deploy in Report-only mode first, review the Sign-in logs for who WOULD be affected, adjust scope/exclusions, then enable for real — never go live untested on a policy with "All users/All apps" scope.

---

# PKI / CERTIFICATES

**Q: An internal app throws trust errors for some users right after a cert renewal.**
A: Compare Certification Path on a failing vs working client — almost always a new Issuing/Intermediate CA cert that hasn't reached the failing clients via GPO auto-enrollment; push the missing intermediate.

**Q: One office's users get certificate errors, others don't.**
A: Check CRL/OCSP reachability from that specific office's network segment — likely a firewall/proxy blocking the revocation-check URL for that site only.

**Q: A certificate is about to expire on a production web server with no one aware until now.**
A: Immediate: renew and rebind before expiry using `netsh http show sslcert` to confirm current binding, replace, validate. Long-term: implement expiry monitoring/alerting (30/60/90-day) so this doesn't happen again — that's the process gap, not just the cert.

---

# INTUNE / AUTOPILOT

**Q: A batch of new laptops is stuck at ESP "Account setup."**
A: Check Intune > Device install status for the specific app/policy blocking it first; if ALL devices are affected, suspect network/proxy blocking Graph/Intune endpoints; if a subset, suspect one Win32 app's detection rule.

**Q: A device shows "Not Compliant" and the user says nothing changed.**
A: Check the Device Compliance blade for which specific check failed — most common causes are BitLocker key not escrowed properly, or a failed Windows Update dropping the OS below minimum version.

**Q: Onboarding is being blocked and IT needs a fast unblock, not a perfect fix.**
A: Temporarily set the ESP profile to non-blocking (or de-scope the failing app from "required") to get people to a working desktop today; fix the actual root cause offline; validate with a small pilot batch before trusting it fleet-wide.

---

# CITRIX

**Q: One Delivery Group has slow logons this week, no config changes made.**
A: Go straight to Citrix Director's logon duration phase breakdown — if Profile Load is the slow phase, check FSLogix share latency/IOPS (often a storage issue, not Citrix); if HDX connection is slow, it's network path; if Brokering is slow, check DDC/database load.

**Q: A single user can't log in to Citrix, everyone else on the same infrastructure is fine.**
A: Suspect a corrupted FSLogix profile container for that specific user — check if their container fails to mount; a fresh/repaired container usually resolves it.

**Q: External users can't connect via Gateway, internal users are fine.**
A: Check the Citrix Gateway's own certificate first — an expired cert on the Gateway is the classic cause of an "external only" symptom, since internal traffic doesn't route through it.

---

# MICROSOFT 365 / EXCHANGE / MAIL SECURITY

**Q: A VIP's email is intermittently delayed.**
A: Pull `Get-MessageTraceDetail`, not just the summary trace, to see exact per-hop timestamps; check for a mail flow rule scoped specifically to that VIP (moderation/DLP rules add delay); check if attachments are triggering Safe Attachments detonation.

**Q: A legitimate marketing platform's emails are landing in spam/failing delivery.**
A: Check if that platform's sending IP is included in SPF (`include:` statement) and whether DKIM is configured for it — most likely it was never added when the tool was onboarded.

**Q: You're asked to move DMARC from monitor-only to enforcement.**
A: Review DMARC aggregate reports first to confirm every legitimate sending source (including third-party platforms) passes and aligns before moving to `p=reject`, or you risk blocking your own legitimate mail.

**Q: A user reports a suspicious email was clicked and now says "Password reset asking me for MFA constantly."**
A: Treat as a compromised mailbox — immediately disable the account/revoke sessions, then search the audit log specifically for a newly created inbox rule (common persistence technique), remove it, then reset credentials and re-register MFA.

**Q: A recipient says they never got an expected email; sender says it was sent successfully.**
A: Check Quarantine — likely sitting there with no notification reaching the recipient; check quarantine notification config and release permissions.

---

# AZURE

**Q: A VM shows "Running" in the portal but nobody can RDP into it.**
A: Boot Diagnostics first — tells you if it's an OS problem (stuck boot, BSOD) or purely network; if OS is healthy, check Effective Security Rules and Effective Routes (not just configured NSG/UDR) for a black-holed path.

**Q: Finance asks you to cut Azure spend by 20% without breaking anything in production.**
A: Start with Azure Advisor (free wins), rightsize based on weeks of utilization data, apply Reserved Instances/Savings Plans to stable production workloads, auto-shutdown non-prod outside hours, tier storage, and use tags/budgets to keep it from creeping back up.

**Q: Two VNets that need to talk to each other can't be peered.**
A: Most likely overlapping IP address spaces — check both VNets' address ranges; peering fails outright if they overlap.

**Q: A backup job keeps showing "Completed with Warnings."**
A: Check `vssadmin list writers` inside the guest — usually a VSS writer issue causing a fallback to crash-consistent backup rather than app-consistent, not a fault in Azure Backup itself.

**Q: A team says a resource "isn't logging anything" in Log Analytics.**
A: Check if a Diagnostic Setting is actually configured on that specific resource — logs don't flow anywhere by default; this is almost always a missing diagnostic setting, not a broken monitoring pipeline.

---

# INCIDENT / CHANGE MANAGEMENT

**Q: A major incident is declared — DCs at one site are down.**
A: Restore service first (failover auth to another site's DCs if possible), classify severity and open a bridge call, communicate on a fixed cadence to stakeholders even without new information, THEN do RCA once service is stable — never skip the fixed-cadence comms even during active troubleshooting.

**Q: An RCA identified a fix but it never actually got implemented.**
A: RCA action items need a named owner and a deadline tracked in the same ITSM record until closed, ideally reviewed in a recurring problem-management meeting — an RCA without assigned follow-through is incomplete, not just a documentation issue.

**Q: A routine patch caused an unexpected production outage.**
A: This is exactly why patches should go through a pilot/canary ring before full fleet rollout — never patch 100% of production simultaneously; the incident RCA should specifically address whether the patch ring process was followed.

---

## HOW TO USE THIS FILE
These are phrased exactly as they'd come up in daily operations, not textbook questions — practice saying each answer out loud in under 30 seconds. If you can answer all of these fluently without hesitation, you're ready for the operational/scenario portion of an L3 panel.
