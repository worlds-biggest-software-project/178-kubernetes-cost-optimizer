# Data Model Suggestion 4: Hybrid Relational + JSONB

> Project: Kubernetes Cost Optimizer · Created: 2026-05-20

## Philosophy

This model uses a pragmatic hybrid approach: a small number of well-defined relational tables with fixed columns for the core domain, combined with JSONB columns for variable, provider-specific, and extensible data. Instead of modelling every cloud provider's billing fields, every Kubernetes label combination, and every future feature as separate columns or tables, the schema uses typed JSONB columns that are indexed for efficient querying.

The insight is that Kubernetes cost optimisation spans multiple cloud providers (AWS, GCP, Azure, on-prem), each with different billing structures, instance types, discount mechanisms, and metadata. A fully normalised schema would need separate tables for AWS Reserved Instances, GCP Committed Use Discounts, Azure Reservations, and on-prem hardware amortisation — each with different fields. A JSONB hybrid stores the universal fields (cost, time window, resource type) as relational columns and the provider-specific details as structured JSONB, avoiding schema migrations when adding new provider support.

This approach is used successfully by platforms like Stripe (event payloads), GitHub (webhook payloads), and Datadog (custom tags). It provides rapid development velocity because adding a new cloud provider, a new Kubernetes resource type, or a new metric dimension does not require ALTER TABLE statements or data migrations — just a new JSON structure documented in code.

**Best for:** Rapid MVP development, multi-cloud deployments where provider-specific fields vary widely, and teams that want to iterate on the schema without downtime-inducing migrations.

**Trade-offs:**
- (+) Fewest tables (~12) — simplest to understand, deploy, and operate
- (+) No schema migrations needed when adding cloud providers or new resource types
- (+) Provider-specific and jurisdiction-specific data handled without new columns
- (+) GIN indexes on JSONB enable efficient containment and path queries
- (+) Rapid feature development — new fields are just JSON keys
- (+) Single cost_records table handles all allocation levels (container, pod, namespace)
- (-) JSONB queries are slower than indexed relational columns for complex aggregations
- (-) No compile-time type safety on JSONB fields (validation is application-layer)
- (-) JSONB storage is less space-efficient than fixed columns for repeated structures
- (-) Foreign key constraints cannot reference inside JSONB values
- (-) Reporting tools (Grafana, Metabase) may struggle with nested JSONB queries

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenCost Specification | Core cost fields (cpu_cost, ram_cost, etc.) as relational columns; extended fields in JSONB |
| FOCUS v1.3 | Cloud billing records use FOCUS-standard top-level columns plus `x_` prefixed JSONB for provider extensions |
| Kubernetes Labels/Annotations | Stored natively as JSONB with GIN indexes — no label flattening required |
| VPA API | Recommendation payloads stored as structured JSONB matching VPA response format |
| ISO 4217 | Currency code as relational column; multi-currency conversion factors in JSONB |
| MCP | JSONB payloads are directly serialisable as MCP tool responses |

---

## Core Tables

```sql
-- Multi-tenant organisations
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    settings        JSONB DEFAULT '{}',
    -- settings example: {"default_currency": "USD", "retention_days": 365, "features": ["spot_optimisation", "gpu_tracking"]}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    name            VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer',
    preferences     JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_org ON users(org_id);

-- Clusters registered for monitoring
CREATE TABLE clusters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    provider        VARCHAR(50) NOT NULL,             -- 'aws', 'gcp', 'azure', 'on_prem'
    region          VARCHAR(100),
    status          VARCHAR(20) DEFAULT 'active',
    -- Provider-specific configuration stored as JSONB
    provider_config JSONB DEFAULT '{}',
    -- provider_config examples:
    -- AWS: {"account_id": "123456789", "eks_cluster_arn": "arn:aws:eks:...", "cur_s3_bucket": "my-cur-bucket", "irsa_role_arn": "arn:aws:iam::..."}
    -- GCP: {"project_id": "my-project", "billing_dataset": "billing_export", "gke_cluster_id": "..."}
    -- Azure: {"subscription_id": "...", "resource_group": "...", "aks_cluster_name": "..."}
    -- On-prem: {"hardware_cost_monthly": 5000, "depreciation_months": 36, "power_cost_per_kwh": 0.12}
    k8s_config      JSONB DEFAULT '{}',
    -- k8s_config example: {"version": "1.29", "karpenter_enabled": true, "vpa_installed": true, "hpa_count": 45}
    labels          JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(org_id, name)
);

CREATE INDEX idx_clusters_org ON clusters(org_id);
CREATE INDEX idx_clusters_provider ON clusters(provider);
```

