# Data Model Suggestion 2: Time-Series First (TimescaleDB Partitioned)

> Project: Kubernetes Cost Optimizer · Created: 2026-05-20

## Philosophy

This model treats cost and resource usage data as what it fundamentally is: time-series data. Instead of storing cost allocations as rows in a traditional relational table, the core data lives in hypertables (TimescaleDB) or range-partitioned tables optimised for time-range queries, continuous aggregations, and data retention policies. The design separates a thin relational layer for entity metadata from a high-throughput time-series layer for metrics and costs.

This architecture mirrors how Prometheus and OpenCost actually generate data: as streams of timestamped metric samples. Rather than transforming these into relational records, the time-series-first model stores them in their natural shape. Continuous aggregates materialise hourly, daily, and monthly rollups automatically, providing fast dashboard queries without expensive GROUP BY operations at read time.

The pattern is proven in observability platforms (Grafana Cloud, Datadog, New Relic) where metrics ingestion rates of millions of samples per second are routine. For a Kubernetes cost optimiser that needs to ingest per-container CPU/memory usage every 30-60 seconds across hundreds of clusters, a time-series-native storage layer avoids the write amplification and locking issues that plague traditional relational models under high ingestion load.

**Best for:** Deployments managing many clusters (50+) with high-frequency metrics ingestion, where historical trend analysis and predictive ML workloads are primary use cases.

**Trade-offs:**
- (+) Write throughput 10-100x higher than standard PostgreSQL for time-series inserts
- (+) Continuous aggregates eliminate expensive GROUP BY queries for dashboards
- (+) Built-in data retention and compression policies reduce storage costs
- (+) Natural fit for Prometheus metric data and ML training datasets
- (+) Range queries on time windows are extremely fast due to chunk-level pruning
- (-) Requires TimescaleDB extension (or manual partitioning management)
- (-) Cross-entity JOINs between hypertables and relational tables can be awkward
- (-) Less intuitive for developers unfamiliar with time-series data patterns
- (-) Continuous aggregate refresh lag means dashboards may be slightly stale
- (-) Foreign key constraints not supported on hypertable dimensions

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenCost Specification | Metric field names (cpu_core_hours, ram_byte_hours, etc.) match OpenCost allocation response fields |
| FOCUS v1.3 | Cloud billing ingestion table uses FOCUS column naming for interoperability |
| Prometheus Data Model | Resource usage metrics stored in a label-indexed time-series structure matching Prometheus semantics |
| FinOps Framework | Continuous aggregates produce pre-computed showback/chargeback views at hourly/daily/monthly granularity |
| OpenTelemetry (OTLP) | Metric ingestion pipeline can accept OTLP metric payloads alongside Prometheus scrapes |

---

## Entity Metadata Tables (Standard PostgreSQL)

```sql
-- Thin relational layer for entity registration and lookup

CREATE TABLE clusters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL UNIQUE,
    provider        VARCHAR(50) NOT NULL,
    region          VARCHAR(100),
    k8s_version     VARCHAR(20),
    status          VARCHAR(20) DEFAULT 'active',
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE namespaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id      UUID NOT NULL REFERENCES clusters(id),
    name            VARCHAR(255) NOT NULL,
    labels          JSONB DEFAULT '{}',
    team_id         UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(cluster_id, name)
);

CREATE TABLE workloads (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    namespace_id    UUID NOT NULL REFERENCES namespaces(id),
    name            VARCHAR(255) NOT NULL,
    kind            VARCHAR(50) NOT NULL,
    labels          JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(namespace_id, name, kind)
);

CREATE TABLE nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id      UUID NOT NULL REFERENCES clusters(id),
    name            VARCHAR(255) NOT NULL,
    instance_type   VARCHAR(100),
    capacity_cpu    NUMERIC(10,3),
    capacity_memory BIGINT,
    capacity_gpu    NUMERIC(10,3) DEFAULT 0,
    is_spot         BOOLEAN DEFAULT FALSE,
    hourly_cost     NUMERIC(12,6),
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Teams for chargeback
CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL UNIQUE,
    cost_centre     VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Multi-tenant organisations
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
```

