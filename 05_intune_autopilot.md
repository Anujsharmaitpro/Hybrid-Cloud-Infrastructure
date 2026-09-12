# 5. MICROSOFT INTUNE / ENDPOINT MANAGER + WINDOWS AUTOPILOT (PRIORITY — unanswered in your mock interview)

## 5.1 Intune / Endpoint Manager — Fundamentals

**What is it?**
Simple: Cloud-based device management — pushes policies, apps, and compliance rules to laptops/phones without needing them on the corporate network or joined to on-prem AD.
Technical: An MDM (Mobile Device Management) / MAM (Mobile Application Management) platform that communicates with enrolled devices via the Windows MDM protocol (OMA-DM/CSPs) or, for apps, via the Intune Management Extension agent for Win32 apps.

**Why used:** Enables management of remote/hybrid workforce devices that never touch the corporate network — the modern replacement/complement for GPO, which requires domain connectivity.

**How it works — enrollment flow:**
Device → Settings > Access work/school account (or Autopilot-driven) → Azure AD Join / Hybrid Azure AD Join → MDM enrollment triggered automatically (via Entra ID's configured MDM authority = Intune) → device registers with Intune → policies/apps/compliance rules assigned based on Entra ID group membership begin deploying.

**Components:** MDM/CSPs (Configuration Service Providers — the actual Windows-side settings interface), Intune Management Extension (IME — agent for Win32/PowerShell script deployment), Compliance Policies, Configuration Profiles, App Deployment (Win32, MSIX, Store apps), Enrollment Status Page (ESP).

**Dependencies:** Entra ID (identity + MDM authority setting) → Network/internet (device needs outbound access to Microsoft Graph/Intune service endpoints — proxy/firewall/SSL-inspection issues here are a VERY common real-world break point) → Licensing (Intune license assigned to the user before enrollment).

## 5.2 Windows Autopilot

**What is it?**
Simple: Lets a brand-new, unboxed laptop configure itself for the company automatically the first time it's turned on — no imaging required.
Technical: A cloud service that ties a device's hardware hash to a tenant, so at OOBE (Out of Box Experience) the device recognizes it belongs to your org, applies an Autopilot Profile, and drives the device through Azure AD Join + Intune enrollment automatically.

**How it works — flow:**
Device hardware hash uploaded to Autopilot (via OEM registration, or `Get-WindowsAutoPilotInfo` script, or Intune "Convert to Autopilot" for existing devices) → device shipped to user, powered on → OOBE detects it's Autopilot-registered → Autopilot Profile applied (join type, skip privacy/EULA screens, deployment mode: User-driven vs Self-deploying/Pre-provisioned) → Azure AD Join → MDM enrollment → **Enrollment Status Page (ESP)** shows progress while required apps/policies install → device usable.

**Components:** Hardware hash (unique device identifier), Autopilot Profile, Deployment Profile assignment (via dynamic group typically), ESP (Enrollment Status Page — configurable to block usage until critical apps/policies finish, or allow through).

## 5.3 Enrollment Status Page (ESP) — Deep Dive (directly answers your unanswered Q6)

**What it is:** A full-screen progress page during Autopilot enrollment that can be configured to **block** the user from reaching the desktop until specified "required" apps and policies finish installing — ensures a fully-configured device before first use.

**Failure scenario — 200 devices stuck at "Account setup":**

**Triage approach (in priority order):**
1. **Determine scope:** All 200, or a subset? If all — suspect a tenant-wide config/licensing/service issue. If a subset — suspect a hardware batch, network segment, or specific app/policy assignment issue.
2. **Pull logs from an affected device** — this is the single most important step, and the one I should have led with in the original answer:
   - `C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\IntuneManagementExtension.log` — shows Win32 app install attempts/failures.
   - `MDMDiagReport.html` via running `mdmdiagnosticstool.exe -area Autopilot;DeviceEnrollment;DeviceProvisioning -cab C:\diag.cab` — comprehensive enrollment diagnostic.
   - Event Viewer → Applications and Services Logs → Microsoft → Windows → **DeviceManagement-Enterprise-Diagnostics-Provider** — shows exact enrollment/ESP failure codes.
3. **Check Intune portal:** Devices > [device name] > Device install status / User install status — shows PER-APP and PER-POLICY status (Installed/Failed/In Progress) — this immediately tells you WHICH required app or policy is hanging.
4. **Most common real-world root causes, in likelihood order:**
   - **A "required" (blocking) app is failing to install** — could be a detection rule misconfiguration, wrong architecture (x86 vs x64), or a dependency app not yet installed.
   - **Network/proxy/SSL-inspection blocking Microsoft Graph/Intune/Windows Update endpoints** — extremely common on new corporate networks with web filtering that hasn't allow-listed Microsoft's required URLs (`*.manage.microsoft.com`, `*.microsoftonline.com`, Windows Update endpoints for driver/app content).
   - **ESP timeout too short** for the number/size of required apps assigned — default technician/user ESP timeout may not be enough for a heavy app payload.
   - **Licensing not yet propagated** — if user accounts were bulk-created/licensed right before deployment, Intune license assignment can take time to fully replicate before enrollment can complete cleanly.
   - **Device not fully registered in Autopilot before OOBE was reached** — hash upload race condition in a rushed bulk deployment.
5. **Immediate mitigation to unblock the onboarding wave:** Temporarily set the ESP profile to **non-blocking** (or remove the specific failing app from "required") so affected users can reach the desktop while you fix root cause separately — critical practical answer showing you understand business impact vs perfect fix trade-off.
6. **Long-term fix:** Correct the failing app's detection rule/architecture, fix firewall/proxy allow-list, adjust ESP timeout, re-validate with a small pilot batch (5-10 devices) before re-running the remaining fleet.

**Validate:** Re-run a test device through full Autopilot OOBE from scratch, confirm ESP completes and required apps show Installed in Intune portal.

**Prevent recurrence:** Always pilot a new ESP/app configuration on a small batch before a mass deployment; document required firewall/URL allow-list for Autopilot/Intune traffic; monitor Intune app deployment success rate as an ongoing metric.

## 5.4 Compliance Policies

**What:** Rules Intune checks continuously (encryption enabled, OS version minimum, jailbreak/root detection, password complexity) — feeds directly into Conditional Access "Require compliant device" grant control.

**Failure scenario:** Device shows "Not compliant" unexpectedly → check Device compliance status in Intune portal for which specific check failed → common cause: BitLocker key not escrowed properly, or OS fell behind minimum version after a failed update.

## 5.5 Configuration Profiles vs Compliance Policies (common interview confusion point)

- **Configuration Profiles** = actively PUSH a setting (e.g., set a specific registry value, Wi-Fi profile, VPN config) — device is CONFIGURED.
- **Compliance Policies** = CHECK a condition and report status (device IS or ISN'T compliant) — doesn't push settings, just evaluates and reports, then Conditional Access acts on that status.

## 5.6 App Deployment Types

Win32 (.intunewin package, with detection rules + install/uninstall commands), MSIX (modern packaged apps), Store apps, Web links. **Win32 app failures are the most common ESP blocker** — usually a detection rule that doesn't correctly identify successful install (e.g., checking for a registry key that the installer doesn't actually create).

## COMMANDS / TOOLS
```
Get-WindowsAutoPilotInfo -OutputFile hash.csv     → generate hardware hash for a device
mdmdiagnosticstool.exe -area Autopilot;DeviceEnrollment;DeviceProvisioning -cab diag.cab
dsregcmd /status                                   → check Azure AD Join / device registration state
Get-MgDeviceManagementManagedDevice (Graph)        → query device status via PowerShell
```
Log paths: `C:\ProgramData\Microsoft\IntuneManagementExtension\Logs\`, Event Viewer > DeviceManagement-Enterprise-Diagnostics-Provider.

## INTERVIEW QUESTIONS

🔴 Walk through the full Autopilot deployment flow from hardware hash to usable desktop.
🔴 200 devices stuck at ESP "Account setup" — full triage, in order.
🔴 Difference between Configuration Profiles and Compliance Policies.
🟠 What logs/tools would you use to diagnose a failed Win32 app deployment via Intune?
🟠 What's the difference between User-driven, Self-deploying, and Pre-provisioned Autopilot modes?
🟡 How does ESP block/non-block behavior affect a mass onboarding deployment?
🟡 What's the risk of setting ESP to non-blocking permanently vs temporarily?
🔴 Scenario: a device shows "Not compliant" in Intune but user says nothing changed — diagnose.
🟠 Scenario: new corporate network blocks Autopilot completion for all new hardware — what do you check?

## RED FLAGS / TRICK QUESTIONS
- "Does Autopilot require imaging?" — No, that's the entire point — it configures a stock OEM image, no custom image required (though "pre-provisioned/white glove" mode does pre-stage apps before shipping for faster end-user experience).
- "Is ESP mandatory?" — No, it's optional/configurable — some orgs skip it and let users into the desktop immediately while apps install in the background.

## IDEAL ANSWER (30-60s)
"Autopilot lets a brand-new device configure itself for the org the first time it's powered on — the hardware hash is pre-registered, so at OOBE the device knows it belongs to the tenant, joins Entra ID, enrolls into Intune, and the Enrollment Status Page shows progress while required apps and policies install before the user reaches the desktop."

## L3 ANSWER (60-120s)
"When a batch of devices gets stuck at ESP, I don't start guessing — I pull the actual device install status in the Intune portal first, because that tells me immediately whether it's a specific required app failing versus a broader connectivity issue. If it's all 200 devices at once, my first suspicion is network — new corporate deployments often have SSL inspection or a proxy that hasn't allow-listed the Microsoft Graph and Intune service endpoints, which silently breaks enrollment. If it's a subset, I'm looking at a Win32 app's detection rule — a very common failure is the install technically succeeding but the detection rule checking for the wrong registry key or file path, so Intune reports it as failed and ESP hangs waiting on it forever. For the business-impact side, since this was blocking a major onboarding wave, I'd set the ESP profile to non-blocking or temporarily drop the failing app from 'required' so people can get to a working desktop today, fix the actual root cause offline, then validate against a small pilot batch before I'd trust it for the rest of the fleet."

## MEMORY TRICK
**ESP stuck = check WHO's stuck (app or policy) via Intune device status, THEN check WHY (logs) — don't guess.**

## 10-SECOND REVISION
- Autopilot = hardware hash + profile → Azure AD Join → Intune enrollment → ESP
- ESP hangs = check Intune "Device install status" first — tells you exactly which app/policy
- All devices affected = suspect network/proxy blocking Graph/Intune endpoints
- Subset affected = suspect a specific Win32 app's detection rule
- Mitigate business impact fast (non-blocking ESP), fix root cause separately
