# Okta Identity and Access Management Lab

**Part of:** Enterprise IT Administration Capstone  
**Platform:** Okta Workforce Identity Cloud (30-day trial)  
**Integration:** Active Directory (on-premises domain)

---

## Overview

This lab documents a complete hands-on Okta Workforce Identity deployment integrated with an on-premises Active Directory domain. The environment simulates an enterprise identity stack where Active Directory serves as the authoritative on-premises directory and Okta functions as the cloud IAM layer, handling authentication policy, MFA enforcement, SSO federation, and audit logging.

Every task reflects real Tier 2 IT administration workflows: provisioning users, enforcing authentication policy, configuring SSO integrations, troubleshooting lifecycle errors, and analyzing audit logs. Errors encountered during the lab were investigated, root-caused, and documented rather than omitted. The findings log reflects how this environment actually behaved, not an idealized walkthrough.

---

## Environment

| Component | Detail |
|---|---|
| Okta tenant | Workforce Identity Cloud (30-day trial) |
| On-premises directory | Active Directory (internal lab domain) |
| AD Agent host | Windows Server 2022 (domain-joined member server) |
| Domain controller | Windows Server 2022 |
| Hypervisor | Proxmox VE |
| Network isolation | pfSense firewall, VLAN-segmented lab network |

### Lab Users

| Display Name | Status |
|---|---|
| Adam Adamson | Active |
| Alice Allison | Active |
| Bob Bobson | Active |
| Chris Chrisson | Active |

---

## Task Summary

| Task | Description | Status |
|---|---|---|
| Setup | AD Agent installation, directory integration, user import | ✅ Complete |
| Task 1 | Group management — AD-sourced vs. Okta-native groups | ✅ Complete |
| Task 2 | User lifecycle — deactivation, reactivation, password reset | ✅ Complete |
| Task 3 | MFA enforcement — authentication policies, conditional access | ✅ Complete |
| Task 4 | SSO — AWS IAM Identity Center SAML federation | ✅ Complete |
| Task 5 | Okta Workflows automation | ⚠️ Partial — trial restriction |
| Task 6 | Identity Governance | ❌ Paid add-on — documented |
| Task 7 | System Log audit analysis | ✅ Complete |

---

## Setup — AD Agent Installation and Directory Integration

### What Was Configured

The Okta AD Agent was installed on a domain-joined Windows Server 2022 member server, separate from the domain controller. The agent establishes a connection between the on-premises domain and the Okta cloud tenant, enabling user import, group sync, and delegated authentication.

**Configuration decisions:**
- OU scoping: Users OU selected for user sync; domain root selected for group sync
- Username format: User Principal Name (UPN)
- Service account: Dedicated OktaService account created by the installer. Domain admin credentials are not used for agent operation, consistent with the principle of least privilege.
- Import result: All users and 20 groups imported successfully on the first full import

**Key findings from setup:**

*SSL not established:* SSL connection between the AD Agent and the Okta tenant could not be configured in the lab environment. Proceeded without SSL with the following compensating control: all AD Agent traffic is confined to the primary lab VLAN, isolated from external networks by pfSense firewall rules. In a production environment, SSL/TLS would be required and configured using an internal CA or a publicly trusted certificate.

*Domain admin excluded from import:* The domain administrator account was not imported during the full user sync. This is correct and expected behavior. Privileged accounts should not be federated to cloud IAM providers, as doing so exposes privileged credentials to cloud authentication flows.

*Attribute mapping warnings:* Warnings appeared for `distinguishedName` and `objectGUID` during attribute mapping configuration. These attributes are managed internally by the AD Agent and are not exposed for manual mapping in the trial tier. User import and sync functioned correctly despite the warnings.

---

## Task 1 — Group Management

**Navigate:** Directory → Groups

### What Was Completed

- Reviewed all AD-sourced groups imported via the AD Agent
- Created an Okta-native group: **IT Staff**
- Assigned all four lab users to IT Staff
- Documented the operational distinction between AD-sourced and Okta-native groups

### AD-Sourced vs. Okta-Native Groups

This distinction is operationally significant in any federated Okta deployment.

| Type | Source | Managed In | Membership Changes |
|---|---|---|---|
| AD-sourced | Active Directory | Okta console (read only) | Made in ADUC, synced to Okta |
| Okta-native | Okta | Okta console | Made directly in Okta |

