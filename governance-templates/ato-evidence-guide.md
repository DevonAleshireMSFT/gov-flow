---
layout: default
title: ATO Evidence Guide
parent: Governance Templates
nav_order: 4
permalink: /governance-templates/ato-evidence-guide/
---

# ATO Evidence Collection Guide — Power Platform GCC High
{: .no_toc }

**Template version:** 1.0 — May 2026
{: .text-delta }

This guide defines what evidence to collect, how to collect it, and at what cadence for Power Platform Authority to Operate (ATO) package preparation. Use this alongside your organization's standard ATO process (DISA RMF, NIST SP 800-37).

{: .important }
> ATO evidence must be **generated from the live system** — not written about the system. A security plan that describes access controls that are not actually implemented is ATO theater. Each evidence item below includes the source and collection method.

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Evidence collection schedule

| Evidence Type | Frequency | Owner | Automated? |
|---|---|---|---|
| Security role export (XML) | Quarterly | CoE / Program Team | Yes (PAC CLI) |
| DLP policy export | Quarterly | CoE | Yes (Admin API) |
| Environment user list with roles | Quarterly | CoE | Yes (Admin API) |
| Managed solution inventory | Quarterly | CoE | Yes (CoE Starter Kit) |
| Unified Audit Log sample | Quarterly | ISSO | Partial (export manual) |
| Dataverse audit log sample | Quarterly | ISSO | Partial |
| Conditional Access policy export | Quarterly | Command IAM | Yes (Entra ID export) |
| Service principal inventory | Quarterly | CoE | Partial |
| Environment register (current state) | Quarterly | CoE | Yes (Dataverse report) |
| Penetration test report | Annual (Mission Critical) | ISSO | No |
| Vulnerability scan results | Annual | ISSO | No |
| Backup restore test results | Annual | CoE | No |

---

## Evidence items by NIST control family

### AC — Access Control

#### AC-2: Account Management

**What to collect:**
- List of all application users (service principals) in each environment with their assigned security roles
- List of all Entra ID security groups mapped to each environment
- Service principal inventory: App ID, environment, last credential rotation date
- Confirmation that no personal accounts hold System Administrator in Test or Prod

**How to collect:**

```powershell
# Export application users and their security roles per environment
pac auth create --name "Prod" --kind ServicePrincipal --applicationId <app-id> `
  --clientSecret <secret> --tenant <tenant-id> --cloud UsGovHigh `
  --environment https://{org}.crm.microsoftdynamics.us

# List application users
pac org list-application-users --environment https://{org}.crm.microsoftdynamics.us

# Export security roles assigned to application users (via Power Platform Admin API)
# GET https://api.bap.microsoft.us/providers/Microsoft.BusinessAppPlatform/
#     environments/{envId}/roleAssignments?api-version=2016-11-01
```

**NIST control statement:**
> Service principal application users are provisioned and documented per environment. No personal credentials are used in Test or Prod environments. All human admin access to production is logged via the Unified Audit Log.

**Evidence artifact:** `AC-2_AppUsers_{ENV}_{DATE}.csv` — exported list of application users, roles, and last-reviewed date.

---

#### AC-3: Access Enforcement

**What to collect:**
- Security role XML exports for each custom role in each environment
- Managed Environments configuration screenshot confirming managed solutions are enforced
- Confirmation that `_Security` solution deployed before `_Core` (pipeline run log)

**How to collect:**

```powershell
# Export unpacked security roles from source control (these ARE the definitions)
# Located at: solutions/{ProjectCode}_Security/src/Roles/

# Export managed solution inventory from environment
pac solution list --environment https://{org}.crm.microsoftdynamics.us
```

**NIST control statement:**
> Security roles deploy before data schema in every environment (LP-ALM `_Security`-first deployment). Managed solutions in Test and Prod prevent ad-hoc customization. Access control structures exist for every data entity at all times.

**Evidence artifact:**
- `AC-3_SecurityRoles_{ENV}_{DATE}.zip` — packed `_Security` solution export
- `AC-3_ManagedSolutions_{ENV}_{DATE}.csv` — solution list showing managed status

---

#### AC-6: Least Privilege

