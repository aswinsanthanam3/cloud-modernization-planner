# EPIC 4: IAM Foundation

**Epic Key:** LZ-IAM
**Epic Name:** Establish full IAM foundation — SSO, four-tier SCPs, roles, cross-account trust, and tag policies
**Priority:** Highest
**Labels:** `landing-zone`, `iam`, `sso`, `scp`, `security`
**Stories:** 10 · **Total Points:** 52
**Sprint Target:** Sprints 3–5
**ADR References:** ADR-002 (`adrs/adr-002-scp-structure-multi-product-org.md`)

---

## Brownfield Context

The existing OU tree already has SCPs and tagging policies on legacy OUs — those are untouched. This epic targets only the new `AFT-Managed` OU branch. SCPs are owned by the InfoSec team — all SCP stories carry an **external dependency** on InfoSec review and approval. Existing SSO permission sets and IAM roles in legacy accounts are not modified.

---

## Epic Description

Build the complete IAM foundation for AFT-managed accounts using the **four-tier layered SCP model** defined in ADR-002. This covers: IAM Identity Center with BU-aware permission sets, the full SCP hierarchy (Foundation → Compliance Envelope → BU Service Profiles → Environment Controls), baseline IAM roles deployed via AFT, cross-account trust policies, tag policies, and full AFT pipeline integration.

### Four-Tier SCP Model Summary

```
Tier 1a  Root              scp-org-integrity, scp-security-services-protect
Tier 1b  AFT-Managed       scp-region-lock (permissive floor), scp-root-and-access-baseline,
                           scp-aft-pipeline-protect
Tier 2   Regulated OU      scp-pci-network-controls, scp-pci-data-controls, scp-nist-controls
Tier 2.5 BU OUs            scp-deny-ai-services (Standard), scp-bu1-service-profile,
                           scp-bu2-service-profile, scp-bu3-service-profile,
                           scp-platform-service-profile
Tier 3   Env OUs           scp-prod-no-direct-deploy, scp-prod-delete-protect,
                           scp-prod-public-s3-deny, scp-staging-moderate
```

### Definition of Done

- IAM Identity Center configured with BU-aware permission sets assigned to groups
- All four SCP tiers deployed, tested in `PolicyStaging`, and attached to target OUs
- Platform OU confirmed as sibling of Standard — AI tools accessible to Platform, denied to product BUs
- BU2 international region access validated end-to-end
- Baseline IAM roles deployed to every AFT-vended account via AFT global customizations
- Cross-account trust matrix documented and implemented
- Tag policies enforced on the `AFT-Managed` branch
- All IAM constructs automated through AFT — zero manual setup on account vend

### External Dependencies

- **InfoSec Team** — authors, reviews, and approves all SCP modules (see LZ-106 in Epic 1)
- **BU Engineering Leads (BU1, BU2, BU3, Platform)** — workshop to define service profile scope per BU (input to Tier 2.5)
- **Compliance Team** — validates PCI-DSS and NIST control mapping for Tier 2 modules

### Blocked By

- Epic 1 (CT upgraded, new OU branch created: `Standard`, `Regulated`, `Platform` sub-OUs needed)
- Epic 2 (AFT repos for customization integration)
- Epic 8 (SCP lifecycle pipeline — Tier 2.5 BU profiles should be deployed through the SCP CI/CD pipeline from Epic 8)

### Blocks

- Epic 2, LZ-204 (smoke test validates IAM and SCP customizations)
- Epic 5 (security services rely on cross-account IAM roles)

---

## Stories

### LZ-401: Configure AWS IAM Identity Center with BU-Aware Permission Sets

**Story Points:** 5 · **Priority:** Highest · **Component:** IAM
**ADR Reference:** ADR-002, Layer 0 (human access model)

**Description:**
As a security engineer, I need IAM Identity Center configured with baseline permission sets that reflect our BU structure so that human access to all accounts goes through SSO with roles that match each persona's actual scope.

**Acceptance Criteria:**

