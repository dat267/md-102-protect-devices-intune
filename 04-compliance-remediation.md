# Compliance Enforcement and Remediation with Microsoft Intune

Source: Microsoft Learn module "Enforce compliance and remediate security issues by using Microsoft Intune", module 4 of 6 in the "Protect devices using Microsoft Intune" learning path (MD-102 material).

Module completed 2026-09-15.
Achievement: https://learn.microsoft.com/api/achievements/share/en-us/DatDo-7483/NQGH5VAF?sharingId=15FA99AA28223E58

## Main takeaways

- Configuration profiles force a state and compliance policies verify it. A device reaches corporate data only by proving it meets the baseline, and Conditional Access is what actually blocks access; Intune only changes the status tag.
- Compliance settings are platform-specific, covering device health, device properties, and system security. Dynamic risk (the Defender or Mobile Threat Defense machine risk score) catches a compromised device that static checks would pass.
- Target with user groups, device groups, dynamic groups, or assignment filters. Filters avoid group bloat for small variations, and dynamic device groups cannot reference compliance status among their supported attributes.
- Actions for noncompliance form a graduated timeline (mark noncompliant with a grace period, email, push notification, retire list, retire). During the grace period the device is reported as "In grace period", an honest noncompliant state that still gets access until the schedule expires.
- Fix drift declaratively with configuration profiles, and custom problems with Remediations (detection plus remediation PowerShell delivered by the Intune Management Extension on Windows) or shell scripts on macOS.
- Monitor in Devices > Compliance > Monitor and Reports > Device compliance, use per-setting drill-down to find the failed setting, and export CSV or JSON for audit evidence. Also set "mark devices with no compliance policy assigned as" to Not compliant.

## 1. The core distinction: configure vs. prove

Configuration profiles do the work (turn on BitLocker); compliance policies do the checking (verify BitLocker is actually on). A device can only access corporate data if it proves it meets the baseline. "Configuration forces the state; compliance verifies the state." They are deployed as separate policies for the same feature — name them so nobody deletes the wrong one (e.g. `Win10-Config-BitLocker` vs `Win10-Comp-BitLockerRequired`).

Static checks alone are insufficient. A device can have a strong password, current OS, and encryption — and still be actively compromised. The dynamic layer closes that gap: Mobile Threat Defense partners and Microsoft Defender for Endpoint assign the device a machine risk score (Clear, Low, Medium, High); Intune compliance reads it continuously, and setting "require machine risk score at or under Low" marks a compromised device noncompliant without waiting for a scheduled check-in.

## 2. Compliance policy design

Create path: Devices → Compliance → Policies → Create policy → pick platform → name per convention → compliance settings → actions for noncompliance → assignments → create.

What to check per category:

- Device health — Windows: health attestation (Secure Boot, BitLocker, code integrity); iOS: no jailbreak; Android: no root.
- Device properties — minimum/maximum OS version; raising the minimum is the primary lever after a zero-day patch (unpatched devices fall out of compliance immediately).
- System security — password required, complexity, max inactivity before lock, encryption of storage, firewall and antivirus active.

Platform specifics (you cannot translate a Windows policy to iOS — different security architectures, and corporate vs. BYOD changes what Intune may check):

- Windows 10/11 — deep stack: Secure Boot, TPM health, Defender Firewall and Antivirus; can enforce a maximum MDE risk score.
- iOS/iPadOS — jailbreak status, Secure Enclave-backed passcode rules, OS currency.
- Android Enterprise — BYOD work profile: container and basic health only; fully managed: whole device, Play Protect, root status, Play Integrity levels (basic → device → strong integrity).
- macOS — FileVault, System Integrity Protection, Gatekeeper.

Custom compliance (Windows and macOS): when native settings don't cover it, upload a PowerShell or Bash script checking specific criteria (registry key, running service, app version); results feed the compliance evaluation.

