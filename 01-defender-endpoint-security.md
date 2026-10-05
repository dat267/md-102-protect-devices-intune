# Endpoint Security with Microsoft Defender and Microsoft Intune

Source: Microsoft Learn module "Implement endpoint security with Microsoft Defender and Microsoft Intune", module 1 of 6 in the "Protect devices using Microsoft Intune" learning path. Maps to MD-102 (Endpoint Administrator Associate); the portal-investigation sections also support SC-200.

Module completed 2026-09-08.
Achievement: https://learn.microsoft.com/api/achievements/share/en-us/DatDo-7483/ABUNJZ87?sharingId=15FA99AA28223E58
Knowledge check: 5/5 (answers in section 8).

## Main takeaways

- Defender's six capability areas are next-generation protection, EDR, attack surface reduction, vulnerability management, automated investigation and remediation, and advanced hunting; they run on endpoint sensors, cloud analysis, and portal correlation, across Windows, macOS, Linux, Android, and iOS.
- Intune manages and configures devices; Defender detects, scores risk, and responds. Onboarding only opens the telemetry channel, so antivirus, ASR, firewall, and web protection each need their own endpoint security policy, and two policy types must never manage the same onboarding setting (conflicts show in Intune policy status).
- Preferred Windows onboarding is an Endpoint detection and response policy with the client package source set to Auto from connector; macOS uses the Defender app plus a configuration profile; mobile platforms use Mobile Threat Defense instead.
- Access becomes risk-based: Defender machine risk feeds Intune compliance, and Entra Conditional Access blocks the device until it is remediated.
- Incidents correlate related alerts; triage asks what happened, who and what is affected, whether the threat is active, and the business impact. AIR verdicts are malicious, suspicious, or no threats found, and Security Copilot assists but does not replace validation in Defender.

## 1. How Defender protects endpoints

Traditional antivirus blocks known malware. Defender is broader: it detects behavior, correlates signals, and supports response. Its six capability areas:

1. Next-generation protection — signatures, behavioral analysis, and cloud machine learning; real-time and scheduled scans; blocks malware and potentially unwanted applications including ransomware.
2. Endpoint detection and response (EDR) — collects detailed endpoint telemetry so analysts can investigate threats that evade prevention: process trees, indicators of compromise, lateral movement.
3. Attack surface reduction (ASR) — rules that block specific techniques such as credential theft, malicious scripts, and executables launched from Office apps; plus application control, web protection, hardware-based isolation.
4. Vulnerability management — continuous assessment of software and configuration weaknesses, prioritized by risk.
5. Automated investigation and remediation (AIR) — Defender investigates alerts on its own and recommends or applies remediation.
6. Advanced hunting — proactive KQL queries across all collected telemetry to find threats that triggered no alert.

Architecture has three parts. Endpoint sensors run locally and collect telemetry (processes, registry, network, files) with low performance overhead, using heuristics and machine learning to flag anomalies. The cloud service analyzes telemetry from all endpoints against threat intelligence gathered from millions of devices, generates alerts, and can trigger automated responses. Integration points correlate endpoint, identity, email, and cloud signals in the Defender portal, and connect Defender to Intune.

Defender runs on Windows, macOS, Linux, Android, and iOS, all visible in one portal.

## 2. Division of labor: Intune vs. Defender

Intune manages devices and deploys policies. Defender provides detection, threat intelligence, device risk signals, vulnerability insights, and response.

When integrated: Intune onboards devices to Defender and delivers security configuration (antivirus, firewall, ASR, EDR); Defender sends device risk signals back to Intune; Intune compliance policies can then mark a high-risk device noncompliant; Entra Conditional Access blocks that device from corporate resources until it is remediated.

This is the core idea of the module: protection becomes risk-based. Instead of asking "is the device enrolled and configured correctly," you also ask "does it currently have active risks," and access control acts on the answer.

## 3. Onboarding devices to Defender

