# EPIC 8: SCP Lifecycle Management

**Epic Key:** LZ-SCPLCM
**Epic Name:** SCP-as-Code — dedicated repositories, CI/CD pipelines, validation, and rollback for SCP governance at scale
**Priority:** High
**Labels:** `landing-zone`, `scp`, `policy-as-code`, `ci-cd`, `governance`, `infosec`
**Stories:** 8 · **Total Points:** 44
**Sprint Target:** Sprints 3–4 (runs in parallel with Epic 4, must be ready before Tier 2.5 BU profiles are deployed)
**ADR References:** ADR-002 (`adrs/adr-002-scp-structure-multi-product-org.md`)

---

## Epic Description

As the organization's SCP library grows across four tiers and multiple BU service profiles, managing policies as manual Terraform edits without a structured pipeline introduces unacceptable blast-radius risk. A deny statement applied at the wrong OU, or a malformed SCP JSON, can break production workloads for an entire BU instantly — with no automated rollback.

This epic establishes the full SCP-as-Code lifecycle: dedicated Git repositories, a CI/CD pipeline with pre-deployment validation (including dry-run simulation via `aws iam simulate-principal-policy`), enforced PR review gates (InfoSec approval required), blast radius analysis before every apply, automated rollback on failure, and an audit trail connecting every policy change to a Jira ticket and a named approver.

The pipeline established here is the delivery mechanism for all SCP work in Epic 4 — Epic 4 stories author the policy content; Epic 8 provides the rails they run on.

---

### Definition of Done

- Dedicated SCP library Git repository exists with enforced branch protection and InfoSec-required PR approval
- CI pipeline runs on every PR: JSON lint, Terraform validate, blast radius report, SCP simulation
- CD pipeline deploys SCPs to `PolicyStaging` OU first, holds for manual gate, then promotes to production OUs
- Automated rollback reverts an SCP attachment within 5 minutes of a detected failure
- Every production SCP change is traceable to: PR author, InfoSec approver, Jira ticket, timestamp
- SCP slot utilization is tracked per OU and alarms trigger at 80% usage (4 of 5 slots)
- Runbook covers: normal change flow, emergency change, rollback procedure, slot exhaustion handling

---

### Key Design Principles

- **InfoSec is the only merge approver** — no SCP reaches any OU without InfoSec Lead sign-off on the PR
- **PolicyStaging first, always** — every module is deployed to the `PolicyStaging` OU and validated before any production OU
- **Dry-run before apply** — `aws iam simulate-principal-policy` runs automatically in CI against representative test principals for each affected OU tier
- **Blast radius before blast** — every PR includes an auto-generated report showing which accounts, BUs, and environments would be affected by the change
- **Rollback is automatic** — if a deployment causes a Config rule violation or a failed smoke test, the previous policy version is automatically re-attached
- **Emergency override is audited** — InfoSec Lead can deploy directly to console in an incident, but the IaC must be updated within 24 hours via a follow-up PR

---

### External Dependencies

- **InfoSec Team** — owns the repository, required PR approver on all merges
- **Platform Engineering** — builds and maintains the CI/CD pipeline, simulation tooling
- **Compliance Team** — reviews Tier 2 (PCI/NIST) module changes

### Blocked By

- Epic 1, LZ-104 (`PolicyStaging` OU must exist)
- Epic 2, LZ-202 (AFT Git repo patterns — SCP repo follows same structure)

### Blocks

- Epic 4 SCP stories (LZ-402 through LZ-405) — all SCP deployments must go through this pipeline

---

## Stories

### LZ-801: Create and Configure the SCP Library Git Repository

**Story Points:** 3 · **Priority:** Highest · **Component:** Repository

**Description:**
As an InfoSec engineer, I need a dedicated Git repository for the SCP policy library with enforced branch protection, directory structure, and ownership so that all policy changes are version-controlled, reviewed, and auditable.

**Acceptance Criteria:**

