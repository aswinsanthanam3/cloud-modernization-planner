# Cloud Platform Taxonomy: Realms, Zones, Environments, and Environment Classes

## Conceptual Architecture — Greenfield Cloud Platform Design

| Attribute       | Value                                      |
|-----------------|--------------------------------------------|
| Status          | DRAFT — Revision 2                         |
| Author          | Platform Engineering                       |
| Date            | 2026-04-13                                 |
| Classification  | Internal — Confidential                    |
| Scope           | AWS Cloud Platform (net-new design)        |
| Compliance      | PCI-DSS v4.0, NIST 800-53 Rev 5           |

### Revision History

| Rev | Date       | Summary of Changes                                                                  |
|-----|------------|--------------------------------------------------------------------------------------|
| 1   | 2026-04-13 | Initial draft — three realms, four environments, zone model                          |
| 2   | 2026-04-13 | Dev/QA moved into Product realm; Environment Class introduced; environments expanded to Dev, QA, Pre-Prod, Post-Prod, Prod, Shadow; OU hierarchy, TGW route tables, promotion pipeline, and control matrices updated throughout |

---

## 1. Purpose and Scope

This document defines a foundational taxonomy for organizing a cloud platform into **Realms**, **Zones**, **Environment Classes**, and **Environments**. The taxonomy serves as the primary architectural framework for enforcing network segmentation, data isolation, security control inheritance, and blast radius containment across the platform.

The design is greenfield — it describes the target-state conceptual model for a cloud-native platform without being constrained by current on-premises or cloud implementations. However, it is explicitly designed to be adoptable incrementally, providing a migration path from existing three-realm data center heritage architectures without requiring a disruptive cutover.

### 1.1 Design Principles

1. **Isolation by default, connectivity by exception.** Realms, zones, and environments are isolated from each other unless an explicit, documented, and auditable connectivity path is established.

2. **Least privilege at every boundary.** Each boundary (realm, zone, environment class, environment) enforces its own least-privilege controls. Controls are inherited downward and can only be tightened, never relaxed.

3. **Environment is a per-realm attribute, not a cross-cutting layer.** Each realm manages its own environment lifecycle independently. A "Production" environment in the Product realm has no implicit trust relationship with a "Production" environment in the Management realm.

4. **Full product lifecycle within the Product realm.** The Product realm owns the complete promotion pipeline from development through production. This ensures consistent SCPs, network topology, and governance across the entire lifecycle, eliminates cross-realm promotion complexity, and gives product executives full ownership and cost visibility over their infrastructure.

5. **PCI scope is minimized by design.** The Cardholder Data Environment (CDE) is architecturally isolated to the smallest possible boundary, reducing compliance scope, audit surface, and operational overhead for all workloads outside the CDE.

6. **Three realms today, five tomorrow.** The taxonomy uses three top-level realms with sub-realm segmentation, but the underlying account structure, network topology, and policy hierarchy are designed for future expansion to five or more realms without re-architecture.

7. **Cloud-native constructs are first-class.** The taxonomy maps directly to AWS organizational constructs (OUs, accounts, SCPs, VPCs, TGW route tables) rather than being a conceptual overlay that requires manual translation.

8. **Data classification drives control intensity.** The security controls applied at each boundary are determined by the data classification of the assets within that boundary, not by arbitrary realm or zone labels.

---

## 2. Current State Problems

The following problems in the current cloud platform design motivate the greenfield taxonomy defined in this document. They are presented here so that each design decision in subsequent sections can be traced to the specific problem it addresses.

### 2.1 Realm Taxonomy Is Data-Center Heritage, Not Cloud-Native

The three-realm model (Product, Management, Corporate) was designed for on-premises data centers where physical network segments defined trust boundaries. It was lifted into AWS without rethinking what realms should mean in a cloud context where accounts, OUs, and TGW route tables are the native isolation constructs. The realm boundaries do not map cleanly to AWS primitives, creating a gap between the conceptual model and the actual enforcement mechanisms.

### 2.2 PCI Scope Is Over-Inclusive

The entire Product realm is considered in PCI scope today, even though only a subset of services actually handle cardholder data. Every account, every developer, and every piece of infrastructure in the Product realm carries PCI compliance overhead — training requirements, audit surface, and operational restrictions — regardless of whether it touches payment card data. There is no architectural CDE boundary that limits scope to only the systems that need it.

### 2.3 Dev/QA Sits in the Wrong Realm

Development and QA environments live in the Corporate realm while Certification and Production live in the Product realm. This creates several downstream problems:

- A cross-realm promotion boundary where artifacts must jump trust domains during the normal development lifecycle, adding complexity and fragility.
- Developers build against Corporate realm SCPs and network topology, then discover policy violations when promoting to the Product realm — a "works in dev, breaks in cert" problem.
- Product executives do not have full ownership or cost visibility over the infrastructure that supports their products because part of it lives in another realm's governance umbrella.
- The promotion pipeline requires mediation between two realms for what should be a routine intra-team workflow.

### 2.4 Inconsistent Environment Semantics Across Product Lines

Different product lines use the same environment names (e.g., "UAT") to mean fundamentally different things. For one product, UAT serves as an internal sign-off environment requiring gateway access to internal systems and production-like controls. For another product, UAT is a merchant-facing testing environment that needs external connectivity and a different access model entirely.

This inconsistency means the same environment label carries different network topologies, different IAM policies, and different security postures depending on which product line owns it. There is no standard taxonomy that defines what each environment is *for*, what data it may contain, and who may access it — making it impossible to apply consistent controls, automate provisioning, or reason about security posture at the platform level.

### 2.5 Overly Restrictive IAM in Non-Production Environments

Because some non-production environments are treated as production-grade (due to the inconsistent semantics described above), IAM access controls are applied at a production level of restrictiveness across environments that do not contain or process real customer data. This creates significant developer friction — engineers face unnecessary approval workflows, limited console access, and constrained permissions in environments where they need the ability to experiment, debug, and iterate quickly. The root cause is the absence of a formal data classification and environment class model that ties access control intensity to actual data sensitivity rather than to environment naming conventions.

### 2.6 Security Services Not Architecturally Independent

Platform services (CI/CD, IaC, observability) and security services (audit logs, detective controls, incident response) both sit within the Management realm without clear sub-realm separation. A compromise of the CI/CD pipeline — one of the highest-value targets — could potentially provide a path to audit logs, security tooling, and detective controls. The security infrastructure has no guaranteed independence from the platform it is supposed to monitor and protect.

### 2.7 Single Transit Gateway Is a Single Point of Failure

The organization operates a single Transit Gateway for all realms, environments, and product lines. This creates two critical problems:

- **Blast radius is organization-wide.** A single misconfiguration — such as a contractor mistakenly deleting a TGW attachment — can sever network connectivity for the entire company, across all realms and all environments simultaneously. There is no fault isolation between realms or environment classes at the network transit layer.
- **No route domain isolation.** Without per-realm or per-environment-class route tables, there is no network-level enforcement preventing a compromised dev account from reaching production, or a corporate workstation from reaching a product environment directly. All routing decisions rely on security groups and NACLs alone, without the organizational-level network segmentation that TGW route tables provide.

### 2.8 Zone Model Is Implicit, Not Formalized

Functional network segmentation (edge, application, data, integration, management tiers) exists in practice but is not codified as an architectural standard with defined controls, AWS construct mappings, and traffic flow rules. Zone boundaries are inconsistently implemented across accounts and environments, and there is no systematic way to audit whether a given account's VPC topology conforms to the expected zone model.

### 2.9 No Formal Promotion Pipeline with Gates

