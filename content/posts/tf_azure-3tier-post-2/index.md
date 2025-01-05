---
title: "IaC on 3-Tier architecture through Terraform in Azure - Part 2 Resource on Azure"
date: 2024-06-28
draft: false
summary: "IaC on 3-Tier architecture through Terraform in Azure"
tags: ["IaC", "Azure", "Terraform"]
series: ["IaC on 3-Tier architecture through Terraform in Azure"]
series_order: 2
---

## Introduction

Welcome to the second part of our guide on deploying a 3-tier AWS infrastructure with Terraform! In this section, we’ll focus on setting up the backbone of your architecture. This includes creating the Virtual Network (VNet) to organize your subnets, setting up a Resource Group to keep everything neatly grouped, configuring a Network Security Group (NSG) to control traffic, and using Key Vault to securely manage sensitive data. By the end of this part, you’ll have a solid, secure foundation ready to support the rest of your infrastructure. Let’s get started!

## Architecture

<img src="diagram.png"
     alt="diagram"
     style="float: left; margin-right: 10px;" />

## Creating Cloud Resource with Terraform

### Variables and Data

Variables in Terraform act in an important role, as it increase security and reusability, preventing hardcoding values and exposure on sensitive data, and utilizing the same Terraform template across multiple environments by easily editing values of variables. 

Specifying their values in variable definitions files(with a filename ending in either .tfvars or .tfvars.json) helps to set lots of variables in different environments. So we divide 1 .tfvar file to set the value of variable per environment and 1 .tf file to declare input variables.

>Note: We can specify file by environment on the terraform command line with flag ***"-var-file"***. The command will be shown later.

```
# test.tfvars
environment = "test"
location    = "East Asia"

# variable.tf
variable "environment" {
  type = string
}

variable "location" {
  type = string
}

```

More, we use terraform data to get current Azure subscription and current configuration of AzureRM provider.

```
# data.tf
data "azurerm_subscription" "current" {}

data "azurerm_client_config" "current" {}

```

>Note: Data sources serve as a bridge between the current infrastructure and the desired configuration, allowing for more dynamic and context-aware provisioning.

### Resource Group
Providing a management layer that enables you to manage resources in Azure account, we create a resource group first and organize our infrastructure within the resource group. In real-world scenarios, companies often organize Resource Groups in different ways—by project, team, region, or even by resource type, like grouping all VNets together. But to keep things straightforward here, we’ll stick to a single Resource Group for now. It’s a practical starting point, but remember that every organization has its own way of managing resources.

```
# Resource Group
resource "azurerm_resource_group" "project" {
  name     = "project-rg-${var.environment}"
  location = var.location

  tags = {
    Environment = "${var.environment}"
  }
}

```

### Virtual Network

Let’s create an Azure Virtual Network and subnets.

#### VNet

This deploys a virtual network called ***"project-vnet-test"*** inside resource group ***"project-rg-test"*** with an address space of ***"10.0.0.0/16"***, which will allow you to deploy multiple subnets in that range. 

Depending on company practices, developers might be required to use a shared VPC/VNet managed by the DevOps or cloud team to centralize network traffic control. However, since this is a demo project, we’ll keep things simple and create a basic VNet.

```
# vnet.tf
# Create a virtual network
resource "azurerm_virtual_network" "project" {
  name                = "project-vnet-${var.environment}"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.project.location
  resource_group_name = azurerm_resource_group.project.name

  tags = {
    Environment = "${var.environment}"
  }
}
```

>Note: When designing a VNet, it’s important to avoid overlapping IP address ranges, especially with AKS. Overlaps can cause routing issues, so plan your CIDR blocks carefully to ensure smooth integration.

#### Subnet

Building 3 subnets for AKS Node pool, Storage Account, and MySQL, we hold CIDR ***"10.0.2.0/24"*** for later defining AKS’s network profile. Subnets are then defined with address prefixes:
  - ***"10.0.1.0/24"*** for AKS node pool subnet
  - ***"10.0.3.0/24"*** for Storage subnet
  - ***"10.0.4.0/24"*** for MySQL subnet

For subnet  ***"project-db-subnet-test"***, we delegated the subnet to ***"Microsoft.DBforMySQL/flexibleServers"*** which means that only Azure Database for MySQL flexible server instances can use that subnet. Last, but not least, all subnets were added ***"Microsoft.Storage"*** for service endpoint for secure and direct connectivity to Azure Storage service.

