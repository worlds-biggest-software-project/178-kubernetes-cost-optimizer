# Kubernetes Cost Optimizer

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source Kubernetes cost optimiser delivering pod rightsizing, namespace cost allocation, and idle resource detection without the vendor lock-in or savings-percentage pricing of commercial incumbents.

Kubernetes Cost Optimizer is a research-stage project to build a vendor-neutral platform that combines real-time cost allocation with autonomous rightsizing for Kubernetes workloads. It is aimed at platform engineering teams, FinOps practitioners, and infrastructure leaders running Kubernetes at scale who need both visibility and automated enforcement without handing over control to a proprietary SaaS.

---

## Why Kubernetes Cost Optimizer?

- The dominant open-source tools (OpenCost, Goldilocks) provide monitoring or recommendations only; no open-source tool currently offers automated rightsizing enforcement.
- Commercial autonomous optimisers (CAST AI, ScaleOps) require granting significant cluster-level permissions and use savings-percentage pricing models that become expensive at large scale.
- Kubecost's pricing jumps significantly from free OSS to enterprise (from $2,199/mo/cluster), and multi-cluster federation is gated behind the enterprise tier.
- AWS-specialised tools like nOps are not suitable for GKE, AKS, or multi-cloud Kubernetes deployments.
- Most tools assume cloud billing APIs and are weak on on-premises or bare-metal Kubernetes cost allocation.

---

## Key Features

### Cost Allocation and Visibility

- Real-time cost allocation by namespace, deployment, label, service, controller, and pod using the OpenCost data model
- Multi-cloud billing API integration across AWS, GCP, and Azure for node-level cost attribution
- Prometheus metrics export for embedding cost data in existing observability stacks
- Budget tracking with namespace and label cost threshold notifications

### Rightsizing and Optimisation

- CPU and memory rightsizing recommendations derived from VPA-based analysis of actual versus requested resources
- Autonomous rightsizing enforcement with configurable guardrails and disruption limits
- Spot instance optimisation with basic interruption handling and workload migration
- Karpenter integration for node provisioning cost recommendations

### Developer and CI/CD Integration

- REST API with OpenAPI documentation for integration into existing tooling
- PR/CI cost-impact preview surfacing projected cost change for a given resource configuration
- Cost anomaly detection with Slack and email alerting
- MCP server providing AI agent access to cost allocation data

### Advanced and Predictive (Backlog)

- Predictive scaling using historical traffic patterns and external event signals
- Cross-cluster workload placement recommendations based on real-time pricing
- Natural-language cost investigation via LLM integration
- GPU cost allocation and optimisation recommendations
- Unit economics modelling such as Cost Per Customer or Cost Per Feature

---

## AI-Native Advantage

ML-based workload profiling can learn per-service traffic patterns and seasonality to set resource requests and limits dynamically, outperforming static VPA recommendations that lag actual demand. Predictive scale-out provisions capacity before anticipated traffic spikes using historical patterns and external signals such as marketing campaigns. Anomaly detection on namespace cost trends with automatic root-cause attribution surfaces cost regressions in the same pull-request review cycle. Spot interruption prediction integrated with scheduling allows proactive pod migration before preemption, making spot adoption safer for stateful or latency-sensitive workloads.

---

## Tech Stack & Deployment

The project targets self-hosted deployment via a Kubernetes operator and Helm chart, building on the OpenCost data model, Kubernetes VPA and HPA APIs, and Karpenter for node provisioning. Integrations include cloud billing APIs (AWS CUR, GCP BigQuery export, Azure Cost Export), Prometheus for metrics, KEDA for event-driven autoscaling, and the OpenCost Specification (CNCF Incubating) as the cost allocation baseline. An MCP server exposes cost data to AI agents over the Model Context Protocol.

---

## Market Context

The Kubernetes cost management market was valued at $2.23B in 2026 and is projected to reach $5.78B by 2030 at a 26.9% CAGR, with North America accounting for the largest regional share (GII Research, 2026). Incumbent pricing ranges from monitoring tools at $500–$2,500 per cluster per month and Finout from $1,000/mo, up to Kubecost Enterprise from $2,199/mo/cluster, with autonomous optimisers commonly charging 15–20% of generated savings. Primary buyers are platform engineering leads, cloud infrastructure managers, FinOps practitioners, and engineering executives at organisations spending $100K+ per month on Kubernetes compute.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
