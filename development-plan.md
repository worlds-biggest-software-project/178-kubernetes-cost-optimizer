# Kubernetes Cost Optimizer — Phased Development Plan

> Project: 178-kubernetes-cost-optimizer · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Go 1.23+ | Kubernetes ecosystem is Go-native; client-go, controller-runtime, Prometheus client, and Karpenter are all Go libraries. Go's compiled binaries simplify in-cluster deployment and reduce container image size. |
| API framework | go-chi/chi v5 with OpenAPI 3.1 via oapi-codegen | Lightweight, stdlib-compatible router; oapi-codegen generates types and server stubs from the OpenAPI spec, ensuring the API documentation is always in sync with the code. |
| Database | PostgreSQL 16 with TimescaleDB extension | Cost allocation data is fundamentally time-series (Data Model Suggestion 2). TimescaleDB hypertables provide chunk-based partitioning, native compression, continuous aggregates, and retention policies on standard PostgreSQL. Relational entity metadata tables coexist naturally. |
| Migrations | golang-migrate/migrate | SQL-file-based migrations compatible with PostgreSQL and TimescaleDB DDL; no ORM dependency. |
| Kubernetes client | client-go + controller-runtime | Standard libraries for building Kubernetes operators and controllers. controller-runtime provides the Reconciler pattern for watching VPA, HPA, and Karpenter CRDs. |
| Metrics collection | Prometheus client_golang + PromQL via prometheus/client_golang | Prometheus is the de-facto standard for Kubernetes metrics. The cost allocation engine scrapes resource usage via the Kubernetes Metrics API and Prometheus. Prometheus metrics are also exported for the tool's own observability. |
| Task queue | In-process Go worker pool (river) | River is a PostgreSQL-backed job queue for Go, eliminating the need for a separate Redis/RabbitMQ deployment. Cost allocation computation, recommendation generation, and alert delivery are queued jobs. |
| Frontend | React 19 + Vite + Tailwind CSS + Recharts | Dashboard-heavy product requires interactive charts (cost trends, efficiency heatmaps). React with Recharts provides composable chart components. Tailwind keeps styling consistent without a heavy component library. |
| API client generation | openapi-typescript | Auto-generates TypeScript types from the OpenAPI spec, keeping frontend types in sync with the backend. |
| Containerisation | Docker multi-stage builds + Helm chart | Standard Kubernetes deployment mechanism. Multi-stage builds produce minimal Go binaries. Helm chart is the expected install method for K8s-native tools (Kubecost, OpenCost, CAST AI all use Helm). |
| Testing (Go) | Go stdlib testing + testify + testcontainers-go | testify for assertions and mocks; testcontainers-go for integration tests against real PostgreSQL/TimescaleDB instances. |
| Testing (Frontend) | Vitest + React Testing Library + Playwright | Vitest for unit tests, RTL for component tests, Playwright for E2E dashboard tests. |
| Linting / Formatting | golangci-lint (Go), ESLint + Prettier (TS) | Standard ecosystem tools. golangci-lint runs 50+ linters including staticcheck, gosec, and errcheck. |
| CI/CD | GitHub Actions | Standard for open-source projects; free for public repos. |
| MCP server | mcp-go | Go SDK for the Model Context Protocol; exposes cost allocation data to AI agents. |
| Cloud billing SDKs | aws-sdk-go-v2, google-cloud-go, azure-sdk-for-go | Direct cloud billing API integration for cost attribution. |

### Project Structure

```
kubernetes-cost-optimizer/
├── api/
│   └── openapi.yaml                    # OpenAPI 3.1 specification (source of truth)
├── cmd/
│   ├── server/
│   │   └── main.go                     # API server + worker entrypoint
│   ├── agent/
│   │   └── main.go                     # In-cluster agent entrypoint
│   └── cli/
│       └── main.go                     # CLI tool (kco)
├── internal/
│   ├── api/                            # Generated + hand-written HTTP handlers
│   │   ├── generated.go                # oapi-codegen output
│   │   ├── handlers_allocation.go
│   │   ├── handlers_recommendation.go
│   │   ├── handlers_budget.go
│   │   ├── handlers_anomaly.go
│   │   └── middleware.go
│   ├── collector/                      # Kubernetes resource usage collector
│   │   ├── metrics_collector.go
│   │   ├── node_collector.go
│   │   ├── workload_watcher.go
│   │   └── vpa_watcher.go
│   ├── allocation/                     # Cost allocation engine
│   │   ├── engine.go
│   │   ├── pricing.go
│   │   ├── shared_cost.go
│   │   └── focus_export.go
│   ├── recommendation/                 # Rightsizing recommendation engine
│   │   ├── analyzer.go
│   │   ├── cpu_rightsizer.go
│   │   ├── memory_rightsizer.go
│   │   └── confidence.go
│   ├── enforcement/                    # Autonomous rightsizing enforcement
│   │   ├── enforcer.go
│   │   ├── guardrails.go
│   │   └── rollback.go
│   ├── spot/                           # Spot instance optimisation
│   │   ├── advisor.go
│   │   ├── migration.go
│   │   └── interruption_handler.go
│   ├── anomaly/                        # Cost anomaly detection
│   │   ├── detector.go
│   │   ├── attribution.go
│   │   └── z_score.go
│   ├── billing/                        # Cloud billing ingestion
│   │   ├── aws_cur.go
│   │   ├── gcp_bigquery.go
│   │   ├── azure_export.go
│   │   └── on_prem.go
│   ├── alert/                          # Notification delivery
│   │   ├── notifier.go
│   │   ├── slack.go
│   │   ├── email.go
│   │   └── webhook.go
│   ├── mcp/                            # MCP server for AI agent access
│   │   ├── server.go
│   │   └── tools.go
│   ├── store/                          # Database access layer
│   │   ├── db.go
│   │   ├── clusters.go
│   │   ├── namespaces.go
│   │   ├── workloads.go
│   │   ├── cost_allocations.go
│   │   ├── recommendations.go
│   │   ├── budgets.go
│   │   ├── anomalies.go
│   │   └── billing.go
│   ├── config/                         # Configuration management
│   │   ├── config.go
│   │   └── validate.go
│   └── domain/                         # Domain types (shared across packages)
│       ├── cluster.go
│       ├── allocation.go
│       ├── recommendation.go
│       ├── budget.go
│       ├── anomaly.go
│       └── billing.go
├── migrations/
│   ├── 001_create_entity_tables.up.sql
│   ├── 001_create_entity_tables.down.sql
│   ├── 002_create_hypertables.up.sql
│   ├── 002_create_hypertables.down.sql
│   ├── 003_create_continuous_aggregates.up.sql
│   ├── ...
├── deploy/
│   └── helm/
│       └── kco/
│           ├── Chart.yaml
│           ├── values.yaml
│           └── templates/
│               ├── deployment-server.yaml
│               ├── deployment-agent.yaml
│               ├── service.yaml
│               ├── configmap.yaml
│               ├── serviceaccount.yaml
│               ├── clusterrole.yaml
│               └── clusterrolebinding.yaml
├── web/                                # React frontend
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx
│   │   ├── api/                        # Generated API client types
│   │   ├── components/
│   │   │   ├── dashboard/
│   │   │   ├── allocation/
│   │   │   ├── recommendations/
│   │   │   ├── budgets/
│   │   │   └── common/
│   │   ├── hooks/
│   │   ├── pages/
│   │   └── utils/
│   └── tests/
│       ├── components/
│       └── e2e/
├── testdata/                           # Fixture files for tests
│   ├── metrics/
│   ├── billing/
│   └── k8s_manifests/
├── Dockerfile
├── Dockerfile.agent
├── docker-compose.yml
├── Makefile
├── go.mod
├── go.sum
└── README.md
```

---

## Phase 1: Foundation — Project Scaffold, Configuration & Database Schema

### Purpose
Establish the project skeleton, build system, configuration loading, database connection management, and the core database schema using TimescaleDB. After this phase, the project compiles, connects to a database, runs migrations, and has a CI pipeline producing Docker images.

### Tasks

#### 1.1 — Project Scaffold and Build System

**What**: Initialise the Go module, directory structure, Makefile targets, Dockerfile, and docker-compose for local development.

**Design**:

```go
// go.mod
module github.com/worlds-biggest-software-project/kubernetes-cost-optimizer

go 1.23

require (
    github.com/go-chi/chi/v5       v5.1.0
    github.com/jackc/pgx/v5         v5.7.0
    github.com/golang-migrate/migrate/v4 v4.18.0
    github.com/stretchr/testify     v1.9.0
    github.com/rs/zerolog           v1.33.0
    github.com/caarlos0/env/v11     v11.0.0
    github.com/riverqueue/river     v0.14.0
)
```

Makefile targets:
- `make build` — compile `cmd/server`, `cmd/agent`, `cmd/cli`
- `make test` — run `go test ./...`
- `make lint` — run `golangci-lint run`
- `make migrate-up` — apply database migrations
- `make migrate-down` — rollback last migration
- `make generate` — run `oapi-codegen` and `sqlc` (future)
- `make docker-build` — build Docker images
- `make dev` — `docker-compose up` for local development

docker-compose.yml services:
- `postgres` — PostgreSQL 16 with TimescaleDB extension
- `server` — API server (hot-reload via air)
- `prometheus` — Prometheus for local metrics testing

**Testing**:
- `Build: go build ./... succeeds with zero warnings`
- `Build: make docker-build produces server and agent images`
- `Compose: docker-compose up starts postgres, server, and prometheus without errors`
- `Lint: golangci-lint run reports no issues on the scaffold`

#### 1.2 — Configuration Management

**What**: Typed configuration loading from environment variables, config file, and CLI flags with validation.

**Design**:

```go
// internal/config/config.go
package config

import "time"

type Config struct {
    // Server
    ServerAddr      string        `env:"KCO_SERVER_ADDR" envDefault:":8080"`
    MetricsAddr     string        `env:"KCO_METRICS_ADDR" envDefault:":9090"`
    MCPAddr         string        `env:"KCO_MCP_ADDR" envDefault:":8081"`

    // Database
    DatabaseURL     string        `env:"KCO_DATABASE_URL,required"`
    MaxDBConns      int           `env:"KCO_MAX_DB_CONNS" envDefault:"25"`
    MinDBConns      int           `env:"KCO_MIN_DB_CONNS" envDefault:"5"`

    // Kubernetes
    KubeConfig      string        `env:"KUBECONFIG"`
    InCluster       bool          `env:"KCO_IN_CLUSTER" envDefault:"false"`

    // Collector
    ScrapeInterval  time.Duration `env:"KCO_SCRAPE_INTERVAL" envDefault:"60s"`
    AllocationWindow time.Duration `env:"KCO_ALLOCATION_WINDOW" envDefault:"1h"`

    // Cloud providers
    AWSRegion       string        `env:"AWS_REGION"`
    AWSCURBucket    string        `env:"KCO_AWS_CUR_BUCKET"`
    GCPProjectID    string        `env:"KCO_GCP_PROJECT_ID"`
    GCPBillingDataset string      `env:"KCO_GCP_BILLING_DATASET"`
    AzureSubscriptionID string    `env:"KCO_AZURE_SUBSCRIPTION_ID"`

    // Alerting
    SlackWebhookURL string        `env:"KCO_SLACK_WEBHOOK_URL"`
    SMTPHost        string        `env:"KCO_SMTP_HOST"`
    SMTPPort        int           `env:"KCO_SMTP_PORT" envDefault:"587"`
    SMTPFrom        string        `env:"KCO_SMTP_FROM"`

    // Enforcement
    EnforcementEnabled bool       `env:"KCO_ENFORCEMENT_ENABLED" envDefault:"false"`
    DryRun             bool       `env:"KCO_DRY_RUN" envDefault:"true"`

    // Log
    LogLevel        string        `env:"KCO_LOG_LEVEL" envDefault:"info"`
    LogFormat       string        `env:"KCO_LOG_FORMAT" envDefault:"json"`
}

func Load() (*Config, error)     // parse env + validate
func (c *Config) Validate() error // required field checks, URL parsing
```

