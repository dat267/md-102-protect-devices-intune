# Secure Mobile Access with Microsoft Tunnel

Source: Microsoft Learn module "Secure mobile access using Microsoft Tunnel", module 5 of 6 in the "Protect devices using Microsoft Intune" learning path (MD-102 material).

Module completed 2026-09-22.
Achievement: https://learn.microsoft.com/api/achievements/share/en-us/DatDo-7483/ZJNQ3L52?sharingId=15FA99AA28223E58

## Main takeaways

- Microsoft Tunnel is a containerized VPN gateway on Linux, configured and profiled from Intune, giving on-demand access to on-premises resources for enrolled and unenrolled mobile devices. Entra Private Access is the cloud-native ZTNA alternative when no Tunnel infrastructure exists.
- Deploy in order: server configuration, then site, then the Linux install (mstunnel-setup, Podman on RHEL and Docker elsewhere), then the Microsoft Defender client app, then a platform-specific VPN profile. The TLS certificate must include the device-facing FQDN or IP address in the Subject Alternative Name.
- Use the APIPA range 169.254.0.0/16 for client IPs, port 443 TCP and UDP/DTLS, at most 500 split-tunneling rules, and never 0.0.0.0 in a split-tunnel rule.
- Tunnel for MAM extends the same gateway to unenrolled BYOD and requires Intune Plan 2 or the Intune Suite. Android 10+ supports per-app or device-wide VPN through the Defender app; iOS 17+ supports per-app VPN only through the Tunnel for MAM SDK, with three policy types against Android's three policies.
- Operate with the Health status dashboard and Trends (averaged in three-hour blocks), mst-cli for status and restarts, and journalctl tags for logs. Access logging is off by default, and verbose log collection runs at level 4 for eight hours and returns an Incident ID for Support.
- The four documented failures are the /dev/net/tun error after a host reboot, Health status offline while devices still connect (reinstall to re-enroll the agent), the Podman checkup error, and certificate expiry.

## 1. What Microsoft Tunnel is

A cloud-managed VPN gateway for reaching on-premises resources from mobile devices. Tunnel Gateway runs as a containerized service on a Linux server; Intune holds the configuration and distributes VPN profiles. Devices get on-demand access to internal resources rather than a traditional always-on VPN, which means less battery drain and less app performance impact.

Two audiences: enrolled Android Enterprise / iOS-iPadOS devices (standard MDM VPN profiles) and unenrolled BYOD devices (Tunnel for MAM).

Alternative to consider: **Microsoft Entra Private Access** (part of Global Secure Access) is the cloud-native ZTNA option. Choose Tunnel when you need Intune-managed VPN profiles, per-app or device-wide VPN behavior, Tunnel for MAM on unenrolled BYOD, or you already run Tunnel. Choose Private Access for new deployments with no existing Tunnel infrastructure, or when you want identity-based private app access without a mobile VPN profile.

## 2. Gateway deployment

Three Intune objects, created in order:

**Server configuration** (Tenant administration → Microsoft Tunnel Gateway → Server configurations): the shared network settings for every server in a site. Settings: IP address range leased to clients (**APIPA 169.254.0.0/16** avoids conflicts with corporate subnets), server port (default **443 TCP and UDP/DTLS**), DNS servers, optional DNS suffix search, and "Disable UDP connections" (only when devices use the Microsoft Defender Tunnel client and TCP-only is required). Split-tunneling rules: include ranges route through the tunnel, exclude ranges go direct; max **500 rules total** across both lists. Never use `0.0.0.0` in a split-tunnel rule — the gateway can't route it. If you use corporate DNS, add those addresses as include rules.

**Site** (… → Sites): a logical group of servers sharing one connection point and one server configuration. Settings: public IP or FQDN that devices connect to (must be publicly resolvable; can be a single server or a load balancer), the server configuration, an internal HTTP/HTTPS URL that each server pings **every five minutes** to verify it can reach the corporate network, optional automatic server upgrades, and an optional maintenance window to limit when upgrades start.

