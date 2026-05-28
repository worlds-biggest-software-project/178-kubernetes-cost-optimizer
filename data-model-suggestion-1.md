# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Kubernetes Cost Optimizer · Created: 2026-05-20

## Philosophy

This model follows classical relational database design with a separate table for every domain concept, strict foreign key relationships, and normalised reference data. Every Kubernetes primitive (cluster, node, namespace, deployment, pod, container) has its own table, and cost allocation records reference these entities via foreign keys. The design prioritises data integrity, queryability, and alignment with the OpenCost Specification's allocation model.

This is the approach most familiar to teams with SQL expertise. It maps directly to how Kubecost and OpenCost expose their allocation APIs — each cost record is a junction of a time window, a Kubernetes entity, and a set of cost/usage metrics. Reference tables for cloud providers, instance types, and pricing tiers provide the normalised lookup data needed for accurate cost attribution.

The trade-off is a higher table count and more complex JOINs for cross-cutting queries (e.g. "total cost by team across all clusters"), but the payoff is strict referential integrity, straightforward indexing, and compatibility with standard BI tools.

**Best for:** Teams with strong SQL skills deploying to a single PostgreSQL instance, where data integrity and ad-hoc querying are more important than write throughput.

**Trade-offs:**
- (+) Strong referential integrity prevents orphaned cost records
- (+) Standard SQL — works with any BI tool, Grafana, Metabase, etc.
- (+) Clear entity boundaries make RBAC straightforward (row-level security per tenant/cluster)
- (+) Easy to understand and debug
- (-) High table count (~30+ tables) increases migration complexity
- (-) JOINs across 5-6 tables for common dashboard queries can be slow at scale
- (-) Schema changes required for each new Kubernetes primitive or cloud provider
- (-) Write-heavy metrics ingestion may bottleneck on foreign key constraint checks

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenCost Specification | Allocation table fields map 1:1 to OpenCost allocation response (cpuCost, ramCost, gpuCost, networkCost, pvCost) |
| FOCUS v1.3 | `focus_billing_records` table stores FOCUS-formatted cloud billing data with standard column names |
| Kubernetes VPA API | `vpa_recommendations` table stores VPA recommendation snapshots with target/lower/upper bounds |
| FinOps Framework | Showback/chargeback modelled via `cost_allocation_rules` and `chargeback_reports` |
| ISO 4217 | Currency codes stored as CHAR(3) references for multi-currency support |
| Prometheus | `metric_snapshots` stores raw Prometheus scrape results for cost derivation |

---

## Cluster & Infrastructure Tables

```sql
-- Registered Kubernetes clusters
CREATE TABLE clusters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    provider        VARCHAR(50) NOT NULL,          -- 'aws', 'gcp', 'azure', 'on_prem'
    region          VARCHAR(100),                   -- e.g. 'us-east-1', 'europe-west1'
    k8s_version     VARCHAR(20),
    karpenter_enabled BOOLEAN DEFAULT FALSE,
    status          VARCHAR(20) DEFAULT 'active',   -- 'active', 'inactive', 'decommissioned'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(name, provider)
);

CREATE INDEX idx_clusters_provider ON clusters(provider);
CREATE INDEX idx_clusters_status ON clusters(status);

-- Nodes within clusters
CREATE TABLE nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id      UUID NOT NULL REFERENCES clusters(id),
    name            VARCHAR(255) NOT NULL,
    instance_type   VARCHAR(100),                   -- e.g. 'm5.xlarge', 'n2-standard-4'
    capacity_cpu    NUMERIC(10,3) NOT NULL,          -- total CPU cores
    capacity_memory BIGINT NOT NULL,                 -- total memory in bytes
    capacity_gpu    NUMERIC(10,3) DEFAULT 0,
    is_spot         BOOLEAN DEFAULT FALSE,
    availability_zone VARCHAR(50),
    hourly_cost     NUMERIC(12,6),                   -- on-demand hourly rate
    spot_hourly_cost NUMERIC(12,6),                  -- spot hourly rate (if applicable)
    status          VARCHAR(20) DEFAULT 'ready',     -- 'ready', 'not_ready', 'terminated'
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_nodes_cluster ON nodes(cluster_id);
CREATE INDEX idx_nodes_instance_type ON nodes(instance_type);
CREATE INDEX idx_nodes_spot ON nodes(is_spot);

-- Cloud provider pricing reference data
CREATE TABLE instance_pricing (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    provider        VARCHAR(50) NOT NULL,
    region          VARCHAR(100) NOT NULL,
    instance_type   VARCHAR(100) NOT NULL,
    cpu_cores       NUMERIC(10,3) NOT NULL,
    memory_gb       NUMERIC(10,3) NOT NULL,
    gpu_count       INTEGER DEFAULT 0,
    on_demand_hourly NUMERIC(12,6) NOT NULL,
    spot_hourly     NUMERIC(12,6),
    currency        CHAR(3) DEFAULT 'USD',           -- ISO 4217
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(provider, region, instance_type, effective_from)
);

CREATE INDEX idx_pricing_lookup ON instance_pricing(provider, region, instance_type, effective_from);
```

