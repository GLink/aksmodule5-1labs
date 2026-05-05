# Chapter 3: Kubernetes RBAC with Azure AD

In this chapter, you'll learn how to implement Role-Based Access Control (RBAC) in AKS integrated with Azure Active Directory (Azure AD). This enables you to control access to Kubernetes resources based on Azure AD identities.

## 🎯 Learning Objectives

- Understand Kubernetes RBAC concepts
- Integrate AKS with Azure AD
- Create Roles and ClusterRoles
- Configure RoleBindings and ClusterRoleBindings
- Test access control with different users/groups
- Implement least-privilege access patterns

## 📚 Prerequisites

- Completed [Chapter 0: Setup and Prerequisites](../chapter-0-setup/README.md)
- AKS cluster with Azure AD integration enabled
- Azure AD permissions to create users/groups (or use existing ones)

## 🔄 Load Your Configuration

```powershell
# Load your lab configuration
. .\psversion\chapter-0-setup\lab-config.ps1

# Verify cluster access
kubectl cluster-info
```

## 📖 Understanding Kubernetes RBAC

RBAC in Kubernetes controls access to resources through:
- **Subjects**: Users, Groups, or Service Accounts
- **Roles/ClusterRoles**: Define permissions (verbs on resources)
- **RoleBindings/ClusterRoleBindings**: Link subjects to roles

**Namespace-scoped:**
- Role: Permissions within a namespace
- RoleBinding: Grants Role permissions to subjects

**Cluster-scoped:**
- ClusterRole: Cluster-wide or namespace-agnostic permissions
- ClusterRoleBinding: Grants ClusterRole permissions cluster-wide

## ☸️ Step 1: Enable Azure AD Integration (if not already enabled)

```powershell
# Check if Azure AD integration is enabled
az aks show `
  --resource-group $RESOURCE_GROUP `
  --name $CLUSTER_NAME `
  --query "aadProfile" -o json

# If not enabled, enable it (note: this may require cluster upgrade)
# az aks update `
#   --resource-group $RESOURCE_GROUP `
#   --name $CLUSTER_NAME `
#   --enable-aad `
#   --enable-azure-rbac

Write-Host "✅ Verified Azure AD integration"
```

## 👥 Step 2: Create Azure AD Group for Developers

```powershell
# Create an Azure AD group for developers
$DEV_GROUP_NAME = "aks-developers-$STUDENT_INITIALS"

az ad group create `
  --display-name $DEV_GROUP_NAME `
  --mail-nickname $DEV_GROUP_NAME

# Get group object ID
$DEV_GROUP_ID = $(az ad group show `
  --group $DEV_GROUP_NAME `
  --query id -o tsv)

Write-Host "✅ Azure AD group created: $DEV_GROUP_NAME"
Write-Host "   Group ID: $DEV_GROUP_ID"
```

## 👤 Step 3: Create Azure AD Group for Viewers

```powershell
# Create an Azure AD group for read-only users
$VIEWER_GROUP_NAME = "aks-viewers-$STUDENT_INITIALS"

az ad group create `
  --display-name $VIEWER_GROUP_NAME `
  --mail-nickname $VIEWER_GROUP_NAME

# Get group object ID
$VIEWER_GROUP_ID = $(az ad group show `
  --group $VIEWER_GROUP_NAME `
  --query id -o tsv)

Write-Host "✅ Azure AD group created: $VIEWER_GROUP_NAME"
Write-Host "   Group ID: $VIEWER_GROUP_ID"
```

## 📝 Step 4: Create Demo Namespaces

```powershell
# Create namespaces for RBAC testing
kubectl create namespace dev-team
kubectl create namespace prod-team
kubectl create namespace shared-services

Write-Host "✅ Demo namespaces created"
```

## 🔐 Step 5: Create Role for Developers

```powershell
# Create a Role with development permissions in dev-team namespace
@"
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: dev-team
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["pods", "deployments", "services", "configmaps", "secrets", "jobs", "cronjobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods/log", "pods/exec"]
  verbs: ["get", "create"]
"@ | kubectl apply -f -

Write-Host "✅ Developer role created in dev-team namespace"
```

## 🔗 Step 6: Create RoleBinding for Developers

```powershell
# Bind the developer role to the Azure AD developers group
@"
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: dev-team
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: developer-role
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: "$DEV_GROUP_ID"
"@ | kubectl apply -f -

