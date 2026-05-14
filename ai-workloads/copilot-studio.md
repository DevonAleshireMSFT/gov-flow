---
layout: default
title: Copilot Studio Agents
parent: AI Workload Security
nav_order: 2
permalink: /ai-workloads/copilot-studio/
---

# Copilot Studio Agent Governance — GCC High / DoD
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## GCC High availability and the IL5 question

Before building anything in Copilot Studio for a GCC High tenant, answer these two questions. They determine the entire governance approach.

### What is available in GCC High?

| Feature | GCC High Available | IL5 Authorized | Notes |
|---|---|---|---|
| Rule-based topics (classic) | Yes | Yes | No generative AI involved; lowest risk |
| Generative answers (grounding against knowledge source) | Yes | **Verify with AO** | Generative AI; requires explicit ATO authorization |
| AI-based intent recognition (NLU) | Yes | **Verify with AO** | Used by default in new agents; can be disabled |
| Azure OpenAI grounding (Azure Government) | Yes | Yes (with proper config) | Must use Azure Government endpoint — see [Azure AI Integration](../azure-ai-integration/) |
| SharePoint GCC High as knowledge source | Yes | CUI with ISSO review | Data stays within boundary |
| Public web search grounding | **No** | **No** | Routes to commercial Bing; not available in GCC High |
| SharePoint commercial as knowledge source | **No** | **No** | Data exits GCC High boundary |
| Microsoft 365 Copilot (not Copilot Studio) | Separate licensing | Separate ATO | Different product; different governance |
| Copilot extensions / connectors to commercial SaaS | **No** | **No** | Connector data residency issue |

{: .warning }
> **IL5 and generative AI:** As of May 2026, DISA has not issued a blanket authorization for generative AI features in IL5 environments. Any agent that uses generative answers, generative orchestration, or Azure OpenAI grounding in an IL5 environment requires explicit AO authorization before deployment. Rule-based agents without generative features are lower risk but still require ISSO review.

### Decision gate: generative or rule-based?

```
Does the agent use generative answers or generative orchestration?
├── YES → Does it process or have access to CUI?
│   ├── YES → Requires AO authorization for generative AI on CUI
│   │         Requires ISSO review before deployment to any environment above Dev
│   │         Must use Azure Government AI endpoints only
│   └── NO  → Requires ISSO review; document that no CUI is in scope
└── NO  → Rule-based agent
          Still requires ISSO review if deployed in an IL5 or CUI environment
          Lower risk; standard intake process applies
```

---

## Impact on environment strategy

Copilot Studio agents are Dataverse-based components — they live in an environment as solution artifacts, alongside the program's apps and flows. This has direct implications for the [8-tier environment model](../../enterprise-strategy/#22-eight-tier-environment-model).

### Agents belong in the program environment, not a shared agent environment

The instinct to create a "central agent environment" should be rejected for the same reason a "central production environment" is rejected: it creates ATO scope contamination between programs. An HR agent and a logistics agent do not share an ATO. They should not share an environment.

**Recommended topology:**

```
{CMD}-{PGM}-DEV     ← Agent built and iterated here (unmanaged)
{CMD}-{PGM}-TEST    ← Agent deployed here as managed solution; tested against live knowledge sources
{CMD}-{PGM}-PROD    ← Agent in production; managed solution only; access restricted by channel
```

The agent's Dev/Test/Prod environments are the **same environments** as the application it serves. There is no separate agent environment unless:

- The agent serves multiple programs (e.g., a platform-level CoE help agent) → deploy to `PLATFORM-SHAREDSVC-PROD`
- The agent has its own ATO boundary (rare) → provision dedicated environments through intake

### Sandbox environments and agent testing

Sandbox environments are appropriate for initial agent exploration. Restrictions still apply:
- No production data in sandbox — no connection to production SharePoint sites
- No IL5 data in sandbox — sandbox environments are explicitly excluded from ATO coverage
- Generative features in sandbox: permitted for non-CUI exploration; document that sandbox agent is separate from any production ATO

---

## LP-ALM layer placement for agent components

Agents are solution components and must follow the LP-ALM model.

| Component | Layer | Notes |
|---|---|---|
| Agent (bot) definition | `_UI` | The agent is a user-facing interface component |
| Agent topics and dialogs | `_UI` | Part of the agent definition |
| Custom actions that call flows | `_Automation` | The underlying flow belongs in `_Automation`; the agent invokes it |
| Tables used by agent (e.g., FAQ table, KB records) | `_Core` | Schema belongs in `_Core` regardless of what consumes it |
| Connection references used by agent actions | `_Automation` | Same rules as any connection reference |
| Knowledge source configuration (SharePoint site URL, AI Search index) | `_Config` | Environment-specific values — SharePoint URL differs between Dev and Prod |
| Agent channel configuration (Teams app ID, embed token) | `_Config` | Channel configuration differs by environment |