## Kubernetes Entity Tables

```sql
-- Namespaces
CREATE TABLE namespaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id      UUID NOT NULL REFERENCES clusters(id),
    name            VARCHAR(255) NOT NULL,
    labels          JSONB DEFAULT '{}',
    annotations     JSONB DEFAULT '{}',
    status          VARCHAR(20) DEFAULT 'active',
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(cluster_id, name)
);

CREATE INDEX idx_namespaces_cluster ON namespaces(cluster_id);
CREATE INDEX idx_namespaces_labels ON namespaces USING GIN(labels);

-- Workload controllers (Deployments, StatefulSets, DaemonSets, Jobs)
CREATE TABLE workloads (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    namespace_id    UUID NOT NULL REFERENCES namespaces(id),
    name            VARCHAR(255) NOT NULL,
    kind            VARCHAR(50) NOT NULL,            -- 'Deployment', 'StatefulSet', 'DaemonSet', 'Job', 'CronJob'
    labels          JSONB DEFAULT '{}',
    annotations     JSONB DEFAULT '{}',
    replicas        INTEGER,
    status          VARCHAR(20) DEFAULT 'active',
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(namespace_id, name, kind)
);

CREATE INDEX idx_workloads_namespace ON workloads(namespace_id);
CREATE INDEX idx_workloads_kind ON workloads(kind);
CREATE INDEX idx_workloads_labels ON workloads USING GIN(labels);

-- Pods
CREATE TABLE pods (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workload_id     UUID REFERENCES workloads(id),   -- nullable for standalone pods
    node_id         UUID REFERENCES nodes(id),
    namespace_id    UUID NOT NULL REFERENCES namespaces(id),
    name            VARCHAR(255) NOT NULL,
    phase           VARCHAR(20),                     -- 'Running', 'Pending', 'Succeeded', 'Failed'
    qos_class       VARCHAR(20),                     -- 'Guaranteed', 'Burstable', 'BestEffort'
    labels          JSONB DEFAULT '{}',
    started_at      TIMESTAMPTZ,
    finished_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_pods_workload ON pods(workload_id);
CREATE INDEX idx_pods_node ON pods(node_id);
CREATE INDEX idx_pods_namespace ON pods(namespace_id);
CREATE INDEX idx_pods_phase ON pods(phase);

-- Containers within pods
CREATE TABLE containers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pod_id          UUID NOT NULL REFERENCES pods(id),
    name            VARCHAR(255) NOT NULL,
    image           VARCHAR(500),
    cpu_request     NUMERIC(10,3),                   -- CPU cores requested
    cpu_limit       NUMERIC(10,3),
    memory_request  BIGINT,                          -- bytes
    memory_limit    BIGINT,
    gpu_request     NUMERIC(10,3) DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(pod_id, name)
);

CREATE INDEX idx_containers_pod ON containers(pod_id);

-- Kubernetes Services
CREATE TABLE services (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    namespace_id    UUID NOT NULL REFERENCES namespaces(id),
    name            VARCHAR(255) NOT NULL,
    type            VARCHAR(20),                     -- 'ClusterIP', 'NodePort', 'LoadBalancer'
    labels          JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(namespace_id, name)
);

-- Persistent Volume Claims
CREATE TABLE persistent_volume_claims (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    namespace_id    UUID NOT NULL REFERENCES namespaces(id),
    name            VARCHAR(255) NOT NULL,
    storage_class   VARCHAR(255),
    capacity_bytes  BIGINT,
    access_mode     VARCHAR(50),                     -- 'ReadWriteOnce', 'ReadWriteMany', 'ReadOnlyMany'
    status          VARCHAR(20),
    hourly_cost     NUMERIC(12,6),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(namespace_id, name)
);
```