## Time-Series Hypertables (TimescaleDB)

```sql
-- Resource usage metrics sampled every 30-60 seconds
-- This is the highest-volume table: ~1 row per container per scrape interval
CREATE TABLE resource_usage_metrics (
    time            TIMESTAMPTZ NOT NULL,
    cluster_id      UUID NOT NULL,
    namespace       VARCHAR(255) NOT NULL,
    workload_name   VARCHAR(255),
    workload_kind   VARCHAR(50),
    pod_name        VARCHAR(255) NOT NULL,
    container_name  VARCHAR(255) NOT NULL,
    node_name       VARCHAR(255),
    -- CPU metrics
    cpu_request     NUMERIC(10,3),          -- cores requested
    cpu_limit       NUMERIC(10,3),          -- cores limit
    cpu_usage       NUMERIC(10,3),          -- actual cores used
    -- Memory metrics
    memory_request  BIGINT,                 -- bytes requested
    memory_limit    BIGINT,                 -- bytes limit
    memory_usage    BIGINT,                 -- actual bytes used
    -- GPU metrics
    gpu_request     NUMERIC(10,3) DEFAULT 0,
    gpu_usage       NUMERIC(10,3) DEFAULT 0,
    -- Network
    network_rx_bytes BIGINT DEFAULT 0,
    network_tx_bytes BIGINT DEFAULT 0
);

-- Convert to TimescaleDB hypertable with 1-hour chunks
SELECT create_hypertable('resource_usage_metrics', 'time',
    chunk_time_interval => INTERVAL '1 hour');

-- Composite index for the most common query pattern
CREATE INDEX idx_usage_cluster_ns ON resource_usage_metrics(cluster_id, namespace, time DESC);
CREATE INDEX idx_usage_workload ON resource_usage_metrics(cluster_id, namespace, workload_name, time DESC);

-- Enable compression after 7 days (10-20x space reduction)
ALTER TABLE resource_usage_metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'cluster_id, namespace, workload_name, container_name',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('resource_usage_metrics', INTERVAL '7 days');

-- Retain raw data for 90 days; aggregates persist longer
SELECT add_retention_policy('resource_usage_metrics', INTERVAL '90 days');


-- Cost allocation records computed from usage metrics + pricing
-- One row per container per hour (computed by the allocation engine)
CREATE TABLE cost_allocations (
    time            TIMESTAMPTZ NOT NULL,
    cluster_id      UUID NOT NULL,
    namespace       VARCHAR(255) NOT NULL,
    workload_name   VARCHAR(255),
    workload_kind   VARCHAR(50),
    pod_name        VARCHAR(255) NOT NULL,
    container_name  VARCHAR(255) NOT NULL,
    node_name       VARCHAR(255),
    -- Costs (OpenCost-aligned field names)
    cpu_cost        NUMERIC(14,6) DEFAULT 0,
    cpu_core_hours  NUMERIC(14,6) DEFAULT 0,
    cpu_efficiency  NUMERIC(5,4),
    ram_cost        NUMERIC(14,6) DEFAULT 0,
    ram_byte_hours  NUMERIC(20,6) DEFAULT 0,
    ram_efficiency  NUMERIC(5,4),
    gpu_cost        NUMERIC(14,6) DEFAULT 0,
    gpu_hours       NUMERIC(14,6) DEFAULT 0,
    network_cost    NUMERIC(14,6) DEFAULT 0,
    pv_cost         NUMERIC(14,6) DEFAULT 0,
    lb_cost         NUMERIC(14,6) DEFAULT 0,
    shared_cost     NUMERIC(14,6) DEFAULT 0,
    total_cost      NUMERIC(14,6) NOT NULL DEFAULT 0,
    -- Pricing context
    node_hourly_rate NUMERIC(12,6),
    is_spot         BOOLEAN DEFAULT FALSE
);

SELECT create_hypertable('cost_allocations', 'time',
    chunk_time_interval => INTERVAL '1 day');

CREATE INDEX idx_cost_cluster_ns ON cost_allocations(cluster_id, namespace, time DESC);
CREATE INDEX idx_cost_workload ON cost_allocations(cluster_id, namespace, workload_name, time DESC);

ALTER TABLE cost_allocations SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'cluster_id, namespace, workload_name',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('cost_allocations', INTERVAL '30 days');
SELECT add_retention_policy('cost_allocations', INTERVAL '2 years');


-- Node-level cost metrics
CREATE TABLE node_cost_metrics (
    time            TIMESTAMPTZ NOT NULL,
    cluster_id      UUID NOT NULL,
    node_name       VARCHAR(255) NOT NULL,
    instance_type   VARCHAR(100),
    is_spot         BOOLEAN DEFAULT FALSE,
    -- Capacity
    capacity_cpu    NUMERIC(10,3),
    capacity_memory BIGINT,
    capacity_gpu    NUMERIC(10,3) DEFAULT 0,
    -- Allocatable
    allocatable_cpu NUMERIC(10,3),
    allocatable_memory BIGINT,
    -- Allocated (sum of requests on this node)
    allocated_cpu   NUMERIC(10,3),
    allocated_memory BIGINT,
    -- Cost
    hourly_cost     NUMERIC(12,6),
    idle_cost       NUMERIC(12,6),          -- cost of unallocated resources
    system_cost     NUMERIC(12,6)           -- cost of system pods (kube-system etc.)
);

SELECT create_hypertable('node_cost_metrics', 'time',
    chunk_time_interval => INTERVAL '1 day');

CREATE INDEX idx_node_cost_cluster ON node_cost_metrics(cluster_id, time DESC);

ALTER TABLE node_cost_metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'cluster_id, node_name',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('node_cost_metrics', INTERVAL '30 days');
```

