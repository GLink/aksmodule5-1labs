# Chapter 6: Taints and Tolerations

In this chapter, you'll learn how to use taints and tolerations to repel pods from nodes and control which pods can tolerate node conditions. This is the opposite approach from node affinity - instead of attracting pods to nodes, you repel pods unless they explicitly tolerate the taint.

## 🎯 Learning Objectives

- Understand taints and tolerations
- Apply different taint effects
- Create pods that tolerate taints
- Implement dedicated node pools
- Use taints for node maintenance
- Combine taints with node affinity

## 📚 Prerequisites

- Completed [Chapter 0: Setup and Prerequisites](../chapter-0-setup/README.md)
- Understanding of node selectors (Chapter 5 recommended)

## 🔄 Load Your Configuration

```powershell
# Load your lab configuration
. .\psversion\chapter-0-setup\lab-config.ps1

# View cluster nodes
kubectl get nodes
```

## 📖 Understanding Taints and Tolerations

**Taint**: A property of a node that repels pods
**Toleration**: A pod property that allows scheduling on tainted nodes

**Taint Effects:**
- `NoSchedule`: New pods won't be scheduled (existing stay)
- `PreferNoSchedule`: Try to avoid scheduling (soft)
- `NoExecute`: Evict existing pods that don't tolerate

**Use Cases:**
- Dedicate nodes to specific workloads
- Node maintenance and draining
- Hardware-specific workloads (GPU nodes)
- Isolate production from development

## 🏷️ Step 1: View Existing Taints

```powershell
# Check existing taints (system nodes may have taints)
kubectl get nodes -o json | ConvertFrom-Json | Select-Object -ExpandProperty items | ForEach-Object { $_.spec.taints }

# Or use describe
kubectl describe nodes | Select-String Taints -Context 0,5

# Get node names
$NODE1 = $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
$NODE2 = $(kubectl get nodes -o jsonpath='{.items[1].metadata.name}')
$NODE3 = $(kubectl get nodes -o jsonpath='{.items[2].metadata.name}')

Write-Host "Node 1: $NODE1"
Write-Host "Node 2: $NODE2"
Write-Host "Node 3: $NODE3"
```

## 🚫 Step 2: Apply Taints to Nodes

```powershell
# Taint node1 for production workloads only (NoSchedule)
kubectl taint nodes $NODE1 environment=production:NoSchedule

# Taint node2 for high-memory workloads (PreferNoSchedule)
kubectl taint nodes $NODE2 workload=high-memory:PreferNoSchedule

# Taint node3 for special hardware (NoExecute - will evict pods!)
kubectl taint nodes $NODE3 dedicated=special-hardware:NoExecute

Write-Host "✅ Taints applied to nodes"

# Verify taints
kubectl describe node $NODE1 | Select-String Taints
kubectl describe node $NODE2 | Select-String Taints
kubectl describe node $NODE3 | Select-String Taints
```

## 📝 Step 3: Deploy Pod Without Toleration (Will Fail on Tainted Nodes)

```powershell
# Create namespace
kubectl create namespace taints-demo

# Deploy pod without toleration
@"
apiVersion: v1
kind: Pod
metadata:
  name: no-toleration-pod
  namespace: taints-demo
spec:
  containers:
  - name: nginx
    image: nginx:alpine
"@ | kubectl apply -f -

# Check which node it landed on (should not be node1 or node3)
Start-Sleep -Seconds 5
kubectl get pod no-toleration-pod -n taints-demo -o wide

# Try to force it on tainted node with nodeSelector (will stay Pending)
@"
apiVersion: v1
kind: Pod
metadata:
  name: pending-pod
  namespace: taints-demo
spec:
  nodeSelector:
    kubernetes.io/hostname: $NODE1
  containers:
  - name: nginx
    image: nginx:alpine
"@ | kubectl apply -f -

Start-Sleep -Seconds 5
kubectl get pod pending-pod -n taints-demo
kubectl describe pod pending-pod -n taints-demo | Select-String Events -Context 0,3
```

## ✅ Step 4: Deploy Pod With Toleration

```powershell
# Deploy pod that tolerates the production taint
@"
apiVersion: v1
kind: Pod
metadata:
  name: production-toleration-pod
  namespace: taints-demo
spec:
  tolerations:
  - key: "environment"
    operator: "Equal"
    value: "production"
    effect: "NoSchedule"
  nodeSelector:
    kubernetes.io/hostname: $NODE1
  containers:
  - name: nginx
    image: nginx:alpine
"@ | kubectl apply -f -

Write-Host "✅ Pod with toleration deployed"

# Verify it can be scheduled on node1
Start-Sleep -Seconds 5
kubectl get pod production-toleration-pod -n taints-demo -o wide
```

