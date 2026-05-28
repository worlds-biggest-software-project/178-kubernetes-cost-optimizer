# Data Model Suggestion 3: Event-Sourced / Audit-First (CQRS)

> Project: Kubernetes Cost Optimizer · Created: 2026-05-20

## Philosophy

This model treats every change in the Kubernetes cost landscape as an immutable event recorded in an append-only event store. The event store is the single source of truth; all queryable state (dashboards, reports, recommendations) is derived by projecting events into materialised read models. This is a Command Query Responsibility Segregation (CQRS) architecture where writes go to the event store and reads come from purpose-built projections.

The rationale is that Kubernetes cost optimisation is fundamentally an audit-sensitive domain. Enterprise FinOps teams need to answer questions like "what was our cost on March 15th?", "who approved this rightsizing action and what changed?", and "show me every cost-impacting event for namespace X in the last quarter." In a traditional relational model, answering these temporal queries requires maintaining separate audit tables or change-data-capture infrastructure. In an event-sourced model, the audit trail is the data model — every state change is recorded as a first-class event, and the complete history is always available.

This pattern is well-proven in financial services (ledgers, transaction logs), compliance platforms (regulatory audit trails), and infrastructure management (CloudTrail, Kubernetes audit logs). The Kubernetes API server itself uses a watch/event model that maps naturally to event sourcing.

**Best for:** Regulated enterprises requiring complete audit trails, organisations where cost optimisation decisions must be traceable and reversible, and teams building ML pipelines that benefit from replaying historical state sequences.

**Trade-offs:**
- (+) Complete, immutable audit trail of every cost-impacting event
- (+) Temporal queries ("what was true at time T?") are trivial via event replay
- (+) Event replay enables ML training on historical state sequences
- (+) Schema evolution is additive — new event types, not schema migrations
- (+) Natural fit for Kubernetes watch events and cloud billing change feeds
- (-) Higher infrastructure complexity: event store + projection pipeline + read models
- (-) Eventual consistency between event store and read models (projection lag)
- (-) Read model rebuild from events can be slow with large event volumes
- (-) More complex than traditional CRUD for simple query-only use cases
- (-) Requires careful event versioning as the domain evolves

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenCost Specification | Event payload fields for cost allocation events use OpenCost field names |
| FOCUS v1.3 | Cloud billing events carry FOCUS-standard attributes for interoperability |
| Kubernetes Watch API | Kubernetes resource change events map directly to domain events |
| CloudTrail / Audit Log patterns | Event schema follows cloud provider audit log conventions |
| FinOps Framework | Inform/Optimise/Operate lifecycle maps to event categories |
| OCSF (Open Cybersecurity Schema) | Event envelope structure (id, time, type, severity, actor) borrows from OCSF patterns |

---

## Event Store

```sql
-- The immutable, append-only event store
-- This is the single source of truth for the entire system
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       VARCHAR(500) NOT NULL,           -- aggregate identity, e.g. 'cluster:abc/namespace:prod/workload:api'
    stream_type     VARCHAR(100) NOT NULL,            -- aggregate type: 'cluster', 'namespace', 'workload', 'node', 'billing'
    event_type      VARCHAR(200) NOT NULL,            -- e.g. 'CostAllocationRecorded', 'RightsizingRecommended', 'NodeScaledDown'
    event_version   INTEGER NOT NULL DEFAULT 1,       -- schema version for this event type
    sequence_num    BIGINT NOT NULL,                   -- monotonically increasing per stream
    occurred_at     TIMESTAMPTZ NOT NULL,              -- when the event actually happened
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),-- when the event was recorded
    actor           VARCHAR(255),                      -- 'system/collector', 'system/autoscaler', 'user:jane@corp.com'
    correlation_id  UUID,                              -- links related events across streams
    causation_id    UUID,                              -- the event that caused this event
    payload         JSONB NOT NULL,                    -- event-specific data
    metadata        JSONB DEFAULT '{}',                -- tracing, source info
    UNIQUE(stream_id, sequence_num)
);

-- Primary query patterns
CREATE INDEX idx_events_stream ON events(stream_id, sequence_num);
CREATE INDEX idx_events_type ON events(event_type, occurred_at);
CREATE INDEX idx_events_time ON events(occurred_at);
CREATE INDEX idx_events_correlation ON events(correlation_id);
CREATE INDEX idx_events_recorded ON events(recorded_at);

-- Partition by month for manageability at scale
-- CREATE TABLE events_2026_01 PARTITION OF events FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- etc.
```

