# 10. POWERSHELL AUTOMATION (unanswered in your mock interview — build a real example now)

## 10.1 What "good" PowerShell automation looks like at L3 (the evaluation criteria interviewers actually use)

It's not about knowing exotic cmdlets — interviewers assess:
1. **Error handling** (try/catch, not letting a script fail silently)
2. **Least privilege** (service account with only the permissions needed, not Domain Admin for convenience)
3. **Idempotency/safety** (a script that can run repeatedly without causing damage — e.g., an allowlist/exclusion list before bulk changes)
4. **Logging/audit trail** (so failures and actions are traceable after the fact)
5. **A measurable outcome** (time saved, incidents reduced, audit findings resolved)

## 10.2 Ready-to-use Real Example (use YOUR real Terraform/Entra ID work at Ensono if you have specifics — this is a strong template)

**Scenario:** Disabled AD user accounts weren't being consistently removed from security/distribution groups, creating stale access and audit findings.

**Approach:**
```powershell
# Get all disabled AD accounts
$disabledUsers = Get-ADUser -Filter {Enabled -eq $false} -Properties MemberOf

# Load a protected-groups allowlist to avoid touching sensitive groups
$protectedGroups = Get-Content "C:\Scripts\ProtectedGroups.txt"

foreach ($user in $disabledUsers) {
    foreach ($group in $user.MemberOf) {
        $groupName = (Get-ADGroup $group).Name
        if ($groupName -notin $protectedGroups) {
            try {
                Remove-ADGroupMember -Identity $group -Members $user -Confirm:$false -ErrorAction Stop
                "$(Get-Date) SUCCESS: Removed $($user.SamAccountName) from $groupName" | Out-File -Append "C:\Logs\GroupCleanup.log"
            }
            catch {
                "$(Get-Date) FAILED: $($user.SamAccountName) from $groupName - $($_.Exception.Message)" | Out-File -Append "C:\Logs\GroupCleanup.log"
            }
        }
    }
}
```
Scheduled as a task under a **least-privilege service account** (delegated only "modify group membership" rights, not Domain Admin), with a **summary email** (`Send-MailMessage`) to security team, and a CSV export of results for audit purposes.

**Outcome to state in an interview:** "Reduced audit findings on stale access by roughly X% over two quarters" — use a real number if you have one from your actual Ensono/FIS work.

## 10.3 Other Common L3 PowerShell Use Cases (know these by name, be ready to discuss)

- **Bulk AD/Entra ID user provisioning/deprovisioning** from HR CSV export (`Import-Csv` → `New-ADUser`/`New-MgUser` loop).
- **Azure resource provisioning/auditing** via Az module (`Get-AzVM`, `Get-AzResource -Tag`) — e.g., a script that reports all VMs missing a required cost-center tag, feeding into your FinOps governance work.
- **Certificate expiry monitoring** — script querying `Get-ChildItem Cert:\LocalMachine\My` across servers, alerting X days before expiry (directly ties to your PKI weak spot — a great "I automated the fix" answer).
- **M365/Exchange Online bulk operations** via Graph PowerShell SDK (`Get-MgUser`, `Update-MgUser`) — note MSOnline/AzureAD modules are deprecated, Graph SDK is current.
- **Intune/Autopilot hash collection** — `Get-WindowsAutoPilotInfo` script for bulk device registration (ties to your Intune weak spot).

## 10.4 Terraform vs PowerShell — know the distinction for interview

Your resume lists Terraform (IaC) — be ready to explain: **Terraform declares desired end-state infrastructure** (VMs, VNets, storage) and reconciles current vs desired state; **PowerShell is imperative** — you write the exact steps to execute. Interviewers may ask when you'd use each — Terraform for provisioning/repeatable infra deployment, PowerShell for operational tasks, ongoing automation, and remediation scripts that don't map cleanly to "declare infrastructure state."

## INTERVIEW QUESTIONS
🔴 Walk me through a real PowerShell automation you built — what problem, what modules, how did you handle errors?
🔴 How do you ensure a bulk-modification script doesn't cause accidental damage?
🟠 Difference between Terraform and PowerShell for infrastructure tasks?
🟠 How would you automate certificate expiry monitoring across multiple servers?
🟡 What's the current recommended module for Entra ID/M365 PowerShell administration, and what did it replace?
🔴 Scenario: you need to bulk-remove 500 users from a stale distribution list safely — what does your script look like?

## L3 ANSWER
"A good example is a script I built to clean up group memberships for disabled AD accounts — we had recurring audit findings because disabled users weren't being removed from security groups. The script pulls all disabled accounts, checks their group memberships, and removes them, but I built in a protected-groups allowlist first so it would never touch something like Domain Admins even if a disabled account was somehow still a member there. Everything runs under a least-privilege service account, wrapped in try/catch so failures get logged with the actual error rather than the script silently skipping, and it emails a summary to the security team with a CSV for the audit trail. That kind of safety net — the allowlist and the logging — is really what separates a script that's safe to run unattended on a schedule from one that just happens to work when you test it manually."

## MEMORY TRICK
**Good script = Try/Catch + Least Privilege + Allowlist/Safety + Logging + Measurable Outcome.**
