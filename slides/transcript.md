# 15-Minute Presentation Transcript

**Topic:** Securing devices with Microsoft Intune
**Source:** three modules from the *Protect devices using Microsoft Intune* learning path

Speaking pace assumed: **100 words per minute** (deliberate, slow delivery). Duration shown is talk time only, excluding pauses and audience interaction. Target total: about 13 minutes of speech, leaving buffer inside a 15-minute slot.

| # | Slide | Words | Talk time |
|---|-------|-------|-----------|
| 1 | Cover - Securing devices with Microsoft Intune | 61 | 0:37 |
| 2 | Agenda - Three building blocks | 42 | 0:25 |
| 3 | Why Zero Trust for devices | 58 | 0:35 |
| 4 | Pillar one divider - Compliance and remediation | 14 | 0:08 |
| 5 | Profiles enforce, compliance verifies | 59 | 0:35 |
| 6 | Risk-based evaluation | 83 | 0:50 |
| 7 | Interactive demo - Compliance evaluator | 59 | 0:35 |
| 8 | Scoping with groups and filters | 77 | 0:46 |
| 9 | Actions for noncompliance - graduated timeline | 79 | 0:47 |
| 10 | Remediation - self-healing devices | 55 | 0:33 |
| 11 | Monitoring and reporting | 47 | 0:28 |
| 12 | Pillar two divider - Microsoft Tunnel | 16 | 0:10 |
| 13 | Tunnel - secure access without always-on VPN | 90 | 0:54 |
| 14 | Tunnel building blocks | 64 | 0:38 |
| 15 | Interactive demo - Split-tunneling router | 67 | 0:40 |
| 16 | Tunnel for MAM and operations | 62 | 0:37 |
| 17 | Pillar three divider - Microsoft Cloud PKI | 11 | 0:07 |
| 18 | Cloud PKI - retiring the old PKI stack | 82 | 0:49 |
| 19 | Certificate lifecycle and renewal | 88 | 0:53 |
| 20 | Interactive demo - Certificate renewal simulator | 78 | 0:47 |
| 21 | Certificate health and governance | 69 | 0:41 |
| 22 | Putting it together | 41 | 0:25 |
| 23 | Key takeaways | 48 | 0:29 |
| 24 | Next steps and Q&A | 43 | 0:26 |
| | **Total** | **1393** | **13:56** |

---

## Slide 1 - Cover - Securing devices with Microsoft Intune (61 words, about 0:37)

Good morning, and thank you for the time.

In fifteen minutes I want to give you a mental model for protecting devices with Microsoft Intune.

Three ideas.

Proving a device is healthy.

Giving mobile devices secure access to on-premises resources.

And giving every device a trustworthy identity.

This stays conceptual, so you leave with a model, not a list of clicks.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 2 - Agenda - Three building blocks (42 words, about 0:25)

Here's our route.

Compliance and remediation, the foundation.

Microsoft Tunnel, for secure mobile access.

Microsoft Cloud PKI, for certificate automation.

One thread runs through all three.

A device is not trusted because it's ours.

It has to keep proving it deserves access.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 3 - Why Zero Trust for devices (58 words, about 0:35)

One distinction first, because it shapes everything else.

Intune is the evaluator.

It inspects the device and reports its state.

But Intune doesn't block anything by itself.

Microsoft Entra Conditional Access is the enforcer.

It reads Intune's verdict and decides to grant or deny.

Intune answers, is this device okay?

Conditional Access answers, should we let it in?


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 4 - Pillar one divider - Compliance and remediation (14 words, about 0:08)

Let's start with pillar one.

Compliance and remediation.

This decides what healthy actually means.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 5 - Profiles enforce, compliance verifies (59 words, about 0:35)

Here's the core distinction.

Configuration profiles enforce.

They do the work: turn on BitLocker, set the firewall, push settings.

Compliance policies verify.

They check the work: is BitLocker really active, is the OS current, is antivirus running.

A profile makes a promise.

A compliance policy confirms it's true.

Use both.

A setting you never verify is just a promise.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 6 - Risk-based evaluation (83 words, about 0:50)

Static checks are necessary, but not enough.

Picture a device with a strong password, the latest OS, every setting in place.

It looks perfect.

But the user just opened a malicious document, and malware is copying data right now.

Static checks never notice.

So we add a dynamic layer.

Defender for Endpoint runs continuously and gives a risk score: clear, low, medium, high.

Intune reads it.

You set the bar.

If risk goes high, the device is noncompliant immediately, no wait for check-in.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 7 - Interactive demo - Compliance evaluator (59 words, about 0:35)

