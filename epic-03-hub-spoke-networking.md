# EPIC 3: Hub-and-Spoke Networking (Transit Gateway)

**Epic Key:** LZ-NET
**Epic Name:** Build hub-and-spoke network topology with Transit Gateway
**Priority:** High
**Labels:** `landing-zone`, `networking`, `transit-gateway`, `vpc`
**Stories:** 6 · **Total Points:** 31
**Sprint Target:** Sprints 4–5

## Brownfield Context

The network hub account is provisioned under `AFT-Managed/Infrastructure`. TGW is shared via RAM at the Organization level, so spoke VPCs in both legacy and new accounts *could* attach — but initial scope is limited to new AFT-managed accounts only. If the organization has existing networking (VPCs, Direct Connect, VPN), coordinate with the network team to avoid CIDR collisions and ensure the new hub does not conflict with existing routing.

## Epic Description

Deploy a centralized network hub using AWS Transit Gateway in a dedicated network account under `AFT-Managed/Infrastructure`, build a hub VPC for shared services (NAT, DNS, VPC endpoints), and create a reusable spoke VPC Terraform module that is integrated into AFT account customizations. Every AFT-vended workload account will automatically receive a spoke VPC connected to the hub.

### Definition of Done

- Transit Gateway is deployed and shared via RAM to the Organization
- Hub VPC is operational with NAT gateway, DNS resolution, and VPC endpoints
- Spoke VPC Terraform module exists and is tested
- Spoke module is integrated into AFT so vended accounts get networking automatically
- End-to-end connectivity verified between spoke and hub
- No CIDR conflicts with existing legacy VPCs

### Dependencies

- Epic 1 complete (new `AFT-Managed/Infrastructure` OU created)
- Epic 2 complete (AFT pipeline for customization integration)
- CIDR allocation plan documented, reviewed against existing allocations, and approved

### Blocked By

- Epic 1 (new OU branch / Infrastructure sub-OU)
- Epic 2 (AFT customization repos for integration)

### Blocks

- Epic 2, LZ-204 (smoke test depends on networking customizations)

---

## Stories

### LZ-301: Deploy Transit Gateway in the Network Hub Account

**Story Points:** 5 · **Priority:** High · **Component:** Networking

**Description:**
As a network engineer, I need a Transit Gateway deployed in a centralized network account so that spoke VPCs in workload accounts can route through a shared hub.

**Acceptance Criteria:**

- Dedicated Network/Infrastructure account is provisioned
- Transit Gateway is deployed with:
  - Auto-accept shared attachments enabled (or manual approval flow)
  - Default route table association and propagation configured
  - CIDR blocks documented and non-overlapping
- TGW is shared via AWS RAM to the Organization
- TGW route tables created: `shared-services`, `workload-spoke`, `inspection` (if applicable)

---

### LZ-302: Create Hub VPC with Shared Services Subnets

**Story Points:** 5 · **Priority:** High · **Component:** Networking

**Description:**
As a network engineer, I need a hub VPC in the network account so that centralized services (DNS, egress, endpoints) are reachable from all spoke accounts.

**Acceptance Criteria:**

- Hub VPC created with CIDR from the IPAM allocation (or documented range)
- Subnets: public (NAT/IGW), private (shared services), TGW attachment subnets
- NAT Gateway deployed for centralized egress (if chosen pattern)
- VPC Flow Logs enabled and shipping to Log Archive
- Route tables configured for TGW <-> IGW/NAT traffic flow

---

### LZ-303: Build Spoke VPC Terraform Module for Workload Accounts

**Story Points:** 8 · **Priority:** High · **Component:** Networking

**Description:**
As a platform engineer, I need a reusable Terraform module that deploys a spoke VPC in any workload account and attaches it to the Transit Gateway, so that every new account gets consistent networking.

**Acceptance Criteria:**

- Module accepts parameters: CIDR block, AZs, number of subnet tiers (public/private/isolated), TGW attachment flag
- Module provisions:
  - VPC with DNS support and hostnames enabled
  - Subnets across specified AZs
  - TGW attachment in dedicated subnets
  - Route tables with default route to TGW for private subnets
  - VPC Flow Logs to centralized logging
- Module is tested with `terraform plan` in a sandbox account
- Module is stored in a shared module registry or Git repo

---

### LZ-304: Integrate Spoke VPC Module into AFT Global/Account Customizations

**Story Points:** 5 · **Priority:** High · **Component:** Networking

**Description:**
As a platform engineer, I need the spoke VPC module invoked as part of AFT account customizations so that every vended account automatically gets a connected VPC.

**Acceptance Criteria:**

- Spoke VPC module is called from `aft-account-customizations` (or `aft-global-customizations` if all accounts get it)
- CIDR allocation is handled via AWS VPC IPAM or a parameter/lookup table
- TGW attachment is created and associated with the correct TGW route table
- Routing from spoke to hub and hub to spoke is verified (ping test, traceroute, or flow log confirmation)

---

### LZ-305: Configure Route 53 Private Hosted Zones & DNS Forwarding

**Story Points:** 5 · **Priority:** Medium · **Component:** Networking

**Description:**
As a network engineer, I need centralized DNS resolution so that spoke accounts can resolve private hosted zone records and forward on-prem DNS queries through the hub.

**Acceptance Criteria:**

- Route 53 Resolver inbound and outbound endpoints deployed in the hub VPC
- Resolver rules created for on-prem domains (if applicable) and shared via RAM
- Private hosted zones associated with spoke VPCs (or via Resolver rules)
- DNS resolution verified from a spoke account to a hub-hosted private record

---

### LZ-306: Deploy Centralized VPC Endpoints in the Hub

**Story Points:** 3 · **Priority:** Medium · **Component:** Networking

**Description:**
As a network engineer, I need centralized VPC endpoints for core AWS services so that spoke accounts access S3, STS, SSM, etc., over private connectivity without individual endpoints per account.

**Acceptance Criteria:**

- Interface VPC endpoints deployed in the hub VPC for: `s3` (gateway), `sts`, `ssm`, `ssmmessages`, `ec2messages`, `logs`, `kms`
- Private DNS enabled or Resolver rules configured for endpoint resolution from spokes
- Spoke accounts verified to reach AWS services via private endpoint (no public internet)
