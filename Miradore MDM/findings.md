# Miradore MDM: Findings

**Section:** IT Administration Capstone — Endpoint Management
**Platform:** Miradore (a GoTo product), free tier with 14-day Premium trial
**Tenant:** online.miradore.com/locallab001

This document captures findings, limitations, unexpected behaviors, and investigative conclusions encountered during the Miradore MDM lab. Findings are recorded in the order they were encountered. Each entry includes the observation, the root cause or explanation, and real-world relevance where applicable.

---

## Finding 01: Windows Server 2022 Does Not Support MDM Client Enrollment

**Category:** Architecture

The first enrollment attempt was made on the lab's domain-joined Windows Server 2022 member server (10.0.10.103). The standard MDM enrollment path under Settings > Accounts > Access work or school does not exist on this machine. Specifically, the "Enroll only in device management" option present in Windows 10 and 11 client editions is absent from Server SKUs.

This is by design. MDM platforms are built to manage end-user workstations and mobile devices. Server infrastructure is managed through Group Policy, Desired State Configuration (DSC), System Center Configuration Manager (SCCM/MECM), or equivalent server management tooling.

A Windows 11 Pro VM was provisioned in Proxmox to serve as the managed endpoint.

**Real-world relevance:** In production, understanding this boundary is fundamental to scoping an MDM deployment. When a ticket comes in asking why a server cannot be enrolled in MDM, this is the answer.

---

## Finding 02: Windows Enrollment Dialog Presents Microsoft Account Lookup

**Category:** Enrollment behavior

During enrollment on the Windows 11 VM, Windows presented a "Set up a work or school account" dialog that attempted to validate the Miradore enrollment credentials against Microsoft's identity directory (Entra ID). The error returned was: "That work or school account couldn't be found. Check the account name and try again."

The Miradore credential format (@online.miradore.com) is not a Microsoft organizational account. Windows interpreted the email address as an Entra ID identity and attempted to resolve it against Microsoft's directory, where it is not registered.

The correct path is the MDM-specific enrollment flow within Settings > Accounts > Access work or school, which communicates directly with the Miradore MDM server using the provided credentials rather than performing a Microsoft identity lookup.

**Real-world relevance:** This is a common point of confusion when enrolling devices into third-party MDM platforms. Support staff onboarding devices need to understand the difference between Microsoft account joining (Entra ID join or hybrid join) and third-party MDM enrollment. The UI entry points look similar but behave differently.

---

## Finding 03: Management Type Confirmed as Client + MDM

**Category:** Enrollment

Following successful enrollment, the Miradore console showed Management type: Client + MDM for the enrolled device. This confirms both the Miradore client agent and the MDM enrollment profile are active on the endpoint.

Miradore's Full Management enrollment installs two components: a lightweight client agent used for inventory, remote actions, and patch management, and a Windows MDM enrollment profile used for CSP policy delivery. Both are required for full management capability. Light Management installs only the client agent, which limits policy enforcement to what the agent can apply directly without an MDM profile.

**Real-world relevance:** The distinction between agent-based and MDM-based management is relevant when evaluating endpoint tooling. RMM platforms (Atera, NinjaOne) rely on agents; MDM platforms rely on OS-native enrollment profiles. In enterprise environments, both are often deployed together and serve complementary purposes.

---

## Finding 04: Custom Policy CSP Requires Miradore Premium

**Category:** Tier limitation

The Custom Policy CSP profile type is not available on the Miradore free tier. Accessing it requires the Premium plan, which was available via a 14-day trial during this lab.

Miradore's free tier includes built-in profile types (Password, Wi-Fi, Windows Update, BitLocker, Mail for Exchange) but does not expose the Custom Policy CSP interface. Premium unlocks the ability to configure any Windows Policy CSP area by specifying the area name, policy name, and value directly.

**Real-world relevance:** In production Miradore deployments, Custom Policy CSP is a key Premium differentiator. Understanding tier capabilities is relevant to MDM platform procurement and scoping decisions. The equivalent feature in Microsoft Intune (custom OMA-URI profiles) is available across all Intune licensing tiers.

---

## Finding 05: Firewall Enforcement Not Available as a Native Profile Type

**Category:** Platform limitation

