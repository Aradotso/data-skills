---
name: terraform-data-engineering-iac
description: Infrastructure-as-Code patterns for data engineering using Terraform to provision AWS resources (S3, EC2, IAM)
triggers:
  - "set up data infrastructure with terraform"
  - "create AWS resources for data engineering"
  - "manage data engineering infrastructure as code"
  - "provision S3 and EC2 with terraform"
  - "deploy data engineering stack on AWS"
  - "infrastructure as code for data pipelines"
  - "terraform for data engineering workflows"
  - "automate AWS data infrastructure setup"
---

# Terraform Data Engineering IaC

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides Infrastructure-as-Code (IaC) patterns for data engineering using Terraform to provision and manage AWS resources including S3 buckets, EC2 instances, and IAM policies.

## What This Project Does

- **Automates AWS infrastructure provisioning** for data engineering workloads
- **Manages S3 buckets** for data storage and pipeline artifacts
- **Provisions EC2 instances** for compute workloads
- **Configures IAM policies** for secure resource access
- **Maintains infrastructure state** using Terraform state files
- **Enables reproducible deployments** across environments

## Prerequisites

Before using this project, ensure you have:

1. **AWS Account** with appropriate permissions
2. **Terraform** installed (v1.0+)
3. **AWS CLI** installed and configured
4. **IAM credentials** with S3, EC2, and IAM access

### Installing Prerequisites

```bash
# Install Terraform (macOS)
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Install AWS CLI (macOS)
brew install awscli

# Configure AWS CLI
aws configure
# Enter your AWS Access Key ID, Secret Access Key, region, and output format
```

## AWS IAM Setup

### Required Permissions

Your IAM user needs the following policies:
- `AmazonS3FullAccess`
- `AmazonEC2FullAccess`
- `IAMFullAccess`

**Note**: For production, use principle of least privilege and scope down permissions.

### Creating IAM User via CLI

```bash
# Create IAM user
aws iam create-user --user-name terraform-data-eng

# Create access key
aws iam create-access-key --user-name terraform-data-eng

# Attach policies
aws iam attach-user-policy \
  --user-name terraform-data-eng \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

aws iam attach-user-policy \
  --user-name terraform-data-eng \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess

aws iam attach-user-policy \
  --user-name terraform-data-eng \
  --policy-arn arn:aws:iam::aws:policy/IAMFullAccess
```

## Project Structure

```
terraform/
├── main.tf           # Main infrastructure definitions
├── variables.tf      # Variable declarations
├── outputs.tf        # Output values
└── terraform.tfstate # State file (auto-generated)
```

## Key Terraform Commands

### Initialize Terraform

```bash
# Initialize the Terraform working directory
terraform -chdir=terraform init

# Validate configuration files
terraform -chdir=terraform validate

# Format configuration files
terraform -chdir=terraform fmt
```

### Plan and Apply Infrastructure

```bash
# Preview changes
terraform -chdir=terraform plan

# Apply changes
terraform -chdir=terraform apply

# Auto-approve (use with caution)
terraform -chdir=terraform apply -auto-approve
```

### Inspect Infrastructure

```bash
# List all resources in state
terraform -chdir=terraform state list

# Show specific resource details
terraform -chdir=terraform state show aws_s3_bucket.data_bucket

# View outputs
terraform -chdir=terraform output
```

### Destroy Infrastructure

```bash
# Destroy all resources
terraform -chdir=terraform destroy

# Destroy specific resource
terraform -chdir=terraform destroy -target=aws_s3_bucket.data_bucket
```

## Configuration Examples

### Basic S3 Bucket for Data Storage

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

resource "aws_s3_bucket" "data_lake" {
  bucket = "my-unique-data-lake-bucket-${var.environment}"
  
  tags = {
    Name        = "Data Lake Bucket"
    Environment = var.environment
    Purpose     = "Data Engineering"
  }
}

