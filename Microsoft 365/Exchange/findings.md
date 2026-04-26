# Findings: Tier Limitations, Errors, and Observations
## Sterling Enterprises Tenant

**Tenant:** SterlingEnterprises  
**License tier:** Microsoft 365 Business Basic + Entra ID Free  
**Purpose:** Honest documentation of licensing boundaries, unexpected behaviors, workarounds, and investigative conclusions encountered during the M365 Administration Lab.  

This file is maintained throughout the lab. Findings are added per section as work is completed.

---

## How to Read This File

Each finding is categorized as one of the following:

- **Tier limitation:** A feature that exists in M365 but is gated behind a higher license tier and was not accessible during this lab.  
- **Observed behavior:** Something that worked differently than expected, or a default configuration worth noting.  
- **Workaround:** An alternative approach used when the primary path was unavailable.  
- **Security observation:** A behavior or default with direct security implications worth flagging.  

---

## Entra ID Findings

### Finding 1: Conditional Access requires Entra ID P1
**Category:** Tier limitation  
**Location:** entra.microsoft.com > Protection > Conditional Access  
**Observed:** Navigation to the Conditional Access section returned: "You do not have access. This feature requires a subscription to P1 or P2." No policy creation, review, or configuration was possible.  
**Implication:** Organizations on Business Basic or Entra ID Free cannot implement granular access control policies. Named location exclusions, device compliance requirements, risk-based sign-in enforcement, and per-application policy scoping all require P1. This is a direct and concrete justification for P1 licensing in security-conscious environments.  
**Workaround:** Security Defaults provide a fixed baseline MFA enforcement posture at no additional cost. They are a binary control (on or off, all users equally) rather than a granular policy engine.  

---

### Finding 2: Security Defaults enabled by default on new tenants  
**Category:** Observed behavior / Security observation  
**Location:** entra.microsoft.com > Overview > Properties > Manage security defaults  
**Observed:** Security Defaults were enabled on the SterlingEnterprises tenant without any administrative action. This is Microsoft's default configuration for tenants created after October 2019.  
**Implication:** New tenants have baseline MFA enforcement and legacy auth blocking active from day one. Administrators inheriting a new tenant should verify this state before assuming no MFA policy is in place. Disabling Security Defaults without a Conditional Access replacement leaves the tenant unprotected.  

---

### Finding 3: Sign-in log and audit log retention is 7 days at Free tier
**Category:** Tier limitation  
**Location:** entra.microsoft.com > Users > Sign-in logs; Monitoring and health > Audit logs  
**Observed:** Both sign-in logs and audit logs are accessible at Entra ID Free tier. However, retention is limited to 7 days.  
**Implication:** A 7-day retention window is operationally insufficient for most incident investigations. If a security event is not detected within 7 days, the relevant authentication evidence may no longer be available. Entra ID P1 extends retention to 30 days. Organizations with compliance or forensic requirements often integrate Entra ID logs with a SIEM (such as Splunk or Microsoft Sentinel) to achieve longer retention independent of the Entra ID tier.  

---

### Finding 4: Error code 50126 does not distinguish username from password failure
**Category:** Security observation  
**Location:** entra.microsoft.com > Users > Sign-in logs  
**Observed:** A deliberate failed sign-in attempt returned error code 50126 with the description "Error validating credentials due to invalid username or password." The message does not specify which field was incorrect.  
**Implication:** This is intentional security design. Returning separate error messages for invalid username vs. invalid password would allow username enumeration attacks against the authentication endpoint. Technicians investigating sign-in failures should not expect the error code to identify the specific cause; the correct diagnostic step is to verify the account exists, confirm it is not blocked, and ask the user to attempt sign-in with a known-good credential.  

---

### Finding 5: Microsoft 365 group creation provisions shared resources automatically
**Category:** Observed behavior  
**Location:** admin.microsoft.com > Teams and groups > Microsoft 365 tab  
**Observed:** Creating the SterlingIT-Team Microsoft 365 group automatically provisioned a shared mailbox and an associated SharePoint document library without any additional administrative steps.  
**Implication:** Administrators creating Microsoft 365 groups should be aware that they are creating multiple resources simultaneously, not just a group membership object. Deleting a Microsoft 365 group deletes the associated mailbox and SharePoint library as well. This is distinct from security group behavior, where deletion removes only the group membership object.  

---

### Finding 6: Deleted users are retained for 30 days
**Category:** Observed behavior  
**Location:** admin.microsoft.com > Users > Deleted users  
**Observed:** Sterling Helpdesk was successfully restored from the deleted users recycle bin after deletion. The retention window for deleted users in Microsoft 365 is 30 days.  
**Implication:** Accidental user deletions are recoverable within 30 days with no data loss. After 30 days, the account and its associated data are permanently deleted. This is relevant for offboarding procedures: if a user is deleted prematurely, a technician has a 30-day recovery window before escalation is required.  

---

## Exchange Online Findings

