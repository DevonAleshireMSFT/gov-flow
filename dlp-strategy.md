---
layout: default
title: "DLP Strategy"
nav_order: 7
---

# Data Loss Prevention Strategy
{: .no_toc }

A governance framework for managing Power Platform connector policies at enterprise and DoD scale.
{: .fs-5 .fw-300 }

<details open markdown="block">
  <summary>Table of contents</summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

---

## What is a DLP Policy in Power Platform

A Data Loss Prevention (DLP) policy in Power Platform is an administrator-controlled configuration that governs which connectors are available in a given environment, and whether data can flow between connectors classified as trusted (Business) versus untrusted (Non-Business).

DLP policies are applied at the **tenant level** or **environment level**. Tenant-level policies establish the baseline — they cannot be overridden by environment-level policies, only further restricted. A user cannot create an environment-level policy that unlocks a connector blocked at the tenant level. This is a critical governance invariant: tenant policy is the ceiling, not the floor.

Power Platform connects to over 1,000 services through pre-built connectors. In an unmanaged tenant, any user with a Power Apps or Power Automate license can create a flow that moves data between SharePoint, external APIs, personal email accounts, consumer storage services, and public HTTP endpoints — with no visibility or audit trail for the organization. DLP policies are the primary technical control that prevents this.

---

## Why DLP is Critical in Enterprise Power Platform Governance

In a standard enterprise IT environment, network controls and endpoint policies govern data movement. Power Platform bypasses many of those controls by design — that is part of its value proposition. A licensed user can connect to an external API, send data to a third-party service, or receive inbound HTTP requests without touching a corporate network gateway or firewall.

This is not a vulnerability in Power Platform. It is a deliberate capability that requires a governance response.

For GCC High and DoD tenants specifically, the stakes are higher:

- Connectors that appear in the standard Power Platform connector catalog may route data through **Microsoft commercial cloud infrastructure**, not through the GCC High or DoD boundary. Using those connectors in an IL4 or IL5 environment is an unauthorized disclosure risk.
- "Convenient" connectors — personal OneDrive, Gmail, consumer Dropbox — are present in the catalog and accessible unless explicitly blocked. Personnel will use them if they are available.
- HTTP connectors can create unauthenticated inbound endpoints that are reachable from the public internet, bypassing perimeter controls entirely.
- Unreviewed third-party connectors may transmit data to services with no FedRAMP authorization, ITAR controls, or data residency guarantees.

DLP policies are the mechanism by which an organization answers the question: *which data flows are authorized?* Everything else must be blocked by default.

---

## Default-Deny Philosophy

{: .important }
> **GovFlow DLP Principle:** Every connector not explicitly approved is blocked. Approval is a governance action, not a default state. "We haven't reviewed it yet" means blocked — not allowed.

The default-deny model means:

- New connectors added to the Power Platform catalog are **blocked by default** until reviewed by the governance team and explicitly added to an approved policy
- Third-party connectors are never approved by default, regardless of their general reputation or commercial use
- Custom connectors require a separate custom DLP policy — they are not covered by standard policy tiers
- HTTP triggers and direct inbound HTTP endpoints are treated as blocked at the tenant level unless a specific, reviewed exception is granted

Default-deny does not mean blocking everything permanently. It means the burden of proof is on enablement, not restriction. A connector earns its place on the approved list through governance review — not through developer convenience.

---

## 1. Base Tenant DLP Policy

The base tenant DLP policy is the governance foundation for the entire Power Platform deployment. It applies to all environments not covered by a more specific environment-group policy. It cannot be overridden at the environment level.

**Recommended baseline configuration:**

