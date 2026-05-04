# Chapter 5: Node Selectors and Node Affinity

In this chapter, you'll learn how to control where pods are scheduled using node selectors and node affinity rules. This is essential for scenarios where you need pods to run on nodes with specific characteristics.

## 🎯 Learning Objectives

- Understand node selection mechanisms in Kubernetes
- Use node selectors for simple pod placement
- Configure node affinity for advanced scheduling
- Apply node labels for categorization
- Implement required and preferred scheduling rules
- Compare node selectors vs node affinity

## 📚 Prerequisites

- Completed [Chapter 0: Setup and Prerequisites](../chapter-0-setup/README.md)
- AKS cluster with multiple nodes

## 🔄 Load Your Configuration

```bash
# Load your lab configuration
source ./chapter-0-setup/lab-config.sh

# View cluster nodes
kubectl get nodes -o wide
```

## 📖 Understanding Node Selection

**Node Selector**: Simple key-value label matching for pod placement
**Node Affinity**: Advanced rules with required/preferred constraints

**Use Cases:**
- Run pods on nodes with specific hardware (GPU, SSD)
- Separate workloads by node pools (system vs user)
- Comply with data locality requirements
- Cost optimization (use spot instances selectively)

## 🏷️ Step 1: Label Nodes

```bash
# View existing node labels
kubectl get nodes --show-labels

# Label nodes by environment
NODE1=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
NODE2=$(kubectl get nodes -o jsonpath='{.items[1].metadata.name}')
NODE3=$(kubectl get nodes -o jsonpath='{.items[2].metadata.name}')

kubectl label nodes $NODE1 environment=production
kubectl label nodes $NODE2 environment=development
kubectl label nodes $NODE3 environment=staging

echo "✅ Nodes labeled by environment"

# Label nodes by workload type
kubectl label nodes $NODE1 workload=frontend
kubectl label nodes $NODE2 workload=backend
kubectl label nodes $NODE3 workload=database

echo "✅ Nodes labeled by workload type"

# View updated labels
kubectl get nodes -L environment,workload
```

## 📝 Step 2: Use Node Selector - Simple Example

```bash
# Create namespace
kubectl create namespace node-selector-demo

# Deploy pod with node selector
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-production
  namespace: node-selector-demo
spec:
  nodeSelector:
    environment: production
  containers:
  - name: nginx
    image: nginx:alpine
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
EOF

echo "✅ Pod with nodeSelector deployed"
```

## 🔍 Step 3: Verify Pod Placement

```bash
# Check which node the pod is scheduled on
kubectl get pod nginx-production -n node-selector-demo -o wide

# Verify it's on the production node
kubectl get pod nginx-production -n node-selector-demo \
  -o jsonpath='{.spec.nodeName}{"\n"}'

# Check node labels
kubectl get node $NODE1 --show-labels | grep environment
```

## 🎯 Step 4: Node Affinity - Required Rules

```bash
# Deploy pod with required node affinity
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-app
  namespace: node-selector-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: workload
                operator: In
                values:
                - backend
                - database
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
EOF

echo "✅ Deployment with required node affinity deployed"
```

## 📊 Step 5: Verify Affinity Placement

```bash
# Check pod distribution
kubectl get pods -n node-selector-demo -l app=backend -o wide

# Count pods per node
echo "=== Pod distribution ==="
for node in $NODE1 $NODE2 $NODE3; do
  count=$(kubectl get pods -n node-selector-demo -l app=backend \
    --field-selector spec.nodeName=$node --no-headers 2>/dev/null | wc -l)
  labels=$(kubectl get node $node -o jsonpath='{.metadata.labels.workload}')
  echo "Node $node (workload=$labels): $count pods"
done
```

## 🌟 Step 6: Node Affinity - Preferred Rules

```bash
# Deploy with preferred node affinity
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-app
  namespace: node-selector-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: workload
                operator: In
                values:
                - frontend
          - weight: 50
            preference:
              matchExpressions:
              - key: environment
                operator: In
                values:
                - production
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
EOF

echo "✅ Deployment with preferred node affinity deployed"
```

## 🔍 Step 7: Analyze Preferred Placement

```bash
# Check where frontend pods are scheduled
kubectl get pods -n node-selector-demo -l app=frontend -o wide

# Show preference weights working
echo "=== Frontend pod distribution (preferred scheduling) ==="
for node in $NODE1 $NODE2 $NODE3; do
  count=$(kubectl get pods -n node-selector-demo -l app=frontend \
    --field-selector spec.nodeName=$node --no-headers 2>/dev/null | wc -l)
  workload=$(kubectl get node $node -o jsonpath='{.metadata.labels.workload}')
  env=$(kubectl get node $node -o jsonpath='{.metadata.labels.environment}')
  echo "Node $node (workload=$workload, env=$env): $count pods"
done
```

## 🔄 Step 8: Combined Required and Preferred