**What to collect:**
- Security role privilege matrix (the completed [LP-ALM security role matrix template](https://devonaleshiremsft.github.io/layered-platform-alm/security-roles/)) for each production environment
- Confirmation that no developer holds write access to production (Entra group membership export)
- Confirmation that System Administrator is only assigned to pipeline service principal

**How to collect:**

```powershell
# Export Entra ID group membership for PP-{ENV}-PROD-Developers
# Confirm group is empty (developers should not have write access to Prod)
# Via Microsoft Graph:
# GET https://graph.microsoft.us/v1.0/groups/{groupId}/members
```

**NIST control statement:**
> Custom security roles are designed per persona with minimum required privileges. The `{ProjectCode} - Automation Service` role is the least-privilege role for flows. System Administrator is assigned only to the pipeline service principal. No developer holds write access to production environments.

**Evidence artifact:** `AC-6_PrivilegeMatrix_{ENV}_{DATE}.xlsx` — completed security role matrix with explicit documentation of why each privilege level was granted.

---

### AU — Audit and Accountability

#### AU-2: Event Logging

**What to collect:**
- Confirmation that Unified Audit Log is enabled for the tenant
- Confirmation that Dataverse auditing is enabled per production environment
- Confirmation that audit logs route to the command SIEM (Azure Government Event Hub / Sentinel)

**How to collect:**

```powershell
# Confirm Unified Audit Log is enabled
# Power Platform Admin Center → Tenant Settings → Audit Log

# Confirm Dataverse auditing per environment
# Power Platform Admin Center → Environment → Settings → Auditing
```

**NIST control statement:**
> The Microsoft Purview Unified Audit Log is enabled for the GCC High tenant. Dataverse activity logging is enabled for all production environments. Audit events route to the command SIEM via Azure Government Event Hub within [SLA].

**Evidence artifact:** `AU-2_AuditConfig_{DATE}.png` — screenshot of Unified Audit Log enabled state and Dataverse audit settings per environment.

---

#### AU-9: Protection of Audit Information

**What to collect:**
- Confirmation that audit logs are write-protected (Unified Audit Log is immutable by design)
- Log retention configuration (minimum 1 year hot, 6 years cold per NARA)
- Access control on the SIEM workspace (who can query logs)

**NIST control statement:**
> Audit logs are stored in Microsoft Purview (immutable by platform design). Retention is configured for [X] years inline and [X] years cold per NARA schedule 2. Access to the SIEM workspace is restricted to the ISSO and SOC team.

---

### CM — Configuration Management

#### CM-2: Baseline Configuration

**What to collect:**
- Current source control state (git commit hash) for each production environment's deployed solutions
- Pipeline run ID that produced the current production deployment
- Environment register entry for each production environment

**How to collect:**

```powershell
# Get current solution versions deployed to Prod
pac solution list --environment https://{org}.crm.microsoftdynamics.us

# Get the git commit hash that produced the current artifact
# From ADO: Pipelines → {run} → Artifacts → Build tag
git log --oneline -1  # from the deployed branch/tag
```

**NIST control statement:**
> The authoritative configuration baseline for each production environment is the LP-ALM source control repository at the tagged release commit. The `_Config` layer values are documented in the environment configuration register (stored outside source control in accordance with LP-ALM security protocols).

**Evidence artifact:**
- `CM-2_SolutionInventory_{ENV}_{DATE}.csv` — solution names, versions, managed status
- `CM-2_GitHash_{ENV}_{DATE}.txt` — git commit hash and tag for current production deployment

---

#### CM-3: Configuration Change Control

**What to collect:**
- ADO pipeline run history for each production environment (showing all changes went through pipeline)
- PR history with approver names (showing change review)
- Confirmation that no manual customizations exist in production (no active unmanaged layers)

**How to collect:**

```powershell
# Check for active customizations (unmanaged layers) in Prod
# Power Platform Admin Center → Environment → Solutions → look for "(Active)" layers
# OR via API:
# GET https://{org}.crm.microsoftdynamics.us/api/data/v9.2/solutions?
#     $filter=ismanaged eq false and isvisible eq true
```

**NIST control statement:**
> All changes to production environments are deployed through Azure DevOps pipelines with mandatory PR review. Managed solutions block direct environment modification. No active unmanaged customizations exist in production environments.

**Evidence artifact:**
- `CM-3_PipelineRuns_{ENV}_{DATE}.csv` — ADO pipeline run history (last 90 days)
- `CM-3_PRHistory_{DATE}.csv` — Pull request history with reviewers (last 90 days)

---

#### CM-6: Configuration Settings

**What to collect:**
- Environment variable values (documented in config management record — NOT in source control)
- DLP policy configuration export
- Managed Environments settings per environment

**How to collect:**

```powershell
# Export DLP policies
# Power Platform Admin Center → Data Policies → Export (JSON)
# OR via Admin API:
# GET https://api.bap.microsoft.us/providers/Microsoft.BusinessAppPlatform/
#     environments/{envId}/governancePolicies
```

{: .warning }
> Environment variable **values** must be documented in the config management record, not in source control. Reference the values by environment variable name in this evidence item — do not include the actual values in the ATO package if they contain endpoint URLs or sensitive configuration.

---

### IA — Identification and Authentication

#### IA-2: Multi-Factor Authentication

**What to collect:**
- Conditional Access policy export showing MFA required for all Power Platform access
- Confirmation of Conditional Access policy for compliant device (Prod environments)
- Sign-in log sample showing MFA-compliant sign-ins

**How to collect:**

```
# Entra ID → Security → Conditional Access → Policies → Export (JSON)
# Entra ID → Sign-in logs → Filter by application: Power Apps
```

**NIST control statement:**
> Multi-factor authentication is enforced for all Power Platform access via Conditional Access policy `[Policy Name]`. Compliant device enforcement is required for production environment access. High-risk sign-ins are blocked automatically.

---

### SA — System and Services Acquisition

#### SA-3: System Development Life Cycle

**What to collect:**
- LP-ALM methodology document (link or export)
- Environment topology diagram (from Enterprise Strategy Section 2.2)
- Dev → Test → Prod promotion evidence (pipeline run sequence)
- Rollback procedure documentation

**NIST control statement:**
> Power Platform solutions follow the Layered Power Platform ALM (LP-ALM) methodology, which implements a structured SDLC with distinct Dev (unmanaged), Test (managed), and Prod (managed) stages. Each deployment layer is independently deployable and rollback-capable. The `_Config` layer is always deployed manually by an authorized person, never through automated pipeline.

---

## Quarterly evidence package procedure

Use this procedure to generate the quarterly evidence package. Build this as an automated Power Automate flow where possible.

1. **Export security roles** for all production environments (PAC CLI)
2. **Export DLP policies** (Power Platform Admin API)
3. **Export solution inventory** per production environment (PAC CLI)
4. **Export pipeline run history** last 90 days (ADO API)
5. **Export Entra ID group membership** for all `PP-{ENV}-*` groups (Graph API)
6. **Export Unified Audit Log** sample — 7 days of Dataverse modification events (Purview)
7. **Pull environment register** current state (Dataverse report)
8. **Confirm no active unmanaged layers** in production environments (Admin API)
9. **Review service principal credential expiry dates** — flag any within 30 days
10. **Document any open findings** from prior quarter and remediation status

Package all artifacts in a dated folder: `ATO-Evidence_{YYYY}-Q{Q}/` stored in SharePoint GCC High.

---

## ATO finding response guide

| Finding category | Typical cause | Remediation |
|---|---|---|
| System Admin assigned to human user in Prod | Developer convenience; emergency access not cleaned up | Remove role; enable PIM JIT for future emergency access; document in ATO |
| Personal credential in production connection reference | Service account not available; developer used own account | Escalate to IAM for service account; rebind connection reference; document in ATO |
| Unmanaged customizations in production | Ad-hoc change bypassed pipeline | Remove active layer; redeploy managed solution from pipeline; implement Entra access controls to prevent future write access |
| Audit log not enabled | Missed in environment provisioning checklist | Enable immediately; note gap period in ATO; assess whether any auditable events during gap require review |
| _Config found in source control | Developer convenience; process not enforced | Remove from source control history (git filter-branch or BFG); rotate any exposed values immediately; add pre-commit hook |
| DLP policy not assigned to environment | New environment provisioned before DLP group assignment | Assign immediately; review connector usage in environment for any unauthorized connections during gap period |
| Missing Append/Append To privileges | Incomplete security role design | Update `_Security` solution; redeploy; document in privilege matrix |
