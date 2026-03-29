# EPIC 1: Control Tower Assessment, Upgrade & OU Foundation

**Epic Key:** LZ-CT
**Epic Name:** Assess, upgrade, and extend existing Control Tower for AFT readiness
**Priority:** Highest
**Labels:** `landing-zone`, `control-tower`, `governance`, `foundation`, `brownfield`
**Stories:** 7 · **Total Points:** 33
**Sprint Target:** Sprints 1–2

## Epic Description

The organization has an existing AWS Control Tower deployment on the billing/management account with an established OU structure, SCPs, tagging policies, and CloudFormation StackSets already in place. This epic focuses on assessing the current state, upgrading Control Tower to the latest landing zone version (4.x) to ensure AFT compatibility, and creating a **new parallel OU branch off root** for AFT-managed accounts — preserving the existing OU tree and its policies untouched.

The approach is explicitly **non-disruptive**: legacy accounts and their governance remain as-is. New AFT-vended accounts will live under the new OU branch with their own controls, SCPs, and customizations, allowing organic growth without risking existing workloads.

### Key Constraints

- **Brownfield environment** — Control Tower already enabled; existing OUs, SCPs, StackSets, and tagging policies must not be disturbed
- **InfoSec dependency** — SCPs are owned/reviewed by the InfoSec team. SCP stories carry an external dependency and require InfoSec review gates
- **Parallel OU strategy** — New OU branch off root isolates AFT-managed accounts from legacy

### Definition of Done

- Current CT state is documented (version, drift, existing controls, OU structure)
- CT landing zone is upgraded to latest version (4.x) compatible with AFT
- OU baselines are re-registered if required by the version jump
- A new top-level OU branch (e.g., `AFT-Managed`) is created under root with sub-OUs for workload stages
- Existing OU structure, SCPs, StackSets, and tagging policies are verified unchanged post-upgrade
- CT Controls (guardrails) on the new OU branch are codified in Terraform via `aws_controltower_control`
- InfoSec has reviewed and approved SCPs for the new OU branch

### External Dependencies

- **InfoSec Team** — SCP authoring, review, and approval for the new OU branch
- **Cloud Operations / Billing** — Management account access for CT upgrade
- **Existing OU Owners** — Validation that legacy OUs are unaffected post-upgrade

### Blocked By

None — this is the first epic.

### Blocks

- Epic 2 (AFT Deployment — requires upgraded CT + new OU branch)
- Epic 4 (IAM Foundation — SCPs and permission sets target new OUs)

---

## Stories

### LZ-101: Audit Existing Control Tower State & Document Baseline

**Story Points:** 5 · **Priority:** Highest · **Component:** Control Tower

**Description:**
As a platform engineer, I need a full audit of the current Control Tower deployment so that we understand the starting point — version, drift status, existing controls, OU structure, and any configuration gaps — before making changes.

**Acceptance Criteria:**

- Current CT landing zone version documented (2.x, 3.x, or 4.x)
- CT drift detection run and results documented (any drift = remediation story)
- Existing OU structure mapped (tree diagram or table)
- Existing SCPs inventoried per OU (policy names, attached OUs, deny effects)
- Existing StackSets inventoried per OU (names, target OUs, regions)
- Existing tagging policies inventoried
- Log Archive and Audit account status verified (active, receiving logs?)
- CloudTrail organization trail status verified
- Home region and governed regions documented
- Findings captured in a baseline assessment document

**Notes:** This is a read-only audit. No changes to CT in this story.

---

### LZ-102: Upgrade Control Tower Landing Zone to Latest Version (4.x)

**Story Points:** 8 · **Priority:** Highest · **Component:** Control Tower

**Description:**
As a platform engineer, I need Control Tower upgraded to the latest landing zone version so that we have AFT compatibility, API-based control management, and the latest features.

**Acceptance Criteria:**

- Pre-upgrade checklist completed:
  - All existing CT drift resolved (from LZ-101 findings)
  - Backup/snapshot of current SCPs and StackSet configurations
  - Change window communicated to stakeholders
- CT landing zone updated to latest 4.x version via the CT console or API
- If upgrading from 2.x to 3.x+: OU baselines re-registered (required for org-level CloudTrail trail migration)
- If upgrading from 3.x to 4.x: review release notes for breaking changes
- Post-upgrade validation:
  - All existing OUs and accounts remain in correct OUs
  - Existing SCPs still attached and functional
  - Existing StackSets still deployed and not drifted
  - CloudTrail organization trail active and delivering to Log Archive
  - No new drift detected in CT dashboard
- Upgrade version and date documented

