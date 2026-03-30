# EPIC 3: Hub-and-Spoke Networking — Spoke Provisioning

**Epic Key:** LZ-NET
**Epic Name:** Automate spoke VPC provisioning connected to the existing hub Transit Gateway
**Priority:** High
**Labels:** `landing-zone`, `networking`, `transit-gateway`, `vpc`, `vpc-endpoints`, `security-groups`
**Stories:** 7 · **Total Points:** 36
**Sprint Target:** Sprints 4–5

---

## Brownfield Context

The network hub account (`AFT-Managed/Infrastructure`) is already provisioned and operational:

- **Transit Gateway (TGW)** — deployed in the hub account, shared via AWS RAM to the Organization. TGW route tables exist (`workload-spoke`, `shared-services`). Auto-accept may or may not be enabled — confirm with the network team.
- **Hub VPC** — deployed with centralized NAT Gateway, Route 53 Resolver endpoints, and shared VPC endpoints. This is the single egress point for all spoke traffic.
- **IPAM** — deployed in the hub account (Epic 7) with regional/environment pools pre-populated.

This epic does **not** redeploy the TGW or hub VPC. It focuses entirely on the consumer side: a reusable Terraform module that vends spoke VPCs, attaches them to the existing TGW, provisions per-account VPC endpoints where needed, and lays down common security groups — all wired into AFT so every new account gets networking automatically.

---

## Epic Description

Build a reusable spoke VPC Terraform module that provisions a fully connected workload network in any AFT-vended account. The module discovers the existing TGW via a data source, allocates a CIDR from AWS VPC IPAM, creates a three-tier subnet layout (public / private / isolated) across availability zones, attaches to the TGW, configures route tables, deploys per-account VPC endpoints for core services, and creates a standard set of environment-aware security groups. The module is registered in the shared module registry and invoked automatically through AFT account customizations.

### Definition of Done

- Spoke VPC module discovers the existing TGW and IPAM pool via data sources — no static IDs hardcoded
- Spoke VPCs deploy with three-tier subnets across all active AZs in the target region
- TGW attachment is created and associated with the correct TGW route table automatically
- Default route for private/isolated subnets points to TGW (not NAT; NAT lives in hub)
- Per-account VPC endpoints deployed for services that cannot route efficiently through hub endpoints (S3 gateway, STS, SSM family)
- Common security groups provisioned per account (baseline, internal-only, restricted-egress)
- AFT integration confirmed: every new AFT-vended account gets a spoke VPC with zero human input after the module is deployed
- End-to-end connectivity verified: spoke → TGW → hub egress and spoke → AWS services via endpoint
- VPC Flow Logs enabled and shipping to Log Archive account

### Dependencies

- Epic 1 complete (AFT-Managed OU branch exists; hub account registered)
- Epic 2 complete (AFT customization repos available)
- Epic 7 complete (IPAM pools live and populated — LZ-705 done)
- Hub TGW ID and route table IDs documented by the network team
- Network team confirms TGW RAM share is active for the Organization

### Blocked By

- Epic 7, LZ-705 (IPAM pools must be populated before spoke VPCs can request CIDRs)

### Blocks

- Epic 2, LZ-204 (AFT smoke test depends on networking customizations being available)
- Epic 4, LZ-407 (IAM permission boundaries reference VPC and subnet IDs provisioned here)

---

## Stories

---

### LZ-301: Discover and Validate the Existing TGW via Terraform Data Sources

**Story Points:** 3 · **Priority:** Highest · **Component:** Networking

**Description:**
As a platform engineer, I need Terraform data sources that reliably discover the existing TGW and its route tables so that the spoke module never has hardcoded IDs and works across accounts and regions without manual configuration.

**Acceptance Criteria:**

