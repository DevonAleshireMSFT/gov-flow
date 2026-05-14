---
layout: default
title: Azure AI Integration
parent: AI Workload Security
nav_order: 1
permalink: /ai-workloads/azure-ai-integration/
---

# Azure AI Integration — GCC High
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The core constraint

{: .important }
> All Azure AI services called from GCC High Power Platform must reside in **Azure Government** (`*.microsoft.us` or `*.usgovcloudapi.net`). There is no authorized path for a GCC High flow to call commercial Azure OpenAI (`api.openai.com` or `cognitiveservices.azure.com`) when processing CUI. This is a data boundary requirement, not a preference.

This single constraint drives every architecture decision in this section.

---

## Azure Government AI service availability

Not every Azure AI capability available in commercial is available in Azure Government. Validate before designing a solution.

| Service | Azure Government Availability | Notes |
|---|---|---|
| Azure OpenAI Service | Available | Endpoint: `{resource}.openai.azure.us` |
| GPT-4o | Check current availability | Model catalog differs from commercial; validate in `portal.azure.us` |
| GPT-4 (0613, turbo) | Available | Most stable choice for GCC High |
| GPT-3.5 Turbo | Available | — |
| text-embedding-ada-002 | Available | Required for RAG patterns with Azure AI Search |
| Azure AI Search | Available | Endpoint: `{resource}.search.azure.us` |
| Azure AI Foundry | Available (limited) | Validate specific features against Azure Government roadmap |
| Azure AI Content Safety | Available | — |
| Azure Document Intelligence | Available | Endpoint: `{resource}.cognitiveservices.azure.us` |
| Bing grounding / web search | **Not available** | Do not use for any workload; routes to commercial |

{: .warning }
> Model availability in Azure Government changes on a different cadence than commercial. Validate model availability in `portal.azure.us` → Azure OpenAI → your resource → Model deployments **before** committing to a model in a program's architecture.

---

## Reference endpoints (GCC High / Azure Government)

Always use these endpoints in `_Config` environment variables. Never hardcode in flows or connectors.

```
Azure OpenAI:          https://{resource-name}.openai.azure.us/
Azure AI Search:       https://{resource-name}.search.azure.us/
Azure AI Foundry:      https://{project}.services.ai.azure.us/
Cognitive Services:    https://{resource-name}.cognitiveservices.azure.us/
Azure Key Vault:       https://{vault-name}.vault.usgovcloudapi.net/
Azure Blob Storage:    https://{account}.blob.core.usgovcloudapi.net/
```

These are `_Config` values — they differ between environments (Dev may point to a lower-capacity deployment, Prod to the authorized production resource). Never commit them to source control.

---

## Authentication: managed identity over API keys

{: .important }
> Managed Identity is the required approach for Prod connections from Power Platform to Azure Government AI services. API keys are acceptable only in Dev environments and must rotate on a 90-day schedule minimum when used.

### Why managed identity

API keys stored as environment variables in `_Config` are not in source control (correct), but they still create a rotation burden, an audit gap, and a single point of failure. When the key rotates, `_Config` must be manually re-applied to every affected environment.

Managed Identity eliminates the credential entirely. Power Platform connects to Azure using the environment's system-assigned managed identity. Authorization is granted via Azure RBAC on the Azure resource — not via a key.

### Setting up managed identity for Power Platform → Azure OpenAI

{: .note }
> Managed Identity for Power Platform environments requires a Premium license and is configured via the Power Platform Admin Center. As of 2026 this is available in GCC High.

**Step 1:** Enable managed identity for the Power Platform environment:
- Power Platform Admin Center (`gcc.admin.powerplatform.microsoft.us`) → Environment → Settings → Enterprise policies → Enable managed identity

**Step 2:** Grant the managed identity access to the Azure Government resource:
```bash
# Assign Cognitive Services OpenAI User role to the Power Platform environment's managed identity
az role assignment create \
  --assignee <managed-identity-object-id> \
  --role "Cognitive Services OpenAI User" \
  --scope /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<openai-resource>
```

