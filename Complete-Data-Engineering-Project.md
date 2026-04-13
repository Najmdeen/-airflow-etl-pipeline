---
title: "Complete Data Engineering Project Walkthrough"
subtitle: "E-commerce Analytics Pipeline with AWS, Terraform, and Airflow"
author: "Data Engineering Team"
date: "2024"
---

# Executive Summary

This document provides a comprehensive walkthrough for building a production-grade data engineering pipeline. The project demonstrates industry best practices including Infrastructure as Code (IaC), orchestration, data quality management, and DataOps principles.

## Project Highlights

- **Infrastructure**: Fully automated AWS infrastructure using Terraform
- **Orchestration**: Apache Airflow for workflow management
- **Storage**: Multi-layer data architecture (S3 + RDS)
- **Data Quality**: Automated validation and monitoring
- **Scalability**: Cloud-native, horizontally scalable design

## Technology Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| IaC | Terraform | Infrastructure provisioning |
| Orchestration | Apache Airflow | Workflow scheduling |
| Compute | AWS EC2 | Processing workloads |
| Storage (Lake) | AWS S3 | Raw data storage |
| Storage (Warehouse) | AWS RDS PostgreSQL | Structured data |
| Container | Docker | Application packaging |
| Language | Python 3.9+ | ETL development |
| Version Control | Git | Code management |

---

# 1. Project Architecture

## 1.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     DATA ENGINEERING PIPELINE                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐      ┌──────────────┐      ┌───────────────┐     │
│  │             │      │              │      │               │     │
│  │ API Source  │─────▶│   Airflow    │─────▶│   S3 Bucket   │     │
│  │ (E-commerce)│      │  (Extract)   │      │   Raw Zone    │     │
│  │             │      │              │      │               │     │
│  └─────────────┘      └──────────────┘      └───────┬───────┘     │
│                                                      │             │
│                       ┌──────────────┐               │             │
│                       │              │               │             │
│                       │  Transform   │◀──────────────┘             │
│                       │   (Pandas)   │                             │
│                       │              │                             │
│                       └──────┬───────┘                             │
│                              │                                     │
│                              ▼                                     │
│                       ┌────────────────────┐                       │
│                       │                    │                       │
│                       │  RDS PostgreSQL    │                       │
│                       │ (Data Warehouse)   │                       │
│                       │                    │                       │
│                       └────────────────────┘                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 1.2 Data Flow Diagram

```
Source Layer    Ingestion Layer    Processing Layer    Storage Layer
─────────────   ─────────────────  ──────────────────  ──────────────

┌──────────┐    ┌──────────────┐   ┌──────────────┐   ┌────────────┐
│          │    │              │   │              │   │            │
│   API    │───▶│ Extract DAG  │──▶│  Validation  │──▶│  S3 Raw    │
│          │    │              │   │              │   │            │
└──────────┘    └──────────────┘   └──────┬───────┘   └────────────┘
                                           │
                                           ▼
                ┌──────────────┐   ┌──────────────┐   ┌────────────┐
                │              │   │              │   │            │
                │Transform DAG │──▶│Data Quality  │──▶│S3 Processed│
                │              │   │              │   │            │
                └──────────────┘   └──────┬───────┘   └────────────┘
                                           │
                                           ▼
                ┌──────────────┐   ┌──────────────┐   ┌────────────┐
                │              │   │              │   │            │
                │  Load DAG    │──▶│   Staging    │──▶│    RDS     │
                │              │   │              │   │ PostgreSQL │
                └──────────────┘   └──────────────┘   └────────────┘
```

## 1.3 Infrastructure Components

### AWS Resources

**1. Networking**
- VPC with public and private subnets
- Internet Gateway for public access
- NAT Gateway for private subnet internet access
- Security Groups for access control

**2. Compute**
- EC2 instance (t3.medium) for Airflow
- Auto Scaling Group (optional for production)

**3. Storage**
- S3 bucket with lifecycle policies
- RDS PostgreSQL (db.t3.micro)

**4. Security**
- IAM roles and policies
- Security groups
- Encrypted storage

---

# 2. Prerequisites and Setup

## 2.1 Required Software

### Local Development Machine

**Operating System Requirements:**
- Ubuntu 20.04 LTS or macOS 11+

