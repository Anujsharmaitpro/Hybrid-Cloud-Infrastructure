# 19. ACTIVE DIRECTORY / DNS / DHCP / GPO — Point-by-Point Breakdown
### Format for every item: **What is it? → What is it used for? → What issues can it cause (if broken/misconfigured)?**

---

# SECTION A — ACTIVE DIRECTORY CORE

## * Domain Controller (DC)

**What is it?**
A server running AD DS holding a writable (or read-only, for RODC) copy of the directory, and running the KDC (Kerberos Key Distribution Center) service for authentication.

**What is it used for?**
Authenticates users/computers, stores directory objects (users, groups, computers), and is the anchor for GPO application and directory lookups (LDAP).

**What issues can it cause?**
- **Single DC per site, and it goes down** — complete authentication outage for that site (no failover option locally); clients may reach a DC in another site over WAN, but with slower logon and potential timeout issues on slow links.
- **DC disk full (NTDS volume)** — halts the JET database engine; looks exactly like a random, unexplained authentication outage if disk space isn't the first thing checked.
- **DC time drift** (see Time Sync below) — causes Kerberos failures that look like random, intermittent auth problems rather than an obvious "DC is down" symptom.

---

## * FSMO Roles (Flexible Single Master Operations)

**What is it?**
Five specific operations that can only be performed by ONE DC at a time (not multi-master like most AD operations): Schema Master, Domain Naming Master (forest-wide, one each); PDC Emulator, RID Master, Infrastructure Master (per-domain, one each).

**What is it used for?**
- **PDC Emulator:** Time sync source for the domain, password change urgency processing, default target for many GPO edits.
- **RID Master:** Allocates blocks of unique Relative IDs so new objects (users/computers) can be created with unique SIDs domain-wide.
- **Infrastructure Master:** Keeps cross-domain group membership references updated (relevant in multi-domain forests).
- **Schema Master:** Only one DC can perform schema extensions (e.g., during Exchange/certain application installs).
- **Domain Naming Master:** Controls adding/removing domains from the forest.

