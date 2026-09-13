# 22. KEY LOGS, TOOLS & MONITORING LOCATIONS — Quick Reference
### Where to actually look, per technology, when something breaks.

---

## ACTIVE DIRECTORY / DOMAIN CONTROLLERS

| Log / Tool | Location / Command | What it tells you |
|---|---|---|
| Directory Service log | Event Viewer > Applications and Services Logs > Directory Service | AD-specific errors, replication issues |
| System log | Event Viewer > Windows Logs > System | Service start/stop, general OS issues |
| Security log | Event Viewer > Windows Logs > Security | Event ID 4740 (lockout), 4771 (Kerberos pre-auth failed), 4768 (TGT requested) |
| Replication health | `repadmin /replsummary`, `repadmin /showrepl` | Replication failures between DCs |
| Full DC health | `dcdiag /v` | Comprehensive DC diagnostic |
| FSMO roles | `netdom query fsmo` | Which DC holds which role |
| Time sync | `w32tm /query /status` | Current sync source and offset |

---

## DNS

| Log / Tool | Location / Command | What it tells you |
|---|---|---|
| DNS Server log | Event Viewer > Applications and Services Logs > DNS Server | Zone transfer errors, resolution failures |
| SRV record check | `nslookup -type=srv _ldap._tcp.dc._msdcs.<domain>` | Whether AD's locator records resolve |
| Zone info | `dnscmd /zoneprint <zone>` | Full zone content |
| Scavenging status | `Get-DnsServerScavenging` | Aging/scavenging configuration |

---

## DHCP

| Log / Tool | Location / Command | What it tells you |
|---|---|---|
| DHCP audit logs | `%windir%\System32\dhcp\DhcpSrvLog-<Day>.log` | Lease activity, denials, conflicts |
| Scope utilization | `Get-DhcpServerv4ScopeStatistics` | % of scope used, near-exhaustion warning |
| Lease details | `Get-DhcpServerv4Lease` | Active leases per scope |
| Failover status | `Get-DhcpServerv4Failover` | Partner sync health |

---

## GROUP POLICY

| Log / Tool | Location / Command | What it tells you |
|---|---|---|
| Group Policy Operational log | Event Viewer > Applications and Services Logs > Microsoft > Windows > GroupPolicy > Operational | Detailed GPO processing events, per-CSE errors |
| Client-side report | `gpresult /h report.html` (run on affected machine) | Exactly which GPOs applied/denied and why |
| SYSVOL replication | `dfsrdiag replicationstate` | DFS-R backlog/health for SYSVOL |
| PowerShell report | `Get-GPOReport -All -ReportType Html` | Bulk GPO reporting |

---

## HYBRID IDENTITY (ENTRA CONNECT)

| Log / Tool | Location / Command | What it tells you |
|---|---|---|
| Synchronization Service Manager | miisclient.exe on the Connect server | Object-level sync errors (duplicate attribute, hard-match failure) |
| Sync scheduler status | `Get-ADSyncScheduler` | Last run time, staging mode status |
| Entra Connect Health | Entra admin center > Entra Connect Health | Sync errors, PTA agent health, alerts |
| Event logs | Event Viewer on Connect server > Application log, source "Directory Synchronization" | Sync engine errors |

---

## ENTRA ID / CONDITIONAL ACCESS

| Log / Tool | Location / Command | What it tells you |
|---|---|---|
| Sign-in logs | Entra admin center > Monitoring > Sign-in logs | Per-sign-in detail, including the Conditional Access tab showing exactly which policies applied |
| What If tool | Entra admin center > Conditional Access > What If | Simulates policy evaluation for a given user/app/condition without a real sign-in |
| Audit logs | Entra admin center > Monitoring > Audit logs | Admin/config changes (role assignments, policy edits) |
| Risky sign-ins / Risky users | Entra ID Protection (P2) | Risk-based signals feeding Conditional Access |
| PowerShell | `Get-MgIdentityConditionalAccessPolicy` | Query CA policies via Graph |

---

## PKI / CERTIFICATES

| Log / Tool | Location / Command | What it tells you |
|---|---|---|
| Certificate chain check | Browser padlock > View Certificate > Certification Path | Where exactly the trust chain breaks |
| Chain build/verify | `certutil -verify -urlfetch <certfile>` | Full chain validation including CRL/AIA reachability |
| CDP/AIA URL test | `certutil -URL <certfile>` (GUI) | Tests each revocation/chain URL live |
| IIS binding | `netsh http show sslcert` | Which cert thumbprint is bound to which site/port |
| CA event logs | Event Viewer on the CA server > Application log | Issuance/revocation errors |
| CA database | Certification Authority MMC snap-in | Issued/revoked/pending certificate requests |

---

## INTUNE / AUTOPILOT

