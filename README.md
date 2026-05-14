# GovFlow

**Government Federated Low-Code Operations Framework**

📖 **[govflowhq.com](https://govflowhq.com)**

---

## What is gov-flow?

gov-flow is an enterprise governance framework for organizations running Microsoft Power Platform in GCC High and DoD tenants. It covers the platform layer — the decisions, policies, structures, and controls that must be in place before programs build solutions.

It is designed to complement [LP-ALM](https://devonaleshiremsft.github.io/layered-platform-alm/), a separate methodology covering solution-level ALM inside the artifact. The two frameworks operate at different layers of the stack and are intended to be used together.

| Layer | Framework | Covers |
|---|---|---|
| Platform | **GovFlow** | Environments, DLP, RBAC, CoE, ATO governance, AI workload security |
| Solution | [LP-ALM](https://devonaleshiremsft.github.io/layered-platform-alm/) | Solution layers, pipelines, component placement, onboarding |

---

## Who is this for?

- **Platform engineers** standing up or maturing an Azure Government / GCC High Power Platform tenant
- **ISSOs and AOs** looking for control mappings and ATO evidence guidance
- **Program leads** onboarding new solutions into a governed tenant
- **CoE teams** responsible for platform standards and operational support

---

## What's inside

| Section | Description |
|---|---|
| [Getting Started](https://govflowhq.com/getting-started/) | Pre-implementation decisions, first 30-day checklist, where to start |
| [Enterprise Strategy](https://govflowhq.com/enterprise-strategy-gcc-high/) | 8-tier environment model, security architecture, governance model, phased roadmap |
| [Governance Templates](https://govflowhq.com/governance-templates/) | PGB charter, environment register, intake checklist, ATO evidence guide |
| [AI Workload Security](https://govflowhq.com/ai-workloads/) | Azure AI integration (GCC High), Copilot Studio agent governance |

---

## Relationship to LP-ALM

gov-flow does not cover solution ALM methodology — that is LP-ALM's domain. If you are a developer building a Power Platform solution inside a gov-flow governed tenant, start with LP-ALM for pipeline structure, component placement, and the onboarding checklist. gov-flow defines the platform environment those solutions deploy into.

---

## Contributing

This framework is maintained as a living document. Issues and pull requests are welcome. Content is scoped to GCC High / DoD governance — commercial-only scenarios are out of scope.

---

## License

Content is provided as reference guidance. See [LICENSE](LICENSE) for terms.
