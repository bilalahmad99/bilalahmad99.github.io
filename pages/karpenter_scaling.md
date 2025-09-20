---
layout: default
---

## Scaling with Karpenter in Kubernetes

A guide to how we used Karpenter's node autoscaling capabilities, covering provisioning strategies, cost optimization, and operational best practices.

### What is Karpenter?

Karpenter is an open-source Kubernetes cluster autoscaler that provisions right-sized compute resources in response to changing application load. They have better performance and flexibilty compare with traditional cluster autoscalers.

### Key Advantages over Cluster Autoscaler

- **Faster scaling**: Provisions nodes in ~30 seconds vs 2-3 minutes
- **Better bin packing**: Optimizes for multiple instance types simultaneously
- **Cost optimization**: Automatically chooses spot instances and right-sized nodes
- **Simplified configuration**: No need to manage Auto Scaling Groups

### Installation & Setup

```bash
# Install Karpenter via Helm
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version ${KARPENTER_VERSION} \
  --namespace karpenter \
  --create-namespace \
  --set settings.aws.clusterName=${CLUSTER_NAME} \
  --set settings.aws.defaultInstanceProfile=KarpenterNodeInstanceProfile \
  --set settings.aws.interruptionQueueName=${CLUSTER_NAME} \
  --wait
```

### Basic Node Pool Configuration

```yaml
# nodepool.yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  # Template for nodes
  template:
    metadata:
      labels:
        node-type: "karpenter"
    spec:
      # Instance requirements
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["m5.large", "m5.xlarge", "m5.2xlarge"]
      
      # Node properties
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: default
      
      # Taints for workload isolation
      taints:
        - key: "workload-type"
          value: "general"
          effect: NoSchedule
  
  # Limits and disruption settings
  limits:
    cpu: 1000
    memory: 1000Gi
  
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
    expireAfter: 30m
---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  # AMI selection
  amiFamily: AL2
  
  # Subnet and security group selection
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}"
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}"
  
  # Instance profile
  instanceProfile: "KarpenterNodeInstanceProfile"
  
  # User data for node bootstrapping
  userData: |
    #!/bin/bash
    /etc/eks/bootstrap.sh ${CLUSTER_NAME}
    
    # Custom configurations
    echo 'vm.max_map_count=262144' >> /etc/sysctl.conf
    sysctl -p
```

### Workload-Specific Node Pools

#### Compute-Intensive Workloads
```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: compute-intensive
spec:
  template:
    metadata:
      labels:
        workload-type: "compute"
    spec:
      requirements:
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["c5.large", "c5.xlarge", "c5.2xlarge", "c5.4xlarge"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
      
      nodeClassRef:
        name: compute-optimized
      
      taints:
        - key: "workload-type"
          value: "compute"
          effect: NoSchedule
  
  limits:
    cpu: 500
  
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 10s
```

#### Memory-Intensive Workloads
```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: memory-intensive
spec:
  template:
    metadata:
      labels:
        workload-type: "memory"
    spec:
      requirements:
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["r5.large", "r5.xlarge", "r5.2xlarge"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
      
      nodeClassRef:
        name: memory-optimized
      
      taints:
        - key: "workload-type"
          value: "memory"
          effect: NoSchedule
  
  limits:
    memory: 500Gi
```

### Cost Optimization Strategies

#### Spot Instance Configuration
```yaml
# Prioritize spot instances for cost savings
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot", "on-demand"]  # spot preferred
    - key: node.kubernetes.io/instance-type
      operator: In
      values: ["m5.large", "m5.xlarge", "m4.large", "m4.xlarge"]
  
  # Enable consolidation for better utilization
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
```

#### Right-sizing with Multiple Instance Types
```yaml
# Allow Karpenter to choose optimal instance types
spec:
  requirements:
    - key: node.kubernetes.io/instance-type
      operator: In
      values: [
        "m5.large", "m5.xlarge", "m5.2xlarge",
        "m4.large", "m4.xlarge", "m4.2xlarge",
        "c5.large", "c5.xlarge", "c5.2xlarge"
      ]
```

### Application Scheduling Examples

#### Deployment with Node Affinity
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: compute-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: compute-app
  template:
    metadata:
      labels:
        app: compute-app
    spec:
      # Tolerate the compute workload taint
      tolerations:
        - key: "workload-type"
          operator: "Equal"
          value: "compute"
          effect: "NoSchedule"
      
      # Prefer compute-optimized nodes
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: workload-type
                    operator: In
                    values: ["compute"]
      
      containers:
        - name: app
          image: my-compute-app:latest
          resources:
            requests:
              cpu: "1000m"
              memory: "2Gi"
            limits:
              cpu: "2000m"
              memory: "4Gi"
```

#### Pod Disruption Budget
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: compute-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: compute-app
```

### Monitoring & Observability

#### Key Metrics to Monitor
```yaml
# Prometheus alerts for Karpenter
groups:
  - name: karpenter
    rules:
      - alert: KarpenterNodePoolLimitReached
        expr: karpenter_nodepool_limit_cpu_cores - karpenter_nodepool_usage_cpu_cores < 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Karpenter NodePool approaching CPU limit"
      
      - alert: KarpenterHighSpotInterruption
        expr: rate(karpenter_interruption_actions_performed_total[5m]) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High spot instance interruption rate"
```

#### Grafana Dashboard Queries
```promql
# Node provisioning rate
rate(karpenter_nodes_created_total[5m])

# Node utilization
karpenter_nodepool_usage_cpu_cores / karpenter_nodepool_limit_cpu_cores

# Cost savings from spot instances
(karpenter_nodepool_usage_spot_capacity_cores / karpenter_nodepool_usage_total_capacity_cores) * 100
```

### Best Practices

#### 1. NodePool Design
- Create separate NodePools for different workload types
- Use appropriate limits to prevent runaway scaling
- Configure consolidation policies for cost optimization

#### 2. Resource Management
```yaml
# Always set resource requests
containers:
  - name: app
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
```

#### 3. Spot Instance Handling
```yaml
# Use pod disruption budgets
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: "50%"
  selector:
    matchLabels:
      app: my-app
```

#### 4. Graceful Shutdowns
```yaml
# Configure termination grace period
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - name: app
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 10"]
```

### Troubleshooting Common Issues

#### 1. Nodes Not Scaling
- Check NodePool resource limits
- Verify subnet and security group tags
- Review pod resource requests

#### 2. High Costs
- Enable consolidation policies
- Use spot instances where appropriate
- Right-size resource requests

#### 3. Pod Scheduling Issues
- Check node selectors and affinity rules
- Verify toleration configurations
- Review resource constraints

### Migration from Cluster Autoscaler

```bash
# 1. Install Karpenter alongside CA
helm install karpenter ...

# 2. Create NodePools matching CA node groups
kubectl apply -f nodepool.yaml

# 3. Gradually migrate workloads
kubectl label nodes <node-name> karpenter.sh/managed-by=karpenter

# 4. Remove CA after validation
kubectl delete deployment cluster-autoscaler -n kube-system
```

### Performance Comparison

| Metric | Cluster Autoscaler | Karpenter |
|--------|-------------------|-----------|
| **Scale-up time** | 2-5 minutes | 30-60 seconds |
| **Instance diversity** | Single type per ASG | Multiple types |
| **Bin packing** | Basic | Advanced |
| **Spot handling** | Manual ASG config | Automatic |
| **Configuration** | Complex ASGs | Simple NodePools |

[back](../)
