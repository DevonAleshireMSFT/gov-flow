---
layout: default
title: Environment Access & RBAC
nav_order: 6
permalink: /environment-access/
---

# Environment Access and RBAC Implementation
{: .no_toc }

How to wire Entra ID security groups to Power Platform environments and configure Dataverse role-based access across Dev, Test, and Production tiers.
{: .fs-5 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The two-layer access model

Access in a Power Platform environment has two distinct layers that must be configured independently. Conflating them is the most common RBAC implementation mistake.

```
Layer 1 — Entra ID Security Group
          Controls who can enter the environment.
          Assigned once per environment in the Admin Center.
          If a user is not in the group, they cannot access the environment at all.
          ↓
Layer 2 — Dataverse Security Role (via group-backed team)
          Controls what a user can see and do inside the environment.
          Assigned through Dataverse team membership.
          Multiple roles/groups can exist within one environment.
```

> **The separation is intentional.** An end user who belongs to the environment access group can enter the environment — but without a Dataverse security role, they cannot read or write any records. Both layers are required for a user to function.

For the full architectural design — the RBAC pyramid, Business Unit strategy, and field-level security — see [Enterprise Strategy §3.1 and §3.5](../enterprise-strategy/#31-rbac-strategy).

---

## Assigning a security group to an environment

Each Power Platform environment accepts **one assigned security group** that gates environment access. Users outside this group cannot access the environment.

**Admin Center steps:**

1. Navigate to [gcc.admin.powerplatform.microsoft.us](https://gcc.admin.powerplatform.microsoft.us) (GCC High) or [dod.admin.powerplatform.microsoft.us](https://dod.admin.powerplatform.microsoft.us) (DoD)
2. Select the environment → **Settings** → **Users + permissions** → **Security groups**
3. Click **Add security group** and select the Entra ID group for this environment
4. Save

{: .note }
> Only one security group can be assigned as the environment access gate. Use a **nested group** strategy for environments with multiple access populations — create a parent group (e.g., `PP-FORSCOM-SYSTRK-PROD-Access`) and nest the role-specific groups inside it (`PP-FORSCOM-SYSTRK-PROD-EndUsers`, `PP-FORSCOM-SYSTRK-PROD-Support`). Membership in any nested group grants environment entry.

**What happens after assignment:**
- New group members are automatically provisioned into the environment when they first sign in or are manually synced
- Removed group members lose access at next sync (typically within 1 hour)
- Existing users who are removed from the group retain their Dataverse records but lose the ability to sign in

---

## Group-backed Dataverse teams

Within an environment, role assignment happens through **Dataverse teams linked to Entra ID groups**. This is the correct pattern for government environments — it eliminates individual role assignments and ensures that personnel transitions are handled through group membership changes, not Dataverse admin operations.

### Creating a group-backed team

1. In the environment, navigate to **Settings** → **Users + permissions** → **Teams**
2. Click **+ New team**
3. Set **Team type** to **Microsoft Entra ID Security Group**
4. Select the Entra ID group to link
5. Assign the appropriate Dataverse security role to the team

Once linked, every member of the Entra group automatically receives the team's security role. Removing a user from the Entra group removes their role — no Dataverse action required.

### Recommended team structure per program environment

| Team name | Linked Entra group | Dataverse security role |
|---|---|---|
| `{PGM} - Program Admins` | `PP-{ENV}-Owners` | Program Admin (custom) |
| `{PGM} - Developers` | `PP-{ENV}-Developers` | Contributor (Dev only) |
| `{PGM} - End Users` | `PP-{ENV}-{AppCode}-Users` | End User (custom, minimum privilege) |
| `{PGM} - Support` | `PP-{ENV}-Support` | Support (custom, read + limited write) |
| `{PGM} - Auditors` | `PP-{ENV}-Auditors` | Read Only |
| `{PGM} - Pipeline SP` | N/A — service principal | System Administrator |

{: .important }
> **System Administrator is assigned to the pipeline service principal only — not to any team.** No human user should hold System Administrator in Test or Production. See [Service Account Security](../service-account-security/) for pipeline identity governance.

---

## Access matrix by environment tier

Who gets what access at each environment tier. Apply this consistently — access should tighten as environments move toward production, never loosen.

| Role | Dev | Test | UAT | Prod |
|---|---|---|---|---|
| **Platform CoE (PIM-elevated)** | Environment Admin | Environment Admin | Environment Admin | Environment Admin (PIM only) |
| **Program Admin** | Program Admin role | Program Admin role | Read + escalation path | Read + escalation path |
| **Developer** | Contributor (full write) | No access | No access | No access |
| **QA / Test team** | — | Support role (scoped) | End User role | No access |
| **End users / mission personnel** | — | — | End User role | End User role |
| **Support team** | — | Support role | Support role | Support role (scoped) |
| **Auditors / ISSO** | Read Only | Read Only | Read Only | Read Only |
| **Pipeline service principal** | System Administrator | System Administrator | System Administrator | System Administrator |

{: .warning }
> Developers do not have access to Test, UAT, or Prod environments. All changes in upper environments are applied exclusively through the deployment pipeline. A developer who needs to investigate a production issue does so through logs and monitoring — not through direct environment access.

### Granting emergency production access

When a production incident requires a developer to access the environment directly:

1. Submit an access request with incident ticket number and ISSO notification
2. Platform CoE grants time-limited access via PIM-eligible environment admin role (maximum 8-hour window)
3. All actions during the access window are captured in Dataverse audit logs
4. Access is removed immediately upon incident resolution
5. Incident access event is documented in the environment register

---

## Security group naming convention

Consistent naming makes groups auditable and self-documenting. Recommended pattern:

```
PP-{ORG}-{CMD}-{PGM}-{TYPE}-{ENV-TIER}
```

| Token | Values | Example |
|---|---|---|
| `PP` | Fixed prefix — Power Platform | `PP` |
| `{ORG}` | Org abbreviation | `FORSCOM` |
| `{CMD}` | Command or program code | `SYSTRK` |
| `{PGM}` | Sub-program (omit if same as CMD) | `MAINT` |
| `{TYPE}` | Access type (see below) | `EndUsers` |
| `{ENV-TIER}` | Dev / Test / UAT / Prod | `Prod` |

**Access type values:**

| Type token | Purpose |
|---|---|
| `Owners` | Program Admins — environment management |
| `Developers` | Dev environment only |
| `EndUsers` | Mission personnel / app end users |
| `Support` | L2 support team |
| `Auditors` | Read-only; ISSO and audit team |
| `Access` | Parent/nested group for environment gate |

**Examples:**
```
PP-FORSCOM-SYSTRK-EndUsers-Prod
PP-FORSCOM-SYSTRK-Developers-Dev
PP-FORSCOM-SYSTRK-Support-Prod
PP-FORSCOM-SYSTRK-Access-Prod       ← parent group containing all Prod sub-groups
```

---

## GCC High / DoD and IL5 considerations

### Dynamic groups for end-user populations

In large DoD organizations, end-user group membership should be maintained dynamically through HR system attributes rather than manually managed. Entra ID dynamic groups populate automatically based on user attributes such as department, unit, or position.

Benefits in government environments:
- Personnel transitions (PCS, retirement, role change) update group membership automatically without IT intervention
- Eliminates the most common access control audit finding: users retaining access after role change
- Reduces administrative burden on program teams and help desk

{: .note }
> Dynamic group membership rules require Entra ID P1 licensing. Validate availability in your GCC High or DoD tenant before designing around dynamic groups.

**Do not use mail-enabled distribution lists** for Power Platform access. Distribution lists are not supported as environment access groups and do not support nested group patterns.

### Restricted management groups for IL5

For IL5 environments, consider **Entra ID restricted management administrative units** to limit who can modify the security groups that control IL5 environment access. Without this, any user with Group Administrator rights in Entra ID can add members to an IL5 access group.

Restricted management administrative units are an Entra ID P2 feature. Validate availability in your DoD tenant and document the configuration in the ATO as a compensating control for AC-3.

### Access review cadence

| Environment tier | Review cadence | Reviewer |
|---|---|---|
| Production | Quarterly | ISSO or designated Platform CoE lead |
| Test / UAT | Semi-annual | Program Admin |
| Dev | Annual | Platform CoE lead |
| IL5 (any tier) | Quarterly minimum | ISSO — mandatory, not delegatable |

Access reviews must be documented. The signed review record is ATO evidence for AC-2 (Account Management). Use the [Environment Register](../governance-templates/environment-register/) to track group assignments alongside environment ownership and ATO status.
