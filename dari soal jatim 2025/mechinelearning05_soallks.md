# 📘 PANDUAN LENGKAP SOAL LKS MACHINE LEARNING
## **Image Processing Pipeline — Terraform + Rekognition + Glue + EKS**

> **Basis dokumen**: `modul-machinelering.pdf`
> **Alokasi waktu**: 6 jam (360 menit) | **Region wajib**: **`us-east-1` (N. Virginia)**
> **Constraint mutlak**: **100% Terraform** — manual provisioning = **0 POIN**
> **Naming prefix wajib**: **`techno-`**

---

## ⚠️ BAGIAN 0 — JEBAKAN FATAL YANG SERING MENGGUGURKAN

Baca ini **sebelum** menyentuh console atau editor.

| No | Jebakan | Jawaban Benar |
|---|---|---|
| 1 | Membuat resource manual di console | **0 POIN** — semua harus via Terraform |
| 2 | Lupa enable IPv6 di VPC | VPC **WAJIB** `assign_generated_ipv6_cidr_block = true` |
| 3 | Subnet tanpa IPv6 CIDR | Setiap subnet **WAJIB** punya `ipv6_cidr_block` (/64) |
| 4 | API Gateway pakai HTTP API | Harus **REST API** (regional) untuk dual-stack IPv4+IPv6 |
| 5 | Naming tanpa prefix `techno-` | **0 penilaian** untuk resource tanpa prefix |
| 6 | DynamoDB Streams tidak aktif | Harus `stream_view_type = "NEW_AND_OLD_IMAGES"` |
| 7 | Kinesis mode ON_DEMAND | Harus **PROVISIONED**, 1 shard, 1 day retention |
| 8 | Glue crawler path salah | Harus `s3://technooutput-<city>-<name>/results/` |
| 9 | Email SNS tidak di-confirm | Klik link konfirmasi di email,否则 notifikasi tidak masuk |
| 10 | EKS node bukan `t3.large` | Soal eksplisit minta **`t3.large`** |
| 11 | Dockerfile bukan Python 3.13 | Soal minta **`python:3.13`** |
| 12 | Terraform code tidak di-upload ke S3 input | Soal minta: *"Upload the code terraform to bucket input"* |
| 13 | Lambda timeout salah | `s3`=2m, `post`=1m, `get`=1m30s |
| 14 | Port container bukan 2000 | Soal minta port **2000** (lihat SG `techno-sg-apps`) |
| 15 | Lupa `depends_on` di S3 notification | Lambda permission harus `depends_on` dulu |

---

## 📋 TARGET AKHIR (Definition of Done)

```
Internet
   │
   ├── HTTP/HTTPS :80/:443 ──► LB ──► EKS Pods (Python app, port 2000)
   │                                       │
   │                                       ├──► DynamoDB "Tokens" (read)
   │                                       ├──► S3 technooutput-* (read results)
   │                                       └──► Glue catalog (schema query)
   │
   └── API Gateway (IPv4+IPv6)
         ├── POST /generate-token ──► techno-lambda-post ──► DynamoDB PutItem
         └── GET  /validate-token ──► techno-lambda-get  ──► DynamoDB GetItem

User upload image
   │
   ▼
S3 technoinput-* ──event──► techno-lambda-s3 (2 min)
                                │
                                ├──► Rekognition DetectLabels
                                ├──► S3 technooutput-*/results/<file>.json
                                ├──► DynamoDB Tokens (PutItem)
                                └──► SNS techno-sns-* (email notification)

DynamoDB Tokens
   │ (Streams: NEW_AND_OLD_IMAGES)
   ▼
Kinesis techno-kinesis-* (1 shard, 1 day, PROVISIONED)

Glue Crawler (scheduled)
   │
   ▼
Glue Table rekognition_results_table (queryable via Athena)
```

---

## 🗓️ RENCANA EKSEKUSI 360 MENIT

| Menit | Fase | Output |
|---|---|---|
| 0–15 | 0. Persiapan, clone repo, struktur TF | Repo siap, folder TF rapi |
| 15–45 | 1. VPC + IPv6 + Subnets + Route Tables + NAT | Jaringan dual-stack |
| 45–60 | 2. Security Groups + IAM | 2 SG + LabRole policies |
| 60–80 | 3. S3 (2 buckets + lifecycle + policy) | Input & output ready |
| 80–100 | 4. DynamoDB + Streams + Kinesis | Streaming pipeline |
| 100–120 | 5. Glue (DB + Table + Crawler) + SNS | Analytics + notification |
| 120–150 | 6. Lambda (3 functions + code) | Backend logic |
| 150–175 | 7. API Gateway REST (dual-stack) | Token API |
| 175–195 | 8. ECR + Docker build & push | Image ready |
| 195–240 | 9. EKS cluster + node group + kubectl | Cluster ready |
| 240–270 | 10. K8s deploy + service + verify | App running |
| 270–310 | 11. End-to-end testing | Semua flow works |
| 310–330 | 12. Upload TF code ke S3 input | Compliance |
| 330–360 | 13. Final audit + cleanup | Selesai |

---

# 🎯 FASE 0 — PERSIAPAN

### 0.1 Login & Region
1. AWS Academy Lab Console → **Start Lab** → **Open AWS Console**.
2. Kanan atas → **US East (N. Virginia) `us-east-1`**.
3. Catat di Notepad:
   - **Account ID** (12 digit)
   - **Access Key ID / Secret Access Key / Session Token**
   - **LabRole ARN**
   - **Nama Anda** (contoh: `budi`)
   - **Kota Anda** (contoh: `jakarta`)