## 🎯 Step 5: Different Toleration Operators

```powershell
# Toleration with "Exists" operator (tolerates any value)
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flexible-tolerations
  namespace: taints-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: flexible
  template:
    metadata:
      labels:
        app: flexible
    spec:
      tolerations:
      - key: "environment"
        operator: "Exists"
        effect: "NoSchedule"
      - key: "workload"
        operator: "Exists"
        effect: "PreferNoSchedule"
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
"@ | kubectl apply -f -

Write-Host "✅ Deployment with flexible tolerations created"

# Check pod distribution
Start-Sleep -Seconds 10
kubectl get pods -n taints-demo -l app=flexible -o wide
```

## 🔄 Step 6: Tolerate All Taints

```powershell
# Deploy pod that tolerates everything
@"
apiVersion: v1
kind: Pod
metadata:
  name: tolerate-all-pod
  namespace: taints-demo
spec:
  tolerations:
  - operator: "Exists"
  containers:
  - name: nginx
    image: nginx:alpine
"@ | kubectl apply -f -

Write-Host "✅ Pod that tolerates all taints deployed"

# This pod can be scheduled anywhere
kubectl get pod tolerate-all-pod -n taints-demo -o wide
```

## ⏰ Step 7: Toleration with Timeout (NoExecute)

```powershell
# Remove NoExecute taint temporarily for this test
kubectl taint nodes $NODE3 dedicated=special-hardware:NoExecute-

# Deploy pods that will be evicted after timeout
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: temporary-workload
  namespace: taints-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: temporary
  template:
    metadata:
      labels:
        app: temporary
    spec:
      nodeSelector:
        kubernetes.io/hostname: $NODE3
      containers:
      - name: nginx
        image: nginx:alpine
"@ | kubectl apply -f -

# Wait for pods to start
kubectl wait --for=condition=Ready pod -l app=temporary -n taints-demo --timeout=60s

# Now add NoExecute taint
kubectl taint nodes $NODE3 dedicated=special-hardware:NoExecute

# Pods without toleration will be evicted immediately
Write-Host "⏳ Watching pods get evicted..."
kubectl get pods -n taints-demo -l app=temporary -w
```

## 🛡️ Step 8: Tolerate NoExecute with Grace Period

```powershell
# Deploy with NoExecute toleration and grace period
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: graceful-eviction
  namespace: taints-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: graceful
  template:
    metadata:
      labels:
        app: graceful
    spec:
      tolerations:
      - key: "dedicated"
        operator: "Equal"
        value: "special-hardware"
        effect: "NoExecute"
        tolerationSeconds: 60
      nodeSelector:
        kubernetes.io/hostname: $NODE3
      containers:
      - name: nginx
        image: nginx:alpine
"@ | kubectl apply -f -

Write-Host "✅ Deployment with NoExecute toleration (60s grace) deployed"

# Pods will stay for 60 seconds after taint is applied
kubectl get pods -n taints-demo -l app=graceful -o wide
```

## 🏗️ Step 9: Dedicated Node Pool Pattern

```powershell
# Simulate dedicated GPU node pool pattern
kubectl label node $NODE2 node-pool=gpu-pool
kubectl taint nodes $NODE2 nvidia.com/gpu=present:NoSchedule

# Deploy GPU workload (simulated)
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-workload
  namespace: taints-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: gpu-app
  template:
    metadata:
      labels:
        app: gpu-app
    spec:
      tolerations:
      - key: "nvidia.com/gpu"
        operator: "Equal"
        value: "present"
        effect: "NoSchedule"
      nodeSelector:
        node-pool: gpu-pool
      containers:
      - name: cuda-app
        image: nginx:alpine
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
"@ | kubectl apply -f -

Write-Host "✅ GPU workload deployed on dedicated node"

kubectl get pods -n taints-demo -l app=gpu-app -o wide
```

## 🔧 Step 10: Node Maintenance Pattern

