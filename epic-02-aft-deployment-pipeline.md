# EPIC 2: AFT Deployment & Pipeline

**Epic Key:** LZ-AFT
**Epic Name:** Deploy Account Factory for Terraform and establish the account vending pipeline
**Priority:** Highest
**Labels:** `landing-zone`, `aft`, `terraform`, `pipeline`
**Stories:** 4 · **Total Points:** 21
**Sprint Target:** Sprint 2

## Brownfield Context

Control Tower is pre-existing on the management/billing account. Epic 1 upgrades CT and creates a new `AFT-Managed` OU branch off root. AFT deploys into a dedicated account under this new branch. Account requests target only `AFT-Managed` sub-OUs — existing accounts and legacy OUs are never touched by AFT.

## Epic Description

Deploy Account Factory for Terraform (AFT) in a dedicated management account under the `AFT-Managed` OU branch, configure the four Git repositories that drive account vending and customization, and create a baseline account request template. The epic concludes with an end-to-end smoke test that vends a vanilla account and validates the full pipeline.

### Definition of Done

- AFT is deployed and the CodePipeline is operational
- Four AFT Git repos are created with branch protection and CI checks
- A baseline account request template exists targeting `AFT-Managed` sub-OUs
- A test account has been successfully vended end-to-end into the correct sub-OU

### Dependencies

- Epic 1 complete (CT upgraded, new `AFT-Managed` OU branch created and registered)
- Git provider decision made (CodeCommit vs GitHub/GitLab)

### Blocked By

- Epic 1 (Control Tower upgrade & new OU branch)

### Blocks

- Epic 3 (Networking — spoke module integration into AFT)
- Epic 4 (IAM — role/SSO integration into AFT)
- Epic 5 (Security — Config rules deployed via AFT)

---

## Stories

### LZ-201: Deploy AFT in the AFT Management Account

**Story Points:** 8 · **Priority:** Highest · **Component:** AFT

**Description:**
As a platform engineer, I need AFT deployed so that we have a Terraform-based pipeline for provisioning and customizing accounts.

**Acceptance Criteria:**

- Dedicated AFT management account is created (or existing one designated)
- AFT Terraform module is deployed (`aws-ia/control_tower_account_factory` or latest official module)
- AFT CodePipeline is operational (CodeCommit/CodeBuild/CodePipeline or GitHub + CodeBuild)
- AFT DynamoDB state table and S3 backend are configured
- Terraform state is encrypted and access-controlled
- AFT version is pinned and documented

**Notes:** Decide Git provider (CodeCommit vs GitHub/GitLab). If GitHub, configure OIDC or PAT integration.

---

### LZ-202: Configure AFT Git Repositories

**Story Points:** 5 · **Priority:** Highest · **Component:** AFT

**Description:**
As a platform engineer, I need the four AFT Git repositories set up with branching strategy and CI checks so that all account definitions and customizations are version-controlled.

**Acceptance Criteria:**

- Four repos created and linked to AFT:
  1. `aft-account-request` — account vending definitions
  2. `aft-global-customizations` — customizations applied to all accounts
  3. `aft-account-customizations` — per-account/OU customizations
  4. `aft-account-provisioning-customizations` — pre/post provisioning hooks
- Branch protection rules configured (PR required, at least 1 approval)
- README in each repo documents purpose and usage
- `terraform fmt` and `terraform validate` run in CI on every PR

---

### LZ-203: Create AFT Account Request Template for Vanilla Account

**Story Points:** 3 · **Priority:** High · **Component:** AFT

**Description:**
As a platform engineer, I need a baseline account request template so that new accounts can be vended with consistent metadata and placed in the correct OU.

**Acceptance Criteria:**

- Account request Terraform module defines:
  - Account name, email, OU placement
  - SSO user/group assignment
  - Tags (environment, cost-center, owner, team)
  - Custom fields for networking tier (e.g., `spoke`, `isolated`)
- At least one sample account request exists and is validated
- Account request triggers AFT pipeline end-to-end

---

### LZ-204: End-to-End Smoke Test — Vend a Vanilla Account via AFT

**Story Points:** 5 · **Priority:** High · **Component:** AFT

**Description:**
As a platform engineer, I need to vend a test account through AFT and verify all customizations apply so that we can confirm the pipeline works before onboarding teams.

**Acceptance Criteria:**

- A test account is vended via a PR to `aft-account-request`
- Account appears in the correct OU
- Global customizations (IAM, security baselines) are applied
- Account-level customizations (networking) are applied
- SSO access to the new account works
- Pipeline logs show successful completion with no errors