{: .important }
> **Knowledge source URLs are `_Config` values.** The SharePoint site URL for a knowledge source differs between Dev and Prod (Dev points to a test SharePoint site; Prod points to the authorized production site). Do not hardcode these in the agent configuration. Configure them as environment variables applied at the `_Config` step.

### When to add an `_Agent` layer

Consider a separate `_Agent` layer (positioned between `_Automation` and `_UI`) when:
- The agent has a significantly different release cadence than the application's UI components
- Multiple downstream applications consume the same shared agent
- The agent team and the app team are different people with different PR approval chains

Apply the same LP-ALM extension principle: does it have a distinct ownership boundary or release cadence? If yes, it earns its own layer.

---

## RBAC and access control for agents

### Who can create and publish agents

Access to Copilot Studio is controlled through the environment's Managed Environments settings. Configure via Power Platform Admin Center → Environment → Managed Environments → Copilot Studio:

| Setting | Recommended value | Rationale |
|---|---|---|
| Allow makers to create agents | Restricted to security group | `PP-{ENV}-Developers` only; not all environment users |
| Allow makers to publish agents | Restricted to security group | Publishing to production channels requires explicit authorization |
| Allow generative AI features | Enabled for Dev; **ISSO approval required** for Test/Prod | Generative features require ATO authorization before reaching upper environments |

### Who can talk to the agent

Agent access is controlled at the channel level, not the environment level. Every production agent must restrict access:

| Channel | GCC High available | Recommended access control |
|---|---|---|
| Microsoft Teams (GCC High) | Yes | Restrict to specific Teams group or security group; do not publish to the entire tenant |
| Embedded in canvas app | Yes | Canvas app RBAC controls who accesses the app; agent inherits the app's user population |
| Direct line (custom web embed) | Yes | Require Azure Government AD authentication; anonymous access not authorized for CUI workloads |
| SharePoint GCC High web part | Yes | SharePoint site permissions control access |
| Email (Power Automate trigger) | Yes | Flow-level access control applies |
| Power Pages | Verify GCC High | Check Power Pages GCC High availability before designing this channel |
| Public website embed | **No for CUI** | Anonymous access to CUI data; not authorized |

{: .important }
> **No anonymous channels for CUI agents.** Any agent that can access CUI data through its knowledge sources or Dataverse queries must require Azure Government AD authentication on every channel it is published to. An authenticated-at-the-app + anonymous-at-the-agent architecture creates a gap. Both layers must require authentication.

### Authentication configuration

In Copilot Studio → Settings → Security:

```
Authentication:  Azure Active Directory (Azure Government)
Tenant:          {your GCC High tenant ID}
Client ID:       {app registration in Azure Government AD}
Require sign-in: Yes — always, for any agent handling CUI
```

Do not use "No authentication" or "Only for Teams and Power Apps" modes for production agents that handle CUI. The Teams-only mode still leaves the Direct Line channel open to anonymous callers with the right token.

---

## Knowledge source governance

Knowledge sources are the data pipeline of the agent. Govern them like any other data integration.

### SharePoint GCC High as knowledge source

This is the standard, authorized knowledge source for internal document grounding.

- Only index **approved SharePoint sites** — document the sites included in the knowledge source in the ATO data flow diagram
- The Copilot Studio service accesses SharePoint using the **signed-in user's credentials** for retrieval (not a service account) — users will only see documents they have access to in SharePoint
- For production agents, ensure the SharePoint sites referenced are finalized and stable — knowledge source URLs are `_Config` values and changing them requires a `_Config` re-application

### Azure AI Search (Azure Government) as knowledge source

