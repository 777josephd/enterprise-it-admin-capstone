# Microsoft Entra ID Administration

**Tenant:** SterlingEnterprises  
**License tier:** Microsoft 365 Business Basic + Entra ID Free  
**Admin center:** admin.microsoft.com, entra.microsoft.com  

---

## Overview

This section documents hands-on Microsoft Entra ID administration performed on a live Microsoft 365 Business Basic tenant. Tasks cover the full scope of identity administration a Tier 2 IT administrator performs routinely: user lifecycle management, group configuration, MFA enforcement, authentication log analysis, and security feature review.

All tasks were performed via the Microsoft 365 Admin Center (admin.microsoft.com) and the Entra ID portal (entra.microsoft.com). Both interfaces access the same Entra ID backend; the admin center provides a simplified surface while entra.microsoft.com exposes the full identity management feature set.

---

## Task 1: User Lifecycle Management

### Objective
Demonstrate the complete lifecycle of a user account from provisioning through deletion and recovery.

### Account Provisioned
| Field | Value |
|---|---|
| Display name | Sterling Helpdesk |
| Username (redacted) | shelpdesk |
| License assigned | Microsoft 365 Business Basic |
| Role | User (no admin access) |
| Password behavior | Forced change on first sign-in |

### Steps Performed

**Provisioning:** User created via admin.microsoft.com > Users > Active Users > Add a user. License assigned at provisioning. Password set with forced change-on-sign-in enforced.

**Password reset:** Password reset performed post-provisioning via the user profile panel. Forced change-on-sign-in confirmed active on the reset dialog.

**Account disable:** Sign-in blocked via user profile > Block sign-in. Account set to blocked state, preventing all authentication.

**Account re-enable:** Sign-in unblocked via the same panel. Account restored to active state.

**Deletion:** User deleted via Active Users. Deletion confirmed.

**Restoration from recycle bin:** User recovered via Users > Deleted Users > Restore user. Account and license assignment restored successfully.

### Findings
- All basic lifecycle operations are available at Entra ID Free tier without P1 requirement.
- Deleted users are retained in the recycle bin for 30 days before permanent deletion. Recovery is not possible after this window without a backup.
- License assignment at provisioning time is the standard workflow; retroactive assignment via the user profile panel is also supported.

---

## Task 2: Group Management

### Objective
Create and configure both primary group types in Entra ID and document the architectural distinction between them.

### Groups Created

| Group name | Type | Purpose | Members |
|---|---|---|---|
| SterlingIT-Security | Security group | Access control, policy assignment | Sterling Helpdesk |
| SterlingIT-Team | Microsoft 365 group | Team collaboration, shared resources | Sterling Helpdesk |
| SterlingIT-Team group email | sterlingit@SterlingEnterprises | Shared mailbox address | N/A |

### Group Type Distinction

**Security groups** control access to resources. They are assigned to SharePoint sites, shared mailboxes, Conditional Access policies, and other resource permission scopes. They carry no collaboration features and provision no associated services.

**Microsoft 365 groups** are unified collaboration objects. Creating one automatically provisions a shared mailbox, a SharePoint document library, and optionally a Microsoft Teams workspace. They are the appropriate group type when a team needs shared communication and file storage alongside access control.

The correct group type for a given use case depends on whether the requirement is access control only (security group) or team collaboration with shared infrastructure (Microsoft 365 group).

### Findings
- Both group types are available at Entra ID Free tier.
- Mail-enabled security groups (which combine resource access control with a distribution list address) are available but were not configured in this task; they are documented in the Exchange Online section.
- Microsoft 365 group creation provisioned a group mailbox automatically, confirming the unified resource provisioning behavior.

---

## Task 3: MFA Enforcement via Security Defaults

### Objective
Verify and document the MFA enforcement posture of the tenant.

### Finding
Security Defaults were found **enabled** on the SterlingEnterprises tenant at the time of review. This is consistent with Microsoft's default configuration for tenants created after October 2019.

