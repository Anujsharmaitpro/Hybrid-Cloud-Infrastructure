# 26. ANSWERS — Email Security (20 Questions)

**1. Third-party HR/payroll system sends "as" your domain and lands in spam — cause and fix?**
The platform's sending IP/domain was never added to your SPF record and likely isn't DKIM-signed for your domain — receiving servers can't verify it's authorized, so it gets flagged. Fix: add the platform's required `include:` to SPF, enable/configure DKIM signing for that sender if the platform supports it (often via a CNAME they provide), and confirm DMARC alignment isn't blocking it.

**2. SPF has 11 `include:` statements and mail starts failing across the board — what happened, fix?**
SPF has a hard limit of 10 DNS lookups; exceeding it causes a PermError, which most receivers treat as an SPF failure for ALL mail from that domain, not just the newest sender. Fix: consolidate includes (flatten nested SPF records, remove unused/duplicate sources) to get back under 10 lookups.

**3. `~all` vs `-all`, and why is leaving `~all` long-term a problem?**
`~all` is a soft fail — unauthorized mail is flagged but still typically delivered. `-all` is a hard fail — unauthorized mail is rejected outright. Leaving `~all` indefinitely means SPF isn't actually enforcing anything meaningful long-term; it should be a temporary state while confirming all legitimate senders are captured, then tightened to `-all`.

**4. Forwarding breaks DKIM validation even with nothing malicious happening — why?**
DKIM signs the message based on its exact original content/headers. When an intermediate server forwards the message and modifies anything (even something small like adding a disclaimer or rewriting a header), the signature no longer matches, and DKIM fails validation at the final destination — a known, non-malicious side effect of forwarding.

**5. Moving DMARC from `p=none` to `p=reject` — what first, and risk of skipping it?**
Review the DMARC aggregate reports first to confirm every legitimate sending source (including third-party platforms) is passing and properly aligned. Skipping this risks blocking your OWN legitimate mail — a marketing tool or CRM sending "as" your domain without proper SPF/DKIM alignment would suddenly get rejected once `p=reject` is enforced.

**6. SPF passes but DMARC still fails — how?**
DMARC requires ALIGNMENT, not just a pass. SPF can pass for the actual sending domain in the technical envelope (e.g., a subdomain or a third-party return-path domain), but if that doesn't match the visible "From" address the recipient sees, DMARC alignment fails even though SPF itself technically passed.

**7. EOP baseline vs Defender for Office 365 — what does the upgrade add?**
EOP (included in all Exchange Online plans) provides baseline anti-spam, anti-malware, and basic anti-phishing. Defender for Office 365 adds Safe Links (time-of-click URL scanning), Safe Attachments (sandbox detonation), advanced anti-phishing with impersonation/mailbox intelligence, and (Plan 2) Threat Explorer, automated investigation/response, and attack simulation training.

**8. Email with attachment takes 5+ minutes, nothing else wrong — likely cause?**
Safe Attachments detonation — the attachment is being sandboxed/detonated to check for malicious behavior before delivery, which can genuinely add several minutes of delay for larger or more complex files. This is expected behavior, not a fault, when Defender for Office 365 is licensed.

**9. Safe Links vs Safe Attachments — what does each catch that the other doesn't?**
Safe Links protects against malicious URLs, specifically ones that are benign at delivery time but weaponized later (checked at click-time, not just delivery-time). Safe Attachments protects against malicious files via sandbox detonation, catching zero-day malware that static signature scanning would miss. They cover completely different attack vectors — links vs files.

**10. Legitimate document with a macro held by Safe Attachments — how do you handle it?**
Review it in Quarantine, confirm it's a genuine false positive (verify sender and content legitimacy), release it to the recipient, and consider whether a targeted exception/allow entry is warranted for that specific trusted sender if this recurs — but avoid broadly disabling macro scanning, since macros are also a very common real attack vector.

**11. Recipient says never received; sender's side shows sent successfully — where do you look first?**
Message Trace for that specific message — it shows the actual path and where it stopped or was actioned (delivered, quarantined, junked, blocked) regardless of what the sender's outbox shows, since "sent" from the sender's side only confirms it left their outbox, not what happened after.

**12. BCL vs SCL — what's the difference?**
SCL (Spam Confidence Level) scores how likely a message is to be spam overall. BCL (Bulk Complaint Level) specifically scores bulk/mass-mail characteristics (newsletters, marketing) based on complaint/engagement patterns. A message can have a low SCL (not spammy) but a high BCL (clearly bulk marketing) — they're measuring different things and can be tuned independently.

**13. What is "mailbox intelligence," and what attack does it catch?**
It's a Defender for Office 365 anti-phishing feature that learns each user's normal communication patterns (who they usually talk to, in what style) and flags anomalies — like an email claiming to be from a regular contact but arriving from an unusual pattern. It's specifically aimed at catching Business Email Compromise (BEC)/impersonation attacks that contain no malware or malicious links at all, which basic anti-malware and Safe Links wouldn't catch.

**14. CFO's account compromised via phishing — first three actions, in order?**
1) Disable the account and revoke all active sign-in sessions immediately (a password reset alone doesn't invalidate an existing token). 2) Check the audit log specifically for any newly created inbox rule (common persistence/hiding technique). 3) Check for new OAuth app consents or delegated mailbox access changes that could allow continued access independent of the password.

**15. Why check for a new inbox rule first in a compromised mailbox investigation?**
Attackers commonly create a rule to auto-forward or auto-delete specific mail (e.g., anything mentioning "invoice" or "wire transfer") to hide their activity and enable follow-on fraud. It's the most commonly MISSED step — if not removed, the attacker keeps receiving copies of sensitive mail even after the password is reset and the "obvious" fix is done.

**16. Inbound vs outbound connector, and the risk of a misconfigured inbound one?**
Inbound connectors accept mail from a specific trusted source (often bypassing some default EOP checks since it's presumed pre-filtered/trusted) — e.g., an on-prem Exchange server in hybrid, or a third-party gateway. Outbound connectors force mail to specific domains through a specific path. Risk: an overly broadly-scoped inbound connector can accidentally bypass spam filtering for a wide swath of mail, a genuine security gap, not just a delivery quirk.

**17. Connector points to a decommissioned on-prem server (6 months gone) — likely symptom?**
Mail meant to route through that connector bounces or queues indefinitely, since the destination no longer exists — users would report mail to/from certain addresses failing or delayed specifically, while general mail flow not touching that connector works fine.

**18. Message Trace retention limit, and alternative for older investigations?**
Historically around 90 days depending on plan. For anything older, you need historical mail flow/Message Trace reports exported earlier, or Microsoft Purview's audit/compliance search tools if retained there — live Message Trace won't have the data past its retention window.

**19. Why is `Get-MessageTrace` alone insufficient for diagnosing a specific delay?**
It gives a high-level summary (delivered, pending, failed) but not per-hop timestamps. `Get-MessageTraceDetail` shows the granular, timestamped sequence of every processing step, which is what actually reveals WHERE time was lost — at a transport rule, at Safe Attachments detonation, or elsewhere — rather than just confirming the message eventually arrived.

**20. Forwarding rule to an external address the user says they didn't create — full response?**
Treat as a likely compromise, not just "remove the rule." Disable the account/revoke sessions immediately, remove the forwarding rule, check the audit log for how/when it was created and from what IP/device, check Sent Items and mail flow for any data that may have already been exfiltrated via that forward, reset credentials, force MFA re-registration, and determine root cause (phishing click, no MFA) to close the actual gap — not just the symptom.