Okta-native groups such as IT Staff are used for application assignment, authentication policy scoping, and Workflows, independently of AD. Help desk staff need to understand which group type they are working with before attempting membership changes.

### Staged User Status

All four imported AD users landed in **Staged** status post-import. Staged users exist in the Okta directory but cannot authenticate to Okta or assigned applications. Admin-initiated activation was performed for all four users. In a production environment, users would self-activate via an emailed activation link. Admin activation without email delivery is a lab constraint due to the absence of a mail server.

---

## Task 2 — User Lifecycle Management

**Navigate:** Directory → People

### What Was Completed

Three core help desk lifecycle operations were performed and documented.

**1. Account deactivation and reactivation**

Deactivating a test user account resulted in a **"Pending User Action — Password Selection Required"** state rather than immediate deactivation. The account required an admin-set temporary password before deactivation could proceed, producing a **"Password Expired"** state on reactivation.

Resolution: Password reset directly in Active Directory Users and Computers (ADUC). The change propagated back to Okta via the AD Agent, confirming that in a federated environment, Active Directory is the authoritative identity source for password management. Password operations do not resolve through the Okta console alone.

**2. Password reset**

An Okta-initiated password reset failed with: *"Password could not be changed, please contact your administrator."*

Root cause investigation:
- The AD account had **Password Never Expires** set, consistent with lab configuration
- Flag was removed but the error persisted
- Most likely cause: AD minimum password age policy blocking an immediate reset, or password writeback restricted in the trial tier

In production with full Okta licensing, password writeback to AD is a standard supported feature. The resolution path would be: verify AD password policy flags, confirm the OktaService account has write permissions to the user object, and test writeback with an account that has no policy restrictions applied.

### Key Takeaway

In an AD-federated Okta environment, help desk staff need access to both Okta (for lifecycle actions in the cloud layer) and ADUC (for password and policy operations at the directory layer). Neither tool alone provides complete lifecycle management capability.

---

## Task 3 — MFA Enforcement

**Navigate:** Security → Authenticators, Security → Authentication Policies

### Available Authenticators

| Authenticator | Factor Type | Characteristics | Status |
|---|---|---|---|
| Email | Possession | | Active |
| Okta Verify | Possession / Possession + Knowledge / Possession + Biometric | Device bound, hardware protected, phishing resistant (FastPass) | Active |
| Password | Knowledge | | Active |

### Authentication Policies Configured

**Policy 1 — Global MFA — All Users**

Baseline policy applied to all users on every application sign-in.

| Setting | Value |
|---|---|
| Scope | Any user |
| Verification required | Any 2 factor types |
| Possession constraint | Phishing resistant |
| Session prompt | After 1 hour since last sign-in |
| Admin exclusion | Admin account excluded via dedicated rule positioned at top of rule order |

**Policy 2 — IT Staff — Elevated Auth**

Stricter policy scoped to the IT Staff group, mirroring a conditional access architecture where privileged or sensitive groups carry elevated authentication requirements.

| Setting | Value |
|---|---|
| Scope | IT Staff group members |
| Verification required | Any 2 factor types |
| Possession constraint | Phishing resistant (Okta FastPass) |
| Session prompt | Every sign-in |

### Policy Architecture Notes

**Two-layer policy system:** Okta separates authentication into two distinct policy categories:
- **App Sign-On Policy** — controls access to assigned applications
- **Okta Account Management Policy** — controls self-service actions such as password reset, MFA enrollment, and profile changes

Tightening an App Sign-On policy does not affect self-service account actions. Both layers require independent configuration.

**Rule first-match logic:** Authentication policy rules are evaluated top-to-bottom. The first matching rule is applied and subsequent rules are not evaluated. Admin exclusion rules must be positioned above enforcement rules or they are silently bypassed. This is a critical operational consideration when managing policies in production.

**Policy staging:** Both policies were configured before applications were assigned. The "No apps assigned" state is expected. App Sign-On policies are enforced at the application level and become active only when assigned to a specific application. This mirrors an enterprise workflow where policies are defined before applications are onboarded.

---

## Task 4 — Single Sign-On Application Integration

