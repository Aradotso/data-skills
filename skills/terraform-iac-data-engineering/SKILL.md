---
name: terraform-iac-data-engineering
description: Infrastructure-as-Code with Terraform for data engineering workloads on AWS (S3, EC2, IAM)
triggers:
  - "set up terraform for data engineering"
  - "create AWS infrastructure with terraform"
  - "manage data engineering infrastructure as code"
  - "deploy S3 and EC2 with terraform"
  - "terraform state management for data pipelines"
  - "provision AWS resources for data engineering"
  - "infrastructure as code for data workflows"
  - "terraform apply and destroy data infrastructure"
---

# Terraform IaC for Data Engineering

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides Infrastructure-as-Code (IaC) fundamentals for data engineers using Terraform to manage AWS resources. It demonstrates how to provision and manage data engineering infrastructure including S3 buckets, EC2 instances, and IAM permissions through declarative configuration files.

## What This Project Does

- Provisions AWS infrastructure for data engineering workloads using Terraform
- Manages S3 buckets for data storage
- Creates and configures EC2 instances for data processing
- Sets up IAM roles and policies for secure access
- Maintains infrastructure state for consistent deployments
- Provides reproducible infrastructure definitions in HCL (HashiCorp Configuration Language)

## Prerequisites

1. **AWS Account** with root or administrative access
2. **Terraform CLI** installed ([installation guide](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli))
3. **AWS CLI** installed ([installation guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html))
4. **AWS CLI configured** with credentials ([quickstart](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html))

## Installation & Setup

### 1. Configure AWS Credentials

Ensure your AWS CLI is configured with appropriate credentials:

```bash
aws configure
# Enter your AWS Access Key ID, Secret Access Key, default region, and output format
```

### 2. Set Up IAM Permissions

Create an IAM user with the following permissions:
- Full S3 access
- Full EC2 access
- Full IAM access

**Note**: For production, use least-privilege policies instead of full access.

### 3. Clone and Initialize

```bash
git clone https://github.com/josephmachado/iac-for-data-engineering-terraform-.git
cd iac-for-data-engineering-terraform-
```

## Core Terraform Commands

### Initialize Terraform

Initialize the working directory and download required providers:

```bash
terraform -chdir=terraform init
```

### Validate Configuration

Check configuration syntax and internal consistency:

```bash
terraform -chdir=terraform validate
```

### Format Configuration Files

Automatically format HCL files to canonical style:

```bash
terraform -chdir=terraform fmt
```

### Plan Infrastructure Changes

Preview changes before applying:

```bash
terraform -chdir=terraform plan
```

### Apply Infrastructure Changes

Create or update infrastructure:

```bash
terraform -chdir=terraform apply
```

You'll be prompted to confirm. To auto-approve:

```bash
terraform -chdir=terraform apply -auto-approve
```

### Destroy Infrastructure

Remove all managed infrastructure:

```bash
terraform -chdir=terraform destroy
```

## Configuration

### Basic Terraform Structure

A typical `main.tf` file for data engineering:

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
  region = var.aws_region
}

