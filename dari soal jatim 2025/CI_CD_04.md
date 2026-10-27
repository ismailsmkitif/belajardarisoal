# 📘 Panduan Lengkap Pengerjaan Saran 4 (DevOps CI/CD)
## "Full CI/CD Pipeline" — 2 Custom Apps + ECR + GitHub Actions

> **Estimasi waktu**: 4-5 jam | **Region**: `us-east-1` | **Console + GitHub**

---

## 🎯 Tujuan Latihan

Membangun **pipeline CI/CD end-to-end** yang mengotomasi build, push, dan deploy container — persis seperti soal LKS DevOps sesungguhnya:
- ✅ 2 aplikasi container **custom** (Node.js Express)
- ✅ **ECR** sebagai private container registry
- ✅ **GitHub Repository** dengan source code
- ✅ **GitHub Fine-Grained Token** untuk authentication
- ✅ **GitHub Actions** dengan multiple jobs paralel
- ✅ **ECS Fargate** + **ALB Multi-Port** + **RDS** + **EFS** + **SSM**
- ✅ **CloudWatch Logs** untuk monitoring

Ini adalah **step 3** dan **latihan terakhir** sebelum soal LKS DevOps CI/CD.

---

## 📋 Daftar Isi
1. [Fase 0: Persiapan GitHub & Repository](#fase-0-persiapan)
2. [Fase 1: Networking (VPC)](#fase-1-networking-vpc)
3. [Fase 2: Security Groups (5-Tier)](#fase-2-security-groups)
4. [Fase 3: RDS MySQL Database](#fase-3-rds-mysql-database)
5. [Fase 4: EFS + Access Points](#fase-4-efs--access-points)
6. [Fase 5: SSM Parameter Store](#fase-5-ssm-parameter-store)
7. [Fase 6: ECR Repositories](#fase-6-ecr-repositories)
8. [Fase 7: Target Groups + ALB Multi-Port](#fase-7-target-groups--alb)
9. [Fase 8: ECS Cluster + Task Definitions](#fase-8-ecs-cluster)
10. [Fase 9: ECS Services](#fase-9-ecs-services)
11. [Fase 10: IAM Policy untuk GitHub Actions](#fase-10-iam-policy)
12. [Fase 11: GitHub Fine-Grained Token](#fase-11-github-token)
13. [Fase 12: GitHub Secrets](#fase-12-github-secrets)
14. [Fase 13: GitHub Actions Workflow](#fase-13-github-actions)
15. [Fase 14: Trigger Pipeline & Verify](#fase-14-trigger-pipeline)
16. [Fase 15: Testing End-to-End](#fase-15-testing)
17. [Fase 16: Cleanup](#fase-16-cleanup)

---

## 🎯 Fase 0: Persiapan GitHub & Repository

### Langkah 0.1: Login & Set Region
1. Login ke **AWS Lab Learner** → klik **Start Lab** → **Open AWS Console**.
2. Region: **US East (N. Virginia) `us-east-1`**.
3. Buka **Notepad** untuk mencatat semua resource IDs.

### Langkah 0.2: Buat GitHub Repository
1. Login ke GitHub → **New repository**.
2. Name: `lks-cicd-project`
3. Visibility: **Private**
4. ✅ Add a README
5. **Create repository**.

### Langkah 0.3: Siapkan Struktur Folder
Buat struktur folder di komputer lokal:

```
lks-cicd-project/
├── .github/
│   └── workflows/
│       ├── example-build-main.yml       (contoh, akan dihapus)
│       ├── example-deploy-main.yml      (contoh, akan dihapus)
│       └── lks-deployment-pipeline.yml  (yang Anda buat)
├── main-app/
│   ├── Dockerfile
│   ├── task-definition.json
│   ├── package.json
│   └── src/
│       └── index.js
├── monitor-app/
│   ├── Dockerfile
│   ├── task-definition.json
│   ├── package.json
│   └── src/
│       └── index.js
└── README.md
```

### Langkah 0.4: Analogi dengan Soal LKS

| Saran 4 (Latihan) | Soal LKS (Sesungguhnya) |
|---|---|
| `main-app` (Node.js Express) | **Leantime** (PHP project management) |
| `monitor-app` (Node.js health-check) | **Gatus** (Go monitoring dashboard) |
| ECR `lks-main-app` | ECR `lks-gatus` |
| GitHub Actions 4 jobs | GitHub Actions 3 jobs |
| Fine-Grained Token | Fine-Grained Token |

---

## 📁 Source Code Aplikasi

### 📄 `main-app/Dockerfile`
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY src/ ./src/
EXPOSE 80
ENV NODE_ENV=production
ENV PORT=80
CMD ["node", "src/index.js"]
```

### 📄 `main-app/package.json`
```json
{
  "name": "lks-main-app",
  "version": "1.0.0",
  "description": "Main application for LKS CI/CD training",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "mysql2": "^3.6.0"
  }
}
```

### 📄 `main-app/src/index.js`
```javascript
const express = require('express');
const mysql = require('mysql2/promise');
const fs = require('fs');
const app = express();
const PORT = process.env.PORT || 80;

// Health check endpoint
app.get('/', (req, res) => {
  res.status(200).json({
    status: 'healthy',
    app: 'lks-main-app',
    timestamp: new Date().toISOString(),
    environment: process.env.NODE_ENV || 'development'
  });
});

// Test database connection
app.get('/db-test', async (req, res) => {
  try {
    const connection = await mysql.createConnection({
      host: process.env.DB_HOST,
      port: process.env.DB_PORT || 3306,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      database: process.env.DB_NAME
    });
    const [rows] = await connection.execute('SELECT 1 + 1 AS result');
    await connection.end();
    res.json({ status: 'connected', result: rows[0].result });
  } catch (err) {
    res.status(500).json({ status: 'error', message: err.message });
  }
});

// Test EFS write
app.post('/write-file', (req, res) => {
  try {
    const filename = `/app/data/test-${Date.now()}.txt`;
    fs.writeFileSync(filename, `Written at ${new Date().toISOString()}`);
    res.json({ status: 'success', file: filename });
  } catch (err) {
    res.status(500).json({ status: 'error', message: err.message });
  }
});

// List files in EFS
app.get('/list-files', (req, res) => {
  try {
    const files = fs.readdirSync('/app/data');
    res.json({ status: 'success', files });
  } catch (err) {
    res.status(500).json({ status: 'error', message: err.message });
  }
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`Main app listening on port ${PORT}`);
});
```

### 📄 `monitor-app/Dockerfile`
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY src/ ./src/
EXPOSE 8080
ENV NODE_ENV=production
ENV PORT=8080
CMD ["node", "src/index.js"]
```

### 📄 `monitor-app/package.json`
```json
{
  "name": "lks-monitor-app",
  "version": "1.0.0",
  "description": "Health monitoring application",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "node-fetch": "^2.7.0"
  }
}
```

### 📄 `monitor-app/src/index.js`
```javascript
const express = require('express');
const fetch = require('node-fetch');
const app = express();
const PORT = process.env.PORT || 8080;
const TARGET_URL = process.env.TARGET_URL || 'http://localhost:80';

app.get('/', (req, res) => {
  res.status(200).json({
    status: 'healthy',
    app: 'lks-monitor-app',
    timestamp: new Date().toISOString()
  });
});

app.get('/check', async (req, res) => {
  const start = Date.now();
  try {
    const response = await fetch(TARGET_URL, { timeout: 5000 });
    const duration = Date.now() - start;
    res.json({
      target: TARGET_URL,
      status: response.status,
      responseTime: `${duration}ms`,
      healthy: response.status === 200 && duration < 300
    });
  } catch (err) {
    res.status(500).json({
      target: TARGET_URL,
      status: 'error',
      message: err.message
    });
  }
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`Monitor app listening on port ${PORT}`);
});
```

### 📄 `main-app/task-definition.json`
```json
{
  "family": "lks-main-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::ACCOUNT_ID:role/LabRole",
  "taskRoleArn": "arn:aws:iam::ACCOUNT_ID:role/LabRole",
  "containerDefinitions": [
    {
      "name": "lks-main-container",
      "image": "ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/lks-main-app:latest",
      "portMappings": [
        {
          "containerPort": 80,
          "hostPort": 80,
          "protocol": "tcp"
        }
      ],
      "essential": true,
      "environment": [
        { "name": "NODE_ENV", "value": "production" },
        { "name": "PORT", "value": "80" }
      ],
      "secrets": [
        { "name": "DB_HOST", "valueFrom": "/lks/cicd/db/host" },
        { "name": "DB_PORT", "valueFrom": "/lks/cicd/db/port" },
        { "name": "DB_NAME", "valueFrom": "/lks/cicd/db/name" },
        { "name": "DB_USER", "valueFrom": "/lks/cicd/db/username" },
        { "name": "DB_PASSWORD", "valueFrom": "/lks/cicd/db/password" }
      ],
      "mountPoints": [
        {
          "sourceVolume": "efs-main-data",
          "containerPath": "/app/data",
          "readOnly": false
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/lks/main-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "main"
        }
      }
    }
  ],
  "volumes": [
    {
      "name": "efs-main-data",
      "efsVolumeConfiguration": {
        "fileSystemId": "FS_ID_HERE",
        "rootDirectory": "/main-data",
        "transitEncryption": "ENABLED",
        "authorizationConfig": {
          "accessPointId": "FSAP_ID_HERE",
          "iam": "ENABLED"
        }
      }
    }
  ]
}
```

### 📄 `monitor-app/task-definition.json`
```json
{
  "family": "lks-monitor-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::ACCOUNT_ID:role/LabRole",
  "taskRoleArn": "arn:aws:iam::ACCOUNT_ID:role/LabRole",
  "containerDefinitions": [
    {
      "name": "lks-monitor-container",
      "image": "ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/lks-monitor-app:latest",
      "portMappings": [
        {
          "containerPort": 8080,
          "hostPort": 8080,
          "protocol": "tcp"
        }
      ],
      "essential": true,
      "environment": [
        { "name": "NODE_ENV", "value": "production" },
        { "name": "PORT", "value": "8080" },
        { "name": "TARGET_URL", "value": "ALB_ENDPOINT_HERE" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/lks/monitor-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "monitor"
        }
      }
    }
  ]
}
```

### Langkah 0.5: Push ke GitHub
```bash
cd lks-cicd-project
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/lks-cicd-project.git
git push -u origin main
```

---

## 🌐 Fase 1: Networking (VPC)

> 💡 **Reuse dari Saran 3** dengan sedikit penyesuaian nama.

1. **VPC** → **Create VPC** → **VPC and more**:
   - Name: `lks-cicd-vpc`
   - CIDR: `10.80.0.0/16`
   - 2 AZ, 2 public subnets, 2 private subnets
   - ✅ NAT gateways (1)
2. Rename resources:
   - IGW: `lks-cicd-igw`
   - NAT: `lks-cicd-natgw`
   - Public RT: `lks-cicd-public-rt`
   - Private RT: `lks-cicd-private-rt`

---

## 🔒 Fase 2: Security Groups (5-Tier)

Buat 5 Security Groups (sama seperti Saran 3):

| SG Name | Inbound | Source |
|---|---|---|
| `lks-sg-alb` | HTTP 80, 8080 | `0.0.0.0/0` |
| `lks-sg-main` | TCP 80 | `lks-sg-alb` |
| `lks-sg-monitor` | TCP 8080 | `lks-sg-alb` |
| `lks-sg-rds` | MySQL 3306 | `lks-sg-main` |
| `lks-sg-efs` | NFS 2049 | `lks-sg-main` |

---

## 🗄️ Fase 3: RDS MySQL Database

1. **RDS** → **Create database** → MySQL 8.0
2. Identifier: `lks-cicd-db`
3. Username: `cicdadmin`
4. Password: `CicdPassword2026!Secure`
5. Initial DB: `cicd_db`
6. VPC: `lks-cicd-vpc`, Public access: **No**
7. SG: `lks-sg-rds`

---

## 📁 Fase 4: EFS + Access Points

### Langkah 4.1: Buat EFS
1. **EFS** → **Create file system**
2. Name: `lks-cicd-efs`
3. VPC: `lks-cicd-vpc`
4. Throughput: **Elastic**
5. ✅ Encryption at rest
6. Subnets: kedua private subnet
7. SG: `lks-sg-efs`

### Langkah 4.2: Buat Access Point
1. **Access Points** → **Create access point**
2. Name: `lks-ap-main-data`
3. Root directory: `/main-data`
4. POSIX User: UID `1000`, GID `1000`
5. Permissions: `775` (rwxrwxr-x)

---

## 🔑 Fase 5: SSM Parameter Store

Buat 5 parameter:

| Name | Type | Value |
|---|---|---|
| `/lks/cicd/db/host` | String | RDS endpoint |
| `/lks/cicd/db/port` | String | `3306` |
| `/lks/cicd/db/name` | String | `cicd_db` |
| `/lks/cicd/db/username` | SecureString | `cicdadmin` |
| `/lks/cicd/db/password` | SecureString | `CicdPassword2026!Secure` |

---

## 📦 Fase 6: ECR Repositories

### Langkah 6.1: Buat Repository untuk Main App
1. **ECR** → **Repositories** → **Create repository**
2. Name: `lks-main-app`
3. Tag mutability: **Mutable** (WAJIB!)
4. ✅ Scan on push
5. **Create**

### Langkah 6.2: Buat Repository untuk Monitor App
1. **Create repository** lagi
2. Name: `lks-monitor-app`
3. Tag mutability: **Mutable**
4. **Create**

### Langkah 6.3: Catat Repository URIs
```
ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/lks-main-app
ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/lks-monitor-app
```

---

## 🌐 Fase 7: Target Groups + ALB

### Langkah 7.1: Target Group Main App
1. **EC2** → **Target Groups** → **Create**
2. Name: `lks-main-app-tg`
3. Target type: **IP addresses**
4. Port: **80**
5. Health check:
   - Path: `/`
   - Success code: `200`
   - Healthy: 3, Unhealthy: 2
   - Interval: 30s

### Langkah 7.2: Target Group Monitor App
1. Name: `lks-monitor-app-tg`
2. Port: **8080**
3. Health check:
   - Path: `/`
   - Success code: `200`
   - Healthy: 5, Unhealthy: 3
   - Interval: 60s

### Langkah 7.3: ALB Multi-Port
1. **Create Load Balancer** → Application
2. Name: `lks-cicd-alb`
3. Scheme: **Internet-facing**
4. Subnets: kedua public subnet
5. SG: `lks-sg-alb`
6. **Listeners**:
   - HTTP:80 → `lks-main-app-tg`
   - HTTP:8080 → `lks-monitor-app-tg`

---

## 🎯 Fase 8: ECS Cluster + Task Definitions

### Langkah 8.1: ECS Cluster
1. **ECS** → **Clusters** → **Create**
2. Name: `lks-cicd-cluster`
3. Infrastructure: **Fargate**

### Langkah 8.2: Task Definition Main App
> ⚠️ **JANGAN buat manual!** Task definition akan di-render otomatis oleh GitHub Actions dari file `task-definition.json`.

Tapi untuk testing awal, Anda bisa buat manual dulu:
1. **Task definitions** → **Create**
2. Name: `lks-main-app`
3. Ganti `ACCOUNT_ID`, `FS_ID_HERE`, `FSAP_ID_HERE` di `task-definition.json`
4. Upload JSON via **JSON tab** di console

### Langkah 8.3: Task Definition Monitor App
Sama seperti di atas, gunakan `monitor-app/task-definition.json`.

---

## 🚀 Fase 9: ECS Services

### Langkah 9.1: Service Main App
1. **Services** → **Create**
2. Task definition: `lks-main-app`
3. Service name: `lks-main-service`
4. Desired tasks: **2**
5. Subnets: private, SG: `lks-sg-main`
6. ALB: `lks-cicd-alb` → listener 80 → `lks-main-app-tg`

### Langkah 9.2: Service Monitor App
1. Task definition: `lks-monitor-app`
2. Service name: `lks-monitor-service`
3. Desired tasks: **2**
4. Subnets: private, SG: `lks-sg-monitor`
5. ALB: `lks-cicd-alb` → listener 8080 → `lks-monitor-app-tg`

---

## 👤 Fase 10: IAM Policy untuk GitHub Actions

### Langkah 10.1: Tambah Inline Policy ke LabRole
1. **IAM** → **Roles** → `LabRole` → **Add permissions** → **Inline policy**
2. JSON:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:CompleteLayerUpload",
        "ecr:GetDownloadUrlForLayer",
        "ecr:InitiateLayerUpload",
        "ecr:PutImage",
        "ecr:UploadLayerPart"
      ],
      "Resource": "arn:aws:ecr:us-east-1:*:repository/lks-*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecs:RegisterTaskDefinition",
        "ecs:DescribeTaskDefinition",
        "ecs:DescribeServices",
        "ecs:UpdateService",
        "ecs:DescribeClusters"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iam:PassRole"
      ],
      "Resource": "arn:aws:iam::*:role/LabRole"
    }
  ]
}
```

3. Name: `lks-cicd-github-actions-policy`

---

## 🔐 Fase 11: GitHub Fine-Grained Token

### Langkah 11.1: Buat Token
1. GitHub → **Settings** → **Developer settings** → **Personal access tokens** → **Fine-grained tokens**
2. **Generate new token**
3. Name: `lks-cicd-token`
4. Expiration: 30 days
5. **Repository access**: **Only select repositories** → pilih `lks-cicd-project`
6. **Permissions** → **Repository permissions**:
   - **Issues**: **Read and write** (untuk alerting)
   - **Metadata**: **Read-only** (otomatis)
7. **Generate token**
8. **COPY TOKEN SEKARANG** (tidak bisa dilihat lagi!)

---

## 🔑 Fase 12: GitHub Secrets

### Langkah 12.1: Tambah Secrets ke Repository
1. GitHub repo → **Settings** → **Secrets and variables** → **Actions**
2. **New repository secret** (tambahkan 1 per 1):

| Name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Dari AWS Lab Details |
| `AWS_SECRET_ACCESS_KEY` | Dari AWS Lab Details |
| `AWS_SESSION_TOKEN` | Dari AWS Lab Details |
| `AWS_REGION` | `us-east-1` |
| `AWS_ACCOUNT_ID` | 12 digit Account ID |
| `ECR_MAIN` | `lks-main-app` |
| `ECR_MONITOR` | `lks-monitor-app` |
| `ECS_CLUSTER` | `lks-cicd-cluster` |
| `ECS_SERVICE_MAIN` | `lks-main-service` |
| `ECS_SERVICE_MONITOR` | `lks-monitor-service` |
| `ALB_ENDPOINT` | `http://lks-cicd-alb-xxx.us-east-1.elb.amazonaws.com` |
| `TOKEN` | Fine-grained token dari Fase 11 |

---

## ⚙️ Fase 13: GitHub Actions Workflow

### Langkah 13.1: Buat File Workflow
Buat file `.github/workflows/lks-deployment-pipeline.yml`:

```yaml
name: LKS CI/CD Deployment Pipeline

on:
  push:
    branches: [main]
    paths:
      - 'main-app/**'
      - 'monitor-app/**'
      - '.github/workflows/lks-deployment-pipeline.yml'

env:
  AWS_REGION: ${{ vars.AWS_REGION }}
  AWS_ACCOUNT_ID: ${{ vars.AWS_ACCOUNT_ID }}

jobs:
  # ============================================
  # JOB 1: Build & Push Main App Image
  # ============================================
  build-main:
    name: Build Main App
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.build.outputs.image_tag }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image to ECR
        id: build
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: ${{ secrets.ECR_MAIN }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          cd main-app
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
          echo "image_tag=$IMAGE_TAG" >> $GITHUB_OUTPUT

  # ============================================
  # JOB 2: Deploy Main App to ECS
  # ============================================
  deploy-main:
    name: Deploy Main App
    needs: build-main
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Fill task definition template
        id: render
        run: |
          cd main-app
          sed -i "s|ACCOUNT_ID|${{ env.AWS_ACCOUNT_ID }}|g" task-definition.json
          sed -i "s|FS_ID_HERE|${{ secrets.EFS_ID }}|g" task-definition.json
          sed -i "s|FSAP_ID_HERE|${{ secrets.EFS_AP_ID }}|g" task-definition.json
          cat task-definition.json

      - name: Register ECS task definition
        id: register-task
        run: |
          TASK_DEF=$(aws ecs register-task-definition \
            --cli-input-json file://main-app/task-definition.json \
            --query 'taskDefinition.taskDefinitionArn' \
            --output text)
          echo "task_def_arn=$TASK_DEF" >> $GITHUB_OUTPUT

      - name: Deploy to ECS service
        run: |
          aws ecs update-service \
            --cluster ${{ secrets.ECS_CLUSTER }} \
            --service ${{ secrets.ECS_SERVICE_MAIN }} \
            --task-definition ${{ steps.register-task.outputs.task_def_arn }} \
            --force-new-deployment

  # ============================================
  # JOB 3: Build & Push Monitor App Image
  # ============================================
  build-monitor:
    name: Build Monitor App
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.build.outputs.image_tag }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image to ECR
        id: build
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: ${{ secrets.ECR_MONITOR }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          cd monitor-app
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
          echo "image_tag=$IMAGE_TAG" >> $GITHUB_OUTPUT

  # ============================================
  # JOB 4: Deploy Monitor App to ECS
  # ============================================
  deploy-monitor:
    name: Deploy Monitor App
    needs: build-monitor
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Fill task definition template
        run: |
          cd monitor-app
          sed -i "s|ACCOUNT_ID|${{ env.AWS_ACCOUNT_ID }}|g" task-definition.json
          sed -i "s|ALB_ENDPOINT_HERE|${{ secrets.ALB_ENDPOINT }}|g" task-definition.json

      - name: Register ECS task definition
        id: register-task
        run: |
          TASK_DEF=$(aws ecs register-task-definition \
            --cli-input-json file://monitor-app/task-definition.json \
            --query 'taskDefinition.taskDefinitionArn' \
            --output text)
          echo "task_def_arn=$TASK_DEF" >> $GITHUB_OUTPUT

      - name: Deploy to ECS service
        run: |
          aws ecs update-service \
            --cluster ${{ secrets.ECS_CLUSTER }} \
            --service ${{ secrets.ECS_SERVICE_MONITOR }} \
            --task-definition ${{ steps.register-task.outputs.task_def_arn }} \
            --force-new-deployment
```

### Langkah 13.2: Tambah Secrets Tambahan
Tambahkan ke GitHub Secrets:
- `EFS_ID`: `fs-xxxxxxxxxxxx`
- `EFS_AP_ID`: `fsap-xxxxxxxxxxxx`

### Langkah 13.3: Hapus File Contoh
Hapus file `.github/workflows/example-*.yml` (jika ada).

### Langkah 13.4: Commit & Push
```bash
git add .
git commit -m "Add CI/CD pipeline"
git push origin main
```

---

## 🚀 Fase 14: Trigger Pipeline & Verify

### Langkah 14.1: Monitor Pipeline
1. GitHub repo → tab **Actions**
2. Klik workflow run terbaru
3. Harus ada 4 jobs berjalan:
   - ✅ `Build Main App`
   - ✅ `Deploy Main App` (setelah build-main selesai)
   - ✅ `Build Monitor App`
   - ✅ `Deploy Monitor App` (setelah build-monitor selesai)

### Langkah 14.2: Verifikasi di AWS
1. **ECR** → Repository `lks-main-app` → harus ada image baru dengan tag commit SHA + `latest`
2. **ECS** → Cluster → Service `lks-main-service` → tab **Tasks** → harus ada task baru dengan image terbaru
3. **CloudWatch** → Log group `/ecs/lks/main-app` → harus ada log startup

---

## 🧪 Fase 15: Testing End-to-End

### Test 1: Akses Main App via Port 80
```
http://lks-cicd-alb-xxx.us-east-1.elb.amazonaws.com/
```
Harus muncul:
```json
{
  "status": "healthy",
  "app": "lks-main-app",
  "timestamp": "2026-10-26T...",
  "environment": "production"
}
```

### Test 2: Akses Monitor App via Port 8080
```
http://lks-cicd-alb-xxx.us-east-1.elb.amazonaws.com:8080/
```
Harus muncul:
```json
{
  "status": "healthy",
  "app": "lks-monitor-app",
  "timestamp": "2026-10-26T..."
}
```

### Test 3: Test Database Connection
```
http://lks-cicd-alb-xxx.us-east-1.elb.amazonaws.com/db-test
```
Harus muncul:
```json
{ "status": "connected", "result": 2 }
```

### Test 4: Test EFS Write
```bash
curl -X POST http://lks-cicd-alb-xxx.us-east-1.elb.amazonaws.com/write-file
```
Lalu cek:
```
http://lks-cicd-alb-xxx.us-east-1.elb.amazonaws.com/list-files
```
Harus muncul file baru di list.

### Test 5: Test Health Check
```
http://lks-cicd-alb-xxx.us-east-1.elb.amazonaws.com:8080/check
```
Harus muncul status main app dengan response time.

### Test 6: Test CI/CD Pipeline
1. Edit `main-app/src/index.js` → ubah response message
2. Commit & push
3. Tunggu pipeline selesai (~3-5 menit)
4. Akses main app → harus ada perubahan!

### Test 7: Verifikasi Image Tag di ECR
1. **ECR** → `lks-main-app` → **Images**
2. Harus ada 2 tag:
   - `abc123def...` (commit SHA)
   - `latest`

### Test 8: Verifikasi Task Definition di ECS
1. **ECS** → Task definitions → `lks-main-app`
2. Harus ada revision baru dengan image tag commit SHA

---

## 🧹 Fase 16: Cleanup

### Urutan Penghapusan:

1. **GitHub Actions** → Cancel workflow yang sedang jalan (jika ada)
2. **ECS Services** (2 services) → Update desired tasks: 0 → Delete
3. **ECS Task Definitions** (2 task defs) → Deregister
4. **ECS Cluster** → Delete
5. **ALB** → Delete
6. **Target Groups** (2 TG) → Delete
7. **RDS Database** → Delete (disable snapshot)
8. **EFS Access Point** → Delete
9. **EFS File System** → Delete
10. **ECR Repositories** (2 repos) → Delete
11. **SSM Parameters** (5 parameters) → Delete
12. **Security Groups** (5 SG) → Delete
13. **NAT Gateway** → Delete
14. **Elastic IP** → Release
15. **VPC** → Delete
16. **CloudWatch Log Groups** (2 groups) → Delete
17. **IAM Inline Policy** → Delete dari LabRole
18. **GitHub Token** → Revoke (Settings → Developer settings → Tokens)

---

## 🎓 Checklist Juri (Self-Assessment)

| # | Item | Status |
|---|---|---|
| 1 | Semua resource AWS ber-prefix `lks-` | ☐ |
| 2 | Semua resource di-tag & ada deskripsi | ☐ |
| 3 | Region: `us-east-1` | ☐ |
| 4 | VPC dengan 2 public + 2 private subnet | ☐ |
| 5 | 5 Security Groups dengan least privilege | ☐ |
| 6 | RDS tidak publicly accessible | ☐ |
| 7 | EFS encrypted at rest | ☐ |
| 8 | EFS Access Point dengan POSIX UID/GID | ☐ |
| 9 | SSM Parameter Store: 5 parameter (SecureString untuk credentials) | ☐ |
| 10 | 2 ECR repositories dengan tag mutable | ☐ |
| 11 | ALB internet-facing dengan 2 listeners (80 & 8080) | ☐ |
| 12 | 2 Target Groups dengan health check benar | ☐ |
| 13 | ECS Cluster Fargate | ☐ |
| 14 | 2 Task Definitions dengan SSM secrets | ☐ |
| 15 | 2 ECS Services multi-AZ | ☐ |
| 16 | IAM role punya izin ECR + ECS + PassRole | ☐ |
| 17 | GitHub Fine-Grained Token dengan Issues R/W | ☐ |
| 18 | GitHub Secrets lengkap (AWS creds, ECR, ECS, ALB, Token) | ☐ |
| 19 | Workflow file: `lks-deployment-pipeline.yml` | ☐ |
| 20 | Workflow trigger: push ke `main` pada path spesifik | ☐ |
| 21 | 4 jobs paralel: build-main, deploy-main, build-monitor, deploy-monitor | ☐ |
| 22 | Image di-push dengan tag commit SHA + latest | ☐ |
| 23 | Task definition di-render dari JSON template | ☐ |
| 24 | ECS service di-update dengan force-new-deployment | ☐ |
| 25 | Main app bisa diakses via port 80 | ☐ |
| 26 | Monitor app bisa diakses via port 8080 | ☐ |
| 27 | Database connection berhasil | ☐ |
| 28 | EFS write/read berhasil | ☐ |
| 29 | CI/CD pipeline berjalan otomatis saat push | ☐ |
| 30 | CloudWatch logs aktif untuk kedua app | ☐ |
| 31 | Cleanup selesai | ☐ |

---

## 💡 Tips Trainer

### 🔴 Jika GitHub Actions Gagal di "Login to ECR"
**Penyebab**: AWS credentials salah atau expired.
**Solusi**:
1. Cek AWS Lab Details → pastikan AccessKey, SecretKey, SessionToken benar.
2. SessionToken di AWS Lab Learner biasanya expire dalam 4-6 jam → refresh lab jika perlu.

### 🔴 Jika Pipeline Gagal di "Register Task Definition"
**Penyebab**: JSON task definition invalid atau IAM role tidak punya izin.
**Solusi**:
1. Validasi JSON di https://jsonlint.com/
2. Pastikan IAM role punya `ecs:RegisterTaskDefinition` dan `iam:PassRole`.

### 🔴 Jika ECS Service Tidak Update Setelah Deploy
**Penyebab**: `force-new-deployment` tidak di-set atau task definition tidak berubah.
**Solusi**:
1. Pastikan command `aws ecs update-service` include `--force-new-deployment`.
2. Cek task definition revision baru sudah terdaftar.

### 🔴 Jika Image di ECR Tidak Bisa Di-Pull oleh ECS
**Penyebab**: Task execution role tidak punya izin ECR.
**Solusi**:
1. Attach `AmazonECSTaskExecutionRolePolicy` ke LabRole.
2. Pastikan image URI di task definition benar formatnya.

### 🔴 Jika Workflow Tidak Trigger
**Penyebab**: Path filter salah atau branch bukan `main`.
**Solusi**:
1. Cek workflow file → `paths` harus match dengan folder yang diubah.
2. Pastikan push ke branch `main` (bukan `master`).

### 🔴 Jika Health Check Gatus/Monitor App UNHEALTHY
**Penyebab**: TARGET_URL salah atau main app belum ready.
**Solusi**:
1. Pastikan `TARGET_URL` di task definition = ALB endpoint (port 80, tanpa trailing slash).
2. Health check grace period = 120s (beri waktu main app startup).

---

## 📊 Apa yang Anda Pelajari dari Saran 4?

| Skill | Status |
|---|---|
| Semua skill dari Saran 1, 2, 3 | ✅ |
| **ECR** (private container registry) | ✅ |
| **Docker build, tag, push** via GitHub Actions | ✅ |
| **GitHub Fine-Grained Token** | ✅ |
| **GitHub Secrets** management | ✅ |
| **GitHub Actions** multi-job parallel workflow | ✅ |
| **AWS CLI** di GitHub Actions runner | ✅ |
| **Task definition rendering** dari JSON template | ✅ |
| **ECS service update** via CLI | ✅ |
| **Image tagging strategy** (commit SHA + latest) | ✅ |
| **End-to-end CI/CD** dari code push sampai deploy | ✅ |

---

## 🚀 Progression Path

```
✅ Saran 1 (1 app Nginx, manual deploy) — 1-2 jam
    ↓
✅ Saran 2 (+ RDS MySQL + SSM) — 2-3 jam
    ↓
✅ Saran 3 (+ EFS + multi-port ALB + 2 apps) — 3-4 jam
    ↓
✅ Saran 4 (+ ECR + GitHub Actions CI/CD) — 4-5 jam ← ANDA DI SINI
    ↓
🏆 SOAL LKS DevOps CI/CD (Leantime + Gatus, 3 jam)
```

---

## 🎯 Analogi Langsung ke Soal LKS

| Yang Anda Lakukan di Saran 4 | Yang Akan Anda Lakukan di Soal LKS |
|---|---|
| `main-app` (Node.js) | **Leantime** (PHP) |
| `monitor-app` (Node.js) | **Gatus** (Go) |
| ECR `lks-main-app` + `lks-monitor-app` | ECR `lks-gatus` (Leantime pakai Docker Hub) |
| 4 jobs paralel | 3 jobs paralel |
| Fine-Grained Token (Issues R/W) | Fine-Grained Token (Issues R/W) |
| GitHub Secrets (AWS creds, ECR, ECS) | GitHub Secrets (AWS creds, ALB, Token) |
| Workflow trigger: push ke main | Workflow trigger: push ke main + path filter |
| Task definition dari JSON | Task definition dari JSON |
| `aws ecs update-service` | `aws ecs update-service` |
| Health check `/` | Health check `/auth/login` (Leantime) & `/` (Gatus) |

---

## 🏆 Anda Siap untuk Soal LKS!

Jika Anda sudah menyelesaikan **Saran 4** dengan lancar, Anda sudah **90% siap** untuk soal LKS DevOps CI/CD. Yang tinggal adalah:

1. **Leantime-specific**:
   - 3 EFS Access Points (bukan 1)
   - Environment variables Leantime (app_url, sitename, timezone)
   - Health check path `/auth/login` dengan success code `302`

2. **Gatus-specific**:
   - YAML config (bukan Node.js app)
   - GitHub Issues alerting (pakai TOKEN dari secrets)
   - Endpoint monitoring dengan interval & conditions

3. **Detail teknis**:
   - Aurora MySQL (bukan RDS MySQL biasa)
   - Instance class `t4g.medium`
   - Auto scaling dengan cooldown 300s

---

## 🎯 Next Step

Apakah Anda ingin saya buatkan:
1. **Panduan lengkap soal LKS DevOps CI/CD sesungguhnya** (Leantime + Gatus + GitHub Actions)?
2. **Template troubleshooting** untuk error yang sering muncul di GitHub Actions?
3. **Script cleanup otomatis** untuk menghapus semua resource sekaligus?
4. **Cheat sheet** perbedaan antara Saran 4 dan soal LKS sesungguhnya?

Silakan pilih, dan saya akan buatkan! 🎯

**Selamat! Anda telah menyelesaikan seluruh progression path latihan DevOps CI/CD.** 🎉