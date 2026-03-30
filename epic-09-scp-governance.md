# EPIC 9: SCP Governance

**Epic Key:** LZ-SCP
**Epic Name:** Deploy and validate the four-tier SCP governance model across all AFT-managed OUs
**Priority:** Highest
**Labels:** `landing-zone`, `scp`, `governance`, `compliance`, `pci-dss`, `nist`
**Stories:** 6 · **Total Points:** 31
**Sprint Target:** Sprints 3–5
**ADR References:** ADR-002 (`adrs/adr-002-scp-structure-multi-product-org.md`)

> **Relationship to other epics:** SCP authoring, review, and attachment is in this epic. The CI/CD pipeline that manages the SCP lifecycle (lint, simulation, staged rollout, rollback, audit trail) is in **Epic 8**. Both epics run in parallel — Epic 8 builds the delivery mechanism; Epic 9 authors and deploys the policy content through it.

---

## Brownfield Context

The existing OU tree already has SCPs on legacy OUs — those are **untouched**. All SCPs in this epic target only the `AFT-Managed` OU branch and its sub-OUs. SCPs are **InfoSec-owned** — every story in this epic carries an external dependency on InfoSec review and sign-off. BU service profile stories (LZ-903) also require BU Engineering Lead workshops. Root-level SCP changes (Tier 1a) require CISO awareness due to org-wide blast radius.

---

## Epic Description

Deploy the complete four-tier layered SCP model defined in **ADR-002** across the `AFT-Managed` OU branch. This covers: Foundation controls at Root and AFT-Managed (Tier 1), PCI-DSS and NIST compliance guardrails at the Regulated OU (Tier 2), BU-specific service profile denies at Standard/BU OUs and Platform (Tier 2.5), environment-tier deployment controls at Prod/Staging OUs (Tier 3), and a full matrix validation across every BU/environment combination. All SCPs are deployed through the Epic 8 SCP lifecycle pipeline.

### Four-Tier SCP Architecture (ADR-002)

```
Root (Tier 1a)
├── scp-org-integrity                    ← deny leave org, deny account suspension
└── scp-security-services-protect        ← deny disabling CT/Config/GD/SH

AFT-Managed OU (Tier 1b)
├── scp-region-lock                      ← permissive floor; BU OUs narrow further
├── scp-root-and-access-baseline         ← deny root actions; force SSO
└── scp-aft-pipeline-protect             ← deny modification of AFT roles and state

  Workloads/Regulated OU (Tier 2)
  ├── scp-pci-network-controls
  ├── scp-pci-data-controls
  └── scp-nist-controls

  Workloads/Standard OU (Tier 2.5 — AI deny for all product BUs)
  └── scp-deny-ai-services

    BU1-API OU (Tier 2.5 — BU service profile)
    └── scp-bu1-service-profile          ← deny data services + int'l regions

    BU2-International OU (Tier 2.5)
    └── scp-bu2-service-profile          ← deny data warehouse; no regional deny

    BU3-Data OU (Tier 2.5)
    └── scp-bu3-service-profile          ← deny non-data services + int'l regions

  Workloads/Platform OU (Tier 2.5 — sibling of Standard; no AI deny inherited)
  └── scp-platform-service-profile       ← deny out-of-domain product services

    [BU*/Prod OUs] (Tier 3)
    ├── scp-prod-no-direct-deploy
    ├── scp-prod-delete-protect
    └── scp-prod-public-s3-deny

    [BU*/Staging OUs] (Tier 3)
    └── scp-staging-moderate
```

### Definition of Done

- All four tiers deployed, tested in `PolicyStaging`, and promoted to production OUs
- Platform OU confirmed as **sibling** of Standard — AI tool access verified for Platform, denied for all Standard BUs
- BU2 international region access validated end-to-end
- No OU exceeds 5 direct SCP attachments
- Full matrix validation completed and signed off by BU Engineering Leads
- All SCPs authored in the Epic 8 SCP Git repository and deployed through the CI/CD pipeline
- InfoSec Lead sign-off on all tiers; Compliance team sign-off on Tier 2

### External Dependencies

- **InfoSec Team** — authors, reviews, and approves all SCP modules (primary owner)
- **BU Engineering Leads (BU1, BU2, BU3, Platform)** — workshop to define service profile scope (input to LZ-903)
- **Compliance Team** — validates PCI-DSS v4.0 and NIST SP 800-53 Rev.5 control mapping for LZ-902
- **CISO** — must be aware of Root-level SCP changes (LZ-901) before deployment

