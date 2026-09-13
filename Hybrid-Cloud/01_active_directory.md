# 1. ACTIVE DIRECTORY DOMAIN SERVICES (AD DS)

## 1.1 Domain Controllers

**What is it?**
Simple: The server that holds the "master rulebook" of who exists and who can access what.
Technical: A server running AD DS that stores a writable (or RODC read-only) copy of the domain's directory partition (Schema, Configuration, Domain NC) and provides authentication (Kerberos/NTLM) and directory lookup (LDAP) services.

**Why used?**
Centralizes identity, authentication, and policy enforcement instead of managing accounts per-machine. Enables SSO within the domain, centralized security policy (GPO), and a single source of truth for objects (users, computers, groups).

**How it works — flow**
Client boot → DNS SRV lookup (`_ldap._tcp.dc._msdcs.<domain>`) → locates nearest DC (site-aware via Subnet-to-Site mapping) → Kerberos AS-REQ/AS-REP → TGT issued → client requests TGS for services → access granted based on Kerberos ticket + group membership.

**Components:** NTDS.dit (database), SYSVOL (policy/scripts share), LSASS (auth process), KDC (Kerberos Key Distribution Center service running on every DC), Global Catalog (forest-wide partial attribute set), DNS (locator service), FSMO roles.

**Dependencies:** DNS (absolute — DC location fails without it) → Time sync (Kerberos allows max 5 min skew) → Network connectivity (RPC/LDAP/Kerberos ports) → Replication topology (KCC-generated or manual).

**What happens when it works:** User logs in → DNS resolves DC → Kerberos pre-auth succeeds → TGT issued → GPO applies via SYSVOL → user gets desktop + correct group-based access.

**What happens when it breaks:**
- Single DC down (multi-DC site): usually transparent — clients failover to another DC via DNS/site awareness. Symptom: slower logons if failover DC is in another site.
- All DCs in a site down: total auth outage for that site — users see "cannot connect to domain," GPO stops applying, no new Kerberos tickets issued (cached tickets on clients still work until TTL expiry, ~10hrs default).
- FSMO role holder down: usually invisible short-term (5 of 5 roles are only needed for specific ops — e.g., PDC emulator for time sync/password changes, RID master for new SID allocation). Symptom appears later: "cannot create new objects" (RID pool exhausted) or password change failures.

**Troubleshooting (L3 method)**
1. Understand: one user, one site, or everywhere? Recent change (patch, network, firewall)?
2. Scope: `nltest /dsgetdc:<domain>` from affected client to see which DC it's using.
3. Check DNS first — always. `nslookup <domain>`, verify SRV records exist and resolve to healthy DC IPs.
4. Check DC health: `dcdiag /v`, `repadmin /replsummary`, `repadmin /showrepl`.
5. Check event logs: Directory Service, DNS Server, System (look for NTDS KCC/replication errors, Kerberos errors 4771/4768).
6. Check time sync: `w32tm /monitor` — Kerberos fails silently if skew > 5 min.
7. Check network: `Test-NetConnection <DC> -Port 389` (LDAP), `-Port 88` (Kerberos), `-Port 445` (SMB for SYSVOL).
8. Root cause narrows to: DNS / replication / network / time / FSMO / disk space on NTDS volume.
9. Fix based on root cause (see command list).
10. Validate: `dcdiag /v` clean, `repadmin /replsummary` shows 0 failures, test login from affected client.
11. Prevent: monitoring on replication latency, DC diskspace, at least 2 DCs per site, documented DR runbook.

**Key commands**
```
dcdiag /v                          → full DC health check
repadmin /replsummary              → quick replication health across all DCs
repadmin /showrepl                 → detailed replication partners/status for one DC
repadmin /syncall /AdeP            → force replication push
nltest /dsgetdc:<domain>           → which DC is a client using
nltest /sc_query:<domain>          → secure channel test
w32tm /query /status               → time sync status
netdom query fsmo                  → show FSMO role holders
```

**Real-world example:** "5,000-user org, auth failures reported at two sites simultaneously." → Check if both sites share a WAN link back to a hub site with the only writable DC (single point of failure) → `repadmin /showrepl` reveals replication has been failing for 3 days due to a firewall rule change blocking RPC dynamic ports → DCs are stale/isolated → fix firewall, force `repadmin /syncall`, validate.

