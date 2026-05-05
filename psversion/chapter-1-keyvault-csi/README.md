# Chapter 1: Azure Key Vault CSI Driver

In this chapter, you'll learn how to integrate Azure Key Vault with AKS using the Secrets Store CSI (Container Storage Interface) driver. This allows your applications to access secrets, certificates, and keys stored in Azure Key Vault as mounted volumes.

## 🎯 Learning Objectives

- Understand the CSI Secret Store driver architecture
- Configure Azure Key Vault access from AKS pods
- Mount Key Vault secrets as volumes
- Sync Key Vault secrets to Kubernetes secrets
- Implement auto-rotation of secrets

## 📚 Prerequisites

- Completed [Chapter 0: Setup and Prerequisites](../chapter-0-setup/README.md)
- AKS cluster with CSI driver enabled
- Azure Key Vault created and populated with secrets

## 🔄 Load Your Configuration

```powershell
# Load your lab configuration
. .\chapter-0-setup\lab-config.sh

# Ensure TENANT_ID is set (required for SecretProviderClass)
if (-not $TENANT_ID) {
  $TENANT_ID = az account show --query tenantId -o tsv
}

Write-Host "Tenant ID: $TENANT_ID"

# Verify Key Vault exists
az keyvault show --name $KEYVAULT_NAME --query name -o tsv
```

## 📖 Understanding CSI Secret Store Driver

The Azure Key Vault Provider for Secrets Store CSI Driver allows Kubernetes to mount multiple secrets, keys, and certificates stored in Azure Key Vault into pods as volumes.

**Key Benefits:**
- Secrets stored centrally in Key Vault (not in etcd)
- Automatic rotation of secrets
- Pod identity-based access control
- No application code changes required

## 🔧 Step 1: Verify CSI Driver Installation

The CSI driver was enabled during cluster creation. Let's verify it's running:

```powershell
# Check CSI driver pods
kubectl get pods -n kube-system -l app=secrets-store-csi-driver

# Check the containers in the CSI driver pods (includes Azure provider as sidecar)
Write-Host "`nCSI Driver Pod Containers:"
kubectl get pods -n kube-system -l app=secrets-store-csi-driver -o jsonpath='{range .items[0].spec.containers[*]}{.name}{"\n"}{end}'

# View CSI driver daemonset
kubectl get daemonset -n kube-system aks-secrets-store-csi-driver

# Check CSI driver arguments
Write-Host "`nCSI Driver Configuration:"
$driverArgs = kubectl get daemonset aks-secrets-store-csi-driver -n kube-system -o jsonpath='{.spec.template.spec.containers[?(@.name=="secrets-store")].args}'
try {
  $driverArgs | ConvertFrom-Json | ForEach-Object { $_ }
} catch {
  Write-Host $driverArgs
}

Write-Host "`n✅ CSI driver verification complete"
Write-Host "Note: Secret rotation is controlled by the pod annotation 'secrets-store.csi.k8s.io/rotation-poll-interval'"
```

## 🆔 Step 2: Configure Workload Identity for Key Vault Access

We'll use Workload Identity to allow pods to authenticate to Azure Key Vault:

```powershell
# Create a managed identity for Key Vault access
$IDENTITY_NAME = "id-keyvault-csi-$STUDENT_INITIALS"

az identity create `
  --resource-group $RESOURCE_GROUP `
  --name $IDENTITY_NAME `
  --location $LOCATION

Write-Host "✅ Managed identity created: $IDENTITY_NAME"

# Get identity details
$IDENTITY_CLIENT_ID = az identity show `
  --resource-group $RESOURCE_GROUP `
  --name $IDENTITY_NAME `
  --query clientId -o tsv

$IDENTITY_PRINCIPAL_ID = az identity show `
  --resource-group $RESOURCE_GROUP `
  --name $IDENTITY_NAME `
  --query principalId -o tsv

Write-Host "Identity Client ID: $IDENTITY_CLIENT_ID"
Write-Host "Identity Principal ID: $IDENTITY_PRINCIPAL_ID"
```

## 🔑 Step 3: Grant Key Vault Access to Managed Identity

```powershell
# Get Key Vault resource ID
$KEYVAULT_RESOURCE_ID = az keyvault show `
  --name $KEYVAULT_NAME `
  --resource-group $RESOURCE_GROUP `
  --query id -o tsv