### Blocked By

- Epic 1 (sub-OUs `Standard`, `Regulated`, `Platform`, BU sub-OUs, and environment sub-OUs must exist)
- Epic 8 (SCP lifecycle pipeline must exist before SCPs are deployed through it — at minimum LZ-801/LZ-802 for Git structure + CI)

### Blocks

- Epic 4, LZ-402 (baseline IAM roles rely on `scp-aft-pipeline-protect` to prevent deletion)
- Epic 2, LZ-204 (smoke test validates SCP enforcement for the `BU3-Data/Dev` vend scenario)

---

## Stories

---

### LZ-901: Deploy Tier 1 Foundation SCPs (Root + AFT-Managed)

**Story Points:** 5 · **Priority:** Highest · **Component:** SCP
**External Dependency:** InfoSec Team, CISO awareness (Root-level)
**ADR Reference:** ADR-002, Tier 1a and Tier 1b

**Description:**
As a security engineer, I need universal Foundation SCPs deployed at Root and AFT-Managed OU so that every account in the org is protected by inescapable baseline controls, and every AFT-managed account is additionally constrained by region lock, root activity deny, and AFT pipeline protection.

**Acceptance Criteria:**

**Tier 1a — Root OU (2 SCPs; 3 slots reserved for future):**

`scp-org-integrity`:
- Deny `organizations:LeaveOrganization` — all principals
- Deny `organizations:CloseAccount` — all principals
- Deny `organizations:DeleteOrganizationalUnit` — all principals
- Exception: management account service principal

`scp-security-services-protect`:
- Deny `cloudtrail:DeleteTrail`, `cloudtrail:StopLogging`, `cloudtrail:UpdateTrail`
- Deny `config:DeleteConfigurationRecorder`, `config:StopConfigurationRecorder`
- Deny `guardduty:DeleteDetector`, `guardduty:DisassociateFromMasterAccount`
- Deny `securityhub:DisableSecurityHub`, `securityhub:DeleteHub`
- Deny `controltower:*` (non-CT service principal)

**Tier 1b — AFT-Managed OU (3 SCPs; 2 slots reserved):**

`scp-region-lock` — **permissive floor design:**
- Deny `"*"` with `StringNotIn aws:RequestedRegion` for the approved region list
- Approved list includes ALL regions any BU may need (including `eu-west-1` for BU2 International)
- Individual BU-level SCPs (LZ-903) will add restrictive regional denies for domestic-only BUs
- Exception: global services (`iam`, `route53`, `cloudfront`, `sts`, `support`, `organizations`, `waf`)

`scp-root-and-access-baseline`:
- Deny `"*"` when `aws:PrincipalIsAWSService = false` and root user detected (deny root API actions)
- Deny `iam:CreateUser`, `iam:CreateLoginProfile`, `iam:CreateAccessKey` — enforce SSO-only human access
- Exception: `iam:CreateServiceLinkedRole` (required by many services)

`scp-aft-pipeline-protect`:
- Deny `iam:DeleteRole` on roles matching `arn:aws:iam::*:role/AWSAFTExecution*` and `arn:aws:iam::*:role/PlatformAutomation*`
- Deny `iam:PutRolePolicy`, `iam:DetachRolePolicy` on the above ARN patterns
- Deny `codepipeline:DeletePipeline`, `codepipeline:StopPipelineExecution` on AFT pipeline names
- Deny `s3:DeleteBucket` on Terraform state buckets (tagged `aft-managed = true`)

**Deployment process:**
- All SCPs authored in the SCP Git repo (Epic 8, LZ-801) and merged via PR
- CI pipeline runs (Epic 8, LZ-802): JSON lint → Terraform validate → blast radius check → `simulate-principal-policy`
- Deployed to `PolicyStaging` OU first; positive and negative simulation results documented
- InfoSec Lead sign-off obtained; CISO notified for Root-level SCPs
- Manual promotion approval gate passed (Epic 8, LZ-803) before attaching to Root and AFT-Managed

**Critical notes:**
- `scp-region-lock` must include all regions BU2-International needs — validate with BU2 Engineering Lead before authoring
- Changes to Root-level SCPs have org-wide blast radius — treat as a production change event

---