- Repository created (GitHub/GitLab/CodeCommit — matches AFT Git provider from Epic 2)
- Directory structure:
  ```
  scp-library/
  ├── policies/
  │   ├── tier1a-root/
  │   │   ├── scp-org-integrity.json
  │   │   └── scp-security-services-protect.json
  │   ├── tier1b-aft-managed/
  │   │   ├── scp-region-lock.json
  │   │   ├── scp-root-and-access-baseline.json
  │   │   └── scp-aft-pipeline-protect.json
  │   ├── tier2-regulated/
  │   │   ├── scp-pci-network-controls.json
  │   │   ├── scp-pci-data-controls.json
  │   │   └── scp-nist-controls.json
  │   ├── tier2.5-bu-profiles/
  │   │   ├── scp-deny-ai-services.json
  │   │   ├── scp-bu1-service-profile.json
  │   │   ├── scp-bu2-service-profile.json
  │   │   ├── scp-bu3-service-profile.json
  │   │   └── scp-platform-service-profile.json
  │   └── tier3-environment/
  │       ├── scp-prod-no-direct-deploy.json
  │       ├── scp-prod-delete-protect.json
  │       ├── scp-prod-public-s3-deny.json
  │       └── scp-staging-moderate.json
  ├── terraform/
  │   ├── main.tf         (policy + attachment resources)
  │   ├── variables.tf
  │   ├── outputs.tf
  │   └── ou-targets.tf   (OU ID references per environment)
  ├── tests/
  │   ├── simulate/       (simulation scripts per tier)
  │   └── scenarios/      (positive + negative test cases)
  └── docs/
      ├── module-catalog.md    (links to ADR-002 sections)
      └── change-process.md    (PR workflow, emergency procedure)
  ```
- Branch protection rules:
  - `main` branch: require PR, require 1 InfoSec Lead approval, no force push, no direct commits
  - `staging` branch: require PR, require 1 platform engineer approval
- CODEOWNERS file: InfoSec Lead owns `policies/`, `terraform/`; Platform owns `tests/`, CI config
- README documents the four-tier model, PR workflow, and emergency change procedure

---

### LZ-802: Build CI Pipeline — Lint, Validate, and Blast Radius Analysis

**Story Points:** 8 · **Priority:** Highest · **Component:** CI Pipeline

**Description:**
As a platform engineer, I need a CI pipeline that runs automatically on every PR to the SCP library so that policy authors get immediate feedback on JSON validity, Terraform correctness, and the blast radius of their proposed change before any human reviews it.

**Acceptance Criteria:**

- CI pipeline triggers on every PR (GitHub Actions / GitLab CI / CodeBuild):

  **Stage 1 — Lint and Validate:**
  - JSON schema validation on all `.json` files under `policies/` (reject malformed SCP JSON)
  - SCP character count check — fail if any policy exceeds 5,000 characters (warn at 4,500)
  - `terraform fmt` and `terraform validate` pass
  - `terraform plan` runs against a non-production AWS account — output attached to PR as a comment

  **Stage 2 — SCP Slot Utilization Check:**
  - Script reads current SCP attachment count per target OU via `aws organizations list-policies-for-target`
  - Fails the PR if adding the new policy would exceed 5 direct attachments on any OU
  - Reports current slot usage for all affected OUs as a PR comment

  **Stage 3 — Blast Radius Report:**
  - Script enumerates all accounts under each target OU using `aws organizations list-accounts-for-parent`
  - Generates a summary: "This change affects X accounts across Y OUs: [list]"
  - Report posted as a PR comment — InfoSec reviewer must acknowledge before approving
  - Changes to Tier 1a (Root) automatically flag as "CISO awareness required" in the PR

  **Stage 4 — Dry-Run Simulation:**
  - `aws iam simulate-principal-policy` runs against representative test principal ARNs (one per BU/env tier) using the proposed new policy as an additional context
  - Positive scenarios (actions that should be allowed) and negative scenarios (actions that should be denied) run from `tests/scenarios/`
  - If any positive scenario is unexpectedly denied, CI fails — the change would break legitimate access
  - If any negative scenario is unexpectedly allowed, CI fails — the change has a coverage gap

- All CI stages must pass before the PR is eligible for InfoSec review
- CI results visible in PR UI (green/red checks per stage)

---

### LZ-803: Build CD Pipeline — PolicyStaging Gate → Production Promotion

**Story Points:** 8 · **Priority:** Highest · **Component:** CD Pipeline

**Description:**
As a platform engineer, I need a CD pipeline that deploys every approved SCP change to the `PolicyStaging` OU first, validates it, waits for an explicit promotion approval, and then applies to production OUs — so that no policy ever reaches production accounts without having been validated in a live AWS environment first.

