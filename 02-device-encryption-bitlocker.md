# Device Encryption and Security Policies with Microsoft Intune

Source: Microsoft Learn module "Implement device encryption and security policies using Microsoft Intune", module 2 of 6 in the "Protect devices using Microsoft Intune" learning path (MD-102 material).

Module completed 2026-09-08.
Achievement: https://learn.microsoft.com/api/achievements/share/en-us/DatDo-7483/9AB5MHGU?sharingId=15FA99AA28223E58

## Main takeaways

- Encryption protects data at rest only. It shrinks the impact of a lost device, and Zero Trust treats it as baseline device health: compliance verifies encryption, and Conditional Access revokes access when a device falls out of compliance.
- Configure BitLocker with an Endpoint security disk encryption policy (preferred over the settings catalog). TPM-only startup gives silent encryption; TPM+PIN adds authentication at the cost of user interaction. Encryption method is fixed when encryption first starts.
- Escrow is the foundation: require recovery information to be stored in Microsoft Entra ID before protection starts, and rotate any key that has been read. Self-service retrieval exists through Company Portal and Entra, but the helpdesk path is still needed for the hard cases.
- Windows Automatic Device Encryption can encrypt Windows 11 24H2 devices before Intune's policy arrives, so an encrypted device showing a profile error is usually a method mismatch, not a failed assignment.
- Personal Data Encryption is file and folder level, bound to Windows Hello, layered on top of BitLocker, and has no admin recovery key; restore from backup if credentials are lost.
- In the encryption report, readiness (can the device support managed encryption) and status (is the OS drive encrypted) are different questions; troubleshoot encryption state before compliance, because Conditional Access acts on compliance.

## 1. Why encryption matters

A lock screen protects only the running OS. On an unencrypted laptop, an attacker removes the SSD, attaches it to another machine, and reads everything — files, cached email, saved credentials. Or boots from USB and bypasses the OS entirely.

Full-disk encryption (BitLocker on Windows, FileVault on macOS) scrambles the drive. The key is tied to the user's login and the device's TPM, so without it the data is unreadable. This changes the impact of a lost device: unencrypted, it is a potential data exposure triggering incident response and notification duties; encrypted, it is a hardware recovery problem. Legal/privacy/compliance teams decide which obligations apply — encryption is the technical control that shrinks the risk.

Scope reminder: encryption protects **data at rest** only. Data in transit needs TLS/HTTPS; data in use needs memory protection and application security.

Zero Trust connection: no device is implicitly trusted, and encryption is a baseline health requirement. The chain: Intune compliance policies verify BitLocker/FileVault is active → if a user disables encryption or malware corrupts the encryption state, the device is flagged noncompliant → Entra Conditional Access revokes access to corporate resources until re-encrypted.

Modern key management removes the old "lost key = lost data" problem through automated escrow: Intune deploys an endpoint security profile that enables encryption silently, the device generates a recovery key and transmits it to its device record in Entra ID, and helpdesk admins (or the users themselves, via the MyAccount portal) can retrieve it.

## 2. BitLocker policy configuration in Intune

Two places to configure BitLocker:

- **Endpoint security → Disk encryption** — dedicated policy, keeps encryption settings separate from unrelated configuration. Recommended for most deployments.
- **Devices → Configuration profiles → Settings catalog** — needed when BitLocker must live in one profile with other Windows settings or when you need granular control; policy review is more complex. Even here, a separate BitLocker policy is still recommended for clean assignment.

A typical corporate configuration: encrypt the OS drive and fixed data drives, escrow recovery keys to Entra ID, TPM-based key protection. The TPM validates device state at boot before releasing the OS-drive key, which defeats drive removal and startup tampering.

Key settings:

- OS drive, fixed drives, removable drives (removable: control write access or require encryption, to limit USB leakage).
- Encryption method: XTS-AES 128 or 256 — only applied when encryption first starts.
- Startup authentication: TPM-only for silent encryption (no prompts), TPM+PIN for stronger protection (more interaction).
- Recovery keys stored in Microsoft Entra ID.

**Windows Automatic Device Encryption (ADE):** Windows itself can encrypt the OS drive during OOBE, before Intune's policy arrives — triggered when setup completes with a Microsoft or work/school account. From Windows 11 24H2, ADE no longer requires Modern Standby or HSTI, so many new devices arrive already encrypted. When the Intune policy hits an already-encrypted device, Intune validates the existing state; if the ADE-applied method differs from policy, the encryption report can show an encrypted device with a profile error. That is not always an assignment failure — encryption method is fixed at first encryption, so a mismatch means either align the policy or plan decrypt/re-encrypt. For most new 24H2 devices the Intune work is escrow, validation, and monitoring rather than initiating encryption.