# S3 bucket for data storage
resource "aws_s3_bucket" "data_lake" {
  bucket = "my-unique-data-lake-bucket-${var.environment}"
  
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

# EC2 instance for data processing
resource "aws_instance" "data_processor" {
  ami           = var.ec2_ami_id
  instance_type = var.instance_type
  
  tags = {
    Name        = "Data Processor"
    Environment = var.environment
  }
}
```

### Variables Configuration

Create `variables.tf`:

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
  default     = "t2.micro"
}

variable "ec2_ami_id" {
  description = "AMI ID for EC2 instance"
  type        = string
}
```

### Outputs Configuration

Create `outputs.tf` to expose resource information:

```hcl
output "s3_bucket_name" {
  description = "Name of the S3 data lake bucket"
  value       = aws_s3_bucket.data_lake.id
}

output "s3_bucket_arn" {
  description = "ARN of the S3 data lake bucket"
  value       = aws_s3_bucket.data_lake.arn
}

output "ec2_instance_id" {
  description = "ID of the data processor EC2 instance"
  value       = aws_instance.data_processor.id
}

output "ec2_public_ip" {
  description = "Public IP of the data processor"
  value       = aws_instance.data_processor.public_ip
}
```

## State Management

### View State Resources

List all resources in the state:

```bash
terraform -chdir=terraform state list
```

### Inspect State in JSON

```bash
cat terraform/terraform.tfstate | jq -r '.resources[] | [.type, .name] | join(",")'
```

### Show Specific Resource

```bash
terraform -chdir=terraform state show aws_s3_bucket.data_lake
```

### Remote State Backend (S3)

For team collaboration, configure remote state:

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

## Common Patterns

### Multi-Environment Setup

Use workspaces or separate directories:

```bash
# Using workspaces
terraform workspace new production
terraform workspace new staging
terraform workspace select production
terraform apply
```

Or use separate variable files:

```bash
terraform apply -var-file="environments/production.tfvars"
terraform apply -var-file="environments/staging.tfvars"
```

### Data Lake with Lifecycle Policies

```hcl
resource "aws_s3_bucket" "data_lake" {
  bucket = "data-lake-${var.environment}"
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

### EC2 with IAM Role for S3 Access

```hcl
resource "aws_iam_role" "data_processor_role" {
  name = "data-processor-role"

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

resource "aws_iam_role_policy" "s3_access" {
  name = "s3-access"
  role = aws_iam_role.data_processor_role.id

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

resource "aws_iam_instance_profile" "data_processor_profile" {
  name = "data-processor-profile"
  role = aws_iam_role.data_processor_role.name
}

resource "aws_instance" "data_processor" {
  ami                  = var.ec2_ami_id
  instance_type        = var.instance_type
  iam_instance_profile = aws_iam_instance_profile.data_processor_profile.name

  tags = {
    Name = "Data Processor"
  }
}
```

### Security Groups for Data Processing

```hcl
resource "aws_security_group" "data_processor_sg" {
  name        = "data-processor-sg"
  description = "Security group for data processing instances"

  ingress {
    description = "SSH access"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["${var.admin_ip}/32"]
  }

  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "Data Processor Security Group"
  }
}
```

## Verification Commands

After applying infrastructure, verify resources:

### List S3 Buckets

```bash
aws s3 ls
```

### Describe EC2 Instances

```bash
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].{ID:InstanceId, Name:Tags[?Key==`Name`].Value, Type:InstanceType, State:State.Name, PublicIP:PublicIpAddress, PrivateIP:PrivateIpAddress}' \
  --output table
```

### Check S3 Bucket Details

```bash
aws s3api get-bucket-versioning --bucket YOUR_BUCKET_NAME
aws s3api get-bucket-lifecycle-configuration --bucket YOUR_BUCKET_NAME
```

## Troubleshooting

### Issue: Bucket Name Already Exists

**Error**: `BucketAlreadyExists` or `BucketAlreadyOwnedByYou`

**Solution**: S3 bucket names must be globally unique. Update the bucket name in `main.tf`:

```hcl
resource "aws_s3_bucket" "data_lake" {
  bucket = "my-unique-prefix-data-lake-${random_id.bucket_suffix.hex}"
}

resource "random_id" "bucket_suffix" {
  byte_length = 4
}
```

### Issue: State Lock Error

**Error**: `Error acquiring the state lock`

**Solution**: If using remote state with DynamoDB locking:

```bash
# Force unlock (use with caution)
terraform -chdir=terraform force-unlock LOCK_ID
```

### Issue: Invalid Credentials

**Error**: `No valid credential sources found`

**Solution**: Verify AWS credentials:

```bash
aws sts get-caller-identity
aws configure list
```

Or use environment variables:

```bash
export AWS_ACCESS_KEY_ID=$YOUR_ACCESS_KEY
export AWS_SECRET_ACCESS_KEY=$YOUR_SECRET_KEY
export AWS_DEFAULT_REGION=us-east-1
```

### Issue: Resource Already Exists

**Error**: Resource conflicts when importing existing infrastructure

**Solution**: Import existing resources into state:

```bash
terraform import aws_s3_bucket.data_lake existing-bucket-name
terraform import aws_instance.data_processor i-1234567890abcdef0
```

### Issue: Plan Shows Unwanted Changes

**Problem**: Terraform wants to modify resources unexpectedly

**Solution**: Check for drift and refresh state:

```bash
terraform refresh
terraform plan -detailed-exitcode
```

### Issue: Destroy Fails Due to Dependencies

**Error**: Resources cannot be destroyed due to dependencies

**Solution**: Remove dependencies manually or use targeted destroy:

```bash
# Empty S3 bucket before destroying
aws s3 rm s3://YOUR_BUCKET_NAME --recursive

# Targeted destroy
terraform destroy -target=aws_instance.data_processor
terraform destroy -target=aws_s3_bucket.data_lake
```

## Best Practices

1. **Always use version control** for Terraform configuration files
2. **Use remote state** with locking for team collaboration
3. **Tag all resources** with Environment, ManagedBy, and Owner tags
4. **Run `terraform plan`** before every apply
5. **Use variables** instead of hardcoded values
6. **Enable S3 versioning** for data protection
7. **Implement least-privilege IAM policies** in production
8. **Use modules** to organize reusable infrastructure components
9. **Store sensitive values** in AWS Secrets Manager or Parameter Store
10. **Always destroy** development/testing infrastructure when not in use to save costs

## Additional Resources

- [Terraform AWS Provider Documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Blog Post: Infrastructure-as-code for Data Engineers with Terraform](https://www.startdataengineering.com/post/iac-for-data-engineering-terraform)
- [Terraform Best Practices](https://www.terraform-best-practices.com/)
