# Declarative Self-Deploy + Tier Promotion

PodMaker treats every operational "tier" as a declarative YAML
manifest. The platform deploys itself against the YAML you point it at,
and grows itself by promoting to a different YAML — same command,
zero-data-loss, zero-downtime cutover. No tier name is special:
T1/T2/T3 are just file names. Operators ship new tiers (mesh
multi-region, regulated-compliance, edge-bursty, …) as new YAML files
without touching CLI or workflow code.

```
pdctl tier promote --to tiers/t2-managed-aws.yaml
pdctl tier promote --to ./tiers/custom/my-mesh.yaml --dry-run
pdctl tier rollback
```

The migration engine reads source + target YAML, computes a diff,
materialises an ordered action list, and runs it through a Temporal
workflow that has compensation for every step. Same workflow handles
the trivial case (scale `agent-gateway` from 2 to 3 replicas) and the
nontrivial case (migrate Postgres from a container to RDS Aurora
Global across two regions).

---

## 1. Design goals

1. **Declarative**: every tier is a YAML manifest. Hand-written or
   templated.
2. **Extensible**: new operational shapes (mesh, edge, regulated)
   are new YAML files + new entries in the action registry. CLI +
   workflow stay frozen.
3. **Zero-downtime**: cutover windows ≤ 60 s end-to-end, measured by
   panel maintenance flag wallclock.
4. **Zero-data-loss**: every state migration uses logical replication
   or snapshot+replay with watermark verification before cutover.
5. **Reversible**: every action carries a compensation. Within the
   cooldown window (default 72 h, configurable) `pdctl tier rollback`
   walks the action list in reverse.
6. **Cost-bounded**: each tier YAML declares `budget_cap_usd_month`;
   the pre-flight aborts if projected spend > cap.
7. **Auditable**: every promotion writes a structured trail to
   `self-deploy.runs` so the admin UI can show diff, actions,
   timings, downtime.

Anti-goals:
- Auto-promotion. Promotion is always an explicit operator action.
- Cross-cloud lift in a single promotion. A region+provider swap is
  two promotions back-to-back.

---

## 2. Tier YAML schema

`tiers/_schema.json` is the JSON Schema authority. The shape:

```yaml
apiVersion: podmaker.sh/v1
kind: Tier
metadata:
  name: <unique-slug>                   # required, kebab-case
  description: <human prose>            # required
  budget_cap_usd_month: <number>        # required, refuse to apply if projected > cap
  labels: { key: value }                # optional, free-form
spec:
  # Compute fleet: where stateless services run.
  compute:
    kind: <compute-kind>                # ec2-single | ec2-asg | gce-mig |
                                        #   droplet-pool | hcloud-single |
                                        #   k8s | mesh
    # ... kind-specific fields below

  # State backends: where stateful data lives. One entry per logical
  # store. Each store has a `kind:` that maps to a known action in the
  # registry; unknown kinds refuse to apply.
  state:
    postgres: { kind: <store-kind>, ... }
    redis:    { kind: <store-kind>, ... }
    secrets:  { kind: <store-kind>, ... }
    registry: { kind: <store-kind>, ... }
    step_ca:  { kind: container | ... }
    temporal: { kind: container | ... }
    nats:     { kind: container, leaves: [<region>...] }

  # Public edge: where traffic enters. Caddy is always the in-cluster
  # reverse proxy; this section describes what fronts Caddy.
  edge:
    kind: <edge-kind>                   # caddy-direct | aws-alb | gcp-glb |
                                        #   cloudflare-load-balancer
    # kind-specific fields

  # Stateless service replica counts. Override the defaults in
  # infra/topology/ops.yaml. Operators tune per-service horizontal
  # scale here. Increasing `agent-gateway` from 2 to 3 is a single
  # promotion that produces only a `ScaleService` action — no state
  # migration.
  scale:
    control-plane:    <n>
    orchestrator:     <n>
    agent-gateway:    <n>
    cloud-broker:     <n>
    vault-broker:     <n>
    topology-planner: <n>
    build-service:    <n>
    repo-scanner:     <n>
    db-controller:    <n>
    lb-controller:    <n>
    mesh-controller:  <n>
    cache-controller: <n>
    event-bridge:     <n>
    metrics-consumer: <n>

  # Promotion-specific knobs. Optional; sensible defaults in
  # apps/control-plane/config/self_deploy.php.
  promotion:
    maintenance_window_target_sec: 30
    canary_seconds: 300
    canary_error_rate_max: 0.02
    cooldown_hours: 72
```

### 2.1 Compute kinds

