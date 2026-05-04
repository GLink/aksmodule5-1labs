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

```bash
# Load your lab configuration
source ./chapter-0-setup/lab-config.sh

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

```bash
# Check existing taints (system nodes may have taints)
kubectl get nodes -o json | jq '.items[].spec.taints'

# Or use describe
kubectl describe nodes | grep -A 5 Taints

# Get node names
NODE1=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
NODE2=$(kubectl get nodes -o jsonpath='{.items[1].metadata.name}')
NODE3=$(kubectl get nodes -o jsonpath='{.items[2].metadata.name}')

echo "Node 1: $NODE1"
echo "Node 2: $NODE2"
echo "Node 3: $NODE3"
```

## 🚫 Step 2: Apply Taints to Nodes

```bash
# Taint node1 for production workloads only (NoSchedule)
kubectl taint nodes $NODE1 environment=production:NoSchedule

# Taint node2 for high-memory workloads (PreferNoSchedule)
kubectl taint nodes $NODE2 workload=high-memory:PreferNoSchedule

# Taint node3 for special hardware (NoExecute - will evict pods!)
kubectl taint nodes $NODE3 dedicated=special-hardware:NoExecute

echo "✅ Taints applied to nodes"

# Verify taints
kubectl describe node $NODE1 | grep Taints
kubectl describe node $NODE2 | grep Taints
kubectl describe node $NODE3 | grep Taints
```

## 📝 Step 3: Deploy Pod Without Toleration (Will Fail on Tainted Nodes)

```bash
# Create namespace
kubectl create namespace taints-demo

# Deploy pod without toleration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-toleration-pod
  namespace: taints-demo
spec:
  containers:
  - name: nginx
    image: nginx:alpine
EOF

# Check which node it landed on (should not be node1 or node3)
sleep 5
kubectl get pod no-toleration-pod -n taints-demo -o wide

# Try to force it on tainted node with nodeSelector (will stay Pending)
cat <<EOF | kubectl apply -f -
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
EOF

sleep 5
kubectl get pod pending-pod -n taints-demo
kubectl describe pod pending-pod -n taints-demo | grep -A 3 Events
```

## ✅ Step 4: Deploy Pod With Toleration

```bash
# Deploy pod that tolerates the production taint
cat <<EOF | kubectl apply -f -
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
EOF

echo "✅ Pod with toleration deployed"

# Verify it can be scheduled on node1
sleep 5
kubectl get pod production-toleration-pod -n taints-demo -o wide
```

## 🎯 Step 5: Different Toleration Operators

```bash
# Toleration with "Exists" operator (tolerates any value)
cat <<EOF | kubectl apply -f -
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
EOF

echo "✅ Deployment with flexible tolerations created"

# Check pod distribution
sleep 10
kubectl get pods -n taints-demo -l app=flexible -o wide
```

## 🔄 Step 6: Tolerate All Taints

```bash
# Deploy pod that tolerates everything
cat <<EOF | kubectl apply -f -
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
EOF

echo "✅ Pod that tolerates all taints deployed"

# This pod can be scheduled anywhere
kubectl get pod tolerate-all-pod -n taints-demo -o wide
```

## ⏰ Step 7: Toleration with Timeout (NoExecute)

```bash
# Remove NoExecute taint temporarily for this test
kubectl taint nodes $NODE3 dedicated=special-hardware:NoExecute-

# Deploy pods that will be evicted after timeout
cat <<EOF | kubectl apply -f -
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
EOF

# Wait for pods to start
kubectl wait --for=condition=Ready pod -l app=temporary -n taints-demo --timeout=60s

# Now add NoExecute taint
kubectl taint nodes $NODE3 dedicated=special-hardware:NoExecute

# Pods without toleration will be evicted immediately
echo "⏳ Watching pods get evicted..."
kubectl get pods -n taints-demo -l app=temporary -w
```

## 🛡️ Step 8: Tolerate NoExecute with Grace Period

```bash
# Deploy with NoExecute toleration and grace period
cat <<EOF | kubectl apply -f -
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
EOF

echo "✅ Deployment with NoExecute toleration (60s grace) deployed"

# Pods will stay for 60 seconds after taint is applied
kubectl get pods -n taints-demo -l app=graceful -o wide
```

## 🏗️ Step 9: Dedicated Node Pool Pattern

```bash
# Simulate dedicated GPU node pool pattern
kubectl label node $NODE2 node-pool=gpu-pool
kubectl taint nodes $NODE2 nvidia.com/gpu=present:NoSchedule

# Deploy GPU workload (simulated)
cat <<EOF | kubectl apply -f -
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
EOF

echo "✅ GPU workload deployed on dedicated node"

kubectl get pods -n taints-demo -l app=gpu-app -o wide
```

## 🔧 Step 10: Node Maintenance Pattern

```bash
# Cordon node (prevent new pods)
kubectl cordon $NODE1

# Add taint for maintenance
kubectl taint nodes $NODE1 maintenance=true:NoExecute

echo "✅ Node $NODE1 cordoned and tainted for maintenance"

# Watch pods get evicted from node1
kubectl get pods -n taints-demo -o wide | grep $NODE1

# After maintenance, uncordon and remove taint
echo "Simulating maintenance complete..."
sleep 10
kubectl uncordon $NODE1
kubectl taint nodes $NODE1 maintenance=true:NoExecute-

echo "✅ Node $NODE1 back in service"
```

## 📊 Step 11: View All Taints and Tolerations

```bash
# View all node taints
echo "=== Node Taints ==="
kubectl get nodes -o json | jq -r '.items[] | "\(.metadata.name): \(.spec.taints)"'

# View pod tolerations
echo -e "\n=== Pod Tolerations ==="
kubectl get pods -n taints-demo -o json | \
  jq -r '.items[] | "\(.metadata.name): \(.spec.tolerations)"'

# Count pods per node
echo -e "\n=== Pods per Node ==="
for node in $NODE1 $NODE2 $NODE3; do
  count=$(kubectl get pods -n taints-demo --field-selector spec.nodeName=$node --no-headers 2>/dev/null | wc -l)
  echo "$node: $count pods"
done
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

```bash
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

echo "✅ Taints and tolerations demo cleaned up"
```

## 🚀 Next Steps

Proceed to:
- **[Chapter 7: Pod Affinity and Anti-Affinity](../chapter-7-pod-affinity/README.md)**

In the next chapter, you'll learn how to control pod placement relative to other pods, not just nodes.
