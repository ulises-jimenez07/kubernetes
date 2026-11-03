# 06 · Namespaces & Context

---

## What is a namespace?

A **namespace** is a **logical separation** inside a cluster — a "virtual cluster" with its own **scope**.

- Objects created in namespace A are only listed when you look at namespace A.
- Lets you run e.g. **dev** and **QA/UAT** in the **same physical cluster** (sharing machines) instead of paying for one cluster per environment.
- Also the unit for **limits/quotas** (LimitRange, ResourceQuota) and **permissions** (RBAC Roles).
- Names must be unique **per namespace** (two `backend` deployments can exist in `dev` and `prod`).

> Namespaces separate *names and policies*, not network traffic: by default pods in different namespaces can still talk to each other (use NetworkPolicies to block that).

## Default namespaces

```bash
kubectl get namespaces        # or: kubectl get ns
```

| Namespace | Purpose |
|---|---|
| `default` | Where everything goes if you don't specify a namespace (everything so far!) |
| `kube-system` | Kubernetes' own components (DNS, kube-proxy, controllers…) — don't touch |
| `kube-public` | Readable by everyone (even unauthenticated); mostly empty |
| `kube-node-lease` | Node heartbeat leases |

```bash
kubectl get pods                        # == kubectl get pods -n default
kubectl get pods -n kube-system
kubectl get pods --all-namespaces       # or -A
```

## Create a namespace

Imperative:
```bash
kubectl create namespace test-ns
kubectl get ns --show-labels
kubectl describe ns test-ns            # shows labels, annotations, and "No resource quota / No LimitRange"
```

Declarative (`ns.yaml`):
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    name: development
```

Delete: `kubectl delete ns test-ns` → **deletes everything inside it**.

## Objects in a namespace

```bash
kubectl run podtest34 --image=nginx:alpine -n test-ns
kubectl get pods                 # not here (that's "default")
kubectl get pods -n test-ns      # works
kubectl delete pod podtest34     # Error: not found (looking in default!)
kubectl delete pod podtest34 -n test-ns
```

In YAML, put the namespace in `metadata.namespace` — see `envs-ns.yaml`: creates `dev` + `prod` namespaces and a Deployment in each.

```bash
kubectl apply -f envs-ns.yaml
kubectl get deploy -n dev
kubectl get pods -n prod
```

## DNS across namespaces

Every Service gets a DNS record:

```
<service>.<namespace>.svc.cluster.local
```

`ns-and-dns.yaml` deploys the hands-on backend + its Service into namespace `ci`.

```bash
kubectl apply -f ns-and-dns.yaml
kubectl run --rm -it tmp --image=nginx:alpine -- sh     # pod in "default"
apk add -q curl
curl backend-k8s-hands-on                               # fails: can't resolve: short names only work in the SAME namespace
curl backend-k8s-hands-on.ci                            # works: <svc>.<ns>
curl backend-k8s-hands-on.ci.svc.cluster.local          # works: fully qualified
```

## Context

Problem: you forget `-n ci` and get "not found" (or worse, data from a same-named object in another namespace).

**kubeconfig** (`~/.kube/config`, or `/root/.kube/config` if the cluster was created as root) contains:
- **clusters** – API server URL + CA certificate
- **users** – credentials (certs, tokens)
- **contexts** – a combination of **cluster + user + namespace**

```bash
kubectl config view                        # see the kubeconfig (secrets redacted)
kubectl config current-context             # e.g. minikube
kubectl config get-contexts

# create a context that points at namespace "ci"
kubectl config set-context ci-context --namespace=ci --cluster=minikube --user=minikube
kubectl config use-context ci-context
kubectl get pods                           # now shows pods in "ci" without -n

kubectl config use-context minikube        # back

# or just change the namespace of the current context
kubectl config set-context --current --namespace=ci
```

> Tools like **kubectx / kubens** make switching faster: `brew install kubectx`.

---

## Cheat sheet

```bash
kubectl get ns
kubectl create ns <name>  |  kubectl delete ns <name>
kubectl <verb> <resource> -n <ns>     |  -A / --all-namespaces
kubectl config get-contexts | current-context | use-context <ctx>
kubectl config set-context --current --namespace=<ns>
# DNS: <svc>.<ns>.svc.cluster.local
```
