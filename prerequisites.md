---
layout: default
title: Prerequisites & Tooling
nav_order: 2
permalink: /prerequisites/
---

# Prerequisites & Tooling
{: .no_toc }

What you need installed, what access you need to request, and what to expect in a DoD GFE environment before you can implement GovFlow.
{: .fs-5 .fw-300 }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Overview

In a commercial environment, a platform engineer can install tools and request admin access within hours. In a DoD organization, software installation requires an approved request, PowerShell execution policies are locked by GPO, and admin roles require PIM elevation through an approval workflow. None of this is a barrier — but failing to account for it adds weeks of unnecessary delay to a platform standup.

This page covers what you need, what to request, and how to navigate common DoD-specific constraints.

---

## Required tooling

### Visual Studio Code

VS Code is the primary authoring environment for GovFlow platform work — solution configuration, YAML pipelines, environment variable management, and documentation.

| Item | Details |
|---|---|
| Download | [code.visualstudio.com](https://code.visualstudio.com/) |
| GFE install | Submit a software request through your org's IT portal; VS Code is widely approved |
| Portable version | If installation is blocked, the portable `.zip` version can run from a user-writable directory without admin rights |

**Required extensions:**

| Extension | Publisher | Purpose |
|---|---|---|
| Power Platform Tools | Microsoft | PAC CLI integration, solution explorer, environment management |
| YAML | Red Hat | Pipeline YAML authoring and validation |
| GitLens | GitKraken | Git history and blame — useful for tracking solution change ownership |
| Markdown All in One | Yu Zhang | For editing GovFlow documentation and runbooks |

Install extensions via the VS Code Extensions panel (`Ctrl+Shift+X`). On GFE with restricted marketplace access, extensions can be downloaded as `.vsix` files and installed offline: `Extensions → ... → Install from VSIX`.

---

### Power Platform CLI (PAC CLI)

PAC CLI is required for all pipeline operations, solution export/import, environment management, and admin automation described in this framework.

**Installation options:**

```powershell
# Option 1: Via .NET tool (recommended if .NET SDK is available)
dotnet tool install --global Microsoft.PowerApps.CLI.Tool

# Option 2: Via npm (requires Node.js)
npm install -g pac

# Option 3: MSI installer
# Download from: https://aka.ms/PowerAppsCLI
# Submit as a software request if MSI installation requires admin
```

**GFE/GPO considerations:**

- `.NET tool install` and `npm install -g` both install to user-profile directories and typically do not require elevation
- The MSI installer requires local admin — submit a software request if needed
- Verify installation: `pac help`

**GCC High authentication:**

PAC CLI must be pointed at the GCC High endpoint, not commercial:

```powershell
pac auth create --environment https://gcc.admin.powerplatform.microsoft.us --cloud UsGov
```

For IL5 environments, use `--cloud UsGovDoD`.

---

### PowerShell modules

Several GovFlow operational tasks use PowerShell. All modules install to the current user profile and do not require elevation.

```powershell
# Power Platform Admin module
Install-Module -Name Microsoft.PowerApps.Administration.PowerShell -Scope CurrentUser -Force

# Power Apps maker module (for app-level operations)
Install-Module -Name Microsoft.PowerApps.PowerShell -Scope CurrentUser -Force

# Azure PowerShell (for Azure Government resource management)
Install-Module -Name Az -Scope CurrentUser -Force -AllowClobber
```

**Execution policy on DoD GFE:**

PowerShell execution policy is commonly set to `Restricted` or `AllSigned` by GPO on DoD workstations, which blocks running scripts.

```powershell
# Check current policy
Get-ExecutionPolicy -List

# Set for current user only (does not require elevation, may be overridden by GPO)
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

If GPO prevents changing execution policy, run scripts via:
```powershell
powershell.exe -ExecutionPolicy Bypass -File .\your-script.ps1
```

Document this in your team's runbook — it is a common onboarding blocker.

---

### Git client

All solution source control and pipeline configuration requires Git.

| Item | Details |
|---|---|
| Git for Windows | [git-scm.com](https://git-scm.com/) — portable version available if MSI install requires admin |
| Azure DevOps Git | If using ADO, configure the Git credential manager for GCC High: `git config --global credential.authority AAD` |
| GitHub (GCC High) | GitHub Enterprise or GitHub.com with GCC High tenant SSO depending on your org's configuration |

---

### Azure CLI (optional but recommended)

Required if managing Azure Government resources (Azure OpenAI, AI Search, Key Vault) directly from the command line.

```powershell
# Install via MSI (requires admin) or via Python pip (user-level)
pip install azure-cli --user

# Connect to Azure Government
az cloud set --name AzureUSGovernment
az login
```

---

## Access requirements by role

Access in a DoD GCC High tenant is not self-service. Request access through your organization's identity and access management process (typically a help desk ticket or ServiceNow request with supervisor approval).

### Platform CoE Engineer

| Access required | Scope | How to request |
|---|---|---|
| Power Platform Administrator | Entra ID role (PIM-eligible) | IAM request — requires supervisor + ISSO approval at most organizations |
| Environment Admin | Individual environments during standup | Granted by Power Platform Admin; no separate ticket needed |
| Azure Government Contributor | Platform subscription or resource group | Azure RBAC request through your Azure team |
| ADO / GitHub pipeline access | Organization or project level | Pipeline team administrator grants access |
| PAC CLI authentication | Requires Power Platform Admin or Environment Admin role | Inherited from above |

### Pipeline service account

| Access required | Scope | Notes |
|---|---|---|
| Environment Maker (at minimum) | All non-production environments | See [Service Account Security](../service-account-security/) |
| Environment Admin | Production environments (pipeline only) | Elevated scope — justify in ATO |
| Dataverse security roles | Per environment, minimum necessary | See [Service Account Security](../service-account-security/) |

### ISSO / Security reviewer

| Access required | Scope | Notes |
|---|---|---|
| Power Platform Admin (read-only view) | Tenant | Use Global Reader + Security Reader instead of full admin where possible |
| Azure Government Reader | Platform subscription | Read-only; no PIM required |
| CoE Starter Kit dashboards | Power BI workspace | Share read access to the CoE reports workspace |

### Break-glass / emergency account

| Access required | Scope | Notes |
|---|---|---|
| Global Administrator | Tenant | PIM not applicable — permanently assigned for emergency use; stored in PAW or CyberArk |
| MFA exclusion | Account level | Must have alternative MFA (FIDO2 key) — not tied to a personal device |

---

## PIM (Privileged Identity Management) patterns

{: .important }
> Permanently assigned Power Platform Administrator or Global Administrator roles are an ATO finding in most DoD environments. Use PIM-eligible assignments with just-in-time activation.

### Which roles should be PIM-eligible

| Role | Permanently assigned? | PIM eligible? | Typical activation window |
|---|---|---|---|
| Global Administrator | No (except break-glass) | Yes | 2–4 hours |
| Power Platform Administrator | No | Yes | 8 hours |
| Azure Subscription Owner | No | Yes | 4–8 hours |
| Environment Admin (specific env) | Acceptable for service accounts | Yes for humans | 8 hours |
| Environment Maker | Yes (acceptable) | N/A | — |
| Dataverse security roles | Yes | N/A | — |

### Activation workflow

1. Navigate to [myaccess.microsoft.us](https://myaccess.microsoft.us) (GCC High) or the Entra ID PIM portal
2. Select **Eligible assignments** → find the role → **Activate**
3. Provide a justification (e.g., "Platform standup — provisioning program environments per intake ticket #1234")
4. Approval is routed to the configured approver (typically ISSO or supervisor)
5. Activation is logged to the Entra ID audit log — this is part of your ATO evidence for AC-2 and AC-6

### PIM for pipeline service accounts

Pipeline service accounts should **not** use PIM — automated pipelines cannot interactively approve an elevation request. Instead:
- Grant the minimum permanent role the pipeline needs to function
- Document the permanent assignment and its justification in the service account register
- Review quarterly and remove access that is no longer required

---

## Network and endpoint access

GCC High endpoints must be reachable from your workstation. The following are commonly blocked by DoD proxy or firewall configurations and may require a firewall exception request.

| Endpoint | Purpose | Required for |
|---|---|---|
| `https://gcc.admin.powerplatform.microsoft.us` | Power Platform Admin Center | Admin operations, PAC CLI |
| `https://*.crm9.dynamics.com` | GCC High Dataverse environments | PAC CLI, Power Automate, model-driven apps |
| `https://*.api.powerplatform.com` | Power Platform API | PAC CLI admin commands |
| `https://login.microsoftonline.us` | GCC High authentication | All GCC High services |
| `https://dev.azure.com` or your ADO GCC High URL | Azure DevOps | Pipeline execution |
| `https://management.usgovcloudapi.net` | Azure Government management | Azure CLI, Az PowerShell |
| `https://*.openai.azure.us` | Azure Government OpenAI | AI workload operations |

{: .note }
> If you are working through a DoD proxy, configure PAC CLI and Azure CLI to use it:
> ```powershell
> $env:HTTPS_PROXY = "http://your-proxy:port"
> az configure --defaults proxy="http://your-proxy:port"
> ```

---

## Onboarding checklist for new platform engineers

- [ ] VS Code installed with Power Platform Tools, YAML, and GitLens extensions
- [ ] PAC CLI installed and authenticated to GCC High tenant (`pac auth list`)
- [ ] Power Platform Admin PowerShell module installed (`Get-AdminPowerAppEnvironment` runs without error)
- [ ] Git configured with name and email (`git config --global user.name`, `git config --global user.email`)
- [ ] PIM-eligible Power Platform Administrator role assigned (ticket submitted and approved)
- [ ] ADO/GitHub access granted to the platform repository
- [ ] Access to CoE Starter Kit Power BI workspace confirmed
- [ ] Azure Government subscription access confirmed (if managing Azure AI resources)
- [ ] Break-glass account documented and tested (platform lead only)
- [ ] Firewall/proxy exceptions confirmed for GCC High endpoints listed above
