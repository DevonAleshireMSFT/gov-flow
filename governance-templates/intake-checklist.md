---
layout: default
title: Environment Request Checklist
parent: Governance Templates
nav_order: 3
permalink: /governance-templates/intake-checklist/
---

# Environment Request Checklist
{: .no_toc }

**Template version:** 1.0 — May 2026
{: .text-delta }

This checklist defines the fields, validation rules, and routing logic for the CoE environment request intake application. Implement these as a Power Apps form in the `PLATFORM-COE-ADMIN` environment.

{: .note }
> The intake application should be built in Power Apps with a Power Automate approval flow backend. This document defines the logical requirements. Do not implement intake via email — email-based intake does not scale and does not produce the audit trail required for ATO evidence.

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Request form fields

### Requestor information

| Field | Type | Required | Validation |
|---|---|---|---|
| Requestor Name | Text | Yes | Auto-populate from Entra ID |
| Requestor DoD ID | Text | Yes | Auto-populate from Entra ID |
| Requestor Email | Text | Yes | Auto-populate from Entra ID |
| Requestor Command | Choice | Yes | Dropdown from approved command list |
| Manager Name | Text | Yes | Manual entry; used for manager approval notification |
| Manager Email | Text | Yes | Manual entry; email validation |

### Program information

| Field | Type | Required | Validation |
|---|---|---|---|
| Program Name | Text | Yes | 3–40 characters |
| Program Code | Text | Yes | 4–8 uppercase alphanumeric; uniqueness check against existing environment register |
| Solution Prefix | Text | Yes | 4–6 lowercase alphabetic only; uniqueness check against existing publishers in tenant |
| ADO Project Name | Text | Yes (LP-ALM programs) | Must match `{CMD}-{PGM}` pattern |
| Program Description | Multiline | Yes | 50–500 characters |
| Estimated User Count | Number | Yes | Must be > 0 |
| Estimated Go-Live Date | Date | Yes | Must be > 30 days from request date |

### Environment details

| Field | Type | Required | Validation |
|---|---|---|---|
| Requested Environment Types | Multi-select | Yes | Options: Sandbox, Dev, Test, Prod (must include Dev if Test is selected; must include Test if Prod is selected) |
| Purpose / Use Case | Multiline | Yes | 50–500 characters |
| Requested Environment Names | Text (generated) | Yes | Auto-generated from `{ORG}-{CMD}-{PGM}-{TYPE}` standard; displayed for requestor to confirm |

### Data classification and security

