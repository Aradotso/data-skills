---
name: terraform-iac-data-engineering
description: Infrastructure-as-Code with Terraform for data engineering - provision AWS S3 buckets, EC2 instances, and IAM resources for data pipelines
triggers:
  - "set up terraform for data engineering"
  - "create AWS infrastructure with terraform"
  - "provision S3 and EC2 using terraform"
  - "infrastructure as code for data pipelines"
  - "terraform state management and AWS resources"
  - "deploy data engineering infrastructure on AWS"
  - "manage terraform resources for data platforms"
  - "terraform IAC patterns for data engineering"
---

# Terraform IAC for Data Engineering

> Skill by [ara.so](https://ara.so) — Data Skills collection.

This project provides Infrastructure-as-Code (IaC) patterns using Terraform to provision AWS resources commonly used in data engineering: S3 buckets for data lakes, EC2 instances for compute workloads, and IAM roles for secure access management.

## Prerequisites

Before using this project, ensure you have:

1. AWS Account with appropriate permissions
2. Terraform CLI installed (`brew install terraform` or from [official site](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli))
3. AWS CLI installed and configured
4. IAM user with S3, EC2, and IAM full access (for dev/test - restrict in production)

## AWS CLI Setup

Configure AWS credentials:

```bash
aws configure
# Enter AWS Access Key ID
# Enter AWS Secret Access Key
# Enter Default region name (e.g., us-east-1)
# Enter Default output format (json)
```

Verify configuration:

```bash
aws sts get-caller-identity
```

## Project Structure

```
.
├── terraform/
│   ├── main.tf           # Main Terraform configuration
│   ├── variables.tf      # Input variables (if present)
│   ├── outputs.tf        # Output values (if present)
│   └── terraform.tfstate # State file (generated after apply)
```

## Basic Terraform Configuration

### main.tf Structure

A typical `main.tf` for data engineering infrastructure:

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

# S3 bucket for data lake
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

# Block public access
resource "aws_s3_bucket_public_access_block" "data_lake_public_access" {
  bucket = aws_s3_bucket.data_lake.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# EC2 instance for data processing
resource "aws_instance" "data_processor" {
  ami           = var.ami_id # Amazon Linux 2 or Ubuntu AMI
  instance_type = var.instance_type

  tags = {
    Name        = "DataProcessor"
    Environment = var.environment
    ManagedBy   = "Terraform"
  }

  # User data script for initialization
  user_data = <<-EOF
              #!/bin/bash
              sudo yum update -y
              sudo yum install -y python3 python3-pip
              pip3 install awscli boto3 pandas
              EOF
}

# IAM role for EC2 to access S3
resource "aws_iam_role" "ec2_s3_access" {
  name = "ec2-s3-access-role-${var.environment}"

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

# IAM instance profile
resource "aws_iam_instance_profile" "ec2_profile" {
  name = "ec2-profile-${var.environment}"
  role = aws_iam_role.ec2_s3_access.name
}
```

### variables.tf

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

variable "ami_id" {
  description = "AMI ID for EC2 instance"
  type        = string
  # Amazon Linux 2 AMI (varies by region)
  default     = "ami-0c55b159cbfafe1f0"
}
```

### outputs.tf

```hcl
output "s3_bucket_name" {
  description = "Name of the S3 data lake bucket"
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

output "ec2_private_ip" {
  description = "Private IP of the EC2 instance"
  value       = aws_instance.data_processor.private_ip
}
```

## Core Terraform Workflow

### 1. Initialize Terraform

Initialize the working directory and download provider plugins:

```bash
terraform -chdir=terraform init
```

This creates `.terraform/` directory and downloads AWS provider.

### 2. Validate Configuration

Check for syntax errors:

```bash
terraform -chdir=terraform validate
```

### 3. Format Code

Format Terraform files to canonical style:

```bash
terraform -chdir=terraform fmt
```

Or format recursively:

```bash
terraform -chdir=terraform fmt -recursive
```

### 4. Plan Changes

Preview infrastructure changes before applying:

```bash
terraform -chdir=terraform plan
```

Save plan to file for review:

```bash
terraform -chdir=terraform plan -out=tfplan
```

### 5. Apply Configuration

Create or update infrastructure:

```bash
terraform -chdir=terraform apply
```

Auto-approve without confirmation (use with caution):

```bash
terraform -chdir=terraform apply -auto-approve
```

Apply saved plan:

```bash
terraform -chdir=terraform apply tfplan
```

### 6. Destroy Infrastructure

Remove all resources managed by Terraform:

```bash
terraform -chdir=terraform destroy
```

Destroy specific resource:

```bash
terraform -chdir=terraform destroy -target=aws_instance.data_processor
```

## State Management

### View State

List all resources in state:

```bash
terraform -chdir=terraform state list
```

Show details of specific resource:

```bash
terraform -chdir=terraform state show aws_s3_bucket.data_lake
```

### Inspect State File

View state as JSON:

```bash
cat terraform/terraform.tfstate | jq '.'
```

List resource types and names:

```bash
cat terraform/terraform.tfstate | jq -r '.resources[] | [.type, .name] | join(",")'
```

### Remote State (Production Pattern)

For team collaboration, use remote state:

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

## Common Data Engineering Patterns

### Pattern 1: Multi-Environment Setup

Use workspaces or separate directories:

```bash
# Using workspaces
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# Switch workspaces
terraform workspace select dev
terraform apply -var="environment=dev"
```

### Pattern 2: Data Lake with Lifecycle Policies

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

    transition {
      days          = 180
      storage_class = "DEEP_ARCHIVE"
    }

    expiration {
      days = 365
    }
  }

  rule {
    id     = "delete-incomplete-uploads"
    status = "Enabled"

    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }
}
```

### Pattern 3: VPC for Secure Data Infrastructure

```hcl
resource "aws_vpc" "data_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "data-engineering-vpc"
  }
}