**Testing**:
- `Unit: Load with all env vars set returns fully populated Config`
- `Unit: Load with missing KCO_DATABASE_URL returns error mentioning the field name`
- `Unit: Validate with invalid ScrapeInterval (<10s) returns validation error`
- `Unit: Default values are applied when env vars are absent`
- `Unit: Load parses duration strings ("60s", "5m") correctly`

#### 1.3 — Database Connection and Migration Framework

**What**: PostgreSQL connection pool with TimescaleDB, migration runner, and health check endpoint.

**Design**:

```go
// internal/store/db.go
package store

import (
    "context"
    "github.com/jackc/pgx/v5/pgxpool"
)

type DB struct {
    Pool *pgxpool.Pool
}

func New(ctx context.Context, databaseURL string, maxConns, minConns int) (*DB, error)
func (db *DB) Close()
func (db *DB) Ping(ctx context.Context) error
func (db *DB) Migrate(migrationsDir string) error  // runs golang-migrate
func (db *DB) MigrateDown(migrationsDir string, steps int) error
```

Health check endpoint: `GET /healthz` returns `{"status": "ok", "db": "connected", "version": "0.1.0"}` or `503` with error details.

**Testing**:
- `Integration (testcontainers): New connects to TimescaleDB container, Ping succeeds`
- `Integration (testcontainers): Migrate applies all migration files in order, tables exist`
- `Integration (testcontainers): MigrateDown rolls back the latest migration`
- `Integration (testcontainers): New with invalid URL returns connection error`
- `Unit: /healthz returns 200 when DB is connected`
- `Unit: /healthz returns 503 when DB is unreachable`

#### 1.4 — Core Database Schema (Entity Metadata + Hypertables)

**What**: Create the database migrations implementing the TimescaleDB data model (Data Model Suggestion 2), covering entity metadata tables, time-series hypertables, and continuous aggregates.

**Design**:

Migration 001 — Entity metadata tables (standard PostgreSQL):
```sql
-- clusters, namespaces, workloads, nodes, teams, organisations, users
-- Directly from Data Model Suggestion 2 entity metadata tables
```

Migration 002 — Time-series hypertables:
```sql
-- resource_usage_metrics hypertable (1-hour chunks)
-- cost_allocations hypertable (1-day chunks)
-- node_cost_metrics hypertable (1-day chunks)
-- With compression policies and retention policies
```

Migration 003 — Continuous aggregates:
```sql
-- cost_by_namespace_hourly
-- cost_by_namespace_daily
-- cost_by_workload_daily
-- node_idle_cost_daily
```

Migration 004 — Operational tables:
```sql
-- rightsizing_recommendations
-- budgets
-- cost_anomalies
```

Migration 005 — Guardrail policies and cloud billing:
```sql
-- guardrail_policies
-- cloud_billing_records (FOCUS v1.3 aligned)
```

The schema is taken directly from Data Model Suggestion 2 with the addition of `guardrail_policies` from Data Model Suggestion 3 and `cloud_billing_records` using FOCUS v1.3 field names from Data Model Suggestion 1.

**Testing**:
- `Integration (testcontainers): All migrations apply cleanly on a fresh TimescaleDB instance`
- `Integration (testcontainers): Hypertables are created with correct chunk intervals`
- `Integration (testcontainers): Continuous aggregates exist and are queryable`
- `Integration (testcontainers): Compression policies are configured on hypertables`
- `Integration (testcontainers): Retention policies are configured (90 days for raw metrics)`
- `Integration (testcontainers): MigrateDown reverses all migrations cleanly`
- `Integration (testcontainers): Re-running migrate on an already-migrated DB is idempotent`

#### 1.5 — Domain Types and Store Layer Foundations

**What**: Define Go domain types matching the database schema and implement basic CRUD store methods for entity metadata tables.

**Design**:

```go
// internal/domain/cluster.go
package domain

import (
    "time"
    "github.com/google/uuid"
)

type Cluster struct {
    ID              uuid.UUID         `json:"id"`
    Name            string            `json:"name"`
    Provider        CloudProvider     `json:"provider"`
    Region          string            `json:"region,omitempty"`
    K8sVersion      string            `json:"k8s_version,omitempty"`
    Status          EntityStatus      `json:"status"`
    Metadata        map[string]any    `json:"metadata,omitempty"`
    CreatedAt       time.Time         `json:"created_at"`
    UpdatedAt       time.Time         `json:"updated_at"`
}

type CloudProvider string

const (
    ProviderAWS    CloudProvider = "aws"
    ProviderGCP    CloudProvider = "gcp"
    ProviderAzure  CloudProvider = "azure"
    ProviderOnPrem CloudProvider = "on_prem"
)

type EntityStatus string

const (
    StatusActive         EntityStatus = "active"
    StatusInactive       EntityStatus = "inactive"
    StatusDecommissioned EntityStatus = "decommissioned"
)

// internal/domain/allocation.go
type CostAllocation struct {
    Time            time.Time  `json:"time"`
    ClusterID       uuid.UUID  `json:"cluster_id"`
    Namespace       string     `json:"namespace"`
    WorkloadName    string     `json:"workload_name,omitempty"`
    WorkloadKind    string     `json:"workload_kind,omitempty"`
    PodName         string     `json:"pod_name"`
    ContainerName   string     `json:"container_name"`
    NodeName        string     `json:"node_name,omitempty"`
    CPUCost         float64    `json:"cpu_cost"`
    CPUCoreHours    float64    `json:"cpu_core_hours"`
    CPUEfficiency   *float64   `json:"cpu_efficiency,omitempty"`
    RAMCost         float64    `json:"ram_cost"`
    RAMByteHours    float64    `json:"ram_byte_hours"`
    RAMEfficiency   *float64   `json:"ram_efficiency,omitempty"`
    GPUCost         float64    `json:"gpu_cost"`
    GPUHours        float64    `json:"gpu_hours"`
    NetworkCost     float64    `json:"network_cost"`
    PVCost          float64    `json:"pv_cost"`
    LBCost          float64    `json:"lb_cost"`
    SharedCost      float64    `json:"shared_cost"`
    TotalCost       float64    `json:"total_cost"`
    NodeHourlyRate  *float64   `json:"node_hourly_rate,omitempty"`
    IsSpot          bool       `json:"is_spot"`
}

type ResourceUsageMetric struct {
    Time           time.Time  `json:"time"`
    ClusterID      uuid.UUID  `json:"cluster_id"`
    Namespace      string     `json:"namespace"`
    WorkloadName   string     `json:"workload_name,omitempty"`
    WorkloadKind   string     `json:"workload_kind,omitempty"`
    PodName        string     `json:"pod_name"`
    ContainerName  string     `json:"container_name"`
    NodeName       string     `json:"node_name,omitempty"`
    CPURequest     float64    `json:"cpu_request"`
    CPULimit       float64    `json:"cpu_limit"`
    CPUUsage       float64    `json:"cpu_usage"`
    MemoryRequest  int64      `json:"memory_request"`
    MemoryLimit    int64      `json:"memory_limit"`
    MemoryUsage    int64      `json:"memory_usage"`
    GPURequest     float64    `json:"gpu_request"`
    GPUUsage       float64    `json:"gpu_usage"`
    NetworkRxBytes int64      `json:"network_rx_bytes"`
    NetworkTxBytes int64      `json:"network_tx_bytes"`
}

// internal/store/clusters.go
type ClusterStore interface {
    Create(ctx context.Context, c *domain.Cluster) error
    GetByID(ctx context.Context, id uuid.UUID) (*domain.Cluster, error)
    GetByName(ctx context.Context, name string) (*domain.Cluster, error)
    List(ctx context.Context, filter ClusterFilter) ([]domain.Cluster, error)
    Update(ctx context.Context, c *domain.Cluster) error
    Delete(ctx context.Context, id uuid.UUID) error
}

type ClusterFilter struct {
    Provider *domain.CloudProvider
    Status   *domain.EntityStatus
    Limit    int
    Offset   int
}
```

Similar store interfaces for `NamespaceStore`, `WorkloadStore`, `NodeStore`, `TeamStore`.

**Testing**:
- `Integration (testcontainers): ClusterStore.Create inserts a cluster, GetByID retrieves it`
- `Integration (testcontainers): ClusterStore.Create with duplicate name returns conflict error`
- `Integration (testcontainers): ClusterStore.List with provider filter returns only AWS clusters`
- `Integration (testcontainers): ClusterStore.Update changes the status, UpdatedAt is advanced`
- `Integration (testcontainers): ClusterStore.Delete removes the cluster`
- `Integration (testcontainers): NamespaceStore.Create with non-existent cluster_id returns FK error`
- `Unit: Domain type JSON marshalling produces OpenCost-aligned field names`

---

## Phase 2: Kubernetes Resource Collection

### Purpose
Build the in-cluster agent that watches Kubernetes resources (namespaces, workloads, pods, nodes) and collects resource usage metrics from the Metrics API. After this phase, the agent discovers cluster topology, writes entity metadata to the database, and streams resource usage samples into the `resource_usage_metrics` hypertable at configurable intervals.

### Tasks

#### 2.1 — Kubernetes Client Bootstrap

**What**: Initialise the Kubernetes client (in-cluster or kubeconfig) and provide typed access to core APIs.

**Design**:

```go
// internal/collector/k8s_client.go
package collector

import (
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/rest"
    metricsv "k8s.io/metrics/pkg/client/clientset/versioned"
)

type K8sClient struct {
    Clientset       kubernetes.Interface
    MetricsClient   metricsv.Interface
    ClusterName     string
    ClusterProvider domain.CloudProvider
}

func NewK8sClient(kubeconfig string, inCluster bool) (*K8sClient, error)
```

Cloud provider detection: inspect node labels (`node.kubernetes.io/instance-type`, `topology.kubernetes.io/zone`) and cloud provider-specific annotations to auto-detect provider and region.

