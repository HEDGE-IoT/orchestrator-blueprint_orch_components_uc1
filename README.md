# Energy Services Offloading

Task placement scheduler for Energy Services on a KubeEdge cluster. When a pod annotated with `hedge-iot/*` task properties enters `Pending` state, the scheduler syncs cluster state to a Neo4j graph, runs an Optimization to select the best node, and binds the pod.

---

## Prerequisites

- `kubectl` configured and pointing to the cluster
- `helm` installed
- [Docker Hub](https://hub.docker.com) account and repository with the Energy Service image

---

## Setup

### 1. Clone the Deployment Project

```bash
git clone https://github.com/HEDGE-IoT/orchestrator-blueprint_orch_components_uc1
cd orchestrator-blueprint_orch_components_uc1
```

### 2. Apply All Manifests
First install the `Kube Prometheus Stack`:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prom-stack prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --create-namespace
```
Then install the Scheduler Service Component:
```bash
kubectl apply -f scheduler_service/neo4j.yaml
kubectl apply -f scheduler_service/rbac.yaml
kubectl apply -f scheduler_service/scheduler-deployment.yaml
```

Verify all components are running:

```bash
kubectl get pods -n kube-system -o wide
```

Wait until both show `Running` before deploying task workloads.

## Usage

### Deploy Task Workloads

Task workloads are standard Deployments with the custom scheduler and `hedge-iot/*` annotations:

```yaml
spec:
  template:
    metadata:
      labels:
        schedulingStrategy: meetup
      annotations:
        hedge-iot/cycles-req: "178565"
        hedge-iot/input-size: "6634"
        hedge-iot/output-size: "4728"
        hedge-iot/exe-size: "32462"
        hedge-iot/deadline: "8838"
        hedge-iot/ccr: "4.52"
        hedge-iot/data-source: "temperature"
    spec:
      schedulerName: my-scheduler #important spec to use the custom scheduler for placing
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
kubectl logs -n kube-system <CUSTOM_SCHEDULER_POD_NAME>
```
---

## Configuration

### Scheduler Environment Variables

Configured in `scheduler-deployment.yaml`:

| Variable | Default | Description |
|---|---|---|
| `KUBE_CONFIG_PATH` | `""` | Empty = in-cluster config |
| `NEO4J_URI` | `bolt://neo4j.kube-system.svc.cluster.local:7687` | Bolt endpoint |
| `NEO4J_USER` | `neo4j` | Neo4j username |
| `NEO4J_PASSWORD` | from Secret | Neo4j password |

### Changing the Neo4j Password

Edit the `neo4j-credentials` Secret in `neo4j.yaml` before applying:

```yaml
stringData:
  auth: "neo4j/<YOUR_PASSWORD>"
  password: "<YOUR_PASSWORD>"
```

Then update the matching reference in `scheduler-deployment.yaml`.

### Neo4j Node Placement

Neo4j is pinned to the cloud/control-plane node via `nodeAffinity`. It targets any node matching **either**:

- Label `hedge-iot/node-type=cloud`, **or**
- Label `node-role.kubernetes.io/control-plane` exists

---

## Supported Task Annotations

All annotations use the `hedge-iot/` prefix:

| Annotation | Type | Description |
|---|---|---|
| `hedge-iot/cycles-req` | int | CPU cycles required |
| `hedge-iot/input-size` | int | Input data size (KB) |
| `hedge-iot/output-size` | int | Output data size (KB) |
| `hedge-iot/exe-size` | int | Executable size (KB) |
| `hedge-iot/deadline` | int | Task deadline (ms) |
| `hedge-iot/ccr` | float | Communication-to-computation ratio |
| `hedge-iot/data-source` | string | Data source identifier |
| `hedge-iot/task-type` | string | Task type label |
