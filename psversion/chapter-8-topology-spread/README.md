# Chapter 8: Pod Topology Spread Constraints

In this chapter, you'll learn how to use Pod Topology Spread Constraints to evenly distribute pods across failure domains. This is a more declarative and flexible approach than pod anti-affinity for achieving high availability.

## 🎯 Learning Objectives

- Understand topology spread constraints
- Distribute pods evenly across zones and nodes
- Configure maxSkew and spread policies
- Implement high availability patterns
- Compare with pod anti-affinity
- Use multiple topology constraints

## 📚 Prerequisites

- Completed [Chapter 0: Setup and Prerequisites](../chapter-0-setup/README.md)
- Understanding of pod affinity (Chapter 7 recommended)
- AKS cluster with nodes in multiple availability zones

## 🔄 Load Your Configuration

```powershell
# Load your lab configuration
. .\psversion\chapter-0-setup\lab-config.ps1

# View nodes with zone labels
kubectl get nodes -L topology.kubernetes.io/zone,kubernetes.io/hostname
```

## 📖 Understanding Topology Spread Constraints

**Pod Topology Spread Constraints** allow you to control how pods are distributed across your cluster based on topology domains (zones, nodes, regions).

**Key Parameters:**
- **maxSkew**: Maximum difference in pod count between domains
- **topologyKey**: Label key defining the domain (zone, hostname)
- **whenUnsatisfiable**: DoNotSchedule (hard) or ScheduleAnyway (soft)
- **labelSelector**: Which pods to consider in the spread

**Benefits over Anti-Affinity:**
- More declarative (specify desired state)
- Better control with maxSkew
- Easier to understand intent
- Automatic rebalancing awareness

## 🌍 Step 1: Basic Zone-Level Spreading

```powershell
# Create namespace
kubectl create namespace topology-demo

# Deploy with zone-level spreading
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: topology-demo
spec:
  replicas: 6
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: web
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
"@ | kubectl apply -f -

Write-Host "✅ Web app with zone spreading deployed"

# Wait for pods
kubectl wait --for=condition=Ready pod -l app=web -n topology-demo --timeout=60s

# View distribution across zones
kubectl get pods -n topology-demo -l app=web -o wide
```

## 📊 Step 2: Analyze Zone Distribution

```powershell
# Count pods per zone
Write-Host "=== Pods per Zone ==="
$zones = "swedencentral-1", "swedencentral-2", "swedencentral-3"
foreach ($zone in $zones) {
  $count = 0
  kubectl get pods -n topology-demo -l app=web -o wide --no-headers 2>$null | ForEach-Object {
    $columns = ($_ -split '\s+')
    if ($columns.Length -ge 7) {
      $node = $columns[6]
      $node_zone = kubectl get node $node -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}' 2>$null
      if ($node_zone -eq $zone) {
        $count++
      }
    }
  }
  Write-Host "Zone ${zone}: $count pods"
}

# Alternative: Distribution by Node and Zone
Write-Host "`n=== Distribution by Node and Zone ==="
kubectl get pods -n topology-demo -l app=web -o wide --no-headers 2>$null | ForEach-Object {
  $columns = ($_ -split '\s+')
  if ($columns.Length -ge 7) {
    $name = $columns[0]
    $node = $columns[6]
    $zone = kubectl get node $node -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}' 2>$null
    Write-Host "$name -> $node (zone: $zone)"
  }
}
```

## 🖥️ Step 3: Node-Level Spreading

```powershell
# Deploy with node-level spreading (maxSkew=2)
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
  namespace: topology-demo
spec:
  replicas: 6
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      topologySpreadConstraints:
      - maxSkew: 2
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: api
      containers:
      - name: api
        image: nginx:alpine
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
"@ | kubectl apply -f -

Write-Host "✅ API service with node spreading deployed"

Start-Sleep -Seconds 10
kubectl get pods -n topology-demo -l app=api -o wide