### LZ-902: Deploy Tier 2 Compliance Envelope SCPs (Regulated OU)

**Story Points:** 5 · **Priority:** Highest · **Component:** SCP
**External Dependency:** InfoSec Team, Compliance Team
**ADR Reference:** ADR-002, Tier 2

**Description:**
As a security engineer, I need PCI-DSS and NIST 800-53 preventive controls deployed at the `Regulated` OU so that compliance-sensitive workloads (e.g., BU-Payments) inherit hard guardrails automatically — without those controls affecting Standard or Platform workloads.

**Acceptance Criteria:**

**OU prerequisite:**
- `AFT-Managed/Workloads/Regulated` OU exists and is registered with Control Tower (Epic 1)
- Confirmed: Standard and Platform OUs do NOT inherit Tier 2 SCPs

**Tier 2 — Regulated OU (3 SCPs; 2 slots reserved for SOC2 or future frameworks):**

`scp-pci-network-controls` (PCI-DSS v4.0: Req 1.3, 1.4, 12.3):
- Deny `ec2:AttachInternetGateway`, `ec2:CreateInternetGateway`
- Deny `rds:CreateDBInstance` with `PubliclyAccessible = true`
- Deny `ec2:ModifySubnetAttribute` enabling public IPv4 assignment
- Deny creation of resources in non-approved PCI services (explicitly enumerated)

`scp-pci-data-controls` (PCI-DSS v4.0: Req 3.3–3.5, 10.7):
- Deny `s3:PutBucketEncryption` without `aws:kms` algorithm
- Deny `s3:PutReplicationConfiguration` to buckets in external account IDs
- Deny `ec2:CreateVolume` without `Encrypted = true`
- Deny `rds:CreateDBInstance` without `StorageEncrypted = true`
- Deny `sqs:SetQueueAttributes` without `SqsManagedSseEnabled`
- Deny `glacier:DeleteVaultAccessPolicy`, `glacier:AbortVaultLock`

`scp-nist-controls` (NIST SP 800-53 Rev.5: AU-9, AU-11, SC-28, AC-6):
- Deny `logs:DeleteLogGroup`, `logs:DeleteLogStream`
- Deny `kms:ScheduleKeyDeletion` without `aws:MultiFactorAuthPresent`
- Deny `kms:DisableKey`
- Deny `s3:PutBucketReplication` to non-approved regions
- Deny `iam:PutUserPolicy` with `"*"` resource (wildcard IAM deny)

**Validation:**
- Compliance team maps each SCP statement to a named PCI-DSS v4.0 or NIST SP 800-53 Rev.5 control requirement
- Simulation tests: attempt to create public subnet in Regulated context → denied; create encrypted RDS → allowed
- InfoSec Lead and Compliance team sign-off documented in PR

---

### LZ-903: Deploy Tier 2.5 BU Service Profile SCPs

**Story Points:** 8 · **Priority:** High · **Component:** SCP
**External Dependency:** InfoSec Team, BU Engineering Leads (BU1, BU2, BU3, Platform)
**ADR Reference:** ADR-002, Tier 2.5

**Description:**
As a security engineer, I need BU-level service profile SCPs deployed so that each business unit is constrained to its intended AWS service footprint — AI tools are locked to Platform, BU1 is API-scoped with domestic-only regions, BU2 has full international region access, BU3 can use data platform services but not AI inference, and Platform has full AI access without inheriting the Standard-level AI deny.

**Acceptance Criteria:**

**OU structure prerequisite:**
- `AFT-Managed/Workloads/Standard/` with child OUs: `BU1-API`, `BU2-International`, `BU3-Data`
- `AFT-Managed/Workloads/Platform/` as a **sibling of Standard** (not a child of Standard)
- OU tree verified: Platform's inheritance path does NOT pass through Standard
- BU Engineering Lead workshops completed — each BU's intended service footprint signed off

**Step 1 — AI Services Deny at Standard OU (1 SCP; 4 slots reserved for BU-specific):**

`scp-deny-ai-services` attached at `Standard` OU:
- Denies: `bedrock:*`, `sagemaker:InvokeEndpoint`, `sagemaker:InvokeEndpointAsync`
- Denies: `comprehend:*`, `rekognition:*`, `polly:*`, `transcribe:*`, `textract:*`, `lex:*`, `kendra:*`
- SageMaker *training* (`sagemaker:CreateTrainingJob`, `sagemaker:CreateProcessingJob`) **NOT denied** (BU3-Data needs ML training)
- Validation: account under `Standard/BU3-Data/Dev` → `bedrock:InvokeModel` → **Denied**
- Validation: account under `Platform/Dev` → `bedrock:InvokeModel` → **Allowed** (no Standard deny inherited)

