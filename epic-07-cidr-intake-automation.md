# EPIC 7: CIDR Intake Process Automation (AWS VPC IPAM)

**Epic Key:** LZ-IPAM
**Epic Name:** Replace manual CIDR intake with AWS VPC IPAM and interim Infoblox sync
**Priority:** High
**Labels:** `landing-zone`, `networking`, `ipam`, `automation`, `infoblox`
**Stories:** 8 · **Total Points:** 42
**Sprint Target:** Sprints 3–4 (runs in parallel with Epic 3 networking setup)

## Epic Description

The current CIDR allocation process is fully manual, takes approximately 2 weeks per request, and is incompatible with AFT's automated account vending goals. This epic replaces that process with AWS VPC IPAM as the cloud CIDR allocation engine, pre-loaded with 500+ CIDR blocks (/21, /22, /23). A lightweight interim sync (EventBridge + Lambda → Infoblox WAPI) keeps Infoblox updated as the enterprise source of truth until a future Infoblox Universal DDI (UDDI) upgrade enables native bidirectional integration.

The architecture decision is captured in **ADR-001** (`adrs/adr-001-aws-ipam-cidr-automation.md`), which must be reviewed and approved before implementation begins.

### Definition of Done

- ADR-001 is reviewed and formally approved by Network, InfoSec, and Platform leads
- 500+ pre-allocated CIDR blocks are loaded into a hierarchical AWS IPAM pool structure
- On-prem CIDR ranges are reserved as non-allocatable in IPAM (collision prevention)
- IPAM pools are shared via AWS RAM to the Organization
- AFT spoke VPC module allocates CIDRs automatically from IPAM — zero human input
- Interim EventBridge + Lambda sync writes new allocations to Infoblox via WAPI
- Manual CIDR intake process is decommissioned
- Runbook covers pool exhaustion, sync failure, and future UDDI upgrade path

### Key Constraints

- **Infoblox UDDI upgrade is out of scope** — native bidirectional sync deferred to a future initiative
- **Mixed data sources** — current CIDR records split between Infoblox and spreadsheets; reconciliation required before loading
- **On-prem overlap risk** — low but must be validated; DC/VPN ranges must be reserved in IPAM before allocations begin
- **InfoSec dependency** — CIDR plan and on-prem reserved ranges need InfoSec approval

### External Dependencies

- **Network Engineering** — owns Infoblox export, CIDR plan approval, WAPI credentials for sync Lambda
- **InfoSec** — approves CIDR allocation plan and on-prem reserved range list
- **Cloud Operations** — validates that DC/VPN route tables won't conflict with new AWS ranges

### Blocked By

- Epic 1, LZ-104 (Network Hub account must exist in `AFT-Managed/Infrastructure` OU)

### Blocks

- Epic 3, LZ-303 (Spoke VPC module depends on IPAM pools being live)
- Epic 3, LZ-304 (AFT CIDR integration requires IPAM to be operational)

---

## Stories

### LZ-701: ADR-001 Review and Formal Approval

**Story Points:** 2 · **Priority:** Highest · **Component:** Governance
**External Dependency:** Network Engineering, InfoSec, Platform Lead

**Description:**
As a platform lead, I need ADR-001 (Replace Manual CIDR Intake with AWS VPC IPAM) formally reviewed and approved by all relevant stakeholders so that the implementation has organizational buy-in before any infrastructure is built.

**Acceptance Criteria:**

- ADR-001 (`adrs/adr-001-aws-ipam-cidr-automation.md`) distributed to: Network Engineering Lead, InfoSec, Cloud Operations, Platform Lead
- Review session held to walk through:
  - Options considered and recommendation (Option A: AWS IPAM)
  - Interim Infoblox sync approach and its limitations (one-directional, UDDI deferred)
  - On-prem CIDR collision prevention strategy
  - Pool hierarchy and sizing decisions
- Feedback incorporated into the ADR
- ADR status updated from `Proposed` → `Accepted` (or `Superseded` if an alternative is chosen)
- Signed-off ADR committed to the `adrs/` folder in the repo