### 0.2 Clone Repository
```bash
git clone https://github.com/LKS-Cloud-Computing/opsml
cd opsml
```

Struktur repo (sesuaikan dengan isi aktual):
```
opsml/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── lambda/
│       ├── techno_lambda_s3.py
│       ├── techno_lambda_post.py
│       └── techno_lambda_get.py
├── app/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── app.py
│   ├── deploy.yaml
│   └── service.yaml
└── README.md
```

### 0.3 Install Terraform di CloudShell
```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
sudo yum -y install terraform

# Install kubectl
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.28.3/2023-11-14/bin/linux/amd64/kubectl
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Install Docker
sudo yum install -y docker
sudo service docker start
sudo usermod -aG docker $USER
newgrp docker

# Configure AWS CLI
aws configure
# Masukkan Access Key, Secret Key, region=us-east-1, output=json
aws configure set aws_session_token <SESSION_TOKEN>
```

### 0.4 Naming Convention
| Resource | Format |
|---|---|
| VPC | `techno-<nama>` |
| S3 input | `technoinput-<kota>-<nama>` |
| S3 output | `technooutput-<kota>-<nama>` |
| Kinesis | `techno-kinesis-<nama>` |
| SNS | `techno-sns-<kota>-<nama>` |
| ECR | `techno-ecr-<kota>-<nama>` |
| EKS | `techno-eks-<kota>-<nama>` |
| Crawler | `techno-crawler-<nama>` |
| API GW | `Techno-API-<nama>` |
| SG | `techno-sg-lb`, `techno-sg-apps` |
| Lambda | `techno-lambda-s3`, `techno-lambda-post`, `techno-lambda-get` |

---

# 📁 FASE 1 — STRUKTUR TERRAFORM

### 📄 `variables.tf`
```hcl
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "project_name" {
  type    = string
  default = "budi"  # GANTI nama Anda
}

variable "city" {
  type    = string
  default = "jakarta"  # GANTI kota Anda
}

variable "vpc_cidr" {
  type    = string
  default = "25.1.0.0/16"
}

variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b"]
}

variable "public_subnet_cidrs" {
  type    = list(string)
  default = ["25.1.0.0/24", "25.1.2.0/24"]
}

variable "private_subnet_cidrs" {
  type    = list(string)
  default = ["25.1.1.0/24", "25.1.3.0/24"]
}

variable "sns_email" {
  type    = string
  default = "handi@seamolec.org"
}

variable "lab_role_arn" {
  type    = string
  default = ""  # Isi di terraform.tfvars
}
```

### 📄 `terraform.tfvars`
```hcl
aws_region   = "us-east-1"
project_name = "budi"       # GANTI
city         = "jakarta"    # GANTI
lab_role_arn = "arn:aws:iam::XXXXXXXXXXXX:role/LabRole"  # GANTI
sns_email    = "handi@seamolec.org"
```

### 📄 `main.tf` — header & locals
```hcl
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
      Environment = "Production"
      Owner       = var.project_name
      ManagedBy   = "Terraform"
    }
  }
}

data "aws_caller_identity" "current" {}
data "aws_availability_zones" "available" { state = "available" }

locals {
  prefix       = "techno"
  name         = var.project_name
  city         = var.city
  account_id   = data.aws_caller_identity.current.account_id
  
  vpc_name            = "${local.prefix}-${local.name}"
  s3_input            = "${local.prefix}input-${local.city}-${local.name}"
  s3_output           = "${local.prefix}output-${local.city}-${local.name}"
  kinesis_name        = "${local.prefix}-kinesis-${local.name}"
  sns_name            = "${local.prefix}-sns-${local.city}-${local.name}"
  ecr_name            = "${local.prefix}-ecr-${local.city}-${local.name}"
  eks_name            = "${local.prefix}-eks-${local.city}-${local.name}"
  crawler_name        = "${local.prefix}-crawler-${local.name}"
  api_name            = "Techno-API-${local.name}"
  sg_lb               = "${local.prefix}-sg-lb"
  sg_apps             = "${local.prefix}-sg-apps"
  lambda_s3           = "${local.prefix}-lambda-s3"
  lambda_post         = "${local.prefix}-lambda-post"
  lambda_get          = "${local.prefix}-lambda-get"
  glue_db             = "rekognition_results_db"
  glue_table          = "rekognition_results_table"
  dynamo_table        = "Tokens"
}
```

---

# 🌐 FASE 2 — VPC DENGAN IPv6

### Tambahkan ke `main.tf`:
```hcl
# ============================================
# VPC + IPv6
# ============================================
resource "aws_vpc" "main" {
  cidr_block                       = var.vpc_cidr
  assign_generated_ipv6_cidr_block = true  # WAJIB
  enable_dns_support               = true
  enable_dns_hostnames             = true
  
  tags = { Name = local.vpc_name }
}

# Internet Gateway (IPv4 + IPv6)
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${local.prefix}-igw-${local.name}" }
}

# Public Subnets (dengan IPv6 /64)
resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  ipv6_cidr_block         = cidrsubnet(aws_vpc.main.ipv6_cidr_block, 8, count.index * 2)
  map_public_ip_on_launch = true
  
  tags = {
    Name = "${local.vpc_name}-public-${var.availability_zones[count.index]}"
    Tier = "Public"
  }
}

# Private Subnets (dengan IPv6 /64)
resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]
  ipv6_cidr_block   = cidrsubnet(aws_vpc.main.ipv6_cidr_block, 8, count.index * 2 + 1)
  
  tags = {
    Name = "${local.vpc_name}-private-${var.availability_zones[count.index]}"
    Tier = "Private"
  }
}

# Elastic IP + NAT Gateway
resource "aws_eip" "nat" {
  domain = "vpc"
  tags   = { Name = "${local.prefix}-natgw-eip" }
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id
  tags          = { Name = "${local.prefix}-natgw-${local.name}" }
  depends_on    = [aws_internet_gateway.main]
}

# Public Route Table (IPv4 + IPv6)
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  route {
    ipv6_cidr_block = "::/0"
    gateway_id      = aws_internet_gateway.main.id
  }
  
  tags = { Name = "${local.prefix}-public-rt" }
}

# Private Route Table (IPv4 via NAT)
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }
  
  tags = { Name = "${local.prefix}-private-rt" }
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
```

