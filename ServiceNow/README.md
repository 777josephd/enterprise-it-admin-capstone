# ServiceNow ITSM Administration
**Platform:** ServiceNow Personal Developer Instance (PDI)
**Release:** Australia
**Section:** enterprise-it-admin-capstone/ServiceNow/

---

## Overview

This section documents hands-on administration of a ServiceNow PDI configured to simulate an enterprise IT service management environment. The work covers the full scope of Tier 2 ITSM responsibilities: user and department administration, incident ticket lifecycle management, knowledge base authoring, and service catalog configuration. All incidents reference real events generated across the broader capstone lab environment, integrating ServiceNow into the full tool stack narrative.

---

## Environment Configuration

### Users Provisioned

| Name | Title | Department |
|---|---|---|
| Adam Adamson | IT Support Specialist | Information Technology |
| Alice Allison | End User | General Staff |
| Bob Bobson | End User | General Staff |
| Chris Chrisson | End User | General Staff |

### Departments Created

| Name | Description |
|---|---|
| Information Technology | IT support and infrastructure operations |
| General Staff | End user population |

Users and departments were created to maintain narrative consistency with the Active Directory environment and the broader capstone tool stack. All user accounts follow the naming conventions used across the lab.

---

## Incident Management

Five incidents were created and resolved, each referencing a real event from another tool in the capstone environment. This demonstrates the ability to document, categorize, prioritize, and resolve support tickets across a realistic enterprise tool stack.

### Incident List

| Number | Short Description | Caller | Category | Priority | Source Tool |
|---|---|---|---|---|---|
| INC0010001 | User unable to complete Okta MFA enrollment | Chris Chrisson | Inquiry / Help | 3 - Moderate | Okta |
| INC0010002 | User mailbox not accessible despite active M365 license | Bob Bobson | Software | 3 - Moderate | Exchange Online |
| INC0010003 | Endpoint reporting non-compliant status in MDM console | Alice Allison | Software | 3 - Moderate | Miradore MDM |
| INC0010004 | Atera RMM agent reporting offline on monitored endpoint | Adam Adamson | Hardware | 3 - Moderate | Atera RMM |
| INC0010005 | SharePoint tenant external sharing configured to allow unauthenticated access | Adam Adamson | Software | 2 - High | SharePoint Online |

### Incident Detail — INC0010001: Okta MFA Enrollment Failure

**Caller:** Chris Chrisson
**Category:** Inquiry / Help
**Priority:** 3 - Moderate
**State:** Resolved

**Description:** End user reported inability to enroll Okta Verify as MFA factor during initial setup. Account verified active in Okta admin console. User guided through re-enrollment via Okta self-service portal. MFA factor confirmed enrolled and authentication policy satisfied.

**Resolution:** Okta Verify enrollment completed successfully after guided re-enrollment walkthrough.

---

### Incident Detail — INC0010002: Exchange Online Mailbox Not Provisioned

**Caller:** Bob Bobson
**Category:** Software
**Priority:** 3 - Moderate
**State:** Resolved

**Description:** End user reported inability to access Exchange Online mailbox following M365 license assignment. Investigation confirmed license was active but mailbox had not been provisioned. Issue resolved by removing and reapplying the license, triggering mailbox provisioning. Mailbox confirmed accessible following reprovisioning.

**Resolution:** Exchange Online mailbox provisioned successfully after license removal and reapplication. Root cause attributed to provisioning delay on initial license assignment.

*This incident mirrors a real observed behavior documented in the M365 Administration Lab (Finding 7).*

---

### Incident Detail — INC0010003: Miradore MDM Compliance Policy Failure

**Caller:** Alice Allison
**Category:** Software
**Priority:** 3 - Moderate
**State:** Resolved

**Description:** Windows 11 Pro endpoint flagged as non-compliant in Miradore MDM console. Investigation confirmed compliance policy was applied but custom Policy CSP registry key had not propagated correctly. Registry value verified manually on endpoint. MDM sync forced and compliance status confirmed restored.

**Resolution:** Compliance status restored following forced MDM sync. Registry key propagation confirmed via manual verification on managed endpoint.

---

### Incident Detail — INC0010004: Atera RMM Agent Offline

**Caller:** Adam Adamson
**Category:** Hardware
**Priority:** 3 - Moderate
**State:** Resolved

