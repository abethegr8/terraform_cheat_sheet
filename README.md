# Terraform Everyday Cheat Sheet

A practical Terraform reference for everyday infrastructure engineering work.

**Primary cloud:** Microsoft Azure ☁️
**Also includes:** AWS examples where useful

This cheat sheet focuses on commands, HCL patterns, state management, modules, troubleshooting, and workflows that are useful when actually working with Terraform.

---

## Table of Contents

1. [Terraform Mental Model](#terraform-mental-model)
2. [Project Structure](#project-structure)
3. [Core Terraform Workflow](#core-terraform-workflow)
4. [Azure Provider](#azure-provider)
5. [Azure Authentication](#azure-authentication)
6. [Resources](#resources)
7. [Data Sources](#data-sources)
8. [Variables](#variables)
9. [Locals](#locals)
10. [Outputs](#outputs)
11. [Resource References](#resource-references)
12. [count vs for_each](#count-vs-for_each)
13. [Modules](#modules)
14. [State](#state)
15. [Azure Remote State](#azure-remote-state)
16. [Import Existing Infrastructure](#import-existing-infrastructure)
17. [Remove Resources From State](#remove-resources-from-state)
18. [Workspaces](#workspaces)
19. [Lifecycle](#lifecycle)
20. [Dependencies](#dependencies)
21. [Terraform Console](#terraform-console)
22. [Logging and Troubleshooting](#logging-and-troubleshooting)
23. [Useful Terraform Commands](#useful-terraform-commands)
24. [Git and Terraform](#git-and-terraform)
25. [HCP Terraform](#hcp-terraform)
26. [AWS Quick Reference](#aws-quick-reference)
27. [Everyday Workflow](#everyday-workflow)
28. [Quick Mental Models](#quick-mental-models)

---

# Terraform Mental Model

Terraform compares configuration, state, and real infrastructure to determine what changes are required.

```text
          Terraform Configuration
                  |
                  v
            terraform plan
                  |
        +---------+---------+
        |                   |
        v                   v
 Terraform State      Real Infrastructure
        |                   |
        +---------+---------+
                  |
                  v
           Desired Changes
```

* **Configuration** describes what you want.
* **State** records what Terraform manages.
* **Providers** communicate with infrastructure APIs.
* **Plan** determines the difference between desired and actual infrastructure.

---

# Project Structure

A common Terraform project structure:

```text
terraform-project/
│
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
├── versions.tf
├── terraform.tfvars
│
└── modules/
    ├── network/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    │
    └── compute/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

Terraform loads all `.tf` files in the current working directory together.

The filenames are primarily for organization.

---

# Core Terraform Workflow

## Initialize

```bash
terraform init
```

Initializes the working directory.

Typically:

* Initializes the backend
* Downloads providers
* Downloads modules
* Creates `.terraform/`
* Creates or updates `.terraform.lock.hcl`

Upgrade providers and modules within configured constraints:

```bash
terraform init -upgrade
```

Reconfigure a backend:

```bash
terraform init -reconfigure
```

---

## Format

```bash
terraform fmt
```

Format recursively:

```bash
terraform fmt -recursive
```

Check formatting without modifying files:

```bash
terraform fmt -check
```

---

## Validate

```bash
terraform validate
```

Checks Terraform configuration for syntax and internal consistency.

A working directory must be initialized before running `terraform validate`.

---

## Plan

```bash
terraform plan
```

Save a plan:

```bash
terraform plan -out=tfplan
```

Inspect the saved plan:

```bash
terraform show tfplan
```

---

## Apply

```bash
terraform apply
```

Apply a previously saved plan:

```bash
terraform apply tfplan
```

Automatic approval:

```bash
terraform apply -auto-approve
```

Use `-auto-approve` carefully.

---

## Destroy

```bash
terraform destroy
```

Preview destruction:

```bash
terraform plan -destroy
```

---

# Azure Provider

Basic AzureRM provider configuration:

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 5.0"
    }
  }
}

provider "azurerm" {
  features {}
}
```

Initialize:

```bash
terraform init
```

---

# Azure Authentication

## Local Development

Azure CLI authentication is convenient for local Terraform development.

Login:

```bash
az login
```

Show the current subscription:

```bash
az account show
```

List subscriptions:

```bash
az account list --output table
```

Select a subscription:

```bash
az account set --subscription "<subscription-id>"
```

Terraform can then use the Azure CLI authentication context.

---

## CI/CD

For automated Terraform runs, prefer workload identities such as:

* Managed Identity
* Service Principal
* OpenID Connect / Workload Identity Federation

Avoid placing credentials directly inside Terraform configuration files.

---

# Resources

Resources represent infrastructure Terraform manages.

```hcl
resource "azurerm_resource_group" "example" {
  name     = "rg-terraform-demo"
  location = "East US"
}
```

Resource address:

```text
azurerm_resource_group.example
```

General format:

```text
resource_type.resource_name
```

---

# Data Sources

A **resource** manages infrastructure.

A **data source** reads information about existing infrastructure.

```hcl
data "azurerm_resource_group" "existing" {
  name = "rg-production"
}
```

Reference:

```hcl
data.azurerm_resource_group.existing.location
```

Mental model:

```text
resource
   |
   +---- CREATE / UPDATE / DELETE

data
   |
   +---- READ
```

---

# Variables

Declare a variable:

```hcl
variable "location" {
  type        = string
  description = "Azure region"
  default     = "East US"
}
```

Reference it:

```hcl
var.location
```

---

## terraform.tfvars

```hcl
location = "West US 2"
```

---

## Variable Types

Common types:

```text
string
number
bool

list(string)
set(string)
map(string)

tuple([])
object({})
```

Example list:

```hcl
variable "subnets" {
  type = list(string)

  default = [
    "10.0.1.0/24",
    "10.0.2.0/24"
  ]
}
```

Example map:

```hcl
variable "tags" {
  type = map(string)

  default = {
    environment = "dev"
    managed_by  = "terraform"
  }
}
```

---

# Locals

Locals help avoid repeating expressions or values.

```hcl
locals {
  common_tags = {
    environment = "dev"
    managed_by  = "terraform"
  }
}
```

Reference:

```hcl
tags = local.common_tags
```

Mental model:

```text
variable = input coming INTO Terraform

local = calculated/reusable value INSIDE Terraform

output = value Terraform exposes OUT
```

---

# Outputs

Declare an output:

```hcl
output "resource_group_id" {
  value = azurerm_resource_group.example.id
}
```

Display outputs:

```bash
terraform output
```

Display one output:

```bash
terraform output resource_group_id
```

---

# Resource References

Terraform automatically creates dependencies when one resource references another.

```hcl
resource "azurerm_virtual_network" "main" {
  name                = "vnet-demo"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name

  address_space = [
    "10.0.0.0/16"
  ]
}
```

Terraform understands:

```text
Resource Group
      |
      v
Virtual Network
```

The VNet depends on the resource group because its attributes are referenced.

---

# count vs for_each

## count

Use `count` when creating **N nearly identical resources**.

```hcl
resource "azurerm_resource_group" "example" {
  count = 3

  name     = "rg-demo-${count.index}"
  location = "East US"
}
```

Addresses:

```text
azurerm_resource_group.example[0]
azurerm_resource_group.example[1]
azurerm_resource_group.example[2]
```

Mental model:

```text
"I need 3 of these."
```

---

## for_each

Use `for_each` when resources have **individual identities**.

```hcl
variable "subnets" {
  default = {
    web = "10.0.1.0/24"
    app = "10.0.2.0/24"
    db  = "10.0.3.0/24"
  }
}
```

```hcl
resource "azurerm_subnet" "subnet" {
  for_each = var.subnets

  name                 = each.key
  address_prefixes     = [each.value]
  virtual_network_name = azurerm_virtual_network.main.name
  resource_group_name  = azurerm_resource_group.example.name
}
```

Addresses:

```text
azurerm_subnet.subnet["web"]
azurerm_subnet.subnet["app"]
azurerm_subnet.subnet["db"]
```

Quick rule:

```text
count
   |
   +---- quantity
         "I need 3 of these"

for_each
   |
   +---- identity
         "I need web, app, and db"
```

---

# Modules

A module is a reusable collection of Terraform configuration.

```text
Root Module
     |
     | inputs
     v
Child Module
     |
     | outputs
     v
Root Module
```

---

## Local Module

```hcl
module "network" {
  source = "./modules/network"

  resource_group_name = "rg-network"
  location            = "East US"
}
```

---

## Public Registry Module

Example source syntax:

```hcl
module "network" {
  source = "Azure/network/azurerm"

  # module inputs...
}
```

Registry source format:

```text
namespace/name/provider
```

Pin module versions when appropriate:

```hcl
module "example" {
  source  = "namespace/module/provider"
  version = "~> 1.0"

  # inputs...
}
```

---

## Module Inputs

Parent/root module:

```hcl
module "network" {
  source = "./modules/network"

  location = "East US"
}
```

Child module:

```hcl
variable "location" {
  type = string
}
```

Flow:

```text
Root Module
     |
     | location = "East US"
     v
Child Module
     |
     v
var.location
```

---

## Module Outputs

Child module:

```hcl
output "vnet_id" {
  value = azurerm_virtual_network.main.id
}
```

Parent module:

```hcl
module.network.vnet_id
```

Flow:

```text
Root Module
     |
     | INPUT
     v
Child Module
     |
     | OUTPUT
     v
Root Module
```

---

# State

Terraform state maps Terraform resource addresses to real infrastructure.

Default local state:

```text
terraform.tfstate
```

Useful commands:

```bash
terraform state list
```

Show one resource:

```bash
terraform state show <resource-address>
```

Example:

```bash
terraform state show azurerm_resource_group.example
```

Remove something from state:

```bash
terraform state rm <resource-address>
```

Move a state address:

```bash
terraform state mv <source> <destination>
```

Pull remote state:

```bash
terraform state pull
```

Avoid manually editing `terraform.tfstate`.

Mental model:

```text
Terraform Configuration
          |
          v
     Terraform State
          |
          v
   Azure Resource ID
          |
          v
   Actual Infrastructure
```

---

# Azure Remote State

A common Azure backend architecture:

```text
Terraform
    |
    v
AzureRM Backend
    |
    v
Storage Account
    |
    v
Blob Container
    |
    v
terraform.tfstate
```

Example:

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "stterraformstate"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

Backend configuration cannot reference standard Terraform variables.

For environment-specific backend settings, partial backend configuration can be useful.

Example:

```bash
terraform init -backend-config=backend.hcl
```

Example `backend.hcl`:

```hcl
resource_group_name  = "rg-tfstate"
storage_account_name = "stterraformstate"
container_name       = "tfstate"
key                  = "prod.terraform.tfstate"
```

---

# Import Existing Infrastructure

Import is used when infrastructure already exists and you want Terraform to begin managing it.

---

## CLI Import

First create the matching resource configuration:

```hcl
resource "azurerm_resource_group" "existing" {
  name     = "rg-existing"
  location = "East US"
}
```

Then import the Azure resource:

```bash
terraform import azurerm_resource_group.existing <azure-resource-id>
```

Conceptually:

```text
Terraform Resource Address
           |
           v
       State
           |
           v
Existing Azure Resource
```

Import does **not** mean Terraform created the infrastructure.

It associates existing infrastructure with a Terraform resource address.

---

## Import Block

Modern Terraform also supports configuration-driven imports.

```hcl
import {
  to = azurerm_resource_group.existing
  id = "<azure-resource-id>"
}
```

Then:

```bash
terraform plan
```

```bash
terraform apply
```

---

# Remove Resources From State

Sometimes you want Terraform to stop managing infrastructure **without destroying it**.

Traditional approach:

```bash
terraform state rm azurerm_resource_group.example
```

Modern configuration-driven approach:

```hcl
removed {
  from = azurerm_resource_group.example

  lifecycle {
    destroy = false
  }
}
```

Mental model:

```text
BEFORE

Terraform State ──────> Azure Resource


AFTER

Terraform State        Azure Resource
      X                      |
                             |
                       still exists
```

---

# Workspaces

List workspaces:

```bash
terraform workspace list
```

Create:

```bash
terraform workspace new dev
```

Select:

```bash
terraform workspace select prod
```

Show current workspace:

```bash
terraform workspace show
```

Reference the workspace in configuration:

```hcl
terraform.workspace
```

Mental model:

```text
Same Terraform Configuration
            |
     +------+------+
     |      |      |
     v      v      v
   dev     test   prod
  state    state  state
```

CLI workspaces primarily provide separate state instances for the same Terraform configuration.

Do not confuse CLI workspaces with HCP Terraform workspaces.

---

# Lifecycle

Terraform lifecycle settings modify how Terraform manages resources.

## prevent_destroy

```hcl
resource "azurerm_resource_group" "example" {
  name     = "rg-production"
  location = "East US"

  lifecycle {
    prevent_destroy = true
  }
}
```

Prevents Terraform from destroying the resource.

---

## create_before_destroy

```hcl
lifecycle {
  create_before_destroy = true
}
```

Creates the replacement before destroying the existing resource when possible.

---

## ignore_changes

```hcl
lifecycle {
  ignore_changes = [
    tags
  ]
}
```

Terraform ignores changes to the specified attributes.

Use this deliberately because Terraform will no longer reconcile those selected changes.

---

# Dependencies

Terraform normally detects dependencies automatically through references.

```hcl
resource "azurerm_virtual_network" "example" {
  resource_group_name = azurerm_resource_group.example.name
}
```

This creates an **implicit dependency**.

```text
Resource Group
      |
      v
Virtual Network
```

Explicit dependency:

```hcl
depends_on = [
  azurerm_resource_group.example
]
```

Prefer implicit dependencies when possible.

---

# Terraform Console

Launch:

```bash
terraform console
```

Useful for testing Terraform expressions and functions.

```text
> upper("azure")
"AZURE"
```

```text
> length(["web", "app", "db"])
3
```

```text
> cidrsubnet("10.0.0.0/16", 8, 1)
"10.0.1.0/24"
```

Exit:

```text
exit
```

---

# Logging and Troubleshooting

Terraform supports detailed logging through environment variables.

Common levels:

```text
TRACE
DEBUG
INFO
WARN
ERROR
```

---

## Linux / macOS

Enable debug logging:

```bash
export TF_LOG=DEBUG
```

Enable maximum verbosity:

```bash
export TF_LOG=TRACE
```

Save logs to a file:

```bash
export TF_LOG_PATH=terraform.log
```

Run Terraform:

```bash
terraform plan
```

Disable logging:

```bash
unset TF_LOG
unset TF_LOG_PATH
```

---

## PowerShell

Enable debug logging:

```powershell
$env:TF_LOG="DEBUG"
```

Write logs to a file:

```powershell
$env:TF_LOG_PATH="terraform.log"
```

Run:

```powershell
terraform plan
```

Disable:

```powershell
Remove-Item Env:TF_LOG
Remove-Item Env:TF_LOG_PATH
```

---

## Troubleshooting Workflow

A useful order when Terraform behaves unexpectedly:

```text
terraform fmt
      |
      v
terraform validate
      |
      v
terraform plan
      |
      v
terraform state list
      |
      v
terraform state show
      |
      v
TF_LOG=DEBUG
      |
      v
TF_LOG=TRACE
```

---

# Useful Terraform Commands

## Version

```bash
terraform version
```

---

## Initialize

```bash
terraform init
```

```bash
terraform init -upgrade
```

```bash
terraform init -reconfigure
```

---

## Format

```bash
terraform fmt
```

```bash
terraform fmt -recursive
```

---

## Validate

```bash
terraform validate
```

---

## Plan

```bash
terraform plan
```

```bash
terraform plan -out=tfplan
```

---

## Apply

```bash
terraform apply
```

```bash
terraform apply tfplan
```

---

## Destroy

```bash
terraform destroy
```

---

## Show

```bash
terraform show
```

---

## Outputs

```bash
terraform output
```

---

## Providers

```bash
terraform providers
```

---

## State

```bash
terraform state list
```

```bash
terraform state show <resource>
```

```bash
terraform state rm <resource>
```

```bash
terraform state mv <source> <destination>
```

```bash
terraform state pull
```

---

## Workspaces

```bash
terraform workspace list
```

```bash
terraform workspace new dev
```

```bash
terraform workspace select dev
```

```bash
terraform workspace show
```

---

## Console

```bash
terraform console
```

---

## Dependency Graph

```bash
terraform graph
```

---

## Replace a Resource

Force Terraform to replace a resource:

```bash
terraform apply -replace="azurerm_linux_virtual_machine.example"
```

Prefer this workflow over the older `terraform taint` command.

---

## Target a Resource

Plan for a specific resource:

```bash
terraform plan -target="azurerm_resource_group.example"
```

Apply against a specific target:

```bash
terraform apply -target="azurerm_resource_group.example"
```

`-target` is intended for exceptional situations rather than normal Terraform workflow.

---

# Git and Terraform

Recommended `.gitignore`:

```gitignore
# Terraform working directory
**/.terraform/*

# State files
*.tfstate
*.tfstate.*

# Crash logs
crash.log
crash.*.log

# Saved plans
*.tfplan

# Override files
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Sensitive variable files
*.tfvars
*.tfvars.json

# Terraform CLI configuration
.terraformrc
terraform.rc
```

Generally commit:

```text
*.tf
.terraform.lock.hcl
README.md
```

Generally do **not** commit:

```text
.terraform/
terraform.tfstate
terraform.tfstate.backup
secret tfvars
credentials
saved Terraform plans containing sensitive data
```

---

# HCP Terraform

HCP Terraform can provide:

* Remote state
* Remote execution
* Workspaces
* VCS integration
* Variables
* Run history
* Team collaboration
* Private module registry
* Policy and governance capabilities

Example:

```hcl
terraform {
  cloud {
    organization = "example-org"

    workspaces {
      name = "azure-production"
    }
  }
}
```

Login:

```bash
terraform login
```

Mental model:

```text
Git Repository
      |
      v
HCP Terraform Workspace
      |
      +---- Configuration
      +---- Variables
      +---- State
      +---- Plans
      +---- Runs
      |
      v
Azure
```

An HCP Terraform workspace is more than just a separate state file.

It represents a collection of Terraform infrastructure and its associated configuration, state, variables, and runs.

---

# AWS Quick Reference

Terraform concepts remain largely the same across providers.

Only the provider and resource implementations change.

---

## Azure Provider

```hcl
terraform {
  required_providers {
    azurerm = {
      source = "hashicorp/azurerm"
    }
  }
}
```

---

## AWS Provider

```hcl
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
    }
  }
}
```

---

## Azure vs AWS

| Concept            | Azure                            | AWS                         |
| ------------------ | -------------------------------- | --------------------------- |
| Provider           | `azurerm`                        | `aws`                       |
| Resource Group     | `azurerm_resource_group`         | N/A                         |
| Virtual Network    | `azurerm_virtual_network`        | `aws_vpc`                   |
| Subnet             | `azurerm_subnet`                 | `aws_subnet`                |
| Virtual Machine    | `azurerm_linux_virtual_machine`  | `aws_instance`              |
| Storage            | `azurerm_storage_account`        | `aws_s3_bucket`             |
| Network Security   | `azurerm_network_security_group` | `aws_security_group`        |
| Load Balancer      | `azurerm_lb`                     | `aws_lb`                    |
| Managed Kubernetes | `azurerm_kubernetes_cluster`     | `aws_eks_cluster`           |
| Secrets            | `azurerm_key_vault`              | `aws_secretsmanager_secret` |

Terraform still follows the same overall lifecycle:

```text
Write
  |
  v
Init
  |
  v
Validate
  |
  v
Plan
  |
  v
Review
  |
  v
Apply
```

---

# Everyday Workflow

A practical sequence for everyday Terraform work.

Start by updating your local Git repository:

```bash
git pull
```

Make Terraform configuration changes.

Format:

```bash
terraform fmt -recursive
```

Validate:

```bash
terraform validate
```

Plan:

```bash
terraform plan
```

Review carefully.

Common Terraform plan symbols:

```text
+     create

~     update in-place

-     destroy

-/+   destroy and recreate
```

Apply:

```bash
terraform apply
```

Then commit your changes:

```bash
git add .
```

```bash
git commit -m "Update Terraform infrastructure"
```

```bash
git push
```

---

## Team Workflow

A common team workflow:

```text
Feature Branch
      |
      v
Terraform Changes
      |
      v
terraform fmt
      |
      v
terraform validate
      |
      v
terraform plan
      |
      v
Pull Request
      |
      v
Code + Plan Review
      |
      v
Merge
      |
      v
CI/CD Pipeline
      |
      v
terraform apply
      |
      v
Azure
```

---

# Quick Mental Models

## Resource vs Data

```text
resource = MANAGE

data = READ
```

---

## Variables / Locals / Outputs

```text
variable → IN

local → INTERNAL

output → OUT
```

---

## count vs for_each

```text
count
  |
  +---- quantity

for_each
  |
  +---- identity
```

Or:

```text
count
"I need 3 servers."

for_each
"I need web, app, and database servers."
```

---

## Terraform State

```text
Configuration
      |
      v
    State
      |
      v
Infrastructure
```

---

## Modules

```text
Root Module
     |
   inputs
     |
     v
Child Module
     |
   outputs
     |
     v
Root Module
```

---

## Core Workflow

```text
INIT
  |
  v
FORMAT
  |
  v
VALIDATE
  |
  v
PLAN
  |
  v
APPLY
```

---

# Useful Resources

* HashiCorp Terraform Documentation
* Terraform Registry
* AzureRM Provider Documentation
* Azure Verified Modules
* HCP Terraform Documentation
* AWS Terraform Provider Documentation

---

# Purpose

This repository is intended to be a living Terraform reference.

As new Terraform patterns, Azure services, commands, troubleshooting techniques, and real-world lessons are encountered, they can be added here.

The goal is not to memorize every Terraform command.

The goal is to understand how Terraform fits together:

```text
Configuration
      |
      v
Providers
      |
      v
Plan
      |
      v
State
      |
      v
Infrastructure
```

> **Learn Terraform by building infrastructure, reading the plan, understanding the state, breaking things in the lab, and figuring out how Terraform puts them back together.**