Miradore does not include a native Firewall configuration profile type for Windows. Available Windows profile types are BitLocker, Custom Policy, Mail for Exchange, Password, Wi-Fi, and Windows Update. No Firewall option exists.

The Custom Policy CSP interface was explored as a potential workaround. The Firewall CSP area (`./Vendor/MSFT/Firewall/`) is not present in Miradore's Area Name dropdown, indicating it is not supported in Miradore's Custom Policy implementation. The Defender CSP area, which is available, was used for antivirus enforcement instead.

In production, firewall policy enforcement would be handled through Microsoft Intune's dedicated Endpoint Security > Firewall policy, which uses the Windows Firewall CSP, or through Group Policy on domain-joined devices.

**Real-world relevance:** Firewall configuration is a standard endpoint hardening control. Recognizing where a platform's native capability ends and identifying the appropriate compensating control is the expected Tier 2 response.

---

## Finding 06: Group Policy Takes Precedence Over CSP MDM Policy by Default

**Category:** Policy precedence

Per Microsoft documentation, when both a Group Policy Object (GPO) and a CSP-based MDM policy are configured for the same setting on a Windows device, the GPO takes precedence by default. This behavior can be overridden using the ControlPolicyConflict CSP, which allows MDM policies to win over conflicting GPOs when explicitly configured.

In this lab, the enrolled Windows 11 Pro VM is not domain-joined and has no GPOs applied, so no policy conflict existed. This finding is documented for its production applicability.

**Real-world relevance:** In co-managed environments where devices are both domain-joined and MDM-enrolled, understanding policy precedence is critical for diagnosing unexpected behavior. A CSP policy that applies correctly in a standalone test environment may be silently overridden by a conflicting GPO in production.

---

## Finding 07: PowerShell Script Block Logging Profile Deployed Without Error but Absent from Deployments Tab

**Category:** Platform behavior

The PowerShell Script Block Logging configuration profile (Custom Policy CSP, WindowsPowerShell/TurnOnPowerShellScriptBlockLogging: 1) was deployed successfully. The console returned "Configuration profile deployment queued successfully to 1 devices." However, this profile did not appear in the device's Deployments tab alongside the other three deployed profiles.

The probable cause is that this policy is an ADMX-backed CSP policy, which is a distinct policy type from standard integer-value Policy CSP entries. Miradore may handle ADMX-backed policies differently in its deployment tracking, resulting in the deployment not surfacing in the standard Deployments view even when queued and processed.

Policy application for this specific profile was not independently verified via registry inspection. The three other profiles (Password, Defender, Storage) were confirmed in the Deployments tab and verified via registry.

**Real-world relevance:** ADMX-backed policies require special handling in MDM platforms and use an XML payload format rather than a simple integer value. When troubleshooting MDM policy application, verifying the registry directly rather than relying solely on console reporting is the correct diagnostic approach.

---

## Finding 08: Per-Profile Compliance Status Not Surfaced in Device View

**Category:** Reporting limitation

The Miradore device profile view does not display per-profile deployment status such as Deployed, Pending, Failed, Compliant, or Non-Compliant for individual configuration profiles. The Deployments tab shows the time of deployment, type, name, and deployment method but no status indicator.

Deployment confirmation is available at the action level ("queued successfully") but the console does not confirm whether the policy was successfully applied on the endpoint.

Registry inspection on the managed endpoint was used as a compensating verification method: `Get-ChildItem "HKLM:\SOFTWARE\Microsoft\PolicyManager\current\device"`.

**Real-world relevance:** In production MDM environments, compliance reporting is a critical operational capability. The absence of granular per-profile compliance state in Miradore's free and basic tiers is a meaningful gap relative to Microsoft Intune, which provides per-policy compliance status, remediation recommendations, and fleet-level compliance dashboards.

---

## Finding 09: Remote Reboot Executed After MDM Check-In Delay

**Category:** Remote actions

The remote reboot action was initiated from the Miradore console via Actions > Reboot device. The action did not execute immediately after confirmation. There was a short delay before the Windows 11 VM displayed a shutdown notification and rebooted.

MDM remote actions are delivered on the device's next check-in cycle, not in real time. The Miradore client agent polls the MDM server at a defined interval; the reboot command is queued and executed when the device checks in. This is standard behavior across all MDM platforms including Intune.

