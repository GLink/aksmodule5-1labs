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

```bash
# Load your lab configuration
source ./chapter-0-setup/lab-config.sh

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

```bash
# Create namespace
kubectl create namespace statefulset-demo

# Create headless service (clusterIP: None)
cat <<EOF | kubectl apply -f -
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
EOF

echo "✅ Headless service created"
```

## 📦 Step 2: Create Basic StatefulSet

```bash
# Create a simple StatefulSet
cat <<EOF | kubectl apply -f -
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
EOF

echo "✅ StatefulSet created"
```

## 🔍 Step 3: Watch Ordered Pod Creation

```bash
# Watch pods being created one by one
echo "=== Watching StatefulSet pod creation ==="
kubectl get pods -n statefulset-demo -w -l app=nginx &
WATCH_PID=$!

# Wait for all pods to be ready
kubectl wait --for=condition=Ready pod -l app=nginx -n statefulset-demo --timeout=180s

kill $WATCH_PID 2>/dev/null

# List pods - note the ordinal naming
kubectl get pods -n statefulset-demo -l app=nginx
```

## 📊 Step 4: Verify Stable Network Identity

```bash
# Each pod gets a stable DNS name: <pod-name>.<service-name>.<namespace>.svc.cluster.local
echo "=== StatefulSet Pod DNS Names ==="
for i in 0 1 2; do
  echo "web-$i.nginx-service.statefulset-demo.svc.cluster.local"
done

# Test DNS resolution from within cluster
kubectl run -it --rm debug --image=busybox --restart=Never -n statefulset-demo -- sh -c "
  nslookup web-0.nginx-service.statefulset-demo.svc.cluster.local
  nslookup web-1.nginx-service.statefulset-demo.svc.cluster.local
  nslookup web-2.nginx-service.statefulset-demo.svc.cluster.local
"
```

## 💾 Step 5: Verify Persistent Storage

```bash
# List PersistentVolumeClaims - one per pod
kubectl get pvc -n statefulset-demo

# Show which PVC is bound to which pod
echo -e "\n=== PVC to Pod Mapping ==="
for i in 0 1 2; do
  echo "www-web-$i -> web-$i"
done

# Write data to each pod's storage
for i in 0 1 2; do
  kubectl exec web-$i -n statefulset-demo -- sh -c "echo 'Hello from web-$i' > /usr/share/nginx/html/index.html"
done

# Verify data persists per pod
echo -e "\n=== Verify Unique Storage per Pod ==="
for i in 0 1 2; do
  echo -n "web-$i: "
  kubectl exec web-$i -n statefulset-demo -- cat /usr/share/nginx/html/index.html
done
```

## 🔄 Step 6: Test Pod Deletion and Recreation

```bash
# Delete a pod and watch it recreate with same identity
echo "=== Deleting web-1 ==="
kubectl delete pod web-1 -n statefulset-demo

# Watch it recreate
kubectl get pods -n statefulset-demo -l app=nginx -w &
WATCH_PID=$!

# Wait for recreation
kubectl wait --for=condition=Ready pod/web-1 -n statefulset-demo --timeout=120s

kill $WATCH_PID 2>/dev/null

# Verify data persisted
echo -e "\n=== Verify Data Persisted ==="
kubectl exec web-1 -n statefulset-demo -- cat /usr/share/nginx/html/index.html
```

## 📈 Step 7: Scale StatefulSet

```bash
# Scale up to 5 replicas (ordered scaling)
echo "=== Scaling up to 5 replicas ==="
kubectl scale statefulset web -n statefulset-demo --replicas=5

# Watch ordered creation of web-3 and web-4
kubectl get pods -n statefulset-demo -l app=nginx -w &
WATCH_PID=$!

sleep 30
kill $WATCH_PID 2>/dev/null

# Verify all pods
kubectl get pods -n statefulset-demo -l app=nginx

# Scale down to 3 (ordered termination from highest index)
echo -e "\n=== Scaling down to 3 replicas ==="
kubectl scale statefulset web -n statefulset-demo --replicas=3

