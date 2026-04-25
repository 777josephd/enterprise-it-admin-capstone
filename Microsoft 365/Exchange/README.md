# Exchange Online Administration
## Sterling Enterprises Tenant

**Tenant:** SterlingEnterprises  
**License tier:** Microsoft 365 Business Basic  
**Admin center:** admin.exchange.microsoft.com  

---

## Overview

This section documents hands-on Exchange Online administration performed on a live Microsoft 365 Business Basic tenant. Tasks cover the core email administration responsibilities of a Tier 2 IT administrator: shared mailbox provisioning and permission delegation, distribution group management, mail flow rule configuration, message trace investigation, and per-user mailbox settings including auto-reply and forwarding.

All tasks were performed via the Exchange Admin Center (admin.exchange.microsoft.com).

---

## Task 1: Shared Mailbox Creation and Permission Delegation

### Objective
Provision a shared mailbox for help desk use and configure Full Access and Send As delegation for a technician account.

### Shared Mailbox Provisioned

| Field | Value |
|---|---|
| Display name | Sterling Helpdesk Shared |
| Email address (redacted) | helpdesk@SterlingEnterprises |
| Alias | helpdesk |

### Delegation Configured

| Permission type | Delegated to |
|---|---|
| Full Access | Admin account, shelpdesk (redacted) |
| Send As | shelpdesk (redacted) |

### Steps Performed

**Mailbox creation:** Shared mailbox created via Exchange Admin Center > Recipients > Shared mailboxes > Add a shared mailbox. Display name, email address, and alias configured at creation.

**Full Access delegation:** Full Access permission granted to both the admin account and the shelpdesk account via the Delegation section of the shared mailbox. Full Access allows the delegated user to open the mailbox and read its contents.

**Send As delegation:** Send As permission granted to the shelpdesk account. Send As allows the delegated user to send email that appears to originate from the shared mailbox address rather than their personal account. This is the standard configuration for a help desk shared mailbox: technicians send responses from the team address, presenting a unified identity to end users.

<img width="1536" height="793" alt="exchange-send-as-delegation" src="https://github.com/user-attachments/assets/7e860158-cce0-4d79-81dd-d0c92174a6da" />


### Permission Distinction

Full Access and Send As are independent permissions serving different functions. Full Access grants read access to mailbox contents; it does not authorize sending on behalf of the mailbox. Send As authorizes sending from the mailbox address; it does not grant read access to contents. In a help desk configuration both are typically granted together. A user with Full Access only will send replies from their personal address; a user with Send As only cannot read the mailbox they are sending from.

### Findings
- Shared mailbox creation is available at Business Basic tier without additional licensing.
- Send As delegation for a newly provisioned user was not immediately available in the Exchange Admin Center picker. The shelpdesk account did not appear as a delegate target despite having a Business Basic license assigned. Root cause was identified as a mailbox provisioning failure: the Exchange Online mailbox had not been provisioned despite the license assignment appearing active in Entra ID. Resolution required manually removing and reassigning the Business Basic license via admin.cloud.microsoft.com > Users > Active Users, which triggered Exchange Online mailbox provisioning. The account became available as a delegation target within minutes of the license reapplication. See findings.md for full detail.

---

## Task 2: Distribution Group Creation and Membership

### Objective
Create a managed distribution list for IT department communications and configure membership and access controls to reflect enterprise standards.

### Group Provisioned

| Field | Value |
|---|---|
| Name | Sterling IT Distribution |
| Group email (redacted) | sterlingit-dl@SterlingEnterprises |
| Type | Distribution list |
| Owner | Admin account |
| Members | Admin account, shelpdesk (redacted) |

### Settings Configured

| Setting | Value | Rationale |
|---|---|---|
| External senders | Disabled | Prevents spam and phishing to internal lists |
| Joining | Closed | IT controls membership; self-enrollment not permitted |
| Leaving | Closed | Membership changes processed by IT administrators |

### Steps Performed

Distribution group created via Exchange Admin Center > Recipients > Groups > Add a group > Distribution list. Owner assigned during wizard. Members added via the Members tab post-creation. Access settings configured on the Settings page of the creation wizard.

### Exchange Online Prompt: Microsoft 365 Groups

During distribution list creation, Exchange Admin Center displayed the following prompt: "Why not create a Microsoft 365 Groups instead? Microsoft 365 Groups have more collaboration tools, like shared calendars, files, and notes."

This prompt was dismissed. A distribution list was the correct choice for this use case: the requirement is email routing only, without the overhead of provisioning a SharePoint library, Teams workspace, or shared calendar. Understanding when to use a distribution list versus a Microsoft 365 group despite platform nudges toward the latter is a relevant administrative judgment call.