- `data.aws_ec2_transit_gateway` lookup written using tag filter (e.g., `tag:Name = "hub-tgw"` and `tag:Environment = "hub"`); no static TGW ID in module inputs
- `data.aws_ec2_transit_gateway_route_table` lookups written for each expected route table (e.g., `workload-spoke`, `shared-services`); filtered by tags
- Data sources tested in a sandbox account where the RAM share is active — verify the TGW is visible from the workload account, not just the hub account
- If TGW is not visible (RAM share not yet active or account not in the Organization), module emits a clear Terraform error with remediation guidance (not a cryptic AWS API error)
- Data source module (`modules/tgw-lookup`) extracted as a standalone reusable component so spoke VPC and any other module can consume it without duplication
- Output values documented: `tgw_id`, `workload_route_table_id`, `shared_services_route_table_id`

**Notes:** The TGW lives in the hub account. The spoke module runs in the workload account. The RAM share makes the TGW visible cross-account — confirm with the network team that RAM share propagation to new accounts is automatic (either via org-wide share or via an AFT customization step).

---

### LZ-302: Build the Spoke VPC Terraform Module — VPC, Subnets, and Route Tables

**Story Points:** 8 · **Priority:** Highest · **Component:** Networking

**Description:**
As a platform engineer, I need a reusable Terraform module that creates a spoke VPC with a three-tier subnet layout (public, private, isolated) across all active AZs in the target region, allocates the CIDR from AWS VPC IPAM, and wires up route tables so that every vended account has consistent, correctly routed networking.

**Acceptance Criteria:**

**VPC provisioning:**
- VPC created using `ipv4_ipam_pool_id` + `ipv4_netmask_length` (no static `cidr_block`)
  - Pool ID looked up via `data.aws_vpc_ipam_pool` filtered by `environment` + `region` tags
  - `vpc_netmask_length` defaults: `prod` → 21, `staging` → 22, `dev`/`sandbox` → 23
  - Static CIDR override accepted as a module variable for exceptional cases
- `enable_dns_support = true`, `enable_dns_hostnames = true`
- VPC Flow Logs enabled: log to CloudWatch Logs group (named `/vpc/<account-id>/<vpc-id>/flow-logs`), log group shipped to Log Archive via subscription filter

**Subnet layout (per AZ, three tiers):**
| Tier | Purpose | Route |
|------|---------|-------|
| `public` | Load balancers, NAT (if ever needed locally) | IGW |
| `private` | Compute, containers, RDS | TGW (default route `0.0.0.0/0 → TGW`) |
| `isolated` | Databases, secrets, internal services | Local only (no internet, no TGW default route) |

- Subnet CIDR slicing is automatic: module divides the VPC CIDR evenly across tiers and AZs using `cidrsubnet()`
- Number of AZs configurable (default: all active AZs in the region via `data.aws_availability_zones`)
- Variable to disable `public` tier for fully private accounts (e.g., `create_public_subnets = false`; default `true`)

**Route tables:**
- One route table per tier (not per subnet)
- Public RT: `0.0.0.0/0 → IGW`
- Private RT: `0.0.0.0/0 → TGW` (egress via hub NAT)
- Isolated RT: no default route; explicit routes to IPAM supernet → TGW only (for internal cross-account traffic)
- On-prem CIDR summary route added to all RTs pointing to TGW (range documented in module variable `onprem_cidr_summary`)
- Internet Gateway created conditionally (only if `create_public_subnets = true`)

**Module outputs:** `vpc_id`, `private_subnet_ids`, `public_subnet_ids`, `isolated_subnet_ids`, `vpc_cidr_block`, `igw_id`

**Module variables (required):** `environment`, `aws_region`
**Module variables (optional with defaults):** `vpc_netmask_length`, `az_count`, `create_public_subnets`, `onprem_cidr_summary`, `ipam_pool_tag_filters`

---

### LZ-303: TGW Attachment, Route Table Association, and Propagation

**Story Points:** 5 · **Priority:** Highest · **Component:** Networking

**Description:**
As a platform engineer, I need the spoke VPC module to create a TGW attachment, associate it with the correct TGW route table, and propagate the spoke CIDR back to the hub route table so that traffic flows bidirectionally between the spoke and the hub without manual network team intervention.

**Acceptance Criteria:**

