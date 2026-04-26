# SharePoint Online Administration
## Sterling Enterprises Tenant

**Tenant:** SterlingEnterprises  
**License tier:** Microsoft 365 Business Basic  
**Admin center:** SterlingEnterprises-admin.sharepoint.com  

---

## Overview

This section documents hands-on SharePoint Online administration performed on a live Microsoft 365 Business Basic tenant. Tasks cover site provisioning, permission management, custom permission level creation, document library configuration, storage allocation review, and external sharing policy analysis.

All tasks were performed via the SharePoint Admin Center and directly within individual SharePoint sites.

---

## Environment Baseline

Upon accessing the SharePoint Admin Center, four sites were present in Active Sites:

| Site | Type | Origin |
|---|---|---|
| All Company | Communication site | Auto-provisioned by Microsoft at tenant creation |
| SterlingIT-Team | Team site | Auto-provisioned when Microsoft 365 group was created in Week 1 |
| Sterling IT Support | Team site | Manually provisioned — Week 3 |
| Sterling IT Announcements | Communication site | Manually provisioned — Week 3 |

The presence of All Company and SterlingIT-Team prior to any manual SharePoint work confirms that Microsoft 365 group creation and tenant provisioning automatically generate backing SharePoint infrastructure without administrator action. This is documented as a cross-week finding: the Microsoft 365 group created in the Entra ID section of this lab directly provisioned a SharePoint Team site as part of its unified resource creation behavior.

---

## Task 1: Site Creation

### Objective
Provision a Team site and a Communication site and document the architectural distinction between them.

### Sites Provisioned

| Site name | Type | Template | Privacy |
|---|---|---|---|
| Sterling IT Support | Team site | Standard | Private |
| Sterling IT Announcements | Communication site | N/A | N/A |

### Site Type Distinction

**Team sites** are collaboration-oriented. Creating a Team site automatically provisions a backing Microsoft 365 group with associated Owners and Members groups, a shared mailbox, and a document library. Team sites are appropriate when a defined group of users needs to collaborate on shared content with controlled membership.

**Communication sites** are broadcast-oriented. They are designed for one-to-many communication where a small number of content authors publish to a broader audience of readers. Communication sites do not provision a backing Microsoft 365 group. They expose Site admins as a distinct role category rather than the Owners and Members group structure used by Team sites.

The correct site type depends on the use case: team collaboration with defined membership (Team site) vs. organization-wide or department-wide content publishing (Communication site).

### Findings
- Team sites auto-generate backing Microsoft 365 group objects for Owners and Members at provisioning time, including associated group email addresses.
- Communication sites do not generate Microsoft 365 group objects; they use a Site admins model instead.
- Two sites were present before any manual provisioning: All Company (auto-provisioned at tenant creation) and SterlingIT-Team (auto-provisioned when the SterlingIT-Team Microsoft 365 group was created during the Entra ID week). This confirms SharePoint infrastructure is provisioned as a side effect of Microsoft 365 group creation, not as a separate administrative action.

---

## Task 2: Permission Configuration

### Objective
Assign Owner, Member, and Visitor permission levels to users on provisioned sites and document the permission model.

### Sterling IT Support — Permissions Configured

| Role | Assigned to |
|---|---|
| Site owners | Admin account; Sterling IT Support Owners group (auto-generated) |
| Site members | shelpdesk (redacted); Sterling IT Support Members group (auto-generated) |
| Site visitors | None assigned |

### Sterling IT Announcements — Permissions Configured

| Role | Assigned to |
|---|---|
| Site members | shelpdesk (redacted) |

### SharePoint Permission Model

SharePoint Online uses three default permission levels mapped to roles:

**Owners** have Full Control. They can manage site settings, add and remove members, create and delete libraries, and modify permissions. Ownership should be restricted to administrators.

**Members** have Edit access by default on Team sites. They can add, edit, and delete list items and documents but cannot modify site structure or manage permissions.

**Visitors** have Read access. They can view content but cannot add, edit, or delete anything. Visitor access is appropriate for stakeholders who need to reference content without contributing to it.

### Findings
- Team site provisioning automatically creates Owners and Members Microsoft 365 groups with associated email addresses. These groups are visible in the Entra ID admin center alongside manually created groups.
- Communication sites present Site admins as the primary administrative role rather than an Owners group, reflecting their different architectural model.
- Visitor role was not assigned on Sterling IT Support as no read-only stakeholder accounts exist in the lab tenant. The role and its permission implications are documented above.

---

## Task 3: Custom Permission Level

### Objective
Attempt to create a custom permission level and document the outcome and tier implications.

### Outcome
Custom permission level creation is **available at Business Basic tier**. The Permission Levels management interface was fully accessible at:

Site Settings > Site Permissions > Advanced permissions settings > Permission Levels

An Add a Permission Level button was present and functional.

### Custom Permission Level Created

