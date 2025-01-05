---
title: "IaC on 3-Tier architecture through Terraform in Azure - Part 3 Resource on Azure"
date: 2024-06-28
draft: false
summary: "IaC on 3-Tier architecture through Terraform in Azure"
tags: ["IaC", "Azure", "Terraform"]
series: ["IaC on 3-Tier architecture through Terraform in Azure"]
series_order: 3
---
## Introduction

Welcome to Part 3 of our guide! In this section, we'll focus on setting up essential components to handle data storage and database management. We'll start with configuring an Azure Storage Account and its Blob Containers, ensuring secure access through network rules and data encryption. Next, we’ll dive into setting up a MySQL database, including implementing a Private DNS Zone and private endpoints for enhanced security and isolation. By the end of this part, you'll have a robust and secure foundation for managing your application’s data and databases efficiently.

## Architecture

<img src="diagram.png"
     alt="diagram"
     style="float: left; margin-right: 10px;" />

## Creating Cloud Resource with Terraform

### Storage

As mentioned before, storage will be created, to store content (eg. image, video) and log from MySQL server.

#### Storage Account

Storage Account is created as Geo-redundant storage for this demo. It takes advantage as Azure will replicate your storage account synchronously across Azure availability zones and regions, enhanced data availability and improved data protection.

When designing your storage account, data redundancy is often a critical business requirement. Factors such as consistency and cost play a key role in determining the best replication type for your scenario. For example, financial institutions often prefer Geo-Redundant Storage (GRS) to ensure maximum system reliability and data durability, as they handle sensitive data that requires strong disaster recovery capabilities. On the other hand, some local companies may opt for Locally Redundant Storage (LRS) or Zone-Redundant Storage (ZRS), as cost tends to be a more pressing concern for them.

**As developers**, it’s important to provide informed guidance to business users to ensure that technical solutions align with their needs. Always ensure the options you recommend fit the use case, and don’t forget to test and validate these choices. Be aware that not all redundancy types are available in every region—Geo-Zone-Redundant Storage (GZRS), for instance, may not be supported every region.

>Note: Flag "account_replication_type" refers to [Azure Storage redundancy](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy). Valid options are LRS, GRS, RAGRS, ZRS, GZRS and RAGZRS. Please adjust the value according to your needs. For example, vaule "GRS" may be needed for production environment.

>Note: Infrastructure encryption is recommended for scenarios where doubly encrypting data is necessary for compliance requirements.

```
# storage.tf
# Storage Account
resource "azurerm_storage_account" "project" {
  name                     = "project${var.environment}storage"
  resource_group_name      = azurerm_resource_group.project.name
  location                 = azurerm_resource_group.project.location
  account_tier             = "Standard" # refer to pricing tier, possible values are standard and premium
  account_replication_type = "ZRS" # Value changing may forces a new resource to be created

  infrastructure_encryption_enabled = "false"

  large_file_share_enabled      = "false"
  public_network_access_enabled = "true" # public network access is enabled as for testing environment

  # Example to setup blob properties

  # blob_properties {
  #   change_feed_enabled      = "true"
  #   last_access_time_enabled = "true"
  #   versioning_enabled       = "true"
  #   delete_retention_policy {
  #     days = 35
  #   }
  #   restore_policy {
  #     days = 30
  #   }
  # }

  identity {
    type = "SystemAssigned" # type of Managed Service Identity configured on Storage Account
  }

  lifecycle {
    ignore_changes = [
      customer_managed_key # customer managed key will be manage in other teeraform resource
    ]
  }
}
```
#### Network Rules for Storage Account

>Note: Only one azurerm_storage_account_network_rules can be tied to an azurerm_storage_account. 

```
# storage.tf
# Network Rules for Storage Account
resource "azurerm_storage_account_network_rules" "myipandsubnet" {
  storage_account_id         = azurerm_storage_account.project.id
  default_action             = "Deny"
  ip_rules                   = ["xxx.x.xxx.xxx"] # edit value to your home IP or whitelisted ip
  virtual_network_subnet_ids = [azurerm_subnet.storage_subnet.id, azurerm_subnet.db_subnet.id, azurerm_subnet.node_subnet.id] # subnet ids to secure the storage account
  bypass                     = ["Metrics"]
}
```

#### Data Encryption for Storage Account

Notes that resource "azurerm_storage_account" are ignoring changes in block "customer_managed_key" as we manage Customer Managed Key by resource "azurerm_storage_account_customer_managed_key".

>Note: It's possible to define a Customer Managed Key both within resource "azurerm_storage_account" via block "customer_managed_key"  and by using the resource "azurerm_storage_account_customer_managed_key". However it's not possible to use both methods to manage a Customer Managed Key for a Storage Account, since there'll be [conflicts](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/storage_account_customer_managed_key). 