**Step 3:** Use the HTTP connector in your flow with managed identity authentication (no key needed):
```
Method: POST
URI: @{variables('AzureOpenAIEndpoint')}/openai/deployments/@{variables('DeploymentName')}/chat/completions?api-version=2024-02-01
Authentication: Managed identity
Audience: https://cognitiveservices.azure.us
```

### When API keys are unavoidable

If managed identity is not yet available for a specific integration path, use this pattern:
- Store the API key in **Azure Government Key Vault** — not in the `_Config` environment variable value
- Retrieve the key at runtime via a Key Vault reference in the environment variable, or via a PAC CLI secret in the pipeline
- Rotate on a 90-day schedule; document the rotation date in the environment register
- Never log the API key value in any flow output or telemetry table

---

## DLP policy configuration for Azure AI connectors

### Connector classification

| Connector | Recommended DLP classification | Notes |
|---|---|---|
| **HTTP with Azure AD** | Business | Used for managed identity calls to Azure OpenAI Gov; restrict endpoint in DLP |
| **HTTP** (generic) | Non-business (blocked) by default | Only enable for specific allowlisted `.microsoft.us` endpoints via Exception policy |
| **Azure OpenAI** (if available in GCC High) | Business | Validate GCC High availability before using |
| **AI Builder** | Business (with caution) | AI Builder model availability in GCC High is limited; validate feature parity |
| Custom connector (Azure OpenAI wrapper) | Business | Preferred pattern — define once, govern once |

### Allowlisting specific Azure Government endpoints

If using the HTTP connector (not recommended for production, but may be needed for custom integrations):

1. Create a **Tier 2 environment group DLP policy** for the specific environments that need AI access
2. In the HTTP connector settings, add an endpoint filter:
   ```
   Allowed endpoints:
   *.openai.azure.us
   *.search.azure.us
   *.cognitiveservices.azure.us
   *.microsoft.us
   ```
3. Block all other HTTP connector endpoints in the same policy
4. Document the exception in the ATO as a DLP policy tier 3 exception if it applies to a single program

{: .warning }
> Do not use an open HTTP connector policy (allow all endpoints). A flow with an unrestricted HTTP connector can exfiltrate CUI to any endpoint. Endpoint allowlisting is non-negotiable for IL5 environments.

---

## LP-ALM placement for AI components

AI-related Power Platform components fit into the existing five-layer model without modification.

| Component | Layer | Notes |
|---|---|---|
| Custom connector definition (Azure OpenAI) | `_Integration` or `_Automation` | Use `_Integration` if shared across programs or has separate release cadence |
| Connection reference (to custom connector) | `_Automation` | Bound to managed identity or service account — not personal credentials |
| Power Automate flows calling AI services | `_Automation` | Same as any other flow |
| Environment variable: AI endpoint URL | Definition in `_Core`, value in `_Config` | Value differs by environment (dev vs prod OpenAI deployment) |
| Environment variable: deployment name | Definition in `_Core`, value in `_Config` | Deployment names differ by environment |
| Environment variable: API key (if used) | Definition in `_Core`, value in `_Config` | Key itself must also be in Key Vault — `_Config` holds the Key Vault reference or key only in Dev |
| Dataverse table for AI response logging | `_Core` | Every production AI call should log to an audit table |
| Canvas app with AI features | `_UI` | No change from standard placement |

---

## AI response logging — ATO requirement

