# Chapter 4: User Assigned Managed Identity and Federated Credentials

In this chapter, you'll learn how to use Azure Workload Identity to grant AKS pods access to Azure resources without managing credentials. This uses OpenID Connect (OIDC) federation to establish trust between Kubernetes and Azure AD.

## 🎯 Learning Objectives

- Understand Workload Identity architecture
- Create User Assigned Managed Identities
- Configure federated identity credentials
- Grant Azure resource permissions to managed identities
- Deploy pods that access Azure resources
- Compare with other authentication methods

## 📚 Prerequisites

- Completed [Chapter 0: Setup and Prerequisites](../chapter-0-setup/README.md)
- AKS cluster with OIDC issuer and workload identity enabled
- Basic understanding of Azure RBAC

## 🔄 Load Your Configuration

```bash
# Load your lab configuration
source ./chapter-0-setup/lab-config.sh

# Verify OIDC issuer is enabled
echo "AKS OIDC Issuer: $AKS_OIDC_ISSUER"
```

## 📖 Understanding Workload Identity

**Workload Identity** allows Kubernetes workloads to access Azure resources using Azure AD identities without storing credentials.

**How it works:**
1. Pod uses a Kubernetes service account with annotations
2. Service account is linked to Azure Managed Identity via federated credential
3. Pod requests Azure AD token using OIDC federation
4. Azure AD validates the token and grants access

**Benefits:**
- No stored credentials or secrets
- Automatic token rotation
- Fine-grained access control
- Audit trail in Azure AD

## 🆔 Step 1: Create User Assigned Managed Identity

```bash
# Create managed identity for blob storage access
IDENTITY_NAME="id-blob-access-$STUDENT_INITIALS"

az identity create \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME \
  --location $LOCATION

echo "✅ Managed identity created: $IDENTITY_NAME"

# Get identity details
IDENTITY_CLIENT_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME \
  --query clientId -o tsv)

IDENTITY_PRINCIPAL_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME \
  --query principalId -o tsv)

echo "Client ID: $IDENTITY_CLIENT_ID"
echo "Principal ID: $IDENTITY_PRINCIPAL_ID"
```

## 📦 Step 2: Create Azure Storage Account

```bash
# Create storage account for testing
STORAGE_ACCOUNT="stakslab${STUDENT_INITIALS}$(date +%s | tail -c 5)"

az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS

echo "✅ Storage account created: $STORAGE_ACCOUNT"

# Get your current user's object ID
CURRENT_USER_OBJECT_ID=$(az ad signed-in-user show --query id -o tsv)

# Get storage account resource ID
STORAGE_ACCOUNT_ID=$(az storage account show \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

# Grant yourself Storage Blob Data Contributor role to upload files
az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee-object-id $CURRENT_USER_OBJECT_ID \
  --assignee-principal-type User \
  --scope $STORAGE_ACCOUNT_ID

echo "✅ Granted Storage Blob Data Contributor role to current user"
echo "⏳ Waiting 30 seconds for role assignment to propagate..."
sleep 30

# Create a blob container
az storage container create \
  --name demo-container \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

echo "✅ Blob container created"

# Upload a test file
echo "Hello from AKS Workload Identity!" > /tmp/test-file.txt

az storage blob upload \
  --account-name $STORAGE_ACCOUNT \
  --container-name demo-container \
  --name test-file.txt \
  --file /tmp/test-file.txt \
  --auth-mode login

echo "✅ Test file uploaded"
```

## 🔑 Step 3: Grant Storage Permissions to Managed Identity

```bash
# Get storage account resource ID
STORAGE_ACCOUNT_ID=$(az storage account show \
  --name $STORAGE_ACCOUNT \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

# Assign Storage Blob Data Reader role (for reading blob data)
az role assignment create \
  --role "Storage Blob Data Reader" \
  --assignee-object-id $IDENTITY_PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --scope $STORAGE_ACCOUNT_ID

echo "✅ Storage Blob Data Reader role assigned"

# Assign Reader role (for reading storage account metadata)
az role assignment create \
  --role "Reader" \
  --assignee-object-id $IDENTITY_PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --scope $STORAGE_ACCOUNT_ID

echo "✅ Reader role assigned for storage account metadata"

# Wait for role propagation
echo "⏳ Waiting 30 seconds for role assignments to propagate..."
sleep 30
```

