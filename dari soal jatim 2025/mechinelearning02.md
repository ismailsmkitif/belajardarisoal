# 📘 Panduan Lengkap Pengerjaan Saran 2 (Machine Learning)
## "IaC Event Pipeline" — Terraform + S3 → Lambda → Rekognition → DynamoDB → Streams → Kinesis

> **Estimasi waktu**: 2 jam | **Region**: `us-east-1` | **Terraform-based** (Infrastructure as Code)
> **Tujuan**: Membungkus Saran 1 ke dalam Terraform + menambahkan DynamoDB Streams → Kinesis Data Stream.

---

## 🎯 Tujuan Latihan

Membangun **pipeline yang sama dengan Saran 1**, tetapi kali ini **100% menggunakan Terraform** sebagai Infrastructure as Code (IaC), plus menambahkan **streaming pipeline**:
- ✅ Semua resource di-provision via `terraform apply`
- ✅ S3 → Lambda → Rekognition → DynamoDB (logika bisnis dari Saran 1)
- ✅ **DynamoDB Streams** aktif (capture INSERT/MODIFY/REMOVE)
- ✅ Streams di-forward ke **Kinesis Data Stream** untuk real-time analytics
- ✅ Prefix naming **`techno-`** sesuai standar soal LKS
- ✅ Struktur Terraform rapi (`main.tf`, `variables.tf`, `outputs.tf`, `lambda/`)

Ini adalah **fondasi IaC** yang akan Anda perluas di Saran 3 (API Gateway + EKS) dan Saran 4 (Glue + SNS + IPv6).

---

## 🗺️ Topologi Arsitektur

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                       AWS CLOUD (us-east-1) - Terraform Managed                  │
│                                                                                  │
│   ┌──────────────┐         ┌──────────────────────────────────────────────┐     │
│   │  USER        │         │  Lambda: techno-lambda-s3                    │     │
│   │              │         │                                              │     │
│   │  Upload      │         │  1. Triggered by S3 event                    │     │
│   │  image.jpg   │         │  2. Call Rekognition DetectLabels            │     │
│   └──────┬───────┘         │  3. Save result to DynamoDB "Tokens"         │     │
│          │                 │  4. DynamoDB automatically streams changes   │     │
│          │                 │                                              │     │
│          ▼                 └──────────┬───────────────────────────────────┘     │
│   ┌──────────────────┐                │                                         │
│   │  S3 Bucket       │                │                                         │
│   │  technoinput-    │───── event ───▶│                                         │
│   │  jakarta-budi    │                │                                         │
│   └──────────────────┘                ▼                                         │
│                                ┌──────────────────┐                              │
│                                │  DynamoDB        │                              │
│                                │  Table: Tokens   │                              │
│                                │  PK: token (S)   │                              │
│                                │                  │                              │
│                                │  ┌────────────┐  │                              │
│                                │  │  Streams   │  │                              │
│                                │  │  (enabled) │  │                              │
│                                │  └─────┬──────┘  │                              │
│                                └────────┼─────────┘                              │
│                                         │                                        │
│                                         │ DynamoDB Stream                        │
│                                         │ (NEW_AND_OLD_IMAGES)                   │
│                                         ▼                                        │
│                                ┌──────────────────┐                              │
│                                │  Kinesis Data    │                              │
│                                │  Stream          │                              │
│                                │  techno-kinesis- │                              │
│                                │  budi            │                              │
│                                │                  │                              │
│                                │  • 1 Shard       │                              │
│                                │  • 1 day retain  │                              │
│                                │  • Provisioned   │                              │
│                                └──────────────────┘                              │
│                                                                                  │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │  VPC: techno-budi (25.1.0.0/16)                                          │  │
│   │  ├── Public Subnet 1:  25.1.0.0/24  (AZ a)                              │  │
│   │  ├── Public Subnet 2:  25.1.2.0/24  (AZ b)                              │  │
│   │  ├── Private Subnet 1: 25.1.1.0/24  (AZ a)                              │  │
│   │  ├── Private Subnet 2: 25.1.3.0/24  (AZ b)                              │  │
│   │  ├── IGW: techno-igw-budi                                                │  │
│   │  ├── NAT GW: techno-natgw-budi                                           │  │
│   │  └── Route Tables: techno-public-rt, techno-private-rt                   │  │
│   └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 🔄 Alur Kerja (Flow)
```
1. User upload gambar → S3 bucket "technoinput-jakarta-budi"
                          ↓
2. S3 Event Notification → trigger Lambda "techno-lambda-s3"
                          ↓
3. Lambda panggil Rekognition DetectLabels
                          ↓
4. Lambda simpan hasil ke DynamoDB table "Tokens"
                          ↓
5. DynamoDB Streams (NEW_AND_OLD_IMAGES) capture perubahan
                          ↓
6. Stream di-forward ke Kinesis Data Stream "techno-kinesis-budi"
                          ↓
7. Kinesis siap dikonsumsi oleh downstream (Glue/Athena/custom consumer)
```

