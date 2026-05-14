---
layout: default
title: AI Workload Security
nav_order: 10
has_children: true
permalink: /ai-workloads/
---

# AI Workload Security — GCC High / DoD
{: .no_toc }

Power Platform AI capabilities introduce data flow and authorization patterns that are not addressed by the baseline governance model. This section covers the two distinct AI workload categories present in GCC High environments and the governance decisions each requires.

{: .warning }
> **CUI data must never leave the Azure Government / GCC High boundary.** This is the governing constraint for all AI workload decisions in this section. Commercial Azure OpenAI, commercial Bing grounding, and commercial Microsoft Graph AI endpoints are not authorized for workloads handling CUI.

---

## Two categories of AI workload

| Category | What it covers | Primary risk |
|---|---|---|
| [**Azure AI Integration**](azure-ai-integration/) | Custom connectors to Azure Government OpenAI · Azure AI Foundry · Azure AI Search · RAG pipelines from Power Automate flows | Data boundary — CUI routed to wrong endpoint; API keys in source control |
| [**Copilot Studio Agents**](copilot-studio/) | Copilot Studio bots and agents · Generative answers · Knowledge sources · Custom actions · Channel publishing | Generative AI authorization for IL5; agent access control; knowledge source data residency |

---

## Governing principle: AI components are not exempt from governance

An AI-powered flow is still a flow. An agent with a SharePoint knowledge source is still a data integration. The LP-ALM layer model, the DLP policy architecture, and the RBAC pyramid from the [Enterprise Strategy](../enterprise-strategy/) all apply to AI components without exception.

What AI adds is a new set of **data flow paths** — between Power Platform and external AI services — that must be mapped, authorized, and included in ATO evidence.

```
Power Platform (GCC High)
    │
    ├── Flow / Custom Connector
    │       ↓
    │   Azure Government OpenAI      ← authorized path for CUI
    │
    ├── Copilot Studio Agent
    │       ↓
    │   SharePoint GCC High          ← authorized knowledge source
    │       ↓
    │   Azure AI Search (Gov)        ← authorized for enterprise RAG
    │
    └── ❌ Commercial Azure OpenAI   ← NOT authorized for CUI
        ❌ Public web grounding      ← NOT authorized for CUI workloads
        ❌ Commercial SharePoint     ← NOT authorized from GCC High
```

---

## ATO impact of AI components

Adding AI to an existing Power Platform workload changes the ATO scope. At minimum, the following must be addressed before any AI-enabled feature reaches production:

| Item | Required for |
|---|---|
| Data flow diagram updated to include AI service endpoints | All AI workloads |
| ISSO review of AI-specific data handling | All AI workloads |
| Content filtering configuration documented | Azure OpenAI integration |
| Azure Government AI service included in authorization boundary | Azure OpenAI integration |
| Generative AI feature authorization confirmed by AO | Copilot Studio generative answers |
| IL5 AI authorization documented | IL5 environments only |
| No-training-on-data commitment documented | All AI workloads (Microsoft enterprise agreement covers this) |