**Risks:**

- 2.x → 3.x jump requires OU baseline re-registration which can take time for large orgs
- Existing Config rules or CloudTrail configurations may shift from account-level to org-level

**Notes:** Schedule during a low-change window. Coordinate with Cloud Operations team.

---

### LZ-103: Design New Parallel OU Branch for AFT-Managed Accounts

**Story Points:** 5 · **Priority:** Highest · **Component:** Control Tower

**Description:**
As a platform engineer, I need a new top-level OU branch off root designed and approved so that AFT-vended accounts are isolated from the existing OU tree and can evolve independently without impacting legacy accounts.

**Acceptance Criteria:**

- New OU branch design documented:
  - Top-level OU under root: `AFT-Managed` (or agreed name)
  - Sub-OUs:
    - `AFT-Managed/Infrastructure` — networking hub, shared services
    - `AFT-Managed/Sandbox` — developer experimentation
    - `AFT-Managed/Workloads/Dev`
    - `AFT-Managed/Workloads/Staging`
    - `AFT-Managed/Workloads/Prod`
    - `AFT-Managed/PolicyStaging` — SCP testing before promotion
  - Rationale documented for why parallel branch vs. nesting under existing OUs
- Design reviewed with:
  - Platform team (approval)
  - InfoSec team (SCP scoping discussion)
  - Existing OU owners (confirmation of no impact)
- Naming convention for new accounts documented (e.g., `aft-<team>-<env>-<purpose>`)

---

### LZ-104: Create and Register New OU Branch in Control Tower

**Story Points:** 3 · **Priority:** Highest · **Component:** Control Tower

**Description:**
As a platform engineer, I need the approved OU branch created in AWS Organizations and registered with Control Tower so that new OUs are governed and eligible for control (guardrail) attachment.

**Acceptance Criteria:**

- Top-level `AFT-Managed` OU created under root in AWS Organizations
- All sub-OUs created per the approved design (LZ-103)
- Each new OU registered with Control Tower (extends governance)
- Mandatory CT controls automatically applied upon registration
- Existing OUs verified unchanged (SCP attachments, StackSets, account membership)
- New OUs visible in the CT dashboard with "Registered" status

---

### LZ-105: Codify Control Tower Controls (Guardrails) as Terraform for New OUs

**Story Points:** 5 · **Priority:** High · **Component:** Control Tower

**Description:**
As a platform engineer, I need all Control Tower controls (preventive, detective, and proactive) for the new OU branch managed as Terraform code so that guardrails are version-controlled, peer-reviewed, and repeatable.

**Acceptance Criteria:**

- Terraform module created using `aws_controltower_control` resource
- Controls organized by type:
  - **Preventive (SCP-backed):** e.g., deny region usage outside approved list, deny root access key creation, deny CT/Config/GuardDuty disable
  - **Detective (Config rule-backed):** e.g., detect unencrypted EBS, detect public S3, detect IAM root access keys
  - **Proactive (CFN Hook-backed):** e.g., require encryption on new RDS instances, require tags on EC2 instances