**Acceptance Criteria:**

- CD pipeline triggers on merge to `main` (or on manual trigger for emergency changes):

  **Stage 1 — Deploy to PolicyStaging:**
  - `terraform apply` scoped to `AFT-Managed/PolicyStaging` OU only
  - Deploys new or changed SCP modules to the staging OU

  **Stage 2 — Live Validation in PolicyStaging:**
  - Automated test suite runs against a real account in `PolicyStaging`:
    - Call denied actions → expect HTTP 403 with an `ExplicitDeny` in the response
    - Call allowed actions → expect success
  - Test results posted to Slack/Teams channel owned by InfoSec
  - If tests fail → pipeline halts, Terraform rollback applied to `PolicyStaging` automatically, alert sent

  **Stage 3 — Manual Promotion Gate:**
  - Pipeline pauses and requires explicit approval from InfoSec Lead (via pipeline UI or Slack approval workflow)
  - Approval request includes: change summary, blast radius report, PolicyStaging test results
  - Gate expires after 48 hours — if not approved, change is held in `PolicyStaging`

  **Stage 4 — Production Apply:**
  - On approval: `terraform apply` to production OUs in order:
    1. Dev/Sandbox OUs first (lowest risk)
    2. Staging OUs
    3. Prod OUs last
  - 5-minute pause between each tier — allows monitoring to detect any anomalies
  - If apply fails at any tier: automatic rollback of all tiers (re-attach previous version)

- Full deployment log stored in S3 with: PR link, approver, timestamps, plan output, apply output
- Every production apply generates a CloudTrail event traceable to the pipeline execution

---

### LZ-804: Implement Automated Rollback on Policy Failure

**Story Points:** 5 · **Priority:** High · **Component:** Safety

**Description:**
As a platform engineer, I need automated rollback of an SCP change if it causes a detected failure after deployment so that a bad policy does not persist in production accounts while humans debug it.

**Acceptance Criteria:**

- Rollback mechanism implemented:
  - **Trigger conditions:**
    - Post-deploy smoke test failures (defined test cases in `tests/scenarios/`)
    - AWS Config rule violation spike in affected accounts within 10 minutes of apply (e.g., sudden jump in `ACCESS_DENIED` events in CloudTrail)
    - Manual rollback triggered by InfoSec Lead via pipeline UI
  - **Rollback action:**
    - Previous SCP version (stored as a versioned S3 object by the CD pipeline) is re-applied via Terraform
    - Target OU is confirmed back to previous state via `aws organizations describe-policy`
    - Alert sent to InfoSec + Platform Slack channel with: what rolled back, when, why, and PR link
  - Rollback completes within **5 minutes** of trigger
- Previous SCP versions retained in S3 for 90 days (lifecycle policy)
- Rollback tested in a drill: intentionally deploy a breaking SCP to `PolicyStaging`, trigger rollback, verify recovery
- Rollback procedure documented in the runbook (LZ-808)

---

### LZ-805: Build SCP Simulation Test Suite

**Story Points:** 5 · **Priority:** High · **Component:** Testing

**Description:**
As a security engineer, I need a comprehensive SCP simulation test suite that validates every module's deny and allow behavior across all BU/environment combinations so that InfoSec can confidently approve changes knowing the full behavioral impact has been verified.

**Acceptance Criteria:**

- Test suite implemented under `tests/scenarios/`, structured by tier:
  - **Tier 1 tests:**
    - Positive: CloudTrail `PutEventSelectors` allowed → should succeed
    - Negative: `guardduty:DeleteDetector` → must be denied in all accounts
    - Negative: `organizations:LeaveOrganization` → must be denied everywhere
  - **Tier 2 tests (Regulated only):**
    - Negative: `ec2:CreateInternetGateway` in Regulated account → denied
    - Positive: `ec2:CreateInternetGateway` in Standard account → allowed
    - Negative: `s3:PutObject` without encryption header → denied
  - **Tier 2.5 tests (BU profiles):**
    - Negative: `bedrock:InvokeModel` from BU1-API/Dev account → denied
    - Positive: `bedrock:InvokeModel` from Platform/Dev account → allowed
    - Negative: `emr:RunJobFlow` from BU1-API account → denied
    - Positive: `emr:RunJobFlow` from BU3-Data account → allowed
    - Positive: `ec2:RunInstances` in `eu-west-1` from BU2-International account → allowed
    - Negative: `ec2:RunInstances` in `eu-west-1` from BU1-API account → denied
  - **Tier 3 tests:**
    - Negative: `lambda:UpdateFunctionCode` via console principal in Prod → denied
    - Positive: `lambda:UpdateFunctionCode` via `PlatformAutomationRole` in Prod → allowed