## Continuous Aggregates

```sql
-- Hourly cost rollup by namespace (auto-refreshed)
CREATE MATERIALIZED VIEW cost_by_namespace_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    cluster_id,
    namespace,
    SUM(cpu_cost) AS cpu_cost,
    SUM(ram_cost) AS ram_cost,
    SUM(gpu_cost) AS gpu_cost,
    SUM(network_cost) AS network_cost,
    SUM(pv_cost) AS pv_cost,
    SUM(total_cost) AS total_cost,
    AVG(cpu_efficiency) AS avg_cpu_efficiency,
    AVG(ram_efficiency) AS avg_ram_efficiency,
    COUNT(DISTINCT workload_name) AS workload_count
FROM cost_allocations
GROUP BY bucket, cluster_id, namespace;

SELECT add_continuous_aggregate_policy('cost_by_namespace_hourly',
    start_offset    => INTERVAL '3 hours',
    end_offset      => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');


-- Daily cost rollup by namespace (built on hourly aggregate)
CREATE MATERIALIZED VIEW cost_by_namespace_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', bucket) AS bucket,
    cluster_id,
    namespace,
    SUM(cpu_cost) AS cpu_cost,
    SUM(ram_cost) AS ram_cost,
    SUM(gpu_cost) AS gpu_cost,
    SUM(network_cost) AS network_cost,
    SUM(pv_cost) AS pv_cost,
    SUM(total_cost) AS total_cost,
    AVG(avg_cpu_efficiency) AS avg_cpu_efficiency,
    AVG(avg_ram_efficiency) AS avg_ram_efficiency,
    MAX(workload_count) AS max_workload_count
FROM cost_by_namespace_hourly
GROUP BY bucket, cluster_id, namespace;

SELECT add_continuous_aggregate_policy('cost_by_namespace_daily',
    start_offset    => INTERVAL '3 days',
    end_offset      => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');


-- Daily cost rollup by workload
CREATE MATERIALIZED VIEW cost_by_workload_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    cluster_id,
    namespace,
    workload_name,
    workload_kind,
    SUM(cpu_cost) AS cpu_cost,
    SUM(cpu_core_hours) AS cpu_core_hours,
    SUM(ram_cost) AS ram_cost,
    SUM(ram_byte_hours) AS ram_byte_hours,
    SUM(gpu_cost) AS gpu_cost,
    SUM(network_cost) AS network_cost,
    SUM(total_cost) AS total_cost,
    AVG(cpu_efficiency) AS avg_cpu_efficiency,
    AVG(ram_efficiency) AS avg_ram_efficiency
FROM cost_allocations
GROUP BY bucket, cluster_id, namespace, workload_name, workload_kind;

SELECT add_continuous_aggregate_policy('cost_by_workload_daily',
    start_offset    => INTERVAL '3 days',
    end_offset      => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');


-- Node idle cost daily rollup
CREATE MATERIALIZED VIEW node_idle_cost_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    cluster_id,
    node_name,
    instance_type,
    is_spot,
    AVG(capacity_cpu) AS avg_capacity_cpu,
    AVG(allocated_cpu) AS avg_allocated_cpu,
    AVG(capacity_memory) AS avg_capacity_memory,
    AVG(allocated_memory) AS avg_allocated_memory,
    SUM(hourly_cost) AS total_cost,
    SUM(idle_cost) AS total_idle_cost
FROM node_cost_metrics
GROUP BY bucket, cluster_id, node_name, instance_type, is_spot;

SELECT add_continuous_aggregate_policy('node_idle_cost_daily',
    start_offset    => INTERVAL '3 days',
    end_offset      => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');
```