## 📝 Step 4: Create Namespace and Service Account

```bash
# Create namespace
kubectl create namespace workload-identity-demo

# Get tenant ID if not already set
if [ -z "$TENANT_ID" ]; then
  export TENANT_ID=$(az account show --query tenantId -o tsv)
  echo "TENANT_ID set to: $TENANT_ID"
fi

# Create service account with workload identity annotation
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: blob-reader-sa
  namespace: workload-identity-demo
  annotations:
    azure.workload.identity/client-id: $IDENTITY_CLIENT_ID
    azure.workload.identity/tenant-id: $TENANT_ID
  labels:
    azure.workload.identity/use: "true"
EOF

echo "✅ Service account created"

# Verify service account
kubectl get serviceaccount blob-reader-sa -n workload-identity-demo -o yaml
```

## 🔗 Step 5: Create Federated Identity Credential

```bash
# Get AKS OIDC Issuer URL (if not already set)
if [ -z "$AKS_OIDC_ISSUER" ]; then
  export AKS_OIDC_ISSUER=$(az aks show \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --query 'oidcIssuerProfile.issuerUrl' -o tsv)
fi

echo "AKS OIDC Issuer: $AKS_OIDC_ISSUER"

# Create federated credential linking K8s SA to Azure MI
az identity federated-credential create \
  --name "fed-blob-reader" \
  --identity-name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --issuer $AKS_OIDC_ISSUER \
  --subject "system:serviceaccount:workload-identity-demo:blob-reader-sa"

echo "✅ Federated credential created"

# Verify federated credential
echo -e "\n=== Verifying Federated Credential ==="
az identity federated-credential show \
  --name "fed-blob-reader" \
  --identity-name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "{name:name, issuer:issuer, subject:subject}" -o table

echo -e "\n⏳ Waiting 30 seconds for federated credential to propagate..."
sleep 30
```

## 🚀 Step 6: Deploy Pod with Workload Identity

```bash
# Create a temporary YAML file - Part 1: Pod metadata and first part of script
cat > /tmp/blob-reader-pod.yaml <<'PART1'
apiVersion: v1
kind: Pod
metadata:
  name: blob-reader-pod
  namespace: workload-identity-demo
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: blob-reader-sa
  containers:
  - name: azure-cli
    image: mcr.microsoft.com/azure-cli:latest
    command:
    - /bin/bash
    - -c
    - |
      echo "=== Testing Workload Identity ==="
      echo "Pod is starting..."
      
      # Get Azure CLI version
      az version
      
      # Display workload identity environment variables
      echo -e "\n=== Workload Identity Environment ==="
      echo "AZURE_CLIENT_ID: $AZURE_CLIENT_ID"
      echo "AZURE_TENANT_ID: $AZURE_TENANT_ID"
      echo "AZURE_FEDERATED_TOKEN_FILE: $AZURE_FEDERATED_TOKEN_FILE"
      
      # Login using workload identity with federated token
      echo -e "\n=== Attempting login with workload identity ==="
      az login --service-principal \
        -u $AZURE_CLIENT_ID \
        -t $AZURE_TENANT_ID \
        --federated-token "$(cat $AZURE_FEDERATED_TOKEN_FILE)"
PART1

# Append Part 2: Middle section of script
cat >> /tmp/blob-reader-pod.yaml <<'PART2'
      
      # List storage accounts (should work if permissions are granted)
      echo -e "\n=== Listing storage account ==="
      az storage account show \
        --name $STORAGE_ACCOUNT \
        --resource-group $RESOURCE_GROUP
      
      # List blobs in container
      echo -e "\n=== Listing blobs in demo-container ==="
      az storage blob list \
        --account-name $STORAGE_ACCOUNT \
        --container-name demo-container \
        --auth-mode login \
        --output table
      
      # Download a blob
      echo -e "\n=== Downloading test file ==="
      az storage blob download \
        --account-name $STORAGE_ACCOUNT \
        --container-name demo-container \
        --name test-file.txt \
        --file /tmp/downloaded-file.txt \
        --auth-mode login
PART2

# Append Part 3: Final section with environment variables (expand shell variables here)
cat >> /tmp/blob-reader-pod.yaml <<PART3
      
      echo -e "\n=== File contents ==="
      cat /tmp/downloaded-file.txt
      
      echo -e "\n=== Workload Identity test completed successfully! ==="
      
      # Keep pod running
      sleep infinity
    env:
    - name: STORAGE_ACCOUNT
      value: "$STORAGE_ACCOUNT"
    - name: RESOURCE_GROUP
      value: "$RESOURCE_GROUP"
    - name: IDENTITY_CLIENT_ID
      value: "$IDENTITY_CLIENT_ID"
PART3

# Apply the complete YAML file
kubectl apply -f /tmp/blob-reader-pod.yaml

echo "✅ Pod deployed"
```