Onboarding is the prerequisite for everything else, but onboarding alone is not protection — it only opens the telemetry channel.

Prerequisites:

- Intune license and Defender for Endpoint license
- Devices enrolled in Intune, joined to Entra ID or hybrid joined
- Service connection between Intune and Defender (one-time tenant configuration)
- Permissions to create/assign Intune policies and view the Defender device inventory
- Supported OS versions and hardware

Enabling the service connection:

1. Intune admin center → Endpoint security → Microsoft Defender for Endpoint
2. If not connected, open the Defender portal → System > Settings > Endpoints > General > Advanced features
3. Turn on "Microsoft Intune connection", save

Windows (recommended method): an Intune Endpoint detection and response policy. The fast path is the preconfigured policy on the "EDR Onboarding Status" tab (Endpoint security → Endpoint detection and response): pick Windows and the EDR profile, name it, create. It always uses the latest onboarding package from your tenant. Use a custom EDR policy instead when you need control over assignments, scope tags, or sample sharing.

For the onboarding package source, prefer "Auto from connector" — it pulls the current package from the live Intune-to-Defender connection. The manual option (upload/paste a package downloaded from the Defender portal) exists for when automatic retrieval is unavailable.

macOS: deploy the Defender app via Intune, and get the onboarding package from the Defender portal (Settings > Endpoints > Device management > Onboarding → macOS, deployment method "Mobile Device Management / Microsoft Intune"), delivered as a configuration profile. Beyond onboarding you also need profiles for system extensions, network filtering, Full Disk Access, background services, notifications, Microsoft AutoUpdate, and Defender preferences — plus optional profiles for network protection, device control, or DLP depending on what you use.

Android and iOS/iPadOS: Defender is onboarded as a mobile threat defense (MTD) solution — deploy the app through Intune with app/device configuration policies, not the Windows EDR onboarding policy. Android adds malware protection for malicious apps and APKs; iOS reports jailbreak detection. Signal-based capabilities include web protection, network protection, unified alerting, privacy controls, and mobile vulnerability assessment.

Two failure modes to remember:

- Deploying an EDR onboarding policy does not configure antivirus, ASR, firewall, web protection, or custom detections. Those need separate endpoint security policies.
- Do not let two policy types manage the same onboarding settings (for example a device configuration policy and an EDR policy). That creates policy conflicts.

## 4. Security settings and baselines

Security baselines give a recommended starting configuration: Endpoint security → Security baselines → "Microsoft Defender for Endpoint security baseline" → Create policy. Name it, review the settings, customize what does not fit, assign, create. Pilot on a small group before broad deployment — baselines can break line-of-business apps. Afterwards, watch profile status, device status, and conflicts. Intune reports a conflict when two policies set the same setting to different values, so avoid configuring the same setting in multiple places unless you are doing it deliberately and monitoring it.

Baselines are a starting point, not a replacement for endpoint security policies, which are purpose-built profiles for specific scenarios:

- Antivirus (Endpoint security → Antivirus): real-time protection, cloud-delivered protection, sample submission, scan behavior, exclusions. Exclusions reduce what gets scanned — keep them minimal, broad exclusions weaken protection.
- Tamper Protection: blocks changes to Defender security settings by users, scripts, malware, or any unauthorized process. Prevents an attacker who is already on the box from turning protection off.
- Attack surface reduction (Endpoint security → Attack surface reduction): ASR rules. Prerequisites: Windows devices, and Microsoft Defender Antivirus must be the primary antivirus. Roll out as audit mode → review results → add exclusions only where needed → switch selected rules to block mode.
- Firewall (Endpoint security → Firewall): firewall state, rules, network protection behavior for Windows and macOS. Keep unrelated settings out of the same profile.
- EDR (Endpoint security → Endpoint detection and response): configures EDR settings and is also the vehicle for Windows onboarding (section 3).

## 5. EDR policies in detail