## Operational Tables (Standard PostgreSQL)

```sql
-- Rightsizing recommendations (derived from time-series analysis)
CREATE TABLE rightsizing_recommendations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id      UUID NOT NULL REFERENCES clusters(id),
    namespace       VARCHAR(255) NOT NULL,
    workload_name   VARCHAR(255) NOT NULL,
    container_name  VARCHAR(255) NOT NULL,
    analysis_window_start TIMESTAMPTZ NOT NULL,
    analysis_window_end   TIMESTAMPTZ NOT NULL,
    -- Current settings
    current_cpu_request NUMERIC(10,3),
    current_memory_request BIGINT,
    -- Recommendations (based on p95/p99 of time-series usage)
    recommended_cpu_request NUMERIC(10,3),
    recommended_memory_request BIGINT,
    -- Statistical basis
    p50_cpu_usage   NUMERIC(10,3),
    p95_cpu_usage   NUMERIC(10,3),
    p99_cpu_usage   NUMERIC(10,3),
    p50_memory_usage BIGINT,
    p95_memory_usage BIGINT,
    p99_memory_usage BIGINT,
    -- Impact
    estimated_monthly_savings NUMERIC(14,2),
    confidence_score NUMERIC(5,4),          -- 0.0 to 1.0
    status          VARCHAR(20) DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rightsizing_cluster ON rightsizing_recommendations(cluster_id, namespace);
CREATE INDEX idx_rightsizing_status ON rightsizing_recommendations(status);

-- Budget tracking
CREATE TABLE budgets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    scope_cluster   UUID REFERENCES clusters(id),
    scope_namespace VARCHAR(255),
    monthly_limit   NUMERIC(14,2) NOT NULL,
    alert_thresholds NUMERIC[] DEFAULT ARRAY[0.5, 0.8, 0.9, 1.0],
    notification_config JSONB DEFAULT '{}',
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Anomaly detections
CREATE TABLE cost_anomalies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id      UUID NOT NULL,
    namespace       VARCHAR(255),
    workload_name   VARCHAR(255),
    detected_at     TIMESTAMPTZ NOT NULL,
    anomaly_type    VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    expected_cost   NUMERIC(14,6),
    actual_cost     NUMERIC(14,6),
    deviation_pct   NUMERIC(8,2),
    description     TEXT,
    status          VARCHAR(20) DEFAULT 'open',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_anomalies_detected ON cost_anomalies(detected_at DESC);
```

## Example Queries