- Tests run via `aws iam simulate-principal-policy` using test principal ARNs per BU/env
- Tests integrated into CI pipeline (Stage 4 in LZ-802) and CD pipeline (Stage 2 in LZ-803)
- Test results output as JUnit XML for CI integration and PR comments
- New BU onboarding checklist includes: add simulation test cases for new BU profile to this suite

---

### LZ-806: Implement SCP Slot Utilization Monitoring and Alerting

**Story Points:** 3 · **Priority:** Medium · **Component:** Observability

**Description:**
As a platform engineer, I need continuous monitoring of SCP slot utilization per OU so that the team is alerted before any OU approaches the 5-attachment limit, avoiding a situation where an urgent security control cannot be deployed due to capacity.

**Acceptance Criteria:**

- Lambda function deployed (scheduled every 6 hours via EventBridge):
  - Calls `aws organizations list-policies-for-target` for every registered OU in the `AFT-Managed` branch
  - Records slot utilization (direct attachments / 5) per OU to CloudWatch custom metrics
- CloudWatch alarms:
  - WARNING: any OU reaches 4/5 slots used → SNS notification to InfoSec + Platform Slack channel
  - CRITICAL: any OU reaches 5/5 slots used → PagerDuty/on-call alert (no more room for new SCPs at this OU)
- CloudWatch dashboard: "SCP Slot Utilization" — shows current usage per OU tier, trend over 30 days
- Alarm tested: manually attach a dummy SCP to a test OU to hit 4/5, verify alert fires
- Slot utilization check also runs in CI (Stage 2 of LZ-802) — PR-time prevention plus runtime monitoring

---

### LZ-807: Establish SCP Change Audit Trail

**Story Points:** 5 · **Priority:** High · **Component:** Audit

**Description:**
As a compliance officer, I need every SCP change to have a complete, immutable audit trail — connecting the policy change to a Jira ticket, PR author, InfoSec approver, and deployment timestamp — so that during a PCI-DSS or NIST audit we can demonstrate that every preventive control change went through a formal review process.

**Acceptance Criteria:**

- Audit record written to a dedicated, tamper-resistant S3 bucket on every production deployment:
  ```json
  {
    "event_type": "scp_deployed",
    "timestamp": "2026-04-01T14:32:00Z",
    "scp_module": "scp-bu3-service-profile",
    "target_ou": "arn:aws:organizations::123456789:ou/o-xxx/ou-yyy",
    "change_type": "update",
    "pr_url": "https://github.com/org/scp-library/pull/42",
    "pr_author": "platform-engineer@org.com",
    "infosec_approver": "infosec-lead@org.com",
    "jira_ticket": "LZ-404",
    "pipeline_run_id": "build-1234",
    "previous_version_s3_key": "scp-versions/scp-bu3-service-profile/v2.json",
    "new_version_s3_key": "scp-versions/scp-bu3-service-profile/v3.json"
  }
  ```
- S3 bucket configuration:
  - Object lock enabled (WORM — write once, read many) — audit records cannot be deleted
  - SSE-KMS encryption
  - Access logs enabled
  - Lifecycle: retain for 7 years (PCI-DSS requirement)
- Jira ticket reference required field in PR description — CI fails if no ticket number found in PR body
- Monthly audit report auto-generated: count of SCP changes by tier, by approver, by BU affected
- Report shared with Compliance team and stored in the same audit bucket

---

### LZ-808: Document SCP Change Runbook and Emergency Procedure

**Story Points:** 7 · **Priority:** Medium · **Component:** Documentation

**Description:**
As a platform engineer, I need a complete operational runbook for the SCP lifecycle so that any engineer on the InfoSec or Platform team can safely make policy changes, respond to emergencies, and manage the pipeline without tribal knowledge.

**Acceptance Criteria:**

