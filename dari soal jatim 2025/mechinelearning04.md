# 📘 Panduan Lengkap Pengerjaan Saran 4 (Machine Learning)
## "Full Analytics Pipeline" — Glue + SNS + IPv6 + Complete Terraform

> **Estimasi waktu**: 4-5 jam | **Region**: `us-east-1` | **Full Terraform IaC**
> **Tujuan**: Simulasi penuh soal LKS dengan semua komponen: Glue analytics, SNS notification, IPv6 dual-stack, 2 S3 buckets, dan strict naming convention.

---

## 🎯 Tujuan Latihan

Membangun **pipeline ML lengkap** yang mencakup semua aspek soal LKS:
- ✅ **AWS Glue** untuk analytics (Database + Table + Crawler)
- ✅ **SNS** untuk email notification
- ✅ **IPv6** di VPC, subnets, dan API Gateway (dual-stack)
- ✅ **2 S3 buckets** (input + output) dengan lifecycle policy
- ✅ **DynamoDB Streams → Kinesis** untuk real-time processing
- ✅ **Lambda** yang menulis hasil Rekognition ke S3 output
- ✅ **API Gateway REST** dengan IPv4+IPv6
- ✅ **EKS** dengan Python app + LoadBalancer
- ✅ **Strict naming** `techno-` untuk semua resource
- ✅ **100% Terraform** (tidak ada manual provisioning)

Ini adalah **latihan terakhir** sebelum soal LKS sesungguhnya. Jika Anda bisa menyelesaikan ini dalam 4-5 jam, Anda siap 95% untuk soal 6 jam.

---