**Required Tools:**
- AWS CLI v2.x
- Terraform v1.6.x
- Python 3.9+
- Docker 20.x
- Git 2.x
- PostgreSQL client
- SSH client

### Installation Steps

#### Install AWS CLI

```bash
# Download and install
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify installation
aws --version
# Expected output: aws-cli/2.x.x Python/3.x.x Linux/x.x.x
```

#### Configure AWS Credentials

```bash
aws configure

# Enter the following when prompted:
AWS Access Key ID [None]: YOUR_ACCESS_KEY_ID
AWS Secret Access Key [None]: YOUR_SECRET_ACCESS_KEY
Default region name [None]: us-east-1
Default output format [None]: json

# Verify configuration
aws sts get-caller-identity
```

#### Install Terraform

```bash
# Download Terraform
wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip

# Unzip and install
unzip terraform_1.6.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# Verify installation
terraform version
# Expected output: Terraform v1.6.0
```

#### Install Python Dependencies

```bash
# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install packages
pip install --upgrade pip
pip install boto3 pandas sqlalchemy psycopg2-binary requests apache-airflow

# Verify installation
python -c "import boto3; print(boto3.__version__)"
```

#### Install Docker

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y docker.io docker-compose

# Start Docker
sudo systemctl start docker
sudo systemctl enable docker

# Add user to docker group
sudo usermod -aG docker $USER

# Verify installation
docker --version
docker-compose --version
```

## 2.2 AWS Account Requirements

### IAM Permissions Required

Create an IAM user with the following policies:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "rds:*",
        "s3:*",
        "iam:*",
        "vpc:*"
      ],
      "Resource": "*"
    }
  ]
}
```

### Cost Estimation

| Resource | Configuration | Monthly Cost (USD) |
|----------|--------------|-------------------|
| EC2 (t3.medium) | 1 instance | ~$30 |
| RDS (db.t3.micro) | 1 instance | ~$15 |
| S3 Storage | 100 GB | ~$2.30 |
| Data Transfer | 50 GB/month | ~$4.50 |
| **Total** | | **~$52/month** |

*Note: Costs are approximate and may vary by region*

## 2.3 Generate SSH Key Pair

```bash
# Generate new SSH key
ssh-keygen -t rsa -b 4096 -f ~/.ssh/aws_data_pipeline_key

# Set proper permissions
chmod 400 ~/.ssh/aws_data_pipeline_key
chmod 644 ~/.ssh/aws_data_pipeline_key.pub

# Display public key (you'll need this for Terraform)
cat ~/.ssh/aws_data_pipeline_key.pub
```

---

# 3. Project Setup

## 3.1 Create Project Structure

```bash
# Create main project directory
mkdir -p ecommerce-data-pipeline
cd ecommerce-data-pipeline

# Create directory structure
mkdir -p terraform/{modules/{networking,compute,storage,database},environments/dev}
mkdir -p airflow/{dags,plugins,config,scripts}
mkdir -p sql/{ddl,dml,views}
mkdir -p tests/{unit,integration}
mkdir -p docs
mkdir -p scripts

# Create README
cat > README.md << 'EOF'
# E-commerce Data Pipeline

Production-grade data engineering pipeline built with Terraform and Apache Airflow.

## Quick Start

1. Set up infrastructure: `cd terraform/environments/dev && terraform apply`
2. Deploy Airflow: `ssh ec2-user@<IP> 'cd /opt/airflow && docker-compose up -d'`
3. Access Airflow UI: http://<EC2-IP>:8080

## Documentation

See `/docs` folder for detailed documentation.
EOF

# Initialize git repository
git init
cat > .gitignore << 'EOF'
# Terraform
*.tfstate
*.tfstate.backup
.terraform/
*.tfvars

# Python
__pycache__/
*.py[cod]
venv/
.env

# Airflow
airflow/logs/
airflow/airflow.db

# IDE
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db
EOF

git add .
git commit -m "Initial project structure"
```

## 3.2 Project Structure Overview

