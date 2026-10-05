# Advanced Threat Protection with Microsoft Intune and Microsoft Defender

Source: Microsoft Learn module "Implement advanced threat protection using Microsoft Intune and Microsoft Defender", module 3 of 6 in the "Protect devices using Microsoft Intune" learning path (MD-102 material).

Module completed 2026-09-08.
Achievement: https://learn.microsoft.com/api/achievements/share/en-us/DatDo-7483/WM59BWWN?sharingId=15FA99AA28223E58

## Main takeaways

- Defense in depth runs across four layers: prevention, detection, response, and recovery and improvement. Intune owns management and policy; Defender owns detection, threat intelligence, and response, tied together by the risk signal into compliance and Conditional Access.
- Cloud Discovery finds shadow IT from snapshot reports, continuous reports, endpoint integration, or network appliances. Use endpoint integration when devices bypass the corporate proxy, and investigate an app before enforcing a block on it.
- ASR rules block attacker behaviors rather than specific malware and can break line-of-business apps, so roll out pilot group, then Audit, then exclusions only where needed, then Block.
- Audit mode shows what would have been blocked without any user impact, which is why it is the safe first step; rule actions are Not configured, Off, Audit, Warn, and Block.
- Proactive remediation (a detection script plus a remediation script on a schedule) is the tool for repeatedly detecting and fixing a configuration problem. ASR rules do not fix configuration and compliance policies only report state.
- Zero Trust for endpoints is the same continuous-verification loop: Defender risk signal, Intune compliance evaluation, Conditional Access decision.

Coverage note: the saved pages cover layered strategy, cloud app discovery, and ASR rules. The module also includes units on Zero Trust endpoint protection and proactive remediation scripts; the latter is summarized in section 5 from the knowledge check and module outline.

## 1. Layered protection

Advanced attacks chain techniques — phishing email, malicious script, vulnerable app, lateral movement. Since no single technique is used, no single control suffices. Defense in depth across the threat lifecycle has four layers:

1. **Prevention** — block threats before impact: security baselines, Defender Antivirus, firewall and network protection, ASR rules, vulnerability management, access controls.
2. **Detection** — monitor for suspicious activity and IoCs: Defender endpoint signals, behavioral analytics, threat intelligence, cloud protection; EDR, AIR, advanced hunting, vulnerability management.
3. **Response** — contain and neutralize: isolate devices, run scans, collect investigation packages, quarantine files, AIR, and automatic attack disruption (Defender correlating endpoint, identity, email, and app signals to disrupt an in-progress attack).
4. **Recovery and improvement** — restore devices, remediate vulnerabilities, update policies, review findings.

Division of labor: Intune is the management and policy layer (onboarding, endpoint security policies, baselines, AV settings, ASR, firewall, compliance, remediation via policy or security tasks). Defender is the detection, intelligence, and response layer (monitoring, risk evaluation, vulnerability identification, investigation, response). The risk-signal chain ties them together: Defender detects active malware → device marked high risk → Intune compliance evaluates the risk → Conditional Access restricts access until remediation.

## 2. Cloud app discovery (shadow IT)

Defender's Cloud Discovery shows which cloud services are in use, by whom, from which devices, and how risky they are — analyzed against the Cloud App Catalog.

Data collection options:

- **Snapshot reports** — manually uploaded firewall/proxy/secure web gateway logs; point-in-time view.
- **Continuous reports** — automatic log upload via a log collector; ongoing visibility.
- **Defender endpoint integration** — discovery from managed endpoints; the right choice for remote users or devices that bypass the corporate firewall/proxy.
- **Network appliance integrations** — supported firewalls, proxies, SWGs.

Review in the Defender portal: Cloud apps > Cloud discovery. Discovered apps can be filtered by risk level, category, users, traffic volume, and usage pattern; each app is sanctioned, unsanctioned, or unreviewed.

The Cloud App Catalog scores apps on compliance, legal, security, privacy, and general risk factors — use it to compare apps against requirements (encryption, audit logs, MFA support, certifications, data retention) and tag apps Sanctioned/Unsanctioned for governance.