**Definition of Blocked:** If approval is delayed, all subsequent stories in this epic are blocked. No IPAM infrastructure should be built until ADR is accepted.

---

### LZ-702: Audit and Reconcile Existing CIDR Data Sources

**Story Points:** 5 · **Priority:** Highest · **Component:** Data
**External Dependency:** Network Engineering

**Description:**
As a network engineer, I need all existing CIDR allocations exported from Infoblox and any spreadsheet/wiki sources, de-duplicated, and reconciled into a single authoritative inventory so that we have a clean dataset to bulk-load into AWS IPAM.

**Acceptance Criteria:**

- Full export of all CIDR blocks from Infoblox (all networks, containers, ranges)
- Full export of all CIDR blocks from spreadsheets/Confluence/wikis
- Exports merged and de-duplicated into a single inventory file (CSV or JSON):
  - Fields: CIDR block, status (allocated/available/reserved), environment, region, owner, notes
- Conflicts and inconsistencies between sources documented and resolved with network team
- On-prem/DC/VPN ranges explicitly flagged as `reserved-do-not-allocate`
- Inventory reviewed and approved by Network Engineering Lead before loading

**Notes:** This is a network engineer–led task; platform team provides the template and tooling to merge/de-duplicate if needed.

---

### LZ-703: Design AWS IPAM Pool Hierarchy and CIDR Allocation Plan

**Story Points:** 5 · **Priority:** Highest · **Component:** Networking
**External Dependency:** Network Engineering, InfoSec

**Description:**
As a platform engineer, I need a formal CIDR allocation plan and IPAM pool hierarchy defined and approved so that IPAM pools reflect our actual organizational structure and account sizing requirements.

**Acceptance Criteria:**

- Pool hierarchy designed and documented:
  - Top-level pool: Organization supernet (e.g., 10.0.0.0/8 or agreed range)
  - Regional pools: one per active AWS region (e.g., us-east-1, us-west-2)
  - Environment pools per region: `prod`, `staging`, `dev`, `sandbox`
  - Reserved pool: on-prem/DC/VPN ranges (non-allocatable, collision fence)
- Allocation sizing defined per environment:
  - Prod accounts: /21 (2,048 IPs)
  - Staging accounts: /22 (1,024 IPs)
  - Dev/Sandbox accounts: /23 (512 IPs)
- 500+ CIDR blocks mapped to their target environment pools in a load manifest (CSV/JSON)
- Overlap check run: no new AWS allocations conflict with on-prem reserved ranges
- Plan reviewed and approved by: Network Engineering Lead, InfoSec, Cloud Operations
- Pool hierarchy diagram included in Epic 3 architecture documentation

---

### LZ-704: Deploy AWS VPC IPAM and Pool Hierarchy via Terraform

**Story Points:** 8 · **Priority:** Highest · **Component:** Networking

**Description:**
As a platform engineer, I need AWS VPC IPAM deployed in the Network Hub account with the full pool hierarchy provisioned via Terraform so that CIDR pools are ready to serve allocation requests.

**Acceptance Criteria:**

- Terraform code written using:
  - `aws_vpc_ipam` — IPAM instance in the Network Hub account
  - `aws_vpc_ipam_scope` — private scope (default)
  - `aws_vpc_ipam_pool` — hierarchical pools (org → regional → environment)
  - `aws_vpc_ipam_pool_cidr` — CIDRs provisioned per pool
- Allocation constraints configured per pool:
  - `allocation_min_netmask_length` and `allocation_max_netmask_length` enforce sizing
  - `allocation_default_netmask_length` set per environment pool
  - Tags on each pool for AFT lookup (e.g., `environment = "prod"`, `region = "us-east-1"`)
- Reserved pool created with on-prem ranges marked as non-allocatable
- IPAM pools shared to the Organization via AWS RAM
- Terraform state stored in secure backend (S3 + DynamoDB, same as AFT state backend)
- `terraform plan` reviewed in PR before `terraform apply`
- IPAM pools visible in the AWS console with correct CIDR ranges and statuses

