# Miradore MDM: Endpoint Enrollment, Policy Deployment, and Remote Management

**Section:** IT Administration Capstone — Endpoint Management
**Platform:** Miradore (a GoTo product), free tier with 14-day Premium trial
**Tenant:** online.miradore.com/locallab001
**Device enrolled:** Windows 11 Pro, Proxmox-hosted VM

---

## Overview

This section covers the configuration and use of Miradore as a cloud-based Mobile Device Management (MDM) platform to enroll, configure, and remotely manage a Windows 11 endpoint.

Miradore was selected because it provides genuine enterprise MDM functionality at no cost. Microsoft Intune, the enterprise-standard equivalent, requires Microsoft 365 Business Premium or higher. Miradore fills that gap for the purposes of this lab, and because both platforms use the same underlying Windows CSP framework, the skills transfer directly.

All tasks performed here map to endpoint management responsibilities commonly listed in Tier 2 Help Desk and IT Support Specialist job postings.

---

## Environment

| Component | Details |
|---|---|
| MDM Platform | Miradore, free tier with 14-day Premium |
| Enrolled Device | Windows 11 Pro, Proxmox VM |
| Hypervisor | Proxmox VE |
| Lab Domain | soc-lab.local (Active Directory on Windows Server 2022) |
| Identity Context | Device user consistent with Okta-managed AD users |

---

## Task 1: Device Enrollment

### Enrollment Path

Miradore supports two Windows enrollment types. **Full management** pushes an MDM profile automatically and enables CSP policy delivery. **Light management** installs the Miradore client manually without pushing an MDM profile, which limits policy enforcement capability. Full management was selected to demonstrate the complete enrollment and policy workflow.

Enrollment was initiated from the Miradore console via My Company > Devices > Enroll Device > Windows > Full Management. The console generates enrollment credentials and directs the administrator to complete enrollment on the target device through Windows Settings.

### Windows 11 Enrollment: Process

On the Windows 11 VM:

1. Opened Settings > Accounts > Access work or school
2. Selected the MDM-specific enrollment option, which is separate from the Connect button used for Entra ID or Microsoft account joining
3. Entered Miradore-provided enrollment credentials
4. Enrollment completed; the console confirmed Status: ENROLLED

**Finding:** During enrollment, Windows presented a Microsoft account lookup dialog and returned the error "That work or school account couldn't be found." Windows was treating the Miradore credential format as an Entra ID organizational account and attempting to resolve it against Microsoft's directory. The correct path uses the MDM-specific enrollment flow, which communicates directly with the Miradore MDM server and bypasses the Microsoft identity lookup entirely.  

**Finding:** The enrolled device shows Management type: Client + MDM, confirming both the Miradore client agent and the MDM enrollment profile are active on the endpoint.  

### Screenshots

<img width="1728" height="906" alt="Enrollment-Win11-Success" src="https://github.com/user-attachments/assets/faf677fc-3406-4592-b89e-1f24225fd359" />


<img width="1536" height="895" alt="Successful-Enroll-2" src="https://github.com/user-attachments/assets/b8fe29b7-8531-41f1-af31-aa4800ec1e62" />

---

## Task 2: Configuration Profile Deployment

Configuration profiles define the security and compliance settings pushed to managed devices. Four profiles were created and deployed through a Business Policy scoped to all enrolled devices.

### Profile 1: Windows Password Policy

**Type:** Built-in Password profile  
**Purpose:** Enforce CIS-aligned password requirements on the managed endpoint  

| Setting | Value | Notes |
|---|---|---|
| Password required | Yes | Baseline requirement |
| Minimum length | 14 characters | CIS Benchmark |
| Minimum password age | 1 day | Prevents immediate cycling |
| Password history | 10 passwords | CIS Benchmark |
| Maximum failed attempts | 4 | CIS Benchmark lockout threshold |
| Maximum screen lock timeout | 15 minutes | CIS Benchmark |
| Password expiration | Not set | Intentional lab decision; production recommendation is 90 days per CIS Benchmark |

### Profile 2: Defender Security Controls

