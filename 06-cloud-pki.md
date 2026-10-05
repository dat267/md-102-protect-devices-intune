# Cloud PKI with Microsoft Intune

Source: Microsoft Learn module "Implement Microsoft Cloud PKI", module 6 of 6 in the "Protect devices using Microsoft Intune" learning path (MD-102 material).

Module completed 2026-09-29.
Achievement: https://learn.microsoft.com/api/achievements/share/en-us/DatDo-7483/K9VXKW7B?sharingId=15FA99AA28223E58
Learning path complete: https://learn.microsoft.com/api/achievements/share/en-us/DatDo-7483/VSY3VCWM?sharingId=15FA99AA28223E58

## Main takeaways

- Microsoft Cloud PKI is a cloud-hosted PKI inside the Intune admin center. It removes the on-premises NDES server, the Intune certificate connector, and the CA infrastructure that needs patching and monitoring; each issuing CA ships with its own built-in SCEP service.
- Licensing is a prerequisite: it is included with the Intune Suite or bought as a standalone per-user add-on, and only six CAs are allowed per tenant counting root, issuing, and bring-your-own CAs together. Trial-created CAs use software-backed keys that can never be converted to HSM-backed.
- Deployment order is mandatory: root CA, then issuing CA, then the trusted certificate profile for the root, then the trusted certificate profile for the issuing CA, then the SCEP profile. SCEP before trust makes devices reject the issued certificates.
- SCEP keeps the private key on the device: the device generates the CSR locally, the encrypted challenge is validated by the SCEP registration authority, and the issuing CA signs and returns the certificate. Everything after profile assignment is automatic.
- Validity period and renewal threshold drive the lifecycle. The default 20% renewal threshold on a one-year certificate starts renewal about 73 days out; raise it to 30 to 40% for remote or occasionally connected devices, and shorten validity when exposure must be limited.
- Revocation feeds a CRL with a 7-day validity refreshed every 3.5 days, and manual revocation updates it immediately, though client-side CRL caching means high-impact revocations should be paired with Conditional Access.
- Monitor from the Cloud PKI dashboard (updated every 24 hours, and only the first 1,000 certificates per CA) and from Devices > Monitor > Certificates for the full cross-CA view, and use the audit log for accountability.

## 1. The problem Cloud PKI solves

Certificate-based authentication for Wi-Fi, VPN, and email traditionally means running an on-premises Network Device Enrollment Service (NDES) server, an Intune certificate connector, and a CA hierarchy that must be patched, monitored, and recovered when it fails. Cloud PKI replaces that: CAs are created directly in the Intune admin center, devices enroll automatically over SCEP, and issuance through renewal runs without administrator intervention.

Typical driver: an on-premises PKI reaching end of life, or a fleet that needs passwordless network authentication at scale. A related path forward is linking SCEP profiles to Wi-Fi and VPN profiles once certificates are issuing.

## 2. Licensing, limits, and permissions

Cloud PKI is a premium add-on. The tenant needs either the **Microsoft Intune Suite** (includes Cloud PKI) or the standalone **Microsoft Cloud PKI add-on** (per user). A free trial of either can be activated from the Intune admin center.

Two constraints worth memorizing before designing a hierarchy:

- **Maximum six CAs per Intune tenant**, counting root CAs, issuing CAs, and BYOCA together.
- CAs created during a trial use **software-backed signing and encryption keys**, and after purchase those keys stay software-backed. They cannot be converted to HSM-backed keys, so plan the pilot knowing that a trial hierarchy is not production-grade key protection.

RBAC permissions for the Intune role managing Cloud PKI:

- **Read CAs** — view CA properties in the admin center.
- **Create certificate authorities (CAs)** — create root and issuing CAs.
- **Revoke issued leaf certificates** — manually revoke a certificate issued by an issuing CA (also requires Read CAs).
- **Disable and reenable CAs** — disable or reenable a CA.