## Cost Allocation Tables

```sql
-- Core cost allocation records (aligned with OpenCost Specification)
-- One record per container per time window
CREATE TABLE cost_allocations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    container_id    UUID NOT NULL REFERENCES containers(id),
    pod_id          UUID NOT NULL REFERENCES pods(id),
    workload_id     UUID REFERENCES workloads(id),
    namespace_id    UUID NOT NULL REFERENCES namespaces(id),
    cluster_id      UUID NOT NULL REFERENCES clusters(id),
    node_id         UUID REFERENCES nodes(id),
    window_start    TIMESTAMPTZ NOT NULL,
    window_end      TIMESTAMPTZ NOT NULL,
    -- CPU costs (OpenCost spec fields)
    cpu_core_hours          NUMERIC(14,6) DEFAULT 0,
    cpu_core_request_avg    NUMERIC(10,3) DEFAULT 0,
    cpu_core_usage_avg      NUMERIC(10,3) DEFAULT 0,
    cpu_cost                NUMERIC(14,6) DEFAULT 0,
    cpu_efficiency          NUMERIC(5,4),              -- 0.0000 to 1.0000
    -- Memory costs
    ram_byte_hours          NUMERIC(20,6) DEFAULT 0,
    ram_byte_request_avg    BIGINT DEFAULT 0,
    ram_byte_usage_avg      BIGINT DEFAULT 0,
    ram_cost                NUMERIC(14,6) DEFAULT 0,
    ram_efficiency          NUMERIC(5,4),
    -- GPU costs
    gpu_hours               NUMERIC(14,6) DEFAULT 0,
    gpu_cost                NUMERIC(14,6) DEFAULT 0,
    -- Network costs
    network_transfer_bytes  BIGINT DEFAULT 0,
    network_cost            NUMERIC(14,6) DEFAULT 0,
    -- Storage costs
    pv_byte_hours           NUMERIC(20,6) DEFAULT 0,
    pv_cost                 NUMERIC(14,6) DEFAULT 0,
    -- Load balancer costs
    lb_cost                 NUMERIC(14,6) DEFAULT 0,
    -- Shared / external costs
    shared_cost             NUMERIC(14,6) DEFAULT 0,
    external_cost           NUMERIC(14,6) DEFAULT 0,
    -- Totals
    total_cost              NUMERIC(14,6) NOT NULL DEFAULT 0,
    total_efficiency        NUMERIC(5,4),
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Partition by month for query performance
CREATE INDEX idx_allocations_window ON cost_allocations(window_start, window_end);
CREATE INDEX idx_allocations_namespace ON cost_allocations(namespace_id, window_start);
CREATE INDEX idx_allocations_cluster ON cost_allocations(cluster_id, window_start);
CREATE INDEX idx_allocations_workload ON cost_allocations(workload_id, window_start);

-- Cloud billing records (FOCUS v1.3 aligned)
CREATE TABLE cloud_billing_records (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id              UUID REFERENCES clusters(id),
    billing_account_id      VARCHAR(255) NOT NULL,
    billing_period_start    DATE NOT NULL,
    billing_period_end      DATE NOT NULL,
    charge_period_start     TIMESTAMPTZ NOT NULL,
    charge_period_end       TIMESTAMPTZ NOT NULL,
    provider                VARCHAR(50) NOT NULL,       -- FOCUS: Provider
    service_name            VARCHAR(255),                -- FOCUS: ServiceName
    service_category        VARCHAR(100),                -- FOCUS: ServiceCategory
    resource_id             VARCHAR(500),                -- FOCUS: ResourceId
    resource_name           VARCHAR(255),                -- FOCUS: ResourceName
    resource_type           VARCHAR(100),                -- FOCUS: ResourceType
    region                  VARCHAR(100),                -- FOCUS: Region
    availability_zone       VARCHAR(50),                 -- FOCUS: AvailabilityZone
    charge_type             VARCHAR(50),                 -- FOCUS: ChargeType ('Usage', 'Purchase', 'Tax')
    charge_frequency        VARCHAR(50),                 -- FOCUS: ChargeFrequency
    pricing_category        VARCHAR(50),                 -- FOCUS: PricingCategory ('OnDemand', 'Spot', 'Commitment')
    billed_cost             NUMERIC(14,6) NOT NULL,      -- FOCUS: BilledCost
    effective_cost          NUMERIC(14,6),                -- FOCUS: EffectiveCost
    list_cost               NUMERIC(14,6),               -- FOCUS: ListCost
    pricing_unit            VARCHAR(50),                 -- FOCUS: PricingUnit
    usage_quantity          NUMERIC(14,6),               -- FOCUS: UsageQuantity
    usage_unit              VARCHAR(50),                 -- FOCUS: UsageUnit
    billing_currency        CHAR(3) DEFAULT 'USD',       -- ISO 4217
    tags                    JSONB DEFAULT '{}',          -- FOCUS: Tags
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_billing_period ON cloud_billing_records(billing_period_start, provider);
CREATE INDEX idx_billing_cluster ON cloud_billing_records(cluster_id, charge_period_start);
CREATE INDEX idx_billing_resource ON cloud_billing_records(resource_id);
```