**Type:** Custom Policy CSP  
**Purpose:** Enforce Windows Defender real-time protection and behavior monitoring via MDM-pushed CSP policy  

| Area | Policy Name | Value | Effect |
|---|---|---|---|
| Defender | AllowRealtimeMonitoring | 1 | Enforces real-time AV monitoring |
| Defender | AllowBehaviorMonitoring | 1 | Enforces behavior-based threat detection |

### Profile 3: Removable Disk Write Protection

**Type:** Custom Policy CSP  
**Purpose:** Block write access to removable storage devices to prevent unauthorized data exfiltration  

| Area | Policy Name | Value | Effect |
|---|---|---|---|
| Storage | RemovableDiskDenyWriteAccess | 1 | Blocks write access to USB and removable media |

This control maps to data loss prevention (DLP) requirements common in regulated environments and is referenced in both CIS and NIST endpoint security frameworks.

### Profile 4: PowerShell Script Block Logging

**Type:** Custom Policy CSP (ADMX-backed)  
**Purpose:** Enforce PowerShell script block logging so all script execution is captured in the Windows event log for security auditing  

| Area | Policy Name | Value | Effect |
|---|---|---|---|
| WindowsPowerShell | TurnOnPowerShellScriptBlockLogging | 1 | Logs all PowerShell script input to the Microsoft-Windows-PowerShell/Operational event log |

**Finding:** This profile is an ADMX-backed CSP policy. Miradore accepted and queued it without error but it did not appear in the device Deployments tab alongside the other three profiles. See findings.md for detail.

### How Custom Policy CSP Works in Miradore

Miradore's Custom Policy profile type uses the Windows Configuration Service Provider (CSP) framework, the same mechanism used by Microsoft Intune. Policies are defined with three fields:

- **Area name:** the CSP category (e.g., Defender, Storage, WindowsPowerShell)
- **Policy name:** the specific policy within that area, sourced from Microsoft's Policy CSP documentation
- **Value:** the integer or string value defining the desired state

Multiple policy entries can be added within a single Custom Policy profile. Miradore validates policy names against the Policy CSP specification and returns an error for unrecognized names.

**Note on policy precedence:** When both a Group Policy Object and a CSP-based MDM policy target the same setting, Group Policy wins by default on Windows. This can be overridden using the ControlPolicyConflict CSP. No conflicting GPOs were present on the standalone Windows 11 VM used in this lab.

**Tier note:** Custom Policy CSP requires Miradore Premium and is not available on the free tier.

### Business Policy

All four profiles were grouped into a single Business Policy named **Endpoint Compliance Policy**, scoped to all enrolled devices. This reflects how MDM policies work in production: profiles are authored individually and bundled into policies that define the compliance baseline for a device population.

In a production environment, Miradore's tag-based device scoping would allow different compliance baselines to be applied to different device groups such as executive devices, shared kiosks, or contractor endpoints.

### Screenshots

<img width="1536" height="770" alt="Deploy-4-confs" src="https://github.com/user-attachments/assets/534baf5f-8ee3-497f-8caa-efbecd2ccc1c" />

<img width="1536" height="801" alt="Config-Profiles" src="https://github.com/user-attachments/assets/474e4b85-b1a1-45c7-a966-e3a06a4d67e4" />


---

## Task 3: Policy Verification via Registry

After deployment, policy application was verified directly on the Windows 11 VM via PowerShell:

```powershell
Get-ChildItem "HKLM:\SOFTWARE\Microsoft\PolicyManager\current\device"
```

Output confirmed MDM-managed policy keys in the registry, verifying that the CSP policy push was received and processed by the Windows MDM client.

### Screenshot
<img width="1536" height="897" alt="terminal" src="https://github.com/user-attachments/assets/1d850b60-7abd-4d3f-bf70-f9389d1ee649" />


---

## Task 4: Remote Action — Device Reboot

A remote reboot was executed from the Miradore console via the Actions dropdown on the device profile page.