These permissions exist only in Intune; in Entra ID the built-in **Intune Administrator** role already carries them.

## 3. Create the CA hierarchy

Cloud PKI uses a **two-tier hierarchy**: a root CA that anchors trust and an issuing CA beneath it that signs device certificates. Both are created under Tenant administration > Cloud PKI.

**Root CA** (the trust anchor for the whole PKI):

- CA type: Root CA
- Validity period: **25 years**
- Extended Key Usages: select what the design needs
- Subject attributes: Common Name, Organization, Country/Region, State
- Encryption: **RSA-4096 with SHA-512** (recommended for root CAs)
- Scope tags as desired

Intune generates the CA keys automatically in **Azure Managed Hardware Security Module (Azure Managed HSM)**, so there is no separate key management step.

**Issuing CA** (signs device certificates, and includes a built-in SCEP service):

- CA type: Issuing CA
- Root CA source: **Intune** (or choose the existing external root for BYOCA)
- Root CA: the root just created
- Validity period: **5 years** (recommended)
- Extended Key Usages and subject attributes as needed
- Encryption is inherited from the root CA

Once active, the issuing CA exposes a **SCEP URI** used in the SCEP certificate profile.

**Bring Your Own CA (BYOCA)** is for organizations that already have an on-premises or third-party root CA. It creates an Intune issuing CA that chains to the existing root, so the external root stays the trust anchor while Cloud PKI provides the automation and SCEP services.

## 4. Publish trust to devices

Devices must trust the root CA and the issuing CA before they accept certificates either signs. Trust is delivered with **trusted certificate profiles**, one for the root CA and one for the issuing CA.

Path: Devices > Configuration > Create > New policy > platform **Windows 10 and later** > **Templates > Trusted certificate**. Name it, upload the CA certificate (downloadable from the Cloud PKI CA detail page), add scope tags, and assign to the appropriate device or user groups. Repeat for the issuing CA certificate.

Assign both trust profiles **before** deploying any SCEP profile. If trust has not installed first, certificate validation fails.

## 5. Configure SCEP certificate profiles

Path: Devices > Configuration > Create > New policy > **Windows 10 and later** > **Templates > SCEP certificate**. Settings on the Configuration settings page:

- **Certificate type** — Device or User.
- **Subject name format** — standard values, with dynamic variables.
- **Certificate validity period** — for example one year.
- **Key storage provider (KSP)** — Software KSP or TPM KSP.
- **Key usage** — Digital signature, Key encipherment.
- **Root certificate** — the trusted certificate profile created for the issuing CA.
- **SCEP Server URLs** — the SCEP URI from the issuing CA detail page.

A separate SCEP profile is required **for each platform** (Windows, iOS/iPadOS, Android, macOS). The configuration is the same; the profiles are platform-specific in Intune.

**Subject name formats** (dynamic variables):

| Use case | Format |
|---|---|
| Device certificate (Windows) | `CN={{DeviceId}}` |
| User certificate | `CN={{UserName}}` |
| User certificate with domain | `CN={{UserPrincipalName}}` |
| Device certificate with serial | `CN={{SerialNumber}}` |

**Subject alternative names** can combine types: UPN (`{{UserPrincipalName}}`), email (`{{EmailAddress}}`), DNS name (`{{FullyQualifiedDomainName}}`). For Wi-Fi and VPN authentication, include the UPN as a SAN so network access servers can identify the user from the certificate without querying Active Directory.

**Validity and renewal:**

- **Certificate validity period** — how long the certificate is trusted; one year is common, 90 days improves security by limiting exposure if a certificate is compromised.
- **Renewal threshold** — the percentage of lifetime remaining when Intune triggers automatic renewal. Default **20%**; for a one-year certificate that starts renewal about **73 days** before expiry.

Renewal only works automatically for enrolled devices that are actively checking in. Offline devices can miss the window, so use a **lower threshold (30 to 40%)** for mobile or remote devices that check in less frequently. When renewal triggers, the device generates a new CSR, repeats the SCEP flow, and the old certificate is automatically revoked.