Artifact promotion from dev through to production does not follow a documented, enforced unidirectional flow with explicit gates (security scanning, artifact signing, regression testing, change approval) at each environment class boundary. There is no systematic assurance that artifacts deployed to production have passed all required validations, and no architectural enforcement preventing a direct push from a lower environment to production.

### 2.10 Account Vending Is Slow and Manual

It currently takes approximately 45 business days to provision a new AWS account. Without standardized realm, zone, and environment class definitions, each new account requires manual wiring of network connectivity, IAM policies, SCP attachment, logging configuration, and security tooling. The absence of formalized account archetypes (driven by realm + environment class) means there is no template-driven automation — every account is a bespoke configuration exercise, creating both lead time and inconsistency.

The environment class and zone taxonomy defined in this document directly enables account vending automation: an account archetype is the intersection of a realm, sub-realm, environment class, and zone profile. Once these dimensions are standardized, AFT account factory templates can be parameterized by archetype, reducing provisioning from 45 business days to same-day.

### 2.11 Problem-to-Solution Traceability

The following table maps each current state problem to the specific design elements in this document that address it:

| Problem | Addressed By |
|---|---|
| 2.1 DC-heritage realm taxonomy | Section 4: Cloud-native realm model with AWS construct mapping |
| 2.2 Over-inclusive PCI scope | Section 4.1: CDE sub-realm with account-level isolation |
| 2.3 Dev/QA in wrong realm | Section 4.1: Full lifecycle within Product realm |
| 2.4 Inconsistent environment semantics | Section 6: Environment Class definitions with standard semantics |
| 2.5 Overly restrictive Non-Prod IAM | Section 6.5: Class-specific controls; Section 7: Data classification drives access |
| 2.6 Security services not independent | Section 4.1: Security Services sub-realm |
| 2.7 Single TGW / single point of failure | Section 8.3: Seven TGW route tables with per-class isolation |
| 2.8 Implicit zone model | Section 5: Formalized zone taxonomy with AWS mappings |
| 2.9 No promotion gates | Section 6.4: Promotion pipeline with gates at each class boundary |
| 2.10 45-day account vending | Section 8.1: OU hierarchy enabling template-driven AFT automation |

---

## 3. Taxonomy Overview

The taxonomy consists of four hierarchical dimensions:

```
Realm (Trust Domain)
  └── Zone (Functional Network Segment)
       └── Environment Class (Governance Grouping)
            └── Environment (Lifecycle Stage)
```

Each dimension serves a distinct purpose:

| Dimension         | Purpose                                        | Boundary Type            | AWS Primary Construct              |
|-------------------|-------------------------------------------------|--------------------------|------------------------------------|
| Realm             | Highest-level trust domain                      | Organizational + Network | OU hierarchy + TGW route domain    |
| Zone              | Functional network segmentation within a realm  | Network + IAM            | VPC subnet tier + Security Groups  |
| Environment Class | Governance grouping of related environments     | Organizational + Policy  | OU tier + SCP set + TGW route table|
| Environment       | Individual lifecycle stage within a class        | Account + IAM            | Account(s) + IAM policies          |

**Figure 1: Taxonomy Hierarchy — Four Dimensions with AWS Construct Mapping**

![Taxonomy Hierarchy](diagrams/01-taxonomy-hierarchy.svg)

### 3.1 Why Four Dimensions

Traditional data center designs often conflate two distinct concerns into a single "zone" or "environment" concept. This taxonomy separates them explicitly:

- **Zones** solve **lateral movement containment** within a running system. If an attacker compromises the application tier, zones prevent direct access to the data tier. This is a runtime network security control, implemented via VPC subnets and security groups.

- **Environment Classes** solve **lifecycle governance grouping**. They cluster related environments (e.g., Dev and QA together in the Development class) that share a common control profile — SCP restrictiveness, data posture, access model, and network connectivity. This is an organizational and policy control, implemented via OU tiers and TGW route tables.

- **Environments** are individual lifecycle stages within a class. They are the unit of account provisioning and artifact deployment.

These three dimensions are orthogonal — they intersect, not replace each other. A Production environment still has edge, application, and data zones internally. A Dev environment also has those zones (or a subset). The environment class determines *what controls apply* to that environment; the zone determines *where within the environment* different security postures are enforced.

### 3.2 Why Environment Is Per-Realm, Not Cross-Cutting

Traditional data center designs often treat environments as a cross-cutting dimension — a single "Dev" VLAN spans all application tiers, a single "Prod" firewall zone spans all workloads. This creates several problems in cloud:

- **Blast radius expansion.** A compromise in any dev workload can potentially reach any other dev workload, regardless of realm or trust level, because they share a network boundary.
- **Policy collision.** SCPs and IAM policies that are appropriate for dev in the Product realm (e.g., permitting experimental services) may be inappropriate for dev in the Management realm (where CI/CD pipelines have elevated privileges).
- **Compliance contamination.** If a PCI-scoped product workload shares a "Dev" environment boundary with non-PCI workloads, the QSA may pull the entire shared environment into PCI scope.
- **Promotion pipeline coupling.** Different realms have different release cadences and promotion gates. Cross-cutting environments force a single promotion model on all realms.

In the target-state model, each realm owns its own environment lifecycle. The Product realm has its own Development/Validation/Production environment classes. The Management realm has its own. There is no shared "Dev" boundary that spans realms.

---

## 4. Realm Model

### 4.1 Realm Definitions

The platform defines three top-level realms, each with optional sub-realm segmentation:

#### Realm 1: Product

| Attribute            | Value                                                                  |
|----------------------|------------------------------------------------------------------------|
| Purpose              | External, revenue-generating workloads — full lifecycle from development through production |
| Trust Level          | Highest sensitivity — processes customer data, financial transactions  |
| Data Classifications | Restricted (CDE only), Confidential, Internal (Dev/QA)                |
| Compliance Scope     | PCI-DSS (CDE sub-realm only), SOC 2, NIST                            |
| Connectivity         | Internet-facing (via edge zone in Validation/Production classes), internal-only (Development class) |
| Sub-Realms           | **CDE** (PCI-scoped), **General Product** (non-PCI)                   |

The Product realm owns the complete workload lifecycle: development, QA, pre-production validation, merchant-facing certification, live production, and migration back-testing. This ensures product executives have full ownership, cost visibility, and governance over their infrastructure from first line of code to production traffic.

**Sub-Realm: CDE (Cardholder Data Environment)**

The CDE sub-realm contains *only* the systems that store, process, or transmit cardholder data (CHD) or sensitive authentication data (SAD). All other Product workloads reside in the General Product sub-realm and are *out of PCI scope*.

CDE isolation is enforced at the account level — CDE workloads run in dedicated AWS accounts with their own VPCs, SCPs, and TGW attachments. There is no shared account boundary between CDE and non-CDE workloads.

Data flows between CDE and General Product sub-realms are permitted only through controlled, audited integration points (e.g., tokenization services that sit within the CDE and expose token-only APIs to General Product). These integration points are the PCI scope boundary — they must be explicitly documented and validated by the QSA.

**Scope Reduction Strategy:**

| Before (Current State)                          | After (Target State)                                    |
|-------------------------------------------------|---------------------------------------------------------|
| Entire Product realm is in PCI scope            | Only CDE sub-realm accounts are in PCI scope            |
| All Product developers subject to PCI training  | Only CDE-authorized engineers subject to PCI training   |
| All Product infra subject to PCI audit          | Only CDE infrastructure subject to PCI audit            |
| PCI controls applied uniformly, increasing cost | PCI controls concentrated, reducing compliance overhead  |