| Connector Category | Default Classification | Rationale |
|---|---|---|
| Microsoft 365 core (SharePoint, Teams, Exchange, OneDrive for Business) | Business | Standard productivity baseline; FedRAMP authorized in GCC High/DoD |
| Microsoft Dataverse | Business | Core platform; required for model-driven apps |
| Azure Government services (verified) | Business | Stays within authorized boundary |
| Power Platform Admin connectors | Business | Platform management — restricted by RBAC separately |
| All third-party connectors | Non-Business (blocked) | Not reviewed; not FedRAMP authorized by default |
| HTTP (outbound) | Non-Business (blocked) | Open outbound HTTP is uncontrolled data egress |
| HTTP Request trigger (inbound) | Blocked entirely | Creates unauthenticated public endpoints |
| Custom connectors | Not covered — require custom DLP policy | See [Custom Connector Governance](#4-custom-connector-governance) |

**On the Business vs. Non-Business boundary:**

Placing two connectors in the same group (both Business, or both Non-Business) allows data to flow between them in a single flow or app. Placing them in different groups blocks that data flow. The tenant baseline must ensure that no path exists from a Business connector (SharePoint, Teams) to a Non-Business one (personal storage, external APIs) without explicit governance review and policy change.

{: .warning }
> **IL5 validation requirement:** Not all connectors in the Microsoft 365 "Business" baseline are authorized for IL5 data. SharePoint, Teams, and Exchange in GCC High route through the GCC High boundary and are generally IL5-supportable — but verify each connector's data residency documentation against your ATO boundary before approving. Some connectors in the Microsoft first-party catalog use commercial cloud backend infrastructure even when accessed from a GCC High tenant.

---

## 2. Environment-Type DLP Policies

The base tenant policy is the floor. Environment-type policies refine that baseline for specific environment categories. These are applied via Managed Environment groupings — environments are assigned to a group, and the group receives the appropriate DLP overlay.

The principle: **policies become more restrictive as data sensitivity and operational criticality increase toward production.** A developer environment may allow additional approved connectors for productivity. A production mission system environment should be the most locked-down tier.

### Personal Developer Environments

**Purpose:** Individual productivity, personal automation, learning, prototyping with non-CUI data.

**Recommended policy tier:** Most permissive of the approved tiers. Allow the full set of approved first-party Microsoft connectors. Block third-party, HTTP, and custom connectors. This is not a sandbox for enterprise development — it is for individual use only.

**Governance note:** Developer environments are not in the ATO boundary. No CUI, no mission data, no production service connections.

### Dataverse for Teams Environments

**Purpose:** Team-level collaboration apps, lightweight automation, departmental productivity.

**Recommended policy tier:** First-party Microsoft connectors approved. Approval actions (Power Automate Approvals), Teams messaging, SharePoint. Third-party and HTTP remain blocked. No Dataverse premium connectors unless specifically approved.

**Governance note:** Teams environments are provisioned by Teams owners — governance must be applied automatically on creation. The CoE Starter Kit monitors these environments for data classification escalation.

### Sandbox Environments

**Purpose:** Development and testing of integrations, connector validation, pre-production integration testing.

**Recommended policy tier:** Base tenant policy plus additional approved connectors specific to the program's integration pattern — Azure Service Bus, Azure Event Hubs, approved API Management endpoints. Third-party and HTTP remain blocked unless a specific exception is approved. No connector is approved in sandbox just because it is convenient.

### Test / UAT Environments

**Purpose:** Integration testing against production-equivalent services, user acceptance testing.

**Recommended policy tier:** Match production policy. Test environments that are less restrictive than production create a false confidence in test results. The same connector restrictions that apply in production must apply in test — otherwise test coverage for connector policy compliance is meaningless.

### Production Environments (Program-Owned)

**Purpose:** Live mission systems, operational business applications, ATO-covered workloads.

**Recommended policy tier:** Most restrictive of the program tiers. Only explicitly approved connectors in the Business group. All approved connectors must be documented in the program's ATO evidence. No connector exceptions without ISSO review and documented business justification.

Production environments are the wrong place to discover a connector is blocked. DLP policy alignment between test and production is a deployment requirement, not an afterthought.

### Enterprise Production Environment (CoE-Owned)

**Purpose:** Org-wide or cross-command applications. See [Default Environment — Enterprise-wide access strategy](../default-environment/#enterprise-wide-application-access-strategy).

**Recommended policy tier:** Match or exceed program production policy. Enterprise environments carry Platform-level ATO coverage — the connector inventory must be validated against that ATO boundary explicitly.

---

## 3. Premium Connector Governance

Premium connectors provide access to enterprise services beyond the standard Microsoft 365 baseline — Azure services, Dataverse advanced capabilities, SAP, ServiceNow, third-party SaaS platforms. They require Power Apps or Power Automate premium licensing and represent elevated governance risk because they expand the organizational data perimeter.

### Approved Premium Connector Catalog

The Platform CoE maintains a centrally approved premium connector allow-list. A connector is added to this list only after:

1. **Security review** — data residency confirmed; connector backend infrastructure verified as FedRAMP authorized or boundary-equivalent for the target classification level
2. **Authentication review** — connector supports OAuth 2.0, service principal, or managed identity authentication; no shared credential or API key stored in the connector definition
3. **Admin consent** — Entra ID admin consent granted for the connector's OAuth application registration where required
4. **Third-party risk assessment** — for non-Microsoft connectors, vendor security posture reviewed; data processing agreements and ITAR/export control implications assessed
5. **Data classification determination** — maximum data classification the connector is approved to handle (Unclassified, CUI, IL4, IL5 where applicable)

The approved catalog is maintained as a living document, reviewed at minimum annually and whenever a connector's underlying service changes its authorization status.

### Connector Lifecycle Management

Approving a connector is not a permanent decision. Connector governance includes:

- **Annual re-review** of all approved connectors against current FedRAMP authorizations and ATO boundaries
- **Change notification process** — Microsoft updates connector behavior through its managed connector model; significant changes trigger re-review
- **Deprecation tracking** — connectors that lose FedRAMP authorization or change their data routing must be removed from approved policies and existing uses tracked for migration

### Avoiding Broad Connector Enablement

The instinct to approve a connector broadly (tenant-wide, all environments) to avoid repeated requests is a governance anti-pattern. Approve connectors at the most specific scope possible:

- Approve for a specific environment group if the use case is limited
- Approve at tenant level only for connectors that are genuinely universal and verified for all data classification levels in use

Every connector approved broadly becomes part of every program's DLP policy surface. Scope minimization is scope control.

---

## 4. Custom Connector Governance

Custom connectors allow Power Platform to communicate with internal line-of-business APIs, agency-specific services, and external third-party APIs that do not have a pre-built connector. They represent the highest governance risk in the connector model because they are developer-created, organization-specific, and not reviewed by Microsoft.

A custom connector that reaches an unauthenticated or weakly authenticated internal API is a data exfiltration path. A custom connector that stores credentials in the connector definition exposes those credentials to any flow or app that uses the connector.

### Custom Connector Review Requirements

Before any custom connector is approved for production use, the following must be validated:

**API security validation:**
- The target API is documented with an OpenAPI specification
- All endpoints require authentication — no anonymous or IP-only access
- Authentication uses OAuth 2.0 with service principal, API key stored in Azure Key Vault (not in the connector definition), or managed identity where supported

**Architecture review:**
- The API endpoint is hosted within the authorized boundary (Azure Government, on-premises behind approved gateway, or FedRAMP-authorized SaaS)
- The API is fronted by Azure API Management (APIM) where possible — APIM provides throttling, logging, authentication enforcement, and a stable contract surface
- Inbound connections from Power Platform to the API are logged at the API layer

**Ownership and sustainment:**
- A named technical owner is registered for the custom connector
- The connector is version-controlled (OpenAPI spec in source control)
- Lifecycle plan: what happens to flows and apps using this connector if the API is retired or updated

**Production deployment:**
- Custom connectors deployed to production environments must be deployed as managed solution components through the LP-ALM pipeline — not manually created in production
- Connection references in solutions point to pre-approved custom connectors, not environment-specific ad hoc connections

### Azure API Management as the Preferred Gateway

Custom connectors that reach internal or external APIs should, where architecturally feasible, be routed through Azure API Management hosted in the Azure Government subscription. APIM provides:

- Centralized authentication enforcement (validate JWT, verify service principal)
- Request/response logging for audit purposes
- Rate limiting and throttling to prevent runaway flows from impacting backend systems
- Stable connector contract even as backend APIs evolve (API versioning)
- Network isolation — internal APIM with Private Link keeps connector traffic inside the authorized boundary

A custom connector that points directly to an internal API without APIM is an ungoverned data path. APIM is not a requirement where technically infeasible, but it is the recommended pattern for any connector serving production workloads.

---

## 5. HTTP Triggers and External Endpoint Governance

### The HTTP Request Trigger Risk

The Power Automate HTTP Request trigger creates a publicly accessible, HTTPS endpoint that can receive inbound requests from any source on the internet. When a flow uses this trigger, a URL is generated that — if reached — immediately executes the flow logic.

By default, this trigger creates an unauthenticated endpoint. Anyone who knows the URL can trigger the flow. The URL can be shared, leaked in browser history, exposed in log files, or discovered through enumeration.

{: .warning }
> **The HTTP Request trigger must be treated as blocked by default in enterprise tenants.** This is not a theoretical risk — unauthenticated inbound triggers are direct attack surfaces in production environments. In GCC High and DoD tenants, creating an unauthenticated inbound endpoint is inconsistent with ATO requirements for access control and audit logging.

Even when the trigger is configured with an OAuth requirement, the implementation places authentication responsibility on the flow developer — a pattern that has historically produced misconfigured or bypassed authentication checks.

### Recommended Alternatives

| Use Case | Recommended Pattern |
|---|---|
| Azure service must trigger a Power Automate flow | Azure Logic Apps (full trigger library) or Event Hub → Power Automate Event Hub trigger |
| External application needs to trigger a workflow | Azure Functions (authorized boundary) as intermediary → Power Automate via service bus or queue |
| Event-driven integration from Azure workloads | Azure Service Bus or Event Hubs → Power Automate trigger (authenticated, queue-based, no public endpoint) |
| Scheduled or batch processing | Recurrence trigger — not inbound HTTP |
| Internal API integration | Custom connector via APIM — outbound from Power Platform, not inbound to it |

The pattern in all cases is the same: **Power Platform reaches out; nothing reaches in through an unauthenticated HTTP endpoint.** Event-driven and queue-based integration keeps Power Platform as a consumer of events, not an unprotected listener.

### Outbound HTTP Connector

The HTTP action connector (outbound) is separate from the HTTP Request trigger (inbound). Outbound HTTP allows flows to call arbitrary external endpoints. This is a controlled but elevated-risk capability:

- Outbound HTTP should require ISSO review before production use
- Endpoints must be documented in the ATO evidence
- Calls must authenticate to the target service — no anonymous outbound HTTP to production APIs
- For IL5, the target endpoint must be within the authorized boundary or have explicit ATO coverage for external communication

---

## 6. Enterprise Connector Governance Model

At DoD scale, individual connector requests handled ad hoc by platform administrators create inconsistency, audit gaps, and administrator burnout. The connector governance model must be systematized.

### Connector Approval Process

```
Developer/PM identifies connector requirement
        ↓
Submit request via CoE intake application
(connector name, use case, data classification, environment target)
        ↓
Automated pre-screening: Is connector already approved?
  → YES: Connector available per existing policy, no action needed
  → NO: Proceed to review
        ↓
Platform CoE initial review (security, data residency, auth model)
        ↓
For third-party / custom / HTTP: ISSO review required
        ↓
Decision: Approve (scope), Conditional Approve (with controls), Deny (with alternatives)
        ↓
Policy updated; connector added to approved catalog
        ↓
Requestor notified; catalog updated
```

Governance review SLA: 5 business days for standard pre-built connectors; 10 business days for third-party or custom connectors requiring security assessment.

### Exception Handling

Connector exceptions (approving a connector outside the standard tier for a specific environment) must be:

- Documented with business justification and data classification impact assessment
- Approved by ISSO and Platform CoE
- Time-limited: 90-day review cycle; exceptions must be re-justified or made permanent through the standard approval process
- Recorded in the environment register against the specific environment

Exceptions are not a fast path to avoid the approval process. They are a formal mechanism for time-limited risk acceptance.

### Connector Inventory and Reporting

The Platform CoE maintains:

- **Approved connector catalog** — all connectors approved by environment tier, with data classification level, approval date, and review date
- **Environment connector report** — which connectors are active in each environment, generated from CoE Starter Kit telemetry
- **Exception register** — all active exceptions, expiry dates, and approval records
- **Periodic review** — full catalog reviewed annually; any connector whose authorization basis changes triggers immediate re-review

DLP violation events (flows blocked by policy) are surfaced in CoE Starter Kit monitoring and reported weekly to the Platform CoE and ISSO. Repeated violations against a specific connector signal that an approval request may be warranted — or that a developer is attempting to bypass policy.

---

## 7. Operational Governance Recommendations

### Centralized Tenant Governance

DLP policy management is a tenant-level administrative function. Individual environment owners, program teams, and citizen developers cannot modify tenant DLP policies — they can only request changes through the governance process. Environment administrators can create environment-level policies, but only to further restrict the tenant baseline.

{: .important }
> Environment-level user-created DLP policies that do not serve a documented governance purpose should be removed. They clutter the admin UI, cause developer confusion about which policy is in effect, and create false confidence that local policies provide additional protection.

### Avoiding Policy Sprawl

Unmanaged DLP policies — policies created by well-intentioned admins or developers to solve immediate problems without going through the governance process — accumulate over time and become impossible to audit. Signs of policy sprawl:

- Multiple overlapping environment-level policies with no clear ownership
- Policies that appear to be testing artifacts from administrator experimentation
- No documentation or approval record for a policy's existence

The Platform CoE should run a quarterly policy inventory, verify every policy has an owner and a documented purpose, and remove orphaned policies.

### Governance Automation

The following DLP governance activities should be automated through the CoE Starter Kit and platform admin APIs:

| Activity | Automation |
|---|---|
| New environment created → policy applied | Automated via environment creation flow in CoE intake app |
| DLP violation detected | Alert to Platform CoE; weekly violation report to ISSO |
| Exception expiry approaching | Automated notification to exception owner 30 days prior |
| New connector added to catalog | Admin notified for review; blocked by default until reviewed |
| Quarterly connector report | Automated from CoE telemetry; no manual assembly |

### Aligning DLP to Enterprise Architecture

DLP policy is not a standalone control — it is one layer in a defense-in-depth model for Power Platform. The full governance stack:

| Layer | Control |
|---|---|
| Identity | Entra ID conditional access, MFA, PIM for admin roles |
| Environment | Managed Environments, security group assignment, RBAC |
| Data | Dataverse field security, sensitivity labels, row-level security |
| Connector | DLP policy (this document) |
| Code | Custom connector review, solution ALM via LP-ALM pipeline |
| Monitoring | CoE Starter Kit, Power Monitoring Framework, audit log export |

DLP policy without RBAC is incomplete — a user who can access an environment can use any connector allowed in that environment. RBAC without DLP is incomplete — a user with appropriate permissions can still move data to unauthorized destinations if DLP does not restrict it. Both controls must be active.

See [Enterprise Strategy §3 — Security & Compliance Architecture](../enterprise-strategy/#3-security--compliance-architecture) for the full security model this DLP strategy operates within.