```yaml
# Single virtual machine on a cloud provider.
compute:
  kind: ec2-single
  provider: aws
  region: eu-central-1
  instance_type: t4g.xlarge
  disk_gib: 80

# Auto-scaling group of identical hosts behind an ALB/LB.
compute:
  kind: ec2-asg
  provider: aws
  region: eu-central-1
  instance_type: t4g.large
  disk_gib: 60
  desired: 3
  min: 2
  max: 5
  multi_az: true

# Kubernetes cluster (managed or self-managed).
compute:
  kind: k8s
  provider: aws                         # eks | gke | aks | k3s
  cluster: eks-prod-1
  namespace: podmaker
  kubeconfig: ~/.kube/eks-prod          # path on operator workstation
  ingress_class: alb
  storage_class: gp3
  external_secrets: true                # bind Secrets Manager via ESO

# Mesh of regional fleets connected by an encrypted overlay.
compute:
  kind: mesh
  overlay: wireguard                    # tailscale | wireguard | netbird
  regions:
    - region: eu-central-1
      provider: aws
      instance_type: t4g.large
      desired: 3
    - region: us-east-1
      provider: aws
      instance_type: t4g.large
      desired: 3
    - region: ap-southeast-2
      provider: gcp
      instance_type: e2-standard-2
      desired: 2
  leader_region: eu-central-1           # singleton services run here
```

### 2.2 State kinds

```yaml
# Container — runs inside the compute fleet, volume-backed.
state:
  postgres:
    kind: container
    volume_gib: 40
    backup:
      schedule: '0 2 * * *'
      retention_days: 30
      destination: local                # local | s3 | gcs | spaces

# AWS RDS — managed single-AZ or Multi-AZ.
state:
  postgres:
    kind: aws-rds
    engine: postgres
    instance_class: db.t4g.small
    storage_gib: 100
    multi_az: false
    backup_retention_days: 7

# AWS RDS Aurora Global — cross-region active-active.
state:
  postgres:
    kind: aws-rds-aurora-global
    primary_region: eu-central-1
    replica_regions: [us-east-1]
    instance_class: db.r6g.large

# GCP Cloud SQL.
state:
  postgres:
    kind: gcp-cloud-sql
    tier: db-custom-2-7680
    region: europe-west3
    ha: ZONAL                           # ZONAL | REGIONAL

# DigitalOcean Managed DB.
state:
  postgres:
    kind: do-managed-db
    size: db-s-1vcpu-1gb
    region: fra1

# Redis variants.
state:
  redis:
    kind: container                     # | aws-elasticache | gcp-memorystore | do-managed-redis
    node_type: cache.t4g.micro          # aws only
    cluster_mode: disabled              # disabled | enabled
    multi_az: false

# Secrets variants.
state:
  secrets:
    kind: container                     # | aws-secrets-manager | gcp-secret-manager
    # container kind keeps openbao-meta container running.

# Registry variants.
state:
  registry:
    kind: container                     # | aws-ecr | gcp-artifact-registry
    region: eu-central-1                # managed kinds only
```

### 2.3 Edge kinds

```yaml
edge:
  kind: caddy-direct                    # Caddy binds to host 80/443

edge:
  kind: aws-alb
  https: true
  waf: false

edge:
  kind: gcp-glb

edge:
  kind: cloudflare-load-balancer
  geo_routing: true
  zone_id: <cf-zone>
```

---

## 3. Pre-canned tiers

Shipped under `tiers/` in this repo. Operators may copy + tweak under
`tiers/custom/` (gitignored by convention).

```
tiers/
├── _schema.json                        # JSON Schema (pdctl tier validate)
├── t1-single-host-aws.yaml
├── t1-single-host-gcp.yaml
├── t1-single-host-do.yaml
├── t1-single-host-hetzner.yaml         # all-container, Hetzner stays here
├── t2-managed-aws.yaml
├── t2-managed-gcp.yaml
├── t2-managed-do.yaml
├── t3-multi-host-aws.yaml
├── t3-multi-host-gcp.yaml
├── t3-k8s-eks.yaml
├── t3-k8s-gke.yaml
├── t3-k8s-doks.yaml
├── t4-mesh-multi-region.yaml           # example future tier
└── custom/                             # operator-defined
```

### 3.1 Example — `t1-single-host-aws.yaml`

```yaml
apiVersion: podmaker.sh/v1
kind: Tier
metadata:
  name: t1-single-host-aws
  description: All-container SaaS stack on a single EC2 host
  budget_cap_usd_month: 200
spec:
  compute:
    kind: ec2-single
    provider: aws
    region: eu-central-1
    instance_type: t4g.xlarge
    disk_gib: 80
  state:
    postgres: { kind: container, volume_gib: 40 }
    redis:    { kind: container, volume_gib: 5 }
    secrets:  { kind: container }
    registry: { kind: container }
    step_ca:  { kind: container }
    temporal: { kind: container }
    nats:     { kind: container, leaves: [eu-central-1] }
  edge:
    kind: caddy-direct
  scale:
    control-plane: 1
    orchestrator: 1
    agent-gateway: 1
    cloud-broker: 1
    vault-broker: 1
    topology-planner: 1
    build-service: 1
    repo-scanner: 1
    db-controller: 1
    lb-controller: 1
    mesh-controller: 1
    cache-controller: 1
    event-bridge: 1
    metrics-consumer: 1
```

