# Chapter 0: Setup and Prerequisites

In this chapter, you'll set up your environment and create the AKS cluster that will be used throughout all subsequent chapters.

## 📋 Prerequisites

Before starting this lab, ensure you have:

### Required Tools

1. **Azure CLI** (version 2.50.0 or later)
   - [Installation guide](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
   - Verify: `az --version`

2. **kubectl** (Kubernetes command-line tool)
   - [Installation guide](https://kubernetes.io/docs/tasks/tools/)
   - Verify: `kubectl version --client`

3. **Helm** (version 3.x)
   - [Installation guide](https://helm.sh/docs/intro/install/)
   - Verify: `helm version`

4. **jq** (JSON processor - optional but helpful)
   - [Installation guide](https://stedolan.github.io/jq/download/)
   - Verify: `jq --version`

### Azure Requirements

- Active Azure subscription
- Permissions to create:
  - Resource Groups
  - AKS Clusters
  - Managed Identities
  - Key Vaults
  - Role Assignments

## 🎯 Learning Objectives

- Set up student-specific naming conventions
- Create a resource group for lab resources
- Deploy an AKS cluster with required features
- Configure kubectl access to the cluster
- Verify cluster functionality

## 📝 Step 1: Set Your Student Initials

To avoid naming conflicts in a shared subscription, you'll use your initials for resource naming.

```powershell
# Set your initials (use 2-4 lowercase letters)
$STUDENT_INITIALS = "abc"  # CHANGE THIS to your initials

# Verify it's set
Write-Host "Your initials: $STUDENT_INITIALS"
```

## 📦 Step 2: Define Resource Names

```powershell
# Set variables for resource names
$LOCATION = "swedencentral"
$RESOURCE_GROUP = "rg-aks-lab-$STUDENT_INITIALS"
$CLUSTER_NAME = "aks-lab-$STUDENT_INITIALS"
$KEYVAULT_NAME = "kv-aks-lab-$STUDENT_INITIALS"

# Display your configuration
Write-Host "Resource Configuration:"
Write-Host "  Location: $LOCATION"
Write-Host "  Resource Group: $RESOURCE_GROUP"
Write-Host "  Cluster Name: $CLUSTER_NAME"
Write-Host "  Key Vault Name: $KEYVAULT_NAME"
```

## 🔐 Step 3: Login to Azure

```powershell
# Login to Azure
az login

# Set the subscription (if you have multiple subscriptions)
# az account set --subscription "YOUR-SUBSCRIPTION-ID"

# Verify your subscription
az account show --query "{Name:name, SubscriptionId:id, TenantId:tenantId}" --output table
```

## 🏗️ Step 4: Create Resource Group

```powershell
# Create the resource group
az group create `
  --name $RESOURCE_GROUP `
  --location $LOCATION

Write-Host "✅ Resource group created: $RESOURCE_GROUP"
```

## ☸️ Step 5: Create AKS Cluster

We'll create an AKS cluster with features needed for all lab chapters:

```powershell
# Create AKS cluster with required features
az aks create `
  --resource-group $RESOURCE_GROUP `
  --name $CLUSTER_NAME `
  --location $LOCATION `
  --node-count 3 `
  --node-vm-size Standard_D4s_v5 `
  --zones 1 2 3 `
  --enable-managed-identity `
  --enable-oidc-issuer `
  --enable-workload-identity `
  --enable-addons azure-keyvault-secrets-provider `
  --enable-secret-rotation `
  --network-plugin azure `
  --max-pods 110 `
  --network-policy azure `
  --enable-aad `
  --enable-azure-rbac `
  --generate-ssh-keys

Write-Host "⏳ AKS cluster creation in progress (this takes 5-10 minutes)..."
```

### Cluster Configuration Explained

- **--node-count 3**: Three nodes for high availability exercises
- **--zones 1 2 3**: Spread nodes across availability zones for topology exercises
- **--enable-managed-identity**: Use managed identity for Azure integrations
- **--enable-oidc-issuer**: Required for Workload Identity
- **--enable-workload-identity**: Enable workload identity federation
- **--enable-addons azure-keyvault-secrets-provider**: Enable Key Vault CSI driver
- **--enable-secret-rotation**: Enable automatic rotation of secrets from Key Vault
- **--network-plugin azure**: Use Azure CNI for advanced networking

## 🔌 Step 6: Configure kubectl Access

```powershell
# Get AKS credentials
az aks get-credentials `
  --resource-group $RESOURCE_GROUP `
  --name $CLUSTER_NAME `
  --overwrite-existing

Write-Host "✅ kubectl configured for cluster: $CLUSTER_NAME"
```

## ✅ Step 7: Verify Cluster

```powershell
# Check cluster connection
kubectl cluster-info

# List nodes
kubectl get nodes -o wide

# Verify nodes are in different zones
kubectl get nodes -o custom-columns=NAME:.metadata.name,ZONE:.metadata.labels."topology\.kubernetes\.io/zone"

# Check system pods
kubectl get pods -n kube-system

# Verify Key Vault CSI driver is installed
kubectl get pods -n kube-system -l app=secrets-store-csi-driver

# Verify workload identity components
kubectl get pods -n kube-system -l azure-workload-identity.io/system=true
```

Expected output:
- 3 nodes in "Ready" state
- Nodes distributed across zones 1, 2, and 3
- CSI driver pods running in kube-system namespace
- Workload identity webhook running

## 🔑 Step 8: Create Azure Key Vault

We'll create a Key Vault with RBAC authorization for use in later chapters:

```powershell
# Create Key Vault with RBAC enabled
az keyvault create `
  --name $KEYVAULT_NAME `
  --resource-group $RESOURCE_GROUP `
  --location $LOCATION `
  --enable-rbac-authorization true

Write-Host "✅ Key Vault created: $KEYVAULT_NAME"

# Get your user's object ID
$CURRENT_USER_OBJECT_ID = az ad signed-in-user show --query id -o tsv

# Grant yourself "Key Vault Secrets Officer" role to manage secrets
$KEYVAULT_RESOURCE_ID = az keyvault show `
  --name $KEYVAULT_NAME `
  --resource-group $RESOURCE_GROUP `
  --query id -o tsv

az role assignment create `
  --role "Key Vault Secrets Officer" `
  --assignee-object-id $CURRENT_USER_OBJECT_ID `
  --assignee-principal-type User `
  --scope $KEYVAULT_RESOURCE_ID

Write-Host "✅ Granted Key Vault Secrets Officer role to current user"
Write-Host "⏳ Waiting 30 seconds for role assignment to propagate..."
Start-Sleep -Seconds 30

# Add sample secrets for later use
az keyvault secret set `
  --vault-name $KEYVAULT_NAME `
  --name "database-username" `
  --value "dbadmin"

az keyvault secret set `
  --vault-name $KEYVAULT_NAME `
  --name "database-password" `
  --value "P@ssw0rd123!"

az keyvault secret set `
  --vault-name $KEYVAULT_NAME `
  --name "api-key" `
  --value "super-secret-api-key-12345"

Write-Host "✅ Sample secrets added to Key Vault"
```

## 📊 Step 9: Save Your Configuration

Create a file to save your environment variables for future sessions:

```powershell
# Create a configuration file
@"
# AKS Lab Configuration for student: $STUDENT_INITIALS
$STUDENT_INITIALS = "$STUDENT_INITIALS"
$LOCATION = "$LOCATION"
$RESOURCE_GROUP = "$RESOURCE_GROUP"
$CLUSTER_NAME = "$CLUSTER_NAME"
$KEYVAULT_NAME = "$KEYVAULT_NAME"

# Derived values
`$SUBSCRIPTION_ID = "`$(az account show --query id -o tsv)"
`$TENANT_ID = "`$(az account show --query tenantId -o tsv)"
`$AKS_OIDC_ISSUER = "`$(az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --query 'oidcIssuerProfile.issuerUrl' -o tsv)"

Write-Host "Configuration loaded for student: `$STUDENT_INITIALS"
"@ | Set-Content -Path .\lab-config.ps1

Write-Host "✅ Configuration saved to lab-config.ps1"
Write-Host "   Load it in future sessions with: . .\lab-config.ps1"
```

## 🎓 Summary

You have successfully:
- ✅ Installed required tools
- ✅ Set up student-specific naming
- ✅ Created a resource group
- ✅ Deployed an AKS cluster with advanced features
- ✅ Configured kubectl access
- ✅ Created an Azure Key Vault
- ✅ Verified cluster functionality
- ✅ Saved your configuration

## 📝 Important Notes

- **Keep your cluster running** - You'll use it for all subsequent chapters
- **Save your lab-config.ps1 file** - You'll need it to restore variables
- **Note your resource names** - You'll reference them throughout the lab

## 🔄 Loading Configuration in Future Sessions

When you start a new terminal session:

```powershell
# Navigate to lab directory
Set-Location C:\path\to\aksmodule5-1labs

# Load your configuration
. .\psversion\chapter-0-setup\lab-config.ps1

# Re-authenticate if needed
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME
```

## 🧹 Cleanup (Only at End of ALL Chapters)

**⚠️ WARNING: Only run this after completing ALL chapters!**

```powershell
# Delete the entire resource group (removes all resources)
az group delete `
  --name $RESOURCE_GROUP `
  --yes `
  --no-wait

Write-Host "🗑️ Resource group deletion initiated"
```

## 🚀 Next Steps

Now that your environment is set up, proceed to:
- **[Chapter 1: Azure Key Vault CSI Driver](../chapter-1-keyvault-csi/README.md)**

---

**Questions or Issues?**
- Verify all commands completed successfully
- Check that all pods in kube-system are running
- Ensure you have proper Azure permissions
- Ask your instructor for assistance
