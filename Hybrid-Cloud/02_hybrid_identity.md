# 2. HYBRID IDENTITY (Entra Connect / Sync)

## 2.1 Microsoft Entra Connect (formerly Azure AD Connect)

**What is it?**
Simple: The bridge that copies your on-prem AD accounts into the cloud (Entra ID) and keeps them in sync.
Technical: A sync engine (based on MIIS/FIM lineage) that runs Connector Space imports/exports between on-prem AD and Entra ID via the Sync Engine + Azure AD Connector, using a defined set of connector spaces and metaverse rules.

**Why used?** Enables hybrid identity — one set of credentials for on-prem AND cloud (M365, Azure) resources — without duplicate account management. Solves "which password do I use where" problem.

**How it works (flow):**
On-prem AD object created/changed → Entra Connect Delta/Full Import (reads from AD via LDAP) → staged in Connector Space → Sync rules project to Metaverse → Export to Entra ID Connector Space → Export to Entra ID via Graph/legacy sync API → object appears in Entra ID (as a "synced" object, not editable in cloud portal for most attributes).

**Components:** Sync Engine (miiserver.exe), SQL Server/LocalDB (stores connector space + metaverse), Connector Spaces (AD CS, Entra ID CS), Metaverse, Synchronization Rules Editor, Scheduler (default 30 min delta cycle).

**Dependencies:** On-prem AD (source) → Network/firewall (outbound 443 to Entra ID endpoints) → Entra ID tenant → SQL (local or full SQL for HA) → the sync server itself (single point unless staging mode secondary configured).

## 2.2 Password Hash Sync (PHS)

**What is it?** Simple: A hash-of-a-hash of the user's AD password is synced to Entra ID every 2 minutes, so cloud auth doesn't need to reach on-prem.
Technical: Entra Connect extracts the AD password hash (already an MD4/NTLM hash), hashes it again with a per-user salt using PBKDF2 (1000 iterations of HMAC-SHA256), and syncs that to Entra ID.

**Why used?** Simplest, most resilient hybrid auth method — cloud auth works even if on-prem network/AD is down (since Entra ID has its own copy of the hash).

**What happens if it breaks:** Sync stops → users can still authenticate to Entra ID with their LAST synced password until they change it on-prem, then there's a gap until next sync (default every 2 min for password changes specifically, faster than the standard 30-min delta).

## 2.3 Pass-Through Authentication (PTA)

**What is it?** Cloud sends the login attempt back to an on-prem PTA Agent (lightweight software), which validates against AD directly — no password hash stored in cloud.

**Why used?** Compliance requirement to never store any password material in the cloud; also enforces on-prem AD sign-in restrictions (e.g., logon hours) in real time.

**Dependency chain:** Entra ID → PTA Agent (must be on a domain-joined server, outbound 443 only) → on-prem DC.

**Failure scenario:** If ALL PTA agents are down (recommend minimum 3 for HA), cloud sign-ins fail completely — this is the key risk vs PHS, which has a cloud-side fallback. This is a critical interview distinction.

## 2.4 Federation (AD FS)

**What is it?** On-prem AD FS servers issue SAML/WS-Fed tokens; Entra ID trusts AD FS as the identity provider instead of authenticating directly.

**Why less common now:** More complex, more infrastructure to maintain and secure (AD FS + WAP servers), single point of failure if not built with redundancy. Microsoft actively recommends migrating to PHS/PTA + Conditional Access instead. Still exists in older/regulated enterprises — interviewers may ask you to compare it.