### Event Type Catalogue

```sql
-- Registry of all known event types with their JSON schema
CREATE TABLE event_type_registry (
    event_type      VARCHAR(200) PRIMARY KEY,
    category        VARCHAR(50) NOT NULL,             -- 'cost', 'resource', 'optimisation', 'billing', 'config', 'alert'
    description     TEXT NOT NULL,
    payload_schema  JSONB NOT NULL,                   -- JSON Schema for payload validation
    version         INTEGER NOT NULL DEFAULT 1,
    deprecated      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Core Event Types and Payload Examples

```jsonc
// Event type: "CostAllocationRecorded"
// Category: "cost"
// Emitted every hour per container by the allocation engine
{
    "cluster_id": "abc-123",
    "namespace": "production",
    "workload_name": "api-server",
    "workload_kind": "Deployment",
    "pod_name": "api-server-7d9f8b-x2k4p",
    "container_name": "api",
    "node_name": "ip-10-0-1-42",
    "window_start": "2026-05-20T10:00:00Z",
    "window_end": "2026-05-20T11:00:00Z",
    "cpu_core_hours": 0.250,
    "cpu_cost": 0.008125,
    "cpu_efficiency": 0.62,
    "ram_byte_hours": 536870912,
    "ram_cost": 0.003400,
    "ram_efficiency": 0.45,
    "gpu_cost": 0.0,
    "network_cost": 0.001200,
    "pv_cost": 0.000800,
    "total_cost": 0.013525,
    "is_spot": false,
    "node_hourly_rate": 0.096
}

// Event type: "RightsizingRecommended"
// Category: "optimisation"
// Emitted when the ML model generates a new recommendation
{
    "cluster_id": "abc-123",
    "namespace": "production",
    "workload_name": "api-server",
    "container_name": "api",
    "current_cpu_request": 1.0,
    "current_memory_request": 1073741824,
    "recommended_cpu_request": 0.4,
    "recommended_memory_request": 536870912,
    "p95_cpu_usage": 0.35,
    "p99_cpu_usage": 0.38,
    "p95_memory_usage": 489000000,
    "confidence_score": 0.92,
    "estimated_monthly_savings": 45.60,
    "analysis_window_hours": 168
}

// Event type: "RightsizingApplied"
// Category: "optimisation"
// Emitted when a rightsizing action is applied (auto or manual)
{
    "cluster_id": "abc-123",
    "namespace": "production",
    "workload_name": "api-server",
    "container_name": "api",
    "recommendation_event_id": "evt-456",
    "previous_cpu_request": 1.0,
    "new_cpu_request": 0.4,
    "previous_memory_request": 1073741824,
    "new_memory_request": 536870912,
    "approval_mode": "auto",
    "guardrail_policy": "moderate"
}

// Event type: "RightsizingRolledBack"
// Category: "optimisation"
// Emitted when a rightsizing action is reversed
{
    "cluster_id": "abc-123",
    "namespace": "production",
    "workload_name": "api-server",
    "container_name": "api",
    "applied_event_id": "evt-789",
    "rollback_reason": "latency_degradation",
    "restored_cpu_request": 1.0,
    "restored_memory_request": 1073741824
}

// Event type: "NodeProvisioned"
// Category: "resource"
{
    "cluster_id": "abc-123",
    "node_name": "ip-10-0-1-99",
    "instance_type": "m5.xlarge",
    "provider": "aws",
    "is_spot": true,
    "spot_hourly_rate": 0.048,
    "on_demand_hourly_rate": 0.192,
    "provisioned_by": "karpenter",
    "trigger": "pending_pods"
}

// Event type: "NodeTerminated"
// Category: "resource"
{
    "cluster_id": "abc-123",
    "node_name": "ip-10-0-1-99",
    "termination_reason": "spot_interruption",
    "pods_migrated": 5,
    "migration_time_seconds": 12
}

// Event type: "CostAnomalyDetected"
// Category: "alert"
{
    "cluster_id": "abc-123",
    "namespace": "production",
    "workload_name": "batch-processor",
    "anomaly_type": "cost_spike",
    "severity": "high",
    "expected_daily_cost": 12.50,
    "actual_daily_cost": 48.30,
    "deviation_pct": 286.4,
    "probable_cause": "New deployment increased replicas from 3 to 12",
    "related_commit": "a1b2c3d4"
}

