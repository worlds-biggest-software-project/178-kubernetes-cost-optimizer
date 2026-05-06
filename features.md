# Kubernetes Cost Optimizer — Feature & Functionality Survey

> Candidate #178 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| CAST AI | SaaS (autonomous optimiser) | Commercial — percentage of savings / custom | https://cast.ai |
| Kubecost (IBM) | OSS + SaaS | Free OSS; Enterprise from $2,199/mo/cluster | https://kubecost.com |
| OpenCost | OSS | Apache 2.0 (CNCF Incubating) | https://opencost.io |
| ScaleOps | SaaS (self-hosted option) | Commercial — custom pricing | https://scaleops.com |
| nOps | SaaS | Commercial — % of savings | https://nops.io |
| Finout | SaaS | Commercial — from $1,000/mo | https://finout.io |
| CloudZero | SaaS | Commercial — custom enterprise | https://cloudzero.com |
| Goldilocks (Fairwinds) | OSS | Apache 2.0 | https://github.com/FairwindsOps/goldilocks |
| Yotascale | SaaS | Commercial — custom pricing | https://yotascale.com |
| Komodor | SaaS | Commercial — custom pricing | https://komodor.com |
| Amnic | SaaS | Commercial — custom pricing | https://amnic.com |

---

## Feature Analysis by Solution

### CAST AI

**Core features**
- Automatic CPU/memory request rightsizing with zero-downtime via in-place pod rightsizing and Live Migration
- Spot instance lifecycle management: automatic interruption handling, spot diversity, on-demand fallback
- Spot interruption prediction up to 30 minutes ahead, with graceful pre-emptive pod migration
- Real-time node scaling and bin-packing without scheduled jobs
- Karpenter instance selection and disruption-budget optimisation
- GPU utilisation maximisation with dynamic GPU sharing and rightsizing
- Node rebalancing across instance types for optimal cost/performance
- Multi-cluster and multi-cloud management from a single console

**Differentiating features**
- Autonomous mode: fully hands-off, continuous optimisation without human approval per action
- Spot interruption prediction (proprietary ML model) — a genuine differentiator versus pure reaction
- Achieved unicorn valuation ($1B+) in January 2026; 2026 State of Kubernetes Optimisation Report underpins market credibility
- Covers all three layers simultaneously: workload rightsizing, node provisioning, and commitment management

**UX patterns**
- Console with cluster-level savings dashboard and per-workload policy controls
- Guardrails allow teams to constrain autonomous actions (e.g. max disruption per deployment)
- Onboarding via Helm chart agent; policies can be set to "observe" before enabling automation
- Savings reports show before/after cost per workload

**Integration points**
- REST API at https://api.cast.ai/v1 with OpenAPI spec
- Pulumi SDK for infrastructure-as-code provisioning
- Helm chart Kubernetes agent (castai/helm-charts)
- AWS, GCP, Azure cloud integrations
- Prometheus metrics export

**Known gaps**
- Requires granting significant cluster-level permissions, raising trust concerns for security-sensitive teams
- Savings-percentage pricing model can become expensive at large scale
- Less suited for on-premises or bare-metal Kubernetes deployments
- GPU optimisation is newer and less mature than CPU/memory features

**Licence / IP notes**
- Proprietary SaaS; no open-source components in the core platform
- Spot interruption prediction algorithm is a trade secret / proprietary IP

---

### Kubecost (IBM)

**Core features**
- Cost allocation by namespace, deployment, label, service, controller, and pod
- Multi-cluster, multi-cloud unified cost view (AWS, GCP, Azure, on-premises)
- Real-time cost monitoring via Prometheus integration
- Rightsizing recommendations based on actual usage vs. requested resources
- Budget tracking with threshold alerts
- Savings recommendations (abandoned workloads, oversized nodes, underutilised namespaces)
- Grafana dashboard integration for cost metrics

**Differentiating features**
- Deep native Kubernetes cost model (allocates shared costs like network egress, load balancers proportionally)
- IBM integration gives enterprise buyers a trusted, supported product
- OpenCost-compatible data model: allocation methodology is open and auditable
- 5-minute installation claim; minimal onboarding friction

