# Atera RMM: Findings and Limitations

This document records findings, tier limitations, architectural observations, and documented workarounds encountered during the Atera RMM lab. Each finding is noted as encountered, including platform constraints, consistent with honest documentation practice.

---

## Finding 1: Remote Script Execution Requires Paid Upgrade

**Category:** Tier Limitation
**Severity:** Moderate (core feature blocked)

Attempting to create and push a remote script to the managed endpoint via the Atera console returned a prompt to upgrade from the free trial plan. Remote PowerShell script execution is gated behind a paid subscription tier.

**Workaround:** Script executed manually within the RMM-initiated remote desktop session. The diagnostic script was run directly in a PowerShell window on the endpoint, with output captured via screenshot.

**Production context:** Remote script execution is a standard RMM capability used extensively in MSP environments for system triage, patch compliance verification, software inventory, and automated remediation. In production, technicians push scripts to hundreds of endpoints simultaneously from the console without opening individual remote sessions. The free trial limitation does not reflect real-world Atera functionality.

**Documentation:** Screenshot captured showing the upgrade prompt.

<img width="1920" height="994" alt="Script-Reject" src="https://github.com/user-attachments/assets/f3439d2b-7e25-4507-b553-934db6fc3a53" />


---

## Finding 2: Threshold Items and Threshold Profiles Are Distinct Objects

**Category:** Platform Architecture
**Severity:** Low (navigational confusion, no data loss)

Initial attempt to configure alerting created a threshold *item* (an individual monitored condition) without first creating a threshold *profile* (the container that holds items and is assigned to devices). The item was created successfully but could not be assigned to a device directly.

**Resolution:** Created a new threshold profile ("Local Lab Profile"), added the CPU Critical threshold item within it, then assigned the profile to the Local Lab site. The Windows 11 VM inherited the profile via site-level assignment.

**Production context:** Understanding the profile/item hierarchy is important in production. A single profile can contain dozens of threshold items covering CPU, memory, disk, services, and event logs. That profile is then assigned once at the customer or site level, applying to all devices automatically. This scales cleanly across large device fleets.

---

## Finding 3: Site Structure Maps to Client Organizations in the MSP Model

**Category:** Architectural Observation
**Severity:** Informational

Atera's "Site" construct is designed around the MSP model where each site represents a distinct client organization. Creating a named site (Local Lab) before agent installation ensures the device is properly scoped from onboarding, affecting alerting, reporting, threshold inheritance, and billing in production tenants.

Devices onboarded without a site assignment receive global default settings and are harder to manage at scale. Best practice is to create the site structure before deploying agents.

---

## Finding 4: Threshold Profile Assignment Follows an Inheritance Hierarchy

**Category:** Architectural Observation
**Severity:** Informational

Atera uses a precedence hierarchy for threshold profile assignment:

```
Agent (direct) > Folder > Site > Global Default
```

A profile assigned at the site level applies to all devices in that site unless overridden at the folder or agent level. This allows a consistent baseline profile across all client devices while permitting exceptions for specific machines (for example, servers with different CPU thresholds than workstations).

In this lab, assignment was made at the site level, which is the correct approach for a single-device lab and consistent with production practice for small client environments.

---

## Finding 5: Agent Installer Token Is Tenant-Scoped

**Category:** Architectural Observation
**Severity:** Informational

The Atera Windows agent installer has the tenant's authentication token baked in at download time. Running the installer on any machine automatically associates that device with the correct Atera tenant without requiring manual configuration post-install.

In production, MSPs can provide clients with a pre-configured installer. The client runs it and the device appears in the correct site within minutes. This is also a security consideration: the installer should be treated as a credential and not distributed publicly.

---

## Finding 6: RMM and MDM Serve Complementary, Non-Overlapping Roles

**Category:** Architectural Observation
**Severity:** Informational

Both Atera (RMM) and Miradore (MDM) were deployed on the same Windows 11 endpoint. These tools do not conflict; they serve distinct operational purposes:

| Concern | Tool | Mechanism |
|---|---|---|
| Password policy enforcement | Miradore MDM | CSP OMA-URI > Registry |
| Defender monitoring controls | Miradore MDM | Custom Policy CSP |
| Disk write protection | Miradore MDM | Custom Policy CSP |
| Script block logging | Miradore MDM | ADMX-backed CSP |
| Remote desktop access | Atera RMM | Agent outbound tunnel |
| Diagnostic scripting | Atera RMM | Console script push (paid) / in-session |
| Threshold alerting | Atera RMM | Agent telemetry > alert engine |

In enterprise environments, both tool categories are typically present. MDM (or Intune) handles compliance and configuration; RMM handles reactive support and proactive monitoring. Competency in both is relevant for Tier 2 IT Support and Junior Systems Administrator roles.
