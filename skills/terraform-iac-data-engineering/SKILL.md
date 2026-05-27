---
name: terraform-iac-data-engineering
description: Infrastructure-as-Code fundamentals for data engineers using Terraform to provision AWS resources (S3, EC2, IAM)
triggers:
  - "set up terraform for data engineering"
  - "provision AWS infrastructure with terraform"
  - "create S3 buckets and EC2 instances with IaC"
  - "manage terraform state for data pipelines"
  - "deploy data engineering infrastructure as code"
  - "configure terraform for AWS data resources"
  - "initialize and apply terraform configurations"
  - "destroy terraform managed infrastructure"
---

# Terraform IaC for Data Engineering

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project demonstrates Infrastructure-as-Code (IaC) fundamentals for data engineers using Terraform to provision and manage AWS resources commonly used in data engineering workflows, including S3 buckets, EC2 instances, and IAM roles.

## What This Project Does

- Provisions AWS infrastructure declaratively using Terraform (HCL)
- Creates S3 buckets for data storage
- Deploys EC2 instances for data processing
- Manages IAM roles and policies for secure access
- Maintains infrastructure state for reproducible deployments
- Provides a foundation for scalable data engineering infrastructure

## Installation

### Prerequisites

1. **Terraform**: Install Terraform CLI
```bash
# macOS
brew install terraform

# Linux
wget https://releases.hashicorp.com/terraform/1.5.0/terraform_1.5.0_linux_amd64.zip
unzip terraform_1.5.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# Verify installation
terraform version
```

2. **AWS CLI**: Install and configure AWS CLI
```bash
# macOS
brew install awscli

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure with your credentials
aws configure
```

3. **AWS Account**: You need an AWS account with appropriate IAM permissions

### IAM Permissions Setup

Create an IAM user with the following permissions (for development/learning):
- Full S3 access
- Full EC2 access
- Full IAM access

**Note**: These broad permissions are for learning purposes only. In production, use least-privilege IAM policies.

## Key Terraform Commands

### Initialize Terraform
```bash
# Initialize terraform working directory
terraform -chdir=terraform init

# This downloads required providers and sets up the backend
```

### Validate Configuration
```bash
# Check syntax and validate configuration
terraform -chdir=terraform validate

# Format HCL files to canonical style
terraform -chdir=terraform fmt
```

### Plan Infrastructure Changes
```bash
# Preview changes without applying
terraform -chdir=terraform plan

# Save plan to file for review
terraform -chdir=terraform plan -out=tfplan
```

### Apply Infrastructure
```bash
# Apply changes interactively
terraform -chdir=terraform apply

# Apply without confirmation prompt
terraform -chdir=terraform apply -auto-approve

# Apply from saved plan
terraform -chdir=terraform apply tfplan
```

### Inspect State
```bash
# List all resources in state
terraform -chdir=terraform state list

# Show details of specific resource
terraform -chdir=terraform state show aws_s3_bucket.data_bucket

# View state file (JSON)
cat terraform/terraform.tfstate | jq -r '.resources[] | [.type, .name] | join(",")'
```

### Destroy Infrastructure
```bash
# Destroy all managed infrastructure
terraform -chdir=terraform destroy

# Destroy without confirmation
terraform -chdir=terraform destroy -auto-approve

# Destroy specific resource
terraform -chdir=terraform destroy -target=aws_instance.data_processor
```

## Configuration

### Basic Terraform Configuration Structure

