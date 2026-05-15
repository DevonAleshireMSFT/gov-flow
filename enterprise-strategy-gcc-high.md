---
layout: default
title: Enterprise Strategy
nav_order: 5
permalink: /enterprise-strategy/
---

# Enterprise Power Platform Environment Strategy and Governance
## A Recommendation for Large-Scale DoD Organizations — GCC High / IL5

**Audience:** Senior Leadership · Enterprise Architects · Platform Administrators · Security & Compliance Teams · Program Managers · Governance Boards

**Classification Applicability:** CUI / IL5 workloads in GCC High tenants. Does not cover SIPRNet or TS/SCI. Consult your AO for classified enclave requirements.

**Well-Architected Alignment:** This document maps recommendations to the five [Power Platform Well-Architected](https://learn.microsoft.com/en-us/power-platform/well-architected/) pillars — **Security**, **Reliability**, **Operational Excellence**, **Performance Efficiency**, and **Experience Optimization** — with emphasis on the constraints that constrained-cloud environments impose on each pillar.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Recommended Environment Strategy](#2-recommended-environment-strategy)
3. [Security & Compliance Architecture](#3-security--compliance-architecture)
4. [Governance Model](#4-governance-model)
5. [ALM / DevSecOps Strategy](#5-alm--devsecops-strategy)
6. [Operational Support Model](#6-operational-support-model)
7. [Leadership Reporting & Metrics](#7-leadership-reporting--metrics)
8. [Recommended Enterprise Standards](#8-recommended-enterprise-standards)
9. [Risks & Anti-Patterns](#9-risks--anti-patterns)
10. [Final Recommended Strategy](#10-final-recommended-strategy)

---

## 1. Executive Summary

### 1.1 Recommended Strategy

A large-scale DoD Power Platform deployment cannot be governed by the same model used for a 50-person commercial enterprise. The combination of strict ATO requirements, personnel churn, classification boundaries, and mission-criticality of some workloads demands a **federated governance model**: centralized platform ownership with decentralized, governed development.

The recommended approach is:

> **One GCC High tenant per branch/agency. One Platform Center of Excellence (CoE) team owning the platform layer. Managed Environments enforced organization-wide. Application development federated to program and command teams within a defined governance framework. LP-ALM used as the standard ALM methodology for all pro-developer workloads.**

This is not a recommendation to lock down everything. It is a recommendation to build the guardrails that allow hundreds of teams to build safely and independently without creating organizational debt that becomes unmanageable at year three.

### 1.2 Why This Approach Works for DoD-Scale Organizations

Commercial Power Platform guidance routinely underestimates two realities of large government organizations:

1. **Personnel turnover destroys undocumented systems.** Applications owned by a single person — with no source control, no documentation, no service accounts — become unrecoverable when that person PCS's or retires. At this scale, this is not a hypothetical; it is the norm.

2. **ATO requirements are non-negotiable but vary by program.** A logistics tracking app and a commander's dashboard require different ATOs, different data handling requirements, and potentially different environment isolation. A one-size governance model fails both.

The federated model addresses both: documentation and ALM discipline are enforced by the platform; isolation boundaries and ATO scope are program-controlled.

### 1.3 Primary Goals and Expected Outcomes

| Goal | Outcome |
|---|---|
| ATO-supportable architecture | Every production environment has documented access control, DLP policy, audit logging, and solution inventory |
| Separation of duties | No individual has development, approval, and deployment rights simultaneously |
| Sustainable ownership | Every application has a documented owner team, service account, and ALM pipeline — not a person |
| Scalable governance | Governance processes do not require the CoE team to be in the critical path for every deployment |
| Cost visibility | Platform leadership can report licensing costs, environment count, and utilization per command |

### 1.4 Well-Architected Pillar Summary

| Pillar | Primary Focus in This Document |
|---|---|
| **Security** | RBAC, BU hierarchy, DLP, IL5 constraints, identity, separation of duties |
| **Reliability** | Environment isolation, managed solutions, rollback strategy, DR |
| **Operational Excellence** | ALM pipelines, DevSecOps, observability, CoE tooling |
| **Performance Efficiency** | Environment sizing, Dataverse capacity, scaling strategy |
| **Experience Optimization** | Governance that enables rather than blocks; fusion team model |

---

## 2. Recommended Environment Strategy

> **Well-Architected:** Security (segmentation), Operational Excellence (deployment confidence), Reliability (isolation)

### 2.1 Tenant Segmentation

This framework assumes a **single government tenant** as the foundation. One tenant per branch or agency is the recommended model — running multiple tenants within a single organization creates federation complexity that eliminates most of the platform-level governance benefits. While the patterns here are written with GCC-High and DoD in mind, they apply equally to any government or federal Power Platform deployment.

Cross-command isolation within a single tenant is achieved through:
- Managed Environments with environment-level access control
- Business Unit segmentation within Dataverse environments
- DLP policies scoped to environment groups
- Azure AD / Entra group ownership per command

> **IL5 consideration:** Not all workloads in a government tenant run at IL5. IL5 authorization requires a DISA provisional authorization and specific Dataverse environment configuration. Separate IL5-designated environments from general-purpose environments within the same tenant. Mixing IL5 and non-IL5 workloads in the same environment is not authorized.

> **Default environment:** The default environment is not suitable for enterprise production applications. See [Default Environment Governance](../default-environment/) for risks, limitations, and the recommended approach for org-wide application access.

### 2.2 Eight-Tier Environment Model

The following topology is the recommended baseline for an organization with multiple commands and multiple development teams:

```
┌─────────────────────────────────────────────────────────────────┐
│                  GOVERNMENT TENANT (GCC HIGH / DoD)             │
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────────────────────────┐  │
│  │  PLATFORM TIER  │  │         SHARED SERVICES TIER        │  │
│  │                 │  │                                     │  │
│  │  CoE-Admin      │  │  SharedSvc-Prod                     │  │
│  │  CoE-Dev        │  │  SharedSvc-Dev                      │  │
│  └─────────────────┘  └─────────────────────────────────────┘  │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              PROGRAM / COMMAND TIERS (per program)      │    │
│  │                                                         │    │
│  │  {CMD}-{PGM}-Sandbox  (self-service, 30-day lifecycle)  │    │
│  │  {CMD}-{PGM}-Dev      (team dev + integration)          │    │
│  │  {CMD}-{PGM}-Test     (SIT / UAT)                       │    │
│  │  {CMD}-{PGM}-Prod     (production)                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              INDIVIDUAL DEVELOPER TIER (optional)       │    │
│  │  {USER}-Dev   (personal dev, provisioned on demand)     │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

| Tier | Count | Owner | Lifecycle | IL5 Eligible |
|---|---|---|---|---|
| CoE Admin | 1 | Platform CoE | Permanent | No |
| CoE Dev | 1 | Platform CoE | Permanent | No |
| Shared Services Dev | 1–2 | Platform CoE | Permanent | Conditional |
| Shared Services Prod | 1 | Platform CoE | Permanent | Conditional |
| Program Sandbox | 1 per program | Program team | 30–90 days | No |
| Program Dev | 1 per program | Program team | Permanent while ATO active | Yes |
| Program Test | 1 per program | Program team | Permanent while ATO active | Yes |
| Program Prod | 1 per program | Program PM + CoE | Permanent while ATO active | Yes |
| Individual Dev | On demand | Individual developer | Provisioned/deprovisioned per project | No |

### 2.3 Centralized vs. Decentralized Environments

**Centralize:** Platform governance tools (CoE Starter Kit), shared component libraries, enterprise connector configurations, and cross-cutting security templates. These belong in the CoE or Shared Services tiers owned by the Platform CoE team.

**Decentralize:** Application development, testing, and production. Each program owns its environment lifecycle, subject to the governance standards enforced at the tenant level.

**Never centralize production.** A single "enterprise production" environment for multiple programs creates blast radius risk, ATO scope contamination, and change management bottlenecks. Reject any proposal to consolidate production environments across programs.

### 2.4 How Many Production Environments

One production environment per program/application portfolio that has a distinct ATO boundary. Programs that share an ATO may share a production environment. Programs with separate ATOs must have separate production environments — ATO scope leakage is a compliance failure, not a preference.

**The exception: Platform CoE-owned enterprise environments.** Some applications legitimately serve the entire organization or span multiple commands. These do not belong in a program-owned environment. The Platform CoE maintains a dedicated enterprise production environment (`ENTERPRISE-PROD`) for these workloads, with its own Platform-level ATO boundary. This is not a workaround — it is a deliberate tier in the environment model. See [Default Environment Governance — Enterprise-wide access strategy](../default-environment/#enterprise-wide-application-access-strategy) for provisioning and access guidance.

At large DoD organization scale, expect 50–200+ production environments over a 3–5 year maturity period. This is expected and manageable with Managed Environments and proper governance tooling.

> **Capacity note:** Each Dataverse environment carries a 1 GB base storage requirement. Environment sprawl has a direct licensing cost — factor this into your provisioning approval process.

### 2.5 Individual Developer Environments

Individual developer environments are personal, single-occupant environments provisioned to a named developer for isolated experimentation, proof-of-concept work, or active development on a specific workload before a program-level Dev environment exists. They sit below the program tier — not a team resource, not tied to an ATO, not used for integration testing.

#### When a developer needs one

An individual developer environment is appropriate when:

- **No program Dev environment exists yet.** A developer has been tasked with building a new solution but the program environment request is still in the approval pipeline. Rather than blocking work, the developer provisions a personal environment to begin canvas app scaffolding, data model design, or flow prototyping.
- **Isolated spike or proof-of-concept work.** The developer is evaluating a connector, a new Dataverse feature, or an architectural pattern that should not contaminate the shared team Dev environment with experimental tables or flows.
- **Assigned to multiple programs.** A pro developer supporting two different programs simultaneously cannot cleanly separate work in a single program Dev environment. A personal environment per active engagement reduces cross-contamination risk.
- **Component library or reusable code development.** A developer building a shared canvas component or a reusable cloud flow pattern benefits from a clean isolated environment before contributing to the shared layer.

#### When a developer does not need one

Individual developer environments are **not** appropriate as a substitute for:

- **Integration testing.** Integration testing belongs in the program's `{CMD}-{PGM}-TEST` environment. A personal dev environment has no service principals, no pipeline configuration, and no representative data model for integration work.
- **User acceptance testing.** Any testing involving end users must be in a CoE-approved Test or UAT environment. Sharing a personal dev environment with stakeholders is a governance violation.
- **Permanent development work.** If a solution is in active team development, the program Dev environment is the right home. Personal environments that become de facto team Dev environments are an unmanaged sprawl risk.
- **Production data of any classification.** Individual developer environments receive the Developer DLP tier — which restricts production data connectors. Placing CUI or higher-classification data in a personal environment is a compliance violation regardless of DLP policy.

#### Naming convention

Individual developer environments follow a distinct pattern to distinguish them from program environments in policy targeting and the environment register:

```
{USERALIAS}-Dev
```

| Example | Developer |
|---|---|
| `jsmith-Dev` | James Smith |
| `mlopez-Dev` | Maria Lopez |
| `rwilliams-Dev` | Robert Williams |

**Rules:**
- Use the developer's network/email alias (characters before `@`), lowercased.
- Single suffix: `-Dev` only. No `{USERALIAS}-Test`, `{USERALIAS}-Prod` — those tiers do not exist at the individual level.
- Maximum 40 characters (Power Platform display name limit). Truncate alias if needed.
- Individual dev environments are **not targeted by program DLP policies** — they receive the tenant-wide Developer DLP policy tier (see [DLP Strategy — Developer tier](../dlp-strategy/)).

#### How many per developer

| Rule | Limit | Rationale |
|---|---|---|
| **Default limit** | **1 per developer** | One personal dev environment is sufficient for the vast majority of individual development work. Multiple simultaneous personal environments are a sprawl indicator. |
| **Exception: active multi-program engagement** | Up to **2** | A developer actively supporting two distinct programs with no overlap may request a second individual dev environment. This requires CoE approval with documentation of both program assignments. |
| **Hard maximum** | **2 per developer** | No individual developer should hold more than 2 active personal dev environments at any time. If a developer needs more, that signals missing program-level environments — fix the root cause. |

The CoE should track individual developer environment count in the Environment Register. Environment sprawl at the individual tier is a common early-stage governance failure in DoD tenants — developers provision environments, finish the work, and never decommission them.

#### Lifecycle

Individual developer environments are **project-scoped**, not indefinite.

| Stage | Action |
|---|---|
| **Provision** | Developer submits request via CoE intake app. Approval is lightweight — CoE validates naming, license availability, and that no duplicate personal environment exists. Target: same-day provisioning. |
| **Active use** | Environment exists for the duration of the development spike or until the program Dev environment is provisioned and the developer has migrated work. |
| **Expiry** | Individual dev environments carry a **90-day lifecycle** from provisioning date. At day 75, the developer receives an automated renewal-or-decommission notification. |
| **Renewal** | Developer may request a single 90-day extension with a brief justification. Second renewal requires CoE review. |
| **Decommission** | Developer confirms any work-in-progress has been exported to source control or migrated. CoE deletes the environment within 5 business days of confirmation or expiry — whichever comes first. |

**There is no grace period for expired individual developer environments that have not responded to renewal notice.** Unresponsive environments are deleted at expiry. This is not punitive — it is the baseline hygiene that prevents 200+ orphaned environments at scale.

#### Relationship to the Sandbox tier

Individual developer environments and sandbox environments serve different purposes and should not be used interchangeably:

| | Individual Dev (`{USERALIAS}-Dev`) | Program Sandbox (`{CMD}-{PGM}-SANDBOX`) |
|---|---|---|
| **Owner** | Named developer | Program team |
| **Access** | Developer only | Program team members |
| **Purpose** | Personal isolated development | Team-level exploration, POC with stakeholders |
| **Managed Environment** | No | No |
| **DLP tier** | Developer (personal tier) | Sandbox (team tier) |
| **Lifecycle** | 90 days (renewable once) | 30–90 days (no renewal) |
| **Count limit** | 1–2 per developer | 1 per program per active initiative |

When a developer needs to demo prototype work to stakeholders or a program team, that work should move into the program Sandbox — not be shared out of a personal dev environment.

### 2.6 Naming Standards

Good names matter more than perfect names. Consistent naming enables DLP policy targeting, pipeline environment resolution, Managed Environment grouping, and audit attribution. A cryptic or inconsistent name breaks automation and slows incident response. At the same time, a naming scheme that becomes a 45-character string no one can type from memory will simply be ignored. Pick a convention, document it, enforce it at the environment tier, and keep everything else as readable guidance.

> **Don't overcomplicate it.** Environment names are the most critical — they are targeted by policy and pipelines. Application and flow names are user-facing — readability matters more than codes at that level. Component naming is internal to an app — consistency within the app is the only real requirement.

### Environment Naming

Environment names are the one place to be strict. They are matched by DLP policies, CoE connectors, and ALM pipelines. An inconsistent environment name means a DLP rule misses it.

**Recommended pattern:**

```
{COMMAND}-{PROGRAM}-{TYPE}
```

**Examples:**
```
FORSCOM-SYSTRK-DEV
FORSCOM-SYSTRK-TEST
FORSCOM-SYSTRK-PROD
NETC-TRNMGR-DEV
MARFORCOM-LOGTRACK-PROD
PLATFORM-COE-ADMIN
PLATFORM-SHAREDSVC-PROD
```

**Rules:**
- All caps. No spaces. Hyphens as delimiters.
- Maximum 40 characters (Power Platform display name limit).
- `{COMMAND}` = major command abbreviation (FORSCOM, TRADOC, NETC, MARFORCOM)
- `{PROGRAM}` = 4–8 character program code; matches Azure DevOps project name
- `{TYPE}` = DEV, TEST, PROD, SANDBOX, ADMIN — spell it out, no abbreviations

### Solution Naming

Solutions map to deployment units, not layers. One solution per deployable chunk of a program.

**Recommended pattern:** `{PGM}-{Descriptor}`

| Example | Purpose |
|---|---|
| `SYSTRK-Core` | Tables, columns, relationships |
| `SYSTRK-Config` | Environment variables, connection references |
| `SYSTRK-Flows` | All cloud flows for the program |

Keep solution names under 30 characters. Avoid embedding environment type in the solution name — that belongs in the environment, not the solution.

### Application Naming

Users see application names in their launcher. Prioritize readability over abbreviation codes.

**Recommended pattern:** `{PGM} {Friendly Name}` (spaces, title case)

Examples: `SYSTRK Ticket Tracker`, `SYSTRK Admin Portal`, `NETC Training Manager`

The program prefix keeps apps sortable and attributable without sacrificing usability.

### Flow Naming

Flows have no namespace in Power Automate. The program prefix is the only thing that groups and attributes them.

**Recommended pattern:** `{PGM} - {Trigger} - {Action}`

Examples: `SYSTRK - New Ticket - Notify Approver`, `SYSTRK - Daily - Archive Closed Items`, `SYSTRK - Record Change - Sync SharePoint`

Avoid vague names like "Approval Flow" or "Send Email" — they become impossible to manage at scale.

### Component Naming (Controls, Variables, Collections)

This is internal to the app. The only rule is: be consistent within the app. A recommended prefix convention:

| Prefix | Type | Example |
|---|---|---|
| `btn` | Button | `btnSubmit`, `btnCancel` |
| `gal` | Gallery | `galTickets`, `galUsers` |
| `lbl` | Label | `lblStatus`, `lblTitle` |
| `txt` | Text input | `txtSearch`, `txtComment` |
| `var` | Variable | `varCurrentUser`, `varSelectedRecord` |
| `col` | Collection | `colItems`, `colFilteredTickets` |

Use camelCase after the prefix. This is a recommendation — adapt to your team's standard if one already exists. The goal is that the next developer can read your app without a map.

### 2.7 Environment Lifecycle Management

**Sandbox environments** are the highest-risk category for sprawl. Enforce:
- Maximum 90-day lifecycle (hard-delete via Power Platform admin API on expiry)
- No production data of any classification
- No ATO coverage — explicitly excluded from ATO boundary documentation
- Provisioned through a self-service form (Power Apps + Approval flow in CoE-Admin) requiring manager approval

**Dev and Test environments** are permanent for the life of the program's ATO. When a program completes:
1. Export final solution artifacts to source control
2. Document environment inventory in the program's decommission record
3. Delete environments within 30 days of program end
4. Revoke service principal credentials within 24 hours

**Production environment deletion** requires ISSO sign-off, data disposal documentation per NIST 800-88, and CoE review board approval.

### 2.8 Environment Request and Approval Process

```
Developer/PM submits request (Power Apps form in CoE-Admin)
        ↓
Automated checks: naming compliance, ATO documentation attached,
                  data classification declared, owner designated
        ↓
CoE Architecture Review (< 5 business days for standard; 
                           < 2 days for sandbox)
        ↓
ISSO review for IL5-designated environments (additional 5 days)
        ↓
Automated provisioning via Power Platform Admin API / PAC CLI
        ↓
Service principal created and registered in environment register
        ↓
Owner team notified with onboarding checklist
```

Do not build this process in email. The request, approval, provisioning, and notification should all be automated through a CoE intake application. Manual email-based processes do not scale past 20 programs.

### 2.9 Managed Environments

**Reality check: Managed Environments require licensing.** A Managed Environment requires at least one qualifying Power Platform premium license in the environment (Power Apps Premium, Power Automate Premium, Power Apps Per App, Dynamics 365, or Copilot Studio). At DoD scale — where many users hold M365 E3/E5 licenses without Power Platform Premium — assuming universal Managed Environment coverage is unrealistic. Every governance plan must account for a mixed Managed and Unmanaged environment estate.

**The target is Managed Environments everywhere. The operational reality is a hybrid model.** Plan for it deliberately rather than discovering unmanaged environments during an ATO review.

#### What Managed Environments Add

| Capability | Managed | Unmanaged |
|---|---|---|
| Environment group DLP targeting | Yes | No — explicit named policy only |
| Canvas app sharing limits (restrict to security groups) | Yes | No |
| Solution checker enforcement on import | Yes | No |
| IP firewall (IL5 network boundary enforcement) | Yes | No |
| Weekly environment activity digest (CoE) | Yes | No |
| Maker welcome / onboarding content | Yes | No |
| Power Platform Pipelines | Yes | No |
| **DLP policy enforcement** | **Yes** | **Yes — DLP applies to all environments regardless of Managed status** |
| Security group assignment | Yes | Yes |

The bolded row is the governance foundation for unmanaged environments: **DLP enforcement is not contingent on Managed Environment licensing.** Every environment — Managed or not — is subject to the tenant's DLP policies. This is the primary technical control available across the entire estate.

#### Prioritization: Which Environments Must Be Managed

In a resource-constrained DoD organization, prioritize Managed Environment activation in this order:

| Priority | Environment Types | Rationale |
|---|---|---|
| **Must** | All Production (`{CMD}-{PGM}-PROD`) | ATO boundary; sharing limits prevent accidental data exposure; IP firewall for IL5 |
| **Must** | CoE-Admin, CoE-Dev | Platform governance tools require solution checker enforcement and sharing limits |
| **Must** | ENTERPRISE-PROD, SharedSvc-Prod | Cross-organizational impact; group DLP targeting required; sharing limits mandatory |
| **Should** | All Test (`{CMD}-{PGM}-TEST`) | Solution checker validates managed deployment path; pipeline enforcement |
| **Should** | Program Dev (`{CMD}-{PGM}-DEV`) | Solution checker catches issues before Test; developer sharing limits |
| **Accept Unmanaged** | Program Sandbox (`{CMD}-{PGM}-SANDBOX`) | Short 30–90 day lifecycle; DLP + security group sufficient; no ATO coverage |
| **Accept Unmanaged** | Individual Dev (`{USERALIAS}-Dev`) | Personal scope; no ATO coverage; DLP + security group sufficient |

When a **Should** environment cannot be Managed due to licensing constraints, document the unmanaged status in the environment register and apply compensating controls:
- Assign the environment security group to restrict access
- Apply an explicit named DLP policy targeting that environment by name (see §3.4)
- Enroll in the CoE Starter Kit inventory for activity monitoring
- Add to the quarterly manual governance review cadence (§7.3)

#### Governing Unmanaged Environments

An unmanaged environment is not an ungoverned environment. The absence of Managed Environment features requires explicit compensating controls.

| Missing Managed Feature | Compensating Control |
|---|---|
| Sharing limits | DLP blocks data exfiltration paths; CoE Starter Kit periodic sharing audit |
| Solution checker on import | PAC CLI solution checker in the ALM pipeline pre-export; developer discipline enforced by code review |
| IP firewall | Conditional Access policy (device compliance, CONUS location); agency network perimeter controls |
| Group-based DLP targeting | Named DLP policy explicitly referencing the environment; must be updated as environments are provisioned |
| Weekly activity digest | CoE Starter Kit environment activity reports (covers all environments, Managed and Unmanaged) |

**Never allow an unmanaged environment to exist outside the Environment Register.** Every environment must have a record with its classification, owner, DLP policy assignment, Managed status, and review date. An unmanaged environment that falls off the register is indistinguishable from a shadow environment.

---

## 3. Security & Compliance Architecture

> **Well-Architected:** Security (all five principles), Reliability (availability protection)

### 3.1 RBAC Strategy

DoD Power Platform RBAC has three layers that must be designed together:

```
Layer 1: Entra ID / Azure AD Groups
         ↓ controls access to environments and Managed Environments features
Layer 2: Dataverse Security Roles
         ↓ controls what a user can see and do within an environment
Layer 3: Field Security Profiles
         ↓ controls access to specific sensitive columns within a table
```

Each layer is independently necessary. Skipping Layer 3 for CUI fields is a compliance gap — column-level sensitivity requires field security profiles, not just table-level security roles.

**The RBAC pyramid for a program environment:**

```
         ┌──────────────────┐
         │  System Admin    │  → CoE service principal only
         ├──────────────────┤
         │  Program Admin   │  → Program PM + designated ISSO
         ├──────────────────┤
         │  Contributor     │  → Developers (Dev only)
         ├──────────────────┤
         │  Support         │  → L2 support team (Test + Prod)
         ├──────────────────┤
         │  Read Only       │  → Auditors, reporting users
         ├──────────────────┤
         │  End User        │  → Mission personnel (Prod)
         └──────────────────┘
```

**Non-negotiable RBAC rules for DoD:**
- System Administrator role is assigned to the **pipeline service principal only** — not to any human user in Test or Prod
- No human user holds System Administrator in production
- The `Contributor` role (with write privileges) is **only available in Dev** — Test and Prod users are read-only or end-user role only
- Role assignments are through **Entra ID security groups** — never individual user assignment in Test or Prod

### 3.2 Business Unit Strategy

Business Unit (BU) design is the most commonly misarchitected element in large government Power Platform deployments. Two failure modes dominate:

**Failure Mode 1 — Flat BU (everything in root):** Eliminates BU-level security isolation. All Dataverse records are accessible to root BU members. At DoD scale, this means a FORSCOM user can query TRADOC records if they hold the same security role. Unacceptable for CUI.

**Failure Mode 2 — Over-granular BU (one BU per team):** Creates a hierarchy that no one understands, breaks Append/Append To privileges systematically, and requires security role duplication at every level. This collapses under its own complexity within 18 months.

**Recommended BU hierarchy (3 levels maximum):**

```
Root BU (tenant)
  ├── Command BU  (FORSCOM, TRADOC, NETC, MARFORCOM)
  │     ├── Program BU  (SYSTRK, TRNMGR, LOGTRACK)
```

Rules:
- Dataverse Owner Teams are always provisioned at the **Program BU level** — not root, not command
- Owner teams use "Direct User (Basic) access level and Team privileges" — not Business Unit level
- Cross-BU data sharing is done through explicit team membership, not BU elevation
- Audit the BU hierarchy annually — unused BUs accumulate and create administrative debt

### 3.3 Dataverse Security Model

Dataverse security is table-level by default, column-level by exception.

**Standard table access pattern:**

| Access Level | Record Ownership | Use Case |
|---|---|---|
| User | Individual | Personal drafts, individual workload items |
| Team | Owner Team | Shared program records (most CUI data) |
| Business Unit | BU | Command-level reference data |
| Organization | All users in environment | Lookup tables, reference data (non-CUI) |

**For IL5 environments:** Default to Team-level record ownership. Organization-level access for any table containing CUI requires ISSO approval and documentation in the ATO.

**Append and Append To** privileges must be explicitly set for every relationship a security role traverses. This is the single most common source of "it works in Dev but breaks in Test" failures. Every security role must be tested against a relationship traversal matrix before promotion to production. Use the security role matrix template in [docs/security-role-matrix-template.md](/security-roles/).

### 3.4 DLP Policy Architecture

DLP policies are the primary technical control governing which connectors are available in each environment and whether data can flow between trusted and untrusted services. In an unmanaged tenant, a licensed user can move data to any connected service — including personal storage, consumer APIs, and unauthenticated inbound endpoints — without visibility or audit. DLP policies close that gap.

**GovFlow DLP operates on a default-deny model:** every connector not explicitly approved is blocked. New connectors added to the Power Platform catalog are blocked by default until reviewed by the Platform CoE. "Not yet reviewed" means blocked, not allowed.

**DLP enforcement is independent of Managed Environment status.** This is the foundational architectural property that makes DLP the primary governance control across the entire estate. Whether an environment is Managed or not, all applicable DLP policies are enforced. Managed Environments add environment group targeting and additional governance features on top — they do not change whether DLP is enforced.

#### DLP Policy Layers

| Layer | Mechanism | Applies To |
|---|---|---|
| **Tenant Default** | Single policy, most restrictive; Microsoft 365 core in Business group only; all else blocked | Every environment not covered by a more specific policy — the automatic catch-all |
| **Environment Group Policy** | Policy scoped to a Managed Environment group | All environments in that group; automatically applies when an environment joins the group. **Requires Managed Environments.** |
| **Named Environment Policy** | Policy explicitly listing one or more environments by name | Exact environments named only; must be maintained manually as environments are created or decommissioned |
| **Exception Policy** | Named policy; time-limited 90-day cycle; ISSO + CoE approved | Specific environment; specific connector only; tracked in the Exception Register |

Managed Environments use environment groups for DLP targeting — add the environment to a group and it automatically inherits the group's policy. **Unmanaged environments cannot join environment groups.** They receive either the Tenant Default or an explicit Named Environment Policy. This means unmanaged environment DLP requires a disciplined provisioning process: every new unmanaged environment must be added to its named policy at creation time (see §2.8 intake workflow).

#### DLP Tier by Environment Type

| Environment Type | DLP Tier | Targeting Method | Key Rules |
|---|---|---|---|
| **Individual Dev** (`{USERALIAS}-Dev`) | Developer | Named policy (list by name at provisioning) | All approved first-party Microsoft connectors; no HTTP outbound to external endpoints; no third-party; no custom unless explicitly approved |
| **Program Sandbox** (`{CMD}-{PGM}-SANDBOX`) | Sandbox | Named policy, or Tenant Default acceptable for short lifecycle | First-party Microsoft + program-approved Azure services; HTTP blocked; no third-party |
| **Program Dev** (`{CMD}-{PGM}-DEV`) | Development | Environment group (Managed) or named policy | First-party Microsoft + all program-approved integration connectors; HTTP to approved Azure Government endpoints only |
| **Program Test** (`{CMD}-{PGM}-TEST`) | **Must match Production exactly** | Environment group (Managed) or named policy | Same connector set as the program's Production environment — any deviation invalidates test coverage |
| **Program Production** (`{CMD}-{PGM}-PROD`) | Production | Environment group (Managed — required) | ATO-documented connectors only; ISSO-reviewed connector list; no additions without ISSO review and change control |
| **Shared Services** (`SharedSvc-PROD/DEV`) | Enterprise | Environment group (Managed — required) | Cross-organizational connectors approved at Platform-ATO level; validated against GCC High data residency |
| **ENTERPRISE-PROD** | Enterprise | Environment group (Managed — required) | Match or exceed Production; Platform-ATO boundary validated; broadest org exposure justifies strictest controls |
| **CoE-Admin / CoE-Dev** | Platform | Environment group (Managed — required) | Power Platform Admin connector, HTTP to Azure Government; isolated from program DLP tiers; CoE staff only |
| **Default (catch-all)** | Tenant Default | Automatic — no configuration required | Microsoft 365 core connectors only; everything else blocked; applies to any environment not explicitly covered |

{: .important }
> **Test must mirror Production DLP.** A connector blocked in Production but allowed in Test produces false-positive test results. DLP policy misalignment between Test and Production is a deployment risk that surfaces at go-live, not in testing. Manage Test and Production in the same environment group where possible, or through synchronized named policies where Managed Environments are unavailable.

#### Base Policy + Program Overlay Model

The recommended structure is a layered composition: a **base tier policy** applied to each environment group, with **program-specific exception policies** applied on top for connectors needed by one program that are not in the standard tier.

```
Tenant Default  (most restrictive — automatic catch-all for unlisted environments)
    │
    ├── Developer Group Policy     →  Individual Dev environments
    ├── Sandbox Group Policy       →  Program Sandbox environments (if Managed)
    ├── Development Group Policy   →  Program Dev environments
    ├── Test/Prod Group Policy     →  Program Test + Prod (identical policy, same group)
    ├── Enterprise Group Policy    →  Shared Services + ENTERPRISE-PROD
    └── Platform Group Policy      →  CoE-Admin + CoE-Dev
                │
                └── Program Exception Policy (named, 90-day, ISSO approved)
                    → One specific connector approved for one program's specific environment
```

**Program exceptions are the pressure valve.** When a program needs a connector outside the standard tier for their environment type, they submit a connector approval request (see [DLP Strategy — Connector Request Process](../dlp-strategy/)) and receive a named exception policy targeting their specific environment. Exceptions are time-limited, ISSO-reviewed, and tracked in the Exception Register.

{: .warning }
> **IL5 validation required:** Not all Microsoft 365 connectors route data through the GCC High or DoD boundary. Validate each connector's data residency documentation against your ATO boundary before approving at any tier.

For the full DLP governance model — connector approval workflow, custom connector requirements, HTTP trigger policy, premium connector governance, and connector catalog management — see the dedicated [DLP Strategy](../dlp-strategy/) page.

### 3.5 Identity and Access Management

**Entra ID Groups — Two-Category Model**

Power Platform Entra ID groups fall into two distinct categories with different purposes, different naming patterns, and different membership models. Mixing them produces an unmaintainable group structure at scale.

- **Category 1 — Environment Access Groups:** Control which users can enter an environment. These are assigned in the Power Platform admin center and enforced by Managed Environment sharing restrictions.
- **Category 2 — Dataverse Security Role Groups:** Back Dataverse Owner Teams inside a specific environment. These determine what a user can see and do within Dataverse. They are configured inside the environment, not at the admin center level.

A user requires membership in both a Category 1 group (to access the environment) and a Category 2 group (to hold appropriate Dataverse permissions). A user in Category 1 only has no Dataverse role and cannot open any app. A user in Category 2 only cannot enter the environment.

For step-by-step implementation — how to assign groups to environments, configure group-backed Dataverse teams, and apply the access matrix across Dev/Test/Prod tiers — see [Environment Access & RBAC](../environment-access/).

#### Category 1: Environment Access Groups

Every Power Platform environment has a single **environment security group** configured in the Power Platform admin center (Environment Settings → Security group). This is the gate — only members of this group can be added as users to the environment at all. It carries no Dataverse role. It is purely an access control boundary.

All role-based groups (Owners, Devs, Support) must have their members also present in this gate group — either through direct membership or by nesting the role groups inside it.

| Group Type | Pattern | Example | Purpose |
|---|---|---|---|
| **Environment Gate** | `PP-{PGM}-{TYPE}` | `PP-SYSTRK-PROD` | **Assigned in the admin center as the environment's security group. No role. Controls who can be added to the environment at all.** |
| Environment Owners | `PP-{PGM}-{TYPE}-Owners` | `PP-SYSTRK-PROD-Owners` | Environment admin; Program Admin Dataverse role; CoE + Program PM only |
| Environment Users | `PP-{PGM}-{TYPE}-Users` | `PP-SYSTRK-PROD-Users` | Standard environment users; Managed Environment sharing target |
| Developers | `PP-{PGM}-DEV-Devs` | `PP-SYSTRK-DEV-Devs` | Dev environment only; Contributor Dataverse role; never assigned to Test or Prod |
| Support | `PP-{PGM}-{TYPE}-Support` | `PP-SYSTRK-PROD-Support` | L2 support access to Test and Prod; read-only Dataverse role |

**Recommended structure:** Nest the role groups inside the gate group so that membership is maintained in one place. Adding a user to `PP-SYSTRK-PROD-Users` automatically satisfies the gate. Removing them from all role groups automatically removes gate access.

```
PP-SYSTRK-PROD  (gate — assigned to environment in admin center)
  ├── PP-SYSTRK-PROD-Owners   (nested)
  ├── PP-SYSTRK-PROD-Users    (nested)
  └── PP-SYSTRK-PROD-Support  (nested)

PP-SYSTRK-DEV  (gate — assigned to Dev environment)
  ├── PP-SYSTRK-DEV-Owners    (nested)
  ├── PP-SYSTRK-DEV-Users     (nested)
  └── PP-SYSTRK-DEV-Devs      (nested)
```

> **`{PGM}` = program code (SYSTRK, TRNMGR). `{TYPE}` = DEV, TEST, PROD, SANDBOX.** The gate group has no role suffix — it is the environment identifier only. The `-Owners` group must stay small — Program PM, one technical lead, CoE service account. Maximum 5–7 members.

#### Category 2: Dataverse Security Role Groups

These groups back Dataverse Owner Teams. The team's membership source is the Entra ID group — changes in Entra propagate to Dataverse automatically (subject to sync latency; allow up to 15 minutes).

| Group Type | Pattern | Example | Purpose |
|---|---|---|---|
| App End Users | `PP-{PGM}-{TYPE}-{AppCode}-Users` | `PP-SYSTRK-PROD-APP1-Users` | Standard end-user security role for a specific app |
| App Admins | `PP-{PGM}-{TYPE}-{AppCode}-Admins` | `PP-SYSTRK-PROD-APP1-Admins` | Admin security role for a specific app (dual app pattern §4.8) |
| ISSO Reviewer | `PP-{PGM}-{TYPE}-ISSO` | `PP-SYSTRK-PROD-ISSO` | ISSO read + review fields access; no write on operational records |
| Read Only | `PP-{PGM}-{TYPE}-ReadOnly` | `PP-SYSTRK-PROD-ReadOnly` | Auditors, reporting consumers, executive dashboard access |

For programs with a single app, omit `{AppCode}`. For programs with multiple apps in one environment, each app requires its own user and admin groups — this enforces the dual application pattern and prevents accidental cross-app access through a single overpermissioned team.

#### Platform / Tenant-Level Groups

Maintained by the Platform CoE. These apply across multiple environments or at the tenant level. Program teams have no membership in these groups — CoE access to program environments is through these groups, not through program-specific groups.

| Group Type | Pattern | Purpose |
|---|---|---|
| CoE Administrators | `PP-Platform-CoE-Admins` | Full tenant admin; Power Platform tenant administrator role; maximum 3–5 people |
| CoE Engineers | `PP-Platform-CoE-Engineers` | Environment admin access across all environments; standard Dataverse access for support |
| Tenant ISSO Reviewers | `PP-Platform-ISSO` | Read access to all production environments for ATO evidence and audit |
| Tenant Read-Only | `PP-Platform-ReadOnly` | Executive dashboard and reporting consumers; no write access anywhere |
| DLP Administrators | `PP-Platform-DLP-Admins` | Power Platform DLP policy management; scoped to Platform CoE only |

#### Membership Model: Dynamic vs. Manual

| Group | Type | Auto-membership Rule / Rationale |
|---|---|---|
| `{PGM}-{TYPE}` (gate) | **Derived from nested groups** | Do not manage membership directly. Nest the role groups inside it — membership is inherited. The gate reflects who holds a role, not a separate list to maintain. |
| `{PGM}-PROD-Users` | **Dynamic (preferred)** | Drive from HR system: `user.department -eq "{COMMAND}"` + job title or extension attribute matching the program. Self-maintaining through PCS/transfers. |
| `{PGM}-DEV-Users` | **Manual** | Dev access is deliberate. Not all program users need Dev environment access. Onboarding action by Platform Champion. |
| `{PGM}-{TYPE}-Owners` | **Manual** | Always small, named individuals. Automatic assignment creates privilege escalation risk. |
| `{PGM}-DEV-Devs` | **Manual** | Named developers only. Program Platform Champion adds on assignment, removes on departure. |
| `{PGM}-{TYPE}-{AppCode}-Users` | **Dynamic (preferred)** | HR attribute or Entra extension attribute set at onboarding: `user.extensionAttribute1 -eq "{PGM}-USER"`. |
| `{PGM}-{TYPE}-{AppCode}-Admins` | **Manual** | Named individuals accountable for data management. Never automatic. |
| `{PGM}-{TYPE}-ISSO` | **Manual** | Organizational designation. ISSO assignments are not derived from HR data. |
| `PP-Platform-*` | **Manual** | Platform team is small. Every member is a named individual accountable for tenant-level actions. |

**Automatic access guidance by persona:**

- **New developers assigned to a program** — Platform Champion manually adds to `{PGM}-DEV-Devs` and `{PGM}-DEV-Users` during onboarding. No automatic assignment at the Dev tier.
- **Mission personnel in a command** — receive Prod `-Users` access via dynamic membership if HR attributes are reliable. If HR data quality is poor (a common DoD reality), use manual with quarterly access review over unreliable dynamic rules.
- **Departing personnel** — dynamic groups shed members automatically on HR attribute change. Manual groups require explicit offboarding action. Document group membership removal in the offboarding checklist for every program.
- **CoE engineers** — never automatically added to program groups. CoE accesses program environments through `PP-Platform-CoE-Engineers`, which is granted a support-level role across all environments via a separate access policy.
- **ISSOs** — manually added to `PP-Platform-ISSO` for tenant-wide read access and to `{PGM}-{TYPE}-ISSO` for program-specific security review access. Access is not inherited — it is assigned per ISSO per program scope.

#### Organizational Recommendations

1. **Prefix all Power Platform groups with `PP-`.** In a large DoD tenant with thousands of Entra groups from Active Directory sync, Teams, SharePoint, and other services, a consistent prefix makes Power Platform groups immediately identifiable and filterable.

2. **One group, one role, one environment.** Never create a catch-all `{PGM}-All` group and assign it multiple roles or share it across environments. Flat membership produces unauditable access and accumulates stale members through personnel transitions.

3. **Register every group in the Environment Register.** Each Entra group associated with an environment is a documented record attribute (see §2.5). The environment decommission workflow must include deletion of all associated Entra groups — not just the Power Platform environment.

4. **Review `-Owners` and `-Admins` group membership quarterly.** These are the highest-privilege groups in any program. They are the most likely to accumulate stale membership through PCS cycles. The quarterly review is a CoE responsibility, not the program's.

5. **Maximum two levels of group nesting; document any nested relationship.** Group nesting is valid for large organizations but creates audit opacity — the effective membership of a Dataverse team may not be obvious if three levels of nesting are in play. Document every nested group relationship in the Environment Register.

6. **Never assign environment access to individual users directly.** All access is via group membership. Direct user assignment to environments bypasses the group-level audit trail and creates offboarding gaps that will be missed.

**Never use mail-enabled distribution lists for Power Platform access.** Distribution lists are not security principals — they cannot be evaluated by Conditional Access or enforced by Managed Environment sharing restrictions. Any existing DL-based access must be migrated to security groups before Managed Environments are activated.

**Service Principals (Pipeline Automation):**
- One service principal per environment tier (Dev, Test, Prod)
- System Administrator role granted to SP only
- Secret rotation on a 90-day schedule, automated where possible
- SP credentials stored in Azure Key Vault (Azure Government) — not in ADO variable groups as plaintext
- SP access is audited monthly

**Service Accounts vs. Personal Accounts:**
In DoD environments, agency IAM policy sometimes prohibits provisioning non-human service accounts. This is a real constraint. See [LP-ALM.md Section 3.5](/methodology/#35-gcc-high-and-fedramp-specific-configurations) for the decision table on how to handle this per environment tier.

### 3.6 Conditional Access

GCC High Entra ID supports Conditional Access. Required policies for IL5:

- **MFA required** for all Power Platform access — no exceptions
- **Compliant device required** for production environment access (STIG-compliant device baseline)
- **Location-based restrictions** — CONUS-only access policy for production environments; exceptions require ISSO-approved named location
- **Sign-in risk policy** — block high-risk sign-ins to production environments
- **Session controls** — app-enforced restrictions for Power Apps access (prevent download/print of CUI)

Conditional Access policies are maintained by the command IAM team, not the Power Platform CoE. The CoE provides requirements; IAM implements. This separation of duties is intentional.

### 3.7 Audit Logging and Monitoring

Power Platform audit events flow to Microsoft Purview (available in GCC High). Required for ATO:

- **Unified Audit Log** enabled for the tenant — captures Power Apps opens, flow runs, Dataverse record modifications
- **Dataverse activity logging** enabled per environment — table-level audit trails for CUI tables
- **Log retention** minimum 1 year hot, 6 years cold (per NARA schedule 2 for federal records)
- **SIEM integration** — route audit logs to the command SIEM (typically Splunk or Microsoft Sentinel on Azure Government) via Event Hub in Azure Government

> **GCC High limitation:** Log export requires Azure Government (not commercial Azure) Event Hub and Storage Account. Ensure all log routing stays within the GCC High / Azure Government boundary.

### 3.8 Common Security Mistakes at DoD Scale

1. **Granting System Administrator to human users "just in case"** — this nullifies every other security control. System Admin sees everything. In production, no human holds System Admin.

2. **Building a 40-role security model** — complex security models are not reviewed during ATOs; they are rubber-stamped. A complex model that nobody understands is less secure than a simple model that is actively maintained. Five roles per application maximum.

3. **Ignoring Append and Append To** — relationships between tables require both privileges. The default "create a role and assign basic read/write" misses relationship traversal. Test every role against every relationship in the data model.

4. **Using one environment for Dev and Test** — Dev and Test in one environment means managed solutions can't be tested, deployment pipelines can't be validated, and the ATO boundary is ambiguous. Separate them.

5. **Flat BU with no owner teams** — results in every record being organization-owned. All users with the security role see all records. For CUI data, this is a compliance failure.

---

## 4. Governance Model

> **Well-Architected:** Operational Excellence (development standards, fusion culture), Experience Optimization (enabling adoption)

### 4.1 Platform Governance Board

The Platform Governance Board (PGB) is the decision authority for the Power Platform tenant. It is not a gating committee for every deployment — it is the body that sets policy and resolves exceptions.

**Composition:**
- **Chair:** Enterprise Architect or CTO/CIO designee
- **Platform CoE Lead:** Technical authority for platform standards
- **ISSO Representative:** Security and compliance sign-off
- **Command Representatives:** 1 per major command (rotating)
- **Program Manager Representative:** Program community voice

**Cadence:** Monthly standing meeting (30 minutes). Exception reviews are async via the intake system — the PGB meeting addresses only policy changes and escalations.

**Decision authority:**
| Decision | Authority |
|---|---|
| New production environment | PGB approval |
| New IL5 environment | PGB + ISSO approval |
| DLP policy exception | PGB + ISSO approval |
| New connector approval | Platform CoE Lead |
| Architecture exception | Platform CoE Lead + ISSO for security exceptions |
| Program onboarding | Platform CoE Lead |
| Sandbox provisioning | Automated (self-service) |

### 4.2 Application Classification Model

Every application must be classified before production deployment. Classification determines the ATO path, environment tier requirements, and support model.

| Class | Description | ATO Requirement | ALM Requirement |
|---|---|---|---|
| **Mission Critical** | Failure directly impacts mission execution or life safety | Full ATO, IL5 mandatory review, annual pen test | LP-ALM mandatory, full pipeline, load tested |
| **Business Critical** | Significant business impact if unavailable | ATO required, quarterly security review | LP-ALM mandatory, full pipeline |
| **Standard** | Normal business application | ATO required | LP-ALM recommended, pipeline required |
| **Citizen / Low-Code** | Departmental productivity, non-CUI | ATO not required; DLP coverage sufficient | Solution export to source control minimum |

Classification is **declared by the program PM**, reviewed by the ISSO, and recorded in the environment register. Reclassification upward (e.g., Citizen → Standard) requires environment reprovisioning — do not deploy Mission Critical applications into environments provisioned for Citizen apps.

### 4.3 Data Classification Approach

Map directly to DoD data classification:

| Power Platform Handling | DoD Classification |
|---|---|
| No field security, org-level access | Unclassified, non-CUI (public) |
| Team-level records, DLP tier 1 | Unclassified, non-sensitive |
| Field security profiles, DLP tier 2, IL4 | CUI (Controlled Unclassified Information) |
| Separate IL5 environment, restricted Conditional Access | CUI requiring IL5 handling |
| **Not supported in Power Platform** | Secret / TS / SAP |

Power Platform is not authorized for Secret or higher. Any application that might aggregate data to Secret classification through combination must be reviewed by the AO before deployment.

### 4.4 Intake and Review Process

The intake process must be fully automated. Manual email-based intake at DoD scale means requests fall through the cracks, governance records are incomplete, and ATO evidence collection fails.

**Automated intake application (built in CoE-Admin environment):**

```
1. Program team submits request (Power Apps form)
   Fields: program name, command, data classification, app class,
           estimated users, requested environments, ATO POC,
           owner name + DoD ID, ISSO name

2. Automated validation:
   - Naming standards check
   - Data classification / app class consistency check
   - ISSO lookup confirmation
   - Licensing capacity check

3. Routing:
   - Sandbox → auto-approve, auto-provision within 1 hour
   - Standard/Citizen Dev → CoE review (2 business days)
   - Business Critical or higher → CoE review + ISSO review (5 days)
   - IL5 → PGB approval required

4. Provisioning (automated on approval):
   - Environment created via Power Platform Admin API
   - Service principal created and registered
   - Entra ID groups created and documented
   - ADO project created if LP-ALM pipeline required
   - Owner team notified with onboarding checklist link
```

### 4.5 Fusion Team Model

DoD programs typically have a technology gap: trained software engineers are rare; Power Platform citizen developers are common; the gap between them is where problems occur. The fusion team model addresses this.

**Recommended fusion team composition per program:**

| Role | Responsibilities | Skills |
|---|---|---|
| **Platform Champion** | Liaison between program and Platform CoE; owns the environment register; runs intake requests | Business + platform admin |
| **Pro Developer** | Designs data model, writes plugins, owns pipeline, enforces LP-ALM | Full-stack + Power Platform |
| **App Developer** | Builds canvas apps, Power Automate flows with guidance from Pro Developer | Power Platform intermediate |
| **Citizen Developer** | Builds personal/team productivity apps in approved citizen environments | Power Platform beginner |
| **ISSO** | Reviews security role design, signs off on ATO artifacts | Security |

The pro developer is the quality gate for the fusion team. They own source control, review all solution exports before commit, and are accountable for the ALM pipeline. They are **not** the bottleneck — they set the standards that allow the team to move independently.

### 4.6 Citizen Development Governance

Citizen development is not ungoverned development. At DoD scale, citizen apps that escape governance become shadow IT that no one can support, audit, or decommission.

**Citizen governance controls:**
- Citizen developers work only in **approved citizen environments** with the most restrictive DLP tier
- All citizen apps that handle CUI must be reviewed by the program ISSO before going to production — no exceptions
- Citizen apps that reach 20+ users are automatically escalated for pro developer review (tracked via CoE Starter Kit telemetry)
- Annual citizen app review: all apps older than 12 months with no activity in 60 days are quarantined (not deleted — quarantined) pending owner confirmation
- Citizen developers should not use the default environment for any app intended for team or organizational use — see [Default Environment Governance](../default-environment/)

### 4.7 Avoiding Governance Bottlenecks

The CoE team is not a delivery team — it is a standards-setting, tooling, and exception-handling team. Governance bottlenecks occur when the CoE is in the critical path for every deployment. Prevent this by:

- **Self-service sandbox provisioning** — CoE is not involved
- **Pipeline automation** — deployments to Test and Prod are pipeline-gated, not CoE-reviewed individually
- **Pre-approved patterns** — the component placement decision tree and LP-ALM methodology are pre-approved architecture; teams that follow them do not need architecture review for individual solutions
- **Exception-only escalation** — the intake process routes only genuine exceptions to the CoE; standard requests auto-approve against checklist
- **Annual governance review, not per-release review** — mature programs review governance quarterly; new programs get monthly touchpoints for the first 6 months

### 4.8 Dual Application Pattern

For any program application that has both an end-user population and an administrative data management function, the recommended pattern is **two separate model-driven apps in the same environment** — a standard user-facing app and a dedicated admin app.

**Why two apps instead of two areas:**

| Reason | Explanation |
|---|---|
| **App-level security enforcement** | Security roles can be scoped per app — end users receive the standard app, administrators receive the admin app. The boundary is enforced at the app level, not through nav visibility alone. |
| **Read-only forms by default** | The standard app surfaces read-only forms for reference data (systems catalog, vendor records, configuration tables), preventing accidental edits even by privileged users. The UI enforces the intent — not just the security role. |
| **Cleaner audit trail** | All reference data changes flow exclusively through the admin app, making data management actions immediately distinguishable from end-user activity in audit logs. |
| **Simpler security role design** | The standard app's security role is genuinely minimal — read access on reference data, limited write on transactional tables only. Role design becomes straightforward when the access surface is constrained. |

**Recommended app sharing pattern:**

| App | Shared with | Entra group |
|---|---|---|
| `{AppName}` (standard) | All mission personnel | `PP-{ENV}-{APP}-Users` |
| `{AppName} Admin` | Program admins + platform team only | `PP-{ENV}-{APP}-Admins` |

{: .important }
> Both apps live in the same Dataverse environment and share the same data. The admin app is not a separate environment — it is a separate entry point with a different security role and elevated form permissions. Ensure the admin app is not accessible to end users through direct URL navigation by scoping the app's sharing to the `Admins` security group only.

**What belongs in each app:**

| Standard app | Admin app |
|---|---|
| Transactional forms (create/edit operational records) | Reference data management (systems catalog, vendors, configuration) |
| Read-only views of reference data | Bulk data operations and imports |
| User-facing dashboards and reports | Administrative dashboards, audit views |
| Self-service actions | User/team management, staging and promotion workflows |

This pattern applies regardless of application classification. A Mission Critical app and a Standard app both benefit from the admin/user separation — the difference is that a Mission Critical admin app requires its own ATO scope documentation.

---

## 5. ALM / DevSecOps Strategy

> **Well-Architected:** Operational Excellence (all five principles), Reliability (safe deployments, rollback), Security (secure development lifecycle)

### 5.1 LP-ALM as the Enterprise Standard

LP-ALM (Layered Power Platform ALM) is the recommended ALM methodology for all pro-developer workloads in the enterprise. It provides:

- A five-layer decomposition model that separates security, schema, configuration, automation, and UI into independently deployable solution artifacts
- A defined pipeline architecture for GCC High
- A source control structure compatible with Azure DevOps
- Explicit handling of the `_Config` layer (never committed, always manual)

All Mission Critical and Business Critical applications must adopt LP-ALM. Standard applications should adopt LP-ALM. Citizen applications are exempt but must export solutions to source control at minimum.

See the [LP-ALM Methodology](/methodology/) for the full reference.

### 5.2 Azure DevOps Integration

**GCC High requires self-hosted agents.** Microsoft-hosted agents are not available in the GCC High ADO cloud. Every organization operating in GCC High must maintain a pool of self-hosted build agents. For the full Azure DevOps + agent architecture, Key Vault integration, and monitoring strategy, see [Azure Integration Strategy](../azure-integration/).

**Self-hosted agent requirements:**
- Windows Server 2022 recommended (PAC CLI requires .NET)
- PAC CLI installed and pinned to a known-good version (update deliberately, not automatically)
- `--cloud UsGovHigh` flag must be present in all `pac auth create` commands
- Agent VMs hosted in Azure Government, not commercial Azure
- Agent pool segmented by classification: one pool for IL5 environments, one for standard
- Agent service account is not a personal account — dedicated managed identity or service account

**ADO project structure:**

```
ADO Organization
  └── {COMMAND}-Platform (CoE pipelines, shared templates)
  └── {COMMAND}-{PROGRAM} (one ADO project per program)
        ├── Repos:    {prefix}-powerplatform (Power Platform solutions)
        │             {prefix}-azure-infra (if Azure integration exists)
        ├── Pipelines: deploy-security, deploy-core, deploy-automation,
        │              deploy-ui, deploy-all, pr-validation
        ├── Variable Groups: {PREFIX}-Common, {PREFIX}-Test, {PREFIX}-Prod
        └── Environments: Test, Prod (with approval gates)
```

### 5.3 CI/CD Pipeline Architecture

```
PR → pr-validation.yml
     ├── Pack all layers (managed)
     ├── Solution checker (enforce Critical: 0, High: 0)
     ├── Schema contamination check (_UI must not contain Entities/)
     └── _Config guard (must not exist in source)

Merge to main → deploy-all.yml (to Test)
     ├── deploy-security
     ├── deploy-core
     ├── [CONFIG GATE — ManualValidation, 4-hour timeout]
     ├── deploy-automation
     └── deploy-ui

Release tag → deploy-all.yml (to Prod)
     ├── deploy-security
     ├── deploy-core
     ├── [CONFIG GATE — ManualValidation, 24-hour timeout]
     ├── [PROD APPROVAL — Reviewer: Program PM + ISSO]
     ├── deploy-automation
     └── deploy-ui
```

**Config Gate:** The manual validation gate exists because `_Config` (environment variables, connection references) cannot be automated. The gate forces a human to apply `_Config` before automation and UI deploy. This is not a process gap — it is an intentional security control. Connection reference credentials and endpoint URLs must never be in source control.

**Solution Checker enforcement:** In GCC High, the solution checker runs against the GCC High endpoint. Do not suppress Critical or High findings. Zero Critical, Zero High is the production gate. Medium findings require documented acceptance before merge.

### 5.4 Branching Strategy

```
main          → Production-ready code. Direct push prohibited.
               Only merges via PR with at least 1 reviewer.

release/vX.Y  → Release candidate branch. Created from main.
               Deployment to Prod triggers from this branch.

feature/{name} → Developer feature branches. Branched from main.
                PR to main required. At least 1 reviewer (pro developer).

hotfix/{issue} → Branched from the release tag in Prod. Emergency fix path.
                Merges to both main and current release branch.
```

**Hotfix process:** Hotfixes skip the standard Test cycle — they do not skip the Config Gate or Prod approval gate. A hotfix with a security vulnerability must be ISSO-reviewed before Prod deployment, regardless of urgency.

### 5.5 Managed vs. Unmanaged Solutions

| Environment | Solution Type | Rationale |
|---|---|---|
| Dev | Unmanaged | Developers must be able to modify components |
| Test | Managed | Validates that the managed deployment path works; prevents Dev-only assumptions |
| Prod | Managed | Prevents ad-hoc modifications; maintains solution history |

**Never import managed solutions to Dev.** Managed solutions in Dev create a layer that blocks local modifications. The most common symptom is "I can't edit this form" — caused by a managed solution imported over an unmanaged one.

**Never deploy unmanaged solutions to Test or Prod.** Unmanaged solutions in production are unauditable, un-rollbackable, and create merge conflicts that can only be resolved by manual component deletion. This is a hard pipeline rule in LP-ALM.

### 5.6 Environment Variables and Connection References

Environment variables are the correct mechanism for per-environment configuration. They are set in `_Config`, which is applied manually — never committed to source control.

| Variable Type | GCC High Consideration |
|---|---|
| Azure endpoint URL | Must be Azure Government URL (`*.usgovcloudapi.net`), not commercial |
| SharePoint site URL | Must be the GCC High SharePoint URL (`*.sharepoint.us`) |
| Exchange/Email endpoint | Must be GCC High Exchange Online endpoint |
| Custom API endpoint | Validate data residency — commercial SaaS APIs may not be authorized |

Connection references bind at the `_Config` layer to service accounts or, where service accounts are prohibited, to designated non-personal credentials per the program's IAM plan. See [LP-ALM.md Section 3.5](/methodology/#35-gcc-high-and-fedramp-specific-configurations).

### 5.7 Rollback Strategy

Power Platform does not have a native atomic rollback. Rollback is an import of the previous known-good managed solution version.

**Pre-deployment requirement:** Before any production deployment, the pipeline must export and archive the current managed solution from Prod. This artifact is the rollback target.

**Rollback procedure:**
1. Identify rollback target version from ADO pipeline artifacts
2. Import the prior managed solution versions in layer order (_Security → _Core → _Automation → _UI)
3. Verify `_Config` values — rollback does not affect environment variables; re-apply if version-specific values changed
4. Notify ISSO of rollback event for ATO audit trail
5. Post-incident review within 48 hours

**What rollback cannot fix:** If a `_Core` rollback removes a table that production data was written to, that data is gone. Data destruction risk must be assessed before any `_Core` rollback. For IL5 data, ISSO must be notified of any data loss event within 1 hour.

### 5.8 Automated Testing

Power Platform has limited native test tooling. The enterprise standard approach:

| Test Type | Tooling | Stage |
|---|---|---|
| Solution validity | PAC CLI solution checker | PR validation |
| Schema contamination | PowerShell script (LP-ALM pr-validation.yml) | PR validation |
| Unit tests (server-side logic) | Power Apps Test Studio (canvas), EasyRepro (model-driven) | PR validation |
| Integration tests | Flow run with test data, verified via assertion flow | Post-deploy to Test |
| UAT | Manual by program team in Test environment | Before Prod promotion |
| Security role test | Role validation script (verify privilege matrix) | Post-deploy to Test |

Automated test execution in GCC High requires the self-hosted agent pool. Test Studio tests can be run headlessly via the command line on the agent.

---

## 6. Operational Support Model

> **Well-Architected:** Reliability (recovery, availability), Operational Excellence (observability, emergency response)

### 6.1 Support Tier Model

| Tier | Owner | Scope | GCC High Consideration |
|---|---|---|---|
| **L0 — Self-service** | End user | Knowledge base, how-to guides, FAQ | Host in SharePoint GCC High — not commercial SharePoint |
| **L1 — Help Desk** | Command IT | Password reset, access requests, basic flow errors | Must have Power Platform tenant admin read access; not edit access |
| **L2 — Application Support** | Program fusion team | App bugs, flow failures, data issues | Requires access to program Test environment for reproduction |
| **L3 — Platform Support** | Platform CoE | Environment issues, DLP policy, pipeline failures, Dataverse infrastructure | Microsoft Premier/Unified support escalation path required |
| **L4 — Microsoft Support** | Microsoft | Product bugs, GCC High infrastructure issues | Unified Support contract required; GCC High tickets go to the federal support queue — response times are longer than commercial |

### 6.2 CoE Starter Kit

The [Power Platform CoE Starter Kit](https://learn.microsoft.com/en-us/power-platform/guidance/coe/starter-kit) is available in GCC High with limitations.

**Available in GCC High:**
- Environment inventory and management
- Maker activity dashboards
- App and flow inventory
- Capacity alerting
- Maker onboarding and welcome emails
- DLP impact analysis

**Not available or limited in GCC High:**
- AI Builder features (limited or unavailable depending on IL level)
- Some Copilot-dependent features — validate against the GCC High feature parity list before relying on them
- Telemetry features that require commercial Azure App Insights must be redirected to Azure Government Application Insights

**CoE Starter Kit deployment:** Deploy to the `PLATFORM-COE-ADMIN` environment. Do not mix CoE tooling with any program environment. Keep CoE data (environment inventory, flow run history) out of ATO scope for program ATOs.

### 6.3 Monitoring Stack

```
Power Platform Unified Audit Log
        ↓
Azure Event Hub (Azure Government)
        ↓
Microsoft Sentinel (Azure Government)  OR  Splunk Enterprise Security
        ↓
SIEM Alerts → SOC Ticket → L3/L4 Response
```

**Alert thresholds to configure:**
- Flow failure rate > 5% over 1 hour → L2 alert
- Dataverse storage > 80% capacity → L3 alert
- API call rate approaching connector limits → L2 alert
- Failed authentication > 10 attempts from single IP → SOC alert (potential brute force)
- New System Administrator assignment in any environment → immediate SOC alert

**Application Insights:** For canvas apps and flows, configure Azure Government Application Insights via the environment diagnostics settings. This provides real-time flow execution telemetry without going through commercial Azure.

### 6.4 Capacity Management

GCC High Power Platform licensing is the same per-user and per-app model as commercial, but the procurement process goes through DoD EA channels. Capacity is not elastic — it must be planned.

**Dataverse capacity alerts:**
- Database capacity: alert at 80%, escalate at 90%
- File capacity: alert at 75% (file storage fills faster than DB in document-heavy apps)
- Log capacity: alert at 80%; log retention configuration directly affects consumption

**API call limits:** Power Automate per-user plan entitlement limits matter at scale. 500,000+ runs/day from a program requires licensed capacity — not free entitlement from the M365 license. Plan this in year 1, not year 3 when you hit the wall.

**Environment capacity reviews:** Quarterly capacity review for all production environments. The CoE Starter Kit capacity dashboard is the primary tool.

### 6.5 Backup and Disaster Recovery

Power Platform (Dataverse) provides automatic daily backups with a 28-day retention window. This covers data loss recovery. It does not cover:

- **Solution loss** — covered by source control (LP-ALM)
- **Environment deletion** — covered by the 7-day soft-delete window in the Power Platform admin center
- **Configuration loss** (`_Config`) — covered by the Config Reference Sheet maintained per the LP-ALM methodology

**Disaster recovery considerations for IL5:**
- RPO: Dataverse automatic backup = up to 24 hours data loss acceptable for most workloads; for Mission Critical, implement daily manual exports to Azure Government Blob Storage
- RTO: Environment restoration from backup = 1–4 hours; solution reimport = depends on pipeline; total RTO for a full environment rebuild from source = 4–8 hours for a well-maintained LP-ALM repository
- Cross-region failover: Dataverse does not support active-active cross-region for GCC High. Passive DR requires a secondary environment (not recommended unless mission criticality warrants the licensing cost)
- Document your DR procedures in the environment register and test them annually

---

## 7. Leadership Reporting & Metrics

> **Well-Architected:** Operational Excellence (observability), Experience Optimization (business outcomes)

### 7.1 What Leadership Actually Cares About

Senior leadership in a DoD organization does not care about flow run counts or connector usage. They care about:

1. **Mission impact** — Are the applications supporting the mission? Are users adopting them?
2. **Risk** — Are there open security findings? ATO gaps? Unsupported applications?
3. **Cost** — Are we getting value from the platform investment?
4. **Ownership** — Who is accountable for what?

Everything else is noise.

### 7.2 Executive Scorecard (Monthly)

| Metric | Target | Why It Matters |
|---|---|---|
| Active production applications | Trend up | Platform is delivering value |
| Applications with no assigned owner | 0 | Ownerless apps become unsupportable |
| Open ATO findings (Critical) | 0 | Compliance posture |
| Open ATO findings (High) | < 5 | Compliance trend |
| Environments past decommission date | 0 | Sprawl control |
| Applications with no activity in 90 days | < 10% | Utilization efficiency |
| Platform-wide licensing utilization | 60–80% | Right-sizing |
| CoE intake SLA met (< 5 business days) | > 95% | Governance enabling, not blocking |

### 7.3 Operational Metrics (CoE Team)

| Metric | Cadence | Owner |
|---|---|---|
| Environment count by tier | Monthly | CoE |
| DLP violation count by policy | Weekly | CoE + ISSO |
| Flow failure rate by environment | Daily | CoE |
| Orphaned flows (owner departed) | Monthly | CoE |
| Orphaned apps (owner departed) | Monthly | CoE |
| Unlicensed maker activity | Weekly | CoE |
| Sandbox environments past expiry | Daily | CoE (automated) |
| Solution checker violations in production | Monthly | CoE |

### 7.4 Metrics That Are Noise

Do not include these in leadership reporting — they create alert fatigue and obscure the signal:

- **Total flow runs** — volume is not value; a broken flow that runs 1M times is not an asset
- **Total apps created** — quantity is not quality; 500 unused apps is not a success story
- **Connector usage count** — irrelevant without context
- **Pipeline execution count** — tells leadership nothing they can act on
- **API call volume** — an operational metric, not a strategic one

### 7.5 Platform Adoption Reporting

Adoption reporting answers: "Are our users actually using the platform?" This matters because unused platforms are cancelled platforms.

Recommended adoption metrics:
- Monthly Active Users (MAU) by application — tracked via unified audit log
- User growth rate by command — identifies where the platform is taking hold
- Application age distribution — old apps never decommissioned = technical debt
- Citizen vs. pro developer ratio — tracks fusion team health
- Self-service requests as % of total intake — tracks governance maturity (high self-service % = governance is enabling, not blocking)

### 7.6 Building the Reporting Infrastructure

The CoE Starter Kit provides dashboards for most operational metrics. For executive reporting:

- Build a Power BI report on top of CoE Dataverse data, deployed in the CoE-Admin environment
- Schedule automated email delivery of the executive scorecard (Power Automate + CoE data)
- Do not build the reporting in Excel. At DoD scale, an Excel-based governance report is outdated the moment it is sent.
- Use Azure Government Power BI Premium for large-scale reporting; do not route tenant governance data through commercial Power BI service

---

## 8. Recommended Enterprise Standards

> **Well-Architected:** Operational Excellence (development standards), Security (consistent posture)

### 8.1 Naming Conventions

All naming follows the pattern defined in LP-ALM Section 4 with the following DoD-specific additions:

| Artifact | Pattern | Example |
|---|---|---|
| Environment | `{CMD}-{PGM}-{TYPE}` | `FORSCOM-SYSTRK-PROD` |
| Publisher | `{CMD}{PGM}` (no hyphen) | `FORSCOMSYSTRK` |
| Solution | `{PGM}-{Descriptor}` | `SYSTRK-Core`, `SYSTRK-Flows` |
| Solution prefix | lowercase, 4–6 chars | `systrk` |
| Application | `{PGM} {Friendly Name}` | `SYSTRK Ticket Tracker` |
| Flow | `{PGM} - {Trigger} - {Action}` | `SYSTRK - New Ticket - Notify Approver` |
| ADO Project | `{CMD}-{PGM}` | `FORSCOM-SYSTRK` |
| Entra Group (env access) | `PP-{PGM}-{TYPE}-{ROLE}` | `PP-SYSTRK-PROD-Users` |
| Entra Group (Dataverse role) | `PP-{PGM}-{TYPE}-{AppCode}-{ROLE}` | `PP-SYSTRK-PROD-APP1-Admins` |
| Entra Group (platform) | `PP-Platform-{ROLE}` | `PP-Platform-CoE-Engineers` |
| Service Principal | `sp-pp-{env}-{type}` | `sp-pp-systrk-prod-pipeline` |
| Table | `{prefix}_TableName` | `systrk_SystemProfile` |
| Column | `{prefix}_columnname` | `systrk_systemstatus` |
| Environment Variable | `{prefix}_VariableName` | `systrk_ApiEndpoint` |
| Flow | `{prefix} - Descriptive Name - Trigger` | `systrk - Create System Record - HTTP` |
| Canvas App | `{prefix} System Tracker` | `systrk System Tracker` |

### 8.2 Publisher Standards

One publisher per program team. Publisher display name includes the command acronym. Publisher prefix is the solution prefix (lowercase, 4–6 characters, unique within tenant). The default solution publisher should never be used — "Default Publisher" in a production environment is a governance failure.

Publisher uniqueness must be enforced at intake — the CoE intake form validates prefix uniqueness before approving a new program.

### 8.3 Solution Segmentation

Follow the LP-ALM five-layer model. The only approved deviation is the extension patterns documented in LP-ALM Section 2.6 (multiple UI solutions, `_Integration` layer, shared `_Core`). Any other deviation requires architecture review.

### 8.4 Canvas App Design Standards

- **One app per canvas solution** — multiple apps in one solution make component-level rollback impossible
- **App checker: Critical = 0, High = 0** before production
- **Named formulas over global variables** for performance and readability
- **No hardcoded URLs** — all endpoints via environment variables in `_Config`
- **Delegation warnings must be resolved** — non-delegable queries are a scalability time bomb; at DoD scale, non-delegable queries against large tables will fail
- **GCC High SharePoint connector** uses the `.sharepoint.us` endpoint — canvas apps calling commercial SharePoint from GCC High are out of compliance
- **Screen loading strategy**: use named formulas in `App.OnStart` sparingly; prefer `App.Formulas` (named formulas) for performance
- **Error handling**: every patch/submit operation must handle errors explicitly — display a user-facing error, log to a Dataverse audit table

### 8.5 Model-Driven App Standards

- Business Process Flows are always in `_Core`, not `_UI`
- Views and forms belong in `_UI`
- Site map is in `_UI`
- Do not add solution components to the default solution — always use the named program solution
- Business rules: simple visibility/requirement rules belong in `_Core`; complex business logic belongs in a plugin (`_Core`) or flow (`_Automation`)

### 8.6 Power Automate Design Standards

- **Cloud flows over desktop flows** wherever possible — desktop flows require an unattended RPA license and a dedicated machine; at DoD scale, this is expensive and fragile
- **Child flows** for reusable logic — do not copy-paste the same 15 actions across 20 flows
- **Error handling on every action** that can fail — configure-run-after settings for error and timeout paths
- **Retry policy**: set explicit retry policy on HTTP calls and connector actions — do not rely on the default 4-retry behavior
- **Concurrency control**: for flows triggered by Dataverse events on high-volume tables, set concurrency to a value that the downstream API can handle
- **No personal connections in any environment except personal Dev** — all Test and Prod connections use service accounts or approved non-personal credentials per program IAM plan
- **Flow ownership**: flows owned by a person become inoperative when that person departs. All production flows must be owned by the program Dataverse Owner Team, not an individual.

### 8.7 Plugin vs. Flow Guidance

| Use Plugin | Use Flow |
|---|---|
| Sub-second synchronous business logic | Asynchronous background processing |
| Data validation on save | Cross-system integration |
| Complex calculations required in the transaction | Approval workflows |
| Rollback behavior required (transaction participation) | Scheduled processing |
| Performance-critical operations | Human-in-the-loop processes |
| Logic that must work offline | Notification and alerting |

Plugins execute in the Dataverse transaction and can roll back. Flows do not participate in Dataverse transactions. The wrong choice here causes data consistency failures at scale.

### 8.8 Logging and Error Handling Standards

Every production flow and plugin must log to a dedicated audit/telemetry table in `_Core`:

```
Table: {prefix}_PlatformLog
Columns:
  {prefix}_name        (string) — component name + operation
  {prefix}_status      (choice) — Success, Warning, Error, Info
  {prefix}_message     (multiline text) — human-readable description
  {prefix}_details     (multiline text) — technical detail / stack trace
  {prefix}_correlationid (string) — trace ID for cross-system correlation
  createdon            (system) — timestamp
  createdby            (system) — triggering user/SP
```

This table feeds both L2 operational support and ATO audit evidence. Do not log sensitive field values — log field names and IDs only.

---

## 9. Risks & Anti-Patterns

> **Well-Architected:** Security (threat model), Reliability (risk identification), Operational Excellence (safe practices)

### 9.1 Environment Sprawl

**What it looks like:** 300 environments with no documented owners, mixed data classifications in the same environment, sandbox environments running for 2 years.

**Why it happens:** Self-service provisioning without lifecycle enforcement. Every team that needed a quick environment made one and never deleted it.

**How to prevent it:** Automated deprovisioning workflows triggered by expiry date. Quarterly environment review sent to all environment owners. Environments with no owner and no activity for 60 days are auto-quarantined (not deleted — quarantined, with 30-day window to claim).

### 9.2 ATO Theater

**What it looks like:** An ATO document that describes a system that no longer exists, or describes perfect security controls that are not actually implemented.

**Why it happens:** ATO is written once at system launch and never updated to reflect changes.

**How to prevent it:** ATO evidence is generated from the system, not written about the system. Security role exports, environment variable documentation, DLP policy screenshots, and audit log samples should be generated automatically from the live environment on a quarterly basis and compared against the ATO description.

### 9.3 Shared Admin Accounts

**What it looks like:** One "PowerPlatformAdmin" email account with the password shared among 8 people.

**Why it happens:** Difficulty provisioning service accounts in DoD agencies; convenience.

**Why it is catastrophic:** Shared accounts cannot be audited. When a security event occurs, you cannot determine which person acted. The account cannot be tied to an individual in the audit log. This is a FISMA compliance failure. Every ISSO reviewing this will flag it.

**How to prevent it:** Use service principals for automation. Use individual named accounts with time-bounded Just-In-Time (JIT) privileged access for human admin activities. Never share credentials.

### 9.4 Personal Connections in Production

**What it looks like:** A Power Automate flow in production runs under a named GS-12's Office 365 connection. When that person transfers, the flow breaks.

**Why it happens:** Service accounts were hard to get. The developer used their own account and forgot.

**The cascade failure:** The GS-12 transfers → flow connection expires → production application stops working → mission impact → emergency ticket → 2 weeks to get a new service account or non-personal credential → production down for 2 weeks.

**How to prevent it:** Pipeline deployment validates that connection references are bound to non-personal accounts before promoting to production (scriptable check). CoE Starter Kit reports on flows with connections owned by individuals.

### 9.5 Over-Engineered Security Models

**What it looks like:** 47 custom security roles. A BU hierarchy 6 levels deep. Field security profiles on every column including non-sensitive lookup fields.

**Why it happens:** Security teams applying maximum restriction by default.

**Why it is a problem:** No one understands the model. Support calls flood in for basic operations. The ISSO who approved it left. The replacement ISSO cannot audit it. The first significant personnel change breaks something and no one knows what.

**How to prevent it:** Five roles per application maximum. Three BU levels maximum. Field security only on columns that contain CUI or PII. Document every security decision in the ATO with a specific justification tied to a NIST control.

### 9.6 Dataverse Misuse

**What it looks like:** Storing documents, images, and attachments in Dataverse Notes as the primary document management system. 500GB database from a 50-user app.

**Why it happens:** Dataverse Notes are convenient. No one planned storage.

**The result:** Storage costs balloon (Dataverse storage is significantly more expensive per GB than SharePoint/Blob). Performance degrades. Backup times increase.

**How to prevent it:** Documents belong in SharePoint (GCC High) or Azure Government Blob Storage. Dataverse stores references (URLs) to documents, not the documents themselves. Establish a storage architecture decision at intake.

### 9.7 ALM Failures

**What it looks like:** Production is different from Test. "We just edited it in production — it was faster." Developers with System Administrator access in production.

**Why it happens:** Pressure to deliver quickly, combined with insufficient pipeline investment early in the program.

**The cascade:** Production diverges from source control → source control becomes unreliable → developers stop trusting the pipeline → more ad-hoc production changes → ATO cannot be supported → ISSO flags production as out-of-compliance with documented baseline.

**How to prevent it:** No developer has write access to the production environment. The pipeline is the only deployment path. This is enforced by Entra ID group membership, not by trust.

### 9.8 Ignoring Licensing Until It Breaks

**What it looks like:** A program builds an application on the assumption that the M365 license includes Power Apps. It launches to 1,000 users. The first large-scale deployment hits the per-user plan limit. Emergency procurement begins.

**Why it happens:** Licensing is not technically complex, so it gets deferred.

**How to prevent it:** Licensing capacity check is part of the intake process. The CoE maintains a capacity dashboard. Any program expected to exceed free-tier entitlements requires a licensing plan as part of the intake request. Plan this in year 1.

### 9.9 CoE Starter Kit Drift

**What it looks like:** The CoE Starter Kit is deployed once and never updated. 18 months later, it runs on deprecated APIs, generates alerts that no one reads, and the CoE team has stopped trusting its data.

**Why it happens:** The CoE Starter Kit requires active maintenance — it is not set-and-forget.

**How to prevent it:** Assign a dedicated CoE engineer responsible for the CoE Starter Kit version and configuration. Update on the Microsoft-published cadence (roughly quarterly). Treat it like any other enterprise application — with a maintenance window, test environment, and change review.

### 9.10 Bypassing the `_Config` Protocol

**What it looks like:** A developer hardcodes an environment variable value inside a flow action. Or commits a `_Config` solution export to source control because "it was easier." Or the pipeline applies `_Config` automatically.

**Why it happens:** The `_Config` manual step feels like friction.

**Why it is dangerous:** `_Config` contains endpoint URLs and connection reference bindings. Committing it to source exposes configuration (and potentially embedded credential references) to every repository reader. Automating it bypasses the intentional human checkpoint that prevents a Test configuration from being applied to Production. This is the foundational LP-ALM security rule — there is no workaround.

---

## 10. Final Recommended Strategy

### 10.1 The Recommended Model

For a large DoD organization, the recommended architecture is:

> **Federated governance. One GCC High tenant. One Platform CoE. Managed Environments enforced everywhere. LP-ALM for all pro-developer workloads. Automated intake and provisioning. Self-hosted ADO agents in Azure Government. CoE Starter Kit as the operational foundation.**

This model delivers:
- ATO-supportable, auditable, documented environments
- Development velocity through self-service sandboxes and pre-approved patterns
- Security posture that survives personnel turnover
- Cost visibility and licensing discipline
- A governance process that enables innovation rather than gating it

### 10.2 Tradeoff Analysis

| Tradeoff | What You Give Up | What You Gain |
|---|---|---|
| One production environment per program (vs. shared) | Lower licensing cost; simpler management of fewer environments | ATO isolation; blast radius containment; program autonomy |
| Self-hosted agents (vs. managed) | Setup and maintenance burden | GCC High compatibility; IP-restricted access to IL5 environments |
| LP-ALM mandatory for Mission/Business Critical | Dev speed for small teams early on | Supportability, rollback capability, ATO evidence, long-term sustainability |
| Citizen dev governance (vs. fully open) | Developer freedom; faster small wins | No shadow IT; no ungovernable orphan apps at year 3 |
| Separate IL5 environments (vs. mixed) | Additional licensing and management overhead | Compliance with IL5 DISA PA; defensible ATO boundary |

The most significant tradeoff is governance overhead vs. velocity. At small scale (< 5 programs, < 50 developers), this model is heavier than necessary. At DoD scale (> 20 programs, hundreds of developers), this model is the minimum required to remain manageable.

### 10.3 Why This Model Works for DoD-Scale Organizations

1. **Personnel is not a dependency.** Service principals, Entra groups, and source control ensure the platform survives individual turnover. The number one operational failure in government Power Platform deployments is an application that only one person understands, and that person is gone.

2. **The governance model scales.** Automated intake, self-service sandboxes, and pre-approved patterns mean the CoE team is not a bottleneck. A 5-person CoE team can govern 100+ programs with this model.

3. **ATO evidence is produced, not assembled.** Security role exports, audit logs, DLP policy records, and environment configuration are maintained as operational artifacts — not assembled in a spreadsheet the week before an ATO review.

4. **It aligns with how DoD actually works.** Commands are decentralized. Programs have their own PMs and ISSOs. Central control of every deployment is not realistic. Centralized standards with decentralized execution matches the organizational reality.

### 10.4 Phased Implementation Roadmap

**Phase 1 — Foundation (Months 1–6)**
- [ ] Establish Platform CoE team and PGB charter
- [ ] Deploy CoE Starter Kit to CoE-Admin environment
- [ ] Implement environment naming standard and enforce at tenant level
- [ ] Activate Managed Environments org-wide
- [ ] Implement 3-tier DLP policy architecture
- [ ] Configure self-hosted ADO agent pool in Azure Government
- [ ] Build automated intake application (CoE-Admin)
- [ ] Train first cohort of pro developers on LP-ALM methodology
- [ ] Publish enterprise standards (this document + LP-ALM + component placement decision tree)
- [ ] Establish SIEM integration for audit log routing

**Phase 2 — Scale (Months 7–18)**
- [ ] Onboard first 10–20 programs through LP-ALM pipeline
- [ ] Implement automated sandbox expiry and deprovisioning
- [ ] Build executive reporting dashboard (CoE data → Power BI)
- [ ] Implement DLP violation alerting in SIEM
- [ ] Establish quarterly governance review cadence
- [ ] Train citizen developer cohort with governance guardrails
- [ ] Implement annual environment owner confirmation workflow
- [ ] Achieve CoE Starter Kit full deployment across all production environments

**Phase 3 — Optimize (Months 19–36)**
- [ ] Full automation of environment lifecycle management
- [ ] Self-service developer environment provisioning
- [ ] Automated ATO evidence collection (quarterly evidence package generation)
- [ ] Platform-level capacity forecasting and procurement automation
- [ ] Cross-command shared component library operational in Shared Services
- [ ] CoE governance board operating with < 5-day SLA for all intake requests
- [ ] Annual Well-Architected review for top 10 production applications
- [ ] Hotfix drill and DR test conducted for all Mission Critical applications

### 10.5 Immediate Priorities

If you implement nothing else from this document, implement these four things first:

1. **No developer writes directly to production.** This single control prevents the most common ALM failures in government Power Platform deployments.

2. **Every environment has a documented human owner.** Not a team name — a specific person accountable for that environment. When that person transfers, the first task in their offboarding is environment ownership transfer.

3. **`_Config` is never committed to source control.** See [LP-ALM.md foundational security rule 1](/methodology/).

4. **Activate Managed Environments and turn on the Unified Audit Log.** Without these two controls, you have no visibility into what is happening in your tenant and no ability to support an ATO.

Everything else in this document is important. These four are non-negotiable.

---

*LP-ALM Enterprise Strategy v1.0 | May 2026*

*Aligned with [Power Platform Well-Architected](https://learn.microsoft.com/en-us/power-platform/well-architected/) — Security, Reliability, Operational Excellence, Performance Efficiency, and Experience Optimization pillars.*

*Built with assistance from GitHub Copilot (Claude Sonnet 4.6). All output reviewed by a human.*
