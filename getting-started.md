---
layout: default
title: Getting Started
nav_order: 3
permalink: /getting-started/
---

# Getting Started
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The framework stack

Before implementing anything, understand where each framework sits. Attempting to use gov-flow without the right platform prerequisites — or LP-ALM without the right governance foundation — leads to partial implementations that fail at ATO time.

```
┌────────────────────────────────────────────────────────────────┐
│              Power Platform Well-Architected                   │
│         (evaluate the workload — do this last)                 │
├────────────────────────────────────────────────────────────────┤
│                       LP-ALM                                   │
│   Five-layer decomposition · Pipeline YAML · PAC CLI workflow  │
│   Security role design · Source control structure              │
│   governs: inside the solution artifact                        │
├────────────────────────────────────────────────────────────────┤
│                      gov-flow  ◄ you are here                  │
│   Environment topology · DLP architecture · RBAC · CoE model   │
│   ATO governance · Leadership reporting · IL5 isolation        │
│   governs: the platform layer around the solutions             │
├────────────────────────────────────────────────────────────────┤
│              Power Platform Landing Zones                      │
│         (establish this first — foundational infra)            │
└────────────────────────────────────────────────────────────────┘
```

{: .important }
> Implement in order: **Landing Zones foundation → gov-flow governance → LP-ALM methodology → Well-Architected review.**
> gov-flow assumes Landing Zones prerequisites are met. LP-ALM assumes gov-flow governance is in place.

---

## Prerequisites

The following must exist before implementing gov-flow. These are Landing Zones / platform admin responsibilities.

| Prerequisite | Why it matters |
|---|---|
| GCC High tenant provisioned | All gov-flow patterns assume a single GCC High tenant per branch/agency |
| Power Platform licenses acquired via DoD EA | Managed Environments and full governance features require premium licenses |
| Entra ID / Azure AD synchronized with HR system | Dynamic security groups for end-user populations require HR attribute synchronization |
| Azure Government subscription | Self-hosted ADO agents, Key Vault, Event Hub for audit log routing — all must be in Azure Government |
| ISSO identified for the platform | The Platform ISSO is required to participate in the PGB and approve IL5 environments |
| Unified Audit Log enabled | Required for ATO; must be enabled at the tenant level by a tenant admin |
| Power Platform Managed Environments license | Required to activate Managed Environments org-wide |

---

## Key decisions before you start

These decisions are hard to reverse. Make them before provisioning any environments.

### Decision 1: Tenant topology

**Recommended:** One GCC High tenant per branch/agency. Running multiple tenants within a single branch creates federation complexity that eliminates most platform-level governance benefits.

**If you have multiple existing tenants:** Do not consolidate mid-program. Document the topology, establish cross-tenant governance as a separate workstream, and standardize on one tenant for all new programs.

### Decision 2: Environment naming standard

Choose and enforce the naming standard **before** provisioning production environments. Renaming environments later requires URL changes in all pipelines, connection references, and documentation.

The gov-flow standard:
```
{ORG}-{COMMAND}-{PROGRAM}-{TYPE}
Examples: ARMY-FORSCOM-SYSTRK-PROD, NAVY-NETC-TRNMGR-DEV
```

