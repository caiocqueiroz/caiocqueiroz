# Guide: Organizing and Accessing Terraform Modules from GitHub via Azure DevOps

## Overview

This guide presents best practices for organizing Terraform modules in GitHub repositories and consuming them in Azure DevOps (ADO) pipelines.

## Table of Contents

1. [Module Repository Structure](#module-repository-structure)
2. [Versioning Strategy](#versioning-strategy)
3. [Authentication Methods](#authentication-methods)
4. [Consuming Modules in Terraform](#consuming-modules-in-terraform)
5. [Azure DevOps Pipeline Configuration](#azure-devops-pipeline-configuration)
6. [Security Best Practices](#security-best-practices)
7. [Practical Examples](#practical-examples)
8. [Troubleshooting](#troubleshooting)

---

## 1. Module Repository Structure

### Option A: Single-Module Repository (Recommended for independent modules)

```
terraform-module-network/
├── README.md
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── examples/
│   └── basic/
│       ├── main.tf
│       └── README.md
└── tests/
    └── basic_test.go
```

**Advantages:**
- Independent versioning
- Easier to manage releases
- Better change control

### Option B: Multi-Module Repository (Recommended for related modules)

```
terraform-modules/
├── README.md
├── modules/
│   ├── network/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── compute/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   └── storage/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── README.md
├── examples/
│   ├── network-example/
│   └── compute-example/
└── tests/
```

**Advantages:**
- Centralized management
- Related modules together
- Common code reuse

---

## 2. Versioning Strategy

### Semantic Versioning (SemVer)

Use Git tags following the **Semantic Versioning** pattern:

```bash
# Format: vMAJOR.MINOR.PATCH
v1.0.0  # Initial stable version
v1.1.0  # New feature (backward compatible)
v1.1.1  # Bug fix
v2.0.0  # Breaking change
```

### Git Commands for Versioning

```bash
# Create a new tag
git tag -a v1.0.0 -m "Release version 1.0.0 - Initial stable release"

# Push tag to remote repository
git push origin v1.0.0

# List all tags
git tag -l

# Create tag for specific modules (multi-module repos)
git tag -a network/v1.0.0 -m "Network module v1.0.0"
git push origin network/v1.0.0
```

### Recommended Branches

- `main` - Stable production code
- `develop` - Active development
- `feature/*` - New features
- `hotfix/*` - Urgent fixes

---

## 3. Authentication Methods

### Method 1: Personal Access Token (PAT) - Recommended

#### Steps:

1. **Create PAT on GitHub:**
   - Go to: Settings → Developer settings → Personal access tokens → Tokens (classic)
   - Required permissions: `repo` (full repository access)
   - Copy the generated token

2. **Configure in Azure DevOps:**
   - Project Settings → Pipelines → Service connections
   - New service connection → GitHub
   - Choose "Personal Access Token"
   - Paste the token and name the connection (e.g., "github-terraform-modules")

3. **Use in Terraform:**
   ```hcl
   module "network" {
     source = "git::https://oauth2:${var.github_token}@github.com/your-user/terraform-modules.git//modules/network?ref=v1.0.0"
   }
   ```

### Method 2: SSH Keys

#### Steps:

1. **Generate SSH key:**
   ```bash
   ssh-keygen -t ed25519 -C "azure-devops@company.com" -f ~/.ssh/ado_terraform
   ```

2. **Add public key to GitHub:**
   - Settings → SSH and GPG keys → New SSH key
   - Paste the contents of `ado_terraform.pub`

3. **Configure in Azure DevOps:**
   - Store private key in Azure Key Vault
   - Configure in pipeline to use the key

4. **Use in Terraform:**
   ```hcl
   module "network" {
     source = "git::ssh://git@github.com/your-user/terraform-modules.git//modules/network?ref=v1.0.0"
   }
   ```

### Method 3: GitHub App (Most Secure - Recommended for Enterprises)

1. Create GitHub App with read permissions on repositories
2. Install on module repository
3. Configure Service Connection in ADO using GitHub App

---

## 4. Consuming Modules in Terraform

### Source Syntax for GitHub

```hcl
# Use specific branch (NOT RECOMMENDED for production)
module "example" {
  source = "git::https://github.com/user/terraform-modules.git//modules/network?ref=develop"
}

# Use specific tag/version (RECOMMENDED)
module "example" {
  source = "git::https://github.com/user/terraform-modules.git//modules/network?ref=v1.0.0"
}

# Use specific commit (for debugging)
module "example" {
  source = "git::https://github.com/user/terraform-modules.git//modules/network?ref=abc123def"
}

# With token authentication
module "example" {
  source = "git::https://oauth2:${var.github_token}@github.com/user/terraform-modules.git//modules/network?ref=v1.0.0"
}

# Via SSH
module "example" {
  source = "git::ssh://git@github.com/user/terraform-modules.git//modules/network?ref=v1.0.0"
}
```

### Complete Usage Example

```hcl
# main.tf
terraform {
  required_version = ">= 1.0"
}

module "network" {
  source = "git::https://github.com/user/terraform-modules.git//modules/network?ref=v1.2.0"
  
  resource_group_name = "rg-production"
  location           = "eastus2"
  vnet_name          = "vnet-prod"
  address_space      = ["10.0.0.0/16"]
  
  subnets = {
    web = {
      address_prefix = "10.0.1.0/24"
    }
    app = {
      address_prefix = "10.0.2.0/24"
    }
  }
}

module "storage" {
  source = "git::https://github.com/user/terraform-modules.git//modules/storage?ref=v2.1.0"
  
  storage_account_name = "stproddata001"
  resource_group_name  = "rg-production"
  location            = "eastus2"
}

output "vnet_id" {
  value = module.network.vnet_id
}
```

---

## 5. Azure DevOps Pipeline Configuration

### Example YAML Pipeline

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
      - develop

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: terraform-secrets  # Variable group with GITHUB_TOKEN
  - name: TF_VERSION
    value: '1.6.0'

stages:
  - stage: Validate
    displayName: 'Terraform Validate'
    jobs:
      - job: ValidateTerraform
        displayName: 'Validate Terraform Configuration'
        steps:
          # Checkout code
          - checkout: self
            clean: true
          
          # Install Terraform
          - task: TerraformInstaller@0
            displayName: 'Install Terraform'
            inputs:
              terraformVersion: $(TF_VERSION)
          
          # Configure Git credentials to access private modules
          - bash: |
              git config --global url."https://oauth2:$(GITHUB_TOKEN)@github.com/".insteadOf "https://github.com/"
            displayName: 'Configure Git credentials'
            env:
              GITHUB_TOKEN: $(GITHUB_TOKEN)
          
          # Terraform Init
          - task: TerraformTaskV4@4
            displayName: 'Terraform Init'
            inputs:
              provider: 'azurerm'
              command: 'init'
              backendServiceArm: 'Azure-Service-Connection'
              backendAzureRmResourceGroupName: 'rg-terraform-state'
              backendAzureRmStorageAccountName: 'sttfstate'
              backendAzureRmContainerName: 'tfstate'
              backendAzureRmKey: 'production.tfstate'
          
          # Terraform Validate
          - task: TerraformTaskV4@4
            displayName: 'Terraform Validate'
            inputs:
              provider: 'azurerm'
              command: 'validate'

  - stage: Plan
    displayName: 'Terraform Plan'
    dependsOn: Validate
    jobs:
      - job: PlanTerraform
        displayName: 'Plan Terraform Changes'
        steps:
          - checkout: self
            clean: true
          
          - task: TerraformInstaller@0
            displayName: 'Install Terraform'
            inputs:
              terraformVersion: $(TF_VERSION)
          
          - bash: |
              git config --global url."https://oauth2:$(GITHUB_TOKEN)@github.com/".insteadOf "https://github.com/"
            displayName: 'Configure Git credentials'
            env:
              GITHUB_TOKEN: $(GITHUB_TOKEN)
          
          - task: TerraformTaskV4@4
            displayName: 'Terraform Init'
            inputs:
              provider: 'azurerm'
              command: 'init'
              backendServiceArm: 'Azure-Service-Connection'
              backendAzureRmResourceGroupName: 'rg-terraform-state'
              backendAzureRmStorageAccountName: 'sttfstate'
              backendAzureRmContainerName: 'tfstate'
              backendAzureRmKey: 'production.tfstate'
          
          - task: TerraformTaskV4@4
            displayName: 'Terraform Plan'
            inputs:
              provider: 'azurerm'
              command: 'plan'
              environmentServiceNameAzureRM: 'Azure-Service-Connection'

  - stage: Apply
    displayName: 'Terraform Apply'
    dependsOn: Plan
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: ApplyTerraform
        displayName: 'Apply Terraform Changes'
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - checkout: self
                  clean: true
                
                - task: TerraformInstaller@0
                  displayName: 'Install Terraform'
                  inputs:
                    terraformVersion: $(TF_VERSION)
                
                - bash: |
                    git config --global url."https://oauth2:$(GITHUB_TOKEN)@github.com/".insteadOf "https://github.com/"
                  displayName: 'Configure Git credentials'
                  env:
                    GITHUB_TOKEN: $(GITHUB_TOKEN)
                
                - task: TerraformTaskV4@4
                  displayName: 'Terraform Init'
                  inputs:
                    provider: 'azurerm'
                    command: 'init'
                    backendServiceArm: 'Azure-Service-Connection'
                    backendAzureRmResourceGroupName: 'rg-terraform-state'
                    backendAzureRmStorageAccountName: 'sttfstate'
                    backendAzureRmContainerName: 'tfstate'
                    backendAzureRmKey: 'production.tfstate'
                
                - task: TerraformTaskV4@4
                  displayName: 'Terraform Apply'
                  inputs:
                    provider: 'azurerm'
                    command: 'apply'
                    environmentServiceNameAzureRM: 'Azure-Service-Connection'
```

### Configure Variable Group

1. **In Azure DevOps:**
   - Pipelines → Library → + Variable group
   - Name: `terraform-secrets`
   - Add variable: `GITHUB_TOKEN`
   - Mark as "secret" (padlock icon)
   - Save

### Pipeline with SSH Keys

```yaml
# azure-pipelines-ssh.yml
steps:
  - task: DownloadSecureFile@1
    name: sshKey
    displayName: 'Download SSH Private Key'
    inputs:
      secureFile: 'terraform_ssh_key'
  
  - bash: |
      mkdir -p ~/.ssh
      cp $(sshKey.secureFilePath) ~/.ssh/id_ed25519
      chmod 600 ~/.ssh/id_ed25519
      ssh-keyscan github.com >> ~/.ssh/known_hosts
    displayName: 'Configure SSH'
  
  - task: TerraformTaskV4@4
    displayName: 'Terraform Init with SSH'
    inputs:
      provider: 'azurerm'
      command: 'init'
```

---

## 6. Security Best Practices

### ✅ Recommendations

1. **Never commit tokens/secrets to code**
   - Use Azure Key Vault
   - Use Variable Groups with secrets
   - Use Service Connections

2. **Use specific versions (tags) in production**
   ```hcl
   # ✅ GOOD
   source = "git::https://github.com/user/modules.git//network?ref=v1.0.0"
   
   # ❌ BAD (for production)
   source = "git::https://github.com/user/modules.git//network?ref=main"
   ```

3. **Limit PAT permissions**
   - Read-only permission (`repo:read`)
   - Set expiration date
   - Create project-specific tokens

4. **Review changes before apply**
   - Always use `terraform plan`
   - Implement manual approvals for production
   - Use ADO environments with gates

5. **Implement CODEOWNERS in module repository**
   ```
   # .github/CODEOWNERS
   * @terraform-team
   /modules/network/ @network-team
   /modules/security/ @security-team
   ```

6. **Use branch protection rules**
   - Require pull request reviews
   - Require status checks (CI)
   - Prevent force pushes

### 🔐 Secrets Configuration in Azure DevOps

```yaml
# Referencing secrets from Azure Key Vault
variables:
  - group: terraform-keyvault-secrets

steps:
  - task: AzureKeyVault@2
    inputs:
      azureSubscription: 'Azure-Service-Connection'
      KeyVaultName: 'kv-terraform-secrets'
      SecretsFilter: 'github-token'
      RunAsPreJob: true
  
  - bash: |
      git config --global url."https://oauth2:$(github-token)@github.com/".insteadOf "https://github.com/"
    displayName: 'Configure Git with Key Vault secret'
```

---

## 7. Practical Examples

### Example 1: Network Module Structure

```hcl
# modules/network/main.tf
resource "azurerm_virtual_network" "main" {
  name                = var.vnet_name
  location            = var.location
  resource_group_name = var.resource_group_name
  address_space       = var.address_space
  
  tags = var.tags
}

resource "azurerm_subnet" "subnets" {
  for_each = var.subnets
  
  name                 = each.key
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [each.value.address_prefix]
}
```

```hcl
# modules/network/variables.tf
variable "vnet_name" {
  description = "Virtual Network name"
  type        = string
}

variable "location" {
  description = "Azure region"
  type        = string
}

variable "resource_group_name" {
  description = "Resource Group name"
  type        = string
}

variable "address_space" {
  description = "VNet address space"
  type        = list(string)
}

variable "subnets" {
  description = "Subnets map"
  type = map(object({
    address_prefix = string
  }))
  default = {}
}

variable "tags" {
  description = "Resource tags"
  type        = map(string)
  default     = {}
}
```

```hcl
# modules/network/outputs.tf
output "vnet_id" {
  description = "Virtual Network ID"
  value       = azurerm_virtual_network.main.id
}

output "vnet_name" {
  description = "Virtual Network name"
  value       = azurerm_virtual_network.main.name
}

output "subnet_ids" {
  description = "Subnet IDs"
  value       = { for k, v in azurerm_subnet.subnets : k => v.id }
}
```

```hcl
# modules/network/versions.tf
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}
```

```markdown
# modules/network/README.md
# Network Module

Terraform module for creating Azure networks.

## Usage

```hcl
module "network" {
  source = "git::https://github.com/user/terraform-modules.git//modules/network?ref=v1.0.0"
  
  vnet_name           = "vnet-prod"
  location            = "eastus2"
  resource_group_name = "rg-prod"
  address_space       = ["10.0.0.0/16"]
  
  subnets = {
    web = { address_prefix = "10.0.1.0/24" }
    app = { address_prefix = "10.0.2.0/24" }
  }
}
```

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|----------|
| vnet_name | VNet name | string | - | yes |
| location | Azure region | string | - | yes |
| resource_group_name | RG name | string | - | yes |
| address_space | Address space | list(string) | - | yes |
| subnets | Subnets map | map(object) | {} | no |

## Outputs

| Name | Description |
|------|-------------|
| vnet_id | VNet ID |
| vnet_name | VNet name |
| subnet_ids | Subnet IDs |
```

### Example 2: Consumer Project

```
infrastructure-project/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       └── terraform.tfvars
├── azure-pipelines.yml
└── README.md
```

```hcl
# environments/prod/main.tf
terraform {
  required_version = ">= 1.0"
  
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "sttfstate"
    container_name       = "tfstate"
    key                  = "prod.tfstate"
  }
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

module "network" {
  source = "git::https://github.com/user/terraform-modules.git//modules/network?ref=v1.2.0"
  
  vnet_name           = var.vnet_name
  location            = var.location
  resource_group_name = var.resource_group_name
  address_space       = var.vnet_address_space
  subnets            = var.subnets
  
  tags = var.common_tags
}

module "storage" {
  source = "git::https://github.com/user/terraform-modules.git//modules/storage?ref=v2.0.0"
  
  storage_account_name = var.storage_account_name
  resource_group_name  = var.resource_group_name
  location            = var.location
  
  tags = var.common_tags
}
```

---

## 8. Troubleshooting

### Problem: "Could not download module"

**Error:**
```
Error: Failed to download module
Could not download module "network" (main.tf:10) source code from
"git::https://github.com/user/terraform-modules.git//modules/network?ref=v1.0.0":
error downloading 'https://github.com/user/terraform-modules.git?ref=v1.0.0':
/usr/bin/git exited with 128: fatal: could not read Username
```

**Solution:**
```bash
# Configure Git credentials
git config --global url."https://oauth2:YOUR_TOKEN@github.com/".insteadOf "https://github.com/"

# Or use SSH
git config --global url."ssh://git@github.com/".insteadOf "https://github.com/"
```

### Problem: "Module not found"

**Error:**
```
Error: Module not found
The module address "git::https://github.com/user/terraform-modules.git//modules/netwrk?ref=v1.0.0"
could not be resolved.
```
*Note: Observe the typo "netwrk" instead of "network" in the module path*

**Solution:**
- Check module path (e.g., typo `netwrk` vs correct `network`)
- Check if tag exists: `git tag -l`
- Check directory structure in repository

### Problem: "Permission denied"

**Error:**
```
Permission denied (publickey).
fatal: Could not read from remote repository.
```

**Solution:**
```bash
# For SSH: Check your keys
ssh -T git@github.com

# For HTTPS: Check your token
git ls-remote https://oauth2:YOUR_TOKEN@github.com/user/repo.git
```

### Problem: Old module cache

**Symptom:** Terraform doesn't download updated module version

**Solution:**
```bash
# Clear module cache
rm -rf .terraform/modules/

# Re-initialize
terraform init -upgrade
```

### Problem: Pipeline fails to download module

**Checklist:**
1. ✅ Variable group configured with `GITHUB_TOKEN`?
2. ✅ Token has correct permissions (`repo:read`)?
3. ✅ Git configuration script is running?
4. ✅ Tag/ref exists in repository?
5. ✅ Service connection configured correctly?

---

## Summary - Quick Reference

### 📋 Implementation Checklist

- [ ] **Module Repository**
  - [ ] Structure modules (mono or multi-repo)
  - [ ] Create README for each module
  - [ ] Implement semantic versioning
  - [ ] Configure CODEOWNERS
  - [ ] Configure branch protection

- [ ] **Authentication**
  - [ ] Create PAT on GitHub or SSH keys
  - [ ] Configure Service Connection in ADO
  - [ ] Store secrets in Key Vault/Variable Groups
  - [ ] Test module access

- [ ] **Azure DevOps Pipeline**
  - [ ] Create azure-pipelines.yml
  - [ ] Configure stages (validate, plan, apply)
  - [ ] Configure Git credentials
  - [ ] Implement approvals for production
  - [ ] Test pipeline

- [ ] **Consumer Projects**
  - [ ] Reference modules with specific tags
  - [ ] Configure remote backend
  - [ ] Document dependencies
  - [ ] Test locally

### 🔗 Useful Links

- [Terraform Module Sources](https://www.terraform.io/language/modules/sources)
- [Azure DevOps Pipeline YAML Schema](https://docs.microsoft.com/azure/devops/pipelines/yaml-schema)
- [GitHub Personal Access Tokens](https://docs.github.com/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)
- [Semantic Versioning](https://semver.org/)

---

**Document created:** January 2026  
**Last update:** January 21, 2026  
**Version:** 1.0.0