**Description:** Threshold alert triggered in Atera RMM console indicating monitored endpoint had gone offline. Remote session attempted and failed. Endpoint verified powered on via Proxmox hypervisor console. Atera agent service confirmed stopped. Service restarted manually and agent connectivity restored. Alert cleared following reconnection.

**Resolution:** Atera agent service restarted successfully. Endpoint connectivity restored and confirmed active in RMM console. Alert cleared.

---

### Incident Detail — INC0010005: SharePoint External Sharing Misconfiguration

**Caller:** Adam Adamson
**Category:** Software
**Priority:** 2 - High
**State:** Resolved

**Description:** Routine SharePoint Online tenant review identified external sharing policy set to maximum permissiveness, allowing unauthenticated guest access via shareable links. Policy reviewed against organizational security requirements. External sharing scope restricted to authenticated guests only. Finding documented as security observation for compliance review.

**Resolution:** External sharing policy updated to restrict unauthenticated access. Tenant sharing configuration confirmed aligned with least-privilege principles following policy change.

*Priority set to 2 - High reflecting the security classification of this finding. This incident references a real observed behavior documented in the M365 Administration Lab (Finding 16).*

---

## Knowledge Base

### Article: How to Reset Your Okta Password and Re-enroll MFA

**Knowledge Base:** IT
**Template:** Standard
**State:** Published

**Purpose:** Documents the end-user procedure for resetting an Okta password and re-enrolling an MFA factor. Authored to support self-service resolution of the most common authentication support request type, reducing repeat incident volume.

**Procedure:**
1. Navigate to the Okta login page and click Forgot Password
2. Enter your username in UPN format and click Reset via Email
3. Open the password reset email and click the reset link
4. Set a new password meeting the minimum requirements: 8 characters, upper and lowercase, number required
5. After password reset, log in and navigate to your profile settings
6. Under Security Methods, click Set Up to re-enroll Okta Verify
7. Follow the on-screen prompts to scan the QR code with the Okta Verify mobile app
8. Confirm enrollment by completing an MFA challenge

**Notes:** If the password reset email is not received within five minutes, check the spam folder. If the issue persists, contact the IT helpdesk to verify the account email address on file.

---

## Service Catalog

### Catalog Item: Shared Mailbox Access Request

**Catalog:** Service Catalog
**Type:** Standard
**State:** Published

**Purpose:** Provides end users with a structured request form for shared mailbox access in Exchange Online. Submissions are reviewed and fulfilled by the IT helpdesk team. This item mirrors a real administrative task performed during the M365 Administration Lab, where shared mailbox permissions were delegated via Full Access and Send As assignments.

**Form Questions:**

| Question | Type | Mandatory |
|---|---|---|
| Mailbox Name or Email Address | Single-line text | Yes |
| Business Justification | Single-line text | Yes |

The business justification field enforces a lightweight approval rationale, consistent with least-privilege access control principles applied throughout the lab.

---

## Screenshots

| File | Description |
|---|---|
| [users-list.png](./screenshots/users-list.png) | User Administration list showing all four provisioned users |
| [departments-list.png](./screenshots/departments-list.png) | Department list showing Information Technology and General Staff |
| [incidents-list.png](./screenshots/incidents-list.png) | Full incident list showing all five resolved tickets |
| [inc-okta-mfa.png](./screenshots/inc-okta-mfa.png) | INC0010001: Okta MFA enrollment failure, resolved state |
| [inc-exchange-mailbox.png](./screenshots/inc-exchange-mailbox.png) | INC0010002: Exchange Online mailbox not provisioned, resolved state |
| [inc-miradore-compliance.png](./screenshots/inc-miradore-compliance.png) | INC0010003: Miradore MDM compliance policy failure, resolved state |
| [inc-atera-offline.png](./screenshots/inc-atera-offline.png) | INC0010004: Atera RMM agent offline, resolved state |
| [inc-sharepoint-sharing.png](./screenshots/inc-sharepoint-sharing.png) | INC0010005: SharePoint external sharing misconfiguration, resolved state |
| [kb-okta-password-reset.png](./screenshots/kb-okta-password-reset.png) | Published Knowledge Base article: Okta password reset and MFA re-enrollment |
| [catalog-item-admin.png](./screenshots/catalog-item-admin.png) | Shared Mailbox Access Request catalog item, admin configuration view |
| [catalog-item-form.png](./screenshots/catalog-item-form.png) | Shared Mailbox Access Request catalog item, end-user form view |
