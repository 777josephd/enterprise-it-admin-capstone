# Enterprise IT Administration Capstone — Okta Findings & Investigation Log

**Project:** Enterprise IT Administration Capstone — Identity, Endpoint, and Support Operations  
**Tool:** Okta Workforce Identity (30-day trial)  
**Domain:** soc-lab.local  
**Okta Tenant:** trial-id.okta.com    

---

## Setup & Installation

### Finding 1 — SSL Not Established for AD Agent
**Observed:** SSL connection between the Okta AD Agent (installed on domain-joined Windows Server 2022 at 10.0.10.103) and the Okta tenant could not be established in the lab environment.
**Resolution:** Proceeded without SSL with documented compensating rationale.
**Compensating control:** All AD Agent traffic remains within internal VLAN 10 (10.0.10.0/24), isolated from external networks by firewall rules. Risk is confined to the internal lab network.
**Production note:** In a production environment, SSL/TLS would be required for AD Agent communication. Certificate configuration would be completed using an internal CA or a publicly trusted certificate.

---

### Finding 2 — MERLIN Domain Admin Account Not Imported
**Observed:** The SOC-LAB\MERLIN domain administrator account was not imported during the full AD user import.
**Root cause:** Expected and correct behavior. Okta's AD Agent is configured to scope imports to standard user OUs. Privileged administrative accounts should not be federated to cloud IAM providers — doing so exposes privileged credentials to cloud authentication flows and violates the principle of least privilege.
**Production note:** Domain admin accounts are intentionally excluded from cloud IAM federation in enterprise environments. Break-glass admin accounts are managed separately with additional safeguards.

---

### Finding 3 — distinguishedName / objectGUID Attribute Mapping Warnings
**Observed:** Warnings encountered during attribute mapping configuration for distinguishedName (dn) and objectGUID (externalId).
**Root cause:** Investigated and confirmed these attributes are managed internally by the AD Agent in the trial tier and are not exposed for manual mapping in the Okta UI. These are used internally by Okta to maintain the AD-to-Okta user relationship.
**Impact:** None — user import and sync functioned correctly despite the warnings. All users and 20 groups imported successfully.
**Production note:** In a full Workforce Identity license, additional attribute mapping options are available. Trial tier restricts some advanced attribute configuration.

---

### Finding 4 — OktaService Dedicated Service Account
**Observed:** Okta AD Agent installation created a dedicated OktaService service account in Active Directory.
**Significance:** Demonstrates principle of least privilege — the agent operates under a dedicated, scoped service account rather than a domain admin credential. OktaService account was noted for addition to the Pre-Windows 2000 Compatible Access group post-installation per Okta documentation requirements.
**Production note:** Service account permissions should be reviewed and scoped to the minimum required for AD read operations and password writeback. Privileged access to the service account should be audited regularly.

---

## Task 1 — Group Management

### Finding 5 — AD-Sourced Groups vs. Okta-Native Groups
**Observed:** AD-sourced groups imported via the AD Agent are read-only within the Okta console. Membership is controlled by Active Directory and synced into Okta — changes cannot be made from the Okta side.
**Action taken:** Created Okta-native group "IT Staff" — managed entirely within Okta, independent of AD. Used for application assignment, authentication policy scoping, and Workflows.
**Production note:** This distinction is operationally significant. Support staff need to understand which groups are AD-authoritative (changes made in ADUC) vs. Okta-native (changes made in Okta console) to route administration tasks correctly.

---

### Finding 6 — Imported AD Users in Staged Status
**Observed:** All four imported AD lab users (Adam Adamson, Alice Allison, Bob Bobson, Chris Chrisson) landed in Staged status following the AD import. The admin enrollment account was Active.
**Root cause:** Expected behavior. Staged is Okta's initial state for AD-sourced users who have not completed the Okta activation flow. Staged users exist in the Okta directory but cannot authenticate to Okta or assigned applications.
**Resolution:** All four users admin-activated via Okta console. Activation was performed without sending activation emails due to absence of a mail server in the lab environment.
**Production note:** In production, users would self-activate via an emailed activation link. Admin-initiated activation without email is a lab constraint — not standard production workflow.