// Event type: "BudgetThresholdBreached"
// Category: "alert"
{
    "budget_id": "budget-001",
    "budget_name": "Production namespace",
    "monthly_limit": 5000.00,
    "current_spend": 4125.00,
    "threshold_pct": 0.80,
    "projected_month_end": 5890.00
}

// Event type: "CloudBillingRecordIngested"
// Category: "billing"
// FOCUS v1.3 aligned
{
    "provider": "aws",
    "billing_period_start": "2026-05-01",
    "service_name": "Amazon Elastic Compute Cloud",
    "service_category": "Compute",
    "resource_id": "i-0abc123def456",
    "resource_type": "Instance",
    "region": "us-east-1",
    "charge_type": "Usage",
    "pricing_category": "Spot",
    "billed_cost": 1.152,
    "effective_cost": 1.152,
    "usage_quantity": 24.0,
    "usage_unit": "Hours",
    "billing_currency": "USD"
}
```

## Read Model Projections (Materialised Views)

```sql
-- Projection: Current state of all workloads with latest cost and recommendation
CREATE TABLE rm_workload_current_state (
    cluster_id      UUID NOT NULL,
    namespace       VARCHAR(255) NOT NULL,
    workload_name   VARCHAR(255) NOT NULL,
    workload_kind   VARCHAR(50),
    -- Latest cost data (projected from CostAllocationRecorded)
    last_hour_total_cost    NUMERIC(14,6),
    last_hour_cpu_cost      NUMERIC(14,6),
    last_hour_ram_cost      NUMERIC(14,6),
    last_hour_cpu_efficiency NUMERIC(5,4),
    last_hour_ram_efficiency NUMERIC(5,4),
    daily_total_cost        NUMERIC(14,6),
    monthly_total_cost      NUMERIC(14,6),
    -- Latest rightsizing recommendation (projected from RightsizingRecommended)
    current_cpu_request     NUMERIC(10,3),
    current_memory_request  BIGINT,
    recommended_cpu_request NUMERIC(10,3),
    recommended_memory_request BIGINT,
    estimated_monthly_savings NUMERIC(14,2),
    recommendation_confidence NUMERIC(5,4),
    recommendation_at       TIMESTAMPTZ,
    -- Status
    last_rightsizing_applied_at TIMESTAMPTZ,
    last_cost_event_at      TIMESTAMPTZ,
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (cluster_id, namespace, workload_name)
);

CREATE INDEX idx_rm_workload_savings ON rm_workload_current_state(estimated_monthly_savings DESC NULLS LAST);

-- Projection: Hourly cost by namespace (for dashboards)
CREATE TABLE rm_namespace_cost_hourly (
    cluster_id      UUID NOT NULL,
    namespace       VARCHAR(255) NOT NULL,
    hour            TIMESTAMPTZ NOT NULL,
    cpu_cost        NUMERIC(14,6) DEFAULT 0,
    ram_cost        NUMERIC(14,6) DEFAULT 0,
    gpu_cost        NUMERIC(14,6) DEFAULT 0,
    network_cost    NUMERIC(14,6) DEFAULT 0,
    pv_cost         NUMERIC(14,6) DEFAULT 0,
    total_cost      NUMERIC(14,6) DEFAULT 0,
    avg_cpu_efficiency NUMERIC(5,4),
    avg_ram_efficiency NUMERIC(5,4),
    workload_count  INTEGER DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (cluster_id, namespace, hour)
);

CREATE INDEX idx_rm_ns_cost_time ON rm_namespace_cost_hourly(hour DESC);

-- Projection: Daily cost by namespace (for trend analysis)
CREATE TABLE rm_namespace_cost_daily (
    cluster_id      UUID NOT NULL,
    namespace       VARCHAR(255) NOT NULL,
    day             DATE NOT NULL,
    cpu_cost        NUMERIC(14,6) DEFAULT 0,
    ram_cost        NUMERIC(14,6) DEFAULT 0,
    gpu_cost        NUMERIC(14,6) DEFAULT 0,
    network_cost    NUMERIC(14,6) DEFAULT 0,
    total_cost      NUMERIC(14,6) DEFAULT 0,
    avg_cpu_efficiency NUMERIC(5,4),
    avg_ram_efficiency NUMERIC(5,4),
    PRIMARY KEY (cluster_id, namespace, day)
);

