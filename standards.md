# Standards & API Reference

> Project: Kubernetes Cost Optimizer · Generated: 2026-05-03

## Industry Standards & Specifications

### Kubernetes Native APIs

**Vertical Pod Autoscaler (VPA) — autoscaling.k8s.io/v1**
- Official URL: https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler
- The primary Kubernetes mechanism for CPU/memory rightsizing recommendations. VPA supports `updatePolicy` modes: `Auto` (applies recommendations automatically), `Initial` (applies only at pod creation), and `Off` (recommendations only, no mutation). The `Off` mode is used by tools like Goldilocks to surface recommendations without risk of disrupting running pods. Any cost optimiser that provides rightsizing must either integrate with or replicate VPA's recommendation engine.

**Horizontal Pod Autoscaler (HPA) — autoscaling/v2**
- Official URL: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
- Native Kubernetes mechanism for scaling pod replica counts based on CPU, memory, or custom metrics. Cost optimisers must be aware of HPA settings to avoid conflicting resource recommendations; VPA and HPA should not both be applied to the same metric on the same workload. HPA v2 supports custom and external metrics APIs.

**Kubernetes Custom Metrics API — custom.metrics.k8s.io/v1beta1**
- Official URL: https://github.com/kubernetes-sigs/custom-metrics-apiserver
- Standardised API extension allowing metrics adapters (Prometheus Adapter, KEDA) to expose application-specific metrics to HPA. Cost optimisers that provide scaling recommendations must understand which custom metrics drive scaling decisions for individual workloads.

**KEDA (Kubernetes Event-Driven Autoscaling) — keda.sh/v1alpha1**
- Official URL: https://keda.sh/docs/
- CNCF project extending HPA for event-driven and external-metric-based scaling. Introduces `ScaledObject` and `ScaledJob` CRDs; supports 50+ built-in scalers (Kafka, Redis, AWS SQS, Prometheus, etc.). Cost optimisers deployed alongside KEDA must integrate without conflicting with its scaling decisions. KEDA registers the `v1beta1.external.metrics.k8s.io` API service.

**Karpenter Node Provisioner — karpenter.sh/v1**
- Official URL: https://karpenter.sh/docs/
- Modern replacement for the Kubernetes Cluster Autoscaler. Introduces `NodeClass` and `NodePool` CRDs for defining node provisioning strategies. Karpenter provisions nodes in real time based on pending pod requirements. Cost optimisers must integrate with Karpenter's disruption budgets and instance selection logic to optimise node-level spend without conflicting with its provisioner.

**Kubernetes Metrics Server**
- Official URL: https://github.com/kubernetes-sigs/metrics-server
- Core Kubernetes metrics aggregator providing the `metrics.k8s.io/v1beta1` API. Supplies CPU and memory usage data to HPA and VPA. Cost optimisers depend on this data (or equivalent Prometheus data) for rightsizing analysis.

---

### FinOps & Cost Allocation Standards

**OpenCost Specification**
- Official URL: https://opencost.io/docs/specification/
- CNCF Incubating project providing a vendor-neutral open specification for Kubernetes cost allocation and chargeback. Defines allocation of CPU, memory, GPU, network, and storage costs to Kubernetes primitives (namespace, pod, deployment, service). Supported by AWS, Google, Microsoft, Adobe, Red Hat, and New Relic. Any new K8s cost tool should align its data model with the OpenCost Specification to ensure interoperability with the ecosystem.

**FOCUS — FinOps Open Cost and Usage Specification v1.3**
- Official URL: https://focus.finops.org/focus-specification/
- GitHub: https://github.com/FinOps-Open-Cost-and-Usage-Spec/FOCUS_Spec
- Open technical specification for technology billing data, normalising cloud, SaaS, and data centre costs into a uniform dataset format. AWS and Azure natively export in FOCUS format; GCP support in progress. Version 1.3 (latest as of 2026) adds contract commitment support and split cost allocation dimensions. A Kubernetes cost tool should be able to ingest and export FOCUS-formatted billing data for interoperability with multi-cloud FinOps platforms.