## Cost Records (Unified Table)

```sql
-- Unified cost allocation records
-- One table handles all granularity levels via the 'scope' field
CREATE TABLE cost_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id      UUID NOT NULL REFERENCES clusters(id),
    -- Time window
    window_start    TIMESTAMPTZ NOT NULL,
    window_end      TIMESTAMPTZ NOT NULL,
    -- Scope identification (what this cost is attributed to)
    scope           VARCHAR(20) NOT NULL,             -- 'container', 'pod', 'workload', 'namespace', 'node', 'cluster'
    namespace       VARCHAR(255),
    workload_name   VARCHAR(255),
    workload_kind   VARCHAR(50),
    pod_name        VARCHAR(255),
    container_name  VARCHAR(255),
    node_name       VARCHAR(255),
    -- Core cost fields (OpenCost-aligned, always present)
    cpu_cost        NUMERIC(14,6) DEFAULT 0,
    ram_cost        NUMERIC(14,6) DEFAULT 0,
    gpu_cost        NUMERIC(14,6) DEFAULT 0,
    network_cost    NUMERIC(14,6) DEFAULT 0,
    storage_cost    NUMERIC(14,6) DEFAULT 0,
    lb_cost         NUMERIC(14,6) DEFAULT 0,
    shared_cost     NUMERIC(14,6) DEFAULT 0,
    total_cost      NUMERIC(14,6) NOT NULL DEFAULT 0,
    -- Efficiency metrics
    cpu_efficiency  NUMERIC(5,4),
    ram_efficiency  NUMERIC(5,4),
    -- Extended metrics and provider-specific data as JSONB
    usage_metrics   JSONB DEFAULT '{}',
    -- usage_metrics example:
    -- {
    --   "cpu_core_hours": 0.25,
    --   "cpu_request_avg": 0.5,
    --   "cpu_usage_avg": 0.31,
    --   "ram_byte_hours": 536870912,
    --   "ram_request_avg": 1073741824,
    --   "ram_usage_avg": 489000000,
    --   "gpu_hours": 0,
    --   "network_rx_bytes": 125000000,
    --   "network_tx_bytes": 45000000,
    --   "pv_byte_hours": 10737418240
    -- }
    provider_details JSONB DEFAULT '{}',
    -- provider_details example:
    -- AWS: {"instance_type": "m5.xlarge", "is_spot": true, "spot_savings_pct": 0.65, "az": "us-east-1a"}
    -- GCP: {"machine_type": "n2-standard-4", "preemptible": false, "cud_applied": true, "cud_discount_pct": 0.40}
    -- Azure: {"vm_size": "Standard_D4s_v3", "spot": false, "reservation_id": "res-123"}
    labels          JSONB DEFAULT '{}',              -- Kubernetes labels on the workload
    annotations     JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Partitioned by month for manageability
-- ALTER TABLE cost_records PARTITION BY RANGE (window_start);
-- CREATE TABLE cost_records_2026_05 PARTITION OF cost_records FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- Primary query indexes
CREATE INDEX idx_cost_window ON cost_records(window_start, window_end);
CREATE INDEX idx_cost_cluster_ns ON cost_records(cluster_id, namespace, window_start);
CREATE INDEX idx_cost_scope ON cost_records(scope, cluster_id, window_start);
CREATE INDEX idx_cost_workload ON cost_records(cluster_id, namespace, workload_name, window_start);

-- JSONB indexes for label-based queries
CREATE INDEX idx_cost_labels ON cost_records USING GIN(labels);
CREATE INDEX idx_cost_provider_details ON cost_records USING GIN(provider_details);

-- Partial index for spot instances
CREATE INDEX idx_cost_spot ON cost_records(cluster_id, window_start)
    WHERE (provider_details->>'is_spot')::BOOLEAN = TRUE
       OR (provider_details->>'preemptible')::BOOLEAN = TRUE
       OR (provider_details->>'spot')::BOOLEAN = TRUE;
```

## Cloud Billing Integration