### 3.2 Example — `t2-managed-aws.yaml`

```yaml
apiVersion: podmaker.sh/v1
kind: Tier
metadata:
  name: t2-managed-aws
  description: AWS managed state + 3 stateless hosts, single region
  budget_cap_usd_month: 400
spec:
  compute:
    kind: ec2-asg
    provider: aws
    region: eu-central-1
    instance_type: t4g.large
    disk_gib: 60
    desired: 3
    min: 2
    max: 5
    multi_az: true
  state:
    postgres:
      kind: aws-rds
      instance_class: db.t4g.small
      storage_gib: 100
      multi_az: false
    redis:
      kind: aws-elasticache
      node_type: cache.t4g.micro
      cluster_mode: disabled
    secrets: { kind: aws-secrets-manager }
    registry:
      kind: aws-ecr
      region: eu-central-1
    step_ca:  { kind: container, placement: singleton }
    temporal: { kind: container }
    nats:     { kind: container, leaves: [eu-central-1] }
  edge:
    kind: aws-alb
  scale:
    control-plane: 3
    orchestrator: 2
    agent-gateway: 2
    cloud-broker: 2
    vault-broker: 2
    topology-planner: 1
    build-service: 2
    repo-scanner: 1
    db-controller: 1
    lb-controller: 1
    mesh-controller: 1
    cache-controller: 1
    event-bridge: 1
    metrics-consumer: 1
```

### 3.3 Example — `t4-mesh-multi-region.yaml` (future)

Demonstrates extensibility. New compute.kind `mesh`, new state.kind
`aws-rds-aurora-global` and `aws-ecr-replicated`, plus
`agent-gateway` scaled to 9 (3 per region × 3 regions).

```yaml
apiVersion: podmaker.sh/v1
kind: Tier
metadata:
  name: t4-mesh-multi-region
  description: WireGuard-meshed regional fleets, geo-routed edge
  budget_cap_usd_month: 2000
spec:
  compute:
    kind: mesh
    overlay: wireguard
    leader_region: eu-central-1
    regions:
      - { region: eu-central-1,   provider: aws, instance_type: t4g.large, desired: 3 }
      - { region: us-east-1,      provider: aws, instance_type: t4g.large, desired: 3 }
      - { region: ap-southeast-2, provider: gcp, instance_type: e2-standard-2, desired: 2 }
  state:
    postgres:
      kind: aws-rds-aurora-global
      primary_region: eu-central-1
      replica_regions: [us-east-1]
      instance_class: db.r6g.large
    redis:
      kind: aws-elasticache-global
      regions: [eu-central-1, us-east-1]
    secrets:
      kind: aws-secrets-manager
      replicas: [eu-central-1, us-east-1]
    registry:
      kind: aws-ecr-replicated
      regions: [eu-central-1, us-east-1, ap-southeast-2]
    step_ca:  { kind: container, placement: leader }
    temporal: { kind: container, placement: leader }
    nats:     { kind: container, leaves: [eu-central-1, us-east-1, ap-southeast-2] }
  edge:
    kind: cloudflare-load-balancer
    geo_routing: true
    zone_id: <cf-zone>
  scale:
    control-plane: 6
    orchestrator: 3
    agent-gateway: 9                    # 3 per region × 3 regions
    cloud-broker: 3
    vault-broker: 3
    topology-planner: 1
    build-service: 3
    repo-scanner: 2
    db-controller: 2
    lb-controller: 2
    mesh-controller: 3
    cache-controller: 2
    event-bridge: 2
    metrics-consumer: 3
```

---

## 4. Promotion engine

```
+-------------------+    +---------------+    +---------------------+
| current.yaml      |    | target.yaml   |    | action registry     |
| (stored on host   |--->|               |--->| (Go map: kind →     |
|  /var/podmaker/   |    |               |    |  Action interface)  |
|  tier-current.yml)|    +---------------+    +---------------------+
+-------------------+            |
        |                        v
        |              +----------------------+
        +------------->|     Diff Engine      |
                       | yields Action[] in   |
                       | safe execution order |
                       +----------+-----------+
                                  |
                                  v
                       +----------------------+
                       | Temporal Workflow    |
                       | ApplyTierPlan        |
                       | - per-action activity|
                       | - compensation       |
                       | - watermark waits    |
                       +----------+-----------+
                                  |
                                  v
                       +----------------------+
                       | self-deploy.runs row |
                       | timeline, downtime,  |
                       | bill check, status   |
                       +----------------------+
```

### 4.1 Diff order rules

Actions are ordered to minimise the cutover window:

1. **Provision** new infrastructure first (terraform apply target=…).
   No traffic moves. Cost begins.
2. **Replicate state** from current → new (start logical replication
   subscriptions, RESTORE seeds, etc.).
3. **Watermark wait** until every replicated store shows lag ≤ ε.
4. **Enter maintenance** mode briefly.
5. **Final flush** + **swap connection strings** (refresh service env
   files / Secrets Manager values referenced by stateless services).
6. **Rolling restart** stateless services in dependency order.
7. **Swap traffic** at the edge (ALB target group register/deregister,
   Route53 weighted swap, CF LB origin swap).
8. **Smoke test** (every /healthz, tenant probe, bill check).
9. **Resume traffic**, clear maintenance mode.
10. **Canary watch** for `canary_seconds`.
11. **Cooldown** of `cooldown_hours`. Old infra stays warm.
12. **Decommission** old infra after cooldown.

If the source and target differ only in `spec.scale`, the diff
collapses to a sequence of `ScaleService` actions and skips steps
1-3 + 5 + 7. Promotion completes in seconds with no downtime.

If the source and target differ only in `spec.state.postgres.kind`,
steps 1-5 are run for Postgres alone. Other stores untouched.

---

## 5. Action registry

Every action implements:

```go
type Action interface {
    Plan(ctx context.Context, src, dst TierSpec, prev []Action) ([]ActionStep, error)
    Apply(ctx context.Context, step ActionStep) error
    Compensate(ctx context.Context, step ActionStep) error
    EstimateDowntimeSec(step ActionStep) int
    EstimateCostDeltaUSD(step ActionStep) float64
}
```

Actions live under `apps/orchestrator/internal/selfdeploy/actions/`
and self-register at init time. The registry is the only place new
operational shapes touch CLI/workflow code.

### 5.1 Starter set (P1 + P2)

| Action | Triggered by diff of … | Compensation |
|---|---|---|
| `ProvisionRDS` | state.postgres.kind: container → aws-rds | terraform destroy of the RDS module |
| `ProvisionElastiCache` | state.redis.kind: container → aws-elasticache | terraform destroy |
| `ProvisionSecretsManager` | state.secrets.kind: container → aws-secrets-manager | delete secrets with recovery window |
| `ProvisionECR` | state.registry.kind: container → aws-ecr | delete repos with retained images |
| `ProvisionALB` | edge.kind: caddy-direct → aws-alb | terraform destroy |
| `ProvisionASG` | compute.kind: ec2-single → ec2-asg | scale to 0 + destroy |
| `MigratePostgres` | state.postgres.kind change | drop logical replication slot, swap DSN back |
| `MigrateRedis` | state.redis.kind change | stop REPLICAOF, swap URL back |
| `MigrateOpenBaoMeta` | state.secrets.kind change | re-push secrets to container vault |
| `MigrateRegistry` | state.registry.kind change | re-tag images at the old registry URL |
| `ScaleService` | spec.scale[name] number change | scale to previous N |
| `SwapTraffic` | edge.kind change | re-attach previous edge |
| `RestartStatelessFleet` | implicit after env file change | re-restart with prior env |
| `EnterMaintenanceMode` | always, before final flush | exit maintenance |
| `ExitMaintenanceMode` | always, after smoke test | re-enter maintenance |
| `WaitForReplicationWatermark` | always after Migrate* | stop watermark wait, declare failure |
| `Decommission` | always at cooldown end | not compensatable (cooldown expired) |

### 5.2 Future actions (P3 — mesh + multi-region)

| Action | Triggered by | Compensation |
|---|---|---|
| `ProvisionMeshOverlay` | compute.kind: ec2-asg → mesh | tear down WireGuard config |
| `JoinRegion` | regions[] grew | remove region, drain traffic first |
| `PromoteAuroraGlobalSecondary` | state.postgres.kind: aws-rds → aws-rds-aurora-global | demote secondary, keep single-region |
| `ConfigureCloudflareLB` | edge.kind: aws-alb → cloudflare-load-balancer | restore ALB DNS |
| `EnableECRReplication` | state.registry.kind: aws-ecr → aws-ecr-replicated | disable replication, retain images in source |
| `ScaleAgentGatewayPerRegion` | spec.scale.agent-gateway > 1 across mesh regions | scale back per-region |

Adding a new action is a small PR: terraform module + Go file + entry
in the kind→action map.

---

## 6. Workflow execution

`apps/orchestrator/internal/selfdeploy/workflow.go` defines
`ApplyTierPlan`:

```go
func ApplyTierPlan(ctx workflow.Context, in ApplyTierPlanInput) (ApplyTierPlanOutput, error) {
    plan, err := DiffTiers(ctx, in.Source, in.Target)
    if err != nil { return out, err }

    if err := PreFlight(ctx, in, plan); err != nil { return out, err }

    var applied []ActionStep
    for _, step := range plan.Steps {
        if err := workflow.ExecuteActivity(ctx, "Apply"+step.Kind, step).Get(ctx, nil); err != nil {
            // Compensate in reverse order
            for i := len(applied)-1; i >= 0; i-- {
                _ = workflow.ExecuteActivity(ctx, "Compensate"+applied[i].Kind, applied[i]).Get(ctx, nil)
            }
            return out, err
        }
        applied = append(applied, step)
        workflow.GetSignalChannel(ctx, "abort").ReceiveAsync(&abort)
        if abort { /* same compensation loop */ break }
    }

    return PostFlight(ctx, plan, applied)
}
```

Signals: `abort` (operator-initiated cancel), `rollback` (post-success
within cooldown).

Heartbeats on long activities (replication wait, terraform apply) keep
the workflow alive on a slow run.

---

## 7. CLI shape

```
pdctl tier list                            # list pre-canned + custom tiers
pdctl tier show <name>                     # render the YAML + cost estimate
pdctl tier current                         # current tier
pdctl tier validate <path-or-name>         # JSON Schema check
pdctl tier diff <path-or-name>             # action list vs current
pdctl tier promote --to <path-or-name>     # apply
pdctl tier promote --to <path-or-name> --dry-run
pdctl tier rollback                        # within cooldown
pdctl tier history                         # past promotions
pdctl tier cancel <run-id>                 # signal abort to a running promotion
```

`<path-or-name>` resolves: if the argument matches a file under
`tiers/` or `tiers/custom/`, use it. Otherwise resolve as `tiers/<arg>.yaml`.

---

## 8. Pre-flight checks

Run before any action. Failure aborts with no state change.

1. **Budget**: estimated monthly bill ≤ `budget_cap_usd_month`.
2. **Provider credentials**: every provider referenced in target must
   have a usable Vault Connection in the panel.
3. **Quota**: provider quota for the resources we will create (EC2
   limit, RDS limit, EIP limit, …).
4. **DNS control**: confirm the zone for the target apex is delegated
   to a DNS provider we hold credentials for.
5. **Compatibility**: target tier YAML must validate against the
   schema; every `kind:` must be in the action registry.
6. **State versions**: source PG version + target managed PG version
   must be compatible with the chosen replication method.
7. **Cooldown clear**: no other promotion is in cooldown for this
   workspace.

---

## 9. Cutover semantics

Wallclock target: ≤ 60 s end-to-end maintenance window.

```
T0   maintenance flag set
T0+1 final logical replication flush
T0+2 swap DSN/URL/Secrets Mgr env values
T0+3 rolling restart stateless services (parallel, capped at 4 at a time)
T0+4 swap ALB / Route53 / CF LB
T0+30 smoke test sweep
T0+45 maintenance flag clear
T0+60 canary watch begins
```

Beyond 60 s the workflow auto-aborts with full rollback unless the
operator explicitly extended the window in the tier spec
(`promotion.maintenance_window_target_sec`).

---

## 10. Failure semantics

| Failure | Behaviour |
|---|---|
| Pre-flight fails | Workflow exits, no state changed |
| Provision* fails | terraform destroy of partial apply |
| Replication never reaches watermark | Drop subs, no DSN swap |
| Maintenance flag transition fails | Auto-revert |
| Final flush fails | Re-enter pre-maintenance state |
| Service rolling restart: 1 service fails | Compensate that one; if cascade detected, full rollback |
| Smoke test fails | Full rollback (Route53 + ALB + maintenance) |
| Canary 5xx breach within `canary_seconds` | Auto-rollback |
| Operator sends `abort` signal | Full rollback (cancels running activity, compensates in reverse) |
| Cooldown expires + `rollback` requested | "Rollback not available — start a reverse promotion" |

---

## 11. Cost by pre-canned tier (USD / month, AWS eu-central-1)

```
                                    T1     T2     T3     T3-k8s  T4-mesh
EC2 / EKS / mesh nodes              ~$110  ~$165  ~$165  ~$235   ~$500
EBS                                 ~$10   ~$18   ~$18   ~$15    ~$50
RDS Postgres                        -      ~$30   ~$60   ~$60    ~$400 (Aurora Global)
ElastiCache                         -      ~$15   ~$25   ~$25    ~$80
Secrets Manager                     -      ~$12   ~$12   ~$15    ~$25
ECR (+ replication)                 -      ~$10   ~$10   ~$10    ~$25
ALB / CF LB                         -      ~$22   ~$22   ~$22    ~$30
NAT Gateway                         -      -      -      ~$35    ~$105
Route53                             $0.50  $0.50  $0.50  $0.50   $0.50
Data egress                         ~$5    ~$10   ~$10   ~$10    ~$80
──────────────────────────────────────────────────────────────────────────
Total                               ~$130  ~$285  ~$325  ~$430   ~$1300
```

