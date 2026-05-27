---
name: terraform-data-engineering-infrastructure
description: Infrastructure-as-Code patterns for data engineering using Terraform to provision AWS resources (S3, EC2, IAM)
triggers:
  - "set up data engineering infrastructure with terraform"
  - "create AWS resources for data pipelines using IaC"
  - "provision S3 buckets and EC2 instances with terraform"
  - "manage data infrastructure as code"
  - "deploy data engineering environment on AWS"
  - "terraform for data platform setup"
  - "infrastructure as code for data engineering"
  - "automate AWS data infrastructure provisioning"
---

# Terraform Data Engineering Infrastructure

> Skill by [ara.so](https://ara.so) — Data Skills collection

This project provides Infrastructure-as-Code (IaC) fundamentals for data engineers using Terraform to provision and manage AWS resources. It demonstrates how to automate the creation of data infrastructure including S3 buckets for data storage and EC2 instances for compute workloads.

## Prerequisites

Before using this project, ensure you have:

1. AWS Account with appropriate access
2. Terraform CLI installed
3. AWS CLI installed and configured
4. IAM user with S3, EC2, and IAM permissions

## Installation

### Install Terraform

**macOS:**
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

**Linux:**
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```

### Install AWS CLI

```bash
# macOS
brew install awscli

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### Configure AWS Credentials

```bash
aws configure
# Enter your AWS Access Key ID: $AWS_ACCESS_KEY_ID
# Enter your AWS Secret Access Key: $AWS_SECRET_ACCESS_KEY
# Default region name: us-east-1
# Default output format: json
```

## Project Structure

A typical Terraform data engineering setup includes:

```
terraform/
├── main.tf           # Main infrastructure definitions
├── variables.tf      # Input variables
├── outputs.tf        # Output values
├── terraform.tfstate # State file (auto-generated)
└── .terraform/       # Provider plugins (auto-generated)
```

## Core Terraform Workflow

### 1. Initialize Terraform

Initialize the working directory and download required providers:

```bash
terraform -chdir=terraform init
```

This command:
- Downloads AWS provider plugins
- Initializes backend configuration
- Prepares the working directory

### 2. Validate Configuration

Check configuration syntax and validity:

```bash
terraform -chdir=terraform validate
```

### 3. Format Configuration

Format Terraform files to canonical style:

```bash
terraform -chdir=terraform fmt
```

### 4. Plan Changes

Preview infrastructure changes before applying:

```bash
terraform -chdir=terraform plan
```

This shows:
- Resources to be created (+)
- Resources to be modified (~)
- Resources to be destroyed (-)

### 5. Apply Changes

Provision the infrastructure:

```bash
terraform -chdir=terraform apply
```

Type `yes` when prompted to confirm the changes.

### 6. Destroy Infrastructure

Remove all provisioned resources:

```bash
terraform -chdir=terraform destroy
```

## Key Terraform Configuration Patterns

### S3 Bucket for Data Storage

```hcl
resource "aws_s3_bucket" "data_lake" {
  bucket = "my-unique-data-lake-bucket-${var.environment}"

  tags = {
    Name        = "Data Lake"
    Environment = var.environment
    Project     = "DataEngineering"
  }
}

# Enable versioning for data protection
resource "aws_s3_bucket_versioning" "data_lake_versioning" {
  bucket = aws_s3_bucket.data_lake.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Configure lifecycle rules
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

### EC2 Instance for Data Processing

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

resource "aws_instance" "data_processor" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  tags = {
    Name        = "DataProcessor"
    Environment = var.environment
  }

  user_data = <<-EOF
              #!/bin/bash
              apt-get update
              apt-get install -y python3-pip
              pip3 install pandas boto3
              EOF
}
```

### IAM Role for EC2 S3 Access

```hcl
resource "aws_iam_role" "ec2_s3_access" {
  name = "ec2-s3-access-role"

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

resource "aws_iam_role_policy" "s3_full_access" {
  name = "s3-full-access"
  role = aws_iam_role.ec2_s3_access.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "s3:*"
        ]
        Effect   = "Allow"
        Resource = "*"
      }
    ]
  })
}

resource "aws_iam_instance_profile" "ec2_profile" {
  name = "ec2-s3-profile"
  role = aws_iam_role.ec2_s3_access.name
}
```

### Variables Configuration

Create `terraform/variables.tf`:

```hcl
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

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.medium"
}

variable "bucket_prefix" {
  description = "S3 bucket name prefix"
  type        = string
}
```

### Outputs Configuration

Create `terraform/outputs.tf`:

```hcl
output "s3_bucket_name" {
  description = "Name of the S3 bucket"
  value       = aws_s3_bucket.data_lake.id
}

output "s3_bucket_arn" {
  description = "ARN of the S3 bucket"
  value       = aws_s3_bucket.data_lake.arn
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

### View Current State

```bash
# List all resources in state
terraform -chdir=terraform state list

# Show specific resource details
terraform -chdir=terraform state show aws_s3_bucket.data_lake

# View state as JSON
cat terraform/terraform.tfstate | jq -r '.resources[] | [.type, .name] | join(",")'
```

### Remote State (Production Pattern)

For team collaboration, use remote state with S3 backend:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "data-engineering/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

## Verification Commands

### Verify S3 Buckets

```bash
# List all S3 buckets
aws s3 ls

# List bucket contents
aws s3 ls s3://your-bucket-name/

# Upload test file
echo "test data" > test.txt
aws s3 cp test.txt s3://your-bucket-name/
```

### Verify EC2 Instances

```bash
# List running instances
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].{ID:InstanceId, Name:Tags[?Key==`Name`].Value, Type:InstanceType, State:State.Name, PublicIP:PublicIpAddress, PrivateIP:PrivateIpAddress}' \
  --output table

