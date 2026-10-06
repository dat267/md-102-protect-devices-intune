# MD-102: Protect devices using Microsoft Intune

Notes from the Microsoft Learn learning path **Protect devices using Microsoft Intune** (six modules). The material maps to MD-102 Endpoint Administrator; the Defender investigation units also support SC-200.

Learning path completed 2026-09-29.
Achievement: https://learn.microsoft.com/api/achievements/share/en-us/DatDo-7483/VSY3VCWM?sharingId=15FA99AA28223E58

Each module file contains main takeaways, full notes, the knowledge-check answers with distractor reasoning, and a self-check. Achievement links are in each file header.

## Modules

1. [Endpoint security with Microsoft Defender and Microsoft Intune](01-defender-endpoint-security.md)
   Defender capabilities and architecture, Intune-versus-Defender division of labor, onboarding via EDR policy, security baselines and endpoint security policies, alert and incident triage, automated investigation and remediation.
   Achievement: https://learn.microsoft.com/api/achievements/share/en-us/DatDo-7483/ABUNJZ87?sharingId=15FA99AA28223E58

2. [Device encryption and security policies](02-device-encryption-bitlocker.md)
   BitLocker policy surfaces and settings, Automatic Device Encryption, Personal Data Encryption, recovery key escrow and rotation, the encryption readiness-versus-status report, and auditing encryption with Defender.
   Achievement: https://learn.microsoft.com/api/achievements/share/en-us/DatDo-7483/9AB5MHGU?sharingId=15FA99AA28223E58

3. [Advanced threat protection](03-advanced-threat-protection.md)
   The four defense-in-depth layers, cloud app discovery and shadow IT investigation, attack surface reduction rules and the audit-first rollout, Zero Trust outline, proactive remediation scripts.
   Achievement: https://learn.microsoft.com/api/achievements/share/en-us/DatDo-7483/WM59BWWN?sharingId=15FA99AA28223E58

4. [Compliance enforcement and remediation](04-compliance-remediation.md)
   Configuration versus compliance, platform-specific checks and machine risk score, group and filter targeting, actions for noncompliance and the grace period, Remediations, compliance reporting and audit exports.
   Achievement: https://learn.microsoft.com/api/achievements/share/en-us/DatDo-7483/NQGH5VAF?sharingId=15FA99AA28223E58

5. [Secure mobile access with Microsoft Tunnel](05-tunnel-mobile-access.md)
   Tunnel Gateway deployment on Linux, server configurations and sites, VPN profiles, Tunnel for MAM on unenrolled BYOD, monitoring with Health status and Trends, mst-cli and log troubleshooting.
   Achievement: https://learn.microsoft.com/api/achievements/share/en-us/DatDo-7483/ZJNQ3L52?sharingId=15FA99AA28223E58

6. [Cloud PKI](06-cloud-pki.md)
   Cloud-hosted CA hierarchy (root plus issuing, and BYOCA), licensing limits, trust deployment, SCEP profile configuration and the enrollment flow, revocation and CRL behavior, certificate health monitoring.
   Achievement: https://learn.microsoft.com/api/achievements/share/en-us/DatDo-7483/K9VXKW7B?sharingId=15FA99AA28223E58

7. Deploy and Manage Applications Using Microsoft Intune
   Achievement: https://learn.microsoft.com/api/achievements/share/en-us/DatDo-7483/P62X7EQ4?sharingId=15FA99AA28223E58

## Slides

- [intune-device-security.html](slides/intune-device-security.html) - self-contained slide deck (24 slides), open directly in a browser. Covers three of the six modules: compliance and remediation, Microsoft Tunnel, and Cloud PKI.
- [transcript.md](slides/transcript.md) - speaker transcript for the deck, timed for a 15-minute talk at a slow speaking pace.