**Name:** Helpdesk Technician
**Description:** Custom permission level for help desk staff. Grants read access to site content and the ability to add and edit list items without the ability to delete or modify site structure.

### Permissions Configured

**List Permissions — Granted:**
- Add Items
- Edit Items
- View Items
- View Application Pages
- Open Items

**List Permissions — Withheld:**
- Delete Items
- Manage Lists

**Site Permissions — Granted:**
- View Pages
- Browse User Information
- Use Remote Interfaces
- Use Client Integration Features
- Open

**Site Permissions — Withheld:**
- Manage Permissions
- Manage Web Site
- Add and Customize Pages

### Design Rationale

The Helpdesk Technician permission level applies the principle of least privilege to SharePoint access. Help desk staff need to read site content, log support items, and update list entries. They do not need to delete records (which could compromise audit trails) or modify site structure (which is an administrative function). This configuration mirrors realistic enterprise permission design for support staff SharePoint access and is directly explainable in an interview context.

### Findings
- Custom permission level creation is available at Business Basic tier without P1 or P2 requirement.
- The default permission levels (Full Control, Design, Edit, Contribute, Read) cannot be deleted but can be supplemented with custom levels.
- Custom permission levels created at the site level are available only within that site collection; they do not propagate to other sites automatically.

---

## Task 4: Document Library with Custom Column and Versioning

### Objective
Create a document library with a custom metadata column, enable versioning, and demonstrate version history functionality.

### Library Created

| Field | Value |
|---|---|
| Name | IT Support Documents |
| Description | Document library for IT support procedures, knowledge base articles, and policy documentation |
| Location | Sterling IT Support site |

### Custom Column Created

| Field | Value |
|---|---|
| Column name | Document Type |
| Column type | Choice |
| Choices | Procedure, Policy, Knowledge Base Article, Reference |
| Default value | Procedure |
| Multiple selections | Disabled |

The Document Type column allows documents to be categorized by type at upload or edit time, enabling filtering and organized navigation of library contents without requiring folder-based organization.

### Versioning Configuration

| Setting | Value |
|---|---|
| Document version history | Major versions enabled |
| Maximum versions retained | 100 |
| Require check-out | No |

**Note on version limit:** SharePoint Online enforces a minimum of 100 major versions. Values below 100 are not accepted. This differs from SharePoint Server on-premises, where lower version limits are permitted. Administrators migrating version policies from on-premises environments should account for this platform difference.

### Version History Demonstrated

A test document was created in the IT Support Documents library, edited to generate a second version, and version history was accessed via right-click > Version history. The version history panel displayed both versions with timestamps and version numbers, confirming versioning is active and recording changes correctly.

### Tier 2 Relevance

Document library administration is one of the most common SharePoint support scenarios. Users frequently request help with permission to a library, inability to find a previous document version, or questions about why a document appears different from what they saved. Understanding how to navigate version history, configure column metadata, and manage library settings allows a Tier 2 technician to resolve these tickets without escalation.

### Findings
- Document libraries, custom columns, and versioning are all available at Business Basic tier.
- SharePoint Online enforces a minimum of 100 major versions; on-premises SharePoint Server allows lower values. This is a platform behavioral difference relevant to migration planning.
- Version history is accessible per-document via right-click context menu; it does not require administrator access to view.

---

## Task 5: OneDrive Storage Allocation Review

### Objective
Review OneDrive storage allocation settings and document the storage model at Business Basic tier.

### Access Path
SharePoint Admin Center > Settings > OneDrive storage limit

### Storage Configuration Observed

| Setting | Value |
|---|---|
| Default OneDrive storage limit per user | 1024 GB |
| Active Sites storage consumed | 0.00 GB across all sites |

### Storage Model at Business Basic

Microsoft 365 Business Basic tenants receive a pooled storage allocation calculated as 1 TB base storage plus 10 GB per licensed user. With 25 licenses provisioned on the SterlingEnterprises tenant, the total pooled storage ceiling is approximately 1.25 TB, distributed across all SharePoint sites and OneDrive accounts in the tenant.

Individual OneDrive accounts are capped at the administrator-configured per-user storage limit. The default of 1024 GB can be lowered by an administrator to enforce storage governance across the organization. Raising it beyond the per-user default requires sufficient pooled tenant storage to support the allocation.

### Tier 2 Relevance

Storage allocation management becomes relevant when users report OneDrive sync errors or are unable to upload files. The common cause is the user approaching their individual storage limit. An administrator can review per-user consumption in the SharePoint Admin Center and either raise the user's limit or work with them to reduce consumption. Tenant-level storage exhaustion is a less common but more serious issue requiring license review or storage add-on procurement.

### Findings
- OneDrive storage limit configuration is available at Business Basic tier.
- Active Sites storage values showed 0.00 GB across all provisioned sites, consistent with minimal content in a new lab tenant.
- The pooled storage model means individual site or user storage limits are constrained by total tenant pool consumption, not just per-user settings.