Let me show you.

Encryption on, OS current, antivirus running, risk clear.

Compliant, and access is granted.

Now I turn off antivirus.

Instantly noncompliant, access blocked.

Let me try another.

Everything back on, but risk set to medium.

Every static check passes, yet the device still fails, because risk is above our bar.

That's the two layers working together.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 8 - Scoping with groups and filters (77 words, about 0:46)

Now, who gets the policy?

The foundation is Entra groups.

Target users, which follows the person across devices, or target devices, which stays with the hardware, like a kiosk.

Include and exclude groups for phased rollouts.

Then dynamic groups, where membership follows attributes like department and updates itself.

Finally, assignment filters give precision without a group for every variation, for example only corporate-owned iPhones.

The aim is to avoid group bloat and still hit the right devices.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 9 - Actions for noncompliance - graduated timeline (79 words, about 0:47)

When a device fails, we don't block brutally on day one.

We escalate gradually.

Day zero: marked noncompliant, and Conditional Access blocks corporate resources.

Day zero or one: an email explains what failed and how to fix it, optionally copying the manager or helpdesk.

Day one: a push notification in the Company Portal app.

And only after a long period, around thirty days, we retire the device and wipe corporate data.

Users get every chance to fix things first.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 10 - Remediation - self-healing devices (55 words, about 0:33)

The best outcome is that users never act at all.

That's remediation.

Two tools.

Declarative fixes: configuration profiles re-apply settings and silently correct drift, like re-enabling a firewall.

Custom fixes: Intune Remediations, PowerShell scripts that detect a problem and fix it.

Target them precisely.

Devices heal in the background, so tickets and disruption both drop.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 11 - Monitoring and reporting (47 words, about 0:28)

Then we need visibility.

Dashboards show compliance at a glance.

Reports and exports serve two audiences: operations troubleshooting the failing devices, and compliance teams needing audit evidence for frameworks like HIPAA or ISO 27001.

You cannot prove what you haven't measured.

Reporting turns device health into evidence.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 12 - Pillar two divider - Microsoft Tunnel (16 words, about 0:10)

Pillar two.

Microsoft Tunnel.

The devices are healthy.

Now they need to reach on-premises resources, securely.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 13 - Tunnel - secure access without always-on VPN (90 words, about 0:54)

Here's the problem.

Mobile users need internal apps.

The traditional answer is an always-on VPN, which drains battery, slows apps, and grants broad network access.

Microsoft Tunnel is different.

It's a cloud-managed VPN gateway with secure, on-demand access.

The tunnel opens when needed, not all the time.

It works for enrolled Android and iOS devices, and through mobile application management, for unenrolled BYOD.

One note: if you're starting fresh with no Tunnel infrastructure, compare Microsoft Entra Private Access.

Tunnel fits when you need Intune-managed VPN profiles, or already run it.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 14 - Tunnel building blocks (64 words, about 0:38)

Four building blocks.

Install the gateway on a Linux server.

Create a server configuration and site in Intune.

Use TLS certificates to secure the gateway and authenticate the client.

Deploy the client app and a VPN profile for Android or iOS.

Split tunneling and DNS decide which traffic travels through the tunnel.

And notice certificates appear here already, a natural bridge to pillar three.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 15 - Interactive demo - Split-tunneling router (67 words, about 0:40)

Here's the routing.

Tunnel connected, split tunneling on.

The internal wiki and HR app travel through the tunnel, because they're corporate.

Bing and YouTube go direct, straight to the internet, so they don't waste gateway bandwidth.

Now I turn split tunneling off.

Everything rides the tunnel, including public sites, a full tunnel.

If I disconnect, internal resources are blocked while public sites still work.

That's the balance.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 16 - Tunnel for MAM and operations (62 words, about 0:37)

Two operations points.

For BYOD, Tunnel for MAM is a three-policy setup: configure Defender, configure Edge, and apply an app protection policy.

Unenrolled users get access without enrolling the whole device.

For health, we watch the dashboard: metrics, trends, certificate expiry.

We use mst-cli for deeper diagnostics, and journalctl for logs.

Proactive monitoring catches a certificate expiry before it becomes an outage.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 17 - Pillar three divider - Microsoft Cloud PKI (11 words, about 0:07)

Pillar three.

Microsoft Cloud PKI.

Giving every device a trustworthy identity.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 18 - Cloud PKI - retiring the old PKI stack (82 words, about 0:49)

Historically this meant on-premises infrastructure: an NDES server, a certificate connector, and a CA to patch, monitor, and repair.

Cloud PKI removes all of it.