```
ecommerce-data-pipeline/
├── README.md
├── .gitignore
├── terraform/
│   ├── modules/
│   │   ├── networking/          # VPC, Subnets, Security Groups
│   │   ├── compute/             # EC2, IAM Roles
│   │   ├── storage/             # S3 Buckets
│   │   └── database/            # RDS PostgreSQL
│   └── environments/
│       └── dev/                 # Development environment
├── airflow/
│   ├── dags/                    # Airflow DAGs
│   ├── plugins/                 # Custom operators
│   ├── config/                  # Configuration files
│   └── docker-compose.yml       # Airflow deployment
├── sql/
│   ├── ddl/                     # Table definitions
│   ├── dml/                     # Data manipulation
│   └── views/                   # View definitions
├── scripts/
│   ├── setup/                   # Setup scripts
│   └── utilities/               # Utility scripts
├── tests/
│   ├── unit/                    # Unit tests
│   └── integration/             # Integration tests
└── docs/                        # Documentation
```

---

# 4. Terraform Infrastructure

## 4.1 Networking Module

**File: `terraform/modules/networking/main.tf`**

```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(var.tags, {
    Name = "${var.project_name}-vpc"
  })
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(var.tags, {
    Name = "${var.project_name}-igw"
  })
}

# Public Subnets
resource "aws_subnet" "public" {
  count                   = length(var.public_subnet_cidrs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = merge(var.tags, {
    Name = "${var.project_name}-public-subnet-${count.index + 1}"
    Type = "Public"
  })
}

# Private Subnets
resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = merge(var.tags, {
    Name = "${var.project_name}-private-subnet-${count.index + 1}"
    Type = "Private"
  })
}

# NAT Gateway
resource "aws_eip" "nat" {
  count  = var.enable_nat_gateway ? 1 : 0
  domain = "vpc"

  tags = merge(var.tags, {
    Name = "${var.project_name}-nat-eip"
  })
}

resource "aws_nat_gateway" "main" {
  count         = var.enable_nat_gateway ? 1 : 0
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id

  tags = merge(var.tags, {
    Name = "${var.project_name}-nat-gateway"
  })

  depends_on = [aws_internet_gateway.main]
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = merge(var.tags, {
    Name = "${var.project_name}-public-rt"
  })
}

resource "aws_route_table" "private" {
  count  = var.enable_nat_gateway ? 1 : 0
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[0].id
  }

  tags = merge(var.tags, {
    Name = "${var.project_name}-private-rt"
  })
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count          = length(var.public_subnet_cidrs)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = var.enable_nat_gateway ? length(var.private_subnet_cidrs) : 0
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[0].id
}
```

**File: `terraform/modules/networking/variables.tf`**

```hcl
variable "project_name" {
  description = "Project name for resource naming"
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnet_cidrs" {
  description = "CIDR blocks for private subnets"
  type        = list(string)
  default     = ["10.0.10.0/24", "10.0.11.0/24"]
}

variable "availability_zones" {
  description = "Availability zones for subnets"
  type        = list(string)
}

variable "enable_nat_gateway" {
  description = "Enable NAT Gateway for private subnets"
  type        = bool
  default     = true
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

**File: `terraform/modules/networking/outputs.tf`**

```hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "CIDR block of the VPC"
  value       = aws_vpc.main.cidr_block
}

output "public_subnet_ids" {
  description = "IDs of public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs of private subnets"
  value       = aws_subnet.private[*].id
}

output "nat_gateway_id" {
  description = "ID of NAT Gateway"
  value       = var.enable_nat_gateway ? aws_nat_gateway.main[0].id : null
}
```

---

# 5. Converting This Document to PDF

## Method 1: Using Pandoc (Recommended)

```bash
# Install Pandoc
sudo apt-get install pandoc texlive-latex-base texlive-fonts-recommended

# Convert to PDF
pandoc Complete-Data-Engineering-Project.md -o Complete-Data-Engineering-Project.pdf \
  --toc \
  --number-sections \
  -V geometry:margin=1in \
  -V fontsize=10pt \
  --highlight-style=tango
```

## Method 2: Online Converter

1. Copy the markdown content
2. Visit: https://www.markdowntopdf.com/
3. Paste content and download PDF

## Method 3: VS Code Extension

1. Install "Markdown PDF" extension in VS Code
2. Open this markdown file
3. Press `Ctrl+Shift+P` (or `Cmd+Shift+P` on Mac)
4. Type "Markdown PDF: Export (pdf)" and press Enter

---

**Document Version:** 1.0  
**Last Updated:** 2024  
**Maintained By:** Data Engineering Team
