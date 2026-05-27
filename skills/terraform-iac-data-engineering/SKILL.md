---
name: terraform-iac-data-engineering
description: Infrastructure-as-Code fundamentals for data engineering using Terraform to provision AWS resources (S3, EC2, IAM)
triggers:
  - "set up infrastructure as code for data engineering"
  - "create terraform config for data pipelines"
  - "provision aws resources with terraform"
  - "manage data engineering infrastructure"
  - "deploy s3 and ec2 with terraform"
  - "infrastructure as code best practices"
  - "terraform for data platforms"
  - "automate aws data infrastructure"
---

# Terraform IaC for Data Engineering

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project teaches Infrastructure-as-Code (IaC) fundamentals for data engineers using Terraform to provision and manage AWS resources. It demonstrates how to create S3 buckets, EC2 instances, and IAM roles/policies programmatically, following best practices for reproducible data infrastructure.

## What This Project Does

- Provisions AWS infrastructure (S3, EC2, IAM) using declarative Terraform configurations
- Demonstrates state management for tracking infrastructure changes
- Shows how to validate, format, and apply infrastructure changes
- Provides patterns for destroying and recreating environments consistently
- Teaches data engineers how to manage infrastructure alongside data pipelines

## Installation & Prerequisites

### Install Required Tools

```bash
# Install Terraform (macOS)
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Install AWS CLI (macOS)
brew install awscli

# Verify installations
terraform --version
aws --version
```

### AWS Account Setup

1. Create an AWS account with root access
2. Configure AWS CLI with credentials:

```bash
aws configure
# Enter:
# - AWS Access Key ID: [from IAM user]
# - AWS Secret Access Key: [from IAM user]
# - Default region: us-east-1 (or your preferred region)
# - Default output format: json
```

### IAM Permissions

Create an IAM user with the following permissions (development only):
- Full S3 access
- Full EC2 access
- Full IAM access

**Note:** For production, use least-privilege policies and role-based access.

## Project Structure

```
terraform/
├── main.tf           # Main infrastructure configuration
├── variables.tf      # Variable definitions
├── outputs.tf        # Output values
└── terraform.tfstate # State file (auto-generated)
```

## Key Terraform Commands

### Initialize Terraform

```bash
# Initialize the working directory
terraform -chdir=terraform init

# This downloads provider plugins and sets up the backend
```

### Validate & Format

```bash
# Validate configuration syntax
terraform -chdir=terraform validate

# Format configuration files
terraform -chdir=terraform fmt

# Check formatting without making changes
terraform -chdir=terraform fmt -check
```

### Plan & Apply

```bash
# Preview changes
terraform -chdir=terraform plan

# Apply changes (create/update infrastructure)
terraform -chdir=terraform apply

# Auto-approve without confirmation (use cautiously)
terraform -chdir=terraform apply -auto-approve
```

### Inspect State

```bash
# List all resources in state
terraform -chdir=terraform state list

# Show detailed resource information
terraform -chdir=terraform state show aws_s3_bucket.data_bucket

# View state file
cat terraform/terraform.tfstate | jq -r '.resources[] | [.type, .name] | join(",")'
```

### Destroy Infrastructure

```bash
# Destroy all managed infrastructure
terraform -chdir=terraform destroy

# Destroy specific resource
terraform -chdir=terraform destroy -target=aws_instance.data_processor
```

## Configuration Examples

### Basic S3 Bucket

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

resource "aws_s3_bucket" "data_lake" {
  bucket = "my-unique-data-lake-bucket-${random_id.suffix.hex}"
  
  tags = {
    Name        = "Data Lake"
    Environment = "Dev"
    ManagedBy   = "Terraform"
  }
}

resource "random_id" "suffix" {
  byte_length = 4
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
# terraform/main.tf
resource "aws_instance" "data_processor" {
  ami           = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2
  instance_type = "t3.medium"

  tags = {
    Name        = "DataProcessor"
    Environment = "Dev"
    ManagedBy   = "Terraform"
  }

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y python3 python3-pip
              pip3 install pandas boto3 sqlalchemy
              EOF
}

output "instance_public_ip" {
  value       = aws_instance.data_processor.public_ip
  description = "Public IP of the data processor instance"
}
```

### IAM Role for EC2 to Access S3

```hcl
# terraform/main.tf
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

resource "aws_iam_role_policy" "s3_access" {
  name = "s3-access-policy"
  role = aws_iam_role.ec2_s3_access.id

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
          aws_s3_bucket.data_lake.arn,
          "${aws_s3_bucket.data_lake.arn}/*"
        ]
      }
    ]
  })
}

resource "aws_iam_instance_profile" "ec2_profile" {
  name = "ec2-s3-profile"
  role = aws_iam_role.ec2_s3_access.name
}

# Attach profile to EC2 instance
resource "aws_instance" "data_processor_with_role" {
  ami                  = "ami-0c55b159cbfafe1f0"
  instance_type        = "t3.medium"
  iam_instance_profile = aws_iam_instance_profile.ec2_profile.name

  tags = {
    Name = "DataProcessorWithS3Access"
  }
}
```

## Common Patterns

### Using Variables

```hcl
# terraform/variables.tf
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

variable "data_bucket_name" {
  description = "Name of the S3 data bucket"
  type        = string
}

# terraform/main.tf
resource "aws_s3_bucket" "data_bucket" {
  bucket = var.data_bucket_name

  tags = {
    Environment = var.environment
  }
}