## Rightsizing & Optimisation Tables

```sql
-- VPA recommendation snapshots
CREATE TABLE vpa_recommendations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workload_id     UUID NOT NULL REFERENCES workloads(id),
    container_name  VARCHAR(255) NOT NULL,
    snapshot_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    -- Target recommendations
    target_cpu      NUMERIC(10,3),                   -- CPU cores
    target_memory   BIGINT,                          -- bytes
    -- Lower bound
    lower_bound_cpu NUMERIC(10,3),
    lower_bound_memory BIGINT,
    -- Upper bound
    upper_bound_cpu NUMERIC(10,3),
    upper_bound_memory BIGINT,
    -- Uncapped target
    uncapped_target_cpu NUMERIC(10,3),
    uncapped_target_memory BIGINT,
    -- Current settings for comparison
    current_request_cpu NUMERIC(10,3),
    current_request_memory BIGINT,
    current_limit_cpu NUMERIC(10,3),
    current_limit_memory BIGINT,
    -- Estimated savings
    estimated_monthly_savings NUMERIC(14,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_vpa_workload ON vpa_recommendations(workload_id, snapshot_at DESC);

-- Rightsizing actions (applied or pending)
CREATE TABLE rightsizing_actions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workload_id     UUID NOT NULL REFERENCES workloads(id),
    recommendation_id UUID REFERENCES vpa_recommendations(id),
    action_type     VARCHAR(50) NOT NULL,            -- 'cpu_resize', 'memory_resize', 'replica_scale', 'spot_migration'
    status          VARCHAR(20) NOT NULL DEFAULT 'pending', -- 'pending', 'approved', 'applied', 'rolled_back', 'rejected'
    previous_value  JSONB NOT NULL,                  -- e.g. {"cpu_request": 0.5, "memory_request": 536870912}
    new_value       JSONB NOT NULL,
    estimated_savings NUMERIC(14,2),
    applied_at      TIMESTAMPTZ,
    applied_by      VARCHAR(255),                    -- 'system/auto' or user identifier
    rollback_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_actions_workload ON rightsizing_actions(workload_id);
CREATE INDEX idx_actions_status ON rightsizing_actions(status);

-- Budget definitions and alerts
CREATE TABLE budgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    scope_type      VARCHAR(20) NOT NULL,            -- 'cluster', 'namespace', 'workload', 'label'
    scope_id        UUID,                            -- references cluster/namespace/workload id
    scope_label_selector JSONB,                      -- for label-based scopes
    monthly_budget  NUMERIC(14,2) NOT NULL,
    currency        CHAR(3) DEFAULT 'USD',
    alert_thresholds JSONB DEFAULT '[0.5, 0.8, 0.9, 1.0]', -- percentage thresholds
    notification_channels JSONB DEFAULT '[]',        -- [{"type": "slack", "webhook": "..."}, {"type": "email", "to": "..."}]
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Cost anomaly detections
CREATE TABLE cost_anomalies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id      UUID NOT NULL REFERENCES clusters(id),
    namespace_id    UUID REFERENCES namespaces(id),
    workload_id     UUID REFERENCES workloads(id),
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    anomaly_type    VARCHAR(50) NOT NULL,            -- 'cost_spike', 'usage_spike', 'idle_resource', 'new_workload'
    severity        VARCHAR(20) NOT NULL,            -- 'low', 'medium', 'high', 'critical'
    expected_cost   NUMERIC(14,6),
    actual_cost     NUMERIC(14,6),
    deviation_pct   NUMERIC(8,2),
    description     TEXT,
    root_cause      TEXT,                            -- AI-generated root cause analysis
    related_commit  VARCHAR(255),                    -- git commit SHA if attributable
    status          VARCHAR(20) DEFAULT 'open',      -- 'open', 'acknowledged', 'resolved', 'false_positive'
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_anomalies_cluster ON cost_anomalies(cluster_id, detected_at DESC);
CREATE INDEX idx_anomalies_status ON cost_anomalies(status);
```

