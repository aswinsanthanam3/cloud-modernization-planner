# EPIC 6: Documentation & Operationalization

**Epic Key:** LZ-DOC
**Epic Name:** Document the landing zone and operationalize for day-2
**Priority:** Medium
**Labels:** `landing-zone`, `documentation`, `runbook`, `adr`
**Stories:** 3 · **Total Points:** 11
**Sprint Target:** Sprint 6

## Brownfield Context

Documentation must clearly distinguish between the legacy OU structure and the new `AFT-Managed` branch. The ADR must capture the rationale for the parallel-OU approach, the CT upgrade decision, and the InfoSec SCP coordination model. The runbook must address interactions between legacy and new accounts (e.g., TGW attachment eligibility, shared security services).

## Epic Description

Capture all key architecture decisions in an ADR (including the brownfield/parallel-OU strategy), create a day-2 operations runbook covering common tasks for the AFT-managed environment, and conduct a formal landing zone review with security sign-off before onboarding workload teams.

### Definition of Done

- ADR documents all major decisions including the parallel-OU rationale, CT upgrade, TGW, SSO source, SCP strategy, InfoSec coordination model, and CIDR plan
- Day-2 runbook covers account vending, SCP/permission set changes, AFT troubleshooting, break-glass access, and legacy/new OU interaction patterns
- Formal review completed with security team sign-off
- Known gaps logged as follow-up stories

### Dependencies

- Epics 1–5 substantially complete
- At least one successful end-to-end account vend (LZ-204)

### Blocked By

- Epics 1–5 (content to document)

### Blocks

- Workload team onboarding (cannot begin until sign-off)

---

## Stories

### LZ-601: Write Landing Zone Architecture Decision Record (ADR)

**Story Points:** 3 · **Priority:** Medium · **Component:** Documentation

**Description:**
As a platform lead, I need an ADR documenting the key decisions (OU structure, networking topology, IAM model, AFT version) so that the team and future maintainers understand the rationale.

**Acceptance Criteria:**

- ADR covers: OU design, TGW vs VPC peering decision, SSO identity source, SCP strategy, Git provider for AFT, CIDR allocation strategy
- ADR is reviewed and stored in the team's documentation repo

---

### LZ-602: Create Day-2 Operations Runbook

**Story Points:** 5 · **Priority:** Medium · **Component:** Documentation

**Description:**
As a platform engineer, I need a runbook covering common day-2 tasks so that the team can self-serve account vending, networking changes, and troubleshooting.

**Acceptance Criteria:**

- Runbook covers:
  - How to vend a new account (PR workflow)
  - How to add a new OU
  - How to modify an SCP
  - How to update a permission set
  - How to troubleshoot AFT pipeline failures
  - How to add a new VPC endpoint
  - How to peer a new spoke to TGW
  - Break-glass access procedure
- Runbook stored in team wiki or documentation repo

---

### LZ-603: Conduct Landing Zone Review & Sign-Off

**Story Points:** 3 · **Priority:** Medium · **Component:** Documentation

**Description:**
As a platform lead, I need a formal review of the landing zone against the original requirements and security baseline so that we can sign off and begin onboarding workload teams.

**Acceptance Criteria:**

- Checklist reviewed: OU structure, SCP enforcement, SSO access, networking connectivity, logging, security services
- At least one account vended end-to-end with all customizations verified
- Security team sign-off obtained
- Known gaps or tech debt logged as follow-up stories
