---
name: iac-data-engineering-terraform
description: Infrastructure-as-Code fundamentals for data engineering using Terraform to provision AWS resources like S3 and EC2
triggers:
  - "set up terraform for data engineering"
  - "create AWS infrastructure with terraform"
  - "provision S3 and EC2 with IaC"
  - "terraform state management for data pipelines"
  - "infrastructure as code for data engineering"
  - "manage AWS resources with terraform"
  - "terraform workflow for data infrastructure"
  - "destroy terraform data infrastructure"
---

# IaC for Data Engineering with Terraform

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project teaches Infrastructure-as-Code (IaC) fundamentals for data engineers using Terraform to provision and manage AWS resources. It demonstrates how to create, manage, and destroy data infrastructure including S3 buckets and EC2 instances using declarative configuration.

## What This Project Does

- Provisions AWS infrastructure for data engineering workloads using Terraform
- Creates S3 buckets for data storage
- Launches EC2 instances for data processing
- Manages infrastructure state with Terraform state files
- Demonstrates IaC best practices for data engineering

## Prerequisites

Before using this project, ensure you have:

1. AWS Account with root access
2. Terraform installed: [Installation Guide](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)
3. AWS CLI installed: [Installation Guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
4. AWS CLI configured with credentials: `aws configure`

## AWS IAM Setup

### Create IAM User and Permissions

You need an IAM user with appropriate permissions to provision resources:

1. **Create IAM User**: Navigate to AWS Console → IAM → Users → Create User
2. **Attach Inline Policy**: Select user → Permissions → Add inline policy
3. **Grant Permissions**: Add Full S3, EC2, and IAM access

**Example IAM Policy (for development only)**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "ec2:*",
        "iam:*"
      ],
      "Resource": "*"
    }
  ]
}
```

> **Warning**: This broad permission set is for learning purposes only. Use least-privilege policies in production.

### Configure AWS CLI

```bash
# Configure with your IAM user credentials
aws configure
# Enter: AWS Access Key ID, Secret Access Key, Default region, Output format
```

## Project Structure

```
terraform/
├── main.tf           # Main Terraform configuration
├── variables.tf      # Variable definitions (if present)
├── outputs.tf        # Output values (if present)
└── terraform.tfstate # State file (created after apply)
```

## Terraform Configuration

### Basic main.tf Example

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# S3 Bucket for data storage
resource "aws_s3_bucket" "data_bucket" {
  bucket = "your-unique-bucket-name-12345"  # MUST be globally unique
  
  tags = {
    Name        = "Data Engineering Bucket"
    Environment = "Dev"
    Project     = "IaC Tutorial"
  }
}

# S3 Bucket versioning
resource "aws_s3_bucket_versioning" "data_bucket_versioning" {
  bucket = aws_s3_bucket.data_bucket.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# EC2 Instance for data processing
resource "aws_instance" "data_processor" {
  ami           = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2 AMI
  instance_type = "t2.micro"
  
  tags = {
    Name        = "Data Processor"
    Environment = "Dev"
    Project     = "IaC Tutorial"
  }
}

# Output values
output "bucket_name" {
  value       = aws_s3_bucket.data_bucket.id
  description = "Name of the S3 bucket"
}

output "instance_id" {
  value       = aws_instance.data_processor.id
  description = "ID of the EC2 instance"
}

output "instance_public_ip" {
  value       = aws_instance.data_processor.public_ip
  description = "Public IP of the EC2 instance"
}
```

### Variables Example (variables.tf)

```hcl
variable "aws_region" {
  description = "AWS region for resources"
  type        = string
  default     = "us-east-1"
}

variable "bucket_name" {
  description = "Name of the S3 bucket"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}
```

## Terraform Workflow

### 1. Initialize Terraform

```bash
# Initialize Terraform in the terraform directory
terraform -chdir=terraform init

# Output shows:
# - Provider downloads
# - Backend initialization
# - Module initialization
```

### 2. Validate Configuration

```bash
# Validate syntax and configuration
terraform -chdir=terraform validate

# Expected output: "Success! The configuration is valid."
```

### 3. Format Configuration

```bash
# Format HCL files to canonical style
terraform -chdir=terraform fmt

# Returns names of files that were formatted
```

### 4. Plan Infrastructure Changes

```bash
# Preview changes before applying
terraform -chdir=terraform plan

# Shows:
# - Resources to be created (+)
# - Resources to be modified (~)
# - Resources to be destroyed (-)
```

### 5. Apply Infrastructure

```bash
# Create/update infrastructure
terraform -chdir=terraform apply

# Review plan and type 'yes' to confirm
# Or auto-approve:
terraform -chdir=terraform apply -auto-approve
```

### 6. Verify Resources

```bash
# List S3 buckets
aws s3 ls

# Describe running EC2 instances
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].{ID:InstanceId, Name:Tags[?Key==`Name`].Value, Type:InstanceType, State:State.Name, PublicIP:PublicIpAddress, PrivateIP:PrivateIpAddress}' \
  --output table
```

