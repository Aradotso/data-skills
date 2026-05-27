---
name: iac-terraform-data-engineering
description: Infrastructure-as-Code fundamentals for data engineering using Terraform to provision AWS resources (S3, EC2, IAM)
triggers:
  - "set up terraform for data engineering"
  - "create AWS infrastructure with terraform"
  - "provision S3 and EC2 with IaC"
  - "manage data engineering infrastructure as code"
  - "deploy AWS resources using terraform"
  - "initialize terraform for data pipelines"
  - "destroy terraform infrastructure"
  - "terraform state management for data engineering"
---

# IaC for Data Engineering with Terraform

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides Infrastructure-as-Code (IaC) fundamentals for data engineers using Terraform to provision and manage AWS resources including S3 buckets, EC2 instances, and IAM policies.

## What It Does

- Provisions AWS infrastructure for data engineering workloads using Terraform
- Creates S3 buckets for data storage
- Deploys EC2 instances for compute workloads
- Manages IAM users and policies for access control
- Maintains infrastructure state for reproducible deployments
- Enables version-controlled infrastructure changes

## Prerequisites

1. AWS Account with root access
2. Terraform installed ([installation guide](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli))
3. AWS CLI installed and configured
4. IAM user with appropriate permissions (S3, EC2, IAM full access)

## Setup AWS Credentials

Configure AWS CLI with your credentials:

```bash
aws configure
# Provide:
# - AWS Access Key ID
# - AWS Secret Access Key
# - Default region (e.g., us-east-1)
# - Output format (json)
```

Alternatively, use environment variables:

```bash
export AWS_ACCESS_KEY_ID=$YOUR_AWS_ACCESS_KEY_ID
export AWS_SECRET_ACCESS_KEY=$YOUR_AWS_SECRET_ACCESS_KEY
export AWS_DEFAULT_REGION=us-east-1
```

## Project Structure

```
terraform/
├── main.tf           # Main Terraform configuration
├── variables.tf      # Variable definitions
├── outputs.tf        # Output definitions
└── terraform.tfstate # State file (generated)
```

## Key Terraform Commands

### Initialize Terraform

Initialize the working directory and download required providers:

```bash
terraform -chdir=terraform init
```

### Validate Configuration

Check if the configuration is syntactically valid:

```bash
terraform -chdir=terraform validate
```

### Format Code

Format Terraform files to canonical style:

```bash
terraform -chdir=terraform fmt
```

### Plan Changes

Preview what Terraform will create/modify/destroy:

```bash
terraform -chdir=terraform plan
```

### Apply Configuration

Create or update infrastructure:

```bash
terraform -chdir=terraform apply
```

For non-interactive mode:

```bash
terraform -chdir=terraform apply -auto-approve
```

### Destroy Infrastructure

Remove all resources managed by Terraform:

```bash
terraform -chdir=terraform destroy
```

For non-interactive mode:

```bash
terraform -chdir=terraform destroy -auto-approve
```

## Configuration Example

### Basic S3 Bucket Configuration

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
  region = var.aws_region
}

resource "aws_s3_bucket" "data_bucket" {
  bucket = "my-unique-data-eng-bucket-${var.environment}"
  
  tags = {
    Name        = "Data Engineering Bucket"
    Environment = var.environment
    Project     = "data-engineering"
  }
}

resource "aws_s3_bucket_versioning" "data_bucket_versioning" {
  bucket = aws_s3_bucket.data_bucket.id
  
  versioning_configuration {
    status = "Enabled"
  }
}
```

### EC2 Instance for Data Processing

```hcl
# terraform/main.tf (continued)
resource "aws_instance" "data_processor" {
  ami           = var.ec2_ami
  instance_type = var.ec2_instance_type
  
  tags = {
    Name        = "Data Processor"
    Environment = var.environment
  }
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y python3 python3-pip
              pip3 install pandas boto3
              EOF
}
```

### Variables Definition

```hcl
# terraform/variables.tf
variable "aws_region" {
  description = "AWS region for resources"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  default     = "dev"
}

variable "ec2_ami" {
  description = "AMI ID for EC2 instance"
  type        = string
  default     = "ami-0c55b159cbfafe1f0" # Amazon Linux 2
}

variable "ec2_instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}
```

### Outputs Definition

```hcl
# terraform/outputs.tf
output "s3_bucket_name" {
  description = "Name of the S3 bucket"
  value       = aws_s3_bucket.data_bucket.id
}

