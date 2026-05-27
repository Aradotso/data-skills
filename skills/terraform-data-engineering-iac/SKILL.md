---
name: terraform-data-engineering-iac
description: Infrastructure-as-Code templates for data engineering on AWS using Terraform
triggers:
  - set up data engineering infrastructure with terraform
  - deploy aws resources for data pipelines
  - create s3 and ec2 infrastructure as code
  - terraform for data engineering projects
  - manage data infrastructure with iac
  - terraform aws data engineering setup
  - infrastructure as code for data workflows
  - terraform state management for data projects
---

# Terraform Data Engineering IaC Skill

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This skill covers using Terraform to provision and manage AWS infrastructure for data engineering workloads, including S3 buckets, EC2 instances, and IAM policies.

## What This Project Does

This project provides Infrastructure-as-Code (IaC) templates using Terraform to set up AWS resources commonly needed for data engineering pipelines. It demonstrates best practices for:

- Provisioning S3 storage for data lakes
- Creating EC2 compute instances for data processing
- Managing IAM permissions
- State management and infrastructure lifecycle

## Prerequisites

Before using this project, ensure you have:

1. AWS Account with appropriate access
2. Terraform installed (`>= 1.0`)
3. AWS CLI installed and configured
4. IAM user with S3, EC2, and IAM full access permissions

## Installation

```bash
# Clone the repository
git clone https://github.com/josephmachado/iac-for-data-engineering-terraform-.git
cd iac-for-data-engineering-terraform-

# Install Terraform (if not already installed)
# macOS
brew install terraform

# Linux
wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip
unzip terraform_1.6.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# Verify installation
terraform --version
```

## AWS Setup

### Configure AWS CLI

```bash
# Configure AWS credentials
aws configure

# Verify configuration
aws sts get-caller-identity
```

### IAM Permissions Required

Your IAM user needs the following permissions:
- `AmazonS3FullAccess`
- `AmazonEC2FullAccess`
- `IAMFullAccess`

> **Warning**: Full access policies are for development/learning only. Use least-privilege policies in production.

## Project Structure

```
terraform/
├── main.tf           # Main configuration file
├── variables.tf      # Input variables (if present)
├── outputs.tf        # Output values (if present)
└── terraform.tfstate # State file (generated)
```

## Key Terraform Commands

### Initialize Terraform

```bash
# Initialize the Terraform working directory
terraform -chdir=terraform init
```

This downloads required providers and sets up the backend.

### Validate Configuration

```bash
# Check syntax and validate configuration
terraform -chdir=terraform validate
```

### Format Code

```bash
# Format Terraform files to canonical style
terraform -chdir=terraform fmt
```

### Plan Infrastructure Changes

```bash
# Preview changes without applying
terraform -chdir=terraform plan
```

### Apply Infrastructure

```bash
# Create or update infrastructure
terraform -chdir=terraform apply

# Auto-approve without confirmation (use with caution)
terraform -chdir=terraform apply -auto-approve
```

### Destroy Infrastructure

```bash
# Remove all managed infrastructure
terraform -chdir=terraform destroy

# Auto-approve destruction
terraform -chdir=terraform destroy -auto-approve
```

### State Management

```bash
# List all resources in state
terraform -chdir=terraform state list

# Show details of a specific resource
terraform -chdir=terraform state show aws_s3_bucket.data_bucket

# View state file (formatted)
cat terraform/terraform.tfstate | jq -r '.resources[] | [.type, .name] | join(",")'
```

## Configuration

### Main Terraform Configuration Example

```hcl
# terraform/main.tf
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

# S3 bucket for data lake
resource "aws_s3_bucket" "data_lake" {
  bucket = "my-unique-data-engineering-bucket-${data.aws_caller_identity.current.account_id}"
  
  tags = {
    Name        = "Data Lake Bucket"
    Environment = "Development"
    ManagedBy   = "Terraform"
  }
}

# Enable versioning for data lake bucket
resource "aws_s3_bucket_versioning" "data_lake_versioning" {
  bucket = aws_s3_bucket.data_lake.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Block public access to data lake
resource "aws_s3_bucket_public_access_block" "data_lake_public_access" {
  bucket = aws_s3_bucket.data_lake.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# EC2 instance for data processing
resource "aws_instance" "data_processor" {
  ami           = "ami-0c55b159cbfafe1f0" # Amazon Linux 2 (update to current AMI)
  instance_type = "t2.micro"

  tags = {
    Name        = "Data Processor"
    Environment = "Development"
    ManagedBy   = "Terraform"
  }

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y python3 python3-pip
              pip3 install boto3 pandas
              EOF
}

# Data source for current AWS account
data "aws_caller_identity" "current" {}

# Outputs
output "bucket_name" {
  value       = aws_s3_bucket.data_lake.id
  description = "Name of the S3 data lake bucket"
}

output "instance_id" {
  value       = aws_instance.data_processor.id
  description = "ID of the EC2 data processor instance"
}

output "instance_public_ip" {
  value       = aws_instance.data_processor.public_ip
  description = "Public IP of the EC2 instance"
}
```