- TGW attachment created:
  ```hcl
  resource "aws_ec2_transit_gateway_vpc_attachment" "spoke" {
    transit_gateway_id = data.aws_ec2_transit_gateway.hub.id
    vpc_id             = aws_vpc.spoke.id
    subnet_ids         = aws_subnet.tgw_attachment[*].id   # dedicated /28 subnets per AZ
    transit_gateway_default_route_table_association = false
    transit_gateway_default_route_table_propagation = false
    tags = { Name = "spoke-${var.environment}-${var.aws_region}" }
  }
  ```
- Dedicated TGW attachment subnets provisioned: one `/28` per AZ, separate from the three-tier subnets (TGW attachments consume IPs; dedicated subnets keep the routing clean)
- Route table association: attachment associated with `workload-spoke` route table (ID from `data.aws_ec2_transit_gateway_route_table`)
- Route propagation: spoke VPC CIDR propagated to the `workload-spoke` route table
- If TGW auto-accept is disabled in the hub account: module outputs the attachment ID and documents the manual approval step in the runbook; a follow-up AFT automation story (LZ-306) can automate the approval via EventBridge + Lambda in the hub account
- Attachment acceptance state validated in module: `terraform apply` does not succeed until the attachment is in `available` state (use `timeouts` block: 10 min)
- Post-apply validation: `aws ec2 describe-transit-gateway-attachments` confirms `state = available`