---

# 🔒 FASE 3 — SECURITY GROUPS

```hcl
# ============================================
# SECURITY GROUPS
# ============================================
resource "aws_security_group" "lb" {
  name        = local.sg_lb
  description = "SG for Load Balancer - HTTP/HTTPS public"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    description = "HTTP public"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
  
  ingress {
    description = "HTTPS public"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
  
  tags = { Name = local.sg_lb }
}

resource "aws_security_group" "apps" {
  name        = local.sg_apps
  description = "SG for applications - port 2000 public"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    description = "App port 2000 public"
    from_port   = 2000
    to_port     = 2000
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
  
  tags = { Name = local.sg_apps }
}
```

---

# 📦 FASE 4 — S3 BUCKETS (INPUT + OUTPUT)

```hcl
# ============================================
# S3 BUCKETS
# ============================================
resource "aws_s3_bucket" "input" {
  bucket = local.s3_input
  tags   = { Name = local.s3_input, Description = "Input bucket for image uploads" }
}

resource "aws_s3_bucket" "output" {
  bucket = local.s3_output
  tags   = { Name = local.s3_output, Description = "Output bucket for Rekognition results" }
}

# Public access block — output bucket allow public read
resource "aws_s3_bucket_public_access_block" "input" {
  bucket                  = aws_s3_bucket.input.id
  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

resource "aws_s3_bucket_public_access_block" "output" {
  bucket                  = aws_s3_bucket.output.id
  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

# Bucket policies — public read
resource "aws_s3_bucket_policy" "input_public" {
  bucket = aws_s3_bucket.input.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "PublicRead"
      Effect    = "Allow"
      Principal = "*"
      Action    = "s3:GetObject"
      Resource  = "${aws_s3_bucket.input.arn}/*"
    }]
  })
}

resource "aws_s3_bucket_policy" "output_public" {
  bucket = aws_s3_bucket.output.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "PublicRead"
      Effect    = "Allow"
      Principal = "*"
      Action    = "s3:GetObject"
      Resource  = "${aws_s3_bucket.output.arn}/*"
    }]
  })
}

# Lifecycle: 30d → Glacier, 365d → delete
resource "aws_s3_bucket_lifecycle_configuration" "input" {
  bucket = aws_s3_bucket.input.id
  rule {
    id     = "archive-and-expire"
    status = "Enabled"
    transition {
      days          = 30
      storage_class = "GLACIER"
    }
    expiration { days = 365 }
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "output" {
  bucket = aws_s3_bucket.output.id
  rule {
    id     = "archive-and-expire"
    status = "Enabled"
    transition {
      days          = 30
      storage_class = "GLACIER"
    }
    expiration { days = 365 }
  }
}
```

---

# 🗄️ FASE 5 — DYNAMODB + STREAMS + KINESIS

```hcl
# ============================================
# DYNAMODB + STREAMS
# ============================================
resource "aws_dynamodb_table" "tokens" {
  name         = local.dynamo_table
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "token"
  
  attribute {
    name = "token"
    type = "S"
  }
  
  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"
  
  tags = { Name = local.dynamo_table, Description = "Token storage" }
}

# ============================================
# KINESIS DATA STREAM
# ============================================
resource "aws_kinesis_stream" "main" {
  name             = local.kinesis_name
  shard_count      = 1
  retention_period = 24  # 1 day = 24 hours
  
  stream_mode_details {
    stream_mode = "PROVISIONED"
  }
  
  tags = { Name = local.kinesis_name }
}

# DynamoDB → Kinesis streaming destination
resource "aws_dynamodb_kinesis_streaming_destination" "tokens_to_kinesis" {
  table_name = aws_dynamodb_table.tokens.name
  stream_arn = aws_kinesis_stream.main.arn
  depends_on = [aws_dynamodb_table.tokens, aws_kinesis_stream.main]
}
```

---

# 🔍 FASE 6 — AWS GLUE + SNS

