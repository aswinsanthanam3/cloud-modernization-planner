# ADR-2025-003: IaC Engine Strategy for Cloud Platform Modules

**Status:** PROPOSED  
**Date:** 2026-04-03  
**Author:** Aswin (Platform Engineering)  
**Reviewers:** [TBD — Security Architecture, Cloud Engineering Lead]  
**Supersedes:** N/A  
**Related:** TP-2025-002 (IaC Module Refactoring & CLI Factory)

---

## 1. Context

The Platform Engineering team maintains 40+ Terraform wrapper modules that provide
standardized, security-hardened infrastructure to application development teams. These
modules enforce mandatory tags, KMS encryption, public access restrictions, and IAM
least-privilege policies.

The current setup has significant operational challenges: zero documentation, no
automated testing, 2–3 week developer lead time, capacity-intensive version upgrades,
and tight coupling to Terragrunt and Harness CD conventions.

As we plan the module refactoring effort (TP-2025-002), we need to decide whether to
continue with Terraform, migrate to AWS CDK, or maintain the current setup with
incremental improvements. This decision has a 3–5 year commitment horizon and affects
the entire engineering organization.

## 2. Decision Drivers

| Driver | Weight | Description |
|--------|--------|-------------|
| Developer experience | High | Reduce lead time from 2–3 weeks to < 1 day |
| Security & governance | High | IAM visibility, policy-as-code, compliance auditability |
| Team capability | High | 4–5 engineers; no CDK expertise today; strong Terraform/Python skills |
| Existing investment | Medium | 40+ TF modules, GitLab CI pipelines, Harness CD integration, OPA/Snyk |
| Multi-cloud portability | Low | AWS-primary strategy, but EKS Automode + on-prem Kubernetes indicates preference for portability where practical |
| Maintenance burden | High | Version upgrades, wrapper tax, test coverage gap |

## 3. Options Considered

### Option A: Current State (Terraform Wrappers + Terragrunt)

Keep the existing 40+ wrapper modules, Terragrunt-based configurations, and
console/tribal-knowledge-based StackSet management. Apply incremental improvements
(documentation, some testing) without architectural change.

**How governance works today:**
- Security hardening is embedded in wrapper module HCL (mandatory tags, KMS, encryption defaults)
- Prisma Cloud scans resources post-deployment and flags violations
- No pre-apply compliance validation
- IAM policies are written explicitly in module code (visible but undocumented)
- No automated testing; compliance is verified reactively

### Option B: Native Terraform + CLI Factory + Policy-as-Code

Refactor modules into composition modules (multi-resource groupings) and standalone
modules using native Terraform. Build a Python CLI factory that generates Terraform
configurations from YAML schemas. Eliminate Terragrunt for new modules. Wire OPA/Conftest
into GitLab CI as pre-apply compliance gates.

**How governance works:**
- Security defaults defined in YAML schemas, injected at generation time by CLI factory
- OPA/Conftest evaluates `terraform plan` JSON output before apply (pre-merge gate)
- Snyk IaC scans for CVEs and misconfigurations
- Prisma Cloud remains as post-deploy safety net
- IAM policies remain explicit in composition module `.tf` files (fully auditable)
- `terraform test` (plan-based) + Terratest (integration) for automated validation
- Provider `default_tags` block applies mandatory tags without wrapper involvement

### Option C: AWS CDK (Python)

Replace Terraform modules with CDK Constructs written in Python. Build L2+
Constructs that encode security defaults. Use cdk-nag for compliance validation.
Deploy via CloudFormation stacks.

**How governance works:**
- Security defaults encoded in custom Construct classes (L2+ wrappers)
- `cdk-nag` validates synthesized CloudFormation templates against rule packs (AWS Solutions, HIPAA, NIST 800-53, PCI DSS)
- CDK Aspects enforce cross-cutting concerns (tags, encryption) across all constructs
- CDK Property Injection (launched May 2025) standardizes default construct properties
- IAM managed via `grant()` methods — CDK auto-generates least-privilege policies
- CloudFormation Guard (`cfn-guard`) can validate synthesized templates in CI
- Prisma Cloud remains as post-deploy safety net

## 4. Evaluation

### 4.1 Developer Experience