**Notes:** Route propagation from spoke → hub route table cannot be done by the spoke account (it targets the hub account's TGW). Confirm with the network team whether propagation is automatic (TGW auto-propagation enabled) or requires a hub-account Terraform step. Document the expectation explicitly in the module README.

---

### LZ-304: Deploy Per-Account VPC Endpoints (Interface and Gateway)

**Story Points:** 5 · **Priority:** High · **Component:** Networking

**Description:**
As a platform engineer, I need a standard set of VPC endpoints deployed in each spoke account so that core AWS service traffic stays on the private network, even for services that do not route efficiently through the hub's centralized endpoints.

**Acceptance Criteria:**

**Gateway endpoints (free, always deploy):**
- `com.amazonaws.<region>.s3` — gateway endpoint attached to all private and isolated route tables
- `com.amazonaws.<region>.dynamodb` — gateway endpoint attached to all private and isolated route tables

**Interface endpoints (deployed per account):**
- `com.amazonaws.<region>.sts` — required for every IAM role assumption
- `com.amazonaws.<region>.ssm` — for Session Manager and Parameter Store
- `com.amazonaws.<region>.ssmmessages` — for Session Manager interactive sessions
- `com.amazonaws.<region>.ec2messages` — for Systems Manager agent communication
- `com.amazonaws.<region>.logs` — for CloudWatch Logs from workloads
- `com.amazonaws.<region>.kms` — for encryption key operations
- `com.amazonaws.<region>.secretsmanager` — for secrets access

**Interface endpoint configuration:**
- All interface endpoints placed in `isolated` subnets (no internet path needed; kept off private compute subnets to reduce blast radius)
- `private_dns_enabled = true` for all interface endpoints
- Endpoint security group: allow HTTPS (443) inbound from the VPC CIDR only
- Endpoint security group tagged `purpose = "vpc-endpoint"` for easy audit

**Conditional endpoints (feature-flagged variables):**
- `enable_ecr_endpoints` (default `false`) — deploys `ecr.api` + `ecr.dkr` + `s3` (required for ECR layer pulls)
- `enable_bedrock_endpoint` (default `false`) — deploys `bedrock-runtime`; only enabled for Platform accounts (enforced by SCP-AI-Deny for Standard accounts)
- `enable_codecommit_endpoint` (default `false`) — deploys `git-codecommit`

**Module variable:** `vpc_endpoint_services` (list) — allows caller to add extra endpoints beyond the baseline

---

### LZ-305: Provision Common Security Groups per Environment

**Story Points:** 5 · **Priority:** High · **Component:** Networking

**Description:**
As a platform engineer, I need a standard set of security groups provisioned in every spoke VPC so that workload teams have consistent, pre-approved network policy primitives to attach to their resources instead of creating ad-hoc security groups with overly permissive rules.

**Acceptance Criteria:**

**Security groups created (all tagged `managed-by = "platform-team"`):**

| SG Name | Description | Inbound | Outbound |
|---------|-------------|---------|----------|
| `sg-baseline-internal` | Default for internal compute | VPC CIDR on all ports | All (platform decides to restrict further at resource level) |
| `sg-restricted-egress` | For workloads that must not reach internet | VPC CIDR on all ports | VPC CIDR + AWS services via endpoints only |
| `sg-tgw-routable` | For resources reachable from other spokes | IPAM supernet on app ports (443, 8443) | VPC CIDR |
| `sg-vpc-endpoint` | Attached to all interface VPC endpoints | VPC CIDR on 443 | None (endpoints are ingress-only) |
| `sg-rds-isolated` | For isolated-tier databases | `sg-baseline-internal` as source on 5432/3306/1433 | None |
| `sg-alb-public` | For internet-facing ALBs (public tier only) | `0.0.0.0/0` on 443 (and 80 redirect) | VPC CIDR on app ports |

**Environment-specific rules:**
- `prod`/`staging`: `sg-alb-public` created only if `create_public_subnets = true`
- `dev`/`sandbox`: `sg-restricted-egress` is the default; teams must explicitly opt into `sg-tgw-routable`
- All SGs deny all inbound by default (AWS default); rules above are additive

**Outputs:**
- All SG IDs exported as module outputs for consumption by workload Terraform: `sg_baseline_internal_id`, `sg_restricted_egress_id`, `sg_tgw_routable_id`, `sg_rds_isolated_id`, `sg_alb_public_id`
- Security group IDs stored in SSM Parameter Store under `/landing-zone/<account-id>/sg/<sg-name>` for cross-module lookup without state coupling

**Validation:**
- `terraform validate` and `terraform plan` pass in sandbox account
- `aws ec2 describe-security-groups` confirms all groups exist with correct rules
- Security group IDs appear in SSM Parameter Store

---

### LZ-306: Integrate Spoke VPC Module into AFT Account Customizations

**Story Points:** 5 · **Priority:** Highest · **Component:** Networking

**Description:**
As a platform engineer, I need the spoke VPC module invoked automatically as part of AFT account customizations so that every newly vended account gets a fully connected VPC, subnets, TGW attachment, VPC endpoints, and security groups with zero human networking input.

**Acceptance Criteria:**

- Spoke VPC module registered in the shared module registry (Git tag `v1.0.0`; semantic versioning enforced)
- Module invocation added to `aft-account-customizations/terraform/networking.tf`:
  ```hcl
  module "spoke_vpc" {
    source      = "git::https://github.com/<org>/terraform-modules.git//modules/spoke-vpc?ref=v1.0.0"
    environment = local.account_tags["environment"]
    aws_region  = data.aws_region.current.name

    enable_ecr_endpoints     = local.account_tags["enable_ecr"] == "true"
    enable_bedrock_endpoint  = local.account_tags["platform_account"] == "true"
    create_public_subnets    = local.account_tags["public_subnets"] == "true"
  }
  ```
- Account request template (AFT `aft-account-request`) updated to include the tags that drive module behavior:
  - `environment` — `prod` | `staging` | `dev` | `sandbox`
  - `enable_ecr` — `true` | `false`
  - `platform_account` — `true` | `false`
  - `public_subnets` — `true` | `false`
- AFT pipeline tested end-to-end on a new sandbox account:
  - AFT customization pipeline runs automatically after account vending
  - Spoke VPC is created with correct CIDR from IPAM (correct env/region pool)
  - TGW attachment reaches `available` state
  - Security groups and VPC endpoints provisioned
  - SSM Parameter Store keys populated under `/landing-zone/<account-id>/`
- Runbook updated: how to re-run networking customizations if they fail on first vend, and how to update the module version for existing accounts

---

### LZ-307: End-to-End Connectivity Validation

**Story Points:** 5 · **Priority:** High · **Component:** Validation

**Description:**
As a platform engineer, I need to validate that full network connectivity works end-to-end — spoke-to-hub egress, spoke-to-spoke (where permitted), and spoke-to-AWS-services via VPC endpoints — so that the networking layer is confirmed fit for workloads before teams begin deploying.

**Acceptance Criteria:**

**Connectivity tests (all executed from a test EC2 instance in the private subnet of the test spoke account):**

| Test | Expected Result |
|------|----------------|
| `curl https://checkip.amazonaws.com` | Returns hub NAT Gateway public IP (not test instance IP) |
| `curl https://s3.amazonaws.com` via S3 gateway endpoint | Succeeds; VPC Flow Logs show no public IP path |
| `aws sts get-caller-identity` | Succeeds; VPC Flow Logs show traffic to STS VPC endpoint, not internet |
| `aws ssm start-session` from on-prem or bastion | Session opens; traffic via SSM endpoint |
| `nslookup ssm.<region>.amazonaws.com` from within VPC | Resolves to private IP (VPC endpoint IP, not public AWS IP) |
| Ping from Spoke A private subnet to Spoke B private subnet | Succeeds if both spokes are on `workload-spoke` route table; routing verified in TGW route table |
| Attempt to reach `0.0.0.0/0` from `isolated` subnet | Fails (no default route in isolated RT); confirm no internet path |

**IPAM and sync validation:**
- IPAM console shows spoke CIDR allocated with account metadata tags
- Infoblox shows the spoke CIDR within 2 minutes (interim sync Lambda fired — Epic 7, LZ-706)
- Pool utilization report shows one CIDR consumed from the correct pool

**Sign-off:**
- Network Engineering Lead confirms TGW route tables are correct
- Security team confirms no unintended public exposure from isolated subnets
- VPC Flow Logs confirmed shipping to Log Archive account (verify in Log Archive, not just spoke account)

---

## Summary

| Story | Title | Points | Priority |
|-------|-------|--------|----------|
| LZ-301 | Discover existing TGW via Terraform data sources | 3 | Highest |
| LZ-302 | Spoke VPC module — VPC, subnets, route tables | 8 | Highest |
| LZ-303 | TGW attachment, route table association, propagation | 5 | Highest |
| LZ-304 | Per-account VPC endpoints (interface + gateway) | 5 | High |
| LZ-305 | Common security groups per environment | 5 | High |
| LZ-306 | Integrate spoke module into AFT customizations | 5 | Highest |
| LZ-307 | End-to-end connectivity validation | 5 | High |
| **Total** | | **36** | |

## Suggested Sequencing Within Epic

```
Sprint 4:
  LZ-301 (TGW data source)     ← no dependencies; de-risks everything else
  LZ-302 (Spoke VPC module)    ← can start in parallel with LZ-301
  LZ-303 (TGW attachment)      ← depends on LZ-301 + LZ-302

Sprint 5:
  LZ-304 (VPC endpoints)       ← depends on LZ-302 (VPC must exist)
  LZ-305 (Security groups)     ← depends on LZ-302 (VPC must exist)
  LZ-306 (AFT integration)     ← depends on LZ-302 + LZ-303 + LZ-304 + LZ-305
  LZ-307 (E2E validation)      ← depends on LZ-306 (full stack must be live)
```

## Key Assumptions / Confirm with Network Team Before Sprint 4

1. **TGW RAM share** — Confirm the RAM share is org-wide (new accounts automatically see the TGW) vs. requires a manual share acceptance step.
2. **TGW auto-accept** — If `auto_accept_shared_attachments = enable`, spoke attachments become active automatically. If disabled, a hub-account approval step is needed (document in runbook; automate in a future story if volume warrants it).
3. **TGW route propagation** — Confirm whether `workload-spoke` route table has auto-propagation enabled for new attachments or whether hub-account Terraform must add a static route per spoke.
4. **Hub NAT topology** — Confirm that the hub NAT Gateway is the intended egress path for all spoke private traffic (`0.0.0.0/0 → TGW` on private route tables) vs. each spoke having its own NAT.
5. **On-prem CIDR summary** — Obtain the CIDR summary for DC/VPN ranges to add as explicit routes in spoke route tables pointing to TGW.
