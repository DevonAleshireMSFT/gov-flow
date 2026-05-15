---
layout: default
title: "Tenant Hardening"
nav_order: 4
---

# Power Platform Tenant Hardening
{: .no_toc }

Required configuration actions for GCC High and DoD tenants before any environment is created or maker is licensed.
{: .fs-5 .fw-300 }

<details open markdown="block">
  <summary>Table of contents</summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

---

## Power Platform is not secure by default

Power Platform is built for accessibility. The default tenant state — before any hardening — allows any licensed user to create environments, share applications with the entire organization, connect to any available connector, and move data to external services without oversight. In commercial tenants this creates governance risk. In GCC High and DoD IL5 tenants it creates compliance risk.

The GCC High boundary provides **data residency and FedRAMP authorization** — it does not configure governance controls for you. The settings below are operator responsibility and must be configured before any maker receives a license.

{: .warning }
> **Do not license makers before completing this checklist.** A licensed user in an unconfigured tenant can create environments, share applications with Everyone, and access all available connectors within minutes. The order matters: harden the tenant first, then license users, then provision environments.

---

## Sequence of Actions

Complete in this order. Later steps depend on earlier ones being in place.

```
1. Restrict environment creation
2. Disable "Share with Everyone"
3. Deploy tenant default DLP policy
4. Configure tenant isolation
5. Enable Managed Environments on existing environments
6. Configure audit logging
7. Set up Conditional Access (Entra ID)
8. Deploy CoE Starter Kit
9. Establish DLP policy for Teams environments
10. Enable governance intake process (GovFlow Request Portal)
```

Steps 1–4 can be completed in under an hour. Steps 7–10 require additional infrastructure and take longer. Do not skip steps 1–4 while waiting to complete the rest.

---

## 1. Restrict Environment Creation

**Location:** Power Platform Admin Center → Settings → Tenant settings → Who can create environments

| Setting | Default | Required |
|---|---|---|
| Who can create production and sandbox environments | Everyone | **Only specific admins** |
| Who can create trial environments | Everyone | **Only specific admins** |
| Who can create developer environments | Everyone | Restrict or leave — see note |

Developer environments are lower risk (1GB, limited to the individual, no sharing). Restricting them reduces sprawl but may slow developer onboarding. Evaluate based on your organization's maturity and maker population size. In either case, developer environments must be covered by the tenant default DLP policy.

**Why this matters:** An unconfigured tenant allows any licensed user to create a production environment with full Dataverse capacity, no DLP policy, and no governance oversight. Environment sprawl begins on day one.

---

## 2. Disable "Share with Everyone"

**Location:** Power Platform Admin Center → Settings → Tenant settings → Share with Everyone

| Setting | Default | Required |
|---|---|---|
| Share canvas apps with Everyone | Enabled | **Disabled** |

"Share with Everyone" allows any maker to share any canvas app with every licensed user in the tenant — including contractors, temporary personnel, and accounts pending offboarding. In GCC High and DoD tenants, "Everyone" cannot be scoped to cleared personnel or a specific command. There is no access audit trail, no approval record, and no graduated rollback when access needs to be revoked.