For enterprise-scale document retrieval or when documents span multiple sources, Azure AI Search is the preferred pattern. See [Azure AI Integration — RAG pattern](../azure-ai-integration/#rag-pattern-power-platform--azure-ai-search--azure-openai-gcc-high).

- AI Search index must be in Azure Government
- Use managed identity for the connection between Copilot Studio and AI Search
- Index configuration (endpoint, index name, key if not using managed identity) = `_Config` values

### Dataverse as knowledge source

Agents can query Dataverse tables directly through custom actions (flows) or via the built-in Dataverse knowledge source feature.

- The agent's Dataverse queries are subject to the security roles of the **signed-in user** — the agent cannot access records the user cannot access
- For automated/background operations (not user-facing queries), the flow uses a service account, which is subject to the `{ProjectCode} - Automation Service` security role — scope this role carefully
- Do not grant the Automation Service role Organization-level read access to sensitive tables just to support an agent — scope to the minimum records required

### What to never use as a knowledge source

| Source | Why not |
|---|---|
| Public web URLs (Bing grounding) | Not available in GCC High; routes to commercial Bing |
| Commercial SharePoint (`*.sharepoint.com`) | Data exits GCC High boundary from the indexing crawl |
| Commercial OneDrive | Same boundary issue |
| Public GitHub repositories | Code and document content may be inadvertently indexed; no access control |
| Commercial SaaS (Salesforce, ServiceNow commercial) | Connector data residency issue; not authorized for CUI |

---

## DLP policy implications for Copilot Studio

Copilot Studio agents use connectors when executing custom actions. Those connectors are subject to the same DLP policies as any Power Automate flow.

**Additional DLP considerations specific to agents:**

1. **The Copilot Studio connector** (used for agent-to-agent communication or external Copilot access) must be classified in the **Business** group or blocked. Review whether your DLP policies allow or block it.

2. **Custom actions in agents invoke flows**, which invoke connectors. The DLP policy applies at the connector level regardless of whether the trigger is a flow or an agent action.

3. **Generative orchestration** (where the agent autonomously selects which actions to call based on user intent) expands the possible execution paths. For CUI workloads, prefer **explicit topic-based orchestration** over fully autonomous generative orchestration — it produces a more predictable data flow that is easier to document for ATO.

4. **Agent-to-agent calls**: If one agent invokes another agent (multi-agent patterns), each agent's access boundaries must be independently reviewed. A high-privilege agent calling a low-privilege agent does not elevate the second agent's access — but the reverse is a risk. Do not build patterns where a lower-trust user-facing agent can invoke a higher-privilege backend agent without explicit authorization checks.

---

## Lifecycle management and governance

### Before deployment to production (checklist)

- [ ] Agent classified under the [application classification model](../../enterprise-strategy/#42-application-classification-model) (Mission Critical, Business Critical, Standard, or Citizen)
- [ ] ISSO review complete for any agent handling CUI or using generative features
- [ ] AO authorization obtained for generative AI features in IL5 environments
- [ ] Authentication mode confirmed: Azure Government AD authentication required; anonymous channels disabled
- [ ] Knowledge source URLs documented as `_Config` values (not hardcoded)
- [ ] Agent included in the program's LP-ALM `_UI` solution
- [ ] Pipeline validated: agent deploys as managed solution to Test and Prod
- [ ] Production channel publishing restricted to approved security group
- [ ] Data flow diagram updated to include all knowledge sources and AI service calls
- [ ] Content filtering confirmed for any Azure OpenAI-grounded agent
- [ ] No-training commitment verified in Microsoft enterprise agreement documentation

### Monitoring agents in production

Copilot Studio provides conversation analytics. For government environments:

- Route conversation telemetry to **Azure Government Application Insights** (not commercial App Insights)
- Monitor for: failed authentication attempts, high abandonment rates, content filter triggers, unexpected topic escalations
- Include agent conversation logs in the program's audit log retention policy (NARA schedule 2 minimum)
- Review agent topic coverage quarterly — unused topics accumulate and obscure the agent's actual behavior profile

### Decommissioning an agent

Agents left in production with no active owner are a governance risk — stale knowledge sources may return outdated or incorrect information, and the agent remains a data access path even if nobody is actively maintaining it.

Apply the same lifecycle rules as applications:
- Annual activity review: agents with < 10 conversations in 90 days are flagged for owner confirmation
- Owner departure: agent ownership transferred immediately; pipeline service account connections re-bound
- Program end: agent decommissioned with the program environment (within 30 days of ATO lapse or program closure)
- Knowledge source retirement: when a SharePoint site is archived, remove it from the agent's knowledge configuration before archival to prevent stale indexing

---

## Anti-patterns specific to Copilot Studio in GCC High

| Anti-pattern | Why it fails | Correct approach |
|---|---|---|
| Connecting agent to commercial SharePoint knowledge source | Data exits GCC High during the indexing crawl; compliance failure | Use SharePoint GCC High only; verify `.sharepoint.us` domain |
| Publishing agent with "No authentication" to handle CUI questions | Anonymous callers can access CUI data through the agent's knowledge sources | Require Azure Government AD authentication on all channels for CUI agents |
| Hardcoding SharePoint site URL in agent knowledge source configuration | URL cannot change without a solution modification; Dev and Prod must point to different sites | Use environment variable → `_Config` for all knowledge source URLs |
| Using generative orchestration for IL5 workloads without AO authorization | Generative AI features require explicit authorization; deploying without it is an ATO violation | Rule-based topics only until AO authorization is obtained; document clearly in ATO package |
| Building agent topics that aggregate CUI fields into a single response | Aggregated PII/CUI in a conversational response is harder to audit and control than structured app output | Design for minimum necessary disclosure; return references to records, not full field values |
| Sharing one Copilot Studio agent across programs with different ATOs | Agent becomes a cross-ATO data access path; ISSO cannot bound the authorization scope | One agent per program ATO boundary; no shared production agents across ATO boundaries |
| Using the default system topic "Escalate to Agent" without configuring a human handoff target | Escalation may route to an unapproved external helpdesk system | Configure human handoff explicitly (Teams channel or Dynamics Customer Service in GCC High) or disable the topic |
