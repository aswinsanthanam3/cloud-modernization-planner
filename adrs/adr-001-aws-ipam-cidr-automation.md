# ADR-001: Replace Manual Infoblox CIDR Intake with AWS VPC IPAM + Infoblox Sync

**Status:** Proposed
**Date:** 2026-03-29
**Deciders:** Platform Engineering Lead, Network Engineering Lead, InfoSec, Cloud Operations
**Related Epic:** Epic 3 — Hub-and-Spoke Networking

---

## Context

### The Problem

The current CIDR allocation workflow for new AWS accounts is fully manual and takes approximately **2 weeks** per request:

1. Platform team requests CIDR range(s) for a new account/VPC
2. Network engineer identifies available ranges in Infoblox IPAM (or a mix of Infoblox + spreadsheets)
3. Network engineer manually tags and reserves the CIDR blocks in Infoblox
4. Network engineer updates documentation and submits for approval
5. Approved CIDRs are communicated back to the platform team
6. Platform team manually configures the VPC with the allocated CIDR

This 2-week cycle is **incompatible with AFT's goal of automated, self-service account vending**. If AFT can spin up an account in ~30 minutes but CIDR allocation takes 2 weeks, networking becomes the critical-path bottleneck for every new account.

### Current State

- **CIDR source of truth:** Mixed — partially in Infoblox, partially in spreadsheets/wikis. No single authoritative source.
- **Scale:** The organization needs to pre-allocate 500+ CIDR blocks (/21, /22, /23) for new AFT-managed accounts.
- **On-prem connectivity:** Direct Connect / VPN exists but uses well-separated ranges from the planned AWS allocation. Overlap risk is low but must be validated.
- **Infoblox role going forward:** Infoblox remains the organization-wide IPAM source of truth. AWS IPAM allocations must sync back to Infoblox.

### Forces

- AFT account vending must be fully automated — no human in the loop for CIDR selection
- Infoblox must remain the enterprise-wide source of truth (organizational policy)
- 500+ pre-allocated CIDR blocks must be available in the pool from day one
- On-prem CIDR ranges must be protected from collision
- The solution must be codified in Terraform and managed via GitOps

---

## Decision

**Adopt AWS VPC IPAM as the cloud CIDR allocation engine with a one-time bulk load of 500+ CIDR blocks and fully automatic allocation triggered by AFT account customizations. Sync back to Infoblox will use a lightweight interim mechanism (EventBridge + Lambda) because the native Infoblox ↔ AWS IPAM integration requires an upgrade to Infoblox Universal DDI (UDDI), which is out of current scope.**

The architecture follows a three-layer model:

```
┌─────────────────────────────────────────────────────────────┐
│                   Infoblox (Enterprise IPAM)                │
│            Organization-wide source of truth                │
│       On-prem + Cloud ranges — sync FROM AWS IPAM           │
└──────────────────────┬──────────────────────────────────────┘
                       │  Infoblox Universal IPAM Integration
                       │  (bidirectional sync)
┌──────────────────────▼──────────────────────────────────────┐
│                    AWS VPC IPAM                              │
│              Cloud CIDR allocation engine                    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Top-Level Pool: Organization (e.g., 10.0.0.0/8)   │    │
│  │                                                     │    │
│  │  ├── Regional Pool: us-east-1                       │    │
│  │  │   ├── Env Pool: Prod    (/21, /22 blocks)       │    │
│  │  │   ├── Env Pool: Staging (/22, /23 blocks)       │    │
│  │  │   └── Env Pool: Dev     (/23 blocks)            │    │
│  │  │                                                  │    │
│  │  ├── Regional Pool: us-west-2                       │    │
│  │  │   ├── Env Pool: Prod                             │    │
│  │  │   ├── Env Pool: Staging                          │    │
│  │  │   └── Env Pool: Dev                              │    │
│  │  │                                                  │    │
│  │  └── Reserved Pool: On-Prem (non-allocatable)       │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  Shared via AWS RAM → Organization                          │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│              AFT Account Customization                      │
│                                                             │
│  1. Reads account metadata (env, region)                    │
│  2. Looks up correct IPAM pool via tags                     │
│  3. Calls aws_vpc with ipv4_ipam_pool_id                   │
│  4. IPAM auto-allocates next available CIDR                 │
│  5. VPC + subnets created — zero human input                │
└─────────────────────────────────────────────────────────────┘
```

---

## Options Considered

### Option A: AWS VPC IPAM with Infoblox Sync (Recommended)