```hcl
# ============================================
# AWS GLUE
# ============================================
resource "aws_iam_role" "glue_crawler" {
  name = "${local.prefix}-glue-role-${local.name}"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "glue.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "glue_service" {
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole"
  role       = aws_iam_role.glue_crawler.name
}

resource "aws_iam_role_policy" "glue_s3" {
  name = "${local.prefix}-glue-s3"
  role = aws_iam_role.glue_crawler.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = ["s3:GetObject", "s3:ListBucket"]
      Resource = [
        aws_s3_bucket.output.arn,
        "${aws_s3_bucket.output.arn}/*"
      ]
    }]
  })
}

resource "aws_glue_catalog_database" "rekognition" {
  name = local.glue_db
}

resource "aws_glue_catalog_table" "rekognition_results" {
  name          = local.glue_table
  database_name = aws_glue_catalog_database.rekognition.name
  table_type    = "EXTERNAL_TABLE"
  
  parameters = { classification = "json" }
  
  storage_descriptor {
    location      = "s3://${aws_s3_bucket.output.id}/results/"
    input_format  = "org.apache.hadoop.mapred.TextInputFormat"
    output_format = "org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat"
    
    ser_de_info {
      name                  = "rekognition-json"
      serialization_library = "org.openx.data.jsonserde.JsonSerDe"
    }
    
    columns { name = "token";        type = "string" }
    columns { name = "image_name";   type = "string" }
    columns { name = "labels";       type = "string" }
    columns { name = "label_count";  type = "int" }
    columns { name = "timestamp";    type = "string" }
  }
}

resource "aws_glue_crawler" "rekognition" {
  name          = local.crawler_name
  role          = aws_iam_role.glue_crawler.arn
  database_name = aws_glue_catalog_database.rekognition.name
  
  s3_target {
    path = "s3://${aws_s3_bucket.output.id}/results/"
  }
  
  schedule = "cron(0/15 * * * ? *)"
  
  schema_change_policy {
    delete_behavior = "LOG"
    update_behavior = "UPDATE_IN_DATABASE"
  }
}

# ============================================
# SNS
# ============================================
resource "aws_sns_topic" "notification" {
  name = local.sns_name
  tags = { Name = local.sns_name }
}

resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.notification.arn
  protocol  = "email"
  endpoint  = var.sns_email
}
```

---

# ⚡ FASE 7 — LAMBDA FUNCTIONS

### 📄 `lambda/techno_lambda_s3.py`
```python
import json, boto3, urllib.parse
from datetime import datetime

s3 = boto3.client('s3')
rekognition = boto3.client('rekognition')
dynamodb = boto3.client('dynamodb')
sns = boto3.client('sns')

OUTPUT_BUCKET = 'technooutput-jakarta-budi'  # GANTI
SNS_TOPIC_ARN = 'arn:aws:sns:us-east-1:ACCOUNT_ID:techno-sns-jakarta-budi'  # GANTI

def lambda_handler(event, context):
    try:
        bucket = event['Records'][0]['s3']['bucket']['name']
        key = urllib.parse.unquote_plus(event['Records'][0]['s3']['object']['key'], encoding='utf-8')
        
        token = f"token-{int(datetime.now().timestamp())}-{key.split('.')[0]}"
        
        # Rekognition DetectLabels
        response = rekognition.detect_labels(
            Image={'S3Object': {'Bucket': bucket, 'Name': key}},
            MaxLabels=20, MinConfidence=70.0
        )
        labels = response.get('Labels', [])
        
        labels_data = [{
            'Name': l['Name'],
            'Confidence': float(l['Confidence'])
        } for l in labels]
        
        timestamp = datetime.now().isoformat()
        
        # Save to S3 output
        result_data = {
            'token': token, 'image_name': key,
            'labels': labels_data, 'label_count': len(labels),
            'timestamp': timestamp, 'status': 'SUCCESS'
        }
        s3.put_object(
            Bucket=OUTPUT_BUCKET,
            Key=f"results/{key.split('.')[0]}.json",
            Body=json.dumps(result_data, indent=2),
            ContentType='application/json'
        )
        
        # Save to DynamoDB
        dynamodb.put_item(
            TableName='Tokens',
            Item={
                'token': {'S': token},
                'image_name': {'S': key},
                'labels': {'S': json.dumps(labels_data)},
                'label_count': {'N': str(len(labels))},
                'timestamp': {'S': timestamp},
                'status': {'S': 'SUCCESS'}
            }
        )
        
        # SNS notification
        sns.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject=f"Image Processed: {key}",
            Message=f"Token: {token}\nLabels: {len(labels)}\nTop: {', '.join([l['Name'] for l in labels[:5]])}"
        )
        
        return {'statusCode': 200, 'body': json.dumps({'token': token, 'labels': len(labels)})}
    except Exception as e:
        return {'statusCode': 500, 'body': json.dumps({'error': str(e)})}
```

### 📄 `lambda/techno_lambda_post.py`
```python
import json, boto3, uuid
from datetime import datetime, timedelta

dynamodb = boto3.client('dynamodb')

def lambda_handler(event, context):
    try:
        token = str(uuid.uuid4())
        deviceid = event.get('headers', {}).get('deviceid', 'unknown')
        expires_at = (datetime.now() + timedelta(hours=24)).isoformat()
        
        dynamodb.put_item(
            TableName='Tokens',
            Item={
                'token': {'S': token},
                'deviceid': {'S': deviceid},
                'expires_at': {'S': expires_at},
                'created_at': {'S': datetime.now().isoformat()},
                'status': {'S': 'ACTIVE'}
            }
        )
        
        return {
            'statusCode': 200,
            'headers': {'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*'},
            'body': json.dumps({'token': token, 'expires_at': expires_at})
        }
    except Exception as e:
        return {'statusCode': 500, 'body': json.dumps({'error': str(e)})}
```

