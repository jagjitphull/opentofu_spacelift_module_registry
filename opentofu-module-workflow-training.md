# OpenTofu Module Workflow with Spacelift Registry

> **Hands-On Lab**: Create → Publish → Consume → Update

| | |
|---|---|
| **Duration** | 90 - 120 minutes |
| **Difficulty** | Intermediate |
| **Cloud Provider** | AWS |

---

## Table of Contents

- [Section 1: Lab Overview and Prerequisites](#section-1-lab-overview-and-prerequisites)
- [Section 2: Understanding the Module Lifecycle](#section-2-understanding-the-module-lifecycle)
- [Section 3: Create Your First Module](#section-3-create-your-first-module)
- [Section 4: Register Module in Spacelift](#section-4-register-module-in-spacelift)
- [Section 5: Consume the Module in a Stack](#section-5-consume-the-module-in-a-stack)
- [Section 6: Update and Version the Module](#section-6-update-and-version-the-module)
- [Section 7: Verification and Testing](#section-7-verification-and-testing)
- [Section 8: Troubleshooting Guide](#section-8-troubleshooting-guide)
- [Section 9: Best Practices and Next Steps](#section-9-best-practices-and-next-steps)
- [Quick Reference Card](#quick-reference-card)

---

## Section 1: Lab Overview and Prerequisites

### 1.1 Learning Objectives

By the end of this lab, you will be able to:

- [ ] Create a reusable OpenTofu module following best practices
- [ ] Register and version modules in the Spacelift private registry
- [ ] Consume private modules from the Spacelift registry in stacks
- [ ] Update modules and manage version upgrades across consumers
- [ ] Implement proper module versioning using semantic versioning
- [ ] Configure module dependencies and stack relationships

### 1.2 Prerequisites

#### Required Access

1. Spacelift account with administrative access
2. GitHub account with repository creation permissions
3. AWS account with permissions to create S3 buckets

#### Required Tools

1. Git CLI installed and configured
2. OpenTofu CLI (version 1.6.0 or later)
3. Text editor or IDE (VS Code recommended)
4. AWS CLI configured with valid credentials

#### Verify Your Environment

Run these commands to verify your setup:

```bash
# Verify OpenTofu
tofu version
# Expected: OpenTofu v1.6.0 or higher

# Verify Git
git --version

# Verify AWS CLI
aws sts get-caller-identity

# Verify AWS permissions
aws s3 ls
```

> 💡 **Environment Check**: If any command fails, resolve the issue before proceeding. All prerequisites must be met for successful lab completion.

---

## Section 2: Understanding the Module Lifecycle

### 2.1 What is a Module?

A module is a container for multiple resources that are used together. Modules allow you to:

1. Encapsulate infrastructure patterns into reusable components
2. Standardize configurations across your organization
3. Reduce code duplication and maintenance overhead
4. Enforce best practices through pre-built configurations

### 2.2 The Module Workflow

This lab follows the complete module lifecycle:

| Phase | Action | Description |
|-------|--------|-------------|
| **1** | **Create** | Build the module with proper structure and code |
| **2** | **Publish** | Register in Spacelift and create versioned releases |
| **3** | **Consume** | Reference the module from stacks with version pinning |
| **4** | **Update** | Modify module, release new version, update consumers |

### 2.3 Spacelift Registry Benefits

The Spacelift private registry provides:

1. Private module hosting for your organization
2. Automatic version detection from Git tags
3. Integration with Spacelift stacks and policies
4. Module usage tracking and dependency management
5. Access control via Spacelift spaces and policies

---

## Section 3: Create Your First Module

### STEP 1: SET UP REPOSITORY STRUCTURE

#### 3.1 Create the Repository

First, create a new GitHub repository for your modules:

1. Go to GitHub and create a new repository
2. Name it: `infrastructure-modules`
3. Initialize with a README
4. Clone to your local machine

```bash
# Clone your repository
git clone https://github.com/YOUR-ORG/infrastructure-modules.git
cd infrastructure-modules

# Create the module directory structure
mkdir -p modules/s3-bucket
```

#### 3.2 Module Directory Structure

Create the following file structure for the S3 bucket module:

```
infrastructure-modules/
├── README.md
└── modules/
    └── s3-bucket/
        ├── README.md
        ├── versions.tf
        ├── variables.tf
        ├── main.tf
        └── outputs.tf
```

> 💡 **Best Practice**: Always organize modules in a dedicated `modules/` directory. This allows you to host multiple modules in a single repository while keeping clear separation.

---

### STEP 2: CREATE THE MODULE FILES

#### 3.3 versions.tf

Define the required OpenTofu and provider versions:

```hcl
# modules/s3-bucket/versions.tf
# Defines version constraints for OpenTofu and providers

terraform {
  required_version = ">= 1.6.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

> ⚠️ **Important**: Do NOT include a provider block in modules. Provider configuration should be done by the consuming stack, not the module itself.

#### 3.4 variables.tf

Define the input variables with descriptions, types, and validation:

```hcl
# modules/s3-bucket/variables.tf
# Input variables for the S3 bucket module

variable "bucket_name" {
  description = "The name of the S3 bucket. Must be globally unique."
  type        = string
  
  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9.-]{1,61}[a-z0-9]$", var.bucket_name))
    error_message = "Bucket name must be 3-63 characters, lowercase, and DNS-compliant."
  }
}

variable "environment" {
  description = "The deployment environment (dev, staging, prod)."
  type        = string
  default     = "dev"
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "enable_versioning" {
  description = "Enable versioning on the S3 bucket."
  type        = bool
  default     = true
}

variable "tags" {
  description = "Additional tags to apply to the bucket."
  type        = map(string)
  default     = {}
}
```

#### 3.5 main.tf

Implement the main resource configuration:

```hcl
# modules/s3-bucket/main.tf
# Main resource definitions for S3 bucket module

locals {
  common_tags = merge(
    {
      Environment = var.environment
      ManagedBy   = "OpenTofu"
      Module      = "s3-bucket"
    },
    var.tags
  )
}

# Primary S3 Bucket
resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name
  tags   = local.common_tags
}

# Bucket Versioning Configuration
resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  
  versioning_configuration {
    status = var.enable_versioning ? "Enabled" : "Disabled"
  }
}

# Block All Public Access (Security Best Practice)
resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Server-Side Encryption Configuration
resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
    bucket_key_enabled = true
  }
}
```

> 💡 **Security First**: This module implements security best practices by default: public access blocked, encryption enabled, and versioning available. These defaults protect your data from common misconfigurations.

#### 3.6 outputs.tf

Define the module outputs for consumers:

```hcl
# modules/s3-bucket/outputs.tf
# Output values exposed by the module

output "bucket_id" {
  description = "The name of the bucket."
  value       = aws_s3_bucket.this.id
}

output "bucket_arn" {
  description = "The ARN of the bucket."
  value       = aws_s3_bucket.this.arn
}

output "bucket_domain_name" {
  description = "The bucket domain name."
  value       = aws_s3_bucket.this.bucket_domain_name
}

output "bucket_regional_domain_name" {
  description = "The bucket region-specific domain name."
  value       = aws_s3_bucket.this.bucket_regional_domain_name
}
```

#### 3.7 Module README.md

Create documentation for module consumers:

```markdown
# S3 Bucket Module

Creates a secure S3 bucket with encryption and versioning.

## Usage

module "my_bucket" {
  source  = "spacelift.io/YOUR-ACCOUNT/s3-bucket/aws"
  version = "1.0.0"

  bucket_name = "my-application-data"
  environment = "prod"
}

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|----------|
| bucket_name | The S3 bucket name | string | n/a | yes |
| environment | Deployment environment | string | "dev" | no |
| enable_versioning | Enable bucket versioning | bool | true | no |
| tags | Additional tags | map(string) | {} | no |

## Outputs

| Name | Description |
|------|-------------|
| bucket_id | The bucket name |
| bucket_arn | The bucket ARN |
```

---

### STEP 3: COMMIT AND PUSH

#### 3.8 Validate and Commit

Validate the module and push to GitHub:

```bash
# Navigate to module directory
cd modules/s3-bucket

# Initialize and validate
tofu init
tofu validate

# Expected output:
# Success! The configuration is valid.

# Return to repository root and commit
cd ../..
git add .
git commit -m "Add s3-bucket module with security defaults"
git push origin main
```

#### 3.9 Verification Checklist

Before proceeding, verify:

- [ ] All files created in correct locations
- [ ] `tofu validate` passes without errors
- [ ] Code pushed to GitHub successfully
- [ ] Repository visible in GitHub

---

## Section 4: Register Module in Spacelift

### STEP 4: CONNECT REPOSITORY TO SPACELIFT

#### 4.1 Prerequisites Check

Before registering the module, ensure:

1. Your GitHub repository is connected to Spacelift
2. You have admin access to your Spacelift account
3. The module code is pushed to the main branch

#### 4.2 Option A: Register via Spacelift UI

Follow these steps to register the module through the web interface:

**Step 4.2.1: Navigate to Modules**

1. Log in to your Spacelift account
2. Click on "Modules" in the left navigation
3. Click the "Create Module" button

**Step 4.2.2: Configure Module Settings**

Fill in the module configuration:

| Field | Value |
|-------|-------|
| **Name** | s3-bucket |
| **Provider** | aws |
| **Repository** | infrastructure-modules |
| **Branch** | main |
| **Project Root** | modules/s3-bucket |

4. Click "Create Module" to save

#### 4.3 Option B: Register via OpenTofu (Admin Stack)

Alternatively, manage modules as code using an administrative stack:

```hcl
# admin/modules.tf
# Module registration via OpenTofu

resource "spacelift_module" "s3_bucket" {
  name        = "s3-bucket"
  description = "Secure S3 bucket with encryption and versioning"
  
  # Repository configuration
  repository   = "infrastructure-modules"
  branch       = "main"
  project_root = "modules/s3-bucket"
  
  # Provider for registry namespace
  terraform_provider = "aws"
  
  # Optional: Enable sharing across spaces
  administrative = false
  
  # Labels for organization
  labels = ["aws", "storage", "security"]
  
  # Optional: Protect from accidental deletion
  protect_from_deletion = true
}
```

> 💡 **Infrastructure as Code**: Managing module registrations via OpenTofu provides version control, audit trails, and automated management of your module registry.

---

### STEP 5: CREATE THE FIRST VERSION

#### 4.4 Understanding Version Tags

Spacelift detects module versions from Git tags. The tag format depends on your repository structure:

| Repository Type | Tag Format | Example |
|-----------------|------------|---------|
| Multi-module repo | `module-name/vX.Y.Z` | `s3-bucket/v1.0.0` |
| Single-module repo | `vX.Y.Z` | `v1.0.0` |

#### 4.5 Create Version 1.0.0

Create and push the first version tag:

```bash
# Ensure you're on main branch with latest changes
git checkout main
git pull origin main

# Create the version tag
# Format: module-name/vX.Y.Z for multi-module repos
git tag s3-bucket/v1.0.0

# Push the tag to GitHub
git push origin s3-bucket/v1.0.0

# Verify the tag was created
git tag -l "s3-bucket/*"
# Expected: s3-bucket/v1.0.0
```

#### 4.6 Verify in Spacelift

1. Navigate to Modules in Spacelift
2. Click on the "s3-bucket" module
3. Verify version 1.0.0 appears in the versions list
4. Note the module source address shown

Your module source address will be:

```
spacelift.io/YOUR-ACCOUNT/s3-bucket/aws
```

> ⚠️ **Version Not Appearing?** If the version doesn't appear within 2 minutes, check: (1) The tag format matches `module-name/vX.Y.Z`, (2) The repository is correctly connected, (3) The `project_root` path is correct.

---

## Section 5: Consume the Module in a Stack

### STEP 6: CREATE A CONSUMING STACK

#### 5.1 Repository Structure

Create a new repository or directory for your infrastructure deployment:

```
infrastructure-live/
├── README.md
└── stacks/
    └── app-storage/
        ├── versions.tf
        ├── variables.tf
        ├── main.tf
        ├── outputs.tf
        └── providers.tf
```

#### 5.2 Create the Stack Files

**versions.tf**

```hcl
# stacks/app-storage/versions.tf

terraform {
  required_version = ">= 1.6.0"
}
```

**providers.tf**

```hcl
# stacks/app-storage/providers.tf

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Project     = "app-storage"
      Environment = var.environment
      ManagedBy   = "OpenTofu-Spacelift"
    }
  }
}
```

**variables.tf**

```hcl
# stacks/app-storage/variables.tf

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"
}

variable "aws_region" {
  description = "AWS region for resources"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Name of the project"
  type        = string
  default     = "myapp"
}
```

**main.tf - Using the Private Module**

```hcl
# stacks/app-storage/main.tf

# Application Data Bucket
module "app_data_bucket" {
  # Source from Spacelift private registry
  source  = "spacelift.io/YOUR-ACCOUNT/s3-bucket/aws"
  version = "1.0.0"  # Always pin to specific version

  bucket_name       = "${var.project_name}-data-${var.environment}"
  environment       = var.environment
  enable_versioning = true
  
  tags = {
    Purpose = "Application Data"
  }
}

# Application Logs Bucket
module "app_logs_bucket" {
  source  = "spacelift.io/YOUR-ACCOUNT/s3-bucket/aws"
  version = "1.0.0"

  bucket_name       = "${var.project_name}-logs-${var.environment}"
  environment       = var.environment
  enable_versioning = false  # Logs don't need versioning
  
  tags = {
    Purpose = "Application Logs"
  }
}

# Backup Bucket
module "backup_bucket" {
  source  = "spacelift.io/YOUR-ACCOUNT/s3-bucket/aws"
  version = "1.0.0"

  bucket_name       = "${var.project_name}-backups-${var.environment}"
  environment       = var.environment
  enable_versioning = true
  
  tags = {
    Purpose   = "Backups"
    Retention = "30-days"
  }
}
```

> ⚠️ **Replace YOUR-ACCOUNT**: Replace `YOUR-ACCOUNT` with your actual Spacelift account name. You can find this in the module details page in Spacelift.

**outputs.tf**

```hcl
# stacks/app-storage/outputs.tf

output "data_bucket_id" {
  description = "ID of the application data bucket"
  value       = module.app_data_bucket.bucket_id
}

output "data_bucket_arn" {
  description = "ARN of the application data bucket"
  value       = module.app_data_bucket.bucket_arn
}

output "logs_bucket_id" {
  description = "ID of the logs bucket"
  value       = module.app_logs_bucket.bucket_id
}

output "backup_bucket_id" {
  description = "ID of the backup bucket"
  value       = module.backup_bucket.bucket_id
}

output "all_bucket_arns" {
  description = "ARNs of all created buckets"
  value = {
    data    = module.app_data_bucket.bucket_arn
    logs    = module.app_logs_bucket.bucket_arn
    backups = module.backup_bucket.bucket_arn
  }
}
```

---

### STEP 7: REGISTER THE STACK IN SPACELIFT

#### 5.3 Create Stack via UI

1. Navigate to Stacks in Spacelift
2. Click "Create Stack"
3. Configure the following settings:

| Setting | Value |
|---------|-------|
| Name | app-storage-dev |
| Repository | infrastructure-live |
| Branch | main |
| Project Root | stacks/app-storage |
| Manage State | Enabled |

#### 5.4 Configure Environment Variables

Add required environment variables to the stack:

1. Go to the stack settings
2. Navigate to Environment
3. Add the following variables:

```
TF_VAR_environment = "dev"
TF_VAR_aws_region = "us-east-1"
TF_VAR_project_name = "myapp"
```

#### 5.5 Configure AWS Integration

Attach your AWS cloud integration to the stack:

1. Go to stack settings
2. Navigate to "Cloud Integrations"
3. Attach your AWS integration
4. Ensure the IAM role has S3 permissions

---

### STEP 8: TRIGGER AND VERIFY DEPLOYMENT

#### 5.6 Trigger the Stack

1. Navigate to your stack in Spacelift
2. Click "Trigger" to start a run
3. Review the plan output
4. Confirm and apply the changes

#### 5.7 Expected Plan Output

You should see resources being created:

```
# Example plan output
Plan: 12 to add, 0 to change, 0 to destroy.

  # module.app_data_bucket.aws_s3_bucket.this will be created
  + resource "aws_s3_bucket" "this" {
      + bucket = "myapp-data-dev"
      ...
    }
    
  # module.app_data_bucket.aws_s3_bucket_versioning.this will be created
  # module.app_data_bucket.aws_s3_bucket_public_access_block.this will be created
  # module.app_data_bucket.aws_s3_bucket_server_side_encryption_configuration.this will be created
  
  # (similar resources for logs and backup buckets)
```

#### 5.8 Verify Deployment

After successful apply:

1. Check the Spacelift run output for success
2. Verify outputs are displayed
3. Confirm buckets exist in AWS console

```bash
# Verify via AWS CLI
aws s3 ls | grep myapp

# Expected output:
# 2024-01-15 10:30:00 myapp-data-dev
# 2024-01-15 10:30:00 myapp-logs-dev
# 2024-01-15 10:30:00 myapp-backups-dev
```

---

## Section 6: Update and Version the Module

### STEP 9: ENHANCE THE MODULE

#### 6.1 Add New Feature

Let's add lifecycle rules to the module for automatic object expiration:

**Add to variables.tf:**

```hcl
# Add to modules/s3-bucket/variables.tf

variable "lifecycle_rules" {
  description = "Lifecycle rules for the bucket"
  type = list(object({
    id                         = string
    enabled                    = bool
    prefix                     = optional(string, "")
    expiration_days            = optional(number, null)
    noncurrent_expiration_days = optional(number, null)
  }))
  default = []
}
```

**Add to main.tf:**

```hcl
# Add to modules/s3-bucket/main.tf

# Lifecycle Configuration (only if rules are defined)
resource "aws_s3_bucket_lifecycle_configuration" "this" {
  count  = length(var.lifecycle_rules) > 0 ? 1 : 0
  bucket = aws_s3_bucket.this.id

  dynamic "rule" {
    for_each = var.lifecycle_rules
    content {
      id     = rule.value.id
      status = rule.value.enabled ? "Enabled" : "Disabled"

      filter {
        prefix = rule.value.prefix
      }

      dynamic "expiration" {
        for_each = rule.value.expiration_days != null ? [1] : []
        content {
          days = rule.value.expiration_days
        }
      }

      dynamic "noncurrent_version_expiration" {
        for_each = rule.value.noncurrent_expiration_days != null ? [1] : []
        content {
          noncurrent_days = rule.value.noncurrent_expiration_days
        }
      }
    }
  }
}
```

---

### STEP 10: RELEASE NEW VERSION

#### 6.2 Commit and Tag

Commit the changes and create a new version:

```bash
# Commit changes
cd infrastructure-modules
git add .
git commit -m "Add lifecycle rules support to s3-bucket module"
git push origin main

# Create new version tag (minor version bump)
git tag s3-bucket/v1.1.0
git push origin s3-bucket/v1.1.0

# Verify tags
git tag -l "s3-bucket/*"
# Expected:
# s3-bucket/v1.0.0
# s3-bucket/v1.1.0
```

#### 6.3 Semantic Versioning Guide

Follow semantic versioning for module releases:

| Version | When to Use | Example Changes |
|---------|-------------|-----------------|
| **MAJOR** | Breaking changes | Removed variables, renamed outputs |
| **MINOR** | New features (backward compatible) | New optional variables, new outputs |
| **PATCH** | Bug fixes only | Fixed validation, corrected defaults |

> 💡 **Version Strategy**: Adding new optional variables with defaults is a MINOR version change (v1.0.0 → v1.1.0) because existing consumers continue to work without modification.

---

### STEP 11: UPDATE CONSUMERS

#### 6.4 Update Stack to Use New Version

Update the consuming stack to use the new module version:

```hcl
# stacks/app-storage/main.tf (updated)

module "app_logs_bucket" {
  source  = "spacelift.io/YOUR-ACCOUNT/s3-bucket/aws"
  version = "1.1.0"  # Updated from 1.0.0

  bucket_name       = "${var.project_name}-logs-${var.environment}"
  environment       = var.environment
  enable_versioning = false
  
  # NEW: Use lifecycle rules to expire old logs
  lifecycle_rules = [
    {
      id              = "expire-old-logs"
      enabled         = true
      prefix          = ""
      expiration_days = 90
    }
  ]
  
  tags = {
    Purpose = "Application Logs"
  }
}
```

#### 6.5 Deploy the Update

```bash
# Commit and push changes
cd infrastructure-live
git add .
git commit -m "Update s3-bucket module to v1.1.0, add log retention"
git push origin main

# Spacelift will automatically detect changes and trigger a run
# (if autodeploy is enabled)
```

#### 6.6 Version Constraints

You can use version constraints for flexibility:

```hcl
# Exact version (recommended for production)
version = "1.1.0"

# Pessimistic constraint (allows patch updates)
version = "~> 1.1.0"  # Allows 1.1.x but not 1.2.0

# Range constraint (use with caution)
version = ">= 1.1.0, < 2.0.0"
```

> ⚠️ **Production Best Practice**: Always use exact version pinning in production environments. This prevents unexpected changes when new module versions are released.

---

## Section 7: Verification and Testing

### 7.1 Module Testing Checklist

Before releasing a new module version, verify:

- [ ] `tofu validate` passes without errors
- [ ] `tofu fmt` shows no formatting issues
- [ ] All variables have descriptions and types
- [ ] All outputs have descriptions
- [ ] README.md is updated with new features
- [ ] CHANGELOG.md documents changes

### 7.2 Local Testing

Test the module locally before pushing:

```bash
# Create a test directory
mkdir -p test/s3-bucket
cd test/s3-bucket

# Create test configuration
cat > main.tf << 'EOF'
provider "aws" {
  region = "us-east-1"
}

module "test_bucket" {
  source = "../../modules/s3-bucket"
  
  bucket_name = "test-bucket-${random_id.suffix.hex}"
  environment = "test"
  
  lifecycle_rules = [
    {
      id              = "test-rule"
      enabled         = true
      expiration_days = 30
    }
  ]
}

resource "random_id" "suffix" {
  byte_length = 4
}

output "bucket_id" {
  value = module.test_bucket.bucket_id
}
EOF

# Initialize and plan
tofu init
tofu plan

# Apply (optional - creates real resources)
tofu apply -auto-approve

# Cleanup
tofu destroy -auto-approve
```

### 7.3 Spacelift Module Tests

Configure automated tests in Spacelift:

```hcl
# In module settings, add test cases
resource "spacelift_module" "s3_bucket" {
  name = "s3-bucket"
  # ... other config ...
  
  # Enable test runs on PR
  workflow_tool = "OPEN_TOFU"
}

# Create a test stack that uses the module
resource "spacelift_stack" "module_test" {
  name         = "s3-bucket-test"
  repository   = "infrastructure-modules"
  branch       = "main"
  project_root = "test/s3-bucket"
  
  labels = ["test", "module-validation"]
}
```

---

## Section 8: Troubleshooting Guide

### 8.1 Module Not Found

**Symptom:** `Error: Module not found` when running `tofu init`

```
Error: Failed to query available provider packages

Could not retrieve the list of available versions for module
"spacelift.io/your-account/s3-bucket/aws"
```

**Solutions:**

1. Verify the module exists in Spacelift Modules section
2. Check the module source path is correct
3. Ensure at least one version is published
4. Verify you have access to the module's space

### 8.2 Version Not Found

**Symptom:** Specified version doesn't exist

```
Error: No available version matches the given constraints
```

**Solutions:**

1. Check available versions in Spacelift module details
2. Verify the tag format matches: `module-name/vX.Y.Z`
3. Ensure the tag was pushed to GitHub
4. Wait 2-3 minutes for Spacelift to detect new tags

### 8.3 Authentication Errors

**Symptom:** 401 Unauthorized when fetching module

**Solutions:**

1. Verify the stack has proper Spacelift context
2. Check the stack is in the correct space
3. Ensure module sharing is configured if cross-space

### 8.4 Tag Not Detected

**Symptom:** Pushed tag but version not appearing in Spacelift

**Diagnostic Steps:**

```bash
# Verify tag exists in GitHub
git ls-remote --tags origin | grep s3-bucket

# Check tag format (must be exact)
# Correct: s3-bucket/v1.0.0
# Wrong:   s3-bucket-v1.0.0
# Wrong:   S3-Bucket/v1.0.0
# Wrong:   v1.0.0 (for multi-module repos)
```

**Solutions:**

1. Delete and recreate with correct format
2. Verify the tag points to the correct commit
3. Check webhook delivery in GitHub settings

```bash
# Delete incorrect tag
git tag -d wrong-tag
git push origin --delete wrong-tag

# Create correct tag
git tag s3-bucket/v1.0.0
git push origin s3-bucket/v1.0.0
```

### 8.5 Plan Shows No Changes

**Symptom:** Updated module version but plan shows no changes

**Solutions:**

1. Run `tofu init -upgrade` to fetch new version
2. Clear `.terraform` directory and reinitialize
3. Verify version is actually changed in source

```bash
# Force module update
tofu init -upgrade

# Or clean and reinitialize
rm -rf .terraform .terraform.lock.hcl
tofu init
```

---

## Section 9: Best Practices and Next Steps

### 9.1 Module Design Best Practices

#### Structure

1. Use consistent file naming: `versions.tf`, `variables.tf`, `main.tf`, `outputs.tf`
2. Keep modules focused on a single responsibility
3. Provide sensible defaults for optional variables
4. Include comprehensive documentation in README

#### Security

1. Enable encryption by default where applicable
2. Block public access unless explicitly required
3. Use least-privilege IAM permissions
4. Validate inputs to prevent misconfigurations

#### Versioning

1. Always use semantic versioning
2. Document breaking changes in CHANGELOG
3. Test thoroughly before releasing major versions
4. Consider deprecation warnings before removing features

### 9.2 Consumer Best Practices

1. Pin to exact versions in production
2. Use version ranges only in development
3. Review module changes before upgrading
4. Test upgrades in non-production first

### 9.3 Recommended Next Steps

Continue your learning with these advanced topics:

- [ ] Create a VPC module with networking best practices
- [ ] Implement module composition (modules calling modules)
- [ ] Add OPA policies to validate module usage
- [ ] Set up automated testing with Spacelift test stacks
- [ ] Configure module dependencies and triggers
- [ ] Implement cross-space module sharing

> 💡 **Congratulations!** You have completed the OpenTofu Module Workflow lab. You now know how to create, publish, consume, and update modules using the Spacelift private registry.

---

## Quick Reference Card

### Essential Commands

```bash
# Module Development
tofu init                    # Initialize module
tofu validate                # Validate configuration
tofu fmt -recursive          # Format all files

# Version Management
git tag module-name/v1.0.0   # Create version tag
git push origin --tags       # Push all tags
git tag -l "module-name/*"   # List module versions

# Consumer Updates
tofu init -upgrade           # Fetch latest module version
tofu plan                    # Preview changes
tofu apply                   # Apply changes
```

### Module Source Formats

```hcl
# Spacelift Private Registry
source  = "spacelift.io/ACCOUNT/MODULE/PROVIDER"
version = "1.0.0"

# GitHub (direct)
source = "github.com/org/repo//path/to/module?ref=v1.0.0"

# Local (development only)
source = "../../modules/s3-bucket"
```

### Key URLs

| Resource | URL |
|----------|-----|
| Spacelift Docs | https://docs.spacelift.io |
| OpenTofu Docs | https://opentofu.org/docs |
| AWS Provider | https://registry.terraform.io/providers/hashicorp/aws |

---

## License

This training material is provided for educational purposes.

## Contributing

Feel free to submit issues and enhancement requests.
