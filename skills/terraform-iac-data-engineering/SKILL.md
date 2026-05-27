---
name: terraform-iac-data-engineering
description: Infrastructure-as-Code fundamentals for data engineering using Terraform to provision AWS resources like S3, EC2, and IAM
triggers:
  - "set up terraform for data engineering"
  - "create AWS infrastructure with terraform"
  - "provision S3 and EC2 with IaC"
  - "terraform data engineering setup"
  - "infrastructure as code for data pipelines"
  - "deploy AWS resources using terraform"
  - "terraform state management for data infra"
  - "automate AWS infrastructure provisioning"
---

# Terraform IaC for Data Engineering

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates Infrastructure-as-Code (IaC) fundamentals for data engineering using Terraform. It provisions AWS resources commonly needed in data engineering workflows, including S3 buckets for data storage and EC2 instances for compute workloads. The project emphasizes declarative infrastructure management, state tracking, and reproducible deployments.

## What This Project Does

- Provisions AWS S3 buckets for data lake/warehouse storage
- Creates EC2 instances for data processing workloads
- Manages IAM permissions for resource access
- Maintains infrastructure state with Terraform state files
- Enables version-controlled, repeatable infrastructure deployments

## Prerequisites

1. **AWS Account** with root or administrative access
2. **Terraform CLI** installed ([installation guide](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli))
3. **AWS CLI** installed ([installation guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html))
4. **AWS CLI configured** with credentials

### AWS CLI Configuration

```bash
# Configure AWS CLI with your credentials
aws configure
# Enter AWS Access Key ID, Secret Access Key, region, and output format
```

### IAM Permissions Setup

The AWS user or role running Terraform needs:
- Full S3 access (`s3:*`)
- Full EC2 access (`ec2:*`)
- Full IAM access (`iam:*`)

**Note**: Full access permissions are for development/learning. Production environments should use least-privilege policies.

## Installation & Setup

1. **Clone the repository**:
```bash
git clone https://github.com/josephmachado/iac-for-data-engineering-terraform.git
cd iac-for-data-engineering-terraform
```

2. **Configure your infrastructure**:
Edit `terraform/main.tf` to customize bucket names and other resources. Bucket names must be globally unique in AWS.

```hcl
# Example main.tf configuration
resource "aws_s3_bucket" "data_bucket" {
  bucket = "your-unique-bucket-name-${random_id.bucket_suffix.hex}"
  
  tags = {
    Name        = "Data Engineering Bucket"
    Environment = "dev"
  }
}
```

3. **Initialize Terraform**:
```bash
terraform -chdir=terraform init
```

This downloads required provider plugins (AWS) and prepares the working directory.

## Core Terraform Commands

### Initialize Project
```bash
terraform -chdir=terraform init
```
Downloads providers and initializes backend. Run this first or after adding new providers.

### Validate Configuration
```bash
terraform -chdir=terraform validate
```
Checks syntax and configuration validity without accessing remote services.

### Format Code
```bash
terraform -chdir=terraform fmt
```
Automatically formats `.tf` files to canonical style.

### Plan Changes
```bash
terraform -chdir=terraform plan
```
Shows what changes Terraform will make without applying them. Always review before applying.

### Apply Changes
```bash
terraform -chdir=terraform apply
```
Creates or updates infrastructure. Prompts for confirmation unless `-auto-approve` is used.

### Destroy Infrastructure
```bash
terraform -chdir=terraform destroy
```
Removes all resources managed by Terraform. Use carefully in production.

### View State
```bash
# List all resources in state
terraform -chdir=terraform state list

# Show details of specific resource
terraform -chdir=terraform state show aws_s3_bucket.data_bucket

# View state as JSON
cat terraform/terraform.tfstate | jq -r '.resources[] | [.type, .name] | join(",")'
```

## Configuration Patterns

### Basic S3 Bucket for Data Storage

```hcl
resource "aws_s3_bucket" "data_lake" {
  bucket = "my-data-lake-${var.environment}"

  tags = {
    Name        = "Data Lake"
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# Enable versioning for data protection
resource "aws_s3_bucket_versioning" "data_lake_versioning" {
  bucket = aws_s3_bucket.data_lake.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Enable encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "data_lake_encryption" {
  bucket = aws_s3_bucket.data_lake.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

### EC2 Instance for Data Processing

```hcl
resource "aws_instance" "data_processor" {
  ami           = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2
  instance_type = "t3.medium"

  tags = {
    Name        = "Data Processing Instance"
    Environment = var.environment
  }

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y python3 python3-pip
              pip3 install pandas boto3
              EOF
}