### 📄 `lambda/techno_lambda_get.py`
```python
import json, boto3
from datetime import datetime

dynamodb = boto3.client('dynamodb')

def lambda_handler(event, context):
    try:
        token = event.get('queryStringParameters', {}).get('token')
        if not token:
            return {'statusCode': 400, 'body': json.dumps({'error': 'Token required'})}
        
        response = dynamodb.get_item(TableName='Tokens', Key={'token': {'S': token}})
        
        if 'Item' not in response:
            return {'statusCode': 404, 'body': json.dumps({'error': 'Not found'})}
        
        item = response['Item']
        expires_at = item.get('expires_at', {}).get('S')
        
        if expires_at and datetime.fromisoformat(expires_at) < datetime.now():
            return {'statusCode': 401, 'body': json.dumps({'error': 'Expired'})}
        
        return {
            'statusCode': 200,
            'headers': {'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*'},
            'body': json.dumps({
                'valid': True, 'token': token,
                'deviceid': item.get('deviceid', {}).get('S'),
                'status': item.get('status', {}).get('S')
            })
        }
    except Exception as e:
        return {'statusCode': 500, 'body': json.dumps({'error': str(e)})}
```

### Tambahkan ke `main.tf`:
```hcl
# ============================================
# IAM ROLE LAMBDA
# ============================================
resource "aws_iam_role" "lambda_exec" {
  name = "${local.prefix}-lambda-role-${local.name}"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy" "lambda_custom" {
  name = "${local.prefix}-lambda-custom"
  role = aws_iam_role.lambda_exec.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["s3:GetObject", "s3:PutObject", "s3:ListBucket"]
        Resource = [aws_s3_bucket.input.arn, "${aws_s3_bucket.input.arn}/*",
                    aws_s3_bucket.output.arn, "${aws_s3_bucket.output.arn}/*"]
      },
      {
        Effect   = "Allow"
        Action   = ["rekognition:DetectLabels", "rekognition:DetectFaces", "rekognition:DetectText"]
        Resource = "*"
      },
      {
        Effect   = "Allow"
        Action   = ["dynamodb:PutItem", "dynamodb:GetItem", "dynamodb:Query", "dynamodb:Scan"]
        Resource = aws_dynamodb_table.tokens.arn
      },
      {
        Effect   = "Allow"
        Action   = ["kinesis:PutRecord", "kinesis:PutRecords"]
        Resource = aws_kinesis_stream.main.arn
      },
      {
        Effect   = "Allow"
        Action   = ["sns:Publish"]
        Resource = aws_sns_topic.notification.arn
      }
    ]
  })
}

# ============================================
# LAMBDA FUNCTIONS
# ============================================
data "archive_file" "lambda_s3" {
  type        = "zip"
  source_file = "${path.module}/lambda/techno_lambda_s3.py"
  output_path = "${path.module}/lambda/techno_lambda_s3.zip"
}

data "archive_file" "lambda_post" {
  type        = "zip"
  source_file = "${path.module}/lambda/techno_lambda_post.py"
  output_path = "${path.module}/lambda/techno_lambda_post.zip"
}

data "archive_file" "lambda_get" {
  type        = "zip"
  source_file = "${path.module}/lambda/techno_lambda_get.py"
  output_path = "${path.module}/lambda/techno_lambda_get.zip"
}

resource "aws_lambda_function" "s3_processor" {
  function_name    = local.lambda_s3
  filename         = data.archive_file.lambda_s3.output_path
  source_code_hash = data.archive_file.lambda_s3.output_base64sha256
  handler          = "techno_lambda_s3.lambda_handler"
  runtime          = "python3.12"
  timeout          = 120  # 2 menit
  memory_size      = 256
  role             = aws_iam_role.lambda_exec.arn
  
  environment {
    variables = {
      OUTPUT_BUCKET = local.s3_output
      SNS_TOPIC_ARN = aws_sns_topic.notification.arn
    }
  }
}

resource "aws_lambda_function" "post" {
  function_name    = local.lambda_post
  filename         = data.archive_file.lambda_post.output_path
  source_code_hash = data.archive_file.lambda_post.output_base64sha256
  handler          = "techno_lambda_post.lambda_handler"
  runtime          = "python3.12"
  timeout          = 60  # 1 menit
  memory_size      = 256
  role             = aws_iam_role.lambda_exec.arn
}

resource "aws_lambda_function" "get" {
  function_name    = local.lambda_get
  filename         = data.archive_file.lambda_get.output_path
  source_code_hash = data.archive_file.lambda_get.output_base64sha256
  handler          = "techno_lambda_get.lambda_handler"
  runtime          = "python3.12"
  timeout          = 90  # 1 menit 30 detik
  memory_size      = 256
  role             = aws_iam_role.lambda_exec.arn
}

# Lambda permissions
resource "aws_lambda_permission" "s3_invoke" {
  statement_id  = "AllowS3Invoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.s3_processor.function_name
  principal     = "s3.amazonaws.com"
  source_arn    = aws_s3_bucket.input.arn
}

# S3 notification → Lambda
resource "aws_s3_bucket_notification" "input_lambda" {
  bucket = aws_s3_bucket.input.id
  lambda_function {
    lambda_function_arn = aws_lambda_function.s3_processor.arn
    events              = ["s3:ObjectCreated:*"]
  }
  depends_on = [aws_lambda_permission.s3_invoke]
}

# CloudWatch Log Groups
resource "aws_cloudwatch_log_group" "lambda_s3" {
  name              = "/aws/lambda/${local.lambda_s3}"
  retention_in_days = 7
}
resource "aws_cloudwatch_log_group" "lambda_post" {
  name              = "/aws/lambda/${local.lambda_post}"
  retention_in_days = 7
}
resource "aws_cloudwatch_log_group" "lambda_get" {
  name              = "/aws/lambda/${local.lambda_get}"
  retention_in_days = 7
}
```

---

# 🚪 FASE 8 — API GATEWAY REST (DUAL-STACK)