---

### LZ-705: Bulk Load 500+ CIDR Blocks into IPAM Pools

**Story Points:** 5 · **Priority:** High · **Component:** Networking

**Description:**
As a platform engineer, I need all 500+ pre-approved CIDR blocks bulk-loaded into the correct IPAM environment pools via Terraform so that there is an immediately available allocation inventory for AFT account vending.

**Acceptance Criteria:**

- Load manifest (from LZ-703) converted to Terraform `for_each` input (map or CSV-parsed locals)
- Each CIDR block provisioned into its target pool via `aws_vpc_ipam_pool_cidr`
- All 500+ blocks appear in the AWS IPAM console with status `provisioned`
- Pool utilization report generated post-load:
  - Total available CIDRs per pool
  - Estimated capacity (accounts supportable) per pool
  - Pools with < 20% available flagged for future expansion planning
- No overlaps detected by IPAM (IPAM validates internally; any conflict fails with an error)

---

### LZ-706: Deploy Interim Infoblox Sync (EventBridge + Lambda → WAPI)

**Story Points:** 8 · **Priority:** High · **Component:** Integration
**External Dependency:** Network Engineering (Infoblox WAPI credentials)

**Description:**
As a platform engineer, I need a lightweight sync mechanism that writes new AWS IPAM allocations back to Infoblox via WAPI so that Infoblox remains the enterprise-wide source of truth while the native UDDI integration is deferred.

**Acceptance Criteria:**

- EventBridge rule captures IPAM allocation events:
  - `AllocateIpamPoolCidr` (direct IPAM allocations)
  - VPC creation events with IPAM-allocated CIDRs (`CreateVpc` with `ipv4_ipam_pool_id`)
- Lambda function deployed:
  - Receives event, extracts CIDR block + metadata (AWS account ID, region, environment, pool)
  - Calls Infoblox WAPI to create a network object for the allocated CIDR
  - Tags the Infoblox object: `allocated-by: aws-ipam`, `aws-account-id`, `aws-region`, `environment`
  - On success: logs to CloudWatch
  - On failure: publishes to SNS topic → alert to network team
- Infoblox WAPI credentials stored in AWS Secrets Manager (not hardcoded)
- Lambda IaC in Terraform (function, IAM role, EventBridge rule, Secrets Manager reference)
- End-to-end validation: allocate test CIDR from IPAM → Lambda fires → CIDR appears in Infoblox within 2 minutes
- CloudWatch dashboard created for sync Lambda health (invocations, errors, duration)
- Runbook written:
  - How to verify sync health
  - How to manually reconcile if Lambda fails
  - Future: how to replace with native UDDI integration

**Notes:** Sync is one-directional (AWS → Infoblox). Changes made directly in Infoblox will NOT flow back to AWS IPAM until UDDI upgrade.

---

### LZ-707: Integrate AWS IPAM into AFT Spoke VPC Module

**Story Points:** 5 · **Priority:** High · **Component:** Networking

**Description:**
As a platform engineer, I need the spoke VPC Terraform module (Epic 3, LZ-303) updated to request CIDRs automatically from AWS IPAM pools instead of accepting a static CIDR input so that account vending requires zero human input for CIDR allocation.

**Acceptance Criteria:**

- Spoke VPC module updated:
  - Replaces `cidr_block` static input with `ipv4_ipam_pool_id` and `ipv4_netmask_length`
  - Pool ID is looked up dynamically using `data.aws_vpc_ipam_pool` filtered by environment + region tags
  - Module variable `vpc_netmask_length` defaults per environment (prod: 21, staging: 22, dev/sandbox: 23)
  - Backward-compatible: static CIDR still accepted as an override for exceptional cases
- VPC resource updated:
  ```hcl
  resource "aws_vpc" "spoke" {
    ipv4_ipam_pool_id   = data.aws_vpc_ipam_pool.env_pool.id
    ipv4_netmask_length = var.vpc_netmask_length
  }
  ```