### Group Type Reference

| Group type | Mail routing | SharePoint library | Teams workspace | Access control |
|---|---|---|---|---|
| Distribution list | Yes | No | No | No |
| Microsoft 365 group | Yes | Yes | Yes | No |
| Security group | No | No | No | Yes |
| Mail-enabled security group | Yes | No | No | Yes |

### Findings
- Distribution lists are available at Business Basic tier.
- Exchange Admin Center actively prompts administrators to use Microsoft 365 Groups during distribution list creation. The prompt does not block creation; it can be dismissed. This reflects Microsoft's preference for unified group architecture but does not change the administrative use case for simple mail routing objects.
- Closed join and leave settings reflect the principle of least privilege applied to mail routing: group membership is administrator-controlled rather than self-managed.

---

## Task 3: Mail Flow Rule Configuration

### Objective
Create a transport rule that automatically appends a confidentiality disclaimer to all outbound email originating from inside the organization.

### Rule Configured

| Field | Value |
|---|---|
| Rule name | Outbound Email Disclaimer |
| Condition | Sender is located: Inside the organization |
| Action | Append disclaimer |
| Fallback action | Wrap |
| Rule mode | Enforce |
| Priority | 0 |
| Severity | Low |
| Match sender address | Header |

### Disclaimer Text

```
This email and any attachments are confidential and intended solely for the use of the named recipient. If you have received this email in error, please notify the sender immediately and delete this message.
```

### Configuration Decisions

**Fallback action: Wrap.** If Exchange Online cannot append the disclaimer directly to a message (for example, due to message encryption or a protected format), the Wrap fallback encapsulates the original message inside a new message with the disclaimer applied to the outer wrapper. This ensures the disclaimer is always delivered without blocking the original message.

**Rule mode: Enforce.** Test mode logs rule matches without taking action, useful during rule development. Enforce applies the action to all matching messages immediately. Enforce is the correct mode for a production disclaimer rule.

**Severity: Low.** Severity tags rule match events in audit logs and reports for filtering and prioritization. A disclaimer rule is informational and non-blocking; Low is the appropriate classification. Medium and High are appropriate for rules flagging sensitive content or policy violations.

**Match sender address: Header.** Header matches against the From address visible in the email client. Envelope matches against the technical SMTP envelope address used during transmission, which can differ from the header address in forwarding or relay scenarios. Header is the correct choice for a straightforward outbound disclaimer applying to all internal senders.

### Findings
- Mail flow rules are available at Business Basic tier.
- Rule execution was confirmed via message trace in Task 4: the Outbound Email Disclaimer rule appeared as a triggered transport rule event in the trace detail panel for outbound test messages, confirming the rule is active and enforcing correctly.

<img width="1536" height="800" alt="exchange-mail-flow-rule" src="https://github.com/user-attachments/assets/2d91a709-bd8a-4a93-9edd-91b7dff98eaf" />


---

## Task 4: Message Trace

### Objective
Run a message trace to investigate email delivery activity and document the trace fields and their Tier 2 support relevance.

### Access Path
Exchange Admin Center > Mail flow > Message trace > Start a trace

### Search Parameters Used

| Parameter | Value |
|---|---|
| Senders | Admin account |
| Recipients | All (blank) |
| Time range | Last 24 hours |
| Report type | Summary report |
| Detailed options | All blank |

### Trace Results

Message trace returned results for test emails sent between lab accounts during the Week 2 session.

### Fields Documented

| Field | Description |
|---|---|
| Message ID | Unique identifier for the email; used to track a specific message across mail systems and in vendor support cases |
| Sender | From address of the message |
| Recipient | To address of the message |
| Subject | Email subject line |
| Status | Delivered, Failed, Pending, Filtered, or Expanded |
| Received | Timestamp of when Exchange Online received the message |
| To IP | Destination mail server IP address |
| From IP | Source mail server IP address |

### Mail Flow Rule Confirmation

The message trace detail panel for outbound test messages showed two transport rule events, both identified as **Outbound Email Disclaimer**, with matching rule IDs. This confirms the mail flow rule configured in Task 3 is triggering correctly on outbound internal messages and provides direct corroborating evidence across both tasks.

<img width="1536" height="795" alt="exchange-message-trace-detail" src="https://github.com/user-attachments/assets/4e3aec6e-465d-442b-afb5-6d86b8afa3cd" />


### Tier 2 Use Cases

Message trace is the primary diagnostic tool for email delivery support tickets. Common scenarios:

