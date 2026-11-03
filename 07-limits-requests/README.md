# 07 · Limit RAM & CPU: requests and limits

---

## Why limits?

A pod **without limits** can use **all the CPU and RAM of its node** — one bad pod can starve the others or take the node down. So you set resource boundaries **per container** (possible because the **cgroups** namespace is per-container, see [01-pods](../01-pods/README.md)).

### Units

| Resource | Units | Examples |
|---|---|---|
| **Memory** | bytes: `K, M, G` (×1000) or `Ki, Mi, Gi` (×1024) | `128Mi`, `500M`, `1Gi` |
| **CPU** | cores or **millicores** (`m`): **1 CPU = 1000m** | `0.5` = `500m`, `0.1` = `100m` |

## Requests vs limits

```yaml
resources:
  requests:        # GUARANTEED. The scheduler only places the pod on a node with this much free.
    memory: 100Mi
    cpu: 500m
  limits:          # MAXIMUM. The container may burst up to here IF the node has spare capacity.
    memory: 200Mi
    cpu: "1"
```

- **request** = reserved for you; used for **scheduling**.
- between request and limit = **not guaranteed** (depends on what the node has free).
- above limit → depends on the resource (see below)

| Exceeds the **limit** in… | What happens |
|---|---|
| **Memory** (incompressible) | Container is **killed: `OOMKilled`** and restarted (→ `CrashLoopBackOff`) |
| **CPU** (compressible) | Container is **throttled** — never killed, just can't use more |

## Experiments

### Memory between request and limit

`polinux/stress` allocates **150M**; request 100Mi, limit 200Mi → **Running** fine (using non-guaranteed memory).

```bash
kubectl apply -f limit-ram.yaml
kubectl get pods
kubectl describe pod memory-demo
kubectl top pod memory-demo        # needs metrics-server: minikube addons enable metrics-server
```

### Memory above the limit

Allocates **250M** > 200Mi limit:
```bash
kubectl apply -f limit-ram2.yaml
kubectl get pods --watch           # OOMKilled → restart → OOMKilled → CrashLoopBackOff
kubectl describe pod memory-demo   # Last State: Terminated, Reason: OOMKilled
```

### Request more memory than any node has

Requests **1000Gi**:
```bash
kubectl apply -f limit-ram3.yaml
kubectl get pods                   # STATUS: Pending (forever)
kubectl describe pod memory-demo   # Events: FailedScheduling … 0/1 nodes are available: Insufficient memory
```
`Pending` = the scheduler can't find a node that satisfies the **request**.

### CPU above the limit

`vish/stress -cpus 2` with limit `1` / request `0.5`:
```bash
kubectl apply -f limit-cpu.yaml
kubectl top pod cpu-demo           # ~1000m — capped, NOT restarted
```

### Request more CPU than any node has

Requests **100 CPUs**:
```bash
kubectl describe node minikube     # Capacity/Allocatable: cpu: 4 (for example)
kubectl apply -f limit-cpu2.yaml
kubectl get pods                   # Pending — Insufficient cpu
```

> Know your nodes: `kubectl describe node <name>` shows **Capacity**, **Allocatable** and **Allocated resources**. In the cloud the machine type tells you (e.g. GKE `e2-medium` = 2 vCPU / 4 GB).

## QoS classes

Kubernetes assigns every pod a **Quality of Service class** based on its requests/limits. It decides **who gets evicted first** when a node runs out of memory.

```bash
kubectl get pod <pod> -o yaml | grep qosClass
```

| QoS class | Condition | Eviction priority |
|---|---|---|
| **Guaranteed** | Every container has requests **=** limits for **both** CPU and memory | Last to be killed |
| **Burstable** | At least one request/limit set, but not Guaranteed (e.g. request < limit) | Middle |
| **BestEffort** | **No** requests or limits at all | **First** to be killed |

> Best practice: always set requests (so scheduling is correct) and memory limits. Critical workloads → Guaranteed.

---

## Cheat sheet

```bash
kubectl describe node <node>          # capacity & allocated
kubectl top nodes | kubectl top pods  # live usage (metrics-server)
kubectl get pods --watch
kubectl get pod <p> -o jsonpath='{.status.qosClass}'
kubectl set resources deployment <d> -c=<container> --requests=cpu=100m,memory=128Mi --limits=cpu=500m,memory=256Mi
```