**Critical architectural constraint for PCI scope containment:** The CDE boundary is defined at the sub-realm/OU level, not the realm level. Dev/QA accounts in the Product realm are *not* in PCI scope because they reside in a separate OU (General Product → Development Class) with no network path to the CDE, no access to cardholder data, and no shared infrastructure with CDE accounts. This distinction must be documented and validated with the QSA before account migration.

**Sub-Realm: General Product**

All non-PCI product workloads across the full lifecycle — from Dev through Production — reside here. The General Product sub-realm uses environment classes (Development, Validation, Production) to provide governance segmentation within the sub-realm.

#### Realm 2: Management

| Attribute            | Value                                                                   |
|----------------------|-------------------------------------------------------------------------|
| Purpose              | Platform shared services, tooling, and security infrastructure          |
| Trust Level          | Elevated — controls CI/CD pipelines, IaC state, observability, identity |
| Data Classifications | Confidential, Internal                                                   |
| Compliance Scope     | SOC 2, NIST (as supporting infrastructure for PCI workloads)            |
| Connectivity         | Hub for inter-realm transit; no direct internet ingress                  |
| Sub-Realms           | **Platform Services**, **Security Services**                             |

**Sub-Realm: Platform Services**

Encompasses CI/CD orchestration (Harness, GitLab runners), IaC state management (Terraform state), container registries, artifact repositories, observability backends (Honeycomb, Splunk forwarders), IPAM integration, and account vending (AFT).

**Sub-Realm: Security Services**

Encompasses audit log aggregation (CloudTrail org trail), security detective controls (Security Hub delegated admin, GuardDuty admin, Prisma Cloud), compliance scanning (AWS Config aggregator, Prowler), incident response tooling, and forensic isolation accounts.

The architectural separation between Platform Services and Security Services ensures that a compromise of the CI/CD pipeline (a high-value attack target) does not grant access to audit logs or security tooling. Even if both sub-realms share a top-level OU initially, the account boundaries, IAM policies, and network paths are designed for hard separation.

**Key Constraint:** The Security Services sub-realm should have *no inbound dependencies* on the Platform Services sub-realm. Security tooling must not rely on the CI/CD pipeline for deployment — it should be deployable independently, even if the rest of the platform is compromised. This is a foundational principle for incident response.

#### Realm 3: Corporate

| Attribute            | Value                                                                 |
|----------------------|-----------------------------------------------------------------------|
| Purpose              | End-user computing, internal tools, office connectivity, sandboxing   |
| Trust Level          | Standard — internal users, non-production data                        |
| Data Classifications | Internal, Public                                                      |
| Compliance Scope     | NIST (baseline)                                                       |
| Connectivity         | VPN/Direct Connect to on-premises, internet egress, no direct Product |
| Sub-Realms           | None initially                                                        |

Corporate encompasses jump servers, VPN termination, SSO integration, internal business tools, and sandbox accounts for experimentation. With Dev/QA moving to the Product realm, Corporate no longer hosts product development workloads — it focuses on end-user computing infrastructure and experimentation.

**Figure 2: Realm and Environment Class Topology**

![Realm and Environment Class Topology](diagrams/02-realm-topology.svg)

### 4.2 Realm Interaction Model

Realms interact through controlled, auditable transit paths. The default posture is **no connectivity** between realms.

```
                    ┌─────────────────────────────────────────────┐
                    │              TRANSIT GATEWAY                │
                    │         (Realm Interconnect Layer)          │
                    └──────┬──────────────┬──────────────┬────────┘
                           │              │              │
              ┌────────────▼───┐  ┌───────▼────────┐  ┌──▼───────────┐
              │   PRODUCT      │  │  MANAGEMENT    │  │  CORPORATE   │
              │   REALM        │  │  REALM         │  │  REALM       │
              │                │  │                │  │              │
              │ ┌────────────┐ │  │ ┌────────────┐ │  │  Sandbox     │
              │ │    CDE     │ │  │ │  Security  │ │  │  Jump Hosts  │
              │ │ Sub-Realm  │ │  │ │  Services  │ │  │  VPN / SSO   │
              │ └────────────┘ │  │ └────────────┘ │  │  Internal    │
              │ ┌────────────┐ │  │ ┌────────────┐ │  │  Tools       │
              │ │  General   │ │  │ │  Platform  │ │  │              │
              │ │  Product   │ │  │ │  Services  │ │  │              │
              │ │            │ │  │ └────────────┘ │  │              │
              │ │ Dev → QA → │ │  │                │  │              │
              │ │ PreProd →  │ │  │                │  │              │
              │ │ PostProd → │ │  │                │  │              │
              │ │ Prod/Shadow│ │  │                │  │              │
              │ └────────────┘ │  │                │  │              │
              └────────────────┘  └────────────────┘  └──────────────┘
```

**Permitted Inter-Realm Flows:**

| Source Realm       | Destination Realm       | Permitted Flows                                             | Control Mechanism         |
|--------------------|-------------------------|-------------------------------------------------------------|---------------------------|
| Product            | Management              | Log forwarding, metrics export, audit trail                 | TGW route + SG + NACLs    |
| Management         | Product                 | CD pipeline deployments, config pushes, artifact scanning   | TGW route + IAM role chain |
| Corporate          | Management              | Developer access to CI/CD dashboards, IaC pipelines         | TGW route + JIT PAM       |
| Corporate          | Product                 | **Denied.** No direct path from Corporate to any Product environment. | TGW route table isolation  |
| Management         | Corporate               | Monitoring dashboards, alerting                             | TGW route + SG             |
| Product (CDE)      | Product (General)       | Tokenized data only, via controlled integration points      | Dedicated TGW route + WAF  |
| Any                | Security Services       | Log/event forwarding (one-way push)                         | TGW route (unidirectional) |
| Security Services  | Any                     | Read-only investigation, forensic snapshot                  | Cross-account IAM roles    |

**Denied by Default:**

- Corporate → Product (any environment): There is no direct network path from end-user environments to any Product realm environment — including Dev/QA. Developers access Product Dev/QA environments through the Management realm's CI/CD tooling and via JIT PAM for console/CLI access. This ensures the Corporate realm cannot be used as a lateral movement path into Product infrastructure.
- Product → Corporate: Product workloads never initiate connections to corporate infrastructure.
- Any realm → Security Services (write): No realm can write to or modify security tooling; flows are unidirectional (push logs/events in, no commands out).

**Developer Access Pattern (Corporate → Product Dev/QA):**

Developers physically sit in the Corporate realm (laptops, VPN, SSO). To access Product Dev/QA environments, they use:

1. **Code deployment:** Push code to GitLab (Corporate) → CI pipeline runs in Management realm → CD pipeline (Harness) deploys to Product Dev/QA accounts. No direct Corporate-to-Product connectivity required.
2. **Console/CLI access:** Request via ServiceNow → JIT PAM grants temporary IAM Identity Center permission set scoped to specific Product Dev/QA accounts → access is time-bound, audited, and auto-revoked.
3. **Application testing:** Product Dev/QA environments expose internal endpoints via Management realm's connectivity layer (e.g., internal ALB accessible through TGW from Management, not from Corporate). Developers access through VPN → Corporate → Management → Product Dev/QA path.

This pattern preserves the realm boundary while enabling developer productivity.

---

## 5. Zone Model

Within each realm, **zones** provide functional network segmentation. Zones map to network tiers (VPC subnets and security groups) and represent the runtime security boundary within any given environment.

Zones are orthogonal to environment classes — every environment, regardless of its class, can have its own set of zones. A Production environment has edge, application, and data zones. A Dev environment also has those zones (or a subset), ensuring architectural representativeness across the lifecycle.

### 5.1 Standard Zone Taxonomy

Every realm uses a subset of the following standard zones:

| Zone            | Purpose                                           | Network Posture                    | Typical AWS Mapping                  |
|-----------------|---------------------------------------------------|------------------------------------|--------------------------------------|
| **Edge**        | Ingress/egress point; load balancers, WAF, CDN    | Internet-facing (public subnets)   | Public subnets + ALB/NLB + WAF       |
| **Application** | Compute workloads; containers, functions, services| Private subnets, no direct internet| Private subnets + EKS/ECS/Lambda     |
| **Data**        | Persistent storage; databases, caches, queues     | Private subnets, most restricted   | Isolated subnets + RDS/DynamoDB/S3   |
| **Integration** | Service mesh, API gateways, event buses           | Private subnets, cross-zone broker | Private subnets + API GW/EventBridge |
| **Management**  | Session manager, monitoring agents                | Private subnets, admin access only | Private subnets + SSM endpoints      |

### 5.2 Zone Mapping by Realm and Environment Class

#### Product Realm — General Product

Zones are consistent across environment classes, with restrictions increasing as the class moves toward production:

| Zone          | Development Class | Validation Class | Production Class | Notes |
|---------------|-------------------|-------------------|-------------------|-------|
| Edge          | Internal ALB only | Internet-facing (Pre-Prod: internal, Post-Prod: merchant-facing) | Internet-facing (Prod + Shadow) | Edge zone internet exposure only in Validation and Production classes |
| Application   | Yes               | Yes               | Yes               | EKS workloads, Lambda functions |
| Data          | Yes               | Yes               | Yes               | Synthetic data in Dev class; masked in Validation; real in Production |
| Integration   | Yes               | Yes               | Yes               | API Gateway, EventBridge |
| Management    | Yes               | Yes               | Yes               | SSM-only across all classes (no bastion) |

#### Product Realm — CDE Sub-Realm

| Zone          | Validation Class | Production Class | Notes |
|---------------|-------------------|-------------------|-------|
| Edge          | Dedicated ALB, dedicated WAF rule set (PCI-specific) | Same | CDE has no Development class |
| Application   | Hardened compute, no internet egress, FIM enabled | Same | |
| Data          | Encrypted at rest (KMS CMK, PCI key rotation), no S3 public | Same | |
| Integration   | Tokenization service endpoint (boundary to General Product) | Same | |
| Management    | Enhanced logging, session recording, MFA-enforced access | Same | |

**CDE-Specific Controls (all zones, all environment classes):**
- All zones enforce TLS 1.2+ for data in transit
- All data-at-rest uses KMS CMKs with automatic annual rotation
- No internet egress from any zone (all external calls via VPC endpoints or proxy)
- File Integrity Monitoring (FIM) on all compute instances
- Network flow logs retained for 12 months minimum
- Security group rules reviewed quarterly (automated drift detection)

#### Management Realm — Platform Services

| Zone          | Present | Notes                                                      |
|---------------|---------|-------------------------------------------------------------|
| Edge          | No      | No internet-facing services                                 |
| Application   | Yes     | CI/CD workers (Harness delegates), GitLab runners           |
| Data          | Yes     | Terraform state (S3 + DynamoDB), artifact stores, registries|
| Integration   | Yes     | Harness hub-spoke connectivity, AFT pipelines               |
| Management    | Yes     | Observability collectors, IPAM integration                  |

#### Management Realm — Security Services

| Zone          | Present | Notes                                                       |
|---------------|---------|-------------------------------------------------------------|
| Edge          | No      | No internet-facing services                                  |
| Application   | Yes     | Security automation (Lambda), forensic analysis instances    |
| Data          | Yes     | CloudTrail log archive (S3, Glacier), Security Hub findings  |
| Integration   | Yes     | EventBridge rules for security event routing                 |
| Management    | Yes     | Incident response tooling, forensic isolation sandbox        |

#### Corporate Realm

| Zone          | Present | Notes                                                      |
|---------------|---------|-------------------------------------------------------------|
| Edge          | Yes     | VPN termination, Direct Connect gateway                     |
| Application   | Yes     | Internal tools, sandbox experimentation workloads           |
| Data          | Yes     | Internal tool databases (no product data)                   |
| Integration   | Yes     | SSO integration (Okta), ServiceNow connectivity             |
| Management    | Yes     | Jump servers, bastion hosts, session manager                |

### 5.3 Zone Isolation Rules

Zones within any environment follow a **tiered trust model** — traffic flows are permitted inbound (edge → application → data) and denied in reverse unless explicitly allowed:

```
    INBOUND                                          OUTBOUND
    ──────►                                          ──────►

    ┌─────────┐    ┌───────────────┐    ┌──────────┐
    │  Edge   │───►│  Application  │───►│   Data   │
    │  Zone   │    │    Zone       │    │   Zone   │
    └─────────┘    └───────┬───────┘    └──────────┘
                           │
                   ┌───────▼───────┐
                   │ Integration   │
                   │    Zone       │
                   └───────────────┘

    Data → Application: Permitted (query responses)
    Data → Edge:        Denied
    Application → Edge: Denied (no direct internet from app tier)
    Management Zone:    Reachable from all zones for ops; no zone reachable from management
                        except via SSM/Session Manager (no SSH/RDP)
```

---

## 6. Environment Class and Environment Model

### 6.1 Environment Class Definitions

An **Environment Class** is a governance grouping that clusters related environments sharing a common control profile. Each class defines the SCP restrictiveness, data posture, access model, network connectivity, and TGW route table association for all environments within it.

| Environment Class | Purpose                                         | Data Posture                  | Access Model                                    | TGW Route Table  |
|-------------------|-------------------------------------------------|-------------------------------|-------------------------------------------------|------------------|
| **Development**   | Internal build, unit testing, integration testing | Synthetic / masked data only  | Developers (broad access), CI/CD pipelines       | Product-Dev-RT   |
| **Validation**    | External-facing pre-production and post-production validation | Masked / test data            | Internal teams + authorized external parties (merchants) | Product-Val-RT   |
| **Production**    | Live customer traffic and migration back-testing | Real customer / business data | CI/CD only + JIT break-glass (time-bound)       | Product-Prod-RT  |

### 6.2 Environment Definitions

Each environment class contains one or more environments:

#### Development Class

| Environment | Purpose                                                  | Lifespan   |
|-------------|----------------------------------------------------------|------------|
| **Dev**     | Active development, feature branches, unit testing        | Persistent |
| **QA**      | Integration testing, regression testing, internal sign-off | Persistent |

#### Validation Class

| Environment  | Purpose                                                  | Lifespan   |
|--------------|----------------------------------------------------------|------------|
| **Pre-Prod** | Release candidate validation before production deployment — internal teams verify the build is production-ready | Persistent |
| **Post-Prod**| Merchant-facing certification environment — external merchants and partners test their integrations against production-grade code before they go live on their end | Persistent |

#### Production Class

| Environment | Purpose                                                  | Lifespan                        |
|-------------|----------------------------------------------------------|---------------------------------|
| **Prod**    | Live customer-facing workloads serving real traffic       | Persistent                      |
| **Shadow**  | Migration back-testing — full production-equivalent where a parallel copy of a system runs during big bang migrations, comparing outputs between old and new implementations to build confidence before cutover | Ephemeral (migration window)    |

### 6.3 Environment Class Applicability by Realm

Not every realm uses every environment class:

| Realm / Sub-Realm       | Development | Validation  | Production  | Sandbox (standalone) |
|-------------------------|-------------|-------------|-------------|----------------------|
| Product — General       | Dev, QA     | Pre-Prod, Post-Prod | Prod, Shadow | —          |
| Product — CDE           | —           | Pre-Prod    | Prod        | —                    |
| Management — Platform   | Non-Prod    | —           | Prod        | —                    |
| Management — Security   | Non-Prod    | —           | Prod        | —                    |
| Corporate               | —           | —           | —           | Yes (ephemeral)      |