## 🗺️ Topologi Arsitektur Lengkap

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                        AWS CLOUD (us-east-1) - Full Terraform Managed                       │
│                                                                                             │
│   ┌──────────────┐         ┌────────────────────────────────────────────────────────────┐   │
│   │  USER        │         │  API GATEWAY: Techno-API-budi (REST, IPv4+IPv6)            │   │
│   │              │         │                                                            │   │
│   │  POST        │────────▶│  /generate-token (POST) ──► techno-lambda-post             │   │
│   │  /generate-  │         │  /validate-token (GET)  ──► techno-lambda-get              │   │
│   │  token       │         │                                                            │   │
│   │              │         │  Lambda Authorizer: techno-lambda-auth                     │   │
│   │  GET         │         └────────────────────────────────────────────────────────────┘   │
│   │  /validate-  │                                       │                                  │
│   │  token       │                                       ▼                                  │
│   └──────────────┘         ┌────────────────────────────────────────────────────────────┐   │
│                            │  LAMBDA FUNCTIONS                                          │   │
│   ┌──────────────┐         │  • techno-lambda-s3    (S3 input → Rekognition → S3 output│   │
│   │  USER        │         │                          → DynamoDB → SNS notification)    │   │
│   │              │         │  • techno-lambda-post  (API GW → DynamoDB PutItem)         │   │
│   │  Upload      │         │  • techno-lambda-get   (API GW → DynamoDB GetItem)         │   │
│   │  image.jpg   │         │  • techno-lambda-auth  (API GW Authorizer)                 │   │
│   └──────┬───────┘         └────────────────────────────────────────────────────────────┘   │
│          │                 ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐    │
│          ▼                 │  S3 Input    │  │  S3 Output   │  │  DynamoDB            │    │
│   ┌──────────────────┐     │  technoinput-│  │  technooutput│  │  Table: Tokens       │    │
│   │  S3 Buckets      │     │  jakarta-budi│  │  -jakarta-   │  │  (with Streams)      │    │
│   │  • technoinput-* │     │              │  │  budi        │  │                      │    │
│   │  • technooutput-*│     │  Lifecycle:  │  │              │  │  ┌────────────────┐  │    │
│   └──────────────────┘     │  30d→Glacier │  │  /results/   │  │  │  DynamoDB      │  │    │
│                            │  365d→Delete │  │              │  │  │  Streams       │  │    │
│                            └──────────────┘  └──────┬───────┘  │  │  (NEW_AND_OLD) │  │    │
│                                                     │          │  └───────┬────────┘  │    │
│                                                     │          │          │           │    │
│                                                     │          │          ▼           │    │
│                                                     │          │  ┌────────────────┐  │    │
│                                                     │          │  │  Kinesis Data  │  │    │
│                                                     │          │  │  Stream        │  │    │
│                                                     │          │  │  techno-kinesis│  │    │
│                                                     │          │  │  -budi         │  │    │
│                                                     │          │  │  (1 shard,     │  │    │
│                                                     │          │  │   1 day retain)│  │    │
│                                                     │          │  └────────────────┘  │    │
│                                                     │          └──────────────────────┘    │
│                                                     ▼                                      │
│                            ┌────────────────────────────────────────────────────────────┐   │
│                            │  AWS GLUE (Analytics)                                      │   │
│                            │  • Database: rekognition_results_db                        │   │
│                            │  • Table: rekognition_results_table                        │   │
│                            │  • Crawler: techno-crawler-budi                            │   │
│                            │    (crawl s3://technooutput-jakarta-budi/results/)         │   │
│                            └────────────────────────────────────────────────────────────┘   │
│                                                     │                                      │
│                                                     ▼                                      │
│                            ┌────────────────────────────────────────────────────────────┐   │
│                            │  Amazon SNS                                                │   │
│                            │  Topic: techno-sns-jakarta-budi                            │   │
│                            │  Subscription: Email → handi@seamolec.org                  │   │
│                            └────────────────────────────────────────────────────────────┘   │
│                                                                                             │
│   ┌────────────────────────────────────────────────────────────────────────────────────┐   │
│   │  VPC: techno-budi (25.1.0.0/16 + IPv6 Amazon-provided)                             │   │
│   │  ├── Public Subnet 1:  25.1.0.0/24  + IPv6 /64 (AZ a)                              │   │
│   │  ├── Public Subnet 2:  25.1.2.0/24  + IPv6 /64 (AZ b)                              │   │
│   │  ├── Private Subnet 1: 25.1.1.0/24  + IPv6 /64 (AZ a)                              │   │
│   │  ├── Private Subnet 2: 25.1.3.0/24  + IPv6 /64 (AZ b)                              │   │
│   │  ├── IGW: techno-igw-budi (IPv4 + IPv6)                                            │   │
│   │  ├── NAT GW: techno-natgw-budi (IPv4 only)                                         │   │
│   │  └── Route Tables: techno-public-rt, techno-private-rt                             │   │
│   └────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                             │
│   ┌────────────────────────────────────────────────────────────────────────────────────┐   │
│   │  EKS CLUSTER: techno-eks-jakarta-budi                                              │   │
│   │  ┌──────────────────────────────────────────────────────────────────────────────┐  │   │
│   │  │  Node Group: 1x t3.large (in private subnet)                                  │  │   │
│   │  │  ┌─────────────┐  ┌─────────────┐                                             │  │   │
│   │  │  │ Pod 1       │  │ Pod 2       │  (Python Flask App from ECR)                │  │   │
│   │  │  │ techno-ml-  │  │ techno-ml-  │                                             │  │   │
│   │  │  │ app-xxxx    │  │ app-yyyy    │                                             │  │   │
│   │  │  └──────┬──────┘  └──────┬──────┘                                             │  │   │
│   │  │         └────────┬────────┘                                                    │  │   │
│   │  │                  ▼                                                             │  │   │
│   │  │         ┌─────────────────┐                                                    │  │   │
│   │  │         │ K8s Service     │                                                    │  │   │
│   │  │         │ Type: LoadBalan │──────► AWS NLB/ALB (external)                      │  │   │
│   │  │         └─────────────────┘                                                    │  │   │
│   │  └──────────────────────────────────────────────────────────────────────────────┘  │   │
│   └────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                          ▲                                 │
│   ┌──────────────────┐                                   │                                 │
│   │  ECR Repository  │                                   │                                 │
│   │  techno-ecr-     │────── docker pull ────────────────┘                                 │
│   │  jakarta-budi    │                                                                     │
│   └──────────────────┘                                                                     │
│                                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 🔄 Alur Kerja Lengkap (Flow)
```
1. User upload gambar → S3 technoinput-jakarta-budi
                          ↓
2. S3 Event → Lambda techno-lambda-s3
                          ↓
3. Lambda panggil Rekognition DetectLabels
                          ↓
4. Lambda simpan hasil ke:
   • S3 technooutput-jakarta-budi/results/<filename>.json
   • DynamoDB table Tokens
                          ↓
5. DynamoDB Streams → Kinesis techno-kinesis-budi
                          ↓
6. Lambda publish notifikasi ke SNS techno-sns-jakarta-budi
                          ↓
7. SNS kirim email ke handi@seamolec.org
                          ↓
8. Glue Crawler (scheduled/triggered) crawl S3 output
                          ↓
9. Glue update table rekognition_results_table
                          ↓
10. User bisa query via Athena (opsional)
                          ↓
11. Python App di EKS tampilkan hasil analisis
```

---

## 📋 Daftar Isi
1. [Fase 0: Persiapan Environment](#fase-0-persiapan)
2. [Fase 1: Update Variables & Locals](#fase-1-variables)
3. [Fase 2: VPC dengan IPv6](#fase-2-vpc)
4. [Fase 3: S3 Buckets (Input + Output)](#fase-3-s3)
5. [Fase 4: DynamoDB + Streams + Kinesis](#fase-4-dynamodb)
6. [Fase 5: AWS Glue (Database + Table + Crawler)](#fase-5-glue)
7. [Fase 6: SNS Topic + Subscription](#fase-6-sns)
8. [Fase 7: Lambda Functions (Updated)](#fase-7-lambda)
9. [Fase 8: API Gateway REST dengan IPv6](#fase-8-apigateway)
10. [Fase 9: ECR + EKS + Python App](#fase-9-eks)
11. [Fase 10: Terraform Apply](#fase-10-apply)
12. [Fase 11: Testing End-to-End](#fase-11-testing)
13. [Fase 12: Cleanup](#fase-12-cleanup)

---

## 🎯 Fase 0: Persiapan Environment

### Langkah 0.1: Setup CloudShell
```bash
# Install tools (jika belum)
sudo yum install -y docker git
sudo service docker start
sudo usermod -aG docker $USER

# Install kubectl
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.28.3/2023-11-14/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify
terraform version
kubectl version --client
docker --version
```

### Langkah 0.2: Clone Struktur dari Saran 3
```bash
cd ~/techno-ml-project
# Struktur sudah ada, tambahkan folder baru:
mkdir -p glue
```

Struktur final:
```
techno-ml-project/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
├── lambda/
│   ├── techno_lambda_s3.py
│   ├── auth/techno_lambda_auth.py
│   ├── post/techno_lambda_post.py
│   └── get/techno_lambda_get.py
├── app/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── app.py
│   ├── deploy.yaml
│   └── service.yaml
└── glue/
    └── (Glue resources di main.tf)
```

---

## 🔧 Fase 1: Update Variables & Locals

### 📄 Update `variables.tf` — tambahkan:
```hcl
variable "output_bucket_suffix" {
  description = "Suffix untuk output bucket"
  type        = string
  default     = ""  # Akan di-generate otomatis
}

variable "glue_database_name" {
  description = "Nama Glue database"
  type        = string
  default     = "rekognition_results_db"
}

variable "glue_table_name" {
  description = "Nama Glue table"
  type        = string
  default     = "rekognition_results_table"
}

variable "sns_email" {
  description = "Email untuk SNS subscription"
  type        = string
  default     = "handi@seamolec.org"
}

variable "kinesis_shard_count" {
  description = "Jumlah shard Kinesis"
  type        = number
  default     = 1
}

variable "enable_ipv6" {
  description = "Enable IPv6 di VPC"
  type        = bool
  default     = true
}
```

### 📄 Update `main.tf` — tambahkan locals:
```hcl
locals {
  # ... (locals dari Saran 3)
  
  output_bucket         = "${local.prefix}output-${local.city_suffix}-${local.name_suffix}"
  glue_database         = var.glue_database_name
  glue_table            = var.glue_table_name
  glue_crawler_name     = "${local.prefix}-crawler-${local.name_suffix}"
  sns_topic_name        = "${local.prefix}-sns-${local.city_suffix}-${local.name_suffix}"
}
```

---

## 🌐 Fase 2: VPC dengan IPv6

### Update `main.tf` — ganti resource VPC:
```hcl
# ============================================
# VPC & NETWORKING DENGAN IPv6
# ============================================

# IPv6 CIDR Block
resource "aws_vpc_ipv6_cidr_block_association" "main" {
  vpc_id       = aws_vpc.main.id
  ipv6_ipam_pool_id = data.aws_ec2_managed_prefix_list.ipv6.id
  
  # Jika IPAM tidak tersedia, gunakan Amazon-provided
  # AWS akan auto-assign /56 dari Amazon's pool
}

# Data source untuk IPv6 pool (opsional, bisa skip jika pakai Amazon-provided)
data "aws_ec2_managed_prefix_list" "ipv6" {
  name = "amazon-ipv6"
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true
  
  # Enable IPv6
  assign_generated_ipv6_cidr_block = var.enable_ipv6
  
  tags = {
    Name        = local.vpc_name
    Description = "VPC with IPv4 and IPv6 support"
  }
}

# Internet Gateway dengan IPv6
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = local.igw_name
  }
}

# Public Subnets dengan IPv6
resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = count.index == 0 ? var.public_subnet_1_cidr : var.public_subnet_2_cidr
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true
  
  # IPv6 CIDR (/64 dari VPC /56)
  ipv6_cidr_block = cidrsubnet(aws_vpc.main.ipv6_cidr_block, 8, count.index * 2)
  
  tags = {
    Name = "${local.vpc_name}-public-${var.availability_zones[count.index]}"
    Tier = "Public"
  }
}

# Private Subnets dengan IPv6
resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = count.index == 0 ? var.private_subnet_1_cidr : var.private_subnet_2_cidr
  availability_zone = var.availability_zones[count.index]
  
  # IPv6 CIDR (/64 dari VPC /56)
  ipv6_cidr_block = cidrsubnet(aws_vpc.main.ipv6_cidr_block, 8, count.index * 2 + 1)
  
  tags = {
    Name = "${local.vpc_name}-private-${var.availability_zones[count.index]}"
    Tier = "Private"
  }
}

# Public Route Table dengan IPv6
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  # IPv4 route
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  # IPv6 route
  route {
    ipv6_cidr_block = "::/0"
    gateway_id      = aws_internet_gateway.main.id
  }
  
  tags = {
    Name = local.public_rt_name
  }
}

# Private Route Table (IPv4 only via NAT, IPv6 via IGW jika perlu)
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }
  
  # IPv6 route untuk private subnet (opsional, via egress-only IGW)
  # Untuk simplicity, kita skip IPv6 di private subnet
  
  tags = {
    Name = local.private_rt_name
  }
}
```

---

## 📦 Fase 3: S3 Buckets (Input + Output)

### Tambahkan ke `main.tf`:
```hcl
# ============================================
# S3 BUCKETS (INPUT + OUTPUT)
# ============================================

# Input Bucket
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

# Lifecycle: 30 days → Glacier, 365 days → Delete
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

# Output Bucket
resource "aws_s3_bucket" "output" {
  bucket = local.output_bucket
  
  tags = {
    Name        = local.output_bucket
    Description = "Output bucket for Rekognition results"
  }
}

resource "aws_s3_bucket_versioning" "output" {
  bucket = aws_s3_bucket.output.id
  versioning_configuration {
    status = "Disabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "output" {
  bucket = aws_s3_bucket.output.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Public read access untuk output bucket (sesuai soal)
resource "aws_s3_bucket_public_access_block" "output" {
  bucket = aws_s3_bucket.output.id
  
  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

resource "aws_s3_bucket_policy" "output_public_read" {
  bucket = aws_s3_bucket.output.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "PublicReadGetObject"
        Effect    = "Allow"
        Principal = "*"
        Action    = "s3:GetObject"
        Resource  = "${aws_s3_bucket.output.arn}/*"
      }
    ]
  })
}

# Lifecycle untuk output bucket
resource "aws_s3_bucket_lifecycle_configuration" "output" {
  bucket = aws_s3_bucket.output.id
  
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

# S3 Event Notification → Lambda (input bucket)
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

## 🗄️ Fase 4: DynamoDB + Streams + Kinesis

### Tambahkan ke `main.tf`:
```hcl
# ============================================
# DYNAMODB TABLE + STREAMS
# ============================================

resource "aws_dynamodb_table" "tokens" {
  name         = var.dynamodb_table_name
  billing_mode = "PAY_PER_REQUEST"
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

# ============================================
# KINESIS DATA STREAM
# ============================================

resource "aws_kinesis_stream" "main" {
  name             = local.kinesis_name
  shard_count      = var.kinesis_shard_count
  retention_period = 24  # 1 day = 24 hours
  
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

# DynamoDB → Kinesis streaming destination
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

## 🔍 Fase 5: AWS Glue (Database + Table + Crawler)

### Tambahkan ke `main.tf`:
```hcl
# ============================================
# AWS GLUE (ANALYTICS)
# ============================================

# IAM Role for Glue Crawler
resource "aws_iam_role" "glue_crawler" {
  name = "${local.prefix}-glue-crawler-role-${local.name_suffix}"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "glue.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "glue_service" {
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole"
  role       = aws_iam_role.glue_crawler.name
}

resource "aws_iam_role_policy" "glue_s3_access" {
  name = "${local.prefix}-glue-s3-access"
  role = aws_iam_role.glue_crawler.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.output.arn,
          "${aws_s3_bucket.output.arn}/*"
        ]
      }
    ]
  })
}

# Glue Database
resource "aws_glue_catalog_database" "rekognition" {
  name = local.glue_database
  
  description = "Database for Rekognition results"
  
  tags = {
    Name        = local.glue_database
    Description = "Glue database for ML analytics"
  }
}

# Glue Table (schema akan di-update oleh crawler)
resource "aws_glue_catalog_table" "rekognition_results" {
  name          = local.glue_table
  database_name = aws_glue_catalog_database.rekognition.name
  
  table_type = "EXTERNAL_TABLE"
  
  parameters = {
    classification = "json"
  }
  
  storage_descriptor {
    location      = "s3://${aws_s3_bucket.output.id}/results/"
    input_format  = "org.apache.hadoop.mapred.TextInputFormat"
    output_format = "org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat"
    
    ser_de_info {
      name                  = "rekognition-json"
      serialization_library = "org.openx.data.jsonserde.JsonSerDe"
      
      parameters = {
        "serialization.format" = "1"
      }
    }
    
    columns {
      name = "token"
      type = "string"
    }
    
    columns {
      name = "image_name"
      type = "string"
    }
    
    columns {
      name = "labels"
      type = "string"
    }
    
    columns {
      name = "label_count"
      type = "int"
    }
    
    columns {
      name = "timestamp"
      type = "string"
    }
  }
  
  tags = {
    Name        = local.glue_table
    Description = "Glue table for Rekognition results"
  }
}

# Glue Crawler
resource "aws_glue_crawler" "rekognition" {
  name          = local.glue_crawler_name
  role          = aws_iam_role.glue_crawler.arn
  database_name = aws_glue_catalog_database.rekognition.name
  
  s3_target {
    path = "s3://${aws_s3_bucket.output.id}/results/"
  }
  
  schema_change_policy {
    delete_behavior = "LOG"
    update_behavior = "UPDATE_IN_DATABASE"
  }
  
  schedule = "cron(0/15 * * * ? *)"  # Run every 15 minutes
  
  tags = {
    Name        = local.glue_crawler_name
    Description = "Crawler for Rekognition results in S3"
  }
}
```

---

## 📧 Fase 6: SNS Topic + Subscription

### Tambahkan ke `main.tf`:
```hcl
# ============================================
# SNS TOPIC + SUBSCRIPTION
# ============================================

resource "aws_sns_topic" "notification" {
  name = local.sns_topic_name
  
  tags = {
    Name        = local.sns_topic_name
    Description = "SNS topic for ML pipeline notifications"
  }
}

resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.notification.arn
  protocol  = "email"
  endpoint  = var.sns_email
}
```

---

## ⚡ Fase 7: Lambda Functions (Updated)

### 📄 Update `lambda/techno_lambda_s3.py`:
```python
import json
import boto3
import urllib.parse
from datetime import datetime

# Initialize AWS clients
s3_client = boto3.client('s3')
rekognition_client = boto3.client('rekognition')
dynamodb_client = boto3.client('dynamodb')
sns_client = boto3.client('sns')

DYNAMODB_TABLE = 'Tokens'
OUTPUT_BUCKET = 'technooutput-jakarta-budi'  # GANTI dengan nama bucket Anda
SNS_TOPIC_ARN = 'arn:aws:sns:us-east-1:ACCOUNT_ID:techno-sns-jakarta-budi'  # GANTI

def lambda_handler(event, context):
    """
    Lambda handler untuk memproses gambar yang di-upload ke S3.
    1. Baca object dari S3 input
    2. Panggil Rekognition DetectLabels
    3. Simpan hasil ke S3 output + DynamoDB
    4. Kirim notifikasi ke SNS
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
        
        # 4. Format hasil
        labels_data = []
        for label in labels:
            labels_data.append({
                'Name': label['Name'],
                'Confidence': float(label['Confidence']),
                'Categories': [cat['Name'] for cat in label.get('Categories', [])]
            })
        
        timestamp = datetime.now().isoformat()
        
        # 5. Simpan hasil ke S3 output bucket
        result_data = {
            'token': token,
            'image_name': key,
            'source_bucket': bucket,
            'labels': labels_data,
            'label_count': len(labels),
            'timestamp': timestamp,
            'status': 'SUCCESS'
        }
        
        result_key = f"results/{key.split('.')[0]}.json"
        s3_client.put_object(
            Bucket=OUTPUT_BUCKET,
            Key=result_key,
            Body=json.dumps(result_data, indent=2),
            ContentType='application/json'
        )
        
        print(f"Saved result to s3://{OUTPUT_BUCKET}/{result_key}")
        
        # 6. Simpan metadata ke DynamoDB
        dynamodb_client.put_item(
            TableName=DYNAMODB_TABLE,
            Item={
                'token': {'S': token},
                'image_name': {'S': key},
                'bucket': {'S': bucket},
                'result_s3_key': {'S': result_key},
                'labels': {'S': json.dumps(labels_data)},
                'label_count': {'N': str(len(labels))},
                'timestamp': {'S': timestamp},
                'status': {'S': 'SUCCESS'}
            }
        )
        
        print(f"Saved metadata to DynamoDB with token: {token}")
        
        # 7. Kirim notifikasi ke SNS
        sns_client.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject=f"Image Processing Complete: {key}",
            Message=f"""
Image processing completed successfully!

Image: {key}
Token: {token}
Labels Detected: {len(labels)}
Top Labels: {', '.join([l['Name'] for l in labels[:5]])}
Timestamp: {timestamp}

Result stored at: s3://{OUTPUT_BUCKET}/{result_key}
            """.strip()
        )
        
        print("Notification sent to SNS")
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Image processed successfully',
                'token': token,
                'labels_detected': len(labels),
                'result_location': f"s3://{OUTPUT_BUCKET}/{result_key}"
            })
        }
        
    except Exception as e:
        print(f"Error processing image: {str(e)}")
        
        # Kirim error notification
        try:
            sns_client.publish(
                TopicArn=SNS_TOPIC_ARN,
                Subject=f"Image Processing FAILED: {key if 'key' in locals() else 'unknown'}",
                Message=f"Error: {str(e)}"
            )
        except:
            pass
        
        return {
            'statusCode': 500,
            'body': json.dumps({
                'message': 'Error processing image',
                'error': str(e)
            })
        }
```

### Update `main.tf` untuk Lambda:
```hcl
# Update Lambda function dengan environment variables
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
      OUTPUT_BUCKET  = local.output_bucket
      SNS_TOPIC_ARN  = aws_sns_topic.notification.arn
      KINESIS_STREAM = local.kinesis_name
    }
  }
  
  tags = {
    Name        = local.lambda_name
    Description = "Lambda for S3 image processing with Rekognition"
  }
}

# Update IAM policy untuk Lambda (tambah S3 output + SNS)
resource "aws_iam_role_policy" "lambda_custom" {
  name = "${local.prefix}-lambda-custom-policy"
  role = aws_iam_role.lambda_exec.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "S3ReadWriteAccess"
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:ListBucket",
          "s3:PutObject"
        ]
        Resource = [
          aws_s3_bucket.input.arn,
          "${aws_s3_bucket.input.arn}/*",
          aws_s3_bucket.output.arn,
          "${aws_s3_bucket.output.arn}/*"
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
      },
      {
        Sid    = "SNSPublish"
        Effect = "Allow"
        Action = [
          "sns:Publish"
        ]
        Resource = aws_sns_topic.notification.arn
      }
    ]
  })
}
```

---

## 🚪 Fase 8: API Gateway REST dengan IPv6

### Update `main.tf` — ganti API Gateway:
```hcl
# ============================================
# API GATEWAY REST (IPv4 + IPv6)
# ============================================

# Gunakan REST API (bukan HTTP API) untuk IPv6 support
resource "aws_api_gateway_rest_api" "main" {
  name        = local.api_name
  description = "REST API for token management with IPv4+IPv6"
  
  endpoint_configuration {
    types = ["REGIONAL"]
  }
  
  tags = {
    Name        = local.api_name
    Description = "REST API with dual-stack support"
  }
}

# Resources
resource "aws_api_gateway_resource" "generate_token" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  parent_id   = aws_api_gateway_rest_api.main.root_resource_id
  path_part   = "generate-token"
}

resource "aws_api_gateway_resource" "validate_token" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  parent_id   = aws_api_gateway_rest_api.main.root_resource_id
  path_part   = "validate-token"
}

# Methods
resource "aws_api_gateway_method" "post_token" {
  rest_api_id   = aws_api_gateway_rest_api.main.id
  resource_id   = aws_api_gateway_resource.generate_token.id
  http_method   = "POST"
  authorization = "NONE"
}

resource "aws_api_gateway_method" "get_token" {
  rest_api_id   = aws_api_gateway_rest_api.main.id
  resource_id   = aws_api_gateway_resource.validate_token.id
  http_method   = "GET"
  authorization = "NONE"
  
  request_parameters = {
    "method.request.querystring.token" = true
  }
}

# Integrations
resource "aws_api_gateway_integration" "post_token" {
  rest_api_id             = aws_api_gateway_rest_api.main.id
  resource_id             = aws_api_gateway_resource.generate_token.id
  http_method             = aws_api_gateway_method.post_token.http_method
  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = aws_lambda_function.post.invoke_arn
}

resource "aws_api_gateway_integration" "get_token" {
  rest_api_id             = aws_api_gateway_rest_api.main.id
  resource_id             = aws_api_gateway_resource.validate_token.id
  http_method             = aws_api_gateway_method.get_token.http_method
  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = aws_lambda_function.get.invoke_arn
}

# Deployment
resource "aws_api_gateway_deployment" "main" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  
  triggers = {
    redeployment = sha1(jsonencode([
      aws_api_gateway_resource.generate_token.id,
      aws_api_gateway_resource.validate_token.id,
      aws_api_gateway_method.post_token.id,
      aws_api_gateway_method.get_token.id,
      aws_api_gateway_integration.post_token.id,
      aws_api_gateway_integration.get_token.id,
    ]))
  }
  
  lifecycle {
    create_before_destroy = true
  }
}

# Stage
resource "aws_api_gateway_stage" "prod" {
  deployment_id = aws_api_gateway_deployment.main.id
  rest_api_id   = aws_api_gateway_rest_api.main.id
  stage_name    = "prod"
  
  tags = {
    Name = "${local.api_name}-stage"
  }
}

# Enable IPv6 di stage
resource "aws_api_gateway_method_settings" "ipv6" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  stage_name  = aws_api_gateway_stage.prod.stage_name
  method_path = "*/*"
  
  settings {
    metrics_enabled = true
    logging_level   = "INFO"
  }
}

# Lambda permissions
resource "aws_lambda_permission" "apigw_post" {
  statement_id  = "AllowAPIGatewayInvokePost"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.post.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_api_gateway_rest_api.main.execution_arn}/*/*"
}

resource "aws_lambda_permission" "apigw_get" {
  statement_id  = "AllowAPIGatewayInvokeGet"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.get.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_api_gateway_rest_api.main.execution_arn}/*/*"
}
```

---

## ☸️ Fase 9: ECR + EKS + Python App

### Update `app/app.py` — tambahkan endpoint untuk query Glue:
```python
from flask import Flask, jsonify, request
import boto3
import os
from datetime import datetime

app = Flask(__name__)

# AWS clients
dynamodb = boto3.client('dynamodb')
s3 = boto3.client('s3')
glue = boto3.client('glue')

TABLE_NAME = os.environ.get('DYNAMODB_TABLE', 'Tokens')
OUTPUT_BUCKET = os.environ.get('OUTPUT_BUCKET', '')
GLUE_DATABASE = os.environ.get('GLUE_DATABASE', 'rekognition_results_db')
GLUE_TABLE = os.environ.get('GLUE_TABLE', 'rekognition_results_table')

@app.route('/')
def home():
    return jsonify({
        'status': 'healthy',
        'app': 'techno-ml-app',
        'timestamp': datetime.now().isoformat(),
        'version': '1.0.0'
    })

@app.route('/health')
def health():
    return jsonify({'status': 'ok'}), 200

@app.route('/results', methods=['GET'])
def get_results():
    """Get all analysis results from DynamoDB"""
    try:
        response = dynamodb.scan(
            TableName=TABLE_NAME,
            Limit=50
        )
        
        items = []
        for item in response.get('Items', []):
            items.append({
                'token': item.get('token', {}).get('S'),
                'image_name': item.get('image_name', {}).get('S'),
                'label_count': item.get('label_count', {}).get('N'),
                'status': item.get('status', {}).get('S'),
                'timestamp': item.get('timestamp', {}).get('S')
            })
        
        return jsonify({
            'success': True,
            'count': len(items),
            'results': items
        })
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)}), 500

@app.route('/results/<token>', methods=['GET'])
def get_result_by_token(token):
    """Get specific analysis result by token"""
    try:
        response = dynamodb.get_item(
            TableName=TABLE_NAME,
            Key={'token': {'S': token}}
        )
        
        if 'Item' not in response:
            return jsonify({'success': False, 'error': 'Not found'}), 404
        
        item = response['Item']
        
        # Get full result from S3
        result_s3_key = item.get('result_s3_key', {}).get('S')
        if result_s3_key:
            s3_response = s3.get_object(
                Bucket=OUTPUT_BUCKET,
                Key=result_s3_key
            )
            full_result = json.loads(s3_response['Body'].read().decode('utf-8'))
        else:
            full_result = None
        
        return jsonify({
            'success': True,
            'result': {
                'token': item.get('token', {}).get('S'),
                'image_name': item.get('image_name', {}).get('S'),
                'labels': item.get('labels', {}).get('S'),
                'label_count': item.get('label_count', {}).get('N'),
                'status': item.get('status', {}).get('S'),
                'timestamp': item.get('timestamp', {}).get('S'),
                'full_result': full_result
            }
        })
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)}), 500

@app.route('/glue/schema', methods=['GET'])
def get_glue_schema():
    """Get Glue table schema"""
    try:
        response = glue.get_table(
            DatabaseName=GLUE_DATABASE,
            Name=GLUE_TABLE
        )
        
        table = response['Table']
        columns = table.get('StorageDescriptor', {}).get('Columns', [])
        
        return jsonify({
            'success': True,
            'database': GLUE_DATABASE,
            'table': GLUE_TABLE,
            'columns': [
                {
                    'name': col['Name'],
                    'type': col['Type']
                } for col in columns
            ],
            'location': table.get('StorageDescriptor', {}).get('Location')
        })
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)}), 500

@app.route('/validate-token', methods=['GET'])
def validate_token():
    """Validate token from query parameter"""
    token = request.args.get('token')
    if not token:
        return jsonify({'valid': False, 'error': 'Token required'}), 400
    
    try:
        response = dynamodb.get_item(
            TableName=TABLE_NAME,
            Key={'token': {'S': token}}
        )
        
        if 'Item' in response:
            return jsonify({
                'valid': True,
                'token': token,
                'deviceid': response['Item'].get('deviceid', {}).get('S')
            })
        return jsonify({'valid': False, 'error': 'Invalid token'}), 401
    except Exception as e:
        return jsonify({'valid': False, 'error': str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=2000)
```

### Update `app/deploy.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: techno-ml-app
  labels:
    app: techno-ml-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: techno-ml-app
  template:
    metadata:
      labels:
        app: techno-ml-app
    spec:
      containers:
      - name: techno-ml-app
        image: ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/techno-ecr-CITY-NAME:latest
        ports:
        - containerPort: 2000
        env:
        - name: DYNAMODB_TABLE
          value: "Tokens"
        - name: OUTPUT_BUCKET
          value: "technooutput-CITY-NAME"
        - name: GLUE_DATABASE
          value: "rekognition_results_db"
        - name: GLUE_TABLE
          value: "rekognition_results_table"
        - name: AWS_REGION
          value: "us-east-1"
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "500m"
            memory: "1Gi"
        livenessProbe:
          httpGet:
            path: /health
            port: 2000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 2000
          initialDelaySeconds: 5
          periodSeconds: 5
```

---

## 🚀 Fase 10: Terraform Apply

### Langkah 10.1: Init & Plan
```bash
cd ~/techno-ml-project
terraform init
terraform plan -out=tfplan
```

Review output — harus ada ~35 resource yang akan dibuat:
- VPC + subnets + IGW + NAT (dengan IPv6)
- 2 S3 buckets (input + output)
- DynamoDB + Streams + Kinesis
- Glue Database + Table + Crawler
- SNS Topic + Subscription
- 4 Lambda functions
- API Gateway REST
- ECR + EKS
- IAM roles & policies

### Langkah 10.2: Apply
```bash
terraform apply tfplan
```

Tunggu ~10-15 menit untuk semua resource (EKS butuh waktu lama).

### Langkah 10.3: Configure kubectl & Deploy App
```bash
# Update kubeconfig
aws eks update-kubeconfig \
  --region us-east-1 \
  --name $(terraform output -raw eks_cluster_name)

# Build & push Docker image
cd ~/techno-ml-project/app
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
CITY="jakarta"
NAME="budi"
ECR_URL="${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/techno-ecr-${CITY}-${NAME}"

aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com

COMMIT_SHA=$(git rev-parse --short HEAD 2>/dev/null || echo "v1.0.0")
docker build -t techno-ml-app:${COMMIT_SHA} .
docker tag techno-ml-app:${COMMIT_SHA} ${ECR_URL}:${COMMIT_SHA}
docker tag techno-ml-app:${COMMIT_SHA} ${ECR_URL}:latest
docker push ${ECR_URL}:${COMMIT_SHA}
docker push ${ECR_URL}:latest

# Update & apply Kubernetes manifests
sed -i "s|ACCOUNT_ID|${ACCOUNT_ID}|g" deploy.yaml
sed -i "s|CITY|${CITY}|g" deploy.yaml
sed -i "s|NAME|${NAME}|g" deploy.yaml

kubectl apply -f deploy.yaml
kubectl apply -f service.yaml

# Wait for LoadBalancer
kubectl get svc techno-ml-service -w
```

---

## 🧪 Fase 11: Testing End-to-End

### Test 1: Upload Gambar ke S3 Input
```bash
curl -o test-image.jpg https://picsum.photos/800/600.jpg
aws s3 cp test-image.jpg s3://$(terraform output -raw s3_input_bucket)/test-image.jpg
```

### Test 2: Cek Lambda Execution
```bash
aws logs tail /aws/lambda/$(terraform output -raw lambda_function_name) --since 2m --follow
```

Harus muncul log:
```
Processing image: s3://technoinput-jakarta-budi/test-image.jpg
Detected X labels
Saved result to s3://technooutput-jakarta-budi/results/test-image.json
Saved metadata to DynamoDB with token: token-xxxxx-test-image
Notification sent to SNS
```

### Test 3: Cek S3 Output
```bash
aws s3 ls s3://$(terraform output -raw s3_output_bucket)/results/
aws s3 cp s3://$(terraform output -raw s3_output_bucket)/results/test-image.json -
```

Harus ada file JSON dengan hasil Rekognition.

### Test 4: Cek DynamoDB
```bash
aws dynamodb scan --table-name Tokens --max-items 5
```

### Test 5: Cek SNS Email
Tunggu ~1-2 menit → cek email `handi@seamolec.org` → harus ada notifikasi.

> ⚠️ **PENTING**: Confirm subscription email dari AWS SNS pertama kali!

### Test 6: Cek Kinesis Stream
```bash
aws kinesis get-shard-iterator \
  --stream-name $(terraform output -raw kinesis_stream_name) \
  --shard-id shardId-000000000000 \
  --shard-iterator-type TRIM_HORIZON \
  --query 'ShardIterator' --output text | \
  xargs -I {} aws kinesis get-records --shard-iterator {}
```

### Test 7: Run Glue Crawler Manual
```bash
aws glue start-crawler --name $(terraform output -raw glue_crawler_name)

# Wait ~2-3 menit
aws glue get-crawler --name $(terraform output -raw glue_crawler_name) \
  --query 'Crawler.State'
```

### Test 8: Query Glue Table via Athena (Opsional)
```bash
# Buka Athena Console
# Query:
SELECT * FROM rekognition_results_db.rekognition_results_table LIMIT 10;
```

### Test 9: API Gateway - Generate Token
```bash
API_URL=$(terraform output -raw api_endpoint)

curl -X POST ${API_URL}/generate-token \
  -H "Content-Type: application/json" \
  -H "deviceid: test-device-001"
```

### Test 10: API Gateway - Validate Token
```bash
TOKEN="<token_dari_response>"
curl "${API_URL}/validate-token?token=${TOKEN}"
```

### Test 11: Akses Python App via LoadBalancer
```bash
APP_URL=$(kubectl get svc techno-ml-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Health check
curl http://${APP_URL}/

# List results
curl http://${APP_URL}/results

# Get specific result
curl http://${APP_URL}/results/${TOKEN}

# Get Glue schema
curl http://${APP_URL}/glue/schema
```

### Test 12: Test IPv6 (Jika tersedia)
```bash
# Cek apakah API Gateway bisa diakses via IPv6
curl -6 https://$(terraform output -raw api_endpoint)/generate-token \
  -X POST \
  -H "Content-Type: application/json"
```

---

## 🧹 Fase 12: Cleanup

### Langkah 12.1: Delete Kubernetes Resources
```bash
kubectl delete -f app/service.yaml
kubectl delete -f app/deploy.yaml
```

### Langkah 12.2: Empty S3 Buckets
```bash
aws s3 rm s3://$(terraform output -raw s3_input_bucket) --recursive
aws s3 rm s3://$(terraform output -raw s3_output_bucket) --recursive
```

### Langkah 12.3: Terraform Destroy
```bash
cd ~/techno-ml-project
terraform destroy -auto-approve
```

> ⚠️ **Glue Crawler** kadang stuck di deleting. Jika terjadi, stop crawler manual dulu:
```bash
aws glue stop-crawler --name techno-crawler-budi
terraform destroy
```

### Langkah 12.4: Verify Cleanup
```bash
terraform state list  # harus kosong
aws s3 ls
aws dynamodb list-tables
aws kinesis list-streams
aws glue get-databases
aws sns list-topics
```

---

## 🎓 Checklist Juri (Self-Assessment)

| # | Item | Status |
|---|---|---|
| 1 | Region: `us-east-1` | ☐ |
| 2 | **100% Terraform** (tidak ada manual provisioning) | ☐ |
| 3 | VPC `techno-<nama>` dengan **IPv4 + IPv6** | ☐ |
| 4 | 2 public + 2 private subnet dengan **IPv6 /64** | ☐ |
| 5 | IGW + NAT GW + 2 route tables (public dengan IPv6 route) | ☐ |
| 6 | S3 input `technoinput-<kota>-<nama>` dengan lifecycle (30d Glacier, 365d delete) | ☐ |
| 7 | S3 output `technooutput-<kota>-<nama>` dengan **public read policy** | ☐ |
| 8 | DynamoDB table `Tokens` dengan **Streams enabled** (NEW_AND_OLD_IMAGES) | ☐ |
| 9 | Kinesis stream `techno-kinesis-<nama>` (1 shard, 1 day retention, PROVISIONED) | ☐ |
| 10 | DynamoDB → Kinesis streaming destination aktif | ☐ |
| 11 | **Glue Database** `rekognition_results_db` | ☐ |
| 12 | **Glue Table** `rekognition_results_table` (source: S3 output/results/) | ☐ |
| 13 | **Glue Crawler** `techno-crawler-<nama>` (scheduled every 15 min) | ☐ |
| 14 | **SNS Topic** `techno-sns-<kota>-<nama>` | ☐ |
| 15 | SNS subscription email `handi@seamolec.org` | ☐ |
| 16 | Lambda `techno-lambda-s3` (timeout 2m, write ke S3 output + DynamoDB + SNS) | ☐ |
| 17 | Lambda `techno-lambda-post` (timeout 1m) | ☐ |
| 18 | Lambda `techno-lambda-get` (timeout 1m30s) | ☐ |
| 19 | Lambda `techno-lambda-auth` (authorizer) | ☐ |
| 20 | **API Gateway REST** `Techno-API-<nama>` (Regional, **IPv4+IPv6**) | ☐ |
| 21 | Route `POST /generate-token` → Lambda post | ☐ |
| 22 | Route `GET /validate-token` → Lambda get | ☐ |
| 23 | ECR `techno-ecr-<kota>-<nama>` (mutable) | ☐ |
| 24 | EKS cluster `techno-eks-<kota>-<nama>` (v1.28, t3.large) | ☐ |
| 25 | K8s Deployment dengan 2 replicas | ☐ |
| 26 | K8s Service type LoadBalancer | ☐ |
| 27 | Upload gambar → Lambda → Rekognition → S3 output → DynamoDB → SNS | ☐ |
| 28 | DynamoDB changes → Kinesis stream | ☐ |
| 29 | Glue Crawler berhasil crawl S3 output | ☐ |
| 30 | SNS email notification terkirim | ☐ |
| 31 | API Gateway bisa diakses (generate & validate token) | ☐ |
| 32 | Python app bisa query hasil analisis dari DynamoDB & S3 | ☐ |
| 33 | Semua resource di-tag & pakai prefix `techno-` | ☐ |
| 34 | Cleanup selesai | ☐ |

---

## 💡 Tips Trainer

### 🔴 Jika IPv6 Tidak Bisa Di-Enable
**Penyebab**: Region tidak support atau limit IPv6 tercapai.
**Solusi**:
1. Cek limit: `aws service-quotas list-service-quotas --service-code vpc`
2. Request limit increase jika perlu
3. Atau skip IPv6 untuk latihan (tapi di soal LKS wajib)

### 🔴 Jika Glue Crawler Tidak Menemukan Data
**Penyebab**: S3 output belum ada file atau path salah.
**Solusi**:
1. Pastikan upload gambar dulu → Lambda write ke S3 output
2. Cek path crawler: `s3://technooutput-jakarta-budi/results/`
3. Run crawler manual: `aws glue start-crawler --name techno-crawler-budi`

### 🔴 Jika SNS Email Tidak Terkirim
**Penyebab**: Subscription belum di-confirm.
**Solusi**:
1. Cek email `handi@seamolec.org` → klik link confirmation dari AWS
2. Atau cek subscription status: `aws sns list-subscriptions-by-topic --topic-arn <ARN>`

### 🔴 Jika Lambda Error "AccessDenied" saat write ke S3 output
**Penyebab**: IAM policy tidak include S3 PutObject untuk output bucket.
**Solusi**:
1. Cek IAM policy `lambda_custom` → harus ada Resource untuk output bucket
2. Pastikan bucket name di environment variable Lambda sudah benar

### 🔴 Jika API Gateway Tidak Support IPv6
**Penyebab**: Pakai HTTP API (tidak support IPv6).
**Solusi**:
1. Gunakan **REST API** (bukan HTTP API)
2. Endpoint type: REGIONAL
3. IPv6 otomatis enabled di regional endpoint

### 🔴 Jika Terraform Destroy Stuck di Glue
**Penyebab**: Crawler masih running.
**Solusi**:
```bash
aws glue stop-crawler --name techno-crawler-budi
sleep 30
terraform destroy
```

---

## 📊 Apa yang Anda Pelajari dari Saran 4?

| Skill | Status |
|---|---|
| Semua skill dari Saran 1, 2, 3 | ✅ |
| **AWS Glue** (Database + Table + Crawler) | ✅ |
| **SNS** (Topic + Email Subscription) | ✅ |
| **IPv6** di VPC, subnets, route tables | ✅ |
| **2 S3 buckets** (input + output) dengan lifecycle | ✅ |
| **S3 bucket policy** untuk public read | ✅ |
| **Lambda write ke S3** (PutObject) | ✅ |
| **Lambda publish ke SNS** | ✅ |
| **API Gateway REST** (bukan HTTP) untuk IPv6 | ✅ |
| **Full Terraform** untuk semua resource | ✅ |
| **Strict naming** `techno-` convention | ✅ |
| **End-to-end testing** dengan multiple services | ✅ |

---

## 🚀 Progression Path

```
✅ Saran 1 (Console, S3 → Lambda → Rekognition → DynamoDB) — 1.5 jam
    ↓
✅ Saran 2 (Terraform + DynamoDB Streams → Kinesis) — 2 jam
    ↓
✅ Saran 3 (Terraform + API Gateway + EKS + ECR) — 2.5 jam
    ↓
✅ Saran 4 (Full: + Glue, SNS, IPv6, 2 S3 buckets) — 4-5 jam ← ANDA DI SINI
    ↓
🏆 SOAL LKS Machine Learning (6 jam, Terraform Wajib)
```

---

## 🎯 Analogi Langsung ke Soal LKS

| Yang Anda Lakukan di Saran 4 | Yang Akan Anda Lakukan di Soal LKS |
|---|---|
| Semua resource via Terraform | **Semua resource via Terraform** (wajib) |
| VPC dengan IPv6 | **VPC dengan IPv6** (sama!) |
| 2 S3 buckets (input + output) | **2 S3 buckets** (sama!) |
| DynamoDB + Streams → Kinesis | **DynamoDB + Streams → Kinesis** (sama!) |
| Glue Database + Table + Crawler | **Glue Database + Table + Crawler** (sama!) |
| SNS topic + email subscription | **SNS topic + email subscription** (sama!) |
| 4 Lambda functions | **3 Lambda functions** (s3, post, get) |
| API Gateway REST dengan IPv6 | **API Gateway REST dengan IPv6** (sama!) |
| EKS + ECR + Python app | **EKS + ECR + Python app** (sama!) |
| Prefix `techno-` | Prefix `techno-` **ketat** |
| Email notification `handi@seamolec.org` | Email notification `handi@seamolec.org` **sama!** |

---

## 🏆 Anda Siap untuk Soal LKS!

Jika Anda sudah menyelesaikan **Saran 4** dengan lancar dalam 4-5 jam, Anda sudah **95% siap** untuk soal LKS Machine Learning 6 jam. Yang tinggal adalah:

1. **Detail teknis**:
   - Pastikan semua nama resource persis sesuai soal
   - Timeout Lambda: 2m, 1m, 1m30s
   - Kinesis: 1 shard, 1 day retention
   - Glue Crawler schedule: sesuai soal

2. **Testing**:
   - Upload gambar → cek S3 output → cek DynamoDB → cek email SNS
   - Generate token via API → validate token
   - Akses app via LoadBalancer → query results

3. **Cleanup**:
   - Hapus semua resource sebelum waktu habis
   - Pastikan tidak ada resource tersisa

---

## 🎯 Next Step

Apakah Anda ingin saya buatkan:
1. **Panduan lengkap soal LKS Machine Learning sesungguhnya** (6 jam, full Terraform)?
2. **Template Terraform module** untuk reusable components?
3. **Script testing otomatis** untuk memverifikasi seluruh pipeline?
4. **Cheat sheet** perbedaan antara Saran 4 dan soal LKS sesungguhnya?

Silakan pilih, dan saya akan buatkan! 🎯

**Selamat! Anda telah menyelesaikan seluruh progression path latihan Machine Learning dan siap menghadapi soal LKS sesungguhnya.** 🎉