---
layout: default
---

## Migrating from Karpenter to EKS Auto Mode: Lessons Learned

A practical guide to moving workloads from Karpenter to Amazon EKS Auto Mode—why you might do it, how to run both side-by-side, and what we learned along the way.

If you’ve been running [Karpenter](https://karpenter.sh/) for node autoscaling (as in [Scaling with Karpenter in Kubernetes](./karpenter_scaling.html)), EKS Auto Mode can look attractive: AWS manages compute, storage, and networking in one place, with built-in security and cost optimization. Migrating doesn’t have to be a big-bang cutover. Here’s a concise path and some lessons learned.

---

### Why Consider Moving to EKS Auto Mode?

| Aspect | Karpenter | EKS Auto Mode |
|--------|-----------|----------------|
| **Who runs it** | You (Helm, CRDs, upgrades) | AWS (managed control plane + node lifecycle) |
| **Node lifecycle** | You configure NodePools, EC2NodeClass | AWS-managed; immutable nodes, auto-rotation of nodes |
| **Security** | You harden AMIs, IAM, networking | Built-in: immutable AMIs, SELinux, read-only root, encryption |
| **Networking / storage** | You wire VPC CNI, EBS CSI | Pre-integrated VPC CNI, EBS CSI |
| **Cost optimization** | Spot, consolidation, right-sizing in your NodePools | AWS applies Spot, right-sizing, instance selection |
| **Operational load** | Higher (you own scaling logic and upgrades) | Lower (fewer components to operate) |

Auto Mode fits teams that want less to run themselves and are okay with fewer knobs than Karpenter’s fine-grained NodePools and instance constraints.

---

### Prerequisites

- **Karpenter v1.1+** on the cluster (required for the migration path below).
- **kubectl** and cluster access.
- **EKS cluster** on a supported Kubernetes version (1.29+ for Auto Mode).

---

### Step 1: Enable EKS Auto Mode (Without the Default Node Pool)

Enable Auto Mode on the existing cluster but **do not** turn on the built-in general-purpose node pool yet. That keeps existing workloads on Karpenter until you explicitly move them.

**Terraform example:**

```hcl
resource "aws_eks_cluster" "main" {
  name     = "my-cluster"
  role_arn = aws_iam_role.eks_cluster.arn
  version  = "1.29"

  vpc_config {
    subnet_ids = var.subnet_ids
  }

  # Enable EKS Auto Mode; do not enable default node pool during migration
  auto_mode_config {
    enabled = true
  }
}
```

**AWS CLI alternative:**

```bash
aws eks update-cluster-config \
  --name my-cluster \
  --auto-mode '{"enabled":true}'
```

During migration, leave the built-in general-purpose node pool **disabled** so only your new, tainted Auto Mode Node Pool receives migrated workloads.

---

### Step 2: Create a Tainted EKS Auto Mode Node Pool

Create an EKS Auto Mode Node Pool with a taint so existing pods do **not** schedule on it until you add tolerations. This lets you migrate workload-by-workload.

```yaml
# eks-auto-mode-nodepool.yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: eks-auto-mode
spec:
  template:
    spec:
      requirements:
        - key: "eks.amazonaws.com/instance-category"
          operator: In
          values: ["c", "m", "r"]
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: default
      taints:
        - key: "eks-auto-mode"
          effect: "NoSchedule"
```

Adjust `requirements` (e.g. instance categories, size) to align with what you had in Karpenter. You need at least one requirement. Apply:

```bash
kubectl apply -f eks-auto-mode-nodepool.yaml
```

---

### Step 3: Update Workloads for Migration

For each workload you want on Auto Mode, add a **toleration** for the taint and a **node selector** so it only schedules on Auto Mode nodes during the transition.

```yaml
# Deployment patch example
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      tolerations:
        - key: "eks-auto-mode"
          effect: "NoSchedule"
      nodeSelector:
        eks.amazonaws.com/compute-type: auto
      # ... rest of pod spec
```

EKS Auto Mode uses labels under `eks.amazonaws.com` (e.g. `eks.amazonaws.com/compute-type: auto`). This is different from Karpenter’s labels—expect to touch node selectors/affinity when migrating.

---

### Step 4: Migrate Gradually

- Move workloads in batches (e.g. by team or risk): add toleration + node selector, roll out, validate.
- Use Pod Disruption Budgets and graceful shutdowns so node churn doesn’t cause outages.
- Keep an eye on pending pods and node capacity; Auto Mode will provision nodes as needed.

---

### Step 5: Remove the Karpenter Node Pool

After all target workloads run on EKS Auto Mode nodes:

```bash
kubectl delete nodepool <your-original-karpenter-nodepool-name>
```

Karpenter will drain and remove its nodes. Confirm no critical workloads are still on those nodes before deleting.

---

### Step 6: Optional — Make Auto Mode the Default

If you want new workloads to use Auto Mode by default:

1. **Remove the taint** from the EKS Auto Mode Node Pool (remove the `taints` section from the NodePool spec).
2. **Optionally remove** the `nodeSelector` from workloads so they don’t need to explicitly target Auto Mode.

---

### Step 7: Uninstall Karpenter

Once everything runs on Auto Mode and you’ve removed Karpenter Node Pools and EC2 Node Classes, uninstall Karpenter the same way you installed it (e.g. Helm):

```bash
helm uninstall karpenter -n karpenter
```

Clean up CRDs, IAM roles, and any Karpenter-specific resources (queues, instance profiles, etc.) as per [Karpenter docs](https://karpenter.sh/docs/getting-started/getting-started-with-karpenter/#create-a-cluster-and-add-karpenter).

---

### Lessons Learned

1. **Label and taint differences**  
   Auto Mode uses `eks.amazonaws.com/*` labels and its own NodeClass. Plan for a one-time update of node selectors, affinity, and tolerations; a small inventory of “what runs where” helps.

2. **Run both in parallel**  
   Using a tainted Auto Mode Node Pool and migrating in batches reduces risk. You can validate behavior and roll back by reverting tolerations/selectors.

3. **PDBs and graceful shutdowns**  
   Auto Mode still does node replacement and scaling. Strong PDBs and `preStop`/grace periods (as in the [Karpenter article](./karpenter_scaling.html)) remain important.

4. **Less control, less ops**  
   You give up Karpenter’s granular NodePools and instance-type constraints. If you need very specific instance families or consolidation policies, evaluate whether Auto Mode’s defaults are enough before committing.

5. **Cost and observability**  
   Auto Mode optimizes for cost; keep using Cost Explorer and your existing billing alerts. Revisit rightsizing and Spot usage after migration—behavior may differ from your previous Karpenter tuning.

6. **Terraform and GitOps**  
   Model Auto Mode enablement and, where possible, Node Pools in Terraform (or your IaC). Keep workload changes (tolerations, node selectors) in Git for a clear audit trail and rollback.

---

### When to Stay on Karpenter

- You rely on **highly custom** NodePools, instance types, or consolidation behavior.
- You need **workload-specific** node pools (e.g. GPU, ARM, or strict Spot/on-demand splits) that Auto Mode doesn’t yet expose the same way.
- You’re **multi-cloud** or need a **portable** autoscaler; Auto Mode is EKS-only.

---

### Summary

Migrating from Karpenter to EKS Auto Mode is doable without downtime: enable Auto Mode, add a tainted Node Pool, move workloads with tolerations and node selectors, then remove Karpenter Node Pools and uninstall Karpenter. The main trade-off is less control for less operational burden. Document label/taint changes, migrate gradually, and keep PDBs and graceful shutdowns—then you can run EKS Auto Mode with confidence.

[back](../)