- Module tested in sandbox account:
  - VPC created with auto-allocated CIDR from correct pool
  - CIDR reflected in IPAM console as allocated
  - CIDR synced to Infoblox via interim Lambda
- AFT account request template (LZ-203) updated — CIDR field removed; replaced with `environment` and `region` metadata used for pool lookup

---

### LZ-708: Validate End-to-End Automated CIDR Flow and Decommission Manual Process

**Story Points:** 4 · **Priority:** High · **Component:** Validation

**Description:**
As a platform engineer, I need to validate the full automated CIDR allocation flow end-to-end and formally decommission the manual Infoblox intake process so that the 2-week bottleneck is officially retired.

**Acceptance Criteria:**

- End-to-end test executed:
  1. New account request submitted via AFT (PR to `aft-account-request`)
  2. AFT pipeline runs, spoke VPC module invoked
  3. CIDR auto-allocated from correct IPAM pool (correct environment + region pool)
  4. VPC created with allocated CIDR — no human selected the CIDR
  5. IPAM console shows allocation with account metadata
  6. Infoblox shows the CIDR within 2 minutes (sync Lambda fired)
  7. Total time from PR merge to CIDR allocated: < 10 minutes
- Pool utilization verified post-test (one CIDR consumed from the correct pool)
- Network Engineering Lead confirms Infoblox record is accurate
- Manual CIDR intake process formally retired:
  - Runbook/wiki updated to redirect to new automated flow
  - Network engineer notified that manual CIDR request queue is closed for AFT-managed accounts
- Known limitations documented (UDDI upgrade path, one-directional sync)

---

## Summary

| Story | Title | Points | Priority | External Dependency |
|-------|-------|--------|----------|---------------------|
| LZ-701 | ADR-001 review and approval | 2 | Highest | Network Eng, InfoSec, Platform Lead |
| LZ-702 | Audit and reconcile CIDR data sources | 5 | Highest | Network Engineering |
| LZ-703 | Design IPAM pool hierarchy and CIDR plan | 5 | Highest | Network Eng, InfoSec, Cloud Ops |
| LZ-704 | Deploy AWS IPAM + pool hierarchy (Terraform) | 8 | Highest | — |
| LZ-705 | Bulk load 500+ CIDRs into IPAM pools | 5 | High | — |
| LZ-706 | Deploy interim Infoblox sync (EventBridge + Lambda) | 8 | High | Network Eng (WAPI creds) |
| LZ-707 | Integrate IPAM into AFT spoke VPC module | 5 | High | — |
| LZ-708 | End-to-end validation + decommission manual process | 4 | High | Network Engineering |
| **Total** | | **42** | | |

## Suggested Sequencing Within Epic

```
Sprint 3:
  LZ-701 (ADR approval)        ← gate: nothing starts until this is done
  LZ-702 (CIDR data audit)     ← network team–led, can run in parallel with LZ-701

Sprint 4:
  LZ-703 (IPAM pool design)    ← depends on LZ-702 (clean data)
  LZ-704 (Deploy IPAM TF)      ← depends on LZ-703 (approved plan)
  LZ-705 (Bulk CIDR load)      ← depends on LZ-704 (pools exist)
  LZ-706 (Interim sync Lambda) ← can start in parallel with LZ-704

Sprint 5:
  LZ-707 (AFT module update)   ← depends on LZ-705 (pools populated)
  LZ-708 (E2E validation)      ← depends on LZ-706 + LZ-707
```

## Future State (Out of Current Scope)

When Infoblox is upgraded to Universal DDI:
- Replace the interim EventBridge + Lambda sync (LZ-706) with native Infoblox ↔ AWS IPAM bidirectional integration
- Enable CloudOps to request CIDRs from Infoblox which are then written into AWS IPAM automatically
- Decommission the sync Lambda and CloudWatch dashboard
- Update runbook to reflect native sync behavior

Reference: [Infoblox Universal IPAM Integration with Amazon VPC IPAM](https://docs.aws.amazon.com/vpc/latest/ipam/integrate-infoblox-ipam.html)
