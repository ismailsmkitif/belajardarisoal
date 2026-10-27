# 📘 Panduan Lengkap Pengerjaan Saran 3 (Machine Learning)
## "API & Container Basics" — Terraform + API Gateway + ECR + EKS + Python App

> **Estimasi waktu**: 2.5 jam | **Region**: `us-east-1` | **Terraform + kubectl**
> **Tujuan**: Membangun di atas Saran 2 dengan menambahkan API Gateway, ECR, dan EKS untuk deployment containerized Python app.

---

## 🎯 Tujuan Latihan

Memperluas pipeline dari Saran 2 dengan menambahkan **lapisan API** dan **container orchestration**:
- ✅ Semua resource via **Terraform** (IaC)
- ✅ **API Gateway REST** (regional, IPv4+IPv6 ready) dengan 2 endpoint
- ✅ **Lambda Authorizer** untuk token validation
- ✅ **ECR** untuk menyimpan Docker image
- ✅ **EKS Cluster** dengan node group `t3.large`
- ✅ **Kubernetes Deployment** (2 replicas) + **Service** (LoadBalancer)
- ✅ Python app (Flask) yang consume DynamoDB tokens
- ✅ Prefix naming **`techno-`** sesuai standar soal LKS

Ini adalah **jembatan terakhir** sebelum Saran 4 (yang menambahkan Glue, SNS, IPv6 penuh).

---

## 🗺️ Topologi Arsitektur

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                        AWS CLOUD (us-east-1) - Terraform Managed                     │
│                                                                                      │
│   ┌──────────────┐         ┌─────────────────────────────────────────────────────┐   │
│   │  USER/CLIENT │         │  API GATEWAY: Techno-API-budi                       │   │
│   │              │         │  (Regional, IPv4+IPv6)                              │   │
│   │  POST        │         │                                                     │   │
│   │  /generate-  │────────▶│  /generate-token (POST) ──► techno-lambda-post      │   │
│   │  token       │         │  /validate-token (GET)  ──► techno-lambda-get       │   │
│   │              │         │                                                     │   │
│   │  GET         │         │  Lambda Authorizer: techno-lambda-auth              │   │
│   │  /validate-  │         │                                                     │   │
│   │  token       │         └─────────────────────────────────────────────────────┘   │
│   └──────────────┘                                       │                          │
│                                                          │                          │
│                                                          ▼                          │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │  LAMBDA FUNCTIONS                                                           │   │
│   │  • techno-lambda-s3    (S3 trigger → Rekognition → DynamoDB)                │   │
│   │  • techno-lambda-post  (API GW → DynamoDB PutItem)                          │   │
│   │  • techno-lambda-get   (API GW → DynamoDB GetItem)                          │   │
│   │  • techno-lambda-auth  (API GW Authorizer → DynamoDB token validation)      │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                          │                          │
│                                                          ▼                          │
│   ┌──────────────────┐         ┌──────────────────┐         ┌──────────────────┐   │
│   │  S3 Buckets      │         │  DynamoDB        │         │  Kinesis Stream  │   │
│   │  technoinput-*   │         │  Table: Tokens   │         │  techno-kinesis- │   │
│   │  technooutput-*  │         │  (with Streams)  │         │  budi            │   │
│   └──────────────────┘         └──────────────────┘         └──────────────────┘   │
│                                                                                      │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │  EKS CLUSTER: techno-eks-jakarta-budi                                       │   │
│   │  ┌─────────────────────────────────────────────────────────────────────┐   │   │
│   │  │  Node Group: 1x t3.large (in private subnet)                        │   │   │
│   │  │  ┌─────────────┐  ┌─────────────┐                                   │   │   │
│   │  │  │ Pod 1       │  │ Pod 2       │  (Python Flask App from ECR)      │   │   │
│   │  │  │ techno-ml-  │  │ techno-ml-  │                                   │   │   │
│   │  │  │ app-xxxx    │  │ app-yyyy    │                                   │   │   │
│   │  │  └──────┬──────┘  └──────┬──────┘                                   │   │   │
│   │  │         └────────┬────────┘                                          │   │   │
│   │  │                  ▼                                                   │   │   │
│   │  │         ┌─────────────────┐                                          │   │   │
│   │  │         │ K8s Service     │                                          │   │   │
│   │  │         │ Type: LoadBalan │──────► AWS NLB/ALB (external)            │   │   │
│   │  │         └─────────────────┘                                          │   │   │
│   │  └─────────────────────────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                          ▲                          │
│                                                          │                          │
│   ┌──────────────────┐                                   │                          │
│   │  ECR Repository  │                                   │                          │
│   │  techno-ecr-     │────── docker pull ────────────────┘                          │
│   │  jakarta-budi    │                                                              │
│   └──────────────────┘                                                              │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

