# Chapter 9: StatefulSets

In this final chapter, you'll learn how to deploy and manage stateful applications using StatefulSets. Unlike Deployments, StatefulSets provide stable network identities, ordered deployment/scaling, and persistent storage for each pod.

## 🎯 Learning Objectives

- Understand StatefulSets vs Deployments
- Deploy stateful applications with persistent storage
- Manage ordered pod creation and termination
- Use headless services for stable network identities
- Implement pod management policies
- Scale stateful applications safely
- Perform rolling updates on StatefulSets

## 📚 Prerequisites

- Completed [Chapter 0: Setup and Prerequisites](../chapter-0-setup/README.md)
- Understanding of Kubernetes storage concepts

## 🔄 Load Your Configuration

```powershell
# Load your lab configuration
. .\chapter-0-setup\lab-config.sh

# Verify cluster access
kubectl cluster-info
```

## 📖 Understanding StatefulSets

**StatefulSets** are designed for applications that require:
- Stable, unique network identifiers
- Stable, persistent storage
- Ordered, graceful deployment and scaling
- Ordered, automated rolling updates

**Key Differences from Deployments:**

| Feature              | Deployment          | StatefulSet                |
| -------------------- | ------------------- | -------------------------- |
| **Pod Names**        | Random suffix       | Ordered index (0, 1, 2...) |
| **Network Identity** | Ephemeral           | Stable (predictable DNS)   |
| **Storage**          | Shared or ephemeral | Dedicated PVC per pod      |
| **Scaling**          | Parallel            | Ordered (one at a time)    |
| **Updates**          | Rolling (parallel)  | Rolling (ordered)          |

**Use Cases:**
- Databases (PostgreSQL, MySQL, MongoDB)
- Message queues (Kafka, RabbitMQ)
- Distributed systems (Elasticsearch, Cassandra)
- Any application requiring stable identity

## 🗄️ Step 1: Create Headless Service

```powershell
# Create namespace
kubectl create namespace statefulset-demo

# Create headless service (clusterIP: None)
@"
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: statefulset-demo
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
"@ | kubectl apply -f -

Write-Host "✅ Headless service created"
```

## 📦 Step 2: Create Basic StatefulSet

```powershell
# Create a simple StatefulSet
@"
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: statefulset-demo
spec:
  serviceName: "nginx-service"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
"@ | kubectl apply -f -

Write-Host "✅ StatefulSet created"
```

## 🔍 Step 3: Watch Ordered Pod Creation

```powershell
# Watch pods being created one by one
Write-Host "=== Watching StatefulSet pod creation ==="
$watchJob = Start-Job { kubectl get pods -n statefulset-demo -w -l app=nginx }

# Wait for all pods to be ready
kubectl wait --for=condition=Ready pod -l app=nginx -n statefulset-demo --timeout=180s

Stop-Job $watchJob
Receive-Job $watchJob
Remove-Job $watchJob

# List pods - note the ordinal naming
kubectl get pods -n statefulset-demo -l app=nginx
```

## 📊 Step 4: Verify Stable Network Identity

```powershell
# Each pod gets a stable DNS name: <pod-name>.<service-name>.<namespace>.svc.cluster.local
Write-Host "=== StatefulSet Pod DNS Names ==="
0..2 | ForEach-Object {
  Write-Host "web-$($_).nginx-service.statefulset-demo.svc.cluster.local"
}

# Test DNS resolution from within cluster
$dnsChecks = @"
nslookup web-0.nginx-service.statefulset-demo.svc.cluster.local
nslookup web-1.nginx-service.statefulset-demo.svc.cluster.local
nslookup web-2.nginx-service.statefulset-demo.svc.cluster.local
"@
kubectl run -it --rm debug --image=busybox --restart=Never -n statefulset-demo -- sh -c $dnsChecks
```

## 💾 Step 5: Verify Persistent Storage