**UX patterns**
- Web dashboard surfacing cost by any Kubernetes dimension
- Cost allocation reports exportable for chargeback / showback workflows
- Alert rules configurable on namespace or label cost thresholds
- Grafana panels for embedding cost metrics in existing observability stacks

**Integration points**
- REST API (port 9090 /model/ prefix); Allocation API, Savings API, Cloud Cost API
- OpenAPI swagger.json available in the kubecost GitHub repo
- Prometheus for metrics; Grafana for visualisation
- Cloud billing API integrations (AWS CUR, GCP BigQuery export, Azure Cost Export)
- Port.io software catalogue integration

**Known gaps**
- No autonomous enforcement: recommendations only, no auto-apply
- Pricing jumps significantly from free OSS to enterprise tier
- IBM ownership adds uncertainty for smaller teams wary of enterprise vendor lock-in
- Multi-cluster federation requires the more expensive enterprise tier

**Licence / IP notes**
- Open-source core (Apache 2.0, maintained by IBM/Kubecost team)
- Enterprise features (multi-cluster, SSO, RBAC, long-term storage) are proprietary add-ons

---

### OpenCost

**Core features**
- Real-time cost allocation by cluster, node, namespace, controller, service, and pod
- Multi-cloud asset cost monitoring (AWS, GCP, Azure, on-premises)
- OpenCost Specification: vendor-neutral open standard for Kubernetes cost allocation and chargeback
- REST API with OpenAPI swagger.json; Allocation API and Cloud Cost API endpoints
- Built-in Model Context Protocol (MCP) server for AI agent access to cost data (2026)
- KubeModel Data Model 2.0 for more precise cost tracking (introduced 2026)
- Plugin architecture for extensible cost source integrations

**Differentiating features**
- The only CNCF Incubating standard for Kubernetes cost monitoring — provides legitimacy as an ecosystem baseline
- MCP server enables AI agents to directly query cost allocation data in a standardised way
- Supported by AWS, Google, Microsoft, Adobe, Red Hat and other major vendors
- Zero cost for the monitoring layer; strong community governance

**UX patterns**
- Lightweight web UI bundled with the deployment
- API-first: most sophisticated users interact via the REST API or CLI
- Prometheus-compatible metrics for embedding in existing dashboards

**Integration points**
- REST API (port 9003; kubectl port-forward); Allocation and Cloud Cost endpoints
- OpenAPI swagger.json: https://github.com/opencost/opencost/blob/develop/docs/swagger.json
- MCP server for LLM/agent integration
- Prometheus for metrics scraping
- Cloud billing API integrations (AWS, GCP, Azure)

**Known gaps**
- Monitoring only — no rightsizing recommendations or automated optimisation actions
- Swagger file is partially out of date for newer endpoints
- UI is minimal; most production use pairs it with Grafana or Kubecost's UI layer
- No alerting or budget-tracking out of the box

**Licence / IP notes**
- Apache 2.0; CNCF Incubating project with open governance
- OpenCost Specification is a free, open standard with multi-vendor support — no patent or IP concerns

---

### ScaleOps

**Core features**
- Fully autonomous, real-time CPU/memory request rightsizing (no manual approval required)
- Dynamic min/max replica management for HPA alongside rightsizing
- Workload context-awareness: accounts for burstiness, Pod Disruption Budgets, statefulness, noisy neighbours
- Spot instance adoption and workload migration automation
- Karpenter instance selection and disruption-budget optimisation
- GPU sharing and automated GPU workload rightsizing
- Node-level bin-packing to eliminate unevictable-pod waste
- Compatible with existing HPA, KEDA, and Karpenter configurations (gradual rollout supported)

**Differentiating features**
- Self-described "industry-first" all-in-one resource management platform combining vertical and horizontal optimisation in one system
- Context-aware intelligence adapts to application behaviour without manual configuration
- $130M Series C raised in March 2026, the largest recent raise in this category
- Claims up to 80% cost reduction
- Self-hosted deployment option for security-sensitive environments

