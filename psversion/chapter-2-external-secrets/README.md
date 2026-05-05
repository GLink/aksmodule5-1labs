# Chapter 2: External Secrets Operator

In this chapter, you'll learn how to use the External Secrets Operator (ESO) as an alternative approach to integrate Azure Key Vault with AKS. ESO provides a more Kubernetes-native way to manage secrets by syncing them from external sources into Kubernetes Secret objects.

## 🎯 Learning Objectives

- Understand External Secrets Operator architecture
- Install and configure ESO in AKS
- Create SecretStore and ClusterSecretStore resources
- Deploy ExternalSecret resources
- Compare ESO with CSI driver approach
- Implement secret refresh strategies

## 📚 Prerequisites

- Completed [Chapter 0: Setup and Prerequisites](../chapter-0-setup/README.md)
- Completed [Chapter 1: Azure Key Vault CSI Driver](../chapter-1-keyvault-csi/README.md) (optional but recommended)
- AKS cluster running
- Azure Key Vault with secrets

## 🔄 Load Your Configuration

```powershell
# Load your lab configuration
. ./chapter-0-setup/lab-config.sh

# Verify cluster access
kubectl cluster-info
```

## 📖 Understanding External Secrets Operator

External Secrets Operator is a Kubernetes operator that integrates external secret management systems (like Azure Key Vault, AWS Secrets Manager, HashiCorp Vault) with Kubernetes.

**Key Benefits:**
- Kubernetes-native API (CRDs)
- Automatic sync to native Kubernetes Secrets
- Multiple provider support
- Fine-grained refresh control
- No need to mount volumes

**Key Components:**
- **SecretStore**: Namespace-scoped configuration for secret backend
- **ClusterSecretStore**: Cluster-wide secret backend configuration
- **ExternalSecret**: Defines which secrets to fetch and sync

## 📦 Step 1: Install External Secrets Operator

```powershell
# Add External Secrets Helm repository
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

# Create namespace for External Secrets
kubectl create namespace external-secrets-system

# Install External Secrets Operator
helm install external-secrets `
  external-secrets/external-secrets `
  --namespace external-secrets-system `
  --set installCRDs=true

Write-Host "✅ External Secrets Operator installed"
```

## ✅ Step 2: Verify Installation

```powershell
# Wait for operator to be ready
Write-Host "⏳ Waiting for External Secrets Operator to be ready..."
kubectl wait --for=condition=Ready pod `
  -l app.kubernetes.io/name=external-secrets `
  -n external-secrets-system `
  --timeout=120s

# Check operator pods
kubectl get pods -n external-secrets-system

# Verify CRDs are installed
Write-Host "`n=== Checking External Secrets CRDs ==="
kubectl get crds | Select-String external-secrets

# Verify specific CRDs needed for this lab
Write-Host "`n=== Verifying Required CRDs ==="
foreach ($crd in "secretstores.external-secrets.io", "clustersecretstores.external-secrets.io", "externalsecrets.external-secrets.io") {
  $null = kubectl get crd $crd 2>$null
  if ($LASTEXITCODE -eq 0) {
    Write-Host "✅ $crd"
  } else {
    Write-Host "❌ $crd - NOT FOUND"
  }
}

# Check API resources
Write-Host "`n=== Available External Secrets API Resources ==="
kubectl api-resources | Select-String external-secrets.io

Write-Host "`n✅ External Secrets Operator verification complete"
```

## 🆔 Step 3: Create Managed Identity for ESO

```powershell
# Create a dedicated managed identity for External Secrets
$IDENTITY_NAME_ESO = "id-external-secrets-$STUDENT_INITIALS"

az identity create `
  --resource-group $RESOURCE_GROUP `
  --name $IDENTITY_NAME_ESO `
  --location $LOCATION

Write-Host "✅ Managed identity created: $IDENTITY_NAME_ESO"

# Get identity details
$IDENTITY_CLIENT_ID_ESO = $(az identity show `
  --resource-group $RESOURCE_GROUP `
  --name $IDENTITY_NAME_ESO `
  --query clientId -o tsv)