>Note: Virtual Network (VNet) service endpoint provides secure and direct connectivity to Azure services over an optimized route over the Azure backbone network.

>Note: Subnet delegation provides full control to the customer on managing the integration of Azure services into their virtual networks. 

```
# vnet.tf
### CIDR "10.0.2.0/24" is delegated for AKS services

# Subnet for AKS node pool
resource "azurerm_subnet" "node_subnet" {
  name                 = "project-node-subnet-${var.environment}"
  resource_group_name  = azurerm_resource_group.project.name
  virtual_network_name = azurerm_virtual_network.project.name
  address_prefixes     = ["10.0.1.0/24"]
  service_endpoints    = ["Microsoft.Storage"]
}

# Subnet for Storage
resource "azurerm_subnet" "storage_subnet" {
  name                 = "project-storage-${var.environment}"
  resource_group_name  = azurerm_resource_group.project.name
  virtual_network_name = azurerm_virtual_network.project.name
  address_prefixes     = ["10.0.3.0/24"]
  service_endpoints    = ["Microsoft.Storage"]
}

# Subnet for MySQL
resource "azurerm_subnet" "db_subnet" {
  name                 = "project-db-subnet-${var.environment}"
  resource_group_name  = azurerm_resource_group.project.name
  virtual_network_name = azurerm_virtual_network.project.name
  address_prefixes     = ["10.0.4.0/24"]
  service_endpoints    = ["Microsoft.Storage"]
  delegation {
    name = "fs"
    service_delegation {
      name = "Microsoft.DBforMySQL/flexibleServers"
      actions = [
        "Microsoft.Network/virtualNetworks/subnets/join/action",
      ]
    }
  }
}

```

### Network Security Group

Security rules in network security groups enable us to filter the type of network traffic that can flow in and out of virtual network subnets and network interfaces. We create 1 NSG to filter traffic that flows in and out MySQL subnet. 

