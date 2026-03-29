# EPIC 4: IAM Foundation

**Epic Key:** LZ-IAM
**Epic Name:** Establish full IAM foundation — SSO, SCPs, roles, cross-account trust, and tag policies
**Priority:** Highest
**Labels:** `landing-zone`, `iam`, `sso`, `scp`, `security`
**Stories:** 6 · **Total Points:** 29
**Sprint Target:** Sprints 3 and 5

## Brownfield Context

The existing OU tree already has SCPs and tagging policies. This epic targets only the new `AFT-Managed` OU branch. SCPs are owned by the InfoSec team — all SCP stories carry an **external dependency** on InfoSec review and approval. Existing SSO permission sets and IAM roles in legacy accounts are not modified. New permission sets may reuse existing SSO groups or create new ones depending on the identity source configuration.

## Epic Description

Build the complete IAM foundation for AFT-managed accounts: configure AWS IAM Identity Center with baseline permission sets scoped to the new OU branch, define and apply SCPs (in coordination with InfoSec) to `AFT-Managed` sub-OUs, deploy baseline IAM roles into every vended account, establish cross-account trust policies, enforce tag policies, and integrate all IAM constructs into the AFT pipeline so they're applied automatically on account vend.

### Definition of Done

- IAM Identity Center is configured with 5+ permission sets assigned to groups for the new OU branch
- SCPs are codified in Terraform, tested in `AFT-Managed/PolicyStaging`, and approved by InfoSec
- Baseline IAM roles are deployed to every AFT-vended account via AFT
- Cross-account trust matrix is documented and implemented
- Tag policies are enforced on the `AFT-Managed` branch (aligned with or extending existing org tag policies)
- All IAM constructs are automated through AFT — zero manual setup on account vend

### External Dependencies

- **InfoSec Team** — SCP authoring, review, and approval (see LZ-106 in Epic 1 for initial coordination; this epic covers AFT-specific SCP integration)

### Dependencies

- Epic 1 complete (new OUs created and registered, InfoSec SCP coordination started)
- Epic 2 complete (AFT repos for customization integration)

### Blocked By

- Epic 1 (new OU branch for SCP targeting)
- Epic 2 (AFT global customizations repo)

### Blocks

- Epic 2, LZ-204 (smoke test validates IAM customizations)
- Epic 5 (security services rely on IAM roles for cross-account access)

---

## Stories

### LZ-401: Configure AWS IAM Identity Center (SSO) with Permission Sets

**Story Points:** 5 · **Priority:** Highest · **Component:** IAM

**Description:**
As a security engineer, I need IAM Identity Center configured with baseline permission sets so that human access to all accounts goes through SSO with consistent roles.

**Acceptance Criteria:**

- IAM Identity Center enabled in the management account (delegated admin to audit/security account if desired)
- Identity source configured (built-in, AD, or external IdP — document choice)
- Baseline permission sets created:
  - `AdministratorAccess` — full admin, restricted to break-glass
  - `PowerUserAccess` — broad access, no IAM mutations
  - `ReadOnlyAccess` — audit and inspection
  - `PlatformEngineer` — custom, scoped to infrastructure operations
  - `DeveloperAccess` — custom, scoped to application workloads
- Permission sets assigned to groups (not individual users)
- Session duration configured (e.g., 4 hours for admin, 8 hours for read-only)

---

### LZ-402: Define and Apply Service Control Policies (SCPs)

**Story Points:** 8 · **Priority:** Highest · **Component:** IAM

**Description:**
As a security engineer, I need SCPs attached to OUs so that accounts are prevented from performing actions that violate our security and compliance posture.

**Acceptance Criteria:**

- SCPs codified in Terraform and version-controlled
- Baseline SCPs implemented:
  - Deny leaving the Organization
  - Deny disabling CloudTrail, Config, or GuardDuty
  - Deny root user actions (except for account recovery)
  - Region deny (restrict to approved regions only)
  - Deny creation of IAM users with console access (force SSO)
  - Deny public S3 buckets at the account level
  - Deny modification of AFT-managed resources (AFT pipeline protection)
- SCPs attached to appropriate OUs (not the root unless intentional)
- SCPs tested in `PolicyStaging` OU before production rollout
- Deny effects documented so teams know what's restricted

---

### LZ-403: Create Baseline IAM Roles for Vended Accounts

**Story Points:** 5 · **Priority:** High · **Component:** IAM

**Description:**
As a platform engineer, I need baseline IAM roles deployed into every vended account via AFT global customizations so that automation and cross-account access work from day one.

**Acceptance Criteria:**

- Roles deployed via `aft-global-customizations`:
  - `OrganizationAccountAccessRole` — exists by default, document trust policy
  - `PlatformAutomationRole` — assumed by CI/CD pipelines, scoped permissions
  - `SecurityAuditRole` — assumed by the security/audit account, read-only
  - `CostManagementRole` — assumed by the billing/FinOps account, cost/billing read
  - `BreakGlassRole` — emergency access, heavily logged, MFA required
- All roles use least-privilege policies
- Trust policies restrict to specific accounts/principals
- Roles are tagged with `managed-by: aft`

---

### LZ-404: Implement Cross-Account Trust Policies

**Story Points:** 5 · **Priority:** High · **Component:** IAM

**Description:**
As a security engineer, I need a documented and automated cross-account trust model so that the security, logging, and platform accounts can assume roles in workload accounts without over-permissioning.

**Acceptance Criteria:**

- Cross-account trust matrix documented:
  - Security account -> `SecurityAuditRole` in all accounts
  - Network account -> `NetworkManagementRole` (if applicable)
  - CI/CD account -> `PlatformAutomationRole` in workload accounts
  - Management account -> `OrganizationAccountAccessRole` (break-glass only)
- Trust policies enforce `aws:PrincipalOrgID` condition to prevent external assumption
- External ID or MFA conditions applied where appropriate
- Cross-account access tested end-to-end (assume-role + verify permissions)

---

### LZ-405: Implement Tag Policies at the Organization Level

**Story Points:** 3 · **Priority:** Medium · **Component:** IAM

**Description:**
As a FinOps/governance lead, I need tag policies enforced across the organization so that resources are consistently tagged for cost allocation, ownership, and compliance.

**Acceptance Criteria:**

- Tag policies codified in Terraform
- Mandatory tags defined:
  - `Environment` (allowed values: `dev`, `staging`, `prod`, `sandbox`)
  - `CostCenter` (format enforced)
  - `Owner` (email format)
  - `ManagedBy` (allowed values: `aft`, `terraform`, `manual`)
  - `Application`
- Tag policies attached to root or specific OUs
- Non-compliant resources detectable via AWS Config rules or Tag Editor

---

### LZ-406: Integrate IAM Customizations into AFT Pipeline

**Story Points:** 3 · **Priority:** High · **Component:** IAM

**Description:**
As a platform engineer, I need all IAM constructs (permission sets, roles, policies) applied automatically when AFT vends an account so that no manual IAM setup is required.

**Acceptance Criteria:**

- SSO permission set assignments are part of the AFT account request or global customization
- Baseline roles are deployed via `aft-global-customizations`
- SCPs are applied at the OU level (not per-account) and verified post-vend
- Tag policies are inherited from OU attachment
- A newly vended account passes an IAM compliance check (Config conformance pack or custom script)