```sql
-- Cloud billing records (FOCUS v1.3 aligned with JSONB extensions)
CREATE TABLE cloud_billing (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    cluster_id      UUID REFERENCES clusters(id),
    -- FOCUS standard columns (relational)
    provider        VARCHAR(50) NOT NULL,
    charge_period_start TIMESTAMPTZ NOT NULL,
    charge_period_end   TIMESTAMPTZ NOT NULL,
    service_name    VARCHAR(255),
    service_category VARCHAR(100),
    resource_id     VARCHAR(500),
    region          VARCHAR(100),
    charge_type     VARCHAR(50),
    pricing_category VARCHAR(50),
    billed_cost     NUMERIC(14,6) NOT NULL,
    effective_cost  NUMERIC(14,6),
    billing_currency CHAR(3) DEFAULT 'USD',
    -- Provider-specific extensions (FOCUS x_ fields)
    provider_extensions JSONB DEFAULT '{}',
    -- AWS example: {"x_aws_account_id": "123456789", "x_aws_usage_type": "BoxUsage:m5.xlarge", "x_aws_operation": "RunInstances"}
    -- GCP example: {"x_gcp_project_id": "my-proj", "x_gcp_sku_id": "...", "x_gcp_credits": -5.00}
    -- Azure example: {"x_azure_subscription_id": "...", "x_azure_meter_id": "...", "x_azure_resource_group": "..."}
    tags            JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_billing_period ON cloud_billing(charge_period_start, provider);
CREATE INDEX idx_billing_org ON cloud_billing(org_id, charge_period_start);
CREATE INDEX idx_billing_cluster ON cloud_billing(cluster_id, charge_period_start);
CREATE INDEX idx_billing_tags ON cloud_billing USING GIN(tags);
```

## Recommendations & Actions

```sql
-- Rightsizing recommendations (flexible payload for different recommendation types)
CREATE TABLE recommendations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id      UUID NOT NULL REFERENCES clusters(id),
    namespace       VARCHAR(255) NOT NULL,
    workload_name   VARCHAR(255) NOT NULL,
    recommendation_type VARCHAR(50) NOT NULL,         -- 'cpu_rightsize', 'memory_rightsize', 'spot_migration', 'node_consolidation', 'hpa_tuning'
    severity        VARCHAR(20) NOT NULL DEFAULT 'medium', -- 'low', 'medium', 'high', 'critical'
    status          VARCHAR(20) NOT NULL DEFAULT 'pending', -- 'pending', 'approved', 'applied', 'dismissed', 'expired'
    estimated_monthly_savings NUMERIC(14,2),
    confidence_score NUMERIC(5,4),
    -- Flexible payload for any recommendation type
    details         JSONB NOT NULL,
    -- cpu_rightsize example:
    -- {
    --   "container_name": "api",
    --   "current": {"cpu_request": 1.0, "cpu_limit": 2.0, "memory_request": 1073741824, "memory_limit": 2147483648},
    --   "recommended": {"cpu_request": 0.4, "cpu_limit": 0.8, "memory_request": 536870912, "memory_limit": 1073741824},
    --   "usage_stats": {"p50_cpu": 0.2, "p95_cpu": 0.35, "p99_cpu": 0.38, "p50_mem": 400000000, "p95_mem": 489000000},
    --   "analysis_window_hours": 168
    -- }
    -- spot_migration example:
    -- {
    --   "current_instance_type": "m5.xlarge",
    --   "current_pricing": "on_demand",
    --   "recommended_instance_types": ["m5.xlarge", "m5a.xlarge", "m5d.xlarge"],
    --   "spot_savings_pct": 0.65,
    --   "interruption_rate_30d": 0.03,
    --   "affected_pods": 8
    -- }
    -- node_consolidation example:
    -- {
    --   "target_nodes": ["ip-10-0-1-42", "ip-10-0-1-43"],
    --   "current_utilisation": {"cpu": 0.25, "memory": 0.30},
    --   "after_consolidation": {"node_count": 1, "cpu_util": 0.55, "memory_util": 0.65},
    --   "pods_to_migrate": 12
    -- }
    applied_at      TIMESTAMPTZ,
    applied_by      VARCHAR(255),
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_recommendations_cluster ON recommendations(cluster_id, namespace);
CREATE INDEX idx_recommendations_status ON recommendations(status, recommendation_type);
CREATE INDEX idx_recommendations_savings ON recommendations(estimated_monthly_savings DESC NULLS LAST)
    WHERE status = 'pending';

-- Optimisation actions log (what was actually done)
CREATE TABLE optimisation_actions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recommendation_id UUID REFERENCES recommendations(id),
    cluster_id      UUID NOT NULL REFERENCES clusters(id),
    namespace       VARCHAR(255) NOT NULL,
    workload_name   VARCHAR(255) NOT NULL,
    action_type     VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- 'pending', 'in_progress', 'success', 'failed', 'rolled_back'
    actor           VARCHAR(255),                     -- 'system/auto-rightsizer', 'user:jane@corp.com'
    -- Before/after state as flexible JSONB
    previous_state  JSONB NOT NULL,
    new_state       JSONB NOT NULL,
    result          JSONB DEFAULT '{}',
    -- result example: {"pods_restarted": 3, "downtime_seconds": 0, "actual_savings_first_hour": 0.045}
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    rolled_back_at  TIMESTAMPTZ,
    rollback_reason TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_actions_cluster ON optimisation_actions(cluster_id, namespace);
CREATE INDEX idx_actions_status ON optimisation_actions(status);
CREATE INDEX idx_actions_time ON optimisation_actions(created_at DESC);
```