Disable this setting permanently. All application sharing must use Entra ID security groups — even if that group contains the entire user population. See [Default Environment Governance](../default-environment/#the-share-with-everyone-problem) for the full governance rationale.

---

## 3. Additional Tenant Settings

**Location:** Power Platform Admin Center → Settings → Tenant settings

Configure the following while in Tenant Settings:

| Setting | Recommended Configuration | Rationale |
|---|---|---|
| Publish canvas apps as org app without admin approval | **Require admin approval** | Prevents unapproved apps from appearing in the org app catalog |
| Power Pages site creation | **Only specific admins** | External-facing sites require explicit approval |
| AI Builder credits | Configure per licensing model | Prevent unplanned AI Builder consumption |
| Weekly digest emails to makers | Enable or disable per policy | Informational only; no security impact |
| Copilot in Power Apps / Power Automate | Evaluate per org policy | Most Copilot features are off by default in GCC High — verify current state before enabling |

---

## 4. Deploy Tenant Default DLP Policy

The tenant default DLP policy must be deployed **before any non-admin user accesses Power Platform.** It applies to all environments not covered by a specific environment-group policy. An environment with no DLP policy has all connectors available.

**Minimum tenant default policy configuration:**

| Connector Category | Classification |
|---|---|
| SharePoint, Teams, Exchange, OneDrive for Business | Business |
| Microsoft Dataverse | Business |
| Power Platform admin connectors | Business |
| All third-party connectors | Non-Business (blocked) |
| HTTP (outbound) | Non-Business (blocked) |
| HTTP Request trigger (inbound) | Blocked |

Create this policy in PPAC → Data policies → New policy. Set the scope to "All environments except" and do not exclude any environments initially. Environment-specific policies are layered on top — this is the floor.

For the full DLP policy architecture, tiering model, and connector governance process, see [DLP Strategy](../dlp-strategy/).

{: .warning }
> **GCC High connector validation:** Before finalizing the tenant default policy, verify which connectors are available in your GCC High tenant. Not all connectors in the commercial catalog are available in GCC High. Attempting to include a connector that does not exist in your tenant will produce errors in the policy configuration.

---

## 5. Configure Tenant Isolation

**Location:** Power Platform Admin Center → Policies → Tenant isolation

Tenant isolation controls whether connectors can authenticate across Entra ID tenant boundaries — inbound (other tenants connecting to your data) and outbound (your users connecting to other tenants' data).

| Setting | Default | Recommended |
|---|---|---|
| Tenant isolation | Off | **Enabled** |
| Inbound connections from other tenants | Allowed (when disabled) | **Block unless explicitly allowlisted** |
| Outbound connections to other tenants | Allowed (when disabled) | **Block unless explicitly allowlisted** |

In a GCC High tenant, cross-tenant connections may route data to commercial cloud tenants. Enabling tenant isolation with an explicit allowlist ensures that only approved partner or vendor tenants can exchange data through Power Platform connectors.

Allowlist entries are managed by the Platform CoE. Requests for cross-tenant connector access go through the same governance intake process as DLP exceptions.

---

## 6. Enable Managed Environments

Managed Environments must be explicitly enabled on each environment. They are not on by default. Managed Environments provide:

- Environment-level DLP policy application (required for tiered DLP)
- Weekly usage digests to environment admins
- Limit sharing controls (prevent sharing beyond a defined number of users)
- CoE Starter Kit integration (Managed Environments required for full CoE inventory sync)
- Premium maker welcome content

**Enablement:** PPAC → Environments → Select environment → Edit → Enable Managed Environments.

Priority order for enablement:
1. `COE-ADMIN` — enable immediately
2. All existing production environments
3. All existing test and dev environments
4. Default environment
5. Dataverse for Teams environments (enable as discovered by CoE sync)

Managed Environments require Power Apps or Power Automate premium licensing on the tenant. Confirm licensing coverage before enabling broadly.

---

## 7. Configure Audit Logging

Power Platform audit logs are written to the Microsoft 365 Unified Audit Log. By default, audit log search may not be enabled on the tenant.

**Steps:**
1. Confirm audit logging is enabled: Microsoft Purview compliance portal → Audit → verify "Start recording user and admin activity" is active
2. Configure audit log retention: default is 90 days (E3) or 1 year (E5/equivalent). For IL5 ATO requirements, 1-year minimum retention is recommended — extend via archive policy if needed
3. Connect to Log Analytics: configure audit log export to Azure Government Log Analytics workspace for operational alerting and long-term retention. See [Azure Integration Strategy §7](../azure-integration/#7-monitoring-logging-and-operational-telemetry) for workspace configuration

Power Platform-specific audit events to verify are being captured:
- Environment created / deleted
- DLP policy created / modified
- App shared / permission changed
- Flow enabled / disabled
- Admin role assigned

---

## 8. Entra ID Conditional Access

Conditional Access for Power Platform admin roles is configured in Entra ID, not in the Power Platform Admin Center. This is the identity team's responsibility but must be coordinated with the Platform CoE before admin roles are assigned.

Minimum policies required before granting Power Platform administrator roles:

| Policy | Scope | Condition |
|---|---|---|
| Require MFA | Power Platform Administrator role | All sign-ins |
| Require compliant device | PPAC access | All admin role holders |
| Block legacy authentication | All Power Platform apps | Legacy protocol detected |

Do not assign Power Platform Administrator or Environment Administrator roles to users until Conditional Access policies are in place. See [Azure Integration Strategy §2](../azure-integration/#2-identity-and-user-synchronization-strategy) for PIM configuration and role governance.

---

## 9. Dataverse for Teams Environment Governance

Teams environment creation cannot be fully disabled — it is controlled by Teams owners, not Power Platform admins. Governance must be automated through the CoE Starter Kit and applied via DLP.

**Immediate controls:**

1. **DLP policy scope** — ensure the tenant default DLP policy applies to Teams environments. Verify after creation that newly provisioned Teams environments inherit the tenant default.
2. **CoE Starter Kit Teams inventory** — the CoE sync identifies all Dataverse for Teams environments, their owners, apps, and flows. Without this, Teams environments are invisible to governance.
3. **Teams environment usage policy** — communicate to Teams owners what Teams environments are and are not for. See [Enterprise Strategy §2.5](../enterprise-strategy/#dataverse-for-teams-environments) for approved use cases and graduation triggers.

---

## 10. Tenant Hardening Verification Checklist

Use this checklist to verify completion before licensing makers or provisioning program environments.

**Environment Creation Controls**
- [ ] Production/sandbox environment creation restricted to admins only
- [ ] Trial environment creation restricted to admins only
- [ ] Developer environment creation policy decided and configured

**Sharing Controls**
- [ ] "Share with Everyone" disabled
- [ ] App catalog publishing requires admin approval
- [ ] Power Pages site creation restricted to admins

**DLP**
- [ ] Tenant default DLP policy deployed and scoped to all environments
- [ ] HTTP connector in Non-Business (blocked)
- [ ] All third-party connectors in Non-Business (blocked)
- [ ] Policy verified against GCC High connector availability list

**Tenant Isolation**
- [ ] Tenant isolation enabled
- [ ] Inbound cross-tenant connections blocked (allowlist empty or configured)
- [ ] Outbound cross-tenant connections blocked (allowlist empty or configured)

**Managed Environments**
- [ ] Managed Environments enabled on COE-ADMIN
- [ ] Managed Environments enabled on all existing production environments
- [ ] Licensing confirmed for Managed Environments tenant-wide

**Audit and Monitoring**
- [ ] Unified Audit Log enabled and verified active
- [ ] Audit log retention meets ATO requirements (minimum 1 year)
- [ ] Log Analytics workspace deployed in Azure Government
- [ ] Power Platform audit export connected to Log Analytics

**Identity**
- [ ] Conditional Access policy requiring MFA for Power Platform admin roles active
- [ ] PIM configured for Power Platform Administrator role (eligible, not permanent)
- [ ] No shared user accounts assigned to admin roles

**CoE**
- [ ] CoE Starter Kit deployed to COE-ADMIN environment
- [ ] CoE inventory sync running and returning environment data
- [ ] DLP violation reporting active in CoE dashboard

---

## Ongoing Tenant Governance

Hardening is not a one-time event. Schedule the following recurring activities:

| Activity | Cadence | Owner |
|---|---|---|
| Review new connectors added to catalog | Monthly | Platform CoE |
| Review DLP policy coverage for new environment types | Quarterly | Platform CoE + ISSO |
| Audit Power Platform admin role holders | Quarterly | Platform CoE + Identity team |
| Review tenant isolation allowlist | Annually | Platform CoE + ISSO |
| Verify audit log retention and export health | Monthly | Platform CoE |
| Review Managed Environment enablement on new environments | As part of environment provisioning | Platform CoE |