**Testing**:
- `Integration (kind): NewK8sClient with in-cluster config connects to the kind cluster`
- `Unit (mocked): NewK8sClient with invalid kubeconfig path returns clear error`
- `Unit (mocked): Provider detection from node labels correctly identifies aws, gcp, azure`
- `Unit (mocked): Provider detection with no cloud labels defaults to on_prem`

#### 2.2 — Workload Watcher (Entity Discovery)

**What**: Watch namespaces, deployments, statefulsets, daemonsets, and jobs; upsert entity metadata into the database.

**Design**:

```go
// internal/collector/workload_watcher.go
package collector

type WorkloadWatcher struct {
    k8s      *K8sClient
    store    store.WorkloadStore
    nsStore  store.NamespaceStore
    logger   zerolog.Logger
}

func NewWorkloadWatcher(k8s *K8sClient, store store.WorkloadStore, nsStore store.NamespaceStore) *WorkloadWatcher

// Start begins watching and calls handlers for Add/Update/Delete events.
func (w *WorkloadWatcher) Start(ctx context.Context) error

// SyncAll performs a full list of all namespaces and workloads, upserting them.
// Used on startup and periodically to catch any missed events.
func (w *WorkloadWatcher) SyncAll(ctx context.Context) error
```

Watch targets:
- `v1/Namespace` — upsert to `namespaces` table
- `apps/v1/Deployment` — upsert to `workloads` with kind="Deployment"
- `apps/v1/StatefulSet` — upsert to `workloads` with kind="StatefulSet"
- `apps/v1/DaemonSet` — upsert to `workloads` with kind="DaemonSet"
- `batch/v1/Job` — upsert to `workloads` with kind="Job"
- `batch/v1/CronJob` — upsert to `workloads` with kind="CronJob"

Labels and annotations stored as JSONB. `last_seen_at` updated on every sync.

**Testing**:
- `Integration (kind): SyncAll discovers all namespaces in the kind cluster`
- `Integration (kind): Creating a new Deployment triggers an Add event and inserts into workloads`
- `Integration (kind): Deleting a Deployment updates its status to inactive`
- `Integration (kind): Labels on a namespace are stored as JSONB and queryable`
- `Unit (mocked): SyncAll handles API pagination (continue token) correctly`
- `Unit (mocked): SyncAll marks workloads not seen in the latest sync as inactive`

#### 2.3 — Node Collector

**What**: Discover and track cluster nodes with their instance types, capacity, spot status, and pricing.

**Design**:

```go
// internal/collector/node_collector.go
package collector

type NodeCollector struct {
    k8s      *K8sClient
    store    store.NodeStore
    pricing  *PricingLookup
    logger   zerolog.Logger
}

func NewNodeCollector(k8s *K8sClient, store store.NodeStore, pricing *PricingLookup) *NodeCollector

func (nc *NodeCollector) SyncNodes(ctx context.Context) error

// PricingLookup resolves instance type + region to hourly cost.
type PricingLookup struct {
    // Static pricing loaded from embedded CSV or cloud billing APIs
    prices map[string]map[string]float64  // provider -> instance_type -> hourly_rate
}

func NewPricingLookup() *PricingLookup
func (p *PricingLookup) GetHourlyCost(provider domain.CloudProvider, instanceType, region string) (float64, error)
func (p *PricingLookup) IsSpot(node corev1.Node) bool
```

Spot detection: check `kubernetes.io/os`, `node.kubernetes.io/lifecycle=spot` (AWS), `cloud.google.com/gke-preemptible` (GCP), `kubernetes.azure.com/scalesetpriority=spot` (Azure).

**Testing**:
- `Integration (kind): SyncNodes discovers all nodes with correct capacity`
- `Unit: IsSpot correctly identifies spot nodes on AWS, GCP, and Azure via labels`
- `Unit: GetHourlyCost returns correct price for m5.xlarge in us-east-1`
- `Unit: GetHourlyCost returns error for unknown instance type`
- `Unit: SyncNodes updates existing node records rather than duplicating`

#### 2.4 — Metrics Collector (Resource Usage Sampling)

**What**: Scrape CPU and memory usage from the Kubernetes Metrics API at configurable intervals and write samples to the `resource_usage_metrics` hypertable.

**Design**:

```go
// internal/collector/metrics_collector.go
package collector

type MetricsCollector struct {
    k8s            *K8sClient
    store          store.MetricsStore
    clusterID      uuid.UUID
    scrapeInterval time.Duration
    logger         zerolog.Logger
}

func NewMetricsCollector(k8s *K8sClient, store store.MetricsStore, clusterID uuid.UUID, interval time.Duration) *MetricsCollector

// Run starts the periodic scrape loop. Blocks until ctx is cancelled.
func (mc *MetricsCollector) Run(ctx context.Context) error

// Scrape performs a single scrape of all pod metrics and writes to the DB.
func (mc *MetricsCollector) Scrape(ctx context.Context) (int, error)  // returns count of samples written

// internal/store/cost_allocations.go (metrics portion)
type MetricsStore interface {
    InsertUsageMetrics(ctx context.Context, metrics []domain.ResourceUsageMetric) error
}
```

The scraper calls `metricsClient.MetricsV1beta1().PodMetricses("").List()` and enriches each sample with namespace, workload name/kind (from owner references), and node name. Batch inserts use `COPY` protocol for high-throughput writes.

**Testing**:
- `Integration (kind + metrics-server): Scrape returns >0 samples from a cluster with running pods`
- `Integration (testcontainers): InsertUsageMetrics batch-inserts 1000 samples in <100ms`
- `Unit (mocked): Scrape correctly maps pod owner references to workload name and kind`
- `Unit (mocked): Scrape handles pods without owner references (standalone pods)`
- `Unit (mocked): Scrape skips pods in Succeeded/Failed phase`
- `Unit (mocked): Run calls Scrape at the configured interval`
- `Unit (mocked): Scrape continues if one pod's metrics are unavailable`

---

## Phase 3: Cost Allocation Engine

### Purpose
Build the core cost allocation engine that combines resource usage metrics with node pricing to compute per-container, per-workload, and per-namespace costs aligned with the OpenCost Specification. After this phase, the system produces hourly cost allocation records and continuous aggregates power efficient dashboard queries.

### Tasks

#### 3.1 — Allocation Engine Core

**What**: Compute cost allocation for each container based on its share of node resources, aligned with OpenCost Specification field names.

**Design**:

```go
// internal/allocation/engine.go
package allocation

type Engine struct {
    metricsStore store.MetricsStore
    allocStore   store.AllocationStore
    nodeStore    store.NodeStore
    pricing      *collector.PricingLookup
    logger       zerolog.Logger
}

func NewEngine(ms store.MetricsStore, as store.AllocationStore, ns store.NodeStore, p *collector.PricingLookup) *Engine

// ComputeWindow calculates cost allocations for a given time window.
// 1. Query resource_usage_metrics for the window
// 2. Group by container
// 3. For each container, compute:
//    - cpu_core_hours = avg(cpu_usage) * window_hours
//    - cpu_cost = cpu_core_hours * (node_hourly_rate * (cpu_capacity_share))
//    - ram_byte_hours = avg(memory_usage) * window_hours
//    - ram_cost = ram_byte_hours * (node_hourly_rate * (memory_capacity_share))
//    - cpu_efficiency = avg(cpu_usage) / avg(cpu_request)
//    - ram_efficiency = avg(memory_usage) / avg(memory_request)
//    - total_cost = cpu_cost + ram_cost + gpu_cost + network_cost + pv_cost + lb_cost + shared_cost
// 4. Insert into cost_allocations hypertable
func (e *Engine) ComputeWindow(ctx context.Context, windowStart, windowEnd time.Time) (int, error)

// RunPeriodic starts a loop computing allocations at the configured window interval.
func (e *Engine) RunPeriodic(ctx context.Context, interval time.Duration) error

// internal/store/cost_allocations.go
type AllocationStore interface {
    InsertAllocations(ctx context.Context, allocs []domain.CostAllocation) error
    QueryAllocations(ctx context.Context, filter AllocationFilter) ([]domain.CostAllocation, error)
    QueryNamespaceCostDaily(ctx context.Context, clusterID uuid.UUID, namespace string, from, to time.Time) ([]NamespaceCostDaily, error)
    QueryWorkloadCostDaily(ctx context.Context, clusterID uuid.UUID, namespace, workload string, from, to time.Time) ([]WorkloadCostDaily, error)
    QueryTopWorkloads(ctx context.Context, clusterID uuid.UUID, from, to time.Time, limit int) ([]WorkloadCostSummary, error)
}

type AllocationFilter struct {
    ClusterID  *uuid.UUID
    Namespace  *string
    Workload   *string
    WindowFrom time.Time
    WindowTo   time.Time
    Limit      int
    Offset     int
}

type NamespaceCostDaily struct {
    Bucket           time.Time `json:"bucket"`
    ClusterID        uuid.UUID `json:"cluster_id"`
    Namespace        string    `json:"namespace"`
    CPUCost          float64   `json:"cpu_cost"`
    RAMCost          float64   `json:"ram_cost"`
    GPUCost          float64   `json:"gpu_cost"`
    NetworkCost      float64   `json:"network_cost"`
    PVCost           float64   `json:"pv_cost"`
    TotalCost        float64   `json:"total_cost"`
    AvgCPUEfficiency *float64  `json:"avg_cpu_efficiency"`
    AvgRAMEfficiency *float64  `json:"avg_ram_efficiency"`
    WorkloadCount    int       `json:"workload_count"`
}
```

Cost computation formula (per container per window):
1. `cpu_share = container_cpu_request / node_allocatable_cpu`
2. `mem_share = container_memory_request / node_allocatable_memory`
3. `cpu_cost = cpu_share * node_hourly_rate * window_hours * cpu_cost_weight` (where `cpu_cost_weight` is typically 0.5, splitting node cost between CPU and memory)
4. `ram_cost = mem_share * node_hourly_rate * window_hours * (1 - cpu_cost_weight)`

**Testing**:
- `Unit: ComputeWindow with 2 containers on 1 node, costs sum to node hourly rate * window hours`
- `Unit: ComputeWindow with spot node uses spot_hourly_rate instead of on-demand rate`
- `Unit: CPU efficiency = 0.5 when usage is half of request`
- `Unit: Container with 0 CPU request gets 0 CPU cost (avoid division by zero)`
- `Unit: Shared cost distribution across containers by weighted resource usage`
- `Integration (testcontainers): ComputeWindow writes allocations queryable via continuous aggregates`
- `Integration (testcontainers): QueryNamespaceCostDaily returns correct daily rollup after multiple hourly inserts`
- `Integration (testcontainers): QueryTopWorkloads returns workloads sorted by total_cost desc`

#### 3.2 — Shared Cost Distribution

**What**: Allocate cluster-level shared costs (control plane, monitoring, ingress, kube-system) proportionally across user namespaces.

**Design**:

```go
// internal/allocation/shared_cost.go
package allocation

type SharedCostDistributor struct {
    allocStore store.AllocationStore
    rules      []SharedCostRule
}

type SharedCostRule struct {
    SourceNamespaces []string   // e.g. ["kube-system", "monitoring", "ingress-nginx"]
    DistributionMode string    // "proportional_cpu", "proportional_memory", "proportional_cost", "equal"
    ExcludeNamespaces []string  // namespaces that don't receive shared costs
}

// Distribute takes the total cost of source namespaces for a window
// and distributes it across target namespaces.
func (d *SharedCostDistributor) Distribute(ctx context.Context, clusterID uuid.UUID, windowStart, windowEnd time.Time) error
```

Default rule: `kube-system`, `monitoring`, and `ingress-nginx` costs are distributed proportionally by total cost across all other namespaces.

**Testing**:
- `Unit: Proportional distribution — namespace consuming 60% of cost receives 60% of shared cost`
- `Unit: Equal distribution — 3 target namespaces each receive 1/3 of shared cost`
- `Unit: Excluded namespaces receive zero shared cost`
- `Unit: Source namespaces are not included in distribution targets`
- `Integration (testcontainers): Distribute updates shared_cost column on existing allocation records`

#### 3.3 — Node Cost Metrics Writer

**What**: Compute and store per-node cost metrics including idle cost (cost of unallocated resources) and system cost (cost of kube-system pods).

**Design**:

```go
// internal/allocation/node_cost.go
package allocation

type NodeCostWriter struct {
    metricsStore store.MetricsStore
    nodeStore    store.NodeStore
    allocStore   store.AllocationStore
}

// ComputeNodeCosts computes node-level metrics for a time window.
// idle_cost = node_hourly_rate * (1 - allocated_fraction)
// system_cost = sum of kube-system pod allocations on this node
func (w *NodeCostWriter) ComputeNodeCosts(ctx context.Context, clusterID uuid.UUID, windowStart, windowEnd time.Time) error
```

Writes to `node_cost_metrics` hypertable. The `node_idle_cost_daily` continuous aggregate auto-materialises daily rollups.

**Testing**:
- `Unit: Node with 50% CPU allocated has idle_cost = 50% of hourly rate`
- `Unit: Node with no pods has idle_cost = 100% of hourly rate`
- `Unit: System cost equals sum of kube-system namespace allocations on the node`
- `Integration (testcontainers): ComputeNodeCosts writes to node_cost_metrics, visible in node_idle_cost_daily aggregate`

---

## Phase 4: REST API and OpenAPI Specification

### Purpose
Expose the cost allocation data, cluster management, and budget configuration via a REST API with a complete OpenAPI 3.1 specification. After this phase, external tools, CI pipelines, and the frontend can query cost data programmatically.

### Tasks

#### 4.1 — OpenAPI Specification

**What**: Define the complete OpenAPI 3.1 spec for all API endpoints. This spec is the source of truth; server stubs and TypeScript client types are generated from it.

**Design**:

```yaml
# api/openapi.yaml (key paths)
openapi: "3.1.0"
info:
  title: Kubernetes Cost Optimizer API
  version: 0.1.0
paths:
  /api/v1/clusters:
    get:
      summary: List registered clusters
      parameters:
        - name: provider
          in: query
          schema: { type: string, enum: [aws, gcp, azure, on_prem] }
        - name: status
          in: query
          schema: { type: string, enum: [active, inactive] }
      responses:
        200:
          content:
            application/json:
              schema:
                type: object
                properties:
                  clusters: { type: array, items: { $ref: '#/components/schemas/Cluster' } }
                  total: { type: integer }
    post:
      summary: Register a new cluster
      requestBody:
        content:
          application/json:
            schema: { $ref: '#/components/schemas/CreateClusterRequest' }
  /api/v1/clusters/{clusterId}:
    get: ...
    put: ...
    delete: ...
  /api/v1/allocations:
    get:
      summary: Query cost allocations
      parameters:
        - name: cluster_id
          in: query
          required: true
          schema: { type: string, format: uuid }
        - name: namespace
          in: query
          schema: { type: string }
        - name: workload
          in: query
          schema: { type: string }
        - name: window_start
          in: query
          required: true
          schema: { type: string, format: date-time }
        - name: window_end
          in: query
          required: true
          schema: { type: string, format: date-time }
        - name: aggregate
          in: query
          schema: { type: string, enum: [container, workload, namespace, cluster] }
  /api/v1/allocations/top-workloads:
    get:
      summary: Top N most expensive workloads
  /api/v1/allocations/namespace-trend:
    get:
      summary: Daily cost trend for a namespace
  /api/v1/allocations/node-idle:
    get:
      summary: Node idle cost analysis
  /api/v1/recommendations:
    get:
      summary: List rightsizing recommendations
  /api/v1/recommendations/{id}/approve:
    post:
      summary: Approve a recommendation for enforcement
  /api/v1/recommendations/{id}/dismiss:
    post:
      summary: Dismiss a recommendation
  /api/v1/budgets:
    get: ...
    post: ...
  /api/v1/budgets/{id}:
    get: ...
    put: ...
    delete: ...
  /api/v1/anomalies:
    get: ...
  /api/v1/anomalies/{id}/acknowledge:
    post: ...
  /healthz:
    get: ...
  /readyz:
    get: ...
```

Components/schemas include: `Cluster`, `Namespace`, `Workload`, `CostAllocation`, `NamespaceCostDaily`, `WorkloadCostDaily`, `Recommendation`, `Budget`, `Anomaly`, `NodeIdleCost`, pagination wrappers, and error responses.

**Testing**:
- `Unit: openapi.yaml passes OpenAPI 3.1 schema validation (via swagger-cli validate)`
- `Unit: oapi-codegen generates Go types and server interface without errors`
- `Unit: openapi-typescript generates TypeScript types without errors`

#### 4.2 — API Handlers — Cluster Management

**What**: Implement CRUD endpoints for cluster registration and management.

**Design**:

```go
// internal/api/handlers_cluster.go
package api

type ClusterHandler struct {
    store store.ClusterStore
}

func (h *ClusterHandler) ListClusters(w http.ResponseWriter, r *http.Request)
func (h *ClusterHandler) CreateCluster(w http.ResponseWriter, r *http.Request)
func (h *ClusterHandler) GetCluster(w http.ResponseWriter, r *http.Request)
func (h *ClusterHandler) UpdateCluster(w http.ResponseWriter, r *http.Request)
func (h *ClusterHandler) DeleteCluster(w http.ResponseWriter, r *http.Request)
```

All handlers validate input against the OpenAPI schema (via oapi-codegen middleware), return standard error responses (`{"error": "...", "code": "CLUSTER_NOT_FOUND"}`), and include pagination for list endpoints.

**Testing**:
- `Integration: POST /api/v1/clusters with valid body returns 201 and cluster with generated ID`
- `Integration: POST /api/v1/clusters with duplicate name returns 409 Conflict`
- `Integration: GET /api/v1/clusters?provider=aws returns only AWS clusters`
- `Integration: GET /api/v1/clusters/{id} with non-existent ID returns 404`
- `Integration: DELETE /api/v1/clusters/{id} returns 204 and subsequent GET returns 404`
- `Unit: Invalid JSON body returns 400 with validation error details`

#### 4.3 — API Handlers — Cost Allocation Queries

**What**: Implement read endpoints for querying cost allocations, namespace trends, top workloads, and node idle costs.

**Design**:

```go
// internal/api/handlers_allocation.go
package api

type AllocationHandler struct {
    store store.AllocationStore
}

func (h *AllocationHandler) QueryAllocations(w http.ResponseWriter, r *http.Request)
func (h *AllocationHandler) GetNamespaceTrend(w http.ResponseWriter, r *http.Request)
func (h *AllocationHandler) GetTopWorkloads(w http.ResponseWriter, r *http.Request)
func (h *AllocationHandler) GetNodeIdleCost(w http.ResponseWriter, r *http.Request)
```

Query endpoints read from continuous aggregates for fast response times. Raw allocation queries for sub-hourly granularity read from the hypertable directly.

Response format aligned with OpenCost allocation API structure for ecosystem compatibility:
```json
{
  "data": [
    {
      "namespace": "production",
      "workload_name": "api-server",
      "cpu_cost": 12.45,
      "ram_cost": 8.30,
      "total_cost": 20.75,
      "cpu_efficiency": 0.62,
      "ram_efficiency": 0.45
    }
  ],
  "window": { "start": "...", "end": "..." },
  "total": 42
}
```

**Testing**:
- `Integration: GET /api/v1/allocations returns allocations for the given time window`
- `Integration: GET /api/v1/allocations?aggregate=namespace returns namespace-level rollup`
- `Integration: GET /api/v1/allocations/top-workloads?limit=5 returns 5 workloads sorted by cost`
- `Integration: GET /api/v1/allocations/namespace-trend returns daily cost series for the namespace`
- `Integration: Time window exceeding 90 days returns 400 (exceeds raw data retention)`
- `Unit: Missing cluster_id parameter returns 400 with field-level error`

#### 4.4 — API Handlers — Budget and Anomaly Endpoints

**What**: CRUD for budgets and read/acknowledge for anomalies.

**Design**:

```go
// internal/api/handlers_budget.go
type BudgetHandler struct {
    store store.BudgetStore
}

func (h *BudgetHandler) ListBudgets(w http.ResponseWriter, r *http.Request)
func (h *BudgetHandler) CreateBudget(w http.ResponseWriter, r *http.Request)
func (h *BudgetHandler) GetBudget(w http.ResponseWriter, r *http.Request)
func (h *BudgetHandler) UpdateBudget(w http.ResponseWriter, r *http.Request)
func (h *BudgetHandler) DeleteBudget(w http.ResponseWriter, r *http.Request)

// internal/api/handlers_anomaly.go
type AnomalyHandler struct {
    store store.AnomalyStore
}

func (h *AnomalyHandler) ListAnomalies(w http.ResponseWriter, r *http.Request)
func (h *AnomalyHandler) AcknowledgeAnomaly(w http.ResponseWriter, r *http.Request)
```

Budget creation validates: scope references an existing cluster/namespace, monthly_limit > 0, alert_thresholds are between 0 and 1 and sorted ascending.

**Testing**:
- `Integration: POST /api/v1/budgets creates budget and returns 201`
- `Integration: POST /api/v1/budgets with negative monthly_limit returns 400`
- `Integration: GET /api/v1/anomalies?status=open returns only open anomalies`
- `Integration: POST /api/v1/anomalies/{id}/acknowledge transitions status to acknowledged`
- `Integration: Acknowledging an already-resolved anomaly returns 409`

#### 4.5 — Prometheus Metrics Export

**What**: Export cost and operational metrics in Prometheus format for integration with existing observability stacks.

**Design**:

Metrics exported on the metrics address (default `:9090/metrics`):