---

## Task 2 — User Lifecycle Management

### Finding 7 — Alice Allison Deactivation: "Pending User Action"
**Observed:** Deactivating Alice Allison's account resulted in a "Pending User Action — Password Selection Required" status rather than immediate deactivation.
**Root cause:** Okta routed through a reactivation confirmation flow requiring admin-set temporary password before deactivation could complete. Account entered "Password Expired" state following temporary password assignment.
**Resolution:** Admin set a temporary password via Okta, producing the expected "Password Expired" state. Password then reset directly in Active Directory Users and Computers (ADUC) — change propagated back to Okta via the AD Agent, confirming AD as the authoritative identity source for password management.
**Production note:** In a federated AD-Okta environment, password management operations resolve through AD, not the Okta console. Help desk staff supporting this environment need ADUC access in addition to Okta admin access for complete lifecycle management.

---

### Finding 8 — Bob Bobson Password Reset Failure
**Observed:** Okta-initiated password reset for Bob Bobson failed with error: "Password could not be changed, please contact your administrator."
**Investigation:** Bob's AD account had the Password Never Expires flag set, consistent with lab configuration. Flag was removed but error persisted.
**Likely root causes (in order of probability):**
1. AD minimum password age policy blocking immediate reset after flag change — requires waiting out the minimum age window or resetting directly in ADUC
2. Trial tier limitation — password writeback to AD is a fully supported feature in production Okta Workforce Identity but may behave inconsistently in trial tier configurations
**Production note:** In a production environment with full Okta licensing, password writeback to AD is a standard supported feature. The resolution path for a similar issue would be: verify AD password policy flags, confirm AD Agent service account has write permissions to the user object, and test writeback with a user account that has no policy restrictions applied.

---

## Task 3 — MFA Enforcement

### Finding 9 — Authentication Policy Structure in Trial Tier
**Observed:** Trial tier presents two distinct policy categories rather than a unified authentication policy interface:
- **App Sign-On Policy** — controls authentication requirements for assigned applications
- **Okta Account Management Policy** — controls authentication for account self-service actions (password reset, MFA enrollment, profile changes)
**Significance:** Okta separates application access authentication from account management authentication as distinct policy layers. Both were reviewed and configured during the lab. This separation is an important operational distinction — tightening app sign-on policy does not automatically affect self-service account actions.

---

### Finding 10 — Rule Ordering and First-Match Logic
**Observed:** Okta authentication policy rules are evaluated top-to-bottom with first-match logic. The first rule whose conditions match the authenticating user is applied — subsequent rules are not evaluated.
**Action taken:** Admin account exclusion implemented as a dedicated rule ("Admin Exclusion — No Prompt") positioned above the MFA enforcement rule in both policies. This ensures the admin account matches the exclusion rule before reaching the MFA requirement.
**Production note:** Incorrect rule ordering causes exclusions to be silently bypassed — the MFA rule would match first and the exclusion would never be evaluated. Rule order is a critical operational consideration when managing authentication policies, particularly when adding break-glass or admin exclusions.

---

### Finding 11 — Policies Staged Without Application Assignment
**Observed:** Both configured authentication policies (Global MFA — All Users, IT Staff — Elevated Auth) showed "No apps assigned" in the Applies To column after creation.
**Root cause:** Expected — Okta App Sign-On policies are enforced at the application level and must be explicitly assigned to each application. Policies are not globally active until assigned.
**Resolution:** Policies staged intentionally — application assignment completed during Task 4 when the AWS IAM Identity Center application was added.
**Production note:** Policy staging prior to application creation mirrors enterprise workflow where authentication policies are defined and reviewed before applications are onboarded. This prevents ad-hoc policy creation under pressure during application deployment.

---

## Task 4 — Single Sign-On Application Integration