**FinOps Framework — FinOps Foundation**
- Official URL: https://www.finops.org/framework/
- Vendor-neutral framework defining cost allocation, showback, chargeback, and continuous optimisation practices for cloud-native environments. Describes the Inform → Optimise → Operate lifecycle. Defines buyer personas (FinOps Practitioner, Platform Engineer, Engineering Lead, Finance). Relevant for positioning an AI-native cost optimiser within the enterprise FinOps tooling ecosystem.

---

### Observability & Metrics Standards

**Prometheus Data Model and Remote Write Protocol**
- Official URL: https://prometheus.io/docs/
- Remote Write spec: https://prometheus.io/docs/specs/remote_write_spec/
- De-facto standard for Kubernetes metrics collection. OpenCost, Kubecost, CAST AI, and most K8s cost tools are built on Prometheus for metrics ingestion. The Remote Write protocol (v1 and v2) allows metrics to be streamed to external storage. A cost tool must either integrate with an existing Prometheus deployment or deploy its own scraper.

**OpenTelemetry (OTLP) — opentelemetry.io**
- Official URL: https://opentelemetry.io/
- OTLP Spec: https://opentelemetry.io/docs/specs/otlp/
- CNCF project for vendor-neutral observability data collection (metrics, traces, logs). Prometheus supports OTLP ingestion via `--web.enable-otlp-receiver` flag. OpenTelemetry Collector can scrape Prometheus endpoints and export via OTLP, bridging the Prometheus and OTel ecosystems. Relevant for cost tools integrating into modern observability stacks.

---

### Security & Authentication Standards

**OAuth 2.0 — RFC 6749**
- Official URL: https://datatracker.ietf.org/doc/html/rfc6749
- Standard authorisation framework used by cloud provider APIs (AWS, GCP, Azure) and most SaaS platforms. A cost optimiser querying cloud billing APIs or calling the Kubernetes API server must implement OAuth 2.0 flows. GCP uses OAuth 2.0 service account credentials; AWS uses IAM with IRSA (IAM Roles for Service Accounts) for Kubernetes workloads.

**OpenID Connect (OIDC) — openid.net**
- Official URL: https://openid.net/connect/
- Identity layer on top of OAuth 2.0. Kubernetes API server can be configured to use OIDC for user authentication. AWS EKS uses OIDC for IAM Roles for Service Accounts (IRSA), which is the recommended way for in-cluster workloads (including cost tools) to authenticate against AWS APIs.

**Kubernetes RBAC — rbac.authorization.k8s.io/v1**
- Official URL: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- Role-Based Access Control is the standard Kubernetes authorisation mechanism. A cost tool deployed in-cluster needs appropriate RBAC roles: read access to pods, nodes, deployments, VPA objects, and optionally write access for rightsizing enforcement. SOC 2 compliance requires RBAC policies to be auditable and least-privilege.

**SOC 2 Type II**
- Reference: https://www.aicpa-cima.com/resources/landing/system-and-organization-controls-soc-suite-of-services
- Enterprise buyers of Kubernetes cost tools require SOC 2 Type II attestation from vendors, particularly for tools that require elevated cluster permissions (node management, pod eviction). An open-source tool targeting enterprise adoption should document its security model and permission scope clearly.

---

### Container & Cloud Standards

**OCI (Open Container Initiative) Image Specification**
- Official URL: https://opencontainers.org/
- Spec: https://github.com/opencontainers/image-spec
- Standard for container image format. Relevant to cost optimisation in that container image layer caching and efficient image sizes reduce compute costs during image pulls and node startup times.

**AWS Cost and Usage Report (CUR) v2**
- Official URL: https://docs.aws.amazon.com/cur/latest/userguide/
- AWS's standard billing export format; FOCUS-compatible export also available. Required for accurate node-level cost attribution in EKS environments. Kubecost, nOps, CAST AI, and CloudZero all integrate with CUR.

**GCP Cloud Billing Export (BigQuery)**
- Official URL: https://cloud.google.com/billing/docs/how-to/export-data-bigquery
- GCP's standard billing export to BigQuery. Required for GKE cost attribution. Most K8s cost tools support BigQuery-based billing ingestion for GCP.

**Azure Cost Management Export**
- Official URL: https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/tutorial-export-acm-data
- Azure's standard billing export. Required for AKS cost attribution. Supported by major K8s cost tools.