-- Projection: Node fleet state (for node optimisation)
CREATE TABLE rm_node_fleet_state (
    cluster_id      UUID NOT NULL,
    node_name       VARCHAR(255) NOT NULL,
    instance_type   VARCHAR(100),
    provider        VARCHAR(50),
    is_spot         BOOLEAN DEFAULT FALSE,
    status          VARCHAR(20) DEFAULT 'active',    -- 'active', 'draining', 'terminated'
    capacity_cpu    NUMERIC(10,3),
    capacity_memory BIGINT,
    allocated_cpu   NUMERIC(10,3),
    allocated_memory BIGINT,
    hourly_cost     NUMERIC(12,6),
    idle_cost_rate  NUMERIC(5,4),                    -- ratio of idle to total cost
    provisioned_at  TIMESTAMPTZ,
    terminated_at   TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (cluster_id, node_name)
);

-- Projection: Optimisation action audit trail (for compliance)
CREATE TABLE rm_optimisation_audit_trail (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id      UUID NOT NULL,
    namespace       VARCHAR(255) NOT NULL,
    workload_name   VARCHAR(255) NOT NULL,
    action_type     VARCHAR(50) NOT NULL,             -- 'rightsizing', 'spot_migration', 'node_scale_down'
    action_status   VARCHAR(20) NOT NULL,             -- 'recommended', 'applied', 'rolled_back'
    actor           VARCHAR(255),
    event_id        UUID NOT NULL,                    -- reference back to event store
    occurred_at     TIMESTAMPTZ NOT NULL,
    details         JSONB NOT NULL,
    previous_state  JSONB,
    new_state       JSONB
);

CREATE INDEX idx_audit_cluster_ns ON rm_optimisation_audit_trail(cluster_id, namespace, occurred_at DESC);
CREATE INDEX idx_audit_type ON rm_optimisation_audit_trail(action_type, occurred_at DESC);