### Finding 12 — Slack SAML SSO Requires Paid Plan
**Observed:** Slack SAML/SSO configuration is locked behind Slack Pro plan or higher. The free Slack tier does not expose SAML settings in the admin panel.
**Action taken:** Okta catalog integration for Slack was added and fully configured on the Okta side — domain entered, sign-on tab reviewed, SWA vs. SAML distinction documented. Slack-side SAML configuration could not be completed without a paid Slack plan.
**SWA note:** Secure Web Authentication (SWA) is available as a fallback — Okta injects credentials rather than performing true federated SSO. SWA does not constitute federated identity and was not documented as SSO.
**Production note:** In an enterprise environment, Slack SSO would require at minimum a Slack Pro plan. SAML configuration on the Slack side involves uploading Okta's IdP metadata and certificate, configuring the ACS URL and Entity ID, and testing the SP-initiated login flow.

---

### Finding 13 — GitHub SSO Requires Paid Organization Plan
**Observed:** All GitHub SSO/SAML federation options (GitHub, GitHub Enterprise Server, GitHub Enterprise Cloud) require GitHub Teams or Enterprise plan at the organization level.
**Action taken:** Multiple GitHub catalog entries reviewed and evaluated. No free path exists for true federated SSO on the GitHub side.
**Production note:** GitHub SAML SSO at the organization level requires GitHub Teams ($4/user/month minimum) or GitHub Enterprise. Individual account OIDC does not support IdP-initiated SSO.

---

### Finding 14 — AWS IAM Identity Center: Organizations Upgrade Prompt
**Observed:** Initial AWS IAM Identity Center enablement prompted for AWS Organizations, which would force an account upgrade from free tier to pay-as-you-go with immediate expiration of free tier credits.
**Resolution:** Identified and selected the **Account Instance** path — designed for single-account use cases without requiring AWS Organizations. Account instance successfully enabled without triggering an account upgrade.
**Production note:** AWS Organizations is required for multi-account IAM Identity Center deployments, which is the standard enterprise configuration. Single-account instances are appropriate for specialized or isolated use cases.

---

### Finding 15 — AWS IAM Identity Center SAML SSO: Successful Integration
**Observed:** Full end-to-end SAML federation established between Okta and AWS IAM Identity Center.
**Configuration completed:**
- AWS IAM Identity Center account instance enabled
- External identity provider configured in AWS — Okta dual-stack metadata XML uploaded to AWS
- AWS service provider metadata (ACS URL, Issuer URL) entered into Okta AWS IAM Identity Center integration
- Okta metadata URL confirmed and metadata file uploaded to AWS
- Identity source successfully changed from IAM Identity Center directory to external IdP (Okta)
- IT Staff group assigned to AWS IAM Identity Center application in Okta
- Global MFA — All Users authentication policy assigned to AWS IAM Identity Center application
- Admin account added to IT Staff group to receive application tile on end-user dashboard

**SSO test result:** End-to-end SSO login flow confirmed — Okta SAML assertion accepted by AWS, user landed in AWS access portal authenticated via Okta identity.

**Initial SSO error and resolution:** First SSO attempt failed with "Something went wrong — this code isn't right." Root cause: no matching user provisioned in AWS IAM Identity Center. Okta sends a SAML assertion with user identity but AWS requires a corresponding local user to map the assertion to. Resolved by manually provisioning user in AWS IAM Identity Center with username matching the exact Okta UPN.

**SCIM note:** In production, SCIM automated provisioning would sync users from Okta to AWS automatically — eliminating manual user creation on the SP side. SCIM was not configured in this lab due to trial tier constraints.

**Pattern documented:** Slack, GitHub, and AWS all presented SP-side requirements during SSO integration attempts. Okta IdP configuration was completed successfully in all cases — the limitation was consistently on the service provider side (paid plan requirements), not the Okta side.

---

## Task 5 — Okta Workflows Automation