- IAM Identity Center enabled in the management account (delegated admin to security account documented)
- Identity source configured and documented (built-in / AD / external IdP)
- Baseline permission sets created and mapped to BU groups:
  - `AdministratorAccess` — break-glass only, MFA enforced, session 1 hour
  - `PowerUserAccess` — broad non-IAM access, session 4 hours
  - `ReadOnlyAccess` — audit/inspection, session 8 hours
  - `PlatformEngineer` — infra-scoped, all AFT-managed accounts, session 4 hours
  - `DeveloperAccess-Standard` — application workload access for BU1/BU2/BU3 Dev + Staging
  - `DeveloperAccess-Prod` — read-only in Prod by default; deploy via CI/CD pipeline only
  - `DataEngineerAccess` — BU3-Data specific; includes Glue, EMR, Athena, Redshift access
  - `PlatformAIAccess` — Platform OU only; includes Bedrock, SageMaker access
- All permission sets assigned to groups (not individual users)
- Session durations configured per sensitivity level
- Permission boundary attached to all permission sets to prevent self-escalation

---

### LZ-402: Deploy Tier 1 Foundation SCPs (Root + AFT-Managed)

**Story Points:** 5 · **Priority:** Highest · **Component:** SCP
**External Dependency:** InfoSec Team
**ADR Reference:** ADR-002, Tier 1a and Tier 1b

**Description:**
As a security engineer, I need the universal Foundation SCPs deployed at Root and AFT-Managed OU so that every account in the org is protected by inescapable baseline controls, and every AFT-managed account is further constrained by region lock, root activity deny, and AFT pipeline protection.

**Acceptance Criteria:**

- **Tier 1a — Root (2 SCPs, 3 slots reserved):**
  - `scp-org-integrity`: deny leaving the Organization, deny account suspension
  - `scp-security-services-protect`: deny disabling CloudTrail, Config, GuardDuty, Security Hub
  - Both SCPs attached at Root and verified affecting all accounts (including legacy)
- **Tier 1b — AFT-Managed OU (3 SCPs, 2 slots reserved):**
  - `scp-region-lock`: deny actions outside approved regions — designed as a **permissive floor** (includes all regions any BU may need; individual BU SCPs narrow further)
  - `scp-root-and-access-baseline`: deny root user actions (except recovery); deny IAM user console creation (force SSO)
  - `scp-aft-pipeline-protect`: deny modification of AFT-managed roles, pipelines, Terraform state
- All SCPs codified in Terraform in the SCP library repo (Epic 8)
- All SCPs tested in `PolicyStaging` OU — positive and negative test results documented
- InfoSec Lead sign-off obtained before attaching to Root or AFT-Managed

**Critical notes:**
- `scp-region-lock` must include all regions BU2-International needs — validate with BU2 Engineering Lead before deploying
- Changes to Root-level SCPs require CISO awareness — blast radius is the entire org

---

### LZ-403: Deploy Tier 2 Compliance Envelope SCPs (Regulated OU)

**Story Points:** 5 · **Priority:** Highest · **Component:** SCP
**External Dependency:** InfoSec Team, Compliance Team
**ADR Reference:** ADR-002, Tier 2

**Description:**
As a security engineer, I need PCI-DSS and NIST 800-53 preventive controls deployed at the `Regulated` OU so that all BUs under it (e.g., BU-Payments) inherit hard compliance guardrails without those controls affecting Standard or Platform workloads.

**Acceptance Criteria:**

- `Regulated` OU created under `AFT-Managed/Workloads` and registered with Control Tower
- **Tier 2 — Regulated OU (3 SCPs, 2 slots reserved for SOC2/future):**
  - `scp-pci-network-controls`: deny IGW creation, public subnets, public RDS instances; deny non-approved PCI services (PCI 1.3, 1.4, 12.3)
  - `scp-pci-data-controls`: deny unencrypted S3/EBS/RDS/SQS; deny S3 replication to external accounts; deny Glacier vault lock removal (PCI 3.3–3.5, 10.7)
  - `scp-nist-controls`: deny CloudWatch log group deletion; deny KMS key deletion without approval; deny cross-region replication to unapproved regions; deny wildcard IAM policies (NIST AU-9, AU-11, SC-28, AC-6)
- All SCPs codified in Terraform
- Compliance team validates each SCP maps to a named PCI-DSS v4.0 or NIST SP 800-53 Rev.5 control
- Tested in `PolicyStaging`: attempt to create public subnet in Regulated context → denied; create encrypted RDS → allowed
- InfoSec Lead and Compliance team sign-off obtained
- `Standard` and `Platform` OUs confirmed as siblings — verify they do NOT inherit Tier 2 SCPs

---

### LZ-404: Deploy Tier 2.5 BU Service Profile SCPs