| Criterion | Option A (Current) | Option B (TF + CLI) | Option C (CDK) |
|---|---|---|---|
| Language for app developers | HCL (unfamiliar) | CLI-generated HCL (they don't touch it) | Python (familiar) |
| IDE support | Minimal (Terragrunt has no language server) | Full (`terraform-ls`, autocomplete, go-to-definition) | Full (PyCharm/VS Code, type checking, autocomplete) |
| Time to first resource | 2–3 weeks | < 1 day (via `infra create`) | < 1 day (if Constructs exist) |
| Learning curve | Terraform + Terragrunt + tribal knowledge | CLI commands only (TF is generated, not hand-written) | CDK framework + CloudFormation mental model |
| Debugging experience | Error traces to Terragrunt cache directory | Error traces to actual `.tf` files | Error traces to generated CF template (verbose, opaque logical IDs) |
| Dynamic logic (loops, conditionals) | HCL `for_each`/`count` (verbose) | HCL generated from Jinja2 templates (app dev doesn't see it) | Native Python (loops, classes, inheritance) |

**Assessment:** Option C has the best raw developer experience for app developers who already know Python. Option B closes most of the gap by hiding HCL behind a CLI — developers interact with commands and YAML, not HCL. Option A is clearly worst.

### 4.2 Security & IAM Governance

| Criterion | Option A (Current) | Option B (TF + CLI) | Option C (CDK) |
|---|---|---|---|
| IAM policy visibility | Explicit in `.tf` files (but undocumented) | Explicit in composition module `.tf` files (documented, tested) | Opaque — `grant()` methods auto-generate policies; must `cdk synth` + inspect CF template to see actual permissions |
| Pre-apply compliance | None | OPA/Conftest on `terraform plan` JSON (mature ecosystem, custom Rego rules) | cdk-nag on synthesized CF template (pre-built rule packs; custom rules less flexible than Rego) |
| Policy-as-code language | N/A | Rego (OPA) — general-purpose, works across TF + K8s + APIs | cdk-nag rules (TypeScript/Python) + cfn-guard (Guard DSL) — CDK-specific only |
| Suppression/exception model | N/A | OPA `warn` vs `deny` with justification annotations | cdk-nag `NagSuppressions` with reason strings (can be gamed by developers using L1 constructs directly) |
| Bypass risk | Developers can use upstream modules directly | Developers can write raw TF (but OPA catches violations at plan time regardless of source) | Developers can bypass L2+ Constructs by using L1 Constructs directly, circumventing all security defaults |
| Auditability | Prisma post-deploy only | Pre-merge OPA results + Prisma post-deploy = dual-layer audit trail | cdk-nag reports + CF drift detection + Prisma post-deploy |
| Cross-tool reuse | N/A | OPA policies reusable for K8s admission control (Gatekeeper) | cdk-nag rules are CDK-specific; cannot reuse for K8s or other IaC tools |

**Assessment:** Option B has the strongest governance story. IAM policies are explicit and auditable. OPA/Conftest provides mature, flexible policy-as-code that works regardless of whether the developer used the CLI or wrote raw Terraform — the enforcement point is the plan output, not the module. Option C has a critical weakness: CDK's `grant()` convenience methods hide generated IAM policies, and developers can bypass L2+ Constructs by dropping to L1 Constructs. cdk-nag catches some violations, but the rule library is narrower than OPA's, and cdk-nag does not implement all rules from its referenced compliance frameworks.

### 4.3 Existing Investment & Migration Cost

| Criterion | Option A (Current) | Option B (TF + CLI) | Option C (CDK) |
|---|---|---|---|
| Existing modules | 40+ (reused as-is) | Refactored incrementally (3 modules in 3 months, remainder over ~6 months) | Full rewrite as CDK Constructs (estimated 9–12 months for 40+ modules) |
| CI/CD integration | Works with Harness today | Works with GitLab CI + Harness today; adds OPA stage | Requires new pipeline stages (`cdk synth` → `cdk-nag` → `cdk deploy`); Harness CDK integration is less mature than TF |
| State management | Terraform state in S3 (team controls) | Same | CloudFormation stacks (AWS controls); state is opaque; no `terraform state mv` equivalent for refactoring |
| Policy tooling | OPA + Snyk exist (underutilized) | OPA + Snyk fully wired into CI | New: cdk-nag + cfn-guard (team must learn two new tools) |
| Team skills | Terraform (strong), Python (strong) | Terraform (strong), Python (strong) | CDK (zero), Python (strong but CDK framework is a separate skill) |
| Training investment | None | Minimal (CLI usage; TF skills transfer) | Significant (CDK framework, CloudFormation mental model, Construct patterns, cdk-nag) |
| Risk of partial adoption | N/A | Low — CLI can coexist with existing modules during migration | High — running TF + CDK creates two governance stacks, two pipelines, two mental models |

**Assessment:** Option B has the lowest migration cost and risk. It builds on existing Terraform and Python skills with no new tools to learn. Option C requires a full rewrite of 40+ modules, new CI/CD pipeline stages, new governance tooling (cdk-nag, cfn-guard), and team training — all with a 4–5 person team on a one-quarter deadline.

### 4.4 Operational Characteristics

| Criterion | Option A (Current) | Option B (TF + CLI) | Option C (CDK) |
|---|---|---|---|
| Deployment speed | Standard TF (direct API calls) | Standard TF (direct API calls) | Slower — CDK deploys via CloudFormation, which is approximately 2–3x slower than Terraform for the same resource set due to the intermediary service layer |
| Drift detection | `terraform plan` against live state | `terraform plan` against live state | CF drift detection (requires manual trigger or automation) |
| Rollback | `terraform apply` with previous state | `terraform apply` with previous state | CF automatic rollback on stack failure (advantage) |
| State inspection | `terraform state list/show` (granular) | Same | CF console or CLI (less granular, no equivalent of `state mv` for refactoring) |
| AFT compatibility | Works with AFT global/account customizations | Works with AFT global/account customizations | CF stacks in AFT would require custom integration; AFT is designed for Terraform |
| Multi-cloud potential | Full (Terraform supports all major providers) | Full | None (CDK is AWS-only) |

**Assessment:** Option B and Option A share Terraform's operational strengths (fast deployment, granular state control, AFT compatibility). Option C trades deployment speed and state control for CloudFormation's automatic rollback capability — a meaningful advantage for some teams, but not a priority given your existing Terraform workflows.

### 4.5 Long-Term Strategic Fit

| Criterion | Option A (Current) | Option B (TF + CLI) | Option C (CDK) |
|---|---|---|---|
| CDKTF (TF + CDK hybrid) | N/A | N/A | CDKTF was deprecated by HashiCorp in December 2025; the "best of both worlds" escape hatch is closed |
| Terraform licensing risk | BUSL license since v1.5 | BUSL license (OpenTofu is a viable fallback) | N/A (CDK is Apache 2.0) |
| AWS lock-in | Low (TF is cloud-agnostic) | Low (TF is cloud-agnostic) | High (CDK is AWS-only by design) |
| OPA policy reuse for K8s | Possible (Gatekeeper uses OPA) | Planned (OPA policies reusable across TF + K8s) | Not possible (cdk-nag is CDK-specific) |
| Community ecosystem | Largest IaC ecosystem | Same | Growing but smaller; Construct Hub has fewer battle-tested constructs than Terraform Registry |

**Assessment:** Option B has the best long-term strategic fit. OPA policies written for Terraform plan validation can be reused for Kubernetes admission control via Gatekeeper — a significant advantage given the EKS Automode + on-prem Kubernetes architecture. CDK's AWS-only nature and the CDKTF deprecation eliminate the hybrid path. Terraform's BUSL licensing is a risk factor, but OpenTofu provides a viable escape hatch if needed.

## 5. Decision

**Selected: Option B — Native Terraform + CLI Factory + Policy-as-Code**

### Rationale

Option B delivers the best balance across all decision drivers:

1. **Developer experience:** The CLI factory closes ~80% of CDK's developer experience advantage by hiding HCL behind `infra create` commands. Developers interact with CLI commands and YAML schemas, not Terraform internals.

2. **Security & governance:** IAM policies remain explicit and auditable in composition module code. OPA/Conftest provides the most mature and flexible pre-apply compliance gate. The critical CDK weakness — developers bypassing L2+ Constructs via L1 Constructs — does not exist in Option B because OPA enforcement operates on the plan output regardless of how the Terraform was authored.

3. **Team capability:** Builds entirely on existing Terraform + Python skills. No new frameworks, no new governance tools, no training gap.

4. **Migration cost:** Incremental — 3 modules in 3 months, remainder over ~6 months. Coexists with existing modules during migration. CDK would require a 9–12 month full rewrite with no coexistence path.

5. **Strategic alignment:** OPA policy reuse across Terraform and Kubernetes (via Gatekeeper). AFT compatibility preserved. Cloud-agnostic foundation maintained.

### What We're Explicitly Trading Away

- **CDK's type-safe, IDE-native developer experience** for developers who want to write infrastructure code directly. We accept this because our CLI factory means most developers don't write infrastructure code at all — they run a command and get generated configs.

- **CDK's L2/L3 Construct abstractions** that hide AWS complexity behind good defaults. We replicate this via composition modules + YAML schema defaults, which requires more manual work but gives us full visibility.

- **CloudFormation's automatic stack rollback on failure.** We accept this because Terraform's explicit plan → apply workflow gives us pre-apply validation that CloudFormation lacks.

## 6. Consequences

### Positive

- No new tools or frameworks to learn; team is productive immediately
- OPA policies are reusable across Terraform and Kubernetes workloads
- AFT global/account customizations continue to work as-is
- Incremental migration reduces delivery risk
- Explicit IAM policies enable clean audit trail for compliance

### Negative

- App developers who prefer writing Python code for infrastructure don't get that option (mitigated: CLI factory abstracts this away)
- Terraform BUSL licensing is a strategic risk (mitigated: OpenTofu fallback, plus schema-driven design makes engine swap feasible)
- HCL's dynamic logic is more verbose than Python for complex conditional infrastructure (mitigated: complexity lives in Jinja2 templates maintained by platform team, not in developer-facing code)

### Neutral

- Prisma Cloud remains as post-deploy safety net regardless of which option is chosen
- All three options would eventually need wrapper/Construct uplift for documentation and testing
- Module version upgrade burden exists in all three options (TF provider versions, CDK Construct Library versions, or wrapper HCL)

## 7. Re-evaluation Triggers

This decision should be revisited if any of the following occur:

1. **Team size grows to 10+ engineers** with dedicated CDK expertise — the training cost argument weakens significantly at scale
2. **AWS launches CDK support in AFT** as a first-class customization engine — eliminates the AFT compatibility concern
3. **OpenTofu diverges significantly from Terraform** — may force an engine decision independent of the wrapper strategy
4. **OPA/Conftest ecosystem for CloudFormation templates matures** to parity with Terraform plan evaluation — weakens the governance argument
5. **Organization adopts a second cloud provider** — reinforces the Terraform decision (CDK is AWS-only)
6. **Internal Developer Portal (Backstage)** is adopted — CDK Constructs could serve as backend provisioning engine behind a portal UI, where developers never see IaC code directly

## 8. Sign-Off

| Role | Name | Date | Decision |
|------|------|------|----------|
| Platform Engineering Lead | | | |
| Security Architecture | | | |
| Cloud Engineering Lead | | | |
| VP Engineering | | | |

---

## Appendix: References

1. **CDK vs Terraform 2026 comparison** — CDK wins for AWS-native teams with programming language experience; Terraform wins for multi-cloud and platform teams. CDKTF deprecation in December 2025 closed the hybrid path. See [Towards The Cloud CDK vs Terraform](https://towardsthecloud.com/blog/aws-cdk-vs-terraform).

2. **CDK governance bypass risk** — Enterprise CDK best practices explicitly warn against relying on wrapper Constructs as the sole compliance mechanism because developers can bypass them using L1 Constructs directly. Multiple enforcement layers (SCPs, cdk-nag, cfn-guard, Aspects) are required. See [Towards The Cloud CDK Best Practices](https://towardsthecloud.com/blog/aws-cdk-best-practices).

3. **cdk-nag capabilities and limitations** — Provides rule packs for AWS Solutions, HIPAA, NIST 800-53, and PCI DSS, but does not implement all rules from the referenced compliance frameworks. Custom rule authoring is less flexible than OPA/Rego. See [AWS DevOps Blog on cdk-nag](https://aws.amazon.com/blogs/devops/manage-application-security-and-compliance-with-the-aws-cloud-development-kit-and-cdk-nag/) and [cdk-nag GitHub](https://github.com/cdklabs/cdk-nag).

4. **CDK Property Injection (May 2025)** — Enables organizations to standardize default construct properties via `IPropertyInjector` interface. Addresses the "secure defaults" concern but does not prevent L1 bypass. See [AWS CDK in Action May 2025](https://dev.to/aws/aws-cdk-in-action-may-2025-empowered-deployments-governance-and-community-1gdb).

5. **CDK IAM grant methods** — AWS CDK documentation acknowledges that auto-generated IAM roles via grant methods meet "most organizations'" security requirements but explicitly notes that manually created roles provide more control at the cost of developer velocity. See [AWS CDK Security Best Practices](https://docs.aws.amazon.com/cdk/v2/guide/best-practices-security.html).

6. **Terraform Wrapper Tax** — Analysis of how abstraction layers over cloud provider modules create feature lag, debugging complexity, and cognitive load. See [Rack2Cloud Wrapper Tax](https://www.rack2cloud.com/terraform-multi-cloud-anti-patterns-wrapper-tax/).

7. **IaC tool governance comparison** — Terraform with Sentinel/OPA has the most mature enterprise governance story; CDK requires assembling AWS-native controls with more intentional architecture. See [StackGen Terraform vs Pulumi vs CDK](https://stackgen.com/blog/terraform-vs-pulumi-vs-cdk).

8. **CDKTF deprecation** — HashiCorp archived CDKTF in December 2025 with no maintenance, updates, or compatibility work going forward. Continued use is at your own risk. See [Spacelift Terraform Test Strategies](https://spacelift.io/blog/terraform-test).

9. **CDK deployment speed** — CDK deploys via CloudFormation, which is approximately 2–3x slower than Terraform for equivalent resource sets due to the intermediary service layer. See [Towards The Cloud CDK vs Terraform benchmarks](https://towardsthecloud.com/blog/aws-cdk-vs-terraform).

10. **OPA reuse across Terraform and Kubernetes** — OPA policies written in Rego can be used for both Terraform plan validation (via Conftest) and Kubernetes admission control (via Gatekeeper), enabling a single policy language across the infrastructure stack. See [Spacelift OPA guide](https://spacelift.io/blog/open-policy-agent-opa-terraform).