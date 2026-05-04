# Chapter 7: Pod Affinity and Anti-Affinity

In this chapter, you'll learn how to use pod affinity and anti-affinity to control pod placement relative to other pods. This is useful for co-locating related services or spreading replicas for high availability.

## 🎯 Learning Objectives

- Understand pod affinity and anti-affinity
- Co-locate related pods with pod affinity
- Spread pods across nodes with anti-affinity
- Use required and preferred rules
- Implement high availability patterns
- Choose appropriate topology keys

## 📚 Prerequisites

- Completed [Chapter 0: Setup and Prerequisites](../chapter-0-setup/README.md)
- Understanding of node affinity (Chapter 5 recommended)

## 🔄 Load Your Configuration

```bash
# Load your lab configuration
source ./chapter-0-setup/lab-config.sh

# View cluster nodes and their zone labels
kubectl get nodes -L topology.kubernetes.io/zone
```

## 📖 Understanding Pod Affinity/Anti-Affinity

**Pod Affinity**: Schedule pods near other pods (co-location)
**Pod Anti-Affinity**: Schedule pods away from other pods (spreading)

**Use Cases:**
- **Affinity**: Co-locate frontend with cache, app with sidecar
- **Anti-Affinity**: Spread replicas for HA, avoid noisy neighbors

**Topology Keys:**
- `kubernetes.io/hostname`: Per-node
- `topology.kubernetes.io/zone`: Per-availability zone
- Custom labels: Per rack, region, etc.

## 🚀 Step 1: Deploy Base Application

```bash
# Create namespace
kubectl create namespace pod-affinity-demo

# Deploy a cache service (redis)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
  namespace: pod-affinity-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: redis
      tier: cache
  template:
    metadata:
      labels:
        app: redis
        tier: cache
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: pod-affinity-demo
spec:
  selector:
    app: redis
  ports:
  - port: 6379
EOF

echo "✅ Redis cache deployed"

# Wait for redis to be ready
kubectl wait --for=condition=Ready pod -l app=redis -n pod-affinity-demo --timeout=60s

# Check where redis pods are running
kubectl get pods -n pod-affinity-demo -l app=redis -o wide
```

## 💕 Step 2: Pod Affinity - Co-locate with Redis

```bash
# Deploy web app with pod affinity to redis
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: pod-affinity-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
      tier: frontend
  template:
    metadata:
      labels:
        app: web
        tier: frontend
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - redis
            topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
EOF

echo "✅ Web app with pod affinity deployed"

# Verify web pods are on same nodes as redis
sleep 10
kubectl get pods -n pod-affinity-demo -o wide
```

## 🔍 Step 3: Analyze Co-location

```bash
# Show which pods are on which nodes
echo "=== Pod Distribution ==="
kubectl get pods -n pod-affinity-demo -o custom-columns=\
NAME:.metadata.name,\
NODE:.spec.nodeName,\
APP:.metadata.labels.app,\
TIER:.metadata.labels.tier

# Count co-located pods
echo -e "\n=== Co-location Analysis ==="
for node in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
  redis_count=$(kubectl get pods -n pod-affinity-demo -l app=redis \
    --field-selector spec.nodeName=$node --no-headers 2>/dev/null | wc -l)
  web_count=$(kubectl get pods -n pod-affinity-demo -l app=web \
    --field-selector spec.nodeName=$node --no-headers 2>/dev/null | wc -l)
  if [ $redis_count -gt 0 ] || [ $web_count -gt 0 ]; then
    echo "$node: Redis=$redis_count, Web=$web_count"
  fi
done
```

## 🚫 Step 4: Pod Anti-Affinity - Spread Replicas

```bash
# Deploy database with anti-affinity for HA
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-db
  namespace: pod-affinity-demo
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
      tier: database
  template:
    metadata:
      labels:
        app: postgres
        tier: database
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - postgres
            topologyKey: kubernetes.io/hostname
      containers:
      - name: postgres
        image: postgres:15-alpine
        env:
        - name: POSTGRES_PASSWORD
          value: "demo-password"
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        ports:
        - containerPort: 5432
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
EOF

echo "✅ PostgreSQL with anti-affinity deployed"

# Wait and check distribution
sleep 15
kubectl get pods -n pod-affinity-demo -l app=postgres -o wide
```

## 📊 Step 5: Verify Anti-Affinity Spreading

```bash
# Verify each postgres pod is on a different node
echo "=== PostgreSQL Pod Distribution (should be on different nodes) ==="
kubectl get pods -n pod-affinity-demo -l app=postgres -o custom-columns=\
NAME:.metadata.name,NODE:.spec.nodeName,STATUS:.status.phase

# Count pods per node
echo -e "\n=== Pods per Node ==="
for node in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
  count=$(kubectl get pods -n pod-affinity-demo -l app=postgres \
    --field-selector spec.nodeName=$node --no-headers 2>/dev/null | wc -l)
  if [ $count -gt 0 ]; then
    echo "$node: $count postgres pod(s)"
  fi
done
```

## 🌟 Step 6: Preferred Pod Affinity

```bash
# Deploy analytics service with preferred affinity to database
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: analytics-service
  namespace: pod-affinity-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: analytics
  template:
    metadata:
      labels:
        app: analytics
    spec:
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - postgres
              topologyKey: kubernetes.io/hostname
      containers:
      - name: analytics
        image: nginx:alpine
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
EOF

echo "✅ Analytics service with preferred affinity deployed"

sleep 10
kubectl get pods -n pod-affinity-demo -l app=analytics -o wide
```

