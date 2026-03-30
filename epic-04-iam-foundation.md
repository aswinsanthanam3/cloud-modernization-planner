# EPIC 4: IAM Foundation

**Epic Key:** LZ-IAM
**Epic Name:** Establish IAM Identity Center, baseline roles, cross-account trust, and tag governance
**Priority:** Highest
**Labels:** `landing-zone`, `iam`, `sso`, `permission-sets`, `cross-account`
**Stories:** 5 · **Total Points:** 23
**Sprint Target:** Sprints 3–4

> **Note:** SCP governance (four-tier model, BU service profiles, environment controls, and SCP validation) has been moved to **Epic 9** to decouple IAM delivery from InfoSec-gated SCP approval cycles. IAM work can progress independently.

> **TEAM (Epic 10):** Permission sets defined here serve as the foundation for AWS TEAM just-in-time elevated access. When designing standing assignments in LZ-401, plan for two categories: *always-standing* (ReadOnly, Dev/Sandbox write access) and *JIT-only via TEAM* (Prod write access, PowerUser, AdministratorAccess). Epic 10 will remove standing write assignments in Prod and replace them with time-bound TEAM requests.

---

## Brownfield Context

Existing SSO permission sets and IAM roles in legacy accounts are not modified. This epic targets only the `AFT-Managed` OU branch. Any manual IAM users or access keys in legacy accounts are out of scope but should be flagged in Epic 1, LZ-106 (InfoSec SCP coordination).

---

## Epic Description

Build the complete human-access and machine-access IAM layer for AFT-managed accounts: IAM Identity Center with BU-aware permission sets, baseline IAM roles deployed to every vended account via AFT global customizations, cross-account trust policies that follow least-privilege and org-boundary scoping, and tag policies that enforce cost/ownership tagging across the `AFT-Managed` branch. Everything is automated through AFT — zero manual IAM setup on account vend.

### Definition of Done

- IAM Identity Center configured with BU-aware permission sets assigned to groups (not individuals)
- Baseline IAM roles deployed to every AFT-vended account via `aft-global-customizations`
- Cross-account trust matrix documented, codified in Terraform, and tested end-to-end
- Tag policies attached at `AFT-Managed` OU with mandatory tag set enforced
- AFT pipeline integration validated: new account vend requires zero manual IAM steps
- All roles tagged `managed-by: aft` and protected by SCP (Epic 9, LZ-901 — `scp-aft-pipeline-protect`)

### Dependencies

- Epic 1 complete (AFT-Managed OU branch and sub-OUs created; CT upgraded)
- Epic 2 complete (AFT customization repos available)
- Epic 9, LZ-901 (Tier 1b SCP `scp-aft-pipeline-protect` must be in place before IAM roles go live — otherwise roles can be deleted manually)

### Blocked By

- Epic 1 (OU structure must exist before permission set assignments and tag policies can be scoped)
- Epic 2 (AFT customization repos required for role deployment)

### Blocks

- Epic 2, LZ-204 (smoke test validates baseline IAM roles and SSO permission set assignment)
- Epic 5 (security services rely on `SecurityAuditRole` and `CostManagementRole` deployed here)
- Epic 3, LZ-306 (AFT integration depends on IAM roles being available to the customization pipeline)

---

## Stories

---

### LZ-401: Configure AWS IAM Identity Center with BU-Aware Permission Sets

**Story Points:** 5 · **Priority:** Highest · **Component:** IAM / SSO

**Description:**
As a security engineer, I need IAM Identity Center configured with baseline permission sets that map to our BU structure so that human access to all accounts flows through SSO with appropriately scoped roles — no IAM users, no long-lived credentials.

**Acceptance Criteria:**

**Identity Center setup:**
- IAM Identity Center enabled in the management account
- Delegated admin configured to the Security account (management account does not own day-to-day SSO administration)
- Identity source configured and documented: built-in directory / AD Connector / external IdP (mark which is chosen and why)
- MFA enforced org-wide on all SSO assignments

**Permission sets (codified in Terraform):**

| Permission Set | Scope | Session Duration | MFA Required |
|---------------|-------|-----------------|-------------|
| `AdministratorAccess` | Break-glass only | 1 hour | Yes (hardware) |
| `PowerUserAccess` | Broad non-IAM; platform engineers | 4 hours | Yes |
| `ReadOnlyAccess` | Audit, compliance, inspection | 8 hours | No |
| `PlatformEngineer` | All AFT-managed accounts; infra-scoped | 4 hours | Yes |
| `DeveloperAccess-Standard` | BU1/BU2/BU3 Dev + Staging only | 4 hours | No |
| `DeveloperAccess-Prod` | Prod read-only; deploys via CI/CD only | 2 hours | Yes |
| `DataEngineerAccess` | BU3 accounts; includes Glue, EMR, Athena, Redshift | 4 hours | No |
| `PlatformAIAccess` | Platform OU accounts only; includes Bedrock, SageMaker | 4 hours | Yes |
| `SecurityAuditAccess` | Security team; read-only + Config/GuardDuty | 8 hours | No |

