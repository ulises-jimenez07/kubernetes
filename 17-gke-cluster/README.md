# 17 · GKE: setup & a real production cluster

These notes use **Google Kubernetes Engine (GKE)**, with AWS EKS equivalents for comparison. Everything Kubernetes-side (kubectl, manifests) is identical — only the cloud tooling changes.

| AWS EKS | GKE |
|---|---|
| AWS CLI (`aws`) | Google Cloud CLI (`gcloud`) |
| IAM user + access keys | your Google account (`gcloud auth login`) + IAM roles |
| `eksctl` | `gcloud container …` (built in) |
| `eksctl create cluster --without-nodegroup` | `gcloud container clusters create` |
| `eksctl create nodegroup` | `gcloud container node-pools create` |
| `aws eks update-kubeconfig` | `gcloud container clusters get-credentials` |
| EC2 instances, CloudFormation stacks | Compute Engine VMs, GKE manages it |

---

## Why a managed cluster?

Minikube = learning. In production you want a cluster where the cloud runs the **control plane** (API server, etcd, scheduler, controllers — HA, upgraded and backed up for you) and you just manage **worker nodes** (or not even that).

GKE modes:
- **Standard** – you manage node pools (machine type, count, autoscaling). Used below.
- **Autopilot** – Google manages nodes too; you pay per pod resource requests. Simplest for production: `gcloud container clusters create-auto my-cluster --region us-central1`.

> **Cost:** GKE charges a cluster management fee, but the **GKE free tier** credit covers it for one zonal (or Autopilot) cluster per billing account. You still pay for nodes (VMs), load balancers and disks. New accounts get free trial credits. **Delete everything when you're done** ([21-gke-cleanup](../21-gke-cleanup/README.md)).

## Install the gcloud CLI

```bash
# macOS (Homebrew)
brew update && brew install --cask gcloud-cli
# or the official installer: https://cloud.google.com/sdk/docs/install
gcloud version

# kubectl auth plugin for GKE (required)
gcloud components install gke-gcloud-auth-plugin
```

## Credentials & permissions

In AWS you'd create an IAM user with access keys. In GCP you log in with your Google identity:

```bash
gcloud auth login                              # opens the browser
gcloud projects create k8s-lab-<random>     # or use an existing project
gcloud config set project k8s-lab-<random>
gcloud billing projects link k8s-lab-<random> --billing-account=XXXXXX-XXXXXX-XXXXXX
gcloud services enable container.googleapis.com compute.googleapis.com
gcloud config set compute/region us-central1
gcloud config set compute/zone   us-central1-a
```
Your account needs the IAM role **Kubernetes Engine Admin** (`roles/container.admin`) — project Owners already have it.
For CI/CD use a **Google service account** with that role (and Workload Identity Federation instead of key files).

## Cluster tooling

Not needed: `gcloud container` does what `eksctl` does. (Optional: Terraform or Config Connector for infrastructure-as-code.)

## Create a cluster

EKS equivalent: `eksctl create cluster --name test-cluster --region us-east-1 --zones us-east-1a,us-east-1b --without-nodegroup`

GKE equivalent. A GKE cluster must be created with at least one node pool, so we create a tiny default pool and replace it with our own pool in a moment:

```bash
gcloud container clusters create test-cluster \
  --zone us-central1-a \
  --num-nodes 1 \
  --machine-type e2-medium \
  --release-channel regular
# ~5–10 minutes
```

Zonal vs regional:
- `--zone` → **zonal** cluster: one control plane in one zone (covered by the free tier — good for learning).
- `--region` (optionally `--node-locations zone-a,zone-b`) → **regional**: control plane replicated across 3 zones (production HA).

```bash
gcloud container clusters list
gcloud container clusters describe test-cluster --zone us-central1-a
```
Also visible in the console: **Kubernetes Engine → Clusters**.

## Configure kubectl

`gcloud container clusters create` already writes the credentials to `~/.kube/config`. On another machine / later:
```bash
gcloud container clusters get-credentials test-cluster --zone us-central1-a
kubectl config current-context    # gke_<project>_us-central1-a_test-cluster
kubectl get nodes
kubectl get pods -A               # GKE system pods in kube-system / gke-* namespaces
```
Everything you learned on Minikube now works here unchanged — Minikube really was a faithful learning environment.

## A test pod

```bash
kubectl run podtest --image=nginx:alpine
kubectl get pods -o wide          # NODE = a gke-test-cluster-… VM
```
On EKS with no node group the pod stays **Pending** — there are no nodes to schedule it on. On GKE it runs on the default pool's node. To see it: `kubectl describe pod` on a pod requesting more than the node has → `FailedScheduling`.

## Add nodes: a node pool

EKS equivalent: `eksctl create nodegroup --cluster=test-cluster --name=test-workers --node-type=t3.medium --nodes=1 --nodes-min=1 --nodes-max=3 --asg-access --spot`

GKE:
```bash
gcloud container node-pools create test-workers \
  --cluster test-cluster --zone us-central1-a \
  --machine-type e2-medium \
  --num-nodes 1 \
  --enable-autoscaling --min-nodes 1 --max-nodes 3 \
  --spot                                    # Spot VMs: much cheaper, can be reclaimed

# remove the bootstrap pool so only test-workers remains (pods get rescheduled)
gcloud container node-pools delete default-pool --cluster test-cluster --zone us-central1-a
```
- `e2-medium` ≈ `t3.medium` (2 vCPU, 4 GB).
- `--enable-autoscaling --min-nodes/--max-nodes` replaces `--nodes-min/--nodes-max` **and** `--asg-access`: on GKE the Cluster Autoscaler is built in (see [20-gke-cluster-autoscaler](../20-gke-cluster-autoscaler/README.md)).
- A node pool = a group of identical VMs (a Managed Instance Group under the hood, like an AWS Auto Scaling Group).

```bash
gcloud container node-pools list --cluster test-cluster --zone us-central1-a
gcloud compute instances list         # the node VMs (≈ EC2 console)
```

## Check the pod after adding nodes

```bash
kubectl get nodes                     # gke-test-cluster-test-workers-…  Ready
kubectl get pods -o wide              # podtest Running on the new node
```
A bare pod on the deleted default pool is gone (nothing recreates bare pods — lesson from [01-pods](../01-pods/README.md)); a Deployment's pods would have been rescheduled automatically.

---

## Cheat sheet

```bash
gcloud container clusters create|create-auto|list|describe|delete <name> --zone|--region …
gcloud container clusters get-credentials <name> --zone …
gcloud container node-pools create|list|delete <pool> --cluster <name> --zone …
gcloud container clusters resize <name> --node-pool <pool> --num-nodes N --zone …
gcloud container get-server-config --zone …       # available versions
```
