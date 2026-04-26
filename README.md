# Enterprise IT Administration Capstone
### Identity, Endpoint, and Support Operations in a Simulated Enterprise Environment

**Joseph De Santiago**
[LinkedIn](https://linkedin.com/in/joseph-d-0875a324b/)

---

## Overview

This repository documents a fully operational, self-built enterprise IT lab environment designed to simulate the day-to-day responsibilities of a Tier 2 IT Administrator or Help Desk professional. Every tool in this project was selected because it appears in real job postings, every task was performed in a live environment rather than a tutorial, and every finding was documented the way a working administrator would document it.

The lab is built on a Proxmox hypervisor running eight virtual machines across four VLAN-segmented networks. At its core is an Active Directory domain (soc-lab.local) hardened to approximately 393 CIS controls, serving as the identity foundation for every other platform in the environment. Cloud IAM, endpoint management, remote monitoring, and IT service management are all layered on top of that foundation, integrated with each other, and documented with the same rigor expected in a professional environment.

The goal is not to demonstrate familiarity with a list of tools. The goal is to demonstrate operation, troubleshooting, and documentation within an enterprise IT environment end to end.

---

## Lab Infrastructure

| Component | Details |
|---|---|
| Hypervisor | Proxmox VE on Ryzen 9 9900X, 64GB RAM, 2TB SSD |
| Domain | soc-lab.local (Windows Server 2022, CIS hardened) |
| Network | pfSense with VLAN segmentation across four subnets |
| SIEM | Splunk ingesting Windows event logs |
| EDR | Velociraptor deployed on endpoints |
| SOAR | n8n on Docker/Portainer |
| Admin workstation | Linux Mint (VLAN 99 isolated admin network) |
| Attack VM | Kali Linux (VLAN 30 isolated attack network) |

---

## What This Project Covers

Each section of this repository represents a distinct area of enterprise IT administration. They are not independent tutorials: the same Active Directory users appear in Okta, the same endpoints enrolled in MDM are monitored in RMM, and the Microsoft 365 tenant is administered against the same identity foundation as the on-premises domain. This is a cohesive environment, not a collection of checkboxes.

---

## Projects

### [Okta — Cloud Identity and Access Management](./Okta)

Okta Workforce Identity was integrated with the on-premises Active Directory domain via the Okta AD Agent, providing cloud-based identity management, MFA enforcement, and SSO capabilities layered on top of an existing Windows directory infrastructure. This is one of the most common enterprise IAM architectures in mid-market organizations today.

**What was configured and documented:**
- AD Agent deployment on a domain-joined Windows Server with a dedicated least-privilege service account
- Directory integration scoped to specific OUs, with full user and group import from soc-lab.local
- Authentication policy requiring MFA for all users, enforced via Okta Verify
- Password policy with complexity, length, and rotation requirements
- SAML 2.0 SSO integration with AWS IAM Identity Center, with Okta serving as the external identity provider
- Sign-in and system log analysis for authentication event investigation
- User lifecycle management: provisioning, suspension, reactivation, and deprovisioning across both Okta and AD simultaneously

**Why it matters:** Okta is one of the most widely deployed cloud IAM platforms in the industry. Administering it in a real environment with AD integration and SAML federation demonstrates a level of hands-on experience that most Help Desk and Tier 2 candidates cannot claim.

See [Okta/README.md](./Okta/README.md) and [Okta/findings.md](./Okta/findings.md) for full documentation.

---

### [Miradore — Mobile Device Management](./Miradore%20MDM)

Miradore MDM was used to enroll, manage, and enforce compliance policies on a Windows 11 Pro endpoint. MDM is one of the primary mechanisms through which enterprise IT teams enforce security baselines on managed devices without requiring direct administrator access to each machine.

**What was configured and documented:**
- Windows 11 Pro VM enrollment as a managed endpoint
- Compliance policy creation and enforcement with documented pass/fail behavior
- Custom Policy CSP implementation with registry-level verification confirming policy application
- Remote action execution: lock, wipe command initiation, and compliance status review
- Enrollment and compliance findings documented including observed platform behaviors

**Why it matters:** MDM administration is a core Tier 2 responsibility in any organization managing a fleet of devices. Understanding how policies are applied, how compliance is enforced, and how to verify that enforcement reached the device is directly applicable to day-one work in a Help Desk or endpoint management role.

See [Miradore MDM/README.md](Miradore%20MDM/README.md) and [Miradore MDM/findings.md](Miradore%20MDM/findings.md) for full documentation.

---

### [Atera — Remote Monitoring and Management](./Atera%20RMM)

Atera RMM was deployed on the same Windows 11 Pro endpoint managed by Miradore, creating an environment where a single device is simultaneously under MDM policy enforcement and RMM remote monitoring. This mirrors the real enterprise configuration where endpoint management and remote support operate in parallel.

**What was configured and documented:**
- Atera agent deployment and registration on a managed Windows 11 endpoint
- Remote session initiation and navigation for live support simulation
- Diagnostic script execution for system health and inventory data collection
- Threshold alerting configuration for CPU, memory, and disk metrics
- RMM and MDM co-existence on a single endpoint documented as an architectural finding

**Why it matters:** RMM tools are the primary mechanism through which Help Desk and Tier 2 technicians deliver remote support at scale. Familiarity with agent deployment, remote session management, scripting, and alerting is a direct indicator of readiness to contribute from day one.

See [Atera RMM/README.md](Atera%20RMM/README.md) and [Atera RMM/findings.md](Atera%20RMM/findings.md) for full documentation.

---

### [Microsoft 365 — Cloud Productivity Administration](Microsoft%20365/)

A live Microsoft 365 Business Basic tenant (SterlingEnterprises) was provisioned and administered across three service areas: Entra ID, Exchange Online, and SharePoint Online. This is the most in-demand skill set appearing in Help Desk and Tier 2 job postings today. Every task was performed in a real tenant, every finding was documented honestly, and every tier limitation encountered was recorded with its operational and licensing implications.

The M365 section is organized into three subsections, each with its own README and findings documentation.

---

#### [Entra ID — Cloud Identity Administration](./Microsoft%20365/Entra%20ID)

**What was configured and documented:**
- Full user lifecycle: provisioning, license assignment, password reset with forced change, account disable and enable, deletion, and restoration from recycle bin
- Security group and Microsoft 365 unified group creation with user assignments and documented architectural distinction between group types
- MFA enforcement via Security Defaults with tenant-wide legacy authentication blocking confirmed active
- Microsoft Authenticator enrollment documented as a step-by-step support procedure
- Sign-in log analysis including deliberate failed authentication event generation and error code 50126 interpretation
- Audit log review covering all administrative actions from the session
- Conditional Access attempted and documented as a P1 tier limitation with full articulation of the Security Defaults vs. Conditional Access distinction

See [Microsoft 365/Entra ID/README.md](Microsoft%20365/Entra%20ID/README.md) for full documentation.

---

#### [Exchange Online — Email Administration](./Microsoft%20365/Exchange)

**What was configured and documented:**
- Shared mailbox provisioned with Full Access and Send As delegation configured for a technician account
- Distribution group created with enterprise-appropriate closed membership and external sender restrictions
- Mail flow rule configured: outbound disclaimer enforced in Enforce mode with documented configuration decisions for fallback action, severity, and sender address matching
- Message trace executed with full field documentation; mail flow rule execution confirmed via transport rule events in trace detail panel
- Out-of-office auto-reply and mailbox forwarding configured via admin center; deliver-to-both retention enabled and documented as a security-relevant default
- Exchange Online mailbox provisioning failure diagnosed and resolved via license reapplication: a realistic Tier 2 troubleshooting scenario documented in full

See [Microsoft 365/Exchange/README.md](Microsoft%20365/Exchange/README.md) and [Microsoft 365/Exchange/findings.md](Microsoft%20365/Exchange/findings.md) for full documentation.

---

#### [SharePoint Online — Collaboration and Document Management](./Microsoft%20365/SharePoint)

**What was configured and documented:**
- Team site and Communication site provisioned with documented architectural distinction between site types and their backing resource models
- Permission levels configured across Owner, Member, and Visitor roles with users assigned
- Custom permission level created (Helpdesk Technician) applying least-privilege design for support staff access
- Document library created with a custom choice column (Document Type), major versioning enabled, and version history demonstrated across multiple document versions
- OneDrive storage allocation reviewed; pooled tenant storage model documented for Business Basic tier
- External sharing policy reviewed: maximally permissive default configuration identified and documented with full security hardening recommendations

See [Microsoft 365/SharePoint/README.md](Microsoft%20365/SharePoint/README.md) and [Microsoft 365/SharePoint/findings.md](Microsoft%20365/SharePoint/findings.md) for full documentation.

---

## Certifications

| Certification | Issuer | Status |
|---|---|---|
| CISSP Associate | ISC² | Active |
| CySA+ | CompTIA | Active |
| Google Cybersecurity Professional | Google | Active |

---

## Repository Structure

```
enterprise-it-admin-capstone/
├── README.md
├── okta/
│   ├── README.md
│   ├── findings.md
│   └── screenshots/
├── Miradore MDM/
│   ├── README.md
│   └── findings.md
├── Atera RMM/
│   ├── README.md
│   └── findings.md
└── Microsoft 365/
    ├── Entra ID/
    │   ├── README.md
    │   └── screenshots/
    ├── Exchange/
    │   ├── README.md
    │   ├── findings.md
    │   └── screenshots/
    └── Sharepoint/
        ├── README.md
        ├── findings.md
        └── screenshots/
```

---

## A Note on Documentation Approach

Every README in this repository was written to reflect what actually happened, not what should have happened. Tier limitations are documented honestly. Errors encountered during configuration are included alongside their diagnosis and resolution. Default platform behaviors that differ from best practice are called out with their security implications explained.

This approach reflects how a working administrator thinks about and documents their environment. A portfolio that only shows successful configurations tells an employer very little. A portfolio that shows what went wrong, why, and how it was fixed tells them considerably more.