**Assignment rules:**
- All permission sets assigned to **groups** (not individual users)
- Group-to-account assignments driven by OU path:
  - `DeveloperAccess-Standard` → all `Standard/BU*/Dev` and `Standard/BU*/Staging` accounts
  - `DeveloperAccess-Prod` → all `Standard/BU*/Prod` accounts
  - `PlatformAIAccess` → all `Platform/*` accounts only
  - `PlatformEngineer` → all `AFT-Managed` accounts
- Permission boundaries attached to all permission sets (except `AdministratorAccess`) to prevent privilege escalation via role chaining

**Validation:**
- Log in as a `DeveloperAccess-Standard` user in a `BU1/Dev` account → can deploy; cannot create IAM roles
- `PlatformAIAccess` user in a Platform account → can invoke `bedrock:InvokeModel`
- `PlatformAIAccess` user assigned to a Standard account → fails (assignment policy blocks it)
- `AdministratorAccess` assumption generates a CloudTrail alert within 5 minutes

---

### LZ-402: Create Baseline IAM Roles for Every Vended Account

**Story Points:** 5 · **Priority:** High · **Component:** IAM

**Description:**
As a platform engineer, I need a standard set of IAM roles deployed into every vended account via AFT global customizations so that automation, cross-account access, security audit, and break-glass access work from day one — regardless of BU or environment.

**Acceptance Criteria:**

**Roles deployed via `aft-global-customizations` to every account:**

| Role Name | Assumed By | Purpose | Key Constraints |
|-----------|-----------|---------|----------------|
| `PlatformAutomationRole` | CI/CD platform account | Infra deployments | Permission boundary; cannot create roles exceeding its own permissions |
| `SecurityAuditRole` | Security/audit account | Read-only security inspection | ReadOnly + Config/GD/SH access; no write |
| `CostManagementRole` | Billing/FinOps account | Cost Explorer + tagging read | Read-only; cannot create resources |
| `NetworkManagementRole` | Network hub account | TGW attachment management | Scoped to EC2:TGW actions only |
| `BreakGlassRole` | Management account | Emergency access | MFA required; CloudTrail alert on every assumption; 1-hour session |

**All roles must:**
- Be tagged: `managed-by = "aft"`, `tier = "foundation"`, `created-by = "aft-global-customizations"`
- Use `aws:PrincipalOrgID` condition in trust policies (no external account assumption)
- Be protected by `scp-aft-pipeline-protect` (Epic 9, LZ-901) — deletion denied by SCP

**Permission boundary for `PlatformAutomationRole`:**
- Boundary policy allows only services in the approved service list
- Cannot create IAM roles, policies, or SAML providers
- Cannot modify SCPs or CT controls

**Validation:**
- Assume each role from the intended source account and verify allowed/denied actions
- Attempt to delete `PlatformAutomationRole` from a non-AFT principal → denied (SCP gate)
- `BreakGlassRole` assumption triggers a CloudWatch alarm and SNS notification within 5 minutes

---

### LZ-403: Implement Cross-Account Trust Policies

**Story Points:** 5 · **Priority:** High · **Component:** IAM

**Description:**
As a security engineer, I need a fully documented and Terraform-codified cross-account trust model so that the security, network, billing, and platform accounts can assume scoped roles in workload accounts without over-permissioning.

**Acceptance Criteria:**

**Trust matrix (codified in Terraform; no manual trust policy edits):**

| Source Account | Target Role | Target Accounts | Additional Condition |
|----------------|------------|-----------------|---------------------|
| Security account | `SecurityAuditRole` | All AFT-managed accounts | `aws:PrincipalOrgID` |
| Network hub account | `NetworkManagementRole` | All Infrastructure accounts | `aws:PrincipalOrgID` |
| CI/CD platform account | `PlatformAutomationRole` | All workload accounts | `aws:PrincipalOrgID` |
| Management account | `BreakGlassRole` | All accounts | `aws:MultiFactorAuthPresent = true` |
| Audit account | `CostManagementRole` | All accounts | `aws:PrincipalOrgID` |

**All trust policies must:**
- Use `aws:PrincipalOrgID` condition — prevents assumption from accounts outside the org even if the account is leaked
- Avoid wildcard account principals — always pin to a specific account ID + org condition
- External ID applied on `BreakGlassRole` and management account roles
- MFA condition (`aws:MultiFactorAuthPresent = true`) on `BreakGlassRole`

**Testing:**
- Positive test: security account assumes `SecurityAuditRole` in a workload account → can list CloudTrail events; cannot write
- Negative test: an IAM principal from outside the org (simulated) attempts to assume `SecurityAuditRole` → denied
- Negative test: assume `BreakGlassRole` without MFA → denied
- All test results documented in story; attached to PR

**Runbook entry:**
- "How to grant a new infrastructure account the right to assume `NetworkManagementRole`" — added to Epic 6 runbook

---

### LZ-404: Enforce Tag Policies on the AFT-Managed Branch

**Story Points:** 3 · **Priority:** Medium · **Component:** IAM / Governance

**Description:**
As a FinOps and governance lead, I need tag policies enforced on the `AFT-Managed` OU branch so that all AFT-vended resources are consistently tagged for cost allocation, compliance, and ownership tracking — and that the `BusinessUnit` tag maps every account to the correct BU tier for chargeback.