**Security Defaults enforce the following tenant-wide:**
- MFA is required for all administrator accounts on every sign-in
- MFA is required for all users when triggered by the risk-based authentication engine
- Legacy authentication protocols (basic auth, IMAP, POP3, SMTP auth) are blocked tenant-wide
- All users are enrolled in MFA registration via a 14-day grace period on first sign-in

No configuration change was required. The correct administrative action was to verify the state, document the enforced controls, and understand the implications.

<img width="1194" height="751" alt="12-SecurityDefaultsEnabled-Entra" src="https://github.com/user-attachments/assets/85902e47-4623-4574-8b8d-e42c5824a67f" />


### Security Defaults vs. Conditional Access

Security Defaults are a binary control: on or off, applying equally to all users. They are appropriate for organizations without Entra ID P1 licensing and provide baseline protection at no additional cost.

Conditional Access policies (P1 required) are granular: scoped by user, group, location, device compliance state, application, sign-in risk level, and other signals. They allow exceptions, named locations, custom enforcement schedules, and targeted policy application that Security Defaults cannot provide.

Understanding this distinction is directly relevant to M365 licensing advisory conversations in a Tier 2 support context.

---

## Task 4: Microsoft Authenticator Enrollment

### Objective
Document the end-user MFA enrollment experience as it appears when Security Defaults are active, simulating the Tier 1/2 support scenario of walking a user through first-time MFA setup.

### Process
Sign-in performed as test helpdesk account via portal.microsoft.com in an InPrivate browser session. Security Defaults triggered the "More information required" prompt on first sign-in.

**Enrollment steps:**
1. "More information required" screen displayed; user directed to click Next
2. Additional security verification screen presented; Mobile app selected as method
3. "Receive notifications for verification" option selected
4. QR code displayed; Microsoft Authenticator opened on mobile device
5. Work or school account added in Authenticator; QR code scanned
6. Test notification sent and approved; enrollment confirmed

### Support Context
This enrollment flow is one of the most common Tier 1/2 support calls in Microsoft 365 environments. Users encounter the "More information required" prompt without warning and contact the help desk for guidance. Familiarity with each screen of the enrollment sequence allows a technician to walk a user through the process efficiently without requiring remote access to the user's device.

---

## Task 5: Sign-In Log Analysis

### Objective
Use Entra ID sign-in logs to investigate authentication events, including failed sign-in analysis with error code interpretation.

### Access Path
entra.microsoft.com > Users > Sign-in logs

### Log Fields Documented
| Field | Description |
|---|---|
| User | Account that attempted authentication |
| Date/time | Timestamp of the authentication event |
| Application | Service or application the user was signing into |
| IP address | Source IP of the authentication request |
| Status | Success, Failure, or Interrupted |
| Failure reason | Plain-language description of failure cause |
| Error code | Numeric code identifying the specific failure type |

### Failed Sign-In Event

A failed authentication event was generated by attempting sign-in with incorrect credentials for shelpdesk.

| Field | Value |
|---|---|
| Error code | 50126 |
| Description | Error validating credentials due to invalid username or password |

<img width="1470" height="747" alt="15-LoginFailure" src="https://github.com/user-attachments/assets/f73a41e1-f927-4f5c-a2da-e65435d7d962" />

**Error code 50126 interpretation:** This is the standard credential failure code returned when submitted credentials do not match the stored credentials for the account. By design, the error message does not distinguish between an incorrect username and an incorrect password. This behavior is intentional: returning different messages for each case would allow an attacker to enumerate valid usernames through the authentication endpoint.

### Tier 2 Relevance
Sign-in log analysis is the primary diagnostic tool for authentication-related support tickets. A user reporting inability to sign in, an account appearing locked, or a security alert on unusual authentication activity all begin with a sign-in log pull. Interpreting error codes accurately allows a technician to diagnose the issue without requiring the user to attempt additional sign-ins.