resource "aws_s3_bucket_versioning" "data_lake_versioning" {
  bucket = aws_s3_bucket.data_lake.id
  
  versioning_configuration {
    status = "Enabled"
  }
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

### EC2 Instance for Data Processing

```hcl
# terraform/main.tf (continued)

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hv-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

resource "aws_instance" "data_processor" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  tags = {
    Name        = "Data Processing Instance"
    Environment = var.environment
  }

  user_data = <<-EOF
              #!/bin/bash
              apt-get update
              apt-get install -y python3-pip
              pip3 install pandas boto3
              EOF
}

resource "aws_ebs_volume" "data_volume" {
  availability_zone = aws_instance.data_processor.availability_zone
  size              = 100
  type              = "gp3"

  tags = {
    Name = "Data Processing Volume"
  }
}

resource "aws_volume_attachment" "data_volume_attachment" {
  device_name = "/dev/sdh"
  volume_id   = aws_ebs_volume.data_volume.id
  instance_id = aws_instance.data_processor.id
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
  description = "Prefix for S3 bucket names"
  type        = string
  default     = "data-eng"
}
```

### Outputs Configuration

```hcl
# terraform/outputs.tf

output "bucket_name" {
  description = "Name of the data lake S3 bucket"
  value       = aws_s3_bucket.data_lake.bucket
}

output "bucket_arn" {
  description = "ARN of the data lake S3 bucket"
  value       = aws_s3_bucket.data_lake.arn
}

output "ec2_instance_id" {
  description = "ID of the EC2 data processing instance"
  value       = aws_instance.data_processor.id
}

output "ec2_public_ip" {
  description = "Public IP of the EC2 instance"
  value       = aws_instance.data_processor.public_ip
}
```

## Common Patterns

### Multi-Environment Setup

```hcl
# terraform/environments/dev.tfvars
environment   = "dev"
instance_type = "t3.small"
aws_region    = "us-east-1"

# terraform/environments/prod.tfvars
environment   = "prod"
instance_type = "t3.large"
aws_region    = "us-west-2"
```

Apply with specific environment:

```bash
terraform -chdir=terraform apply -var-file=environments/dev.tfvars
```

### IAM Role for EC2 to Access S3

```hcl
resource "aws_iam_role" "ec2_s3_access_role" {
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

resource "aws_iam_role_policy" "ec2_s3_policy" {
  name = "ec2-s3-policy"
  role = aws_iam_role.ec2_s3_access_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:ListBucket"
        ]
        Effect = "Allow"
        Resource = [
          aws_s3_bucket.data_lake.arn,
          "${aws_s3_bucket.data_lake.arn}/*"
        ]
      }
    ]
  })
}

resource "aws_iam_instance_profile" "ec2_profile" {
  name = "ec2-s3-access-profile"
  role = aws_iam_role.ec2_s3_access_role.name
}

# Attach to EC2 instance
resource "aws_instance" "data_processor" {
  ami                  = data.aws_ami.ubuntu.id
  instance_type        = var.instance_type
  iam_instance_profile = aws_iam_instance_profile.ec2_profile.name
  
  # ... other configuration
}
```

### Remote State Backend

```hcl
# terraform/backend.tf
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

Create backend resources:

```bash
# Create S3 bucket for state
aws s3 mb s3://my-terraform-state-bucket --region us-east-1

# Create DynamoDB table for state locking
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

## Verifying Deployed Resources

### Check S3 Buckets

```bash
# List all buckets
aws s3 ls

# List bucket contents
aws s3 ls s3://my-unique-data-lake-bucket-dev/

# Get bucket details
aws s3api get-bucket-location --bucket my-unique-data-lake-bucket-dev
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

### Inspect Terraform State

```bash
# View state as JSON
cat terraform/terraform.tfstate | jq -r '.resources[] | [.type, .name] | join(",")'

# Get specific resource from state
terraform -chdir=terraform state show aws_s3_bucket.data_lake

# Pull remote state
terraform -chdir=terraform state pull > state-backup.json
```

## Troubleshooting

### Bucket Name Already Exists

**Error**: `BucketAlreadyExists: The requested bucket name is not available`

**Solution**: S3 bucket names must be globally unique. Update your bucket name:

```hcl
resource "aws_s3_bucket" "data_lake" {
  bucket = "my-company-data-lake-${var.environment}-${random_id.bucket_suffix.hex}"
}

resource "random_id" "bucket_suffix" {
  byte_length = 4
}
```

### AWS Credentials Not Found

**Error**: `NoCredentialProviders: no valid providers in chain`

**Solution**: Ensure AWS credentials are configured:

```bash
# Check current configuration
aws configure list

# Set credentials
export AWS_ACCESS_KEY_ID="${AWS_ACCESS_KEY_ID}"
export AWS_SECRET_ACCESS_KEY="${AWS_SECRET_ACCESS_KEY}"
export AWS_DEFAULT_REGION="us-east-1"
```

### State Lock Error

**Error**: `Error acquiring the state lock`

**Solution**: Force unlock (use carefully):

```bash
# Get lock ID from error message
terraform -chdir=terraform force-unlock <LOCK_ID>
```

### Resource Already Exists

**Error**: Resource already exists but not in state

**Solution**: Import existing resource:

```bash
# Import S3 bucket
terraform -chdir=terraform import aws_s3_bucket.data_lake my-existing-bucket

# Import EC2 instance
terraform -chdir=terraform import aws_instance.data_processor i-1234567890abcdef0
```

### Permission Denied Errors

**Error**: `AccessDenied` or `UnauthorizedOperation`

**Solution**: Verify IAM permissions:

```bash
# Test S3 access
aws s3 ls

# Test EC2 access
aws ec2 describe-instances

# Check IAM user permissions
aws iam get-user
aws iam list-attached-user-policies --user-name terraform-data-eng
```

## Best Practices

1. **Never commit state files** - Add to `.gitignore`:
   ```
   terraform.tfstate
   terraform.tfstate.backup
   .terraform/
   ```

2. **Use variables** for environment-specific values:
   ```bash
   terraform -chdir=terraform apply -var="environment=prod" -var="aws_region=us-west-2"
   ```

3. **Tag all resources** for cost tracking and organization:
   ```hcl
   tags = {
     Environment = var.environment
     Project     = "data-engineering"
     ManagedBy   = "terraform"
   }
   ```

4. **Enable versioning** on S3 buckets for data protection

5. **Use remote state** for team collaboration

6. **Run `terraform plan`** before applying changes

7. **Destroy resources** when not in use to save costs:
   ```bash
   terraform -chdir=terraform destroy
   ```