Promotion to a more expensive tier requires `metadata.budget_cap_usd_month`
≥ the projected total, which the pre-flight estimator computes from the
tier YAML + the provider's current price list.

---

## 12. Implementation phases

### P1 — T1 first install (current sprint)

Goal: `pdctl bootstrap --provider aws --domain X --email Y` brings up
`t1-single-host-aws.yaml` on a fresh EC2.

Deliverables:
- `tiers/_schema.json` JSON Schema.
- `tiers/t1-single-host-aws.yaml`,
  `tiers/t1-single-host-gcp.yaml`,
  `tiers/t1-single-host-do.yaml`,
  `tiers/t1-single-host-hetzner.yaml`.
- `install-podmaker.sh` reads the tier YAML, writes
  `/var/podmaker/tier-current.yaml`, brings up the prod compose.
- 9 missing env templates under `deploy/bootstrap-templates/`.
- step-ca fingerprint extraction already exists; rewire to read
  service names from the tier YAML.
- `pdctl bootstrap` accepts `--tier` (default: `t1-single-host-<provider>`).
- Doc: `docs/release/self-deploy.md` rewritten to point here.

### P2 — Promotion engine + T2 on AWS

Goal: `pdctl tier promote --to t2-managed-aws` works end-to-end.

Deliverables:
- `apps/orchestrator/internal/selfdeploy/diff/` diff engine.
- `apps/orchestrator/internal/selfdeploy/actions/` action interface +
  starter set per §5.1.
- `infra/terraform/aws/t2/` modules per action.
- Temporal workflow `ApplyTierPlan` + signal handling.
- `pdctl tier <list|show|current|validate|diff|promote|rollback|history|cancel>`.
- `tiers/t2-managed-aws.yaml`.
- pglogical / DMS Postgres replication wrapper.
- Redis REPLICAOF wrapper.
- OpenBao-meta → Secrets Manager export.
- skopeo sync wrapper for zot → ECR.
- Sandbox AWS account validation: real bill within ±10% of §11.

### P3 — T3 + T3-k8s + GCP + DO

Deliverables:
- `tiers/t3-multi-host-aws.yaml`, `tiers/t3-multi-host-gcp.yaml`,
  `tiers/t3-multi-host-do.yaml`.
- `tiers/t3-k8s-eks.yaml`, `tiers/t3-k8s-gke.yaml`,
  `tiers/t3-k8s-doks.yaml`.
- Actions: `ProvisionASG`, `ProvisionK8sNamespace`, ALB target
  flip, ASG-to-Ingress traffic swap.
- `infra/k8s/base/` Kustomize + `infra/k8s/overlays/{aws,gcp,do}/`.
- Finish `apps/orchestrator/internal/selfdeploy/k8s_executor.go`.
- ESO wiring for k8s overlays.

### P4 — Mesh + multi-region

Deliverables:
- `tiers/t4-mesh-multi-region.yaml`.
- Actions: `ProvisionMeshOverlay`, `JoinRegion`,
  `PromoteAuroraGlobalSecondary`, `ConfigureCloudflareLB`,
  `EnableECRReplication`, `ScaleAgentGatewayPerRegion`.
- Terraform modules for mesh overlay (WireGuard / Tailscale / NetBird).
- Geo-DNS Cloudflare LB config.

### P5 — Operator-defined custom tiers

Goal: operators write `tiers/custom/foo.yaml` and promote without
PR to PodMaker.

Deliverables:
- `pdctl tier lint <path>` validates against schema + checks
  action registry contains every kind referenced.
- Admin UI tier editor + cost estimator.
- Custom action SDK so operators can ship action plugins.

---

## 13. Acceptance criteria

### P1
- `pdctl bootstrap --provider aws --domain X --email Y --region R`
  brings up `tiers/t1-single-host-aws.yaml` on one EC2.
- All 20 services reach /healthz within 15 minutes of command start.
- `pdctl tier current` returns `t1-single-host-aws`.
- An agent enrolled from a second VM completes the enrollment loop.

### P2
- `pdctl tier promote --to t2-managed-aws --dry-run` prints the action
  list + cost delta + downtime estimate + budget check, exits 0.
- The real run leaves a healthy T2 with bill within ±10% of §11.
- Cutover wallclock ≤ 60 s end-to-end.
- A row written 5 s before cutover is readable on the new RDS.
- `pdctl tier rollback` within cooldown returns to T1 with no data
  loss in the reverse direction.

### P3 / P4 / P5 — defined when each phase starts.

---

## 14. Open design questions

1. **Cooldown default**: 24 h vs 72 h. 72 h doubles infra cost for
   that window but gives weekend headroom. Recommend 72 h, document
   the cost.