Two traps worth memorizing: Windows LAPS is not a compliance setting — it lives under Endpoint security → Account protection (compliance evaluates state; LAPS actively rotates local admin passwords). And the tenant-wide setting "Mark devices with no compliance policy assigned as" must be **Not compliant** — the default "Compliant" lets a brand-new, unchecked device pass Conditional Access.

## 3. Assignments: groups and filters

- User groups: policy follows the user across every device they sign into — the default for baselines (e.g. BitLocker for Finance).
- Device groups: policy binds to hardware regardless of user — kiosks, shared devices, conference-room systems.
- Include/exclude assignments: e.g. include All Users, exclude Executive Team during rollout.
- Dynamic groups: rule-based membership from Entra attributes (`user.department -eq "Engineering"`, `device.deviceOwnership -eq "Company"`). Updates propagate automatically when attributes change. Limits: a dynamic group holds users or devices, not both; device rules reference only device attributes, never the enrolling user's.
- Assignment filters: narrow an assignment to a broad group by device properties at check-in time — include mode (only matching devices get the policy) or exclude mode (matching devices skip it). Prefer filters over creating a group per variation, to avoid group bloat. Example: strict iOS policy assigned to All Users with an include filter `device.deviceOwnership -eq "Corporate"` hits corporate iPhones and skips personal iPads. Filters are created under Tenant administration → Filters (or Devices → Filters), platform-scoped, rule-builder or syntax editor.

Timing: policies apply at device check-in (~8 hours scheduled, sooner after enrollment) — not instantly.