Write-Host "✅ RoleBinding created for developers in dev-team namespace"
```

## 👁️ Step 7: Create ClusterRole for Viewers

```powershell
# Create a ClusterRole for read-only access
@"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: viewer-clusterrole
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["pods", "deployments", "services", "configmaps", "jobs", "cronjobs", "namespaces"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
"@ | kubectl apply -f -

Write-Host "✅ Viewer ClusterRole created"
```

## 🔗 Step 8: Create ClusterRoleBinding for Viewers

```powershell
# Bind the viewer ClusterRole to the Azure AD viewers group
@"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: viewer-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: viewer-clusterrole
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: "$VIEWER_GROUP_ID"
"@ | kubectl apply -f -

Write-Host "✅ ClusterRoleBinding created for viewers"
```

## 🧪 Step 9: Create Test Resources

```powershell
# Deploy a sample application in dev-team namespace
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
  namespace: dev-team
spec:
  replicas: 2
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
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: dev-team
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
"@ | kubectl apply -f -

Write-Host "✅ Test application deployed in dev-team namespace"

# Deploy in prod-team namespace
kubectl create deployment nginx-prod --image=nginx:alpine -n prod-team
kubectl expose deployment nginx-prod --port=80 -n prod-team

Write-Host "✅ Test application deployed in prod-team namespace"
```

## 🔍 Step 10: Test RBAC with kubectl auth can-i

```powershell
# Test what the current user can do
Write-Host "=== Testing current user permissions ==="

# Check if you can create pods in dev-team
kubectl auth can-i create pods -n dev-team

# Check if you can delete deployments in dev-team
kubectl auth can-i delete deployments -n dev-team

# Check if you can create pods in prod-team
kubectl auth can-i create pods -n prod-team

# Test as developer group
Write-Host "`n=== Testing developer group permissions ==="

# Simulate developer group access (requires both --as and --as-group)
kubectl auth can-i create pods -n dev-team --as=dev-user@example.com --as-group=$DEV_GROUP_ID
kubectl auth can-i delete deployments -n dev-team --as=dev-user@example.com --as-group=$DEV_GROUP_ID
kubectl auth can-i create pods -n prod-team --as=dev-user@example.com --as-group=$DEV_GROUP_ID

# Test as viewer group
Write-Host "`n=== Testing viewer group permissions ==="

kubectl auth can-i get pods -n dev-team --as=viewer-user@example.com --as-group=$VIEWER_GROUP_ID
kubectl auth can-i list pods -n prod-team --as=viewer-user@example.com --as-group=$VIEWER_GROUP_ID
kubectl auth can-i delete pods -n dev-team --as=viewer-user@example.com --as-group=$VIEWER_GROUP_ID
```

## 📋 Step 11: Create Role for Prod Admin

```powershell
# Create a Role for production administrators
@"
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: prod-admin-role
  namespace: prod-team
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
"@ | kubectl apply -f -

Write-Host "✅ Production admin role created"
```

## 🎯 Step 12: Create Service Account with Specific Permissions

```powershell
# Create a service account for CI/CD deployment
kubectl create serviceaccount cicd-deployer -n dev-team

# Create a Role for CI/CD deployment
@"
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cicd-deployer-role
  namespace: dev-team
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["services", "configmaps"]
  verbs: ["get", "list", "create", "update", "patch"]
"@ | kubectl apply -f -

# Bind the role to the service account
@"
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-deployer-binding
  namespace: dev-team
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: cicd-deployer-role
subjects:
- kind: ServiceAccount
  name: cicd-deployer
  namespace: dev-team
"@ | kubectl apply -f -

Write-Host "✅ CI/CD deployer service account and role created"
```

## 🔑 Step 13: Test Service Account Permissions

```powershell
# Save the service account token locally (Kubernetes 1.24+)
$CICD_TOKEN_PATH = ".\cicd-token.txt"
kubectl create token cicd-deployer -n dev-team --duration=1h | Set-Content -Path $CICD_TOKEN_PATH

# Generate a sample deployment manifest locally
$TEST_DEPLOYMENT_PATH = ".\test-deployment.yaml"
kubectl create deployment test-deploy --image=nginx:alpine -n dev-team `
  --dry-run=client -o yaml | Set-Content -Path $TEST_DEPLOYMENT_PATH

# Apply using service account token (for testing)
Write-Host "Service account token created at $CICD_TOKEN_PATH"

# Test permissions
kubectl auth can-i create deployments -n dev-team --as=system:serviceaccount:dev-team:cicd-deployer
kubectl auth can-i delete pods -n dev-team --as=system:serviceaccount:dev-team:cicd-deployer
```

## 📊 Step 14: View RBAC Resources

```powershell
# List all roles
kubectl get roles --all-namespaces

# List all rolebindings
kubectl get rolebindings --all-namespaces

# List all clusterroles (system and custom)
kubectl get clusterroles | Select-String -Pattern "developer|viewer|prod"

# List all clusterrolebindings
kubectl get clusterrolebindings | Select-String -Pattern "developer|viewer|prod"

# Describe a specific role
kubectl describe role developer-role -n dev-team

# Describe a rolebinding
kubectl describe rolebinding developer-binding -n dev-team
```

## 🔐 Step 15: Create Aggregated ClusterRole

```powershell
# Create an aggregated ClusterRole that combines multiple roles
@"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-role
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
rules:
- apiGroups: [""]
  resources: ["pods", "services", "endpoints"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-aggregated
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: [] # Rules are automatically filled by aggregation
"@ | kubectl apply -f -

Write-Host "✅ Aggregated ClusterRole created"
```

## 🎓 Summary

You have successfully:
- ✅ Integrated AKS with Azure AD
- ✅ Created Azure AD groups for different roles
- ✅ Created Roles and ClusterRoles with specific permissions
- ✅ Configured RoleBindings and ClusterRoleBindings
- ✅ Created service accounts with limited permissions
- ✅ Tested RBAC policies with kubectl auth can-i
- ✅ Implemented aggregated ClusterRoles

## 🔑 Key Concepts Learned

1. **Role vs ClusterRole**: Namespace-scoped vs cluster-wide permissions
2. **RoleBinding vs ClusterRoleBinding**: How to grant permissions
3. **Azure AD Integration**: Using AAD groups as RBAC subjects
4. **Service Accounts**: For application and CI/CD authentication
5. **Least Privilege**: Granting minimum necessary permissions
6. **Aggregation**: Combining multiple ClusterRoles

## 📝 Best Practices

- ✅ Use Azure AD groups instead of individual users
- ✅ Follow least-privilege principle (grant minimum permissions)
- ✅ Use RoleBindings (namespace-scoped) over ClusterRoleBindings when possible
- ✅ Create service accounts for applications and automation
- ✅ Document permissions and regularly audit access
- ✅ Use descriptive names for roles and bindings
- ✅ Test permissions with `kubectl auth can-i` before granting access

## 🔒 Common RBAC Patterns

**Read-Only Access:**
```yaml
verbs: ["get", "list", "watch"]
```

**Developer Access:**
```yaml
verbs: ["get", "list", "watch", "create", "update", "patch"]
```

**Admin Access:**
```yaml
verbs: ["*"]
resources: ["*"]
```

**CI/CD Deployment:**
```yaml
resources: ["deployments", "services", "configmaps"]
verbs: ["get", "list", "create", "update", "patch"]
```

## 🧹 Cleanup

```powershell
# Delete RBAC resources
kubectl delete rolebinding developer-binding cicd-deployer-binding -n dev-team
kubectl delete rolebinding -n prod-team prod-admin-role --ignore-not-found
kubectl delete clusterrolebinding viewer-binding

kubectl delete role developer-role cicd-deployer-role -n dev-team
kubectl delete role prod-admin-role -n prod-team --ignore-not-found
kubectl delete clusterrole viewer-clusterrole monitoring-role monitoring-aggregated

# Delete service accounts
kubectl delete serviceaccount cicd-deployer -n dev-team

# Delete test deployments
kubectl delete deployment nginx-app -n dev-team
kubectl delete deployment nginx-prod -n prod-team
kubectl delete service nginx-service -n dev-team
kubectl delete service nginx-prod -n prod-team

# Delete Azure AD groups (optional)
# az ad group delete --group $DEV_GROUP_ID
# az ad group delete --group $VIEWER_GROUP_ID

# Keep namespaces for future chapters or delete them
# kubectl delete namespace dev-team prod-team shared-services

Write-Host "✅ RBAC demo resources cleaned up"
```

## 🔍 Troubleshooting

**Issue**: "User cannot list resource" errors
- Check RoleBinding/ClusterRoleBinding is correctly configured
- Verify subject (user/group ID) matches Azure AD
- Ensure role has the required verbs and resources

**Issue**: Service account token not working
- Verify service account exists
- Check RoleBinding links service account to role
- Ensure token hasn't expired

**Issue**: Azure AD group not recognized
- Verify Azure AD integration is enabled on cluster
- Check group Object ID (not Display Name) is used
- Ensure user is member of the group

## 🚀 Next Steps

Proceed to:
- **[Chapter 4: Managed Identity and Federated Credentials](../chapter-4-workload-identity/README.md)**

In the next chapter, you'll dive deeper into workload identity and learn how to grant AKS pods access to Azure resources securely.
