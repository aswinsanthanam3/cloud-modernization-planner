# EPIC 5: Security & Logging Baseline

**Epic Key:** LZ-SEC
**Epic Name:** Enable organization-wide security services and centralized logging
**Priority:** High
**Labels:** `landing-zone`, `security`, `logging`, `guardduty`, `securityhub`, `config`
**Stories:** 3 · **Total Points:** 13
**Sprint Target:** Sprint 5

## Brownfield Context

Since Control Tower is pre-existing, security services (GuardDuty, Security Hub, Config, CloudTrail) may already be enabled org-wide. This epic focuses on **verifying existing coverage**, ensuring new AFT-managed accounts automatically inherit enrollment, and deploying AFT-specific Config rules. The Log Archive and Audit accounts already exist from the original CT deployment — stories here validate they're receiving logs correctly and extend coverage to new accounts.

## Epic Description

Verify that core AWS security services (GuardDuty, Security Hub, Config, Macie) are enabled org-wide with delegated admin, confirm centralized logging to the existing Log Archive account, and deploy baseline Config rules via AFT to detect compliance drift in every newly vended account.

### Definition of Done

- GuardDuty, Security Hub, and Config verified as org-wide with delegated admin (or enabled if missing)
- All logs (CloudTrail, Config, VPC Flow Logs) confirmed shipping to Log Archive with encryption and lifecycle policies
- Baseline Config rules are deployed to every AFT-vended account via global customizations
- Findings from new accounts are aggregated in Security Hub for centralized triage

### Dependencies

- Epic 1 complete (CT upgraded, Log Archive and Audit accounts verified)
- Epic 2 complete (AFT global customizations repo for Config rule deployment)
- Epic 4 partially complete (cross-account roles for security access)

### Blocked By

- Epic 1 (CT upgrade + Log Archive/Audit account validation)
- Epic 2 (AFT customization pipeline)

### Blocks

- Epic 2, LZ-204 (smoke test validates security baseline)
- Epic 6 (documentation references security architecture)

---

## Stories

### LZ-501: Enable Security Services Across All Accounts

**Story Points:** 5 · **Priority:** High · **Component:** Security

**Description:**
As a security engineer, I need core AWS security services enabled organization-wide so that every account has detective controls from the moment it's created.

**Acceptance Criteria:**

- AWS GuardDuty enabled with delegated admin in the security account
- AWS Security Hub enabled with delegated admin, CIS and AWS Foundational benchmarks active
- AWS Config enabled in all accounts, aggregated to the security account
- Amazon Macie enabled on S3 buckets containing sensitive data (or org-wide if policy requires)
- Findings aggregated to the security account for centralized triage

---

### LZ-502: Configure Centralized Logging Architecture

**Story Points:** 5 · **Priority:** High · **Component:** Security

**Description:**
As a security engineer, I need all account-level logs shipped to the Log Archive account so that we have a tamper-resistant, centralized audit trail.

**Acceptance Criteria:**

- CloudTrail organization trail -> Log Archive S3 bucket (SSE-KMS encrypted, lifecycle policy)
- AWS Config delivery channel -> Log Archive S3 bucket
- VPC Flow Logs -> centralized CloudWatch Log Group or S3 in Log Archive
- GuardDuty findings -> Security Hub (and optionally S3 export)
- S3 bucket policies on log buckets deny deletion and enforce encryption
- Log retention policy documented (e.g., 1 year hot, 7 years glacier)

---

### LZ-503: Deploy Baseline Config Rules via AFT

**Story Points:** 3 · **Priority:** Medium · **Component:** Security

**Description:**
As a security engineer, I need AWS Config rules deployed into every account via AFT so that compliance drift is automatically detected.

**Acceptance Criteria:**

- Config rules deployed via `aft-global-customizations`:
  - `s3-bucket-public-read-prohibited`
  - `iam-root-access-key-check`
  - `restricted-ssh`
  - `encrypted-volumes`
  - `rds-instance-public-access-check`
  - `cloudtrail-enabled`
- Conformance pack or rule set documented
- Non-compliant resources generate findings in Security Hub
