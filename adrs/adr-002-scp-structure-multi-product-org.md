# ADR-002: SCP Structure for a Multi-Product, Business-Unit-Driven Organization

**Status:** Proposed
**Date:** 2026-03-29
**Deciders:** InfoSec Lead, Platform Engineering Lead, BU Engineering Leads, Cloud Operations
**Related Epic:** Epic 4 — IAM Foundation · Epic 1 — Control Tower & OU Foundation

---

## Context

### The Problem

The organization operates multiple business units (e.g., Payments, Lending, Analytics, Operations) with varied engineering teams, different risk and compliance postures, and different rates of change. Some BUs operate under PCI-DSS and NIST 800-53 compliance frameworks; others do not.

InfoSec owns all SCPs centrally, but there is no consistent structural pattern for how SCPs are authored, applied, or maintained. The current tension is:

- **Too tight:** SCPs block legitimate product team actions, creating friction and driving shadow workarounds
- **Too loose:** Without a consistent model, controls are inconsistent across BUs, making audits painful
- **No layering:** When a new compliance requirement arrives, there's no clean place to add it without touching every OU's policies

The core design question has two dimensions:

1. **Compliance posture:** how do we enforce org-wide baselines and scope PCI/NIST controls precisely to regulated workloads without strangling non-regulated teams?
2. **Service identity:** how do we deny services that are out of scope for a given BU — so an API-heavy team can't accidentally spin up a Redshift cluster, a data platform team can use AI/ML tools while those same tools are locked down for other BUs, and an international BU isn't constrained by domestic-only region locks that apply to others?

These two dimensions require two different structural patterns. Compliance posture is handled by OU depth (regulated vs standard). Service identity requires a **BU-level OU layer** where each BU's permitted service footprint is explicitly bounded.

### Key Constraints

- InfoSec authors, reviews, and approves all SCPs — product teams do not write or attach SCPs directly
- AWS Organizations limits: **up to 5 SCPs directly attached per entity** (root, OU, or account), 5,120 characters per SCP, max 500 SCPs per org
- **The 5-SCP limit applies to direct attachments only — inherited SCPs from parent OUs do NOT count toward it.** An account in `AFT-Managed/Workloads/Regulated/Prod` is governed by SCPs from every level (Root, AFT-Managed, Regulated, Prod), but each level can independently attach up to 5. The effective policy is the intersection of all of them.
- SCPs can only DENY — they cannot grant permissions
- SCP inheritance: effective permissions = intersection of all SCPs in the path from root → OU → account
- A Deny at any level in the hierarchy cannot be overridden by a lower-level SCP or IAM policy
- The 5-per-level limit means module catalogs must be designed to fit within 5 direct attachments per OU tier — individual modules may be consolidated into themed groupings (within the 5,120 char limit) to stay within this ceiling while reserving slots for future controls
- The new `AFT-Managed` OU branch (from Epic 1) is the primary target; legacy OUs are out of scope for this ADR
- Compliance frameworks in scope: **PCI-DSS v4.0** and **NIST SP 800-53 Rev. 5**

### Forces at Play

- **Autonomy:** product teams need freedom to experiment in dev/sandbox environments
- **Security:** regulated workloads (PCI, NIST) require hard preventive controls, not just detective ones
- **Service scoping:** BUs have distinct technology footprints — an API-heavy BU should not be able to spin up EMR or Redshift; a Platform AI team should not be blocked by denies applied to product BUs
- **International constraints:** a globally operating BU cannot be subject to the same domestic-only region lock that applies to domestic BUs — but the region lock must still exist for domestic teams
- **AI access control:** AI/ML services (Bedrock, SageMaker inference) are approved for Platform teams only; denying them across all product BUs is a hard security requirement driven by data governance and cost control
- **Maintainability:** InfoSec should be able to add, update, or revoke a control in one place and have it propagate correctly
- **Auditability:** every SCP must have a clear owner, rationale, and compliance mapping
- **Speed:** adding a new BU or product should not require InfoSec to author a net-new set of SCPs from scratch

---

## Decision

**Adopt a four-tier layered SCP model using a centrally maintained, modular SCP library — where each SCP has a single responsibility, each OU tier inherits progressively stricter controls, compliance-specific controls are scoped precisely to regulated OUs, and BU service profiles define per-BU service footprint boundaries.**

The four tiers are:

```
Tier 1 — FOUNDATION (Root / AFT-Managed OU)
  Applied universally. Cannot be escaped by any account.
  Includes: region lock, org integrity, root activity, security services, AFT pipeline protection.
  Owned by: InfoSec. Changed only via a formal change request.
  ⚠ Region lock at AFT-Managed is a PERMISSIVE FLOOR — allows all regions any BU might
    ever need. Individual BU OUs narrow this further with additive restrictions.

Tier 2 — COMPLIANCE ENVELOPE (Regulated sub-OU only)
  Applied to BUs under PCI-DSS / NIST. Standard, Platform, and other BUs never inherit these.
  Owned by: InfoSec in coordination with Compliance team.

Tier 2.5 — BU SERVICE PROFILE (per-BU OUs under Standard / Regulated)
  NEW TIER. Defines which AWS service classes each BU is permitted to use.
  AI services denied at Standard parent — Platform (a sibling, not a child) escapes this.
  BU1 (API-heavy) gets data platform service restrictions.
  BU2 (International) gets a wider region floor via no additional region deny.
  BU3 (Data-heavy) gets no data service restrictions but gets API-infrastructure restrictions.
  Platform gets no AI deny (it lives outside Standard).
  Owned by: InfoSec, with BU Engineering Lead input on service scope.

Tier 3 — ENVIRONMENT CONTROLS (Prod / Staging / Dev / Sandbox OUs)
  Applied to an environment stage. Prod is strictest; Sandbox is lightest.
  Owned by: InfoSec, with platform team proposing changes via PR.
```

### Critical Structural Decision: Platform as a Sibling, Not a Child

This is the single most important structural insight in this model.

**Problem:** If AI services are denied at the `Standard` OU level, every child OU under Standard — including Platform — inherits that deny and cannot escape it (a child SCP can never override a parent deny).

**Solution:** Platform is placed as a **sibling of Standard** under `Workloads`, not a child of it. This means the AI deny at `Standard` is never in Platform's inheritance path.

```
AFT-Managed/Workloads/
  ├── Regulated/          ← PCI/NIST BUs land here
  │   └── BU-Payments/
  ├── Standard/           ← AI deny lives HERE (applies to all BU children)
  │   ├── BU1-API/
  │   ├── BU2-International/
  │   └── BU3-Data/
  └── Platform/           ← SIBLING of Standard — AI deny never inherited
```

### Critical Structural Decision: Region Lock as a Permissive Floor

**Problem:** BU2 is international and needs regions beyond a domestic-only approved list. But the region lock at `AFT-Managed` is inherited by all BUs including BU2. A child SCP cannot relax a parent deny.

**Solution:** The `scp-region-lock` at `AFT-Managed` is designed as a **permissive floor** — it denies truly off-limits regions (sanctioned countries, unapproved regions org-wide) but allows all regions that any BU might legitimately need. Individual BU OUs then **add further restrictions** to narrow their allowed regions. Since SCPs are ANDed, a domestic-only BU with an additional regional deny is effectively constrained to domestic regions. BU2, with no further regional SCP, operates across the full approved floor.

```
AFT-Managed scp-region-lock: deny all except [us-east-1, us-west-2, eu-west-1, ap-southeast-1, ...]
                                                          ↓ (inheritance, cannot override)
BU1-API scp-bu1-service-profile: further deny [eu-west-1, ap-southeast-1] → effective = domestic only
BU2-International: (no extra region deny) → effective = full approved floor
BU3-Data: further deny [eu-west-1, ap-southeast-1] → effective = domestic only
```

Each tier is implemented as a **library of small, focused SCP modules** — not one monolithic policy per OU. Modules are composed per OU by attaching the appropriate subset. Each module has a single clear responsibility, a descriptive name, and a compliance tag.

---

## SCP Library: Module Catalog

> **Important — the 5 SCP attachment limit:** Each OU can have at most 5 SCPs directly attached. Inherited SCPs from parent OUs are evaluated but do not count toward this limit at the child level. The catalog below is designed with this constraint in mind: related controls are consolidated into themed groupings where needed, and each tier deliberately reserves 1–2 attachment slots for future controls.

### Tier 1a — Universal Foundation (attached at **Root**)

Applies to every account in the entire organization, including legacy OUs. These are the two controls that should truly be inescapable everywhere — including accounts outside the AFT-Managed branch.