```sql
-- Monthly cost trend for a namespace (uses continuous aggregate, fast)
SELECT
    bucket,
    total_cost,
    avg_cpu_efficiency,
    avg_ram_efficiency
FROM cost_by_namespace_daily
WHERE cluster_id = '...'
  AND namespace = 'production'
  AND bucket >= NOW() - INTERVAL '30 days'
ORDER BY bucket;

-- Top 10 most expensive workloads this week (uses continuous aggregate)
SELECT
    namespace,
    workload_name,
    SUM(total_cost) AS weekly_cost,
    AVG(avg_cpu_efficiency) AS cpu_eff,
    AVG(avg_ram_efficiency) AS mem_eff
FROM cost_by_workload_daily
WHERE cluster_id = '...'
  AND bucket >= NOW() - INTERVAL '7 days'
GROUP BY namespace, workload_name
ORDER BY weekly_cost DESC
LIMIT 10;

-- P95 CPU usage for a container over the last 7 days (raw time-series)
SELECT
    percentile_cont(0.95) WITHIN GROUP (ORDER BY cpu_usage) AS p95_cpu
FROM resource_usage_metrics
WHERE cluster_id = '...'
  AND namespace = 'production'
  AND workload_name = 'api-server'
  AND container_name = 'api'
  AND time >= NOW() - INTERVAL '7 days';

-- Node idle cost ratio by instance type (uses continuous aggregate)
SELECT
    instance_type,
    is_spot,
    SUM(total_cost) AS total_cost,
    SUM(total_idle_cost) AS idle_cost,
    ROUND(SUM(total_idle_cost) / NULLIF(SUM(total_cost), 0) * 100, 1) AS idle_pct
FROM node_idle_cost_daily
WHERE cluster_id = '...'
  AND bucket >= NOW() - INTERVAL '30 days'
GROUP BY instance_type, is_spot
ORDER BY idle_cost DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Entity Metadata | 7 | clusters, namespaces, workloads, nodes, teams, organisations, users |
| Time-Series Hypertables | 3 | resource_usage_metrics, cost_allocations, node_cost_metrics |
| Continuous Aggregates | 4 | hourly/daily by namespace, daily by workload, node idle daily |
| Operational Tables | 3 | rightsizing_recommendations, budgets, cost_anomalies |
| **Total** | **17** | 10 tables + 3 hypertables + 4 continuous aggregates |

---

## Key Design Decisions

1. **Hypertables for all time-series data.** Resource usage metrics and cost allocations are stored in TimescaleDB hypertables with automatic partitioning by time. This enables chunk-level compression, retention policies, and fast range queries without manual partition management.

2. **String-based entity references in hypertables instead of UUIDs.** Hypertables use `cluster_id` + `namespace` + `workload_name` string combinations rather than foreign keys to entity tables. This avoids the foreign-key limitation of TimescaleDB hypertables and reduces JOIN requirements for common queries.

3. **Three-tier aggregation: raw -> hourly -> daily.** Raw metrics are retained for 90 days for detailed debugging and ML training. Hourly aggregates serve real-time dashboards. Daily aggregates serve trend analysis and reporting. This tiered approach balances storage cost with query speed.

4. **Compression after 7 days on raw metrics.** TimescaleDB's native compression achieves 10-20x space reduction on time-series data. The segmentby configuration on cluster_id/namespace/workload allows compressed chunks to still be queried efficiently by the most common filter dimensions.

5. **Percentile-based rightsizing recommendations.** The time-series foundation enables direct computation of p50/p95/p99 usage percentiles from raw data, providing statistically grounded rightsizing recommendations rather than simple average-based VPA suggestions.

6. **Separate node_cost_metrics hypertable.** Node-level metrics are stored separately from container-level costs because they have different cardinality and query patterns. Node idle cost analysis is a first-class concern for identifying over-provisioned clusters.

7. **No FOCUS billing table in the hypertable layer.** Cloud billing data arrives in batch (daily) and has different access patterns from real-time metrics. It can be stored in a standard relational table and joined with aggregated cost data for reconciliation.