We create the certification authorities right in the Intune admin center.

No servers, no connectors.

Each issuing CA has a built-in SCEP service.

And we use a two-tier hierarchy, a root CA and an issuing CA.

The root stays protected while the issuing CA does daily work.

It's available in the Intune Suite or as an add-on.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 19 - Certificate lifecycle and renewal (88 words, about 0:53)

Enrollment is almost entirely automatic.

The device generates its private key locally, and it never leaves the device.

It sends a signing request to Cloud PKI.

Cloud PKI signs and returns the certificate.

It renews automatically before expiry.

We configure the SCEP profile once; devices do the rest.

Renewal timing is controlled by the renewal threshold.

Practical tip: for remote devices that check in rarely, use a higher threshold, thirty to forty percent, so they renew in time.

And deploy the trusted certificate profile before the SCEP profile.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 20 - Interactive demo - Certificate renewal simulator (78 words, about 0:47)

Let me make this concrete.

Validity of one hundred eighty days, threshold thirty percent.

The shaded area is the renewal window, starting in the last fifty-four days.

Now imagine a laptop that connects once a month.

I raise the threshold to forty percent.

The window grows and renewal starts earlier, giving more chances to renew.

And this button revokes the certificate.

It moves to revoked, rejected through the revocation list.

The whole life is visible in one place.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 21 - Certificate health and governance (69 words, about 0:41)

We also need certificate health.

The Cloud PKI dashboard, and certificates under Devices and Monitor, show every state: active, expired, revoked.

Reports refresh every twenty-four hours, and the device monitor has no record limit.

Audit logs record every administrative action, creating a CA, revoking a certificate, changing properties.

That supports accountability and compliance.

Build alerts so failures surface before users hit them.

Revocation only matters if it's actually enforced.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 22 - Putting it together (41 words, about 0:25)

Put the three together.

Cloud PKI gives each device a verifiable identity.

Tunnel uses that identity for secure on-demand access.

Compliance, with Conditional Access, decides whether the device is trusted enough to use either.

Identity, access, trust, all reinforcing each other.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 23 - Key takeaways (48 words, about 0:29)

Five lines to remember.

One: profiles enforce, compliance verifies, use both.

Two: layer Defender risk on top of static checks.

Three: scope with groups and filters, and escalate gradually.

Four: remediate automatically so devices heal before users are blocked.

Five: Tunnel delivers on-demand access, Cloud PKI automates identity.


> Delivery cue: pause at each full stop. Let the point land before moving on.

## Slide 24 - Next steps and Q&A (43 words, about 0:26)

That's the tour.

The learning path has a full module on each pillar with hands-on detail.

A good next step: add a Conditional Access policy that requires device compliance, and watch enforcement work end to end.

Thank you.

I'm happy to take questions.


> Delivery cue: pause at each full stop. Let the point land before moving on.


---


## Pacing plan (staying inside 15 minutes)

The script totals about **13:56 of talk time at 100 words per minute**. Slow, deliberate delivery is assumed. Real pacing adds breathing room and demo interaction:

- Natural pauses and slide transitions add roughly 30-60 seconds overall.
- Clicking through each interactive demo adds 20-40 seconds.
- So plan for about 15 minutes with the current script at 100 wpm.

**If you speak slower than 100 wpm (85-90 wpm), the script runs 15:30 to 16:20. Cut in this order:**

1. Slide 11 (Monitoring): keep only "Dashboards and reports give us visibility and audit evidence." Saves about 20 seconds.
2. Slide 16 (Tunnel operations): drop the `mst-cli` and `journalctl` sentence. Saves about 12 seconds.
3. Demos: run one example instead of two or three. Saves about 20 seconds per demo, up to a minute total.
4. Slide 3: stop after "It reads Intune's verdict and decides to grant or deny." Saves about 15 seconds.

**If you finish early, expand rather than rush:**

- In each demo, narrate one extra scenario (for example, set risk to High on slide 7).
- On slide 22 (Putting it together), add one concrete example from your own environment.
- On slide 24, invite a specific question, for example "What would break first if we turned this on tomorrow?"

## Rebuilding the deck from this transcript

Each section above maps one-to-one to a slide in `intune-device-security.html`. The deck no longer carries a separate speaker-notes panel. Instead, each slide shows a short **Terms** footer that explains the abbreviations and jargon used on that slide, for example NDES, SCEP, CA, CSR, EKU, MAM, VPN, and CRL.

If you edit a script section here, treat this file as the source of truth for what you say. The on-slide Terms footers are independent and can be edited directly in the HTML.
