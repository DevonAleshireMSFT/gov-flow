---
layout: default
title: Environment Register
parent: Governance Templates
nav_order: 2
permalink: /governance-templates/environment-register/
---

# Enterprise Environment Register — GCC High / DoD
{: .no_toc }

**Template version:** 1.0 — May 2026 — DoD / GCC High extension
{: .text-delta }

This register is the authoritative inventory of all Power Platform environments in the GCC High tenant. It is maintained by the Platform CoE and updated within 24 hours of any environment provisioning or decommission event.

{: .note }
> For the per-project LP-ALM environment register (a single program's Dev/Test/Prod), see the [LP-ALM Environment Register](https://devonaleshiremsft.github.io/layered-platform-alm/environment-register/) ↗. This document is the tenant-wide inventory maintained by the CoE.

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## How to use this register

1. **One row per environment.** Every environment in the tenant — including CoE environments and sandboxes — has a row.
2. **Update within 24 hours** of provisioning, decommission, or owner change.
3. **Never store secrets here.** Connection reference values, service principal secrets, and `_Config` values are stored in Azure Government Key Vault and the program's config management record — not this document.
4. **Store in SharePoint GCC High or Dataverse.** Do not maintain this in a personal OneDrive or local file. The CoE Starter Kit environment inventory can serve as a live data source for this register.

---

## Register fields reference

| Field | Required | Description |
|---|---|---|
| Environment Name | Yes | Follows `{ORG}-{CMD}-{PGM}-{TYPE}` standard |
| Environment ID | Yes | GUID from Power Platform Admin Center |
| Environment URL | Yes | `https://{orgname}.crm.microsoftdynamics.us` |
| Type | Yes | CoE, Shared Services, Program Dev, Program Test, Program Prod, Sandbox, Individual Dev |
| Data Classification | Yes | Unclassified Non-CUI, CUI, IL5 |
| ATO Status | Yes | Not Required, In Process, Active (expiry date), Lapsed |
| ATO Expiry | If ATO Active | Date ATO expires; trigger review 90 days prior |
| Human Owner | Yes | Name + DoD ID of the accountable individual (not a team name) |
| Owner Command | Yes | Command/org the owner belongs to |
| ISSO | Yes for IL5 | Name of the ISSO responsible for ATO coverage |
| Service Principal | Yes (Test/Prod) | App registration name (not secret) |
| ADO Project | Yes (LP-ALM) | Azure DevOps project name for this program |
| Managed Environment | Yes | Enabled / Disabled |
| Provisioned Date | Yes | ISO 8601 date |
| Expiry Date | Sandbox only | ISO 8601 date; auto-quarantine trigger |
| Last Reviewed | Yes | Date of last quarterly owner confirmation |
| Status | Yes | Active, Quarantined, Decommission Pending, Decommissioned |
| Notes | No | Free text; include links to ATO artifacts, exception approvals |

---

## Platform tier environments

| Environment Name | Environment ID | URL | Type | Classification | ATO Status | Human Owner | Managed Env | Status |
|---|---|---|---|---|---|---|---|---|
| PLATFORM-COE-ADMIN | | | CoE | Unclassified Non-CUI | Not Required | | Enabled | Active |
| PLATFORM-COE-DEV | | | CoE | Unclassified Non-CUI | Not Required | | Enabled | Active |
| PLATFORM-SHAREDSVC-PROD | | | Shared Services | CUI | Active | | Enabled | Active |
| PLATFORM-SHAREDSVC-DEV | | | Shared Services | Unclassified Non-CUI | Not Required | | Enabled | Active |

---

## Program environments

*Add one row per program environment. Group by program for readability.*

### `[ORG]-[COMMAND]-[PROGRAM]` program

| Environment Name | Environment ID | URL | Type | Classification | ATO Status | ATO Expiry | Human Owner | ISSO | Service Principal | ADO Project | Managed Env | Provisioned | Expiry | Last Reviewed | Status |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| | | | Program Dev | | | | | | | | | | | | |
| | | | Program Test | | | | | | | | | | | | |
| | | | Program Prod | | | | | | | | | | | | |

---

## Sandbox environments

{: .warning }
> Sandboxes older than 90 days are automatically quarantined by the CoE expiry workflow. Quarantined environments are held for 30 days pending owner confirmation before deletion.

| Environment Name | Environment ID | Owner | Command | Provisioned | Expiry | Status |
|---|---|---|---|---|---|---|
| | | | | | | |

---

## Individual developer environments

| Environment Name | Environment ID | Developer | Command/Program | Provisioned | Status |
|---|---|---|---|---|---|
| | | | | | |

---

## Decommissioned environments

Retain records of decommissioned environments for 3 years (NARA retention schedule).

| Environment Name | Environment ID | Type | Program | Decommissioned Date | Data Disposal Method | ISSO Sign-off | Notes |
|---|---|---|---|---|---|---|---|
| | | | | | | | |

---

## Quarterly owner confirmation process

Each quarter, the CoE sends an automated notification to every environment owner. Owners must confirm:

1. The environment is still active and in use
2. The listed human owner is still correct
3. The data classification is still accurate
4. The ATO status is current

**No response within 30 days:** Environment is escalated to the owner's command representative.

**No response after 60 days:** Environment is quarantined. Owner has 30 days to respond before deletion.

**Owner transfer process:**

When an environment owner transfers or departs:
1. Outgoing owner (or their supervisor) submits ownership transfer request via the intake application
2. New owner is designated and accepts responsibility in writing
3. CoE updates this register within 24 hours
4. If service principals are tied to the departing person's credentials, rotate immediately

---

## ATO coverage summary

*Maintained separately by the ISSO. Link to the ATO evidence package storage location.*

| Program | ATO Type | Status | Expiry | ISSO | Last Evidence Package | Next Review |
|---|---|---|---|---|---|---|
| | | | | | | |

---

## Register maintenance log

| Date | Change | Changed By |
|---|---|---|
| | Register initialized | |