**UX patterns**
- Policy-based guardrails define how aggressively the platform can act per workload class
- Out-of-the-box operation: minimal configuration required to activate
- Progressive rollout: observe mode before enabling automation, workload-by-workload

**Integration points**
- Kubernetes operator-based deployment
- Compatible with AWS, GCP, Azure managed Kubernetes
- Red Hat OpenShift certified
- AWS Marketplace listing
- HPA, KEDA, and Karpenter integration (non-conflicting by design)

**Known gaps**
- Limited public API documentation; primarily a managed-platform experience
- No native cost-allocation or chargeback reporting (focused purely on resource optimisation)
- Pricing is custom and not publicly available, making budget approval difficult
- Less established than CAST AI or Kubecost in terms of public case studies

**Licence / IP notes**
- Proprietary SaaS / self-hosted commercial licence
- No open-source components in the core platform

---

### nOps

**Core features**
- EKS workload cost allocation down to namespace, workload, and product without manual tagging
- Container rightsizing tailored to workload priorities with in-place EKS rightsizing support
- Spot instance automation with intelligent workload-to-spot matching
- AI-driven Reserved Instance and Savings Plan management (ShareSave): 100% utilisation guarantee
- Dynamic commitment seeding, expansion, and squishing in real time
- Shared cost allocation for load balancers, data transfer, and other shared EKS resources
- Lightweight EKS agent with significantly lower overhead than Kubecost-based alternatives
- Full-stack EKS optimisation: containers, nodes, and pricing layer covered together

**Differentiating features**
- ShareSave commitment management: AI continuously adjusts RI/SP commitments to capture all possible savings, claimed to outperform competitors by 20%
- EKS-first specialisation: deepest coverage of AWS EKS-specific features (auto-mode, in-place resize)
- Savings-percentage pricing aligns vendor incentive with customer outcomes
- Ultra-lightweight agent reduces operational burden vs. Kubecost

**UX patterns**
- Unified dashboard covering EKS costs alongside other AWS spend
- Policy-driven rightsizing: teams set optimisation priorities (cost vs. performance)
- Automated RI/SP purchase and adjustment with minimal human oversight

**Integration points**
- AWS-native integrations: EKS, Cost and Usage Report, Savings Plans, Reserved Instances
- AWS Marketplace listing
- Slack and email alerting for cost anomalies
- API-based commitment management

**Known gaps**
- AWS/EKS-only: not suitable for GKE, AKS, or multi-cloud Kubernetes
- Less feature-rich for cost allocation reporting vs. Kubecost or Finout
- Custom pricing with no public rate card

**Licence / IP notes**
- Proprietary SaaS; no open-source components

---

### Finout

**Core features**
- Unified "MegaBill" consolidating Kubernetes costs with cloud and SaaS spend (Datadog, Snowflake, AWS, GCP, Azure)
- Virtual Tagging: retroactive cost allocation without retagging cloud resources or modifying infrastructure
- Cost allocation down to pod level using existing Prometheus or Datadog monitoring data (no additional agent required)
- CostGuard: automated Kubernetes CPU/memory waste detection with rightsizing simulation
- Budget management and chargeback/showback reporting
- Custom allocation rules manageable by FinOps team without engineering involvement

**Differentiating features**
- Agentless Kubernetes cost allocation using existing observability data: lowest operational overhead of any solution
- Virtual Tagging enables FinOps teams to retroactively correct tagging inconsistencies without engineering cycles
- "Agentic era" FinOps positioning: AI-driven cost analysis and natural-language cost queries (2026)
- Broadest SaaS cost consolidation: includes non-infrastructure spend like Datadog, Snowflake in a single bill

**UX patterns**
- Unified dashboard: one view across all cloud and SaaS spend
- FinOps-team-friendly: allocation rules and virtual tags require no code changes
- Rightsizing recommendations via CostGuard simulations before applying changes