```bash
# Deploy with both required and preferred affinity
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-app
  namespace: node-selector-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: environment
                operator: In
                values:
                - production
                - staging
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: workload
                operator: In
                values:
                - database
      containers:
      - name: postgres
        image: postgres:15-alpine
        env:
        - name: POSTGRES_PASSWORD
          value: "demopassword"
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
EOF

echo "✅ Deployment with combined affinity deployed"
```

## 🧪 Step 9: Test Node Selector Failure Scenario

```bash
# Try to deploy on non-existent label (will remain Pending)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: impossible-pod
  namespace: node-selector-demo
spec:
  nodeSelector:
    hardware: gpu-node
  containers:
  - name: nginx
    image: nginx:alpine
EOF

# Check pod status (should be Pending)
sleep 5
kubectl get pod impossible-pod -n node-selector-demo

# Describe to see why it's pending
kubectl describe pod impossible-pod -n node-selector-demo | grep -A 5 Events

# Cleanup
kubectl delete pod impossible-pod -n node-selector-demo
```

## 📋 Step 10: Advanced: Multiple Expressions

```bash
# Deploy with multiple match expressions
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: complex-affinity-pod
  namespace: node-selector-demo
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
          - key: environment
            operator: NotIn
            values:
            - development
        - matchExpressions:
          - key: workload
            operator: Exists
  containers:
  - name: nginx
    image: nginx:alpine
EOF

echo "✅ Pod with complex affinity rules deployed"

# Check placement
kubectl get pod complex-affinity-pod -n node-selector-demo -o wide
```

## 📊 Step 11: Node Affinity Operators

```bash
# Demonstrate different operators
echo "=== Node Affinity Operators ==="

# In operator
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: operator-in
  namespace: node-selector-demo
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: environment
            operator: In
            values:
            - production
            - staging
  containers:
  - name: nginx
    image: nginx:alpine
EOF

# NotIn operator
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: operator-notin
  namespace: node-selector-demo
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: environment
            operator: NotIn
            values:
            - development
  containers:
  - name: nginx
    image: nginx:alpine
EOF

# Exists operator
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: operator-exists
  namespace: node-selector-demo
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: workload
            operator: Exists
  containers:
  - name: nginx
    image: nginx:alpine
EOF

# Check all pods
sleep 5
kubectl get pods -n node-selector-demo -o wide | grep operator
```

## 🎓 Summary

You have successfully:
- ✅ Applied custom labels to nodes
- ✅ Used node selectors for simple pod placement
- ✅ Configured required node affinity rules
- ✅ Implemented preferred node affinity with weights
- ✅ Combined required and preferred affinity
- ✅ Tested various affinity operators
- ✅ Analyzed pod placement decisions

## 🔑 Key Concepts Learned

1. **Node Selector**: Simple label-based scheduling
2. **Required Affinity**: Hard constraints (must match)
3. **Preferred Affinity**: Soft constraints (best effort)
4. **Weight**: Priority for preferred rules (1-100)
5. **Operators**: In, NotIn, Exists, DoesNotExist, Gt, Lt

## 📊 Node Selector vs Node Affinity

| Feature             | Node Selector    | Node Affinity             |
| ------------------- | ---------------- | ------------------------- |
| **Syntax**          | Simple key-value | Complex expressions       |
| **Operators**       | Equality only    | Multiple operators        |
| **Required Rules**  | Yes              | Yes                       |
| **Preferred Rules** | No               | Yes                       |
| **Multiple Terms**  | AND only         | OR with terms, AND within |
| **Use Case**        | Simple scenarios | Complex requirements      |

## 📝 Best Practices

- ✅ Use node selectors for simple, strict requirements
- ✅ Use node affinity for complex scheduling logic
- ✅ Combine required and preferred affinity for flexibility
- ✅ Use meaningful, hierarchical label names
- ✅ Document node labeling strategy
- ✅ Test with replica counts > node count
- ✅ Monitor pod scheduling events

## 🔍 Common Patterns

**Dedicated Nodes:**
```yaml
nodeSelector:
  workload-type: batch-processing
```

**Hardware Requirements:**
```yaml
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
    - matchExpressions:
      - key: node.kubernetes.io/instance-type
        operator: In
        values:
        - Standard_D4s_v3
        - Standard_D8s_v3
```

**Avoid Specific Nodes:**
```yaml
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
    - matchExpressions:
      - key: node-role
        operator: NotIn
        values:
        - system
```

## 🧹 Cleanup

```bash
# Delete deployments and pods
kubectl delete namespace node-selector-demo

# Remove custom labels from nodes
kubectl label nodes $NODE1 environment- workload-
kubectl label nodes $NODE2 environment- workload-
kubectl label nodes $NODE3 environment- workload-

echo "✅ Node selector demo resources cleaned up"
```

## 🚀 Next Steps

Proceed to:
- **[Chapter 6: Taints and Tolerations](../chapter-6-taints-tolerations/README.md)**

In the next chapter, you'll learn the opposite approach: how to repel pods from nodes using taints and tolerations.