| Log / Tool | Location / Command | What it tells you |
|---|---|---|
| Win32 app install logs | `C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\IntuneManagementExtension.log` | Win32 app install attempts/failures on the device |
| MDM diagnostic report | `mdmdiagnosticstool.exe -area Autopilot;DeviceEnrollment;DeviceProvisioning -cab diag.cab` | Comprehensive enrollment/Autopilot diagnostic package |
| Enrollment event log | Event Viewer > Applications and Services > Microsoft > Windows > DeviceManagement-Enterprise-Diagnostics-Provider | Enrollment/ESP failure codes |
| Device install status | Intune admin center > Devices > [device] > Device/User install status | Per-app, per-policy install status |
| Join/registration status | `dsregcmd /status` (run on device) | Azure AD Join / device registration state |

---

## CITRIX VIRTUAL APPS AND DESKTOPS

| Log / Tool | Location / Command | What it tells you |
|---|---|---|
| Citrix Director | Director web console | Logon duration phase breakdown (Broker/Boot/HDX/Profile/GPO), session details, machine health |
| Broker session status | `Get-BrokerSession` (Citrix PowerShell SDK) | Active session details |
| Machine registration | `Get-BrokerMachine` | VDA registration state |
| Historical trends | Citrix Analytics / `Get-LogSummary` | Logon duration trends over time |

---

## MICROSOFT 365 / EXCHANGE ONLINE

| Log / Tool | Location / Command | What it tells you |
|---|---|---|
| Message Trace (summary) | Exchange admin center > Mail flow > Message trace, or `Get-MessageTrace` | High-level path/status of a message |
| Message Trace (detailed) | `Get-MessageTraceDetail -MessageTraceId <id> -RecipientAddress <addr>` | Per-hop timestamps — where delay actually occurred |
| Unified Audit Log | Microsoft Purview compliance portal, or `Search-UnifiedAuditLog` | Mailbox rule creation, sign-ins, admin actions — critical for compromised mailbox investigation |
| Quarantine | Microsoft 365 Defender portal > Email & collaboration > Review > Quarantine | Held messages pending review/release |
| DMARC/DKIM/SPF reports | DMARC aggregate report emails (rua= address), or Defender portal | Authentication pass/fail trends, abuse attempts |
| Service health | Microsoft 365 admin center > Service health | Microsoft-side outages/incidents |

---

## AZURE — COMPUTE / NETWORKING / STORAGE

| Log / Tool | Location / Command | What it tells you |
|---|---|---|
| Boot Diagnostics | Azure Portal > VM > Boot diagnostics | Screenshot/log of actual console output |
| Serial Console | Azure Portal > VM > Serial console | Live text console even without network |
| Effective Security Rules | Azure Portal > NIC > Effective security rules, or `Get-AzEffectiveNetworkSecurityGroup` | Actual merged NSG rules applying to that NIC |
| Effective Routes | Azure Portal > NIC > Effective routes, or `Get-AzNetworkWatcherEffectiveRoute` | Actual merged routing table |
| VM status | `Get-AzVM -Status` | Power/provisioning state |
| Diagnostic settings | Resource > Diagnostic settings blade, or `Get-AzDiagnosticSetting` | Whether logs are actually being sent to Log Analytics |
| Log Analytics (KQL) | Azure Portal > Log Analytics workspace > Logs | Deep queryable investigation across resources |
| Network Watcher | Azure Portal > Network Watcher > Connection Troubleshoot / IP Flow Verify | Programmatic reachability/rule testing |
| VSS writers (in-guest, for Backup issues) | `vssadmin list writers` (run inside the VM) | App-consistent backup readiness |

---

## AZURE — COST / GOVERNANCE

| Log / Tool | Location / Command | What it tells you |
|---|---|---|
| Azure Advisor | Azure Portal > Advisor > Cost | Free automated cost-saving recommendations |
| Cost Management | Azure Portal > Cost Management + Billing | Spend breakdown, budgets, forecasts |
| Policy compliance | Azure Portal > Policy > Compliance | Which resources violate which policies |
| Activity Log | Azure Portal > Monitor > Activity log | Who did what, when, at the control-plane level (resource create/delete/modify) |

---

## SECURITY (DEFENDER SUITE)

| Log / Tool | Location / Command | What it tells you |
|---|---|---|
| Defender for Endpoint | Microsoft 365 Defender portal > Endpoints | Device risk, EDR alerts, threat detections |
| Defender for Identity | Microsoft 365 Defender portal > Identities | On-prem AD attack detection (Pass-the-Hash, DCSync), sensor health |
| Defender for Cloud | Azure Portal > Defender for Cloud | Resource security posture, workload protection alerts |
| Defender XDR | Microsoft 365 Defender portal > Incidents | Cross-domain correlated incident view |

---

## HOW TO USE THIS FILE
When an interviewer asks "what would you check" for any scenario, naming the SPECIFIC tool/command from this list (not just "I'd check the logs") is what signals real hands-on experience versus memorized theory. Practice pairing each scenario from file 21 with the exact tool from this file you'd use to investigate it.