### Integration Attempts and Outcomes

| Platform | Protocol | Outcome | Reason |
|---|---|---|---|
| Slack | SAML 2.0 | ❌ Blocked | Slack Pro plan required for SP-side SAML |
| GitHub | OIDC | ❌ Blocked | GitHub Teams or Enterprise required for org-level SSO |
| AWS IAM Identity Center | SAML 2.0 | ✅ Complete | Account instance path available at no cost |

**Pattern noted:** Okta IdP configuration was completed successfully for all three targets. SP-side plan requirements were the consistent blocker for Slack and GitHub. This reflects real enterprise procurement reality where SSO integration requires licensing decisions on both the IdP and SP sides.

---

### AWS IAM Identity Center — SAML Federation

**AWS side:**
- IAM Identity Center account instance enabled. The account instance path was used specifically to avoid the AWS Organizations requirement, which would have forced a free-tier account upgrade.
- External identity provider configured
- Okta dual-stack metadata XML uploaded to AWS
- Identity source changed from IAM Identity Center directory to external IdP (Okta)
- User manually provisioned in AWS IAM Identity Center with a username matching the Okta UPN exactly

**Okta side:**
- AWS IAM Identity Center integration added from the Okta Integration Network
- AWS dual-stack SP metadata (ACS URL, Issuer URL) entered into the Okta integration
- IT Staff group assigned to the application
- Global MFA — All Users authentication policy assigned to the application

**End-to-end SSO test result:** Confirmed. Okta SAML assertion accepted by AWS. User landed in the AWS access portal authenticated via Okta identity.

**Initial SSO error and resolution:**

First login attempt failed: *"Something went wrong — this code isn't right."*

Root cause: No matching user provisioned in AWS IAM Identity Center. Okta sends a SAML assertion carrying the user's identity, but AWS requires a corresponding local user record to map the assertion to. Resolved by creating a user in AWS IAM Identity Center with a username exactly matching the Okta UPN.

In production, SCIM automated provisioning would sync users from Okta to AWS automatically, eliminating manual user creation on the SP side. SCIM was not configured in this lab.

---

## Task 5 — Okta Workflows Automation

**Navigate:** Workflow → Workflows Console

### What Was Explored

The Workflows console was navigated fully. The template library was reviewed, the flow canvas was explored, and the **User Deactivated** trigger was selected as the automation entry point. The intended workflow would trigger on any user deactivation event and log the event with a timestamp and username, demonstrating identity lifecycle automation.

### Trial Tier Restriction

OAuth connector authentication to the Okta tenant could not be completed. The Create button remained greyed out after all required connection fields were entered. This is a trial tier restriction. Full Okta Workflows functionality requires a paid Workflows entitlement.

### Production Context

In production, Okta Workflows connects natively to the Okta tenant via a pre-authorized connection. Common enterprise automation use cases include:

- Auto-deprovisioning users from all assigned applications on account deactivation
- Sending notifications on lifecycle events such as new hire, termination, or role change
- Syncing group membership changes across connected systems
- Triggering access certification reviews when a user's role changes

---

## Task 6 — Identity Governance

### Trial Tier Status

The Identity Governance menu item is not present in the trial admin console. Per Okta documentation, Okta Identity Governance (OIG) is Generally Available on a subscription basis as a paid add-on, separate from Workforce Identity licensing.

### Features Covered by OIG

| Feature | Purpose |
|---|---|
| Access Certifications | Periodic user access reviews — approve or revoke access by campaign |
| Access Requests | Structured request and approval workflows for application access |
| Entitlement Management | Fine-grained app entitlement assignment and policy |

### Compliance Relevance

OIG features directly support SOC2 and SOX audit requirements by providing documented, auditable access review campaigns. Access Certifications automates what would otherwise be a manual, spreadsheet-driven review process, which is a common pain point for organizations undergoing compliance audits.

---

## Task 7 — System Log Audit Analysis

**Navigate:** Reports → System Log

### What Was Completed

The System Log captured **689 events** across the full lab session, covering AD import, user activation, group assignment, policy creation, application integration, SSO testing, and lifecycle management actions.

