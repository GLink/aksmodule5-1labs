# AKS Advanced Concepts - Hands-on Lab

Welcome to the Azure Kubernetes Service (AKS) Advanced Concepts Lab! This comprehensive lab will guide you through essential AKS features and integrations used in production environments.

## 🐚 Choose Your Shell

This lab is available in two versions:

| Version | Description | Path |
|---------|-------------|------|
| **Bash** | Original labs using Bash/Linux shell syntax | Root chapter folders |
| **PowerShell** | Labs converted to PowerShell syntax | [`./psversion/`](./psversion/) folder |

> **PowerShell users**: Use the links in the **PowerShell Version** column below, or navigate directly to the [`psversion`](./psversion/) folder.

## 🎯 Learning Objectives

By completing this lab, you will gain hands-on experience with:

- **Security & Secrets Management**: Integrate Azure Key Vault with AKS using CSI drivers and External Secrets Operator
- **Identity & Access Management**: Configure Kubernetes RBAC with Azure AD and Workload Identity
- **Workload Scheduling**: Master node selectors, affinity rules, taints, tolerations, and topology spread constraints
- **Stateful Applications**: Deploy and manage StatefulSets for persistent workloads

## 📚 Lab Structure

This lab is organized into the following chapters:

| Chapter | Topic | Bash Version | PowerShell Version |
|---------|-------|--------------|-------------------|
| 0 | Setup and Prerequisites | [Bash](./chapter-0-setup/README.md) | [PowerShell](./psversion/chapter-0-setup/README.md) |
| 1 | Azure Key Vault CSI Driver | [Bash](./chapter-1-keyvault-csi/README.md) | [PowerShell](./psversion/chapter-1-keyvault-csi/README.md) |
| 2 | External Secrets Operator | [Bash](./chapter-2-external-secrets/README.md) | [PowerShell](./psversion/chapter-2-external-secrets/README.md) |
| 3 | Kubernetes RBAC with Azure AD | [Bash](./chapter-3-rbac-azuread/README.md) | [PowerShell](./psversion/chapter-3-rbac-azuread/README.md) |
| 4 | Managed Identity and Federated Credentials | [Bash](./chapter-4-workload-identity/README.md) | [PowerShell](./psversion/chapter-4-workload-identity/README.md) |
| 5 | Node Selectors and Affinity | [Bash](./chapter-5-node-selectors/README.md) | [PowerShell](./psversion/chapter-5-node-selectors/README.md) |
| 6 | Taints and Tolerations | [Bash](./chapter-6-taints-tolerations/README.md) | [PowerShell](./psversion/chapter-6-taints-tolerations/README.md) |
| 7 | Pod Affinity and Anti-Affinity | [Bash](./chapter-7-pod-affinity/README.md) | [PowerShell](./psversion/chapter-7-pod-affinity/README.md) |
| 8 | Pod Topology Spread Constraints | [Bash](./chapter-8-topology-spread/README.md) | [PowerShell](./psversion/chapter-8-topology-spread/README.md) |
| 9 | StatefulSets | [Bash](./chapter-9-statefulsets/README.md) | [PowerShell](./psversion/chapter-9-statefulsets/README.md) |

### Chapter Details

- **Chapter 0: Setup and Prerequisites**
  - Setting your student initials
  - Creating resource group and AKS cluster
  - Installing required tools

- **Chapter 1: Azure Key Vault CSI Driver**
  - Enable and configure CSI Secret Store driver
  - Mount secrets as volumes
  - Sync secrets to Kubernetes secrets

- **Chapter 2: External Secrets Operator**
  - Install External Secrets Operator
  - Configure SecretStore and ExternalSecret
  - Compare with CSI driver approach

- **Chapter 3: Kubernetes RBAC with Azure AD**
  - Enable Azure AD integration
  - Create roles and role bindings
  - Test access control

- **Chapter 4: Managed Identity and Federated Credentials**
  - Create User Assigned Managed Identity
  - Configure Workload Identity
  - Access Azure resources from pods

- **Chapter 5: Node Selectors and Affinity**
  - Use node selectors for pod placement
  - Configure node affinity rules
  - Understand scheduling preferences

- **Chapter 6: Taints and Tolerations**
  - Apply taints to nodes
  - Configure tolerations in pods
  - Control workload placement

- **Chapter 7: Pod Affinity and Anti-Affinity**
  - Configure pod affinity rules
  - Implement anti-affinity for high availability
  - Balance workload distribution

- **Chapter 8: Pod Topology Spread Constraints**
  - Define topology spread constraints
  - Distribute pods across zones
  - Ensure resilience and availability

- **Chapter 9: StatefulSets**
  - Deploy StatefulSets
  - Configure persistent volumes
  - Manage stateful applications

## ⏱️ Estimated Time

- **Total Lab Duration**: 4-6 hours
- **Each Chapter**: 20-40 minutes

## 🔧 Prerequisites

- Azure subscription with appropriate permissions
- Basic knowledge of Kubernetes concepts
- Familiarity with Azure CLI and kubectl
- Text editor (VS Code recommended)

## 🚀 Getting Started

1. Start with **[Chapter 0: Setup and Prerequisites](./chapter-0-setup/README.md)**
2. Complete each chapter in sequence
3. Keep your AKS cluster running throughout the lab
4. Each chapter builds on previous concepts

## 💡 Tips for Success

- **Set your initials**: You'll use student initials for resource naming to avoid conflicts
- **One cluster for all**: Use the same AKS cluster throughout all chapters
- **Save your work**: Keep track of resource names and configurations
- **Clean up**: Follow cleanup instructions at the end of each chapter if needed
- **Ask for help**: Don't hesitate to ask your instructor if you get stuck

## 📝 Notes

- All students in the same subscription should use unique initials for naming
- Some resources may take several minutes to provision
- Keep your Azure CLI session active throughout the lab
- Save important commands and outputs for reference

## 🧹 Cleanup

After completing all chapters, refer to the cleanup section in Chapter 0 to remove all resources and avoid unnecessary charges.

---

**Ready to begin?** Head over to **[Chapter 0: Setup and Prerequisites](./chapter-0-setup/README.md)** to get started!