**Key Design Decisions:**

- **Product realm owns the full lifecycle.** Dev, QA, Pre-Prod, Post-Prod, Prod, and Shadow all reside within the Product realm. This gives product executives full ownership, eliminates cross-realm promotion complexity, and ensures developers build against the same SCP baseline and network topology they'll encounter in production.

- **CDE has no Development class.** CDE workloads are developed and tested in the General Product Development class using tokenized/mocked payment interfaces. Only validated builds are promoted into CDE Validation and CDE Production. This minimizes the number of accounts, developers, and infrastructure within PCI scope.

- **Corporate realm hosts only Sandbox.** With Dev/QA in the Product realm, Corporate no longer hosts product development workloads. Sandboxes remain in Corporate because they require fundamentally different controls (no connectivity to other realms, auto-cleanup, relaxed SCPs for experimentation) that would conflict with the Product realm's governance model.

- **Management realm uses a simplified two-tier model.** Platform and security tooling needs a staging environment (Non-Prod) for testing upgrades before rolling to production, but does not need the full Development/Validation/Production class structure.

- **Shadow is ephemeral.** Shadow environments are provisioned during migration windows, run for the duration of the back-testing period, and are decommissioned afterward. While active, they inherit Production class controls because they process production-scale data. Shadow accounts should be pre-provisioned via AFT with a "dormant" state and activated on-demand.

### 6.4 Promotion Pipeline

The environment promotion model enforces a unidirectional flow. Artifacts (container images, IaC modules, Lambda packages) are built once and promoted through environments — never rebuilt at each stage.

**Figure 4: Artifact Promotion Pipeline**

![Promotion Pipeline](diagrams/04-promotion-pipeline.svg)

```
  PRODUCT REALM — GENERAL PRODUCT
  ════════════════════════════════

  DEVELOPMENT CLASS          VALIDATION CLASS                PRODUCTION CLASS
  ─────────────────          ────────────────                ────────────────

  ┌───────┐   ┌──────┐     ┌──────────┐   ┌───────────┐    ┌──────┐   ┌────────┐
  │  Dev  │──►│  QA  │────►│ Pre-Prod │──►│ Post-Prod │───►│ Prod │   │ Shadow │
  │       │   │      │     │          │   │(Merchant) │    │      │   │(Migr.) │
  └───────┘   └──────┘     └──────────┘   └───────────┘    └──────┘   └────────┘
       │           │             │               │              │           │
   Feature     Integration   Release         Merchant       Live        Back-test
   dev &       & regression  candidate       integration    customer    migration
   unit test   testing       validation      certification  traffic     comparison


  PROMOTION GATES:
  Dev → QA:        Code review + unit test pass
  QA → Pre-Prod:   Integration tests + security scan (Snyk, Prisma) + artifact signing
  Pre-Prod → Post-Prod: Full regression pass + performance benchmarks
  Post-Prod → Prod: Merchant sign-off (where applicable) + change approval (CAB)
  Shadow:          Provisioned independently for migration back-testing (not part of linear promotion)


  CORPORATE REALM (ISOLATED)
  ══════════════════════════

  ┌───────────┐
  │  Sandbox  │   No promotion path to any other environment.
  │(ephemeral)│   Auto-cleaned. Internet-only. No TGW attachment.
  └───────────┘
```

**Artifact Promotion Mechanics:**

All promotion flows through the Management realm's artifact pipeline:

1. Developer pushes code to GitLab (in Corporate realm, via SSO).
2. GitLab CI (runners in Management realm) builds and runs unit tests.
3. On CI pass, the artifact is published to the **Development artifact registry** (in Management realm's Platform Services).
4. CD pipeline (Harness, in Management realm) deploys to Product Dev, then QA.
5. On QA pass, artifact is scanned (Snyk, Prisma container scan) and signed.
6. Signed artifact is published to the **Validated artifact registry** (separate registry in Management realm).
7. CD pipeline deploys to Pre-Prod, then Post-Prod, then Prod — pulling only from the Validated registry.

At no point does a developer workstation or Corporate account directly push artifacts to any Product account. The Management realm mediates all deployments.

### 6.5 Environment-Class-Specific Controls

| Control                          | Sandbox (Corp) | Development     | Validation      | Production       |
|----------------------------------|----------------|-----------------|-----------------|------------------|
| SCP restrictiveness              | Moderate       | Moderate-High   | High            | Highest          |
| Internet egress                  | Permitted      | Permitted (NAT) | Restricted      | Denied (VPC endpoints only) |
| Real customer data               | Prohibited     | Prohibited      | Prohibited      | Permitted        |
| PII/CHD (even masked)           | Prohibited     | Prohibited      | Masked only     | Per classification |
| MFA for console access          | Optional       | Required        | Required        | Required + JIT   |
| Change approval                 | None           | Peer review     | CAB review      | CAB + auto-rollback |
| Auto-cleanup                    | 7–30 days      | None            | None            | None (Shadow: post-migration) |
| Audit log retention             | 30 days        | 90 days         | 1 year          | 7 years (PCI)    |
| Deployment method               | Manual OK      | CI/CD           | CI/CD only      | CI/CD only       |
| Drift detection                 | None           | Weekly          | Daily           | Real-time        |
| External party access           | None           | None            | Merchants (Post-Prod only) | None |

### 6.6 Environment-Level Control Variations

Within a class, individual environments may have targeted control variations:

| Environment | Variation from Class Baseline                                          |
|-------------|------------------------------------------------------------------------|
| Dev         | Broader IAM permissions for developers; feature branch deployments allowed |
| QA          | Tighter IAM; only CI/CD deploys; test data fixtures loaded automatically |
| Pre-Prod    | Internal access only; no merchant connectivity                          |
| Post-Prod   | Merchant-facing endpoints exposed; merchant credentials managed via dedicated IAM |
| Prod        | Full Production class controls; no variations                           |
| Shadow      | Production class data controls; lighter change approval (migration team autonomy); ephemeral lifecycle with auto-decommission |

---

## 7. Data Classification Scheme

### 7.1 Classification Levels

| Level          | Definition                                                                 | Examples                                       |
|----------------|---------------------------------------------------------------------------|------------------------------------------------|
| **Restricted** | Data whose unauthorized disclosure would cause severe harm; subject to specific regulatory requirements (PCI-DSS, HIPAA) | Cardholder data (PAN, CVV), encryption keys, authentication secrets, SSNs |
| **Confidential** | Data whose unauthorized disclosure would cause significant business harm; limited to need-to-know access | Customer PII, financial reports, source code, API keys, infrastructure configurations, audit logs |
| **Internal**   | Data intended for internal use only; not sensitive but not for public release | Internal documentation, architecture diagrams, internal communications, non-production configurations |
| **Public**     | Data explicitly approved for public release                                | Marketing materials, public APIs, open-source code, published documentation |

### 7.2 Classification-to-Boundary Mapping

Data classification determines the minimum boundary controls required:

| Classification | Minimum Realm     | Minimum Env Class | Minimum Zone     | Encryption at Rest         | Encryption in Transit | Access Control              | Audit                       |
|----------------|-------------------|--------------------|-------------------|-----------------------------|----------------------|-----------------------------|-----------------------------|
| Restricted     | Product (CDE)     | Production         | Data zone         | KMS CMK, annual rotation    | TLS 1.2+ mutual      | JIT + MFA + role separation | Real-time + 7yr retention   |
| Confidential   | Product or Mgmt   | Validation+        | Data or App zone  | KMS CMK                     | TLS 1.2+             | RBAC + MFA                  | Daily + 1yr retention       |
| Internal       | Any               | Any                | Any               | SSE-S3 or KMS               | TLS 1.2+             | RBAC                        | Weekly + 90d retention      |
| Public         | Any               | Any                | Edge or App zone  | Optional                    | TLS recommended      | Open or API key             | Minimal                     |

### 7.3 Data Residency Rules

| Rule                                                         | Enforcement Mechanism            |
|--------------------------------------------------------------|----------------------------------|
| Restricted data must not exist outside the CDE sub-realm     | SCP (deny S3/RDS in non-CDE accounts), DLP scanning |
| Confidential data must not exist in Sandbox environments     | SCP (deny data service creation in sandbox OUs)       |
| Development class environments must use synthetic or masked data | Pipeline gate (data masking step before Dev/QA load) |
| Validation class may use production-structure data but must mask PII | Automated masking pipeline with validation       |
| Production data must not be copied to lower environment classes | SCP (deny cross-account copy to Dev/Validation OUs) |
| Shadow environments inherit Production class data controls   | Shadow accounts provisioned under Production Class OU |

---

## 8. AWS Construct Mapping

### 8.1 Organizational Unit Hierarchy

The OU structure maps directly to the realm, sub-realm, environment class, and environment taxonomy:

```
Root
├── Product OU
│   ├── CDE OU
│   │   ├── CDE-Validation OU
│   │   │   └── [CDE Pre-Prod Accounts]
│   │   └── CDE-Production OU
│   │       └── [CDE Prod Accounts]
│   │
│   └── General-Product OU
│       ├── Product-Development OU          ← Development Class
│       │   ├── [Dev Accounts]
│       │   └── [QA Accounts]
│       ├── Product-Validation OU           ← Validation Class
│       │   ├── [Pre-Prod Accounts]
│       │   └── [Post-Prod Accounts]
│       └── Product-Production OU           ← Production Class
│           ├── [Prod Accounts]
│           └── [Shadow Accounts — ephemeral]
│
├── Management OU
│   ├── Platform-Services OU
│   │   ├── Platform-NonProd OU
│   │   │   └── [Platform Non-Prod Accounts]
│   │   └── Platform-Production OU
│   │       └── [Platform Production Accounts]
│   └── Security-Services OU
│       ├── Security-NonProd OU
│       │   └── [Security Non-Prod Accounts]
│       └── Security-Production OU
│           └── [Security Production Accounts]
│
├── Corporate OU
│   ├── Sandbox OU
│   │   └── [Sandbox Accounts — ephemeral, no TGW attachment]
│   └── Corporate-Internal OU
│       └── [Internal Tool Accounts, Jump Hosts, VPN]
│
├── Connectivity OU (Shared)
│   └── [Network Hub Account — TGW, DNS, Direct Connect]
│
└── Suspended OU
    └── [Decommissioned accounts]
```

**Notes:**

- **Environment Class maps to OU tier.** Product-Development, Product-Validation, and Product-Production OUs correspond directly to the three environment classes. Each OU tier gets its own SCP set and TGW route table association, providing the governance grouping that environment classes are designed to deliver.

- The **Connectivity OU** is realm-neutral. It hosts the Transit Gateway owner account, DNS hub, and Direct Connect gateway. This account has no workloads — it is purely a network transit plane. This is a natural candidate for a fourth realm when organizational maturity supports it.

- The **Suspended OU** captures decommissioned accounts with a deny-all SCP, preventing any resource creation or API calls while preserving the account for audit trail purposes.

- **Shadow accounts** are provisioned under the Product-Production OU (not a separate OU) because they require Production class controls. They are tagged `Lifecycle: ephemeral` and `Purpose: migration-backtest` for cost tracking and automated decommission.

### 8.2 SCP Inheritance Model

SCPs are applied at the OU level and inherited downward. The design uses a **layered SCP model**:

| SCP Layer          | Applied At             | Purpose                                          | Example Policies                              |
|--------------------|------------------------|--------------------------------------------------|-----------------------------------------------|
| Baseline           | Root                   | Universal guardrails across all accounts         | Deny region outside approved set, deny leaving org, require IMDSv2 |
| Realm              | Realm OU               | Realm-specific restrictions                      | Product: deny VPN gateway creation; Mgmt: deny internet-facing resources |
| Sub-Realm          | Sub-Realm OU           | Sub-realm hardening                              | CDE: deny non-approved services, deny unencrypted storage creation |
| Environment Class  | Environment Class OU   | Class-level governance                           | Development: permit broader service set; Production: deny manual resource creation |
| Environment        | Account-level tags/IAM | Environment-specific fine-tuning                 | Shadow: permit specific migration tooling; Post-Prod: permit merchant IAM federation |

**SCP Tightening (never relaxing):**

```
Root SCP (Baseline)
  → restricts regions, org exit, IMDSv2
    └── Product OU SCP
        → adds: deny VPC peering outside realm
          └── General-Product OU SCP
              → adds: deny direct internet-facing resources (except via edge zone)
                └── Product-Development OU SCP (Development Class)
                │   → adds: deny production data services (RDS Multi-AZ, etc.)
                └── Product-Validation OU SCP (Validation Class)
                │   → adds: deny manual resource creation
                └── Product-Production OU SCP (Production Class)
                    → adds: deny console actions, deny manual deployments, deny resource creation outside IaC
          └── CDE OU SCP
              → adds: deny all services except approved list
                └── CDE-Production OU SCP
                    → adds: deny console actions, deny manual deployments
```

Each layer can only add restrictions. A child OU can never permit an action that a parent OU has denied.

### 8.3 Network Topology

The network design uses Transit Gateway with **route domain isolation** to enforce realm and environment class boundaries:

**Figure 3: Transit Gateway Route Domain Isolation**

![TGW Route Domain Isolation](diagrams/03-tgw-route-isolation.svg)

```
                         ┌──────────────────────────────────────┐
                         │        Transit Gateway (TGW)         │
                         │                                      │
                         │  Route Tables:                       │
                         │  ┌────────────────────────────────┐  │
                         │  │ Product-Dev-RT                 │  │
                         │  │ Product-Val-RT                 │  │
                         │  │ Product-Prod-RT                │  │
                         │  │ CDE-RT                         │  │
                         │  │ Management-RT                  │  │
                         │  │ Corporate-RT                   │  │
                         │  │ Shared-Services-RT             │  │
                         │  └────────────────────────────────┘  │
                         └──────────────────────────────────────┘

  Each environment class within the Product realm gets its own TGW
  route table. This is the primary blast radius control that
  prevents a compromised Dev account from reaching Production.
```

**Route Domain Design:**

| Route Table       | Can Route To                                           | Cannot Route To                              |
|-------------------|--------------------------------------------------------|----------------------------------------------|
| Product-Dev-RT    | Management-RT (for CI/CD, artifact pull), Shared-Services-RT (DNS, logging) | Product-Val-RT, Product-Prod-RT, CDE-RT, Corporate-RT |
| Product-Val-RT    | Management-RT (for CD pipeline), Shared-Services-RT    | Product-Dev-RT, Product-Prod-RT, CDE-RT, Corporate-RT |
| Product-Prod-RT   | Shared-Services-RT (DNS, logging)                      | Product-Dev-RT, Product-Val-RT, Corporate-RT |
| CDE-RT            | Shared-Services-RT (logging only)                      | Product-Dev-RT, Product-Val-RT, Product-Prod-RT (except tokenization endpoint), Corporate-RT |
| Management-RT     | Product-Dev-RT, Product-Val-RT, Product-Prod-RT (for deployments), Corporate-RT (for developer access), Shared-Services-RT | CDE-RT (no direct management access to CDE) |
| Corporate-RT      | Management-RT (for CI/CD dashboards, admin tools)      | Product-Dev-RT, Product-Val-RT, Product-Prod-RT, CDE-RT |
| Shared-Services-RT| All route tables (DNS, logging, NTP)                   | N/A                                          |

**Critical Blast Radius Controls:**

- **Product-Dev-RT has no route to Product-Prod-RT.** A compromised Dev account cannot reach Production at the network level, even though they are in the same realm. The environment class boundary, enforced via separate TGW route tables, provides equivalent or stronger isolation than the previous cross-realm (Corporate → Product) boundary.
- **Product-Val-RT has no route to Product-Prod-RT.** Validation environments cannot directly communicate with Production environments. Artifact promotion between classes flows through the Management realm's pipeline, not via direct network connectivity.
- **Corporate-RT has no route to any Product route table.** Developer workstations in the Corporate realm cannot reach any Product environment — Dev, QA, Pre-Prod, Post-Prod, Prod, or Shadow — directly.

**CDE Network Hardening:**

The CDE route table is the most restrictive:
- No default route (no internet egress, even via NAT)
- VPC endpoints for all AWS service access (S3, KMS, CloudWatch, STS)
- Dedicated VPC endpoints (not shared with General Product)
- Network ACLs as a secondary control layer (defense in depth, even though SGs are primary)
- VPC flow logs enabled and shipped to Security Services sub-realm

### 8.4 Account-to-Construct Quick Reference

| Concept               | AWS Construct(s)                                                   |
|-----------------------|--------------------------------------------------------------------|
| Realm                 | Top-level OU + SCP set                                              |
| Sub-Realm             | Nested OU + dedicated TGW route table (if network isolation needed) |
| Zone (Edge)           | Public subnets + ALB/NLB + WAF + Shield                            |
| Zone (Application)    | Private subnets + EKS/ECS/Lambda + Security Groups                  |
| Zone (Data)           | Isolated subnets + RDS/ElastiCache/S3 + KMS + SG                   |
| Zone (Integration)    | Private subnets + API Gateway + EventBridge + SG                    |
| Zone (Management)     | Private subnets + SSM endpoints + CloudWatch agent                  |
| Environment Class     | OU tier + SCP set + TGW route table                                 |
| Environment           | Account(s) within Environment Class OU + IAM policies               |
| Realm boundary        | OU boundary + TGW route table isolation + SCP deny cross-realm AssumeRole |
| Zone boundary         | Subnet tier + Security Group rules + NACLs (CDE only)              |
| Env Class boundary    | OU tier boundary + TGW route table isolation + SCP set              |
| Environment boundary  | Account boundary + IAM permission boundary                         |

---

## 9. Security Control Inheritance

Controls cascade from realm → zone → environment class → environment. Each layer inherits the parent's controls and may add (but never remove) restrictions.

**Figure 5: Security Control Inheritance Cascade**

![Security Control Inheritance](diagrams/05-control-inheritance.svg)

### 9.1 Control Matrix

| Control Category                | Realm Level                          | Zone Level                          | Environment Class Level             | Environment Level                |
|---------------------------------|--------------------------------------|-------------------------------------|-------------------------------------|----------------------------------|
| **Network Isolation**           | TGW route domain, SCP deny cross-realm | SG rules, subnet NACLs              | TGW route table per class          | Account-level SG defaults         |
| **Identity & Access**           | IAM Identity Center permission sets per realm | IAM roles scoped to zone resources  | Class-wide access model (dev: broad, prod: JIT) | Env-specific IAM (e.g., merchant federation for Post-Prod) |
| **Encryption**                  | Realm-wide KMS key policy            | Per-zone CMK (CDE) or shared CMK    | Key rotation schedule per class    | Per-env key aliases               |
| **Logging & Audit**             | CloudTrail org trail to Security Services | VPC flow logs, access logs          | Retention period per class         | Per-env log stream tagging        |
| **Compliance Scanning**         | AWS Config conformance pack per realm | Prisma Cloud policy group per zone  | Scan frequency per class           | Per-env exception handling        |
| **Deployment Controls**         | SCP deny manual changes (Prod class) | Pipeline-only for App/Data zones    | Change approval gates per class    | Per-env promotion gate criteria   |
| **Data Protection**             | Classification labeling required     | DLP scanning on zone egress (CDE)   | Data posture per class (synthetic/masked/real) | Per-env data fixture management |
| **Incident Response**           | Realm-level playbook                 | Zone-specific containment actions   | Class-specific blast radius procedures | Env-specific runbooks            |

### 9.2 PCI-DSS v4.0 Control Mapping (CDE Sub-Realm)

The following maps selected PCI-DSS v4.0 requirements to their implementation within the CDE sub-realm:

| PCI-DSS v4.0 Req | Requirement Summary                    | Implementation                                             |
|-------------------|----------------------------------------|-----------------------------------------------------------|
| 1.2               | Network security controls              | Dedicated TGW route table, VPC endpoints only, no internet egress, NACLs |
| 1.4               | Network connections between trusted and untrusted | WAF + ALB in Edge zone, no direct internet to App/Data zones |
| 2.2               | Secure system configurations           | Hardened AMIs, CIS benchmarks, AWS Config rules             |
| 3.5               | Protect stored account data            | KMS CMK encryption, no S3 public access, tokenization      |
| 4.2               | Protect CHD during transmission        | TLS 1.2+ enforced, mutual TLS for CDE internal traffic     |
| 5.2               | Malicious software prevention          | GuardDuty runtime monitoring, container image scanning      |
| 6.3               | Security vulnerabilities identified and addressed | Snyk + Prisma container scan in promotion pipeline         |
| 7.2               | Access to system components restricted | IAM least privilege, JIT PAM, no standing production access |
| 8.3               | Strong authentication                  | MFA required, session duration limits, no shared accounts   |
| 10.2              | Audit logs capture                     | CloudTrail + VPC flow logs + access logs, 7-year retention  |
| 10.4              | Audit logs are reviewed                | Security Hub automated findings + Splunk SIEM correlation   |
| 11.3              | Vulnerability scans and penetration tests | Quarterly scans (Prisma), annual pen test, automated weekly scans |
| 12.10             | Incident response plan                 | CDE-specific IR playbook, forensic isolation account, evidence preservation |

### 9.3 NIST 800-53 Rev 5 Family Mapping

| NIST Family       | Implementation Across All Realms                                |
|-------------------|-----------------------------------------------------------------|
| AC (Access Control) | IAM Identity Center, JIT PAM, SCP guardrails, least privilege |
| AU (Audit)         | CloudTrail org trail, VPC flow logs, centralized log archive   |
| CA (Assessment)    | AWS Config conformance packs, Prisma Cloud, Prowler            |
| CM (Configuration) | IaC-only deployment, drift detection, AWS Config rules         |
| CP (Contingency)   | Multi-AZ by default, cross-region backup for CDE               |
| IA (Identification)| Okta SSO, SCIM to IAM Identity Center, MFA everywhere          |
| IR (Incident Response) | Realm-specific playbooks, forensic isolation accounts       |
| RA (Risk Assessment) | Quarterly risk reviews, automated vulnerability scanning     |
| SC (System/Comms)  | TLS 1.2+, VPC endpoints, TGW route isolation, KMS encryption  |
| SI (System Integrity) | GuardDuty, Security Hub, FIM (CDE), container scanning     |

---

## 10. Migration Considerations

### 10.1 From Three Realms to Five

The three-realm model is designed with explicit expansion points. When organizational maturity supports it, the following sub-realms can graduate to full realms:

| Current Sub-Realm            | Target Realm          | Trigger for Graduation                                      |
|------------------------------|-----------------------|-------------------------------------------------------------|
| Security Services (in Mgmt)  | Security Realm        | Audit finding requiring full independence of security tooling from platform services |
| Connectivity OU (shared)     | Connectivity Realm    | Multi-cloud or complex hybrid networking requiring dedicated governance |
| CDE (in Product)             | Regulated Realm       | Multiple regulatory frameworks (PCI + HIPAA) requiring distinct compliance domains |

Graduation involves: creating a new top-level OU, migrating accounts, establishing a new TGW route table, and updating SCP inheritance. Because the account boundaries and network paths are already designed for separation, this is an organizational and policy change, not an infrastructure re-architecture.

### 10.2 From Current State to Target State (Dev/QA Migration)

Moving Dev/QA from the Corporate realm to the Product realm requires the following steps:

| Phase | Action                                                                         | Risk Mitigation                                |
|-------|--------------------------------------------------------------------------------|------------------------------------------------|
| 1     | Provision new Dev/QA accounts under Product-Development OU via AFT             | Parallel run — don't decommission old accounts until validated |
| 2     | Create Product-Dev-RT in TGW with no routes to Product-Prod-RT or CDE-RT      | Verify isolation before migrating any workloads |
| 3     | Establish developer access path: Corporate → Management → Product Dev/QA via JIT PAM | Test with pilot team before org-wide rollout |
| 4     | Migrate CI/CD pipelines to target new Product Dev/QA accounts                  | Canary deployment — migrate one service first   |
| 5     | Update SCP inheritance for Product-Development OU                              | Dry-run SCP changes with AWS Access Analyzer    |
| 6     | Validate with QSA that Dev/QA in Product realm does not expand PCI scope       | Document OU isolation, TGW route table separation, data posture |
| 7     | Decommission old Corporate Dev/QA accounts → Suspended OU                     | Retain for 90 days before final suspension       |

### 10.3 Coexistence with Current On-Premises

The realm taxonomy maps to on-premises equivalents through clear boundary points:

| Cloud Realm       | On-Prem Equivalent          | Interconnect                                        |
|--------------------|-----------------------------|-----------------------------------------------------|
| Product            | Production DC zone          | Direct Connect (dedicated VIF per realm)            |
| Management         | Management / NOC zone       | Direct Connect (shared services VIF)                |
| Corporate          | Corporate LAN / VPN zone    | Direct Connect or VPN (user connectivity)           |

The key principle is: **on-premises connectivity does not bypass cloud realm boundaries.** A Direct Connect circuit from the on-prem production zone terminates in the Product realm's connectivity account and is subject to the same TGW route table restrictions as any other traffic source. On-premises networks are treated as external to the cloud trust model.

---

## 11. Open Questions and Future Work

| ID   | Question                                                                                           | Owner                 | Target Date |
|------|----------------------------------------------------------------------------------------------------|-----------------------|-------------|
| OQ-1 | Should the Connectivity OU be elevated to a full realm from day one given Cloud WAN migration plans? | Platform Engineering  | TBD         |
| OQ-2 | What is the minimum viable set of AWS services to allowlist in the CDE SCP?                        | Security Architecture | TBD         |
| OQ-3 | Should Validation class environments use a separate artifact registry from Production class, or the same one with promotion tags? | Platform Engineering | TBD |
| OQ-4 | How do we handle data masking pipeline failures — block promotion or allow with manual approval?    | Data Engineering      | TBD         |
| OQ-5 | Confirm with QSA that Dev/QA accounts within the Product realm (under a separate OU with no CDE network path) do not expand PCI scope | Security Architecture | TBD |
| OQ-6 | What is the role-chaining depth limit for cross-realm IAM role assumptions (Management → Product)?  | Security Architecture | TBD         |
| OQ-7 | Should the CDE sub-realm use a separate Transit Gateway entirely (physical isolation) or TGW route table isolation (logical)? | Network Engineering | TBD |
| OQ-8 | Define the merchant access model for Post-Prod — dedicated IAM federation vs. API key vs. mutual TLS | Product Engineering + Security Architecture | TBD |
| OQ-9 | Define Shadow environment provisioning automation — AFT dormant account pattern vs. on-demand vending | Platform Engineering | TBD |
| OQ-10 | Should Dev and QA be separate accounts per product/service, or shared accounts with namespace isolation (e.g., EKS namespaces)? | Platform Engineering | TBD |

---

## Appendix A: Glossary

| Term                | Definition                                                                                   |
|---------------------|----------------------------------------------------------------------------------------------|
| Realm               | Highest-level trust domain; defines organizational, network, and security boundaries          |
| Zone                | Functional network segment within an environment (e.g., edge, application, data)              |
| Environment Class   | Governance grouping of related environments that share a common control profile (e.g., Development, Validation, Production); maps to OU tier + SCP set + TGW route table |
| Environment         | Individual lifecycle stage within an environment class (e.g., Dev, QA, Pre-Prod, Post-Prod, Prod, Shadow) |
| Sub-Realm           | A segment within a realm with harder boundaries, designed for future elevation to full realm  |
| CDE                 | Cardholder Data Environment — PCI-DSS scope boundary for systems handling payment card data   |
| SCP                 | Service Control Policy — AWS Organizations policy that sets permission guardrails at OU level |
| TGW                 | Transit Gateway — AWS network hub for inter-VPC and inter-account routing                    |
| Route Domain        | A TGW route table that defines an isolated routing context                                   |
| Environment Class OU| The OU tier that implements an environment class (e.g., Product-Development OU)               |
| JIT PAM             | Just-In-Time Privileged Access Management — temporary, audited elevation of access            |
| Promotion           | The process of moving a build artifact from a lower to higher environment                     |
| Promotion Gate      | A set of automated and/or manual checks that must pass before an artifact can be promoted     |
| Blast Radius        | The maximum scope of impact from a security incident or operational failure                   |
| QSA                 | Qualified Security Assessor — the auditor who certifies PCI-DSS compliance                   |
| Shadow Environment  | An ephemeral, production-class environment used for back-testing during big bang migrations   |
| Post-Prod           | A merchant-facing certification environment where external partners validate their integrations against production-grade code |

## Appendix B: References

| # | Reference                                                                                          |
|---|---------------------------------------------------------------------------------------------------|
| 1 | AWS Security Reference Architecture (AWS SRA), v3 — [AWS SRA Documentation](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/welcome.html) |
| 2 | AWS Organizations — Service Control Policies Best Practices — [AWS Organizations Docs](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html) |
| 3 | PCI-DSS v4.0 — Payment Card Industry Data Security Standard — [PCI SSC](https://www.pcisecuritystandards.org/document_library/) |
| 4 | NIST SP 800-53 Rev 5 — Security and Privacy Controls — [NIST](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final) |
| 5 | AWS Transit Gateway — Network Segmentation Patterns — [AWS TGW Docs](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html) |
| 6 | AWS Control Tower Landing Zone — [AWS Control Tower Docs](https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html) |
| 7 | NIST SP 800-125B — Secure Virtual Network Configuration for VM Protection — [NIST](https://csrc.nist.gov/publications/detail/sp/800-125b/final) |
| 8 | CIS AWS Foundations Benchmark — [CIS](https://www.cisecurity.org/benchmark/amazon_web_services) |
| 9 | AWS Account Factory for Terraform (AFT) — [AWS AFT Docs](https://docs.aws.amazon.com/controltower/latest/userguide/aft-overview.html) |
| 10 | AWS Well-Architected Framework — Security Pillar — [AWS Well-Architected](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html) |