# Get instance details
aws ec2 describe-instances --instance-ids i-1234567890abcdef0
```

## Common Data Engineering Patterns

### Multi-Environment Setup

```hcl
# main.tf
locals {
  environments = {
    dev = {
      instance_type = "t3.small"
      bucket_suffix = "dev"
    }
    prod = {
      instance_type = "t3.large"
      bucket_suffix = "prod"
    }
  }
  
  env_config = local.environments[var.environment]
}

resource "aws_s3_bucket" "data_lake" {
  bucket = "${var.bucket_prefix}-${local.env_config.bucket_suffix}"
}
```

### Data Lake Zones

```hcl
resource "aws_s3_bucket" "raw_zone" {
  bucket = "data-lake-raw-${var.environment}"
}

resource "aws_s3_bucket" "processed_zone" {
  bucket = "data-lake-processed-${var.environment}"
}

resource "aws_s3_bucket" "curated_zone" {
  bucket = "data-lake-curated-${var.environment}"
}
```

### VPC for Isolated Data Processing

```hcl
resource "aws_vpc" "data_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true

  tags = {
    Name = "DataEngineering-VPC"
  }
}

resource "aws_subnet" "private_subnet" {
  vpc_id     = aws_vpc.data_vpc.id
  cidr_block = "10.0.1.0/24"

  tags = {
    Name = "Private-Subnet"
  }
}
```

## Troubleshooting

### Bucket Name Already Exists

**Error:** `BucketAlreadyExists: The requested bucket name is not available`

**Solution:** S3 bucket names must be globally unique. Change the bucket name in `main.tf`:

```hcl
resource "aws_s3_bucket" "data_lake" {
  bucket = "my-unique-prefix-${random_id.bucket_suffix.hex}"
}

resource "random_id" "bucket_suffix" {
  byte_length = 4
}
```

### Insufficient IAM Permissions

**Error:** `AccessDenied` or `UnauthorizedOperation`

**Solution:** Ensure your IAM user has necessary permissions:

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

### State Lock Issues

**Error:** `Error acquiring the state lock`

**Solution:** If using DynamoDB for state locking:

```bash
# Force unlock (use with caution)
terraform -chdir=terraform force-unlock LOCK_ID
```

### Resource Already Exists

**Error:** Resource already exists in AWS but not in state

**Solution:** Import existing resource:

```bash
terraform -chdir=terraform import aws_s3_bucket.data_lake my-existing-bucket-name
```

## Best Practices for Data Engineering IaC

1. **Use unique bucket names**: Include timestamps or random suffixes
2. **Enable versioning**: Protect against accidental data deletion
3. **Tag resources**: Use consistent tagging for cost tracking and organization
4. **Separate environments**: Use workspaces or separate state files
5. **Secure credentials**: Never commit AWS credentials; use environment variables
6. **Regular backups**: Back up Terraform state files
7. **Plan before apply**: Always review changes with `terraform plan`
8. **Cost monitoring**: Tag resources and enable AWS cost allocation tags

## Environment Variables

```bash
# AWS credentials
export AWS_ACCESS_KEY_ID=your_access_key
export AWS_SECRET_ACCESS_KEY=your_secret_key
export AWS_DEFAULT_REGION=us-east-1

# Terraform variables
export TF_VAR_environment=dev
export TF_VAR_bucket_prefix=my-company-data-lake
```

## Integration with Data Pipelines

After provisioning infrastructure, reference outputs in data pipelines:

```python
import boto3
import os

# Use the bucket created by Terraform
bucket_name = os.environ.get('TERRAFORM_S3_BUCKET')
s3_client = boto3.client('s3')

# Upload data
s3_client.upload_file('local_file.csv', bucket_name, 'raw/data.csv')
```
