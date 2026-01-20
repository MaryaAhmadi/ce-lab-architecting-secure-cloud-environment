# Lab M8.08 - Architecting a Secure Cloud Environment

**Repository:** [https://github.com/cloud-engineering-bootcamp/ce-lab-architecting-secure-cloud-environment](https://github.com/cloud-engineering-bootcamp/ce-lab-architecting-secure-cloud-environment)

**Activity Type:** Individual  
**Estimated Time:** 90 minutes

## Learning Objectives

- [ ] Design a secure three-tier architecture
- [ ] Implement defense-in-depth security controls
- [ ] Create Terraform baseline for secure deployments
- [ ] Document architecture with security annotations

## Prerequisites

- [ ] Terraform installed
- [ ] AWS account
- [ ] Completed Module 8 Lesson 8

## Task

Design and deploy a secure three-tier architecture using Terraform, implementing all security best practices from the module.

## Architecture Requirements

**Web Tier (Public Subnet):**
- Application Load Balancer
- HTTPS only (redirect HTTP to HTTPS)
- WAF enabled

**App Tier (Private Subnet):**
- EC2 or ECS Fargate
- IAM role with least privilege
- Secrets Manager for database credentials

**Data Tier (Private Subnet):**
- RDS PostgreSQL (encrypted)
- No internet access
- Automated backups enabled

## Step-by-Step Instructions

### Step 1: Create VPC with Proper Segmentation

```hcl
# vpc.tf
resource "aws_vpc" "secure" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true

  tags = {
    Name        = "secure-vpc"
    Environment = "production"
  }
}

# Public subnets (ALB)
resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.secure.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "public-subnet-${count.index + 1}"
    Tier = "public"
  }
}

# Private app subnets
resource "aws_subnet" "private_app" {
  count             = 2
  vpc_id            = aws_vpc.secure.id
  cidr_block        = "10.0.${count.index + 11}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "private-app-subnet-${count.index + 1}"
    Tier = "private-app"
  }
}

# Private data subnets (NO internet route)
resource "aws_subnet" "private_data" {
  count             = 2
  vpc_id            = aws_vpc.secure.id
  cidr_block        = "10.0.${count.index + 21}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "private-data-subnet-${count.index + 1}"
    Tier = "private-data"
  }
}
```

### Step 2: Configure Security Groups (Least Privilege)

```hcl
# security-groups.tf

# ALB security group
resource "aws_security_group" "alb" {
  name        = "alb-sg"
  vpc_id      = aws_vpc.secure.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }
}

# App security group
resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = aws_vpc.secure.id

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.database.id]
  }
}

# Database security group
resource "aws_security_group" "database" {
  name   = "database-sg"
  vpc_id = aws_vpc.secure.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  # No egress rules = no outbound internet
}
```

### Step 3: Deploy Encrypted RDS Database

```hcl
# rds.tf
resource "aws_db_instance" "main" {
  identifier        = "secure-database"
  engine            = "postgres"
  instance_class    = "db.t3.micro"
  allocated_storage = 20

  db_name  = "myapp"
  username = "dbadmin"
  password = random_password.db_password.result

  # Security
  storage_encrypted          = true
  kms_key_id                 = aws_kms_key.rds.arn
  vpc_security_group_ids     = [aws_security_group.database.id]
  db_subnet_group_name       = aws_db_subnet_group.main.name
  publicly_accessible        = false
  iam_database_authentication_enabled = true

  # Backups
  backup_retention_period = 30
  backup_window           = "03:00-04:00"
  maintenance_window      = "sun:04:00-sun:05:00"

  # Deletion protection
  deletion_protection = true
  skip_final_snapshot = false
  final_snapshot_identifier = "secure-db-final-snapshot"

  tags = {
    Name        = "secure-database"
    Environment = "production"
  }
}

resource "aws_kms_key" "rds" {
  description = "RDS encryption key"
  deletion_window_in_days = 30
}
```

### Step 4: Enable Logging and Monitoring

```hcl
# monitoring.tf

# CloudTrail
resource "aws_cloudtrail" "main" {
  name                          = "main-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  is_multi_region_trail         = true
  enable_log_file_validation    = true
  include_global_service_events = true
}

# VPC Flow Logs
resource "aws_flow_log" "main" {
  vpc_id          = aws_vpc.secure.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.flow_logs.arn
  log_destination = aws_cloudwatch_log_group.flow_logs.arn
}

# GuardDuty
resource "aws_guardduty_detector" "main" {
  enable = true
}

# Security Hub
resource "aws_securityhub_account" "main" {}
```

### Step 5: Document Architecture

Create `architecture-doc.md`:

```markdown
# Secure Architecture Documentation

## Network Design
- **VPC:** 10.0.0.0/16
- **Public Subnets:** 10.0.1.0/24, 10.0.2.0/24 (ALB only)
- **Private App Subnets:** 10.0.11.0/24, 10.0.12.0/24 (ECS/EC2)
- **Private Data Subnets:** 10.0.21.0/24, 10.0.22.0/24 (RDS, NO internet)

## Security Controls

### Defense in Depth Layers
1. **Perimeter:** WAF (OWASP rules)
2. **Network:** Security groups (least privilege)
3. **Compute:** IAM roles (no access keys)
4. **Data:** Encryption at rest (KMS), in transit (TLS)
5. **Monitoring:** CloudTrail, Flow Logs, GuardDuty

### Encryption
- **RDS:** Encrypted with KMS
- **S3:** Default encryption enabled
- **ALB:** HTTPS only, TLS 1.2+

### Access Control
- **App Tier:** No SSH access (use Systems Manager Session Manager)
- **Database:** No public access, security group restricted to app tier only
- **Secrets:** Stored in Secrets Manager, retrieved at runtime

## Threat Mitigation
| Threat | Mitigation |
|--------|------------|
| SQL Injection | Parameterized queries, WAF |
| Data Breach | Encryption, private subnets |
| DDoS | AWS Shield, CloudFront |
| Privilege Escalation | Least privilege IAM |
```

## Submission

- Complete Terraform code for secure architecture
- `architecture-doc.md` with security annotations
- Architecture diagram (draw.io or similar)
- Screenshot of deployed resources

## Verification Checklist

- [ ] VPC with public, private app, and private data subnets
- [ ] Security groups with least-privilege rules
- [ ] RDS encrypted and in private subnet
- [ ] CloudTrail, Flow Logs, GuardDuty enabled
- [ ] Documentation complete

**Good luck! 🔒**
