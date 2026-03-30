# EPIC 10: Just-In-Time Elevated Access (AWS TEAM Framework)

**Epic Key:** LZ-JIT
**Epic Name:** Deploy AWS TEAM for temporary elevated access management across all AFT-managed accounts
**Priority:** High
**Labels:** `landing-zone`, `iam`, `jit-access`, `team`, `privileged-access`, `zero-standing-privilege`
**Stories:** 8 · **Total Points:** 39
**Sprint Target:** Sprints 5–7
**References:** [AWS TEAM GitHub](https://github.com/aws-samples/iam-identity-center-team) · [AWS Security Blog](https://aws.amazon.com/blogs/security/temporary-elevated-access-management-with-iam-identity-center/)

---

## Context

### What is AWS TEAM?

[TEAM (Temporary Elevated Access Management)](https://github.com/aws-samples/iam-identity-center-team) is an AWS open-source solution (MIT-0 license) that integrates with AWS IAM Identity Center to provide just-in-time (JIT) privileged access to AWS accounts. Instead of standing permission set assignments, users request time-bound elevated access through a self-service portal; approvers review and grant/deny; and access is automatically revoked when the time window expires.

### Architecture (AWS-Managed Services)

TEAM is a full-stack serverless SPA built on:

| Component | AWS Service | Purpose |
|-----------|------------|---------|
| Frontend | AWS Amplify (React SPA) | Self-service portal for request/approve/audit |
| API layer | AWS AppSync (GraphQL) | Handles all UI actions; real-time subscriptions |
| Data store | DynamoDB | Request state, eligibility rules, approval records |
| Orchestration | Step Functions + Lambda | Request routing, auto-grant, scheduled auto-revoke |
| Auth/AuthZ | Amazon Cognito + SAML 2.0 | Group-based eligibility and approval authorization |
| Audit | CloudTrail Lake | Elevated session activity logs, queryable |
| Identity | IAM Identity Center | Permission set assignment/removal (the actual access grant) |

### How It Works

1. **Eligibility** — Cognito groups define who can request what. Group membership → eligible account + permission set combinations.
2. **Request** — User opens TEAM portal (SAML app in the Identity Center portal), selects account + permission set + duration, provides justification.
3. **Approval** — Approvers (also group-based) receive notification (SNS/Slack/email); approve or deny with comments.
4. **Grant** — Step Functions calls `CreateAccountAssignment` on IAM Identity Center, temporarily assigning the permission set.
5. **Revoke** — Step Functions fires `DeleteAccountAssignment` when the time window expires. Zero standing access.
6. **Audit** — CloudTrail Lake stores all elevated session activity; queryable by time, user, account, permission set.

### Why This Epic Exists

Our current model (Epic 4, LZ-401) uses standing permission set assignments — developers always have `DeveloperAccess-Standard` in their BU accounts, platform engineers always have `PlatformEngineer` across all AFT-managed accounts. TEAM shifts the model toward **zero standing privilege**: baseline access is read-only, and any write/deploy/admin access requires a JIT request with approval, justification, and time limit. This is a significant security maturity uplift and is required for SOC2 Type II and recommended for PCI-DSS v4.0 Requirement 7.2.

---

## Epic Description

Deploy the AWS TEAM framework into a dedicated TEAM account (or the delegated Identity Center admin account), integrate it with IAM Identity Center as a SAML 2.0 custom application, configure eligibility and approval groups aligned to our BU structure, define elevated permission sets with time-bound durations, integrate with our notification and ticketing systems, and operationalize with runbooks and monitoring. The end state: no engineer has standing write access to any production account — all elevated access flows through TEAM.

### Definition of Done

- TEAM application deployed, accessible from the IAM Identity Center portal as a SAML app
- Eligibility groups configured per BU and environment tier (who can request what)
- Approval groups configured per BU and environment tier (who can approve what)
- Elevated permission sets defined with maximum session durations
- Standing production write access removed — replaced by TEAM JIT requests
- CloudTrail Lake enabled and queryable for elevated session audit
- Slack/PagerDuty notifications for approval requests and grant events
- Day-2 runbook covering onboarding new BUs, adding permission sets, troubleshooting access issues
- End-to-end test: developer requests prod access → approval → time-limited access → auto-revocation verified

### Dependencies

- **Epic 4** — IAM Identity Center must be configured (LZ-401) with permission sets before TEAM can manage them
- **Epic 9, LZ-904** — Prod environment SCPs (`scp-prod-no-direct-deploy`) already enforce pipeline-only deploys; TEAM adds JIT for the exceptions (break-glass, debugging, incident response)
- **Epic 1** — AFT-Managed OU structure must be in place for account targeting

### Blocked By

- Epic 4, LZ-401 (IAM Identity Center and permission sets must exist)
- Epic 2 (AFT pipeline for deploying TEAM infrastructure)

### Blocks

- Nothing directly; TEAM is additive security hardening

---

## Stories

---

### LZ-1001: Architecture Decision — TEAM Deployment Model and Account Placement

**Story Points:** 3 · **Priority:** Highest · **Component:** Architecture

**Description:**
As a platform engineer, I need to decide where TEAM deploys (management account, delegated admin account, or dedicated TEAM account), how it integrates with our existing Identity Center setup, and whether Amplify hosting fits our security posture — before any infrastructure is provisioned.

**Acceptance Criteria:**

**Decision points to document:**

1. **Deployment account:**
   - Option A: Deploy in the existing Security/Audit account (if it's the delegated Identity Center admin)
   - Option B: Dedicated `TEAM` account under `AFT-Managed/Infrastructure`
   - Option C: Management account (not recommended — blast radius)
   - Document chosen option with rationale

2. **Delegated admin:**
   - TEAM requires delegated admin for IAM Identity Center management (`sso:*` and `identitystore:*` permissions)
   - Confirm whether the TEAM account or the existing Security account holds delegated admin
   - If Security account is already delegated admin, TEAM can deploy there

3. **Amplify vs. alternative hosting:**
   - TEAM ships as an Amplify app (CI/CD pipeline + hosting built-in)
   - Evaluate whether Amplify meets org security requirements (WAF, private access, VPN-only)
   - If Amplify is unacceptable: document CloudFront + S3 alternative with Amplify backend only

4. **Identity source alignment:**
   - TEAM uses Cognito with SAML federation back to Identity Center
   - Confirm compatibility with our chosen identity source (built-in / AD / external IdP from LZ-401)

**Output:** ADR-003 (`adrs/adr-003-team-jit-deployment-model.md`) — follows the same format as ADR-001 and ADR-002

---

### LZ-1002: Deploy TEAM Infrastructure (Amplify App, AppSync, DynamoDB, Step Functions)

**Story Points:** 8 · **Priority:** Highest · **Component:** Infrastructure

**Description:**
As a platform engineer, I need the TEAM application deployed in the chosen account with all backend services provisioned and the SAML integration with IAM Identity Center configured so that the portal is accessible and functional.

**Acceptance Criteria:**

**Infrastructure deployment:**
- CloudFormation stack deployed (TEAM ships as CFN; wrap in a Terraform `aws_cloudformation_stack` resource for state management if needed)
- Amplify app created with CI/CD pipeline connected to the TEAM source (forked to our org's GitHub/CodeCommit)
- AppSync GraphQL API operational
- DynamoDB tables created: `team-requests`, `team-eligibility`, `team-approvals`
- Step Functions state machine deployed: handles request → approval → grant → scheduled-revoke workflow
- Lambda functions deployed for: request routing, grant execution, revoke execution, notification dispatch
- Cognito User Pool configured with SAML 2.0 identity provider pointing to IAM Identity Center

**IAM Identity Center integration:**
- TEAM registered as a Custom SAML 2.0 Application in the Identity Center portal
- SAML assertion mapping configured (user attributes, group memberships)
- TEAM appears as a tile in the Identity Center portal for all eligible users
- TEAM account configured as delegated admin for Identity Center management (or shares delegated admin with Security account)

**Security hardening:**
- Amplify app accessible only through the Identity Center portal (SAML-enforced; no direct URL access without auth)
- WAF attached to the CloudFront distribution (if applicable) — rate limiting, geo-restriction
- All DynamoDB tables encrypted with KMS (customer-managed key)
- All Lambda functions run in VPC with no public internet access (VPC endpoints for AWS services)
- CloudWatch Logs for all Lambda functions shipped to Log Archive

**Validation:**
- Open Identity Center portal → TEAM tile visible → click → redirected to TEAM app → authenticated via SAML
- AppSync API responds to GraphQL queries (test with `listRequests`)
- Step Functions state machine visible and in `ACTIVE` state

---

### LZ-1003: Configure Eligibility Groups Aligned to BU Structure

**Story Points:** 5 · **Priority:** Highest · **Component:** IAM / TEAM

**Description:**
As a security engineer, I need eligibility groups that define which users can request elevated access to which accounts and permission sets, aligned to our BU and environment structure so that a BU1 developer cannot request access to a BU3 production account.

**Acceptance Criteria:**

**Eligibility model (Cognito groups → TEAM eligibility rules):**

| Eligibility Group | Can Request Access To | Eligible Permission Sets | Max Duration |
|-------------------|----------------------|------------------------|-------------|
| `team-bu1-dev` | BU1-API / Dev, Sandbox | `DeveloperAccess-Standard` | 8 hours |
| `team-bu1-elevated` | BU1-API / Staging, Prod | `DeveloperAccess-Prod`, `PowerUserAccess` | 2 hours |
| `team-bu2-dev` | BU2-Intl / Dev, Sandbox | `DeveloperAccess-Standard` | 8 hours |
| `team-bu2-elevated` | BU2-Intl / Staging, Prod | `DeveloperAccess-Prod`, `PowerUserAccess` | 2 hours |
| `team-bu3-dev` | BU3-Data / Dev, Sandbox | `DeveloperAccess-Standard`, `DataEngineerAccess` | 8 hours |
| `team-bu3-elevated` | BU3-Data / Staging, Prod | `DeveloperAccess-Prod`, `DataEngineerAccess` | 2 hours |
| `team-platform-dev` | Platform / Dev, Sandbox | `PlatformEngineer`, `PlatformAIAccess` | 8 hours |
| `team-platform-elevated` | Platform / Staging, Prod | `PlatformEngineer`, `PlatformAIAccess` | 4 hours |
| `team-security` | All AFT-managed accounts | `SecurityAuditAccess` | 8 hours |
| `team-breakglass` | All accounts | `AdministratorAccess` | 1 hour |

**Group membership sourced from:**
- Identity Center groups (synced from IdP if external) → mapped to Cognito groups via SAML attribute
- No direct Cognito user management; all group membership flows from the IdP

**Validation:**
- User in `team-bu1-dev` → can see BU1 Dev accounts in TEAM portal; cannot see BU3 or Prod accounts
- User in `team-breakglass` → can see all accounts but max duration is 1 hour
- User NOT in any `team-*` group → sees no eligible accounts in TEAM portal

---

### LZ-1004: Configure Approval Workflows and Notification Integration

**Story Points:** 5 · **Priority:** High · **Component:** TEAM / Notifications

**Description:**
As a security engineer, I need approval workflows configured so that elevated access requests are routed to the right approvers based on the target account and permission set, with notifications sent via Slack and email so approvers don't miss time-sensitive requests.

**Acceptance Criteria:**

**Approval groups (Cognito groups → TEAM approval authorization):**

| Approval Group | Can Approve Requests For | Auto-Approve? |
|---------------|------------------------|---------------|
| `team-approve-bu1` | BU1-API accounts (all environments) | No |
| `team-approve-bu2` | BU2-Intl accounts (all environments) | No |
| `team-approve-bu3` | BU3-Data accounts (all environments) | No |
| `team-approve-platform` | Platform accounts (all environments) | No |
| `team-approve-security` | Security/Audit access requests | No |
| `team-approve-breakglass` | Break-glass requests (all accounts) | No — requires 2 approvers |

**Approval rules:**
- Dev/Sandbox access: single approver required (BU lead or on-call engineer)
- Staging access: single approver required (BU lead or platform engineer)
- Prod access: single approver required (BU lead, security engineer, or on-call)
- Break-glass (`AdministratorAccess`): **two approvers required** — one from BU, one from Security
- Self-approval: **denied** — requestor cannot approve their own request
- Auto-deny after 30 minutes if no approval action taken (configurable per eligibility group)

**Notification integration:**
- SNS topic created for TEAM events: `team-access-requests`
- Slack integration via AWS Chatbot or Lambda webhook:
  - `#access-requests` channel: all new requests (requestor, account, permission set, duration, justification)
  - `#access-grants` channel: all approved grants (approver, grant time, expiry time)
  - `#access-revocations` channel: all revocations (auto-revoke or manual)
- Email notification to approvers for their group's requests
- PagerDuty escalation for break-glass requests (urgency = high)

**Validation:**
- Submit a request → Slack message appears in `#access-requests` within 30 seconds
- Approve a request → Slack message in `#access-grants`; IAM Identity Center shows temporary assignment
- Request expires → Slack message in `#access-revocations`; assignment removed from Identity Center
- Submit break-glass request → PagerDuty alert fires

---

### LZ-1005: Define Elevated Permission Sets for JIT Access

**Story Points:** 5 · **Priority:** High · **Component:** IAM

**Description:**
As a platform engineer, I need a refined set of permission sets specifically designed for JIT elevated access — tighter than standing access, with shorter sessions, enhanced logging, and tagged for audit — so that TEAM-granted access is clearly distinguishable from baseline access.

**Acceptance Criteria:**

**New JIT-specific permission sets (in addition to existing LZ-401 sets):**

| Permission Set | Purpose | Max Session | Key Differences from Standing |
|---------------|---------|-------------|------------------------------|
| `JIT-IncidentResponder` | Production debugging during incidents | 2 hours | Full read + targeted write (logs, EC2 describe/reboot, ECS force-deploy); no IAM/infra changes |
| `JIT-DatabaseAdmin` | Emergency DB maintenance (RDS, DynamoDB) | 1 hour | RDS modify, snapshot, failover; DynamoDB CRUD; no table deletion |
| `JIT-DeployOverride` | Manual production deploy (when pipeline is broken) | 1 hour | Lambda deploy, ECS update, EC2 run-instances; restricted to CI/CD-tagged resources |
| `JIT-SecurityInvestigator` | Active threat investigation | 4 hours | GuardDuty, Detective, CloudTrail, VPC Flow Logs read; Security Group modify for containment |
| `JIT-CostAnalyst` | FinOps deep-dive (reserved instances, savings plans) | 4 hours | Cost Explorer full, Billing read, resource tagging write |

**All JIT permission sets must:**
- Include a session tag: `team-request-id = <request-id>` (injected by TEAM via permission set inline policy condition)
- Include a session tag: `team-justification = <justification-hash>` (for CloudTrail correlation)
- Have shorter max session durations than their standing equivalents
- Be tagged `access-pattern = "jit"`, `managed-by = "team"`
- Have permission boundaries that prevent IAM escalation

**Standing access adjustments (impacts Epic 4, LZ-401):**
- `DeveloperAccess-Standard` in Prod accounts: **remove standing assignment** → only requestable via TEAM
- `PowerUserAccess` in all accounts: **remove standing assignment** → only via TEAM
- `AdministratorAccess`: was already break-glass only → now requires TEAM + 2-approver flow
- Keep standing: `ReadOnlyAccess` in all accounts (baseline read-only remains standing)
- Keep standing: `DeveloperAccess-Standard` in Dev/Sandbox (developers need frictionless dev access)

**Validation:**
- JIT permission set assignments include `team-request-id` session tag → visible in CloudTrail `AssumeRole` events
- Attempt to use `JIT-DeployOverride` to create an IAM role → denied (boundary)

---

### LZ-1006: Remove Standing Privileged Access and Enforce JIT-Only for Prod

**Story Points:** 5 · **Priority:** High · **Component:** IAM / Security

**Description:**
As a security engineer, I need standing write access to production accounts removed and replaced with JIT-only access through TEAM so that we achieve zero standing privilege in production — the single biggest security posture improvement this epic delivers.

**Acceptance Criteria:**

**Standing access removal (phased rollout):**

| Phase | Scope | Standing Access Removed | Replaced By |
|-------|-------|------------------------|-------------|
| Phase 1 | Platform / Prod | `PlatformEngineer` standing assignment | TEAM `JIT-IncidentResponder` or `JIT-DeployOverride` |
| Phase 2 | BU1-API / Prod | `DeveloperAccess-Prod` standing assignment | TEAM `JIT-IncidentResponder` |
| Phase 3 | BU2-Intl / Prod, BU3-Data / Prod | Same as Phase 2 | Same as Phase 2 |
| Phase 4 | All Staging accounts | `DeveloperAccess-Standard` write access | TEAM `DeveloperAccess-Standard` (JIT) |

**What remains as standing access (not removed):**
- `ReadOnlyAccess` in all accounts (engineers can always read)
- `DeveloperAccess-Standard` in Dev and Sandbox accounts (frictionless dev experience)
- `SecurityAuditAccess` for Security team (read-only; not privileged)

**Rollout process per phase:**
1. Announce the phase 2 weeks in advance with documentation link
2. Verify TEAM eligibility groups include all affected users
3. Remove standing permission set assignment via Terraform (Identity Center)
4. Monitor TEAM request volume for 1 week — look for users who can't find the TEAM portal or don't have eligibility
5. Collect feedback, adjust eligibility groups if needed
6. Phase complete when zero escalations for 5 business days

**Validation:**
- `aws sso-admin list-account-assignments` for a Prod account → no write permission sets standing
- Developer attempts `aws lambda update-function-code` in Prod without TEAM session → denied (only `ReadOnlyAccess` standing)
- Same developer requests JIT access via TEAM → approved → can deploy for the granted duration → auto-revoked

---

### LZ-1007: Enable CloudTrail Lake for Elevated Session Auditing

**Story Points:** 5 · **Priority:** High · **Component:** Audit / Compliance

**Description:**
As a security engineer, I need CloudTrail Lake configured to capture and query all elevated session activity so that auditors can trace every action taken during a TEAM-granted session back to the original request, justification, and approver.

**Acceptance Criteria:**

**CloudTrail Lake setup:**
- Event data store created in the Security/Audit account (or TEAM account — align with ADR-003)
- Retention: 7 years (matches SCP audit trail in Epic 8 and compliance requirements)
- Ingests CloudTrail management events from all AFT-managed accounts
- Ingests TEAM application events (request, approval, grant, revoke) via custom events

**Queryable dimensions:**
- By `team-request-id` session tag → all API calls made during that elevated session
- By user → all TEAM requests and their outcomes over time
- By account → all elevated sessions in a specific account
- By permission set → all grants of `AdministratorAccess` in the last 90 days
- By time window → all elevated access during an incident window

**Pre-built queries (saved in CloudTrail Lake):**
- "All break-glass sessions in the last 30 days" (for monthly security review)
- "Sessions exceeding 80% of their granted duration" (users who max out their time)
- "Denied requests" (for pattern analysis — are eligibility groups too restrictive?)
- "All elevated actions in account X during time window Y" (incident forensics)

**Compliance integration:**
- Monthly report generated: number of JIT requests, approval rate, average session duration, break-glass count
- Report pushed to S3 in Log Archive (WORM bucket — aligns with Epic 8, LZ-807)
- SecurityHub custom insight: "Break-glass sessions > 2 per month" → triggers finding

**Validation:**
- Request JIT access → perform actions → revoke → query CloudTrail Lake by `team-request-id` → all actions visible
- Run "break-glass sessions" query → returns expected results
- Monthly report generates and lands in S3

---

### LZ-1008: Operationalize TEAM — Runbook, Monitoring, and BU Onboarding

**Story Points:** 3 · **Priority:** Medium · **Component:** Operations

**Description:**
As a platform engineer, I need a day-2 runbook, operational monitoring, and a documented BU onboarding process so that TEAM is sustainable beyond initial deployment — new BUs can be onboarded, issues can be diagnosed, and the system health is monitored continuously.

**Acceptance Criteria:**

**Runbook (added to Epic 6 documentation):**
- "How to onboard a new BU to TEAM" — create eligibility group, approval group, map accounts, test
- "How to add a new elevated permission set" — create in Identity Center, add to TEAM eligibility, test
- "User cannot see accounts in TEAM portal" — troubleshoot: Cognito group membership, SAML attribute mapping, eligibility rule
- "TEAM grant failed" — troubleshoot: Step Functions execution logs, Identity Center API errors, permission boundary conflicts
- "Emergency: TEAM is down and we need prod access" — break-glass via direct Identity Center console (requires management account access; document the manual process)
- "How to revoke access before expiry" — manual revoke via TEAM portal or direct `DeleteAccountAssignment`

**Monitoring:**
- CloudWatch dashboard: `TEAM-Operations`
  - TEAM request volume (last 24h, 7d, 30d)
  - Approval latency (p50, p95 — target: < 5 minutes for Dev, < 15 minutes for Prod)
  - Grant success/failure rate
  - Active elevated sessions (real-time count)
  - Step Functions execution failures
- Alarms:
  - Step Functions failure rate > 5% → PagerDuty
  - Approval latency > 30 minutes → Slack
  - DynamoDB throttling → Slack
  - Break-glass request submitted → PagerDuty (immediate)
  - TEAM app health check failure (Amplify) → PagerDuty

**BU onboarding checklist (template):**
- [ ] Identify BU accounts and their environment tiers
- [ ] Create `team-<bu>-dev` and `team-<bu>-elevated` eligibility groups
- [ ] Create `team-approve-<bu>` approval group
- [ ] Map groups to Identity Center groups (IdP sync)
- [ ] Configure eligibility rules in TEAM: account → permission set → max duration
- [ ] Test: BU user requests Dev access → approved → granted → revoked
- [ ] Test: BU user requests Prod access → approved → granted → revoked
- [ ] Announce TEAM to BU with documentation link
- [ ] Remove standing write access per LZ-1006 phased rollout

---

## Summary

| Story | Title | Points | Priority |
|-------|-------|--------|----------|
| LZ-1001 | ADR-003: TEAM deployment model and account placement | 3 | Highest |
| LZ-1002 | Deploy TEAM infrastructure (Amplify, AppSync, Step Functions) | 8 | Highest |
| LZ-1003 | Configure eligibility groups aligned to BU structure | 5 | Highest |
| LZ-1004 | Approval workflows and Slack/PagerDuty notifications | 5 | High |
| LZ-1005 | Define JIT-specific elevated permission sets | 5 | High |
| LZ-1006 | Remove standing privileged access; enforce JIT-only for Prod | 5 | High |
| LZ-1007 | CloudTrail Lake for elevated session auditing | 5 | High |
| LZ-1008 | Runbook, monitoring, and BU onboarding process | 3 | Medium |
| **Total** | | **39** | |

## Sprint Sequencing

```
Sprint 5:
  LZ-1001 (ADR-003)              ← must be decided before infra; gate for everything else
  LZ-1002 (Deploy TEAM infra)    ← starts after ADR decision; longest lead time
  LZ-1005 (JIT permission sets)  ← can be authored in parallel with infra deployment

Sprint 6:
  LZ-1003 (Eligibility groups)   ← needs TEAM infra live
  LZ-1004 (Approval workflows)   ← needs TEAM infra live; parallel with LZ-1003
  LZ-1007 (CloudTrail Lake)      ← independent; can run parallel

Sprint 7:
  LZ-1006 (Remove standing access) ← needs LZ-1003 + LZ-1004 + LZ-1005 all done; phased rollout
  LZ-1008 (Operationalize)         ← parallel with LZ-1006 phase 1
```

## Cross-Epic Impact

### Epic 4 (IAM Foundation) — LZ-401 Update Required

LZ-401 (SSO permission sets) must be updated to account for TEAM:

- **Standing permission set assignments now have two categories:**
  - *Always standing:* `ReadOnlyAccess` (all accounts), `DeveloperAccess-Standard` (Dev/Sandbox only), `SecurityAuditAccess`
  - *JIT-only via TEAM:* `DeveloperAccess-Prod`, `PowerUserAccess`, `AdministratorAccess`, `PlatformEngineer` (Prod), `PlatformAIAccess` (Prod)
- **New permission sets defined in LZ-1005** must be created in Identity Center alongside LZ-401 sets
- **LZ-401 acceptance criteria updated:** "standing permission set assignment" now means "standing for Dev/Sandbox; JIT-only for Staging/Prod" for write-capable sets

### Epic 9 (SCP Governance) — LZ-904 Alignment

- `scp-prod-no-direct-deploy` (Tier 3) and TEAM are **complementary, not redundant:**
  - SCP denies console/CLI deploys from non-pipeline principals
  - TEAM JIT `JIT-DeployOverride` permission set must be exempted in the SCP condition (its role ARN added to the `StringNotLike` list)
  - This means LZ-904 needs a TEAM role ARN placeholder or a tag-based condition (`aws:PrincipalTag/access-pattern = "jit"`)

### Epic 8 (SCP Lifecycle) — Audit Trail Alignment

- TEAM audit trail (CloudTrail Lake, LZ-1007) and SCP audit trail (Epic 8, LZ-807) should use the same S3 WORM bucket in Log Archive for 7-year retention
- Monthly compliance report (LZ-1007) should be co-located with SCP change audit reports

### Epic 5 (Security Logging Baseline)

- CloudTrail Lake (LZ-1007) requires org-wide CloudTrail to be shipping events — confirmed by Epic 5
- GuardDuty should flag anomalous TEAM usage patterns (e.g., break-glass from unexpected IP) — add as a custom finding in Epic 5

### Epic 6 (Documentation)

- TEAM runbook (LZ-1008) adds entries to the landing zone day-2 operations runbook