- Controls are applied per OU (not at root level) targeting only the `AFT-Managed` branch
- Control identifiers (ARNs) sourced from the [AWS Control Tower controls reference](https://docs.aws.amazon.com/controltower/latest/controlreference/control-identifiers.html)
- Terraform state managed in a secure backend (S3 + DynamoDB)
- Terraform plan reviewed in PR before apply
- Applied controls verified in CT dashboard

**Notes:** Mandatory controls are auto-applied on OU registration and cannot be managed via Terraform. Only strongly recommended and elective controls need IaC management.

---

### LZ-106: Coordinate SCP Design with InfoSec Team for New OU Branch

**Story Points:** 5 · **Priority:** High · **Component:** Control Tower
**External Dependency:** InfoSec Team

**Description:**
As a platform engineer, I need to work with the InfoSec team to define, review, and approve SCPs for the new `AFT-Managed` OU branch so that governance policies meet organizational security requirements.

**Acceptance Criteria:**

- Kick-off meeting held with InfoSec to discuss:
  - Which existing SCPs (from legacy OUs) should be replicated or adapted for the new branch
  - Any new SCPs required for AFT-managed accounts
  - SCP testing strategy using the `PolicyStaging` OU
- InfoSec delivers or approves the following baseline SCPs for the new branch:
  - Region deny policy (approved regions only)
  - Deny leaving the Organization
  - Deny disabling security services (CloudTrail, Config, GuardDuty)
  - Deny root user actions (except account recovery)
  - Deny public S3 bucket creation
  - Deny IAM user creation with console access (force SSO)
  - Deny modification of AFT-managed resources (pipeline protection)
- SCPs codified in Terraform and version-controlled
- SCPs tested in `AFT-Managed/PolicyStaging` OU:
  - Positive tests: allowed actions succeed
  - Negative tests: denied actions blocked
- InfoSec sign-off obtained before SCPs are promoted to workload OUs

**Definition of Blocked:**
If InfoSec review is delayed, SCPs can be drafted by the platform team and held in `PolicyStaging` pending approval. Account vending (Epic 2) can proceed with mandatory CT controls only, but workload accounts should not move to `Prod` OUs without InfoSec-approved SCPs.

---

### LZ-107: Validate Legacy OU Integrity Post-Changes

**Story Points:** 2 · **Priority:** High · **Component:** Control Tower

**Description:**
As a platform engineer, I need to verify that all changes made in this epic (CT upgrade, new OUs, new controls) have not impacted the existing OU structure, SCPs, StackSets, or tagging policies.

**Acceptance Criteria:**

- Existing OU tree structure matches pre-change baseline (from LZ-101)
- All existing SCPs still attached to their original OUs with no policy drift
- All existing StackSets still targeting their original OUs and regions, no drift
- Existing tagging policies unchanged
- Spot-check 2–3 existing accounts:
  - CloudTrail still delivering logs
  - Config rules still evaluating
  - SCP deny effects still enforced
- Validation results documented and shared with existing OU owners

---

## Summary

| Story | Title | Points | Priority | External Dependency |
|-------|-------|--------|----------|---------------------|
| LZ-101 | Audit existing CT state | 5 | Highest | — |
| LZ-102 | Upgrade CT landing zone | 8 | Highest | Cloud Ops (change window) |
| LZ-103 | Design new OU branch | 5 | Highest | InfoSec (review) |
| LZ-104 | Create & register new OUs | 3 | Highest | — |
| LZ-105 | Codify CT controls as Terraform | 5 | High | — |
| LZ-106 | SCP design with InfoSec | 5 | High | **InfoSec Team** |
| LZ-107 | Validate legacy OU integrity | 2 | High | Existing OU owners |
| **Total** | | **33** | | |

## Brownfield Impact on Other Epics

> These notes summarize how the brownfield/parallel-OU approach affects Epics 2–6. Each epic file should be updated accordingly.

**Epic 2 (AFT Deployment):** AFT deploys into an account under the `AFT-Managed` branch. Account requests target sub-OUs under `AFT-Managed` only. Existing accounts are never touched by AFT.

**Epic 3 (Networking):** The network hub account is provisioned under `AFT-Managed/Infrastructure`. TGW is shared via RAM to the Organization, so spoke VPCs in both legacy and new accounts *could* attach — but initial scope is new accounts only.

**Epic 4 (IAM Foundation):** SSO permission sets are assigned to groups scoped to new accounts. SCPs are attached only to `AFT-Managed` sub-OUs. InfoSec dependency carries through — SCP stories in Epic 4 should also flag the external gate.

**Epic 5 (Security):** Security services (GuardDuty, Security Hub, Config) are likely already org-wide. Stories should verify existing coverage and ensure new accounts inherit enrollment automatically.

**Epic 6 (Documentation):** ADR must document the parallel-OU decision and rationale. Runbook must cover both legacy and new OU interaction patterns.

---

## References

- [AWS Control Tower Landing Zone Versions](https://docs.aws.amazon.com/controltower/latest/userguide/lz-version-selection.html)
- [Upgrade CT Landing Zone from 2.x to 3.x](https://repost.aws/articles/AROghsOAN3QwOpF9COQDsC3w/upgrade-control-tower-landing-zone-from-2-x-to-3-x)
- [Deploy and Manage CT Controls with Terraform](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/deploy-and-manage-aws-control-tower-controls-by-using-terraform.html)
- [aws-samples/aws-control-tower-controls-terraform (GitHub)](https://github.com/aws-samples/aws-control-tower-controls-terraform)
- [Register Existing OUs with Control Tower](https://docs.aws.amazon.com/controltower/latest/userguide/importing-existing.html)
- [Extend Governance to an Existing Organization](https://docs.aws.amazon.com/controltower/latest/userguide/about-extending-governance.html)
- [Best Practices for CT Administrators](https://docs.aws.amazon.com/controltower/latest/userguide/best-practices.html)
- [Organizing CT with Nested OUs](https://aws.amazon.com/blogs/mt/organizing-your-aws-control-tower-landing-zone-with-nested-ous/)