>Note: When creating a Network Security Group (NSG), default security rules are automatically generated. These rules typically allow basic [inbound and outbound traffic](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview#default-security-rules). For environments like development, UAT, or SIT, additional rules with higher priorities may need to be configured. These rules ensure that only authorized business users and developers can access these environments, maintaining proper security and isolation.

```
# nsg.tf
# Security Group
resource "azurerm_network_security_group" "nsg" {
  name                = "project-sg-${var.environment}"
  location            = azurerm_resource_group.project.location
  resource_group_name = azurerm_resource_group.project.name

  security_rule {
    name                       = "AllowMySQL"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "3306"
    source_address_prefix      = "10.0.2.0/24"
    destination_address_prefix = "10.0.4.0/24"
  }

  tags {
    environment = "${var.environment}"
  }
}

resource "azurerm_subnet_network_security_group_association" "data" {
  subnet_id                 = azurerm_subnet.example.id
  network_security_group_id = azurerm_network_security_group.nsg.id
}
```

### User Assigned Identity

We then create 1 UAI as prerequisite for MySQL server. The managed identity will be configured to MySQL server, so we can authorize UAI to MySQL database securely.

>Note: You may also create a [managed identity](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview) as a standalone Azure resource. You can create a user-assigned managed identity and assign it to one or more Azure Resources.

```
# uai.tf
# User Assigned Identity
resource "azurerm_user_assigned_identity" "uai" {
  name                = "project-uai-${var.environment}"
  resource_group_name = azurerm_resource_group.project.name
  location            = azurerm_resource_group.project.location
}

```

### Key Vault

Moreover, we create Key Vault as secure store for secrets. And 2 2048-bit RSA keys will be created, separately for [MySQL](https://learn.microsoft.com/en-us/azure/mysql/flexible-server/concepts-customer-managed-key) and [Storage Account](https://learn.microsoft.com/en-us/azure/storage/common/customer-managed-keys-overview?toc=%2Fazure%2Fstorage%2Fblobs%2Ftoc.json&bc=%2Fazure%2Fstorage%2Fblobs%2Fbreadcrumb%2Ftoc.json). 

>Note: Premium level pricing tier adds Thales HSM (Hardware Security Modules) to Key Vault comparing to standard level pricing tier. please refer to [Azure Key Vault pricing](https://azure.microsoft.com/en-us/pricing/details/key-vault/).

>Note: Access policys of key vault is nearly providing full access to UAI and user as for demo. Please adjust according to your need. Here is one example on requirement of access policy for [Data encryption for Azure Database for MySQLMySQL fexible server](https://learn.microsoft.com/en-us/azure/mysql/flexible-server/how-to-data-encryption-portal).

```
# key_vault.tf
# Key Vault
resource "azurerm_key_vault" "key" {
  name                        = "project-dbkey-${var.environment}"
  location                    = azurerm_resource_group.project.location
  resource_group_name         = azurerm_resource_group.project.name
  enabled_for_disk_encryption = true
  tenant_id                   = data.azurerm_client_config.current.tenant_id
  soft_delete_retention_days  = 90    # value can be between 7 and 90 (the default) days
  purge_protection_enabled    = true  # Once Purge Protection has been enabled it's not possible to disable it
  sku_name = "standard" # refer to pricing tier, possible values are standard and premium
}

# Access Policy for yourself
resource "azurerm_key_vault_access_policy" "yourself" {
  key_vault_id = azurerm_key_vault.key.id
  tenant_id = data.azurerm_client_config.current.tenant_id
  object_id = data.azurerm_client_config.current.object_id

  key_permissions = [
    "Get", "List", "Update", "Create", "Import", "Delete", "Recover", "Backup", "Restore",
    "Decrypt", "Encrypt", "UnwrapKey", "WrapKey", "Verify", "Sign", "Purge",
    "Release", "Rotate", "GetRotationPolicy", "SetRotationPolicy"
  ]

  secret_permissions = [
    "Backup", "Delete", "Get", "List", "Purge", "Recover", "Restore", "Set"
  ]

  storage_permissions = [
    "Backup", "Delete", "DeleteSAS", "Get", "GetSAS", "List", "ListSAS", "Purge",
    "Recover", "RegenerateKey", "Restore", "Set", "SetSAS", "Update"
  ]
}

# Access Policy for UAI
resource "azurerm_key_vault_access_policy" "uai" {
  key_vault_id = azurerm_key_vault.key.id
  tenant_id = azurerm_user_assigned_identity.uai.tenant_id
  object_id = azurerm_user_assigned_identity.uai.principal_id

  key_permissions = [
    "Get", "List", "Update", "Create", "Import", "Delete", "Recover", "Backup", "Restore",
    "Decrypt", "Encrypt", "UnwrapKey", "WrapKey", "Verify", "Sign", "Purge",
    "Release", "Rotate", "GetRotationPolicy", "SetRotationPolicy"
  ]

  secret_permissions = [
    "Backup", "Delete", "Get", "List", "Purge", "Recover", "Restore", "Set"
  ]

  storage_permissions = [
    "Backup", "Delete", "DeleteSAS", "Get", "GetSAS", "List", "ListSAS", "Purge",
    "Recover", "RegenerateKey", "Restore", "Set", "SetSAS", "Update"
  ]
}

# We will add Access Policy for Storage Account later

# Key
resource "azurerm_key_vault_key" "db" {
  name         = "project-dbkey-${var.environment}"
  key_vault_id = azurerm_key_vault.key.id
  key_type     = "RSA"
  key_size     = 2048

  key_opts = [
    "decrypt",
    "encrypt",
    "sign",
    "unwrapKey",
    "verify",
    "wrapKey",
  ]

  rotation_policy {
    automatic {
      time_before_expiry = "P30D" #ISO 8601 duration
    }

    expire_after         = "P3Y" #ISO 8601 duration
    notify_before_expiry = "P29D" #ISO 8601 duration
  }
}

resource "azurerm_key_vault_key" "storage" {
  name         = "project-storagekey-${var.environment}"
  key_vault_id = azurerm_key_vault.key.id
  key_type     = "RSA"
  key_size     = 2048

  key_opts = [
    "decrypt",
    "encrypt",
    "sign",
    "unwrapKey",
    "verify",
    "wrapKey",
  ]

  rotation_policy {
    automatic {
      time_before_expiry = "P30D" #ISO 8601 duration
    }

    expire_after         = "P3Y" #ISO 8601 duration
    notify_before_expiry = "P29D" #ISO 8601 duration
  }
}
```

  <!-- access_policy {
    tenant_id = data.azurerm_client_config.current.tenant_id
    object_id = azurerm_storage_account.project.identity.0.principal_id

    key_permissions    = ["Get", "Create", "List", "Restore", "Recover", "UnwrapKey", "WrapKey", "Purge", "Encrypt", "Decrypt", "Sign", "Verify"]
    secret_permissions = ["Get"]
  } -->
