# Guia: Organização e Acesso a Módulos Terraform do GitHub via Azure DevOps

## Visão Geral

Este guia apresenta as melhores práticas para organizar módulos Terraform em repositórios GitHub e consumi-los em pipelines do Azure DevOps (ADO).

## Índice

1. [Estrutura do Repositório de Módulos](#estrutura-do-repositório-de-módulos)
2. [Estratégia de Versionamento](#estratégia-de-versionamento)
3. [Métodos de Autenticação](#métodos-de-autenticação)
4. [Consumindo Módulos no Terraform](#consumindo-módulos-no-terraform)
5. [Configuração de Pipelines Azure DevOps](#configuração-de-pipelines-azure-devops)
6. [Boas Práticas de Segurança](#boas-práticas-de-segurança)
7. [Exemplos Práticos](#exemplos-práticos)
8. [Troubleshooting](#troubleshooting)

---

## 1. Estrutura do Repositório de Módulos

### Opção A: Repositório Mono-Módulo (Recomendado para módulos independentes)

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

**Vantagens:**
- Versionamento independente
- Mais fácil de gerenciar releases
- Melhor controle de mudanças

### Opção B: Repositório Multi-Módulos (Recomendado para módulos relacionados)

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

**Vantagens:**
- Gerenciamento centralizado
- Módulos relacionados juntos
- Reutilização de código comum

---

## 2. Estratégia de Versionamento

### Semantic Versioning (SemVer)

Use tags Git seguindo o padrão **Semantic Versioning**:

```bash
# Formato: vMAJOR.MINOR.PATCH
v1.0.0  # Versão inicial estável
v1.1.0  # Nova funcionalidade (backward compatible)
v1.1.1  # Bug fix
v2.0.0  # Breaking change
```

### Comandos Git para Versionamento

```bash
# Criar uma nova tag
git tag -a v1.0.0 -m "Release version 1.0.0 - Initial stable release"

# Enviar tag para o repositório remoto
git push origin v1.0.0

# Listar todas as tags
git tag -l

# Criar tag para módulos específicos (multi-módulos)
git tag -a network/v1.0.0 -m "Network module v1.0.0"
git push origin network/v1.0.0
```

### Branches Recomendadas

- `main` - Código de produção estável
- `develop` - Desenvolvimento ativo
- `feature/*` - Novas funcionalidades
- `hotfix/*` - Correções urgentes

---

## 3. Métodos de Autenticação

### Método 1: Personal Access Token (PAT) - Recomendado

#### Passos:

1. **Criar PAT no GitHub:**
   - Acesse: Settings → Developer settings → Personal access tokens → Tokens (classic)
   - Permissões necessárias: `repo` (acesso total ao repositório)
   - Copie o token gerado

2. **Configurar no Azure DevOps:**
   - Project Settings → Pipelines → Service connections
   - New service connection → GitHub
   - Escolha "Personal Access Token"
   - Cole o token e nomeie a conexão (ex: "github-terraform-modules")

3. **Usar no Terraform:**
   ```hcl
   module "network" {
     source = "git::https://oauth2:${var.github_token}@github.com/seu-usuario/terraform-modules.git//modules/network?ref=v1.0.0"
   }
   ```

### Método 2: SSH Keys

#### Passos:

1. **Gerar chave SSH:**
   ```bash
   ssh-keygen -t ed25519 -C "azure-devops@empresa.com" -f ~/.ssh/ado_terraform
   ```

2. **Adicionar chave pública no GitHub:**
   - Settings → SSH and GPG keys → New SSH key
   - Cole o conteúdo de `ado_terraform.pub`

3. **Configurar no Azure DevOps:**
   - Armazene a chave privada em Azure Key Vault
   - Configure no pipeline para usar a chave

4. **Usar no Terraform:**
   ```hcl
   module "network" {
     source = "git::ssh://git@github.com/seu-usuario/terraform-modules.git//modules/network?ref=v1.0.0"
   }
   ```

### Método 3: GitHub App (Mais Seguro - Recomendado para Empresas)

1. Criar GitHub App com permissões de leitura em repositórios
2. Instalar no repositório de módulos
3. Configurar Service Connection no ADO usando GitHub App

---

## 4. Consumindo Módulos no Terraform

### Sintaxe de Source para GitHub

```hcl
# Usar branch específica (NÃO RECOMENDADO para produção)
module "example" {
  source = "git::https://github.com/usuario/terraform-modules.git//modules/network?ref=develop"
}

# Usar tag/versão específica (RECOMENDADO)
module "example" {
  source = "git::https://github.com/usuario/terraform-modules.git//modules/network?ref=v1.0.0"
}

# Usar commit específico (para debugging)
module "example" {
  source = "git::https://github.com/usuario/terraform-modules.git//modules/network?ref=abc123def"
}

# Com autenticação via token
module "example" {
  source = "git::https://oauth2:${var.github_token}@github.com/usuario/terraform-modules.git//modules/network?ref=v1.0.0"
}

# Via SSH
module "example" {
  source = "git::ssh://git@github.com/usuario/terraform-modules.git//modules/network?ref=v1.0.0"
}
```

### Exemplo Completo de Uso

```hcl
# main.tf
terraform {
  required_version = ">= 1.0"
}

module "network" {
  source = "git::https://github.com/usuario/terraform-modules.git//modules/network?ref=v1.2.0"
  
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
  source = "git::https://github.com/usuario/terraform-modules.git//modules/storage?ref=v2.1.0"
  
  storage_account_name = "stproddata001"
  resource_group_name  = "rg-production"
  location            = "eastus2"
}

output "vnet_id" {
  value = module.network.vnet_id
}
```

---

## 5. Configuração de Pipelines Azure DevOps

### Pipeline YAML Exemplo

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
  - group: terraform-secrets  # Variable group com GITHUB_TOKEN
  - name: TF_VERSION
    value: '1.6.0'

stages:
  - stage: Validate
    displayName: 'Terraform Validate'
    jobs:
      - job: ValidateTerraform
        displayName: 'Validate Terraform Configuration'
        steps:
          # Checkout do código
          - checkout: self
            clean: true
          
          # Instalar Terraform
          - task: TerraformInstaller@0
            displayName: 'Install Terraform'
            inputs:
              terraformVersion: $(TF_VERSION)
          
          # Configurar credenciais Git para acessar módulos privados
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

### Configurar Variable Group

1. **No Azure DevOps:**
   - Pipelines → Library → + Variable group
   - Nome: `terraform-secrets`
   - Adicionar variável: `GITHUB_TOKEN`
   - Marcar como "secreta" (ícone de cadeado)
   - Salvar

### Pipeline com SSH Keys

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

## 6. Boas Práticas de Segurança

### ✅ Recomendações

1. **Nunca commitar tokens/secrets no código**
   - Use Azure Key Vault
   - Use Variable Groups com secrets
   - Use Service Connections

2. **Usar versões específicas (tags) em produção**
   ```hcl
   # ✅ BOM
   source = "git::https://github.com/user/modules.git//network?ref=v1.0.0"
   
   # ❌ RUIM (para produção)
   source = "git::https://github.com/user/modules.git//network?ref=main"
   ```

3. **Limitar permissões do PAT**
   - Apenas permissão de leitura (`repo:read`)
   - Definir data de expiração
   - Criar tokens específicos por projeto

4. **Revisar mudanças antes de apply**
   - Sempre usar `terraform plan`
   - Implementar aprovações manuais para produção
   - Usar ambientes no ADO com gates

5. **Implementar CODEOWNERS no repositório de módulos**
   ```
   # .github/CODEOWNERS
   * @terraform-team
   /modules/network/ @network-team
   /modules/security/ @security-team
   ```

6. **Usar branch protection rules**
   - Require pull request reviews
   - Require status checks (CI)
   - Prevent force pushes

### 🔐 Configuração de Secrets no Azure DevOps

```yaml
# Referenciando secrets do Azure Key Vault
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

## 7. Exemplos Práticos

### Exemplo 1: Estrutura de Módulo Network

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
  description = "Nome da Virtual Network"
  type        = string
}

variable "location" {
  description = "Região Azure"
  type        = string
}

variable "resource_group_name" {
  description = "Nome do Resource Group"
  type        = string
}

variable "address_space" {
  description = "Espaço de endereçamento da VNet"
  type        = list(string)
}

variable "subnets" {
  description = "Mapa de subnets"
  type = map(object({
    address_prefix = string
  }))
  default = {}
}

variable "tags" {
  description = "Tags para os recursos"
  type        = map(string)
  default     = {}
}
```

```hcl
# modules/network/outputs.tf
output "vnet_id" {
  description = "ID da Virtual Network"
  value       = azurerm_virtual_network.main.id
}

output "vnet_name" {
  description = "Nome da Virtual Network"
  value       = azurerm_virtual_network.main.name
}

output "subnet_ids" {
  description = "IDs das subnets"
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

Módulo Terraform para criação de redes no Azure.

## Uso

```hcl
module "network" {
  source = "git::https://github.com/usuario/terraform-modules.git//modules/network?ref=v1.0.0"
  
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
| vnet_name | Nome da VNet | string | - | yes |
| location | Região Azure | string | - | yes |
| resource_group_name | Nome do RG | string | - | yes |
| address_space | Espaço de endereçamento | list(string) | - | yes |
| subnets | Mapa de subnets | map(object) | {} | no |

## Outputs

| Name | Description |
|------|-------------|
| vnet_id | ID da VNet |
| vnet_name | Nome da VNet |
| subnet_ids | IDs das subnets |
```

### Exemplo 2: Projeto Consumidor

```
projeto-infraestrutura/
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
  source = "git::https://github.com/usuario/terraform-modules.git//modules/network?ref=v1.2.0"
  
  vnet_name           = var.vnet_name
  location            = var.location
  resource_group_name = var.resource_group_name
  address_space       = var.vnet_address_space
  subnets            = var.subnets
  
  tags = var.common_tags
}

module "storage" {
  source = "git::https://github.com/usuario/terraform-modules.git//modules/storage?ref=v2.0.0"
  
  storage_account_name = var.storage_account_name
  resource_group_name  = var.resource_group_name
  location            = var.location
  
  tags = var.common_tags
}
```

---

## 8. Troubleshooting

### Problema: "Could not download module"

**Erro:**
```
Error: Failed to download module
Could not download module "network" (main.tf:10) source code from
"git::https://github.com/user/terraform-modules.git//modules/network?ref=v1.0.0":
error downloading 'https://github.com/user/terraform-modules.git?ref=v1.0.0':
/usr/bin/git exited with 128: fatal: could not read Username
```

**Solução:**
```bash
# Configure credenciais Git
git config --global url."https://oauth2:YOUR_TOKEN@github.com/".insteadOf "https://github.com/"

# Ou use SSH
git config --global url."ssh://git@github.com/".insteadOf "https://github.com/"
```

### Problema: "Module not found"

**Erro:**
```
Error: Module not found
The module address "git::https://github.com/user/terraform-modules.git//modules/netwrk?ref=v1.0.0"
could not be resolved.
```

**Solução:**
- Verifique o caminho do módulo (ex: `netwrk` vs `network`)
- Verifique se a tag existe: `git tag -l`
- Verifique a estrutura de diretórios no repositório

### Problema: "Permission denied"

**Erro:**
```
Permission denied (publickey).
fatal: Could not read from remote repository.
```

**Solução:**
```bash
# Para SSH: Verifique suas chaves
ssh -T git@github.com

# Para HTTPS: Verifique seu token
git ls-remote https://oauth2:YOUR_TOKEN@github.com/user/repo.git
```

### Problema: Cache de módulos antigos

**Sintoma:** Terraform não baixa versão atualizada do módulo

**Solução:**
```bash
# Limpar cache de módulos
rm -rf .terraform/modules/

# Re-inicializar
terraform init -upgrade
```

### Problema: Pipeline falha ao baixar módulo

**Checklist:**
1. ✅ Variable group configurado com `GITHUB_TOKEN`?
2. ✅ Token tem permissões corretas (`repo:read`)?
3. ✅ Script de configuração Git está sendo executado?
4. ✅ Tag/ref existe no repositório?
5. ✅ Service connection configurado corretamente?

---

## Resumo - Quick Reference

### 📋 Checklist de Implementação

- [ ] **Repositório de Módulos**
  - [ ] Estruturar módulos (mono ou multi-repo)
  - [ ] Criar README para cada módulo
  - [ ] Implementar versionamento semântico
  - [ ] Configurar CODEOWNERS
  - [ ] Configurar branch protection

- [ ] **Autenticação**
  - [ ] Criar PAT no GitHub ou SSH keys
  - [ ] Configurar Service Connection no ADO
  - [ ] Armazenar secrets no Key Vault/Variable Groups
  - [ ] Testar acesso aos módulos

- [ ] **Pipeline Azure DevOps**
  - [ ] Criar azure-pipelines.yml
  - [ ] Configurar stages (validate, plan, apply)
  - [ ] Configurar credenciais Git
  - [ ] Implementar aprovações para produção
  - [ ] Testar pipeline

- [ ] **Projetos Consumidores**
  - [ ] Referenciar módulos com tags específicas
  - [ ] Configurar backend remoto
  - [ ] Documentar dependências
  - [ ] Testar localmente

### 🔗 Links Úteis

- [Terraform Module Sources](https://www.terraform.io/language/modules/sources)
- [Azure DevOps Pipeline YAML Schema](https://docs.microsoft.com/azure/devops/pipelines/yaml-schema)
- [GitHub Personal Access Tokens](https://docs.github.com/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)
- [Semantic Versioning](https://semver.org/)

---

**Documento criado em:** Janeiro 2026  
**Última atualização:** 21/01/2026  
**Versão:** 1.0.0