**terraform/main.tf**:
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
    Name        = "Data Lake Bucket"
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# S3 bucket versioning
resource "aws_s3_bucket_versioning" "data_lake_versioning" {
  bucket = aws_s3_bucket.data_lake.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# EC2 instance for data processing
resource "aws_instance" "data_processor" {
  ami           = var.ec2_ami_id
  instance_type = "t3.medium"
  
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

# IAM role for EC2 to access S3
resource "aws_iam_role" "ec2_s3_access" {
  name = "ec2-s3-access-role"
  
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

resource "aws_iam_role_policy" "s3_access_policy" {
  name = "s3-access-policy"
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
```

**terraform/variables.tf**:
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

variable "ec2_ami_id" {
  description = "AMI ID for EC2 instance"
  type        = string
  default     = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2
}

variable "s3_bucket_prefix" {
  description = "Prefix for S3 bucket names"
  type        = string
  default     = "data-engineering"
}
```

**terraform/outputs.tf**:
```hcl
output "s3_bucket_name" {
  description = "Name of the data lake S3 bucket"
  value       = aws_s3_bucket.data_lake.id
}

output "s3_bucket_arn" {
  description = "ARN of the data lake S3 bucket"
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

## Common Patterns

### Multiple Environment Setup

**terraform/environments/dev.tfvars**:
```hcl
aws_region        = "us-east-1"
environment       = "dev"
ec2_ami_id        = "ami-0c55b159cbfafe1f0"
s3_bucket_prefix  = "dev-data-engineering"
```

**terraform/environments/prod.tfvars**:
```hcl
aws_region        = "us-west-2"
environment       = "prod"
ec2_ami_id        = "ami-0c55b159cbfafe1f0"
s3_bucket_prefix  = "prod-data-engineering"
```

Apply with environment-specific variables:
```bash
terraform -chdir=terraform apply -var-file=environments/dev.tfvars
terraform -chdir=terraform apply -var-file=environments/prod.tfvars
```

### Data Pipeline Infrastructure

```hcl
# Raw data ingestion bucket
resource "aws_s3_bucket" "raw_data" {
  bucket = "${var.s3_bucket_prefix}-raw-${var.environment}"
  
  tags = {
    Layer = "Raw"
  }
}

# Processed data bucket
resource "aws_s3_bucket" "processed_data" {
  bucket = "${var.s3_bucket_prefix}-processed-${var.environment}"
  
  tags = {
    Layer = "Processed"
  }
}

# Analytics-ready data bucket
resource "aws_s3_bucket" "analytics_data" {
  bucket = "${var.s3_bucket_prefix}-analytics-${var.environment}"
  
  tags = {
    Layer = "Analytics"
  }
}

# Lifecycle policy for raw data
resource "aws_s3_bucket_lifecycle_configuration" "raw_data_lifecycle" {
  bucket = aws_s3_bucket.raw_data.id
  
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

### EC2 Instance with Data Processing Tools

```hcl
resource "aws_instance" "airflow_server" {
  ami           = var.ec2_ami_id
  instance_type = "t3.large"
  
  iam_instance_profile = aws_iam_instance_profile.data_processor_profile.name
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y docker
              systemctl start docker
              systemctl enable docker
              
              # Install Docker Compose
              curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
              chmod +x /usr/local/bin/docker-compose
              
              # Clone and start Airflow
              mkdir -p /opt/airflow
              cd /opt/airflow
              # Add your Airflow setup here
              EOF
  
  tags = {
    Name = "Airflow Server"
    Tool = "Airflow"
  }
}

resource "aws_iam_instance_profile" "data_processor_profile" {
  name = "data-processor-profile"
  role = aws_iam_role.ec2_s3_access.name
}
```

### Remote State Management

**terraform/backend.tf**:
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

Create state backend resources:
```hcl
# In a separate bootstrap configuration
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state-bucket"
  
  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "terraform_state_versioning" {
  bucket = aws_s3_bucket.terraform_state.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

## Verification Commands

After applying infrastructure, verify resources:

```bash
# List S3 buckets
aws s3 ls

# Describe running EC2 instances
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].{ID:InstanceId, Name:Tags[?Key==`Name`].Value, Type:InstanceType, State:State.Name, PublicIP:PublicIpAddress, PrivateIP:PrivateIpAddress}' \
  --output table

# Check S3 bucket details
aws s3api get-bucket-versioning --bucket my-unique-data-lake-bucket-dev

# List IAM roles
aws iam list-roles --query 'Roles[?contains(RoleName, `ec2-s3`)].RoleName'
```

## Troubleshooting

### Common Issues

**1. Bucket Name Already Exists**
```
Error: Error creating S3 bucket: BucketAlreadyExists
```
Solution: S3 bucket names must be globally unique. Change the bucket name in your configuration:
```hcl
resource "aws_s3_bucket" "data_lake" {
  bucket = "my-company-data-lake-${random_id.bucket_suffix.hex}"
}

resource "random_id" "bucket_suffix" {
  byte_length = 4
}
```

**2. Insufficient IAM Permissions**
```
Error: Error creating EC2 instance: UnauthorizedOperation
```
Solution: Ensure your IAM user/role has the required permissions. Check with:
```bash
aws iam get-user
aws sts get-caller-identity
```

**3. State Lock Timeout**
```
Error: Error acquiring the state lock
```
Solution: Another process may be running. If stuck, manually release the lock:
```bash
terraform -chdir=terraform force-unlock <LOCK_ID>
```

**4. Invalid AMI ID**
```
Error: Invalid id: "ami-xxxxx" (expecting "ami-...")
```
Solution: AMI IDs are region-specific. Find the correct AMI for your region:
```bash
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text
```

**5. Terraform State Corruption**
Solution: Restore from backup or rebuild state:
```bash
# List backups
ls terraform/terraform.tfstate.backup

# Restore from backup
cp terraform/terraform.tfstate.backup terraform/terraform.tfstate

# Or rebuild state by importing existing resources
terraform -chdir=terraform import aws_s3_bucket.data_lake my-bucket-name
```

### Debugging Terraform

Enable detailed logging:
```bash
export TF_LOG=DEBUG
export TF_LOG_PATH=./terraform-debug.log
terraform -chdir=terraform apply
```

Inspect specific resource state:
```bash
# Show resource details
terraform -chdir=terraform state show aws_instance.data_processor

# Refresh state from actual infrastructure
terraform -chdir=terraform refresh
```

### Best Practices

1. **Always use version control** for Terraform configurations
2. **Use remote state** for team collaboration
3. **Implement state locking** to prevent concurrent modifications
4. **Tag all resources** for cost tracking and organization
5. **Use variables** instead of hardcoded values
6. **Run `terraform plan`** before every apply
7. **Use modules** for reusable infrastructure components
8. **Implement least-privilege IAM policies** in production
9. **Store sensitive values** in AWS Secrets Manager or Parameter Store
10. **Regularly update** Terraform and provider versions

### Cost Management

Monitor costs after deployment:
```bash
# List all resources with tags
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=ManagedBy,Values=Terraform \
  --query 'ResourceTagMappingList[].ResourceARN'

# Estimate costs (use AWS Cost Explorer or third-party tools)
```

Always destroy unused infrastructure:
```bash
terraform -chdir=terraform destroy
```
