# 21 · GKE: destroy everything you created

On EKS you'd delete the node group CloudFormation stack first, then the cluster stack, and check no load balancers are left. On GKE:

## 1. Delete what creates billable resources *outside* the nodes

```bash
kubectl get ingress -A
kubectl get svc -A | grep LoadBalancer
kubectl get pvc -A

kubectl delete ingress --all -n game-2048       # removes the Application Load Balancer
kubectl delete -f ../18-gke-ingress/game-2048.yaml
kubectl delete -f ../19-gke-hpa/hpa.yaml
kubectl delete pvc --all -A                     # StorageClass PVs (reclaim Delete) remove their Persistent Disks
```
Do this **before** deleting the cluster so the controllers clean up the cloud resources they created.

## 2. Delete the cluster (node pools are deleted with it)

```bash
gcloud container clusters delete test-cluster --zone us-central1-a --quiet
```

## 3. Verify nothing is left (and still billing)

```bash
gcloud container clusters list
gcloud compute instances list
gcloud compute forwarding-rules list            # load balancers
gcloud compute addresses list                   # reserved static IPs (billed when unused!)
gcloud compute disks list                       # leftover PVs with reclaimPolicy Retain
gcloud compute network-endpoint-groups list
gcloud compute firewall-rules list --filter="name~gke-test-cluster"
```
Delete anything orphaned, e.g. `gcloud compute disks delete <disk> --zone us-central1-a`.

## 4. Nuclear option

If the project was only for these labs:
```bash
gcloud projects delete k8s-lab-<random>     # shuts down everything in it
```

Also check **Billing → Reports** in the console a day later.
