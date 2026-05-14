---
layout: default
title: Default Environment & Org-Wide Access
nav_order: 9
permalink: /default-environment/
---

# Default Environment Governance and Enterprise-Wide Application Access
{: .no_toc }

Why the default Power Platform environment is not suitable for enterprise applications, and how to properly govern applications that legitimately serve large portions of the organization.
{: .fs-5 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What the default environment is

Every Power Platform tenant has exactly one default environment. It is created automatically when the tenant is provisioned and cannot be deleted. Every licensed user in the tenant is automatically provisioned into it with the Environment Maker role — no admin action required.

This behavior is by design. The default environment is intended for individual productivity and personal automation — Power Automate flows for personal use, exploratory canvas apps, and individual Copilot Studio experiments. It is the onramp for citizen development.

It is not designed for enterprise applications, mission systems, or production workloads.

{: .important }
> **The default environment should not host any enterprise production application, CUI data, or organization-wide operational platform.** Placing mission applications in the default environment is a governance failure that creates ATO, security, and sustainment risks that are difficult to remediate after the fact.

---

## Why the default environment creates enterprise governance challenges

### Uncontrollable access scope

Every licensed user in the tenant automatically receives Environment Maker access to the default environment. In a large DoD organization, this means thousands of users can:

- Create apps and flows that read from any table they have Dataverse access to
- Share apps broadly within the environment
- Create connections to external systems

Defining an ATO boundary for this environment is effectively impossible — the user population is unbounded and changes continuously with personnel onboarding and offboarding.

### No meaningful environment owner

The default environment has no program owner, no designated ISSO, and no ATO. There is no single person accountable for what runs in it. This creates an accountability gap that fails audit scrutiny — "the platform team owns it" is not a defensible answer when an auditor asks who reviewed the data flows of an app serving 500 users.

### DLP enforcement limitations

While DLP policies can be applied to the default environment, the environment's open access model makes enforcement harder to sustain:

- Any Environment Maker can build flows using connectors available under the DLP policy
- The combination of broad maker access and a shared environment creates a high surface area for accidental policy circumvention
- A Tier 1 tenant-wide DLP policy should be your most restrictive policy — the default environment must be covered by it, not exempted from it

### Shadow IT accumulation

The default environment is the primary source of shadow IT in large organizations. Apps built by individuals, shared with teams, and then forgotten when the maker PCS's or retires accumulate without ownership records, documentation, or source control. At DoD scale this is not an edge case — it is the norm.

CoE Starter Kit telemetry typically reveals that the majority of unmaintained and orphaned apps in a large tenant originated in the default environment.

### ALM and DevSecOps cannot be enforced

LP-ALM pipelines, managed solution deployment, source control, and DevSecOps practices require dedicated environments with controlled access. None of these can be reliably enforced in the default environment — any maker can modify unmanaged solutions, override components, and deploy changes without a pipeline.

An application that grows from a personal project in the default environment into a mission-critical system is one of the hardest remediation scenarios in Power Platform governance. Avoid it by establishing the environment boundary before applications reach production scale.

---

## What belongs in the default environment

| Appropriate | Not appropriate |
|---|---|
| Individual personal automation (personal flows, no CUI) | Any production application serving an organizational function |
| Exploratory / learning exercises | Applications handling CUI or sensitive data |
| Temporary prototypes (< 30 days, no real data) | Mission-critical or business-critical applications |
| Copilot Studio personal agents (non-CUI) | Applications shared with more than a small team |
| Quick personal productivity apps with no shared data | Any workload requiring ATO coverage |

When a citizen-developed app in the default environment reaches 20+ users or begins handling CUI, it must be migrated to an appropriate dedicated environment. This is enforced through CoE Starter Kit telemetry — see [§4.6 Citizen Development Governance](../enterprise-strategy/#46-citizen-development-governance).

---

## Enterprise-wide application access strategy

Some applications legitimately need to be accessible by a large portion of the organization — or the entire tenant. This is a valid operational requirement. The answer is **not** to use the default environment or to share with "Everyone." The answer is a dedicated enterprise environment with governed access.

### The "share with Everyone" problem

Power Platform allows sharing a canvas app with "Everyone in the organization." This appears convenient but creates serious governance problems:

- Every licensed user — including contractors, temporary personnel, and accounts pending offboarding — can access the application
- Access cannot be audited meaningfully (no group membership log, no approval record)
- Revoking access requires removing the app from Everyone sharing, which affects all users simultaneously — no graduated rollback
- For GCC High / DoD tenants, "Everyone" includes all user types in the tenant; this cannot be scoped to cleared personnel or specific commands

{: .warning }
> **Never share a production application with "Everyone in the organization" in a GCC High or DoD tenant.** Use an Entra ID security group — even if that group contains all users. Group-based sharing is auditable, revocable, and ATO-supportable. "Everyone" sharing is not.

### Dedicated enterprise production environment

Applications that serve large user populations should be deployed to a dedicated managed environment provisioned specifically for that purpose.

**Environment provisioning for org-wide apps:**

| Environment | Purpose | Access gate |
|---|---|---|
| `ENTERPRISE-PROD` | Org-wide or cross-command production apps | `PP-ENTERPRISE-PROD-Access` (parent group) |
| `ENTERPRISE-TEST` | UAT and integration testing | `PP-ENTERPRISE-TEST-Access` |
| `ENTERPRISE-DEV` | Development for enterprise platform apps | `PP-ENTERPRISE-DEV-Developers` |

This environment is owned by the Platform CoE — not by an individual program — with a Platform-level ATO boundary that covers enterprise apps hosted within it.

### Group-based access for large populations

For applications that legitimately serve most or all of the organization, use a single Entra ID group as the environment access gate — populated dynamically from HR system attributes where possible.

```
PP-ENTERPRISE-PROD-AllUsers
├── PP-ENTERPRISE-PROD-{CMD1}-Users    (nested)
├── PP-ENTERPRISE-PROD-{CMD2}-Users    (nested)
└── PP-ENTERPRISE-PROD-{CMD3}-Users    (nested)
```

Dynamic group membership rules (based on department, command, or position attributes) ensure new personnel are automatically included and departing personnel are automatically removed. This directly addresses the personnel turnover problem without manual group management.

Within the environment, Dataverse security roles are assigned through group-backed teams as described in [Environment Access & RBAC](../environment-access/). End users receive the minimum role required — read access on reference data, targeted write access on transactional tables.

### Approval process before broad deployment

No application should be shared with an organization-wide group without a documented approval process. This is not bureaucracy — it is the governance control that prevents a prototype from becoming a mission dependency without ATO coverage.

**Minimum review gate for org-wide deployment:**

- [ ] Application classified under the [application classification model](../enterprise-strategy/#42-application-classification-model) (Mission Critical, Business Critical, or Standard)
- [ ] ISSO review complete — data flow documented, security roles validated
- [ ] ATO authorization confirmed for the Enterprise environment
- [ ] LP-ALM pipeline confirmed — application is deployed as a managed solution
- [ ] Support model established — named owner, support queue, escalation path
- [ ] CoE sign-off on environment group assignment
- [ ] Rollout plan confirmed — phased by command group if possible; not all-at-once

### Centralized ownership and support

Enterprise-wide applications require a different support model than program-specific applications. Because they serve multiple commands, no single program team owns the support queue.

| Function | Owner |
|---|---|
| Platform environment | Platform CoE |
| Application functionality | Designated program lead (even if users span commands) |
| Security role design | ISSO + Platform CoE |
| Group membership management | Identity team (Entra ID admin) |
| User support / help desk | Program lead or designated L1 support queue |
| Monitoring and telemetry | Platform CoE via CoE Starter Kit + [Power Monitoring Framework](https://devonaleshiremsft.github.io/power-monitoring-framework/) |

### Telemetry and auditing requirements

Enterprise-wide applications have broader audit obligations precisely because of their large user population.

**Required for any org-wide production application:**
- CoE Starter Kit telemetry enabled — usage, errors, sharing events logged
- Power Platform audit logging enabled at the environment level
- Audit logs routed to Azure Government Log Analytics (90-day retention minimum; 1-year recommended)
- Monthly usage review by the application owner — identify inactive users for group cleanup
- Quarterly access review by ISSO — confirm group membership is current and justified

---

## GCC High / DoD and IL5 considerations

### Default environment in GCC High / DoD

The default environment in a GCC High or DoD tenant follows the same rules as commercial — every licensed user is automatically an Environment Maker. The GCC High data boundary means data stays within the government cloud, but the governance and accountability gaps are identical.

**Additional DoD-specific risk:** In many DoD organizations, the Power Platform license is provisioned broadly as part of an enterprise agreement. This means the default environment is accessible to a very large user population from day one of tenant deployment — before any governance controls are in place.

The first action when standing up a GCC High or DoD tenant is to restrict default environment access by assigning a security group to the default environment. This limits who can create apps and flows in it immediately.

```
Recommended: Assign a security group to the default environment at tenant standup.
Group: PP-DEFAULT-ENV-Makers (citizen developers and CoE team only)
All other licensed users: can consume apps but cannot create in the default environment
```

### IL5 and the default environment

The default environment is **never appropriate for IL5 data.** IL5 designation requires:
- A DISA provisional authorization specific to the environment
- Restricted Conditional Access policies
- Named ATO boundary with designated ISSO
- Environment access controlled by a cleared personnel group

None of these can be achieved in the default environment. Any workload that handles IL5 data and is found in the default environment is an ATO violation requiring immediate remediation.

### Managed Environments for the default environment

Enable Managed Environments on the default environment to gain:
- Usage insights and maker activity telemetry
- Sharing controls (restrict canvas app sharing to security groups only — block "Everyone" sharing)
- IP firewall (prevent access from non-GFE networks)
- Weekly digest reports to the platform admin team

Managed Environments sharing controls are the primary enforcement mechanism for preventing "Everyone" sharing in the default environment. Enable them before broad tenant rollout.
