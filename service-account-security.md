---
layout: default
title: Service Account Security
nav_order: 7
permalink: /service-account-security/
---

# Service Account Security and Operational Governance
{: .no_toc }

How to provision, secure, govern, and sustain service accounts in an enterprise government Power Platform environment.
{: .fs-5 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Why service accounts are critical in enterprise low-code environments

Power Platform solutions rely on connections — to Dataverse, SharePoint, Exchange, custom APIs, and external systems. Those connections are authenticated under a user identity. In most low-code environments, that identity defaults to whoever built the flow. This works until that person PCS's, retires, changes roles, or has their account disabled.

In a DoD organization, personnel turnover is not an edge case. It is a constant. A service account strategy is not optional infrastructure — it is a continuity control.

Beyond continuity, service accounts in GCC High environments serve a second purpose: **ATO supportability**. An ATO package that documents "flows run under individual contractor accounts" cannot be defended. An ATO package that documents named service accounts with defined roles, rotation schedules, and audit trails can be.

{: .important }
> Every production Power Automate flow, every scheduled process, and every pipeline step that interacts with a Dataverse environment must run under a dedicated service account — not a personal user account. This is a non-negotiable baseline for enterprise government environments.

---

## Risks of unmanaged or shared service accounts

Understanding the risk profile drives the governance model.

| Risk | Description | Impact |
|---|---|---|
| **Account departure** | A flow runs under a personal account; the person leaves; the account is disabled; all flows and connections owned by that account fail | Production outage with no clear recovery path |
| **Credential sprawl** | Passwords stored in flow inputs, environment variable values, or shared documents | Credential exposure; failed rotation; no audit trail |
| **Overprivileged accounts** | A single service account holds Organization-level read on all tables "to keep things simple" | Blast radius of a compromised account is the entire Dataverse environment |
| **Shared accounts** | Multiple pipelines or environments share one service account password | Rotation requires coordinated downtime across all consumers; one misconfiguration breaks everything |
| **No ownership record** | Service accounts exist with no documented owner | Account persists after program end; continues to hold access with no active ISSO awareness |
| **MFA exceptions without policy** | Service accounts exempt from Conditional Access to enable unattended automation | Creates a persistent authentication gap exploitable without MFA |

---

## Service account naming and classification

Use a consistent naming standard that makes the account's purpose, owning program, and environment immediately clear.

### Recommended naming pattern

```
svc-{org}-{program}-{environment}@{tenant}.onmicrosoft.us
```

Examples:
```
svc-platform-coe-prod@agency.onmicrosoft.us          ← CoE platform service account
svc-forscom-systrk-prod@agency.onmicrosoft.us         ← Program-specific production account
svc-forscom-systrk-dev@agency.onmicrosoft.us          ← Program-specific dev account
svc-platform-pipeline@agency.onmicrosoft.us           ← CI/CD pipeline account (cross-env)
```

### Account classification

| Class | Purpose | Scope |
|---|---|---|
| **Platform service account** | CoE automation, tenant-wide admin tasks, CoE Starter Kit flows | Power Platform Admin (PIM-eligible) or scoped to platform environments only |
| **Program service account** | Flows, scheduled processes, and connections within a specific program's environments | Scoped to program environments only; no cross-program access |
| **Pipeline service account** | CI/CD deployment pipelines (ADO, GitHub Actions) | Deployment-scoped; no runtime data access |
| **Integration service account** | Connections to external systems (SharePoint, Exchange, external APIs) | Scoped to the specific connector and resource; no Dataverse write unless required |

---

## Least privilege and role-based access

{: .important }
> A service account should hold the minimum access required for its defined function — nothing more. Access is scoped to the environment, the table, and the operation. "Organization-level Read on all tables" is not an acceptable baseline for any service account.

### Dataverse security role design for service accounts

Define custom security roles for each service account class rather than reusing existing roles.

| Account class | Recommended base role | Scope |
|---|---|---|
| Program automation account | `{ProjectCode} - Automation Service` (custom) | Business Unit level; specific tables with Create/Read/Write only — no Delete unless required |
| Pipeline deployment account | `{ProjectCode} - Deployment Service` (custom) | System Customizer equivalent for import operations; no runtime data access |
| CoE platform account | Scoped admin role | Power Platform Admin operations; no access to program data tables |
| Integration account (SharePoint) | No Dataverse role needed | SharePoint permissions only; Dataverse access through flow context |

### Power Platform admin roles

Service accounts that require Power Platform Admin for pipeline operations should use that role **only for the operations the pipeline performs**. Document the specific admin API calls made by the pipeline — this is your ATO evidence for least privilege (AC-6).

Where possible, replace Power Platform Admin with the more scoped **Environment Admin** role scoped to specific environments. A pipeline that deploys to `{ORG}-{CMD}-{PGM}-PROD` only needs Environment Admin on that one environment — not Power Platform Admin on the entire tenant.

---

## Credential rotation and lifecycle management

### Rotation schedule

| Account type | Password rotation | Notes |
|---|---|---|
| Program service account (password auth) | 90 days minimum | DoD baseline; some programs require 60 days |
| Platform service account (password auth) | 90 days minimum | Coordinate rotation with all dependent flows and connections |
| Pipeline service account (client secret) | 90 days or at expiry | Set calendar reminder 30 days before expiry; secret expiry causes pipeline failure |
| Certificate-based authentication | Annual or per certificate validity period | Preferred over passwords for pipeline accounts |

{: .warning }
> Service account password expiry causes silent failures in Power Automate flows — flows fail with a connection error, not an authentication error. This is a common production issue that goes undetected until users report it. Set monitoring alerts on flow failure rates and connection health in CoE Starter Kit dashboards.

### Rotation procedure

When rotating a service account password or client secret:

1. Generate the new credential in Entra ID (GCC High: [entra.microsoft.us](https://entra.microsoft.us))
2. Store the new credential in Azure Government Key Vault **before** changing any connections
3. Update all Power Platform connection references to the service account — in each affected environment
4. Update any ADO service connections or GitHub Actions secrets that reference the account
5. Validate that all dependent flows run successfully in a non-production environment first
6. Log the rotation date and approver in the service account register
7. Revoke the old credential only after validation is complete

### Client secrets vs passwords

For pipeline service accounts and service principals, use **client secrets or certificates** rather than passwords where the integration supports it. Client secrets:
- Have explicit expiry dates (visible in Entra ID — no surprise lockouts)
- Can be rotated without changing the username
- Are compatible with managed identity patterns where supported

For Power Platform connection references, standard user accounts with passwords are currently the most compatible approach. Managed identity support in Power Platform is expanding — evaluate it for new environments.

---

## Service principals vs user accounts

Not every automation scenario requires a licensed user account. Understand when each applies.

| Scenario | Use service principal | Use service account (user) | Notes |
|---|---|---|---|
| Power Platform Admin API calls | Yes | Yes | Both work; service principal requires app registration |
| Power Automate flows (connections) | **No** — not supported | **Yes** | Flows require a licensed user account for connections |
| PAC CLI in pipelines | Yes (app ID + secret) | Yes | Service principal preferred for pipelines |
| Azure Government resource management | Yes | Avoid | Azure RBAC is designed for service principals |
| SharePoint connections in flows | **No** | **Yes** | SharePoint connector requires a user account |
| Dataverse connector in flows | **No** | **Yes** | Dataverse connector requires a licensed user account |
| Custom connector (HTTP + Azure AD) | Yes (managed identity or app reg) | Acceptable | Managed identity preferred where supported |

### App registrations for pipeline service principals

When using service principals for PAC CLI or Admin API access in pipelines:

1. Register an application in Entra ID (GCC High: `entra.microsoft.us`)
2. Grant the application the **Dynamics CRM user_impersonation** permission (required for Dataverse operations)
3. Add the application user to the Dataverse environment with the appropriate security role
4. Store the client secret in Azure Government Key Vault or ADO secret variables — never in pipeline YAML

```powershell
# Authenticate PAC CLI with service principal
pac auth create \
  --applicationId <app-id> \
  --clientSecret <secret> \
  --tenant <tenant-id> \
  --cloud UsGov
```

---

## Environment-specific access strategy

Service accounts should not hold cross-environment access unless their function explicitly requires it. The pipeline deployment account is the exception — it needs access to both source and target environments to move solutions.

| Environment tier | Service account access pattern |
|---|---|
| **Sandbox / Individual Dev** | No service account needed — maker's personal account is acceptable |
| **Program Dev** | Program automation service account with Environment Maker or custom role |
| **Program Test** | Same program account as Dev; separate connection references pointing to Test resources |
| **Program Prod** | Same program account; access confirmed through pipeline — no direct human access to prod service account credentials |
| **Platform / Shared Services** | Platform service account only; no program service accounts have access |
| **IL5 environments** | Service account licensed, provisioned, and documented within the IL5 boundary; credentials stored in IL5-authorized secret store |

---

## CI/CD and automation account considerations

Pipeline service accounts are a distinct class with their own governance requirements.

### ADO service connections

Azure DevOps service connections store the credentials that pipelines use to connect to Power Platform, Azure Government, and other systems. These are not visible as plain text in ADO, but they are a credential store that must be governed.

- Use **service principal** authentication for ADO service connections where supported (Power Platform pipelines support this via the Power Platform Build Tools extension)
- Rotate service connection secrets on the same 90-day schedule as other credentials
- Restrict service connection usage to specific pipelines — do not allow all pipelines in a project to consume a production service connection
- Audit service connection usage quarterly: ADO → Project Settings → Service connections → `...` → Security

### GitHub Actions secrets

If using GitHub Actions for deployment:
- Store credentials as **repository secrets** or **environment secrets** — never as plain text in workflow YAML
- Use environment secrets (scoped to `production`, `test` environments) rather than repository-wide secrets for production credentials
- Enable required reviewers on the `production` GitHub environment to require human approval before a workflow can access production secrets

### Preventing credential exposure in pipeline logs

Pipelines that log their inputs or outputs can inadvertently expose credentials. Enforce these practices:

```yaml
# In ADO YAML — mask sensitive variables
variables:
  - name: ServiceAccountPassword
    value: $(service-account-password)  # Sourced from variable group, marked secret
    # Secret variables are automatically masked in logs
```

Never pass credentials as direct flow inputs or environment variable values that appear in solution export artifacts.

---

## Ownership and accountability

Every service account must have a documented owner — a named individual (not a team alias) who is responsible for its lifecycle.

### Service account register

Maintain a service account register as part of your platform documentation. Minimum fields:

| Field | Description |
|---|---|
| Account UPN | Full email address |
| Account class | Platform / Program / Pipeline / Integration |
| Owning program | Program name and ATO boundary |
| Primary owner | Named individual (not a shared alias) |
| Secondary owner | Backup for continuity |
| Environments with access | List all environments the account has access to |
| Roles assigned | All Dataverse security roles and Power Platform admin roles |
| Last password rotation | Date |
| Next rotation due | Date |
| License assigned | License type and assignment date |
| Review status | Last quarterly review date and reviewer |

### Ownership transfer on personnel departure

When a service account owner PCS's or departs:
1. Identify all service accounts where the departing individual is primary owner
2. Assign a new primary owner before the departure date — not after
3. Validate that the departing individual has no personal credentials for the account stored locally
4. Log the ownership transfer in the service account register
5. Confirm with the new owner that they have tested access and understand the rotation schedule

---

## Audit logging and monitoring

Service account activity must be auditable for ATO compliance. The following must be in place for any service account operating in a production or IL5 environment.

### Entra ID audit logs

All sign-ins by service accounts are logged in Entra ID. Route these to Azure Government Monitor or a SIEM:

- **Sign-in logs**: Every authentication by the service account, including failed attempts
- **Audit logs**: Password changes, role assignments, group membership changes
- **Conditional Access logs**: CA policy evaluation results for each sign-in

Retention: minimum 90 days in Entra ID; route to Log Analytics workspace in Azure Government for 1-year retention minimum (NARA schedule 2).

### Power Platform audit logs

Power Platform audit logging must be enabled at the environment level for any environment where a service account operates:

- Admin Center → Environment → Settings → Auditing → Enable auditing
- Audit logs capture: record create/read/update/delete operations by the service account
- Route to Azure Government Log Analytics via the Dataverse audit log export or CoE Starter Kit audit components

### Monitoring alerts

Configure alerts for service account anomalies:

| Condition | Alert target | Action |
|---|---|---|
| Failed sign-in > 3 in 1 hour | Service account | Notify ISSO; investigate immediately |
| Sign-in from unexpected IP or location | Service account | Notify ISSO; potential credential compromise |
| Flow failure rate spike (> 20% in 1 hour) | Program flows | Investigate connection health; may indicate expired credential |
| Role assignment change | Service account | Notify platform lead; confirm it was authorized |
| Password not rotated within 95 days | Service account | Escalate to account owner; block account if no response within 5 days |

---

## GCC High and IL5 specific requirements

### Licensing in GCC High

Service accounts must hold an appropriate Power Platform license. Unlicensed service accounts in GCC High will fail at runtime with a licensing error that is difficult to diagnose.

- Power Automate Premium (or Power Platform per-user plan with attended RPA) is required for flows using premium connectors
- The license must be assigned in the GCC High tenant admin center (`gcc.admin.microsoft.us`) — commercial license assignments do not carry over
- For IL5 environments, confirm license types are authorized at IL5 before assigning

### Conditional Access for service accounts

GCC High Conditional Access policies must explicitly account for service accounts. A blanket "require MFA" policy that applies to service accounts will break unattended flows.

Recommended approach:
- Create a **named location** for Azure Government service IPs (the IPs from which flows run)
- Create a **Conditional Access exclusion** for service accounts scoped to that named location
- Require **certificate-based authentication** or client secret authentication instead of MFA for service accounts
- Document the CA exclusion in the ATO as a compensating control with the rationale

### IL5 secret storage

For environments authorized at IL5, service account credentials must be stored in a secret store that is authorized for IL5 data:

- **Azure Government Key Vault** in a subscription with IL5 authorization is the recommended store
- Document the Key Vault resource in the ATO as the authoritative credential store
- Access to Key Vault must itself be controlled by RBAC — the service account should have `Key Vault Secrets User` only, not `Key Vault Contributor`

---

## Operational sustainment and personnel turnover

The most common service account failure mode in DoD organizations is not a security breach — it is an undocumented account whose owner departed and whose credentials no longer rotate.

### Quarterly service account review

Every quarter, the platform CoE lead (or designated ISSO) reviews the service account register:

1. Confirm every account has a current primary and secondary owner
2. Confirm rotation dates are current — flag any account past rotation due date
3. Confirm all environments listed in the register still exist and are active — remove access for decommissioned environments
4. Confirm license assignments are current
5. Confirm no service accounts are members of groups they should not be in
6. Sign off on the review — the signed register is ATO evidence for AC-2 (Account Management)

### Annual access recertification

Once per year, the ISSO or AO reviews and formally recertifies all service account access. This is typically tied to the ATO annual review cycle. The service account register is the primary document reviewed. Any account without a clear business justification is disabled during the recertification period.

### Off-boarding triggers

Service account governance must be triggered automatically by off-boarding events:

| Trigger | Action | Timeline |
|---|---|---|
| Primary owner departure confirmed | Assign new owner; verify credentials are not stored on departing individual's device | Before departure date |
| Program ATO lapses or expires | Disable program service accounts; document in environment register | Within 30 days of ATO lapse |
| Environment decommissioned | Remove service account access to decommissioned environment; update register | Within 7 days of decommission |
| Program ends | Disable all program service accounts; preserve audit logs per retention schedule | Within 30 days of program closure |
