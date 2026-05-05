# AKS Advanced Concepts - Hands-on Lab (PowerShell Version)

This folder contains the **PowerShell-friendly** version of the AKS Advanced Concepts labs. All bash commands have been converted to PowerShell syntax.

## 📚 Chapters

| # | Chapter | Link |
|---|---------|------|
| 0 | Setup and Prerequisites | [README](./chapter-0-setup/README.md) |
| 1 | Azure Key Vault CSI Driver | [README](./chapter-1-keyvault-csi/README.md) |
| 2 | External Secrets Operator | [README](./chapter-2-external-secrets/README.md) |
| 3 | Kubernetes RBAC with Azure AD | [README](./chapter-3-rbac-azuread/README.md) |
| 4 | Managed Identity and Federated Credentials | [README](./chapter-4-workload-identity/README.md) |
| 5 | Node Selectors and Affinity | [README](./chapter-5-node-selectors/README.md) |
| 6 | Taints and Tolerations | [README](./chapter-6-taints-tolerations/README.md) |
| 7 | Pod Affinity and Anti-Affinity | [README](./chapter-7-pod-affinity/README.md) |
| 8 | Pod Topology Spread Constraints | [README](./chapter-8-topology-spread/README.md) |
| 9 | StatefulSets | [README](./chapter-9-statefulsets/README.md) |

## 🔧 Requirements

- **PowerShell 7+** (Windows PowerShell or PowerShell Core)
- **Azure CLI** (`az`)
- **kubectl**
- **Helm** (v3.x)

## 🚀 Getting Started

Start with **[Chapter 0: Setup and Prerequisites](./chapter-0-setup/README.md)** and complete each chapter in sequence.

## 📝 Key Differences from Bash Version

| Bash | PowerShell |
|------|-----------|
| `export VAR="value"` | `$VAR = "value"` |
| `source ./file.sh` | `. ./file.ps1` |
| `\` line continuation | `` ` `` backtick continuation |
| `sleep 30` | `Start-Sleep -Seconds 30` |
| `echo "text"` | `Write-Host "text"` |
| `cmd \| grep pattern` | `cmd \| Select-String pattern` |
| `$(command)` subshell | `$(command)` or `$var = command` |
| Heredoc `<<EOF` | Here-string `@" ... "@` |

> **Note**: `kubectl` and `az` CLI commands work identically in both shells. The main differences are in shell-level syntax for variables, piping, and control flow.