## 🔄 Step 7: Combined Affinity and Anti-Affinity

```bash
# Deploy service with both affinity and anti-affinity
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  namespace: pod-affinity-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
      tier: backend
  template:
    metadata:
      labels:
        app: api
        tier: backend
    spec:
      affinity:
        # Co-locate with redis for performance
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - redis
              topologyKey: kubernetes.io/hostname
        # Spread API replicas across nodes for HA
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - api
              topologyKey: kubernetes.io/hostname
      containers:
      - name: api
        image: nginx:alpine
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
EOF

echo "✅ API service with combined affinity rules deployed"

sleep 10
kubectl get pods -n pod-affinity-demo -l app=api -o wide
```

## 🌍 Step 8: Zone-Level Anti-Affinity

```bash
# Deploy service spread across availability zones
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-service
  namespace: pod-affinity-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: critical
  template:
    metadata:
      labels:
        app: critical
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - critical
            topologyKey: topology.kubernetes.io/zone
      containers:
      - name: app
        image: nginx:alpine
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
EOF

echo "✅ Critical service with zone anti-affinity deployed"

# Verify pods are in different zones
sleep 10
kubectl get pods -n pod-affinity-demo -l app=critical -o wide
kubectl get pods -n pod-affinity-demo -l app=critical \
  -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,\
ZONE:.spec.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[0].matchExpressions[0].values[0]
```

## 🧪 Step 9: Test Scheduling Constraints

```bash
# Try to deploy more replicas than nodes (with required anti-affinity)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: over-constrained
  namespace: pod-affinity-demo
spec:
  replicas: 10
  selector:
    matchLabels:
      app: over-constrained
  template:
    metadata:
      labels:
        app: over-constrained
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - over-constrained
            topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx:alpine
EOF

# Some pods will remain Pending (not enough nodes)
sleep 10
kubectl get pods -n pod-affinity-demo -l app=over-constrained

# Describe pending pods
kubectl describe pod -n pod-affinity-demo -l app=over-constrained | \
  grep -A 5 "Events:" | head -20
```

## 📋 Step 10: View All Affinity Rules

```bash
# List all deployments with their affinity rules
echo "=== Deployment Affinity Rules ==="
kubectl get deployments -n pod-affinity-demo -o custom-columns=\
NAME:.metadata.name,\
REPLICAS:.spec.replicas,\
AFFINITY:.spec.template.spec.affinity

# View detailed affinity for specific deployment
echo -e "\n=== API Service Affinity Details ==="
kubectl get deployment api-service -n pod-affinity-demo -o yaml | \
  grep -A 20 "affinity:"
```

## 🎓 Summary

You have successfully:
- ✅ Used pod affinity to co-locate related pods
- ✅ Implemented pod anti-affinity for HA
- ✅ Configured required and preferred rules
- ✅ Combined affinity and anti-affinity
- ✅ Spread pods across availability zones
- ✅ Tested scheduling constraints

## 🔑 Key Concepts Learned

1. **Pod Affinity**: Attract pods to pods (co-location)
2. **Pod Anti-Affinity**: Repel pods from pods (spreading)
3. **Topology Key**: Defines scope (node, zone, region)
4. **Required**: Hard constraint (must satisfy)
5. **Preferred**: Soft constraint (best effort)
6. **Weight**: Priority for preferred rules (1-100)

## 📊 When to Use What

| Scenario                  | Use               | Topology Key  |
| ------------------------- | ----------------- | ------------- |
| **Co-locate app + cache** | Pod Affinity      | hostname      |
| **Spread replicas (HA)**  | Pod Anti-Affinity | hostname      |
| **Zone redundancy**       | Pod Anti-Affinity | zone          |
| **Avoid noisy neighbor**  | Pod Anti-Affinity | hostname      |
| **Near data/service**     | Pod Affinity      | hostname/zone |

## 📝 Best Practices

- ✅ Use anti-affinity for high-availability services
- ✅ Use affinity for latency-sensitive co-location
- ✅ Prefer soft (preferred) over hard (required) when possible
- ✅ Choose appropriate topology keys
- ✅ Consider cluster size vs replica count
- ✅ Test with more replicas than nodes
- ✅ Monitor pod scheduling events
- ✅ Balance HA with resource utilization

## 🔍 Common Patterns

**High Availability Pattern:**
```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector:
      matchLabels:
        app: myapp
    topologyKey: kubernetes.io/hostname
```

**Cache Co-location:**
```yaml
podAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    podAffinityTerm:
      labelSelector:
        matchLabels:
          app: cache
      topologyKey: kubernetes.io/hostname
```

**Zone Distribution:**
```yaml
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    podAffinityTerm:
      labelSelector:
        matchLabels:
          app: myapp
      topologyKey: topology.kubernetes.io/zone
```

## 🧹 Cleanup

```bash
# Delete all resources
kubectl delete namespace pod-affinity-demo

echo "✅ Pod affinity demo resources cleaned up"
```

## 🚀 Next Steps

Proceed to:
- **[Chapter 8: Pod Topology Spread Constraints](../chapter-8-topology-spread/README.md)**

In the next chapter, you'll learn a more advanced way to control pod distribution across failure domains.