**Step 2 — BU1-API Service Profile (1 SCP at BU1-API OU):**

`scp-bu1-service-profile`:
- Denies: `elasticmapreduce:*`, `glue:*`, `athena:*`, `redshift:*`, `kafka:*` (MSK)
- Denies international regions: `StringNotIn aws:RequestedRegion` → domestic-approved list only (narrows AFT-Managed floor)
- Validation: BU1 Dev cannot create EMR cluster; cannot deploy to `eu-west-1`

**Step 3 — BU2-International Service Profile (1 SCP at BU2-International OU):**

`scp-bu2-service-profile`:
- Denies: `elasticmapreduce:*`, `redshift:*` (data warehouse not in BU2 scope)
- **No regional deny added** — BU2 inherits the full AFT-Managed approved region floor
- Validation: BU2 Dev CAN deploy to `eu-west-1`; BU1 Dev cannot

**Step 4 — BU3-Data Service Profile (1 SCP at BU3-Data OU):**

`scp-bu3-service-profile`:
- Denies: `cloudfront:*`, `gamelift:*`, `pinpoint:*`, `iot:*`, `connect:*`
- Denies international regions (same domestic-only list as BU1)
- Does NOT deny: `elasticmapreduce:*`, `glue:*`, `athena:*`, `redshift:*`, `kinesis:*`, `sagemaker:CreateTrainingJob`
- Validation: BU3 Dev can create Glue job and EMR cluster; cannot invoke Bedrock; cannot deploy to `eu-west-1`

**Step 5 — Platform Service Profile (1 SCP at Platform OU):**

`scp-platform-service-profile`:
- Denies: `gamelift:*`, `pinpoint:*`, `connect:*`, retail-specific services not relevant to platform infra
- No AI deny, no regional deny (Platform needs broad access)
- Validation: Platform Dev CAN invoke `bedrock:InvokeModel` and `sagemaker:InvokeEndpoint`

**All profiles:**
- Codified in Terraform in SCP Git repo
- Authored, PR-reviewed, and InfoSec-approved before deployment
- BU Engineering Lead sign-off on each profile's deny list before attaching

---

### LZ-904: Deploy Tier 3 Environment Control SCPs

**Story Points:** 3 · **Priority:** High · **Component:** SCP
**External Dependency:** InfoSec Team
**ADR Reference:** ADR-002, Tier 3

**Description:**
As a security engineer, I need environment-tier SCPs at Prod and Staging OUs so that production accounts enforce pipeline-only deployments and deletion protection, while Dev accounts remain unrestricted by environment controls — keeping developer velocity high while hardening production.

**Acceptance Criteria:**

**OU prerequisite:**
- Environment OUs (`Prod`, `Staging`, `Dev`) created under each BU OU

**Prod OUs (3 SCPs; 2 slots reserved):** Applied to `BU1-API/Prod`, `BU2-International/Prod`, `BU3-Data/Prod`, `Platform/Prod`

`scp-prod-no-direct-deploy`:
- Deny console-user-initiated `ec2:RunInstances`, `rds:CreateDBInstance`, `lambda:UpdateFunctionCode`, `ecs:UpdateService`, `ecs:RegisterTaskDefinition`
- Condition: `StringNotLike aws:PrincipalArn` → CI/CD pipeline role ARNs are exempt
- **TEAM alignment (Epic 10):** `JIT-DeployOverride` role ARN must also be exempted — use tag-based condition `aws:PrincipalTag/access-pattern = "jit"` instead of hardcoding ARNs, so new JIT permission sets are automatically allowed
- Validation: developer SSO session in `BU1-API/Prod` → `aws lambda update-function-code` → **Denied**
- Validation: CI/CD pipeline role in `BU1-API/Prod` → `aws lambda update-function-code` → **Allowed**
- Validation: TEAM `JIT-DeployOverride` session in `BU1-API/Prod` → `aws lambda update-function-code` → **Allowed** (tag condition met)