**Story Points:** 8 · **Priority:** High · **Component:** SCP
**External Dependency:** InfoSec Team, BU Engineering Leads (BU1, BU2, BU3, Platform)
**ADR Reference:** ADR-002, Tier 2.5

**Description:**
As a security engineer, I need BU-level service profile SCPs deployed so that each business unit is constrained to its intended AWS service footprint — AI tools are locked to Platform, BU1 is API-scoped, BU2 has full international region access, and BU3 can use data platform services but not AI inference.

**Acceptance Criteria:**

**OU Structure prerequisite:**
- `AFT-Managed/Workloads/Standard/` created with child OUs: `BU1-API`, `BU2-International`, `BU3-Data`
- `AFT-Managed/Workloads/Platform/` created as **sibling of Standard** (not a child of it)
- Verified: Platform OU inheritance path does NOT pass through Standard

**AI Services Deny (Standard OU — 1 SCP, 4 slots reserved):**
- `scp-deny-ai-services` attached at `Standard` OU:
  - Denies: `bedrock:*`, `sagemaker:InvokeEndpoint`, `sagemaker:InvokeEndpointAsync`, `comprehend:*`, `rekognition:*`, `polly:*`, `transcribe:*`, `textract:*`, `lex:*`, `kendra:*`
  - SageMaker *training* jobs NOT denied (BU3-Data may need ML training)
- Validated: account under `Standard/BU3-Data/Dev` cannot invoke Bedrock; account under `Platform/Dev` CAN invoke Bedrock

**BU1-API Service Profile (1 SCP at BU1-API OU):**
- `scp-bu1-service-profile`:
  - Denies: EMR, Glue, Athena, Redshift, MSK (Managed Kafka) — data platform services out of scope for API teams
  - Denies: international regions (e.g., `eu-west-1`, `ap-southeast-1`) — narrows AFT-Managed floor to domestic only
- Validated: BU1 Dev account cannot create an EMR cluster; cannot deploy to `eu-west-1`

