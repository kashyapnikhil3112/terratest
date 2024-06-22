# Terraform Infrastructure for Azure PaaS Service

This repository contains the Terraform code to set up an Azure PaaS service, including Identity and Access Management (IAM) best practices and a storage account for the Terraform state file.

## Prerequisites

1. **Azure Subscription**: You need an Azure account. If you don't have one, create a subscription using the free tier.
2. **Terraform**: Install Terraform on your local machine. Follow the installation guide [here](https://learn.hashicorp.com/tutorials/terraform/install-cli).
3. **Azure CLI**: Install Azure CLI. Follow the installation guide [here](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli).
4. **Service Principal**: Create a service principal and configure your local environment. Guide [here](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/guides/service_principal_client_secret).
5. **GitHub Account**: To clone and push changes to the repository.
6. **GitHub Secrets**: Store the following secrets in your GitHub repository settings:
    - `ARM_CLIENT_ID`
    - `ARM_CLIENT_SECRET`
    - `ARM_SUBSCRIPTION_ID`
    - `ARM_TENANT_ID`
    - `TF_API_TOKEN`

## Repository Structure
.
├── .github
│ └── workflows
│ └── terraform.yml
├── backend_module
│ ├── main.tf
│ └── README.md
└── main.tf


**main.tf** Terraform configuration sets up a complete Azure environment with the following components:

Azure Resource Group
Azure Storage Account and Container for storing the Terraform state file
Azure App Service Plan
Azure App Service with system-assigned identity
IAM Role Assignment for the App Service
All resource names include a randomly generated suffix to ensure uniqueness. The configuration also leverages modules and best practices for infrastructure as code (IaC).


**Explanation of the Terraform Code**

  # Configuration options
**1. Providers Configuration**
Configures the Azure Resource Manager (azurerm) provider, enabling Terraform to interact with Azure resources.
provider "azurerm" {
  features {}
}

**2. Random provider**
provider "random" {
}
Configures the Random provider, used for generating random values.

resource "random_string" "suffix" {
  length  = 8
  upper   = false
  special = false
}
Creates a random string with a length of 8 characters, used as a suffix for naming resources to ensure uniqueness.

**3. Backend Module Configuration**

module "backend" {
  source              = "./backend_module"
  resource_group_name = azurerm_resource_group.demo.name
  storage_account_name = "demostrgacnt${random_string.suffix.result}"
  container_name      = "tfstate${random_string.suffix.result}"
  key                 = "terraform.tfstate"
}
Configures the backend module to store the Terraform state file in an Azure Storage Account. Uses the generated random string for unique naming.

**4. Terraform Configuration**

terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0.2"
    }
  }
  required_version = ">= 1.1.0"
}
Specifies the required providers and versions for this configuration, ensuring compatibility.

**5. Resource Group**
resource "azurerm_resource_group" "demo" {
  name     = "demo-resources"
  location = "Central US"
}
Creates an Azure Resource Group named demo-resources in the Central US region.

**6. Storage Account**
resource "azurerm_storage_account" "demo" {
  name                     = "demostrgacnt${random_string.suffix.result}"
  resource_group_name      = azurerm_resource_group.demo.name
  location                 = azurerm_resource_group.demo.location
  account_tier             = "Standard"
  account_replication_type = "LRS"

  identity {
    type = "SystemAssigned"
  }
}
Creates a Storage Account with a system-assigned managed identity, using the random string for the name.

**7. Storage Container**
resource "azurerm_storage_container" "demo" {
  name                  = "tfstate${random_string.suffix.result}"
  storage_account_name  = azurerm_storage_account.demo.name
  container_access_type = "private"
}
Creates a private Storage Container within the Storage Account to store the Terraform state file.

**8. App Service Plan**
resource "azurerm_app_service_plan" "demo" {
  name                = "demoappserviceplan${random_string.suffix.result}"
  location            = azurerm_resource_group.demo.location
  resource_group_name = azurerm_resource_group.demo.name
  sku {
    tier = "Basic"
    size = "B1"
  }
}
Creates an App Service Plan with the Basic pricing tier and B1 size, using the random string for the name.

**9. App Service**
resource "azurerm_app_service" "demo" {
  name                = "demoappservice${random_string.suffix.result}"
  location            = azurerm_resource_group.demo.location
  resource_group_name = azurerm_resource_group.demo.name
  app_service_plan_id = azurerm_app_service_plan.demo.id

  identity {
    type = "SystemAssigned"
  }

  app_settings = {
    "WEBSITE_RUN_FROM_PACKAGE" = "1"
  }
}
Creates an App Service using the App Service Plan, with a system-assigned managed identity and configured to run from a package.

**10. Role Assignment**
resource "azurerm_role_assignment" "demo" {
  principal_id         = azurerm_app_service.demo.identity[0].principal_id
  role_definition_name = "Contributor"
  scope                = azurerm_resource_group.demo.id
}
Assigns the Contributor role to the App Service within the Resource Group, granting necessary permissions.



## GitHub Actions for CI/CD

The CI/CD pipeline is configured using GitHub Actions. The pipeline performs the following steps:
- Checks out the code from the repository.
- Sets up Terraform.
- Initializes Terraform with backend configuration.
- Runs `terraform plan` to preview changes.
- (Optional) Applies the Terraform configuration.

**##Secrets Configuration**
Ensure the following secrets are configured in your GitHub repository settings:

ARM_CLIENT_ID
ARM_CLIENT_SECRET
ARM_SUBSCRIPTION_ID
ARM_TENANT_ID
TF_API_TOKEN

**Resources Created**
Azure Resource Group: demo-resources
Azure Storage Account: demostrgacnt<random_suffix>
Azure Storage Container: tfstate<random_suffix>
Azure App Service Plan: demoappserviceplan<random_suffix>
Azure App Service: demoappservice<random_suffix>
IAM Role Assignment: Contributor role to the App Service

**IAM Best Practices**
Principle of Least Privilege: The App Service is granted only the necessary permissions.
Terraform Module: backend_module
The backend_module is used to configure the backend storage for Terraform state files.

**Module Variables**
resource_group_name: The name of the resource group.
storage_account_name: The name of the storage account.
container_name: The name of the storage container.
key: The name of the state file.

**Cleanup**
To destroy the infrastructure, run: terraform destroy
