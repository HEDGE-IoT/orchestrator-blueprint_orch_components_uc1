# UC1 Energy Services Offloading

Computational orchestration for Energy Services on a KubeEdge cluster. Two components:

- **Scheduler (C1.1)** — places energy services. When a pod annotated with `hedge-iot.eu/*` task properties enters `Pending` state, the scheduler syncs cluster state to a Neo4j graph, runs an Ant Colony Optimisation to select the best node, and binds the pod. It also executes *solving events* from the Alert Manager: deploying, scaling and migrating services one step at a time.
- **Alert Manager (C1.2)** — decides *when* to act. It evaluates operator-defined rules against Prometheus metrics (computational alerts) and RabbitMQ grid events (grid alerts), and sends the scheduler an ordered mitigation plan when one fires.

```
 Prometheus (E2) ──pull──┐
                         ├──> Alert Manager ──solving event──> Scheduler ──> Kubernetes
 RabbitMQ   (E4) ──push──┘         ^                              │
                                   └──────── status callback ─────┘
```

The Alert Manager is optional: the scheduler places pods on its own without it.

---

## Prerequisites

- A running and healthy KubeEdge cluster (edge and cloud nodes joined) and `kubectl` configured and pointing to the cluster - [E0](https://github.com/HEDGE-IoT/orchestrator-blueprint_external_prereq/blob/main/KubeEdge.md)
- Cilium [E1](https://github.com/HEDGE-IoT/orchestrator-blueprint_external_prereq/blob/main/Cilium.md)
- `helm` installed [Helm Package Manager](https://github.com/HEDGE-IoT/orchestrator-blueprint_external_prereq/blob/main/Helm.md)
- Kube Prometheus Stack [E2](https://github.com/HEDGE-IoT/orchestrator-blueprint_external_prereq/blob/main/KubePrometheusStack.md) — required for computational alerts
- RabbitMQ (E4) — only for grid alerts

---

## Setup

### 1. Clone the Deployment Project

```bash
git clone https://github.com/HEDGE-IoT/orchestrator-blueprint_orch_components_uc1
cd orchestrator-blueprint_orch_components_uc1
```

### 2. Install the Scheduler

```bash
kubectl apply -f scheduler_service/neo4j.yaml
kubectl apply -f scheduler_service/rbac.yaml
kubectl apply -f scheduler_service/scheduler-deployment.yaml
```

Verify that the scheduler and Neo4j are running:

```bash
kubectl get pods -n kube-system -l 'app in (hedge-iot-scheduler, neo4j)' -o wide
```

Wait until both show `Running` before deploying task workloads.

### 3. Install the Alert Manager (optional)

Before applying, review in `alert_manager/alert-manager-deployment.yaml`:

- the `alert-manager-broker` Secret — your RabbitMQ URL and credentials;
- `PROMETHEUS_URL` — defaults to a Kube Prometheus Stack release named `kube-prom` in `monitoring`;
- `AMQP_ENABLED` / `PROMETHEUS_ENABLED` — set either to `"false"` to run without that prerequisite (at least one must stay on).

```bash
kubectl apply -f alert_manager/crds/
kubectl apply -f alert_manager/rbac.yaml
kubectl apply -f alert_manager/alert-manager-deployment.yaml
kubectl apply -f alert_manager/servicemonitor.yaml   # optional, needs E2
```

Verify:

```bash
kubectl get pods -n kube-system -l app=alert-manager
kubectl logs -n kube-system deploy/alert-manager
```

The log opens with the effective configuration, e.g.
`mode=cluster ... broker=on prometheus=on`.

### 4. Load Alert Rules

`alert-rules/` contains a starter set:

| File | Kind | What it does |
|---|---|---|
| `rule-node-cpu-pressure.yaml` | computational | Rebalances when a node's CPU pressure (PSI) stays above 20% for 90s. Ships with `dryRun: true`. |
| `rule-grid-congestion.yaml` | grid | Fires on a congestion event with severity ≥ 3 from the `hedge.grid` exchange. |
| `workflow-congestion-mitigation.yaml` | workflow | The grid rule's plan: deploy a congestion handler near the substation, then scale `task-power-monitor`. |

```bash
kubectl apply -f alert-rules/
kubectl get alertrules,mitigationworkflows -A
```

Rules are picked up without a restart.

---

## Usage

### Deploy Task Workloads

Task workloads are standard Deployments with the custom scheduler and `hedge-iot.eu/*` annotations:

```yaml
spec:
  template:
    metadata:
      labels:
        schedulingStrategy: meetup
      annotations:
        hedge-iot.eu/cycles-required: "100000000"
        hedge-iot.eu/input-size: "6793216"
        hedge-iot.eu/output-size: "4841472"
        hedge-iot.eu/exe-size: "33241088"
        hedge-iot.eu/deadline: "8838"
        hedge-iot.eu/ccr: "4.52"
        hedge-iot.eu/data-source: "temperature"
    spec:
      schedulerName: my-scheduler #important spec to use the custom scheduler for placing
      containers:
      - name: app
        resources:
          requests:
            cpu: 100m
```

Example workloads are provided in `task-deployments/`:

```bash
kubectl apply -f task-deployments/nginx-tasks.yaml
```

#### Verify Placement
Check Deployment is assigned and running:

```bash
kubectl get pods -n <DEPLOYED_NAMESPACE> -o wide
```
Check the scheduler logs:
```bash
kubectl logs -n kube-system deploy/hedge-iot-scheduler
```

### Trigger a Grid Alert

Publish a congestion event to the `hedge.grid` exchange with a routing key under `grid.congestion.`, using any AMQP client or the RabbitMQ management UI:

```json
{
  "specversion": "1.0",
  "type": "eu.hedge-iot.grid.congestion",
  "source": "congestion-forecasting",
  "id": "c-001",
  "data": {
    "substation_id": "SS-14",
    "severity": 4,
    "node": "<EDGE_NODE_NAME>",
    "start_ts": "2026-06-01T12:00:00Z",
    "end_ts": "2026-06-01T12:30:00Z"
  }
}
```

- `substation_id`, `severity`, `start_ts`, `end_ts` and `node` are all required; severity below 3 is ignored.
- Omit the CloudEvents `time` field, or set it to now. Events older than 300s are discarded as stale.
- Send a new `id` for every event. The same substation will not fire again until its previous mitigation finishes and a 120s cooldown passes.

Within a second, `congestion-handler-ss-14` should appear, placed by the scheduler:

```bash
kubectl logs -n kube-system deploy/alert-manager    # received ... evaluated ... firing
kubectl logs -n kube-system deploy/hedge-iot-scheduler
kubectl get deploy congestion-handler-ss-14 -o wide
```

To check a rule without the broker, preview what it would do:

```bash
kubectl port-forward -n kube-system svc/alert-manager 8080:8080 &
curl -s -X POST localhost:8080/v1/simulate/default/grid-congestion-detected \
  -H 'content-type: application/json' \
  -d '{"payload":{"substation_id":"SS-14","severity":4,"node":"edge-3",
       "start_ts":"2026-06-01T12:00:00Z","end_ts":"2026-06-01T12:30:00Z"}}'
```

The response shows the extracted fields and the fully rendered plan, or which check rejected the payload.

---

## Configuration

### Scheduler Environment Variables

Configured in `scheduler_service/scheduler-deployment.yaml`:

| Variable | Default | Description |
|---|---|---|
| `SCHEDULER_NAME` | `my-scheduler` | Must match `spec.schedulerName` in task manifests |
| `WORKLOAD_SELECTOR` | `schedulingStrategy=meetup` | Label a pod needs to be watched |
| `NAMESPACE` | `default` | Namespace whose pods are scheduled |
| `NEO4J_URI` | `bolt://neo4j.kube-system.svc.cluster.local:7687` | Bolt endpoint. Unreachable ⇒ resource-spread placement instead of ACO |
| `NEO4J_USER` | `neo4j` | Neo4j username |
| `NEO4J_PASSWORD` | from Secret | Neo4j password |

### Alert Manager Environment Variables

Configured in `alert_manager/alert-manager-deployment.yaml`:

| Variable | Default | Description |
|---|---|---|
| `PROMETHEUS_ENABLED` | `true` | Computational alerts on/off |
| `PROMETHEUS_URL` | `http://kube-prom-kube-prometheus-prometheus.monitoring.svc:9090` | E2 Prometheus |
| `AMQP_ENABLED` | `true` | Grid alerts on/off |
| `AMQP_URL` | from Secret `alert-manager-broker` | E4 RabbitMQ |
| `AMQP_EXCHANGE` | `hedge.grid` | Topic exchange carrying grid events |
| `EVENT_LISTENER_URL` | `http://hedge-iot-scheduler.kube-system.svc:8080` | Where solving events are sent |
| `CALLBACK_BASE_URL` | `http://alert-manager.kube-system.svc:8080` | Where the scheduler reports status. Must point at the Alert Manager itself |
| `GLOBAL_DRY_RUN` | `false` | `true`: rules evaluate, nothing is sent to the scheduler |

### Changing the Neo4j Password

Edit the `neo4j-credentials` Secret in `neo4j.yaml` before applying:

```yaml
stringData:
  auth: "neo4j/<YOUR_PASSWORD>"
  password: "<YOUR_PASSWORD>"
```

The scheduler reads the `password` key from the same Secret.

### Neo4j Node Placement

Neo4j, the scheduler and the Alert Manager are pinned to the control-plane node via `nodeAffinity` on the label `node-role.kubernetes.io/control-plane`.

---

## Supported Task Annotations

All annotations use the `hedge-iot.eu/` prefix. Every one is optional: a missing value is derived from the container's resource requests.

| Annotation | Type | Description |
|---|---|---|
| `hedge-iot.eu/cycles-required` | int | CPU demand in millicores × 10⁶ (`100m` = `100000000`) |
| `hedge-iot.eu/input-size` | int | Input data size (bytes) |
| `hedge-iot.eu/output-size` | int | Output data size (bytes) |
| `hedge-iot.eu/exe-size` | int | Executable size (bytes) |
| `hedge-iot.eu/deadline` | int | Task deadline (ms) |
| `hedge-iot.eu/ccr` | float | Communication-to-computation ratio |
| `hedge-iot.eu/data-source` | string | Data source identifier |
