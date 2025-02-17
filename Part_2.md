# Part 2: Terraform – Deploying Azure Infrastructure

![Architecture](./images/net_diag.png)
This README provides a structured, step-by-step guide for deploying Azure infrastructure using Terraform.

## Table of Contents
1. [Introduction](#introduction)
2. [Resource Naming Convention](#resource-naming-convention)
3. [Terraform Script Overview](#terraform-script-overview)
4. [Resources Deployed](#resources-deployed)
5. [How to Use](#how-to-use)
6. [Customization](#customization)

## Introduction
This repository contains a Terraform script to deploy the architecture explained in [Part 1](./Part_1.md).

We use the **`azurerm`** provider, which allows Terraform to manage Azure infrastructure. The **`features {}`** block is required but can remain empty for now.

### Directory Structure
- **hub** – Contains Terraform scripts to deploy common Azure services such as Bastion Host, Azure Front Door, Azure NetApp Files, and Key Vault.
- **web** – Contains Terraform scripts to deploy production & preproduction environments for our hosting platform. It includes web servers, MySQL PaaS, and supporting configurations, integrating with resources from the hub. Environment-specific deployment is controlled with Terraform variable files.

## Resource Naming Convention
We follow Azure best practices for naming resources. The naming convention used is as follows:

`[Resource Type] - [Project Abbreviation] - [Environment Abbreviation] - [Region Abbreviation]`

Example: `fd-str-prd-inc`
- `fd`: Abbreviation for Azure Front Door
- `str`: Project abbreviation (e.g., Start)
- `prd`: Environment abbreviation (e.g., Production)
- `inc`: Region abbreviation (e.g., Central India)

Refer to:
- [Azure naming convention documentation](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-naming)
- [Azure abbreviation recommendations](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations)
- [Azure region abbreviations](https://github.com/Azure/terraform-azurerm-caf-enterprise-scale/blob/main/modules/connectivity/locals.geo_codes.tf.json)

## Resources Deployed
### HUB Resources
- **Resource Group**: Container for Azure resources shared across environments.
- **Virtual Network**: Provides network services with a single `web` subnet for the Bastion host.
- **Network Security Group (NSG)**: Protects the Bastion host with:
  - Allow rules for HTTP/HTTPS from the internet.
  - Restricted SSH and Jenkins (8080) access via IP whitelisting.
- **Bastion Host**: Ubuntu 24.04 VM with an attached disk, SSH key setup, and used for Jenkins and Ansible.
- **Azure Front Door**: Manages traffic with separate endpoints and origin groups for production and preproduction.
- **Azure NetApp Files**: Creates a NetApp account and capacity pool, with volumes provisioned per environment.
- **Private DNS Zone for MySQL**: Enables private name resolution for MySQL.
- **Key Vault**: Stores secrets, including SSL certificates and database credentials.

### WEB Resources
- **Resource Group**: Container for environment-specific resources.
- **Virtual Network**: Provides three subnets:
  - `web` - Web servers
  - `db` - Delegated for MySQL PaaS
  - `netapp` - Delegated for Azure NetApp Files
  - Peered with the hub VNet.
- **Network Security Group (NSG)**: Protects web servers, allowing only HTTP/HTTPS traffic.
- **Load Balancer**: Distributes traffic to web servers, ensuring high availability.
- **Web Servers (VMs)**: Ubuntu 24.04 instances with SSH keys for access.
- **Azure Front Door Origin & Route**: Configures the load balancer as an origin with HTTP-to-HTTPS redirection.
- **Azure NetApp Volume**: Mounted on web servers for web data storage.
- **MySQL**: Flexible server with auto-generated credentials stored in Key Vault.

## How to Use
### Prerequisites
- Azure Account
- Service Principal with Contributor role on the subscription

Authenticate Terraform using a Service Principal with a Client Secret, as detailed in the [Terraform documentation](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/guides/service_principal_client_secret#configuring-the-service-principal-in-terraform).

Create a `.creds` file in the Terraform directory and add:
```sh
export ARM_CLIENT_ID="00000000-0000-0000-0000-000000000000"
export ARM_CLIENT_SECRET="12345678-0000-0000-0000-000000000000"
export ARM_TENANT_ID="10000000-0000-0000-0000-000000000000"
export ARM_SUBSCRIPTION_ID="20000000-0000-0000-0000-000000000000"
```

Generate an SSH key pair for VM access:
```sh
ssh-keygen -f sshkey/azureuser_rsa -t rsa
```

### Deploying Hub Resources
Navigate to the Terraform directory:
```sh
cd terraform/hub
```
Initialize Terraform:
```sh
terraform init -var-file hub-inputs.tfvars
```
View and confirm the Terraform plan:
```sh
terraform plan -var-file hub-inputs.tfvars -state hub.tfstate
```
Apply the Terraform configuration:
```sh
terraform apply -var-file hub-inputs.tfvars -state hub.tfstate
```
Terraform will output below resource information which need to be used in next environment deployments. Save it somewhere.

```sh
Resource Group Name
VENT Name
VNET ID
Front Door Name
Production Endpoint ID
Production Origin Group ID
PreProduction Endpoint ID
PreProduction Origin Group ID
Netapp Account Name
Netapp Pool Name
Private DNS Zone Name
Private DNS Zone ID
Key Vault Name
Key Vault ID
VM IP
```

### Deploying Web Resources
Navigate to the Terraform directory:
```sh
cd terraform/web
```
#### Deploying Production Resources
We need to update update [`prod-inputs.tfvars`](../terraform/web/prod-inputs.tfvars) with information gathered from hub output
Update below variables

```sh
hub_rg_name => Resource Group Name
hub_vnet_name => VENT Name
hub_vnet_id => VNET ID
endpoint_id => PreProduction Endpoint ID
origin_group_id => PreProduction Origin Group ID 
netapp_account_name => Netapp Account Name
netapp_pool_name => Netapp Pool Name
dns_zone_name => Private DNS Zone Name
dns_zone_id => Private DNS Zone ID
key_vault_id => Key Vault ID
```

Initialize Terraform:
```sh
terraform init -var-file prod-inputs.tfvars
```
View and confirm the Terraform plan:
```sh
terraform plan -var-file prod-inputs.tfvars -state prod.tfstate
```
Apply the Terraform configuration:
```sh
terraform apply -var-file prod-inputs.tfvars -state prod.tfstate
```

Terraform will output below resource information which need to be used in next step. Save it somewhere.
```
Resource Group Name
MySQL server Name
Load Balancer IP
Web Server Private IPs
```
#### Deploying Preproduction Resources
We need to update update [`preprod-inputs.tfvars`](../terraform/web/preprod-inputs.tfvars) with information gathered from hub output
Update below variables
```sh
hub_rg_name => Resource Group Name
hub_vnet_name => VENT Name
hub_vnet_id => VNET ID
endpoint_id => Production Endpoint ID
origin_group_id => Production Origin Group ID 
netapp_account_name => Netapp Account Name
netapp_pool_name => Netapp Pool Name
dns_zone_name => Private DNS Zone Name
dns_zone_id => Private DNS Zone ID
key_vault_id => Key Vault ID
```

Initialize Terraform:
```sh
terraform init -var-file preprod-inputs.tfvars
```
View and confirm the Terraform plan:
```sh
terraform plan -var-file preprod-inputs.tfvars -state preprod.tfstate
```
Apply the Terraform configuration:
```sh
terraform apply -var-file preprod-inputs.tfvars -state preprod.tfstate
```

Terraform will output below resource information which need to be used in next step. Save it somewhere.
```sh
Resource Group Name
MySQL server Name
Load Balancer IP
Web Server Private IPs
```

## Customization
To customize Azure resources or other variables you need to update

hub - [`hub-inputs.tfvars`](../terraform/hub/hub-inputs.tfvars)
Production - 
[`prod-inputs.tfvars`](../terraform/web/prod-inputs.tfvars).

Production - 
[`preprod-inputs.tfvars`](../terraform/web/preprod-inputs.tfvars)

Description for variables can be found in variable file in respective directory. 