`scp-prod-delete-protect`:
- Deny `rds:DeleteDBInstance`, `rds:DeleteDBCluster` without `aws:MultiFactorAuthPresent`
- Deny `s3:DeleteBucket` without `aws:MultiFactorAuthPresent`
- Deny `kms:ScheduleKeyDeletion`, `kms:DisableKey` without MFA
- Deny `ec2:TerminateInstances` for instances tagged `protect-from-termination = true`

`scp-prod-public-s3-deny`:
- Deny `s3:PutBucketAcl` with public ACL values
- Deny `s3:PutBucketPolicy` that would enable public access (enforced via S3 Block Public Access as well, belt-and-suspenders)
- Deny `s3:DeletePublicAccessBlock`

**Staging OUs (1 SCP; 4 slots reserved):**

`scp-staging-moderate`:
- Deny public S3 (same as `scp-prod-public-s3-deny`)
- Deny delete operations without MFA (same as `scp-prod-delete-protect`)
- Allow direct console deploys (no pipeline-only restriction — staging allows manual testing)

**Dev OUs: no Tier 3 attachments** — developers experience only Tier 1 + Tier 2.5 controls for maximum velocity

**Validation:**
- Developer in `BU1-API/Prod` → `aws lambda update-function-code` → Denied
- Same developer in `BU1-API/Dev` → `aws lambda update-function-code` → Allowed (no Tier 3)
- CI/CD role in `BU1-API/Prod` → deploy via pipeline → Allowed

---

### LZ-905: Validate Full SCP Effective Policy Matrix

**Story Points:** 5 · **Priority:** High · **Component:** SCP / Validation
**ADR Reference:** ADR-002, Composition Matrix

**Description:**
As a platform engineer, I need end-to-end validation that every BU's effective SCP count and enforcement is correct per the ADR-002 Composition Matrix so that there are no gaps, unexpected denies, or slot overflows before teams begin onboarding workloads.

**Acceptance Criteria:**

**For every BU/environment combination, execute the validation checklist:**

| Account | SCP Count | AI Tools | Int'l Regions | Data Platform | Prod Deploy |
|---------|-----------|----------|---------------|---------------|-------------|
| BU1-API / Dev | 7 | ✗ Denied | ✗ Denied | ✗ Denied | N/A |
| BU1-API / Staging | 8 | ✗ Denied | ✗ Denied | ✗ Denied | Allowed direct |
| BU1-API / Prod | 10 | ✗ Denied | ✗ Denied | ✗ Denied | Pipeline only |
| BU2-Intl / Dev | 7 | ✗ Denied | ✓ Allowed | ✗ Denied | N/A |
| BU2-Intl / Staging | 8 | ✗ Denied | ✓ Allowed | ✗ Denied | Allowed direct |
| BU2-Intl / Prod | 10 | ✗ Denied | ✓ Allowed | ✗ Denied | Pipeline only |
| BU3-Data / Dev | 7 | ✗ Denied | ✗ Denied | ✓ Allowed | N/A |
| BU3-Data / Staging | 8 | ✗ Denied | ✗ Denied | ✓ Allowed | Allowed direct |
| BU3-Data / Prod | 10 | ✗ Denied | ✗ Denied | ✓ Allowed | Pipeline only |
| Platform / Dev | 6 | ✓ Allowed | Domestic | N/A | N/A |
| Platform / Prod | 9 | ✓ Allowed | Domestic | N/A | Pipeline only |
| Regulated / Prod | 11 | ✗ Denied | ✗ Denied | Restricted | Pipeline only |

**Validation method per cell:**
- API call attempted via `aws iam simulate-principal-policy` with the appropriate principal ARN and action
- Or direct API call from a test EC2 instance / Lambda in the target account
- Result (Allow / Deny) documented against expected result; mismatches are blockers

**Slot count verification:**
- `aws organizations list-policies-for-target --target-id <OU_ID> --filter SERVICE_CONTROL_POLICY`
- Verify no OU has more than 5 direct SCP attachments
- Inherited SCPs are not counted (confirmed per AWS documentation)

**Sign-off required:**
- BU Engineering Lead for each BU reviews and approves their row in the matrix
- InfoSec Lead confirms slot counts and overall model integrity
- Results attached to this story as a validation report artifact

---

### LZ-906: Integrate SCP Model into AFT Pipeline

**Story Points:** 5 · **Priority:** High · **Component:** SCP / AFT

