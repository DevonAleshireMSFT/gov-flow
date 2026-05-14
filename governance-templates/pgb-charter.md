---
layout: default
title: Platform Governance Board Charter
parent: Governance Templates
nav_order: 1
permalink: /governance-templates/pgb-charter/
---

# Platform Governance Board Charter
{: .no_toc }

**Template version:** 1.0 — May 2026
{: .text-delta }

Copy this template. Replace all `[bracketed]` fields with organization-specific values.

---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Purpose

The Platform Governance Board (PGB) is the decision authority for the Power Platform tenant at `[ORGANIZATION NAME]`. The PGB sets platform policy, approves exceptions, and resolves escalations. It is not a deployment review body — it does not review individual solutions or sprint deliverables.

---

## Scope

This charter governs the Microsoft Power Platform GCC High tenant for `[ORGANIZATION NAME]` (`[TENANT ID]`). It applies to all environments, solutions, connectors, and users within that tenant.

Workloads classified at Secret or above are outside the scope of this charter and of the Power Platform platform entirely.

---

## Composition

| Role | Title | Authority |
|---|---|---|
| **Chair** | `[Enterprise Architect / CTO / CIO Designee]` | Final decision authority on policy disputes |
| **Platform CoE Lead** | `[Name / Billet]` | Technical authority for platform standards; primary facilitator |
| **ISSO Representative** | `[Name / Billet]` | Security and compliance sign-off; required for IL5 approvals |
| **Command Rep — `[COMMAND 1]`** | `[Name / Billet]` | Command community voice; 1-year rotating |
| **Command Rep — `[COMMAND 2]`** | `[Name / Billet]` | Command community voice; 1-year rotating |
| **Program Manager Rep** | `[Name / Billet]` | Program community voice; 1-year rotating |

**Quorum:** Chair + Platform CoE Lead + ISSO Representative. Decision cannot be made without all three.

**Voting:** Decisions are by consensus. The Chair has final authority when consensus is not reached.

---

## Decision authority matrix

| Decision | Authority | Escalation path |
|---|---|---|
| New production environment (Standard/Business Critical) | PGB approval | N/A |
| New IL5 environment | PGB approval + ISSO sign-off | N/A |
| DLP policy exception | PGB approval + ISSO sign-off | N/A |
| New connector approval (non-IL5) | Platform CoE Lead | Escalate to PGB if contested |
| New connector approval (IL5) | Platform CoE Lead + ISSO sign-off | Escalate to PGB if contested |
| Architecture exception (LP-ALM deviation) | Platform CoE Lead | Escalate to PGB if contested |
| Security exception (RBAC deviation from standard) | ISSO sign-off | Escalate to PGB |
| Program onboarding (Standard/Citizen) | Platform CoE Lead | N/A |
| Sandbox provisioning | Automated (self-service) — no PGB involvement | N/A |
| Tenant default DLP policy change | PGB approval | N/A |
| CoE Starter Kit major version upgrade | Platform CoE Lead | N/A |
| Environment decommission (non-production) | Platform CoE Lead | N/A |
| Production environment deletion | PGB approval + ISSO sign-off | N/A |

{: .important }
> **Sandbox provisioning is self-service.** The PGB is not in the critical path for sandbox environments. Self-service reduces CoE bottleneck and allows teams to move quickly for exploration and development.

---

## Cadence

**Standing meeting:** Monthly, 30 minutes.
- Review open exceptions and policy change proposals
- Review platform metrics dashboard (environment count, ATO findings, CoE SLA)
- Address escalations from the intake system
- Approve any policy changes from the prior month

**Async decisions:** Standard intake requests (non-exceptions) are routed asynchronously through the intake application. The PGB meeting addresses only policy changes, contested decisions, and escalations — not routine approvals.

**Exception SLA:** PGB members must respond to exception requests within `[5]` business days of notification. No response within the SLA window is treated as approval for Standard-class requests and escalated to the Chair for IL5 and DLP exception requests.

---

## Platform CoE responsibilities

The Platform CoE team operates the tenant on behalf of the PGB. CoE responsibilities include:

- Maintaining the CoE Starter Kit deployment in `PLATFORM-COE-ADMIN`
- Operating the environment request intake application
- Enforcing naming standards at provisioning time
- Maintaining the DLP policy architecture
- Publishing platform standards documentation
- Quarterly capacity reporting to the PGB
- Coordinating with the ISSO on audit log review
- Maintaining the self-hosted ADO agent pool

The CoE team is **not** responsible for individual program ALM implementations, solution architecture within a program's environment, or application-level security design. Those responsibilities belong to program teams operating under LP-ALM.

---

## Command representative responsibilities

Command representatives on the PGB are liaisons, not approval gatekeepers. Their responsibilities:

- Surface program community concerns and requirements to the PGB
- Communicate PGB policy decisions back to programs in their command
- Participate in monthly meeting and async decisions
- Identify programs in their command that need onboarding or elevated support
- Rotate annually (staggered to maintain continuity)

---

## Policy change process

1. Any PGB member or command program team may submit a policy change proposal
2. Proposal is submitted via the intake application, tagged as `Policy Change`
3. Platform CoE Lead reviews for technical feasibility within 5 business days
4. ISSO reviews for compliance impact within 5 business days
5. Proposal is added to the next PGB meeting agenda
6. PGB votes; Chair records decision in the PGB decision log
7. Approved changes are published to the platform standards documentation and communicated to all active programs within 10 business days

---

## Charter governance

**Effective date:** `[DATE]`

**Review cycle:** Annual, at the start of each fiscal year.

**Amendment:** Requires PGB consensus. Chair records amendment in the PGB decision log and publishes updated charter within 10 business days.

**Signatures:**

| Role | Name | Signature | Date |
|---|---|---|---|
| Chair | | | |
| Platform CoE Lead | | | |
| ISSO Representative | | | |

---

## Appendix A: PGB decision log

Maintain a running log of all PGB decisions. Store in `PLATFORM-COE-ADMIN` environment as a Dataverse record or SharePoint GCC High document.

| Date | Decision | Authority | Outcome | Notes |
|---|---|---|---|---|
| | | | | |

---

## Appendix B: Contacts

| Role | Name | Contact | Alternate |
|---|---|---|---|
| Chair | | | |
| Platform CoE Lead | | | |
| ISSO Representative | | | |
| Microsoft Unified Support TAM | | | |
| GCC High Tenant Admin | | | |