The onboarding package contains tenant-specific configuration, service endpoints, authentication material, and initial communication settings. It enables telemetry flow and nothing more.

Windows EDR policy settings:

- Client configuration package type: onboarding or offboarding; "Auto from connector" or manual package (see section 3).
- Sample sharing: whether files can be sent to Microsoft for deeper analysis.
- Telemetry Reporting Frequency: deprecated. Visible only for old policy compatibility; has no effect on new devices.

For macOS and Linux, EDR policies can set device tags, which are used for organizing and filtering in the Defender portal. Linux additionally has a separate "Microsoft Defender Global Exclusions (AV+EDR)" profile.

Creation path: Endpoint security → Endpoint detection and response → Create policy → platform + EDR profile → name → settings (Auto from connector if the connection is active) → sample sharing → assign to a device group → create.

EDR in block mode is configured in the Defender portal, not Intune: Settings > Endpoints > General > Advanced features → "Enable EDR in block mode". It exists for the case where Microsoft Defender Antivirus is not the primary antivirus and sits in passive mode — EDR can then still remediate malicious artifacts the third-party AV missed. Requires Defender for Endpoint Plan 2 and supported Windows devices.

## 6. Investigating and responding to threats

An alert is a single detection of suspicious or malicious activity, from Defender or other Microsoft security services. An incident is a set of correlated alerts representing one attack story. An incident view shows: the triggering alerts; involved entities (devices, users, files, processes, services, IPs); a timeline; evidence and response status; automated investigation details; and remediation actions, done or recommended.

Triage starts with four questions: What happened? Which devices and users are affected? Is the threat still active? What is the business impact?

The alerts queue can be filtered by severity, status, category, service/detection source, tags, product name, impacted entities, and automated investigation state — use filters to prioritize.

Deep investigation usually involves: suspicious processes and their parent-child relationships, command lines, malicious files, registry/file-system changes, network connections to bad IPs or domains, users signed in to affected devices, and other devices or users touched by the same indicators. The attack story and incident graph show relationships between alerts, entities, and assets — that is how you spot attacker movement or confirm an alert is isolated. The device page gives device details, related alerts, signed-in users, timeline, exposure information, and response actions.

Manual response actions from the device page: initiate automated investigation, start a Live Response session, collect an investigation package, run an antivirus scan, restrict app execution, isolate the device, contain the device, consult a threat expert, open the Action center.

A workable response sequence: confirm the threat is real; identify affected devices, users, files, and network indicators; isolate or contain devices while the threat is active; stop or quarantine malicious files and processes; run scans or collect packages for more evidence; hunt the same indicators elsewhere; fix the root cause (vulnerable software, weak configuration, compromised credentials); update incident status, classification, and notes.

AIR assigns one of three verdicts — malicious, suspicious, or no threats found — and can remediate automatically (quarantine file, stop process, isolate device, block URL) or queue actions for approval in the Action center. Automate the routine work, but review high-impact actions manually when privileged accounts, sensitive users, production devices, or lateral movement are involved.

Security Copilot assists — it summarizes incidents, explains alerts, identifies affected users and devices, generates investigation questions, helps with queries, and drafts recommendations. It is an enhancement, not the system of record: Defender stays the primary tool, and Copilot findings must be validated in Defender before high-impact actions such as isolating devices, blocking files, or resetting privileged accounts. Promptbooks are saved, reusable Copilot workflows for structured tasks like incident review, script analysis, CVE impact assessment, or threat actor research.

## 7. Monitoring and triaging incidents

The incident queue (Investigation & response → Incidents & alerts → Incidents) lists incidents across devices, users, mailboxes, and other resources, with severity, status, owner, impacted assets, categories, detection sources, and timestamps. Filter and sort to prioritize: critical devices, privileged users, active malware, signs of lateral movement.

Triage extends the four questions from section 6 with a fifth — what next? Decide: investigate further, escalate, remediate, or close.

