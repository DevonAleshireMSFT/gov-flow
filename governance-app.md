---
layout: default
title: "Platform Governance App"
nav_order: 14
---

# GovFlow Platform Governance App
{: .no_toc }

Requirements and technical reference for building the GovFlow companion governance application in Power Platform and Dataverse.
{: .fs-5 .fw-300 }

{: .label .label-purple }
Technical Reference

<details open markdown="block">
  <summary>Table of contents</summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

---

## Overview

The GovFlow Platform Governance App is a purpose-built Power Platform application that operationalizes the GovFlow framework. Where GovFlow defines *what* governance looks like, this app *implements* it — providing structured workflows for environment requests, application approvals, connector governance, service account lifecycle, and platform reporting in a single governed, ATO-supportable data model.

It is designed for GCC High and DoD tenants operating at enterprise scale. It is not a replacement for the CoE Starter Kit — it is the governance workflow layer that the CoE Starter Kit does not provide.

### What it is not

The Microsoft Power Platform Center of Excellence (CoE) Starter Kit provides environment inventory, app telemetry, and basic compliance flows. It is a community sample that requires significant customization before production deployment and has known GCC High connector compatibility limitations. The GovFlow Platform Governance App does not duplicate CoE Starter Kit telemetry functions — it consumes that data and adds the governance decision workflow above it.

| CoE Starter Kit | GovFlow Platform Governance App |
|---|---|
| Environment inventory and sync | Environment request, approval, and provisioning |
| App usage telemetry | Application approval and org-wide deployment governance |
| DLP policy reporting | DLP exception requests and connector approval workflow |
| Maker compliance process | ISSO review gates and ATO-linked governance records |
| Community sample | Purpose-built for GovFlow-aligned government organizations |

### Marketing positioning

> *GovFlow tells you what to build. The GovFlow Platform Governance App builds it.*

Deploy it alongside the GovFlow framework to give Platform CoE teams an operational governance system from day one — no custom-built intake apps, no email-based approvals, no manual environment registers. Governance workflows, audit trails, and ATO evidence are produced by the system, not assembled after the fact.

---

## Application Architecture

### Three-App Model

One Dataverse environment (`COE-ADMIN`), one shared data model, three purpose-built apps serving distinct audiences.

| App | Type | Primary Audience | Description |
|---|---|---|---|
| **GovFlow Request Portal** | Canvas App | Developers, Program Managers | Guided request submission for environments, app deployments, connector exceptions, and service account provisioning |
| **GovFlow Admin Portal** | Model-Driven App | Platform CoE, ISSOs, Approvers | Full governance administration — review queues, environment register, connector catalog, exception management, approval workflow |
| **GovFlow Governance Dashboard** | Power BI (embedded) | Leadership, ISSO, CoE | Read-only reporting — environment health, ATO status, DLP violation trends, request SLA tracking |

**Why Canvas for requestors:** Developers and program managers are occasional users who need a clean, guided, form-based experience. Canvas delivers accessible UX without requiring Dataverse knowledge.

**Why Model-Driven for admins:** Platform CoE staff and ISSOs work in the system daily. They need views, advanced filters, timeline activity, business process flows, and the full Dataverse feature set. Model-driven apps are the correct tool for power users operating a governance system.

**Why three apps:** A single app serving all audiences produces a bloated, confusing UX. The three-app pattern is the recommended Power Platform architecture for systems with divergent user personas.

### Deployment Environment

All three apps deploy to the `COE-ADMIN` environment — the Platform CoE's dedicated administrative environment.

```
COE-ADMIN Environment
├── GovFlow Request Portal (Canvas App)
├── GovFlow Admin Portal (Model-Driven App)
├── GovFlow Governance Dashboard (Power BI embedded)
├── Dataverse (shared data model — all tables)
├── Power Automate flows (approval notifications, provisioning automation)
└── Service Principal (pipeline deployment only)
```

The `COE-ADMIN` environment is itself governed by GovFlow principles — Managed Environment enabled, DLP policy applied, service principal for pipeline, no manual production deployments.