**Acceptance Criteria:**

**Tag policies codified in Terraform and attached to `AFT-Managed` OU:**

| Tag Key | Enforced Values | Format |
|---------|----------------|--------|
| `Environment` | `dev`, `staging`, `prod`, `sandbox` | Lowercase enum |
| `CostCenter` | — | 6-digit numeric string |
| `Owner` | — | Email format (regex) |
| `ManagedBy` | `aft`, `terraform`, `manual` | Lowercase enum |
| `Application` | — | Free text (required) |
| `BusinessUnit` | `bu1-api`, `bu2-international`, `bu3-data`, `platform`, `regulated`, `shared` | Lowercase enum |

**Scope:**
- Tag policies attached to `AFT-Managed` OU only — legacy OUs untouched
- Enforced resource types: EC2 instances, RDS instances, S3 buckets, Lambda functions, ECS tasks, VPCs, Load Balancers
- Non-compliant resources surfaced via AWS Config Tag Editor conformance pack (not hard-blocked at creation — tag policy enforcement mode is report-only for first 30 days, then enforce)

**AFT integration:**
- `aft-account-request` template updated to require `BusinessUnit`, `CostCenter`, `Owner` fields
- AFT-vended account root tags automatically applied via `aft-global-customizations`

**Validation:**
- Create an EC2 instance without a `CostCenter` tag → Config rule flags non-compliant within 15 minutes
- All AFT-vended test account resources have correct tags from first apply

---

### LZ-405: Integrate IAM Layer into AFT Pipeline

**Story Points:** 5 · **Priority:** High · **Component:** IAM / AFT

**Description:**
As a platform engineer, I need the full IAM layer — permission sets, baseline roles, cross-account trust, and tag policies — applied automatically when AFT vends a new account so that zero manual IAM steps are required post-vend.

**Acceptance Criteria:**

**AFT account request template enforces required fields:**
- `ou_path` — must be a valid path under `AFT-Managed` (validated by pre-hook)
- `business_unit` — enum; drives permission set assignment and `BusinessUnit` tag
- `environment` — enum; drives permission set scoping and tag
- `cost_center` — 6-digit; drives `CostCenter` tag and `CostManagementRole` tagging

**`aft-global-customizations` deploys to every account:**
- All five baseline IAM roles (LZ-402)
- Permission boundary policy (pre-requisite for `PlatformAutomationRole`)
- Config rule: `IAM_NO_INLINE_POLICIES`, `IAM_POLICY_NO_STATEMENTS_WITH_ADMIN_ACCESS`, `ROOT_ACCOUNT_MFA_ENABLED`
- CloudWatch alarm: `BreakGlassRole` assumption → SNS → PagerDuty/Slack

**`aft-account-customizations` selects correct SSO permission set assignment based on `business_unit` + `environment`:**
- `business_unit = bu3-data` + `environment = dev` → assigns `DeveloperAccess-Standard` + `DataEngineerAccess`
- `business_unit = platform` → assigns `PlatformEngineer` + `PlatformAIAccess`

**End-to-end smoke test (extends LZ-204 in Epic 2):**
Vend a new `BU3-Data/Dev` account and verify:
- [ ] Account lands in `AFT-Managed/Workloads/Standard/BU3-Data/Dev` OU
- [ ] All 5 baseline IAM roles are present and correctly tagged
- [ ] `SecurityAuditRole` assumable from security account; permission boundary applied
- [ ] `DeveloperAccess-Standard` and `DataEngineerAccess` SSO assignments visible in Identity Center
- [ ] All vended resources tagged with `BusinessUnit = bu3-data`, `Environment = dev`
- [ ] `BreakGlassRole` assumption triggers CloudWatch alarm

---

## Summary

| Story | Title | Points | Priority | Ext. Dependency |
|-------|-------|--------|----------|-----------------|
| LZ-401 | SSO permission sets (BU-aware) | 5 | Highest | — |
| LZ-402 | Baseline IAM roles for vended accounts | 5 | High | — |
| LZ-403 | Cross-account trust policies | 5 | High | — |
| LZ-404 | Tag policies at AFT-Managed branch | 3 | Medium | — |
| LZ-405 | IAM integration into AFT pipeline | 5 | High | — |
| **Total** | | **23** | | |

## Sprint Sequencing

```
Sprint 3:
  LZ-401 (SSO permission sets) ← no external dependencies; start immediately
  LZ-402 (Baseline IAM roles)  ← parallel with LZ-401

Sprint 4:
  LZ-403 (Cross-account trust) ← needs LZ-402 (roles must exist before trust policies)
  LZ-404 (Tag policies)        ← independent; can run parallel with LZ-403
  LZ-405 (AFT integration)     ← needs LZ-401 + LZ-402 + LZ-403 + LZ-404
```

> **SCP stories** (four tiers + matrix validation) are in **Epic 9**. Epic 4 has no InfoSec-gated blockers and can be delivered in Sprints 3–4 while Epic 9 runs in parallel with InfoSec engagement.