```
# Namespace cost gauges (updated hourly)
kco_namespace_cost_hourly{cluster="prod", namespace="default", cost_type="cpu"} 12.45
kco_namespace_cost_hourly{cluster="prod", namespace="default", cost_type="ram"} 8.30
kco_namespace_cost_hourly{cluster="prod", namespace="default", cost_type="total"} 20.75

# Efficiency gauges
kco_namespace_cpu_efficiency{cluster="prod", namespace="default"} 0.62
kco_namespace_ram_efficiency{cluster="prod", namespace="default"} 0.45

# Node idle cost
kco_node_idle_cost_ratio{cluster="prod", instance_type="m5.xlarge"} 0.35

# Recommendation metrics
kco_pending_recommendations_total{cluster="prod"} 15
kco_estimated_savings_monthly{cluster="prod"} 450.00

# Operational metrics
kco_scrape_duration_seconds{cluster="prod"} 2.3
kco_allocation_compute_duration_seconds{cluster="prod"} 5.1
kco_scrape_errors_total{cluster="prod"} 0
```

**Testing**:
- `Integration: GET /metrics returns valid Prometheus exposition format`
- `Integration: kco_namespace_cost_hourly gauge reflects latest allocation computation`
- `Unit: Metrics are registered without naming collisions`
- `Unit: Histogram buckets for duration metrics cover expected ranges`

---

## Phase 5: Rightsizing Recommendation Engine

### Purpose
Analyse historical resource usage patterns to generate statistically-grounded rightsizing recommendations for CPU and memory requests. After this phase, the system produces actionable recommendations with confidence scores and estimated monthly savings, viewable through the API.

### Tasks

#### 5.1 — Usage Pattern Analyzer

**What**: Compute usage percentiles (p50, p95, p99) from the resource_usage_metrics hypertable for each container.

**Design**:

```go
// internal/recommendation/analyzer.go
package recommendation

type Analyzer struct {
    metricsStore store.MetricsStore
    logger       zerolog.Logger
}

type UsageProfile struct {
    ClusterID     uuid.UUID
    Namespace     string
    WorkloadName  string
    ContainerName string
    AnalysisWindow struct {
        Start time.Time
        End   time.Time
    }
    CPU struct {
        P50     float64
        P95     float64
        P99     float64
        Max     float64
        Request float64
        Limit   float64
    }
    Memory struct {
        P50     int64
        P95     int64
        P99     int64
        Max     int64
        Request int64
        Limit   int64
    }
    SampleCount int
}

// Analyze computes usage profiles for all containers in a cluster
// over the specified analysis window.
// Minimum 24 hours of data required for a valid profile.
// Minimum 100 samples required for statistical significance.
func (a *Analyzer) Analyze(ctx context.Context, clusterID uuid.UUID, window time.Duration) ([]UsageProfile, error)

// AnalyzeWorkload computes a usage profile for a specific workload.
func (a *Analyzer) AnalyzeWorkload(ctx context.Context, clusterID uuid.UUID, namespace, workload string, window time.Duration) ([]UsageProfile, error)
```

Uses TimescaleDB's `percentile_cont` function for efficient percentile computation directly in SQL rather than loading all samples into memory.

**Testing**:
- `Integration (testcontainers): Analyze with 7 days of synthetic data returns profiles for all containers`
- `Integration (testcontainers): P95 CPU usage computed correctly matches expected value from synthetic data`
- `Unit: Analyze with <24h of data returns empty profiles with a warning`
- `Unit: Analyze with <100 samples returns empty profiles with insufficient_data flag`
- `Unit: Memory percentiles are computed in bytes, not converted`

#### 5.2 — CPU and Memory Rightsizer

**What**: Generate rightsizing recommendations from usage profiles, applying configurable safety margins and headroom.

**Design**:

```go
// internal/recommendation/cpu_rightsizer.go
package recommendation

type RightsizeConfig struct {
    CPUTargetPercentile  float64 // default: 0.95 (use P95)
    MemTargetPercentile  float64 // default: 0.99 (use P99 — memory is less elastic)
    CPUHeadroomPct       float64 // default: 0.15 (15% above target percentile)
    MemHeadroomPct       float64 // default: 0.20 (20% above target percentile)
    MinCPURequest        float64 // default: 0.010 (10m)
    MinMemoryRequest     int64   // default: 33554432 (32Mi)
    MinSavingsPct        float64 // default: 0.10 (ignore recommendations saving <10%)
}

type Rightsizer struct {
    config   RightsizeConfig
    pricing  *collector.PricingLookup
    logger   zerolog.Logger
}

type Recommendation struct {
    ClusterID            uuid.UUID
    Namespace            string
    WorkloadName         string
    ContainerName        string
    AnalysisWindowStart  time.Time
    AnalysisWindowEnd    time.Time
    CurrentCPURequest    float64
    CurrentMemoryRequest int64
    RecommendedCPURequest    float64
    RecommendedMemoryRequest int64
    P50CPUUsage          float64
    P95CPUUsage          float64
    P99CPUUsage          float64
    P50MemoryUsage       int64
    P95MemoryUsage       int64
    P99MemoryUsage       int64
    EstimatedMonthlySavings float64
    ConfidenceScore      float64    // 0.0 to 1.0
    Status               string     // "pending", "approved", "applied", "dismissed", "expired"
}

func NewRightsizer(config RightsizeConfig, pricing *collector.PricingLookup) *Rightsizer

// GenerateRecommendations takes usage profiles and produces recommendations.
// Only generates a recommendation if:
// 1. Estimated savings exceed MinSavingsPct
// 2. Confidence score > 0.5
// 3. The container has been running for the full analysis window
func (r *Rightsizer) GenerateRecommendations(profiles []UsageProfile) ([]Recommendation, error)

// ConfidenceScore factors:
// - Sample density (samples / expected_samples)
// - Usage stability (coefficient of variation)
// - Sufficient history (penalty if analysis window < 7 days)
func (r *Rightsizer) ConfidenceScore(profile UsageProfile) float64
```

Recommended CPU = `P95_cpu * (1 + CPUHeadroomPct)`, rounded up to nearest 10m.
Recommended Memory = `P99_memory * (1 + MemHeadroomPct)`, rounded up to nearest 16Mi.

**Testing**:
- `Unit: Container with P95 CPU=0.35, request=1.0 recommends ~0.41 CPU (0.35 * 1.15, rounded)`
- `Unit: Container with P99 memory=489MB, request=1GB recommends ~587MB (489 * 1.20, rounded to 16Mi)`
- `Unit: No recommendation generated when savings < 10%`
- `Unit: No recommendation generated when confidence < 0.5`
- `Unit: ConfidenceScore is 1.0 with 7 days of dense, stable data`
- `Unit: ConfidenceScore is penalised with <48h of data`
- `Unit: Minimum CPU request enforced (never recommends below 10m)`
- `Unit: Minimum memory request enforced (never recommends below 32Mi)`
- `Unit: EstimatedMonthlySavings correctly computed from cost difference`
- `Integration (testcontainers): GenerateRecommendations writes to rightsizing_recommendations table`

#### 5.3 — Recommendation Lifecycle Management

**What**: Track recommendation status (pending, approved, applied, dismissed, expired) and provide API endpoints for managing them.

**Design**:

```go
// internal/store/recommendations.go
type RecommendationStore interface {
    Create(ctx context.Context, rec *domain.Recommendation) error
    GetByID(ctx context.Context, id uuid.UUID) (*domain.Recommendation, error)
    List(ctx context.Context, filter RecommendationFilter) ([]domain.Recommendation, error)
    UpdateStatus(ctx context.Context, id uuid.UUID, status string) error
    ExpireOlderThan(ctx context.Context, age time.Duration) (int, error)
    GetTotalSavings(ctx context.Context, clusterID uuid.UUID, status string) (float64, error)
}

type RecommendationFilter struct {
    ClusterID *uuid.UUID
    Namespace *string
    Status    *string
    MinSavings *float64
    SortBy    string // "savings_desc", "confidence_desc", "created_at_desc"
    Limit     int
    Offset    int
}
```

Recommendations expire after 7 days if not acted upon (configurable). A background job runs `ExpireOlderThan` daily.

**Testing**:
- `Integration: GET /api/v1/recommendations returns pending recommendations sorted by savings`
- `Integration: POST /api/v1/recommendations/{id}/approve transitions to approved`
- `Integration: POST /api/v1/recommendations/{id}/dismiss transitions to dismissed`
- `Integration: Approving an already-applied recommendation returns 409`
- `Integration: Expired recommendations are not returned in default list query`
- `Unit: ExpireOlderThan marks recommendations older than 7 days as expired`
- `Unit: GetTotalSavings sums estimated savings for pending recommendations`

---

## Phase 6: Budget Tracking and Anomaly Detection

### Purpose
Implement proactive cost governance through budget tracking with threshold alerts and automated cost anomaly detection using statistical methods. After this phase, users receive Slack/email notifications when budgets approach limits or unusual cost patterns are detected.

### Tasks

#### 6.1 — Budget Evaluation Engine

**What**: Periodically evaluate current spend against budget limits and trigger alerts at configured thresholds.

**Design**:

```go
// internal/alert/budget_evaluator.go
package alert

type BudgetEvaluator struct {
    budgetStore store.BudgetStore
    allocStore  store.AllocationStore
    notifier    *Notifier
    logger      zerolog.Logger
}

// Evaluate checks all active budgets against current month spend.
// Triggers notifications when spend crosses threshold boundaries.
func (e *BudgetEvaluator) Evaluate(ctx context.Context) error

// EvaluateBudget checks a single budget.
// Returns the alert threshold that was crossed, or nil if no new threshold was crossed.
func (e *BudgetEvaluator) EvaluateBudget(ctx context.Context, budget domain.Budget) (*float64, error)

// ProjectMonthEnd estimates end-of-month spend based on burn rate.
// burn_rate = current_spend / days_elapsed_this_month
// projected = burn_rate * days_in_month
func (e *BudgetEvaluator) ProjectMonthEnd(currentSpend float64, now time.Time) float64
```

Budget scope resolution: namespace scope queries `cost_by_namespace_daily`, label scope queries `cost_allocations` with JSONB label containment, cluster scope queries all namespaces.

**Testing**:
- `Unit: 85% spend with thresholds [0.5, 0.8, 0.9, 1.0] triggers the 0.8 threshold`
- `Unit: 85% spend when 0.8 was already triggered does not re-trigger`
- `Unit: 95% spend when 0.8 was last triggered triggers 0.9`
- `Unit: ProjectMonthEnd on day 15 of a 30-day month doubles current spend`
- `Unit: ProjectMonthEnd on day 1 projects full month based on first day`
- `Integration (testcontainers): Evaluate with budget and allocation data triggers the correct alert`
- `Integration (testcontainers): Budget with label scope correctly sums costs for matching labels`

#### 6.2 — Cost Anomaly Detector

**What**: Detect unusual cost patterns using Z-score analysis against rolling baselines.

**Design**:

```go
// internal/anomaly/detector.go
package anomaly

type Detector struct {
    allocStore   store.AllocationStore
    anomalyStore store.AnomalyStore
    logger       zerolog.Logger
}

type DetectorConfig struct {
    BaselineWindowDays int     // default: 14 (2-week rolling baseline)
    ZScoreThreshold    float64 // default: 3.0 (3 standard deviations)
    MinDailyCost       float64 // default: 1.0 (ignore workloads costing <$1/day)
    DetectionInterval  time.Duration // default: 1h
}

// Detect runs anomaly detection for all namespaces in a cluster.
// For each namespace:
//   1. Compute mean and stddev of daily cost over the baseline window
//   2. Compare today's cost to the baseline
//   3. If z-score > threshold, create a CostAnomaly record
func (d *Detector) Detect(ctx context.Context, clusterID uuid.UUID) ([]domain.CostAnomaly, error)

// internal/anomaly/z_score.go
func ZScore(value, mean, stddev float64) float64 {
    if stddev == 0 {
        if value == mean {
            return 0
        }
        return math.Inf(1)
    }
    return (value - mean) / stddev
}

// Severity mapping:
// z-score 3.0-4.0 → low
// z-score 4.0-6.0 → medium
// z-score 6.0-10.0 → high
// z-score >10.0 → critical
func SeverityFromZScore(z float64) string
```

**Testing**:
- `Unit: ZScore(15, 10, 2) returns 2.5`
- `Unit: ZScore(20, 10, 2) returns 5.0`
- `Unit: ZScore with stddev=0 and value!=mean returns +Inf`
- `Unit: SeverityFromZScore(3.5) returns "low"`
- `Unit: SeverityFromZScore(5.0) returns "medium"`
- `Unit: SeverityFromZScore(8.0) returns "high"`
- `Unit: SeverityFromZScore(12.0) returns "critical"`
- `Unit: Workload costing $0.50/day is skipped (below MinDailyCost)`
- `Integration (testcontainers): Detect finds a cost spike injected into test data`
- `Integration (testcontainers): Detect does not create duplicate anomalies for same workload within 24h`

#### 6.3 — Notification Delivery (Slack, Email, Webhook)

**What**: Send alert notifications through configurable channels.

**Design**:

```go
// internal/alert/notifier.go
package alert

type Notifier struct {
    slack   *SlackSender
    email   *EmailSender
    webhook *WebhookSender
    logger  zerolog.Logger
}

type Notification struct {
    Type    string         // "budget_alert", "anomaly_detected", "recommendation_ready"
    Title   string
    Message string
    Severity string
    Fields  map[string]string  // key-value pairs for structured display
    URL     string             // link to the dashboard
}

type SlackSender struct {
    WebhookURL string
}
func (s *SlackSender) Send(ctx context.Context, n Notification) error

type EmailSender struct {
    Host     string
    Port     int
    From     string
    Username string
    Password string
}
func (e *EmailSender) Send(ctx context.Context, recipients []string, n Notification) error

type WebhookSender struct{}
func (w *WebhookSender) Send(ctx context.Context, url string, n Notification) error
```

Slack message format uses Block Kit with:
- Header: notification title
- Section: message body
- Fields: key-value pairs (namespace, current spend, budget limit, etc.)
- Action button: link to dashboard

**Testing**:
- `Unit (mocked HTTP): SlackSender.Send posts correct Block Kit payload`
- `Unit (mocked HTTP): SlackSender.Send retries once on 429 with Retry-After`
- `Unit (mocked SMTP): EmailSender.Send sends HTML email with correct subject and body`
- `Unit (mocked HTTP): WebhookSender.Send posts JSON payload to the target URL`
- `Integration: Budget alert Notification renders correctly in Slack Block Kit Builder`

---

## Phase 7: Cloud Billing Integration

### Purpose
Ingest cloud billing data from AWS CUR, GCP BigQuery, and Azure Cost Export to provide accurate node-level cost attribution and reconcile Kubernetes cost allocation against actual cloud invoices. After this phase, cost allocation uses real cloud pricing instead of static pricing tables.

### Tasks

#### 7.1 — AWS Cost and Usage Report (CUR) Ingestion

**What**: Read AWS CUR v2 data from S3 (Parquet format) and ingest into the cloud_billing table.

**Design**:

```go
// internal/billing/aws_cur.go
package billing

type AWSCURIngester struct {
    s3Client     *s3.Client
    store        store.BillingStore
    bucket       string
    prefix       string
    logger       zerolog.Logger
}

func NewAWSCURIngester(cfg config.AWSConfig) (*AWSCURIngester, error)

// Ingest reads the latest CUR manifest, downloads Parquet files,
// and upserts billing records into cloud_billing_records.
// Filters to EC2, EKS, and EBS line items relevant to Kubernetes nodes.
func (a *AWSCURIngester) Ingest(ctx context.Context) (int, error)

// MapToNode maps a CUR line item to a Kubernetes node by matching
// resource_id to the node's provider_id (EC2 instance ID).
func (a *AWSCURIngester) MapToNode(resourceID string, nodes []domain.Node) *domain.Node
```

FOCUS v1.3 field mapping:
- `line_item_unblended_cost` -> `billed_cost`
- `line_item_usage_type` -> mapped to `service_category`
- `pricing_term` -> `pricing_category` (OnDemand, Spot, Reserved)

**Testing**:
- `Unit (mocked S3): Ingest reads manifest.json, downloads listed Parquet files`
- `Unit: MapToNode matches EC2 instance ID to node with matching provider_id`
- `Unit: FOCUS field mapping correctly translates CUR column names`
- `Unit: Non-compute line items (S3, Lambda) are filtered out`
- `Integration (localstack): Ingest reads real Parquet files from localstack S3`

#### 7.2 — GCP BigQuery Billing Export Ingestion

**What**: Query GCP billing export from BigQuery and ingest GKE-related records.

**Design**:

```go
// internal/billing/gcp_bigquery.go
package billing

type GCPBillingIngester struct {
    bqClient     *bigquery.Client
    store        store.BillingStore
    projectID    string
    dataset      string
    logger       zerolog.Logger
}

func NewGCPBillingIngester(cfg config.GCPConfig) (*GCPBillingIngester, error)

func (g *GCPBillingIngester) Ingest(ctx context.Context, from, to time.Time) (int, error)
```

Query filters: `service.description = 'Compute Engine'` and `project.id = <project_id>`. Maps GKE node resource names to Kubernetes node names.

**Testing**:
- `Unit (mocked BigQuery): Ingest executes correct SQL query with date range`
- `Unit: GKE resource name mapping to K8s node name`
- `Unit: CUD (Committed Use Discount) records set pricing_category = 'Commitment'`

#### 7.3 — Azure Cost Export Ingestion

**What**: Read Azure Cost Management exports and ingest AKS-related records.

**Design**:

```go
// internal/billing/azure_export.go
package billing

type AzureBillingIngester struct {
    blobClient   *azblob.Client
    store        store.BillingStore
    subscription string
    container    string
    logger       zerolog.Logger
}

func NewAzureBillingIngester(cfg config.AzureConfig) (*AzureBillingIngester, error)

func (a *AzureBillingIngester) Ingest(ctx context.Context) (int, error)
```

**Testing**:
- `Unit (mocked blob): Ingest reads CSV export from Azure blob storage`
- `Unit: AKS resource group filtering correctly isolates Kubernetes costs`
- `Unit: Azure Reservation records set pricing_category = 'Commitment'`

#### 7.4 — Billing Reconciliation

**What**: Reconcile Kubernetes cost allocations against actual cloud billing to ensure accuracy.

**Design**:

```go
// internal/billing/reconciliation.go
package billing

type Reconciler struct {
    billingStore store.BillingStore
    allocStore   store.AllocationStore
    logger       zerolog.Logger
}

type ReconciliationResult struct {
    ClusterID         uuid.UUID
    Period            string    // "2026-05"
    AllocatedTotal    float64
    BilledTotal       float64
    Variance          float64
    VariancePct       float64
    UnmatchedBilling  int       // billing records not mapped to nodes
    UnmatchedNodes    int       // nodes without billing records
}

func (r *Reconciler) Reconcile(ctx context.Context, clusterID uuid.UUID, month time.Time) (*ReconciliationResult, error)
```

**Testing**:
- `Unit: Variance of 0% when allocated and billed totals match exactly`
- `Unit: Variance correctly computed as (billed - allocated) / billed * 100`
- `Unit: Unmatched billing records counted when resource_id has no matching node`
- `Integration (testcontainers): Reconcile produces correct result with mixed billing and allocation data`

---

## Phase 8: Web Dashboard

### Purpose
Build a React dashboard providing visual cost allocation views, efficiency heatmaps, recommendation management, and budget tracking. After this phase, users can explore costs, approve recommendations, and manage budgets through a browser interface.

### Tasks

#### 8.1 — Frontend Scaffold and API Client

**What**: Set up the React + Vite + Tailwind project and generate TypeScript API client types from the OpenAPI spec.

**Design**:

```typescript
// web/src/api/client.ts
import type { paths } from './generated/openapi';
import createClient from 'openapi-fetch';

export const api = createClient<paths>({ baseUrl: '/api/v1' });

// Usage:
// const { data } = await api.GET('/allocations/top-workloads', {
//   params: { query: { cluster_id: '...', limit: 10, window_start: '...', window_end: '...' } }
// });
```

```typescript
// web/src/hooks/useAllocations.ts
import { useQuery } from '@tanstack/react-query';

export function useNamespaceTrend(clusterId: string, namespace: string, days: number) {
    return useQuery({
        queryKey: ['namespace-trend', clusterId, namespace, days],
        queryFn: () => api.GET('/allocations/namespace-trend', {
            params: { query: { cluster_id: clusterId, namespace, days } }
        }).then(r => r.data),
        refetchInterval: 60_000,
    });
}
```

**Testing**:
- `Unit: openapi-typescript generates types matching the OpenAPI spec`
- `Unit: API client correctly constructs request URLs with query parameters`
- `Component: App renders without console errors`

#### 8.2 — Cost Overview Dashboard

**What**: Main dashboard showing cluster cost summary, namespace cost breakdown, and efficiency gauges.

**Design**:

Pages:
- `/` — Cluster overview: total cost (24h, 7d, 30d), top 5 namespaces by cost, cluster efficiency score
- `/clusters/{id}` — Cluster detail: namespace cost breakdown table, cost trend chart, node utilisation

Components:
```typescript
// web/src/components/dashboard/CostSummaryCard.tsx
interface CostSummaryCardProps {
    title: string;
    cost: number;
    trend: number;       // percentage change vs previous period
    currency: string;
}

// web/src/components/dashboard/NamespaceCostChart.tsx
interface NamespaceCostChartProps {
    data: NamespaceCostDaily[];
    days: number;
}
// Renders a stacked area chart (Recharts) with cpu_cost, ram_cost, gpu_cost, network_cost

// web/src/components/dashboard/EfficiencyGauge.tsx
interface EfficiencyGaugeProps {
    cpuEfficiency: number;  // 0-1
    ramEfficiency: number;  // 0-1
    label: string;
}
// Renders a radial gauge using Recharts RadialBarChart
```