---

## Task 6: External Sharing Settings

### Objective
Review and document the tenant-wide external sharing configuration, identify the current posture, and document recommended security hardening for a production environment.

### Access Path
SharePoint Admin Center > Policies > Sharing

### Current Configuration Observed

| Setting | Current value |
|---|---|
| SharePoint external sharing | Anyone (most permissive) |
| OneDrive external sharing | Anyone (most permissive) |
| Limit external sharing by domain | Disabled |
| Allow only specific security groups to share externally | Disabled |
| Allow guests to share items they don't own | Enabled |
| Guest access expiration | Disabled |
| Verification code reauthentication | Disabled |
| Default link type | Anyone with the link |
| Default link permission | Edit |

### Security Analysis

The current configuration represents the most permissive external sharing posture available in SharePoint Online. Combined, these settings allow any user to generate an anonymous edit link to any document and share it with anyone outside the organization without restriction, authentication requirement, or expiration.

This is the Microsoft out-of-box default for new tenants. It is a realistic finding: inheriting a tenant with this configuration is a common scenario for a Tier 2 administrator or junior sysadmin. Identifying and remediating overly permissive sharing settings is a standard task during tenant security reviews.

### Recommended Hardening for Production Environment

| Setting | Current | Recommended | Rationale |
|---|---|---|---|
| SharePoint external sharing | Anyone | New and existing guests minimum | Requires authentication for all external access |
| OneDrive external sharing | Anyone | New and existing guests minimum | Consistent with SharePoint posture |
| Default link type | Anyone with the link | Only people in your organization | Prevents accidental anonymous sharing |
| Default link permission | Edit | View | Reduces accidental write access grants |
| Guest access expiration | Disabled | 60 days | Limits persistent guest access without review |
| Verification code reauthentication | Disabled | 30 days | Ensures periodic re-verification of external users |
| Domain restrictions | Disabled | Enable; allowlist known partner domains | Restricts external sharing to vetted organizations |

### Findings
- External sharing settings are fully configurable at Business Basic tier; no sharing controls are gated behind higher tiers.
- The default tenant configuration is maximally permissive. Organizations should treat external sharing settings as a priority item during tenant onboarding and security review.
- Changing the default link type from Anyone with the link to Only people in your organization is the single highest-impact low-effort change available, as it prevents accidental anonymous sharing without blocking intentional external collaboration.
- Guest access expiration and verification code reauthentication are available at Business Basic tier but are disabled by default; both should be enabled in production environments.

---

## Screenshots

See `screenshots/` folder for visual documentation of each task.

| Filename | Task | Description |
|---|---|---|
| [sharepoint-active-sites.png](screenshots/sharepoint-active-sites.png) | Task 1 | Active Sites showing all four sites including auto-provisioned |
| [sharepoint-support-owners.png](screenshots/sharepoint-support-owners.png) | Task 2 | Sterling IT Support membership — owners section |
| [sharepoint-support-members.png](screenshots/sharepoint-support-members.png) | Task 2 | Sterling IT Support membership — members section |
| [sharepoint-announcements-members.png](screenshots/sharepoint-announcements-members.png) | Task 2 | Sterling IT Announcements membership — shelpdesk as member |
| [sharepoint-custom-permission-config.png](screenshots/sharepoint-custom-permission-config.png) | Task 3 | Helpdesk Technician permission level configuration before saving |
| [sharepoint-permission-levels-list.png](screenshots/sharepoint-permission-levels-list.png) | Task 3 | Permission Levels list showing Helpdesk Technician added |
| [sharepoint-library-landing.png](screenshots/sharepoint-library-landing.png) | Task 4 | IT Support Documents library landing page |
| [sharepoint-custom-column.png](screenshots/sharepoint-custom-column.png) | Task 4 | Document Type choice column visible in library headers |
| [sharepoint-versioning-settings.png](screenshots/sharepoint-versioning-settings.png) | Task 4 | Versioning settings panel showing major versions enabled at 100 |
| [sharepoint-version-history.png](screenshots/sharepoint-version-history.png) | Task 4 | Version history panel showing two versions with timestamps |
| [sharepoint-onedrive-storage.png](screenshots/sharepoint-onedrive-storage.png) | Task 5 | OneDrive storage limit settings showing 1024 GB default |
| [sharepoint-external-sharing.png](screenshots/sharepoint-external-sharing.png) | Task 6 | Sharing settings panel showing Most Permissive configuration |

---

## Summary

This section demonstrates proficiency in SharePoint Online administration at the Business Basic tier. Site provisioning, permission management including a custom permission level, document library configuration with metadata columns and versioning, storage allocation review, and external sharing policy analysis were all completed and documented. The external sharing configuration finding is the most security-relevant outcome of this section: identifying a maximally permissive default posture and documenting the remediation rationale reflects the kind of security-aware administrative judgment relevant to Tier 2 and junior sysadmin roles.