### 🔄 Alur Kerja (Flow)
```
1. User POST /generate-token → API Gateway → Lambda techno-lambda-post
   → Generate token → Simpan ke DynamoDB "Tokens"
                          ↓
2. User GET /validate-token?token=xxx → API Gateway → Lambda techno-lambda-get
   → Query DynamoDB → Return token info
                          ↓
3. User upload gambar → S3 technoinput-* → Lambda techno-lambda-s3
   → Rekognition DetectLabels → Simpan hasil ke DynamoDB
                          ↓
4. Python App di EKS (via LoadBalancer):
   → Tampilkan hasil analisis dari DynamoDB
   → Butuh valid token untuk akses (via API Gateway)
```

---

## 📋 Daftar Isi
1. [Fase 0: Persiapan Environment](#fase-0-persiapan)
2. [Fase 1: Ekstensi Terraform dari Saran 2](#fase-1-ekstensi)
3. [Fase 2: API Gateway + Lambda Authorizer](#fase-2-api-gateway)
4. [Fase 3: Lambda Functions (POST, GET, Auth)](#fase-3-lambda)
5. [Fase 4: ECR Repository](#fase-4-ecr)
6. [Fase 5: Python App + Dockerfile](#fase-5-python-app)
7. [Fase 6: EKS Cluster + Node Group](#fase-6-eks)
8. [Fase 7: Kubernetes Manifests (deploy.yaml + service.yaml)](#fase-7-k8s)
9. [Fase 8: Build, Push, Deploy](#fase-8-deploy)
10. [Fase 9: Testing End-to-End](#fase-9-testing)
11. [Fase 10: Cleanup](#fase-10-cleanup)

---

## 🎯 Fase 0: Persiapan Environment

### Langkah 0.1: Setup CloudShell
```bash
# Install kubectl
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.28.3/2023-11-14/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install Docker (jika belum)
sudo yum install -y docker
sudo service docker start
sudo usermod -aG docker $USER

# Install AWS IAM Authenticator for EKS
curl -o aws-iam-authenticator https://amazon-eks.s3.us-west-2.amazonaws.com/1.28.3/2023-11-14/bin/linux/amd64/aws-iam-authenticator
chmod +x aws-iam-authenticator
sudo mv aws-iam-authenticator /usr/local/bin/

# Verify
kubectl version --client
docker --version
aws-iam-authenticator version
```

### Langkah 0.2: Reuse Struktur Saran 2
```bash
cd ~/techno-ml-project
# Struktur sudah ada dari Saran 2, tambahkan folder baru:
mkdir -p lambda/auth lambda/post lambda/get app
```

Struktur final:
```
techno-ml-project/
├── main.tf                    # (update dari Saran 2)
├── variables.tf               # (reuse)
├── outputs.tf                 # (update)
├── terraform.tfvars           # (reuse)
├── lambda/
│   ├── techno_lambda_s3.py    # (dari Saran 2)
│   ├── auth/
│   │   └── techno_lambda_auth.py
│   ├── post/
│   │   └── techno_lambda_post.py
│   └── get/
│       └── techno_lambda_get.py
└── app/
    ├── Dockerfile
    ├── requirements.txt
    └── app.py
```

---

## 🔌 Fase 1: Ekstensi Terraform dari Saran 2

### 📄 Update `variables.tf` — tambahkan:
```hcl
variable "api_name" {
  description = "Nama API Gateway"
  type        = string
  default     = "Techno-API"
}

variable "eks_cluster_name" {
  description = "Nama EKS cluster"
  type        = string
  default     = ""  # Akan di-generate otomatis
}

variable "ecr_repo_name" {
  description = "Nama ECR repository"
  type        = string
  default     = ""  # Akan di-generate otomatis
}

variable "eks_node_instance_type" {
  description = "Instance type untuk EKS node"
  type        = string
  default     = "t3.large"
}

variable "eks_node_desired_size" {
  description = "Jumlah node yang diinginkan"
  type        = number
  default     = 1
}

variable "eks_node_min_size" {
  description = "Minimum node"
  type        = number
  default     = 1
}

variable "eks_node_max_size" {
  description = "Maximum node"
  type        = number
  default     = 2
}
```

### 📄 Update `main.tf` — tambahkan locals:
```hcl
locals {
  # ... (locals dari Saran 2)
  
  api_name              = "${var.api_name}-${local.name_suffix}"
  eks_cluster_name      = "${local.prefix}-eks-${local.city_suffix}-${local.name_suffix}"
  ecr_repo_name         = "${local.prefix}-ecr-${local.city_suffix}-${local.name_suffix}"
  lambda_auth_name      = "${local.prefix}-lambda-auth"
  lambda_post_name      = "${local.prefix}-lambda-post"
  lambda_get_name       = "${local.prefix}-lambda-get"
  sg_eks_name           = "${local.prefix}-sg-eks"
}
```

---

## 🚪 Fase 2: API Gateway + Lambda Authorizer

### Tambahkan ke `main.tf`:
```hcl
# ============================================
# API GATEWAY REST
# ============================================

resource "aws_apigatewayv2_api" "main" {
  name          = local.api_name
  protocol_type = "HTTP"
  
  cors_configuration {
    allow_origins = ["*"]
    allow_methods = ["GET", "POST", "OPTIONS"]
    allow_headers = ["Content-Type", "Authorization"]
  }
  
  tags = {
    Name        = local.api_name
    Description = "REST API for token management"
  }
}

# Stage
resource "aws_apigatewayv2_stage" "prod" {
  api_id      = aws_apigatewayv2_api.main.id
  name        = "prod"
  auto_deploy = true
  
  tags = {
    Name = "${local.api_name}-stage"
  }
}

# ============================================
# INTEGRATIONS (Lambda)
# ============================================

# Integration for /generate-token
resource "aws_apigatewayv2_integration" "post_token" {
  api_id             = aws_apigatewayv2_api.main.id
  integration_type   = "AWS_PROXY"
  integration_uri    = aws_lambda_function.post.arn
  payload_format_version = "2.0"
}

# Integration for /validate-token
resource "aws_apigatewayv2_integration" "get_token" {
  api_id             = aws_apigatewayv2_api.main.id
  integration_type   = "AWS_PROXY"
  integration_uri    = aws_lambda_function.get.arn
  payload_format_version = "2.0"
}

# ============================================
# ROUTES
# ============================================

resource "aws_apigatewayv2_route" "post_token" {
  api_id    = aws_apigatewayv2_api.main.id
  route_key = "POST /generate-token"
  target    = "integrations/${aws_apigatewayv2_integration.post_token.id}"
}

resource "aws_apigatewayv2_route" "get_token" {
  api_id    = aws_apigatewayv2_api.main.id
  route_key = "GET /validate-token"
  target    = "integrations/${aws_apigatewayv2_integration.get_token.id}"
}

# Lambda permissions for API Gateway
resource "aws_lambda_permission" "apigw_post" {
  statement_id  = "AllowAPIGatewayInvokePost"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.post.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_apigatewayv2_api.main.execution_arn}/*/*"
}

resource "aws_lambda_permission" "apigw_get" {
  statement_id  = "AllowAPIGatewayInvokeGet"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.get.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_apigatewayv2_api.main.execution_arn}/*/*"
}
```

---

## ⚡ Fase 3: Lambda Functions (POST, GET, Auth)

### 📄 `lambda/post/techno_lambda_post.py`
```python
import json
import boto3
import uuid
from datetime import datetime, timedelta

dynamodb = boto3.client('dynamodb')
TABLE_NAME = 'Tokens'

def lambda_handler(event, context):
    """Generate token and save to DynamoDB"""
    try:
        # Generate unique token
        token = str(uuid.uuid4())
        device_id = event.get('headers', {}).get('deviceid', 'unknown')
        expires_at = (datetime.now() + timedelta(hours=24)).isoformat()
        
        # Save to DynamoDB
        dynamodb.put_item(
            TableName=TABLE_NAME,
            Item={
                'token': {'S': token},
                'deviceid': {'S': device_id},
                'expires_at': {'S': expires_at},
                'created_at': {'S': datetime.now().isoformat()},
                'status': {'S': 'ACTIVE'}
            }
        )
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Token generated successfully',
                'token': token,
                'expires_at': expires_at
            })
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }
```

### 📄 `lambda/get/techno_lambda_get.py`
```python
import json
import boto3
from datetime import datetime

dynamodb = boto3.client('dynamodb')
TABLE_NAME = 'Tokens'

def lambda_handler(event, context):
    """Validate token from query parameter"""
    try:
        # Get token from query string
        token = event.get('queryStringParameters', {}).get('token')
        
        if not token:
            return {
                'statusCode': 400,
                'body': json.dumps({'error': 'Token parameter required'})
            }
        
        # Query DynamoDB
        response = dynamodb.get_item(
            TableName=TABLE_NAME,
            Key={'token': {'S': token}}
        )
        
        if 'Item' not in response:
            return {
                'statusCode': 404,
                'body': json.dumps({'error': 'Token not found'})
            }
        
        item = response['Item']
        expires_at = item.get('expires_at', {}).get('S')
        
        # Check expiration
        if expires_at and datetime.fromisoformat(expires_at) < datetime.now():
            return {
                'statusCode': 401,
                'body': json.dumps({'error': 'Token expired'})
            }
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'valid': True,
                'token': token,
                'deviceid': item.get('deviceid', {}).get('S'),
                'expires_at': expires_at,
                'status': item.get('status', {}).get('S')
            })
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }
```

### 📄 `lambda/auth/techno_lambda_auth.py`
```python
import json
import boto3

dynamodb = boto3.client('dynamodb')
TABLE_NAME = 'Tokens'

def lambda_handler(event, context):
    """Lambda Authorizer for API Gateway"""
    try:
        token = event.get('identitySource', [None])[0]
        method_arn = event.get('methodArn', '*')
        
        if not token:
            return generate_policy('anonymous', 'Deny', method_arn)
        
        # Check token in DynamoDB
        response = dynamodb.get_item(
            TableName=TABLE_NAME,
            Key={'token': {'S': token}}
        )
        
        if 'Item' in response:
            return generate_policy(token, 'Allow', method_arn)
        else:
            return generate_policy(token, 'Deny', method_arn)
            
    except Exception as e:
        print(f"Auth error: {str(e)}")
        return generate_policy('error', 'Deny', event.get('methodArn', '*'))

def generate_policy(principal_id, effect, resource):
    return {
        'principalId': principal_id,
        'policyDocument': {
            'Version': '2012-10-17',
            'Statement': [{
                'Action': 'execute-api:Invoke',
                'Effect': effect,
                'Resource': resource
            }]
        }
    }
```

### Tambahkan ke `main.tf`:
```hcl
# ============================================
# LAMBDA FUNCTIONS (POST, GET, AUTH)
# ============================================

# Archive Lambda codes
data "archive_file" "lambda_post" {
  type        = "zip"
  source_file = "${path.module}/lambda/post/techno_lambda_post.py"
  output_path = "${path.module}/lambda/post/techno_lambda_post.zip"
}

data "archive_file" "lambda_get" {
  type        = "zip"
  source_file = "${path.module}/lambda/get/techno_lambda_get.py"
  output_path = "${path.module}/lambda/get/techno_lambda_get.zip"
}

data "archive_file" "lambda_auth" {
  type        = "zip"
  source_file = "${path.module}/lambda/auth/techno_lambda_auth.py"
  output_path = "${path.module}/lambda/auth/techno_lambda_auth.zip"
}

# Lambda POST
resource "aws_lambda_function" "post" {
  function_name    = local.lambda_post_name
  filename         = data.archive_file.lambda_post.output_path
  source_code_hash = data.archive_file.lambda_post.output_base64sha256
  handler          = "techno_lambda_post.lambda_handler"
  runtime          = "python3.12"
  timeout          = 60  # 1 menit sesuai soal
  memory_size      = 256
  role             = aws_iam_role.lambda_exec.arn
  
  environment {
    variables = {
      DYNAMODB_TABLE = var.dynamodb_table_name
    }
  }
  
  tags = { Name = local.lambda_post_name }
}

# Lambda GET
resource "aws_lambda_function" "get" {
  function_name    = local.lambda_get_name
  filename         = data.archive_file.lambda_get.output_path
  source_code_hash = data.archive_file.lambda_get.output_base64sha256
  handler          = "techno_lambda_get.lambda_handler"
  runtime          = "python3.12"
  timeout          = 90  # 1 menit 30 detik sesuai soal
  memory_size      = 256
  role             = aws_iam_role.lambda_exec.arn
  
  environment {
    variables = {
      DYNAMODB_TABLE = var.dynamodb_table_name
    }
  }
  
  tags = { Name = local.lambda_get_name }
}

# Lambda AUTH (Authorizer)
resource "aws_lambda_function" "auth" {
  function_name    = local.lambda_auth_name
  filename         = data.archive_file.lambda_auth.output_path
  source_code_hash = data.archive_file.lambda_auth.output_base64sha256
  handler          = "techno_lambda_auth.lambda_handler"
  runtime          = "python3.12"
  timeout          = 30
  memory_size      = 128
  role             = aws_iam_role.lambda_exec.arn
  
  tags = { Name = local.lambda_auth_name }
}

# Permission for authorizer
resource "aws_lambda_permission" "apigw_auth" {
  statement_id  = "AllowAPIGatewayInvokeAuth"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.auth.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_apigatewayv2_api.main.execution_arn}/authorizers/*"
}

# CloudWatch Log Groups
resource "aws_cloudwatch_log_group" "post" {
  name              = "/aws/lambda/${local.lambda_post_name}"
  retention_in_days = 7
}

resource "aws_cloudwatch_log_group" "get" {
  name              = "/aws/lambda/${local.lambda_get_name}"
  retention_in_days = 7
}

resource "aws_cloudwatch_log_group" "auth" {
  name              = "/aws/lambda/${local.lambda_auth_name}"
  retention_in_days = 7
}
```

---

## 📦 Fase 4: ECR Repository

### Tambahkan ke `main.tf`:
```hcl
# ============================================
# ECR REPOSITORY
# ============================================

resource "aws_ecr_repository" "main" {
  name                 = local.ecr_repo_name
  image_tag_mutability = "MUTABLE"
  
  image_scanning_configuration {
    scan_on_push = true
  }
  
  encryption_configuration {
    encryption_type = "AES256"
  }
  
  tags = {
    Name        = local.ecr_repo_name
    Description = "ECR for ML Python app"
  }
}

# Lifecycle policy to keep only last 5 images
resource "aws_ecr_lifecycle_policy" "main" {
  repository = aws_ecr_repository.main.name
  
  policy = jsonencode({
    rules = [{
      rulePriority = 1
      description  = "Keep last 5 images"
      selection = {
        tagStatus   = "any"
        countType   = "imageCountMoreThan"
        countNumber = 5
      }
      action = {
        type = "expire"
      }
    }]
  })
}
```

---

## 🐍 Fase 5: Python App + Dockerfile

### 📄 `app/requirements.txt`
```
flask==3.0.0
boto3==1.34.0
gunicorn==21.2.0
```

### 📄 `app/app.py`
```python
from flask import Flask, jsonify, request
import boto3
import os
from datetime import datetime

app = Flask(__name__)

# AWS clients
dynamodb = boto3.client('dynamodb')
s3 = boto3.client('s3')

TABLE_NAME = os.environ.get('DYNAMODB_TABLE', 'Tokens')
OUTPUT_BUCKET = os.environ.get('OUTPUT_BUCKET', '')

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
        return jsonify({
            'success': True,
            'result': {
                'token': item.get('token', {}).get('S'),
                'image_name': item.get('image_name', {}).get('S'),
                'labels': item.get('labels', {}).get('S'),
                'label_count': item.get('label_count', {}).get('N'),
                'status': item.get('status', {}).get('S'),
                'timestamp': item.get('timestamp', {}).get('S')
            }
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

### 📄 `app/Dockerfile`
```dockerfile
FROM python:3.13-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY app.py .

# Expose port (sesuai soal: 2000)
EXPOSE 2000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:2000/health || exit 1

# Run with gunicorn
CMD ["gunicorn", "--bind", "0.0.0.0:2000", "--workers", "2", "app:app"]
```

---

## ☸️ Fase 6: EKS Cluster + Node Group

### Tambahkan ke `main.tf`:
```hcl
# ============================================
# EKS CLUSTER
# ============================================

# IAM Role for EKS Cluster
resource "aws_iam_role" "eks_cluster" {
  name = "${local.prefix}-eks-cluster-role-${local.name_suffix}"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "eks.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.eks_cluster.name
}

# Security Group for EKS
resource "aws_security_group" "eks" {
  name        = local.sg_eks_name
  description = "Security group for EKS nodes"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    description = "App port from ALB"
    from_port   = 2000
    to_port     = 2000
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    description = "Node to node communication"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    self        = true
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = { Name = local.sg_eks_name }
}

# EKS Cluster
resource "aws_eks_cluster" "main" {
  name     = local.eks_cluster_name
  role_arn = aws_iam_role.eks_cluster.arn
  version  = "1.28"
  
  vpc_config {
    subnet_ids = concat(
      aws_subnet.private[*].id,
      aws_subnet.public[*].id
    )
    security_group_ids = [aws_security_group.eks.id]
    endpoint_public_access = true
    endpoint_private_access = true
  }
  
  tags = {
    Name        = local.eks_cluster_name
    Description = "EKS cluster for ML application"
  }
  
  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy
  ]
}

# IAM Role for EKS Node Group
resource "aws_iam_role" "eks_nodes" {
  name = "${local.prefix}-eks-node-role-${local.name_suffix}"
  
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

resource "aws_iam_role_policy_attachment" "eks_worker_node" {
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

# Inline policy untuk akses DynamoDB, S3, Rekognition
resource "aws_iam_role_policy" "eks_nodes_custom" {
  name = "${local.prefix}-eks-nodes-custom-policy"
  role = aws_iam_role.eks_nodes.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:Query",
          "dynamodb:Scan",
          "dynamodb:PutItem"
        ]
        Resource = aws_dynamodb_table.tokens.arn
      },
      {
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
        Effect = "Allow"
        Action = [
          "rekognition:DetectLabels",
          "rekognition:DetectFaces"
        ]
        Resource = "*"
      }
    ]
  })
}

# EKS Node Group
resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "${local.prefix}-node-group-${local.name_suffix}"
  node_role_arn   = aws_iam_role.eks_nodes.arn
  subnet_ids      = aws_subnet.private[*].id
  
  instance_types = [var.eks_node_instance_type]  # t3.large sesuai soal
  
  scaling_config {
    desired_size = var.eks_node_desired_size
    max_size     = var.eks_node_max_size
    min_size     = var.eks_node_min_size
  }
  
  update_config {
    max_unavailable = 1
  }
  
  tags = {
    Name = "${local.prefix}-node-group"
  }
  
  depends_on = [
    aws_iam_role_policy_attachment.eks_worker_node,
    aws_iam_role_policy_attachment.eks_cni,
    aws_iam_role_policy_attachment.eks_ecr
  ]
}
```

---

## 📝 Fase 7: Kubernetes Manifests

### 📄 `app/deploy.yaml`
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

### 📄 `app/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: techno-ml-service
  labels:
    app: techno-ml-app
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

## 🚀 Fase 8: Build, Push, Deploy

### Langkah 8.1: Terraform Apply
```bash
cd ~/techno-ml-project
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

### Langkah 8.2: Configure kubectl
```bash
# Update kubeconfig
aws eks update-kubeconfig \
  --region us-east-1 \
  --name $(terraform output -raw eks_cluster_name)

# Verify
kubectl get nodes
# Harus muncul 1 node t3.large dengan status Ready
```

### Langkah 8.3: Build & Push Docker Image
```bash
# Login ke ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  $(terraform output -raw ecr_repository_url | cut -d'/' -f1)

# Build image
cd ~/techno-ml-project/app
COMMIT_SHA=$(git rev-parse --short HEAD 2>/dev/null || echo "v1.0.0")
ECR_URL=$(terraform -chdir=.. output -raw ecr_repository_url)

docker build -t techno-ml-app:${COMMIT_SHA} .
docker tag techno-ml-app:${COMMIT_SHA} ${ECR_URL}:${COMMIT_SHA}
docker tag techno-ml-app:${COMMIT_SHA} ${ECR_URL}:latest

# Push
docker push ${ECR_URL}:${COMMIT_SHA}
docker push ${ECR_URL}:latest
```

### Langkah 8.4: Update & Apply Kubernetes Manifests
```bash
# Ganti placeholder di deploy.yaml
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
CITY="jakarta"  # ganti dengan kota Anda
NAME="budi"     # ganti dengan nama Anda

sed -i "s|ACCOUNT_ID|${ACCOUNT_ID}|g" deploy.yaml
sed -i "s|CITY|${CITY}|g" deploy.yaml
sed -i "s|NAME|${NAME}|g" deploy.yaml

# Apply
kubectl apply -f deploy.yaml
kubectl apply -f service.yaml

# Verify
kubectl get deployments
kubectl get pods
kubectl get services
```

### Langkah 8.5: Tunggu LoadBalancer Ready
```bash
# Lihat external IP (butuh ~2-3 menit)
kubectl get svc techno-ml-service -w

# Catat EXTERNAL-IP untuk testing
```

---

## 🧪 Fase 9: Testing End-to-End

### Test 1: API Gateway - Generate Token
```bash
API_URL=$(terraform output -raw api_endpoint)

# Generate token
curl -X POST ${API_URL}/generate-token \
  -H "Content-Type: application/json" \
  -H "deviceid: test-device-001"

# Simpan token dari response
TOKEN="<token_dari_response>"
```

### Test 2: API Gateway - Validate Token
```bash
curl "${API_URL}/validate-token?token=${TOKEN}"
# Harus return: {"valid": true, "token": "...", "deviceid": "test-device-001"}
```

### Test 3: Upload Gambar ke S3
```bash
curl -o test-image.jpg https://picsum.photos/800/600.jpg
aws s3 cp test-image.jpg s3://$(terraform output -raw s3_input_bucket)/test-image.jpg

# Tunggu ~30 detik untuk Lambda processing
```

### Test 4: Cek DynamoDB
```bash
aws dynamodb scan --table-name Tokens --max-items 5
```

### Test 5: Akses Python App via LoadBalancer
```bash
APP_URL=$(kubectl get svc techno-ml-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Health check
curl http://${APP_URL}/

# List results
curl http://${APP_URL}/results

# Get specific result
curl http://${APP_URL}/results/${TOKEN}

# Validate token via app
curl "http://${APP_URL}/validate-token?token=${TOKEN}"
```

### Test 6: Cek EKS Pods
```bash
kubectl get pods -l app=techno-ml-app
# Harus ada 2 pods RUNNING

kubectl logs -l app=techno-ml-app --tail=20
```

### Test 7: Cek Kinesis Stream
```bash
# Upload gambar lagi
aws s3 cp test-image.jpg s3://$(terraform output -raw s3_input_bucket)/test-$(date +%s).jpg

# Cek Kinesis records
aws kinesis get-shard-iterator \
  --stream-name $(terraform output -raw kinesis_stream_name) \
  --shard-id shardId-000000000000 \
  --shard-iterator-type TRIM_HORIZON \
  --query 'ShardIterator' --output text | \
  xargs -I {} aws kinesis get-records --shard-iterator {}
```

---

## 🧹 Fase 10: Cleanup

### Langkah 10.1: Delete Kubernetes Resources
```bash
kubectl delete -f app/service.yaml
kubectl delete -f app/deploy.yaml
```

### Langkah 10.2: Empty S3 Buckets
```bash
aws s3 rm s3://$(terraform output -raw s3_input_bucket) --recursive
# Jika ada output bucket, kosongkan juga
```

### Langkah 10.3: Terraform Destroy
```bash
cd ~/techno-ml-project
terraform destroy -auto-approve
```

> ⚠️ **EKS Node Group** kadang butuh waktu lama untuk delete (~10 menit). Jika stuck, force delete via console.

### Langkah 10.4: Verify Cleanup
```bash
terraform state list  # harus kosong
aws eks list-clusters --query 'clusters[?starts_with(@, `techno-`)]'
aws ecr describe-repositories --query 'repositories[?starts_with(repositoryName, `techno-`)]'
```

---

## 🎓 Checklist Juri (Self-Assessment)

| # | Item | Status |
|---|---|---|
| 1 | Region: `us-east-1` | ☐ |
| 2 | Semua resource via Terraform | ☐ |
| 3 | API Gateway `Techno-API-<nama>` (HTTP API, regional) | ☐ |
| 4 | Route `POST /generate-token` → Lambda techno-lambda-post | ☐ |
| 5 | Route `GET /validate-token` → Lambda techno-lambda-get | ☐ |
| 6 | Lambda techno-lambda-post timeout 1 menit | ☐ |
| 7 | Lambda techno-lambda-get timeout 1 menit 30 detik | ☐ |
| 8 | Lambda techno-lambda-auth sebagai authorizer | ☐ |
| 9 | ECR `techno-ecr-<kota>-<nama>` (mutable) | ☐ |
| 10 | EKS cluster `techno-eks-<kota>-<nama>` (v1.28) | ☐ |
| 11 | Node group dengan instance `t3.large` | ☐ |
| 12 | Node di private subnet | ☐ |
| 13 | Docker image Python 3.13 + Flask | ☐ |
| 14 | Image pushed ke ECR dengan tag commit SHA + latest | ☐ |
| 15 | K8s Deployment dengan 2 replicas | ☐ |
| 16 | K8s Service type LoadBalancer | ☐ |
| 17 | App accessible via LoadBalancer URL | ☐ |
| 18 | App listen on port 2000 | ☐ |
| 19 | Token generation via API Gateway works | ☐ |
| 20 | Token validation via API Gateway works | ☐ |
| 21 | S3 upload → Lambda → Rekognition → DynamoDB works | ☐ |
| 22 | DynamoDB changes → Kinesis stream works | ☐ |
| 23 | App bisa query hasil analisis dari DynamoDB | ☐ |
| 24 | Semua resource di-tag & pakai prefix `techno-` | ☐ |
| 25 | Cleanup selesai | ☐ |

---

## 💡 Tips Trainer

### 🔴 Jika EKS Node Group stuck di `CREATE_IN_PROGRESS`
**Penyebab**: IAM role belum propagate atau subnet tidak punya route ke internet (untuk pull image).
**Solusi**:
1. Cek IAM role sudah attach 3 policy: `AmazonEKSWorkerNodePolicy`, `AmazonEKS_CNI_Policy`, `AmazonEC2ContainerRegistryReadOnly`
2. Pastikan private subnet punya route ke NAT Gateway
3. Tunggu max 15 menit

### 🔴 Jika `kubectl` Error "couldn't get current server API group list"
**Penyebab**: kubeconfig belum ter-update atau cluster belum ready.
**Solusi**:
```bash
aws eks update-kubeconfig --region us-east-1 --name <cluster-name>
kubectl cluster-info
```

### 🔴 Jika Pod stuck di `ImagePullBackOff`
**Penyebab**: Node tidak bisa pull image dari ECR.
**Solusi**:
1. Cek IAM role node punya `AmazonEC2ContainerRegistryReadOnly`
2. Cek image URI di `deploy.yaml` sudah benar
3. Cek image sudah di-push ke ECR

### 🔴 Jika LoadBalancer tidak dapat External IP
**Penyebab**: AWS Controller for Kubernetes belum install atau subnet tidak public-facing.
**Solusi**:
1. Tunggu ~3-5 menit (AWS provision NLB/CLB butuh waktu)
2. Cek service: `kubectl describe svc techno-ml-service`
3. Pastikan subnet di VPC punya tag untuk load balancer

### 🔴 Jika API Gateway Return 403
**Penyebab**: Lambda permission belum ter-attach atau authorizer menolak.
**Solusi**:
1. Cek `aws_lambda_permission` sudah ada untuk API Gateway
2. Test Lambda langsung via console
3. Cek CloudWatch Logs Lambda

### 🔴 Jika Terraform Destroy Gagal di EKS
**Penyebab**: Node group masih ada running pods.
**Solusi**:
```bash
# Delete semua resources di cluster dulu
kubectl delete all --all -n default
terraform destroy
```

---

## 📊 Apa yang Anda Pelajari dari Saran 3?

| Skill | Status |
|---|---|
| Semua skill dari Saran 1 & 2 | ✅ |
| **API Gateway HTTP API** (skill baru!) | ✅ |
| **Lambda Authorizer** (skill baru!) | ✅ |
| **ECR** repository dengan lifecycle policy | ✅ |
| **EKS Cluster** provisioning via Terraform | ✅ |
| **EKS Node Group** dengan IAM roles | ✅ |
| **Docker build & push** to ECR | ✅ |
| **Kubernetes Deployment** YAML | ✅ |
| **Kubernetes Service** (LoadBalancer) YAML | ✅ |
| **kubectl** commands | ✅ |
| Multi-Lambda architecture (S3, POST, GET, AUTH) | ✅ |

---

## 🚀 Progression Path

```
✅ Saran 1 (Console, S3 → Lambda → Rekognition → DynamoDB) — 1.5 jam
    ↓
✅ Saran 2 (Terraform + DynamoDB Streams → Kinesis) — 2 jam
    ↓
✅ Saran 3 (Terraform + API Gateway + EKS + ECR) — 2.5 jam ← ANDA DI SINI
    ↓
🔴 Saran 4 (Full: + Glue Crawler, SNS, IPv6, Strict Naming) — 4-5 jam
    ↓
🏆 SOAL LKS Machine Learning (6 jam, Terraform Wajib)
```

---

## 🎯 Analogi Langsung ke Soal LKS

| Yang Anda Lakukan di Saran 3 | Yang Akan Anda Lakukan di Soal LKS |
|---|---|
| API Gateway HTTP API | **API Gateway REST** (dual-stack IPv4+IPv6) |
| 4 Lambda (s3, post, get, auth) | **3 Lambda** (s3, post, get) + authorizer |
| EKS dengan 1 node t3.large | **EKS dengan 1 node t3.large** (sama!) |
| 2 replicas | **2 pods** (sama!) |
| ECR 1 repo | **ECR 1 repo** (sama!) |
| Python Flask app | **Python app dari repo GitHub** |
| deploy.yaml + service.yaml | **deploy.yaml + service.yaml** (sama!) |
| Tidak ada Glue | **AWS Glue** (Database + Table + Crawler) |
| Tidak ada SNS | **SNS** (email subscription) |
| Tidak ada IPv6 | **VPC + API Gateway IPv6** |
| Prefix `techno-` | Prefix `techno-` **ketat** |

---

## 🎯 Next Step

Setelah Saran 3 berhasil, Anda siap lanjut ke **Saran 4**: menambahkan **AWS Glue** (Database + Table + Crawler), **SNS** (email notification), **IPv6** penuh di VPC dan API Gateway, serta memperketat naming convention.

Apakah Anda ingin saya buatkan:
1. **Panduan lengkap Saran 4** (Glue + SNS + IPv6 + Full Terraform)?
2. **Template Terraform module** untuk reusable EKS/VPC components?
3. **Script testing otomatis** untuk memverifikasi seluruh pipeline?

Silakan pilih, dan saya akan buatkan! 🎯

**Selamat! Anda telah menguasai API Gateway + EKS + ECR — fondasi container orchestration untuk soal LKS Machine Learning.** 🎉