```powershell
# List PersistentVolumeClaims - one per pod
kubectl get pvc -n statefulset-demo

# Show which PVC is bound to which pod
Write-Host "`n=== PVC to Pod Mapping ==="
0..2 | ForEach-Object {
  Write-Host "www-web-$($_) -> web-$($_)"
}

# Write data to each pod's storage
0..2 | ForEach-Object {
  kubectl exec "web-$($_)" -n statefulset-demo -- sh -c "echo 'Hello from web-$($_)' > /usr/share/nginx/html/index.html"
}

# Verify data persists per pod
Write-Host "`n=== Verify Unique Storage per Pod ==="
0..2 | ForEach-Object {
  Write-Host -NoNewline "web-$($_): "
  kubectl exec "web-$($_)" -n statefulset-demo -- cat /usr/share/nginx/html/index.html
}
```

## 🔄 Step 6: Test Pod Deletion and Recreation

```powershell
# Delete a pod and watch it recreate with same identity
Write-Host "=== Deleting web-1 ==="
kubectl delete pod web-1 -n statefulset-demo

# Watch it recreate
$watchJob = Start-Job { kubectl get pods -n statefulset-demo -l app=nginx -w }

# Wait for recreation
kubectl wait --for=condition=Ready pod/web-1 -n statefulset-demo --timeout=120s

Stop-Job $watchJob
Receive-Job $watchJob
Remove-Job $watchJob

# Verify data persisted
Write-Host "`n=== Verify Data Persisted ==="
kubectl exec web-1 -n statefulset-demo -- cat /usr/share/nginx/html/index.html
```

## 📈 Step 7: Scale StatefulSet

```powershell
# Scale up to 5 replicas (ordered scaling)
Write-Host "=== Scaling up to 5 replicas ==="
kubectl scale statefulset web -n statefulset-demo --replicas=5

# Watch ordered creation of web-3 and web-4
$watchJob = Start-Job { kubectl get pods -n statefulset-demo -l app=nginx -w }

Start-Sleep -Seconds 30
Stop-Job $watchJob
Receive-Job $watchJob
Remove-Job $watchJob

# Verify all pods
kubectl get pods -n statefulset-demo -l app=nginx

# Scale down to 3 (ordered termination from highest index)
Write-Host "`n=== Scaling down to 3 replicas ==="
kubectl scale statefulset web -n statefulset-demo --replicas=3

Start-Sleep -Seconds 10
kubectl get pods -n statefulset-demo -l app=nginx

# Note: PVCs are not deleted when scaling down
kubectl get pvc -n statefulset-demo
```

## 🗃️ Step 8: Deploy PostgreSQL StatefulSet

```powershell
# Deploy a PostgreSQL StatefulSet with real database use case
@"
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: statefulset-demo
data:
  POSTGRES_DB: mydb
  POSTGRES_USER: myuser
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: statefulset-demo
type: Opaque
stringData:
  POSTGRES_PASSWORD: "MyS3cretP@ssw0rd"
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: statefulset-demo
spec:
  ports:
  - port: 5432
    name: postgres
  clusterIP: None
  selector:
    app: postgres
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: statefulset-demo
spec:
  serviceName: "postgres-service"
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_DB
          valueFrom:
            configMapKeyRef:
              name: postgres-config
              key: POSTGRES_DB
        - name: POSTGRES_USER
          valueFrom:
            configMapKeyRef:
              name: postgres-config
              key: POSTGRES_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRES_PASSWORD
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 2Gi
"@ | kubectl apply -f -

Write-Host "✅ PostgreSQL StatefulSet deployed"

# Wait for PostgreSQL pods
kubectl wait --for=condition=Ready pod -l app=postgres -n statefulset-demo --timeout=180s

kubectl get pods -n statefulset-demo -l app=postgres -o wide
```

## 🧪 Step 9: Test PostgreSQL StatefulSet

```powershell
# Connect to postgres-0 and create a database
kubectl exec -it postgres-0 -n statefulset-demo -- psql -U myuser -d mydb -c "
  CREATE TABLE IF NOT EXISTS test_data (
    id SERIAL PRIMARY KEY,
    pod_name VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  );
  INSERT INTO test_data (pod_name) VALUES ('postgres-0');