Assignment and user experience: device-based assignment for corporate/shared devices (consistent regardless of who signs in); user-based where policy should follow the user. Silent enablement starts encryption with no prompts and without local admin rights, provided prerequisites are met (supported edition, TPM, compatible startup config). Standard enablement shows prompts and creates support variance when users delay or misunderstand them.

## 3. Personal Data Encryption (PDE)

PDE is file-based, not volume-based. BitLocker protects whole volumes and releases the drive key at startup when protectors pass. PDE protects **selected user folders** while Windows is running, releasing file access only after the user signs in with **Windows Hello**. It covers a different risk than BitLocker: a powered-on or locked device where sensitive files are still reachable.

Rules: PDE works alongside BitLocker, never replaces it. Enable BitLocker first as the baseline, then layer PDE. Configure from Endpoint security → Disk encryption → Create policy → platform Windows → profile Personal Data Encryption → enable and choose folders (Desktop, Documents, Pictures).

Prerequisites: Windows 11 22H2+, Intune enrollment, Entra joined or hybrid joined, Windows Hello sign-in.

Recovery is fundamentally different: BitLocker has admin-retrievable recovery keys; PDE does not. PDE access depends on the user's protected credentials and working sign-in. If the credential is permanently lost, restore PDE-protected files from backup. PDE also does not show up in the BitLocker encryption report — verify via PDE policy status, device configuration status, and the PDE protection indicator in File Explorer on a test device.

## 4. Recovery keys and self-service

Escrow is the foundation: the BitLocker policy should save recovery information to Entra ID and **require the backup to complete before BitLocker enables protection** — no encryption starts without a recoverable key. Require the 48-digit recovery password format. Decide in advance who can retrieve keys and how requests are verified.

A recovery password is sensitive: anyone holding it can unlock the drive. Store only in approved locations, gate with RBAC.

Self-service paths:

- Company Portal website — sign in, select the locked device, "Get recovery key"; key is shown for a limited session.
- Company Portal app (mobile/desktop) — select the managed Windows device before viewing.
- Entra self-service — tenant settings control whether users can view keys for devices associated with them.

Self-service does not replace the helpdesk. Helpdesk path is still needed when ownership is unclear, the key is missing, the user cannot sign in, or org policy restricts self-service. The technician process: verify the user's identity, record the device name, record the recovery key ID from the recovery screen, locate the matching password, document the reason, read out the 48-digit value carefully.

Administrative retrieval and least privilege: built-in roles such as Helpdesk Administrator or Cloud Device Administrator can read BitLocker keys in Entra ID; a custom role can grant only `microsoft.directory/bitlockerKeys/key/read`. Every key read creates an audit record — review key management events in Entra audit logs.

Rotation after disclosure — a viewed key is a disclosed key and should not stay valid:

- Manual: Intune remote action "BitLocker key rotation" on the device; new password is generated and escrowed per policy.
- Automatic: policy setting "Configure recovery password rotation" in the BitLocker policy — options are Refresh off (default), Refresh on for Entra ID-joined devices, Refresh on for both Entra ID-joined and hybrid-joined. Rotates after a recovery password is used. Requires Windows 10 1909+ or Windows 11. Manual rotation works regardless of this setting.
- Also rotate when a device changes owner or returns from service, or when a key shows up in an incident review (then investigate the access event).

If a device shows no key in Entra: check that policy backs up recovery info before encryption, and that the device checks in.

## 5. Monitoring in Intune

Encryption report: Devices → Monitor → Device encryption status. Fields: device name, OS/version, TPM version, **encryption readiness** (can the device support managed encryption — TPM etc.), **encryption status** (is the OS drive actually encrypted), UPN. Readiness and status answer different questions: a device can be ready but not yet encrypted, or encrypted but in profile error because its method/protectors mismatch policy.

Result patterns and responses:

- Ready + encrypted → protected; check compliance state if access depends on it.
- Ready + not encrypted → check for required user interaction, recent check-in, encryption in progress.
- Not ready + not encrypted → review TPM, Windows Recovery Environment, OS support, policy settings.
- Encrypted with profile error → compare policy against the device's actual method and protectors (often ADE-applied).
- Unknown/not applicable → sync the device, wait for check-in, or gather client-side data.

Timing note: devices report encryption state at check-in, and compliance evaluation via health attestation can require a reboot — a stale compliance result is not automatically a policy problem.

Compliance side: settings "Require BitLocker" (evaluated via Windows health attestation / TPM-based health data) and "Encryption of data storage on a device" (OS-drive-level check, BitLocker on Windows). The encryption report says whether the device is encrypted; the compliance view says whether that satisfies your rules. Conditional Access acts on compliance, so an encryption failure can cut off resource access even when everything else passes. Troubleshoot in that order: encryption state first, then compliance.

