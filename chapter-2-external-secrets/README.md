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

```bash
# Load your lab configuration
source ./chapter-0-setup/lab-config.sh

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

```bash
# Add External Secrets Helm repository
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

# Create namespace for External Secrets
kubectl create namespace external-secrets-system

# Install External Secrets Operator
helm install external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets-system \
  --set installCRDs=true

echo "✅ External Secrets Operator installed"
```

## ✅ Step 2: Verify Installation

```bash
# Wait for operator to be ready
echo "⏳ Waiting for External Secrets Operator to be ready..."
kubectl wait --for=condition=Ready pod \
  -l app.kubernetes.io/name=external-secrets \
  -n external-secrets-system \
  --timeout=120s

# Check operator pods
kubectl get pods -n external-secrets-system

# Verify CRDs are installed
echo -e "\n=== Checking External Secrets CRDs ==="
kubectl get crds | grep external-secrets

# Verify specific CRDs needed for this lab
echo -e "\n=== Verifying Required CRDs ==="
for crd in secretstores.external-secrets.io clustersecretstores.external-secrets.io externalsecrets.external-secrets.io; do
  if kubectl get crd $crd &>/dev/null; then
    echo "✅ $crd"
  else
    echo "❌ $crd - NOT FOUND"
  fi
done

# Check API resources
echo -e "\n=== Available External Secrets API Resources ==="
kubectl api-resources | grep external-secrets.io

echo -e "\n✅ External Secrets Operator verification complete"
```

## 🆔 Step 3: Create Managed Identity for ESO

```bash
# Create a dedicated managed identity for External Secrets
IDENTITY_NAME_ESO="id-external-secrets-$STUDENT_INITIALS"

az identity create \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME_ESO \
  --location $LOCATION

echo "✅ Managed identity created: $IDENTITY_NAME_ESO"

# Get identity details
IDENTITY_CLIENT_ID_ESO=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME_ESO \
  --query clientId -o tsv)

IDENTITY_PRINCIPAL_ID_ESO=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME_ESO \
  --query principalId -o tsv)