## State Management

### View State Resources

```bash
# List all resources in state
terraform -chdir=terraform state list

# View state as JSON
cat terraform/terraform.tfstate | jq -r '.resources[] | [.type, .name] | join(",")'

# Show specific resource details
terraform -chdir=terraform state show aws_s3_bucket.data_bucket
```

### State File Commands

```bash
# Pull current state
terraform -chdir=terraform state pull

# Refresh state from actual infrastructure
terraform -chdir=terraform refresh

# Remove resource from state (doesn't destroy resource)
terraform -chdir=terraform state rm aws_instance.data_processor
```

## Common Patterns for Data Engineering

### S3 Bucket with Lifecycle Policy

```hcl
resource "aws_s3_bucket" "data_lake" {
  bucket = "my-data-lake-bucket-${var.environment}"
}

resource "aws_s3_bucket_lifecycle_configuration" "data_lake_lifecycle" {
  bucket = aws_s3_bucket.data_lake.id

  rule {
    id     = "archive-old-data"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    expiration {
      days = 365
    }
  }
}
```

### EC2 with User Data for Data Tools

```hcl
resource "aws_instance" "data_processor" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y python3 python3-pip
              pip3 install pandas numpy boto3
              EOF
  
  tags = {
    Name = "Data Processor ${var.environment}"
  }
}
```

### Multiple Environments with Workspaces

```bash
# Create new workspace for staging
terraform -chdir=terraform workspace new staging

# List workspaces
terraform -chdir=terraform workspace list

# Switch workspace
terraform -chdir=terraform workspace select production

# Show current workspace
terraform -chdir=terraform workspace show
```

### Using Terraform Variables File

Create `terraform.tfvars`:

```hcl
aws_region    = "us-west-2"
bucket_name   = "my-unique-data-bucket-2024"
instance_type = "t3.medium"
environment   = "production"
```

Apply with variables:

```bash
terraform -chdir=terraform apply -var-file="terraform.tfvars"
```

## Destroy Infrastructure

### Destroy All Resources

```bash
# Destroy all managed infrastructure
terraform -chdir=terraform destroy

# Review destruction plan and type 'yes' to confirm
# Or auto-approve:
terraform -chdir=terraform destroy -auto-approve
```

### Destroy Specific Resources

```bash
# Destroy only the EC2 instance
terraform -chdir=terraform destroy -target=aws_instance.data_processor

# Destroy multiple specific resources
terraform -chdir=terraform destroy \
  -target=aws_instance.data_processor \
  -target=aws_s3_bucket.data_bucket
```

## Troubleshooting

### Bucket Name Already Exists

**Error**: `BucketAlreadyExists: The requested bucket name is not available`

**Solution**: S3 bucket names must be globally unique. Change the bucket name in `main.tf`:

```hcl
resource "aws_s3_bucket" "data_bucket" {
  bucket = "your-unique-name-${random_id.bucket_suffix.hex}"
}

resource "random_id" "bucket_suffix" {
  byte_length = 4
}
```

### AWS Credentials Not Found

**Error**: `NoCredentialProviders: no valid providers in chain`

**Solution**: Configure AWS CLI or set environment variables:

```bash
# Option 1: Configure AWS CLI
aws configure

# Option 2: Set environment variables
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="us-east-1"
```

### State Lock Errors

**Error**: `Error acquiring the state lock`

**Solution**: If using remote state with DynamoDB locking:

```bash
# Force unlock (use with caution)
terraform -chdir=terraform force-unlock <LOCK_ID>
```

### Permission Denied Errors

**Error**: `UnauthorizedOperation` or `AccessDenied`

**Solution**: Verify IAM permissions for the user/role:

```bash
# Check current identity
aws sts get-caller-identity

# Verify policies attached to user
aws iam list-user-policies --user-name <your-username>
aws iam list-attached-user-policies --user-name <your-username>
```

### Resource Already Exists

**Error**: Resource already exists in AWS but not in state

**Solution**: Import existing resource:

```bash
# Import S3 bucket
terraform -chdir=terraform import aws_s3_bucket.data_bucket existing-bucket-name

# Import EC2 instance
terraform -chdir=terraform import aws_instance.data_processor i-1234567890abcdef0
```

## Best Practices

1. **Always run `terraform plan`** before `apply` to review changes
2. **Use version control** for `.tf` files (exclude `terraform.tfstate`)
3. **Use remote state** for team collaboration (S3 + DynamoDB)
4. **Tag all resources** for cost tracking and organization
5. **Use variables** instead of hardcoded values
6. **Implement least-privilege IAM** policies in production
7. **Enable state locking** to prevent concurrent modifications
8. **Use modules** for reusable infrastructure components

## Remote State Configuration

For production use, store state remotely:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "data-infrastructure/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

Initialize with remote backend:

```bash
terraform -chdir=terraform init -backend-config="bucket=${TERRAFORM_STATE_BUCKET}"
```