sleep 10
kubectl get pods -n statefulset-demo -l app=nginx

# Note: PVCs are not deleted when scaling down
kubectl get pvc -n statefulset-demo
```

## 🗃️ Step 8: Deploy PostgreSQL StatefulSet

```bash
# Deploy a PostgreSQL StatefulSet with real database use case
cat <<EOF | kubectl apply -f -
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
EOF

echo "✅ PostgreSQL StatefulSet deployed"

# Wait for PostgreSQL pods
kubectl wait --for=condition=Ready pod -l app=postgres -n statefulset-demo --timeout=180s

kubectl get pods -n statefulset-demo -l app=postgres -o wide
```

## 🧪 Step 9: Test PostgreSQL StatefulSet

```bash
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
for i in 0 1 2; do
  kubectl exec postgres-$i -n statefulset-demo -- psql -U myuser -d mydb -c "
    INSERT INTO test_data (pod_name) VALUES ('postgres-$i');
  " 2>/dev/null || echo "Note: Each pod has its own database instance"
done

# Query data
echo -e "\n=== Data in postgres-0 ==="
kubectl exec postgres-0 -n statefulset-demo -- psql -U myuser -d mydb -c "SELECT * FROM test_data;"
```

## 🔄 Step 10: Rolling Update of StatefulSet

```bash
# Update the StatefulSet image
kubectl set image statefulset/web nginx=nginx:1.25-alpine -n statefulset-demo

# Watch the rolling update (updates pods in reverse order: 2, 1, 0)
echo "=== Watching Rolling Update ==="
kubectl rollout status statefulset/web -n statefulset-demo

# Check updated pods
kubectl get pods -n statefulset-demo -l app=nginx -o wide

# View rollout history
kubectl rollout history statefulset/web -n statefulset-demo
```

## 📋 Step 11: Configure Pod Management Policy

```bash
# Create StatefulSet with Parallel pod management
cat <<EOF | kubectl apply -f -
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
EOF

echo "✅ StatefulSet with Parallel management policy created"

# Watch all pods start in parallel
kubectl get pods -n statefulset-demo -l app=parallel -w &
WATCH_PID=$!
sleep 20
kill $WATCH_PID 2>/dev/null

kubectl get pods -n statefulset-demo -l app=parallel
```

## 🔍 Step 12: Examine StatefulSet Details

```bash
# Describe StatefulSet
kubectl describe statefulset web -n statefulset-demo

# View StatefulSet status
kubectl get statefulset -n statefulset-demo -o wide

# Check update strategy
kubectl get statefulset web -n statefulset-demo -o jsonpath='{.spec.updateStrategy}'
echo ""

# View all PVCs created by StatefulSets
kubectl get pvc -n statefulset-demo
```

## ⚙️ Step 13: Partition Updates

```bash
# Update with partition (only pods >= partition will update)
kubectl patch statefulset web -n statefulset-demo -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}'

# Update the image
kubectl set image statefulset/web nginx=nginx:1.26-alpine -n statefulset-demo

# Only web-2 will update (partition=2)
kubectl rollout status statefulset/web -n statefulset-demo

# Check which pods updated
echo "=== Pod Image Versions ==="
for i in 0 1 2; do
  image=$(kubectl get pod web-$i -n statefulset-demo -o jsonpath='{.spec.containers[0].image}')
  echo "web-$i: $image"
done

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

```bash
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

echo "✅ StatefulSet demo resources cleaned up"
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

```bash
# Load your configuration
source ./chapter-0-setup/lab-config.sh

# Delete the entire resource group
az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait

echo "🗑️ Complete lab cleanup initiated"
echo "All Azure resources will be deleted"
```

## 📚 Further Learning

- [AKS Best Practices](https://learn.microsoft.com/en-us/azure/aks/best-practices)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [AKS Roadmap](https://aka.ms/aks/roadmap)
- [Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/)

**Thank you for completing this lab!** 🚀