**Steps taken:**
1. Opened the device profile in the Miradore console
2. Selected Actions > Reboot device
3. Confirmed the action
4. The Windows 11 VM displayed a shutdown notification confirming receipt of the remote command
5. The VM rebooted successfully

**Finding:** The reboot did not execute immediately after the action was confirmed. There was a short delay before the shutdown notification appeared. This is standard MDM behavior: remote commands are delivered on the device's next check-in cycle rather than in real time. RMM tools such as Atera or NinjaOne typically execute remote commands faster due to persistent agent connections.

**Policy enforcement confirmed on reboot:** Following the reboot, the Windows Password Policy configuration was visibly enforced on the endpoint. This confirms the complete end-to-end delivery chain: profile authored in Miradore > deployed via Business Policy > received by the Windows MDM client > enforced at OS level.

### Screenshots
<img width="1536" height="768" alt="Device-Reboot" src="https://github.com/user-attachments/assets/a98e6d23-e585-4e6e-818c-b08592716bc7" />

<img width="1044" height="651" alt="Remote-Reboot" src="https://github.com/user-attachments/assets/9a133365-e096-4a16-854f-4d765e910ac1" />

<img width="1044" height="651" alt="Password-Policy-Enforced-2" src="https://github.com/user-attachments/assets/c34e0d73-ad60-44cf-98c8-2604df3ca058" />


---

## Findings Summary

| # | Finding | Category |
|---|---|---|
| 01 | Windows Server 2022 does not support MDM client enrollment; Windows 11 client VM required | Architecture |
| 02 | Enrollment dialog presented Microsoft account lookup; MDM-specific path bypasses Entra ID validation | Enrollment |
| 03 | Management type confirmed as Client + MDM | Enrollment |
| 04 | Custom Policy CSP requires Miradore Premium; not available on free tier | Tier limitation |
| 05 | Firewall enforcement not available as a native profile type; documented as Intune gap | Platform limitation |
| 06 | Group Policy takes precedence over CSP MDM policy by default; ControlPolicyConflict CSP can override | Policy precedence |
| 07 | PowerShell Script Block Logging profile deployed without error but absent from Deployments tab | Platform behavior |
| 08 | Per-profile compliance status not surfaced in device view; registry verification used instead | Reporting limitation |
| 09 | Remote reboot executed after MDM check-in delay; consistent with MDM command delivery behavior | Remote actions |
| 10 | Password policy enforcement confirmed visually on endpoint post-reboot | Verification |
| 11 | Device enrolled under admin account due to lab mail server constraint | Lab constraint |
| 12 | Miradore CSP framework is identical to Intune OMA-URI; skills transfer directly | Architecture note |

Full detail on each finding is in [findings.md](./findings.md).

---

## How This Maps to Real IT Support Workflows

| Lab Task | Real-World Equivalent |
|---|---|
| Device enrollment | Onboarding a new employee laptop into corporate MDM |
| Password policy deployment | Enforcing organizational password standards across the device fleet without touching each machine manually |
| Defender CSP enforcement | Ensuring AV protection remains active regardless of user action |
| Removable disk write protection | DLP control preventing data exfiltration via USB in regulated environments |
| PowerShell script block logging | Security hardening for forensic visibility into script-based attacks |
| Remote reboot | Tier 2 support action for remotely restarting an unresponsive or post-patch endpoint |
| Registry verification | Confirming policy application during troubleshooting without physical device access |

---

## Platform Notes

**Miradore vs. Microsoft Intune:**
Miradore and Intune both use the Windows CSP framework for policy delivery. The area names, policy names, and values used in Miradore's Custom Policy profiles are identical to those used in Intune's custom OMA-URI profiles. Intune provides a broader built-in policy library and more detailed compliance reporting, but the underlying technical mechanism is the same. Hands-on experience with Miradore's Custom Policy CSP workflow applies directly to Intune administration.

**Free tier vs. Premium:**
Core enrollment, built-in configuration profiles (Password, Wi-Fi, Windows Update), and remote actions are available on the free tier. Custom Policy CSP, which enables the Defender, Storage, and PowerShell profiles documented here, requires Premium.