- A user reports an expected email never arrived: run a trace on the recipient to confirm delivery status and identify any filtering or rejection events
- An email sent to a distribution group was not received by all members: run a trace on the group address to confirm expansion and per-recipient delivery
- An outbound message was flagged or blocked by a mail flow rule: the trace detail panel identifies which rule triggered and what action was taken
- A user suspects their email is being forwarded without their knowledge: run a trace on the mailbox to identify unexpected forwarding events

### Findings
- Message trace is available at Business Basic tier.
- Trace results for very recent messages may take 5 to 10 minutes to populate after the message is sent. Running a trace immediately after sending may return no results; waiting before searching is standard practice.
- Transport rule events appear inline in the message trace detail panel, making trace the single tool for both delivery confirmation and rule trigger verification.

---

## Task 5: Out-of-Office Auto-Reply and Mailbox Forwarding

### Objective
Configure automatic reply and email forwarding on a user mailbox via the Exchange Admin Center, simulating the administrative path used when acting on a help desk ticket on behalf of a user.

### Access Path
Exchange Admin Center > Recipients > Mailboxes > Sterling Helpdesk

---

### Part A: Out-of-Office Auto-Reply

**Configuration applied to:** Sterling Helpdesk mailbox

| Setting | Value |
|---|---|
| Automatic replies | Enabled |
| Internal reply | This is an automated reply. Sterling Helpdesk is currently out of office and will respond upon return. For immediate assistance, please contact the IT help desk at helpdesk@SterlingEnterprises |
| External reply | Enabled; same message text |

Auto-reply configured via the Automatic replies tab of the mailbox settings panel in the Exchange Admin Center.

---

### Part B: Mailbox Forwarding

**Configuration applied to:** Sterling Helpdesk mailbox

| Setting | Value |
|---|---|
| Forward all email to | Admin account |
| Deliver to both forwarding address and mailbox | Enabled |

The deliver-to-both setting is critical in production configurations. Leaving it disabled means the original mailbox retains no copy of forwarded messages, creating gaps in audit trails and potential compliance issues. Enabling it ensures messages are retained in the original mailbox and forwarded simultaneously.

### Tier 2 Relevance

Auto-reply and forwarding configuration are among the most common administrative tasks during employee offboarding, leave of absence, and role transitions. Configuring both via the Exchange Admin Center allows a technician to fulfill these requests without requiring the user to access their own mailbox settings.

### Security Dimension: Forwarding as an Indicator of Compromise

Unauthorized mailbox forwarding rules are a well-documented indicator of account compromise. An attacker who gains access to a user's mailbox frequently configures a silent forwarding rule to an external address, allowing ongoing mail collection after the initial access event is remediated. A Tier 2 technician investigating a suspected compromised account should check for unexpected forwarding rules as a standard step in the triage process, alongside password reset and session revocation.

### Findings
- Automatic reply configuration via the admin center is available at Business Basic tier.
- Mailbox forwarding configuration via the admin center is available at Business Basic tier.
- The deliver-to-both setting defaults to disabled; administrators must enable it explicitly to ensure mailbox retention during forwarding.

---

## Screenshots

See [screenshots/](./screenshots) folder for visual documentation of each task.

| Filename | Task | Description |
|---|---|---|
| exchange-shared-mailbox-created.png | Task 1 | Shared mailbox creation confirmation |
| exchange-full-access-delegation.png | Task 1 | Full Access delegation panel showing both accounts |
| exchange-distribution-group-created.png | Task 2 | Distribution group creation confirmation |
| exchange-distribution-members.png | Task 2 | Members list showing both accounts |
| exchange-mail-flow-rule.png | Task 3 | Completed rule configuration before saving |
| exchange-rule-enabled.png | Task 3 | Rules list showing Outbound Email Disclaimer enabled |
| exchange-message-trace-results.png | Task 4 | Message trace results list |
| exchange-message-trace-detail.png | Task 4 | Trace detail panel showing transport rule events |
| exchange-auto-reply.png | Task 5 | Automatic replies configuration panel |
| exchange-forwarding.png | Task 5 | Forwarding configuration showing destination and deliver-to-both enabled |

---

## Summary

This section demonstrates proficiency in the core Exchange Online administration tasks performed at the Business Basic tier. Shared mailbox provisioning, permission delegation, distribution group management, mail flow rule enforcement, message trace investigation, and per-user mailbox configuration were all completed and documented. A mailbox provisioning failure was encountered, diagnosed, and resolved via license reapplication, providing a realistic troubleshooting finding. Mail flow rule execution was confirmed independently through message trace, producing corroborating evidence across two tasks.
