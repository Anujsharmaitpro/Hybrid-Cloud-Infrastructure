# 16. MICROSOFT INTUNE / ENDPOINT MANAGER & WINDOWS AUTOPILOT — Point-by-Point Breakdown
### Format for every item: **What is it? → What is it used for? → What issues can it cause (if broken/misconfigured)?**

---

## * MDM Enrollment (Mobile Device Management)

**What is it?**
Simple: The process by which a device officially "joins" Intune's management, so policies/apps can be pushed to it.
Technical: The device establishes a management channel with Intune via OMA-DM (Open Mobile Alliance Device Management) protocol after registering with Entra ID (Azure AD Join, Hybrid Azure AD Join, or Azure AD Registered for BYOD).

**What is it used for?**
The prerequisite for ANY Intune management — without enrollment, no policy, compliance check, or app can be pushed. It's the "handshake" step.

**What issues can it cause?**
- **MDM authority not set correctly** — if Entra ID's MDM authority isn't pointed at Intune (can happen in orgs migrating from a legacy MDM), enrollment silently fails or routes to the wrong management system.
- **Licensing not assigned before enrollment attempt** — user has no Intune license → enrollment fails with a licensing error that's often misread as a technical fault.
- **Enrollment restrictions/limits** — Intune has configurable per-user device limits and platform restrictions; a user hitting their device limit or enrolling an unsupported OS version gets a silent or unclear block.

---

## * Compliance Policies

**What is it?**
Rules Intune continuously checks against enrolled devices (BitLocker/encryption enabled, minimum OS version, jailbreak/root detection status, password/PIN complexity, Defender status) — devices are marked **Compliant** or **Not Compliant** based on the results.

**What is it used for?**
Feeds directly into Conditional Access as a grant control ("Require compliant device") — this is the mechanism that actually enforces device health as an access gate, not just a reporting dashboard.