| Module Name | Purpose | Compliance Mapping | Slots Used |
|-------------|---------|-------------------|------------|
| `scp-org-integrity` | Deny leaving the Organization, deny account suspension | NIST AC-2, PCI 12.3 | 1 of 5 |
| `scp-security-services-protect` | Deny disabling CloudTrail, Config, GuardDuty, Security Hub | NIST AU-2, PCI 10.2 | 2 of 5 |
| *(3 slots reserved for future org-wide controls)* | | | |

**Why only 2 at root?** Controls like region-lock and force-SSO must NOT apply to legacy accounts that have existing IAM users or operate in multiple regions. Scoping them to the AFT-Managed OU protects legacy environments from unintended breakage.

### Tier 1b — AFT-Managed Foundation (attached at **AFT-Managed OU**)

Applies to all AFT-managed accounts. Inherited by every child OU below.

| Module Name | Purpose | Compliance Mapping | Slots Used |
|-------------|---------|-------------------|------------|
| `scp-region-lock` | Deny all actions outside approved regions | NIST SC-7, PCI 1.3 | 1 of 5 |
| `scp-root-and-access-baseline` | Deny root user actions (except recovery); deny IAM user console creation (force SSO) | NIST AC-6, IA-2, PCI 7.2, 8.2 | 2 of 5 |
| `scp-aft-pipeline-protect` | Deny modification of AFT-managed roles, pipelines, and Terraform state | Internal — pipeline integrity | 3 of 5 |
| *(2 slots reserved)* | | | |

> **Consolidation note:** `scp-root-activity-deny` and `scp-deny-iam-user-console` from the original draft are combined into `scp-root-and-access-baseline`. Both are access-integrity controls and their deny statements together fit well within the 5,120-character limit.

### Tier 2 — Compliance Envelope (attached at **Regulated OU**)

Applies only to BUs operating under PCI-DSS and/or NIST. Standard BUs never inherit these.
Original 7 individual modules are consolidated into 3 themed groupings to stay within the 5-attachment limit while reserving 2 slots for future frameworks (e.g., SOC 2, HIPAA).

| Module Name | Contains | Compliance Mapping | Slots Used |
|-------------|---------|-------------------|------------|
| `scp-pci-network-controls` | Deny IGW creation, public subnets, public RDS; deny non-approved PCI services | PCI 1.3, 1.4, 12.3 | 1 of 5 |
| `scp-pci-data-controls` | Deny unencrypted S3/EBS/RDS/SQS; deny S3 replication to external accounts; deny Glacier vault lock removal | PCI 3.3, 3.4, 3.5, 10.7 | 2 of 5 |
| `scp-nist-controls` | Deny deletion of CloudWatch log groups; deny KMS key deletion without approval; deny cross-region replication to unapproved regions; deny wildcard IAM policies | NIST AU-9, AU-11, SC-28, SC-36, AC-6 | 3 of 5 |
| *(2 slots reserved for SOC2, future frameworks)* | | | |

> **Consolidation note:** Each grouped policy combines logically related deny statements. PCI network and data controls are distinct enough to warrant separation (different audit domains). NIST controls are consolidated into one since they share the audit/data theme. Each combined policy remains within the 5,120-character limit.

### Tier 2.5 — BU Service Profile (attached at per-BU OUs)

Each BU OU gets one service profile SCP that defines which AWS service classes are **out of scope** for that BU. The profile is authored collaboratively by InfoSec and the BU Engineering Lead. The goal is not to micro-restrict — it's to prevent BUs from accidentally (or intentionally) using expensive, high-risk, or data-sensitive services outside their domain.

> **AI services are denied at the `Standard` OU level** — not per-BU — because the deny needs to apply uniformly to all product BUs. Platform, as a sibling OU, inherits nothing from Standard and is not subject to this deny.

#### Standard OU — AI Services Deny (attached at **Standard OU**, affects all BUs under it)

| Module Name | Purpose | Slots Used |
|-------------|---------|------------|
| `scp-deny-ai-services` | Deny Amazon Bedrock, SageMaker inference endpoints (`InvokeEndpoint`, `InvokeEndpointAsync`), Comprehend, Rekognition, Polly, Transcribe, Textract, Lex, Kendra | 1 of 5 (Standard OU) |

> **Scope note:** SageMaker *training* jobs are NOT denied here — BU3-Data may need ML training. Only inference/invocation APIs are denied. SageMaker Studio and notebook instances are also denied since they are AI-interaction surfaces. The Platform OU, as a sibling, is entirely unaffected.