| Dimension | Assessment |
|-----------|------------|
| Complexity | Medium — initial pool hierarchy setup + Infoblox integration |
| Cost | Low — AWS IPAM is free for a single IPAM; Advanced tier ~$0.01/hr per active IP if needed |
| Automation | Full — AFT requests CIDR automatically from IPAM pool, zero human in the loop |
| Infoblox compatibility | Interim — native sync requires UDDI upgrade (out of scope). Interim sync via EventBridge + Lambda writing to Infoblox WAPI. Future: upgrade to UDDI for native bidirectional sync. |
| Terraform support | Strong — `aws_vpc_ipam`, `aws_vpc_ipam_pool`, `aws_vpc_ipam_pool_cidr`, `aws_vpc_ipam_pool_cidr_allocation` |
| Time to allocate | Seconds (API call) vs. current 2 weeks |
| Team familiarity | Low-Medium — new service, but well-documented AWS patterns exist |

**Pros:**
- Eliminates the 2-week bottleneck entirely — allocation happens in seconds during AFT pipeline
- Native AWS service — no third-party agents in the data path
- Hierarchical pool model enforces CIDR discipline (region → env → account)
- Built-in overlap detection prevents collisions with on-prem ranges (reserved pool)
- Infoblox stays updated via interim sync (EventBridge + Lambda → Infoblox WAPI); upgradeable to native sync when UDDI is in scope
- Fully IaC-able — pool hierarchy, CIDRs, and sharing all in Terraform
- AWS prescriptive guidance pattern exists specifically for [AFT + IPAM integration](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/automate-amazon-vpc-ipam-ipv4-cidr-allocations-for-new-aws-accounts-by-using-aft.html)
- RAM sharing makes pools available to all accounts in the Organization

**Cons:**
- One-time effort to reconcile mixed Infoblox/spreadsheet data and bulk-load into IPAM
- **Native Infoblox ↔ AWS IPAM sync requires Infoblox UDDI upgrade — currently out of scope.** Interim solution: EventBridge + Lambda pushes new allocations to Infoblox via WAPI. This is custom code to maintain until UDDI upgrade happens.
- Team needs to learn AWS IPAM concepts (scopes, pools, allocations)
- Interim sync is one-directional (AWS → Infoblox). Changes made directly in Infoblox won't reflect in AWS IPAM until UDDI is in place.
- If interim sync Lambda fails, Infoblox drifts until the Lambda is fixed (allocations in AWS still work)

---

### Option B: Keep Infoblox as Allocator, Automate via API

| Dimension | Assessment |
|-----------|------------|
| Complexity | High — custom Lambda/Step Function to call Infoblox WAPI from AFT pipeline |
| Cost | Medium — Lambda + Infoblox licensing + custom code maintenance |
| Automation | Partial — automated request, but Infoblox API can be fragile and slow |
| Infoblox compatibility | Full — Infoblox is the allocator |
| Terraform support | Weak — Infoblox Terraform provider exists but is limited and community-maintained |
| Time to allocate | 1–5 minutes (API call + Infoblox processing) |
| Team familiarity | Medium — team knows Infoblox, but API automation is new |

**Pros:**
- Infoblox stays as both source of truth AND allocator — single system
- No new AWS service to learn
- Existing Infoblox investment is fully leveraged

**Cons:**
- Infoblox API introduces a fragile external dependency into the AFT pipeline — if Infoblox is down, no accounts can be vended
- Custom glue code (Lambda, Step Functions) to bridge AFT → Infoblox → Terraform is complex and hard to maintain
- Infoblox Terraform provider is community-maintained, not HashiCorp-supported
- Doesn't solve the mixed spreadsheet/Infoblox data problem — still need a reconciliation effort
- Latency and reliability of Infoblox API in the critical path of account provisioning
- No native overlap detection with AWS VPC space

---

### Option C: Manual Process with Faster SLA

| Dimension | Assessment |
|-----------|------------|
| Complexity | Low — process change only, no new tooling |
| Cost | None — just team time |
| Automation | None — still manual |
| Infoblox compatibility | Full |
| Terraform support | N/A |
| Time to allocate | 2–3 days (reduced from 2 weeks with process optimization) |
| Team familiarity | High |

**Pros:**
- No new tooling to learn or maintain
- Infoblox workflow unchanged
- Could pre-allocate batches to reduce per-request overhead

**Cons:**
- Still manual — fundamentally incompatible with automated account vending
- Scales linearly with headcount — more accounts = more network engineer time
- Doesn't solve the mixed data source problem
- 2–3 days is still too slow for AFT's promise of minutes-to-account

---

## Trade-off Analysis

The core trade-off is **automation speed vs. architectural complexity**.