resource "aws_instance" "processor" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type

  tags = {
    Environment = var.environment
  }
}

data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
```

Apply with variables:

```bash
# Using command line
terraform -chdir=terraform apply -var="data_bucket_name=my-unique-bucket"

# Using tfvars file
# terraform/terraform.tfvars
environment       = "dev"
instance_type     = "t3.large"
data_bucket_name  = "my-data-lake-dev"

terraform -chdir=terraform apply
```

### Outputs for Integration

```hcl
# terraform/outputs.tf
output "s3_bucket_name" {
  value       = aws_s3_bucket.data_lake.id
  description = "Name of the S3 data lake bucket"
}

output "s3_bucket_arn" {
  value       = aws_s3_bucket.data_lake.arn
  description = "ARN of the S3 data lake bucket"
}

output "ec2_instance_id" {
  value       = aws_instance.data_processor.id
  description = "ID of the EC2 data processor instance"
}

output "ec2_public_ip" {
  value       = aws_instance.data_processor.public_ip
  description = "Public IP of the EC2 instance"
  sensitive   = false
}
```

Query outputs:

```bash
# View all outputs
terraform -chdir=terraform output

# Get specific output
terraform -chdir=terraform output s3_bucket_name

# Get JSON format for scripting
terraform -chdir=terraform output -json | jq -r '.s3_bucket_name.value'
```

### Multiple Environments

```hcl
# terraform/environments/dev/main.tf
module "data_infrastructure" {
  source = "../../modules/data-infra"

  environment       = "dev"
  instance_type     = "t3.small"
  data_bucket_name  = "data-lake-dev"
}

# terraform/environments/prod/main.tf
module "data_infrastructure" {
  source = "../../modules/data-infra"

  environment       = "prod"
  instance_type     = "t3.xlarge"
  data_bucket_name  = "data-lake-prod"
}
```

## Verification Commands

### Verify S3 Buckets

```bash
# List all S3 buckets
aws s3 ls

# Check bucket contents
aws s3 ls s3://your-bucket-name/

# Test upload
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

### Verify IAM Resources

```bash
# List IAM roles
aws iam list-roles --query 'Roles[?contains(RoleName, `ec2`)].RoleName'

# Get role policies
aws iam list-role-policies --role-name ec2-s3-access-role
```

## Workflow Example

Complete workflow for provisioning data infrastructure:

```bash
# 1. Clone or create terraform configuration
mkdir -p terraform
cd terraform

# 2. Create main.tf with your infrastructure
cat > main.tf <<'EOF'
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
  bucket = "data-lake-${var.environment}-${random_id.suffix.hex}"
  
  tags = {
    Name        = "DataLake"
    Environment = var.environment
  }
}

resource "random_id" "suffix" {
  byte_length = 4
}

variable "aws_region" {
  default = "us-east-1"
}

variable "environment" {
  default = "dev"
}

output "bucket_name" {
  value = aws_s3_bucket.data_lake.id
}
EOF

# 3. Initialize
terraform init

# 4. Validate and format
terraform validate
terraform fmt

# 5. Plan changes
terraform plan -out=tfplan

# 6. Review plan and apply
terraform apply tfplan

# 7. Verify infrastructure
BUCKET_NAME=$(terraform output -raw bucket_name)
aws s3 ls s3://$BUCKET_NAME

# 8. When done, destroy
terraform destroy
```

## Troubleshooting

### State Lock Issues

```bash
# Force unlock if state is stuck
terraform -chdir=terraform force-unlock LOCK_ID

# Remove corrupted state backup
rm terraform/terraform.tfstate.backup
```

### Import Existing Resources

```bash
# Import existing S3 bucket
terraform -chdir=terraform import aws_s3_bucket.data_lake existing-bucket-name

# Import EC2 instance
terraform -chdir=terraform import aws_instance.data_processor i-1234567890abcdef0
```

### Refresh State

```bash
# Sync state with real infrastructure
terraform -chdir=terraform refresh

# Replace deprecated refresh
terraform -chdir=terraform apply -refresh-only
```

### Debug Mode

```bash
# Enable detailed logging
export TF_LOG=DEBUG
terraform -chdir=terraform apply

# Log to file
export TF_LOG=TRACE
export TF_LOG_PATH=terraform.log
terraform -chdir=terraform apply
```

### Resource Dependencies

```bash
# Visualize dependency graph
terraform -chdir=terraform graph | dot -Tpng > graph.png

# Target specific resource
terraform -chdir=terraform apply -target=aws_s3_bucket.data_lake
```

### Common Errors

**Bucket name already exists:**
```hcl
# Use random suffix
resource "random_id" "bucket_suffix" {
  byte_length = 8
}

resource "aws_s3_bucket" "data_lake" {
  bucket = "my-data-lake-${random_id.bucket_suffix.hex}"
}
```

**AWS credentials not found:**
```bash
# Check AWS configuration
aws configure list

# Set credentials via environment
export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
export AWS_DEFAULT_REGION=us-east-1
```

**Provider version conflicts:**
```bash
# Upgrade providers
terraform -chdir=terraform init -upgrade

# Lock specific version
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "= 5.31.0"
    }
  }
}
```

This skill enables AI agents to help developers provision and manage AWS data infrastructure using Terraform, following IaC best practices for data engineering workflows.