### Solution Structure (LP-ALM Aligned)

| Solution | Contents |
|---|---|
| `GOVFLOW-Core` | Tables, columns, relationships, choice columns, business rules |
| `GOVFLOW-Config` | Environment variables, connection references, security roles |
| `GOVFLOW-RequestPortal` | GovFlow Request Portal canvas app |
| `GOVFLOW-AdminPortal` | GovFlow Admin Portal model-driven app, site maps, views, forms |
| `GOVFLOW-Flows` | All Power Automate approval and notification flows |
| `GOVFLOW-Dashboard` | Power BI report definition and embedded canvas wrapper |

Follow [LP-ALM Methodology](https://devonaleshiremsft.github.io/layered-platform-alm/) for pipeline YAML, environment variable management, and deployment gate configuration.

---

## User Roles and Personas

### Personas

| Persona | App Used | Frequency | Responsibilities |
|---|---|---|---|
| **Developer / Maker** | Request Portal | Occasional | Submit environment requests, connector requests, service account requests; track status |
| **Program Manager / PM** | Request Portal | Occasional | Submit app approval requests; track deployment status; review onboarding checklist |
| **Platform CoE Engineer** | Admin Portal | Daily | Review and process requests; manage environment register; manage connector catalog |
| **ISSO** | Admin Portal | Weekly | Security review of DLP exceptions, connector approvals, IL5 environment requests |
| **CoE Lead / Platform Architect** | Admin Portal + Dashboard | Weekly | Exception approvals; connector governance board; capacity planning |
| **Leadership / Commanding Officer** | Dashboard | Monthly | Environment health, ATO status, request SLA compliance |

### Security Roles (Dataverse)

| Role | Description |
|---|---|
| `GovFlow Administrator` | Full read/write across all tables; manage connector catalog; process all request types |
| `GovFlow ISSO Reviewer` | Read all request records; write on security review and ISSO approval fields only |
| `GovFlow Requestor` | Create new requests; read own records only; no access to other users' requests or admin tables |
| `GovFlow Reader` | Read-only across all tables; no create/edit; for dashboard and reporting consumers |
| `GovFlow CoE Lead` | Read all requests; approve/deny final escalations; manage exceptions |

---

## Functional Requirements

### Environment Request Management

| ID | Requirement | Priority |
|---|---|---|
| ENV-001 | Requestor can submit a new environment request via the Request Portal with the following fields: Program Name, Command, Environment Type, Data Classification, Business Justification, ATO Reference (or ATO Not Yet Obtained), Requested Environment Name (validated against `{CMD}-{PGM}-{TYPE}` pattern), Named Environment Owner | Must Have |
| ENV-002 | System validates environment name against the GovFlow naming standard (`{COMMAND}-{PROGRAM}-{TYPE}`) before submission is accepted. Invalid names are rejected with an inline error message explaining the pattern. | Must Have |
| ENV-003 | Submitted requests enter a review queue in the Admin Portal with status: Submitted → CoE Review → ISSO Review (if IL5) → Approved / Denied | Must Have |
| ENV-004 | For IL5-designated environments, ISSO review is mandatory and cannot be bypassed. The workflow enforces this gate. | Must Have |
| ENV-005 | Approved environments are registered in the Environment Register table with all request metadata captured | Must Have |
| ENV-006 | Approval or denial triggers an automated notification to the requestor with outcome, reviewer name, and (for denials) reason | Must Have |
| ENV-007 | All environment requests are retained indefinitely as governance records. Denied requests are retained with denial reason. | Must Have |
| ENV-008 | Platform CoE can initiate environment provisioning directly from an approved request record (Power Platform Admin API integration — see INT-002) | Should Have |
| ENV-009 | Requestor can view the current status of all their submitted requests from the Request Portal | Must Have |
| ENV-010 | Admin Portal provides a prioritized review queue filtered by status, request type, and date submitted | Must Have |
| ENV-011 | Request records include a full activity timeline — every status change, comment, and approval action is logged with timestamp and user | Must Have |
| ENV-012 | Environments marked as Sandbox type automatically set a 90-day expiry date. Expiry notification is sent to the owner 30 days prior. | Must Have |

### Application Approval Workflow

| ID | Requirement | Priority |
|---|---|---|
| APP-001 | Program Manager can submit an application approval request for a canvas app, model-driven app, or Power Pages site, specifying: Application Name, Application Type, Target Environment, Data Classification, ATO Coverage Reference, LP-ALM Pipeline Confirmed (Y/N), Support Model Established (Y/N), Named Application Owner | Must Have |
| APP-002 | Applications requesting org-wide or cross-command deployment (enterprise environment) require an extended approval checklist: ISSO review, data flow documentation, security role validation, group assignment confirmation, rollout plan | Must Have |
| APP-003 | Approved applications are registered in the Application Catalog table with environment, classification level, ATO reference, and deployment date | Must Have |
| APP-004 | Application records track deployment history — each version approved for production creates a deployment record with artifact version, approver, and date | Should Have |
| APP-005 | Applications without confirmed LP-ALM pipeline cannot be approved for production deployment. The workflow enforces this as a hard gate, not a soft warning. | Must Have |
| APP-006 | Admin Portal provides an Application Catalog view — all registered apps with status, environment, classification, and owner | Must Have |
| APP-007 | Applications approaching ATO expiry (configurable threshold, default 60 days) trigger an automated alert to the application owner and ISSO | Should Have |

### DLP and Connector Governance

| ID | Requirement | Priority |
|---|---|---|
| DLP-001 | Requestor can submit a connector approval request specifying: Connector Name, Connector Type (Pre-built / Premium / Third-Party / Custom / HTTP), Target Environment or Environment Group, Business Justification, Data Classification of Data to Be Transmitted, Authentication Method | Must Have |
| DLP-002 | Custom connector requests require additional fields: API Endpoint, OpenAPI Specification attached, Authentication Method, API Owner Contact, APIM fronted (Y/N), Data Residency Confirmation | Must Have |
| DLP-003 | HTTP trigger / inbound endpoint requests are flagged with an automatic warning and require ISSO approval as a mandatory gate | Must Have |
| DLP-004 | Approved connectors are added to the Connector Catalog table with: Connector Name, Type, Approval Scope (tenant-wide / environment group / specific environment), Data Classification Max, FedRAMP Authorization status, Approved Date, Next Review Date | Must Have |
| DLP-005 | Connector exceptions (approved outside standard tier) are recorded in the Exception Register with expiry date and 90-day review cycle. System sends automated renewal reminder 30 days before expiry. | Must Have |
| DLP-006 | Admin Portal provides a Connector Catalog view — all approved connectors with approval scope, classification level, and review date | Must Have |
| DLP-007 | Admin Portal provides an Exception Register view — all active exceptions with expiry dates and approval records | Must Have |
| DLP-008 | New connectors added to the Power Platform catalog (detected via Power Platform Admin API or manual notification) are automatically added to a "Pending Review" list in the Admin Portal | Should Have |
| DLP-009 | Weekly DLP violation report (sourced from CoE Starter Kit or Admin API) is surfaced in Admin Portal — violations by environment, connector, and user | Should Have |

### Service Account and Service Principal Lifecycle

| ID | Requirement | Priority |
|---|---|---|
| SVC-001 | Platform CoE can register a service principal in the Service Principal Register, capturing: Display Name, App Registration ID, Client ID, Environment(s) Used, Purpose, Technical Owner, Provisioning Date, Rotation Schedule | Must Have |
| SVC-002 | Service principals approaching credential rotation date (configurable threshold, default 30 days) trigger automated notification to the technical owner | Must Have |
| SVC-003 | Service principal records include a decommission workflow — when an environment is decommissioned, associated service principals are flagged for credential revocation within 24 hours | Must Have |
| SVC-004 | Admin Portal provides a Service Principal Register view with rotation status, environment associations, and last review date | Must Have |
| SVC-005 | Service principal records are linked to their associated environment records — navigating from an environment record shows all associated service principals | Must Have |

### Environment Register

| ID | Requirement | Priority |
|---|---|---|
| REG-001 | Every approved and provisioned environment has a corresponding Environment Register record that captures: Environment Name, Display Name, Environment ID (Power Platform GUID), Type, Data Classification, ATO Reference, Owner (user lookup), Service Principal (linked record), DLP Policy Tier, Managed Environment (Y/N), Creation Date, Last Review Date, Status | Must Have |
| REG-002 | Environment Register is the single source of truth for the environment inventory. It is not a manual spreadsheet — it is a Dataverse table updated by governance workflows. | Must Have |
| REG-003 | Environment records include a child relationship to all associated: Application records, Service Principal records, Connector Exception records, DLP policy tier | Must Have |
| REG-004 | Environment status lifecycle: Requested → Approved → Provisioned → Active → Under Review → Decommissioned | Must Have |
| REG-005 | Decommissioning an environment triggers a checklist workflow: solution export confirmed, data disposal documented, service principals flagged for revocation, owner notified, record archived | Must Have |
| REG-006 | Admin Portal provides an Environment Register view filterable by type, classification, status, command, and program | Must Have |
| REG-007 | Environment register data is exportable as CSV for ATO evidence submission | Must Have |

### Reporting and Telemetry

| ID | Requirement | Priority |
|---|---|---|
| RPT-001 | Governance Dashboard displays: total environment count by type, request queue depth, average request-to-approval SLA by request type, DLP violation trend (7/30/90 day), exceptions approaching expiry, ATO expiry status | Must Have |
| RPT-002 | Dashboard includes an environment health matrix — each environment mapped to: Managed Environment status, DLP policy tier, ATO status, last review date | Should Have |
| RPT-003 | All request types include SLA tracking — target resolution time is configurable; overdue requests are highlighted in Admin Portal review queue | Should Have |
| RPT-004 | Monthly governance report exportable from Admin Portal: environment inventory snapshot, new environments provisioned, requests processed, exceptions active, DLP violations summary | Should Have |
| RPT-005 | ATO evidence export — from any environment record, generate a structured data export of: security role assignments, DLP policy tier, service principals, connector approvals, and deployment history for ATO documentation | Should Have |

---

## Non-Functional Requirements

### Performance

| ID | Requirement |
|---|---|
| NFR-001 | Request Portal must load within 3 seconds on first launch for users on standard government network connections |
| NFR-002 | Admin Portal views must return filtered results within 5 seconds for data sets up to 10,000 records |
| NFR-003 | Approval notification flows must trigger within 2 minutes of a status change |
| NFR-004 | The system must support up to 500 concurrent requestors and 50 concurrent Admin Portal users without degradation |

### Availability and Resilience

| ID | Requirement |
|---|---|
| NFR-005 | The system relies on Power Platform service availability in GCC High; no custom infrastructure availability SLA is imposed beyond the Power Platform SLA |
| NFR-006 | All approval workflows use Power Automate with retry logic — transient failures do not drop notifications or fail request records silently |
| NFR-007 | Dataverse is the system of record; no data is stored exclusively in flow run history or email |

### GCC High and IL5 Compliance

| ID | Requirement |
|---|---|
| NFR-008 | All components deploy to GCC High (or DoD) tenant only. No commercial cloud connectors used in the solution. |
| NFR-009 | All connector usage must be validated against the GCC High connector availability list before inclusion in the solution |
| NFR-010 | The solution must be deployable as managed solutions only — no unmanaged component deployment to production |
| NFR-011 | No sensitive data (credentials, ATO artifact content, SSNs, classified data) is stored in Dataverse. ATO references are document references only — not document storage. |
| NFR-012 | All Dataverse tables used in this solution enable auditing. Audit logs are retained per organizational data retention policy. |

---

## Data Model

All tables use the `govflow_` publisher prefix. Column names use the `govflow_` prefix. Table display names use "GovFlow" prefix for clarity in the Admin Portal.

### Core Tables

#### `govflow_environment` — Environment Register

| Column | Type | Notes |
|---|---|---|
| `govflow_name` | Text (PK display) | Environment display name |
| `govflow_environmentid_pp` | Text | Power Platform environment GUID |
| `govflow_type` | Choice | Developer, Sandbox, Test, UAT, Production, Enterprise, COE-Admin |
| `govflow_dataclassification` | Choice | Unclassified, CUI, IL4, IL5 |
| `govflow_commandcode` | Text | e.g., FORSCOM |
| `govflow_programcode` | Text | e.g., SYSTRK |
| `govflow_status` | Choice | Requested, Approved, Provisioned, Active, Under Review, Decommissioned |
| `govflow_owner` | Lookup → SystemUser | Environment owner |
| `govflow_atoreference` | Text | ATO document reference number |
| `govflow_managedenv` | Yes/No | Managed Environment enabled |
| `govflow_dlppolicytier` | Choice | Tenant Default, Dev, Test, Production, Enterprise, Exception |
| `govflow_expirydate` | Date | For Sandbox types — auto-set 90 days from provision date |
| `govflow_createddate` | Date | Provisioning date |
| `govflow_lastreviewdate` | Date | Last governance review |
| `govflow_sourcerequest` | Lookup → govflow_environmentrequest | The request that created this record |

#### `govflow_environmentrequest` — Environment Requests

| Column | Type | Notes |
|---|---|---|
| `govflow_requestnumber` | Auto Number | e.g., ENV-2026-0001 |
| `govflow_requestor` | Lookup → SystemUser | |
| `govflow_programname` | Text | Human-readable program name |
| `govflow_commandcode` | Text | |
| `govflow_programcode` | Text | |
| `govflow_requestedenvironmentname` | Text | Validated against naming standard |
| `govflow_environmenttype` | Choice | |
| `govflow_dataclassification` | Choice | |
| `govflow_atoreference` | Text | |
| `govflow_atoobtained` | Yes/No | ATO in hand or planned |
| `govflow_businessjustification` | Multiline Text | |
| `govflow_status` | Choice | Submitted, CoE Review, ISSO Review, Approved, Denied, Provisioned |
| `govflow_reviewernotes` | Multiline Text | CoE reviewer notes |
| `govflow_issoreviewstatus` | Choice | Not Required, Pending, Approved, Denied |
| `govflow_denialreason` | Multiline Text | |
| `govflow_resolveddate` | Date | |

#### `govflow_application` — Application Catalog

| Column | Type | Notes |
|---|---|---|
| `govflow_applicationname` | Text | |
| `govflow_type` | Choice | Canvas, Model-Driven, Power Pages, Copilot Studio |
| `govflow_environment` | Lookup → govflow_environment | |
| `govflow_programcode` | Text | |
| `govflow_dataclassification` | Choice | |
| `govflow_atoreference` | Text | |
| `govflow_atoexpirydate` | Date | |
| `govflow_owner` | Lookup → SystemUser | |
| `govflow_supportcontact` | Text | |
| `govflow_alpipelineconfirmed` | Yes/No | LP-ALM pipeline in place |
| `govflow_status` | Choice | Registered, Active, Under Review, Decommissioned |
| `govflow_lastdeploymentdate` | Date | |
| `govflow_deploymentenvironment` | Choice | Program-Owned, Enterprise |

#### `govflow_applicationrequest` — Application Approval Requests

| Column | Type | Notes |
|---|---|---|
| `govflow_requestnumber` | Auto Number | e.g., APP-2026-0001 |
| `govflow_requestor` | Lookup → SystemUser | |
| `govflow_applicationname` | Text | |
| `govflow_applicationtype` | Choice | |
| `govflow_targetenvironment` | Lookup → govflow_environment | |
| `govflow_deploymentscope` | Choice | Program, Cross-Command, Org-Wide |
| `govflow_dataclassification` | Choice | |
| `govflow_atoreference` | Text | |
| `govflow_alpipelineconfirmed` | Yes/No | |
| `govflow_supportmodelestablished` | Yes/No | |
| `govflow_dataflowdocumented` | Yes/No | Required for enterprise scope |
| `govflow_securityrolesvalidated` | Yes/No | Required for enterprise scope |
| `govflow_issoreviewstatus` | Choice | Not Required, Pending, Approved, Denied |
| `govflow_status` | Choice | Submitted, CoE Review, ISSO Review, Approved, Denied |

#### `govflow_connectorcatalog` — Approved Connector Catalog

| Column | Type | Notes |
|---|---|---|
| `govflow_connectorname` | Text | |
| `govflow_connectortype` | Choice | Microsoft First-Party, Premium, Third-Party, Custom |
| `govflow_approvalscope` | Choice | Tenant-Wide, Environment Group, Specific Environment |
| `govflow_targetenvironmentgroup` | Text | If scoped to group |
| `govflow_dataclassificationmax` | Choice | Maximum classification approved for |
| `govflow_fedrampstatus` | Choice | Authorized, In Process, Not Authorized, Requires Validation |
| `govflow_approveddate` | Date | |
| `govflow_nextreviewdate` | Date | |
| `govflow_approvedby` | Lookup → SystemUser | |
| `govflow_notes` | Multiline Text | Review notes, boundary caveats |
| `govflow_status` | Choice | Active, Pending Review, Suspended, Revoked |

#### `govflow_connectorrequest` — Connector / DLP Requests

| Column | Type | Notes |
|---|---|---|
| `govflow_requestnumber` | Auto Number | e.g., DLP-2026-0001 |
| `govflow_requestor` | Lookup → SystemUser | |
| `govflow_connectorname` | Text | |
| `govflow_connectortype` | Choice | Pre-Built, Premium, Third-Party, Custom, HTTP |
| `govflow_targetscope` | Choice | Tenant-Wide, Environment Group, Specific Environment |
| `govflow_targetenvironment` | Lookup → govflow_environment | |
| `govflow_businessjustification` | Multiline Text | |
| `govflow_dataclassification` | Choice | |
| `govflow_authenticationmethod` | Text | |
| `govflow_apiendpoint` | Text | Custom/HTTP only |
| `govflow_apimfronted` | Yes/No | Custom only |
| `govflow_dataresidencyconfirmed` | Yes/No | |
| `govflow_issoreviewstatus` | Choice | Not Required, Pending, Approved, Denied |
| `govflow_status` | Choice | Submitted, CoE Review, ISSO Review, Approved, Denied |
| `govflow_isexception` | Yes/No | True if approved outside standard tier |
| `govflow_exceptionexpiry` | Date | If exception — 90 days from approval |

#### `govflow_serviceprincipal` — Service Principal Register

| Column | Type | Notes |
|---|---|---|
| `govflow_displayname` | Text | |
| `govflow_appregistrationid` | Text | Entra ID App Registration ID (GUID) |
| `govflow_clientid` | Text | Client ID |
| `govflow_purpose` | Choice | Pipeline Deployment, Service-to-Service, Dataverse API |
| `govflow_owner` | Lookup → SystemUser | Technical owner |
| `govflow_environment` | Lookup → govflow_environment | Primary environment |
| `govflow_provisioningdate` | Date | |
| `govflow_rotationscheduledays` | Whole Number | Default: 90 |
| `govflow_nextrotationdate` | Date | Calculated from last rotation + schedule |
| `govflow_lastrotationdate` | Date | |
| `govflow_status` | Choice | Active, Rotation Due, Rotation Overdue, Revoked |

#### `govflow_governanceexception` — Exception Register

| Column | Type | Notes |
|---|---|---|
| `govflow_exceptiontype` | Choice | Connector, DLP Policy Tier, Environment Config |
| `govflow_description` | Multiline Text | |
| `govflow_environment` | Lookup → govflow_environment | |
| `govflow_connectorrequest` | Lookup → govflow_connectorrequest | |
| `govflow_approvedby` | Lookup → SystemUser | |
| `govflow_issoapproval` | Lookup → SystemUser | |
| `govflow_approvaldate` | Date | |
| `govflow_expirydate` | Date | |
| `govflow_renewalstatus` | Choice | Active, Renewal Pending, Expired, Closed |
| `govflow_businessjustification` | Multiline Text | |

### Table Relationships

```
govflow_environmentrequest  1──* (creates) ──1  govflow_environment
govflow_environment         1──*            *   govflow_application
govflow_environment         1──*            *   govflow_serviceprincipal
govflow_environment         1──*            *   govflow_connectorrequest
govflow_environment         1──*            *   govflow_governanceexception
govflow_connectorrequest    1──1 (exception)──1  govflow_governanceexception
govflow_applicationrequest  1──1 (creates) ──1  govflow_application
```

---

## Integration Requirements

| ID | Integration | Priority | Notes |
|---|---|---|---|
| INT-001 | **Entra ID** — Lookup and validate user identities in request forms; resolve command/department attributes for auto-populating requestor details | Must Have | Use Office 365 Users connector (GCC High compatible) |
| INT-002 | **Power Platform Admin API** — Query environment inventory on-demand from Admin Portal; validate environment names against existing environments before approval | Should Have | Use Power Platform for Admins connector or HTTP with service principal |
| INT-003 | **Power Automate Approvals** — All governance workflows use Power Automate approval actions for structured approval with audit trail | Must Have | Approvals connector is GCC High compatible |
| INT-004 | **Outlook / Teams notification** — Requestors and approvers receive notifications via Outlook or Teams when request status changes | Must Have | Use Office 365 Outlook or Teams connector; both GCC High compatible |
| INT-005 | **SharePoint** — ATO reference documents optionally linked from environment and application records (document library reference only — no content stored in Dataverse) | Should Have | SharePoint connector GCC High compatible |
| INT-006 | **Azure DevOps / GitHub** — Environment records optionally linked to ADO project; application records linked to ADO pipeline run for deployment traceability | Should Have | ADO connector — validate GCC High availability |
| INT-007 | **CoE Starter Kit Dataverse** — If CoE Starter Kit is deployed, import environment and app telemetry data from CoE tables into GovFlow environment register (scheduled sync flow) | Should Have | Same Dataverse environment or cross-environment lookup |
| INT-008 | **Power BI** — Governance Dashboard connects to Dataverse via Power BI Dataverse connector for near-real-time reporting | Must Have | Dataverse connector supported in GCC High |

---

## Security Requirements

| ID | Requirement |
|---|---|
| SEC-001 | All Dataverse tables implement row-level security. Requestors can only read their own request records. CoE staff read all records. ISSO reads all records but writes only to review/approval fields. |
| SEC-002 | The `govflow_connectorcatalog` table is read-only for all roles except `GovFlow Administrator`. No requestor can modify the approved connector list. |
| SEC-003 | Service principal credentials (client secrets) are never stored in Dataverse. The `govflow_serviceprincipal` table stores the App Registration ID and Client ID only — not secrets. Secrets are managed in Azure Key Vault. |
| SEC-004 | All sensitive fields (ATO references, data classification determinations) are audited. Field-level audit is enabled on classification and ATO fields. |
| SEC-005 | The Admin Portal is restricted to users assigned `GovFlow Administrator`, `GovFlow ISSO Reviewer`, or `GovFlow CoE Lead` roles. It is not shared broadly. |
| SEC-006 | The Request Portal is shared with all licensed Power Platform users in the `COE-ADMIN` environment's access group. |
| SEC-007 | Environment variables are used for all configurable values (approval SLA thresholds, notification email addresses, rotation warning periods). No hardcoded values in flows or apps. |
| SEC-008 | Connection references are used for all connectors. No personal connections are embedded in flows. All connections authenticate via service principal where the connector supports it. |
| SEC-009 | The solution is deployed as a managed solution. Customization of forms, views, and flows in production is blocked for non-administrator roles. |
| SEC-010 | No CUI data is stored in the governance app. Request records contain metadata about systems — not the data those systems process. |

---

## Approval Workflows

### Environment Request Workflow

```
Requestor submits request (Request Portal)
        ↓
Automated validation: naming standard check, required fields present
  → FAIL: Return to requestor with inline error
  → PASS: Request created with status = "Submitted"
        ↓
Notification → CoE Review queue (Admin Portal)
        ↓
CoE Engineer reviews (target: 5 business days)
  → Deny: Status = Denied; denial reason recorded; requestor notified
  → Approve: Continue
        ↓
Is environment type IL5 or data classification IL5?
  → NO: Status = Approved; requestor notified; proceed to provisioning
  → YES: Status = "ISSO Review"; ISSO notified
        ↓
ISSO reviews (target: 5 additional business days)
  → Deny: Status = Denied; requestor notified
  → Approve: Status = Approved; requestor notified
        ↓
CoE Engineer provisions environment
(manual with Admin API assist, or automated via PAC CLI flow)
        ↓
Status = Provisioned; environment register record created
Requestor receives onboarding checklist notification
```

### Connector / DLP Exception Workflow

```
Requestor submits connector request (Request Portal)
        ↓
Is connector type HTTP or inbound trigger?
  → YES: Automatic warning displayed; ISSO review flag set
        ↓
CoE review (target: 5 business days standard, 10 for third-party/custom)
  → Deny: Requestor notified with alternatives
  → Approve standard scope: Added to connector catalog
  → Approve as exception: Exception record created with 90-day expiry
        ↓
Is connector third-party, custom, or HTTP?
  → YES: ISSO review required before catalog addition
        ↓
Policy updated; requestor notified; catalog updated
```

---

## ALM and Deployment Requirements

| ID | Requirement |
|---|---|
| ALM-001 | All six solutions deploy via LP-ALM pipeline. No manual deployment to the COE-ADMIN production environment. |
| ALM-002 | The COE-ADMIN environment has a dedicated ADO project `PLATFORM-GOVFLOWAPP` with pipelines for each solution layer |
| ALM-003 | Environment variables for all configurable values (SLA thresholds, notification targets, rotation warning days) are managed per environment using connection references and environment variable definitions in `GOVFLOW-Config` |
| ALM-004 | All Power Automate flows are solution-aware and deployed as managed — no flows created manually in the COE-ADMIN environment |
| ALM-005 | The canvas app (`GOVFLOW-RequestPortal`) is built with responsive layout for tablet and desktop — government users may access on government-issued devices with varying screen sizes |
| ALM-006 | A `DEV` and `TEST` environment exist for the GovFlow app itself (`COE-DEV`, `COE-TEST`) following the same governance the app enforces for others |
| ALM-007 | Data migration scripts (for populating initial environment register from existing inventory) are maintained as PAC CLI scripts in source control |

---

## Out of Scope (Version 1)

The following capabilities are intentionally deferred to a future version:

- Automated environment provisioning via Power Platform Admin API (approval workflow is V1; automated provisioning hook is V2)
- PAC CLI automated pipeline creation on environment approval
- Direct integration with XACTA, eMASS, or other ATO management systems
- Mobile-optimized canvas app layout
- Multi-tenant / multi-agency support (single-tenant GovFlow architecture assumed)
- Self-service sandbox provisioning without CoE review (governance gate always present in V1)

---

## Relationship to GovFlow Framework

The GovFlow Platform Governance App is the operational implementation of the following GovFlow framework sections:

| App Capability | Framework Reference |
|---|---|
| Environment request and approval | [Enterprise Strategy §2.7 — Environment Request and Approval Process](../enterprise-strategy/#27-environment-request-and-approval-process) |
| Environment register | [Governance Templates — Environment Register](../governance-templates/environment-register/) |
| DLP and connector governance | [DLP Strategy](../dlp-strategy/) |
| Service principal lifecycle | [Service Account Security](../service-account-security/) |
| Application approval for org-wide deployment | [Default Environment — Approval process before broad deployment](../default-environment/#approval-process-before-broad-deployment) |
| Environment access and RBAC | [Environment Access & RBAC](../environment-access/) |

The app does not replace the GovFlow documentation — it implements the workflows that documentation describes. Governance decisions are still made by people; this app structures, records, and enforces those decisions.