**What issues can it cause?**
- **PDC Emulator unavailable** — time sync degrades domain-wide (since it's the authoritative time source), which cascades into Kerberos failures if left unresolved for an extended period; password change urgency handling also degrades.
- **RID Master unavailable for an extended period** — eventually the local RID pool on other DCs is exhausted, and NO NEW objects (users, computers, groups) can be created anywhere in the domain — a silent, delayed-onset failure that doesn't show up until pools run dry.
- **FSMO role holder physically decommissioned without properly transferring/seizing the role first** — leaves the domain without that critical function until an admin manually seizes the role onto another DC — a common disaster-recovery scenario question.

---

## * AD Replication (Multi-Master Model)

**What is it?**
The process by which changes made on any writable DC propagate to all other DCs, using a topology automatically built by the KCC (Knowledge Consistency Checker) based on site links and cost.

**What is it used for?**
Ensures every DC eventually has a consistent view of the directory — critical because AD is multi-master (any DC can accept a write), so replication is how consistency is maintained afterward.

**What issues can it cause?**
- **Replication silently failing** (firewall change blocking RPC dynamic ports, a decommissioned DC not properly removed from the topology, network link down) — DCs become increasingly inconsistent over time; symptoms include password changes not appearing everywhere, GPO changes inconsistent across sites, and eventually **lingering objects** (deleted objects that never got the deletion replicated to an isolated DC, then get "resurrected" when connectivity is restored, unless tombstone lifetime handling catches it first).
- **USN Rollback** — occurs when a DC is improperly restored from an old snapshot/backup (e.g., a VM snapshot restore instead of a proper AD-aware backup) — the DC's Update Sequence Numbers become inconsistent with what other DCs believe it already replicated, causing serious, hard-to-diagnose replication corruption; this is exactly why VM-level snapshot restores of DCs are strongly discouraged without VSS-aware/AD-aware backup tools.
- **Replication latency across WAN-linked sites** — normal to some degree (this is why site link costs/schedules exist), but if it grows far beyond expected convergence time, indicates an underlying network or KCC topology problem.

---

## * Time Synchronization

**What is it?**
The hierarchy where the PDC Emulator (per domain) is the authoritative time source (ideally itself synced to an external NTP source), and all other DCs/domain members sync from the domain hierarchy automatically via the Windows Time service (W32Time).

**What is it used for?**
Kerberos authentication REQUIRES time to be within a default 5-minute skew between client and DC — time sync isn't a "nice to have," it's a hard authentication dependency.

**What issues can it cause?**
- **Skew exceeds 5 minutes** — Kerberos pre-authentication fails, producing what looks like a completely unrelated, random authentication failure — this is one of the most commonly MISSED root causes in real-world AD troubleshooting precisely because the symptom (auth failure) doesn't obviously point to "check the clock."
- **PDC Emulator not synced to a reliable external source** — the entire domain's time baseline can drift collectively, especially in virtualized environments where hypervisor time sync and Windows Time service can conflict if both are trying to manage the guest's clock simultaneously (a well-known VMware/Hyper-V + AD gotcha).
- **Time zone vs UTC confusion in troubleshooting** — Kerberos actually works in UTC internally; skew issues aren't about "wrong time zone displayed" but actual UTC time difference — worth knowing precisely for an L3-level answer.

---

## * SYSVOL

**What is it?**
A shared folder on every DC (replicated via DFS-R, or legacy FRS in older environments) containing GPO template files, logon scripts, and other domain-wide policy content.

**What is it used for?**
The actual file-based backing store for Group Policy — the AD database stores GPO metadata/links, but the actual policy settings/scripts live in SYSVOL.

**What issues can it cause?**
- **DFS-R replication backlog/failure** — GPOs become inconsistent across sites/DCs; a client might get an old or missing version of a GPO depending on which DC it authenticates against — directly causes "GPO applies inconsistently" symptoms.
- **SYSVOL share unreachable** (SMB port blocked, DC role issue) — GPO processing fails entirely for clients pointed at that DC, even if AD authentication itself succeeded.
- **Migration from legacy FRS to DFS-R done incorrectly** — can leave SYSVOL in an inconsistent state; this migration should be a deliberate, monitored process (`dfsrmig` states), not something left incomplete.

---

# SECTION B — DNS

## * AD-Integrated DNS Zones

**What is it?**
DNS zone data stored within AD itself and replicated via normal AD replication (rather than traditional zone transfers) — multi-master, meaning any DC can accept DNS updates.

**What is it used for?**
Provides the SRV records (`_ldap`, `_kerberos`, `_gc`) that clients use to locate DCs — this is a genuinely hard dependency: AD literally cannot function without DNS resolving correctly.

**What issues can it cause?**
- **Missing or stale SRV records** (after a DC rename, IP change, or decommission not properly cleaned up) — clients at a given site can't locate a DC, producing symptoms that look exactly like an authentication outage even though AD itself is perfectly healthy.
- **Zone replication scope misconfigured** (e.g., set to replicate only to DCs in the same domain when it needs forest-wide visibility, or vice versa) — some DCs/sites have incomplete DNS data, causing intermittent, site-specific resolution failures.
- **Secure dynamic updates causing update failures** — if a client's computer account lacks proper permissions or there's a Kerberos issue preventing secure dynamic DNS updates, its record can go stale, again feeding back into "clients can't locate the right DC" type issues.

---

## * DNS Scavenging

**What is it?**
An automated cleanup process that removes stale DNS records (records that haven't been refreshed within a configured aging period) — prevents zones from accumulating outdated entries for decommissioned or renamed machines indefinitely.

**What is it used for?**
Keeps DNS zones accurate over time as devices come and go (especially relevant with DHCP-assigned dynamic records) — without it, zones slowly fill with dead entries that can cause resolution to the wrong/decommissioned host.

**What issues can it cause?**
- **Scavenging too aggressive** (short aging periods) — can remove records for devices that are simply offline for an extended period (e.g., a laptop on long leave) but still legitimately exist, causing unexpected resolution failures when that device returns.
- **Scavenging not enabled at all** — stale records accumulate indefinitely; a decommissioned server's old IP might get reassigned to a new device, and the stale DNS entry causes clients to intermittently resolve to the WRONG host.
- **Scavenging enabled inconsistently across zones/DCs** — some DNS servers scavenge, others don't, leading to inconsistent record staleness across replicas.

---

## * Forwarders / Conditional Forwarders

**What is it?**
**Forwarders** send queries the internal DNS server can't answer (e.g., internet domains) to an external/upstream DNS server. **Conditional Forwarders** do the same but ONLY for specific named domains (e.g., forward all `partner.com` queries to the partner's DNS server specifically), while everything else follows normal resolution.

**What is it used for?**
Lets internal AD-integrated DNS servers resolve external names without being internet-facing themselves, and lets specific cross-organization/cross-forest name resolution work correctly (common in M&A scenarios, hybrid on-prem/cloud environments, or partner network integrations).

**What issues can it cause?**
- **Forwarder unreachable** — internal clients can resolve internal AD names fine but can't reach external sites/services — a common "internal stuff works, internet doesn't" symptom that points straight at the forwarder configuration, not general DNS health.
- **Conditional forwarder misconfigured or missing** for a needed cross-domain resolution — causes name resolution failures specifically for that partner/other-forest domain, while everything else resolves normally — narrows the diagnosis significantly once you know to check this.
- **Forwarder pointing to a DNS server that itself has become unreliable** — intermittent external resolution failures that look random until you specifically test the forwarder path.

---

# SECTION C — DHCP

## * DHCP Scopes

**What is it?**
A defined range of IP addresses (with associated options — gateway, DNS servers, lease duration) that a DHCP server can hand out to clients on a given subnet.

**What is it used for?**
Automates IP address assignment so devices don't need static configuration — the DORA process (Discover, Offer, Request, Acknowledge) is the actual exchange that happens at lease time.

**What issues can it cause?**
- **Scope exhaustion** — no more addresses available; NEW devices requesting an address fail to get one (symptom: "limited connectivity" or APIPA 169.254.x.x address), while EXISTING devices with active leases are unaffected until their own renewal fails too — a classic "only new devices affected" symptom.
- **Scope options misconfigured** (wrong gateway or DNS server pushed to clients) — devices get an IP but can't route correctly or can't resolve names — looks like "no internet" but is actually a DHCP option error, not a routing or DNS server health issue per se.
- **Overlapping scopes on two DHCP servers without proper split/failover coordination** — can cause duplicate IP address assignment conflicts on the network.

---

## * DHCP Reservations

**What is it?**
A specific IP address permanently tied to a specific device's MAC address within a scope — ensures that device always gets the SAME IP via DHCP rather than a random available one.

**What is it used for?**
Needed for devices that require a consistent, predictable IP (printers, certain servers, network appliances) without resorting to full static IP configuration on the device itself — keeps IP management centralized in DHCP even for "static-feeling" devices.

**What issues can it cause?**
- **Reservation IP falls outside the actual scope range** (after a scope was resized) — reservation silently stops being honored, and the device may get a different IP unexpectedly.
- **MAC address changed** (hardware replacement, NIC swap) without updating the reservation — device gets a different, unreserved IP instead of its expected reserved one.

---

## * DHCP Failover (Load Balance vs Hot Standby)

**What is it?**
A mechanism for two DHCP servers to share responsibility for a scope for redundancy. **Load Balance mode** splits responses roughly evenly between both servers actively. **Hot Standby mode** has one server as primary (active) and the other purely as a passive backup that only responds if the primary is unreachable.

**What is it used for?**
DHCP redundancy — without failover, a single DHCP server outage means no new IP assignments possible anywhere that scope serves.

**What issues can it cause?**
- **Assuming Hot Standby provides load balancing** — it doesn't; it's active/passive only — a common misconception that leads to incorrect capacity planning (assuming both servers share load when only one is actually serving requests under normal conditions).
- **Failover relationship desynchronized** (partner servers' lease databases fall out of sync) — can cause inconsistent behavior, duplicate assignments, or one partner not properly taking over if the other fails.
- **DHCP relay/IP helper misconfigured** on routers for multi-subnet environments — clients on a remote subnet without a local DHCP server, and without a properly configured relay pointing to the central DHCP server, get no IP address at all — a network-side (not DHCP server-side) root cause that's often mistaken for a DHCP server problem.

---

# SECTION D — GROUP POLICY (GPO)

## * GPO Application Order (LSDOU)

**What is it?**
The order in which Group Policy is applied: **L**ocal policy → **S**ite-linked GPOs → **D**omain-linked GPOs → **O**rganizational **U**nit-linked GPOs (innermost OU applied last, generally "winning" on conflicting settings, EXCEPT for Enforced policies).

**What is it used for?**
A predictable, layered mechanism for how settings from different scopes combine and, when they conflict, which one takes precedence.

**What issues can it cause?**
- **Assuming OU-level GPOs always win** — they normally do for CONFLICTING settings (last-applied wins), but an **Enforced** GPO at a higher level (Domain or Site) always overrides even a conflicting OU-level setting, regardless of normal LSDOU order — a very common source of "why isn't my OU policy taking effect" confusion.
- **Block Inheritance at an OU** — stops normal inheritance from higher levels, EXCEPT Enforced GPOs still apply anyway — another commonly misunderstood interaction that causes unexpected policy behavior.

---

## * Security Filtering & WMI Filtering

**What is it?**
**Security Filtering** restricts WHICH users/computers within a GPO's linked scope actually have the GPO applied, based on Read + Apply Group Policy permissions (default: Authenticated Users, but can be scoped to specific security groups). **WMI Filtering** restricts application based on a queried condition evaluated on the target machine (e.g., "only apply if OS is Windows 11").

**What is it used for?**
Fine-grained targeting beyond just OU structure — lets you scope a GPO to a specific subset of an OU's members without restructuring OUs.

**What issues can it cause?**
- **Security filtering misconfigured** (removed "Authenticated Users" without adding the intended replacement group) — GPO link exists and looks correctly linked, but applies to NOBODY — a classic "why isn't this GPO working at all" scenario that `gpresult` will reveal as "Denied" due to security filtering.
- **WMI filter querying an attribute that doesn't evaluate as expected** (e.g., an OS version check written for the wrong build number format) — GPO silently doesn't apply to machines it should, with no obvious error beyond checking `gpresult`'s detailed output.
- **WMI filter evaluation adds slight processing time at logon** — usually negligible, but with many complex WMI filters across many GPOs, can measurably add to logon/boot time.

---

## * GPO Replication and Version Consistency

**What is it?**
The requirement that a GPO's AD-stored metadata (version number) and its SYSVOL-stored actual policy files stay in sync across all DCs — a mismatch between the two indicates a replication problem.

**What is it used for?**
Ensures clients apply a complete, consistent version of a GPO rather than a partially-replicated or mismatched combination of old/new settings.

**What issues can it cause?**
- **AD replication and SYSVOL/DFS-R replication fall out of sync with each other** (they're technically two separate replication mechanisms even though they work together for GPO) — a client might see the GPO metadata (in AD) as updated, but pull the old policy FILES from SYSVOL if DFS-R hasn't caught up yet — a subtle "why does the GPO show the new setting in the console but not actually apply it" scenario.
- **Slow WAN links delaying SYSVOL replication specifically more than AD object replication** — creates a window where different sites are legitimately running different GPO versions temporarily, which is normal in transit but problematic if assumed to be instant.

---

## * Group Policy Client-Side Extensions (CSEs)

**What is it?**
The actual components on a client machine responsible for interpreting and applying specific categories of GPO settings (Registry CSE, Security CSE, Scripts CSE, Software Installation CSE, etc.) — each GPO setting type has a corresponding CSE that must process it.

**What is it used for?**
Modularizes GPO processing — different setting types are handled by their relevant specialized component rather than one monolithic engine.

**What issues can it cause?**
- **A specific CSE fails/errors** — only settings of THAT type fail to apply while everything else in the same GPO applies normally — a nuanced diagnostic point: "the GPO partially applied" is a real, specific failure mode, not just "the GPO worked or didn't."
- **Slow-link detection** — GPO processing over a detected slow connection (e.g., VPN) can SKIP certain CSEs by design (e.g., Software Installation is skipped over slow links by default) — this is often mistaken for a bug when it's actually intended behavior to avoid a huge software push over a poor connection.

---

## END-TO-END FLOW: Full domain logon incorporating AD + DNS + DHCP + GPO

```
Device powers on -> DHCP DORA process assigns IP + DNS server + gateway
   |
Client queries DNS for SRV record (_ldap._tcp.dc._msdcs.<domain>) to locate nearest DC (via Sites/Subnets)
   |
Kerberos AS-REQ/AS-REP exchange (requires time sync within 5 min skew)
   |
TGT issued -> client is authenticated to the domain
   |
Client queries AD for linked GPOs (Site > Domain > OU, respecting Enforced/Block Inheritance)
   |
Client evaluates Security Filtering + WMI Filtering to determine which GPOs actually apply to it
   |
Client pulls actual GPO policy files from SYSVOL (via SMB, DFS-R replicated)
   |
Relevant Client-Side Extensions process and apply each setting type
   |
User reaches desktop with correct network config (DHCP), correct policy (GPO), authenticated (Kerberos)
```

## BREAK/FIX MASTER TABLE

| Component | Purpose | Dependency | If it breaks | Symptom | First Check | Command | Fix |
|---|---|---|---|---|---|---|---|
| DC / FSMO | Auth + directory ops | DNS, time, network | RID pool exhaustion, PDC unreachable | Can't create objects; time drift domain-wide | `netdom query fsmo` | `netdom query fsmo` | Seize/transfer role appropriately |
| AD Replication | Directory consistency | Network, DNS, time | Silent replication failure | Password changes not everywhere; lingering objects | `repadmin /replsummary` | `repadmin /showrepl` | Fix network/firewall, force sync |
| Time Sync | Kerberos validity | PDC Emulator, NTP | Skew > 5 min | "Random" auth failures | `w32tm /query /status` | `w32tm /resync` | Fix NTP hierarchy |
| SYSVOL/DFS-R | GPO file distribution | AD, DNS, DFS-R service | Replication backlog | GPOs inconsistent across sites | `dfsrdiag replicationstate` | `Get-GPOReport` | Resolve DFS-R backlog |
| DNS SRV records | DC location | AD-integrated zone | Stale/missing records | Auth failures at specific site | `nslookup -type=srv` | `Resolve-DnsName -Type SRV` | Fix zone/replication scope |
| DNS Scavenging | Zone hygiene | Aging config | Too aggressive or disabled | Stale records causing wrong resolution | Zone aging properties | `Get-DnsServerScavenging` | Tune aging period, enable consistently |
| DHCP Scope | IP assignment | Network | Exhaustion | New devices can't get IP; existing unaffected | `Get-DhcpServerv4ScopeStatistics` | — | Expand scope, add reservations, clean stale leases |
| DHCP Failover | Redundancy | Two DHCP servers | Desync or misunderstood mode | Duplicate assignment or no true load balance | Failover relationship status | `Get-DhcpServerv4Failover` | Resync partners, correct mode expectations |
| GPO Security Filtering | Scope GPO to specific principals | AD group membership | "Authenticated Users" removed by mistake | GPO applies to nobody | `gpresult /h report.html` | `Get-GPPermission` | Restore correct filtering group |
| GPO WMI Filter | Conditional scoping | WQL query accuracy | Filter evaluates unexpectedly | GPO silently skips intended machines | `gpresult /h` detailed output | — | Correct WMI filter query |

## TOP 20 INTERVIEW QUESTIONS — THIS CLUSTER

🔴 Walk through the complete flow from device boot to a fully policy-applied, authenticated desktop.
🔴 What are FSMO roles and what specifically breaks if each is unavailable for an extended period?
🔴 A GPO isn't applying to a specific OU but works everywhere else — full diagnosis using gpresult.
🔴 Explain LSDOU order and how Enforced/Block Inheritance actually interact with it.
🔴 DNS SRV records are missing/stale after a DC decommission — what breaks and how do you fix it?
🟠 What is USN Rollback and why is restoring a DC from a VM snapshot dangerous?
🟠 Difference between DHCP Failover Load Balance and Hot Standby modes?
🟠 What is DNS scavenging and what happens if it's too aggressive vs not enabled at all?
🟠 Why does time sync matter so much for Kerberos specifically, and what's the failure threshold?
🟡 What's the difference between AD replication and SYSVOL/DFS-R replication, and why can they be out of sync with each other?
🟡 What is a Conditional Forwarder and when would you use one over a standard forwarder?
🟡 What is a Client-Side Extension and how can a GPO "partially" apply?
🔴 Scenario: two sites report authentication failures simultaneously — walk through your full diagnostic process.
🔴 Scenario: replication has been silently failing for two weeks — how do you discover and fix it?
🟠 Scenario: a GPO shows updated in the console but the new setting isn't taking effect on clients — why?
🟠 Scenario: DHCP scope shows healthy utilization but new devices in one building specifically can't get an IP — what do you check?
🟡 Scenario: a decommissioned server's old IP was reassigned, and clients intermittently resolve to the wrong host — what's the root cause?
🔴 Architecture: design DC and DNS placement for an org with 15 sites of varying WAN quality.
🟠 Architecture: how would you plan DHCP redundancy across a multi-subnet campus with router-based relay?
🟡 What is an RODC (Read-Only Domain Controller) and when would you deploy one?

## RED FLAGS / TRICK QUESTIONS
- "Does DNS scavenging deleting a record mean that device is gone from AD?" — No, DNS and AD computer objects are separate; scavenging only removes the DNS RECORD, not the underlying AD object.
- "If one DC goes down, do users immediately lose all access?" — No, only if it's the sole DC in that site or the client can't reach any other DC — otherwise, transparent failover occurs (possibly slower if failing over cross-site).
- "Does restoring a DC from any backup work the same way?" — No — VM snapshot-style restores without VSS/AD-awareness can cause USN Rollback; only AD-aware backup tools (Windows Server Backup with System State, or third-party AD-aware backup) restore safely.
- "Is Hot Standby DHCP failover the same as load balancing?" — No, it's active/passive, not active/active.

## L3 ANSWER — Full Diagnostic Framework for "Random" Auth Failures
"When authentication issues look random or intermittent, my instinct isn't to start with AD itself — it's to check the dependencies AD relies on, in order: DNS first, because if SRV records are stale or a zone hasn't replicated, clients simply can't locate a DC, and that looks exactly like an auth outage even though AD is healthy. Then time sync, because Kerberos has a hard five-minute skew tolerance, and drift is one of the most commonly missed root causes precisely because the symptom doesn't obviously point back to 'check the clock.' Then I check actual replication health with `repadmin /replsummary`, because silent replication failures — often from a firewall change blocking RPC dynamic ports — cause exactly the kind of inconsistent, 'some users affected, others not' pattern that's otherwise hard to explain. Only after ruling out those dependencies do I look at the DC's own health and FSMO role status. This order matters because jumping straight to 'the DC is broken' without checking DNS and time sync first wastes time chasing symptoms instead of the actual dependency chain that's failing."

## MEMORY TRICK
**Auth failure troubleshooting order: DNS -> Time -> Network -> Replication -> FSMO.**
**GPO not applying: check gpresult FIRST — it tells you exactly why (Denied by security filtering, WMI filter false, not linked, Enforced conflict) instead of guessing.**
**LSDOU — but Enforced always wins regardless of level or Block Inheritance.**
**DHCP: Load Balance = active/active. Hot Standby = active/passive. Don't confuse them.**

## 10-SECOND REVISION
- DNS is a hard dependency for AD — stale SRV records look exactly like an auth outage
- Kerberos tolerates only 5 minutes of clock skew — time sync issues are commonly missed
- `repadmin /replsummary` and `gpresult /h` are your two fastest, highest-value diagnostic commands
- RID Master and PDC Emulator failures are SILENT and DELAYED — they don't announce themselves immediately
- Enforced GPOs always win, even over Block Inheritance — a very common misunderstanding
- Never restore a DC from a raw VM snapshot without VSS/AD-awareness — risk of USN Rollback
- Hot Standby DHCP failover is NOT load balancing — it's active/passive
