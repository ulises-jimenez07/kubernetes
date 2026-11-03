# 15 · RBAC: ServiceAccounts

---

## What is a ServiceAccount?

Users are for **humans** outside the cluster. A **ServiceAccount (SA)** is an identity for **processes running in pods** that need to talk to the Kubernetes API — e.g. a monitoring pod that asks "how are the other pods doing?", a CI runner, an operator, an Ingress controller.

- A ServiceAccount **is** a Kubernetes object (namespaced) — unlike Users.
- It authenticates with a **token** (a JWT).
- You give it permissions with **Roles + RoleBindings**, exactly like users.

## Default ServiceAccounts

Every namespace automatically gets a ServiceAccount called **`default`**:
```bash
kubectl get serviceaccounts           # or: kubectl get sa
kubectl describe sa default
kubectl create ns hola && kubectl get sa -n hola    # "default" created automatically
kubectl get sa -n kube-system                       # lots of SAs used by K8s components
```

## The token behind a ServiceAccount

Before Kubernetes 1.24 every SA had an auto-generated Secret `default-token-xxxxx` (type `kubernetes.io/service-account-token`) containing three keys:

| Key | Content |
|---|---|
| `ca.crt` | the cluster CA's public certificate (to trust the API server's TLS) |
| `namespace` | the SA's namespace |
| `token` | a **JWT** signed by the cluster — decode it at jwt.io to see `serviceaccount` name, namespace, uid |

```bash
kubectl get secret <sa-token-secret> -o yaml
echo <value> | base64 --decode
```

> **Changed in Kubernetes 1.24+**: these long-lived token Secrets are **no longer created automatically**. Pods get a short-lived, auto-rotated **projected token** instead (same path, same 3 files). To get a token manually:
> ```bash
> kubectl create token my-sa            # short-lived token
> ```
> If you really need a long-lived Secret token, create a Secret of type `kubernetes.io/service-account-token` with annotation `kubernetes.io/service-account.name: my-sa`.

## Create your own ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-sa
```
```bash
kubectl apply -f sa.yaml
kubectl get sa
kubectl describe sa my-sa
```
(`kubectl create sa my-sa` works too.)

## Pods ↔ ServiceAccounts

```bash
kubectl get pod <pod> -o yaml | grep -i serviceaccount
#   serviceAccount: default
#   serviceAccountName: default
```
- If you don't specify one, every pod uses the namespace's **`default`** SA.
- The token is **automatically mounted** into every container at:

```
/var/run/secrets/kubernetes.io/serviceaccount/
    ├── ca.crt
    ├── namespace
    └── token
```
```bash
kubectl exec -it <pod> -- ls /var/run/secrets/kubernetes.io/serviceaccount
```
Disable it when a pod doesn't need API access: `automountServiceAccountToken: false` (on the pod spec or the SA).

## Call the API without auth

Inside a pod there's no kubectl, but the API is just HTTPS. The API server is reachable through the `kubernetes` Service in `default`:
```bash
kubectl exec -it <pod> -- sh
apk add -q curl
env | grep KUBERNETES_SERVICE            # KUBERNETES_SERVICE_HOST / _PORT
curl https://kubernetes/api/v1 --insecure
# 403 Forbidden: User "system:anonymous" cannot get path "/api/v1"
```

## Call the API with the JWT

```bash
SA=/var/run/secrets/kubernetes.io/serviceaccount
TOKEN=$(cat $SA/token)
curl -H "Authorization: Bearer $TOKEN" --cacert $SA/ca.crt https://kubernetes/api/v1
# works: returns the list of API resources → authenticated as system:serviceaccount:default:default
curl -H "Authorization: Bearer $TOKEN" --cacert $SA/ca.crt https://kubernetes/api/v1/namespaces/default/pods
# fails: 403 — the default SA has no permission to list pods
```
Full name of an SA identity: **`system:serviceaccount:<namespace>:<name>`**.

## Assign a ServiceAccount to a Deployment

```yaml
spec:
  template:
    spec:
      serviceAccountName: my-sa      # pod spec level, next to "containers"
      containers:
      - name: nginx
        image: nginx:alpine
```
Behind the scenes K8s mounts *my-sa*'s token instead of default's — that's all you need to write.

## Give the ServiceAccount permissions

Same Role/RoleBinding as for users, but the subject is a ServiceAccount:
```yaml
kind: Role
metadata:
  namespace: default
  name: sa-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "watch", "list"]
---
kind: RoleBinding
metadata:
  name: sa-pods
  namespace: default
subjects:
- kind: ServiceAccount
  name: my-sa
  namespace: default        # recommended for SA subjects
roleRef:
  kind: Role
  name: sa-reader
  apiGroup: rbac.authorization.k8s.io
```

## Validate

```bash
kubectl apply -f sa.yaml
kubectl exec -it <test-pod> -- sh
# (TOKEN/SA as above)
curl -H "Authorization: Bearer $TOKEN" --cacert $SA/ca.crt \
  https://kubernetes/api/v1/namespaces/default/pods                       # works: PodList JSON
curl … https://kubernetes/api/v1/namespaces/hola/pods                     # fails: other namespace
curl … https://kubernetes/apis/apps/v1/namespaces/default/deployments     # works: after adding the apps rule
```
Note the URL: core group = `/api/v1/…`, named groups = `/apis/<group>/<version>/…`.

From outside the pod:
```bash
kubectl auth can-i list pods --as=system:serviceaccount:default:my-sa
```

---

## ServiceAccounts on GKE: Workload Identity Federation

A Kubernetes SA only grants access to the **Kubernetes API**. To let pods call **Google Cloud APIs** (Cloud Storage, Pub/Sub, Secret Manager…) without key files, GKE uses **Workload Identity Federation for GKE**:

```bash
# cluster created with --workload-pool=PROJECT_ID.svc.id.goog (Autopilot: on by default)
gcloud projects add-iam-policy-binding PROJECT_ID \
  --role=roles/storage.objectViewer \
  --member=principal://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/PROJECT_ID.svc.id.goog/subject/ns/default/sa/my-sa
```
Pods running as `my-sa` then automatically get Google credentials with those IAM roles.

---

## Cheat sheet

```bash
kubectl get sa [-n ns] | kubectl describe sa <sa>
kubectl create sa <sa>
kubectl create token <sa> [--duration=1h]
kubectl set serviceaccount deployment <dep> <sa>
kubectl create rolebinding x --role=<role> --serviceaccount=<ns>:<sa> -n <ns>
kubectl auth can-i <verb> <res> --as=system:serviceaccount:<ns>:<sa>
```