-- Projection: Budget tracking state
CREATE TABLE rm_budget_state (
    budget_id       UUID PRIMARY KEY,
    budget_name     VARCHAR(255) NOT NULL,
    scope_cluster   UUID,
    scope_namespace VARCHAR(255),
    monthly_limit   NUMERIC(14,2) NOT NULL,
    current_spend   NUMERIC(14,6) DEFAULT 0,
    spend_rate_per_hour NUMERIC(14,6),               -- current burn rate
    projected_month_end NUMERIC(14,6),
    last_alert_threshold NUMERIC(5,4),
    last_alert_at   TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Projection: Anomaly state
CREATE TABLE rm_active_anomalies (
    id              UUID PRIMARY KEY,
    cluster_id      UUID NOT NULL,
    namespace       VARCHAR(255),
    workload_name   VARCHAR(255),
    anomaly_type    VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    detected_at     TIMESTAMPTZ NOT NULL,
    expected_cost   NUMERIC(14,6),
    actual_cost     NUMERIC(14,6),
    deviation_pct   NUMERIC(8,2),
    description     TEXT,
    status          VARCHAR(20) DEFAULT 'open',
    resolved_at     TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_active_anomalies_status ON rm_active_anomalies(status, severity);
```

## Projection Checkpoints

```sql
-- Tracks the last processed event for each projection
-- Enables projection rebuild and catchup
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_sequence   BIGINT NOT NULL,
    last_processed_at TIMESTAMPTZ NOT NULL,
    status          VARCHAR(20) DEFAULT 'running',   -- 'running', 'rebuilding', 'paused', 'error'
    error_message   TEXT,
    events_processed BIGINT DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Multi-Tenancy & Configuration

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    role            VARCHAR(50) DEFAULT 'viewer',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Guardrail policies (also event-sourced, but stored here for fast lookup)
CREATE TABLE guardrail_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id      UUID NOT NULL,
    namespace       VARCHAR(255),                     -- NULL = cluster-wide
    workload_name   VARCHAR(255),                     -- NULL = namespace-wide
    policy_name     VARCHAR(100) NOT NULL,
    max_cpu_change_pct  NUMERIC(5,2) DEFAULT 50,      -- max % change per action
    max_memory_change_pct NUMERIC(5,2) DEFAULT 50,
    min_replica_count   INTEGER DEFAULT 1,
    max_disruption_pct  NUMERIC(5,2) DEFAULT 25,      -- max % of pods disrupted
    allow_spot      BOOLEAN DEFAULT TRUE,
    require_approval BOOLEAN DEFAULT FALSE,
    approval_timeout_hours INTEGER DEFAULT 24,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Example: Replaying History

```sql
-- "What was the cost of namespace 'production' on March 15, 2026?"
-- Replay all CostAllocationRecorded events for that date
SELECT
    (payload->>'namespace') AS namespace,
    SUM((payload->>'total_cost')::NUMERIC) AS total_cost,
    SUM((payload->>'cpu_cost')::NUMERIC) AS cpu_cost,
    SUM((payload->>'ram_cost')::NUMERIC) AS ram_cost
FROM events
WHERE event_type = 'CostAllocationRecorded'
  AND payload->>'namespace' = 'production'
  AND occurred_at >= '2026-03-15T00:00:00Z'
  AND occurred_at < '2026-03-16T00:00:00Z'
GROUP BY payload->>'namespace';

-- "Show me every optimisation action for workload 'api-server' and who did it"
SELECT
    event_type,
    occurred_at,
    actor,
    payload
FROM events
WHERE stream_id LIKE '%workload:api-server'
  AND event_type IN ('RightsizingRecommended', 'RightsizingApplied', 'RightsizingRolledBack')
ORDER BY occurred_at;

-- "Rebuild the namespace cost daily projection from scratch"
-- (executed by the projection engine, not ad-hoc)
TRUNCATE rm_namespace_cost_daily;
INSERT INTO rm_namespace_cost_daily (cluster_id, namespace, day, cpu_cost, ram_cost, gpu_cost, network_cost, total_cost)
SELECT
    (payload->>'cluster_id')::UUID,
    payload->>'namespace',
    DATE(occurred_at),
    SUM((payload->>'cpu_cost')::NUMERIC),
    SUM((payload->>'ram_cost')::NUMERIC),
    SUM((payload->>'gpu_cost')::NUMERIC),
    SUM((payload->>'network_cost')::NUMERIC),
    SUM((payload->>'total_cost')::NUMERIC)
FROM events
WHERE event_type = 'CostAllocationRecorded'
GROUP BY (payload->>'cluster_id')::UUID, payload->>'namespace', DATE(occurred_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | events (partitioned by month) |
| Event Registry | 1 | event_type_registry |
| Read Model Projections | 7 | workload state, namespace hourly/daily, node fleet, audit trail, budgets, anomalies |
| Projection Infrastructure | 1 | projection_checkpoints |
| Multi-tenancy & Config | 4 | organisations, users, guardrail_policies |
| **Total** | **14** | 1 event store + 13 supporting tables |

---

## Key Design Decisions

1. **Single event store table as the sole source of truth.** All state is derived from events. If a read model is corrupted or needs to be restructured, it can be rebuilt by replaying events from the beginning — no data loss is possible.

2. **Stream-based event organisation.** Events are grouped by `stream_id` (e.g. `cluster:abc/namespace:prod/workload:api`) which represents an aggregate boundary. This enables efficient replay of a single workload's history without scanning the entire event store.

3. **Correlation and causation IDs for event tracing.** `correlation_id` links all events triggered by a single user action or system event. `causation_id` tracks direct cause-effect chains (e.g. a RightsizingRecommended event causes a RightsizingApplied event). This enables full traceability for audit purposes.

4. **Read models are disposable and rebuildable.** Every `rm_*` table can be dropped and rebuilt from events. The `projection_checkpoints` table tracks processing state so projections can resume from where they left off after an interruption.

5. **Event versioning for schema evolution.** Each event has an `event_version` field. When the payload schema for an event type changes, the version is incremented. Projection code handles multiple versions via upcasting, avoiding the need for batch migrations on the event store.

6. **JSONB payloads for flexibility.** Event payloads are stored as JSONB rather than in fixed columns. This allows new event types to be added without schema migrations, and enables rich querying via PostgreSQL's JSONB operators for ad-hoc analysis.

7. **Guardrail policies stored relationally for fast lookup.** While policy changes are recorded as events for audit purposes, the current policy state is also maintained in a relational table for microsecond-latency lookups during automated rightsizing decisions.