# Count pods per node
Write-Host "`n=== Pods per Node ==="
$nodes = (kubectl get nodes -o jsonpath='{.items[*].metadata.name}') -split " " | Where-Object { $_ }
foreach ($node in $nodes) {
  $count = (kubectl get pods -n topology-demo -l app=api `
    --field-selector "spec.nodeName=$node" --no-headers 2>$null | Measure-Object -Line).Lines
  if ($count -gt 0) {
    Write-Host "${node}: $count pods"
  }
}
```

## 🔄 Step 4: Multiple Topology Constraints

```powershell
# Deploy with both zone and node spreading
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-service
  namespace: topology-demo
spec:
  replicas: 6
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      topologySpreadConstraints:
      # Spread across zones first
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: database
      # Then spread across nodes within zones
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: database
      containers:
      - name: postgres
        image: postgres:15-alpine
        env:
        - name: POSTGRES_PASSWORD
          value: "demo-password"
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
"@ | kubectl apply -f -

Write-Host "✅ Database with multi-level spreading deployed"

Start-Sleep -Seconds 10
kubectl get pods -n topology-demo -l app=database -o wide
```

## 🌟 Step 5: Soft Constraint (ScheduleAnyway)

```powershell
# First, check current pod count per node to see if we're near limits
Write-Host "=== Current pod count per node ==="
$nodes = (kubectl get nodes -o jsonpath='{.items[*].metadata.name}') -split ' ' | Where-Object { $_ }
foreach ($node in $nodes) {
  $count = (kubectl get pods --all-namespaces --field-selector "spec.nodeName=$node" --no-headers 2>$null | Measure-Object -Line).Lines
  $limit = kubectl get node $node -o jsonpath='{.status.capacity.pods}'
  Write-Host "${node}: $count/$limit pods"
}

# Deploy with soft spreading (will schedule even if skew exceeds maxSkew)
# Note: If nodes are near pod capacity, some pods may remain pending
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-worker
  namespace: topology-demo
spec:
  replicas: 10
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: worker
      containers:
      - name: worker
        image: nginx:alpine
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
"@ | kubectl apply -f -

Write-Host "✅ Batch workers with soft spreading deployed"

Start-Sleep -Seconds 10
kubectl get pods -n topology-demo -l app=worker -o wide

# Check distribution (some pods may be pending if node pod limit is reached)
Write-Host "`n=== Worker Distribution ==="
foreach ($node in $nodes) {
  $count = (kubectl get pods -n topology-demo -l app=worker `
    --field-selector "spec.nodeName=$node" --no-headers 2>$null | Measure-Object -Line).Lines
  Write-Host "${node}: $count workers"
}

# Check for pending pods
$pending = (kubectl get pods -n topology-demo -l app=worker --field-selector status.phase=Pending --no-headers 2>$null | Measure-Object -Line).Lines
if ($pending -gt 0) {
  Write-Host "`n⚠️  Note: $pending pods pending (likely due to node pod capacity limits)"
  Write-Host "This is expected behavior when nodes reach their maximum pod count."
}
```

## 🎯 Step 6: MinDomains for Guaranteed Distribution

```powershell
# Deploy with minDomains to ensure spreading (K8s 1.25+)
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-service
  namespace: topology-demo
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
      topologySpreadConstraints:
      - maxSkew: 1
        minDomains: 3  # Ensure spreading across at least 3 zones
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: critical
      containers:
      - name: app
        image: nginx:alpine
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
"@ | kubectl apply -f -

Write-Host "✅ Critical service with minDomains deployed"

Start-Sleep -Seconds 10
kubectl get pods -n topology-demo -l app=critical -o wide
```

## 🧪 Step 7: Test Scaling and Rebalancing

```powershell
# Scale up web app and watch distribution
Write-Host "=== Scaling web app to 12 replicas ==="
kubectl scale deployment web-app -n topology-demo --replicas=12

# Watch pods being created
$watchJob = Start-Job -ScriptBlock { kubectl get pods -n topology-demo -l app=web -w }
Start-Sleep -Seconds 15
Stop-Job $watchJob
Receive-Job $watchJob
Remove-Job $watchJob

# Check new distribution
Write-Host "`n=== Updated Zone Distribution ==="
$zones = "swedencentral-1", "swedencentral-2", "swedencentral-3"
foreach ($zone in $zones) {
  $count = 0
  kubectl get pods -n topology-demo -l app=web -o wide --no-headers 2>$null | ForEach-Object {
    $columns = ($_ -split '\s+')
    if ($columns.Length -ge 7) {
      $node = $columns[6]
      $node_zone = kubectl get node $node -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}' 2>$null
      if ($node_zone -eq $zone) {
        $count++
      }
    }
  }
  Write-Host "Zone ${zone}: $count pods"
}
```

## 📋 Step 8: Compare with Pod Anti-Affinity

```powershell
# Deploy same workload with pod anti-affinity for comparison
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: antiaffinity-comparison
  namespace: topology-demo
spec:
  replicas: 6
  selector:
    matchLabels:
      app: comparison
      method: antiaffinity
  template:
    metadata:
      labels:
        app: comparison
        method: antiaffinity
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: comparison
                  method: antiaffinity
              topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
"@ | kubectl apply -f -

Write-Host "✅ Anti-affinity comparison deployed"

Start-Sleep -Seconds 10

# Compare the two approaches
Write-Host "`n=== Topology Spread vs Anti-Affinity ==="
Write-Host "Topology Spread Constraint pods:"
kubectl get pods -n topology-demo -l app=web -o wide --no-headers |
  ForEach-Object { ($_ -split '\s+')[6] } |
  Group-Object |
  Sort-Object Name |
  ForEach-Object { Write-Host ("{0,3} {1}" -f $_.Count, $_.Name) }

Write-Host "`nPod Anti-Affinity pods:"
kubectl get pods -n topology-demo -l method=antiaffinity -o wide --no-headers |
  ForEach-Object { ($_ -split '\s+')[6] } |
  Group-Object |
  Sort-Object Name |
  ForEach-Object { Write-Host ("{0,3} {1}" -f $_.Count, $_.Name) }
```

## 🔍 Step 9: View Scheduling Events

```powershell
# Check events for topology-related decisions
kubectl get events -n topology-demo --sort-by='.lastTimestamp' | Select-String -Pattern 'topology'

# Describe a pod to see topology decisions
$POD_NAME = kubectl get pods -n topology-demo -l app=web -o jsonpath='{.items[0].metadata.name}'
kubectl describe pod $POD_NAME -n topology-demo | Select-String -Pattern 'Topology Spread Constraints' -Context 0,10
```

## 📊 Step 10: Visualize Complete Distribution

```powershell
# Create a summary of all workloads
Write-Host "=== Complete Topology Distribution Summary ==="
Write-Host ""

foreach ($app in "web", "api", "database", "worker", "critical") {
  Write-Host "Application: $app"
  $podData = kubectl get pods -n topology-demo -l "app=$app" -o json 2>$null | ConvertFrom-Json
  if ($podData.items) {
    $podData.items |
      Group-Object -Property { $_.spec.nodeName } |
      Sort-Object Name |
      ForEach-Object { Write-Host "  Node $($_.Name): $($_.Count) pods" }
  }
  Write-Host ""
}
```

## 🎓 Summary

You have successfully:
- ✅ Implemented zone-level topology spreading
- ✅ Configured node-level pod distribution
- ✅ Used multiple topology constraints
- ✅ Applied hard and soft spread policies
- ✅ Tested scaling with topology constraints
- ✅ Compared topology spread with pod anti-affinity

## 🔑 Key Concepts Learned

1. **maxSkew**: Maximum allowed imbalance between domains
2. **topologyKey**: Defines the failure domain (zone, node)
3. **whenUnsatisfiable**: DoNotSchedule (hard) vs ScheduleAnyway (soft)
4. **labelSelector**: Identifies pods to include in spread calculation
5. **minDomains**: Minimum number of domains to spread across

## 📊 Topology Spread vs Anti-Affinity

| Feature         | Topology Spread     | Pod Anti-Affinity  |
| --------------- | ------------------- | ------------------ |
| **Approach**    | Declarative balance | Pairwise repulsion |
| **Control**     | maxSkew parameter   | Hard constraints   |
| **Flexibility** | Soft/hard options   | Preferred/required |
| **Intent**      | Even distribution   | Separation         |
| **Complexity**  | Simpler             | More verbose       |
| **Rebalancing** | Aware of existing   | Considers pairs    |

**Recommendation**: Use Topology Spread Constraints for most HA scenarios!

## 📝 Best Practices

- ✅ Use topology spread for high-availability workloads
- ✅ Start with maxSkew=1 for even distribution
- ✅ Use DoNotSchedule for critical services
- ✅ Use ScheduleAnyway for best-effort spreading
- ✅ Combine zone and node topology keys
- ✅ Set minDomains for guaranteed distribution
- ✅ Monitor pod distribution regularly
- ✅ Test with various replica counts

## 🔍 Common Patterns

**High Availability (Strict):**
```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: myapp
```

**Best Effort Distribution:**
```yaml
topologySpreadConstraints:
- maxSkew: 2
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels:
      app: myapp
```

**Multi-Level Spreading:**
```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: myapp
- maxSkew: 2
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels:
      app: myapp
```

## 🧹 Cleanup

```powershell
# Delete all resources
kubectl delete namespace topology-demo

Write-Host "✅ Topology spread demo resources cleaned up"
```

## 🚀 Next Steps

Proceed to:
- **[Chapter 9: StatefulSets](../chapter-9-statefulsets/README.md)**

In the final chapter, you'll learn how to deploy and manage stateful applications that require stable network identities and persistent storage.