```hcl
# ============================================
# API GATEWAY REST (IPv4 + IPv6)
# ============================================
resource "aws_api_gateway_rest_api" "main" {
  name        = local.api_name
  description = "REST API for token management with IPv4+IPv6"
  
  endpoint_configuration {
    types = ["REGIONAL"]
  }
}

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
  
  lifecycle { create_before_destroy = true }
}

resource "aws_api_gateway_stage" "prod" {
  deployment_id = aws_api_gateway_deployment.main.id
  rest_api_id   = aws_api_gateway_rest_api.main.id
  stage_name    = "prod"
}

# Lambda permissions for API GW
resource "aws_lambda_permission" "apigw_post" {
  statement_id  = "AllowAPIGWPost"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.post.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_api_gateway_rest_api.main.execution_arn}/*/*"
}

resource "aws_lambda_permission" "apigw_get" {
  statement_id  = "AllowAPIGWGet"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.get.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_api_gateway_rest_api.main.execution_arn}/*/*"
}
```

---

# 📦 FASE 9 — ECR

```hcl
# ============================================
# ECR
# ============================================
resource "aws_ecr_repository" "main" {
  name                 = local.ecr_name
  image_tag_mutability = "MUTABLE"
  
  image_scanning_configuration { scan_on_push = true }
  
  tags = { Name = local.ecr_name }
}
```

---

# ☸️ FASE 10 — EKS CLUSTER + NODE GROUP

```hcl
# ============================================
# EKS CLUSTER
# ============================================
resource "aws_iam_role" "eks_cluster" {
  name = "${local.prefix}-eks-cluster-role-${local.name}"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "eks.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.eks_cluster.name
}

resource "aws_eks_cluster" "main" {
  name     = local.eks_name
  role_arn = aws_iam_role.eks_cluster.arn
  version  = "1.28"
  
  vpc_config {
    subnet_ids              = concat(aws_subnet.private[*].id, aws_subnet.public[*].id)
    security_group_ids      = [aws_security_group.apps.id]
    endpoint_public_access  = true
    endpoint_private_access = true
  }
  
  depends_on = [aws_iam_role_policy_attachment.eks_cluster_policy]
}

# Node IAM
resource "aws_iam_role" "eks_nodes" {
  name = "${local.prefix}-eks-node-role-${local.name}"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "eks_worker" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  role       = aws_iam_role.eks_nodes.name
}
resource "aws_iam_role_policy_attachment" "eks_cni" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.eks_nodes.name
}
resource "aws_iam_role_policy_attachment" "eks_ecr" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  role       = aws_iam_role.eks_nodes.name
}

# Inline policy untuk akses DynamoDB, S3, Rekognition, Glue
resource "aws_iam_role_policy" "eks_nodes_custom" {
  name = "${local.prefix}-eks-custom"
  role = aws_iam_role.eks_nodes.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["dynamodb:GetItem", "dynamodb:Query", "dynamodb:Scan", "dynamodb:PutItem"]
        Resource = aws_dynamodb_table.tokens.arn
      },
      {
        Effect   = "Allow"
        Action   = ["s3:GetObject", "s3:ListBucket"]
        Resource = [aws_s3_bucket.input.arn, "${aws_s3_bucket.input.arn}/*",
                    aws_s3_bucket.output.arn, "${aws_s3_bucket.output.arn}/*"]
      },
      {
        Effect   = "Allow"
        Action   = ["rekognition:*", "glue:GetTable", "glue:GetDatabase"]
        Resource = "*"
      }
    ]
  })
}

resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${local.prefix}-nodegroup-${local.name}"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = aws_subnet.private[*].id
  
  instance_types = ["t3.large"]  # WAJIB sesuai soal
  
  scaling_config {
    desired_size = 1
    max_size     = 2
    min_size     = 1
  }
  
  depends_on = [
    aws_iam_role_policy_attachment.eks_worker,
    aws_iam_role_policy_attachment.eks_cni,
    aws_iam_role_policy_attachment.eks_ecr
  ]
}
```

---

# 🐍 FASE 11 — PYTHON APP + DOCKERFILE + K8S

### 📄 `app/Dockerfile`
```dockerfile
FROM python:3.13-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .

EXPOSE 2000
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:2000/health || exit 1

CMD ["python", "app.py"]
```

### 📄 `app/requirements.txt`
```
flask==3.0.0
boto3==1.34.0
```

### 📄 `app/app.py`
```python
from flask import Flask, jsonify, request
import boto3, os, json
from datetime import datetime

app = Flask(__name__)
dynamodb = boto3.client('dynamodb')
s3 = boto3.client('s3')
glue = boto3.client('glue')

TABLE = os.environ.get('DYNAMODB_TABLE', 'Tokens')
OUTPUT_BUCKET = os.environ.get('OUTPUT_BUCKET', '')
GLUE_DB = os.environ.get('GLUE_DB', 'rekognition_results_db')
GLUE_TABLE = os.environ.get('GLUE_TABLE', 'rekognition_results_table')

@app.route('/')
def home():
    return jsonify({'status': 'healthy', 'app': 'techno-ml-app', 'timestamp': datetime.now().isoformat()})

@app.route('/health')
def health():
    return jsonify({'status': 'ok'}), 200

@app.route('/results', methods=['GET'])
def get_results():
    try:
        response = dynamodb.scan(TableName=TABLE, Limit=50)
        items = [{
            'token': i.get('token', {}).get('S'),
            'image_name': i.get('image_name', {}).get('S'),
            'label_count': i.get('label_count', {}).get('N'),
            'status': i.get('status', {}).get('S'),
            'timestamp': i.get('timestamp', {}).get('S')
        } for i in response.get('Items', [])]
        return jsonify({'success': True, 'count': len(items), 'results': items})
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)}), 500

@app.route('/validate-token', methods=['GET'])
def validate_token():
    token = request.args.get('token')
    if not token:
        return jsonify({'valid': False, 'error': 'Token required'}), 400
    try:
        response = dynamodb.get_item(TableName=TABLE, Key={'token': {'S': token}})
        if 'Item' in response:
            return jsonify({'valid': True, 'token': token})
        return jsonify({'valid': False, 'error': 'Invalid token'}), 401
    except Exception as e:
        return jsonify({'valid': False, 'error': str(e)}), 500

@app.route('/glue/schema', methods=['GET'])
def glue_schema():
    try:
        response = glue.get_table(DatabaseName=GLUE_DB, Name=GLUE_TABLE)
        columns = response['Table'].get('StorageDescriptor', {}).get('Columns', [])
        return jsonify({
            'success': True, 'database': GLUE_DB, 'table': GLUE_TABLE,
            'columns': [{'name': c['Name'], 'type': c['Type']} for c in columns]
        })
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=2000)
```