#### BU1-API Service Profile (attached at **BU1-API OU**)

BU1 is API-heavy: API Gateway, Lambda, DynamoDB, ElastiCache, SQS, SNS, AppSync. It does not need big-data processing infrastructure or extended international regions.

| Module Name | Denies | Slots Used |
|-------------|--------|------------|
| `scp-bu1-service-profile` | Deny EMR, Glue, Athena, Redshift, MSK (Managed Kafka), Kinesis Data Firehose at scale; deny `eu-west-1`, `ap-southeast-1` and other international regions (narrows the AFT-Managed floor to domestic only) | 1 of 5 (BU1-API OU) |

#### BU2-International Service Profile (attached at **BU2-International OU**)

BU2 operates internationally — no additional region restriction is added here, giving it the full approved region floor from `AFT-Managed`. It may restrict some services irrelevant to international ops.

| Module Name | Denies | Slots Used |
|-------------|--------|------------|
| `scp-bu2-service-profile` | Deny EMR, Glue, Redshift (heavy data warehouse not in BU2's domain); no region restrictions added — BU2 uses the full AFT-Managed approved region list | 1 of 5 (BU2-International OU) |

> **Key point:** Because BU2 adds NO regional deny SCP, and the AFT-Managed floor allows international regions, BU2 effectively operates multi-region. Domestic BUs (BU1, BU3) add regional deny SCPs in their profile that subtract international regions from their effective set.

#### BU3-Data Service Profile (attached at **BU3-Data OU**)

BU3 is data-heavy: Glue, EMR, Athena, Redshift, Kinesis, S3, SageMaker (training only — inference denied at Standard). It does not need API infrastructure at scale and operates domestically.

| Module Name | Denies | Slots Used |
|-------------|--------|------------|
| `scp-bu3-service-profile` | Deny CloudFront, API Gateway (WebSocket/HTTP large-scale), AppSync; deny `eu-west-1`, `ap-southeast-1` and other international regions (domestic only); deny IoT Core, GameLift, Pinpoint (out-of-domain services) | 1 of 5 (BU3-Data OU) |

#### Platform Service Profile (attached at **Platform OU**)

Platform is a sibling of Standard — it does NOT inherit the AI services deny. Platform teams have access to AI tools. The platform profile restricts services that are product-BU concerns, not platform infrastructure concerns.

| Module Name | Denies | Slots Used |
|-------------|--------|------------|
| `scp-platform-service-profile` | Deny direct provisioning of business-logic workload services (Retail, GameLift, Pinpoint campaigns); no AI deny; domestic region floor (same as BU1/BU3 unless platform needs international) | 1 of 5 (Platform OU) |

> **Slot reservation:** Each BU OU uses 1 of 5 slots for its service profile, leaving 4 slots available for future use — for example, a second BU-specific module if the profile becomes too large to fit in 5,120 characters.

### Tier 3 — Environment Controls (attached at environment-tier OUs)

Directly attached at Prod, Staging, Dev, or Sandbox OUs. Inherits everything from Tier 1a + 1b above.

| Module Name | Applied To | Purpose | Slots Used |
|-------------|-----------|---------|------------|
| `scp-prod-no-direct-deploy` | Prod OUs | Deny console-based changes to EC2, RDS, Lambda — enforce pipeline-only deploys | 1 of 5 |
| `scp-prod-delete-protect` | Prod OUs | Deny deletion of databases, S3 buckets, and KMS keys without MFA | 2 of 5 |
| `scp-prod-public-s3-deny` | Prod OUs | Deny public S3 ACLs and bucket policies | 3 of 5 |
| *(2 slots reserved)* | | | |
| `scp-staging-moderate` | Staging OUs | Deny public S3; deny delete without MFA; allow direct console deploys | 1 of 5 |
| *(4 slots reserved)* | | | |
| `scp-sandbox-open` | Sandbox OU | Explicit marker — no Tier 3 controls; only Tier 1a/1b inherited | 1 of 5 |
| *(4 slots reserved)* | | | |

> Dev OUs carry no Tier 3 attachments by default — developers only feel Tier 1a and 1b Foundation controls, giving genuine freedom to experiment within the org-wide baseline.

---

## OU × SCP Composition Matrix

This table shows which modules are **directly attached** at each OU level. Inherited SCPs from parent levels don't count toward the 5-per-level limit but ARE evaluated — effective access is the intersection of all SCPs from Root down to the account.

```
OU Level                    Direct Attachments                      Slots  Effective total
─────────────────────────────────────────────────────────────────────────────────────────────────
Root                        scp-org-integrity                        2/5   2
                            scp-security-services-protect

AFT-Managed                 scp-region-lock (permissive floor)       3/5   5
                            scp-root-and-access-baseline
                            scp-aft-pipeline-protect

  Workloads/Regulated       scp-pci-network-controls                 3/5   8
                            scp-pci-data-controls
                            scp-nist-controls

    Regulated/BU-Payments   (no direct — inherits compliance env.)   0/5   8

      BU-Payments/Prod      scp-prod-no-direct-deploy                3/5   11  ← strictest
                            scp-prod-delete-protect
                            scp-prod-public-s3-deny

      BU-Payments/Staging   scp-staging-moderate                     1/5   9

      BU-Payments/Dev       (no direct)                              0/5   8

  Workloads/Standard        scp-deny-ai-services                     1/5   6
                            (AI deny flows to all BU children)

    Standard/BU1-API        scp-bu1-service-profile                  1/5   7
                            (data platform deny + domestic regions)

      BU1-API/Prod          scp-prod-no-direct-deploy                3/5   10
                            scp-prod-delete-protect
                            scp-prod-public-s3-deny

      BU1-API/Staging       scp-staging-moderate                     1/5   8

      BU1-API/Dev           (no direct)                              0/5   7

    Standard/BU2-Intl       scp-bu2-service-profile                  1/5   7
                            (data warehouse deny, NO region deny)

      BU2-Intl/Prod         scp-prod-no-direct-deploy                3/5   10
                            scp-prod-delete-protect
                            scp-prod-public-s3-deny

      BU2-Intl/Staging      scp-staging-moderate                     1/5   8

      BU2-Intl/Dev          (no direct)                              0/5   7
                            ← can use all approved international regions

    Standard/BU3-Data       scp-bu3-service-profile                  1/5   7
                            (API infra deny + domestic regions)

      BU3-Data/Prod         scp-prod-no-direct-deploy                3/5   10
                            scp-prod-delete-protect
                            scp-prod-public-s3-deny

      BU3-Data/Staging      scp-staging-moderate                     1/5   8

      BU3-Data/Dev          (no direct)                              0/5   7
                            ← data engineers can use Glue/EMR/Athena
                              but NOT AI inference (Standard deny)

  Workloads/Platform        scp-platform-service-profile             1/5   6
                            ← NO AI deny inherited (sibling of Standard)
                            ← Platform teams CAN use Bedrock/SageMaker

    Platform/Prod           scp-prod-no-direct-deploy                3/5   9
                            scp-prod-delete-protect
                            scp-prod-public-s3-deny

    Platform/Staging        scp-staging-moderate                     1/5   7

    Platform/Dev            (no direct)                              0/5   6
                            ← most capable non-regulated dev environment

AFT-Managed/Sandbox         scp-sandbox-open (marker)                1/5   6

AFT-Managed/PolicyStaging   (modules under test)                   0-5/5  (test bed)
```

**Reading the matrix:** Follow any account's OU path from Root down — every SCP attached at each level is evaluated. The path to a `BU3-Data/Dev` account gives 7 effective SCPs (2 root + 3 AFT-Managed + 1 Standard AI deny + 1 BU3 service profile). A `BU-Payments/Prod` account has 11. A `Platform/Dev` engineer has just 6 and can use AI tools because Platform never passes through Standard.

**Slot headroom summary:**

| OU Level | Slots Used | Slots Reserved |
|----------|------------|----------------|
| Root | 2 | 3 |
| AFT-Managed | 3 | 2 |
| Regulated | 3 | 2 |
| Standard | 1 | 4 |
| Each BU OU | 1 | 4 |
| Platform | 1 | 4 |
| Prod (any) | 3 | 2 |
| Staging (any) | 1 | 4 |

---

## Options Considered

### Option A: Monolithic SCPs Per OU (current ad-hoc approach)

| Dimension | Assessment |
|-----------|------------|
| Complexity | Low to author initially; High to maintain |
| Auditability | Poor — large policies are hard to audit against compliance requirements |
| Team autonomy | Poor — blunt instrument; hard to be precise |
| Scalability | Poor — adding a new BU or compliance requirement means editing multiple large documents |
| InfoSec overhead | High — every change requires careful editing of existing policies |

**Pros:** Simple to understand initially. No new patterns to learn.

**Cons:** Policies grow large and brittle. Compliance mapping is buried inside monolithic documents. Adding a new BU requires copying and modifying existing SCPs, creating drift. When PCI-DSS v4.0 adds a new control requirement, InfoSec must find and update every affected OU's policy. No reuse. Fails at scale.

---

### Option B: Flat Root-Level SCP (everything at root)

| Dimension | Assessment |
|-----------|------------|
| Complexity | Low |
| Auditability | Medium — easy to see what applies, hard to understand who it affects |
| Team autonomy | Very poor — one size fits all; regulated controls apply even to Sandbox |
| Scalability | Poor — root SCPs affect every account including Log Archive, Audit, AFT management |
| InfoSec overhead | Medium — one place to edit, but unintended blast radius is high |

**Pros:** Easy to enforce universally. Minimal structural design required.

**Cons:** PCI encryption mandates applying to a developer's sandbox account prevent any unencrypted S3 bucket for development purposes — creating huge friction. Blunt controls at root cannot be relaxed for lower-risk accounts. Violates the principle of least friction for non-regulated workloads.

---

### Option C: Layered Modular SCP Library with OU Inheritance **(Recommended)**

| Dimension | Assessment |
|-----------|------------|
| Complexity | Medium to design; Low to operate once established |
| Auditability | Excellent — each module maps to a specific control; compliance reports are straightforward |
| Team autonomy | Good — non-regulated, non-prod teams only feel Foundation controls |
| Scalability | Excellent — new BU = assign existing modules to new OUs; new compliance req = new module |
| InfoSec overhead | Low — update one module and it propagates to all attached OUs automatically |

**Pros:** Each module is independently auditable and maps directly to a compliance control. Adding a new BU is a configuration change (attach modules to new OU), not an authoring exercise. PCI controls only apply where they're needed. Prod and sandbox accounts have genuinely different postures. Terraform-manageable (`aws_organizations_policy` + `aws_organizations_policy_attachment`). Policy-staging OU enables safe testing of new modules before production rollout.

**Cons:** Initial design effort to define the module catalog and composition rules. Requires discipline to keep modules small and single-purpose (governance overhead). Teams need to understand which OU tier they're in to understand their constraints.

---

## Trade-off Analysis

The central trade-off is **initial design effort vs. long-term maintainability**. Options A and B are faster to start but accumulate structural debt rapidly as the org scales — each new product, BU, or compliance update becomes progressively harder to handle cleanly.

Option C requires a considered upfront design (the module catalog and composition matrix), but once established, every future change has a clear, predictable home. A new PCI control becomes a new module added to the Regulated tier. A new BU with standard workloads just needs its new OUs assigned the appropriate tier modules. InfoSec reviews one focused policy document rather than hunting through monolithic per-OU policies.

The character limit concern (5,120 per SCP) actually *favors* modularity — small focused policies are less likely to hit the limit than monolithic ones. And with up to 5 SCPs per OU, there's sufficient room for Tier 1 (6 modules, attached at the top) plus Tier 2 and Tier 3 rollups without hitting attachment limits, because Tier 1 is inherited, not re-attached at each level.

**Recommendation: Option C.**

---

## SCP Governance Model

Because InfoSec owns all SCPs centrally, the following governance process applies:

**For new SCP modules:**
1. Platform team or BU identifies a gap and opens a request (Jira story or RFC)
2. InfoSec drafts the module with a compliance mapping
3. Module is tested in `AFT-Managed/PolicyStaging` OU — positive tests (allowed actions succeed) and negative tests (denied actions blocked) must both pass
4. InfoSec Lead approves and merges to the SCP library repo
5. Terraform applies the attachment to the target OU(s) via the AFT customization pipeline

**For changes to existing modules:**
- Same PR process; changes require InfoSec Lead review
- Blast radius analysis required before apply: which accounts are currently attached to this module?
- Changes to Tier 1 Foundation modules require CISO awareness (these affect all accounts)

**For emergency changes (incident response):**
- InfoSec Lead can apply directly via AWS console with mandatory post-incident PR to reflect the change in code within 24 hours

**For BU-specific exceptions:**
- BU requests a specific action be allowed that a module is blocking
- InfoSec evaluates: can the module be scoped more precisely via `aws:RequestedRegion`, resource ARN conditions, or tag conditions?
- If yes: module is updated with the condition. If no: escalate to CISO for exception approval.
- No BU-specific SCPs are created — all changes go into the shared library

---

## Terraform Implementation Pattern

Each SCP module is a separate `.json` policy file in a Git repository owned by InfoSec. Terraform manages creation and attachment:

```hcl
# Each module is a separate policy
resource "aws_organizations_policy" "scp_pci_encryption_mandate" {
  name        = "scp-pci-encryption-mandate"
  description = "PCI-DSS 3.4/3.5: Deny unencrypted S3, EBS, RDS, SQS. Scope: Regulated OU."
  type        = "SERVICE_CONTROL_POLICY"
  content     = file("${path.module}/policies/scp-pci-encryption-mandate.json")

  tags = {
    tier              = "compliance"
    compliance-framework = "PCI-DSS-v4"
    control-id        = "PCI-3.4,3.5"
    owner             = "infosec"
    managed-by        = "terraform"
  }
}

# Attachment is explicit and reviewable
resource "aws_organizations_policy_attachment" "regulated_pci_encryption" {
  policy_id = aws_organizations_policy.scp_pci_encryption_mandate.id
  target_id = aws_organizations_organizational_unit.regulated.id
}
```

The SCP JSON itself follows a deny-with-exceptions pattern using condition keys to avoid breaking AWS-managed services:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedS3",
      "Effect": "Deny",
      "Action": ["s3:PutObject"],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": ["AES256", "aws:kms"]
        }
      }
    }
  ]
}
```

---

## Consequences

### What Becomes Easier

- Compliance audits: each SCP module maps 1:1 to a control requirement — evidence collection is straightforward
- Onboarding a new BU: create its OU under Standard (or Regulated), attach its service profile SCP (1 new module), assign environment-tier OUs. No new Tier 1 or Tier 2 authoring required.
- AI access control: Platform teams get AI tools automatically by virtue of OU placement — no IAM exceptions or special-casing needed
- International BU support: BU2's multi-region access is structural (no regional deny SCP) rather than a policy exception, so it's fully auditable and self-documenting
- Responding to new compliance requirements: author one new module, attach to the affected tier
- Debugging access denials: the deny is in a named, single-purpose module with a known OU scope — follow the OU path to find the culprit
- Testing: the `PolicyStaging` OU provides a safe path to validate every module before production rollout

### What Becomes Harder

- Initial catalog design: BU service profiles require InfoSec to collaborate with each BU Engineering Lead to define their intended service footprint — this is a one-time conversation per BU but requires scheduling and alignment
- Governance discipline: the model only works if BU-specific SCPs stay in the shared library. One-off BU SCPs outside this model create drift and must be avoided
- New AI services: as AWS releases new AI/ML services, `scp-deny-ai-services` at the Standard level must be actively maintained to include new service actions. A new Bedrock model or AWS AI service not in the deny list is implicitly allowed for all BUs. Assign an owner to monitor this.
- SCP attachment limit: each OU can directly attach at most 5 SCPs. Slot headroom is tracked in the Composition Matrix above. New controls must fit within reserved slots or be merged into an existing themed grouping.

### What We'll Need to Revisit

- **New BU onboarding:** Each new BU requires a service profile SCP authored with that BU's Engineering Lead. Document this as part of the account vending runbook (Epic 6).
- **AI service catalog expansion:** Review `scp-deny-ai-services` quarterly as AWS releases new AI/ML services. Assign InfoSec as the owner of this watch list.
- **BU2 region floor review:** If the org approves new regions, the `scp-region-lock` at `AFT-Managed` must be updated. This affects all BUs' effective region set — validate with each BU's engineering lead.
- **SOC 2 expansion:** If additional compliance frameworks come in scope (SOC 2, ISO 27001), a new Tier 2 compliance envelope module set fits in the 2 reserved slots at the Regulated OU.
- **BU-specific tagging policies:** SCPs cannot enforce tag formats — that's handled by Tag Policies (LZ-405 in Epic 4). Ensure the two don't overlap in intent.
- **PCI-DSS v4.0 enforcement dates:** Review the `scp-pci-*` module set against the current enforcement schedule annually.
- **SCP size growth:** Monitor `scp-bu*-service-profile` character counts as service lists grow. At 4,000+ characters, split into two modules and use the second reserved slot at the BU OU level.

---

## Action Items

**Phase 1 — Design & Approval**
1. [ ] InfoSec and Platform leads review and accept this ADR
2. [ ] InfoSec workshops with each BU Engineering Lead to define service profile scope (BU1, BU2, BU3, Platform) — outputs: approved service deny lists per BU
3. [ ] SCP library Git repository created under InfoSec ownership with PR-based merge process

**Phase 2 — Author & Test**
4. [ ] InfoSec authors Tier 1a/1b Foundation SCP modules (5 modules: `scp-org-integrity`, `scp-security-services-protect`, `scp-region-lock`, `scp-root-and-access-baseline`, `scp-aft-pipeline-protect`)
5. [ ] InfoSec + Compliance authors Tier 2 PCI-DSS and NIST modules (3 consolidated modules)
6. [ ] InfoSec authors Tier 2.5 BU service profile modules (5 modules: `scp-deny-ai-services`, `scp-bu1-service-profile`, `scp-bu2-service-profile`, `scp-bu3-service-profile`, `scp-platform-service-profile`)
7. [ ] Platform team authors Tier 3 environment control modules (3 prod + 1 staging) with InfoSec review
8. [ ] All Terraform code written and peer-reviewed
9. [ ] Full test pass in `PolicyStaging` OU for each module — positive (allowed) and negative (denied) tests documented

**Phase 3 — Deploy**
10. [ ] Tier 1a modules attached at Root; Tier 1b at `AFT-Managed`
11. [ ] `Regulated`, `Standard`, and `Platform` sub-OUs created under `Workloads`; verify Platform is a sibling of Standard (not a child)
12. [ ] Tier 2 modules attached to `Regulated` OU
13. [ ] `scp-deny-ai-services` attached to `Standard` OU; verify Platform accounts can invoke Bedrock/SageMaker while Standard BU accounts cannot
14. [ ] BU-level OUs created (`BU1-API`, `BU2-International`, `BU3-Data`) under Standard; service profile SCPs attached
15. [ ] Environment-tier OUs created under each BU OU; Tier 3 modules attached
16. [ ] Validate BU2 international region access: account in `BU2-International/Dev` can operate in `eu-west-1`; account in `BU1-API/Dev` cannot

**Phase 4 — Operationalize**
17. [ ] Compliance team performs control mapping review (SCP module → PCI/NIST control)
18. [ ] BU Engineering Leads briefed on their OU tier, effective SCP count, and what is/isn't permitted
19. [ ] Governance process documented in day-2 runbook (Epic 6): how to request SCP changes, BU service profile update process, AI service watch list, emergency procedure
20. [ ] Quarterly calendar reminder set: review `scp-deny-ai-services` for new AWS AI service actions
21. [ ] Annual calendar reminder set: review `scp-pci-*` against PCI-DSS v4.0 enforcement updates

---

## References

- [AWS Best Practices for SCPs in a Multi-Account Environment](https://aws.amazon.com/blogs/industries/best-practices-for-aws-organizations-service-control-policies-in-a-multi-account-environment/) — AWS Industries Blog
- [Get More Out of SCPs in a Multi-Account Environment](https://aws.amazon.com/blogs/security/get-more-out-of-service-control-policies-in-a-multi-account-environment/) — AWS Security Blog
- [SCP Evaluation and Inheritance](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_inheritance_auth.html) — AWS Organizations Docs
- [Architecting for PCI DSS Scoping and Segmentation on AWS](https://d1.awsstatic.com/whitepapers/compliance/architecting-pci-dss-segmentation-scoping-aws.pdf) — AWS Whitepaper
- [NIST SP 800-53 Rev. 5 Compliance on AWS](https://aws.amazon.com/blogs/security/implementing-a-compliance-and-reporting-strategy-for-nist-sp-800-53-rev-5/) — AWS Security Blog
- [PCI DSS v4.0 on AWS](https://d1.awsstatic.com/whitepapers/compliance/pci-dss-compliance-on-aws-v4-102023.pdf) — AWS Whitepaper
- ADR-001: AWS VPC IPAM CIDR Automation (`adrs/adr-001-aws-ipam-cidr-automation.md`)
- Epic 1: Control Tower & OU Foundation (`epic-01-control-tower-foundation.md`)
- Epic 4: IAM Foundation (`epic-04-iam-foundation.md`)
