# Atera RMM: Remote Monitoring and Management Lab

## Overview

This lab documents the deployment and configuration of Atera, a cloud-based Remote Monitoring and Management (RMM) platform used by MSPs and IT support teams to monitor endpoints, initiate remote sessions, and execute remote scripts without physical access to the device.

Atera was deployed against the same Windows 11 Pro endpoint already enrolled in Miradore MDM, demonstrating a realistic multi-tool endpoint management stack where MDM handles policy enforcement and compliance while RMM handles remote support and monitoring.

---

## Environment

| Component | Detail |
|---|---|
| Hypervisor | Proxmox VE (Ryzen 9 9900X, 64GB RAM) |
| Managed Endpoint | Windows 11 Pro VM on VLAN 10 (10.0.10.0/24) |
| Atera Tenant | Free trial (31-day, 300 agent installs) |
| Site | Local Lab |
| Also enrolled in | Miradore MDM (same endpoint) |

---

## Tasks Completed

### 1. Agent Deployment

- Created a named customer site ("Local Lab") in the Atera console prior to agent installation
- Downloaded the Windows agent installer from the Atera dashboard; the agent token is baked into the installer and automatically associates the device with the Local Lab tenant on install
- Executed installer on the Windows 11 Pro VM as Administrator
- Device appeared online in the Atera Devices view within approximately 2 minutes of installation
- Device assigned to Local Lab site, confirming site-level agent onboarding workflow

**Note:** In production MSP environments, Atera's site structure maps to individual client organizations. Every device belongs to a named site, enabling scoped alerting, reporting, and billing. The one-installer-per-tenant model means technicians can onboard client devices without configuring agent tokens manually.

---

### 2. Remote Desktop Session

- Initiated remote desktop session from the Atera console directly to the Windows 11 Pro VM
- Navigated the remote desktop to verify live connection and confirmed active session
- Session established without requiring VPN or direct network access beyond the Atera agent's outbound connection

**Screenshot:**
<img width="1917" height="995" alt="Atera-Connect" src="https://github.com/user-attachments/assets/cc53f965-dbc9-4916-b45d-6548490f2b34" />


---

### 3. Remote PowerShell Script Execution

Remote script execution via the Atera console requires a paid plan upgrade (see [findings.md](./findings.md)). The workaround was executed within the established remote desktop session:

- Opened PowerShell on the managed endpoint from within the RMM-initiated remote session
- Executed diagnostic script to pull system information:

```powershell
hostname
systeminfo | findstr /C:"OS Name" /C:"System Boot Time"
whoami
```

- Output confirmed: hostname, OS version, last boot time, and logged-in user returned successfully

**Screenshot:**
<img width="1914" height="995" alt="Atera-Remote-PowerShell" src="https://github.com/user-attachments/assets/23a8f251-9470-4cb1-9e68-6a4e31938cf0" />


**Production context:** Remote script execution is a core RMM capability used for system triage, patch verification, software inventory, and support automation. It allows technicians to remediate issues without interrupting end users or requiring physical access to the endpoint.

---

### 4. Threshold Alert Configuration

- Navigated to Admin > Thresholds in the Atera console
- Created a new threshold profile: **"Local Lab Profile"**
- Added a custom threshold item with the following configuration:

| Field | Value |
|---|---|
| Name | Local Lab Item |
| Category | CPU Load |
| Severity | Critical |
| Threshold | 80% |
| Time Period | 1.5 minutes |

- Assigned Local Lab Profile to the Local Lab site; profile is inherited by all agents under that site
- Windows 11 VM confirmed as assigned to Local Lab Profile in the device's Profiles section

**Screenshot:**
<img width="1690" height="895" alt="Threshold-Profile" src="https://github.com/user-attachments/assets/69f5f644-a3a3-4892-8465-9224f16f3c58" />

<img width="1920" height="994" alt="Profile-Assignment" src="https://github.com/user-attachments/assets/ff394a4e-5e60-4d62-be67-ae72dd27834e" />


**Production context:** Threshold alerting enables proactive monitoring so technicians are notified before users report issues. CPU and disk alerts are among the most common triggers in MSP environments, used to identify runaway processes, storage exhaustion, and performance degradation.

---

## Tool Stack Context

Atera was deployed alongside Miradore MDM on the same Windows 11 endpoint to demonstrate complementary tooling:

| Tool | Role |
|---|---|
| Miradore MDM | Policy enforcement: password policy, Defender controls, disk write protection, script block logging |
| Atera RMM | Remote access and monitoring: remote desktop, diagnostic scripting, threshold alerting |

In production, MDM and RMM serve different functions and are commonly deployed together. MDM enforces compliance configuration at the OS level; RMM enables reactive and proactive support operations.

---

## Skills Demonstrated

- RMM agent deployment and device onboarding into a named customer site
- Remote desktop session initiation from a cloud-based RMM console
- Endpoint diagnostic scripting via remote session
- Threshold profile creation and site-level assignment
- MSP support workflow documentation from device onboarding to remote remediation
- Tier limitation identification and workaround documentation