## Cost Allocation Rules & Reporting

```sql
-- Teams / cost centres for chargeback
CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL UNIQUE,
    cost_centre     VARCHAR(100),
    contact_email   VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Mapping rules: which namespaces/labels belong to which team
CREATE TABLE cost_allocation_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID NOT NULL REFERENCES teams(id),
    rule_type       VARCHAR(20) NOT NULL,            -- 'namespace', 'label', 'annotation'
    match_cluster   UUID REFERENCES clusters(id),    -- NULL = all clusters
    match_namespace VARCHAR(255),                     -- regex or exact match
    match_label_key VARCHAR(255),
    match_label_value VARCHAR(255),
    shared_cost_weight NUMERIC(5,4) DEFAULT 1.0,     -- weight for shared cost distribution
    priority        INTEGER DEFAULT 0,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rules_team ON cost_allocation_rules(team_id);

-- Multi-tenancy: organisations
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) DEFAULT 'free',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Users
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    name            VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer', -- 'admin', 'editor', 'viewer'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_org ON users(org_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Cluster & Infrastructure | 3 | clusters, nodes, instance_pricing |
| Kubernetes Entities | 5 | namespaces, workloads, pods, containers, services |
| Storage | 1 | persistent_volume_claims |
| Cost Allocation | 2 | cost_allocations, cloud_billing_records |
| Rightsizing & Optimisation | 2 | vpa_recommendations, rightsizing_actions |
| Budgets & Anomalies | 2 | budgets, cost_anomalies |
| Teams & Chargeback | 2 | teams, cost_allocation_rules |
| Multi-tenancy & Auth | 2 | organisations, users |
| **Total** | **19** | |

---

## Key Design Decisions

1. **Container-level cost allocation as the atomic unit.** Following the OpenCost Specification, costs are attributed at the container level and aggregated upward. This provides maximum granularity for rightsizing decisions.

2. **FOCUS v1.3 aligned cloud billing table.** The `cloud_billing_records` table uses FOCUS column names directly, ensuring interoperability with multi-cloud FinOps platforms and native cloud billing exports.

3. **Separate VPA recommendation snapshots from actions.** Recommendations are recorded as immutable snapshots; actions track the lifecycle from pending to applied/rolled-back. This separation enables trend analysis on recommendation stability.

4. **Label/annotation storage as JSONB on entity tables.** While the core schema is normalised, Kubernetes labels and annotations are inherently key-value and vary per workload. JSONB with GIN indexes provides efficient containment queries without a separate labels junction table.

5. **Month-based partitioning recommended for cost_allocations.** The largest table by volume should be range-partitioned on `window_start` to maintain query performance as data grows.

6. **Team-based cost allocation rules with weighted shared costs.** Shared infrastructure costs (control plane, monitoring, ingress) are distributed across teams based on configurable weights, following the FinOps Framework chargeback model.

7. **Foreign key chain from container up to cluster.** Denormalised cluster_id and namespace_id on `cost_allocations` avoid multi-hop JOINs for the most common dashboard queries, at the cost of some redundancy.
