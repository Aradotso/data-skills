---
name: iac-terraform-data-engineering
description: Infrastructure-as-Code with Terraform for data engineering projects on AWS
triggers:
  - set up terraform for data engineering
  - create aws infrastructure with terraform
  - deploy data engineering resources using iac
  - configure s3 and ec2 with terraform
  - manage terraform state for data pipelines
  - provision aws resources for data engineering
  - terraform infrastructure for data projects
  - destroy terraform data engineering infrastructure
---

# Infrastructure-as-Code with Terraform for Data Engineering

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides Terraform configurations for provisioning AWS infrastructure commonly used in data engineering workflows, including S3 buckets and EC2 instances. It demonstrates infrastructure-as-code (IaC) principles for managing cloud resources declaratively.

## What It Does

- Provisions AWS S3 buckets for data storage
- Creates EC2 instances for data processing workloads
- Manages IAM roles and policies for resource access
- Maintains infrastructure state using Terraform state files
- Provides reproducible infrastructure deployments

## Prerequisites

Before using this project, ensure you have:

1. AWS Account with appropriate permissions
2. Terraform CLI installed
3. AWS CLI installed and configured
4. IAM user with S3, EC2, and IAM full access

## Setup

### Install Required Tools

```bash
# Install Terraform (macOS example)
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Install AWS CLI
brew install awscli

# Configure AWS CLI with credentials
aws configure
```

### IAM Permissions Setup

Create an IAM user with the following permissions (via inline policy):
- Full S3 access
- Full EC2 access
- Full IAM access

**Note**: These broad permissions are for development/learning. Production environments should use least-privilege principles.

## Basic Project Structure

```
terraform/
├── main.tf           # Main infrastructure definitions
├── variables.tf      # Input variables (if present)
├── outputs.tf        # Output values (if present)
└── terraform.tfstate # State file (generated)
```

## Key Commands

### Initialize Terraform

```bash
# Initialize working directory and download providers
terraform -chdir=terraform init
```

### Validate Configuration

```bash
# Check syntax and validate configuration
terraform -chdir=terraform validate
```

### Format Configuration

```bash
# Auto-format HCL files to canonical style
terraform -chdir=terraform fmt
```

### Plan Changes

```bash
# Preview infrastructure changes without applying
terraform -chdir=terraform plan
```

### Apply Configuration

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

# Auto-approve destruction (use with caution)
terraform -chdir=terraform destroy -auto-approve
```

### Inspect State

```bash
# List all resources in state
terraform -chdir=terraform state list

# Show details of specific resource
terraform -chdir=terraform state show <resource_name>

# View state file directly
cat terraform/terraform.tfstate | jq -r '.resources[] | [.type, .name] | join(",")'
```

## Configuration Examples

### Basic S3 Bucket

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
  region = "us-east-1"
}

resource "aws_s3_bucket" "data_lake" {
  bucket = "my-unique-data-lake-bucket-12345"
  
  tags = {
    Name        = "Data Lake"
    Environment = "Development"
    ManagedBy   = "Terraform"
  }
}

resource "aws_s3_bucket_versioning" "data_lake_versioning" {
  bucket = aws_s3_bucket.data_lake.id
  
  versioning_configuration {
    status = "Enabled"
  }
}
```

### EC2 Instance for Data Processing

```hcl
# Get latest Amazon Linux 2 AMI
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
  instance_type = "t3.medium"

  tags = {
    Name        = "Data Processor"
    Environment = "Development"
    ManagedBy   = "Terraform"
  }

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y python3 python3-pip
              pip3 install pandas boto3
              EOF
}
```

### IAM Role for EC2 with S3 Access

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

resource "aws_iam_role_policy_attachment" "s3_full_access" {
  role       = aws_iam_role.ec2_s3_access.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
}

resource "aws_iam_instance_profile" "ec2_profile" {
  name = "ec2-s3-profile"
  role = aws_iam_role.ec2_s3_access.name
}
```

### Using Variables

```hcl
# variables.tf
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "development"
}

variable "bucket_prefix" {
  description = "Prefix for S3 bucket name"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.medium"
}

# main.tf
resource "aws_s3_bucket" "data_lake" {
  bucket = "${var.bucket_prefix}-${var.environment}-data-lake"
  
  tags = {
    Environment = var.environment
  }
}

# Apply with variables
# terraform -chdir=terraform apply -var="bucket_prefix=mycompany" -var="environment=staging"
```

### Outputs

```hcl
# outputs.tf
output "s3_bucket_name" {
  description = "Name of the created S3 bucket"
  value       = aws_s3_bucket.data_lake.id
}