"

# Insert data from each pod
0..2 | ForEach-Object {
  kubectl exec "postgres-$($_)" -n statefulset-demo -- psql -U myuser -d mydb -c "
    INSERT INTO test_data (pod_name) VALUES ('postgres-$($_)');
  " 2>$null

  if ($LASTEXITCODE -ne 0) {
    Write-Host "Note: Each pod has its own database instance"
  }
}

# Query data
Write-Host "`n=== Data in postgres-0 ==="
kubectl exec postgres-0 -n statefulset-demo -- psql -U myuser -d mydb -c "SELECT * FROM test_data;"
```

## 🔄 Step 10: Rolling Update of StatefulSet

```powershell
# Update the StatefulSet image
kubectl set image statefulset/web nginx=nginx:1.25-alpine -n statefulset-demo

# Watch the rolling update (updates pods in reverse order: 2, 1, 0)
Write-Host "=== Watching Rolling Update ==="
kubectl rollout status statefulset/web -n statefulset-demo

# Check updated pods
kubectl get pods -n statefulset-demo -l app=nginx -o wide

# View rollout history
kubectl rollout history statefulset/web -n statefulset-demo
```

## 📋 Step 11: Configure Pod Management Policy

```powershell
# Create StatefulSet with Parallel pod management
@"
apiVersion: v1
kind: Service
metadata:
  name: parallel-service
  namespace: statefulset-demo
spec:
  ports:
  - port: 80
  clusterIP: None
  selector:
    app: parallel
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: parallel-statefulset
  namespace: statefulset-demo
spec:
  serviceName: "parallel-service"
  podManagementPolicy: Parallel  # All pods start/stop in parallel
  replicas: 3
  selector:
    matchLabels:
      app: parallel
  template:
    metadata:
      labels:
        app: parallel
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
"@ | kubectl apply -f -

Write-Host "✅ StatefulSet with Parallel management policy created"

# Watch all pods start in parallel
$watchJob = Start-Job { kubectl get pods -n statefulset-demo -l app=parallel -w }
Start-Sleep -Seconds 20
Stop-Job $watchJob
Receive-Job $watchJob
Remove-Job $watchJob

kubectl get pods -n statefulset-demo -l app=parallel
```

## 🔍 Step 12: Examine StatefulSet Details

```powershell
# Describe StatefulSet
kubectl describe statefulset web -n statefulset-demo

# View StatefulSet status
kubectl get statefulset -n statefulset-demo -o wide

# Check update strategy
kubectl get statefulset web -n statefulset-demo -o jsonpath='{.spec.updateStrategy}'
Write-Host ""

# View all PVCs created by StatefulSets
kubectl get pvc -n statefulset-demo
```

## ⚙️ Step 13: Partition Updates

```powershell
# Update with partition (only pods >= partition will update)
kubectl patch statefulset web -n statefulset-demo -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}'

# Update the image
kubectl set image statefulset/web nginx=nginx:1.26-alpine -n statefulset-demo

# Only web-2 will update (partition=2)
kubectl rollout status statefulset/web -n statefulset-demo

# Check which pods updated
Write-Host "=== Pod Image Versions ==="
0..2 | ForEach-Object {
  $image = kubectl get pod "web-$($_)" -n statefulset-demo -o jsonpath='{.spec.containers[0].image}'
  Write-Host "web-$($_): $image"
}