```
# key.tf
resource "azurerm_key_vault_access_policy" "storage" {
  key_vault_id = azurerm_key_vault.key.id
  tenant_id = data.azurerm_client_config.current.tenant_id
  object_id = azurerm_storage_account.project.identity.0.principal_id # Principal ID for the Service Principal associated with Identity of Storage Account

  secret_permissions = ["Get"]
  key_permissions    = ["Get", "Create", "List", "Restore", "Recover", "UnwrapKey", "WrapKey", "Purge", "Encrypt", "Decrypt", "Sign", "Verify"]
}

# storage.tf
# Data Encrytion for Storage Account
resource "azurerm_storage_account_customer_managed_key" "storage_key" {
  storage_account_id = azurerm_storage_account.project.id
  key_vault_id       = azurerm_key_vault.key.id
  key_name           = azurerm_key_vault_key.storage.name
}
```

#### Storage Blob Container

After all settings are created, we create Blob Container to storage files.

```
# Storage Blob Container
resource "azurerm_storage_container" "project" {
  name                  = "project-blob"
  storage_account_name  = azurerm_storage_account.project.name
  container_access_type = "blob"
}
```

### MySQL

For MySQL, we choose to provision MySQL Flexible Server (Azure Database for MySQL - Single Server is scheduled for retirement by [September 16, 2024](https://learn.microsoft.com/en-us/azure/mysql/single-server/whats-happening-to-mysql-single-server)).
<!-- Terraform code for MySQL will be divided into a few parts: server and database, IAM, logging,  -->

>Note: To learn difference between MySQL Single Server and MySQL Flexible Server, please refer to [here](https://learn.microsoft.com/en-us/azure/mysql/select-right-deployment-type).

#### Private DNS Zone

We create Private DNS Zone for MySQL server to make sure database connects securely within our VNet.

```
# private_dns.tf
resource "azurerm_private_dns_zone" "database" {
  name                = "project.${var.environment}.mysql.database.azure.com"
  resource_group_name = azurerm_resource_group.project.name
}

resource "azurerm_private_dns_zone_virtual_network_link" "db_links" {
  name                  = "${azurerm_virtual_network.project.name}-${var.environment}.com"
  private_dns_zone_name = azurerm_private_dns_zone.database.name
  resource_group_name   = azurerm_resource_group.project.name
  virtual_network_id    = azurerm_virtual_network.project.id
}
```

#### MySQL Database Server

Note that compute size "D2ads v5" is newly added in Region East Asia, pricing calculator isn't updated on the choice of MySQL compute size when I used this size. The cost is USD 0.1420 on 6 Feb 2024. For more information, please refer to [the compute size](https://learn.microsoft.com/en-us/azure/mysql/flexible-server/concepts-service-tiers-storage#service-tiers-size-and-server-types) and [cost](https://azure.microsoft.com/en-gb/pricing/calculator/).

By experience, I advise setting mode of block "high_availability" to "ZoneRedundant". It can only be set before we create it. You CANNOT change HA mode to Zone-redundant if you created server with same-zone mode once.

>Note: Flag "private_dns_zone_id" is required when setting flag "delegated_subnet_id". Private DNS zone should end with suffix '.mysql.database.azure'.com.

```
# mysql.tf
# MySQL Database Server
resource "azurerm_mysql_flexible_server" "main" {
  name                = "project-mysqlserver-${var.environment}"
  location            = azurerm_resource_group.project.location
  resource_group_name = azurerm_resource_group.project.name

  delegated_subnet_id = azurerm_subnet.db_subnet.id #  Changing value forces a new MySQL Flexible Server to be created
  private_dns_zone_id = azurerm_private_dns_zone.database.id

  zone                   = "1"
  administrator_login    = "projectadmin"
  administrator_password = "xxxxxx" # change to your own password

  sku_name = "GP_Standard_D2ads_v5" # refer to SKU tier and compute size

  storage {
    size_gb           = 20
    auto_grow_enabled = true # must be "true" to enable `high_availability`
  }

  identity {
    type         = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.uai.id] # use created UAI as Service Identity
  }

  # Data Encrytion
  customer_managed_key {
    key_vault_key_id                  = azurerm_key_vault_key.db.id # set Encrytion key with created key
    primary_user_assigned_identity_id = azurerm_user_assigned_identity.uai.id
  }

  version = "5.7" # MySQL version, change to your own required version

  backup_retention_days = 30 # default 7 days

  high_availability {
    mode                      = "ZoneRedundant" # prefer set it to ZoneRedundant, mode can be change when you created server
    standby_availability_zone = 2
  }

  tags = {
    Environment = "${var.environment}"
  }

  depends_on = [azurerm_private_dns_zone_virtual_network_link.db_links]

  lifecycle {
    ignore_changes = [
      zone, high_availability.0.standby_availability_zone # avoid to migrate MySQL Flexible Server back to primary Availability Zone if a fail-over occured
    ]
    # prevent_destroy = true # prevent destroy for production environment
  }
}
```

#### Active Directory administrator

Here is an example to provide admin access to myself. The solution can be used to provide server admin access to corresponding team with UAI.

```
resource "azurerm_mysql_flexible_server_active_directory_administrator" "me" {
  server_id   = azurerm_mysql_flexible_server.main.id
  identity_id = azurerm_user_assigned_identity.prod.id
  login       = "sqladmin"
  object_id   = data.azurerm_client_config.current.client_id # myself
  tenant_id   = data.azurerm_client_config.current.tenant_id # myself
}
```

#### Server Audit Log

We enable audit logs by updating server configurations. Audit log will be exported to Storage by updating Diagnostic Setting. Retention policy of audit log will be handled later.

>Note: Please refer to [audit-logs](https://learn.microsoft.com/en-us/azure/mysql/single-server/concepts-audit-logs) and [tutorial](https://learn.microsoft.com/en-us/azure/mysql/flexible-server/tutorial-configure-audit) to learn more on audit log setting, including audit log events.

>Note: Feature "retention_policy" has been [deprecated](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/migrate-to-azure-storage-lifecycle-policy?tabs=cli) in favor of resource "azurerm_storage_management_policy".

```
# MySQL Database Server Parameters
resource "azurerm_mysql_flexible_server_configuration" "audit_log_enabled" {
  name                = "audit_log_enabled"
  resource_group_name = azurerm_resource_group.project.name
  server_name         = azurerm_mysql_flexible_server.main.name
  value               = "ON" # enable audit log
}

resource "azurerm_mysql_flexible_server_configuration" "audit_log_events" {
  name                = "audit_log_events"
  resource_group_name = azurerm_resource_group.project.name
  server_name         = azurerm_mysql_flexible_server.main.name
  value               = "CONNECTION,GENERAL" # controls the events to be logged
}

# Example on setting MySQL users to be included for logging. 

# resource "azurerm_mysql_flexible_server_configuration" "audit_log_include_users" {
#   name                = "audit_log_include_users"
#   resource_group_name = azurerm_resource_group.project.name
#   server_name         = azurerm_mysql_flexible_server.main.name
#   value               = "projectadmin"
# }


# MySQL Database Diagnostic Setting
resource "azurerm_monitor_diagnostic_setting" "mysqlauditlog" {
  name               = "mysqlauditlog"
  target_resource_id = azurerm_mysql_flexible_server.main.id
  storage_account_id = azurerm_storage_account.project.id # storage account that logs will be sent

  enabled_log {
    category = "MySqlAuditLogs" # possible values are "MySqlSlowLogs" and "MySqlAuditLogs" for MySQL flexible server
    retention_policy {
      enabled = false
    }
  }

  metric {
    category = "AllMetrics"
    enabled  = false
    retention_policy {
      enabled = false
    }
  }

  depends_on = [
    azurerm_mysql_flexible_server_configuration.audit_log_enabled, azurerm_mysql_flexible_server_configuration.audit_log_events
  ]
}

# Storage Lifecycle for MySQL Audit Log
resource "azurerm_storage_management_policy" "project_mysql" {
  storage_account_id = azurerm_storage_account.project.id

  rule {
    name    = "mysql-auditlog-lifecycle"
    enabled = true
    filters {
      prefix_match = ["am-containerlog/WorkspaceResourceId=/subscriptions"]
      blob_types   = ["blockBlob"]
    }
    actions {
      base_blob {
        tier_to_cool_after_days_since_modification_greater_than    = 30
        tier_to_archive_after_days_since_modification_greater_than = 90
        delete_after_days_since_modification_greater_than          = 2555
      }
      version {
        change_tier_to_archive_after_days_since_creation = 15
        change_tier_to_cool_after_days_since_creation    = 7
        delete_after_days_since_creation                 = 30
      }
    }
  }
}
```
#### MySQL Database

After all audit log settings are created, we create MySQL Database.

```
# MySQL Database
resource "azurerm_mysql_flexible_database" "project" {
  name                = "project-mysqldb-${var.environment}"
  resource_group_name = azurerm_resource_group.project.name
  server_name         = azurerm_mysql_flexible_server.main.name
  charset             = "utf8mb4"
  collation           = "utf8mb4_unicode_ci"
}
```