### 📄 `app/deploy.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: techno-ml-app
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
        - name: GLUE_DB
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
```

### 📄 `app/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: techno-ml-service
spec:
  type: LoadBalancer
  selector:
    app: techno-ml-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 2000
```

---

# 🚀 FASE 12 — TERRAFORM APPLY + DEPLOY

### 12.1 Init & Apply
```bash
cd ~/opsml/terraform
terraform init
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
```

### 12.2 Configure kubectl
```bash
aws eks update-kubeconfig --region us-east-1 --name techno-eks-jakarta-budi
kubectl get nodes  # Harus muncul 1 node t3.large Ready
```

### 12.3 Build & Push Docker Image
```bash
cd ~/opsml/app
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_URL="${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/techno-ecr-jakarta-budi"

aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com

COMMIT_SHA=$(git rev-parse --short HEAD 2>/dev/null || echo "v1.0.0")
docker build -t techno-ml-app:${COMMIT_SHA} .
docker tag techno-ml-app:${COMMIT_SHA} ${ECR_URL}:${COMMIT_SHA}
docker tag techno-ml-app:${COMMIT_SHA} ${ECR_URL}:latest
docker push ${ECR_URL}:${COMMIT_SHA}
docker push ${ECR_URL}:latest
```

### 12.4 Apply K8s Manifests
```bash
sed -i "s|ACCOUNT_ID|${ACCOUNT_ID}|g" deploy.yaml
sed -i "s|CITY|jakarta|g" deploy.yaml
sed -i "s|NAME|budi|g" deploy.yaml