Note the boundary (module 3's knowledge check leaned on it): dynamic device groups **cannot** reference Intune compliance status. Supported attributes include deviceCategory, deviceOSType, deviceOSVersion, deviceOwnership, deviceTrustType, enrollmentProfileName, extensionAttribute1-15, isRooted, managementType, objectId — compliance state is not among them.

## 4. Actions for noncompliance

Intune doesn't block access itself — it changes the device's status tag; Conditional Access enforces downstream. Blocking instantly floods the helpdesk, so build a graduated timeline:

- **Mark device noncompliant** (default action, 0 days by default). Configurable to a schedule, creating a **grace period**. During the grace period the device is reported to Entra ID in a distinct **"In grace period"** state — honestly noncompliant, but not the final "Not compliant" state — and standard compliance-requiring Conditional Access policies still allow access. No special CA grant control is needed; the schedule plus standard policies is the whole mechanism. After the schedule expires, the state flips to Not compliant and CA blocks.
- **Send email to end user** — from a Notification message template (logo, dynamic variables like `{{DeviceName}}`, link to fix instructions); can CC manager/helpdesk after days of noncompliance. Delaying email by a day lets config profiles and remediation scripts fix things silently first.
- **Send push notification** — via Company Portal/Intune app; Android and iOS/iPadOS only, **not Windows**; delivery not guaranteed, so not for urgent messages.
- **Add device to retire list** — surfaces the device under Devices > Compliance > Retire noncompliant devices for admin review; retirement is not automatic.
- **Retire the noncompliant device** — final escalation (30/60/90 days): issues Retire, severing management and deleting corporate data, Wi-Fi profiles, apps.

A workable timeline: Day 0 detected → "In grace period", email Day 1 → grace expires Day 3, CA blocks → push notification Day 7 (mobile) → retire-list review / retire Day 30.

## 5. Remediation: self-healing

Declarative fixes first: configuration profiles continuously re-apply settings when they drift. A user disables the firewall; the firewall profile re-enables it at next sync; compliance re-evaluates; Conditional Access restores access — zero tickets.

Custom fixes on Windows: **Remediations** (formerly Proactive Remediations) — two-part PowerShell package delivered by the Intune Management Extension to Entra-joined or hybrid-joined Windows Pro/Enterprise/Education. Detection script runs on a schedule; non-zero exit code triggers the remediation script, which fixes the condition (e.g. repair a registry key) — often before the compliance policy even flags the device. On macOS the equivalent is shell scripts (Devices > macOS > Scripts), a separate feature.

Targeting remediations: don't run heavy scripts fleet-wide. Option A (recommended): assignment filters — tag devices by category or attribute (`device.deviceCategory -eq "VPN-Managed"`) and filter the script assignment. Option B: assigned group + actions for noncompliance — since "add device to Entra ID group" is not a built-in action, surface noncompliant devices (retire list, email, remote lock) and have an admin or automation (e.g. a Logic App) add them to the remediation group; remember devices don't leave the group automatically.

Manual actions from the device overview page: **Sync** (force immediate check-in), **Update Windows Defender Security Intelligence** (force signature refresh), **Restart** (finish pending updates, clear hung states).

## 6. Monitoring and reporting

Views: Devices → Compliance → Monitor for the dashboard (compliant, noncompliant, without policy, un-evaluated counts) and per-policy drill-down; Reports → Device compliance → Reports for tenant-wide, filterable, exportable data.

Reading results: per-device vs. per-setting drill-down pinpoints the exact failed setting (a missing update vs. a high risk score). Reports show the last signed-in user; device-group-targeted policies may show "System" when nobody is signed in.

Audit evidence: filter the Device compliance report (state, OS, ownership), Generate report, adjust columns, Export to CSV/JSON. For enterprise analytics, export the data warehouse into Power BI and combine with Entra sign-in logs and Defender signals.

Operational rhythms: daily helpdesk use of per-device view ("you're blocked because the update needs a reboot"); hunting persistently noncompliant devices for remediation; trend-spotting (500 macOS devices failing overnight = policy conflict or bad OS update, not user error); quarterly exports for auditors.

## 7. Knowledge check — answers and reasoning

1. BitLocker required but devices noncompliant; what enforces encryption so devices self-heal → **A device configuration profile (or Endpoint Security disk encryption policy) that actively configures BitLocker.** Compliance checks state, configuration creates it — the module's core distinction. A second compliance policy adds another alarm with no fix; Conditional Access blocks but doesn't repair.

2. Strict policy to all Windows devices except executive laptops, without a new Entra group → **An assignment filter evaluating device properties at assignment time.** Filters exist precisely to avoid group bloat. A dynamic group per executive laptop is the anti-pattern; scope tags control admin visibility, not device targeting.

3. "Mark device noncompliant" scheduled 3 days — how does Entra see a just-failed device → **Reported as "In grace period" (still a noncompliant state), and standard compliance-requiring Conditional Access policies block by default.** Note this is the knowledge-check framing — the unit text itself says in-grace devices retain access until the period expires. The exam point either way: "In grace period" is a distinct, honest state — never reported as Compliant, never retired automatically.

4. Auto-detect and fix a missing LOB registry key → **Intune Remediations (detection script + remediation PowerShell script).** A compliance policy can check the key but not fix it; Conditional Access only gates access.

5. Auditor wants offline point-in-time evidence of BitLocker + Defender AV state → **Generate and export compliance reports (Devices > Compliance > Monitor).** Per-device browsing doesn't scale and isn't point-in-time; Entra sign-in logs don't carry compliance data.

## 8. Self-check

- I can state the configure-vs-verify split and give a worked example for one setting.
- I can list the four compliance categories and the platform differences for Windows, iOS, Android Enterprise (BYOD vs fully managed), and macOS.
- I know where LAPS is configured and why "mark devices with no policy assigned" must be Not compliant.
- I can choose user group vs. device group vs. dynamic group vs. assignment filter for a given targeting scenario, and name the dynamic-group attribute limits (including the compliance-status exclusion).
- I can build the escalation timeline and explain the "In grace period" state's effect on Conditional Access.
- I can pick declarative config vs. Remediations vs. shell scripts for a given fix, on each OS.
- I can produce auditor-ready exports and explain per-device vs. per-setting reporting.