```powershell
# Cordon node (prevent new pods)
kubectl cordon $NODE1

# Add taint for maintenance
kubectl taint nodes $NODE1 maintenance=true:NoExecute

Write-Host "✅ Node $NODE1 cordoned and tainted for maintenance"

# Watch pods get evicted from node1
kubectl get pods -n taints-demo -o wide | Select-String $NODE1

# After maintenance, uncordon and remove taint
Write-Host "Simulating maintenance complete..."
Start-Sleep -Seconds 10
kubectl uncordon $NODE1
kubectl taint nodes $NODE1 maintenance=true:NoExecute-

Write-Host "✅ Node $NODE1 back in service"
```

## 📊 Step 11: View All Taints and Tolerations

```powershell
# View all node taints
Write-Host "=== Node Taints ==="
kubectl get nodes -o json | ConvertFrom-Json | Select-Object -ExpandProperty items | ForEach-Object { "$($_.metadata.name): $($_.spec.taints | ConvertTo-Json -Compress)" }

# View pod tolerations
Write-Host "`n=== Pod Tolerations ==="
kubectl get pods -n taints-demo -o json | `
  ConvertFrom-Json | Select-Object -ExpandProperty items | ForEach-Object { "$($_.metadata.name): $($_.spec.tolerations | ConvertTo-Json -Compress)" }

# Count pods per node
Write-Host "`n=== Pods per Node ==="
$nodes = @($NODE1, $NODE2, $NODE3)
foreach ($node in $nodes) {
  $count = @(kubectl get pods -n taints-demo --field-selector spec.nodeName=$node --no-headers 2>$null).Count
  Write-Host "$node: $count pods"
}
```

## 🎓 Summary

You have successfully:
- ✅ Applied taints with different effects to nodes
- ✅ Created pods with tolerations
- ✅ Used different toleration operators (Equal, Exists)
- ✅ Implemented NoExecute with grace periods
- ✅ Created dedicated node pools
- ✅ Performed node maintenance with taints
- ✅ Combined taints with node selectors

## 🔑 Key Concepts Learned

1. **NoSchedule**: Prevents new pods without toleration
2. **PreferNoSchedule**: Soft constraint (try to avoid)
3. **NoExecute**: Evicts existing pods
4. **tolerationSeconds**: Grace period before eviction
5. **Operators**: Equal (exact match), Exists (any value)
6. **Cordon**: Prevents scheduling without eviction

## 📊 Taint Effects Comparison

| Effect               | New Pods | Existing Pods | Use Case              |
| -------------------- | -------- | ------------- | --------------------- |
| **NoSchedule**       | Blocked  | Stay          | Dedicated nodes       |
| **PreferNoSchedule** | Avoided  | Stay          | Soft preferences      |
| **NoExecute**        | Blocked  | Evicted       | Maintenance, failures |

## 📝 Best Practices

- ✅ Use NoSchedule for dedicated workloads
- ✅ Use PreferNoSchedule for soft preferences
- ✅ Use NoExecute cautiously (evicts pods!)
- ✅ Set tolerationSeconds for graceful eviction
- ✅ Combine taints with node labels
- ✅ Document taint policies clearly
- ✅ Use cordon before NoExecute for maintenance
- ✅ Test taints in non-production first

## 🔍 Common Patterns

**Dedicated Nodes:**
```yaml
# Node taint
kubectl taint nodes <node> workload=special:NoSchedule

# Pod toleration
tolerations:
- key: "workload"
  operator: "Equal"
  value: "special"
  effect: "NoSchedule"
```

**Temporary Toleration:**
```yaml
tolerations:
- key: "maintenance"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300
```

**Tolerate Everything:**
```yaml
tolerations:
- operator: "Exists"
```

## 🧹 Cleanup

```powershell
# Delete namespace
kubectl delete namespace taints-demo

# Remove taints
kubectl taint nodes $NODE1 environment=production:NoSchedule-
kubectl taint nodes $NODE2 workload=high-memory:PreferNoSchedule-
kubectl taint nodes $NODE2 nvidia.com/gpu=present:NoSchedule-
kubectl taint nodes $NODE3 dedicated=special-hardware:NoExecute-

# Remove labels
kubectl label node $NODE2 node-pool-

# Ensure all nodes are uncordoned
kubectl uncordon $NODE1 $NODE2 $NODE3

Write-Host "✅ Taints and tolerations demo cleaned up"
```

## 🚀 Next Steps

Proceed to:
- **[Chapter 7: Pod Affinity and Anti-Affinity](../chapter-7-pod-affinity/README.md)**

In the next chapter, you'll learn how to control pod placement relative to other pods, not just nodes.