| Field | Type | Required | Validation |
|---|---|---|---|
| Data Classification | Choice | Yes | Options: Unclassified Non-CUI, CUI, IL5 |
| IL5 Justification | Multiline | If IL5 selected | Required when IL5 selected; minimum 100 characters |
| PII Present | Yes/No | Yes | If Yes: display reminder about field security profile requirement |
| External Integrations | Multiline | No | List any external systems the solution will call; used for DLP review |
| Application Classification | Choice | Yes | Options: Mission Critical, Business Critical, Standard, Citizen / Low-Code (see [Enterprise Strategy Section 4.2](../../enterprise-strategy/#42-application-classification-model)) |

### Ownership and ATO

| Field | Type | Required | Validation |
|---|---|---|---|
| Human Environment Owner | Text | Yes | Name + DoD ID; this person is accountable — not a team |
| Owner Email | Email | Yes | Must be in the organization's GCC High Entra ID |
| ISSO Name | Text | Required for Business Critical+ | Name + contact |
| ISSO Email | Email | Required for Business Critical+ | Must be in the organization's Entra ID |
| ATO Documentation Attached | File / Link | Required for IL5 | Link to ATO artifacts or ISSO acknowledgment |

### Service account / pipeline

| Field | Type | Required | Validation |
|---|---|---|---|
| Pipeline Service Principal Needed | Yes/No | Yes for LP-ALM | If Yes: CoE will provision; requestor provides Azure Government AD app registration |
| Connection Reference Accounts Available | Yes/No | Yes for LP-ALM programs | If No: add note to track with IAM; flag for ISSO review |
| Licensing Capacity Confirmed | Yes/No | Yes | Checkbox; CoE validates against capacity dashboard |

---

## Automated validation rules

Run these checks immediately on form submission, before routing to any reviewer.

| Check | Failure action |
|---|---|
| Environment name uniqueness (generated name not already in register) | Block submission; display conflicting name |
| Solution prefix uniqueness (no existing publisher in tenant with same prefix) | Block submission; display conflicting prefix |
| Data classification / app class consistency: Citizen class cannot be IL5 | Block submission; display error |
| Sandbox request has no ATO, no external integrations | Auto-approve path confirmed |
| Prod request without Test request in same submission or existing | Block submission; require Test environment first |
| LP-ALM programs: ADO project name provided | Block submission if missing |
| IL5 request: ISSO field populated | Block submission if missing |
| Estimated user count > 500 without Business Critical or Mission Critical classification | Display warning (not block): "Applications with >500 users should consider Business Critical classification" |
| Connection Reference Accounts = No for Test or Prod | Display warning: "Test and Prod require non-personal connection reference accounts. Ensure IAM request is in progress before provisioning." |

---

## Routing logic

```
Submission passes automated validation
        ↓
Is this a Sandbox request only?
  YES → Auto-approve → Auto-provision within 1 hour → Notify requestor
  NO  → Continue ↓

Is this Mission Critical or IL5?
  YES → Route to ISSO review (5 business day SLA)
        AND Platform CoE Architecture Review (5 business day SLA, concurrent)
        AND PGB approval (next meeting or async within 5 days)
  NO  → Continue ↓

Is this Business Critical?
  YES → Route to Platform CoE Architecture Review (5 business day SLA)
        AND ISSO notification (no approval required unless IL5)
  NO  → Continue ↓

Is this Standard or Citizen?
  YES → Route to Platform CoE Architecture Review (2 business day SLA)
        → Auto-approve if no concerns raised within SLA
```

---

## Provisioning actions (on approval)

The following are automated actions triggered by approval. Implement via Power Automate calling the Power Platform Admin API.

| Action | Automated | Notes |
|---|---|---|
| Create environment via Admin API | Yes | Use approved environment name; set region to US Gov |
| Apply Managed Environments settings | Yes | Enable for all non-sandbox environments |
| Apply DLP policy assignment | Yes | Assign to appropriate environment group based on data classification |
| Create Entra ID security groups | Yes | `PP-{ENV}-Owners`, `PP-{ENV}-Users`, `PP-{ENV}-Developers`, `PP-{ENV}-Support` |
| Register in environment register (Dataverse) | Yes | Populate all required fields from the intake form |
| Create ADO project | Yes (if LP-ALM) | Create project, initialize repo from LP-ALM template |
| Create ADO variable groups | Yes (if LP-ALM) | `{PREFIX}-Common`, `{PREFIX}-Test`, `{PREFIX}-Prod` (secrets empty — require manual population) |
| Notify requestor with onboarding checklist | Yes | Send onboarding email with links to LP-ALM onboarding, environment URLs, Entra group names |
| Create calendar reminder for owner confirmation | Yes | Set 90-day quarterly review reminder |
| Sandbox: set expiry date and auto-quarantine workflow | Yes | 90-day default; configurable up to 90 days |

{: .warning }
> ADO variable group **secrets** (client secrets, tenant IDs) are never auto-populated. After ADO project creation, a CoE engineer must manually populate the secret-flagged variables. This is a deliberate security control.

---

## Post-provisioning owner checklist

Send this to the environment owner on provisioning:

- [ ] Log in to the environment and confirm access: `{ENV_URL}`
- [ ] Review the Entra ID groups created: `PP-{ENV}-Owners`, `PP-{ENV}-Users`
- [ ] Assign initial users to appropriate Entra groups
- [ ] Complete [LP-ALM onboarding Phase 1](https://devonaleshiremsft.github.io/layered-platform-alm/onboarding/) (publisher setup, solution shells, repository initialization)
- [ ] Register app registration in Azure Government (`portal.azure.us`) for the pipeline service principal
- [ ] Create application user in the provisioned environment and assign System Administrator role to the service principal
- [ ] Populate ADO variable group secrets with the CoE engineer
- [ ] Apply `_Config` for Dev environment (see [LP-ALM Section 6.3](https://devonaleshiremsft.github.io/layered-platform-alm/methodology/#63-handling-the-_config-layer))
- [ ] Confirm data classification is correctly documented in the environment register
- [ ] Schedule first monthly touchpoint with Platform CoE (new programs: monthly for 6 months)

---

## Intake SLA targets

| Request Type | Target Turnaround | Measurement |
|---|---|---|
| Sandbox only | < 1 hour (automated) | Time from submission to environment active |
| Standard / Citizen (no IL5) | < 2 business days | Time from submission to CoE decision |
| Business Critical | < 5 business days | Time from submission to CoE decision |
| IL5 or Mission Critical | < 10 business days | Time from submission to PGB decision |

The platform CoE must report intake SLA compliance monthly to the PGB. Target: **> 95% of requests within SLA**.
