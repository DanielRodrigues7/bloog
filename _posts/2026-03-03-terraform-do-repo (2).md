---
title: "Infra como Código com Terraform — Meu repositório na prática"
date: 2026-03-03 11:00:00 -0300
categories: [devops, terraform]
tags: [terraform, iac, cloud]
description: "Guia prático e completo para criar VMs Linux no Azure usando Terraform — estado remoto, Key Vault, boas práticas e execução segura."
image: /assets/img/terraform.png
---

# **Provisionando VMs Linux no Azure com Terraform (estado remoto + Key Vault)**

Neste artigo vamos criar uma infraestrutura completa de **VMs Linux no Azure** usando **Terraform**, aplicando práticas recomendadas como:

- **Estado remoto** no Azure Storage  
- **Segredos protegidos** no Azure Key Vault  
- **Execução segura** (init/plan/apply/destroy)  
- **Organização limpa** do projeto e variáveis por ambiente  
- **Zero secrets no código**: nada de senhas commitadas no Git

---

## ⚠️ Segurança em primeiro lugar
Nenhuma senha é armazenada no repositório. Toda credencial é consumida automaticamente via:

```hcl
data.azurerm_key_vault_secret
```

---

# 🗂 Estrutura do Repositório

```
.
├─ .terraform/                  # cache do Terraform (não versionar)
├─ vm-linux-02/                 # (opcional) módulo/projeto isolado
│  ├─ main.tf
│  ├─ outputs.tf
│  ├─ variables.tf
│  └─ versions.tf
├─ hml.tfvars                   # variáveis por ambiente
├─ main.tf
├─ outputs.tf
├─ provider.tf
├─ variables.tf
└─ versions.tf                  # backend + required_providers
```

📌 **Dica:** Se você usar o subdiretório `vm-linux-02`, execute os comandos dentro dele.

---

# 🧱 O que este projeto cria

O Terraform provisiona:

- **Resource Group**
- **Virtual Network + Subnet**
- **Network Security Group + regras básicas**
- **Public IP (opcional)**
- **NIC**
- **VM Linux (ex.: Ubuntu 22.04)** com:
  - `admin_username` vindo de variável
  - `admin_password` vindo do **Azure Key Vault**
- **Outputs úteis:** IP público, FQDN, IDs

Tudo parametrizado via `variables.tf`.

---

# 🔐 Autenticação e boas práticas

## 1) Login via Azure CLI
```sh
az login
az account set --subscription "<SUBSCRIPTION_ID>"
```

## 2) Service Principal (recomendado para CI/CD)
Crie a identidade:

```sh
az ad sp create-for-rbac   --name "sp-terraform"   --role "Contributor"   --scopes /subscriptions/<SUBSCRIPTION_ID>
```

Exporte as variáveis:

```sh
export ARM_CLIENT_ID="<appId>"
export ARM_CLIENT_SECRET="<password>"
export ARM_TENANT_ID="<tenant>"
export ARM_SUBSCRIPTION_ID="<subscriptionId>"
```

✔️ Assim você evita expor segredos no provider.

---

# ☁️ Estado remoto no Azure Storage (Backend)

Trecho do `versions.tf`:

```hcl
terraform {
  backend "azurerm" {}

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}
```

## Criando a infraestrutura do backend (uma vez)

```sh
RG_STATE="rg-state-kv"
STG_NAME="stateremote"
CONTAINER="terraform"

az group create -n "$RG_STATE" -l "brazilsouth"
az storage account create -n "$STG_NAME" -g "$RG_STATE" -l "brazilsouth" --sku Standard_LRS
az storage container create -n "$CONTAINER" --account-name "$STG_NAME"
```

## Exportando a chave do Storage sem commitar

```sh
export ARM_ACCESS_KEY=$(az storage account keys list   -g "$RG_STATE" -n "$STG_NAME" --query "[0].value" -o tsv)
```

## Inicializando o Terraform com backend remoto