### Findings
- Sign-in logs are available at Entra ID Free tier.
- Log retention at Free tier is 7 days. Entra ID P1 extends retention to 30 days, which is the minimum operationally sufficient window for most incident investigations. This is a concrete justification for P1 licensing in a security-conscious organization.

---

## Task 6: Audit Log Review and Conditional Access

### Part A: Audit Log Review

#### Access Path
entra.microsoft.com > Identity > Monitoring and health > Audit logs

#### Log Fields Documented
| Field | Description |
|---|---|
| Date/time | When the admin action occurred |
| Service | Which M365 service generated the event |
| Category | Type of operation (UserManagement, GroupManagement, etc.) |
| Activity | Specific action taken |
| Status | Success or failure |
| Actor | Admin account that performed the action |
| Target | Object that was acted upon |

#### Events Visible in Log
All administrative actions from the Week 1 session were visible in the audit log, including: user creation, password reset, license assignment, block sign-in, restore sign-in, user deletion, user restoration, security group creation, Microsoft 365 group creation, and member assignments.

The detail panel for the Sterling Helpdesk creation event showed a modified properties section listing all attributes set during provisioning, including display name, UPN, license assignment, and password policy.

#### Findings
- Audit logs are available at Entra ID Free tier.
- Retention at Free tier is 7 days; P1 extends to 30 days.
- Audit logs capture all administrative actions with actor, target, and modified property detail, making them the authoritative source for admin activity investigation and compliance review.

---

### Part B: Conditional Access

#### Access Path
entra.microsoft.com > Protection > Conditional Access

#### Finding
Access denied. The following message was returned:

> "You do not have access. This feature requires a subscription to P1 or P2."

Conditional Access is fully gated behind Entra ID P1. No policy creation, review, or configuration is possible at Entra ID Free or Microsoft 365 Business Basic tier.

<img width="1207" height="751" alt="19-ConditionalAccessBlock" src="https://github.com/user-attachments/assets/21751303-db28-4169-8b5d-8b3d05cf8af3" />

#### Tier Limitation Documentation
This is an expected and documentable finding. Security Defaults serve as the Free-tier substitute for Conditional Access, providing a fixed baseline enforcement posture. The inability to configure Conditional Access at the Business Basic tier is a direct driver for P1 licensing in organizations with more complex access control requirements, including named location exclusions, device compliance requirements, risk-based sign-in policies, and per-application enforcement.

Recognizing and clearly communicating this licensing boundary is a relevant skill in Tier 2 and IT advisory roles.

---

## Screenshots

See `screenshots/` folder for visual documentation of each task.

| Filename | Task | Description |
|---|---|---|
| entra-user-created.png | Task 1 | Sterling Helpdesk creation confirmation |
| entra-password-reset.png | Task 1 | Password reset dialog with forced change confirmed |
| entra-user-blocked.png | Task 1 | User profile showing sign-in blocked |
| entra-user-restored.png | Task 1 | User restored from recycle bin |
| entra-security-group.png | Task 2 | SterlingIT-Security group with member |
| entra-m365-group.png | Task 2 | SterlingIT-Team group with email and member |
| entra-security-defaults.png | Task 3 | Security Defaults enabled state |
| entra-mfa-enrollment.png | Task 4 | Authenticator enrollment confirmation (outstanding) |
| entra-signin-logs.png | Task 5 | Sign-in log list view |
| entra-signin-failure.png | Task 5 | Failed sign-in event detail with error code 50126 |
| entra-audit-logs.png | Task 6 | Audit log list showing Week 1 admin activity |
| entra-audit-detail.png | Task 6 | Audit log detail for user creation event |
| entra-conditional-access.png | Task 6 | Conditional Access access denied screen |

---

## Summary

This section demonstrates proficiency in the core identity administration tasks performed in Microsoft Entra ID at the Business Basic tier. All primary lifecycle, group, MFA, and log analysis functions were completed and documented. Tier limitations were encountered at Conditional Access and documented honestly, with clear articulation of the P1 licensing boundary and its operational implications.