- Runbook written and stored in the `docs/change-process.md` of the SCP library repo and in the team wiki:

  **Normal Change Flow:**
  1. Engineer opens a Jira story for the SCP change
  2. Branch created from `main` in the SCP library repo
  3. Policy JSON updated; Terraform attachment updated if needed
  4. PR opened — CI runs automatically (lint → validate → blast radius → simulation)
  5. InfoSec Lead reviews CI output, blast radius report, and simulation results
  6. InfoSec Lead approves PR → merge to `main`
  7. CD pipeline deploys to `PolicyStaging` → runs live validation → pauses at promotion gate
  8. InfoSec Lead reviews live test results → approves promotion
  9. CD pipeline deploys to production OUs (Dev → Staging → Prod, with pauses)
  10. Audit record written; Jira ticket updated

  **Emergency Change Procedure (incident response):**
  1. InfoSec Lead applies SCP change directly via AWS console
  2. Immediate Slack/PagerDuty notification sent to Platform + CISO
  3. Within 24 hours: PR opened to reflect console change in IaC; CI/CD pipeline run to sync state
  4. Post-incident review within 48 hours: was the emergency justified? Can the pipeline be made faster?

  **Rollback Procedure:**
  1. Identify the failing SCP module and target OU
  2. Trigger manual rollback in the CD pipeline UI (or run `terraform apply` with previous `.tfvars`)
  3. Verify via `aws organizations list-policies-for-target` that previous policy is re-attached
  4. Open a Jira story for root cause analysis
  5. Do not re-deploy the failing version until root cause is resolved and simulation tests updated

  **Slot Exhaustion Procedure:**
  1. Alert fires: OU at 4/5 slots
  2. Review existing modules at that OU level — can any two be merged into one (within 5,120 char limit)?
  3. If new control is urgent: merge two existing modules; re-deploy
  4. If not urgent: schedule design review to reorganize OU tier structure

  **New BU Onboarding:**
  1. BU Engineering Lead workshop to define service profile scope
  2. InfoSec authors `scp-buN-service-profile.json`
  3. Add simulation test cases to `tests/scenarios/bu-N/`
  4. Normal change flow applies

- Runbook reviewed by InfoSec Lead, Platform Lead, and Compliance team
- Runbook linked from the SCP library repo README

---

## Summary

| Story | Title | Points | Priority | Ext. Dependency |
|-------|-------|--------|----------|-----------------|
| LZ-801 | SCP library Git repo setup | 3 | Highest | — |
| LZ-802 | CI pipeline — lint, validate, blast radius | 8 | Highest | — |
| LZ-803 | CD pipeline — PolicyStaging gate → prod | 8 | Highest | InfoSec |
| LZ-804 | Automated rollback on failure | 5 | High | — |
| LZ-805 | SCP simulation test suite | 5 | High | InfoSec, BU Leads |
| LZ-806 | Slot utilization monitoring + alerting | 3 | Medium | — |
| LZ-807 | SCP change audit trail | 5 | High | Compliance |
| LZ-808 | Runbook and emergency procedure | 7 | Medium | InfoSec, Platform |
| **Total** | | **44** | | |

## Sprint Sequencing

```
Sprint 3 (parallel with Epic 4 SCP authoring):
  LZ-801 (repo setup)           ← must exist before any SCP is written
  LZ-802 (CI pipeline)          ← unblocks InfoSec authoring flow
  LZ-805 (simulation test suite) ← needed by CI and CD pipelines

Sprint 4:
  LZ-803 (CD pipeline + PolicyStaging gate)  ← needed before Tier 2.5 BU profiles deploy
  LZ-804 (automated rollback)
  LZ-806 (slot monitoring)
  LZ-807 (audit trail)

Sprint 5:
  LZ-808 (runbook)              ← written last, captures everything learned in Sprints 3–4
```

## Relationship to Epic 4

Epic 8 provides the infrastructure; Epic 4 provides the content. The dependency is:

```
Epic 8, LZ-801 + LZ-802 (repo + CI) must complete BEFORE
  → Epic 4, LZ-402 (Tier 1 SCPs authored and PR'd through the pipeline)

Epic 8, LZ-803 (CD pipeline) must complete BEFORE
  → Epic 4, LZ-404 (Tier 2.5 BU profiles deployed to production OUs)
```

No SCP from Epic 4 should be deployed directly via console or ad-hoc Terraform — the pipeline in Epic 8 is the only path to production.