$IDENTITY_PRINCIPAL_ID_ESO = $(az identity show `
  --resource-group $RESOURCE_GROUP `
  --name $IDENTITY_NAME_ESO `
  --query principalId -o tsv)

Write-Host "Identity Client ID: $IDENTITY_CLIENT_ID_ESO"
Write-Host "Identity Principal ID: $IDENTITY_PRINCIPAL_ID_ESO"
```

## 🔑 Step 4: Grant Key Vault Access

```powershell
# Get Key Vault resource ID
$KEYVAULT_RESOURCE_ID = $(az keyvault show `
  --name $KEYVAULT_NAME `
  --resource-group $RESOURCE_GROUP `
  --query id -o tsv)

Write-Host "Key Vault Resource ID: $KEYVAULT_RESOURCE_ID"

# Assign Key Vault Secrets User role
az role assignment create `
  --role "Key Vault Secrets User" `
  --assignee-object-id $IDENTITY_PRINCIPAL_ID_ESO `
  --assignee-principal-type ServicePrincipal `
  --scope $KEYVAULT_RESOURCE_ID

Write-Host "✅ Granted Key Vault access to managed identity"

# Wait for role assignment to propagate
Write-Host "⏳ Waiting 30 seconds for role assignment to propagate..."
Start-Sleep -Seconds 30
```

## 📝 Step 5: Create Service Account with Workload Identity

```powershell
# Create namespace for demo apps
kubectl create namespace eso-demo

# Create federated identity credential
az identity federated-credential create `
  --name "fed-external-secrets" `
  --identity-name $IDENTITY_NAME_ESO `
  --resource-group $RESOURCE_GROUP `
  --issuer $AKS_OIDC_ISSUER `
  --subject "system:serviceaccount:eso-demo:external-secrets-sa"

Write-Host "✅ Federated credential created"

# Create service account
@"
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-secrets-sa
  namespace: eso-demo
  annotations:
    azure.workload.identity/client-id: $IDENTITY_CLIENT_ID_ESO
  labels:
    azure.workload.identity/use: "true"
"@ | kubectl apply -f -

Write-Host "✅ Service account created"
```

## 🗄️ Step 6: Create SecretStore

```powershell
# Ensure TENANT_ID is set
if (-not $TENANT_ID) {
  $TENANT_ID = $(az account show --query tenantId -o tsv)
}

Write-Host "Using Tenant ID: $TENANT_ID"

# Create SecretStore for Azure Key Vault
@"
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: azure-keyvault-store
  namespace: eso-demo
spec:
  provider:
    azurekv:
      authType: WorkloadIdentity
      tenantId: $TENANT_ID
      vaultUrl: https://${KEYVAULT_NAME}.vault.azure.net/
      serviceAccountRef:
        name: external-secrets-sa
"@ | kubectl apply -f -

Write-Host "✅ SecretStore created"
```

## 🔍 Step 7: Verify SecretStore

```powershell
# Check SecretStore status
kubectl get secretstore -n eso-demo

# Describe the SecretStore
kubectl describe secretstore azure-keyvault-store -n eso-demo

# Check for any validation errors
kubectl get events -n eso-demo --sort-by='.lastTimestamp' | Select-String SecretStore
```

## 🔐 Step 8: Create ExternalSecret

```powershell
# Create ExternalSecret to sync specific secrets
@"
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: app-secrets-external
  namespace: eso-demo
spec:
  refreshInterval: 1m
  secretStoreRef:
    name: azure-keyvault-store
    kind: SecretStore
  target:
    name: app-secrets-synced
    creationPolicy: Owner
  data:
  - secretKey: db-username
    remoteRef:
      key: database-username
  - secretKey: db-password
    remoteRef:
      key: database-password
  - secretKey: api-key
    remoteRef:
      key: api-key
"@ | kubectl apply -f -

Write-Host "✅ ExternalSecret created"
```

## ✅ Step 9: Verify Secret Synchronization