**Key usage and EKU** must match the service consuming the certificate, because mismatched EKUs cause authentication failures:

| Scenario | Key usage | EKU |
|---|---|---|
| Device authentication | Digital signature, Key encipherment | Client authentication (1.3.6.1.5.5.7.3.2) |
| Wi-Fi authentication | Digital signature | Client authentication |
| Email signing | Digital signature | Secure email (1.3.6.1.5.5.7.3.4) |
| Code signing | Digital signature | Code signing (1.3.6.1.5.5.7.3.3) |

**Platform key storage:** Windows supports TPM key storage; iOS/iPadOS stores certificates in the device keychain; Android has separate profiles per enrollment mode (Device Administrator and Android Enterprise); macOS stores them in the system keychain. On Windows the KSP can be set to "Enroll to TPM KSP if present, otherwise Software KSP" (use TPM when available) or "Enroll to TPM KSP, otherwise fail" (enforce hardware-backed keys and fail enrollment without a TPM). TPM-protected keys resist extraction far better than software-backed keys.

## 6. What happens after assignment

Once profiles are saved and assigned, the sequence runs automatically at device check-in:

1. The device receives the trusted certificate profiles and installs the root and issuing CA certificates.
2. The device receives the SCEP profile and generates a CSR. **The private key is created on the device and never leaves it.**
3. The device sends the CSR plus an encrypted SCEP challenge to the Cloud PKI SCEP service. The challenge is encrypted and signed with Intune's SCEP registration authority keys.
4. The SCEP validation service verifies the request came from an enrolled, managed device and that the challenge is authentic and untampered.
5. The SCEP validation service (the registration authority) asks the issuing CA to sign the CSR.
6. The issuing CA signs and Intune delivers the certificate to the device.

The whole process finishes within minutes of check-in. Steps 2 through 6 are fully automated from the administrator's perspective.

## 7. Revocation and the CRL

Revocation removes trust in a certificate before it expires, which matters when a device is lost, stolen, decommissioned, or a user leaves. Cloud PKI keeps a **CRL per issuing CA**:

- CRL validity period: **7 days**
- CRL refresh interval: **every 3.5 days**
- Manual revocation updates the CRL **immediately**

Because validity (7 days) is longer than the refresh interval (3.5 days), a valid CRL always exists even if a refresh cycle is missed. Manual revocation: Tenant administration > Cloud PKI > select the issuing CA > **View all certificates** > select the certificate > **Revoke** (requires the Revoke issued leaf certificates permission). Retiring or wiping a device in Intune automatically revokes that device's certificates.

Important caveat: applications often cache the CRL and check it only periodically. When revocation must take effect immediately, such as blocking a lost device, pair it with a **Conditional Access policy** so access is blocked on compliance in near-real time while the CRL cache catches up.

## 8. Monitoring certificate health

**Cloud PKI dashboard** (Tenant administration > Cloud PKI > select issuing CA > View all certificates) lists device name, certificate state (Active, Expired, Revoked), issued date, expiry date, subject, and serial number. Reports here refresh **every 24 hours**; the CA details page shows aggregate counts that refresh more often. When an issuing CA has issued more than **1,000** certificates, the CA details page shows only the first 1,000.

**Devices > Monitor > Certificates** is the broader view across all CAs and devices, filterable by device, platform, certificate state, or CA, with no 1,000-record limit. It is the better tool for troubleshooting a single device and seeing every certificate it holds.

**Certificate states:** Active means valid and within its period; Expired means the device failed to renew (usually offline past the renewal window) and will fail authentication until it re-enrolls; Revoked means manually revoked or automatically revoked by wipe, retirement, or unenrollment, and the certificate is on the CRL and no longer trusted.

**Audit log** (Tenant administration > Audit logs, filter Category > Cloud PKI) records CA creation, certificate revocation, certificate searches, and CA property changes, with the actor and timestamp. Export to a SIEM for long-term retention.

