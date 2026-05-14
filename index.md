---
layout: home
title: Home
nav_order: 1
permalink: /
---

# GovFlow
{: .fs-9 }

Government Federated Low-Code Operations Framework
{: .fs-6 .fw-300 }

[Get started](getting-started/){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[View on GitHub](https://github.com/DevonAleshireMSFT/gov-flow){: .btn .fs-5 .mb-4 .mb-md-0 }

---

## What is GovFlow?

**GovFlow** is the enterprise governance layer for Microsoft Power Platform in GCC High / DoD environments. It covers the platform-layer concerns that are explicitly out of scope for [LP-ALM](https://devonaleshiremsft.github.io/layered-platform-alm/): environment topology, DLP architecture, RBAC design, CoE model, ATO-supportable governance, and organizational structure at Army/Navy/USMC scale.

It is not a starter kit. It is a reference architecture with templates. Teams implement it by applying the patterns to their own GCC High tenants.

{: .important }
> **GovFlow governs the platform layer. [LP-ALM](https://devonaleshiremsft.github.io/layered-platform-alm/) governs the solution layer.** Read both. Implement both. They are designed to be used together.

## The two-layer stack

| Layer | Framework | What it governs |
|---|---|---|
| **Platform layer** | GovFlow (this site) | Tenant setup · Environment topology · DLP · RBAC · CoE · Managed Environments · ATO governance · Leadership reporting |
| **Solution layer** | [LP-ALM](https://devonaleshiremsft.github.io/layered-platform-alm/) | Five-layer decomposition · Pipeline YAML · Source control structure · Security role design · PAC CLI workflow |


## Who this is for

| Role | What GovFlow provides | What LP-ALM provides |
|---|---|---|
| **Platform CoE Engineers** | Environment governance, DLP architecture, CoE Starter Kit setup | Pipeline YAML, PAC CLI workflow, source control structure |
| **Enterprise Architects** | 8-tier environment topology, BU hierarchy, tenant segmentation | Layer decomposition model, solution structure |
| **Security / ISSO** | RBAC pyramid, Conditional Access, audit log routing, IL5 isolation | Security role design, field security profiles, managed solution enforcement |
| **Program Managers** | Application classification, intake process, governance board | Project onboarding checklist, per-program environment setup |
| **Senior Leadership** | Executive scorecard, adoption metrics, cost visibility | Not in scope for LP-ALM |

---

## Recommended reading order

{: .note }
> Read [Power Platform Landing Zones](https://github.com/microsoft/industry/tree/main/foundations/powerPlatform) → **GovFlow** (this site) → [LP-ALM](https://devonaleshiremsft.github.io/layered-platform-alm/) → [Power Platform Well-Architected](https://learn.microsoft.com/en-us/power-platform/well-architected/)

1. **[Getting Started](getting-started/)** — First 30-day priorities, key decisions, prerequisites
2. **[Enterprise Strategy — GCC High / DoD](enterprise-strategy/)** — Full environment topology, security architecture, governance model, ALM strategy, operational support, and implementation roadmap for large-scale DoD organizations
3. **[Governance Templates](governance-templates/)** — Fillable templates: PGB charter, environment register, intake checklist, ATO evidence guide
4. **[LP-ALM Methodology](https://devonaleshiremsft.github.io/layered-platform-alm/methodology/)** ↗ — Five-layer solution decomposition, pipeline YAML, PAC CLI workflow, security role design (external site)

---

## Framework principles

**1. ATO evidence is produced, not assembled.**
Security role exports, audit logs, DLP policy records, and environment configuration are maintained as operational artifacts — generated from the live environment quarterly, not written in a spreadsheet the week before review.

**2. The platform survives personnel turnover.**
Service principals, Entra ID groups, and source control replace individual-owned connections, accounts, and credentials. In DoD organizations, personnel churn is not a hypothetical — it is the operational norm.

**3. Governance enables rather than gates.**
Self-service sandboxes, pre-approved patterns, and automated intake mean the CoE team is not in the critical path for every deployment. A 5-person CoE team can govern 100+ programs with this model.

**4. One production environment per ATO boundary.**
ATO scope leakage is a compliance failure. Programs with separate ATOs have separate production environments — always.

---

## Version and alignment

- **Framework version:** 1.0 — May 2026
- **Power Platform Well-Architected:** Security · Reliability · Operational Excellence · Performance Efficiency · Experience Optimization
- **NIST SP 800-53:** AC-2, AC-3, AC-6, AU-2, AU-9, CM-2, CM-3, CM-6, IA-2, SA-3, SI-2
- **FedRAMP / DISA PA:** GCC High FedRAMP High, IL5 DISA provisional authorization guidance
