# 7. MICROSOFT 365 ADMINISTRATION (weak-to-thin area on your resume — build this up)

## 7.1 Exchange Online — Mail Flow Fundamentals

**How it works — inbound mail flow:**
Internet → DNS MX record for domain → Exchange Online Protection (EOP) — spam/malware filtering → Transport Rules (mail flow rules) evaluated → Defender for Office 365 (if licensed — Safe Links/Safe Attachments) → Mailbox delivery.

**Outbound:** Mailbox → Transport Rules → EOP outbound filtering (checks for compromised-account send patterns) → Internet, via connectors if routing through on-prem/hybrid first.

**Components:** MX record, EOP, Mail Flow (Transport) Rules, Connectors (inbound/outbound, used heavily in hybrid Exchange), Accepted Domains, Message Trace.

## 7.2 Diagnosing Intermittent Mail Delay (fixes gap from your Q5 answer)

**Beyond Message Trace (which you named), a full L3 approach checks:**
1. **Message Trace with full detail** — `Get-MessageTrace` / `Get-MessageTraceDetail` in EXO PowerShell — shows every hop and timestamp, not just pass/fail.
2. **Queue vs delivery time comparison** — is the delay happening at acceptance, at a transport rule evaluation step, or at final mailbox delivery? The detailed trace shows this.
3. **Mail flow/transport rules scoped to that specific user** — a rule with a "moderate messages" or "redirect" action targeting just that VIP (common for executive protection/DLP scenarios) can silently add delay.
4. **EOP/tenant throttling limits** — recipient rate limits or unusual sending-pattern throttling could apply if the VIP's mailbox is also used for a lot of automated sends (calendar bots, assistants using shared access).
5. **Connector health** (if hybrid) — check connector logs for delays specifically on the on-prem↔cloud path if mail is routed through an on-prem Exchange server for compliance/journaling reasons.
6. **Client-side factors** — since it's "intermittent," rule out Outlook cached mode sync delay, large OST corruption, or a slow/unstable network on the VIP's specific device — not every mail delay is server-side.
7. **Defender for Office 365 scanning time** — Safe Attachments detonation can add real (sometimes multi-minute) delay for attachments — check if the VIP frequently receives attachments from new/unrecognized senders, triggering deeper scanning.

**Root cause narrowing:** compare timestamps across the full trace detail — if the gap is between "Transport rule evaluated" and "delivered," suspect a rule or Defender scanning; if the gap is before acceptance, suspect throttling or DNS/connector issues.

## 7.3 Teams (core admin points)

**Common admin areas:** Teams Admin Center policies (messaging, calling, meeting policies), Direct Routing / Calling Plans for PSTN, Teams-SharePoint integration for files, external access / guest access settings, Teams meeting security defaults (lobby, presenter/attendee roles).

**Failure scenario:** External guest can't join Teams meeting → check External Access (federation) settings, meeting policy "who can bypass lobby" setting, and (if pertinent) Conditional Access blocking guest sign-in.

## 7.4 SharePoint Online / OneDrive

**Key architecture point:** OneDrive is technically a personal SharePoint site collection per user — same underlying platform, different UX. Sync client (OneDrive.exe) uses Microsoft Graph/SharePoint REST under the hood.

**Common issue:** Sync errors/conflicts — check OneDrive sync client diagnostic logs, check for file path length limits (260 char legacy limit, though modern clients handle longer paths in most cases), check for blocked file types by tenant DLP/compliance policy.

**External sharing:** Controlled at tenant level (SharePoint Admin Center) AND per-site level — commonly misconfigured when a site owner can't share externally despite tenant allowing it, because the specific site's sharing setting is more restrictive than the tenant default (site-level always overrides down, never up).

## 7.5 Common Exchange/M365 PowerShell (Graph/EXO V2 module)

```
Get-MessageTrace -SenderAddress <addr> -StartDate <> -EndDate <>
Get-MessageTraceDetail -MessageTraceId <id> -RecipientAddress <addr>
Get-TransportRule
Get-MailboxStatistics <user>
Get-MailFlowStatusReport
Get-MgUser (Graph, replaces older Get-AzureADUser / Get-MsolUser — both deprecated)
```

**Important currency note for interview:** MSOnline and AzureAD PowerShell modules are deprecated (retirement in progress/complete) — Microsoft Graph PowerShell SDK (`Get-MgUser`, `Update-MgUser`, etc.) is the current standard. Mentioning this shows you're current, since many candidates still reference the old modules.

## INTERVIEW QUESTIONS

🔴 Walk through inbound mail flow from internet to mailbox delivery.
🔴 A VIP's mail is intermittently delayed — full diagnostic approach beyond Message Trace.
🟠 What's the difference between a Mail Flow Rule and a Defender for Office 365 policy?
🟠 How does external sharing get controlled at tenant vs site level in SharePoint?
🟡 What replaced the deprecated MSOnline/AzureAD PowerShell modules?
🔴 Scenario: external guest cannot join a Teams meeting — diagnose.
🟠 Scenario: a site owner can't share externally despite tenant policy allowing it — why?

## RED FLAGS / TRICK QUESTIONS
- "Does disabling a user in on-prem AD immediately kill their M365 access?" — Not necessarily immediate; depends on sync cycle (up to 30 min delta, or immediate if you force `Start-ADSyncSyncCycle`) plus existing token validity until Continuous Access Evaluation or token expiry revokes active sessions.
- "Is OneDrive separate infrastructure from SharePoint?" — No, it's the same platform (personal site collections).

## L3 ANSWER (60-120s)
"For an intermittent VIP mail delay, I don't stop at basic Message Trace — I pull the full trace detail to compare timestamps between each hop, because that tells me whether the delay is at acceptance, at a transport rule, or at final delivery. I also specifically check for any mail flow rule scoped to that user — VIP mailboxes often have moderation, redirect, or DLP rules attached for executive protection that regular users don't have, and those add processing time. If attachments are involved, Defender for Office 365 Safe Attachments detonation can add real delay, especially from new senders. And since it's described as intermittent rather than constant, I don't rule out the client side either — Outlook cached mode sync issues or OST problems on that specific device are a common cause that looks like a server-side delay but isn't."

## MEMORY TRICK
**Mail flow = MX → EOP → Rules → Defender → Mailbox — delay could live at any one of these four checkpoints, check each independently.**