**Real-world relevance:** In Tier 2 support, understanding MDM command delivery latency matters for managing user expectations. A remote action queued through MDM may take minutes to execute depending on the check-in interval. RMM tools such as Atera or NinjaOne execute remote commands faster due to persistent agent connections, which is one reason both MDM and RMM tools are often deployed together in enterprise environments.

---

## Finding 10: Password Policy Enforcement Confirmed on Endpoint Post-Reboot

**Category:** Verification

Following the remotely-initiated reboot, the Windows Password Policy configuration profile was visibly enforced on the Windows 11 VM at the OS level.

This is the primary end-to-end evidence for the Miradore lab. The complete delivery chain was confirmed: profile authored in Miradore console > bundled into Business Policy > deployed to device > received by Windows MDM client > enforced at OS level post-reboot. Registry inspection earlier in the session confirmed policy key presence; the post-reboot enforcement confirms that the policies were not only written to the registry but actively applied by the operating system.

**Real-world relevance:** A reboot is often required for certain policy types, particularly password and security policies, to take full effect. Coordinating MDM policy deployment with scheduled reboots or maintenance windows is standard practice in production environments.

---

## Finding 11: Device Enrolled Under Admin Account Due to Lab Mail Server Constraint

**Category:** Lab constraint

The enrolled device was assigned to the administrator's account rather than a named lab user such as Adam Adamson. This was necessary because the lab domain (@soc-lab.local) has no external mail relay. Miradore's enrollment invitation flow sends credentials to the device user's email address, which cannot be delivered to an internal-only domain.

In production, the MDM administrator initiates enrollment from the console, which sends credentials to the end user's corporate email address. The user then completes enrollment on their own device. The device is linked to that user in the MDM console, connecting device records to user identity.

All other lab context (AD users, Okta provisioning) uses the named test accounts. The admin account appears in the MDM device record for enrollment mechanics only; the compliance and policy configuration work is independent of the assigned user.

---

## Finding 12: Miradore Custom Policy CSP Framework Is Directly Transferable to Microsoft Intune

**Category:** Architecture note

Miradore's Custom Policy CSP implementation uses the same Windows Configuration Service Provider framework, area names, policy names, and integer values as Microsoft Intune's custom OMA-URI profiles. The Miradore documentation explicitly references Microsoft's Policy CSP documentation as the authoritative source for available policies and supported values.

Both Miradore and Intune are MDM platforms that communicate with Windows devices using the OMA-DM protocol and the Windows CSP framework. The policy delivery mechanism is identical. A policy configured as Defender / AllowRealtimeMonitoring / 1 in Miradore is configured as OMA-URI `./Device/Vendor/MSFT/Policy/Config/Defender/AllowRealtimeMonitoring` with value `1` in Intune. The administrative interface differs, but the underlying technical implementation is the same.

**Real-world relevance:** Experience with Miradore's Custom Policy CSP workflow applies directly to Intune administration. In interviews, this can be framed as: "I have hands-on experience configuring Windows CSP policies via MDM including Defender enforcement, removable media controls, and PowerShell logging. The policy framework is identical between Miradore and Intune."

---

## Summary Table

| # | Finding | Category |
|---|---|---|
| 01 | Windows Server 2022 does not support MDM enrollment; Windows 11 client VM required | Architecture |
| 02 | Enrollment dialog presented Microsoft account lookup; MDM path bypasses Entra ID | Enrollment |
| 03 | Management type confirmed as Client + MDM | Enrollment |
| 04 | Custom Policy CSP requires Miradore Premium; not available on free tier | Tier limitation |
| 05 | Firewall enforcement not available natively; Intune gap documented | Platform limitation |
| 06 | Group Policy takes precedence over CSP by default; ControlPolicyConflict CSP can override | Policy precedence |
| 07 | PowerShell Script Block Logging deployed without error but absent from Deployments tab | Platform behavior |
| 08 | Per-profile compliance status not surfaced in device view; registry used for verification | Reporting limitation |
| 09 | Remote reboot executed after MDM check-in delay; consistent with MDM command delivery | Remote actions |
| 10 | Password policy enforcement confirmed post-reboot; end-to-end delivery chain verified | Verification |
| 11 | Device enrolled under admin account due to lab mail server constraint | Lab constraint |
| 12 | Miradore CSP framework identical to Intune OMA-URI; skills transfer directly | Architecture note |