### Variables Configuration

```hcl
# terraform/variables.tf
variable "aws_region" {
  description = "AWS region for resources"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "development"
}

variable "bucket_prefix" {
  description = "Prefix for S3 bucket name"
  type        = string
  default     = "data-engineering"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}
```

### Using Variables

```bash
# Pass variables via command line
terraform -chdir=terraform apply -var="environment=staging" -var="instance_type=t2.small"

# Use a variables file
echo 'environment = "production"' > terraform/terraform.tfvars
terraform -chdir=terraform apply
```

## Common Patterns

### Pattern 1: Multi-Environment Setup

```hcl
# terraform/main.tf
locals {
  environment = terraform.workspace
  
  bucket_name = "${var.bucket_prefix}-${local.environment}-${data.aws_caller_identity.current.account_id}"
  
  common_tags = {
    Environment = local.environment
    ManagedBy   = "Terraform"
    Project     = "DataEngineering"
  }
}

resource "aws_s3_bucket" "data_lake" {
  bucket = local.bucket_name
  tags   = merge(local.common_tags, {
    Name = "Data Lake - ${local.environment}"
  })
}
```

```bash
# Create and use workspaces
terraform -chdir=terraform workspace new staging
terraform -chdir=terraform workspace new production
terraform -chdir=terraform workspace select staging
terraform -chdir=terraform apply
```

### Pattern 2: IAM Role for EC2 to Access S3

```hcl
# IAM role for EC2 instance
resource "aws_iam_role" "ec2_s3_access" {
  name = "ec2-data-processor-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
  })
}

# IAM policy for S3 access
resource "aws_iam_role_policy" "s3_access" {
  name = "s3-data-lake-access"
  role = aws_iam_role.ec2_s3_access.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ]
      Resource = [
        aws_s3_bucket.data_lake.arn,
        "${aws_s3_bucket.data_lake.arn}/*"
      ]
    }]
  })
}

# Instance profile
resource "aws_iam_instance_profile" "ec2_profile" {
  name = "ec2-data-processor-profile"
  role = aws_iam_role.ec2_s3_access.name
}

# Attach profile to EC2 instance
resource "aws_instance" "data_processor" {
  ami                  = "ami-0c55b159cbfafe1f0"
  instance_type        = "t2.micro"
  iam_instance_profile = aws_iam_instance_profile.ec2_profile.name

  tags = {
    Name = "Data Processor with S3 Access"
  }
}
```