Continuous monitoring: discovery policies (alert on new risky apps, high-volume unsanctioned use, specific risk factors), OAuth app monitoring (third-party apps connected to corporate accounts and their delegated permissions), and app governance (permissions, consent, app-to-app access patterns).

Response principle: investigate before enforcing. Not every risky app gets blocked — some have a valid business case and need education, review, or a sanctioned replacement. Investigation checklist: business-approved or shadow IT? Catalog risk score and factors? Which users, devices, IPs? Traffic volume and trends? Sanctioned status? OAuth permissions and who consented? Anomalies such as upload spikes or unexpected locations? High-risk patterns: mass adoption of an unsanctioned storage app, low-rated service receiving large uploads, OAuth app requesting broad mail/file access.

## 3. Attack surface reduction rules

ASR rules target behaviors used in attacks rather than specific malware: Office apps creating child processes, executable content from email/webmail, credential theft from LSASS (lsass.exe), JavaScript/VBScript launching downloaded executables, abuse of vulnerable signed drivers, WMI event-subscription persistence, and copied/impersonated system tools (Living-off-the-Land techniques).

Rules are for supported Windows devices, configured in Intune from Endpoint security → Attack surface reduction → Create policy → platform Windows → profile "Attack Surface Reduction Rules".

Rule actions: Not configured (policy doesn't touch it; Windows default usually off), Off, **Audit** (logs what would be blocked without blocking), **Warn** (blocks but may allow user bypass), **Block**.

Deployment pattern, because ASR rules can break line-of-business apps that legitimately behave like malware techniques: pilot group → Audit mode → review events and user impact → exclusions only when required → move to Warn or Block → expand to larger groups.

Monitoring: Intune shows deployment status, assignment results, errors, conflicts. The Defender portal's ASR report shows rules enforced, detected and blocked threats, and devices not configured for standard protection rules.

## 4. Zero Trust for endpoints (outline)

Continuous verification of identity, device health, compliance, and risk: Defender risk signals feed Intune compliance policies; Conditional Access enforces access decisions on the result. (Dedicated unit page not saved; the mechanics are covered in modules 1–2 notes, sections 2 and 1 respectively.)

## 5. Proactive remediation scripts (outline)

Intune proactive remediation = a script package with a **detection script** and a **remediation script**, running on a schedule against assigned devices. Detection checks for a problem state (e.g. a misconfigured registry value); if detected, remediation fixes it — no manual intervention. Fits the requirement "repeatedly detect and fix a configuration problem" that neither ASR rules (behavior blocking, not config fixing) nor compliance policies (state evaluation only, no fix) address. (Dedicated unit page not saved.)

## 6. Knowledge check — answers and reasoning

1. ASR rule may break a LOB macro → **Configure the rule in Audit mode for a pilot group and review reported events before enforcing.** Audit shows exactly what would have been blocked, without user impact — that is its purpose. Block-first risks breaking the LOB app; a permanent exclusion before testing defeats the rule.

2. Ongoing cloud discovery for endpoints bypassing the corporate proxy → **Defender's endpoint integration for cloud discovery.** Snapshot reports are point-in-time; a continuous report from the on-premises SWG never sees traffic that doesn't route through it. Endpoint-based discovery follows the managed device regardless of network.

3. Repeatedly detect and fix a misconfigured registry value automatically → **A proactive remediation script package with detection and remediation scripts.** ASR rules block behaviors; compliance policies only flag noncompliance — neither fixes configuration. Remediation scripts run detection on schedule and repair when found.

## 7. Self-check

- I can name the four protection layers and give two concrete controls per layer.
- I can state which layer Intune owns and which Defender owns, and trace the risk-signal → compliance → Conditional Access chain.
- I can pick the right Cloud Discovery data source for a given network topology and explain snapshot vs. continuous.
- I can run the shadow-IT investigation checklist and justify "investigate before enforce."
- I know the five ASR rule actions and can defend the audit-first rollout order.
- I can describe the detection/remediation script pair and place it against ASR and compliance policies as alternative tools.