```powershell
# Wait for secret to be created
Start-Sleep -Seconds 10

# Check ExternalSecret status
kubectl get externalsecret -n eso-demo

# Describe ExternalSecret
kubectl describe externalsecret app-secrets-external -n eso-demo

# Verify Kubernetes Secret was created
kubectl get secret app-secrets-synced -n eso-demo

# View secret data (base64 encoded)
kubectl get secret app-secrets-synced -n eso-demo -o yaml

# Decode and view secret values
$dbUsername = kubectl get secret app-secrets-synced -n eso-demo -o jsonpath='{.data.db-username}'
Write-Host ([System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($dbUsername)))
Write-Host ""
$apiKey = kubectl get secret app-secrets-synced -n eso-demo -o jsonpath='{.data.api-key}'
Write-Host ([System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($apiKey)))
Write-Host ""
```

## 🚀 Step 10: Deploy Application Using External Secrets

```powershell
# Deploy pod using the synced secret
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-external-secrets
  namespace: eso-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
      - name: app
        image: nginx:alpine
        env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: app-secrets-synced
              key: db-username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets-synced
              key: db-password
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets-synced
              key: api-key
        command:
        - sh
        - -c
        - |
          echo "Application started with secrets from External Secrets Operator"
          echo "DB Username: \$DB_USERNAME"
          echo "API Key: \$API_KEY"
          echo "Secrets loaded successfully!"
          nginx -g 'daemon off;'
"@ | kubectl apply -f -

Write-Host "✅ Application deployed"
```

## 🔍 Step 11: Verify Application

```powershell
# Wait for deployment to be ready
kubectl wait --for=condition=Available deployment/app-with-external-secrets `
  -n eso-demo --timeout=60s

# Check pods
kubectl get pods -n eso-demo -l app=demo-app

# View pod logs
kubectl logs -n eso-demo -l app=demo-app --tail=10