kubectl apply -f deploy.yaml
kubectl apply -f service.yaml
kubectl get pods,svc -w  # Tunggu LoadBalancer dapat external IP
```

---

# 🧪 FASE 13 — END-TO-END TESTING

### Test 1: Upload Gambar
```bash
curl -o test.jpg https://picsum.photos/800/600.jpg
aws s3 cp test.jpg s3://technoinput-jakarta-budi/test.jpg
```

### Test 2: Cek Lambda Logs
```bash
aws logs tail /aws/lambda/techno-lambda-s3 --since 2m --follow
```

### Test 3: Cek S3 Output
```bash
aws s3 ls s3://technooutput-jakarta-budi/results/
aws s3 cp s3://technooutput-jakarta-budi/results/test.json -
```

### Test 4: Cek DynamoDB
```bash
aws dynamodb scan --table-name Tokens --max-items 5
```

### Test 5: Confirm SNS Email
Cek email `handi@seamolec.org` → klik link konfirmasi AWS.

### Test 6: API Gateway - Generate Token
```bash
API_URL="https://$(terraform output -raw api_endpoint)/prod"
curl -X POST ${API_URL}/generate-token -H "deviceid: test-001"
# Simpan TOKEN dari response
```

### Test 7: API Gateway - Validate Token
```bash
curl "${API_URL}/validate-token?token=${TOKEN}"
```

### Test 8: Akses App via LoadBalancer
```bash
APP_URL=$(kubectl get svc techno-ml-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://${APP_URL}/
curl http://${APP_URL}/results
curl "http://${APP_URL}/validate-token?token=${TOKEN}"
curl http://${APP_URL}/glue/schema
```

### Test 9: Run Glue Crawler
```bash
aws glue start-crawler --name techno-crawler-budi
# Tunggu 2-3 menit, cek Athena:
# SELECT * FROM rekognition_results_db.rekognition_results_table LIMIT 10;
```

### Test 10: Test IPv6
```bash
curl -6 https://${API_URL}/generate-token -X POST
```

---

# 📤 FASE 14 — UPLOAD TERRAFORM CODE KE S3 INPUT

> Soal menyatakan: *"Upload the code terraform to bucket input"*

```bash
cd ~/opsml
zip -r terraform-code.zip terraform/
aws s3 cp terraform-code.zip s3://technoinput-jakarta-budi/terraform-code.zip
```

---

# 🎓 CHECKLIST JURI

| # | Kriteria | Status |
|---|---|---|
| 1 | Region `us-east-1` | ☐ |
| 2 | **100% Terraform** (tidak ada resource manual) | ☐ |
| 3 | VPC `techno-<nama>` dengan **IPv4 + IPv6** | ☐ |
| 4 | 2 public + 2 private subnet dengan **IPv6 /64** | ☐ |
| 5 | IGW + NAT GW + 2 route tables | ☐ |
| 6 | SG `techno-sg-lb` (80, 443 public) | ☐ |
| 7 | SG `techno-sg-apps` (2000 public) | ☐ |
| 8 | S3 `technoinput-<kota>-<nama>` + lifecycle 30d/365d | ☐ |
| 9 | S3 `technooutput-<kota>-<nama>` + public read + lifecycle | ☐ |
| 10 | DynamoDB `Tokens` (PK: token String) + **Streams enabled** | ☐ |
| 11 | Kinesis `techno-kinesis-<nama>` (1 shard, 1 day, PROVISIONED) | ☐ |
| 12 | DynamoDB → Kinesis streaming destination aktif | ☐ |
| 13 | Glue DB `rekognition_results_db` | ☐ |
| 14 | Glue Table `rekognition_results_table` | ☐ |
| 15 | Glue Crawler `techno-crawler-<nama>` | ☐ |
| 16 | SNS `techno-sns-<kota>-<nama>` + email `handi@seamolec.org` | ☐ |
| 17 | Lambda `techno-lambda-s3` timeout 2m | ☐ |
| 18 | Lambda `techno-lambda-post` timeout 1m | ☐ |
| 19 | Lambda `techno-lambda-get` timeout 1m30s | ☐ |
| 20 | API Gateway REST `Techno-API-<nama>` (regional, dual-stack) | ☐ |
| 21 | Route `POST /generate-token` → Lambda post | ☐ |
| 22 | Route `GET /validate-token` → Lambda get | ☐ |
| 23 | ECR `techno-ecr-<kota>-<nama>` (mutable) | ☐ |
| 24 | EKS `techno-eks-<kota>-<nama>` (v1.28) | ☐ |
| 25 | Node group dengan **`t3.large`** | ☐ |
| 26 | Dockerfile pakai **Python 3.13** | ☐ |
| 27 | K8s Deployment 2 replicas | ☐ |
| 28 | K8s Service type LoadBalancer | ☐ |
| 29 | App listen on port **2000** | ☐ |
| 30 | Upload gambar → Lambda → Rekognition → S3 output | ☐ |
| 31 | DynamoDB → Kinesis stream | ☐ |
| 32 | SNS email terkirim | ☐ |
| 33 | Glue Crawler berhasil crawl | ☐ |
| 34 | API Gateway generate & validate token works | ☐ |
| 35 | App accessible via LoadBalancer | ☐ |
| 36 | Terraform code di-upload ke S3 input | ☐ |
| 37 | Semua resource di-tag & prefix `techno-` | ☐ |

---

# 🧹 FASE 15 — CLEANUP

```bash
# 1. Delete K8s resources
kubectl delete -f app/service.yaml
kubectl delete -f app/deploy.yaml

# 2. Empty S3 buckets
aws s3 rm s3://technoinput-jakarta-budi --recursive
aws s3 rm s3://technooutput-jakarta-budi --recursive

# 3. Stop Glue Crawler
aws glue stop-crawler --name techno-crawler-budi

# 4. Terraform destroy
cd ~/opsml/terraform
terraform destroy -auto-approve

# 5. Verify
terraform state list  # harus kosong
aws s3 ls
aws dynamodb list-tables
aws kinesis list-streams
```

---

# 💡 TROUBLESHOOTING CEPAT

| Gejala | Solusi |
|---|---|
| `terraform apply` error IPv6 | Pastikan `assign_generated_ipv6_cidr_block = true` di VPC |
| Lambda timeout saat panggil Rekognition | Cek IAM role punya `rekognition:DetectLabels` |
| S3 notification tidak trigger Lambda | Cek `depends_on = [aws_lambda_permission.s3_invoke]` |
| DynamoDB Stream tidak ke Kinesis | Cek `aws_dynamodb_kinesis_streaming_destination` status `ENABLED` |
| API Gateway 403 | Cek Lambda permission untuk API GW |
| EKS node stuck `CREATE_IN_PROGRESS` | Tunggu 15 menit, cek IAM role + subnet route |
| Pod `ImagePullBackOff` | Cek IAM node punya `AmazonEC2ContainerRegistryReadOnly` |
| SNS email tidak masuk | Klik link konfirmasi di email |
| Glue crawler tidak temukan data | Pastikan path `s3://technooutput-*/results/` dan ada file JSON |
| LoadBalancer tidak dapat IP | Tunggu 3-5 menit, cek subnet tag |

---

# 🏆 KUNCI KEBERHASILAN

1. **Terraform dulu, testing belakangan** — jangan coba-coba manual.
2. **IPv6 adalah wajib** — juri akan cek VPC dan API Gateway.
3. **Naming `techno-` ketat** — bahkan typo kecil = 0 poin.
4. **Email SNS harus di-confirm** — ini jebakan klasik.
5. **Upload TF code ke S3** — ini requirement eksplisit di soal.
6. **Port 2000** — bukan 80, bukan 8080, tapi 2000 sesuai SG.
7. **Python 3.13** di Dockerfile — bukan 3.11 atau 3.12.
8. **t3.large** untuk EKS node — bukan t3.medium.

---

## 🎯 Anda Siap!

Jika Anda sudah menyelesaikan **Saran 1 → 2 → 3 → 4**, panduan ini adalah **final boss**. Dengan 6 jam, Anda punya waktu cukup untuk:
- Terraform apply (~30 menit)
- EKS provisioning (~15 menit)
- Testing & debugging (~1 jam)
- Upload TF code + cleanup (~30 menit)
- Buffer untuk error (~3 jam)

**Selamat berkompetisi!** 🚀