Option A (AWS IPAM) introduces a new AWS service and a one-time migration effort, but it's the only option that achieves **zero-human-in-the-loop CIDR allocation** — which is the entire point of AFT. AWS has published a [prescriptive guidance pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/automate-amazon-vpc-ipam-ipv4-cidr-allocations-for-new-aws-accounts-by-using-aft.html) specifically for this use case, which significantly de-risks the implementation.

Option B keeps Infoblox as the allocator but introduces brittle custom code in the critical path. If the Infoblox API has an outage or latency spike, the entire AFT pipeline stalls. This trades one bottleneck (manual process) for another (API reliability).

Option C is a non-starter for an automated landing zone.

The Infoblox sync requirement is well-served by Option A: the [Infoblox Universal IPAM integration with AWS VPC IPAM](https://docs.aws.amazon.com/vpc/latest/ipam/integrate-infoblox-ipam.html) provides native bidirectional sync, meaning Infoblox stays as the enterprise source of truth without being in the critical allocation path.

**Recommendation: Option A.**

---

## Implementation Approach

### Phase 1: Data Reconciliation & CIDR Plan (Week 1–2)

1. **Audit current state** — Export all CIDR allocations from Infoblox AND spreadsheets/wikis. Identify overlaps, gaps, and inconsistencies.
2. **Define pool hierarchy** — Map out top-level → regional → environment pools with prefix lengths:
   - Prod accounts: /21 (2,048 IPs) or /22 (1,024 IPs)
   - Staging accounts: /22 or /23 (512 IPs)
   - Dev/Sandbox accounts: /23
3. **Reserve on-prem ranges** — Create a non-allocatable pool with all on-prem/DC CIDR blocks to prevent collision.
4. **Generate the bulk load manifest** — CSV/JSON of 500+ CIDR blocks mapped to their target pools.
5. **Network team + InfoSec review** — Approve the CIDR plan and pool hierarchy before loading.

### Phase 2: AWS IPAM Deployment (Week 2–3)

1. **Deploy IPAM** in the Network Hub account via Terraform:
   - `aws_vpc_ipam` — create the IPAM instance
   - `aws_vpc_ipam_scope` — use default private scope (or create custom)
   - `aws_vpc_ipam_pool` — hierarchical pools (org → region → env)
   - `aws_vpc_ipam_pool_cidr` — bulk-provision 500+ CIDRs into pools (use `for_each` over the manifest)
2. **Share pools** via AWS RAM to the Organization.
3. **Configure allocation rules** on each pool:
   - `allocation_min_netmask_length` and `allocation_max_netmask_length` to enforce sizing
   - `allocation_default_netmask_length` for the common case
   - Tags for pool identification by AFT customization code

### Phase 3: Interim Infoblox Sync (Week 3–4)

> **Note:** The native Infoblox ↔ AWS IPAM bidirectional sync requires an upgrade to Infoblox Universal DDI (UDDI), which is **out of current scope**. This phase implements a lightweight interim sync mechanism. When UDDI is upgraded in a future initiative, this interim solution can be replaced with the native integration.

1. **Deploy EventBridge rule** that captures IPAM allocation events (`aws.ec2` → `AllocateIpamPoolCidr` and VPC creation events).
2. **Deploy Lambda function** that:
   - Receives the allocation event (CIDR block, pool, account, region)
   - Calls the Infoblox WAPI to create/update a network object with the allocated CIDR
   - Tags the Infoblox object with metadata (AWS account ID, environment, allocated-by: aws-ipam)
   - Logs success/failure to CloudWatch; publishes failures to an SNS topic for alerting
3. **Validate**: Allocate a test CIDR from AWS IPAM → confirm the Lambda fires → confirm the range appears in Infoblox within minutes.
4. **Document the interim sync model**:
   - Direction: one-way (AWS IPAM → Infoblox)
   - Limitation: changes made directly in Infoblox are NOT reflected back in AWS IPAM
   - Failure mode: if Lambda fails, AWS allocations proceed but Infoblox drifts; SNS alert triggers manual reconciliation
   - Future state: replace with native UDDI integration when Infoblox upgrade is in scope

### Phase 4: AFT Integration (Week 4–5)

1. **Update the spoke VPC Terraform module** (Epic 3, LZ-303) to accept `ipv4_ipam_pool_id` instead of a static CIDR:
   ```hcl
   resource "aws_vpc" "spoke" {
     ipv4_ipam_pool_id   = data.aws_vpc_ipam_pool.env_pool.id
     ipv4_netmask_length = var.vpc_netmask_length  # e.g., 22
     # CIDR is auto-allocated from the pool — no manual input
   }
   ```
2. **Wire pool lookup into AFT customization** — use account metadata (env tag, region) to select the correct pool.
3. **End-to-end test** — vend a test account via AFT, verify VPC gets an auto-allocated CIDR from the correct pool, confirm it appears in Infoblox.

---

## Consequences

### What Becomes Easier

- Account vending is fully automated end-to-end — CIDR allocation takes seconds, not weeks
- CIDR discipline is enforced by pool hierarchy — no more ad-hoc range selection
- Overlap prevention is automatic — on-prem ranges in a reserved pool can't be allocated
- Infoblox stays updated via interim EventBridge + Lambda sync — no manual double-entry (upgradeable to native sync with UDDI)
- Network engineer time is freed from repetitive CIDR intake work
- Scaling to hundreds of accounts requires no additional manual effort

### What Becomes Harder

- Initial migration effort to reconcile mixed data sources and bulk-load 500+ CIDRs
- Team must learn AWS IPAM concepts (one-time learning curve)
- Two systems to monitor (AWS IPAM + interim Lambda sync health)
- Interim sync is one-directional (AWS → Infoblox) — changes in Infoblox don't flow back until UDDI upgrade
- If interim sync Lambda fails, manual reconciliation is needed until fixed
- Interim Lambda is custom code that must be maintained until replaced by native UDDI integration
- Pool hierarchy changes (e.g., adding a new region) require Terraform changes and review

### What We'll Need to Revisit

- **Infoblox UDDI upgrade** — when budget/scope allows, upgrade Infoblox to Universal DDI and replace the interim Lambda sync with native bidirectional integration
- **Interim sync reliability** — monitor Lambda invocations and alert on failures; define a runbook for drift reconciliation
- **Pool exhaustion** — monitor pool utilization; alert when pools drop below 20% available CIDRs
- **IPv6 strategy** — this ADR covers IPv4 only; IPv6 pool strategy may be needed later
- **On-prem Direct Connect routing** — if new AWS ranges need to be advertised to on-prem, BGP/route updates will be needed as pools are consumed
- **IPAM Advanced tier** — if we need features like compliance checks or external integrations beyond basic, evaluate the Advanced tier pricing

---

## Action Items

1. [ ] Network team exports current Infoblox + spreadsheet CIDR data for reconciliation
2. [ ] Platform + Network team jointly define pool hierarchy and CIDR allocation plan
3. [ ] InfoSec reviews and approves the CIDR plan (especially on-prem reserved ranges)
4. [ ] Platform team deploys AWS IPAM + pools + bulk CIDR load via Terraform
5. [ ] Platform team deploys interim sync (EventBridge + Lambda → Infoblox WAPI) with alerting
6. [ ] Platform + Network team validate interim sync (allocate test CIDR → verify it appears in Infoblox)
7. [ ] Platform team updates spoke VPC module to use IPAM pool allocation
8. [ ] End-to-end test via AFT: vend account → auto-CIDR → VPC created → Infoblox updated
9. [ ] Network team decommissions the manual CIDR intake process
10. [ ] Platform team documents new process in day-2 runbook (Epic 6)
11. [ ] **Future:** When Infoblox UDDI upgrade is in scope, replace interim Lambda sync with native Infoblox Universal IPAM ↔ AWS IPAM integration

---

## References

- [Automate Amazon VPC IPAM IPv4 CIDR Allocations for New AWS Accounts by Using AFT](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/automate-amazon-vpc-ipam-ipv4-cidr-allocations-for-new-aws-accounts-by-using-aft.html) — AWS Prescriptive Guidance
- [Create a Hierarchical, Multi-Region IPAM Architecture on AWS Using Terraform](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/multi-region-ipam-architecture.html) — AWS Prescriptive Guidance
- [Integrate VPC IPAM with Infoblox Infrastructure](https://docs.aws.amazon.com/vpc/latest/ipam/integrate-infoblox-ipam.html) — AWS Documentation
- [Infoblox Universal IPAM Integration with Amazon VPC IPAM](https://www.infoblox.com/blog/company/infoblox-integrates-with-amazon-vpc-ipam-delivering-simplified-ip-address-management/) — Infoblox Blog
- [terraform-aws-ipam Module](https://github.com/aws-ia/terraform-aws-ipam) — AWS IA GitHub
- [aws-samples: sample-amazon-vpc-ipam-terraform](https://github.com/aws-samples/sample-amazon-vpc-ipam-terraform) — AWS Samples GitHub
- [aws_vpc_ipam_pool_cidr Terraform Resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc_ipam_pool_cidr) — Terraform Registry
- [aws_vpc_ipam_pool_cidr_allocation Terraform Resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc_ipam_pool_cidr_allocation) — Terraform Registry