Every flow that sends CUI to an AI service must log the transaction to the `{prefix}_PlatformLog` table (defined in the [Enterprise Strategy Section 8.8](../../enterprise-strategy/#88-logging-and-error-handling-standards)) with the following additions:

```
{prefix}_aimodel        (string)  — model name and version (e.g., gpt-4-0613)
{prefix}_aiendpoint     (string)  — Azure Government endpoint hostname only (not full URL)
{prefix}_promptlength   (integer) — token count of the prompt (not the prompt text)
{prefix}_responselength (integer) — token count of the response (not the response text)
{prefix}_contentfilter  (choice)  — Pass / Filtered / Error
```

{: .important }
> **Do not log prompt content or response content** containing CUI to Dataverse. Log lengths and metadata only. The full prompt and response are transient — they should not be persisted in a searchable Dataverse table. If the workload requires prompt/response audit storage, use Azure Government Blob Storage with appropriate access controls and retention policy, referenced from the Dataverse log record by URL.

---

## Content filtering — required configuration

Azure OpenAI content filtering is enabled by default but must be explicitly documented in the ATO. Do not disable or reduce content filters in production.

Minimum required configuration for production AI deployments in Azure Government:

| Category | Severity threshold | Action |
|---|---|---|
| Hate | Medium | Block |
| Violence | Medium | Block |
| Sexual | Low | Block |
| Self-harm | Low | Block |
| Indirect attack (prompt injection) | Low | Block |
| Groundedness (RAG only) | Enabled | Detect and flag |

Document these settings in the ATO as security control evidence for the AI component. Export the content filtering configuration from `portal.azure.us` as part of the quarterly ATO evidence package.

---

## RAG pattern: Power Platform + Azure AI Search + Azure OpenAI (GCC High)

For workloads that need to ground AI responses in agency documents (the most common enterprise AI use case), the recommended architecture in GCC High:

```
User input (in Canvas App or Copilot Studio)
        ↓
Power Automate flow (_Automation)
        ↓
Azure AI Search (Azure Government)   ← indexes SharePoint GCC High documents
        ↓                                or Azure Government Blob Storage
Retrieve top-K chunks
        ↓
Azure OpenAI GPT-4 (Azure Government) ← receives chunks + user question
        ↓
Structured response returned to flow
        ↓
Log transaction metadata → {prefix}_PlatformLog
        ↓
Response displayed to user
```

**Key implementation notes:**

1. **Indexing source:** SharePoint GCC High → Azure AI Search indexer using managed identity. No data transits commercial endpoints.

2. **Chunking strategy:** Documents chunked at index time (not at query time in the flow). Store chunk metadata in Azure AI Search index, not in Dataverse.

3. **The flow has one responsibility:** Retrieve context from Search, construct the prompt, call OpenAI, return the response, log the transaction. Keep it simple — single child flow per AI call pattern.

4. **Environment variables required (per environment):**
   - `{prefix}_AISearchEndpoint` — Azure AI Search endpoint
   - `{prefix}_AISearchIndex` — Index name (may differ between Dev/Prod)
   - `{prefix}_OpenAIEndpoint` — Azure OpenAI resource endpoint
   - `{prefix}_OpenAIDeployment` — Deployment/model name
   - `{prefix}_OpenAIApiVersion` — API version string

5. **All four above are `_Config` values.** They differ between Dev, Test, and Prod. Dev may point to a lower-capacity OpenAI deployment. Prod points to the PTU or standard deployment with production capacity.

---

## Common mistakes in GCC High AI integration

| Mistake | Why it fails | Prevention |
|---|---|---|
| Using commercial Azure OpenAI endpoint (`cognitiveservices.azure.com`) | Data exits the Azure Government boundary; compliance failure | Validate all endpoints start with `.azure.us` or `.usgovcloudapi.net` before ATO |
| Hardcoding the OpenAI endpoint URL in a flow action | Schema contamination equivalent — the endpoint becomes part of the solution artifact | Always use environment variables; endpoint URL lives in `_Config` only |
| Logging prompt and response content to Dataverse | CUI in prompt text persists indefinitely in a searchable table; retention and access control become audit findings | Log metadata only; full content to governed Blob Storage if required |
| Using personal credentials for the Azure OpenAI connection | Connection breaks on user departure; same pattern as personal connection references | Managed identity or service account only in Test/Prod |
| Not validating model availability before architecture commitment | Program commits to GPT-4o; availability in Azure Government is limited or delayed; program must redesign mid-flight | Validate in `portal.azure.us` before finalizing architecture |
| Open HTTP connector DLP policy | Any flow can call any external endpoint | Endpoint allowlist in DLP; `.microsoft.us` only for AI endpoints |
| Disabling content filtering "for testing" and not re-enabling | Testing config reaches production; ATO finding | Content filtering state is documented in ATO; changes require ISSO sign-off |