**Integration points**
- Prometheus and Datadog integration for Kubernetes metrics (agentless model)
- AWS, GCP, Azure billing API integrations
- SaaS cost integrations (Datadog, Snowflake, and others)
- REST API for cost data extraction

**Known gaps**
- Rightsizing is advisory only — no autonomous enforcement
- Not a Kubernetes-native tool; K8s features are part of a broader multi-cloud platform
- Less granular Kubernetes optimisation than purpose-built tools (CAST AI, ScaleOps)
- Pricing starts at $1,000/mo, potentially steep for smaller teams

**Licence / IP notes**
- Proprietary SaaS; no open-source components
- Virtual Tagging mechanism is a proprietary implementation

---

### CloudZero

**Core features**
- 100% Kubernetes container cost allocation at hourly granularity by namespace, label, and pod
- Unit economics: Cost Per Customer, Cost Per Product, Cost Per API Token dashboards
- Custom dimension modelling for SaaS business metrics (cost per feature, cost per transaction)
- Real-time anomaly detection with Slack and email alerting
- AI Hub: natural-language cost investigation using LLM integration
- Cloud Efficiency Rate (CER) benchmarking across customer base
- Managing $14B+ in cloud spend across its customer base (2026)
- Claude Code plugin for embedding cost intelligence in developer agentic workflows

**Differentiating features**
- Unit economics as a first-class feature: only platform purpose-built to answer "cost per customer" at engineering-team granularity
- AI Hub integrates LLM-based cost analysis directly into developer tools (Claude Code, Amazon Q)
- CloudZero MCP server for cost intelligence via Model Context Protocol
- Engineering-focused analytics: built for engineering and platform teams, not just FinOps practitioners

**UX patterns**
- Dashboards designed for engineering teams rather than finance; progressive disclosure of business impact
- Anomaly alerts surface cost regressions with context about what changed
- Slack-first alerting for engineers already working in chat-based workflows

**Integration points**
- REST API V2 at docs.cloudzero.com/reference/introduction; key-based auth; JSON responses; paginated
- `/v2/billing/costs` endpoint; 60 req/day rate limit
- CloudZero MCP server (community-built: burkestar/cloudzero-mcp)
- AWS Cost and Usage Report, GCP, Azure billing integrations
- Kubernetes cluster integration for namespace/label cost allocation
- Amazon Q Developer and Claude Code plugin integrations

**Known gaps**
- No automated optimisation or rightsizing — purely an analytics and intelligence platform
- Custom enterprise pricing with no public rate card
- Unit economics modelling requires upfront configuration effort
- Less K8s-specific than purpose-built tools; broader cloud cost platform

**Licence / IP notes**
- Proprietary SaaS; no open-source components
- CloudZero MCP server is community-built (MIT licence)

---

### Goldilocks (Fairwinds)

**Core features**
- Automatic VPA (VerticalPodAutoscaler) object creation for every Deployment in labelled namespaces
- VPA set to `Mode: Off` — recommendations only, no automatic resource mutation
- Web dashboard displaying CPU/memory recommendations per workload
- Two QoS class recommendation modes: Burstable and Guaranteed
- Supports DaemonSets and StatefulSets in addition to Deployments

**Differentiating features**
- Simplest possible VPA-based recommendation tool: minimal footprint, zero risk of resource disruption
- AWS Open Source Blog endorsement as a cost optimisation reference architecture
- CNCF ecosystem native: pure open source, no vendor tie-in
- Fairwinds also offers Insights (commercial) for policy enforcement and multi-cluster coverage

**UX patterns**
- Namespace labelling to opt in: low friction for adoption in specific namespaces
- Dashboard is read-only: engineers review and manually apply recommendations
- No alerting or budget tracking — purely a recommendation display tool

**Integration points**
- Kubernetes VPA API (autoscaling.k8s.io/v1)
- Helm chart installation
- No external API or cloud billing integration

