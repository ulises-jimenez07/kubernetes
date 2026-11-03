# 01 · Pods

---

## Part 1 — Theory: from servers to containers to pods

### History of deployments

| Era | How it worked | Problem |
|---|---|---|
| **Traditional** | App installed directly on a physical server's OS | Scaling = buying more hardware; resources wasted |
| **Virtual machines** | A *hypervisor* runs many full OSes on one machine | Each VM still needs its own OS, fixed RAM/disk (e.g. 2 GB RAM, 50 GB disk) |
| **Containers** | A *container runtime* (Docker) runs isolated processes sharing the host kernel | — lightweight; you can give a container just 50–100 MB RAM |

A **container** = an isolated process that ships with all the libraries and binaries the app needs.

### What makes a container isolated: Linux namespaces + cgroups

When a container is created it gets its **own copy** of each of these:

| Namespace | What it isolates |
|---|---|
| **IPC** (Inter-Process Communication) | Processes can only talk to processes inside the same container |
| **cgroups** | Resource limits: CPU, RAM, network bandwidth… |
| **Network** | Own network stack and **own IP**; can't see other containers' networks |
| **Mount** | Own filesystem mounts (a folder/NFS mounted in container A isn't visible in B) |
| **PID** | Own process IDs (both containers can have a PID 1, neither sees the other's) |
| **User** | Own users (users created in A don't exist in B) |
| **UTS** | Own **hostname** |

So by default two containers are completely isolated mini-systems.

### How a pod is built behind the scenes

- **LXC** (Linux Containers) gave full isolation but made container-to-container communication hard.
- **Docker** made it easier with a virtual **bridge network** that containers join.
- **Kubernetes** goes further: a **Pod** = one or more containers that **share some namespaces**.

How K8s builds a pod:
1. It starts a tiny temporary *dummy/"pause"* container.
2. It takes that container's **Network**, **IPC** and **UTS** namespaces and shares them with every container in the pod.
3. The setup container is no longer used; your containers remain.

Result:

| Shared inside a pod | Still per-container |
|---|---|
| **Network** → one IP for the whole pod; containers reach each other via `localhost` | **cgroups** → set CPU/RAM per container |
| **IPC** → containers see each other's processes | **Mount** → each container mounts its own volumes/filesystem |
| **UTS** → same hostname | **PID**, **User** |

> **Tip:** A Pod doesn't "run" anything itself — it's a **wrapper** around containers. The containers run; the pod gives them one shared IP/identity.

---

## Part 2 — Hands-on with kubectl

### Create a pod imperatively

```bash
kubectl run podtest --image=nginx:alpine
kubectl get pods            # or: kubectl get po
```

> **Warning:** Older tutorials use `kubectl run --generator=run-pod/v1 ...`. The `--generator` flag **no longer exists** — just omit it.

`kubectl get pods` columns:
- **READY** `1/1` → ready containers / total containers in the pod
- **STATUS** → `Running`, `ContainerCreating`, `ErrImagePull`, `ImagePullBackOff`, `CrashLoopBackOff`, `Error`…
- **RESTARTS** → how many times containers restarted
- **AGE**

### Describe — debug with events

```bash
kubectl run podtest2 --image=nginx:doesnotexist   # broken image on purpose
kubectl get pods                                   # stuck: 0/1, ErrImagePull
kubectl describe pod podtest2                      # read the "Events" section at the bottom
```

`describe` shows the object's full state and, most importantly, the **Events** at the end (Scheduled → Pulling → *Failed to pull image…*). This is the first place to look when a pod won't start.

### Delete pods

```bash
kubectl delete pod podtest2 podtest3   # one or many
kubectl api-resources                  # shows that "po" is the shortname for pods
```

Deleted bare pods **are not recreated** — nothing is watching them.

### Get the YAML of a pod

```bash
kubectl get pod podtest              # only that pod
kubectl get pod podtest -o yaml      # the full manifest stored in the API
```

When you run `kubectl run`, kubectl turns your command into JSON and sends it to the API server. `-o yaml` shows you what was actually stored (with lots of default fields).

> A **Pod is the smallest, atomic unit** you can create in Kubernetes.

### Pods vs Docker containers

- `docker ps` on the node shows the actual container behind your pod.
- **Never manage those containers directly with Docker** — always work at the pod level via kubectl.
- Flow when you create a pod: `kubectl` → **API server** → **scheduler** picks a node → the **kubelet** on that node asks the **container runtime** (Docker) to create the container.
- Delete the pod and Kubernetes removes the container too.

### Pod IP

```bash
kubectl describe pod podtest | grep IP
curl <pod-ip>:80                     # "Welcome to nginx"
```

- Pod IPs are **internal / virtual** — only reachable from inside the cluster. Reaching one from your laptop with Minikube only works because the laptop *is* the cluster.
- If you can't reach it (VirtualBox/HyperV driver, Docker Desktop on macOS, etc.), use port-forwarding:

```bash
kubectl port-forward podtest 7000:80   # then open http://localhost:7000
```

### Get a shell inside a pod

```bash
kubectl exec -it podtest -- sh
# inside:
cat /usr/share/nginx/html/index.html
echo ":)" > /usr/share/nginx/html/index.html   # refresh the browser → you see :)
```

Each pod has its own filesystem. In real life you rarely exec in and change things — pods die and get recreated all the time, so changes are lost.

### Logs

```bash
kubectl logs podtest
kubectl logs -f podtest      # follow (like tail -f)
```

Troubleshooting combo:
- **`kubectl describe pod`** → infrastructure problems (bad image, scheduling…) via *Events*.
- **`kubectl logs`** → application problems (unhandled error, missing config, crash on startup).

---

## Part 3 — Manifests (YAML)

### Why manifests instead of the CLI

Imperative (`kubectl run` ×50, `kubectl delete` ×50) has no history, is hard to manage and hard to clean up.
**Declarative manifests**:
- describe the desired state in a file,
- can be versioned in Git, shared, reviewed,
- create/update/delete everything in the file with one command.

```bash
kubectl apply  -f pod.yaml     # create / update
kubectl delete -f pod.yaml     # delete everything defined in the file
```

Where the top-level fields come from:

```bash
kubectl api-resources | grep pods
# NAME   SHORTNAMES  APIVERSION  NAMESPACED  KIND
# pods   po          v1          true        Pod
```

- `apiVersion: v1` → pods have **no API group** (core group), so just `v1`.
- `kind: Pod` → the **KIND** column (capitalised).

See `pod.yaml`. Several objects can live in one file separated by `---`.

### Multiple containers in one pod

The common pattern is **one container per pod**. Multi-container pods are for tightly coupled helpers (e.g. one writes data, the other serves it — "sidecar").

Exercise: two `python:3.6-alpine` containers, each writes its own `index.html` and serves it with `python -m http.server`.

**First attempt (both on port 8082) fails** → the pod goes into `Error` / `CrashLoopBackOff`.

```bash
kubectl describe pod doscont          # only says the container keeps restarting
kubectl logs doscont -c cont1         # fine
kubectl logs doscont -c cont2         # OSError: Address in use  ← the real cause
```

Why: containers in a pod **share the network namespace** = same IP, same `localhost`, so they can't both bind the same port.

Fix: give the second container port 8083 — see [`two-containers.yaml`](two-containers.yaml).

```bash
kubectl exec -it doscont -c cont1 -- sh
apk add -q curl
curl localhost:8082    # → cont1
curl localhost:8083    # → cont2   (other container, same localhost!)
```

Also notice: each container still has **its own filesystem** (`index.html` differs), because the Mount namespace isn't shared.

> **Warning:** Trying to `kubectl apply` a changed `command` on an existing pod fails: **pods are mostly immutable** (only a few fields like `image` can change). You must delete and recreate it — another hint that higher-level objects are needed.

`-c <container>` is required for `logs` / `exec` when a pod has more than one container.

`command` must be a list, e.g.:

```yaml
command: ["sh", "-c", "echo cont1 > index.html && python -m http.server 8082"]
```

### Labels

Labels = arbitrary **key/value metadata** under `metadata.labels` to identify and group objects (by app, environment, team…). They don't change what the pod does.

```bash
kubectl apply -f labels.yaml
kubectl get pods -l app=backend
kubectl get pods -l env=dev
kubectl get pods --show-labels
kubectl label pod <name> app=frontend      # add a label to a running pod
```

**Why they matter:** higher-level objects (ReplicaSets, Deployments, Services) find "their" pods **only by labels** (selectors). Always label your pods, at least with `app`.

See `labels.yaml`.

### Problems with bare pods

1. **No self-healing** – delete or lose a pod and nobody recreates it.
2. **No replication** – want 50 copies? Copy-paste the manifest 50 times.
3. **Can't update themselves** – changing command/config requires delete + recreate.

Solution: let a higher-level object manage pods → **ReplicaSet** ([next](../02-replicasets/README.md)).

---

## Cheat sheet

```bash
kubectl run <name> --image=<img>
kubectl get pods [-o wide|-o yaml] [--show-labels] [-l key=value]
kubectl describe pod <name>
kubectl logs [-f] <pod> [-c <container>]
kubectl exec -it <pod> [-c <container>] -- sh
kubectl port-forward <pod> <local>:<podPort>
kubectl label pod <pod> key=value
kubectl delete pod <name>...
kubectl apply -f file.yaml / kubectl delete -f file.yaml
kubectl api-resources / kubectl api-versions
```