**Common mistakes:** Restarting DC services blindly without checking replication first (can worsen inconsistency); ignoring time sync as a cause of "random" auth failures; not checking whether the issue is site-specific (assuming it's forest-wide).

**Interview questions**
- Basic: What is a Domain Controller? What is SYSVOL used for? What is the Global Catalog?
- Intermediate: What are FSMO roles and what happens if each is unavailable? How does DNS integrate with AD?
- L3: How does the KCC build replication topology? What's the difference between USN and high-watermark in replication? How do you troubleshoot lingering objects?
- Scenario: "One site's DCs are online but users can't log in" — walk through diagnosis. "Replication has been failing silently for 2 weeks" — how do you find and fix it?
- Architecture: How would you design DC placement for a global org with 20 sites? How do you plan for DC disaster recovery?

**Ideal answer (30-60s):** "A Domain Controller hosts a writable or read-only copy of AD, handles Kerberos and LDAP authentication, and replicates changes to other DCs via a multi-master model. Clients locate the nearest DC through DNS SRV records and AD Sites and Services, which is why DNS health is the first thing I check in any auth issue."

**L3 answer (60-120s):** "When I get an auth outage, I don't touch anything until I've scoped it — one user vs one site vs global tells me whether it's a client issue, a site-local DC issue, or a replication/FSMO issue. My first two checks are always DNS resolution of the SRV records and `repadmin /replsummary`, because 90% of the 'random' AD issues I've hit trace back to either DNS or silent replication failures — usually caused by a firewall change blocking dynamic RPC ports, or time skew breaking Kerberos pre-auth. I also check disk space on the NTDS volume, because a full volume halts the JET database and looks exactly like a random outage. Once I isolate root cause I fix at the source — never just restart NTDS blindly, because that can mask or worsen replication inconsistency. For prevention, I put automated alerting on replication latency and DC free space, and make sure every site has at least two DCs so a single failure doesn't cause a site-wide outage."

**Memory trick:** DNS → Time → Network → Replication → FSMO — check in that order for any AD issue.

**10-second revision:**
- DC = database (NTDS.dit) + KDC (Kerberos) + LDAP + replication partner
- DNS failure = DC location failure = auth failure
- `repadmin /replsummary` and `dcdiag /v` are your first two commands, always
- FSMO roles fail silently — symptoms appear later (RID exhaustion, time drift)

**Red flag / trick question:** "If one DC goes down, do users immediately lose access?" — No, if there's another healthy DC in the site or the client can reach one in another site (with slower logon). Only a single-DC site/environment causes immediate total outage.

---

## 1.2 Group Policy (GPO)

**What is it?** A mechanism to centrally push configuration and security settings to users/computers.
Technical: XML/registry.pol-based settings stored in SYSVOL + AD GPO container objects, linked to OUs/domains/sites, applied via Group Policy Client Service using LDAP + SMB (SYSVOL).

**How it works:** Computer boot/user logon → client queries AD for linked GPOs (respecting site > domain > OU order, with Block Inheritance / Enforced overrides) → downloads GPO files from SYSVOL via SMB → applies Client-Side Extensions (CSEs) in order → background refresh every 90-120 min (randomized).

**Dependencies:** AD (link data) → SYSVOL/DFS-R or FRS replication (policy files) → DNS/Kerberos (auth to fetch) → WMI filters (optional scoping).

**Failure scenarios:** GPO not applying — could be SYSVOL replication failure (DFS-R backlog), WMI filter evaluating false, security filtering excluding the user/computer, GPO link disabled, "Enforced" conflict, slow-link detection (VPN) skipping policy.

**Troubleshooting:**
```
gpresult /h report.html      → shows exactly which GPOs applied/denied and why
gpupdate /force              → force refresh
gpresult /r                  → quick summary in console
Get-GPOReport -All -ReportType Html   → PowerShell equivalent
dfsrdiag replicationstate    → check SYSVOL replication (DFS-R) backlog
```
Check event log: Group Policy Operational log (Applications and Services Logs → Microsoft → Windows → GroupPolicy → Operational).

**Real-world example:** New security baseline GPO not applying to a subset of laptops → `gpresult /h` shows GPO listed under "Denied" due to WMI filter mismatch (filter checks OS version, laptops were on a different build) → fix WMI filter, validate via gpresult.

**Interview:** Difference between GPO enforcement order (LSDOU: Local, Site, Domain, OU) and "Enforced"/"Block Inheritance" interactions is a classic L3 question — Enforced always wins even over Block Inheritance further down.

**Memory trick:** LSDOU = Local → Site → Domain → OU (last writer wins, except Enforced always wins).

---

## 1.3 DNS (AD-integrated)

**What is it?** The locator service without which AD literally cannot function — DCs are found via DNS SRV records, not IP.

**Key subtopics:** AD-integrated zones (replicated via AD, multi-master), SRV records (`_ldap`, `_kerberos`, `_gc`), scavenging (removing stale records), forwarders/conditional forwarders (external resolution), zone transfer security.

**Failure scenario:** Stale/missing SRV records after a DC rename or IP change → clients can't locate DC → intermittent-seeming auth failures at specific sites.

**Commands:**
```
nslookup -type=srv _ldap._tcp.dc._msdcs.<domain>
Resolve-DnsName -Type SRV _kerberos._tcp.<domain>
dnscmd /enumzones
dnscmd /zoneprint <zone>
Get-DnsServerScavenging
```

**L3 answer:** "DNS is a hard dependency for AD, not just a network service — if SRV records are wrong or the zone hasn't replicated, Kerberos and LDAP simply can't locate a DC, and that looks exactly like an authentication outage even though AD itself is healthy. So DNS is always my first check, before I even open dcdiag."

---

## 1.4 DHCP

**Key subtopics:** Scopes, reservations, superscopes, failover (load-balance/hot-standby), DHCP relay/IP helper, lease process (DORA: Discover-Offer-Request-Acknowledge).

**Failure scenario:** Scope exhaustion — new devices can't get IPs, existing leases unaffected until renewal. Symptom: intermittent "limited connectivity" on new devices only.

**Commands:**
```
Get-DhcpServerv4Scope
Get-DhcpServerv4ScopeStatistics   → shows % utilization, flags near-exhaustion
Get-DhcpServerv4Lease
ipconfig /release & /renew
```

**Interview trap:** "Does DHCP failover give you load balancing automatically?" — Only if configured in Load Balance mode; Hot Standby mode is active/passive, not load-balanced.

---

## END-TO-END FLOW: AD DS

**User logs into a domain-joined PC:**
Power on → DHCP assigns IP → DNS SRV lookup finds nearest DC (Sites/Subnets) → Kerberos AS-REQ/AS-REP (time sync required) → TGT issued → GPO settings pulled from SYSVOL (DFS-R replicated) → user's group memberships evaluated → access granted to resources per Kerberos service tickets.

## BREAK/FIX MASTER TABLE

| Component | Purpose | Dependency | If it breaks | Symptoms | First Check | Command | Fix |
|---|---|---|---|---|---|---|---|
| DNS SRV records | DC location | AD-integrated zone | Clients can't find DC | Auth failures site-wide | nslookup SRV | `nslookup -type=srv` | Fix zone/replication |
| Replication | Sync directory data | Network, DNS, time | Stale objects, inconsistent GPO | Password changes not visible everywhere | repadmin /replsummary | `repadmin /showrepl` | Fix network/firewall, force sync |
| Time sync | Kerberos ticket validity | PDC emulator, NTP | Auth fails silently (skew >5min) | "Clock skew" errors, random auth failures | w32tm /query /status | `w32tm /resync` | Fix NTP source hierarchy |
| SYSVOL/DFS-R | GPO/script distribution | AD, DNS, DFS-R service | GPOs don't apply | Policy inconsistency | dfsrdiag replicationstate | `Get-GPOReport` | Fix DFS-R backlog |
| DHCP scope | IP assignment | Network, AD (optional auth) | New devices can't get IP | New device "limited connectivity" | Get-DhcpServerv4ScopeStatistics | — | Expand scope / add reservations |

## TOP 20 INTERVIEW QUESTIONS (AD DS) — ranked

🔴 What is the AD replication topology and how does KCC build it?
🔴 Walk me through Kerberos authentication step by step.
🔴 What are FSMO roles and what breaks if each is unavailable?
🔴 How do you troubleshoot GPO not applying?
🔴 DNS SRV records — what are they and why does AD depend on them?
🟠 Difference between AD-integrated and standalone DNS zones?
🟠 What is a lingering object and how do you fix it?
🟠 What is USN rollback and why is it dangerous?
🟠 Explain LSDOU and GPO enforcement order.
🟠 What's the difference between DFS-R and the legacy FRS for SYSVOL replication?
🟡 What is an RODC and when would you use one?
🟡 How does DHCP failover work — load balance vs hot standby?
🟡 What is SYSVOL and what happens if it stops replicating?
🟡 How do you recover a deleted OU with AD Recycle Bin?
🟡 What is a Global Catalog and why does the forest need one?
🔴 Scenario: two sites both report login failures at once — walk through your process.
🔴 Scenario: replication has silently failed for 2 weeks — how do you find and fix it?
🟠 Scenario: a GPO applies to 90% of a group but not the other 10% — why?
🟠 Scenario: RID pool almost exhausted — what do you do?
🟡 Architecture: design DC placement across 15 global sites with WAN constraints.
