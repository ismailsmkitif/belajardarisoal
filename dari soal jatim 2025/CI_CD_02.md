# 📘 Panduan Lengkap Pengerjaan Saran 2 (DevOps CI/CD)
## "Container + Database" — WordPress di ECS Fargate + ALB + RDS MySQL + SSM

> **Estimasi waktu**: 2-3 jam | **Region**: `us-east-1` | **Console-based**

---

## 🎯 Tujuan Latihan

Membangun aplikasi container yang **terintegrasi dengan database** dan **mengelola credential secara aman**:
- ✅ 1 aplikasi web dinamis (WordPress)
- ✅ ECS Fargate (container orchestration)
- ✅ RDS MySQL (database relasional)
- ✅ SSM Parameter Store (credential management)
- ✅ Application Load Balancer (internet-facing)
- ✅ Security Groups yang ketat (multi-tier)

Ini adalah **step 1** sebelum Anda mengerjakan soal LKS DevOps CI/CD yang sesungguhnya (Leantime + Gatus + EFS + GitHub Actions).

---

## 📋 Daftar Isi
1. [Fase 0: Persiapan](#fase-0-persiapan)
2. [Fase 1: Networking (VPC)](#fase-1-networking-vpc)
3. [Fase 2: Security Groups (Multi-Tier)](#fase-2-security-groups)
4. [Fase 3: RDS MySQL Database](#fase-3-rds-mysql-database)
5. [Fase 4: SSM Parameter Store](#fase-4-ssm-parameter-store)
6. [Fase 5: Application Load Balancer + Target Group](#fase-5-alb--target-group)
7. [Fase 6: ECS Cluster](#fase-6-ecs-cluster)
8. [Fase 7: Task Definition dengan SSM Integration](#fase-7-task-definition)
9. [Fase 8: ECS Service](#fase-8-ecs-service)
10. [Fase 9: Testing End-to-End](#fase-9-testing)
11. [Fase 10: Cleanup](#fase-10-cleanup)

---

## 🎯 Fase 0: Persiapan

### Langkah 0.1: Login & Set Region
1. Login ke **AWS Lab Learner** → klik **Start Lab** → **Open AWS Console**.
2. Di pojok kanan atas, pastikan region = **US East (N. Virginia) `us-east-1`**.
3. Buka **Notepad** lokal untuk mencatat:
   - Account ID (12 digit)
   - VPC ID (akan diisi di Fase 1)
   - RDS Endpoint (akan diisi di Fase 3)
   - ALB DNS Name (akan diisi di Fase 5)

### Langkah 0.2: Tagging Strategy
Setiap resource yang Anda buat, **WAJIB** isi:
- **Name tag**: `lks-<nama-resource>`
- **Tags**: `Project=LKS-DevOps-Training`, `Environment=Lab`
- **Description**: tujuan resource

---

## 🌐 Fase 1: Networking (VPC)

### Langkah 1.1: Buat VPC dengan Wizard
1. Buka service **VPC** → **Your VPCs** → **Create VPC**.
2. Pilih **VPC and more**.
3. Konfigurasi:
   - Name tag: `lks-wordpress-vpc`
   - IPv4 CIDR: `10.60.0.0/16`
   - Number of Availability Zones: **2**
   - Number of public subnets: **2**
   - Number of private subnets: **2**
   - ✅ Centang **NAT gateways (1)** — untuk pull image WordPress dari Docker Hub
   - ❌ **Tidak perlu** VPC endpoints (kita pakai NAT untuk simplicity)
4. Klik **Create VPC**. Tunggu ~2 menit.

### Langkah 1.2: Catat Resource IDs
Buka **VPC** → catat:
- **VPC ID**: `vpc-xxxxxxxx`
- **Public Subnet 1**: `subnet-pub1-xxxx` (AZ a)
- **Public Subnet 2**: `subnet-pub2-xxxx` (AZ b)
- **Private Subnet 1**: `subnet-priv1-xxxx` (AZ a)
- **Private Subnet 2**: `subnet-priv2-xxxx` (AZ b)

### Langkah 1.3: Rename Resources
1. **Internet Gateway** → rename ke `lks-wordpress-igw`
2. **NAT Gateway** → rename ke `lks-wordpress-natgw`
3. **Public Route Table** → rename ke `lks-wordpress-public-rt`
4. **Private Route Table** → rename ke `lks-wordpress-private-rt`

---

## 🔒 Fase 2: Security Groups (Multi-Tier)

### Langkah 2.1: SG untuk ALB (Public)
1. **VPC** → **Security Groups** → **Create Security Group**.
2. Name: `lks-sg-alb`
3. Description: `Security group for ALB - allows HTTP from internet`
4. VPC: `lks-wordpress-vpc`
5. **Inbound rules**:
   | Type | Protocol | Port | Source | Description |
   |---|---|---|---|---|
   | HTTP | TCP | 80 | `0.0.0.0/0` | Allow HTTP from internet |
6. **Outbound rules**: All traffic (default)
7. **Create**.

### Langkah 2.2: SG untuk ECS Task (WordPress App)
1. **Create Security Group** lagi.
2. Name: `lks-sg-app`
3. Description: `Security group for WordPress ECS tasks - only from ALB`
4. VPC: `lks-wordpress-vpc`
5. **Inbound rules**:
   | Type | Protocol | Port | Source | Description |
   |---|---|---|---|---|
   | Custom TCP | TCP | 80 | `lks-sg-alb` | Allow traffic from ALB only |
6. **Outbound rules**: All traffic (default)
7. **Create**.

### Langkah 2.3: SG untuk RDS MySQL (Database)
1. **Create Security Group** lagi.
2. Name: `lks-sg-rds`
3. Description: `Security group for RDS MySQL - only from WordPress app`
4. VPC: `lks-wordpress-vpc`
5. **Inbound rules**:
   | Type | Protocol | Port | Source | Description |
   |---|---|---|---|---|
   | MySQL/Aurora | TCP | 3306 | `lks-sg-app` | Allow MySQL from WordPress only |
6. **Outbound rules**: All traffic (default)
7. **Create**.

> ⚠️ **PENTING**: Perhatikan prinsip **defense in depth**:
> - ALB hanya izinkan dari internet (port 80)
> - App hanya izinkan dari ALB (port 80)
> - RDS hanya izinkan dari App (port 3306)
> - **Tidak ada** resource yang bisa diakses langsung dari internet kecuali ALB

---

## 🗄️ Fase 3: RDS MySQL Database

### Langkah 3.1: Buat RDS MySQL Instance
1. Buka **RDS** → **Databases** → **Create database**.
2. Pilih **Standard create**.
3. Engine options:
   - Engine type: **MySQL**
   - Version: **8.0.x** (atau terbaru)
4. Templates: **Free tier** (untuk latihan) atau **Production** (untuk Multi-AZ)
5. **Settings**:
   - DB instance identifier: `lks-wordpress-db`
   - Master username: `wpadmin`
   - Master password: `WordPress2026!Secure` (catat ini!)
   - Confirm password
6. **Instance configuration**:
   - DB instance class: `db.t3.micro` (free tier) atau `db.t3.small`
   - Storage: 20 GB (default)
   - ✅ Enable storage autoscaling (opsional)
7. **Connectivity**:
   - VPC: `lks-wordpress-vpc`
   - Public access: **No** ❌ (WAJIB)
   - VPC security group: `lks-sg-rds` (hapus default)
   - Availability Zone: No preference
8. **Additional configuration**:
   - Initial database name: `wordpress_db`
9. **Create database**. Tunggu ~5-10 menit.

### Langkah 3.2: Catat Endpoint RDS
Setelah status `Available`:
- Klik DB → tab **Connectivity & security** → catat **Endpoint**:
  ```
  lks-wordpress-db.xxxxx.us-east-1.rds.amazonaws.com
  ```

---

## 🔑 Fase 4: SSM Parameter Store

### Langkah 4.1: Buat 4 Parameter untuk Database Credentials
1. Buka **Systems Manager** → **Application Management** → **Parameter Store**.
2. **Create parameter** (ulangi 4x):

| Name | Type | Value | Description |
|---|---|---|---|
| `/lks/wordpress/db/host` | String | `lks-wordpress-db.xxxxx.us-east-1.rds.amazonaws.com` | Database endpoint |
| `/lks/wordpress/db/port` | String | `3306` | Database port |
| `/lks/wordpress/db/name` | String | `wordpress_db` | Database name |
| `/lks/wordpress/db/username` | **SecureString** | `wpadmin` | Database username |
| `/lks/wordpress/db/password` | **SecureString** | `WordPress2026!Secure` | Database password |

> ⚠️ Untuk `username` dan `password`, pilih **KMS key source**: `aws/ssm` (default alias).

### Langkah 4.2: Verifikasi Parameter
1. Klik setiap parameter → pastikan value sudah benar.
2. Untuk SecureString, klik **Show** → masukkan IAM role `LabRole` untuk decrypt (opsional, untuk testing).

---

## 🌐 Fase 5: ALB + Target Group

### Langkah 5.1: Buat Target Group
1. Buka **EC2** → **Target Groups** → **Create target group**.
2. Target type: **IP addresses** (WAJIB untuk Fargate)
3. Name: `lks-wordpress-tg`
4. Protocol: **HTTP**
5. Port: **80**
6. VPC: `lks-wordpress-vpc`
7. **Health check settings**:
   - Protocol: HTTP
   - Path: `/wp-admin/install.php` (WordPress installer page)
   - Success code: `200,302` (WordPress redirect saat belum di-setup)
   - Healthy threshold: **3**
   - Unhealthy threshold: **2**
   - Timeout: **5 seconds**
   - Interval: **30 seconds**
8. **Create**.

### Langkah 5.2: Buat Application Load Balancer
1. **EC2** → **Load Balancers** → **Create Load Balancer**.
2. Pilih **Application Load Balancer** → **Create**.
3. **Basic configuration**:
   - Name: `lks-wordpress-alb`
   - Scheme: **Internet-facing**
   - IP address type: **IPv4**
4. **Network mapping**:
   - VPC: `lks-wordpress-vpc`
   - Mappings: ✅ centang **kedua public subnet** (AZ a & b)
5. **Security groups**:
   - Hapus default, pilih: `lks-sg-alb`
6. **Listeners**:
   - Protocol: **HTTP**, Port: **80**
   - Default action: **Forward to** → `lks-wordpress-tg`
7. **Create load balancer**. Tunggu ~2 menit.

### Langkah 5.3: Catat ALB DNS Name
Setelah status **Active**:
- Klik ALB → tab **Description** → catat **DNS name**:
  ```
  lks-wordpress-alb-xxxxxxxx.us-east-1.elb.amazonaws.com
  ```

---

## 🎯 Fase 6: ECS Cluster

### Langkah 6.1: Buat ECS Cluster
1. Buka **ECS** → **Clusters** → **Create cluster**.
2. Cluster name: `lks-wordpress-cluster`
3. Infrastructure: **AWS Fargate (Serverless)**
4. **Create**.

---

## 📝 Fase 7: Task Definition dengan SSM Integration

### Langkah 7.1: Buat Task Definition
1. **ECS** → **Task definitions** → **Create new task definition**.
2. **Task definition configuration**:
   - Name: `lks-wordpress-task`
   - Launch compatibility: **Fargate** ✅
3. **Infrastructure requirements**:
   - OS: **Linux/X86_64**
   - Task CPU: **512** (.5 vCPU) — WordPress butuh lebih banyak resource
   - Task memory: **1024 MB** (1 GB)
   - Task IAM role: `LabRole`
   - Task execution role: `LabRole`
4. **Container - 1**:
   - Name: `lks-wordpress-container`
   - Image URI: `wordpress:latest` (dari Docker Hub)
   - Container port: **80** (TCP)
   - Port name: `http`

5. **Environment variables** (dari SSM Parameter Store):
   - Klik **Environment** → **Add environment variable**:
     - Key: `WORDPRESS_DB_HOST`
     - Value: (kosongkan, akan diisi dari SSM)
     - ✅ **ValueFrom**: `/lks/wordpress/db/host`
   - Ulangi untuk:
     - `WORDPRESS_DB_PORT` → `/lks/wordpress/db/port`
     - `WORDPRESS_DB_NAME` → `/lks/wordpress/db/name`
     - `WORDPRESS_DB_USER` → `/lks/wordpress/db/username`
     - `WORDPRESS_DB_PASSWORD` → `/lks/wordpress/db/password`

6. **Logging**:
   - Log driver: **awslogs**
   - Log group: `/ecs/lks/wordpress-app` (auto-create)
   - Region: `us-east-1`
   - Stream prefix: `wordpress`

7. **Monitoring**:
   - ✅ Enable Container Insights (opsional)

8. **Create**.

### Langkah 7.2: Verifikasi IAM Role untuk SSM Access
1. Buka **IAM** → **Roles** → `LabRole`.
2. Pastikan ada policy yang mengizinkan:
   - `ssm:GetParameter`
   - `ssm:GetParameters`
   - `kms:Decrypt` (untuk SecureString)
3. Jika belum ada → **Add permissions** → **Create inline policy** → paste JSON:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters"
      ],
      "Resource": "arn:aws:ssm:us-east-1:*:parameter/lks/wordpress/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt"
      ],
      "Resource": "arn:aws:kms:us-east-1:*:key/*"
    }
  ]
}
```

4. **Next** → Name: `lks-ssm-access-policy` → **Create policy**.

---

## 🚀 Fase 8: ECS Service

### Langkah 8.1: Buat ECS Service
1. **ECS** → **Clusters** → klik `lks-wordpress-cluster`.
2. Tab **Services** → **Create**.
3. **Environment**:
   - Cluster: `lks-wordpress-cluster`
   - Capacity provider: **Fargate**
   - Platform version: **LATEST**
4. **Deployment configuration**:
   - Application type: **Task**
   - Task definition: `lks-wordpress-task` (pilih revision terbaru)
   - Service name: `lks-wordpress-service`
   - Desired tasks: **2** (untuk high availability)
5. **Deployment options**:
   - Deployment type: **Rolling update**
   - Minimum healthy percentage: **100**
   - Maximum percentage: **200**
6. **Networking**:
   - ✅ Use existing VPC: `lks-wordpress-vpc`
   - Subnets: pilih **kedua private subnet**
   - Security group: `lks-sg-app`
   - Public IP: ❌ **Disable**
7. **Load balancing**:
   - ✅ **Application Load Balancer**
   - Select existing: `lks-wordpress-alb`
   - Container to load balance: `lks-wordpress-container:80`
   - Listener: **80:HTTP** → forward to `lks-wordpress-tg`
8. **Health check grace period**: **120 seconds** (WordPress butuh waktu startup)
9. **Auto scaling**: ❌ Disable (untuk latihan sederhana)
10. **Create**.

### Langkah 8.2: Tunggu Service Stabil
- Tunggu ~5-7 menit sampai status service = **ACTIVE**.
- Tab **Tasks** → harus ada 2 tasks dengan status **RUNNING**.
- Jika task status **STOPPED** → klik task → cek **Events** & **Logs** untuk troubleshooting.

---

## 🧪 Fase 9: Testing End-to-End

### Test 1: Akses WordPress Installer via ALB
1. Buka browser → akses:
   ```
   http://lks-wordpress-alb-xxxxxxxx.us-east-1.elb.amazonaws.com
   ```
2. Harus muncul **WordPress installation page** (pilih bahasa, setup admin, dll).

### Test 2: Cek Target Group Health
1. **EC2** → **Target Groups** → klik `lks-wordpress-tg`.
2. Tab **Targets** → harus ada 2 targets dengan status **Healthy**.

### Test 3: Cek ECS Tasks
1. **ECS** → **Clusters** → `lks-wordpress-cluster` → **Tasks**.
2. Harus ada 2 tasks **RUNNING** di subnet berbeda (multi-AZ).

### Test 4: Cek CloudWatch Logs
1. **CloudWatch** → **Log groups** → `/ecs/lks/wordpress-app`.
2. Klik log stream → harus ada log WordPress startup.

### Test 5: Verifikasi Database Connection
1. Di WordPress installer → isi:
   - Database Name: `wordpress_db`
   - Username: `wpadmin`
   - Password: `WordPress2026!Secure`
   - Database Host: `lks-wordpress-db.xxxxx.us-east-1.rds.amazonaws.com`
   - Table Prefix: `wp_`
2. Klik **Submit** → **Run the installation**.
3. Jika sukses → WordPress bisa connect ke RDS via private subnet!

### Test 6: Test High Availability (Opsional)
1. **ECS** → Tasks → **Stop** 1 task.
2. Tunggu ~1 menit → ECS akan otomatis spawn task baru.
3. Akses ALB lagi → WordPress tetap bisa diakses (downtime minimal).

---

## 🧹 Fase 10: Cleanup

> ⚠️ **PENTING**: Hapus semua resource untuk menghindari tagihan!

### Urutan Penghapusan (WAJIB berurutan):

1. **ECS Service** → ECS → Clusters → `lks-wordpress-cluster` → Services → pilih `lks-wordpress-service` → **Update** → Desired tasks: 0 → **Save** → tunggu task berhenti → **Delete service**.

2. **ECS Task Definition** → Task definitions → `lks-wordpress-task` → **Deregister** → **Delete**.

3. **ECS Cluster** → Clusters → `lks-wordpress-cluster` → **Delete cluster**.

4. **ALB** → EC2 → Load Balancers → `lks-wordpress-alb` → **Actions** → **Delete**.

5. **Target Group** → Target Groups → `lks-wordpress-tg` → **Actions** → **Delete**.

6. **RDS Database** → RDS → Databases → `lks-wordpress-db` → **Actions** → **Delete** → ❌ disable snapshot → **Delete**.

7. **SSM Parameters** → Systems Manager → Parameter Store → pilih 5 parameter `/lks/wordpress/*` → **Delete**.

8. **Security Groups** → VPC → Security Groups → delete `lks-sg-alb`, `lks-sg-app`, `lks-sg-rds`.

9. **NAT Gateway** → VPC → NAT Gateways → delete `lks-wordpress-natgw`.

10. **Elastic IP** → EC2 → Elastic IPs → **Release** (agar tidak ditagih!).

11. **VPC** → VPC → `lks-wordpress-vpc` → **Actions** → **Delete VPC**.

12. **CloudWatch Log Group** → CloudWatch → Log groups → `/ecs/lks/wordpress-app` → **Delete**.

---

## 🎓 Checklist Juri (Self-Assessment)

| # | Item | Status |
|---|---|---|
| 1 | Semua resource ber-prefix `lks-` | ☐ |
| 2 | Semua resource di-tag & ada deskripsi | ☐ |
| 3 | Region: `us-east-1` | ☐ |
| 4 | VPC: `10.60.0.0/16` dengan 2 public + 2 private subnet | ☐ |
| 5 | SG ALB hanya izinkan HTTP 80 dari `0.0.0.0/0` | ☐ |
| 6 | SG App hanya izinkan port 80 dari SG ALB | ☐ |
| 7 | SG RDS hanya izinkan port 3306 dari SG App | ☐ |
| 8 | RDS MySQL tidak publicly accessible | ☐ |
| 9 | RDS di private subnet | ☐ |
| 10 | SSM Parameter Store: 5 parameter (host, port, name, username, password) | ☐ |
| 11 | SSM username & password = SecureString | ☐ |
| 12 | Task Definition pakai SSM ValueFrom (bukan hardcode) | ☐ |
| 13 | ALB internet-facing dengan listener HTTP 80 | ☐ |
| 14 | Target Group: IP target, health check `/wp-admin/install.php` | ☐ |
| 15 | ECS Cluster Fargate (serverless) | ☐ |
| 16 | Task Definition: Fargate, 512 CPU, 1024 MB, awslogs | ☐ |
| 17 | ECS Service: 2 tasks, multi-AZ, private subnet | ☐ |
| 18 | Task status RUNNING di 2 subnet berbeda | ☐ |
| 19 | Target group health: 2 targets HEALTHY | ☐ |
| 20 | WordPress bisa diakses via ALB DNS | ☐ |
| 21 | WordPress bisa connect ke RDS MySQL | ☐ |
| 22 | CloudWatch logs aktif | ☐ |
| 23 | Cleanup selesai (tidak ada resource tersisa) | ☐ |

---

## 💡 Tips Trainer

### 🔴 Jika Task STOPPED dengan Error "CannotPullContainerError"
**Penyebab**: IAM role tidak punya izin pull dari Docker Hub atau ECR.
**Solusi**: 
1. Pastikan task execution role = `LabRole`.
2. Attach policy `AmazonECSTaskExecutionRolePolicy` ke LabRole.
3. Restart task.

### 🔴 Jika Task STOPPED dengan Error "ResourceInitializationError"
**Penyebab**: Task tidak bisa ambil parameter dari SSM.
**Solusi**:
1. Cek IAM role punya izin `ssm:GetParameter` dan `kms:Decrypt`.
2. Pastikan path parameter benar: `/lks/wordpress/db/*`.
3. Cek CloudWatch logs untuk detail error.

### 🔴 Jika Target Group UNHEALTHY
**Penyebab**: WordPress belum startup atau health check path salah.
**Solusi**:
1. Tunggu ~2-3 menit (WordPress butuh waktu startup).
2. Health check path = `/wp-admin/install.php` (bukan `/`).
3. Success code = `200,302` (WordPress redirect saat belum di-setup).

### 🔴 Jika WordPress Tidak Bisa Connect ke Database
**Penyebab**: Security Group RDS tidak izinkan dari SG App.
**Solusi**:
1. Cek SG `lks-sg-rds` → Inbound rules → Source harus `lks-sg-app`.
2. Pastikan port = 3306, protocol = TCP.
3. Cek environment variable di task definition sudah benar.

### 🔴 Jika NAT Gateway Tidak Bekerja
**Penyebab**: Route table private subnet tidak point ke NAT.
**Solusi**:
1. VPC → Route Tables → `lks-wordpress-private-rt`.
2. Routes → harus ada: `0.0.0.0/0` → target `lks-wordpress-natgw`.

---

## 📊 Apa yang Anda Pelajari dari Saran 2?

| Skill | Status |
|---|---|
| VPC dengan public + private subnet | ✅ |
| Security Groups multi-tier (ALB → App → RDS) | ✅ |
| RDS MySQL di private subnet | ✅ |
| SSM Parameter Store (String & SecureString) | ✅ |
| ECS Task Definition dengan SSM ValueFrom | ✅ |
| IAM policy untuk SSM access | ✅ |
| ALB + Target Group (health check WordPress) | ✅ |
| ECS Fargate service dengan multi-AZ | ✅ |
| WordPress installation via ALB | ✅ |
| Database connection dari private subnet | ✅ |
| Cleanup yang benar (urutan penting!) | ✅ |

---

## 🚀 Progression Path

```
✅ Saran 1 (1 app Nginx, manual deploy) — 1-2 jam
    ↓
✅ Saran 2 (+ RDS MySQL + SSM Parameter Store) — 2-3 jam ← ANDA DI SINI
    ↓
🟠 Saran 3 (+ EFS + multi-port ALB + 2 apps) — 3-4 jam
    ↓
🔴 Saran 4 (+ ECR + GitHub Actions CI/CD) — 4-5 jam
    ↓
🏆 SOAL LKS DevOps CI/CD (Leantime + Gatus, 3 jam)
```

---

## 🎯 Next Step

Setelah Saran 2 berhasil, Anda siap lanjut ke **Saran 3**: menambahkan **EFS (Elastic File System)** + **multi-port ALB** untuk deploy **2 aplikasi** (WordPress + phpMyAdmin) dengan shared storage.

Apakah Anda ingin saya buatkan:
1. **Panduan lengkap Saran 3** (WordPress + phpMyAdmin + EFS + multi-port ALB)?
2. **Template troubleshooting** untuk error yang sering muncul di ECS + RDS?
3. **Script SQL** untuk setup database WordPress secara manual?

Silakan pilih, dan saya akan buatkan! 🎯