# Exec into a pod and check environment variables
$POD_NAME = $(kubectl get pod -n eso-demo -l app=demo-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n eso-demo $POD_NAME -- env | Select-String "DB_|API_"
```

## 🔄 Step 12: Test Secret Refresh

```powershell
# Update a secret in Key Vault
az keyvault secret set `
  --vault-name $KEYVAULT_NAME `
  --name "api-key" `
  --value "refreshed-api-key-from-eso"

Write-Host "✅ Secret updated in Key Vault"
Write-Host "⏳ External Secrets Operator will sync within 1 minute (refreshInterval)"

# Watch ExternalSecret status
kubectl get externalsecret app-secrets-external -n eso-demo -w

# In a new terminal, check the Kubernetes secret after 1 minute
# kubectl get secret app-secrets-synced -n eso-demo -o jsonpath='{.data.api-key}'
```

## 📊 Step 13: Create ClusterSecretStore (Optional)

For cluster-wide secret management:

```powershell
# Create a cluster-scoped service account
@"
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cluster-external-secrets-sa
  namespace: external-secrets-system
  annotations:
    azure.workload.identity/client-id: $IDENTITY_CLIENT_ID_ESO
  labels:
    azure.workload.identity/use: "true"
"@ | kubectl apply -f -

# Update federated credential for cluster scope
az identity federated-credential create `
  --name "fed-cluster-external-secrets" `
  --identity-name $IDENTITY_NAME_ESO `
  --resource-group $RESOURCE_GROUP `
  --issuer $AKS_OIDC_ISSUER `
  --subject "system:serviceaccount:external-secrets-system:cluster-external-secrets-sa"

# Create ClusterSecretStore
@"
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: azure-keyvault-cluster-store
spec:
  provider:
    azurekv:
      authType: WorkloadIdentity
      tenantId: $TENANT_ID
      vaultUrl: https://${KEYVAULT_NAME}.vault.azure.net/
      serviceAccountRef:
        name: cluster-external-secrets-sa
        namespace: external-secrets-system
"@ | kubectl apply -f -

Write-Host "✅ ClusterSecretStore created"

# Verify ClusterSecretStore
kubectl get clustersecretstore
kubectl describe clustersecretstore azure-keyvault-cluster-store
```

## 📋 Step 14: Advanced: Sync All Secrets with dataFrom

```powershell
# Create ExternalSecret that syncs all secrets from Key Vault
@"
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: all-secrets-external
  namespace: eso-demo
spec:
  refreshInterval: 5m
  secretStoreRef:
    name: azure-keyvault-store
    kind: SecretStore
  target:
    name: all-keyvault-secrets
    creationPolicy: Owner
  dataFrom:
  - find:
      name:
        regexp: ".*"
"@ | kubectl apply -f -

Write-Host "✅ ExternalSecret with dataFrom created"

# Wait and verify
Start-Sleep -Seconds 10
kubectl get secret all-keyvault-secrets -n eso-demo
kubectl describe secret all-keyvault-secrets -n eso-demo
```

## 🎓 Summary

You have successfully:
- ✅ Installed External Secrets Operator
- ✅ Created managed identity with workload identity
- ✅ Configured SecretStore and ClusterSecretStore
- ✅ Created ExternalSecret resources
- ✅ Deployed applications using synced secrets
- ✅ Tested secret refresh functionality
- ✅ Compared ESO with CSI driver approach

## 🔑 Key Concepts Learned

1. **External Secrets Operator**: Kubernetes operator for external secret management
2. **SecretStore vs ClusterSecretStore**: Namespace vs cluster-scoped configuration
3. **ExternalSecret**: CRD that defines secret sync behavior
4. **Refresh Interval**: Automatic secret rotation without pod restart
5. **dataFrom**: Bulk secret synchronization

## 📊 Comparison: ESO vs CSI Driver

| Feature             | External Secrets Operator  | CSI Secret Store Driver              |
| ------------------- | -------------------------- | ------------------------------------ |
| **Secret Format**   | Native Kubernetes Secrets  | Mounted files (optional K8s secrets) |
| **Refresh Method**  | Controller polling         | Pod restart or polling               |
| **API Style**       | Kubernetes CRDs            | Volume mounts                        |
| **Multi-provider**  | Yes (unified API)          | Provider-specific                    |
| **Use as Env Vars** | Yes (native)               | Only with sync enabled               |
| **Cluster Scope**   | ClusterSecretStore support | Node-level daemonset                 |
| **Resource Usage**  | Central operator           | Per-node daemonset                   |

**When to use ESO:**
- Need native Kubernetes Secrets
- Want centralized secret management
- Require fine-grained refresh control
- Use multiple secret providers

**When to use CSI Driver:**
- Prefer file-based secrets
- Want secrets outside etcd
- Need per-pod secret access
- Minimal cluster-wide components

## 📝 Best Practices

- ✅ Use ClusterSecretStore for shared configurations
- ✅ Set appropriate refresh intervals (balance freshness vs API calls)
- ✅ Use `creationPolicy: Owner` for automatic cleanup
- ✅ Monitor ExternalSecret status regularly
- ✅ Use RBAC to restrict ExternalSecret creation
- ✅ Consider secret versioning for rollback capability

## 🧹 Cleanup

```powershell
# Delete demo resources
kubectl delete deployment app-with-external-secrets -n eso-demo
kubectl delete externalsecret app-secrets-external all-secrets-external -n eso-demo
kubectl delete secretstore azure-keyvault-store -n eso-demo
kubectl delete clustersecretstore azure-keyvault-cluster-store
kubectl delete namespace eso-demo

# Optionally uninstall External Secrets Operator
# helm uninstall external-secrets -n external-secrets-system
# kubectl delete namespace external-secrets-system

Write-Host "✅ Cleaned up External Secrets demo resources"
```

## 🔍 Troubleshooting

**Issue**: ExternalSecret status shows "SecretSyncedError"
- Check SecretStore status and configuration
- Verify workload identity and role assignments
- Check operator logs: `kubectl logs -n external-secrets-system -l app.kubernetes.io/name=external-secrets`

**Issue**: Secrets not refreshing
- Verify refreshInterval is set
- Check ExternalSecret conditions
- Ensure network connectivity to Key Vault

**Issue**: Permission denied errors
- Verify managed identity has correct role assignment
- Check federated credential subject matches service account
- Confirm serviceAccountRef in SecretStore is correct

## 🚀 Next Steps

Proceed to:
- **[Chapter 3: Kubernetes RBAC with Azure AD](../chapter-3-rbac-azuread/README.md)**

In the next chapter, you'll learn how to implement fine-grained access control using Kubernetes RBAC integrated with Azure AD.