output "s3_bucket_arn" {
  description = "ARN of the S3 bucket"
  value       = aws_s3_bucket.data_bucket.arn
}

output "ec2_instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.data_processor.id
}

output "ec2_public_ip" {
  description = "Public IP of the EC2 instance"
  value       = aws_instance.data_processor.public_ip
}
```

## State Management

### List Resources in State

```bash
terraform -chdir=terraform state list
```

### Show Resource Details

```bash
terraform -chdir=terraform state show aws_s3_bucket.data_bucket
```

### View State as JSON

```bash
cat terraform/terraform.tfstate | jq -r '.resources[] | [.type, .name] | join(",")'
```

## Verification Commands

### Verify S3 Buckets

```bash
aws s3 ls
```

### Verify EC2 Instances

```bash
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].{ID:InstanceId, Name:Tags[?Key==`Name`].Value, Type:InstanceType, State:State.Name, PublicIP:PublicIpAddress, PrivateIP:PrivateIpAddress}' \
  --output table
```

## Common Patterns

### Data Lake Storage Setup

```hcl
resource "aws_s3_bucket" "data_lake" {
  bucket = "data-lake-${var.project_name}-${var.environment}"
}

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
  }
}
```

### IAM Role for Data Processing

```hcl
resource "aws_iam_role" "data_processor_role" {
  name = "data-processor-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "s3_full_access" {
  role       = aws_iam_role.data_processor_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
}
```

### Multiple Environment Setup

```hcl
# terraform/environments/dev.tfvars
environment       = "dev"
ec2_instance_type = "t2.micro"
```

```hcl
# terraform/environments/prod.tfvars
environment       = "prod"
ec2_instance_type = "t3.large"
```

Apply with specific environment:

```bash
terraform -chdir=terraform apply -var-file="environments/dev.tfvars"
```

## Troubleshooting

### Bucket Name Already Exists

**Error**: `BucketAlreadyExists: The requested bucket name is not available`

**Solution**: Change the bucket name in `main.tf` to something globally unique:

```hcl
resource "aws_s3_bucket" "data_bucket" {
  bucket = "my-unique-prefix-${random_id.bucket_suffix.hex}-data-bucket"
}

resource "random_id" "bucket_suffix" {
  byte_length = 8
}
```

### State Lock Issues

**Error**: `Error acquiring the state lock`

**Solution**: If a previous operation crashed, manually remove the lock:

```bash
terraform -chdir=terraform force-unlock LOCK_ID
```

### AWS Credentials Not Found

**Error**: `No valid credential sources found`

**Solution**: Ensure AWS CLI is configured or environment variables are set:

```bash
aws configure list
# or
echo $AWS_ACCESS_KEY_ID
```

### Insufficient IAM Permissions

**Error**: `AccessDenied` or `UnauthorizedOperation`

**Solution**: Ensure your IAM user/role has necessary permissions (S3, EC2, IAM). For development, you can attach these AWS managed policies:
- `AmazonS3FullAccess`
- `AmazonEC2FullAccess`
- `IAMFullAccess`

### Resource Already Exists

**Error**: Resource already exists in AWS but not in state

**Solution**: Import the existing resource:

```bash
terraform -chdir=terraform import aws_s3_bucket.data_bucket existing-bucket-name
```

### Destroy Fails on S3 Bucket

**Error**: `BucketNotEmpty: The bucket you tried to delete is not empty`

**Solution**: Empty the bucket first or force deletion:

```hcl
resource "aws_s3_bucket" "data_bucket" {
  bucket        = "my-data-bucket"
  force_destroy = true
}
```

## Best Practices

1. **Always use variables** for configurable values (region, instance types, bucket names)
2. **Enable versioning** on S3 buckets for data protection
3. **Use remote state** for team collaboration (e.g., S3 backend)
4. **Tag all resources** for cost tracking and organization
5. **Run `terraform plan`** before `apply` to preview changes
6. **Use workspaces** for managing multiple environments
7. **Store state securely** and enable state locking (using DynamoDB)
8. **Never commit** `terraform.tfstate` or `.terraform/` to version control

## Additional Resources

- Blog post: https://www.startdataengineering.com/post/iac-for-data-engineering-terraform/
- Terraform AWS Provider: https://registry.terraform.io/providers/hashicorp/aws/latest/docs