Incident management: assign an owner, add tags, update status, add comments, and classify the incident after review.

Escalate when the incident involves a privileged account, a critical server, suspected data theft, legal or compliance impact, or looks like part of a broader attack. The escalation workflow: assign to the right analyst or team; tag it (High impact, Privileged user, Ransomware, Needs escalation); write comments summarizing triage findings; set status to in progress; notify the security, identity, endpoint, or infrastructure team; track remediation until resolved. The bar for a good escalation: the next responder can act immediately without repeating your investigation.

## 8. Knowledge check — answers and reasoning

1. Windows 11 devices, already enrolled in Intune, service connection enabled, least administrative effort → Create an Endpoint detection and response policy for Windows with the package type set to Auto from connector. The EDR policy is the recommended onboarding method and Auto from connector retrieves the package with no manual steps. Manually downloading and deploying the package is more work; an antivirus policy does not onboard anything.
2. Two policies set the same setting differently → Intune admin center policy status pages, which report conflicts. The Defender portal configures tenant features, not Intune conflicts; the local Windows Security app knows nothing about Intune policies.
3. Best artifact for triage on the incident page → correlated alerts, affected assets, evidence, and the attack story view. The other options (AV exclusions list, EDR Onboarding Status tab) are configuration surfaces, not investigation surfaces.
4. ASR rules in audit mode → workload: Endpoint security > Attack surface reduction; prerequisite: Defender Antivirus as the primary antivirus on Windows. The distractors pair the wrong workload with the wrong prerequisite (firewall + third-party AV; admin templates + on-prem AD).
5. High-severity incident, privileged account, lateral movement → assign an owner, tag (Privileged user, Needs escalation), summarize findings in comments, notify the response team. This is the escalation workflow from section 7. Closing as informational ignores the risk; deleting accounts and reimaging without triage destroys evidence and skips root-cause analysis.

## 9. Click-path reference

| Task | Path |
|---|---|
| Enable Intune↔Defender connection | Defender portal: System > Settings > Endpoints > General > Advanced features |
| Windows onboarding (preconfigured) | Intune: Endpoint security > Endpoint detection and response > EDR Onboarding Status tab |
| Security baseline | Intune: Endpoint security > Security baselines |
| Antivirus settings | Intune: Endpoint security > Antivirus |
| ASR rules | Intune: Endpoint security > Attack surface reduction |
| Firewall | Intune: Endpoint security > Firewall |
| EDR in block mode | Defender portal: Settings > Endpoints > General > Advanced features |
| macOS onboarding package | Defender portal: Settings > Endpoints > Device management > Onboarding → macOS |
| Incident queue | Defender portal: Investigation & response > Incidents & alerts > Incidents |

## 10. Self-check

These are the claims I should be able to explain without notes:

- The three-component architecture and the telemetry flow from sensor through cloud analysis to risk signal.
- How Defender risk signals, Intune compliance policies, and Conditional Access chain together into risk-based access control.
- All onboarding prerequisites, and why "Auto from connector" is preferred over a manual package.
- What onboarding does and does not configure, and which settings must ship as separate policies.
- Why two policy types managing onboarding settings cause conflicts, and where conflicts are reported.
- The macOS profile set beyond onboarding, and why mobile platforms use MTD instead of the EDR onboarding policy.
- ASR prerequisites and the audit → block rollout pattern.
- EDR in block mode: what problem it solves, its Plan 2 requirement, where it is enabled.
- Alert vs. incident; the triage questions; the escalation workflow; the three AIR verdicts.
- Where Copilot helps, and why it does not replace validation in Defender.

## 11. What is next

Modules 2–6 of the learning path are not covered here: device encryption, advanced threat protection scenarios, compliance and remediation, Microsoft Tunnel, and Cloud PKI. Hands-on practice worth doing: trial tenant, connect Intune to MDE, run an EDR onboarding policy on a test VM, then try advanced hunting and the incident graph.