**Testing**:
- `Component: CostSummaryCard renders cost formatted as $X.XX`
- `Component: CostSummaryCard shows green arrow for negative trend (cost decrease)`
- `Component: CostSummaryCard shows red arrow for positive trend (cost increase)`
- `Component: NamespaceCostChart renders stacked area with correct legend`
- `Component: EfficiencyGauge shows red below 0.4, yellow 0.4-0.7, green above 0.7`
- `E2E (Playwright): Dashboard loads and displays cost data from the API`

#### 8.3 — Recommendations Page

**What**: Table of rightsizing recommendations with approve/dismiss actions and estimated savings.

**Design**:

Page: `/recommendations`

Components:
```typescript
// web/src/components/recommendations/RecommendationTable.tsx
// Columns: Namespace, Workload, Container, Current CPU, Recommended CPU,
//          Current Memory, Recommended Memory, Monthly Savings, Confidence, Actions

// web/src/components/recommendations/RecommendationDetail.tsx
// Shows usage percentile charts, current vs recommended resource bars,
// and a confirmation dialog for approve/dismiss actions.

// web/src/components/recommendations/SavingsSummary.tsx
// Shows total potential savings across all pending recommendations.
```

**Testing**:
- `Component: RecommendationTable sorts by estimated_monthly_savings descending`
- `Component: Approve button calls POST /recommendations/{id}/approve`
- `Component: Dismiss button calls POST /recommendations/{id}/dismiss and removes row`
- `Component: SavingsSummary displays sum of all pending recommendation savings`
- `E2E (Playwright): User can view, approve, and dismiss recommendations`

#### 8.4 — Budget Management Page

**What**: CRUD interface for budgets with visual spend-vs-limit progress bars.

**Design**:

Page: `/budgets`

Components:
```typescript
// web/src/components/budgets/BudgetCard.tsx
interface BudgetCardProps {
    budget: Budget;
}
// Shows progress bar (current_spend / monthly_limit), color-coded by threshold

// web/src/components/budgets/BudgetForm.tsx
// Create/edit form with scope selector (cluster, namespace, label), monthly limit,
// alert thresholds, and notification channel configuration
```

**Testing**:
- `Component: BudgetCard progress bar is green at 40%, yellow at 75%, red at 95%`
- `Component: BudgetForm validates monthly_limit > 0`
- `Component: BudgetForm submits POST /budgets with correct payload`
- `E2E (Playwright): User creates a budget, sees it in the list, edits it, deletes it`

---

## Phase 9: Autonomous Rightsizing Enforcement

### Purpose
Enable opt-in autonomous enforcement of rightsizing recommendations with configurable guardrails, disruption limits, and automatic rollback. This is the key differentiator over recommendation-only tools like OpenCost and Goldilocks. After this phase, approved recommendations can be automatically applied to Kubernetes workloads.

### Tasks

#### 9.1 — Guardrail Policy Engine

**What**: Evaluate guardrail policies before applying any rightsizing action.

**Design**:

```go
// internal/enforcement/guardrails.go
package enforcement

type GuardrailEngine struct {
    store store.GuardrailStore
}

type GuardrailPolicy struct {
    ID                  uuid.UUID
    ClusterID           uuid.UUID
    Namespace           *string  // nil = cluster-wide
    WorkloadName        *string  // nil = namespace-wide
    PolicyName          string
    MaxCPUChangePct     float64  // max % change per action (default: 50)
    MaxMemoryChangePct  float64  // max % change per action (default: 50)
    MinReplicaCount     int      // never scale below this (default: 1)
    MaxDisruptionPct    float64  // max % of pods disrupted simultaneously (default: 25)
    AllowSpot           bool     // allow spot migration (default: true)
    RequireApproval     bool     // require manual approval before enforcement (default: false)
    ApprovalTimeoutHours int     // auto-expire approval after N hours (default: 24)
    BlackoutWindows     []BlackoutWindow
    CooldownMinutes     int      // min time between actions on same workload (default: 60)
}

type BlackoutWindow struct {
    Day   string // "monday", "tuesday", ..., "sunday"
    After string // "HH:MM" (24h format), optional
}

type GuardrailResult struct {
    Allowed    bool
    Violations []string  // human-readable list of violated policies
}

// Evaluate checks whether a proposed action is allowed by all applicable policies.
// Policies are evaluated from most-specific (workload) to least-specific (cluster).
// If any policy blocks, the action is denied.
func (g *GuardrailEngine) Evaluate(ctx context.Context, action ProposedAction) (*GuardrailResult, error)
```

**Testing**:
- `Unit: Action changing CPU by 60% blocked by 50% max change policy`
- `Unit: Action during Friday 18:00 blocked by weekend blackout policy`
- `Unit: Action within 30 minutes of previous action blocked by 60-minute cooldown`
- `Unit: Action reducing replicas to 0 blocked by min_replica_count=1`
- `Unit: Workload-specific policy overrides namespace-wide policy`
- `Unit: No policies defined allows all actions`
- `Unit: RequireApproval=true blocks auto-enforcement, allows manual`

#### 9.2 — Rightsizing Enforcer

**What**: Apply approved rightsizing recommendations to Kubernetes workloads by patching resource requests/limits.

**Design**:

```go
// internal/enforcement/enforcer.go
package enforcement

type Enforcer struct {
    k8s        *collector.K8sClient
    guardrails *GuardrailEngine
    store      store.RecommendationStore
    actionStore store.ActionStore
    notifier   *alert.Notifier
    dryRun     bool
    logger     zerolog.Logger
}

type ProposedAction struct {
    ClusterID    uuid.UUID
    Namespace    string
    WorkloadName string
    WorkloadKind string
    ContainerName string
    ActionType   string   // "cpu_resize", "memory_resize"
    PreviousCPU  float64
    NewCPU       float64
    PreviousMem  int64
    NewMem       int64
}

// Apply executes a rightsizing action on a Kubernetes workload.
// Steps:
// 1. Evaluate guardrails
// 2. Record the action as "in_progress"
// 3. Patch the Deployment/StatefulSet/DaemonSet with new resource requests
// 4. Wait for rollout to complete (with timeout)
// 5. Verify pod health (readiness probes passing)
// 6. Mark action as "success" or "failed"
func (e *Enforcer) Apply(ctx context.Context, rec domain.Recommendation) error

// ApplyAll processes all approved recommendations.
func (e *Enforcer) ApplyAll(ctx context.Context) error
```

Kubernetes patch is a strategic merge patch on the container's resources:
```json
{
    "spec": {
        "template": {
            "spec": {
                "containers": [{
                    "name": "api",
                    "resources": {
                        "requests": { "cpu": "400m", "memory": "512Mi" },
                        "limits": { "cpu": "800m", "memory": "1Gi" }
                    }
                }]
            }
        }
    }
}
```

**Testing**:
- `Unit (mocked K8s): Apply patches the correct Deployment with new resource values`
- `Unit (mocked K8s): Apply in dry-run mode logs the patch but does not execute it`
- `Unit (mocked K8s): Apply blocked by guardrails records action as "rejected"`
- `Unit (mocked K8s): Apply records previous state for rollback`
- `Unit (mocked K8s): Apply with rollout timeout marks action as "failed"`
- `Integration (kind): Apply patches a Deployment and new pods have updated resource requests`
- `Integration (kind): Apply waits for rollout completion before marking success`

#### 9.3 — Automatic Rollback

**What**: Detect degraded workload health after enforcement and automatically rollback to previous resource settings.

**Design**:

```go
// internal/enforcement/rollback.go
package enforcement

type RollbackMonitor struct {
    k8s         *collector.K8sClient
    actionStore store.ActionStore
    notifier    *alert.Notifier
    logger      zerolog.Logger
}

type RollbackConfig struct {
    MonitorDuration   time.Duration // how long to watch after enforcement (default: 10m)
    CheckInterval     time.Duration // how often to check health (default: 30s)
    MaxRestarts       int           // max restarts before rollback (default: 3)
    LatencyThreshold  float64       // max p99 latency increase % before rollback (default: 50)
}

// Monitor watches a workload after enforcement and triggers rollback if health degrades.
// Health signals:
// 1. Pod restart count exceeds threshold
// 2. Readiness probe failures
// 3. OOMKilled events
func (rm *RollbackMonitor) Monitor(ctx context.Context, action domain.OptimisationAction) error

// Rollback restores the previous resource settings.
func (rm *RollbackMonitor) Rollback(ctx context.Context, action domain.OptimisationAction, reason string) error
```

**Testing**:
- `Unit (mocked K8s): OOMKilled event on a pod triggers rollback`
- `Unit (mocked K8s): 3+ restarts within monitoring window triggers rollback`
- `Unit (mocked K8s): Rollback patches workload with previous resource values`
- `Unit (mocked K8s): Rollback sends notification explaining the reason`
- `Unit (mocked K8s): No degradation within monitoring window marks action as confirmed`
- `Integration (kind): Rollback restores previous Deployment resources`

---

## Phase 10: MCP Server and CI/CD Integration

### Purpose
Expose cost data via the Model Context Protocol for AI agent access and provide CI/CD integration for PR-level cost impact previews. After this phase, AI agents can query cost allocation data, and CI pipelines can surface estimated cost impact of resource changes before they reach production.

### Tasks

#### 10.1 — MCP Server

**What**: Implement an MCP server exposing cost allocation tools that AI agents can invoke.

**Design**:

```go
// internal/mcp/server.go
package mcp

import (
    "github.com/mark3labs/mcp-go/server"
)

type CostMCPServer struct {
    allocStore store.AllocationStore
    recStore   store.RecommendationStore
    anomStore  store.AnomalyStore
}

// Tools exposed:
// 1. get_namespace_cost — query cost for a namespace over a time window
// 2. get_top_workloads — get the N most expensive workloads
// 3. get_recommendations — list pending rightsizing recommendations
// 4. get_anomalies — list active cost anomalies
// 5. explain_cost_change — explain why cost changed for a namespace/workload
// 6. get_cluster_efficiency — get CPU and RAM efficiency scores

// Tool: get_namespace_cost
// Input: { "cluster": "prod", "namespace": "default", "days": 7 }
// Output: { "total_cost": 142.50, "cpu_cost": 85.30, "ram_cost": 45.20, ... }

// Tool: explain_cost_change
// Input: { "cluster": "prod", "namespace": "production", "compare_days": 7 }
// Output: { "previous_cost": 120.00, "current_cost": 180.00, "change_pct": 50,
//           "top_contributors": [{"workload": "api-server", "change": +40.00, "reason": "replica increase"}] }
```

**Testing**:
- `Integration: MCP client connects to server and lists available tools`
- `Integration: get_namespace_cost returns correct cost data`
- `Integration: get_recommendations returns pending recommendations`
- `Unit: explain_cost_change computes correct percentage change`
- `Unit: Tool input validation rejects missing required fields`

#### 10.2 — CI/CD Cost Impact Preview

**What**: A CLI command and GitHub Action that estimates the cost impact of resource changes in a PR.

**Design**:

```go
// cmd/cli/cost_preview.go
// kco preview --manifest deployment.yaml --cluster prod

type CostPreview struct {
    ManifestPath string
    ClusterID    string
    OutputFormat string // "text", "json", "markdown"
}

type PreviewResult struct {
    Workload        string
    CurrentCost     float64
    ProjectedCost   float64
    MonthlyCostDelta float64
    Changes         []ResourceChange
}

type ResourceChange struct {
    Container string
    Resource  string // "cpu_request", "memory_request", "replicas"
    Previous  string
    New       string
}
```

The CLI reads the manifest, extracts resource requests/limits and replica counts, queries the API for current workload cost, and projects the new cost based on the resource changes.

GitHub Action:
```yaml
# .github/actions/kco-preview/action.yml
name: KCO Cost Preview
inputs:
  api-url:
    description: KCO API endpoint
    required: true
  cluster-id:
    description: Target cluster ID
    required: true
  manifests:
    description: Glob pattern for Kubernetes manifests
    default: "k8s/*.yaml"
```

Output posted as a PR comment:
```markdown
## Cost Impact Preview

| Workload | Current (monthly) | Projected (monthly) | Change |
|----------|-------------------|---------------------|--------|
| api-server | $142.50 | $98.30 | -$44.20 (-31%) |
| worker | $85.00 | $85.00 | $0.00 (0%) |

**Total monthly impact: -$44.20**
```

**Testing**:
- `Unit: Preview parses Deployment YAML and extracts resource requests`
- `Unit: Preview computes cost delta from current allocation and new requests`
- `Unit: Preview handles multi-container pods`
- `Unit: Markdown output format renders correct table`
- `Integration: CLI reads manifest file and queries API for current cost`
- `E2E: GitHub Action posts cost preview comment on a test PR`

---

## Phase 11: Spot Instance Optimisation

### Purpose
Provide spot instance recommendations and basic interruption handling, enabling safer adoption of spot instances for Kubernetes workloads. After this phase, users receive recommendations for workloads that can safely run on spot instances, and the system handles graceful migration when spot interruptions occur.

### Tasks

#### 11.1 — Spot Instance Advisor

**What**: Analyse workload characteristics to recommend which workloads are suitable for spot instances.

**Design**:

```go
// internal/spot/advisor.go
package spot

type Advisor struct {
    workloadStore store.WorkloadStore
    allocStore    store.AllocationStore
    metricsStore  store.MetricsStore
    pricing       *collector.PricingLookup
}

type SpotRecommendation struct {
    ClusterID        uuid.UUID
    Namespace        string
    WorkloadName     string
    WorkloadKind     string
    Suitability      string  // "high", "medium", "low", "not_recommended"
    Reasons          []string
    EstimatedSavings float64 // monthly
    SuggestedInstanceTypes []string
    InterruptionRisk string  // "low", "medium", "high"
}

// Analyze evaluates all workloads for spot suitability.
// Criteria:
// 1. Stateless (Deployment, not StatefulSet) → higher suitability
// 2. Multiple replicas (can tolerate losing one) → higher suitability
// 3. No PVCs (stateless storage) → higher suitability
// 4. Tolerates restarts (short startup time) → higher suitability
// 5. Not latency-critical (batch, async worker) → higher suitability
func (a *Advisor) Analyze(ctx context.Context, clusterID uuid.UUID) ([]SpotRecommendation, error)
```

**Testing**:
- `Unit: StatefulSet with PVCs rated "not_recommended"`
- `Unit: Deployment with 5 replicas and no PVCs rated "high"`
- `Unit: Deployment with 1 replica rated "low" (cannot tolerate instance loss)`
- `Unit: EstimatedSavings correctly computed from on-demand vs spot price difference`
- `Unit: DaemonSets always rated "not_recommended"`

#### 11.2 — Spot Interruption Handler

**What**: Watch for spot interruption notices and gracefully drain pods before termination.

**Design**:

```go
// internal/spot/interruption_handler.go
package spot

type InterruptionHandler struct {
    k8s      *collector.K8sClient
    notifier *alert.Notifier
    logger   zerolog.Logger
}

// Watch monitors node events and AWS/GCP/Azure interruption metadata endpoints.
// On AWS: polls http://169.254.169.254/latest/meta-data/spot/instance-action
// On GCP: polls http://metadata.google.internal/computeMetadata/v1/instance/preempted
// On Azure: polls http://169.254.169.254/metadata/scheduledevents
func (ih *InterruptionHandler) Watch(ctx context.Context) error

// HandleInterruption cordons the node and gracefully evicts pods.
func (ih *InterruptionHandler) HandleInterruption(ctx context.Context, nodeName string) error
```

**Testing**:
- `Unit (mocked): AWS interruption notice triggers HandleInterruption`
- `Unit (mocked): HandleInterruption cordons the node`
- `Unit (mocked): HandleInterruption evicts pods respecting PodDisruptionBudgets`
- `Unit (mocked): Notification sent when interruption is detected`
- `Integration (kind): HandleInterruption moves pods to a healthy node`

---

## Phase 12: Helm Chart and Production Deployment

### Purpose
Package the entire system as a production-ready Helm chart with configurable values for all components, RBAC, resource limits, and monitoring. After this phase, users can deploy the Kubernetes Cost Optimizer to any Kubernetes cluster with a single `helm install`.

### Tasks

#### 12.1 — Helm Chart Structure

**What**: Create the Helm chart with templates for all Kubernetes resources.

**Design**:

Chart components:
- **server Deployment**: API server + worker
- **agent DaemonSet**: In-cluster metrics collector (runs on every node)
- **PostgreSQL** (optional): embedded PostgreSQL+TimescaleDB via subchart, or external DB URL
- **ServiceAccount + ClusterRole + ClusterRoleBinding**: RBAC for agent
- **ConfigMap**: configuration values
- **Secret**: database credentials, Slack webhook, cloud provider keys
- **Service**: ClusterIP for API, NodePort/Ingress optional
- **Ingress** (optional): for external dashboard access
- **PodDisruptionBudget**: for server deployment
- **ServiceMonitor** (optional): for Prometheus Operator integration

```yaml
# deploy/helm/kco/values.yaml
replicaCount: 1

server:
  image:
    repository: ghcr.io/worlds-biggest-software-project/kco-server
    tag: ""  # defaults to chart appVersion
  resources:
    requests: { cpu: 200m, memory: 256Mi }
    limits: { cpu: 1000m, memory: 512Mi }

agent:
  image:
    repository: ghcr.io/worlds-biggest-software-project/kco-agent
    tag: ""
  scrapeInterval: 60s
  resources:
    requests: { cpu: 50m, memory: 64Mi }
    limits: { cpu: 200m, memory: 128Mi }

database:
  external: false  # set true to use externalURL
  externalURL: ""
  embedded:
    enabled: true
    storage: 10Gi
    storageClass: ""

enforcement:
  enabled: false
  dryRun: true

alerting:
  slack:
    webhookURL: ""
  email:
    host: ""
    port: 587
    from: ""

rbac:
  create: true
  clusterRole:
    rules:
      - apiGroups: [""]
        resources: ["pods", "nodes", "namespaces", "services"]
        verbs: ["get", "list", "watch"]
      - apiGroups: ["apps"]
        resources: ["deployments", "statefulsets", "daemonsets"]
        verbs: ["get", "list", "watch", "patch"]
      - apiGroups: ["autoscaling.k8s.io"]
        resources: ["verticalpodautoscalers"]
        verbs: ["get", "list", "watch"]
      - apiGroups: ["metrics.k8s.io"]
        resources: ["pods", "nodes"]
        verbs: ["get", "list"]
      - apiGroups: ["karpenter.sh"]
        resources: ["nodepools", "nodeclaims"]
        verbs: ["get", "list", "watch"]

serviceMonitor:
  enabled: false
  interval: 30s
```

**Testing**:
- `Unit: helm template renders valid YAML for all templates`
- `Unit: helm template with enforcement.enabled=true adds patch verb to ClusterRole`
- `Unit: helm template with database.external=true omits embedded PostgreSQL resources`
- `Unit: helm lint passes with no errors or warnings`
- `Integration (kind): helm install deploys all components successfully`
- `Integration (kind): Agent pods start and begin collecting metrics`
- `Integration (kind): Server pod starts and /healthz returns 200`

#### 12.2 — CI/CD Pipeline

**What**: GitHub Actions workflow for build, test, lint, Docker image publishing, and Helm chart publishing.

**Design**:

```yaml
# .github/workflows/ci.yml
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: timescale/timescaledb:latest-pg16
        env:
          POSTGRES_DB: kco_test
          POSTGRES_PASSWORD: test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: '1.23' }
      - run: make lint
      - run: make test
      - run: make build

  docker:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}/kco-server:latest

  helm:
    needs: docker
    runs-on: ubuntu-latest
    steps:
      - run: helm package deploy/helm/kco
      - run: helm push kco-*.tgz oci://ghcr.io/${{ github.repository }}/charts
```

**Testing**:
- `CI: Workflow completes successfully on a clean push`
- `CI: Docker images are published to GHCR on main branch push`
- `CI: Helm chart is packaged and published to GHCR OCI registry`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                    ─── required by everything
    │
Phase 2: Resource Collection           ─── requires Phase 1
    │
Phase 3: Cost Allocation Engine        ─── requires Phase 2
    │
    ├── Phase 4: REST API              ─── requires Phase 3
    │       │
    │       ├── Phase 5: Recommendations ─── requires Phase 3 + 4
    │       │
    │       ├── Phase 6: Budgets & Anomalies ─── requires Phase 3 + 4
    │       │       │
    │       │       └── Phase 7: Cloud Billing ─── can parallel with Phase 5, 6
    │       │
    │       └── Phase 8: Web Dashboard ─── requires Phase 4
    │
    ├── Phase 9: Enforcement           ─── requires Phase 5
    │
    ├── Phase 10: MCP + CI/CD          ─── requires Phase 4
    │
    └── Phase 11: Spot Optimisation    ─── requires Phase 2 + 5

Phase 12: Helm & Production           ─── requires all prior phases
```

**Parallelism opportunities:**
- Phases 5, 6, and 7 can be developed concurrently after Phase 4
- Phase 8 can be developed concurrently with Phases 5-7 (frontend team)
- Phase 10 can be developed concurrently with Phases 5-9
- Phase 11 can start as soon as Phase 5 is complete

---

## Definition of Done (per phase)

1. All tasks in the phase are implemented.
2. All unit tests pass (`make test`).
3. All integration tests pass (including testcontainers-based tests).
4. `golangci-lint run` reports no issues.
5. `go vet ./...` reports no issues.
6. Docker images build successfully (`make docker-build`).
7. The server starts and `/healthz` returns 200.
8. Database migrations apply cleanly on a fresh database.
9. New API endpoints appear in the OpenAPI spec and generated types are up to date.
10. New configuration options are documented in `values.yaml` and README.
11. Frontend tests pass (if Phase 8+ touched frontend code).
12. Helm chart renders and lints without errors (`helm lint`).