# Security group for EC2
resource "aws_security_group" "data_processor_sg" {
  name        = "data-processor-sg"
  description = "Security group for data processing instance"

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["YOUR_IP_ADDRESS/32"]  # Restrict to your IP
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### Using Variables

```hcl
# variables.tf
variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  default     = "dev"
}

variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "bucket_prefix" {
  description = "Prefix for S3 bucket names"
  type        = string
}

# terraform.tfvars (create this file, don't commit secrets)
environment   = "dev"
region        = "us-east-1"
bucket_prefix = "mycompany-data"
```

### Using Outputs

```hcl
# outputs.tf
output "bucket_name" {
  description = "Name of the created S3 bucket"
  value       = aws_s3_bucket.data_lake.id
}

output "bucket_arn" {
  description = "ARN of the created S3 bucket"
  value       = aws_s3_bucket.data_lake.arn
}

output "ec2_instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.data_processor.id
}

output "ec2_public_ip" {
  description = "Public IP of EC2 instance"
  value       = aws_instance.data_processor.public_ip
}
```

## Real-World Example: Complete Data Infrastructure

```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region
}

# Random suffix for unique bucket names
resource "random_id" "bucket_suffix" {
  byte_length = 4
}

# Raw data bucket
resource "aws_s3_bucket" "raw_data" {
  bucket = "${var.bucket_prefix}-raw-${random_id.bucket_suffix.hex}"

  tags = {
    Name        = "Raw Data Bucket"
    Environment = var.environment
    Layer       = "raw"
  }
}

# Processed data bucket
resource "aws_s3_bucket" "processed_data" {
  bucket = "${var.bucket_prefix}-processed-${random_id.bucket_suffix.hex}"

  tags = {
    Name        = "Processed Data Bucket"
    Environment = var.environment
    Layer       = "processed"
  }
}

# IAM role for EC2 to access S3
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

# IAM policy for S3 access
resource "aws_iam_role_policy" "s3_access_policy" {
  name = "s3-access-policy"
  role = aws_iam_role.data_processor_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.raw_data.arn,
          "${aws_s3_bucket.raw_data.arn}/*",
          aws_s3_bucket.processed_data.arn,
          "${aws_s3_bucket.processed_data.arn}/*"
        ]
      }
    ]
  })
}

# Instance profile for EC2
resource "aws_iam_instance_profile" "data_processor_profile" {
  name = "data-processor-profile"
  role = aws_iam_role.data_processor_role.name
}

# EC2 instance with IAM role
resource "aws_instance" "data_processor" {
  ami                    = "ami-0c55b159cbfafe1f0"
  instance_type          = "t3.medium"
  iam_instance_profile   = aws_iam_instance_profile.data_processor_profile.name

  tags = {
    Name        = "Data Processor"
    Environment = var.environment
  }
}
```

## Verifying Infrastructure

### Check S3 Buckets
```bash
# List all S3 buckets
aws s3 ls

# List contents of specific bucket
aws s3 ls s3://your-bucket-name/
```

### Check EC2 Instances
```bash
# List running EC2 instances
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].{ID:InstanceId, Name:Tags[?Key==`Name`].Value, Type:InstanceType, State:State.Name, PublicIP:PublicIpAddress, PrivateIP:PrivateIpAddress}' \
  --output table
```

### Inspect Terraform State
```bash
# List all managed resources
terraform -chdir=terraform state list

# Show resource details
terraform -chdir=terraform state show aws_s3_bucket.data_bucket

# Export state as JSON for parsing
terraform -chdir=terraform show -json > state.json
cat state.json | jq '.values.root_module.resources[] | {type: .type, name: .name}'
```

## Common Workflows

### Initial Deployment
```bash
cd iac-for-data-engineering-terraform

# Initialize Terraform
terraform -chdir=terraform init

# Review what will be created
terraform -chdir=terraform plan

# Apply changes
terraform -chdir=terraform apply
```

### Making Changes
```bash
# 1. Edit .tf files
# 2. Validate syntax
terraform -chdir=terraform validate

# 3. Format code
terraform -chdir=terraform fmt

# 4. Review changes
terraform -chdir=terraform plan

# 5. Apply changes
terraform -chdir=terraform apply
```

### Tear Down
```bash
# Remove all infrastructure
terraform -chdir=terraform destroy

# Auto-approve (use carefully)
terraform -chdir=terraform destroy -auto-approve
```

## Troubleshooting

### Bucket Name Already Exists
**Error**: `BucketAlreadyExists` or `BucketAlreadyOwnedByYou`

**Solution**: S3 bucket names must be globally unique. Add a random suffix:
```hcl
resource "random_id" "suffix" {
  byte_length = 4
}

resource "aws_s3_bucket" "data_bucket" {
  bucket = "my-bucket-${random_id.suffix.hex}"
}
```

### Authentication Errors
**Error**: `Error: error configuring Terraform AWS Provider: no valid credential sources`

**Solution**: Ensure AWS CLI is configured:
```bash
aws configure
# Or use environment variables
export AWS_ACCESS_KEY_ID="${AWS_ACCESS_KEY_ID}"
export AWS_SECRET_ACCESS_KEY="${AWS_SECRET_ACCESS_KEY}"
export AWS_DEFAULT_REGION="us-east-1"
```

### State Lock Issues
**Error**: `Error acquiring the state lock`

**Solution**: If Terraform crashed, manually unlock:
```bash
terraform -chdir=terraform force-unlock LOCK_ID
```

### Resource Already Exists
**Error**: Resource already exists but not in state

**Solution**: Import existing resource:
```bash
terraform -chdir=terraform import aws_s3_bucket.data_bucket existing-bucket-name
```

### Permission Denied
**Error**: `AccessDenied` or `UnauthorizedOperation`

**Solution**: Verify IAM permissions for your AWS user/role include necessary services (S3, EC2, IAM).

### State Out of Sync
**Solution**: Refresh state to match real infrastructure:
```bash
terraform -chdir=terraform refresh
```

## Best Practices

1. **Never commit `terraform.tfstate`** - Contains sensitive data and should be stored in remote backend (S3 + DynamoDB for locking)

2. **Use variables** - Make configurations reusable across environments

3. **Enable versioning** - For S3 buckets containing critical data

4. **Tag everything** - Use consistent tagging for cost tracking and resource management

5. **Use remote state** - For team collaboration:
```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state-bucket"
    key            = "data-infra/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

6. **Run `terraform plan`** - Always review before applying changes

7. **Use `.gitignore`** - Exclude sensitive files:
```
.terraform/
*.tfstate
*.tfstate.backup
.terraform.lock.hcl
terraform.tfvars
```
