# Module 08: Infrastructure as Code — Terraform & OpenTofu

> Part of the [DevOps Career Course](./README.md) by UncleJS

[![CC BY-NC-SA 4.0](https://img.shields.io/badge/license-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/) ![Module 08 of 15](https://img.shields.io/badge/module-08%20of%2015-grey) ![Level](https://img.shields.io/badge/level-Intermediate%20%E2%86%92%20Advanced-red) ![Terraform 1.9+](https://img.shields.io/badge/Terraform-1.9%2B-7B42BC?logo=terraform&logoColor=white) ![OpenTofu 1.8+](https://img.shields.io/badge/OpenTofu-1.8%2B-FFDA18?logo=opentofu&logoColor=black) ![HCL · State Management](https://img.shields.io/badge/IaC-HCL%20%C2%B7%20State%20Management-5C4EE5)

---

## Table of Contents

- [Overview](#overview)
- [Learning Objectives](#learning-objectives)
- [Beginner: What is IaC & Why It Matters](#beginner-what-is-iac--why-it-matters)
- [Beginner: Terraform vs OpenTofu — What's the Difference?](#beginner-terraform-vs-opentofu--whats-the-difference)
- [Beginner: Installation](#beginner-installation)
- [Beginner: HCL Language Fundamentals](#beginner-hcl-language-fundamentals)
- [Beginner: The Core Workflow](#beginner-the-core-workflow)
- [Intermediate: Variables, Outputs & Locals](#intermediate-variables-outputs--locals)
- [Intermediate: State Management](#intermediate-state-management)
- [Intermediate: Modules](#intermediate-modules)
- [Intermediate: Providers & Multiple Environments](#intermediate-providers--multiple-environments)
- [Intermediate: Workspaces](#intermediate-workspaces)
- [Intermediate: Testing & Validation](#intermediate-testing--validation)
- [Intermediate: Migrating from Terraform to OpenTofu](#intermediate-migrating-from-terraform-to-opentofu)
- [Advanced: Remote State Backends & Terragrunt](#advanced-remote-state-backends--terragrunt)
- [Tools & Commands Reference](#tools--commands-reference)
- [Hands-On Labs](#hands-on-labs)
- [Further Reading](#further-reading)

---

## Overview

Infrastructure as Code (IaC) means defining your infrastructure — servers, networks, databases, DNS records — in version-controlled configuration files. Instead of clicking through a web console or running manual commands, you write declarative code and let the tool figure out how to make reality match your desired state.

**Terraform** was the dominant open-source IaC tool for years. In 2023, HashiCorp changed Terraform's license from MPL-2.0 to BSL (Business Source License), restricting commercial use. The open-source community forked it as **OpenTofu**, now maintained by the Linux Foundation and hosted under the CNCF.

This module covers both tools — they share 99% of their syntax, and you can use either in practice.

[↑ Back to TOC](#table-of-contents)

---

## Learning Objectives

By the end of this module you will be able to:

- Explain what IaC is and why it matters
- Understand the difference between Terraform and OpenTofu
- Write HCL to provision cloud infrastructure
- Use the core `init`, `plan`, `apply`, `destroy` workflow
- Parameterize configurations with variables and outputs
- Manage remote state safely
- Organize code into reusable modules
- Work with multiple environments (dev/staging/prod)
- Migrate from Terraform to OpenTofu
- Configure remote state backends (S3, GCS, Azure Blob) with locking
- Use Terragrunt to keep IaC DRY across many environments

[↑ Back to TOC](#table-of-contents)

---

## Beginner: What is IaC & Why It Matters

### Before IaC (ClickOps)

- Engineer logs into AWS Console
- Manually creates VPC, subnet, security group, EC2 instance
- Process is undocumented, unrepeatable, error-prone
- A second environment requires doing it all over again manually

### With IaC

```hcl
# This file defines the same infrastructure — repeatable, reviewable, version-controlled
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"  # Replace with a current AMI for your region — see: aws ec2 describe-images
  instance_type = "t3.micro"
  tags = {
    Name = "webserver"
  }
}
```

### Benefits

| Benefit | Description |
|---|---|
| **Repeatability** | Same code → same infrastructure, every time |
| **Version control** | Every change tracked in Git, reviewable via PR |
| **Collaboration** | Team works on infrastructure the same way as code |
| **Automation** | Integrate with CI/CD pipelines |
| **Documentation** | Code IS the documentation |
| **Disaster recovery** | Rebuild from scratch in minutes |

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Terraform vs OpenTofu — What's the Difference?

| Feature | Terraform | OpenTofu |
|---|---|---|
| **License** | BSL 1.1 (since v1.6 — restricts commercial use) | MPL-2.0 (fully open source) |
| **Maintainer** | HashiCorp / IBM | Linux Foundation / OpenTofu TSC |
| **CLI binary** | `terraform` | `tofu` |
| **Compatible with** | — | Terraform ≤ 1.5.x (100% compatible) |
| **Registry** | registry.terraform.io | registry.opentofu.org |
| **New features** | Continues development | Continues independent development |
| **Community** | Large (existing) | Growing rapidly |

### When to Choose Each

- **Choose Terraform**: Your team already uses it, you use HashiCorp's Terraform Cloud, or you need HCP Vault integrations
- **Choose OpenTofu**: You want fully open-source tooling, avoid BSL restrictions, or are starting fresh

### Syntax Differences

For Terraform ≤ 1.5 compatibility, the syntax is **identical**. Newer OpenTofu-specific features (like `provider_meta`, early `for_each` enhancements) may differ. In practice: **almost everything in this module works with both**.

```bash
# Terraform
terraform init
terraform plan
terraform apply

# OpenTofu (same commands, different binary)
tofu init
tofu plan
tofu apply
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Installation

### Terraform

```bash
# Ubuntu/Debian
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

terraform version
```

### OpenTofu

```bash
# Ubuntu/Debian — official installer
curl --proto '=https' --tlsv1.2 -fsSL https://get.opentofu.org/install-opentofu.sh | sudo sh -s -- --install-method deb

# Or via GitHub release
TOFU_VERSION="1.7.0"
curl -Lo tofu.zip "https://github.com/opentofu/opentofu/releases/download/v${TOFU_VERSION}/tofu_${TOFU_VERSION}_linux_amd64.zip"
unzip tofu.zip && sudo mv tofu /usr/local/bin/

tofu version
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: HCL Language Fundamentals

HCL (HashiCorp Configuration Language) is the declarative language used by both Terraform and OpenTofu.

### File Structure

```
project/
├── main.tf          # Main resources
├── variables.tf     # Input variable declarations
├── outputs.tf       # Output value declarations
├── providers.tf     # Provider configuration
├── versions.tf      # Required versions
└── terraform.tfvars # Variable values (gitignored for secrets)
```

### Blocks

```hcl
# Provider block — tells Terraform which cloud to use
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  required_version = ">= 1.5.0"
}

provider "aws" {
  region = "us-east-1"
}

# Resource block — creates infrastructure
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags = {
    Name        = "main-vpc"
    Environment = "production"
  }
}

# Data source block — reads existing infrastructure
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-24.04-amd64-server-*"]
  }
}

# Resource referencing another resource
resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.main.id      # Reference: type.name.attribute
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
}
```

### HCL Data Types

```hcl
# Strings
name = "production"
name = "web-${var.environment}"   # String interpolation

# Numbers
port = 8080
count = 3

# Booleans
enabled = true

# Lists
availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]

# Maps
tags = {
  Environment = "production"
  Owner       = "devops-team"
  CostCenter  = "engineering"
}

# Conditionals
instance_type = var.environment == "production" ? "t3.large" : "t3.micro"

# for_each — create multiple resources from a map
resource "aws_s3_bucket" "env_buckets" {
  for_each = toset(["dev", "staging", "prod"])
  bucket   = "myapp-${each.key}-data"
}

# count — create N copies of a resource
resource "aws_instance" "web" {
  count         = 3
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  tags = {
    Name = "web-${count.index + 1}"
  }
}
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: The Core Workflow

```bash
# 1. Initialize — download providers and modules
terraform init      # or: tofu init
# Downloads provider plugins to .terraform/
# Creates .terraform.lock.hcl (pin provider versions)

# 2. Plan — preview changes (dry run)
terraform plan      # or: tofu plan
# Shows what will be created, changed, or destroyed
# Green = create, yellow = modify, red = destroy

# 3. Apply — create/update infrastructure
terraform apply     # or: tofu apply
# Prompts for confirmation unless you use -auto-approve
terraform apply -auto-approve

# 4. Destroy — tear down all resources
terraform destroy   # or: tofu destroy
# DANGEROUS in production — always review carefully

# Other useful commands
terraform fmt                    # Format code to HCL standard
terraform validate               # Check syntax without hitting cloud API
terraform show                   # Show current state in human-readable form
terraform state list             # List all resources in state
terraform output                 # Show output values
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Variables, Outputs & Locals

### Variables (variables.tf)

```hcl
variable "environment" {
  description = "Deployment environment (dev, staging, prod)"
  type        = string
  default     = "dev"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "allowed_cidrs" {
  description = "List of CIDRs allowed SSH access"
  type        = list(string)
  default     = []
}

variable "tags" {
  description = "Common tags for all resources"
  type        = map(string)
  default     = {}
}

# Sensitive variable (value masked in plan output)
variable "db_password" {
  type      = string
  sensitive = true
}
```

### Setting Variable Values

```bash
# Method 1: terraform.tfvars file
# environment = "production"
# instance_type = "t3.large"

# Method 2: Command line
terraform apply -var="environment=production"

# Method 3: Environment variables (prefix TF_VAR_)
export TF_VAR_environment=production
export TF_VAR_db_password=secret123

# Method 4: -var-file
terraform apply -var-file="production.tfvars"
```

### Outputs (outputs.tf)

```hcl
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "db_endpoint" {
  value     = aws_db_instance.main.endpoint
  sensitive = true
}
```

### Locals

```hcl
locals {
  common_tags = merge(var.tags, {
    Environment = var.environment
    ManagedBy   = "terraform"
    Project     = "myapp"
  })

  name_prefix = "${var.environment}-myapp"
}

resource "aws_instance" "web" {
  tags = local.common_tags
}
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: State Management

State is how Terraform/OpenTofu tracks what infrastructure exists. By default, it's stored in `terraform.tfstate` locally.

### Remote State (Required for Teams)

```hcl
# backend.tf — AWS S3 backend
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "production/myapp/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"   # Prevent concurrent modifications
    encrypt        = true
  }
}

# OpenTofu also supports S3, GCS, Azure Blob, HTTP, and more
# backend.tf — GCS backend
terraform {
  backend "gcs" {
    bucket = "mycompany-tofu-state"
    prefix = "production/myapp"
  }
}
```

### State Commands

```bash
# List all managed resources
terraform state list

# Show a specific resource
terraform state show aws_instance.web

# Move a resource (rename without recreating)
terraform state mv aws_instance.web aws_instance.webserver

# Remove a resource from state (without destroying it)
terraform state rm aws_instance.old_web

# Import existing infrastructure into state
terraform import aws_instance.web i-1234567890abcdef0
```

> ⚠️ **Critical**: Never edit `terraform.tfstate` manually. Never commit sensitive state to Git. Always use remote state with locking in team environments.

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Modules

Modules are reusable packages of Terraform/OpenTofu configuration.

### Module Directory Structure

```
modules/
└── vpc/
    ├── main.tf          # VPC resources
    ├── variables.tf     # Module inputs
    ├── outputs.tf       # Module outputs
    └── README.md
```

### Writing a Module

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  tags = {
    Name = var.name
  }
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnets)
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.public_subnets[count.index]
  availability_zone = var.azs[count.index]
}

# modules/vpc/variables.tf
variable "name" { type = string }
variable "cidr_block" { type = string }
variable "public_subnets" { type = list(string) }
variable "azs" { type = list(string) }

# modules/vpc/outputs.tf
output "vpc_id" { value = aws_vpc.this.id }
output "public_subnet_ids" { value = aws_subnet.public[*].id }
```

### Using a Module

```hcl
# main.tf
module "vpc" {
  source = "./modules/vpc"      # Local module

  name           = "production-vpc"
  cidr_block     = "10.0.0.0/16"
  public_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  azs            = ["us-east-1a", "us-east-1b"]
}

# Use the module's outputs
resource "aws_instance" "web" {
  subnet_id = module.vpc.public_subnet_ids[0]
}

# Registry module (from registry.terraform.io or registry.opentofu.org)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"
  azs  = ["us-east-1a", "us-east-1b"]
}
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Providers & Multiple Environments

### Environment Separation Strategy

```
environments/
├── dev/
│   ├── main.tf
│   └── terraform.tfvars
├── staging/
│   ├── main.tf
│   └── terraform.tfvars
└── prod/
    ├── main.tf
    └── terraform.tfvars
```

Each environment directory calls the same modules with different values:

```hcl
# environments/prod/main.tf
module "app" {
  source        = "../../modules/app"
  environment   = "prod"
  instance_type = "t3.large"
  min_replicas  = 3
  max_replicas  = 20
}
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Workspaces

Workspaces let you maintain multiple state files in the same configuration directory.

```bash
terraform workspace new staging       # Create staging workspace
terraform workspace new production    # Create production workspace
terraform workspace list              # List all workspaces
terraform workspace select staging    # Switch to staging
terraform workspace show              # Current workspace

# Use workspace name in code
resource "aws_s3_bucket" "app" {
  bucket = "myapp-${terraform.workspace}-data"
}
```

> **Note**: Workspaces work well for minor variations. For significantly different environments (different region, account, or major infra differences), use separate directories.

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Testing & Validation

```bash
# Built-in validation
terraform validate        # Check syntax and internal consistency
tofu validate

# Format check
terraform fmt -check      # Exit code 1 if formatting needed
terraform fmt -recursive  # Format all .tf files in subdirectories

# Security scanning with tfsec
tfsec .                   # Scan for misconfigurations and security issues

# Cost estimation with Infracost
infracost breakdown --path .
infracost diff --path .

# Terraform-native testing (Terraform 1.6+ / OpenTofu 1.6+)
# tests/main.tftest.hcl
run "creates_vpc" {
  command = plan
  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR block must be 10.0.0.0/16"
  }
}
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Migrating from Terraform to OpenTofu

OpenTofu is compatible with Terraform ≤ 1.5.x. Migration is straightforward.

```bash
# Step 1: Install OpenTofu
# Step 2: In your existing project:

# Replace the binary (most cases — just substitute)
alias terraform=tofu

# Step 3: Re-initialize with OpenTofu
tofu init

# Step 4: Verify the plan is identical
tofu plan

# Step 5: If you use provider lock file, update it
tofu providers lock

# For Terraform Cloud / HCP Terraform migrations:
# - Export state from TFC workspace
# - Configure a compatible backend (S3, GCS, etc.)
# - Import state with: tofu state push terraform.tfstate
```

### What to Watch For

| Item | Notes |
|---|---|
| Provider versions | OpenTofu registry: `registry.opentofu.org` (same providers) |
| State format | Compatible — no migration needed |
| `.terraform.lock.hcl` | Re-generate with `tofu providers lock` |
| Modules | 100% compatible |
| New OpenTofu features | Some features (like `tofu test`) have OpenTofu-specific syntax |

[↑ Back to TOC](#table-of-contents)

---

## Advanced: Remote State Backends & Terragrunt

### Remote State Backends

By default, Terraform/OpenTofu stores state in a local `terraform.tfstate` file. In a team environment this is dangerous — two engineers running `apply` simultaneously can corrupt state. Remote backends solve this with **shared storage + state locking**.

#### AWS S3 + DynamoDB (most common)

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/web/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"    # For state locking
    encrypt        = true
  }
}
```

```bash
# Create the S3 bucket and DynamoDB table (one-time setup)
aws s3api create-bucket --bucket my-terraform-state --region us-east-1
aws s3api put-bucket-versioning --bucket my-terraform-state \
  --versioning-configuration Status=Enabled

aws dynamodb create-table \
  --table-name terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

#### GCS (Google Cloud Storage)

```hcl
terraform {
  backend "gcs" {
    bucket = "my-terraform-state"
    prefix = "prod/web"
  }
}
```

#### Azure Blob Storage

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate-rg"
    storage_account_name = "tfstateaccount"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

### Reading Remote State from Another Module

```hcl
# Reference outputs from a different state file (e.g., network module)
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "prod/network/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "app" {
  ami           = "ami-abc123"
  instance_type = "t3.micro"
  subnet_id     = data.terraform_remote_state.network.outputs.private_subnet_id
}
```

### Terragrunt — DRY Infrastructure at Scale

As infrastructure grows, copy-pasting backend configurations and provider blocks across many modules becomes error-prone. **Terragrunt** is a thin wrapper that generates boilerplate and enforces DRY patterns.

```
(typical Terragrunt layout)
infrastructure/
├── terragrunt.hcl           ← root config (backend, provider defaults)
├── prod/
│   ├── terragrunt.hcl       ← env-level overrides
│   ├── network/
│   │   └── terragrunt.hcl   ← module instance
│   └── web/
│       └── terragrunt.hcl
└── staging/
    ├── terragrunt.hcl
    ├── network/
    │   └── terragrunt.hcl
    └── web/
        └── terragrunt.hcl
```

```hcl
# infrastructure/terragrunt.hcl  (root)
locals {
  account_id = get_aws_account_id()
  region     = "us-east-1"
  env        = path_relative_to_include()
}

remote_state {
  backend = "s3"
  config = {
    bucket         = "my-terraform-state-${local.account_id}"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = local.region
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region = "${local.region}"
}
EOF
}
```

```hcl
# infrastructure/prod/web/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "../../../modules/web-tier"
}

inputs = {
  instance_type = "t3.medium"
  min_size      = 3
  max_size      = 10
  environment   = "prod"
}
```

```bash
# Install Terragrunt
brew install terragrunt          # macOS
# or download from https://github.com/gruntwork-io/terragrunt/releases

# Run against a single module
terragrunt apply

# Run against all modules in a directory tree (respects dependencies)
terragrunt run-all apply

# Plan everything in staging
terragrunt run-all plan --terragrunt-working-dir infrastructure/staging
```

### Backend Migration

Moving from local to remote state (or between backends):

```bash
# 1. Update backend block in backend.tf
# 2. Re-initialize — Terraform will offer to copy existing state
terraform init -migrate-state

# Verify state was migrated correctly
terraform state list

# Remove local state file only after confirming remote is correct
rm terraform.tfstate terraform.tfstate.backup
```

[↑ Back to TOC](#table-of-contents)

---

## Tools & Commands Reference

| Command | Terraform | OpenTofu |
|---|---|---|
| Initialize | `terraform init` | `tofu init` |
| Preview changes | `terraform plan` | `tofu plan` |
| Apply changes | `terraform apply` | `tofu apply` |
| Destroy all | `terraform destroy` | `tofu destroy` |
| Format code | `terraform fmt` | `tofu fmt` |
| Validate syntax | `terraform validate` | `tofu validate` |
| Show state | `terraform state list` | `tofu state list` |
| Import resource | `terraform import` | `tofu import` |
| Show outputs | `terraform output` | `tofu output` |
| Workspaces | `terraform workspace` | `tofu workspace` |

[↑ Back to TOC](#table-of-contents)

---

## Hands-On Labs

### Lab 8.1 — Install Both Tools

1. Install Terraform and OpenTofu on your machine
2. Verify: `terraform version` and `tofu version`
3. Note the version numbers and compare the changelogs

### Lab 8.2 — Your First Infrastructure (Local with LocalStack or AWS free tier)

1. Create a directory with `main.tf`, `variables.tf`, `outputs.tf`
2. Configure the AWS provider
3. Write a resource to create an S3 bucket with your name
4. Run `terraform init`, `terraform plan`, `terraform apply`
5. Verify the bucket exists in the console or via `aws s3 ls`
6. Run `terraform destroy`
7. Repeat all steps using `tofu` instead

### Lab 8.3 — Variables & Outputs

1. Add a `variable "bucket_name"` and a `variable "environment"` to your config
2. Create a `terraform.tfvars` with your values
3. Add a `tags` map to the bucket using `local` computed tags
4. Output the bucket ARN and region
5. Run `terraform apply` and verify the outputs

### Lab 8.4 — Build a Module

1. Create a `modules/s3-bucket` directory with `main.tf`, `variables.tf`, `outputs.tf`
2. The module should accept: `bucket_name`, `environment`, `versioning_enabled`
3. Use the module from your root `main.tf` to create 3 buckets for dev/staging/prod
4. Run plan and apply — verify 3 buckets are created

### Lab 8.5 — Remote State

1. Create an S3 bucket manually for storing state (or use a GCS bucket)
2. Configure an S3 backend in your project
3. Run `terraform init` and migrate state to S3
4. Verify the state file exists in S3
5. Delete your local `terraform.tfstate` and run `terraform plan` — it should read from S3

[↑ Back to TOC](#table-of-contents)

---

## Further Reading

- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
- [OpenTofu Documentation](https://opentofu.org/docs/)
- [OpenTofu GitHub](https://github.com/opentofu/opentofu)
- [Terraform Best Practices](https://www.terraform-best-practices.com/)
- [tfsec — IaC Security Scanner](https://aquasecurity.github.io/tfsec/)
- [Infracost — Cost Estimation](https://www.infracost.io/)
- [Glossary: IaC](./glossary.md#i), [HCL](./glossary.md#h), [State](./glossary.md#s), [OpenTofu](./glossary.md#o), [Module](./glossary.md#m)
- **Certification**: HashiCorp Certified Terraform Associate

[↑ Back to TOC](#table-of-contents)

---

*© 2026 UncleJS — Licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)*