**Actions performed:**
- Filtered log by a specific lab user — confirmed admin-performed lifecycle events visible including deactivation and reactivation
- Identified failed authentication events from SSO testing — error codes and reason fields reviewed and documented
- Exported a filtered log segment as CSV
- Reviewed and documented available event fields

### Event Fields Available in System Log

| Field | Description |
|---|---|
| Event type | Categorized action identifier (e.g., `user.session.start`, `user.lifecycle.deactivate`) |
| Outcome | Success or failure with reason code |
| Actor | Who performed the action (admin, system, or end user) |
| Target | Who or what was affected |
| Client | IP address and user agent of the requesting client |
| Timestamp | UTC timestamp of the event |
| Debug context | Additional error detail for failed events |

### Support Use Cases

System Log is the primary audit and investigation tool for Okta administrators. Common Tier 1/2 support use cases include:

- Tracing a user's sign-in failures and identifying the error reason
- Confirming when a password reset was performed and by which admin
- Auditing application access events for a specific user
- Identifying unusual authentication patterns such as repeated failures or unexpected locations

**SIEM integration note:** In production, Okta System Log events would be forwarded to a SIEM via the Okta System Log API or a pre-built connector. This lab environment runs Splunk Enterprise on the same network, making Okta-to-Splunk log forwarding a logical next integration step.

---

## Key Findings Reference

The complete investigation log is documented in [`findings.md`](./findings.md). Summary of all 18 findings:

| # | Finding | Category |
|---|---|---|
| 1 | SSL not established — compensating control documented | Setup |
| 2 | Domain admin excluded from import — correct behavior | Setup |
| 3 | Attribute mapping warnings — trial tier limitation, no functional impact | Setup |
| 4 | OktaService dedicated service account — least privilege demonstrated | Setup |
| 5 | AD-sourced groups read-only in Okta — operational distinction documented | Task 1 |
| 6 | Imported users in Staged status — activation constraint documented | Task 1 |
| 7 | Deactivation "Pending User Action" — AD password authority confirmed | Task 2 |
| 8 | Password reset failure — AD policy conflict documented | Task 2 |
| 9 | Two-layer authentication policy structure documented | Task 3 |
| 10 | Rule first-match logic — ordering criticality documented | Task 3 |
| 11 | Policies staged without app assignment — intentional, resolved in Task 4 | Task 3 |
| 12 | Slack SAML — Pro plan required on SP side | Task 4 |
| 13 | GitHub SSO — Teams or Enterprise required on SP side | Task 4 |
| 14 | AWS Organizations upgrade prompt — account instance path identified | Task 4 |
| 15 | AWS SAML SSO — successful end-to-end integration confirmed | Task 4 |
| 16 | Workflows connector — trial tier restriction documented | Task 5 |
| 17 | Identity Governance — paid subscription add-on, not included in trial | Task 6 |
| 18 | System Log — 689 events captured, full audit analysis completed | Task 7 |

---

## Skills Demonstrated

- Active Directory and Okta integration via AD Agent including OU scoping, UPN format configuration, and attribute mapping
- User lifecycle management in a federated identity environment
- MFA enforcement via layered authentication policy with baseline and conditional access tiers
- SAML SSO federation including IdP and SP configuration, metadata exchange, and end-to-end login testing
- Authentication policy architecture including rule ordering, first-match logic, and break-glass exclusion patterns
- Audit log analysis including event filtering, field interpretation, and CSV export
- Systematic troubleshooting documentation with root cause analysis for every error encountered
- Trial tier vs. production feature boundary identification and documentation

---

## Repository Structure

```
okta/
├── README.md               <- This file
├── findings.md             <- Complete investigation log (18 findings)
└── screenshots/
    ├── ad-integration/     <- Agent installation, import results, attribute mapping
    ├── group-management/   <- Groups list, IT Staff creation, member assignment
    ├── user-lifecycle/     <- Deactivation, reactivation, password reset
    ├── mfa-policy/         <- Authenticators, policy configuration, rule order
    ├── sso-aws/            <- AWS SAML config, SSO login flow, access portal
    ├── system-log/         <- Log overview, filtered events, CSV export
    └── workflows/          <- Workflows canvas, trigger selection
```

---

*Part of the Enterprise IT Administration Capstone — a simulated enterprise IT environment covering identity, endpoint, remote monitoring, and ITSM operations.*