**Failure scenario:** AD FS farm down = ALL federated cloud sign-ins fail (Entra ID has zero fallback since it doesn't own the credential at all) — worse blast radius than PTA.

## 2.5 Seamless SSO

**What is it?** A computer object (AZUREADSSOACC) created in on-prem AD lets domain-joined, on-network devices silently authenticate to Entra ID without a password prompt — works alongside PHS or PTA (not standalone).

## COMPARISON TABLE — Critical for interview

| Method | Password stored in cloud? | On-prem dependency for cloud login | Resilience if on-prem down | Complexity |
|---|---|---|---|---|
| PHS | Yes (hash of hash) | No | High — cloud auth still works | Low |
| PTA | No | Yes (agent + DC) | Low — fails if no agent reachable | Medium |
| Federation (AD FS) | No | Yes (AD FS farm) | Lowest — total outage if farm down | High |

## TROUBLESHOOTING METHODOLOGY

**Step 1 — Understand:** Is it sync not happening, or sign-in failing? These are different problems (sync engine vs auth path).

**Step 2 — Scope:** All users or specific OU/group? Recently modified accounts only, or everyone?

**Step 3 — Check sync health:**
```
Get-ADSyncScheduler                       → shows sync cycle status, last run
Start-ADSyncSyncCycle -PolicyType Delta   → force a delta sync
Get-ADSyncConnectorRunStatus              → check for stuck/running connector
```
Open **Synchronization Service Manager** (miisclient.exe) → Operations tab → look for red X errors (object-level: duplicate attribute, hard-match failure on UPN/proxyAddress).

**Step 4 — Check Entra Connect Health portal** (if configured) — shows sync errors, agent health (for PTA), alerts on stale sync.

**Step 5 — Check staging mode:** `Get-ADSyncScheduler` shows `StagingModeEnabled` — if accidentally true on the active server, NO exports happen at all (classic silent-failure cause when someone set up a secondary server for DR and never confirmed which is primary).

**Step 6 — For PTA specifically:** Check PTA agent status in Entra admin center (Hybrid Identity > Connect > Pass-through authentication) — agents show Active/Inactive.

**Step 7 — Root cause typically one of:** staging mode on wrong server, object-level sync error (duplicate proxyAddress/UPN across on-prem and cloud-only object), firewall blocking outbound 443, expired Entra Connect certificate/credentials, AD FS farm cert expired (if federated).

**Step 8 — Fix:** Correct staging mode, resolve duplicate attribute conflict in Sync Manager, re-authenticate connector credentials, renew certs.

**Step 9 — Validate:** Force delta sync, confirm object shows updated attribute in Entra ID via `Get-MgUser` or portal, test actual sign-in.

**Step 10 — Prevent:** Alerting on Entra Connect Health, documented failover runbook for staging server, regular cert-expiry monitoring for AD FS/PTA agents.

## REAL-WORLD EXAMPLE

"Password changes on-prem aren't reflecting in M365 for SOME accounts, others sync fine." → This is almost never a PHS engine failure (PHS syncs continuously); check per-user: are affected accounts in the correctly-scoped OU for sync (OU filtering in Entra Connect)? Check Sync Manager Operations for object-specific errors — most likely a **duplicate proxyAddress or UPN mismatch** causing those specific objects to fail export while others succeed. Fix the duplicate attribute, re-run delta sync, validate.

## COMMON MISTAKES
Junior approach: restart Entra Connect service blindly. L3 approach: check Sync Manager for object-level errors first — most "some users affected" issues are per-object data conflicts, not a systemic sync failure.

## INTERVIEW QUESTIONS

🔴 What's the difference between PHS, PTA, and Federation — including resilience trade-offs?
🔴 Walk me through what happens end-to-end when Entra Connect syncs a password change.
🔴 What is staging mode and why is it dangerous if misconfigured?
🟠 What happens if all PTA agents go down?
🟠 How do you troubleshoot a "some users sync, others don't" issue?
🟡 What is Seamless SSO and what does it depend on?
🟡 How would you migrate from Federation to PHS with minimal disruption?

## FOLLOW-UP DRILLING (example)
**Interviewer:** "How does Entra Connect work?"
Possible drill-down: What happens if sync stops for a week? → What's a hard-match vs soft-match failure? → How do you recover from accidental staging mode on production server? → Difference between Connect and Entra Cloud Sync (lighter-weight, agent-based, no full sync engine, used for simpler/multi-forest scenarios)?

## L3 ANSWER (60-120s)
"Hybrid identity comes down to picking the right auth method for the resilience and compliance profile you need. PHS is my default recommendation for most orgs — it syncs a hash of the hash to Entra ID, so cloud sign-in keeps working even during an on-prem outage, which is the opposite of Federation, where an AD FS farm outage kills cloud sign-in entirely. PTA sits in between — no password material leaves on-prem, but you're trading that for a dependency on PTA agents being reachable. When something breaks, I don't assume it's the whole sync engine — I check Synchronization Service Manager first for object-level errors, because 'some users sync, others don't' is almost always a duplicate UPN or proxyAddress conflict on specific objects, not a systemic failure. I've also seen staging mode left on accidentally on a production server after someone built a DR secondary — that silently stops all exports and looks exactly like a total sync failure, so I always check `Get-ADSyncScheduler` for that first."

## MEMORY TRICK
**PHS** = Password (hash) stays synced, cloud works alone.
**PTA** = Pass-Through, on-prem Agent must answer.
**Federation** = Farm owns everything, farm down = everything down.

## 10-SECOND REVISION
- PHS = most resilient (cloud has its own copy)
- PTA = needs an agent up, no password stored in cloud
- Federation = highest complexity, highest blast radius on failure
- "Some users not syncing" → check Sync Manager for object-level errors, not the whole engine
- Staging mode = silent killer if left on by accident