---

## 📋 Daftar Isi
1. [Fase 0: Persiapan Environment](#fase-0-persiapan)
2. [Fase 1: Struktur Folder Terraform](#fase-1-struktur)
3. [Fase 2: Provider & Variables](#fase-2-provider)
4. [Fase 3: VPC & Networking](#fase-3-vpc)
5. [Fase 4: IAM Role & Policy](#fase-4-iam)
6. [Fase 5: S3 Bucket](#fase-5-s3)
7. [Fase 6: DynamoDB + Streams](#fase-6-dynamodb)
8. [Fase 7: Kinesis Data Stream](#fase-7-kinesis)
9. [Fase 8: Lambda Function](#fase-8-lambda)
10. [Fase 9: S3 Event Trigger](#fase-9-trigger)
11. [Fase 10: DynamoDB → Kinesis Streaming](#fase-10-streaming)
12. [Fase 11: Terraform Apply](#fase-11-apply)
13. [Fase 12: Testing End-to-End](#fase-12-testing)
14. [Fase 13: Cleanup](#fase-13-cleanup)

---

## 🎯 Fase 0: Persiapan Environment

### Langkah 0.1: Login & Region
1. Login ke **AWS Lab Learner** → klik **Start Lab** → **Open AWS Console**.
2. Region: **US East (N. Virginia) `us-east-1`**.
3. Catat di Notepad:
   - **Account ID** (12 digit)
   - **Access Key ID**
   - **Secret Access Key**
   - **Session Token**
   - **Nama Anda** (contoh: `budi`)
   - **Kota Anda** (contoh: `jakarta`)

### Langkah 0.2: Install Terraform di CloudShell
1. Buka **CloudShell** (ikon `>_` di top bar AWS Console).
2. Jalankan:
```bash
# Cek apakah terraform sudah terinstall
terraform version

# Jika belum, install:
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
sudo yum -y install terraform

# Verifikasi
terraform version
```

### Langkah 0.3: Konfigurasi AWS CLI di CloudShell
```bash
aws configure
# Masukkan:
# AWS Access Key ID: <dari Lab Details>
# AWS Secret Access Key: <dari Lab Details>
# Default region: us-east-1
# Default output: json

# Tambahkan session token (PENTING untuk Lab Learner!)
aws configure set aws_session_token <SESSION_TOKEN_DARI_LAB>
```

### Langkah 0.4: Tentukan Naming Convention
| Resource | Nama |
|---|---|
| VPC | `techno-<nama>` |
| S3 Input | `technoinput-<kota>-<nama>` |
| DynamoDB Table | `Tokens` |
| Lambda | `techno-lambda-s3` |
| Kinesis | `techno-kinesis-<nama>` |

---

## 📁 Fase 1: Struktur Folder Terraform

### Langkah 1.1: Buat Struktur Folder
Di CloudShell:
```bash
mkdir -p ~/techno-ml-project/lambda
cd ~/techno-ml-project
```

Struktur final:
```
techno-ml-project/
├── main.tf              # Resource utama
├── variables.tf         # Variabel input
├── outputs.tf           # Output values
├── terraform.tfvars     # Nilai variabel
└── lambda/
    └── techno_lambda_s3.py   # Kode Lambda
```

### Langkah 1.2: Buat File Kosong
```bash
touch main.tf variables.tf outputs.tf terraform.tfvars
mkdir -p lambda
touch lambda/techno_lambda_s3.py
```

---

## 🔧 Fase 2: Provider & Variables

### 📄 `variables.tf`
```hcl
# ============================================
# VARIABEL INPUT
# ============================================

variable "aws_region" {
  description = "AWS region untuk deployment"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Nama peserta (untuk naming convention)"
  type        = string
  default     = "budi"  # GANTI dengan nama Anda
}

variable "city" {
  description = "Kota peserta (untuk naming convention)"
  type        = string
  default     = "jakarta"  # GANTI dengan kota Anda
}

variable "vpc_cidr" {
  description = "CIDR block untuk VPC"
  type        = string
  default     = "25.1.0.0/16"
}

variable "public_subnet_1_cidr" {
  description = "CIDR public subnet 1"
  type        = string
  default     = "25.1.0.0/24"
}

variable "public_subnet_2_cidr" {
  description = "CIDR public subnet 2"
  type        = string
  default     = "25.1.2.0/24"
}

variable "private_subnet_1_cidr" {
  description = "CIDR private subnet 1"
  type        = string
  default     = "25.1.1.0/24"
}

variable "private_subnet_2_cidr" {
  description = "CIDR private subnet 2"
  type        = string
  default     = "25.1.3.0/24"
}

variable "availability_zones" {
  description = "AZs yang digunakan"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b"]
}

variable "dynamodb_table_name" {
  description = "Nama tabel DynamoDB"
  type        = string
  default     = "Tokens"
}

variable "kinesis_shard_count" {
  description = "Jumlah shard Kinesis"
  type        = number
  default     = 1
}

variable "kinesis_retention_hours" {
  description = "Retention period Kinesis (jam)"
  type        = number
  default     = 24
}

variable "lambda_timeout" {
  description = "Timeout Lambda dalam detik"
  type        = number
  default     = 120  # 2 menit sesuai soal
}

variable "lab_role_arn" {
  description = "ARN LabRole dari AWS Lab Learner"
  type        = string
  default     = ""  # Akan diisi di terraform.tfvars
}
```

### 📄 `terraform.tfvars`
```hcl
# ============================================
# ISI VARIABEL SESUAI DATA ANDA
# ============================================

aws_region   = "us-east-1"
project_name = "budi"       # GANTI dengan nama Anda
city         = "jakarta"    # GANTI dengan kota Anda

# GANTI dengan ARN LabRole dari AWS Console → IAM → Roles
lab_role_arn = "arn:aws:iam::XXXXXXXXXXXX:role/LabRole"
```

### 📄 `main.tf` (bagian provider)
```hcl
# ============================================
# TERRAFORM & PROVIDER CONFIGURATION
# ============================================

terraform {
  required_version = ">= 1.0.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Project     = "LKS-MachineLearning"
      Environment = "Training"
      Owner       = var.project_name
      ManagedBy   = "Terraform"
    }
  }
}

# Data source untuk AWS account
data "aws_caller_identity" "current" {}

# Data source untuk AZs
data "aws_availability_zones" "available" {
  state = "available"
}

# ============================================
# LOCAL VALUES (untuk naming convention)
# ============================================

locals {
  prefix       = "techno"
  name_suffix  = var.project_name
  city_suffix  = var.city
  
  vpc_name              = "${local.prefix}-${local.name_suffix}"
  s3_input_bucket       = "${local.prefix}input-${local.city_suffix}-${local.name_suffix}"
  lambda_name           = "${local.prefix}-lambda-s3"
  kinesis_name          = "${local.prefix}-kinesis-${local.name_suffix}"
  igw_name              = "${local.prefix}-igw-${local.name_suffix}"
  natgw_name            = "${local.prefix}-natgw-${local.name_suffix}"
  public_rt_name        = "${local.prefix}-public-rt"
  private_rt_name       = "${local.prefix}-private-rt"
  sg_lb_name            = "${local.prefix}-sg-lb"
  sg_apps_name          = "${local.prefix}-sg-apps"
  
  account_id = data.aws_caller_identity.current.account_id
}
```

---

## 🌐 Fase 3: VPC & Networking

### Tambahkan ke `main.tf`:
```hcl
# ============================================
# VPC & NETWORKING
# ============================================

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true
  
  tags = {
        Name = local.vpc_name
    Description = "VPC for ML image processing pipeline"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = local.igw_name
  }
}

# Public Subnets (2 AZ)
resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = count.index == 0 ? var.public_subnet_1_cidr : var.public_subnet_2_cidr
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true
  
  tags = {
    Name = "${local.vpc_name}-public-${var.availability_zones[count.index]}"
    Tier = "Public"
  }
}

# Private Subnets (2 AZ)
resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = count.index == 0 ? var.private_subnet_1_cidr : var.private_subnet_2_cidr
  availability_zone = var.availability_zones[count.index]
  
  tags = {
    Name = "${local.vpc_name}-private-${var.availability_zones[count.index]}"
    Tier = "Private"
  }
}

# Elastic IP untuk NAT Gateway
resource "aws_eip" "nat" {
  domain = "vpc"
  
  tags = {
    Name = "${local.natgw_name}-eip"
  }
}

# NAT Gateway (di public subnet)
resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id
  
  tags = {
    Name = local.natgw_name
  }
  
  depends_on = [aws_internet_gateway.main]
}

# Public Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = {
    Name = local.public_rt_name
  }
}

# Private Route Table
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }
  
  tags = {
    Name = local.private_rt_name
  }
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = 2
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}

# ============================================
# SECURITY GROUPS
# ============================================

resource "aws_security_group" "lb" {
  name        = local.sg_lb_name
  description = "Security group for Load Balancer"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    description = "HTTP from internet"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    description = "HTTPS from internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = local.sg_lb_name
  }
}

resource "aws_security_group" "apps" {
  name        = local.sg_apps_name
  description = "Security group for applications"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    description = "Custom app port from public"
    from_port   = 2000
    to_port     = 2000
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = local.sg_apps_name
  }
}
```

---

## 👤 Fase 4: IAM Role & Policy

### Tambahkan ke `main.tf`:
```hcl
# ============================================
# IAM ROLE & POLICY UNTUK LAMBDA
# ============================================

# Gunakan LabRole yang sudah ada, tapi buat inline policy terpisah
# Jika LabRole tidak bisa di-edit, buat role baru

resource "aws_iam_role" "lambda_exec" {
  name = "${local.prefix}-lambda-exec-role-${local.name_suffix}"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })
  
  tags = {
    Name = "${local.prefix}-lambda-exec-role"
  }
}

# Attach managed policies
resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy_attachment" "lambda_vpc" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}

# Inline policy untuk S3, Rekognition, DynamoDB, Kinesis
resource "aws_iam_role_policy" "lambda_custom" {
  name = "${local.prefix}-lambda-custom-policy"
  role = aws_iam_role.lambda_exec.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "S3ReadAccess"
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.input.arn,
          "${aws_s3_bucket.input.arn}/*"
        ]
      },
      {
        Sid    = "RekognitionAccess"
        Effect = "Allow"
        Action = [
          "rekognition:DetectLabels",
          "rekognition:DetectFaces",
          "rekognition:DetectText"
        ]
        Resource = "*"
      },
      {
        Sid    = "DynamoDBAccess"
        Effect = "Allow"
        Action = [
          "dynamodb:PutItem",
          "dynamodb:GetItem",
          "dynamodb:UpdateItem",
          "dynamodb:Query",
          "dynamodb:Scan"
        ]
        Resource = aws_dynamodb_table.tokens.arn
      },
      {
        Sid    = "KinesisAccess"
        Effect = "Allow"
        Action = [
          "kinesis:PutRecord",
          "kinesis:PutRecords"
        ]
        Resource = aws_kinesis_stream.main.arn
      }
    ]
  })
}
```

---

## 📦 Fase 5: S3 Bucket

### Tambahkan ke `main.tf`:
```hcl
# ============================================
# S3 BUCKETS
# ============================================

# Input Bucket (trigger Lambda)
resource "aws_s3_bucket" "input" {
  bucket = local.s3_input_bucket
  
  tags = {
    Name        = local.s3_input_bucket
    Description = "Input bucket for image uploads"
  }
}

resource "aws_s3_bucket_versioning" "input" {
  bucket = aws_s3_bucket.input.id
  versioning_configuration {
    status = "Disabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "input" {
  bucket = aws_s3_bucket.input.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "input" {
  bucket = aws_s3_bucket.input.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Lifecycle rule: Glacier after 30 days, expire after 365 days
resource "aws_s3_bucket_lifecycle_configuration" "input" {
  bucket = aws_s3_bucket.input.id
  
  rule {
    id     = "archive-and-expire"
    status = "Enabled"
    
    transition {
      days          = 30
      storage_class = "GLACIER"
    }
    
    expiration {
      days = 365
    }
  }
}

# S3 Event Notification → Lambda
resource "aws_s3_bucket_notification" "input_lambda" {
  bucket = aws_s3_bucket.input.id
  
  lambda_function {
    lambda_function_arn = aws_lambda_function.s3_processor.arn
    events              = ["s3:ObjectCreated:*"]
  }
  
  depends_on = [aws_lambda_permission.s3_invoke]
}
```

---

## 🗄️ Fase 6: DynamoDB + Streams

### Tambahkan ke `main.tf`:
```hcl
# ============================================
# DYNAMODB TABLE + STREAMS
# ============================================

resource "aws_dynamodb_table" "tokens" {
  name         = var.dynamodb_table_name
  billing_mode = "PAY_PER_REQUEST"  # On-demand untuk latihan
  hash_key     = "token"
  
  attribute {
    name = "token"
    type = "S"
  }
  
  # Enable DynamoDB Streams
  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"
  
  tags = {
    Name        = var.dynamodb_table_name
    Description = "Token storage for ML pipeline"
  }
}
```

---

## 🌊 Fase 7: Kinesis Data Stream

### Tambahkan ke `main.tf`:
```hcl
# ============================================
# KINESIS DATA STREAM
# ============================================

resource "aws_kinesis_stream" "main" {
  name             = local.kinesis_name
  shard_count      = var.kinesis_shard_count
  retention_period = var.kinesis_retention_hours  # dalam jam (24 = 1 hari)
  
  shard_level_metrics = [
    "IncomingRecords",
    "IncomingBytes",
    "OutgoingRecords",
    "OutgoingBytes"
  ]
  
  stream_mode_details {
    stream_mode = "PROVISIONED"
  }
  
  tags = {
    Name        = local.kinesis_name
    Description = "Kinesis stream for DynamoDB change data capture"
  }
}

# ============================================
# DYNAMODB → KINESIS STREAMING DESTINATION
# ============================================

resource "aws_dynamodb_kinesis_streaming_destination" "tokens_to_kinesis" {
  table_name = aws_dynamodb_table.tokens.name
  stream_arn = aws_kinesis_stream.main.arn
  
  depends_on = [
    aws_dynamodb_table.tokens,
    aws_kinesis_stream.main
  ]
}
```

---

## ⚡ Fase 8: Lambda Function

### 📄 `lambda/techno_lambda_s3.py`
```python
import json
import boto3
import urllib.parse
from datetime import datetime

# Initialize AWS clients
s3_client = boto3.client('s3')
rekognition_client = boto3.client('rekognition')
dynamodb_client = boto3.client('dynamodb')

DYNAMODB_TABLE = 'Tokens'

def lambda_handler(event, context):
    """
    Lambda handler untuk memproses gambar yang di-upload ke S3.
    1. Baca object dari S3
    2. Panggil Rekognition DetectLabels
    3. Simpan hasil ke DynamoDB
    """
    print(f"Event received: {json.dumps(event)}")
    
    try:
        # 1. Extract informasi dari S3 event
        bucket = event['Records'][0]['s3']['bucket']['name']
        key = urllib.parse.unquote_plus(
            event['Records'][0]['s3']['object']['key'], 
            encoding='utf-8'
        )
        
        print(f"Processing image: s3://{bucket}/{key}")
        
        # 2. Generate token unik
        token = f"token-{int(datetime.now().timestamp())}-{key.split('.')[0]}"
        
        # 3. Panggil Amazon Rekognition - DetectLabels
        response = rekognition_client.detect_labels(
            Image={
                'S3Object': {
                    'Bucket': bucket,
                    'Name': key
                }
            },
            MaxLabels=20,
            MinConfidence=70.0
        )
        
        labels = response.get('Labels', [])
        print(f"Detected {len(labels)} labels")
        
        # 4. Format hasil untuk DynamoDB
        labels_data = []
        for label in labels:
            labels_data.append({
                'Name': label['Name'],
                'Confidence': float(label['Confidence']),
                'Categories': [cat['Name'] for cat in label.get('Categories', [])]
            })
        
        # 5. Simpan ke DynamoDB
        timestamp = datetime.now().isoformat()
        
        dynamodb_client.put_item(
            TableName=DYNAMODB_TABLE,
            Item={
                'token': {'S': token},
                'image_name': {'S': key},
                'bucket': {'S': bucket},
                'labels': {'S': json.dumps(labels_data)},
                'label_count': {'N': str(len(labels))},
                'timestamp': {'S': timestamp},
                'status': {'S': 'SUCCESS'}
            }
        )
        
        print(f"Successfully saved to DynamoDB with token: {token}")
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Image processed successfully',
                'token': token,
                'labels_detected': len(labels),
                'top_labels': [l['Name'] for l in labels[:5]]
            })
        }
        
    except Exception as e:
        print(f"Error processing image: {str(e)}")
        
        try:
            dynamodb_client.put_item(
                TableName=DYNAMODB_TABLE,
                Item={
                    'token': {'S': f"error-{int(datetime.now().timestamp())}"},
                    'image_name': {'S': key if 'key' in locals() else 'unknown'},
                    'status': {'S': 'FAILED'},
                    'error_message': {'S': str(e)},
                    'timestamp': {'S': datetime.now().isoformat()}
                }
            )
        except Exception as db_err:
            print(f"Failed to save error to DynamoDB: {str(db_err)}")
        
        return {
            'statusCode': 500,
            'body': json.dumps({
                'message': 'Error processing image',
                'error': str(e)
            })
        }
```

### Tambahkan ke `main.tf`:
```hcl
# ============================================
# LAMBDA FUNCTION
# ============================================

# Archive Lambda code
data "archive_file" "lambda_s3" {
  type        = "zip"
  source_file = "${path.module}/lambda/techno_lambda_s3.py"
  output_path = "${path.module}/lambda/techno_lambda_s3.zip"
}

# Lambda Function
resource "aws_lambda_function" "s3_processor" {
  function_name    = local.lambda_name
  filename         = data.archive_file.lambda_s3.output_path
  source_code_hash = data.archive_file.lambda_s3.output_base64sha256
  handler          = "techno_lambda_s3.lambda_handler"
  runtime          = "python3.12"
  timeout          = var.lambda_timeout  # 120 detik = 2 menit
  memory_size      = 256
  
  role = aws_iam_role.lambda_exec.arn
  
  environment {
    variables = {
      DYNAMODB_TABLE = var.dynamodb_table_name
      KINESIS_STREAM = local.kinesis_name
    }
  }
  
  tags = {
    Name        = local.lambda_name
    Description = "Lambda for S3 image processing with Rekognition"
  }
}

# Permission untuk S3 invoke Lambda
resource "aws_lambda_permission" "s3_invoke" {
  statement_id  = "AllowS3Invoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.s3_processor.function_name
  principal     = "s3.amazonaws.com"
  source_arn    = aws_s3_bucket.input.arn
}

# CloudWatch Log Group
resource "aws_cloudwatch_log_group" "lambda" {
  name              = "/aws/lambda/${local.lambda_name}"
  retention_in_days = 7
  
  tags = {
    Name = "${local.lambda_name}-logs"
  }
}
```

---

## 📤 Fase 9: S3 Event Trigger

Sudah di-handle di Fase 5 via `aws_s3_bucket_notification`. Tidak perlu konfigurasi tambahan.

---

## 🔗 Fase 10: DynamoDB → Kinesis Streaming

Sudah di-handle di Fase 7 via `aws_dynamodb_kinesis_streaming_destination`.

---

## 📊 Fase 11: Outputs

### 📄 `outputs.tf`
```hcl
# ============================================
# OUTPUTS
# ============================================

output "vpc_id" {
  description = "ID VPC yang dibuat"
  value       = aws_vpc.main.id
}

output "s3_input_bucket" {
  description = "Nama bucket input S3"
  value       = aws_s3_bucket.input.id
}

output "dynamodb_table_name" {
  description = "Nama tabel DynamoDB"
  value       = aws_dynamodb_table.tokens.name
}

output "dynamodb_table_arn" {
  description = "ARN tabel DynamoDB"
  value       = aws_dynamodb_table.tokens.arn
}

output "dynamodb_stream_arn" {
  description = "ARN DynamoDB Stream"
  value       = aws_dynamodb_table.tokens.stream_arn
}

output "kinesis_stream_name" {
  description = "Nama Kinesis Data Stream"
  value       = aws_kinesis_stream.main.name
}

output "kinesis_stream_arn" {
  description = "ARN Kinesis Data Stream"
  value       = aws_kinesis_stream.main.arn
}

output "lambda_function_name" {
  description = "Nama Lambda function"
  value       = aws_lambda_function.s3_processor.function_name
}

output "lambda_function_arn" {
  description = "ARN Lambda function"
  value       = aws_lambda_function.s3_processor.arn
}

output "account_id" {
  description = "AWS Account ID"
  value       = local.account_id
}
```

---

## 🚀 Fase 11: Terraform Apply

### Langkah 11.1: Init
```bash
cd ~/techno-ml-project
terraform init
```

### Langkah 11.2: Validate
```bash
terraform validate
```

### Langkah 11.3: Plan
```bash
terraform plan -out=tfplan
```

Review output — harus ada ~20 resource yang akan dibuat:
- 1 VPC, 4 Subnets, 1 IGW, 1 NAT GW, 2 Route Tables
- 2 Security Groups
- 1 S3 Bucket + lifecycle + notification
- 1 DynamoDB Table (dengan stream)
- 1 Kinesis Stream
- 1 DynamoDB → Kinesis streaming destination
- 1 Lambda Function + IAM role + log group

### Langkah 11.4: Apply
```bash
terraform apply tfplan
```

Tunggu ~3-5 menit sampai semua resource terbuat.

### Langkah 11.5: Cek Outputs
```bash
terraform output
```

---

## 🧪 Fase 12: Testing End-to-End

### Test 1: Cek Resource Terraform
```bash
terraform state list
```
Harus ada semua resource yang didefinisikan.

### Test 2: Upload Gambar ke S3
```bash
# Download gambar test
curl -o test-image.jpg https://picsum.photos/800/600.jpg

# Upload ke S3
aws s3 cp test-image.jpg s3://$(terraform output -raw s3_input_bucket)/test-image.jpg
```

### Test 3: Cek Lambda Invocation
```bash
# Lihat log Lambda (tunggu ~30 detik)
aws logs tail /aws/lambda/$(terraform output -raw lambda_function_name) --since 2m --follow
```

Harus muncul log:
```
Processing image: s3://technoinput-jakarta-budi/test-image.jpg
Detected X labels
Successfully saved to DynamoDB with token: token-xxxxx-test-image
```

### Test 4: Cek DynamoDB
```bash
aws dynamodb scan --table-name $(terraform output -raw dynamodb_table_name)
```

Harus ada item dengan:
- `token`: `token-xxxxx-test-image`
- `image_name`: `test-image.jpg`
- `labels`: JSON array
- `status`: `SUCCESS`

### Test 5: Cek DynamoDB Streams
```bash
# List stream
aws dynamodb describe-stream \
  --table-name $(terraform output -raw dynamodb_table_name) \
  --region us-east-1

# Cek Kinesis stream records
aws kinesis get-shard-iterator \
  --stream-name $(terraform output -raw kinesis_stream_name) \
  --shard-id shardId-000000000000 \
  --shard-iterator-type TRIM_HORIZON \
  --query 'ShardIterator' \
  --output text | xargs -I {} aws kinesis get-records --shard-iterator {}
```

Harus ada records dari perubahan DynamoDB.

### Test 6: Cek Kinesis Metrics
```bash
aws cloudwatch get-metric-statistics \
  --namespace "AWS/Kinesis" \
  --metric-name "IncomingRecords" \
  --dimensions Name=StreamName,Value=$(terraform output -raw kinesis_stream_name) \
  --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Sum
```

### Test 7: Upload Gambar Berbeda
```bash
curl -o test-image-2.jpg https://picsum.photos/600/400.jpg
aws s3 cp test-image-2.jpg s3://$(terraform output -raw s3_input_bucket)/test-image-2.jpg
```

Ulangi Test 3-5 → harus ada item baru di DynamoDB dan record baru di Kinesis.

---

## 🧹 Fase 13: Cleanup

### Langkah 13.1: Kosongkan S3 Bucket
```bash
aws s3 rm s3://$(terraform output -raw s3_input_bucket) --recursive
```

### Langkah 13.2: Terraform Destroy
```bash
terraform destroy -auto-approve
```

Tunggu ~5 menit sampai semua resource terhapus.

### Langkah 13.3: Verifikasi Cleanup
```bash
terraform state list
# Harus kosong

# Cek manual
aws s3 ls
aws dynamodb list-tables
aws kinesis list-streams
aws lambda list-functions --query 'Functions[?starts_with(FunctionName, `techno-`)]'
```

---

## 🎓 Checklist Juri (Self-Assessment)

| # | Item | Status |
|---|---|---|
| 1 | Region: `us-east-1` | ☐ |
| 2 | Semua resource di-provision via Terraform (bukan manual) | ☐ |
| 3 | Struktur Terraform rapi (main.tf, variables.tf, outputs.tf) | ☐ |
| 4 | VPC `techno-<nama>` dengan CIDR `25.1.0.0/16` | ☐ |
| 5 | 2 public + 2 private subnet di 2 AZ | ☐ |
| 6 | IGW + NAT GW + 2 route tables | ☐ |
| 7 | 2 Security Groups (`techno-sg-lb`, `techno-sg-apps`) | ☐ |
| 8 | S3 bucket `technoinput-<kota>-<nama>` dengan lifecycle (30d Glacier, 365d expire) | ☐ |
| 9 | DynamoDB table `Tokens` dengan PK `token` (String) | ☐ |
| 10 | DynamoDB Streams enabled (NEW_AND_OLD_IMAGES) | ☐ |
| 11 | Kinesis stream `techno-kinesis-<nama>` (1 shard, 1 day retention, PROVISIONED) | ☐ |
| 12 | DynamoDB → Kinesis streaming destination aktif | ☐ |
| 13 | Lambda `techno-lambda-s3` (Python 3.12, timeout 2 menit) | ☐ |
| 14 | IAM role dengan least privilege (S3, Rekognition, DynamoDB, Kinesis) | ☐ |
| 15 | S3 event trigger → Lambda aktif | ☐ |
| 16 | Upload gambar → Lambda ter-invoke → Rekognition jalan → DynamoDB tertulis | ☐ |
| 17 | DynamoDB changes → Kinesis stream terisi | ☐ |
| 18 | Semua resource di-tag (Project, Environment, Owner, ManagedBy) | ☐ |
| 19 | `terraform destroy` berhasil, tidak ada resource tersisa | ☐ |

---

## 💡 Tips Trainer

### 🔴 Jika `terraform apply` Error "AccessDenied"
**Penyebab**: Session token AWS Lab expired.
**Solusi**:
```bash
# Refresh credential di Lab Learner
aws configure set aws_session_token <NEW_SESSION_TOKEN>
terraform apply
```

### 🔴 Jika Lambda Error "AccessDenied" saat panggil Rekognition
**Penyebab**: IAM policy belum ter-attach dengan benar.
**Solusi**:
1. Cek `terraform state show aws_iam_role_policy.lambda_custom`
2. Pastikan Resource ARN sudah benar

### 🔴 Jika DynamoDB Stream tidak muncul di Kinesis
**Penyebab**: `aws_dynamodb_kinesis_streaming_destination` butuh waktu propagasi.
**Solusi**:
1. Tunggu ~1-2 menit
2. Cek status: `aws dynamodb kinesis-streaming-destination describe --table-name Tokens`
3. Harus `StreamStatus: ENABLED`

### 🔴 Jika `terraform destroy` stuck di NAT Gateway
**Penyebab**: NAT GW butuh waktu untuk release EIP.
**Solusi**:
1. Tunggu ~5 menit
2. Jika masih stuck, release EIP manual dulu:
```bash
aws ec2 release-address --allocation-id <EIP_ALLOC_ID>
terraform destroy
```

### 🔴 Jika Lambda tidak ter-trigger setelah upload S3
**Penyebab**: Event notification belum aktif atau bucket permission salah.
**Solusi**:
1. Cek `aws s3api get-bucket-notification-configuration --bucket technoinput-jakarta-budi`
2. Pastikan `LambdaFunctionConfigurations` ada
3. Cek CloudWatch Logs Lambda untuk error

---

## 📊 Apa yang Anda Pelajari dari Saran 2?

| Skill | Status |
|---|---|
| Terraform basics (provider, variables, resources) | ✅ |
| Terraform state management | ✅ |
| VPC dengan public + private subnet via Terraform | ✅ |
| S3 bucket dengan lifecycle & event notification via Terraform | ✅ |
| DynamoDB table + streams via Terraform | ✅ |
| **DynamoDB → Kinesis streaming destination** (skill baru!) | ✅ |
| Kinesis Data Stream (provisioned mode) via Terraform | ✅ |
| Lambda function dengan archive_file data source | ✅ |
| IAM role + inline policy via Terraform | ✅ |
| S3 → Lambda trigger via Terraform | ✅ |
| `terraform init/plan/apply/destroy` workflow | ✅ |
| Outputs & state inspection | ✅ |

---

## 🚀 Progression Path

```
✅ Saran 1 (Console, S3 → Lambda → Rekognition → DynamoDB) — 1-1.5 jam
    ↓
✅ Saran 2 (Terraform + DynamoDB Streams → Kinesis) — 2 jam ← ANDA DI SINI
    ↓
🟠 Saran 3 (Terraform + API Gateway + EKS Deployment) — 2.5 jam
    ↓
🔴 Saran 4 (Full: + Glue Crawler, SNS, IPv6, Strict Naming) — 4-5 jam
    ↓
🏆 SOAL LKS Machine Learning (6 jam, Terraform Wajib)
```

---

## 🎯 Analogi Langsung ke Soal LKS

| Yang Anda Lakukan di Saran 2 | Yang Akan Anda Lakukan di Soal LKS |
|---|---|
| Terraform dasar | **Terraform lengkap** (semua resource) |
| 1 bucket input | **2 bucket** (input + output) |
| DynamoDB → Kinesis | DynamoDB → Kinesis (**sama persis**) |
| 1 Lambda | **3 Lambda** (s3, post, get) |
| Tidak ada API | **API Gateway** (REST, dual-stack IPv4/IPv6) |
| Tidak ada container | **EKS** dengan Python app + LoadBalancer |
| Tidak ada analytics | **AWS Glue** (Database + Table + Crawler) |
| Tidak ada notification | **SNS** (email subscription) |
| Prefix `techno-` sederhana | Prefix `techno-` **ketat** untuk semua resource |

---

## 🎯 Next Step

Setelah Saran 2 berhasil, Anda siap lanjut ke **Saran 3**: menambahkan **API Gateway** (REST, dual-stack) + **ECR** + **EKS** dengan deployment via `kubectl`.

Apakah Anda ingin saya buatkan:
1. **Panduan lengkap Saran 3** (API Gateway + EKS + ECR)?
2. **Template Terraform module** untuk reusable components?
3. **Script testing otomatis** untuk memverifikasi semua resource?

Silakan pilih, dan saya akan buatkan! 🎯

**Selamat! Anda telah menguasai fondasi Infrastructure as Code untuk soal LKS Machine Learning.** 🎉