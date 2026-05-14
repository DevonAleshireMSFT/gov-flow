---
layout: default
title: JavaScript & Code Security
nav_order: 9
permalink: /code-security/
---

# JavaScript & Code Security in Power Platform
{: .no_toc }

Work in Progress
{: .label .label-yellow }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

{: .warning }
> **Validation required** — This page contains guidance that requires additional validation against your organization's current DISA policy, AO authorization, and cloud boundary confirmation (GCC High vs. DoD IL5). Do not apply to production IL5 environments without independent security review and AO sign-off.

## Scope

Power Platform supports several patterns for embedding executable code into solutions:

- **JavaScript web resources** — attached to model-driven app forms and events
- **Power Apps Code Components (PCF)** — custom controls deployed via managed solutions
- **Power Apps Code Apps** — full code-first canvas apps
- **Power Pages custom scripts** — JavaScript in government-facing portals

These are not configuration — they are code. They must be treated as enterprise software and governed through a DevSecOps pipeline, not manually edited in production.

---

## The governing principle

> Develop outside production → scan in a controlled DevSecOps pipeline → package into a managed solution → deploy through approved ALM only.

No JavaScript enters production unless it came from source control, passed security scanning, was packaged into a managed solution, and was deployed by an approved pipeline identity. This gives you the evidence trail for ATO support: commit record, PR + reviewer, scan results, artifact hash, release approval, deployment identity, and production deployment record.

---

## GCC High vs. DoD IL5 — cloud boundary

{: .important }
> Microsoft states that Dynamics 365 Government (DoD cloud) is designed to align with DISA SRG IL5 requirements, while GCC High aligns to IL4. Confirm the exact cloud boundary authorization with your AO before committing to an IL5 architecture. See [Microsoft Learn — Dynamics 365 US Government](https://learn.microsoft.com/en-us/power-platform/admin/microsoft-dynamics-365-government) for current guidance.

This distinction matters for code security because:
- IL5 scanners and pipelines must run inside the approved DoD boundary — typically through self-hosted agents in Azure Government or an approved software factory
- GCC High pipelines can use Azure Government-hosted agents; some DISA-approved commercial SaaS scanning tools may be acceptable
- Confirm with your AO which scanner vendors are authorized for your cloud tier before selecting tooling

---

## Recommended implementation pattern

### 1. Use a controlled source repository

Store all JavaScript, PCF components, web resources, Code Apps, and Power Pages scripts in Git. Require:

- Pull request workflow with at least one reviewer
- Branch protection on `main` — no direct pushes
- Signed commits where policy requires it
- Commit messages that reference the work item or change ticket for audit traceability

### 2. Develop in a non-production environment

Do not write or edit JavaScript directly in a Test, UAT, or Production environment. Follow the standard Dev → Test → UAT → Prod pipeline pattern. Code modified directly in a production Dataverse environment bypasses every security gate in the pipeline and cannot be audited.

### 3. Run automated security gates

At minimum, every PR should pass the following before the artifact is promoted:

| Gate | Tool category | Purpose |
|---|---|---|
| SAST | Static code analysis | Identify insecure code patterns before runtime |
| SCA | Software composition analysis | Dependency and package vulnerability scanning |
| Secret scanning | Credential detection | Prevent API keys and tokens from entering source control |
| License scanning | Compliance | Identify open-source license restrictions |
| Linting | Code quality | Enforce style and catch common errors |
| Unit / build validation | Correctness | Confirm the code compiles and passes unit tests |
| Power Platform Solution Checker | Platform-specific | OWASP-aligned checks for Power Platform solutions |

For IL5, scanners and pipeline agents must run inside the approved boundary — self-hosted agents in Azure Government or an approved software factory. See [DISA DevSecOps guidance](https://public.cyber.mil/devsecops/) for current approved patterns.

### 4. Never store secrets in JavaScript

JavaScript that runs in the browser or in a Power Platform client context is not a secure location for credentials. Never include in web resources, PCF, or Code Apps:

- API keys or client secrets
- Bearer tokens
- Connection strings
- Privileged endpoint URLs
- Any value that grants access to a backend system

Place privileged logic behind:
- **Azure Government APIs** (secured via managed identity or service principal)
- **Custom connectors** (credential stored in the Power Platform connection, not the script)
- **Dataverse plug-ins** (server-side, not exposed to the client)
- **Power Automate flows triggered by the app** (flow credentials are not visible to the browser)

{: .note }
> Custom connectors used to call external or internal APIs require governance review before production use. For custom connector security requirements, authentication standards, and APIM integration guidance, see [DLP Strategy — Custom Connector Governance](../dlp-strategy/#4-custom-connector-governance).

### 5. Package and deploy through ALM

JavaScript must be deployed as part of a **managed solution**, not copied manually into a production environment. Manual copy operations:
- Bypass the pipeline approval gate
- Are not recorded in ADO/GitHub deployment history
- Cannot be reliably rolled back
- Create an audit gap that fails ATO review

### 6. Use government-approved tooling

In IL5, all tooling in the pipeline — including scanners, agents, and artifact stores — must be authorized for the DoD boundary. The DoD DevSecOps model emphasizes automated testing, control gates, guardrails, and continuous monitoring as part of delivery. See [DoD Enterprise DevSecOps Fundamentals](https://dodcio.defense.gov/Portals/0/Documents/Library/DoD%20Enterprise%20DevSecOps%20Fundamentals%20v2.5.pdf) for the current reference model.

---

## Recommended pipeline structure

```
Pull Request
  → npm audit / approved SCA scanner
  → ESLint (or equivalent linter)
  → TypeScript build (if applicable)
  → Unit tests
  → Secret scan
  → SAST scan
  → Dependency / license scan
  → Power Platform Solution Checker
  → Package managed solution artifact
  → Publish artifact
  ↓
Approval gate (reviewer sign-off)
  → Deploy to TEST
  ↓
Approval gate (ISSO or program lead)
  → Deploy to PROD
```

All approval gates are logged. The approver identity, timestamp, and artifact version are recorded in ADO/GitHub and constitute ATO evidence for CM-3 (Configuration Change Control) and SA-3 (System Development Life Cycle).

---

## Relationship to LP-ALM

JavaScript and PCF components are solution artifacts. They follow the same LP-ALM layer placement as any other component:

| Component | LP-ALM layer |
|---|---|
| PCF control definition | `_UI` |
| JavaScript web resource | `_UI` |
| Code App | `_UI` |
| Power Pages custom script | `_UI` |
| Server-side plug-in called by code | `_Automation` or `_Core` |

Source control structure and pipeline YAML for these components follow LP-ALM conventions. See [LP-ALM Methodology](https://devonaleshiremsft.github.io/layered-platform-alm/) for pipeline YAML patterns and the component placement decision tree.