### Finding 16 — Workflows Console Accessible but Connector Authentication Restricted
**Observed:** Okta Workflows console is accessible in trial tier. Flow canvas, trigger selection, template library, and card-based flow builder UI were all navigated successfully. User Deactivated trigger selected and connection dialog opened.
**Blocker:** OAuth connection to Okta tenant could not be completed — Create button remained greyed out after entering all required connection fields (name, domain, Client ID, Client Secret). API Services app type returned 400 Bad Request. OIDC Web Application app type created but connector still could not authenticate.
**Root cause:** Workflows connector authentication to the Okta tenant is restricted in trial tier. Full Okta Workflows functionality requires a paid Workflows entitlement.
**Production note:** In production, Okta Workflows connects natively to the Okta tenant via a pre-authorized connection. Common automation use cases include: auto-deprovisioning users from all apps on deactivation, sending Slack/email notifications on lifecycle events, and syncing group membership changes across connected systems.

---

## Task 6 — Identity Governance

### Finding 17 — Identity Governance is a Paid Subscription Add-On
**Observed:** Identity Governance menu item not present in trial tier admin console. No navigation path to Access Certifications, Access Requests, or Entitlement Management features.
**Root cause:** Confirmed via Okta documentation — Okta Identity Governance (OIG) is Generally Available on a subscription basis as a paid add-on, separate from Workforce Identity licensing. It is not included in any trial tier.
**Features covered by OIG:**
- Access Certifications — periodic user access reviews and campaign management
- Access Requests — structured access request and approval workflows
- Entitlement Management — fine-grained app entitlement assignment and policy
**Compliance relevance:** OIG features directly support SOC2 and SOX audit requirements by providing documented, auditable access review campaigns.
**Production note:** Organizations subject to compliance frameworks typically require periodic access certification campaigns. OIG automates what would otherwise be a manual spreadsheet-driven review process.

---

## Task 7 — System Log Audit Analysis

### Finding 18 — System Log: 689 Events Captured Across Lab Session
**Observed:** System Log populated with 689 events across the full lab session — covering AD import, user activation, group assignment, policy creation, application integration, SSO attempts, and lifecycle management actions.
**Actions completed:**
- Filtered System Log by Alice Allison (aallison@soc-lab.local) — confirmed admin-performed lifecycle events visible including deactivation attempt and reactivation
- Identified failed authentication events from SSO testing — error codes and reason fields documented
- Exported filtered log segment as CSV — confirmed download
- Reviewed available event fields: event type, outcome (success/failure), actor, target, client (IP/user agent), timestamp, and debug context

**Production note:** System Log is the primary audit and investigation tool for Okta administrators. Common support use cases include: tracing a user's sign-in failures, confirming when a password reset was performed and by whom, auditing application access events, and identifying suspicious authentication patterns. Log retention and export to a SIEM (such as Splunk) is standard practice in enterprise environments — directly applicable to the existing Splunk infrastructure in this lab.

---

## Cross-Task Pattern Summary

**SP-Side Paywall Pattern:** Across Task 4, three SSO integration targets were attempted — Slack, GitHub, and AWS. In all three cases, Okta's IdP configuration was completed successfully. SP-side paywalls were encountered at Slack (Pro plan required for SAML), GitHub (Teams/Enterprise required for org SSO), and AWS (Organizations upgrade prompted before account instance path was identified). AWS IAM Identity Center via account instance was the only successful end-to-end integration. This pattern reflects real enterprise procurement reality — SSO integration requires licensing decisions on both the IdP and SP sides.

**Trial Tier Feature Boundary:** The following features were confirmed unavailable in the Okta Workforce Identity trial tier: Workflows connector authentication, Identity Governance (OIG), full password writeback to AD, and some advanced attribute mapping options. All unavailable features were documented with production context rather than omitted.

**AD as Authoritative Identity Source:** Throughout all lifecycle management tasks, Active Directory was confirmed as the authoritative identity source. Password operations, group membership for AD-sourced groups, and user attribute updates all resolve through AD — not the Okta console. This is the correct architecture for an AD-federated Okta deployment and was demonstrated through practical troubleshooting in Tasks 2 and 3.