**Server install** on Linux: download `mstunnel-setup` (from https://aka.ms/microsofttunneldownload or `wget`), run as root (`sudo ./mstunnel-setup`; for rootless Podman use `mst_rootless_mode=1 ./mstunnel-setup`), accept EULA, edit `/etc/mstunnel/env.sh` (including proxy settings), copy the TLS cert, authenticate via device code at https://microsoft.com/devicelogin with an Intune Administrator account, then select the site to join. Verify the server shows online in Health status.

Prerequisites to confirm before installing (always check the current prerequisites page — supported distros and versions change): supported Linux distribution; sizing (small deployments start at 4 CPU / 4 GB RAM / 30 GB disk, scaled up by device and site count); outbound access to Microsoft endpoints including TCP 443 and, per current guidance, TCP 8090; container engine — **Podman for RHEL, Docker for others**, rootless Podman supported.

TLS certificate requirement: it must include the server's IP address or FQDN in the **Subject Alternative Name (SAN)**. Format: PFX to `/etc/mstunnel/private/site.pfx`, or PEM as `fullchain.crt` in `/etc/mstunnel/certs/` plus `private.key` in `/etc/mstunnel/private/`.

Client and profiles: deploy the **Microsoft Defender app** from Google Play (Android) or the App Store (iOS) via Intune to the same groups that get the VPN profile. VPN profiles are platform-specific (Devices → Configuration → Create). Android Enterprise: profile type Templates → VPN, Corporate-Owned or Personally Owned Work Profile, connection type Microsoft Tunnel, with base VPN (connection name, tunnel site, per-app VPN, always-on VPN, proxy). iOS/iPadOS: Templates → VPN, connection type Microsoft Tunnel, with connection name, site, per-app VPN, on-demand VPN rules, proxy. Note: on iOS, enabling per-app VPN causes split-tunneling rules to be ignored. If iOS devices use both Tunnel and Defender web protection, add an on-demand rule with Connect VPN and the target domains so "Disconnect on Sleep" behaves.

## 3. Tunnel for MAM (unenrolled BYOD)

Extends the same gateway to unenrolled personal Android/iOS devices. Users authenticate with Entra ID, Conditional Access applies, and personal data stays separate — corporate data is protected by app protection policies, and the VPN is scoped to MAM-enabled apps rather than all device traffic. No separate infrastructure: existing server configurations and sites serve enrolled and unenrolled devices at once.

Licensing: requires **Intune Plan 2 or the Intune Suite add-on**.

Platform support: Android Enterprise **Android 10+**, per-app or device-wide VPN, delivered through the Microsoft Defender app. iOS/iPadOS **iOS 17+**, **per-app VPN only** (no device-wide option), delivered through the Tunnel for MAM iOS SDK integrated into LOB apps. Intune supports "current + 2" iOS versions.

**Android requires three policies** on the same user groups:

1. App configuration policy for Microsoft Defender (Apps → Configuration → Create → Managed Apps; app: Microsoft Defender Endpoint): under Microsoft Tunnel for MAM settings, set Use Microsoft Tunnel for MAM = Yes, give a base VPN connection name, select the site, optionally per-app VPN and a root certificate for private-CA resources. Include Microsoft Edge in the per-app VPN list for identity switching and Tunnel notifications. MAM Tunnel on Android does **not** support always-on VPN.
2. App configuration policy for Microsoft Edge (managed app config, General configuration settings): `com.microsoft.intune.mam.managedbrowser.StrictTunnelMode = True` (blocks internet if VPN is down under a work account) and `com.microsoft.intune.mam.managedbrowser.TunnelAvailable.IntuneMAMOnly = True` (enables identity-switch behavior).
3. App protection policy for Edge (Apps → Protection → Create → Android): on Data protection, set "Start Microsoft Tunnel connection on app-launch" = Yes. This is what makes the VPN start when Edge launches.

**iOS requires three policy types:**

1. App configuration policy (managed apps; LOB app by bundle ID or Edge as public app) with Microsoft Tunnel for MAM settings: use MAM = Yes, connection name, site, root certificate if private CA, proxy if needed. For federated Entra tenants, add general config key `com.microsoft.tunnel.custom_configuration` with a value containing the federation STS URL as a bypassed URL, e.g. `{"bypassedUrls":["sts.contoso.com"]}`.
2. App protection policy (Apps → Protection → Create → iOS/iPadOS) with data protection, access requirements, conditional launch settings — also required for app configuration to reach apps.
3. Trusted certificate profile, only if apps reach resources behind a private/on-prem CA. The chain of trust lets the device verify the server certificate. Android uses the Intune App SDK `MAMTrustedRootCertsManager` API; iOS uses the Tunnel for MAM SDK with DER-encoded binary X.509 or PEM. A trusted certificate profile from any platform (Android, iOS, Windows) works for iOS Tunnel for MAM — no iOS-specific profile needed.

## 4. Monitoring and troubleshooting

Admin center: Tenant administration → Microsoft Tunnel Gateway → **Health status** for the server dashboard; a server's **Health check** tab for detail (Warning/Unhealthy metrics highlighted, thresholds configurable for CPU, memory, disk, latency). The **Trends** tab charts connections, CPU, disk, memory, latency, throughput — averaged in three-hour blocks, so trends can lag up to three hours.

`mst-cli` (installed at `/usr/sbin/mst-cli`, run as root/sudo):

- `mst-cli server status`, `mst-cli server restart`, `mst-cli agent restart`, `mst-cli server show` (active connections)
- `mst-cli import_cert` then `mst-cli server restart` after replacing a certificate; `mst-cli import_cert delay <minutes>` stages the change in a maintenance window
- `mst-cli uninstall` (used in the offline-server fix)

Logs: written to the system journal, filtered by tag — `journalctl -t ocserv` (server), `-t mstunnel-agent`, `-t mstunnel_monitor`; `journalctl -t ocserv | grep TELEMETRY` for connection telemetry; `-f` across tags for live tailing. Access logging is **off by default**; enable with `TRACE_SESSIONS=1` in `/etc/mstunnel/env.sh` (2 to include DNS), then restart the server — but it can hurt performance on busy servers, so use it only during active troubleshooting.

Verbose logs for Microsoft Support: admin center → Microsoft Tunnel Gateway → select server → Logs tab → Send logs. Intune uploads standard logs, enables verbosity level 4 for **eight hours** so you can reproduce, uploads a second verbose set, then resets to level 0. Each upload produces an **Incident ID** to give Support.

Common failures and fixes:

- Device can't connect; log shows `Can't open /dev/net/tun: Operation not permitted` after a host reboot → `sudo mst-cli server restart` (automate with cron if reboots are frequent).
- Health status shows offline but devices connect fine → known issue where the agent loses its Intune enrollment registration while the server keeps working. Fix: `sudo mst-cli uninstall` then `sudo ./mstunnel-setup` to re-enroll. Prevention: install agent and server updates promptly.
- Podman "Error executing checkup" in the monitor log → Podman can't identify running containers; `podman restart [container-name]`, or cron it if recurring.
- TLS certificate expiring (metric shows Warning) → copy the new cert, run `mst-cli import_cert`, then `mst-cli server restart` (optionally with `delay`).

## 5. Knowledge check — answers and reasoning

1. TLS certificate for a server reached at `tunnel.contoso.com` → **The Subject Alternative Name must include the IP address or FQDN devices use to reach the server.** That's the explicit naming requirement. Certificates are not self-signed by mst-cli and are not issued by a Microsoft Intune root CA.

2. Unenrolled Android devices using Tunnel through Edge — prerequisite → **An Intune Plan 2 license (or Intune Suite add-on) plus app configuration and app protection policies on Edge that enable Tunnel on app launch.** Tunnel for MAM is the whole point of *not* enrolling, so enrollment is the wrong prerequisite; SSH has nothing to do with it.

3. iOS Tunnel for MAM VPN scope → **Per-app VPN only, scoped to the LOB app and other SDK-integrated apps such as Edge.** Device-wide VPN and always-on at boot are Android-only / unsupported on iOS.

4. Server shows offline in Health status but devices still connect → **Reinstall Microsoft Tunnel (`sudo mst-cli uninstall` then `sudo ./mstunnel-setup`) so the agent re-registers with Intune.** This is the documented known issue: the agent lost its enrollment registration while the server component still works. A plain restart doesn't re-register; deleting the site is destructive and unnecessary.

5. Firewall blocks UDP → **In the server configuration, enable "Disable UDP connections" (supported with the Microsoft Defender Tunnel client).** Disabling the agent breaks the server; `0.0.0.0/0` is explicitly invalid in split-tunnel rules and forcing everything through the tunnel wouldn't disable UDP anyway.

## 6. Self-check

- I can order the deployment: server configuration → site → Linux install → client app → VPN profile, and say what belongs in each.
- I know why the client IP range should be APIPA and why `0.0.0.0` is banned from split-tunnel rules.
- I can list the Linux prerequisites and name the cert SAN requirement in one sentence.
- I can explain Tunnel for MAM's licensing, platform support matrix, and per-platform VPN scope differences.
- I can name the three Android policies and the three iOS policy types, and say what each contributes.
- I can diagnose the four documented failure modes by symptom without looking them up.
- I can explain why Tunnel might be the wrong choice versus Entra Private Access.