**What issues can it cause?**
- **Device shows Not Compliant unexpectedly** — most common causes: BitLocker key failed to escrow to Entra ID (encryption technically on, but Intune can't confirm/recover the key so marks non-compliant), a failed Windows Update pushed OS below the minimum required version, or Defender definitions became stale.
- **Compliance evaluation delay** — a device that just became compliant (e.g., user just enabled encryption) doesn't reflect instantly — there's a check-in interval (default ~8 hours, can be forced via `Sync` in Company Portal) — causes "I just fixed it, why am I still blocked" confusion.
- **Conflicting compliance policies** from overlapping group assignments — Intune generally applies the strictest combination, which can produce an unexpected non-compliant result if two policies with different thresholds both apply.

---

## * Configuration Profiles

**What is it?**
Profiles that actively PUSH a specific setting to a device — Wi-Fi/VPN configuration, registry values, device restrictions (camera disabled, USB restrictions), certificate deployment, Windows Update rings.

**What is it used for?**
The Intune-native replacement/complement for GPO — configuring devices that may never be domain-joined or on the corporate network, entirely over the internet via MDM channel.

**What issues can it cause?**
- **Profile conflict** — two profiles pushing contradictory values to the same setting on the same device causes a "conflict" error status in Intune, and neither setting reliably applies until resolved.
- **Assignment scope error** — profile assigned to the wrong group (or an empty/misconfigured dynamic group) — silently never reaches intended devices, with no obvious error unless someone checks "Device status" for that profile.
- **CSP (Configuration Service Provider) not supported on that specific OS build/edition** — e.g., a setting only supported on Windows Enterprise, applied to a Pro-edition device, fails silently for that device.

---

## * App Deployment (Win32, MSIX, Store, Web Link)

**What is it?**
The mechanism for pushing software to managed devices. **Win32** (.intunewin packaged, most flexible, uses the Intune Management Extension agent), **MSIX** (modern, sandboxed packaging), **Store apps** (Microsoft Store for Business/new Store integration), **Web links** (just a shortcut, no real install).

**What is it used for?**
Ensures every managed device has required business software installed/kept updated without manual per-device installation — critical for the Autopilot "required apps" experience specifically.

**What issues can it cause?**
- **Detection rule misconfiguration** — the single most common Win32 app deployment failure: the install technically succeeds, but the detection rule (checking for a specific registry key, file version, or MSI product code) doesn't match what the installer actually created — Intune reports "Failed" even though the app is present and working.
- **Wrong architecture targeting** — deploying an x86 package expecting it on x64-only devices (or vice versa) causes install failures for a subset of the fleet depending on hardware/OS mix.
- **Dependency ordering** — an app requiring a prerequisite (e.g., a specific .NET runtime) that isn't deployed first, or isn't marked as a dependency in Intune, fails to install.
- **Intune Management Extension not installed/functioning** on the device — Win32 apps specifically require this agent; if it's not present or corrupted, ALL Win32 app deployments to that device fail while other management functions (profiles, compliance) may work fine.

---

## * Windows Autopilot — Hardware Hash & Registration

**What is it?**
A unique hardware identifier (hash) collected from a device (via OEM factory registration, or manually via `Get-WindowsAutoPilotInfo` PowerShell script, or "Convert to Autopilot" within Intune for already-deployed devices) and uploaded to the Autopilot service, tying that specific physical device to your tenant.

**What is it used for?**
Lets a brand-new, unopened device recognize at first boot (OOBE) that it belongs to your organization — the entire "zero-touch" deployment experience depends on this registration existing BEFORE the device reaches OOBE.

**What issues can it cause?**
- **Device shipped before hash upload completed** — race condition in rushed bulk deployments; device reaches OOBE with no matching Autopilot registration, so it falls through to a generic/manual setup experience instead of the intended zero-touch flow.
- **Hash registered to the wrong tenant** — common in multi-tenant reseller/managed-service scenarios; device silently enrolls into the wrong company's management.
- **Duplicate/stale hash entries** — a device re-imaged or repurposed without removing its old Autopilot registration can cause profile assignment confusion.

---

## * Autopilot Deployment Profile

**What is it?**
The configuration applied during OOBE: join type (Azure AD Joined vs Hybrid Azure AD Joined), whether to skip privacy/EULA/OEM registration screens, and deployment mode.

**What is it used for?**
Controls exactly what the end user sees and experiences during first boot — streamlining it to feel like a guided, branded corporate setup rather than a generic Windows OOBE.

**What issues can it cause?**
- **Wrong join type selected for the environment** — e.g., Hybrid Azure AD Join selected but the device has no line-of-sight to an on-prem DC during deployment (common for fully remote employees) — device gets stuck because hybrid join requires domain connectivity to complete.
- **Profile not assigned before shipping** — device reaches OOBE with a registered hash but no profile assigned yet — falls back to default/manual OOBE instead of the intended automated flow.

---

## * Autopilot Deployment Modes: User-driven / Self-deploying / Pre-provisioned (White Glove)

**What is it?**
- **User-driven:** Standard mode — end user unboxes device, signs in with their own credentials, device configures around their identity.
- **Self-deploying:** No user interaction/sign-in at all — used for kiosks, shared devices, meeting-room devices — device joins and enrolls with no specific user context.
- **Pre-provisioned (White Glove):** IT/reseller pre-stages the device (installs apps/policies) BEFORE shipping to the end user, so the end user's own OOBE experience is much faster (most heavy lifting already done).

**What is it used for?**
Matching the deployment approach to the actual use case and desired end-user experience/speed — a large-scale corporate rollout with heavy required-app payloads often uses White Glove specifically to avoid a 45+ minute first-boot wait for the end user.

**What issues can it cause?**
- **Wrong mode chosen for the use case** — e.g., using User-driven for shared kiosk devices creates unnecessary per-user identity binding that complicates the intended shared-device model.
- **White Glove requires a technician step** that's easy to skip/forget in a rushed rollout — if skipped, the device just behaves as standard User-driven anyway (not a hard failure, but loses the intended time-saving benefit).

---

## * Enrollment Status Page (ESP)

**What is it?**
A full-screen, configurable progress page shown during Autopilot enrollment that can BLOCK the user from reaching the desktop until specified "required" apps/policies finish installing (or allow through immediately while installs continue in the background).

**What is it used for?**
Guarantees a fully-configured, compliant device before first use — important for security-conscious orgs that don't want users active on a device before critical policies (encryption, Defender, required apps) have actually applied.

**What issues can it cause?**
- **Stuck at a specific phase (e.g., "Account setup")** because a REQUIRED app or policy assigned to the ESP profile is failing — the device waits indefinitely (until the configured timeout) for something that will never succeed on its own.
- **Timeout too short for the app payload size** — large or numerous required apps combined with a conservative timeout setting causes otherwise-successful-but-slow installs to be treated as failures.
- **Network/proxy blocking Microsoft Graph/Intune/Windows Update endpoints** — a very common cause when ESP fails for an ENTIRE new batch of devices at once, especially on a newly deployed corporate network with web filtering/SSL inspection that hasn't allow-listed the required Microsoft URLs.
- **Set to non-blocking permanently (as a "fix" rather than temporary mitigation)** — users reach the desktop before required security policies are confirmed applied, creating a real security gap since Conditional Access "require compliant device" checks may not yet have current data on that device.

---

## END-TO-END FLOW: Autopilot to fully-managed desktop

```
Hardware hash registered to tenant (before shipping)
   ↓
Device shipped, powered on by end user → OOBE detects Autopilot registration
   ↓
Autopilot Profile applied (join type, skip screens, deployment mode)
   ↓
Azure AD Join (or Hybrid AAD Join) completes
   ↓
MDM enrollment triggers automatically (Intune is the configured MDM authority)
   ↓
Enrollment Status Page (ESP) shows progress:
   - Device-phase (required device-targeted apps/profiles)
   - User-phase (required user-targeted apps/profiles)
   ↓
Compliance Policy evaluates device health (encryption, OS version, Defender)
   ↓
Conditional Access checks compliance status on subsequent sign-ins
   ↓
Desktop reached — fully configured, policy-compliant device
```

## BREAK/FIX MASTER TABLE

| Component | Purpose | If it breaks/misconfigured | Symptom | First Check | Fix |
|---|---|---|---|---|---|
| Hardware hash | Ties device to tenant | Not uploaded before shipping | Device falls to generic OOBE | Autopilot devices list in Intune | Re-register, or run `Get-WindowsAutoPilotInfo` on-site |
| Deployment Profile | Controls OOBE experience | Wrong join type for environment | Hybrid join hangs (no DC line of sight) | Check profile join type vs network reality | Switch to Azure AD Join if no hybrid connectivity |
| ESP | Blocks until ready | Required app failing | Stuck at specific phase | Intune > Device install status | Fix app detection rule, or temporarily de-scope from required |
| Win32 App | Software deployment | Detection rule mismatch | Reports Failed despite successful install | Check detection rule vs actual install artifact | Correct detection rule (registry/file/MSI code) |
| Compliance Policy | Health gate for CA | BitLocker key not escrowed | Device Not Compliant unexpectedly | Device compliance blade > specific failed check | Fix escrow/encryption, force compliance re-check |
| MDM Enrollment | Prerequisite handshake | Wrong MDM authority set | Enrollment fails/misroutes | Entra ID > Mobility (MDM/MAM) settings | Correct MDM authority to Intune |

## TOP INTERVIEW QUESTIONS

🔴 Walk through the full Autopilot flow from hardware hash to a usable desktop.
🔴 200 devices stuck at ESP "Account setup" — full triage in priority order.
🔴 What's the difference between a Compliance Policy and a Configuration Profile?
🟠 What's the most common real-world cause of a Win32 app deployment "failure" that isn't actually a failed install?
🟠 Difference between User-driven, Self-deploying, and White Glove Autopilot modes — when would you use each?
🟡 What happens if ESP is left in non-blocking mode permanently rather than as a temporary fix?
🟡 Why might Hybrid Azure AD Join specifically fail for a remote employee?
🔴 Scenario: a device shows Not Compliant unexpectedly with no user-reported changes — diagnose.
🟠 Scenario: an entire new office's devices fail Autopilot enrollment on day one — what's your first suspicion and why?

## RED FLAGS / TRICK QUESTIONS
- "Does Autopilot require a custom image?" — No, that's the core point of Autopilot — it configures a stock OEM image; White Glove pre-stages apps but doesn't require custom imaging either.
- "Is ESP mandatory for Autopilot to work?" — No, it's optional/configurable per profile; some orgs skip it and let apps install silently in the background after desktop access.
- "Does marking a device Compliant happen instantly after fixing the issue?" — No — there's a check-in interval; users can force it via Company Portal "Sync," but it's not automatic and immediate.

## L3 ANSWER — ESP Stuck (60-120s)
"I don't start guessing when ESP is stuck — I go straight to the Intune portal's device install status for one of the affected devices, because that tells me immediately whether a specific required app or policy is the blocker, versus something more systemic. If it's all 200 devices simultaneously, my first suspicion is network — a new corporate deployment often has a proxy or SSL inspection that hasn't allow-listed the Microsoft Graph and Intune service endpoints, and that silently breaks enrollment for everyone at once. If it's a subset, I'm almost always looking at a Win32 app's detection rule — the install itself frequently succeeds, but the detection rule checks for the wrong registry key or file path, so Intune reports it as failed and ESP just waits forever for something that will never resolve on its own. Given the business pressure of a blocked onboarding wave, I'd set that ESP profile to non-blocking or drop the failing app from 'required' so people can get to a working desktop today, fix the actual detection rule or network rule offline, and validate against a small pilot batch before trusting it for the rest of the fleet."

## MEMORY TRICK
**ESP stuck = check the SPECIFIC blocker (Intune device install status) before checking WHY (logs).**
**All devices at once = network/proxy. Subset of devices = app detection rule.**
**Compliant ≠ instant — there's always a check-in interval.**

## 10-SECOND REVISION
- Hardware hash must be registered BEFORE the device reaches OOBE
- ESP stuck → Intune "Device install status" first, tells you exactly which app/policy
- All devices failing = suspect network/proxy blocking Graph/Intune endpoints
- Subset failing = suspect one Win32 app's detection rule
- Compliance Policy feeds Conditional Access — it's a gate, not just a report
- Non-blocking ESP should be temporary mitigation, not a permanent fix (security gap if left on)
