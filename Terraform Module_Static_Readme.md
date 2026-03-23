# Terraform Module/Static Infrastructure Documentation

---
## Document Details

| Author | Created on | Version | Last updated by | Last edited on | Pre Reviewer | L0 Reviewer | L1 Reviewer | L2 Reviewer |
|--------|------------|---------|-----------------|----------------|--------------|-------------|-------------|-------------|
| Suraj Tripathi | 20-03-2026 | v1.0 | Suraj Tripathi | 20-03-2026 |              | Aniruddh    | Shreya S    | Ashwani |

---

## Table of Contents

1. [Overview](#1-overview)
2. [Purpose](#2-purpose)
3. [Directory Structure](#3-directory-structure)
4. [Environment-Specific Values](#4-environment-specific-values)
5. [Dependencies](#5-dependencies)
6. [Getting Started](#6-getting-started)
7. [Usage](#7-usage)
8. [Inputs](#8-inputs)
9. [Outputs](#9-outputs)
10. [Advanced Configuration](#10-advanced-configuration)
11. [State Management](#11-state-management)
12. [Security Considerations](#12-security-considerations)
13. [Troubleshooting](#13-troubleshooting)
14. [Contributing](#14-contributing)
15. [Contact Information](#15-contact-information)

---

## 1. Overview

The static Terraform infrastructure code configurations that define fixed, long-lived, or baseline cloud resources that are not expected to change frequently but must be consistently reproducible across environments.

Unlike dynamic modules that are parameterized and reused generically, static configurations are opinionated and scoped to a specific infrastructure concern (e.g., a shared VPC, an IAM baseline, a DNS zone, or a global S3 backend).

| Property       | Value                          |
|----------------|--------------------------------|
| Type           | Static                         |
| Category       | Terraform Module/Static        |
| Terraform Min  | `>= 1.3.0`                     |
| Provider Lock  | Yes (see `.terraform.lock.hcl`)|

---

## 2. Purpose

This static configuration was created to:

- Provision a defined set of cloud resources that serve as foundational infrastructure for one or more services.
- Ensure consistency across `dev`, `staging`, and `production` environments by encoding all decisions explicitly.
- Reduce drift by treating infrastructure as reviewed, version-controlled code rather than ad-hoc changes.
- Enable safe modification — anyone on the team can read, understand, and propose changes via pull request.

> **Note:** Because this is *static* infrastructure (not a reusable module), it is not intended to be called as a child module by other configurations. It is applied directly via `terraform apply`.

---

## 3. Directory Structure

```
.
├── main.tf               # Core resource definitions
├── variables.tf          # Input variable declarations
├── outputs.tf            # Output value declarations
├── versions.tf           # Required providers and Terraform version constraints
├── backend.tf            # Remote state backend configuration
├── locals.tf             # Local values and computed expressions
├── data.tf               # Data source lookups (existing resources)
├── environments/
│   ├── dev.tfvars        # Variable overrides for development
│   ├── staging.tfvars    # Variable overrides for staging
│   └── prod.tfvars       # Variable overrides for production
├── modules/              # Any local submodules (if applicable)
├── scripts/              # Helper scripts (init, plan wrappers, etc.)
├── tests/                # Terraform test files (*.tftest.hcl)
├── .terraform.lock.hcl   # Provider version lock file (commit this!)
```

---

## 4. Environment-Specific Values

Each environment (`dev`, `staging`, `prod`) has its own `.tfvars` file under `environments/`. These files override default variable values to reflect environment-specific sizing, naming, and configuration.

### Example: `environments/dev.tfvars`

```hcl
environment         = "dev"
region              = "us-east-1"
instance_type       = "t3.micro"
enable_deletion_protection = false
tags = {
  Team        = "platform"
  CostCenter  = "engineering"
  ManagedBy   = "terraform"
}
```

### Example: `environments/prod.tfvars`

```hcl
environment         = "prod"
region              = "us-east-1"
instance_type       = "m5.large"
enable_deletion_protection = true
tags = {
  Team        = "platform"
  CostCenter  = "engineering"
  ManagedBy   = "terraform"
}
```

> **Note:** Never commit secrets (passwords, tokens, keys) to `.tfvars` files. Use environment variables (`TF_VAR_*`) or a secrets manager integration instead.

---

## 5. Dependencies

### External Dependencies

| Dependency             | Version       | Purpose                              |
|------------------------|---------------|--------------------------------------|
| Terraform CLI          | `>= 1.3.0`    | Infrastructure provisioning engine   |
| AWS Provider           | `~> 5.0`      | Cloud resource management (example)  |
| Remote State Backend   | S3 + DynamoDB | Shared state storage and locking     |

### Infrastructure Dependencies

This configuration **assumes the following already exist** and references them via `data` sources:

- **Remote State**: A Terraform backend S3 bucket and DynamoDB lock table (provisioned separately, typically in a `bootstrap` config).
- **VPC / Networking**: If this config deploys compute resources, the VPC and subnet IDs are expected to exist and are read via `data "aws_vpc"` or passed as variables.
- **IAM Roles**: Any IAM role ARNs required by resources must exist prior to apply.

> See `data.tf` for all `data` source lookups that reflect these dependencies.

### Declaring the Backend (bootstrap prerequisite)

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-org-terraform-state"
    key            = "static/my-component/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

---

## 6. Getting Started

### Prerequisites

- Terraform `>= 1.3.0` installed ([download](https://developer.hashicorp.com/terraform/downloads))
- AWS CLI configured with appropriate credentials (`aws configure` or role assumption)
- Access to the remote state S3 bucket

### 1. Clone the Repository

```bash
git clone https://github.com/your-org/your-infra-repo.git
cd your-infra-repo/path/to/this/static
```

### 2. Initialize Terraform

```bash
terraform init
```

This downloads providers and configures the backend. You should see:

```
Terraform has been successfully initialized!
```

### 3. Validate Configuration

```bash
terraform validate
```

### 4. Preview Changes

```bash
# For development environment
terraform plan -var-file="environments/dev.tfvars"

# For production
terraform plan -var-file="environments/prod.tfvars"
```

### 5. Apply

```bash
terraform apply -var-file="environments/dev.tfvars"
```

Type `yes` when prompted, or use `-auto-approve` in automated pipelines (with caution).

---

## 7. Usage

### Targeting a Specific Resource

To apply or destroy only a specific resource (useful during debugging):

```bash
terraform apply -target="aws_s3_bucket.main" -var-file="environments/dev.tfvars"
```

### Refreshing State

To sync state with real infrastructure without making changes:

```bash
terraform refresh -var-file="environments/dev.tfvars"
```

### Destroying Infrastructure

```bash
terraform destroy -var-file="environments/dev.tfvars"
```

> This is irreversible. Always review the plan output carefully before confirming.

---

## 8. Inputs

All input variables are declared in `variables.tf`. Below is a reference table:

| Name                        | Type           | Default       | Required | Description                                      |
|-----------------------------|----------------|---------------|----------|--------------------------------------------------|
| `environment`               | `string`       | —             | ✅ Yes   | Deployment environment (`dev`, `staging`, `prod`)|
| `region`                    | `string`       | `"us-east-1"` | No       | AWS region to deploy resources into              |
| `instance_type`             | `string`       | `"t3.micro"`  | No       | EC2 instance type (if applicable)                |
| `enable_deletion_protection`| `bool`         | `false`       | No       | Protect key resources from accidental deletion   |
| `tags`                      | `map(string)`  | `{}`          | No       | Tags to apply to all resources                   |

---

## 9. Outputs

Declared in `outputs.tf`. These values are written to the remote state and can be consumed by other configurations.

| Name              | Description                                    |
|-------------------|------------------------------------------------|
| `resource_arn`    | ARN of the primary resource created            |
| `resource_id`     | Unique ID of the primary resource              |
| `endpoint_url`    | Public or internal endpoint (if applicable)    |

### Consuming Outputs in Another Config

```hcl
data "terraform_remote_state" "this_static" {
  backend = "s3"
  config = {
    bucket = "my-org-terraform-state"
    key    = "static/my-component/terraform.tfstate"
    region = "us-east-1"
  }
}

locals {
  resource_arn = data.terraform_remote_state.this_static.outputs.resource_arn
}
```

---

## 10. Advanced Configuration

### Using Workspaces (Alternative to tfvars)

While `.tfvars` files are the recommended approach here, Terraform workspaces can also be used:

```bash
terraform workspace new staging
terraform workspace select staging
terraform plan
```

> Note: Workspaces share the same backend bucket but use isolated state keys. Use them carefully in teams.

### Overriding Variables via Environment Variables

Any variable can be set without a `.tfvars` file using `TF_VAR_` prefixed environment variables:

```bash
export TF_VAR_environment="prod"
export TF_VAR_instance_type="m5.xlarge"
terraform plan
```

### Adding a New Resource

1. Define the resource in `main.tf` (or a new `.tf` file if the config grows large).
2. Add any new required variables to `variables.tf` with descriptions and types.
3. Add relevant outputs to `outputs.tf`.
4. Update all three environment `.tfvars` files with values.
5. Run `terraform validate` and `terraform plan` before opening a pull request.

### Locking Provider Versions

The `.terraform.lock.hcl` file pins exact provider versions. **Always commit this file.** To upgrade a provider:

```bash
terraform init -upgrade
# Review .terraform.lock.hcl diff carefully before committing
```

---

## 11. State Management

- Remote State is stored in S3 with encryption at rest.
- State Locking is handled by DynamoDB to prevent concurrent applies.
- State is environment-isolated— each environment uses a separate `key` in the backend.

### Importing Existing Resources

If a resource was created manually and needs to be brought under Terraform management:

```bash
terraform import aws_s3_bucket.main my-existing-bucket-name
```

After import, run `terraform plan` to verify no unintended changes will be made.

### Moving Resources in State

When refactoring (e.g., renaming a resource block), use `terraform state mv` to avoid destroy/recreate:

```bash
terraform state mv aws_s3_bucket.old_name aws_s3_bucket.new_name
```

---

## 12. Security Considerations

| Concern                    | Mitigation                                                                 |
|----------------------------|----------------------------------------------------------------------------|
| Secrets in state           | Use AWS Secrets Manager / SSM Parameter Store; avoid sensitive outputs     |
| Overprivileged CI role     | Scope IAM permissions to only the resources this config manages            |
| State file access          | Restrict S3 bucket access via bucket policy and IAM                       |
| Unreviewed applies         | Require PR approval + plan review before merging to `main`                |
| Deletion protection        | Set `enable_deletion_protection = true` for all production resources       |

---

## 13. Troubleshooting

### `Error: Backend configuration changed`

Run `terraform init -reconfigure` to re-initialize with updated backend settings.

### `Error acquiring the state lock`

Another process may hold the lock, or a previous run was interrupted. To force-unlock (use with caution):

```bash
terraform force-unlock <LOCK_ID>
```

The lock ID is shown in the error message.

### `Error: Reference to undeclared resource`

A `data` source or resource reference cannot be found. Check:
- The target resource exists in the environment
- The correct `.tfvars` file is being used
- Remote state outputs are accessible from the backend

### Plan shows unexpected destroy

Run `terraform state list` and `terraform state show <resource>` to inspect current state. A destroy may be caused by a renamed resource block — use `terraform state mv` before applying.

---

## 14. Contributing

1. **Branch** from `main` using a descriptive name: `feat/add-cloudwatch-alarms`
2. **Make your changes** and run `terraform validate` + `terraform fmt -recursive`
3. **Plan against dev** to verify intent: `terraform plan -var-file="environments/dev.tfvars"`
4. **Open a Pull Request** — include the plan output and a description of what changes and why
5. **Await review** from at least one team member familiar with this static config
6. **Merge** only after CI passes and approval is granted

### Coding Standards

- Use `terraform fmt` before every commit
- All variables must have a `description`
- All outputs must have a `description`
- Tag every resource using the `var.tags` map
- Avoid hardcoded values — use variables or `locals`

---
## 15. Contact Information

| Contact Type | Details                                                             |
| ------------ | ------------------------------------------------------------------- |
| Name         | Suraj Tripathi                                                      |
| Role         | DevOps Trainee                                                      |
| Email        | [suraj.tripathi.snaatak@mygurukulam.co](mailto:suraj.tripathi.snaatak@mygurukulam.co) |

---