See [Enterprise Strategy — Section 2.5](../enterprise-strategy/#25-environment-naming-standard) for full rules.

### Decision 3: Publisher prefix

One publisher prefix per program. Choose 4–6 lowercase characters. This prefix is prepended to every custom table, column, and option set in the program's solutions. It cannot be changed after data exists in production.

Validate uniqueness at intake time — the CoE intake form must reject duplicate prefixes.

### Decision 4: IL5 boundary

Identify which programs handle workloads requiring IL5 authorization **before** provisioning. IL5 environments require:
- Separate Dataverse environments from non-IL5 workloads
- DISA provisional authorization documentation
- Restricted Conditional Access (compliant device, CONUS-only)
- Restricted DLP policy (no connectors routing data to commercial infrastructure)

Mixing IL5 and non-IL5 workloads in the same environment is not authorized. Retroactively separating them requires data migration.

### Decision 5: Service account strategy

Determine early whether your agency IAM policy permits non-interactive service accounts for Power Platform pipelines and connection references. This affects your LP-ALM implementation significantly.

- **If service accounts are permitted:** Use dedicated service accounts per environment tier for connection references; use service principals for pipelines.
- **If service accounts are prohibited:** Escalate to IAM for just-in-time access exceptions for Test/Prod connection references. See [LP-ALM Section 5.6.9](https://devonaleshiremsft.github.io/layered-platform-alm/methodology/#569-connection-reference-binding-in-developer-environments) for the full decision table.

{: .warning }
> Test and Prod connection references **cannot use personal credentials**. If service accounts are prohibited, this must be resolved through agency IAM before any program reaches Test deployment.

---

## First 30 days: Platform CoE foundation

These are the minimum actions to establish governance before programs start requesting environments. Defer everything else.

### Week 1–2: Governance structure

- [ ] Establish the Platform CoE team (minimum: 1 platform lead, 1 ISSO representative)
- [ ] Draft and socialize the [Platform Governance Board charter](../governance-templates/pgb-charter/)
- [ ] Identify command representatives for the PGB
- [ ] Schedule the first PGB meeting

### Week 2–3: Technical foundation

- [ ] Activate Managed Environments for the entire tenant
- [ ] Enable the Unified Audit Log at the tenant level
- [ ] Implement the [three-tier DLP policy architecture](../enterprise-strategy/#34-dlp-policy-architecture) (Tenant Default → Environment Group → Exception)
- [ ] Provision the `PLATFORM-COE-ADMIN` environment
- [ ] Deploy the [CoE Starter Kit](https://learn.microsoft.com/en-us/power-platform/guidance/coe/starter-kit) to `PLATFORM-COE-ADMIN`
- [ ] Configure self-hosted ADO agent pool in Azure Government

### Week 3–4: Intake and provisioning automation

- [ ] Build the environment request intake application (Power Apps form in `PLATFORM-COE-ADMIN`)
- [ ] Implement automated naming standard validation in the intake form
- [ ] Configure automated sandbox provisioning (Power Platform Admin API)
- [ ] Set up sandbox expiry workflow (auto-quarantine at 90 days)
- [ ] Publish the [environment naming standard](../enterprise-strategy/#25-environment-naming-standard) to all program teams

### Month 2: Training and first programs

- [ ] Train first cohort of pro developers on [LP-ALM](https://devonaleshiremsft.github.io/layered-platform-alm/)
- [ ] Onboard first 1–3 programs through the intake process
- [ ] Validate the end-to-end pipeline for the first LP-ALM program
- [ ] Configure audit log routing to Azure Government SIEM

---

## Immediate priorities (if you implement nothing else)

From the [Enterprise Strategy](../enterprise-strategy/#105-immediate-priorities):

**1. No developer writes directly to production.**
This single control prevents the most common ALM failures in government Power Platform deployments. Enforce through Entra ID group membership — not trust.

**2. Every environment has a documented human owner.**
Not a team name — a specific person accountable for that environment. Offboarding must include environment ownership transfer.

**3. Activate Managed Environments and turn on the Unified Audit Log.**
Without these two controls, there is no visibility into what is happening in the tenant and no ability to support an ATO.

**4. `_Config` is never committed to source control.**
This is LP-ALM's foundational security rule. See [LP-ALM Methodology](https://devonaleshiremsft.github.io/layered-platform-alm/methodology/#23-layer-3-_config) for the full rationale.

---

## Migrating an existing deployment

If environments and programs already exist, do not attempt a full restructure immediately. Stabilize first.

1. **Inventory what exists.** Use the CoE Starter Kit environment dashboard or the [environment register template](../governance-templates/environment-register/) to document every environment, its owner, data classification, and ATO status.

2. **Identify the highest-risk gaps.** Check for: developers with System Administrator in production; flows running on personal credentials; environments with no documented owner; production environments without audit logging.

3. **Fix the non-negotiables first.** Remove System Admin from human accounts in production. Document environment owners. Enable audit logging. These do not require program downtime.

4. **Adopt LP-ALM for new programs, not existing ones.** Migrating an existing monolithic solution to LP-ALM requires a maintenance window and data backup planning. See [LP-ALM Section 10.4](https://devonaleshiremsft.github.io/layered-platform-alm/methodology/#104-migrating-an-existing-monolithic-solution-to-lp-alm) for the migration steps.

5. **Use the 3-phase roadmap.** The [phased implementation roadmap](../enterprise-strategy/#104-phased-implementation-roadmap) in the Enterprise Strategy is the playbook for getting from current state to full governance maturity.

---

## Where to go next

| If you are... | Read next |
|---|---|
| Standing up the platform for the first time | [Enterprise Strategy — GCC High / DoD](../enterprise-strategy/) — start at Section 2 |
| Setting up governance bodies and processes | [Enterprise Strategy — Section 4: Governance Model](../enterprise-strategy/#4-governance-model) |
| Configuring DLP policies | [Enterprise Strategy — Section 3.4](../enterprise-strategy/#34-dlp-policy-architecture) |
| Setting up your first LP-ALM program pipeline | [LP-ALM Onboarding Checklist](https://devonaleshiremsft.github.io/layered-platform-alm/onboarding/) ↗ |
| Designing security roles for a program | [LP-ALM Security Role Matrix Template](https://devonaleshiremsft.github.io/layered-platform-alm/security-roles/) ↗ |
| Filling in governance documentation | [Governance Templates](../governance-templates/) |
| Building the leadership scorecard | [Enterprise Strategy — Section 7](../enterprise-strategy/#7-leadership-reporting--metrics) |