resource "aws_subnet" "private_subnet" {
  vpc_id            = aws_vpc.data_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "${var.aws_region}a"

  tags = {
    Name = "private-subnet"
  }
}

resource "aws_security_group" "data_processor_sg" {
  name        = "data-processor-sg"
  description = "Security group for data processing instances"
  vpc_id      = aws_vpc.data_vpc.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }
}
```

### Pattern 4: RDS Database for Metadata

```hcl
resource "aws_db_instance" "metadata_db" {
  identifier        = "metadata-db-${var.environment}"
  engine            = "postgres"
  engine_version    = "14.7"
  instance_class    = "db.t3.micro"
  allocated_storage = 20
  storage_encrypted = true

  db_name  = "metadata"
  username = "admin"
  password = var.db_password # Use AWS Secrets Manager in production

  vpc_security_group_ids = [aws_security_group.db_sg.id]
  db_subnet_group_name   = aws_db_subnet_group.db_subnet.name

  backup_retention_period = 7
  skip_final_snapshot     = var.environment == "dev" ? true : false

  tags = {
    Name        = "metadata-database"
    Environment = var.environment
  }
}
```

### Pattern 5: EMR Cluster for Big Data Processing

```hcl
resource "aws_emr_cluster" "data_pipeline" {
  name          = "data-pipeline-emr-${var.environment}"
  release_label = "emr-6.10.0"
  applications  = ["Spark", "Hadoop", "Hive"]

  ec2_attributes {
    subnet_id                         = aws_subnet.private_subnet.id
    emr_managed_master_security_group = aws_security_group.emr_master.id
    emr_managed_slave_security_group  = aws_security_group.emr_worker.id
    instance_profile                  = aws_iam_instance_profile.emr_profile.arn
  }

  master_instance_group {
    instance_type = "m5.xlarge"
  }

  core_instance_group {
    instance_type  = "m5.xlarge"
    instance_count = 2
  }

  service_role = aws_iam_role.emr_service_role.arn

  log_uri = "s3://${aws_s3_bucket.data_lake.id}/emr-logs/"
}
```

## Verification Commands

After applying infrastructure, verify resources:

### Check S3 Buckets

```bash
aws s3 ls
aws s3 ls s3://your-bucket-name/
```

### Check EC2 Instances

```bash
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].{ID:InstanceId, Name:Tags[?Key==`Name`].Value, Type:InstanceType, State:State.Name, PublicIP:PublicIpAddress, PrivateIP:PrivateIpAddress}' \
  --output table
```

### Check IAM Roles

```bash
aws iam list-roles --query 'Roles[?contains(RoleName, `ec2-s3-access`)].RoleName'
```

### Test Terraform Outputs

```bash
terraform -chdir=terraform output
terraform -chdir=terraform output s3_bucket_name
terraform -chdir=terraform output -json
```

## Troubleshooting

### Issue: "Bucket name already exists"

S3 bucket names must be globally unique. Update your bucket name:

```hcl
resource "aws_s3_bucket" "data_lake" {
  bucket = "my-company-data-lake-${random_id.bucket_suffix.hex}"
}

resource "random_id" "bucket_suffix" {
  byte_length = 4
}
```

### Issue: "Insufficient IAM permissions"

Ensure your IAM user has required permissions. Minimal policy:

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

**Note:** This is overly permissive. In production, use least-privilege policies.

### Issue: "Error acquiring state lock"

If state is locked:

```bash
terraform -chdir=terraform force-unlock <LOCK_ID>
```

### Issue: "Provider plugin not found"

Re-initialize Terraform:

```bash
rm -rf terraform/.terraform
terraform -chdir=terraform init
```

### Issue: Resource already exists (import)

Import existing resources into Terraform state:

```bash
terraform -chdir=terraform import aws_s3_bucket.data_lake my-existing-bucket-name
terraform -chdir=terraform import aws_instance.data_processor i-1234567890abcdef0
```

## Best Practices

1. **Never commit state files** - Add to `.gitignore`:
   ```
   *.tfstate
   *.tfstate.*
   .terraform/
   ```

2. **Use variables for environment-specific values**:
   ```bash
   terraform apply -var="environment=prod" -var="instance_type=t3.large"
   ```

3. **Use tfvars files**:
   ```hcl
   # terraform.tfvars
   aws_region    = "us-west-2"
   environment   = "production"
   instance_type = "t3.large"
   ```

4. **Tag all resources** for cost tracking and management:
   ```hcl
   tags = {
     Project     = "DataPipeline"
     Environment = var.environment
     ManagedBy   = "Terraform"
     Owner       = "data-team"
   }
   ```

5. **Use modules for reusable components**:
   ```hcl
   module "data_lake" {
     source      = "./modules/s3-data-lake"
     bucket_name = "my-data-lake"
     environment = var.environment
   }
   ```

6. **Enable state locking with DynamoDB** to prevent concurrent modifications

7. **Store sensitive values in AWS Secrets Manager**:
   ```hcl
   data "aws_secretsmanager_secret_version" "db_password" {
     secret_id = "prod/db/password"
   }
   ```

8. **Use `terraform plan` before every apply** to review changes

9. **Document your infrastructure** with comments in `.tf` files

10. **Version control your Terraform code** but exclude state files