**BU2-International Service Profile (1 SCP at BU2-International OU):**
- `scp-bu2-service-profile`:
  - Denies: EMR, Glue, Redshift (data warehouse not in BU2's scope)
  - No regional deny added — BU2 uses the full AFT-Managed approved region floor
- Validated: BU2 Dev account CAN deploy to `eu-west-1` and other approved international regions; BU1 Dev account cannot

**BU3-Data Service Profile (1 SCP at BU3-Data OU):**
- `scp-bu3-service-profile`:
  - Denies: CloudFront, large-scale API Gateway WebSocket APIs, AppSync, GameLift, Pinpoint, IoT Core — out-of-domain services
  - Denies: international regions (domestic only, same as BU1)
  - Does NOT deny: EMR, Glue, Athena, Redshift, Kinesis, SageMaker training — these are in-scope
- Validated: BU3 Dev account can create a Glue job and an EMR cluster; cannot invoke Bedrock (Standard deny); cannot deploy to `eu-west-1`

**Platform Service Profile (1 SCP at Platform OU):**
- `scp-platform-service-profile`:
  - Denies: business-product services not relevant to platform infra (GameLift, Pinpoint, retail-specific services)
  - No AI deny — Platform has access to Bedrock, SageMaker, etc.
- Validated: Platform Dev account CAN invoke Bedrock and SageMaker endpoints

**All profiles:**
- Codified in Terraform, PR-reviewed, tested in `PolicyStaging`
- InfoSec sign-off + BU Engineering Lead confirmation of service scope

---

### LZ-405: Deploy Tier 3 Environment Control SCPs

**Story Points:** 3 · **Priority:** High · **Component:** SCP
**External Dependency:** InfoSec Team
**ADR Reference:** ADR-002, Tier 3

**Description:**
As a security engineer, I need environment-tier SCPs deployed at Prod and Staging OUs under each BU so that production accounts enforce pipeline-only deployments and deletion protection, while Dev accounts remain unrestricted by environment controls.

**Acceptance Criteria:**

- Environment-tier OUs created under each BU OU: `Prod`, `Staging`, `Dev`
- **Prod OUs (3 SCPs, 2 slots reserved) — applies to all BU Prod OUs:**
  - `scp-prod-no-direct-deploy`: deny console-initiated changes to EC2, RDS, Lambda, ECS (enforce pipeline-only)
  - `scp-prod-delete-protect`: deny deletion of RDS instances, S3 buckets, KMS keys without MFA condition
  - `scp-prod-public-s3-deny`: deny public S3 ACLs and public bucket policies
- **Staging OUs (1 SCP, 4 slots reserved):**
  - `scp-staging-moderate`: deny public S3; deny delete without MFA; allow direct console deploys
- **Dev OUs: no Tier 3 attachments** — developers only feel Tier 1 + Tier 2.5 controls
- Validated: developer in `BU1-API/Prod` cannot run `aws lambda update-function-code` from CLI; same developer in `BU1-API/Dev` can

---

### LZ-406: Validate Full SCP Effective Policy Matrix

**Story Points:** 5 · **Priority:** High · **Component:** SCP
**ADR Reference:** ADR-002, Composition Matrix

**Description:**
As a platform engineer, I need end-to-end validation that every BU's effective SCP count and enforcement is correct per the ADR-002 Composition Matrix so that there are no gaps, unexpected denies, or slot overflows before teams onboard.

**Acceptance Criteria:**

- For each BU/environment combination, run the validation checklist:

| Account | Expected SCPs | AI Tools | Int'l Regions | Data Platform | Prod Deploy |
|---------|--------------|----------|---------------|---------------|-------------|
| BU1-API/Dev | 7 | ✗ Denied | ✗ Denied | ✗ Denied | N/A |
| BU1-API/Prod | 10 | ✗ Denied | ✗ Denied | ✗ Denied | Pipeline only |
| BU2-Intl/Dev | 7 | ✗ Denied | ✓ Allowed | ✗ Denied | N/A |
| BU2-Intl/Prod | 10 | ✗ Denied | ✓ Allowed | ✗ Denied | Pipeline only |
| BU3-Data/Dev | 7 | ✗ Denied | ✗ Denied | ✓ Allowed | N/A |
| BU3-Data/Prod | 10 | ✗ Denied | ✗ Denied | ✓ Allowed | Pipeline only |
| Platform/Dev | 6 | ✓ Allowed | Domestic | N/A | N/A |
| Platform/Prod | 9 | ✓ Allowed | Domestic | N/A | Pipeline only |
| Regulated/Prod | 11 | ✗ Denied | ✗ Denied | Restricted | Pipeline only |

- Each cell tested with a real API call or `aws iam simulate-principal-policy`
- No OU exceeds 5 direct SCP attachments — verified via `aws organizations list-policies-for-target`
- BU Engineering Leads for each BU sign off that their effective access matches expectations
- Results documented and attached to this story

---

### LZ-407: Create Baseline IAM Roles for Vended Accounts

**Story Points:** 5 · **Priority:** High · **Component:** IAM

**Description:**
As a platform engineer, I need baseline IAM roles deployed into every vended account via AFT global customizations so that automation, cross-account access, and break-glass access work from day one regardless of BU or environment.

**Acceptance Criteria:**

- Roles deployed via `aft-global-customizations` to every account:
  - `OrganizationAccountAccessRole` — exists by default; trust policy documented
  - `PlatformAutomationRole` — assumed by CI/CD pipelines; scoped to infra actions only
  - `SecurityAuditRole` — assumed by the security/audit account; read-only
  - `CostManagementRole` — assumed by the billing/FinOps account; cost explorer + tagging read
  - `BreakGlassRole` — emergency access; MFA required; CloudTrail alert on every assumption
- All roles tagged `managed-by: aft`, `tier: foundation`
- Permission boundary attached to `PlatformAutomationRole` — cannot create roles exceeding its own permissions
- Trust policies use `aws:PrincipalOrgID` condition on all roles
- Roles validated in a test account: assume each role, verify permitted and denied actions

---

### LZ-408: Implement Cross-Account Trust Policies

**Story Points:** 5 · **Priority:** High · **Component:** IAM

**Description:**
As a security engineer, I need a documented and automated cross-account trust model so that the security, network, and platform accounts can assume scoped roles in workload accounts without over-permissioning.

**Acceptance Criteria:**

- Trust matrix documented and codified:
  - Security account → `SecurityAuditRole` in all AFT-managed accounts
  - Network account → `NetworkManagementRole` in accounts needing TGW attachment management
  - CI/CD platform account → `PlatformAutomationRole` in all workload accounts
  - Management account → `OrganizationAccountAccessRole` (break-glass use only)
- All trust policies:
  - Use `aws:PrincipalOrgID` condition to prevent external account assumption
  - External ID or MFA conditions applied on `BreakGlassRole` and `OrganizationAccountAccessRole`
- End-to-end tested: security account assumes `SecurityAuditRole` in a workload account and can read CloudTrail; cannot write

---

### LZ-409: Implement Tag Policies at the Organization Level

**Story Points:** 3 · **Priority:** Medium · **Component:** IAM

**Description:**
As a FinOps/governance lead, I need tag policies enforced on the `AFT-Managed` branch so that all AFT-vended resources are consistently tagged for cost allocation, ownership, and compliance — including a `BusinessUnit` tag that maps accounts to the BU OU tier.

**Acceptance Criteria:**

- Tag policies codified in Terraform
- Mandatory tags enforced:
  - `Environment`: allowed values `dev`, `staging`, `prod`, `sandbox`
  - `CostCenter`: format enforced
  - `Owner`: email format
  - `ManagedBy`: allowed values `aft`, `terraform`, `manual`
  - `Application`: free text
  - `BusinessUnit`: allowed values `bu1-api`, `bu2-international`, `bu3-data`, `platform`, `regulated`, `shared`
- Tag policies attached to `AFT-Managed` OU (not root — legacy accounts excluded)
- AFT account request template updated to require `BusinessUnit` field
- Non-compliant resources surfaced via AWS Config Tag Editor or conformance pack

---

### LZ-410: Integrate Full IAM + SCP Stack into AFT Pipeline

**Story Points:** 8 · **Priority:** High · **Component:** IAM

**Description:**
As a platform engineer, I need all IAM constructs and SCP structures applied automatically when AFT vends a new account so that no manual IAM or SCP setup is ever required — the account's OU placement drives everything.

**Acceptance Criteria:**

- AFT account request template enforces required fields: `ou_path`, `business_unit`, `environment`, `cost_center`
- OU placement drives SCP inheritance automatically (no SCP attachment needed per-account)
- `aft-global-customizations` deploys to every account:
  - Baseline IAM roles (LZ-407)
  - Tag enforcement check (Config rule)
  - IAM compliance check (no root access keys, no overly broad roles)
- `aft-account-customizations` selects the correct permission set assignment based on `business_unit` tag
- Smoke test (LZ-204 in Epic 2) extended: vend a BU3-Data/Dev account, verify:
  - Correct OU placement (`AFT-Managed/Workloads/Standard/BU3-Data/Dev`)
  - Bedrock invocation denied (inherits `scp-deny-ai-services`)
  - Glue job creation allowed
  - International region (`eu-west-1`) denied
  - Baseline IAM roles present and tagged
  - SSO permission set `DeveloperAccess-Standard` assigned

---

## Summary

| Story | Title | Points | Priority | Ext. Dependency |
|-------|-------|--------|----------|-----------------|
| LZ-401 | SSO permission sets (BU-aware) | 5 | Highest | — |
| LZ-402 | Tier 1 Foundation SCPs | 5 | Highest | InfoSec |
| LZ-403 | Tier 2 Compliance Envelope SCPs | 5 | Highest | InfoSec, Compliance |
| LZ-404 | Tier 2.5 BU Service Profile SCPs | 8 | High | InfoSec, BU Leads |
| LZ-405 | Tier 3 Environment Control SCPs | 3 | High | InfoSec |
| LZ-406 | Validate full SCP effective policy matrix | 5 | High | BU Eng Leads |
| LZ-407 | Baseline IAM roles for vended accounts | 5 | High | — |
| LZ-408 | Cross-account trust policies | 5 | High | — |
| LZ-409 | Tag policies | 3 | Medium | — |
| LZ-410 | Integrate IAM + SCP into AFT pipeline | 8 | High | — |
| **Total** | | **52** | | |

## Sprint Sequencing

```
Sprint 3:
  LZ-401 (SSO permission sets)
  LZ-402 (Tier 1 SCPs) ← needs InfoSec; start immediately, gate on approval
  LZ-404 BU workshops (service profile scope alignment) ← run in parallel with LZ-402

Sprint 4:
  LZ-403 (Tier 2 Compliance SCPs) ← needs Compliance team
  LZ-404 (Tier 2.5 BU Service Profiles) ← unblocks after workshops
  LZ-405 (Tier 3 Environment Controls)
  LZ-407 (Baseline IAM roles)
  LZ-408 (Cross-account trust)

Sprint 5:
  LZ-406 (Validate full SCP matrix) ← all tiers must be deployed first
  LZ-409 (Tag policies)
  LZ-410 (AFT pipeline integration + smoke test)
```