### Pattern 3: S3 Bucket Lifecycle Rules

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "data_lake_lifecycle" {
  bucket = aws_s3_bucket.data_lake.id

  rule {
    id     = "archive-old-data"
    status = "Enabled"

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    expiration {
      days = 365
    }

    filter {
      prefix = "raw-data/"
    }
  }

  rule {
    id     = "delete-temp-data"
    status = "Enabled"

    expiration {
      days = 7
    }

    filter {
      prefix = "temp/"
    }
  }
}
```

### Pattern 4: Remote State Backend with S3

```hcl
# terraform/backend.tf
terraform {
  backend "s3" {
    bucket         = "terraform-state-bucket-${AWS_ACCOUNT_ID}"
    key            = "data-engineering/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

```bash
# First, create the backend resources manually or with a separate Terraform config
# Then initialize with backend
terraform -chdir=terraform init -backend-config="bucket=my-terraform-state-bucket"
```

## Verifying Deployed Infrastructure

### Check S3 Buckets

```bash
# List all S3 buckets
aws s3 ls

# List contents of specific bucket
aws s3 ls s3://your-bucket-name/

# Get bucket details
aws s3api get-bucket-versioning --bucket your-bucket-name
aws s3api get-bucket-tagging --bucket your-bucket-name
```

### Check EC2 Instances

```bash
# List running instances
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].{ID:InstanceId, Name:Tags[?Key==`Name`].Value, Type:InstanceType, State:State.Name, PublicIP:PublicIpAddress, PrivateIP:PrivateIpAddress}' \
  --output table

# Get specific instance details
aws ec2 describe-instances --instance-ids i-1234567890abcdef0
```

### Check IAM Resources

```bash
# List IAM roles
aws iam list-roles --query 'Roles[?starts_with(RoleName, `ec2-data`)].RoleName'

# Get role policy
aws iam get-role-policy --role-name ec2-data-processor-role --policy-name s3-data-lake-access
```

## Troubleshooting

### Issue: Bucket Name Already Exists

**Problem**: S3 bucket names must be globally unique.

```
Error: creating S3 Bucket: BucketAlreadyExists
```

**Solution**: Use a unique bucket name with account ID or random suffix.

```hcl
data "aws_caller_identity" "current" {}

resource "aws_s3_bucket" "data_lake" {
  bucket = "my-data-lake-${data.aws_caller_identity.current.account_id}"
}
```

### Issue: Invalid Credentials

**Problem**: AWS credentials not configured or expired.

```
Error: error configuring Terraform AWS Provider: no valid credential sources
```

**Solution**: Configure AWS CLI or use environment variables.

```bash
# Configure AWS CLI
aws configure

# Or set environment variables
export AWS_ACCESS_KEY_ID=your_access_key
export AWS_SECRET_ACCESS_KEY=your_secret_key
export AWS_DEFAULT_REGION=us-east-1
```

### Issue: State Lock Conflict

**Problem**: Another Terraform process is holding the state lock.

```
Error: Error acquiring the state lock
```

**Solution**: Wait for other operation to complete or force unlock (use carefully).

```bash
# Force unlock (only if you're certain no other process is running)
terraform -chdir=terraform force-unlock LOCK_ID
```

### Issue: Resource Already Exists

**Problem**: Resource exists outside of Terraform state.

```
Error: resource already exists
```

**Solution**: Import existing resource into state.

```bash
# Import S3 bucket
terraform -chdir=terraform import aws_s3_bucket.data_lake existing-bucket-name

# Import EC2 instance
terraform -chdir=terraform import aws_instance.data_processor i-1234567890abcdef0
```

### Issue: Permission Denied

**Problem**: IAM user lacks required permissions.

```
Error: UnauthorizedOperation: You are not authorized to perform this operation
```

**Solution**: Verify IAM permissions include necessary actions.

```bash
# Check current user identity
aws sts get-caller-identity

# Test specific permissions
aws s3 ls  # Test S3 access
aws ec2 describe-instances  # Test EC2 access
```

### Issue: Invalid AMI ID

**Problem**: AMI ID not available in your region.

```
Error: InvalidAMIID.NotFound
```

**Solution**: Use data source to find current AMI.

```hcl
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "data_processor" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
}
```

## Best Practices

1. **Always use version control**: Track your Terraform configurations in Git
2. **Use remote state**: Store state in S3 with DynamoDB locking for team collaboration
3. **Tag all resources**: Include Environment, ManagedBy, and Project tags
4. **Use variables**: Make configurations reusable across environments
5. **Plan before apply**: Always run `terraform plan` to review changes
6. **Avoid hardcoded credentials**: Use IAM roles and environment variables
7. **Use `.gitignore`**: Never commit `.tfstate`, `.tfvars`, or credential files
8. **Implement least privilege**: Grant minimal IAM permissions needed
9. **Enable encryption**: Use encryption for S3 buckets and EBS volumes
10. **Cost management**: Always destroy resources when not needed

## Example `.gitignore`

```
# terraform/.gitignore
*.tfstate
*.tfstate.*
.terraform/
.terraform.lock.hcl
terraform.tfvars
*.tfvars
override.tf
override.tf.json
*_override.tf
*_override.tf.json
crash.log
```

## Quick Start Workflow

```bash
# 1. Clone and navigate to project
git clone https://github.com/josephmachado/iac-for-data-engineering-terraform-.git
cd iac-for-data-engineering-terraform-

# 2. Update bucket name in main.tf to be unique
# Edit terraform/main.tf and change bucket name

# 3. Initialize Terraform
terraform -chdir=terraform init

# 4. Validate configuration
terraform -chdir=terraform validate

# 5. Format code
terraform -chdir=terraform fmt

# 6. Preview changes
terraform -chdir=terraform plan

# 7. Apply infrastructure
terraform -chdir=terraform apply

# 8. Verify resources
aws s3 ls
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running"

# 9. When done, destroy infrastructure
terraform -chdir=terraform destroy
```

This skill provides the foundation for managing data engineering infrastructure as code using Terraform on AWS.