## 🔍 Step 7: Verify Workload Identity

```bash
# Wait for pod to be ready
echo "⏳ Waiting for pod to be ready..."
kubectl wait --for=condition=Ready pod/blob-reader-pod \
  -n workload-identity-demo --timeout=120s

# View pod logs
kubectl logs blob-reader-pod -n workload-identity-demo

# Check if pod successfully accessed storage
kubectl logs blob-reader-pod -n workload-identity-demo | grep -i "Hello from AKS"
```

## 🧪 Step 8: Test Without Workload Identity Label

```bash
# Deploy a pod WITHOUT the workload identity label (should fail)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: blob-reader-no-wi
  namespace: workload-identity-demo
spec:
  serviceAccountName: blob-reader-sa
  containers:
  - name: azure-cli
    image: mcr.microsoft.com/azure-cli:latest
    command:
    - /bin/bash
    - -c
    - |
      echo "Attempting to access Azure without workload identity label..."
      az login --identity --client-id $IDENTITY_CLIENT_ID 2>&1 || echo "Failed as expected"
      sleep 60
    env:
    - name: IDENTITY_CLIENT_ID
      value: "$IDENTITY_CLIENT_ID"
EOF

# Check logs (should show failure because workload identity env vars are not injected)
sleep 10
kubectl logs blob-reader-no-wi -n workload-identity-demo

# Cleanup
kubectl delete pod blob-reader-no-wi -n workload-identity-demo
```

## 🎯 Step 9: Create Multiple Identities for Different Resources

```bash
# Create identity for Azure SQL access
IDENTITY_SQL="id-sql-access-$STUDENT_INITIALS"

az identity create \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_SQL \
  --location $LOCATION

IDENTITY_SQL_CLIENT_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_SQL \
  --query clientId -o tsv)

echo "✅ SQL identity created: $IDENTITY_SQL"

# Create service account for SQL access
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sql-access-sa
  namespace: workload-identity-demo
  annotations:
    azure.workload.identity/client-id: $IDENTITY_SQL_CLIENT_ID
  labels:
    azure.workload.identity/use: "true"
EOF

# Create federated credential
az identity federated-credential create \
  --name "fed-sql-access" \
  --identity-name $IDENTITY_SQL \
  --resource-group $RESOURCE_GROUP \
  --issuer $AKS_OIDC_ISSUER \
  --subject "system:serviceaccount:workload-identity-demo:sql-access-sa"

echo "✅ SQL access service account and federated credential created"
```

## 📊 Step 10: View Workload Identity Configuration

```bash
# List all service accounts with workload identity
kubectl get serviceaccounts -n workload-identity-demo \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.azure\.workload\.identity/client-id}{"\n"}{end}'

# Describe service account
kubectl describe sa blob-reader-sa -n workload-identity-demo

# List federated credentials for the identity
az identity federated-credential list \
  --identity-name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --output table
```

## 🔐 Step 11: Use Workload Identity with Deployment

```bash
# Create a deployment that uses workload identity
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blob-reader-deployment
  namespace: workload-identity-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: blob-reader
  template:
    metadata:
      labels:
        app: blob-reader
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: blob-reader-sa
      containers:
      - name: app
        image: mcr.microsoft.com/azure-cli:latest
        command:
        - /bin/bash
        - -c
        - |
          echo "Blob reader app started"
          az login --service-principal \
            -u \$AZURE_CLIENT_ID \
            -t \$AZURE_TENANT_ID \
            --federated-token "\$(cat \$AZURE_FEDERATED_TOKEN_FILE)"
          while true; do
            echo "Checking blob storage at \$(date)"
            az storage blob list \
              --account-name \$STORAGE_ACCOUNT \
              --container-name demo-container \
              --auth-mode login \
              --output table
            sleep 60
          done
        env:
        - name: STORAGE_ACCOUNT
          value: "$STORAGE_ACCOUNT"
EOF

echo "✅ Deployment created with workload identity"

# Wait and verify
kubectl wait --for=condition=Available deployment/blob-reader-deployment \
  -n workload-identity-demo --timeout=120s

kubectl get pods -n workload-identity-demo -l app=blob-reader
```