## Alerts & Budgets

```sql
-- Budgets with flexible scope definition
CREATE TABLE budgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    -- Flexible scope using JSONB
    scope           JSONB NOT NULL,
    -- scope examples:
    -- {"type": "namespace", "cluster_id": "...", "namespace": "production"}
    -- {"type": "label", "label_key": "team", "label_value": "payments"}
    -- {"type": "cluster", "cluster_id": "..."}
    -- {"type": "provider", "provider": "aws", "region": "us-east-1"}
    monthly_limit   NUMERIC(14,2) NOT NULL,
    currency        CHAR(3) DEFAULT 'USD',
    alert_config    JSONB NOT NULL DEFAULT '{"thresholds": [0.5, 0.8, 0.9, 1.0]}',
    -- alert_config example:
    -- {
    --   "thresholds": [0.5, 0.8, 0.9, 1.0],
    --   "channels": [
    --     {"type": "slack", "webhook_url": "https://hooks.slack.com/..."},
    --     {"type": "email", "recipients": ["finops@corp.com"]},
    --     {"type": "pagerduty", "service_key": "..."}
    --   ],
    --   "suppress_after_hours": 4
    -- }
    current_spend   NUMERIC(14,6) DEFAULT 0,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_budgets_org ON budgets(org_id);
CREATE INDEX idx_budgets_scope ON budgets USING GIN(scope);

-- Cost anomalies
CREATE TABLE anomalies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id      UUID NOT NULL REFERENCES clusters(id),
    namespace       VARCHAR(255),
    workload_name   VARCHAR(255),
    detected_at     TIMESTAMPTZ NOT NULL,
    anomaly_type    VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    status          VARCHAR(20) DEFAULT 'open',
    -- Flexible details
    details         JSONB NOT NULL,
    -- details example:
    -- {
    --   "expected_daily_cost": 12.50,
    --   "actual_daily_cost": 48.30,
    --   "deviation_pct": 286.4,
    --   "detection_method": "z_score",
    --   "z_score": 4.2,
    --   "probable_causes": [
    --     {"type": "replica_increase", "from": 3, "to": 12, "deployment_time": "2026-05-20T08:15:00Z"},
    --     {"type": "commit", "sha": "a1b2c3d", "author": "jane@corp.com", "message": "Scale up for load test"}
    --   ],
    --   "affected_resources": ["cpu", "memory"],
    --   "recommendation": "Review replica count after load test completes"
    -- }
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_anomalies_cluster ON anomalies(cluster_id, detected_at DESC);
CREATE INDEX idx_anomalies_status ON anomalies(status, severity);
```

## Guardrail Policies

```sql
-- Guardrail policies with flexible rules
CREATE TABLE guardrail_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id      UUID NOT NULL REFERENCES clusters(id),
    name            VARCHAR(100) NOT NULL,
    -- Scope (what this policy applies to)
    scope           JSONB NOT NULL DEFAULT '{"type": "cluster"}',
    -- scope examples: {"type": "cluster"}, {"type": "namespace", "namespace": "production"}, {"type": "label", "key": "tier", "value": "critical"}
    -- Rules as flexible JSONB
    rules           JSONB NOT NULL,
    -- rules example:
    -- {
    --   "max_cpu_change_pct": 50,
    --   "max_memory_change_pct": 50,
    --   "min_replicas": 2,
    --   "max_disruption_pct": 25,
    --   "allow_spot": false,
    --   "require_approval": true,
    --   "blackout_windows": [{"day": "friday", "after": "16:00"}, {"day": "saturday"}, {"day": "sunday"}],
    --   "cooldown_minutes": 60,
    --   "excluded_workloads": ["critical-db", "payments-service"]
    -- }
    is_active       BOOLEAN DEFAULT TRUE,
    priority        INTEGER DEFAULT 0,               -- higher priority = evaluated first
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_guardrails_cluster ON guardrail_policies(cluster_id);
CREATE INDEX idx_guardrails_scope ON guardrail_policies USING GIN(scope);
```

