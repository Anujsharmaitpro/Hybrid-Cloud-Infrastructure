# 24. DAY-TO-DAY OPERATIONAL SCENARIOS — BATCH 3 (Additional Scenarios)
### Format: Scenario → Short Answer (interview-ready, no duplicates from files 21 or 23)

---

# ACTIVE DIRECTORY / DNS / DHCP / GPO

**Q: A merger requires renaming the AD domain itself — what's your approach?**
A: Domain rename (`rendom`) is a complex, high-risk, rarely-recommended operation affecting every DC, GPO, and trust; in most real-world cases a new domain/forest with a migration tool (ADMT) and trust relationship is safer than an in-place rename.

**Q: You suspect an attacker has compromised a service account with excessive AD permissions.**
A: Immediately reset its password and rotate any dependent application configs, audit its actual group memberships/delegated permissions (often over-privileged from years of accumulated "just in case" grants), and check Security log for recent usage from unexpected sources.

**Q: A GPO needs to apply differently based on whether a laptop is on VPN vs the corporate LAN.**
A: GPO itself has limited native network-location awareness beyond slow-link detection; better handled by scoping via a security group updated by a login script/Intune, or migrating that specific requirement to a Conditional Access + Intune Configuration Profile approach instead.

**Q: DNS zone transfers are failing between a primary and secondary (non-AD-integrated) DNS server.**
A: Check zone transfer settings on the primary (Name Servers tab, or explicit IP allow-list under Zone Transfers) — a common cause is the secondary's IP not being explicitly permitted to pull transfers.

**Q: An audit finds several AD user accounts with "password never expires" set inappropriately.**
A: Query with `Get-ADUser -Filter {PasswordNeverExpires -eq $true}`, review each for legitimate justification (some service accounts genuinely need it), remediate the rest, and recommend Group Managed Service Accounts (gMSA) going forward instead of manually-set non-expiring passwords for new service accounts.

---

# HYBRID IDENTITY / ENTRA ID

**Q: A department wants self-service group management so they don't need to file tickets for every access change.**
A: Enable self-service group management in Entra ID (with an approval workflow if needed) — lets designated owners add/remove members from specific groups without needing an admin ticket, while still being audited.

**Q: You need to prevent users from consenting to risky third-party OAuth app permissions themselves.**
A: Restrict user consent settings in Entra ID (Enterprise applications > Consent and permissions) to require admin approval for anything beyond low-risk permissions — prevents a phishing-style "consent grant" attack from succeeding via end-user approval.

**Q: An acquisition means you need guest access for 200 external users quickly, but with tight expiration.**
A: Use Entra ID Entitlement Management (Access Packages) with a defined access review/expiration policy rather than manually inviting 200 individual guests — automates both the onboarding and the eventual offboarding/expiry.

**Q: Security asks whether your Conditional Access setup would survive an Entra Connect outage.**
A: Conditional Access itself is a cloud-side Entra ID function and doesn't depend on Entra Connect being currently online — existing synced identities and PHS-cached credentials continue to allow cloud sign-in and CA evaluation even if Connect itself is down; only NEW changes from on-prem wouldn't sync until Connect recovers.

---

# PKI / CERTIFICATES