2. **step-ca singleton through every tier**: agent fleet pins to one
   fingerprint, so a moved CA forces re-enrollment. Decision: keep
   single-replica step-ca on the leader host through T3; T4+ revisit.
3. **State migration on rollback**: if an operator rolls back from T2
   after writing to RDS for 30 minutes, those 30 minutes have to
   replicate back to the container Postgres. Same machinery, opposite
   direction. Verify the replication-engine wrapper handles reverse.
4. **Budget cap source of truth**: per-workspace setting + per-promote
   override. Default to workspace.
5. **Pre-cutover EBS snapshot**: cheap, always-on safety. Recommend
   `ProvisionEBSSnapshot` action runs before `EnterMaintenanceMode`
   unconditionally.
6. **Cross-cloud migration**: AWS → GCP needs both providers
   credentialed simultaneously. Promote-engine should be able to read
   credentials for two providers per single workflow. Out of P1-3,
   land in P4+.
7. **Tier inheritance**: `extends: t2-managed-aws` so a custom tier
   only overrides the diff. Out of P1, land in P2 or P5.

---

## 15. Related docs

- `docs/release/self-deploy.md` — operator-facing quick start (will
  point here for depth).
- `docs/release/release-flow.md` — release tagging + SHA propagation.
- `infra/topology/ops.yaml` — service topology consumed by tier `spec.scale`.
- `apps/orchestrator/internal/selfdeploy/` — engine + action registry
  home.
- `apps/control-plane/config/self_deploy.php` — canary / cooldown /
  budget defaults.

---

## 16. Closeout — Hardening pass (2026-06-02)

After the initial tier ladder shipped, an end-to-end coverage audit
surfaced four blind spots in the diff engine, one missing action
registration, multiple missing tier YAMLs, and one runtime gap. All
four are closed in this pass; the runtime gap stays open as the next
sprint's headline (`§17`).

### 16.1 Attribute-blind diff (fixed)

`diffStore` only emitted steps when `Kind` changed. T2→T3 same-cloud
promotes (RDS instance class bump, ElastiCache cluster mode flip,
EBS storage grow, multi-AZ toggle, backup retention bump) compiled to
0 state steps — the promotion looked like "scale only" and skipped
the cutover wrap.

Fix: `diffStoreAttributes` (Postgres / Redis / Secrets / Registry /
each store) and `diffComputeAttributes` emit per-field steps. New
action kinds (registered as stateless stubs with realistic wallclock +
downtime so projections are honest):

| Step kind | Source |
|---|---|
| `ResizeStoreInstance` | RDS / Cloud SQL instance class change |
| `ResizeStoreNode` | ElastiCache / Memorystore node type |
| `ResizeStoreStorage` | RDS / Cloud SQL storage GiB |
| `ResizeContainerVolume` | Host docker-compose volume |
| `ToggleStoreMultiAz` | RDS Multi-AZ on/off |
| `ToggleStoreClusterMode` | ElastiCache cluster mode flip |
| `BumpStoreBackupRetention` | RDS / Cloud SQL retention bump |
| `ResizeStoreManaged` | DO Managed DB / Redis size swap |
| `BumpStoreTier` | DOCR / Cloud SQL tier swap (basic→pro) |
| `ResizeComputeInstance` | EC2 / GCE / droplet size bump |
| `ResizeComputeDisk` | EBS / PD / Cloud Volume online grow |
| `ToggleComputeMultiAz` | ASG AZ list change |
| `ResizeComputePool` | desired / min / max retag |

These do NOT carry a Migrate prefix, so `wrapCutover` correctly leaves
them online — the operator sees an attribute change as a non-disruptive
plan.

### 16.2 Region / Provider drift (fixed)

`Diff` now emits a `MigrateRegion` step whenever `Compute.Provider` or
`Compute.Region` differ between source and target, threading
source/target provider+region into the action Params. Per-store
region drift (`StoreSpec.Region` / `StoreSpec.PrimaryRegion`) triggers
the full Provision+Migrate+Wait ladder for that store, even when the
kind stayed the same.

### 16.3 `ProvisionContainer` registration (fixed)

`pascalCase("Provision", "container") == "ProvisionContainer"` was the
action lookup the diff engine emitted for a downgrade
(T2→T1, *→container). It wasn't registered → the workflow walker
panicked at the registry lookup. Registered with a 30 s wallclock /
$0 cost-month delta — the real cost saving comes from the
managed-bill teardown that the matching `Decommission` step queues.

### 16.4 Tier YAML coverage (filled)