## Example Queries

```sql
-- Total cost by namespace for the last 7 days (uses relational indexes)
SELECT
    namespace,
    SUM(total_cost) AS total_cost,
    SUM(cpu_cost) AS cpu_cost,
    SUM(ram_cost) AS ram_cost,
    AVG(cpu_efficiency) AS avg_cpu_efficiency
FROM cost_records
WHERE cluster_id = '...'
  AND scope = 'namespace'
  AND window_start >= NOW() - INTERVAL '7 days'
GROUP BY namespace
ORDER BY total_cost DESC;

-- Find all spot instance costs across providers (JSONB query)
SELECT
    namespace,
    workload_name,
    SUM(total_cost) AS spot_cost,
    AVG((provider_details->>'spot_savings_pct')::NUMERIC) AS avg_savings_pct
FROM cost_records
WHERE cluster_id = '...'
  AND window_start >= NOW() - INTERVAL '30 days'
  AND (
    (provider_details->>'is_spot')::BOOLEAN = TRUE
    OR (provider_details->>'preemptible')::BOOLEAN = TRUE
    OR (provider_details->>'spot')::BOOLEAN = TRUE
  )
GROUP BY namespace, workload_name
ORDER BY spot_cost DESC;

-- Cost by team label
SELECT
    labels->>'team' AS team,
    SUM(total_cost) AS total_cost
FROM cost_records
WHERE cluster_id = '...'
  AND scope = 'workload'
  AND window_start >= NOW() - INTERVAL '30 days'
  AND labels ? 'team'
GROUP BY labels->>'team'
ORDER BY total_cost DESC;

-- Pending recommendations sorted by savings potential
SELECT
    namespace,
    workload_name,
    recommendation_type,
    estimated_monthly_savings,
    confidence_score,
    details->'current' AS current_resources,
    details->'recommended' AS recommended_resources
FROM recommendations
WHERE cluster_id = '...'
  AND status = 'pending'
ORDER BY estimated_monthly_savings DESC
LIMIT 20;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-tenancy & Auth | 2 | organisations, users |
| Cluster Management | 1 | clusters (with JSONB provider_config) |
| Cost Data | 2 | cost_records, cloud_billing |
| Optimisation | 2 | recommendations, optimisation_actions |
| Alerts & Budgets | 2 | budgets, anomalies |
| Policy | 1 | guardrail_policies |
| **Total** | **10** | |

---

## Key Design Decisions

1. **Single `cost_records` table for all allocation levels.** Instead of separate tables for container-level, pod-level, and namespace-level costs, a `scope` discriminator column allows one table to serve all granularities. This simplifies queries that need to aggregate across levels and reduces table count dramatically.

2. **JSONB `provider_details` for multi-cloud variance.** AWS spot pricing fields, GCP committed-use discount fields, and Azure reservation fields are fundamentally different. Rather than a UNION of provider-specific tables or dozens of nullable columns, provider-specific data lives in an indexed JSONB column that each provider's ingestion pipeline populates.

3. **JSONB `details` on recommendations for extensibility.** Different recommendation types (CPU rightsize, spot migration, node consolidation, HPA tuning) have completely different data shapes. A typed JSONB payload avoids creating a separate table per recommendation type while still allowing rich structured data.

4. **Labels stored as native JSONB with GIN indexes.** Kubernetes labels are the primary grouping mechanism for cost allocation (team, environment, service). Storing them as JSONB with GIN indexes enables efficient `?` (key exists), `@>` (containment), and `->>`  (path access) queries without a separate labels junction table.

5. **FOCUS standard columns as relational + `x_` extensions as JSONB.** The cloud billing table uses FOCUS v1.3's universal columns as relational fields (for fast filtering and aggregation) and stores provider-specific `x_` extension fields in a JSONB column. This ensures FOCUS compliance while accommodating provider diversity.

6. **Flexible budget scoping via JSONB.** Budget scope can be a namespace, a label selector, a cluster, or a provider/region. Rather than a polymorphic foreign key pattern with multiple nullable columns, the scope is a structured JSONB document that the application layer validates.

7. **Only 10 tables total.** The extreme simplicity of this schema means faster development, simpler backups, easier debugging, and lower operational overhead. The trade-off is that some query patterns require JSONB path operations rather than simple column comparisons.