**Known gaps**
- No automated enforcement of recommendations
- No cost data or dollar amounts — only resource unit recommendations
- No multi-cluster support in the open-source version
- No rightsizing for node selection or spot instances
- Recommendations lag behind actual usage changes (VPA's inherent limitation)

**Licence / IP notes**
- Apache 2.0; Fairwinds OSS project with active maintenance
- No patent or IP concerns; VPA is a standard Kubernetes API

---

### Yotascale

**Core features**
- Kubernetes cost allocation across namespaces, clusters, and teams using ML-driven analysis
- Multi-cloud support: AWS, Azure, GCP hybrid environments
- Dashboards and cost alerts for both engineering and finance stakeholders
- Label-based cost allocation and cost centre mapping
- Rightsizing recommendations (advisory)

**Differentiating features**
- ML-driven cost allocation models that attribute shared infrastructure costs with higher accuracy
- Dual-audience reporting: engineering-friendly and finance-friendly views from the same data
- Focus on FinOps cultural bridge between engineering and finance teams

**UX patterns**
- Dashboard with role-appropriate views (engineering vs. finance)
- Custom cost centre mapping without requiring tag standardisation

**Integration points**
- Cloud billing API integrations (AWS, GCP, Azure)
- Kubernetes cluster agent
- Slack and email alerting

**Known gaps**
- Limited automation: primarily a visibility and allocation tool
- No spot instance management or autoscaling optimisation
- Smaller market presence vs. CAST AI, Kubecost, ScaleOps
- Limited public documentation and API reference

**Licence / IP notes**
- Proprietary SaaS; no open-source components

---

### Komodor

**Core features**
- Kubernetes troubleshooting platform with cost visibility added as secondary capability
- Cost Optimisation Suite: namespace and workload cost tracking, rightsizing recommendations
- Incident correlation: links cost anomalies to deployment events
- AI SRE platform for root-cause analysis and runbook automation
- Resource utilisation monitoring across clusters

**Differentiating features**
- Unique positioning: cost optimisation embedded within a troubleshooting and reliability platform
- Deployment-event correlation: cost changes are automatically linked to the causal deployment or config change
- AI SRE agent automates investigation and proposes remediations

**UX patterns**
- Troubleshooting-first UX: cost features discovered within incident investigation workflows
- Timeline view of deployment events with correlated cost impact
- Targeted at platform engineering and SRE teams, not dedicated FinOps practitioners

**Integration points**
- Kubernetes cluster agent
- CI/CD pipeline integrations (GitHub Actions, Argo CD, Flux)
- PagerDuty, Slack alerting
- REST API for programmatic access

**Known gaps**
- Cost features are secondary to troubleshooting; less depth than dedicated cost tools
- No autonomous optimisation or spot instance management
- Pricing is custom; no public rate card

**Licence / IP notes**
- Proprietary SaaS; no open-source components

---

### Amnic

**Core features**
- Kubernetes cost observability: cluster, namespace, node-level reporting
- Karpenter integration for automated node provisioning optimisation recommendations
- Resource utilisation monitoring and waste identification
- Team-level cost allocation and reporting
- Karpenter configuration compliance recommendations

**Differentiating features**
- Karpenter-native integration: one of the few tools providing Karpenter-specific optimisation guidance
- FinOps observability focus: positioned as an observability tool rather than an automation engine
- Lightweight deployment targeting cost transparency rather than autonomous control

**UX patterns**
- Dashboard-first: cluster cost breakdown as the primary entry point
- Recommendation-oriented: engineers receive actionable insights, apply changes manually
- Team-level reporting for chargeback and showback

**Integration points**
- Kubernetes cluster integration
- Karpenter CRD awareness
- AWS integrations

**Known gaps**
- No autonomous optimisation or rightsizing enforcement
- Limited commercial traction vs. larger players
- No multi-cloud breadth comparable to Kubecost or Finout

**Licence / IP notes**
- Proprietary SaaS; no open-source components

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Cost allocation by namespace, deployment, label, and pod
- Integration with AWS, GCP, and Azure billing APIs
- Prometheus-based metrics collection
- Rightsizing recommendations based on actual vs. requested resources
- Budget alerts and threshold notifications
- Multi-cluster support (at least at dashboard level)
- Web dashboard with cluster cost overview

### Differentiating Features
- Autonomous optimisation without human approval per action (CAST AI, ScaleOps)
- Spot instance lifecycle management with interruption prediction (CAST AI)
- Unit economics (Cost Per Customer, Cost Per Product) tying K8s spend to business outcomes (CloudZero)
- Virtual Tagging for retroactive cost allocation without retagging infrastructure (Finout)
- Agentless cost allocation using existing Prometheus/Datadog data (Finout)
- AI/LLM integration for natural-language cost investigation (CloudZero, OpenCost MCP)
- RI/SP commitment management aligned with Kubernetes usage patterns (nOps ShareSave)
- Troubleshooting + cost correlation in a single platform (Komodor)

### Underserved Areas / Opportunities
- Cross-cluster and cross-cloud workload placement optimisation (placing workloads on cheapest matching compute)
- PR-level cost impact previews — surfacing cost regressions in CI/CD before they reach production
- Predictive capacity planning that incorporates external signals (marketing events, time-of-day seasonality)
- Cost anomaly attribution to specific Git commits or Helm chart changes
- GPU cost optimisation — most tools are immature here despite GPU spend being a fast-growing category
- On-premises / bare-metal Kubernetes cost allocation (most tools assume cloud billing APIs)
- FinOps for AI agent workloads: allocating costs of LLM inference pods running in Kubernetes
- Open-source autonomous optimisation: no open-source tool currently offers automated rightsizing enforcement

### AI-Augmentation Candidates
- Workload profiling: ML models learning per-service traffic patterns to set dynamic resource requests
- Spot interruption prediction: ML models predicting cloud provider interruptions before they occur
- Cost anomaly root-cause attribution: LLM-assisted explanation linking cost spikes to deployment changes
- Predictive scaling: forecasting demand from historical patterns and external business signals
- Natural-language FinOps queries: "Why did our costs increase 30% this week?" answered with structured data
- Optimal workload placement: ML-driven cross-cluster scheduling recommendations based on real-time pricing

---

## Legal & IP Summary

No copyright, patent, or licensing conflicts were identified during this research. The dominant open-source tools (OpenCost, Goldilocks) are Apache 2.0 licensed with no known patent claims. The OpenCost Specification is a vendor-neutral CNCF standard with multi-vendor backing. Commercial tools (CAST AI, ScaleOps, Kubecost Enterprise, nOps, Finout, CloudZero) are proprietary SaaS products; their internal algorithms (e.g. CAST AI's spot interruption prediction) are trade secrets but do not encumber the broader open-source ecosystem. An AI-native open-source Kubernetes cost optimiser built on OpenCost, VPA, and Karpenter APIs would have no IP conflicts.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Real-time cost allocation by namespace, deployment, and label using the OpenCost data model
- Rightsizing recommendations (CPU and memory) using VPA-derived analysis
- Multi-cloud billing API integration (AWS, GCP, Azure) for node-level cost attribution
- Budget alerts and namespace cost threshold notifications
- REST API with OpenAPI documentation for integration into existing tooling
- Prometheus metrics export for embedding in existing observability stacks

**Should-have (v1.1)**
- Autonomous rightsizing enforcement with configurable guardrails and disruption limits
- Spot instance optimisation with basic interruption handling and workload migration
- Karpenter integration for node provisioning cost recommendations
- PR/CI cost-impact preview: surface projected cost change for a given resource configuration
- Cost anomaly detection with Slack and email alerting
- MCP server for AI agent access to cost allocation data

**Nice-to-have (backlog)**
- Predictive scaling using historical traffic patterns and external event signals
- Cross-cluster workload placement recommendations based on real-time pricing
- Natural-language cost investigation via LLM integration
- GPU cost allocation and optimisation recommendations
- RI/SP commitment management recommendations based on Kubernetes usage patterns
- Unit economics modelling (Cost Per Customer, Cost Per Feature)