---

### Model Context Protocol (MCP)

**Model Context Protocol — modelcontextprotocol.io**
- Official URL: https://modelcontextprotocol.io/
- Spec: https://spec.modelcontextprotocol.io/
- Open protocol (developed by Anthropic, now community-governed) for standardising how AI agents access external data sources and tools. OpenCost has introduced a built-in MCP server (2026), enabling AI agents to directly query Kubernetes cost allocation data. CloudZero has a community MCP server. An AI-native K8s cost optimiser should expose an MCP server as a first-class integration point to allow LLM-based agents to query, explain, and act on cost data.

---

## Similar Products — Developer Documentation & APIs

### CAST AI

- **Description:** Autonomous Kubernetes cost optimisation platform covering rightsizing, spot instance management, and node provisioning for EKS, GKE, and AKS.
- **API Documentation:** https://docs.cast.ai/docs/api
- **API Spec (OpenAPI):** https://api.cast.ai/v1/spec/
- **SDK/Libraries:** Pulumi provider (https://www.pulumi.com/registry/packages/castai/); Terraform provider available
- **Kubernetes Agent:** https://github.com/castai/k8s-agent (Helm chart)
- **Developer Guide:** https://docs.cast.ai/docs/getting-started
- **Standards:** REST/JSON; OpenAPI 3.x; Helm for agent deployment
- **Authentication:** API Key via `X-API-Key` header

---

### Kubecost (IBM)

- **Description:** Kubernetes-native cost monitoring and allocation tool with rightsizing recommendations; backed by IBM as part of the FinOps Suite.
- **API Documentation:** https://docs.kubecost.com (IBM Kubecost Self Hosted: https://www.ibm.com/docs/en/kubecost/self-hosted/2.x)
- **API Spec (OpenAPI):** Available as swagger.json via the Kubecost GitHub repository; https://github.com/kubecost
- **SDK/Libraries:** Grafana JSON integration (https://github.com/kubecost/kubecost-grafana-json-integration); Port.io integration
- **Developer Guide:** https://kubecost.awsworkshop.io/5_using_kubecost/52_api.html
- **Standards:** REST/JSON; Prometheus metrics; Grafana dashboards
- **Authentication:** API key or OIDC (enterprise tier); in-cluster access at `<KUBECOST_IP>:9090/model/`
- **Key Endpoints:** `/model/allocation` (cost allocation), `/model/savings/requestSizing` (rightsizing recommendations), `/model/cloudCost` (cloud asset costs)

---

### OpenCost

- **Description:** CNCF Incubating open-source Kubernetes cost monitoring project; the vendor-neutral standard for K8s cost allocation. Includes MCP server for AI agent access.
- **API Documentation:** https://opencost.io/docs/integrations/api/
- **API Examples:** https://opencost.io/docs/integrations/api-examples/
- **API Spec (OpenAPI):** https://github.com/opencost/opencost/blob/develop/docs/swagger.json
- **SDK/Libraries:** No official SDK; REST API is the primary integration surface. MCP server built-in.
- **Developer Guide:** https://opencost.io/docs/
- **Standards:** REST/JSON; OpenAPI (swagger.json); Prometheus metrics
- **Authentication:** No authentication on the allocation API by default (in-cluster access); recommend kubectl port-forward for local access
- **Key Endpoints:** `/allocation` (cost allocation by window/aggregate), `/cloudCost` (cloud asset costs); port 9003

---

### CloudZero

- **Description:** Cloud cost intelligence platform with Kubernetes cost allocation, unit economics, and AI-powered cost investigation. Manages $14B+ in cloud spend.
- **API Documentation:** https://docs.cloudzero.com/reference/introduction
- **API Reference:** https://docs.cloudzero.com/reference/getbillingcosts
- **Authorization:** https://docs.cloudzero.com/reference/authorization
- **SDK/Libraries:** Community MCP server: https://github.com/burkestar/cloudzero-mcp; liteLLM integration: https://docs.litellm.ai/docs/observability/cloudzero
- **Developer Guide:** https://docs.cloudzero.com/docs/getting-started
- **Standards:** REST/JSON; paginated responses (10,000 records/page)
- **Authentication:** API key in `Authorization` header
- **Key Endpoints:** `/v2/billing/costs` (cost data); rate limit: 60 req/day

---

### ScaleOps

- **Description:** Autonomous Kubernetes resource optimisation platform: rightsizing, HPA management, spot adoption, GPU sharing. Self-hosted option available. $130M Series C (March 2026).
- **API Documentation:** Not publicly documented; primarily a managed-platform experience
- **SDK/Libraries:** Kubernetes operator (Red Hat certified); AWS Marketplace: https://aws.amazon.com/marketplace/pp/prodview-t6ctoxyyxp5zs
- **Developer Guide:** https://scaleops.com/ (documentation requires account)
- **Standards:** Kubernetes operator pattern; compatible with HPA, KEDA, Karpenter
- **Authentication:** Cluster-level credentials for the operator; SaaS console login
- **Note:** Limited public API surface; integration is primarily via Kubernetes operator CRDs

---

### nOps

- **Description:** AWS-native EKS cost optimisation platform covering workload rightsizing, spot instance management, and RI/SP commitment management (ShareSave).
- **API Documentation:** Not publicly documented; AWS Marketplace and console-based
- **SDK/Libraries:** Lightweight EKS agent (proprietary); AWS Marketplace listing
- **Developer Guide:** https://www.nops.io/blog/kubernetes-k8s-cost-allocation-and-optimization/
- **Standards:** AWS-native integrations (CUR, EKS API, Savings Plans API); REST API for commitment management
- **Authentication:** AWS IAM/IRSA for cluster integration; API key for nOps console

---

### Finout

- **Description:** Enterprise FinOps platform providing a unified MegaBill across Kubernetes, cloud, and SaaS spend. Virtual Tagging and agentless Kubernetes cost allocation.
- **API Documentation:** https://docs.finout.io (account required for full access)
- **SDK/Libraries:** Prometheus and Datadog integration (agentless); cloud billing API integrations
- **Developer Guide:** https://www.finout.io/
- **Standards:** REST/JSON; integrates with Prometheus, Datadog, AWS CUR, GCP BigQuery, Azure Cost Export
- **Authentication:** API key; SAML/SSO for enterprise

---

### Goldilocks (Fairwinds)

- **Description:** Open-source Kubernetes rightsizing recommendation tool using VPA in `Off` mode. Dashboard displays CPU/memory recommendations per workload.
- **API Documentation:** No external REST API; interacts via Kubernetes VPA API
- **GitHub:** https://github.com/FairwindsOps/goldilocks
- **SDK/Libraries:** Helm chart for installation; Kubernetes VPA API (autoscaling.k8s.io/v1)
- **Developer Guide:** https://www.fairwinds.com/blog/goldilocks-kubernetes-resource-requests; https://aws.amazon.com/blogs/opensource/right-size-your-kubernetes-applications-using-open-source-goldilocks-for-cost-optimization/
- **Standards:** Kubernetes VPA API; Helm
- **Authentication:** Kubernetes RBAC (reads VPA and Deployment objects)

---

## Notes

**Emerging standards to watch:**
- **FOCUS v1.3+**: The FinOps Foundation conformance certification programme launching in 2026 will make FOCUS a de-facto requirement for enterprise tooling. Any new tool should plan for native FOCUS export.
- **OpenCost Data Model 2.0 (KubeModel)**: The new data model is more precise for shared cost allocation and GPU cost tracking. Tools built today should evaluate compatibility with KubeModel rather than the legacy allocation model.
- **MCP as an integration standard**: OpenCost and CloudZero both have MCP servers in 2026. MCP is emerging as the standard integration surface for AI-agent access to cost data — an AI-native cost tool should include MCP server support at launch.
- **GPU cost allocation**: No current standard covers GPU cost allocation in Kubernetes. As GPU workloads grow (inference, training), this is an emerging area where OpenCost and other tools are beginning to define models. An AI-native tool has an opportunity to lead here.
- **On-premises cost models**: No current standard covers bare-metal or on-premises Kubernetes cost allocation. Most tools assume cloud billing APIs. This is an underserved gap for hybrid-cloud enterprise deployments.