### Finding 7: Exchange Online mailbox not provisioned despite active license assignment
**Category:** Observed behavior / Workaround  
**Location:** Exchange Admin Center > Recipients > Shared mailboxes > Delegation; admin.cloud.microsoft.com > Users > Active Users  
**Observed:** The shelpdesk account did not appear as an available delegate target in the Exchange Admin Center Send As picker despite having a Microsoft 365 Business Basic license assigned in Entra ID. The account had been provisioned approximately 12 hours prior, well beyond the normal Exchange Online propagation window of 15 to 30 minutes. Investigation confirmed the Exchange Online mailbox had not been provisioned: shelpdesk did not appear in Exchange Admin Center > Recipients > Mailboxes, indicating the license assignment had not triggered mailbox creation.
**Resolution:** The Business Basic license was manually removed and reassigned via admin.cloud.microsoft.com > Users > Active Users. License reapplication triggered Exchange Online mailbox provisioning. The shelpdesk account appeared in the Exchange recipient index and became available as a Send As delegate target within minutes of reapplication.  
**Implication:** License assignment in Entra ID does not guarantee immediate or automatic provisioning of all associated services. Exchange Online mailbox provisioning can fail silently: the license appears assigned in the admin center but the mailbox is never created. When a newly licensed user is not visible in Exchange as expected, the correct diagnostic step is to verify the account appears in Exchange Admin Center > Recipients > Mailboxes. If absent, license removal and reapplication is the standard remediation. This is a realistic Tier 2 troubleshooting scenario.  

---

### Finding 8: Exchange Admin Center prompts administrators toward Microsoft 365 Groups during distribution list creation
**Category:** Observed behavior  
**Location:** Exchange Admin Center > Recipients > Groups > Add a group > Distribution list  
**Observed:** During distribution list creation, Exchange Admin Center displayed the prompt: "Why not create a Microsoft 365 Groups instead? Microsoft 365 Groups have more collaboration tools, like shared calendars, files, and notes." The prompt does not block creation and can be dismissed.  
**Implication:** Microsoft's platform architecture favors Microsoft 365 unified groups over standalone distribution lists. The nudge is informational; administrators retain the ability to create distribution lists when the use case requires mail routing only without unified resource provisioning.  Understanding when to dismiss this prompt and why is a relevant administrative judgment call.  

---

### Finding 9: Message trace results for recent messages may have a population delay
**Category:** Observed behavior  
**Location:** Exchange Admin Center > Mail flow > Message trace  
**Observed:** Running a message trace immediately after sending test emails returned no results. Waiting 5 to 10 minutes before running the trace returned results successfully.  
**Implication:** Message trace is near-real-time but not instantaneous. In a support context, a technician running a trace on a very recently sent message should wait before concluding the message was not received. A no-results trace on a message sent within the last few minutes is not a reliable indicator of delivery failure.  

---

### Finding 10: Mail flow rule execution confirmed via message trace
**Category:** Observed behavior  
**Location:** Exchange Admin Center > Mail flow > Message trace > detail panel  
**Observed:** The message trace detail panel for outbound test messages showed two transport rule events, both identified as Outbound Email Disclaimer with matching rule IDs, confirming the mail flow rule configured in Task 3 is triggering correctly on outbound internal messages.  
**Implication:** Message trace is the reliable method for confirming mail flow rule execution on specific messages. Transport rule events appear inline in the trace detail panel alongside delivery hop information, making trace the single tool for both delivery confirmation and rule trigger verification. This is the correct approach for troubleshooting reports of missing disclaimers or unexpectedly triggered rules.  

---

### Finding 11: Mailbox forwarding deliver-to-both setting defaults to disabled
**Category:** Security observation  
**Location:** Exchange Admin Center > Recipients > Mailboxes > Mail flow settings > Manage email forwarding  
**Observed:** When configuring mailbox forwarding, the deliver-to-both-forwarding-address-and-mailbox setting defaults to disabled. When disabled, forwarded messages are not retained in the original mailbox.  
**Implication:** Administrators configuring forwarding must explicitly enable deliver-to-both to ensure mailbox retention. Leaving it disabled creates gaps in audit trails and may violate organizational retention policies. Additionally, unauthorized forwarding rules with deliver-to-both disabled are a common indicator of account compromise: an attacker configuring silent forwarding to an external address would typically disable retain-in-mailbox to reduce visibility. Absence of copies in the original mailbox for an account with forwarding configured should be treated as a suspicious finding during incident investigation.  

---

## SharePoint Online Findings

*To be populated during Week 3.*

---

## Cross-Cutting Observations

### M365 Business Basic licensing boundary summary
The following features were confirmed unavailable at Business Basic / Entra ID Free tier during this lab:

| Feature | Requires | Notes |
|---|---|---|
| Conditional Access policies | Entra ID P1 | Fully gated; access denied at navigation |
| Sign-in log retention beyond 7 days | Entra ID P1 | Logs exist but truncated at 7 days |
| Audit log retention beyond 7 days | Entra ID P1 | Same retention limitation as sign-in logs |
| Entra ID Identity Protection | Entra ID P2 | Risk-based policies require P2 |
| Entra ID Identity Governance | Entra ID P2 | Access reviews, entitlement management |

*Additional entries will be added as SharePoint findings are documented.*

---

*This file will be finalized at the conclusion of Week 3 with all findings consolidated.*