**Q: A cloud migration means some internal certificates need to move from an on-prem CA to a cloud-based CA-as-a-service.**
A: Plan a coexistence period where both CAs are trusted (distribute the new CA's root/intermediate to all clients before cutover), migrate certificate issuance workload gradually by service, and only decommission the old CA once 100% of dependent certs are confirmed reissued from the new source.

**Q: A user's smart card login suddenly stops working after a certificate renewal.**
A: Check if the renewed certificate still has the correct UPN mapping in the Subject Alternative Name — smart card logon specifically requires this mapping, and a renewal template misconfiguration can drop it silently.

**Q: An external partner needs to trust your internally-issued certificates for a B2B integration.**
A: Export your Root (and any relevant Intermediate) CA public certificate and provide it to the partner to import into their trust store — never share the private key; this is a common real-world B2B PKI trust-establishment scenario.

---

# INTUNE / AUTOPILOT

**Q: A shared kiosk device in a warehouse needs to auto-login without any specific user identity.**
A: Use Autopilot Self-Deploying mode combined with a Shared Device / Kiosk Configuration Profile — no user credential required, ties to the device identity rather than a person.

**Q: You need to prevent users from installing unauthorized software on managed Windows devices.**
A: Combine an AppLocker or Windows Defender Application Control (WDAC) Configuration Profile with an Intune app deployment allowlist — blocks execution of anything not explicitly approved, rather than relying solely on standard user (non-admin) restrictions.

**Q: A remote employee's device compliance keeps flipping between Compliant and Not Compliant repeatedly.**
A: Check for a borderline setting causing repeated toggling (e.g., Defender definitions barely staying current, or intermittent connectivity causing missed check-ins) — investigate the specific failing check's history rather than treating it as a one-time issue.

---

# CITRIX

**Q: You need to give temporary published-app access to a contractor for exactly 2 weeks.**
A: Add them to a time-bound dynamic/nested group tied to the relevant Delivery Group entitlement, or use Entra ID Entitlement Management with an expiration policy feeding into the Citrix-entitled group — avoids manual cleanup being forgotten.

**Q: A Citrix environment needs to support a sudden 3x increase in concurrent users for a seasonal peak.**
A: Check Machine Catalog capacity and scale via MCS/PVS to add more VDAs ahead of the peak (not during it), verify License Server has sufficient concurrent license count, and consider Azure-based autoscale-capable host pools if the catalog is cloud-hosted.

**Q: Users complain of Citrix session freezes specifically during a specific time window each day.**
A: Correlate the time window against scheduled tasks (backup jobs, antivirus full scans, patch windows) running on the underlying VDA hosts or storage array — a resource contention pattern tied to a schedule, not a random Citrix fault.

---

# MICROSOFT 365 / EXCHANGE / MAIL SECURITY

**Q: An employee is leaving the company — what needs to happen to their mailbox?**
A: Convert to a shared mailbox (retains content without consuming a license) or set a forwarding/delegate rule per legal/business retention policy, place on litigation hold if relevant to any ongoing matter, then disable the account (not delete immediately, to preserve recoverability window).

**Q: A department wants to send mass email campaigns without triggering spam filters or looking phishy.**
A: Recommend a dedicated subdomain for bulk sending with its own SPF/DKIM/DMARC configured specifically for that use case, separate from the main corporate domain's mail reputation — protects the primary domain's deliverability if the bulk sender's reputation degrades.

**Q: Teams external access needs to be allowed for one specific partner domain only, blocked for everyone else.**
A: Configure Teams Admin Center > External access with that specific domain explicitly allowed and the default set to blocked for all other domains — default-allow-all is a common oversight that should be tightened.

---

# AZURE

**Q: A production database VM needs zero-downtime patching.**
A: Use an Availability Set/Zone with multiple VM instances behind a load balancer so patches can be rolled through one instance at a time (similar concept to Citrix drain mode), or use a PaaS database service (Azure SQL) where Microsoft handles patching transparently.

**Q: You need to prove compliance posture for an audit across dozens of subscriptions.**
A: Use Azure Policy's compliance dashboard at the Management Group level combined with Microsoft Defender for Cloud's Secure Score — gives an audit-ready, centralized compliance view rather than manually checking each subscription.

**Q: A subscription's cost suddenly spikes 400% overnight with no known change.**
A: Check Cost Management's cost analysis broken down by resource — common causes are a forgotten test resource left running at scale, a runaway autoscale event, or (less commonly but seriously) a compromised account provisioning cryptomining VMs; the latter warrants an immediate security review, not just a cost review.

**Q: An application team wants direct database access from their on-prem network to an Azure SQL instance without going over the public internet.**
A: Configure a Private Endpoint for the Azure SQL instance plus ExpressRoute or Site-to-Site VPN connectivity from on-prem — keeps the traffic entirely on private connectivity.

---

# VMWARE (relevant given your resume background — cross-hybrid scenarios)

**Q: A vSphere HA event fails over VMs, but one specific VM doesn't restart on the new host.**
A: Check that VM's HA restart priority setting and whether it was explicitly excluded from HA protection; also check if the destination host had insufficient resources reserved (admission control) to accommodate it.

**Q: You need to migrate a large VMware VM to Azure with minimal downtime.**
A: Use Azure Migrate (agentless or agent-based, depending on the workload) for continuous block-level replication ahead of cutover, similar in principle to a River Meadow-style approach — schedule the final cutover during a low-usage window to minimize the actual downtime window to just the final delta sync.

**Q: DRS keeps migrating a specific VM between hosts repeatedly (vMotion storm).**
A: Check for a resource contention pattern or an overly aggressive DRS automation/migration threshold setting; may need a DRS affinity/anti-affinity rule or to address the underlying resource imbalance rather than letting DRS keep "fighting" it.

---

# INCIDENT / CHANGE / GOVERNANCE

**Q: Two different teams both claim ownership (or disclaim ownership) of a failing shared service.**
A: This is a CMDB/service-mapping gap — the incident response should proceed with whichever team can act fastest to restore service, while a follow-up action item formally clarifies ownership in the CMDB to prevent the same confusion in future incidents.

**Q: An Emergency Change is requested outside the normal CAB approval window.**
A: Emergency Change process should still require at minimum a verbal/expedited approval from an authorized approver (not skip approval entirely) — document the change and get full retrospective approval/review as soon as the CAB can convene, since "emergency" doesn't mean "unapproved."

**Q: A vendor's SLA was missed during a major incident — how do you handle it in the post-incident process?**
A: Include vendor performance as a specific factor in the RCA if it contributed to extended downtime, and route the SLA breach through the formal vendor management/contract review process separately from the technical RCA — these are related but distinct follow-up tracks.

---

## HOW TO USE THIS FILE
Combined with files 21 and 23, this gives roughly 110+ scenario/answer pairs spanning the full JD scope plus adjacent real-world situations (VMware, vendor management, M&A identity integration) that a panel may probe given your actual background. Prioritize re-reading the VMware and PKI sections here specifically, since they connect directly to your resume's strongest area (VMware/Azure dual-cloud) and weakest area (PKI) respectively.