```sh
terraform init   -backend-config="resource_group_name=$RG_STATE"   -backend-config="storage_account_name=$STG_NAME"   -backend-config="container_name=$CONTAINER"   -backend-config="key=terraformdanielr2.tfstate"
```

---

# 🔑 Senha do administrador via Key Vault

Crie o Key Vault e o segredo:

```sh
KV_RG="rg-state-kv"
KV_NAME="rg-state-kv"
SECRET_NAME="vmAdminPassword"

az keyvault create -g "$KV_RG" -n "$KV_NAME"
az keyvault secret set   --vault-name "$KV_NAME"   --name "$SECRET_NAME"   --value "<SENHA_FORTE>"
```

## Consumindo no Terraform

```hcl
data "azurerm_key_vault" "kv" {
  name                = var.key_vault_name
  resource_group_name = var.key_vault_rg
}

data "azurerm_key_vault_secret" "vm_pwd" {
  name         = var.vm_password_secret_name
  key_vault_id = data.azurerm_key_vault.kv.id
}
```

## Uso dentro da VM

```hcl
resource "azurerm_linux_virtual_machine" "vm" {
  name                            = var.vm_name
  resource_group_name             = azurerm_resource_group.main.name
  location                        = azurerm_resource_group.main.location
  size                            = var.vm_size

  admin_username = var.admin_username
  admin_password = data.azurerm_key_vault_secret.vm_pwd.value

  disable_password_authentication = false
  network_interface_ids           = [azurerm_network_interface.nic.id]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts"
    version   = "latest"
  }
}
```

---

# ⚙️ Variáveis (variables.tf)

```hcl
variable "location"                { type = string }
variable "resource_group_name"     { type = string }
variable "vnet_name"               { type = string }
variable "subnet_name"             { type = string }
variable "vm_name"                 { type = string }
variable "vm_size"                 { type = string  default = "Standard_B2s" }
variable "admin_username"          { type = string  default = "azureadmin" }
variable "key_vault_rg"            { type = string }
variable "key_vault_name"          { type = string }
variable "vm_password_secret_name" { type = string  default = "vmAdminPassword" }
variable "create_public_ip"        { type = bool    default = true }
```

## Exemplo de hml.tfvars

```hcl
location                = "brazilsouth"
resource_group_name     = "rg-linux-hml"
vnet_name               = "vnet-linux-hml"
subnet_name             = "snet-linux-hml"
vm_name                 = "vm-linux-02"
vm_size                 = "Standard_B2s"
admin_username          = "azureadmin"
key_vault_rg            = "rg-state-kv"
key_vault_name          = "rg-state-kv"
vm_password_secret_name = "vmAdminPassword"
create_public_ip        = true
```

---

# 🚀 Execução (passo a passo)

```sh
terraform fmt

terraform init   -backend-config="resource_group_name=rg-state-kv"   -backend-config="storage_account_name=stateremote"   -backend-config="container_name=terraform"   -backend-config="key=terraformdanielr2.tfstate"

terraform validate

terraform plan -var-file="hml.tfvars"

terraform apply -var-file="hml.tfvars" -auto-approve

terraform state list

terraform output

terraform destroy -var-file="hml.tfvars" -auto-approve
```

---

# 🧪 Conectando na VM

```sh
ssh azureadmin@<IP_PUBLICO>
```

Recomendações pós-criação:
- Migrar para **SSH Key**
- Desabilitar **senha de admin**

---

# 🧰 Troubleshooting

### **Error acquiring the state lock**
Espere alguns segundos ou use `-lock=false` (com cautela).

### **Insufficient privileges to complete the operation**
O Service Principal precisa de **Contributor**.

### **Invalid index / null**
Verifique variáveis obrigatórias no `.tfvars`.

### **access_key no backend**
Nunca coloque no código. Prefira **ARM_ACCESS_KEY** ou SAS.

---

# 📄 Licença
Este projeto usa **MIT License**.

---

# 📌 Notas finais

- Testado com Terraform **≥ 1.6**
- Provider **azurerm ~> 4.x**
- Região padrão: **brazilsouth**
- Zero senhas no repositório — tudo via Key Vault
