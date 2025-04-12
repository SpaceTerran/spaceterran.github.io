---
title: Setting Up a Secure Terraform State Backend in AWS
author: isaac
date: 2025-04-11 07:00:00 -0700
categories: [DevOps, Terraform]
tags: [terraform, aws, infrastructure, s3, security]
render_with_liquid: false
toc: true
comments: true
image:
  path: /assets/img/myphotos/tf-backend/tf-backend.png
  alt: 
---

## Introduction

Welcome to this guide where I walk you through automating the backend infrastructure necessary for storing your Terraform state files in AWS. This setup costs just a few cents a month! I'll demonstrate how to establish a secure Terraform state backend using AWS S3 for both storage and state locking. While this setup might seem to diverge from traditional homelabbing principles, it's akin to my choice of using GitHub over hosting Gitea or GitLab locally—having a reliable place to secure these files is crucial. What are your thoughts on this approach?

## Why Remote State Storage Matters

Storing Terraform state remotely offers several benefits:

- **Team Collaboration**: Allows multiple team members to work on the same infrastructure—though in a homelab, this might not be a priority 😉
- **State Locking**: Protects your state from concurrent modifications that could lead to corruption
- **Backup and Versioning**: Secures your infrastructure from accidental state loss
- **Secrets Management**: Keeps sensitive data out of local files and version control

## Project Structure

This project consists of Terraform configurations aimed at creating the necessary AWS resources for effective state management:

- S3 bucket for state storage
- IAM user with the necessary permissions
- Security configurations for the S3 bucket

## Prerequisites

- AWS account
- Terraform installed on your device

## Implementation

### Step 1: Define Variables

Start by defining the variables in the `variables.tf` file for our configuration:

**File: variables.tf**

```terraform
variable "aws_region" {
  description = "AWS region for resources"
  type        = string
  default     = "us-east-1"
}

variable "aws_access_key" {
  description = "AWS access key"
  type        = string
  sensitive   = true
}

variable "aws_secret_key" {
  description = "AWS secret key"
  type        = string
  sensitive   = true
}

variable "bucket_name" {
  description = "Name of the S3 bucket for Terraform state"
  type        = string
}

variable "terraform_iam_user" {
  description = "Name of the IAM user for Terraform"
  type        = string
  default     = "terraform-backend-user"
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default = {
    ManagedBy   = "Terraform"
    Environment = "Management"
    Purpose     = "Terraform State"
  }
}
```

### Step 2: Configure the AWS Provider

**File: provider.tf**

```terraform
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  required_version = ">= 1.0.0"
}

provider "aws" {
  region     = var.aws_region
  access_key = var.aws_access_key
  secret_key = var.aws_secret_key
}
```

### Step 3: Create the S3 Bucket for State Storage

**File: s3_bucket.tf**

```terraform
# S3 bucket for Terraform state
resource "aws_s3_bucket" "terraform_state" {
  bucket = var.bucket_name

  # Prevent accidental deletion of this critical infrastructure component
  lifecycle {
    prevent_destroy = true
  }
  tags = var.tags
}

# Enable versioning to keep a history of state files and prevent data loss
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Enable server-side encryption for security of state files at rest
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Block all public access to the bucket for security
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket                  = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### Step 4: Create IAM User and Permissions

**File: iam.tf**

```terraform
# Create an IAM user specifically for Terraform operations
resource "aws_iam_user" "terraform" {
  name = var.terraform_iam_user

  tags = var.tags
}

# Create an IAM policy for Terraform state management
resource "aws_iam_policy" "terraform_state" {
  name        = "TerraformStateAccess"
  description = "Policy allowing access to Terraform state bucket with S3 locking"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = "s3:ListBucket"
        Resource = aws_s3_bucket.terraform_state.arn
      },
      {
        Effect = "Allow"
        Action = ["s3:GetObject", "s3:PutObject"]
        Resource = "${aws_s3_bucket.terraform_state.arn}/*/terraform.tfstate"
      },
      {
        Effect = "Allow"
        Action = ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"]
        Resource = "${aws_s3_bucket.terraform_state.arn}/*/terraform.tfstate.tflock"
      }
    ]
  })
}

# Attach the policy to the IAM user
resource "aws_iam_user_policy_attachment" "terraform_state" {
  user       = aws_iam_user.terraform.name
  policy_arn = aws_iam_policy.terraform_state.arn
}

# Create access keys for the IAM user
resource "aws_iam_access_key" "terraform" {
  user = aws_iam_user.terraform.name
}
```

### Step 5: Define Outputs

**File: outputs.tf**

```terraform
# Output the backend configuration to use in other Terraform projects
output "backend_configuration" {
  description = "Backend configuration for Terraform projects"
  value = <<-EOT
    # Add this to your terraform block in terraform.tf:

    backend "s3" {
      bucket         = "${aws_s3_bucket.terraform_state.bucket}"
      key            = "path/to/your/terraform.tfstate"  # Customize this path
      region         = "${var.aws_region}"
      encrypt        = true
      use_lockfile   = true
    }
  EOT
}

# Output the bucket name for reference
output "bucket_name" {
  description = "The name of the S3 bucket for Terraform state"
  value       = aws_s3_bucket.terraform_state.bucket
}

