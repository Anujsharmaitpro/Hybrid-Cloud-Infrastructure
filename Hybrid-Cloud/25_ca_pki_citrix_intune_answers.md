# 25. ANSWERS — Conditional Access / PKI / Citrix / Intune (20 Questions)

---

# CONDITIONAL ACCESS

**1. Grant control vs Session control?**
Grant controls decide WHETHER access is allowed (require MFA, require compliant device, block). Session controls decide WHAT HAPPENS during the session once access is allowed (sign-in frequency forcing re-auth, limited browser access for unmanaged devices, Continuous Access Evaluation). Grant = gate. Session = ongoing constraints after the gate.

**2. Why exclude break-glass accounts from every CA policy?**
If a policy is misconfigured (e.g., "All users, All apps, Block"), it can lock out every admin simultaneously, including the people who'd need to fix it. A break-glass account excluded from all policies guarantees at least one path back in. Without it, recovery requires emergency Microsoft support intervention — much slower and not guaranteed.

**3. User in a Block-targeted group AND a Grant-targeted group for the same app — who wins?**
Block always wins. Conditional Access evaluates all applicable policies together, and any Block result overrides any Grant result, regardless of policy priority or order.

**4. What is Report-only mode, and what do you check before going live?**
Report-only evaluates the policy against real sign-ins and logs what WOULD have happened, without enforcing it. Before going live, check the Sign-in logs filtered to that policy to see exactly who would be blocked/challenged, confirm it matches intent, and make sure no legitimate user group is caught unexpectedly.

**5. "Any MFA method" vs Authentication Strength?**
"Any MFA method" accepts whatever the user has registered — push, SMS, phone call, FIDO2, all treated equally. Authentication Strength lets you require a SPECIFIC tier of method — e.g., only phishing-resistant methods like FIDO2/Windows Hello for Business — used for high-privilege or high-risk access where SMS/push isn't considered strong enough.

---

# PKI / CERTIFICATES

**6. Same CN, same key, renewed cert — can it still break trust?**
Yes — if the renewal was issued by a DIFFERENT Issuing/Intermediate CA than the original. The leaf cert's own CN and key don't matter if the chain now includes an intermediate the client doesn't yet trust/have in its store. This is the #1 real-world cause of "renewal broke trust for some users."

**7. CRL vs OCSP, and hard-fail vs soft-fail?**
CRL is a periodically-downloaded full list of revoked certs (works offline once cached, but can be stale). OCSP is a real-time, single-certificate check (faster, needs live connectivity). If the check is unreachable: hard-fail rejects the connection outright (more secure, less resilient); soft-fail lets the connection proceed with just a warning/no check performed (more resilient, weaker guarantee).

**8. Why keep Root CA offline, and how is it used when needed?**
The Root CA is the ultimate trust anchor — if compromised, every certificate under it becomes untrustworthy, requiring a full re-issuance of the entire hierarchy. Keeping it offline minimizes attack surface. It's brought online in a controlled, audited process specifically to sign a new Issuing CA's certificate, then taken offline again immediately after.

**9. What is OCSP Stapling and what problem does it solve?**
Instead of every client independently querying the OCSP responder on every connection, the SERVER pre-fetches its own OCSP response and "staples" it to the TLS handshake. This reduces load on the OCSP responder and removes the privacy exposure of the responder seeing every client checking a given certificate.

**10. Auto-enrollment works for domain-joined desktops but not remote/Entra-joined-only laptops — why, and fix?**
GPO-driven auto-enrollment requires the device to process Group Policy, which requires domain connectivity/line-of-sight — remote or cloud-only joined devices often never get that GPO processing cycle. Fix: deploy the certificate via an Intune Configuration Profile (Trusted Certificate / SCEP/PKCS profile) instead, which pushes over the internet via the MDM channel independent of on-prem network reachability.

---

# CITRIX

**11. What does Director's logon duration breakdown show?**
It splits total logon time into distinct phases: Brokering (DDC assigning a VDA), VM start/Boot (only for pooled/non-persistent desktops booting on demand), HDX connection (network path establishment), Profile load (FSLogix/UPM mounting), and GPO/logon scripts. Each phase is timed separately, letting you pinpoint exactly which layer is slow instead of guessing.

**12. One user can't log in while everyone else on the same infrastructure is fine — first suspicion?**
A corrupted or conflicted FSLogix profile container specific to that user — since the shared infrastructure is clearly healthy (everyone else works), the problem is isolated to that user's own profile data, not the environment.

**13. MCS vs PVS, and when to pick PVS?**
MCS uses hypervisor snapshots/differencing disks — simpler, storage-efficient, Citrix-native, generally preferred for moderate scale and cloud/Azure deployments. PVS streams a single shared vDisk over the network to many target devices — more complex to set up, but scales far more efficiently for very large (thousands of machines), non-persistent pools by offloading the I/O pattern differently than per-VM differencing disks would.

**14. External users blocked via Gateway, internal LAN users fine — where do you look first?**
The Citrix Gateway itself, most likely its certificate — since internal traffic doesn't route through Gateway at all, an "external only" symptom points directly at that specific component rather than general Citrix health (DDC, VDA, StoreFront are shared by both paths and would affect internal users too if they were the cause).

**15. Machine Catalog vs Delivery Group?**
A Machine Catalog defines the actual VM template/provisioning method (MCS or PVS) used to create a pool of machines — it's about HOW the machines are built. A Delivery Group is a logical grouping of VDAs (drawn from one or more catalogs) assigned to serve specific users with specific published resources — it's about WHO gets access to WHAT. A catalog can feed multiple delivery groups, or vice versa depending on design.

---

# INTUNE / AUTOPILOT

**16. Most common reason a Win32 app shows "Failed" despite a successful install?**
A misconfigured detection rule — Intune checks for a specific registry key, file version, or MSI product code to confirm the app installed, and if that rule doesn't actually match what the installer created, Intune reports Failed even though the app is present and working fine.

**17. 200 laptops stuck at ESP — first thing you check?**
Intune's "Device install status" for one of the affected devices — this immediately shows whether a specific required app or policy is the actual blocker, rather than guessing. It's the fastest path to distinguishing a network-wide issue (all devices affected) from an app-specific issue (subset affected).

**18. User-driven vs Self-deploying vs White Glove Autopilot modes?**
User-driven: standard — the end user signs in with their own identity during OOBE. Self-deploying: no user interaction/identity at all, used for kiosks/shared/meeting-room devices. White Glove (Pre-provisioned): IT/reseller pre-stages apps/policies BEFORE shipping, so the end user's actual first-boot experience is much faster since the heavy lifting is already done.

**19. Device shows "Not Compliant" with no reported user changes — common causes?**
BitLocker key failed to escrow to Entra ID (encryption may technically be on, but Intune can't confirm/recover the key, so it marks non-compliant); a failed Windows Update silently dropped the OS below the minimum required version; or Defender definitions became stale without the user noticing.

**20. Why is permanent non-blocking ESP a security risk, not just a fix?**
It lets users reach the desktop and start working before required security policies (encryption, Defender, required apps) are confirmed applied. That means Conditional Access "require compliant device" checks may not have current data on that device yet, creating a window where a device is in active use without confirmed compliance — acceptable as a short-term mitigation during an incident, but a real gap if left permanent.