echo "Identity Client ID: $IDENTITY_CLIENT_ID_ESO"
echo "Identity Principal ID: $IDENTITY_PRINCIPAL_ID_ESO"
```

## 🔑 Step 4: Grant Key Vault Access

```bash
# Get Key Vault resource ID
KEYVAULT_RESOURCE_ID=$(az keyvault show \
  --name $KEYVAULT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

echo "Key Vault Resource ID: $KEYVAULT_RESOURCE_ID"

# Assign Key Vault Secrets User role
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee-object-id $IDENTITY_PRINCIPAL_ID_ESO \
  --assignee-principal-type ServicePrincipal \
  --scope $KEYVAULT_RESOURCE_ID

echo "✅ Granted Key Vault access to managed identity"

# Wait for role assignment to propagate
echo "⏳ Waiting 30 seconds for role assignment to propagate..."
sleep 30
```

## 📝 Step 5: Create Service Account with Workload Identity

```bash
# Create namespace for demo apps
kubectl create namespace eso-demo

# Create federated identity credential
az identity federated-credential create \
  --name "fed-external-secrets" \
  --identity-name $IDENTITY_NAME_ESO \
  --resource-group $RESOURCE_GROUP \
  --issuer $AKS_OIDC_ISSUER \
  --subject "system:serviceaccount:eso-demo:external-secrets-sa"

echo "✅ Federated credential created"

# Create service account
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-secrets-sa
  namespace: eso-demo
  annotations:
    azure.workload.identity/client-id: $IDENTITY_CLIENT_ID_ESO
  labels:
    azure.workload.identity/use: "true"
EOF

echo "✅ Service account created"
```

## 🗄️ Step 6: Create SecretStore

```bash
# Ensure TENANT_ID is set
if [ -z "$TENANT_ID" ]; then
  export TENANT_ID=$(az account show --query tenantId -o tsv)
fi

echo "Using Tenant ID: $TENANT_ID"

# Create SecretStore for Azure Key Vault
cat <<EOF | kubectl apply -f -
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
      vaultUrl: https://$KEYVAULT_NAME.vault.azure.net/
      serviceAccountRef:
        name: external-secrets-sa
EOF

echo "✅ SecretStore created"
```

## 🔍 Step 7: Verify SecretStore

```bash
# Check SecretStore status
kubectl get secretstore -n eso-demo

# Describe the SecretStore
kubectl describe secretstore azure-keyvault-store -n eso-demo

# Check for any validation errors
kubectl get events -n eso-demo --sort-by='.lastTimestamp' | grep SecretStore
```

## 🔐 Step 8: Create ExternalSecret

```bash
# Create ExternalSecret to sync specific secrets
cat <<EOF | kubectl apply -f -
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
EOF

echo "✅ ExternalSecret created"
```

## ✅ Step 9: Verify Secret Synchronization

```bash
# Wait for secret to be created
sleep 10

# Check ExternalSecret status
kubectl get externalsecret -n eso-demo

# Describe ExternalSecret
kubectl describe externalsecret app-secrets-external -n eso-demo

# Verify Kubernetes Secret was created
kubectl get secret app-secrets-synced -n eso-demo

# View secret data (base64 encoded)
kubectl get secret app-secrets-synced -n eso-demo -o yaml

# Decode and view secret values
kubectl get secret app-secrets-synced -n eso-demo -o jsonpath='{.data.db-username}' | base64 -d
echo ""
kubectl get secret app-secrets-synced -n eso-demo -o jsonpath='{.data.api-key}' | base64 -d
echo ""
```

## 🚀 Step 10: Deploy Application Using External Secrets

```bash
# Deploy pod using the synced secret
cat <<EOF | kubectl apply -f -
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
EOF

echo "✅ Application deployed"
```

## 🔍 Step 11: Verify Application

```bash
# Wait for deployment to be ready
kubectl wait --for=condition=Available deployment/app-with-external-secrets \
  -n eso-demo --timeout=60s

# Check pods
kubectl get pods -n eso-demo -l app=demo-app

# View pod logs
kubectl logs -n eso-demo -l app=demo-app --tail=10

# Exec into a pod and check environment variables
POD_NAME=$(kubectl get pod -n eso-demo -l app=demo-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n eso-demo $POD_NAME -- env | grep -E "DB_|API_"
```

## 🔄 Step 12: Test Secret Refresh

```bash
# Update a secret in Key Vault
az keyvault secret set \
  --vault-name $KEYVAULT_NAME \
  --name "api-key" \
  --value "refreshed-api-key-from-eso"

echo "✅ Secret updated in Key Vault"
echo "⏳ External Secrets Operator will sync within 1 minute (refreshInterval)"

# Watch ExternalSecret status
kubectl get externalsecret app-secrets-external -n eso-demo -w

# In a new terminal, check the Kubernetes secret after 1 minute
# kubectl get secret app-secrets-synced -n eso-demo -o jsonpath='{.data.api-key}' | base64 -d
```

## 📊 Step 13: Create ClusterSecretStore (Optional)

For cluster-wide secret management:

```bash
# Create a cluster-scoped service account
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cluster-external-secrets-sa
  namespace: external-secrets-system
  annotations:
    azure.workload.identity/client-id: $IDENTITY_CLIENT_ID_ESO
  labels:
    azure.workload.identity/use: "true"
EOF

# Update federated credential for cluster scope
az identity federated-credential create \
  --name "fed-cluster-external-secrets" \
  --identity-name $IDENTITY_NAME_ESO \
  --resource-group $RESOURCE_GROUP \
  --issuer $AKS_OIDC_ISSUER \
  --subject "system:serviceaccount:external-secrets-system:cluster-external-secrets-sa"

# Create ClusterSecretStore
cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: azure-keyvault-cluster-store
spec:
  provider:
    azurekv:
      authType: WorkloadIdentity
      tenantId: $TENANT_ID
      vaultUrl: https://$KEYVAULT_NAME.vault.azure.net/
      serviceAccountRef:
        name: cluster-external-secrets-sa
        namespace: external-secrets-system
EOF

echo "✅ ClusterSecretStore created"

# Verify ClusterSecretStore
kubectl get clustersecretstore
kubectl describe clustersecretstore azure-keyvault-cluster-store
```

## 📋 Step 14: Advanced: Sync All Secrets with dataFrom

```bash
# Create ExternalSecret that syncs all secrets from Key Vault
cat <<EOF | kubectl apply -f -
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
EOF

echo "✅ ExternalSecret with dataFrom created"

# Wait and verify
sleep 10
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

```bash
# Delete demo resources
kubectl delete deployment app-with-external-secrets -n eso-demo
kubectl delete externalsecret app-secrets-external all-secrets-external -n eso-demo
kubectl delete secretstore azure-keyvault-store -n eso-demo
kubectl delete clustersecretstore azure-keyvault-cluster-store
kubectl delete namespace eso-demo

# Optionally uninstall External Secrets Operator
# helm uninstall external-secrets -n external-secrets-system
# kubectl delete namespace external-secrets-system

echo "✅ Cleaned up External Secrets demo resources"
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