| Added | Note |
|---|---|
| `tiers/t2-managed-hetzner.yaml` | Hetzner has no managed state — this is an honest "bigger CCX + Hetzner LB" tier. |
| `tiers/t3-multi-host-hetzner.yaml` | Horizontal scale on Hetzner — 3 CCX nodes behind a Hetzner LB. |
| `tiers/t3-k8s-aws.yaml` | EKS variant of T3; reuses `infra/terraform/aws/t3/` for managed state. |
| `tiers/t3-k8s-gcp.yaml` | GKE variant. |
| `tiers/t3-k8s-do.yaml` | DOKS variant. |
| `tiers/t4-mesh-aws.yaml` | Aurora Global + ElastiCache Global + VPC peering across 2 regions. |
| `tiers/t4-mesh-gcp.yaml` | Cloud SQL HA + Memorystore + Artifact Registry cross-region. |
| `tiers/t4-mesh-multi-cloud.yaml` | AWS + GCP active/active with the state pinned to AWS. |
| Edge `do-lb`, `hetzner-lb` | Added to schema + cost estimator; T2/T3 DO tiers swapped from `caddy-direct`. |
| State `do-managed-registry` | DigitalOcean Container Registry. `infra/terraform/do/t2/registry.tf` + `ProvisionDoManagedRegistry` action. |

### 16.5 Pre-flight validation in the CP (added)

`/v1/self-deploy/preflight` is a new orchestrator endpoint that takes
the same payload as `/promote` but only runs the diff + checks every
plan step Kind is registered + checks the cost delta against the
budget cap. Returns `200 OK` with the resolved counts or
`422 Unprocessable Entity` with `missing_kinds` / `reason`.

`TierPromoteController::promote` calls preflight first; a 422
short-circuits the dispatch and writes the `rejected_preflight`
status to the `TierPromotionRun` row so the operator sees why on the
Filament dashboard. The same check runs server-side on `/promote` as
a backstop in case the CP skips preflight.

### 16.6 Test coverage delta

`packages/shared-go/pkg/tier/diff_test.go` gained:
- `TestDiff_ComputeRegionChange_EmitsMigrateRegion`
- `TestDiff_ComputeProviderChange_EmitsMigrateRegion`
- `TestDiff_StoreRegionChange_TriggersMigration`
- `TestDiff_StoreAttributeChanges_EmitResizeSteps`
- `TestDiff_ComputeAttributeChanges_EmitResizeSteps`

`apps/orchestrator/internal/api/server_test.go` (new):
- `TestPreflight_OK`
- `TestPreflight_MissingKind`
- `TestPreflight_BadJSON`

### 16.7 Cost estimator provider-awareness (fixed)

`Estimate` was AWS-only for `compute.kind: k8s` and `compute.kind: mesh`
— GKE / DOKS / multi-cloud tiers reported nonsense (e.g. an "EKS
control plane" line on a `t3-k8s-do.yaml`). Fixed: `k8s` branches on
`provider` (eks/gke/doks) and `mesh` branches per `regions[*].provider`.
`hcloud-single` with `desired > 1` now reports `Hetzner pool` instead
of a single-node SKU.

---

## 17. Open after closeout (next sprint)

### 17.1 Terraform runner (P3a)

`provision_aws.go` actions all log-only Apply (`slog.Info`). A concrete
runner — shell out to `terraform apply -auto-approve` with the module
path resolved from `action.Kind` (e.g. `ProvisionAwsRds` → `infra/
terraform/aws/t2/rds.tf` as part of the T2 module) and `-var` flags
from `action.Params` — turns the entire diff into a real provisioning
workflow. Reuse the `execRunner` seam in `actions/exec_runner.go` so
tests stay sealed.

### 17.2 Concrete cutover actions (P3b)

`SwapTraffic`, `RestartStatelessFleet`, `RefreshSecretsAtCutover`,
`SmokeTest` (partly done), `Decommission`, `MigrateRegion`,
`MigrateTemporal`/`Nats`/`StepCa`/`Openbao`, every Resize* / Toggle* /
Bump* — currently emit + project but don't move bytes. P3b lands the
cloud-API calls per action. Each takes ≤ 1 file.

### 17.3 Kubernetes executor (P3c)

`tiers/t3-k8s-*.yaml` need an `ApplyTierPlan` branch that drives
`helm upgrade --install podmaker deploy/helm/podmaker-platform`
instead of the docker-compose / cloud-init flow. The orchestrator
already has the Helm chart wired (Sprint 4.6 scaffold); P3c finishes
the wiring.

### 17.4 Aurora Global migration shape (P3d)

`MigratePostgres`'s pglogical implementation can't seed an Aurora
Global secondary — Global Datastore replication is built-in and uses
its own bootstrap (`source_region`). Detect the destination kind in
`MigratePostgres` Apply and route to either pglogical or native Aurora
Global seeding.

### 17.5 Multi-cloud state seeding (P4)

`tiers/t4-mesh-multi-cloud.yaml` ships with the GCP leg reading off
the Aurora Global secondary — the cross-cloud transit is a research
problem the manifest defers. Track separately under the multi-region
work plan.
- `tiers/_schema.json` — JSON Schema (lands in P1).