output "s3_bucket_arn" {
  description = "ARN of the created S3 bucket"
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

## Common Patterns

### Remote State Backend

Store state in S3 for team collaboration:

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

### Module Organization

```hcl
# modules/s3_bucket/main.tf
variable "bucket_name" {
  type = string
}

variable "environment" {
  type = string
}

resource "aws_s3_bucket" "bucket" {
  bucket = var.bucket_name
  
  tags = {
    Environment = var.environment
  }
}

output "bucket_id" {
  value = aws_s3_bucket.bucket.id
}

# main.tf
module "raw_data_bucket" {
  source      = "./modules/s3_bucket"
  bucket_name = "my-raw-data-bucket"
  environment = "production"
}

module "processed_data_bucket" {
  source      = "./modules/s3_bucket"
  bucket_name = "my-processed-data-bucket"
  environment = "production"
}
```

### Data Engineering Stack

```hcl
# Complete data engineering infrastructure
resource "aws_s3_bucket" "raw_data" {
  bucket = "company-raw-data-${random_id.suffix.hex}"
}

resource "aws_s3_bucket" "processed_data" {
  bucket = "company-processed-data-${random_id.suffix.hex}"
}

resource "aws_s3_bucket" "logs" {
  bucket = "company-logs-${random_id.suffix.hex}"
}

resource "random_id" "suffix" {
  byte_length = 4
}

resource "aws_instance" "airflow_server" {
  ami                    = data.aws_ami.amazon_linux_2.id
  instance_type          = "t3.large"
  iam_instance_profile   = aws_iam_instance_profile.ec2_profile.name
  
  tags = {
    Name = "Airflow Server"
    Role = "Orchestration"
  }
}

resource "aws_instance" "spark_master" {
  ami                    = data.aws_ami.amazon_linux_2.id
  instance_type          = "t3.xlarge"
  iam_instance_profile   = aws_iam_instance_profile.ec2_profile.name
  
  tags = {
    Name = "Spark Master"
    Role = "Processing"
  }
}
```

## Verification Commands

### Check S3 Buckets

```bash
# List all S3 buckets
aws s3 ls

# List contents of specific bucket
aws s3 ls s3://my-bucket-name/

# Get bucket details
aws s3api head-bucket --bucket my-bucket-name
```

### Check EC2 Instances

```bash
# List running instances in table format
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].{ID:InstanceId, Name:Tags[?Key==`Name`].Value, Type:InstanceType, State:State.Name, PublicIP:PublicIpAddress, PrivateIP:PrivateIpAddress}' \
  --output table

# Get instance details by ID
aws ec2 describe-instances --instance-ids i-1234567890abcdef0
```

## Troubleshooting

### Bucket Name Already Exists

S3 bucket names must be globally unique. Modify your bucket name:

```hcl
resource "aws_s3_bucket" "data_lake" {
  bucket = "my-company-data-lake-${random_id.bucket_suffix.hex}"
}

resource "random_id" "bucket_suffix" {
  byte_length = 8
}
```

### State Lock Issues

If apply/destroy hangs due to state lock:

```bash
# Force unlock (use with extreme caution)
terraform -chdir=terraform force-unlock <LOCK_ID>
```

### Permission Denied Errors

Verify your AWS credentials and IAM permissions:

```bash
# Check current AWS identity
aws sts get-caller-identity

# Verify credentials are configured
aws configure list
```

### Resource Already Exists

Import existing resource into state:

```bash
terraform -chdir=terraform import aws_s3_bucket.data_lake my-existing-bucket-name
```

### Destroy Failures

If destroy fails due to dependencies:

```bash
# Target specific resource for destruction
terraform -chdir=terraform destroy -target=aws_instance.data_processor

# Remove resource from state without destroying
terraform -chdir=terraform state rm aws_s3_bucket.data_lake
```

### Drift Detection

Check if infrastructure has drifted from state:

```bash
# Show differences between state and actual infrastructure
terraform -chdir=terraform plan -refresh-only
terraform -chdir=terraform apply -refresh-only
```

## Best Practices

1. **Unique Bucket Names**: Always use unique prefixes or random suffixes for S3 buckets
2. **Tag Resources**: Add consistent tags for cost tracking and management
3. **State Management**: Use remote state for team collaboration
4. **Least Privilege**: Grant minimal IAM permissions needed (not full access in production)
5. **Version Control**: Store Terraform configurations in Git
6. **Environment Separation**: Use workspaces or separate state files per environment
7. **Cost Awareness**: Always run `terraform destroy` when done with development resources