**Description:**
As a platform engineer, I need the SCP inheritance model wired into the AFT account vending process so that OU placement is the single mechanism that drives SCP enforcement — no per-account SCP attachments, no manual SCP steps after vend.

**Acceptance Criteria:**

**OU path as the control plane:**
- AFT `aft-account-request` `ou_name` field drives all SCP inheritance automatically
- No SCP is ever attached directly to an account; only to OUs (SCPs on accounts consume one of the account's 5 slots and are hard to audit)
- AFT pre-account-provisioning hook validates `ou_name` is a recognized path under `AFT-Managed/Workloads/` before provisioning begins

**New account vend smoke test (extends LZ-204 in Epic 2):**
Vend a new `BU3-Data/Dev` account:
- [ ] Account lands in `AFT-Managed/Workloads/Standard/BU3-Data/Dev`
- [ ] Effective SCPs confirmed: 7 SCPs inherited from Root + AFT-Managed + Standard + BU3-Data + Dev chain
- [ ] `bedrock:InvokeModel` → Denied (inherits `scp-deny-ai-services` via Standard)
- [ ] `glue:CreateJob` → Allowed (not in BU3's deny list)
- [ ] `eu-west-1` deployment → Denied (inherits BU3's regional deny)
- [ ] `lambda:UpdateFunctionCode` from console → Allowed (Dev OU has no Tier 3 SCP)
- [ ] Slot count for `BU3-Data/Dev` OU: 0 direct SCPs (all inherited); no slot overflow

**Pre-vend OU validation hook (AFT account-request pre-hook Lambda):**
- Validates `ou_name` exists in the approved OU path list
- Validates the environment sub-OU matches the `environment` tag value
- Returns hard failure if path is invalid — prevents account from being placed in wrong OU (which would give wrong SCP inheritance)

**Runbook updated:**
- "Adding a new BU" — onboarding steps: new BU OU, new BU service profile SCP, BU workshop, matrix validation row
- "Changing a BU's service profile" — PR in SCP repo → CI pipeline → InfoSec review → PolicyStaging → prod promotion via Epic 8 pipeline

---

## Summary

| Story | Title | Points | Priority | Ext. Dependency |
|-------|-------|--------|----------|-----------------|
| LZ-901 | Tier 1 Foundation SCPs (Root + AFT-Managed) | 5 | Highest | InfoSec, CISO |
| LZ-902 | Tier 2 Compliance Envelope SCPs (Regulated) | 5 | Highest | InfoSec, Compliance |
| LZ-903 | Tier 2.5 BU Service Profile SCPs | 8 | High | InfoSec, BU Leads |
| LZ-904 | Tier 3 Environment Control SCPs | 3 | High | InfoSec |
| LZ-905 | Validate full SCP effective policy matrix | 5 | High | BU Eng Leads |
| LZ-906 | Integrate SCP model into AFT pipeline | 5 | High | — |
| **Total** | | **31** | | |

## Sprint Sequencing

```
Sprint 3 (parallel with Epic 4):
  LZ-901 (Tier 1 SCPs)           ← InfoSec engagement starts here; highest blast radius; start early
  BU service profile workshops    ← informal; needed before LZ-903 can be authored

Sprint 4:
  LZ-902 (Tier 2 Compliance SCPs) ← needs Compliance team; parallel with Tier 2.5 authoring
  LZ-903 (Tier 2.5 BU Profiles)  ← workshops done in Sprint 3; InfoSec authors in Sprint 4
  LZ-904 (Tier 3 Env Controls)   ← lightweight; parallel with LZ-902/903

Sprint 5:
  LZ-905 (Matrix validation)      ← all tiers must be deployed first
  LZ-906 (AFT pipeline integration + smoke test)
```

## Key Constraints Carried from ADR-002

| Constraint | Implication |
|-----------|-------------|
| Max 5 direct SCP attachments per OU | Every tier has reserved slots — see ADR-002 Composition Matrix |
| Inherited SCPs don't count toward the 5-limit | Plan composition at direct-attachment level only |
| Parent deny cannot be overridden by child allow | All denies must be placed at the correct tier; no "override" mechanism exists |
| Platform must be sibling of Standard | Placing Platform under Standard would inherit `scp-deny-ai-services`; OU tree must be correct before any SCP is attached |
| Region-lock is a permissive floor | AFT-Managed allows all org-approved regions; individual BU OUs restrict to their subset |