Fleet analysis: export the report to CSV and group by readiness, status, profile state, OS/TPM version, or user. Patterns: many "not ready" = hardware/firmware issue (group by model/TPM); many "ready but not encrypted" = assignment or user-interaction issue; uniform profile errors on encrypted devices = policy/method mismatch; recovery-key-backup failures = escrow config; noncompliant-after-encryption = needs sync/reboot. Prioritize by business risk, not raw counts.

## 6. Auditing encryption with Defender

Defender does not replace the Intune report — it gives SecOps a hunting view of whether endpoint posture matches the baseline, using Defender Vulnerability Management secure configuration assessments plus advanced hunting (KQL).

Relevant tables: `DeviceTvmSecureConfigurationAssessment` (per-device results: DeviceId, ConfigurationId, IsApplicable, IsCompliant, Timestamp, Context) and `DeviceTvmSecureConfigurationAssessmentKB` (readable names, risk, remediation guidance). The assessment table says what a device reports; the KB says what it means. `DeviceTvmInfoGathering` adds inventory context; exports give snapshots for offline/long-term audit.

Audit pattern (don't hard-code configuration IDs — names and availability change by platform and licensing):

1. Discover: query the KB for ConfigurationName/Description/RiskDescription containing "BitLocker"/"Encryption", project the IDs, order by ConfigurationImpact.
2. Query devices: join assessment (filtered `IsApplicable == true`) to those ConfigurationIds, left-join latest inventory, prioritize by IsCompliant asc and ConfigurationImpact desc.
3. Detect anomalies: group by device — flag devices failing ≥1 applicable control or with assessment timestamps older than a freshness threshold (e.g. 7 days). Failed control → compare with Intune encryption report; stale timestamp → check sensor health, onboarding, connectivity.
4. Correlate with Intune before concluding: Intune remains the source of truth for policy assignment, readiness, escrow, and compliance.

Result interpretation: applicable+noncompliant = in scope, failing; applicable+compliant = audit evidence; missing inventory fields = onboarding/sensor problem.

Advanced hunting has limited retention — export snapshots if you need trend history.

## 7. Knowledge check — answers and reasoning

1. Silent BitLocker rollout, TPM 2.0 devices → **Endpoint security disk encryption policy that uses TPM-only startup authentication and stores recovery keys in Microsoft Entra ID.** TPM-only means no startup prompts (TPM+PIN adds user friction), disk encryption policy is the recommended surface, and Entra escrow is mandatory for a managed recovery path. Disabling escrow is the opposite of correct — it removes the recovery path.

2. Technician read the recovery password to the user; what next → **Trigger the BitLocker key rotation remote action in Intune.** A read password is a disclosed password and must not remain valid. Deleting the device record doesn't invalidate the key, and emailing the password increases exposure.

3. Only support team reads recovery keys, least privilege → **Built-in helpdesk role or a custom role granting only the BitLocker key read permission, scoped to supported devices.** Global Administrator is the opposite of least privilege; shared service accounts break accountability and auditing.

4. Resilient, risk-prioritized encryption audit in Defender → **Query the KB first for BitLocker/Encryption configurations, join those IDs to DeviceTvmSecureConfigurationAssessment with IsApplicable == true, prioritize by ConfigurationImpact and compliance.** Hard-coding one ConfigurationId is brittle and narrow; skipping Defender loses the security-operations view entirely.

5. Prevent encryption starting without a recoverable key → **Save BitLocker recovery information to Microsoft Entra ID and require recovery information stored before BitLocker starts.** That is the pre-encryption backup requirement. Disabling escrow or relying on the user to record a PIN both create unmanaged keys.

## 8. Self-check

- I can explain why a login password is not data-at-rest protection, and what encryption changes for breach-notification obligations.
- I can state the three-part Zero Trust chain: compliance evaluation → noncompliant flag → Conditional Access revocation.
- I can choose between disk encryption policy and settings catalog, and justify the dedicated policy default.
- I can explain what ADE does, what changed in Windows 11 24H2, and why an encrypted device can show a profile error.
- I can position PDE against BitLocker, list its prerequisites, and explain why its recovery model has no admin key.
- I can configure escrow so no encryption starts without a recoverable key, and name the self-service and helpdesk recovery paths.
- I know when key rotation is required and both ways to trigger it.
- I can read the encryption report's readiness vs. status distinction and the five result patterns.
- I can explain the two-table Defender audit pattern (KB discovery → assessment join) and why not to hard-code ConfigurationIds.