# Output the bucket ARN for reference
output "bucket_arn" {
  description = "The ARN of the S3 bucket for Terraform state"
  value       = aws_s3_bucket.terraform_state.arn
}

# Output the IAM user name for reference
output "terraform_user_name" {
  description = "The name of the IAM user for Terraform"
  value       = aws_iam_user.terraform.name
}

# Output the access key ID for the created IAM user
output "terraform_access_key_id" {
  description = "The access key ID for the Terraform IAM user"
  value       = aws_iam_access_key.terraform.id
}

# Output the secret access key (marked as sensitive)
output "terraform_secret_access_key" {
  description = "The secret access key for the Terraform IAM user"
  value       = aws_iam_access_key.terraform.secret
  sensitive   = true
}
```

### Screenshot of the Folder Structure

![Folder Structure](/assets/img/myphotos/tf-backend/FolderSructure.png){: width="400" .center }

## Handling Sensitive Information

### Initial Setup with Root Credentials

For the initial setup, you'll need credentials with sufficient permissions to create S3 buckets and IAM users. While this isn't best practice for regular use, you may need to use your AWS root or an admin IAM user for this one-time setup.

**Step 1: Create access keys for initial setup**

1. Log into your AWS Management Console.
2. Navigate to IAM → Users → Your Username → Security credentials.
3. Click "Create access key."

   ![Create key](/assets/img/myphotos/tf-backend/create_key.png){: width="600" .center }
4. Acknowledge the security recommendations and create the key.

   ![Understand risk](/assets/img/myphotos/tf-backend/understand_risk.png){: width="600" .center }
5. **Important**: This is the only time AWS will show you the secret key, so save it securely.

   ![Key](/assets/img/myphotos/tf-backend/key.png){: width="600" .center }

**Step 2: Store credentials securely**

Create a file named `terraform.tfvars` and add it to your `.gitignore`:

**File: terraform.tfvars (DO NOT COMMIT THIS FILE)**

```terraform
aws_access_key = "YOUR_ROOT_OR_ADMIN_ACCESS_KEY"
aws_secret_key = "YOUR_ROOT_OR_ADMIN_SECRET_KEY"
bucket_name    = "your-unique-bucket-name"  # Must be globally unique, all lowercase
```

**Example Edited File**  
![TFVars file](/assets/img/myphotos/tf-backend/tfvars-file.png){: width="600" .center }

**Step 3: Apply the Terraform configuration**

```bash
terraform init
terraform apply -var-file="terraform.tfvars"
```

**Example Output**  
![Apply](/assets/img/myphotos/tf-backend/TF-apply.png){: width="400" .normal }

**Step 4: Retrieve the new IAM user credentials**

After applying the configuration, retrieve the access keys generated for your IAM user. Since the secret access key is marked as sensitive, it won't be displayed in regular output. Here's how to get it:

```bash
# Get the access key ID
terraform output terraform_access_key_id

# Get the secret access key using the -raw flag
terraform output -raw terraform_secret_access_key
```

The `-raw` flag is essential; it outputs just the value without formatting, perfect for scripting or setting environment variables. Store these credentials securely to configure other Terraform projects with this backend.

**Example Output**  
![Key for user](/assets/img/myphotos/tf-backend/KeyforUser.png){: width="800" .center }

**Step 5: Securely store the new credentials**

Store these credentials securely in a password manager or secure vault for all future Terraform operations.

**Step 6: Delete or disable the initial admin access keys**

Once the IAM user is confirmed functional, delete or disable those initial access keys, especially if they were root account keys.

> **IMPORTANT**:
> - Never commit credentials to version control
> - Use root credentials only for this initial setup
> - Delete or disable the root/admin access keys after setup
> - Use the generated IAM user credentials for all future Terraform operations

For future Terraform projects using this backend, configure them with the IAM user credentials generated here, not root credentials.

## Using the Backend in Other Projects

Once the backend is established, incorporate it into your other Terraform projects by adding the following configuration to your `terraform` block:

```terraform
terraform {
  backend "s3" {
    bucket         = "your-terraform-state-bucket"
    key            = "path/to/your/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    use_lockfile   = true
  }
}
```

Note that `use_lockfile = true` enables S3 state locking, using a lock file in S3 rather than the older DynamoDB-based method.

## Security Considerations

This setup implements several security best practices:

- Server-side encryption for the S3 bucket
- Public access is blocked for the S3 bucket
- Versioning is enabled to avert accidental state loss
- Fine-grained IAM permissions specific to state file access
- Sensitive variables are marked to avoid exposure
- S3 state locking prevents concurrent modifications
- IAM permissions are precisely configured for lock files (*.tflock)

## Conclusion

We're now accomplished in setting up a Terraform state backend—a pivotal task for infrastructure-as-code projects.

Here's a brief recap of what we've achieved:

- Securely stored state in an S3 bucket
- Granted IAM permissions with the principle of least privilege
- Formed a reusable backend configuration for future projects
- Implemented S3-based state locking for concurrent operations

Before you go, share your insights:

1. What challenges have you encountered when setting up Terraform backends, and how did you overcome them?
2. Are there other tools or methods you've used to secure the Terraform state that you'd recommend?

I'm excited to hear from you in the comments!