## 🎓 Summary

You have successfully:
- ✅ Created User Assigned Managed Identities
- ✅ Configured federated identity credentials with OIDC
- ✅ Granted Azure RBAC permissions to managed identities
- ✅ Created Kubernetes service accounts with workload identity
- ✅ Deployed pods that access Azure resources without credentials
- ✅ Tested workload identity with Azure Storage

## 🔑 Key Concepts Learned

1. **Workload Identity**: Pod-level Azure authentication using OIDC
2. **Federated Credentials**: Trust relationship between K8s and Azure AD
3. **Service Account Annotations**: Link K8s SA to Azure MI
4. **Pod Labels**: Enable workload identity injection
5. **Zero Credentials**: No secrets stored in cluster or pods

## 📊 Authentication Methods Comparison

| Method                        | Credentials | Rotation  | Scope        | Complexity |
| ----------------------------- | ----------- | --------- | ------------ | ---------- |
| **Workload Identity**         | None        | Automatic | Pod-level    | Low        |
| **Pod Identity** (deprecated) | None        | Automatic | Pod-level    | Medium     |
| **Service Principal**         | Secret      | Manual    | Cluster-wide | High       |
| **Managed Identity (VM)**     | None        | Automatic | Node-level   | Medium     |

**Workload Identity is the recommended approach for AKS!**

## 📝 Best Practices

- ✅ Use workload identity for all Azure resource access
- ✅ Create separate managed identities per application/service
- ✅ Grant least-privilege Azure RBAC permissions
- ✅ Always include the `azure.workload.identity/use: "true"` label
- ✅ Use unique service accounts per workload
- ✅ Audit managed identity access regularly
- ✅ Test with and without the label to verify configuration

## 🔒 Required Components

For workload identity to work, you need:
1. ✅ AKS cluster with `--enable-oidc-issuer` and `--enable-workload-identity`
2. ✅ User Assigned Managed Identity
3. ✅ Federated identity credential linking K8s SA to Azure MI
4. ✅ Service account with `azure.workload.identity/client-id` annotation
5. ✅ Pod with `azure.workload.identity/use: "true"` label
6. ✅ Pod using the annotated service account

## 🧹 Cleanup

```bash
# Delete Kubernetes resources
kubectl delete deployment blob-reader-deployment -n workload-identity-demo
kubectl delete pod blob-reader-pod -n workload-identity-demo
kubectl delete serviceaccount blob-reader-sa sql-access-sa -n workload-identity-demo
kubectl delete namespace workload-identity-demo

# Delete Azure resources
az storage account delete --name $STORAGE_ACCOUNT --resource-group $RESOURCE_GROUP --yes

# Optionally delete managed identities
# az identity delete --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP
# az identity delete --name $IDENTITY_SQL --resource-group $RESOURCE_GROUP

echo "✅ Workload identity demo resources cleaned up"
```

## 🔍 Troubleshooting

**Issue**: Pod can't authenticate to Azure
- Verify OIDC issuer is enabled: `az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --query oidcIssuerProfile`
- Check service account has correct annotation
- Ensure pod has the `azure.workload.identity/use: "true"` label
- Verify federated credential subject matches service account
- Check pod logs for authentication errors

**Issue**: "Forbidden" or "Unauthorized" errors
- Verify managed identity has the required Azure RBAC role
- Check role assignment scope is correct
- Wait 1-2 minutes for role assignments to propagate

**Issue**: Workload identity webhook not injecting tokens
- Check webhook is running: `kubectl get pods -n kube-system -l azure-workload-identity.io/system=true`
- Verify MutatingWebhookConfiguration exists
- Check pod events for admission errors

## 🚀 Next Steps

Proceed to:
- **[Chapter 5: Node Selectors and Affinity](../chapter-5-node-selectors/README.md)**

In the next chapter, you'll learn how to control pod placement on specific nodes using node selectors and affinity rules.