## 9. Troubleshooting and proactive alerts

Certificates not being issued:

- Confirm both the trusted certificate profiles and the SCEP profile are assigned to the device's group.
- Check profile order: trust must reach the device before SCEP. Re-check that no assignment filter is blocking the trust profiles.
- Confirm the device is enrolled and can reach the Cloud PKI SCEP URI (network access to Microsoft endpoints).

Certificates expiring unexpectedly:

- The device likely was not checking in during the renewal window. Lower the renewal threshold (for example 40%) to extend the window.
- Verify the device's Last check-in time in the Devices workload.
- For remote or occasionally connected devices, pair a shorter validity (90 to 180 days) with a higher renewal threshold so there is always time to renew.

Authentication failures despite an issued certificate:

- Confirm both root and issuing CA trusted certificate profiles reached the device (check the profile's Device status tab).
- Confirm the EKU values match what the authentication server requires.
- Confirm the subject name and SAN match what the server expects.
- On Windows, inspect with `certmgr.msc` or `certutil -store My`, which shows the certificate, its subject and SAN, and the issuer chain.

Proactive alerts, since Cloud PKI has no built-in email alerts for expiry:

- Export the Intune certificate report regularly and parse for certificates expiring inside the alert window.
- Route Entra sign-in logs (certificate authentication failures appear as sign-in failures) to Azure Monitor or a Log Analytics workspace and build alert rules.
- Use a compliance policy that marks a device noncompliant when a required certificate is missing or expired, with Conditional Access blocking access until compliance is restored.

## 10. Knowledge check — answers and reasoning

1. Maximum CAs in a single Intune tenant → **6.** The limit counts root CAs, issuing CAs, and BYOCA together, so a two-tier production hierarchy plus pilot hierarchies must fit inside it. 2 and 10 are not the documented limit.

2. Devices receive the SCEP profile but reject the issued certificates → **The trusted certificate profiles for the root CA and issuing CA were not deployed before the SCEP profile.** Devices must install the trust chain first. A 5-year issuing CA validity is the recommendation, and a 20% renewal threshold has no bearing on initial enrollment.

3. Where the private key is created during SCEP enrollment → **On the device itself; it never leaves the device.** This is the fundamental SCEP security property. The key is not generated on the SCEP service or the issuing CA and then shipped.

4. Remote, infrequently connecting devices showing expired certificates → **Increase the renewal threshold to 40% and shorten validity to 180 days to extend the renewal window.** Raising validity to 5 years keeps a stale certificate around and does not fix missed check-ins, and manually revoking and reissuing monthly is unmanaged toil that defeats the automation.

5. Tool to verify on a Windows device that a certificate arrived with the right subject and SAN → **`certmgr.msc` or `certutil -store My` run on the device.** The Cloud PKI dashboard shows issued records, not the on-device store, and Entra sign-in logs show authentication failures, not certificate contents.

## 11. Self-check

- I can explain what on-premises infrastructure Cloud PKI eliminates (NDES, certificate connector, CA servers) and how each issuing CA provides its own SCEP service.
- I can state the licensing options, the six-CA tenant limit, and the trial key limitation.
- I can name the four RBAC permissions and the order of hierarchy creation.
- I can describe the mandatory deployment order and explain why SCEP before trust fails.
- I can pick subject name formats and SAN types for device, user, Wi-Fi, and VPN certificates.
- I can calculate a renewal window from validity and threshold, and adjust both for remote devices.
- I can match key usage and EKU to a scenario and explain why a mismatch breaks authentication.
- I can trace the SCEP flow and state where the private key lives.
- I can explain CRL validity versus refresh, and when to pair revocation with Conditional Access.
- I can choose the dashboard versus Devices > Monitor > Certificates and account for the 24-hour refresh and 1,000-record limits.
- I can diagnose the three common failure classes and name the built-in proactive alert options.
