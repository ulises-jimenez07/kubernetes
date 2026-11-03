# 09 · ResourceQuota — namespace-wide limits

---

## What is a ResourceQuota?

A LimitRange caps **each** pod (e.g. 1 CPU per pod) — but someone can still create a Deployment with **200 replicas** = 200 CPUs.

A **ResourceQuota** caps the **total** for the whole **namespace**:

> "In namespace `uat`, the sum of all requests may not exceed 1 CPU / 1Gi, and the sum of all limits may not exceed 2 CPU / 2Gi."

It can limit:
- **compute**: `requests.cpu`, `requests.memory`, `limits.cpu`, `limits.memory`
- **object counts**: `pods`, `services`, `configmaps`, `secrets`, `persistentvolumeclaims`, `services.loadbalancers`, `count/deployments.apps`…
- **storage**: `requests.storage`

## Your first ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: res-quota
  namespace: uat
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```

```bash
kubectl apply -f res-quota.yaml
kubectl describe ns uat                       # shows Resource Quotas: Used / Hard
kubectl get resourcequota -n uat              # or: kubectl get quota -n uat
kubectl describe quota res-quota -n uat
```

## Exceeding the quota

The file also has a Deployment with **3 replicas × 500m CPU / 500M memory**.

- 2 replicas × 500m = 1 CPU → exactly at the `requests.cpu` quota
- the 3rd replica would make 1.5 CPU → rejected

```bash
kubectl get deploy -n uat          # READY 2/3
kubectl describe deploy deployment-test -n uat   # not very informative
kubectl get rs -n uat
kubectl describe rs <rs-name> -n uat
# Events: Error creating: pods "…" is forbidden: exceeded quota: res-quota, requested: requests.cpu=500m, used: requests.cpu=1, limited: requests.cpu=1
```

**Important:** once a namespace has a compute quota, every pod **must declare** requests/limits for the quota'd resources, otherwise:
```
failed quota: res-quota: must specify limits.cpu, limits.memory, requests.cpu, requests.memory
```
(Tip: combine with a **LimitRange** that injects defaults.)

## Limit the number of pods

```yaml
spec:
  hard:
    pods: "3"
```

Deployment asks for **4** replicas in namespace `qa`:
```bash
kubectl apply -f pod-quota.yaml
kubectl get pods -n qa             # only 3
kubectl describe rs -n qa          # exceeded quota: pod-demo, requested: pods=1, used: pods=3, limited: pods=3
```

---

## Cheat sheet

```bash
kubectl get quota -n <ns>
kubectl describe quota <name> -n <ns>
kubectl describe ns <ns>
kubectl describe rs <rs> -n <ns>        # where quota errors show up for Deployments
kubectl create quota my-quota --hard=pods=10,requests.cpu=2 -n <ns>
```