# Remove partition to complete update
kubectl patch statefulset web -n statefulset-demo -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
kubectl rollout status statefulset/web -n statefulset-demo
```

## 🎓 Summary

You have successfully:
- ✅ Created headless services for stable network identities
- ✅ Deployed StatefulSets with persistent storage
- ✅ Verified ordered pod creation and scaling
- ✅ Tested data persistence across pod deletions
- ✅ Deployed a real-world PostgreSQL StatefulSet
- ✅ Performed rolling updates on StatefulSets
- ✅ Configured pod management policies
- ✅ Implemented partitioned updates

## 🔑 Key Concepts Learned

1. **Stable Identity**: Pods get predictable names (0, 1, 2...)
2. **Headless Service**: Enables direct pod-to-pod DNS resolution
3. **VolumeClaimTemplates**: Creates dedicated PVC per pod
4. **Ordered Operations**: Sequential creation, scaling, updates
5. **Pod Management Policy**: OrderedReady vs Parallel
6. **Update Partition**: Control which pods get updated
7. **Persistent Storage**: PVCs remain even when pods are deleted

## 📊 When to Use StatefulSets

✅ **Use StatefulSets for:**
- Databases (SQL, NoSQL)
- Message queues (Kafka, RabbitMQ)
- Distributed caches (Redis cluster)
- Stateful distributed systems
- Applications requiring stable network identity

❌ **Use Deployments for:**
- Stateless applications
- Microservices without state
- Applications where any pod is interchangeable

## 📝 Best Practices

- ✅ Always use headless services with StatefulSets
- ✅ Configure appropriate resource requests/limits
- ✅ Use pod disruption budgets for availability
- ✅ Implement proper backup strategies for PVCs
- ✅ Test failover scenarios before production
- ✅ Monitor storage usage and capacity
- ✅ Use readiness probes for database initialization
- ✅ Consider pod anti-affinity for HA
- ✅ Document scaling procedures
- ✅ Plan for storage expansion

## 🔍 StatefulSet Patterns

**Primary-Replica Database:**
- Pod 0: Primary (read/write)
- Pods 1+: Replicas (read-only)
- Init containers for replication setup

**Sharded System:**
- Each pod handles a specific shard
- Consistent hash routing
- Data locality per pod

**Quorum-based Systems:**
- Odd number of replicas
- Leader election
- Distributed consensus

## 🧹 Cleanup

```powershell
# Delete StatefulSets (cascades to pods)
kubectl delete statefulset web postgres parallel-statefulset -n statefulset-demo

# Delete services
kubectl delete service nginx-service postgres-service parallel-service -n statefulset-demo

# Delete ConfigMaps and Secrets
kubectl delete configmap postgres-config -n statefulset-demo
kubectl delete secret postgres-secret -n statefulset-demo

# PVCs must be deleted manually (by design)
kubectl delete pvc --all -n statefulset-demo

# Delete namespace
kubectl delete namespace statefulset-demo

Write-Host "✅ StatefulSet demo resources cleaned up"
```

## 🏁 Lab Completion

**Congratulations! 🎉** You have completed all chapters of the AKS Advanced Concepts Lab!

You've learned:
- ✅ **Chapters 1-2**: Azure Key Vault integration (CSI Driver & External Secrets)
- ✅ **Chapter 3**: Kubernetes RBAC with Azure AD
- ✅ **Chapter 4**: Workload Identity and Federated Credentials
- ✅ **Chapter 5**: Node Selectors and Affinity
- ✅ **Chapter 6**: Taints and Tolerations
- ✅ **Chapter 7**: Pod Affinity and Anti-Affinity
- ✅ **Chapter 8**: Pod Topology Spread Constraints
- ✅ **Chapter 9**: StatefulSets

## 🧹 Final Cleanup

To clean up all lab resources:

```powershell
# Load your configuration
. .\chapter-0-setup\lab-config.sh

# Delete the entire resource group
az group delete `
  --name $RESOURCE_GROUP `
  --yes `
  --no-wait

Write-Host "🗑️ Complete lab cleanup initiated"
Write-Host "All Azure resources will be deleted"
```

## 📚 Further Learning

- [AKS Best Practices](https://learn.microsoft.com/en-us/azure/aks/best-practices)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [AKS Roadmap](https://aka.ms/aks/roadmap)
- [Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/)

**Thank you for completing this lab!** 🚀