Write-Host "Key Vault Resource ID: $KEYVAULT_RESOURCE_ID"

# Assign Key Vault Secrets User role to the managed identity
az role assignment create `
  --role "Key Vault Secrets User" `
  --assignee-object-id $IDENTITY_PRINCIPAL_ID `
  --assignee-principal-type ServicePrincipal `
  --scope $KEYVAULT_RESOURCE_ID

Write-Host "✅ Granted Key Vault access to managed identity"

# Wait for role assignment to propagate
Write-Host "⏳ Waiting 30 seconds for role assignment to propagate..."
Start-Sleep -Seconds 30
```

## 🔗 Step 4: Create Federated Identity Credential

Link the managed identity to a Kubernetes service account:

```powershell
# Create namespace for demo apps
kubectl create namespace keyvault-demo

# Get AKS OIDC Issuer URL (if not already set)
if (-not $AKS_OIDC_ISSUER) {
  $AKS_OIDC_ISSUER = az aks show `
    --resource-group $RESOURCE_GROUP `
    --name $CLUSTER_NAME `
    --query 'oidcIssuerProfile.issuerUrl' -o tsv
}

Write-Host "AKS OIDC Issuer: $AKS_OIDC_ISSUER"

# Create federated identity credential
az identity federated-credential create `
  --name "fed-keyvault-csi" `
  --identity-name $IDENTITY_NAME `
  --resource-group $RESOURCE_GROUP `
  --issuer $AKS_OIDC_ISSUER `
  --subject "system:serviceaccount:keyvault-demo:workload-identity-sa"

Write-Host "✅ Federated credential created"
```

## 📝 Step 5: Create Kubernetes Service Account

Create a service account with workload identity annotations:

```powershell
# Create service account YAML
@"
apiVersion: v1
kind: ServiceAccount
metadata:
  name: workload-identity-sa
  namespace: keyvault-demo
  annotations:
    azure.workload.identity/client-id: $IDENTITY_CLIENT_ID
"@ | kubectl apply -f -

Write-Host "✅ Service account created with workload identity"
```

## 🗂️ Step 6: Create SecretProviderClass

The SecretProviderClass defines which secrets to retrieve from Key Vault:

```powershell
# Ensure TENANT_ID is set
if (-not $TENANT_ID) {
  $TENANT_ID = az account show --query tenantId -o tsv
}

Write-Host "Using Tenant ID: $TENANT_ID"

# Create SecretProviderClass YAML
@"
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-keyvault-secrets
  namespace: keyvault-demo
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "false"
    clientID: $IDENTITY_CLIENT_ID
    keyvaultName: $KEYVAULT_NAME
    cloudName: ""
    objects: |
      array:
        - |
          objectName: database-username
          objectType: secret
          objectVersion: ""
        - |
          objectName: database-password
          objectType: secret
          objectVersion: ""
        - |
          objectName: api-key
          objectType: secret
          objectVersion: ""
    tenantId: $TENANT_ID
"@ | kubectl apply -f -

Write-Host "✅ SecretProviderClass created"
```

## 🚀 Step 7: Deploy Application Using CSI Driver

Deploy a pod that mounts Key Vault secrets as volumes:

```powershell
# Create deployment YAML
@'
apiVersion: v1
kind: Pod
metadata:
  name: nginx-keyvault-csi
  namespace: keyvault-demo
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: workload-identity-sa
  containers:
  - name: nginx
    image: nginx:alpine
    volumeMounts:
    - name: secrets-store
      mountPath: "/mnt/secrets"
      readOnly: true
    command:
    - "/bin/sh"
    - "-c"
    - |
      echo "Starting nginx with Key Vault secrets..."
      while true; do
        echo "=== Secrets from Azure Key Vault ==="
        echo "Database Username: $(cat /mnt/secrets/database-username)"
        echo "Database Password: $(cat /mnt/secrets/database-password)"
        echo "API Key: $(cat /mnt/secrets/api-key)"
        echo "Sleeping for 30 seconds..."
        sleep 30
      done
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: azure-keyvault-secrets
'@ | kubectl apply -f -

Write-Host "✅ Pod deployed with CSI driver volume"
```

## 🔍 Step 8: Verify Secrets are Mounted

```powershell
# Wait for pod to be ready
Write-Host "⏳ Waiting for pod to be ready..."
kubectl wait --for=condition=Ready pod/nginx-keyvault-csi -n keyvault-demo --timeout=60s

# Check pod status
kubectl get pod nginx-keyvault-csi -n keyvault-demo

# View pod logs
kubectl logs nginx-keyvault-csi -n keyvault-demo --tail=20

# Exec into pod and check mounted secrets
kubectl exec nginx-keyvault-csi -n keyvault-demo -- ls -la /mnt/secrets

# Read a secret from the mounted volume
kubectl exec nginx-keyvault-csi -n keyvault-demo -- cat /mnt/secrets/database-username
```

Expected: You should see the secrets mounted as files in `/mnt/secrets/`

## 🔄 Step 9: Sync Secrets to Kubernetes Secrets (Optional)

You can also sync Key Vault secrets to native Kubernetes secrets:

```powershell
# Update SecretProviderClass to enable sync
@"
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-keyvault-sync
  namespace: keyvault-demo
spec:
  provider: azure
  secretObjects:
  - secretName: app-secrets
    type: Opaque
    data:
    - objectName: database-username
      key: db-username
    - objectName: database-password
      key: db-password
    - objectName: api-key
      key: api-key
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "false"
    clientID: $IDENTITY_CLIENT_ID
    keyvaultName: $KEYVAULT_NAME
    cloudName: ""
    objects: |
      array:
        - |
          objectName: database-username
          objectType: secret
        - |
          objectName: database-password
          objectType: secret
        - |
          objectName: api-key
          objectType: secret
    tenantId: $TENANT_ID
"@ | kubectl apply -f -

Write-Host "✅ SecretProviderClass with sync enabled"
```

## 📦 Step 10: Deploy Pod with Synced Secrets

```powershell
# Deploy pod using the sync-enabled SecretProviderClass
@'
apiVersion: v1
kind: Pod
metadata:
  name: nginx-keyvault-sync
  namespace: keyvault-demo
  labels:
    azure.workload.identity/use: "true"
  annotations:
    secrets-store.csi.k8s.io/rotation-poll-interval: "120s"
spec:
  serviceAccountName: workload-identity-sa
  containers:
  - name: nginx
    image: nginx:alpine
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: db-username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: db-password
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: api-key
    volumeMounts:
    - name: secrets-store
      mountPath: "/mnt/secrets"
      readOnly: true
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: azure-keyvault-sync
'@ | kubectl apply -f -

Write-Host "✅ Pod deployed with synced secrets"
```

## ✅ Step 11: Verify Synced Secrets

```powershell
# Wait for pod and secret to be created
Start-Sleep -Seconds 10

# Check if Kubernetes secret was created
kubectl get secret app-secrets -n keyvault-demo

# Describe the secret
kubectl describe secret app-secrets -n keyvault-demo

# View environment variables in the pod
kubectl exec nginx-keyvault-sync -n keyvault-demo -- env | Select-String -Pattern "DB_|API_"
```

## 🔄 Step 12: Test Secret Rotation

Update a secret in Key Vault and observe auto-rotation in volume mounts:

```powershell
# Update a secret in Key Vault
az keyvault secret set `
  --vault-name $KEYVAULT_NAME `
  --name "api-key" `
  --value "updated-secret-key-67890"

Write-Host "✅ Secret updated in Key Vault"

# Check current secret values in the mounted volumes
Write-Host "`n=== Current secret values in volume mounts ==="
Write-Host "nginx-keyvault-csi: " -NoNewline
$csiValue = kubectl exec nginx-keyvault-csi -n keyvault-demo -- cat /mnt/secrets/api-key 2>$null
if ($LASTEXITCODE -eq 0) {
  Write-Host $csiValue
} else {
  Write-Host "Pod not ready"
}

Write-Host "nginx-keyvault-sync: " -NoNewline
$syncValue = kubectl exec nginx-keyvault-sync -n keyvault-demo -- cat /mnt/secrets/api-key 2>$null
if ($LASTEXITCODE -eq 0) {
  Write-Host $syncValue
} else {
  Write-Host "Pod not ready"
}

# The CSI driver polls for changes based on the rotation-poll-interval annotation (120s)
Write-Host "`n⏳ Waiting for secret rotation (polling interval: 120 seconds)..."
Write-Host "Checking every 30 seconds for up to 4 minutes..."

for ($i = 1; $i -le 8; $i++) {
  Write-Host "`n--- Check $i/8 ($(Get-Date -Format T)) ---"

  Write-Host "nginx-keyvault-csi volume: " -NoNewline
  $csiValue = kubectl exec nginx-keyvault-csi -n keyvault-demo -- cat /mnt/secrets/api-key 2>$null
  if ($LASTEXITCODE -eq 0) {
    Write-Host $csiValue
  } else {
    Write-Host "Error"
  }

  Write-Host "nginx-keyvault-sync volume: " -NoNewline
  $syncValue = kubectl exec nginx-keyvault-sync -n keyvault-demo -- cat /mnt/secrets/api-key 2>$null
  if ($LASTEXITCODE -eq 0) {
    Write-Host $syncValue
  } else {
    Write-Host "Error"
  }

  Start-Sleep -Seconds 30
}

Write-Host "`n📝 Key Points:"
Write-Host "✅ Volume-mounted secrets: Files update automatically after rotation interval"
Write-Host "❌ Kubernetes secret objects: Do NOT auto-rotate (created once at pod startup)"
Write-Host "💡 Applications must re-read mounted files to see updated values"
Write-Host "💡 For K8s secrets in env vars, restart the pod to see changes"
Write-Host ""
Write-Host "Note: Secret rotation requires --enable-secret-rotation flag during cluster creation"
```

## 📊 Step 13: View CSI Driver Metrics

```powershell
# Check CSI driver metrics
kubectl get csinodes

# View SecretProviderClass status
kubectl describe secretproviderclass azure-keyvault-secrets -n keyvault-demo

# Check for any errors
kubectl get events -n keyvault-demo --sort-by='.lastTimestamp'
```

## 🎓 Summary

You have successfully:
- ✅ Verified CSI Secret Store driver installation
- ✅ Created a managed identity for Key Vault access
- ✅ Configured workload identity federation
- ✅ Created SecretProviderClass resources
- ✅ Mounted Key Vault secrets as volumes in pods
- ✅ Synced Key Vault secrets to Kubernetes secrets
- ✅ Tested secret rotation

## 🔑 Key Concepts Learned

1. **CSI Driver Architecture**: Secrets are mounted as volumes, not stored in etcd
2. **Workload Identity**: Pods authenticate to Azure using federated credentials
3. **SecretProviderClass**: Defines which secrets to fetch from Key Vault
4. **Auto-rotation**: Secrets automatically update (default polling interval: 2 minutes)
5. **Sync to K8s Secrets**: Optional feature to create native Kubernetes secrets

## 📝 Best Practices

- ✅ Use workload identity instead of service principal credentials
- ✅ Grant least-privilege access (Key Vault Secrets User role)
- ✅ Use specific object versions for immutable deployments
- ✅ Enable sync only when you need to use secrets as environment variables
- ✅ Monitor CSI driver logs for issues

## 🧹 Cleanup

```powershell
# Delete demo resources (keep namespace for next chapter)
kubectl delete pod nginx-keyvault-csi nginx-keyvault-sync -n keyvault-demo
kubectl delete secretproviderclass azure-keyvault-secrets azure-keyvault-sync -n keyvault-demo

# Keep the namespace, service account, and identity for Chapter 2
Write-Host "✅ Cleaned up CSI driver demo resources"
```

## 🔍 Troubleshooting

**Issue**: Pod fails to mount secrets
- Check workload identity configuration
- Verify role assignment propagated (wait 1-2 minutes)
- Check pod logs: `kubectl describe pod <pod-name> -n keyvault-demo`

**Issue**: Secrets not updating
- CSI driver polls every 2 minutes by default
- Force update by deleting and recreating the pod

**Issue**: Permission denied errors
- Verify managed identity has "Key Vault Secrets User" role
- Check federated credential subject matches service account

## 🚀 Next Steps

Proceed to:
- **[Chapter 2: External Secrets Operator](../chapter-2-external-secrets/README.md)**

In the next chapter, you'll learn an alternative approach to secrets management using the External Secrets Operator.
