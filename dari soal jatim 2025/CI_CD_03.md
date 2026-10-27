# 📘 Panduan Lengkap Pengerjaan Saran 3 (DevOps CI/CD)
## "Multi-App with Shared Storage" — WordPress + phpMyAdmin di ECS Fargate + ALB Multi-Port + RDS + EFS

> **Estimasi waktu**: 3-4 jam | **Region**: `us-east-1` | **Console-based**

---

## 🎯 Tujuan Latihan

Membangun **2 aplikasi dalam 1 cluster ECS** dengan **shared storage (EFS)** dan **multi-port ALB** — persiapan matang untuk soal LKS sesungguhnya (Leantime + Gatus):
- ✅ 2 aplikasi container: **WordPress** (port 80) + **phpMyAdmin** (port 8080)
- ✅ **EFS** dengan Access Points untuk shared storage
- ✅ **RDS MySQL** sebagai database bersama
- ✅ **ALB multi-port** (HTTP 80 → WordPress, HTTP 8080 → phpMyAdmin)
- ✅ **SSM Parameter Store** untuk credential management
- ✅ **2 Task Definitions** + **2 ECS Services** dalam 1 cluster

Ini adalah **step 2** dan **jembatan terpenting** menuju soal LKS DevOps CI/CD.

---

## 📋 Daftar Isi
1. [Fase 0: Persiapan](#fase-0-persiapan)
2. [Fase 1: Networking (VPC)](#fase-1-networking-vpc)
3. [Fase 2: Security Groups (5-Tier)](#fase-2-security-groups)
4. [Fase 3: RDS MySQL Database](#fase-3-rds-mysql-database)
5. [Fase 4: EFS + Access Points](#fase-4-efs--access-points)
6. [Fase 5: SSM Parameter Store](#fase-5-ssm-parameter-store)
7. [Fase 6: Target Groups (2 TG)](#fase-6-target-groups)
8. [Fase 7: ALB Multi-Port](#fase-7-alb-multi-port)
9. [Fase 8: ECS Cluster](#fase-8-ecs-cluster)
10. [Fase 9: Task Definitions (2 Apps)](#fase-9-task-definitions)
11. [Fase 10: ECS Services (2 Services)](#fase-10-ecs-services)
12. [Fase 11: Testing End-to-End](#fase-11-testing)
13. [Fase 12: Cleanup](#fase-12-cleanup)

---

## 🎯 Fase 0: Persiapan

### Langkah 0.1: Login & Set Region
1. Login ke **AWS Lab Learner** → klik **Start Lab** → **Open AWS Console**.
2. Di pojok kanan atas, pastikan region = **US East (N. Virginia) `us-east-1`**.
3. Buka **Notepad** lokal untuk mencatat:
   - Account ID (12 digit)
   - VPC ID, Subnet IDs
   - RDS Endpoint
   - EFS ID & Access Point IDs
   - ALB DNS Name

### Langkah 0.2: Tagging Strategy
Setiap resource yang Anda buat, **WAJIB** isi:
- **Name tag**: `lks-<nama-resource>`
- **Tags**: `Project=LKS-DevOps-Training`, `Environment=Lab`
- **Description**: tujuan resource

### Langkah 0.3: Pahami Analogi dengan Soal LKS

| Saran 3 (Latihan) | Soal LKS (Sesungguhnya) |
|---|---|
| WordPress (port 80) | **Leantime** (port 80) |
| phpMyAdmin (port 8080) | **Gatus** (port 8080) |
| 1 RDS MySQL | **RDS Aurora MySQL** |
| 1 EFS + 1 Access Point | **1 EFS + 3 Access Points** |
| ALB 2 listeners | **ALB 2 listeners** |
| Manual deploy | **GitHub Actions CI/CD** |

---

## 🌐 Fase 1: Networking (VPC)

### Langkah 1.1: Buat VPC dengan Wizard
1. Buka service **VPC** → **Your VPCs** → **Create VPC**.
2. Pilih **VPC and more**.
3. Konfigurasi:
   - Name tag: `lks-multiapp-vpc`
   - IPv4 CIDR: `10.70.0.0/16`
   - Number of Availability Zones: **2**
   - Number of public subnets: **2**
   - Number of private subnets: **2**
   - ✅ Centang **NAT gateways (1)**
   - ❌ Tidak perlu VPC endpoints
4. Klik **Create VPC**. Tunggu ~2 menit.

### Langkah 1.2: Catat Resource IDs
- **VPC ID**: `vpc-xxxxxxxx`
- **Public Subnet 1 & 2**: untuk ALB
- **Private Subnet 1 & 2**: untuk ECS, RDS, EFS

### Langkah 1.3: Rename Resources
- Internet Gateway → `lks-multiapp-igw`
- NAT Gateway → `lks-multiapp-natgw`
- Public Route Table → `lks-multiapp-public-rt`
- Private Route Table → `lks-multiapp-private-rt`

---

## 🔒 Fase 2: Security Groups (5-Tier)

### Langkah 2.1: SG untuk ALB (Public)
1. **VPC** → **Security Groups** → **Create Security Group**.
2. Name: `lks-sg-alb`
3. Description: `ALB - allows HTTP 80 & 8080 from internet`
4. VPC: `lks-multiapp-vpc`
5. **Inbound rules**:
   | Type | Protocol | Port | Source | Description |
   |---|---|---|---|---|
   | HTTP | TCP | 80 | `0.0.0.0/0` | WordPress via ALB |
   | Custom TCP | TCP | 8080 | `0.0.0.0/0` | phpMyAdmin via ALB |
6. **Create**.

### Langkah 2.2: SG untuk WordPress App
1. Name: `lks-sg-wordpress`
2. Description: `WordPress ECS tasks - only from ALB`
3. **Inbound rules**:
   | Type | Protocol | Port | Source |
   |---|---|---|---|
   | Custom TCP | TCP | 80 | `lks-sg-alb` |
4. **Outbound rules**: All traffic (untuk akses internet update plugin, dll)
5. **Create**.

### Langkah 2.3: SG untuk phpMyAdmin App
1. Name: `lks-sg-phpmyadmin`
2. Description: `phpMyAdmin ECS tasks - only from ALB`
3. **Inbound rules**:
   | Type | Protocol | Port | Source |
   |---|---|---|---|
   | Custom TCP | TCP | 8080 | `lks-sg-alb` |
4. **Outbound rules**: All traffic
5. **Create**.

### Langkah 2.4: SG untuk RDS MySQL
1. Name: `lks-sg-rds`
2. Description: `RDS MySQL - only from WordPress & phpMyAdmin`
3. **Inbound rules**:
   | Type | Protocol | Port | Source |
   |---|---|---|---|
   | MySQL/Aurora | TCP | 3306 | `lks-sg-wordpress` |
   | MySQL/Aurora | TCP | 3306 | `lks-sg-phpmyadmin` |
4. **Create**.

### Langkah 2.5: SG untuk EFS
1. Name: `lks-sg-efs`
2. Description: `EFS - NFS only from WordPress`
3. **Inbound rules**:
   | Type | Protocol | Port | Source |
   |---|---|---|---|
   | NFS | TCP | 2049 | `lks-sg-wordpress` |
4. **Create**.

> ⚠️ **PENTING**: 5 Security Groups ini mencerminkan **prinsip least privilege** yang sama dengan soal LKS. Setiap layer hanya bisa diakses oleh layer di atasnya.

---

## 🗄️ Fase 3: RDS MySQL Database

### Langkah 3.1: Buat RDS MySQL Instance
1. **RDS** → **Databases** → **Create database**.
2. **Standard create** → MySQL 8.0.x
3. Templates: **Free tier**
4. **Settings**:
   - DB instance identifier: `lks-multiapp-db`
   - Master username: `appadmin`
   - Master password: `MultiApp2026!Secure`
5. **Instance configuration**:
   - DB instance class: `db.t3.micro`
6. **Connectivity**:
   - VPC: `lks-multiapp-vpc`
   - Public access: **No** ❌
   - VPC security group: `lks-sg-rds` (hapus default)
7. **Additional configuration**:
   - Initial database name: `multiapp_db`
8. **Create database**. Tunggu ~5-10 menit.

### Langkah 3.2: Catat Endpoint RDS
```
lks-multiapp-db.xxxxx.us-east-1.rds.amazonaws.com
```

---

## 📁 Fase 4: EFS + Access Points

> 💡 **Ini adalah bagian tersulit dan terpenting** — langsung melatih skill yang dibutuhkan untuk soal LKS (Leantime dengan 3 access points).

### Langkah 4.1: Buat EFS File System
1. Buka **EFS** → **File systems** → **Create file system**.
2. Name: `lks-multiapp-efs`
3. VPC: `lks-multiapp-vpc`
4. **Storage**:
   - Performance mode: **General Purpose**
   - Throughput mode: **Elastic**
   - ✅ Enable automatic encryption at rest
5. **Network** (untuk setiap AZ):
   - Pilih **kedua private subnet**
   - Security group: `lks-sg-efs` (hapus default)
6. **File system policy**: **Enforce** (opsional, untuk compliance)
7. **Create**.

### Langkah 4.2: Catat File System ID
- **EFS ID**: `fs-xxxxxxxxxxxx`

### Langkah 4.3: Buat Access Point untuk WordPress
1. Klik EFS → tab **Access Points** → **Create access point**.
2. Name: `lks-ap-wordpress-content`
3. **Root directory path**: `/wordpress/wp-content`
4. **POSIX User**:
   - User ID: `33` (UID www-data di Debian/Ubuntu, sama dengan image WordPress)
   - Group ID: `33`
5. **Root directory creation permissions**:
   - ✅ Create new directory with the following settings
   - Owner user ID: `33`
   - Owner group ID: `33`
   - POSIX permissions: `755` (rwxr-xr-x)
6. **Create access point**.

### Langkah 4.4: Catat Access Point ID
- **Access Point ID**: `fsap-xxxxxxxxxxxx`

> 💡 **Catatan Trainer**: Di soal LKS, Anda akan buat **3 access points** untuk Leantime:
> - `/public_userfiles` → `/var/www/html/public/userfiles`
> - `/userfiles` → `/var/www/html/userfiles`
> - `/plugins` → `/var/www/html/app/Plugins`
> 
> Konsepnya sama persis!

---

## 🔑 Fase 5: SSM Parameter Store

### Langkah 5.1: Buat 5 Parameter
1. **Systems Manager** → **Parameter Store** → **Create parameter** (ulangi 5x):

| Name | Type | Value |
|---|---|---|
| `/lks/multiapp/db/host` | String | `lks-multiapp-db.xxxxx.us-east-1.rds.amazonaws.com` |
| `/lks/multiapp/db/port` | String | `3306` |
| `/lks/multiapp/db/name` | String | `multiapp_db` |
| `/lks/multiapp/db/username` | **SecureString** | `appadmin` |
| `/lks/multiapp/db/password` | **SecureString** | `MultiApp2026!Secure` |

---

## 🎯 Fase 6: Target Groups (2 TG)

### Langkah 6.1: Target Group untuk WordPress
1. **EC2** → **Target Groups** → **Create target group**.
2. Target type: **IP addresses**
3. Name: `lks-wordpress-tg`
4. Protocol: **HTTP**, Port: **80**
5. VPC: `lks-multiapp-vpc`
6. **Health check**:
   - Path: `/wp-admin/install.php`
   - Success code: `200,302`
   - Healthy threshold: **3**
   - Unhealthy threshold: **2**
   - Timeout: **5 seconds**, Interval: **30 seconds**
7. **Create**.

### Langkah 6.2: Target Group untuk phpMyAdmin
1. **Create target group** lagi.
2. Target type: **IP addresses**
3. Name: `lks-phpmyadmin-tg`
4. Protocol: **HTTP**, Port: **8080**
5. VPC: `lks-multiapp-vpc`
6. **Health check**:
   - Path: `/`
   - Success code: `200`
   - Healthy threshold: **3**
   - Unhealthy threshold: **2**
   - Timeout: **5 seconds**, Interval: **30 seconds**
7. **Create**.

---

## 🌐 Fase 7: ALB Multi-Port

### Langkah 7.1: Buat Application Load Balancer
1. **EC2** → **Load Balancers** → **Create Load Balancer** → **Application Load Balancer**.
2. **Basic configuration**:
   - Name: `lks-multiapp-alb`
   - Scheme: **Internet-facing**
   - IP address type: **IPv4**
3. **Network mapping**:
   - VPC: `lks-multiapp-vpc`
   - Mappings: ✅ **kedua public subnet**
4. **Security groups**: `lks-sg-alb`
5. **Listeners** (jangan deploy dulu, tambahkan 2 listeners):
   - **Listener 1**: HTTP:80 → forward ke `lks-wordpress-tg`
   - Klik **Add listener** → **HTTP:8080** → forward ke `lks-phpmyadmin-tg`
6. **Create load balancer**.

### Langkah 7.2: Catat ALB DNS Name
```
lks-multiapp-alb-xxxxxxxx.us-east-1.elb.amazonaws.com
```

### Langkah 7.3: Verifikasi Listeners
1. Klik ALB → tab **Listeners**.
2. Harus ada 2 listeners:
   - `HTTP:80` → forward ke `lks-wordpress-tg`
   - `HTTP:8080` → forward ke `lks-phpmyadmin-tg`

> 💡 **Ini persis seperti soal LKS**: Leantime di port 80, Gatus di port 8080.

---

## 🎯 Fase 8: ECS Cluster

### Langkah 8.1: Buat ECS Cluster
1. **ECS** → **Clusters** → **Create cluster**.
2. Cluster name: `lks-multiapp-cluster`
3. Infrastructure: **AWS Fargate (Serverless)**
4. **Create**.

---

## 📝 Fase 9: Task Definitions (2 Apps)

### Langkah 9.1: Task Definition #1 - WordPress

1. **ECS** → **Task definitions** → **Create new task definition**.
2. Name: `lks-wordpress-task`
3. Launch compatibility: **Fargate** ✅
4. **Infrastructure requirements**:
   - OS: Linux/X86_64
   - Task CPU: **512** (.5 vCPU)
   - Task memory: **1024 MB**
   - Task IAM role: `LabRole`
   - Task execution role: `LabRole`

5. **Container - 1**:
   - Name: `lks-wordpress-container`
   - Image URI: `wordpress:latest`
   - Container port: **80** (TCP)
   - Port name: `http`

6. **Environment variables** (dari SSM):
   - `WORDPRESS_DB_HOST` → ValueFrom: `/lks/multiapp/db/host`
   - `WORDPRESS_DB_USER` → ValueFrom: `/lks/multiapp/db/username`
   - `WORDPRESS_DB_PASSWORD` → ValueFrom: `/lks/multiapp/db/password`
   - `WORDPRESS_DB_NAME` → ValueFrom: `/lks/multiapp/db/name`

7. **Logging**:
   - Log driver: **awslogs**
   - Log group: `/ecs/lks/wordpress-app`
   - Region: `us-east-1`

8. **Volumes** (EFS Mount):
   - Klik **Volumes** → **Add volume**
   - Name: `efs-wordpress`
   - Volume type: **EFS**
   - File system ID: `fs-xxxxxxxxxxxx` (catat dari Fase 4)
   - Root directory: `/wordpress/wp-content`
   - Transit encryption: ✅ Enable
   - Authorization: **Access point ID** → `fsap-xxxxxxxxxxxx`
   - ✅ Enable IAM authorization (opsional)

9. **Mount points** (di container definition):
   - Source volume: `efs-wordpress`
   - Container path: `/var/www/html/wp-content`
   - Read only: ❌ **Disable** (WordPress butuh write access)

10. **Create**.

### Langkah 9.2: Task Definition #2 - phpMyAdmin

1. **Create new task definition**.
2. Name: `lks-phpmyadmin-task`
3. Launch compatibility: **Fargate** ✅
4. **Infrastructure requirements**:
   - OS: Linux/X86_64
   - Task CPU: **256** (.25 vCPU)
   - Task memory: **512 MB**
   - Task IAM role: `LabRole`
   - Task execution role: `LabRole`

5. **Container - 1**:
   - Name: `lks-phpmyadmin-container`
   - Image URI: `phpmyadmin:latest`
   - Container port: **8080** (TCP) — phpMyAdmin default listen di 80, tapi kita map ke 8080
   - Port name: `http`

6. **Environment variables** (dari SSM):
   - `PMA_HOST` → ValueFrom: `/lks/multiapp/db/host`
   - `PMA_PORT` → ValueFrom: `/lks/multiapp/db/port`
   - `PMA_ABSOLUTE_URI` → Value: `http://lks-multiapp-alb-xxxxx.us-east-1.elb.amazonaws.com:8080/`

7. **Logging**:
   - Log driver: **awslogs**
   - Log group: `/ecs/lks/phpmyadmin-app`
   - Region: `us-east-1`

8. **Create**.

### Langkah 9.3: Verifikasi IAM Role
Pastikan `LabRole` punya policy:
- `AmazonECSTaskExecutionRolePolicy`
- `AmazonECS_FullAccess`
- Inline policy untuk SSM access (dari Saran 2)
- Inline policy untuk EFS access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "elasticfilesystem:ClientMount",
        "elasticfilesystem:ClientWrite",
        "elasticfilesystem:DescribeMountTargets"
      ],
      "Resource": "arn:aws:elasticfilesystem:us-east-1:*:file-system/*"
    }
  ]
}
```

---

## 🚀 Fase 10: ECS Services (2 Services)

### Langkah 10.1: Service #1 - WordPress
1. **ECS** → **Clusters** → `lks-multiapp-cluster` → **Services** → **Create**.
2. **Environment**:
   - Capacity provider: **Fargate**
   - Platform version: **LATEST**
3. **Deployment configuration**:
   - Task definition: `lks-wordpress-task:1`
   - Service name: `lks-wordpress-service`
   - Desired tasks: **2**
4. **Networking**:
   - Subnets: **kedua private subnet**
   - Security group: `lks-sg-wordpress`
   - Public IP: ❌ Disable
5. **Load balancing**:
   - ✅ Application Load Balancer
   - Select: `lks-multiapp-alb`
   - Container to load balance: `lks-wordpress-container:80:80`
   - Listener: **80:HTTP** → `lks-wordpress-tg`
6. **Health check grace period**: **120 seconds**
7. **Create**.

### Langkah 10.2: Service #2 - phpMyAdmin
1. **Create** service lagi.
2. **Deployment configuration**:
   - Task definition: `lks-phpmyadmin-task:1`
   - Service name: `lks-phpmyadmin-service`
   - Desired tasks: **2**
3. **Networking**:
   - Subnets: **kedua private subnet**
   - Security group: `lks-sg-phpmyadmin`
   - Public IP: ❌ Disable
4. **Load balancing**:
   - ✅ Application Load Balancer
   - Select: `lks-multiapp-alb`
   - Container to load balance: `lks-phpmyadmin-container:8080:8080`
   - Listener: **8080:HTTP** → `lks-phpmyadmin-tg`
5. **Health check grace period**: **90 seconds**
6. **Create**.

### Langkah 10.3: Tunggu Service Stabil
- Tunggu ~5-7 menit.
- Kedua service harus **ACTIVE** dengan 2 tasks **RUNNING** masing-masing.

---

## 🧪 Fase 11: Testing End-to-End

### Test 1: Akses WordPress via Port 80
1. Buka browser:
   ```
   http://lks-multiapp-alb-xxxxxxxx.us-east-1.elb.amazonaws.com
   ```
2. Harus muncul **WordPress installer page**.

### Test 2: Akses phpMyAdmin via Port 8080
1. Buka browser:
   ```
   http://lks-multiapp-alb-xxxxxxxx.us-east-1.elb.amazonaws.com:8080
   ```
2. Harus muncul **phpMyAdmin login page**.

### Test 3: Login ke phpMyAdmin
1. Username: `appadmin`
2. Password: `MultiApp2026!Secure`
3. Harus bisa login dan melihat database `multiapp_db`.

### Test 4: Setup WordPress
1. Di WordPress installer → isi DB credentials:
   - Database Name: `multiapp_db`
   - Username: `appadmin`
   - Password: `MultiApp2026!Secure`
   - Database Host: (dari SSM, atau endpoint RDS)
   - Table Prefix: `wp_`
2. Klik **Submit** → **Run the installation**.
3. Setup admin user → login ke WordPress dashboard.

### Test 5: Verifikasi EFS Mount
1. Upload gambar/media di WordPress.
2. Cek di EFS Console → file harus ada di `/wordpress/wp-content/uploads/`.
3. Stop 1 task WordPress → ECS spawn task baru → **gambar tetap ada** (karena disimpan di EFS, bukan di container).

### Test 6: Cek Target Group Health
1. **EC2** → **Target Groups** → `lks-wordpress-tg` → 2 targets **Healthy**.
2. **EC2** → **Target Groups** → `lks-phpmyadmin-tg` → 2 targets **Healthy**.

### Test 7: Cek CloudWatch Logs
1. `/ecs/lks/wordpress-app` → log WordPress startup.
2. `/ecs/lks/phpmyadmin-app` → log phpMyAdmin startup.

### Test 8: High Availability Test
1. **ECS** → Tasks → **Stop** 1 task WordPress.
2. Tunggu ~1 menit → ECS spawn task baru.
3. Akses WordPress lagi → **tetap bisa diakses** (downtime minimal).
4. Upload gambar baru → **gambar lama tetap ada** (EFS persisten).

---

## 🧹 Fase 12: Cleanup

### Urutan Penghapusan (WAJIB berurutan):

1. **ECS Services** (2 services):
   - Update → Desired tasks: 0 → Save → Delete.

2. **ECS Task Definitions** (2 task defs):
   - Deregister → Delete.

3. **ECS Cluster**:
   - Delete cluster.

4. **ALB**:
   - EC2 → Load Balancers → Delete.

5. **Target Groups** (2 TG):
   - Delete.

6. **RDS Database**:
   - Delete → ❌ disable snapshot.

7. **EFS Access Points** (1 AP):
   - Delete access point.

8. **EFS File System**:
   - Delete file system.

9. **SSM Parameters** (5 parameters):
   - Delete.

10. **Security Groups** (5 SG):
    - Delete semua.

11. **NAT Gateway**:
    - Delete.

12. **Elastic IP**:
    - Release.

13. **VPC**:
    - Delete VPC.

14. **CloudWatch Log Groups** (2 groups):
    - Delete.

---

## 🎓 Checklist Juri (Self-Assessment)

| # | Item | Status |
|---|---|---|
| 1 | Semua resource ber-prefix `lks-` | ☐ |
| 2 | Semua resource di-tag & ada deskripsi | ☐ |
| 3 | Region: `us-east-1` | ☐ |
| 4 | VPC dengan 2 public + 2 private subnet | ☐ |
| 5 | 5 Security Groups (ALB, WordPress, phpMyAdmin, RDS, EFS) | ☐ |
| 6 | SG ALB izinkan HTTP 80 & 8080 dari `0.0.0.0/0` | ☐ |
| 7 | SG WordPress hanya izinkan dari SG ALB (port 80) | ☐ |
| 8 | SG phpMyAdmin hanya izinkan dari SG ALB (port 8080) | ☐ |
| 9 | SG RDS hanya izinkan dari SG WordPress & phpMyAdmin (port 3306) | ☐ |
| 10 | SG EFS hanya izinkan NFS dari SG WordPress (port 2049) | ☐ |
| 11 | RDS tidak publicly accessible | ☐ |
| 12 | EFS encrypted at rest | ☐ |
| 13 | EFS Access Point dengan POSIX UID/GID 33:33 | ☐ |
| 14 | SSM Parameter Store: 5 parameter (host, port, name, username, password) | ☐ |
| 15 | SSM username & password = SecureString | ☐ |
| 16 | ALB internet-facing dengan 2 listeners (80 & 8080) | ☐ |
| 17 | 2 Target Groups (IP target, health check benar) | ☐ |
| 18 | ECS Cluster Fargate | ☐ |
| 19 | 2 Task Definitions dengan SSM ValueFrom | ☐ |
| 20 | WordPress task definition mount EFS (read-write) | ☐ |
| 21 | 2 ECS Services (multi-AZ, private subnet) | ☐ |
| 22 | WordPress bisa diakses via port 80 | ☐ |
| 23 | phpMyAdmin bisa diakses via port 8080 | ☐ |
| 24 | WordPress bisa connect ke RDS | ☐ |
| 25 | phpMyAdmin bisa login ke database | ☐ |
| 26 | EFS persisten setelah task restart | ☐ |
| 27 | CloudWatch logs aktif untuk kedua app | ☐ |
| 28 | Cleanup selesai | ☐ |

---

## 💡 Tips Trainer

### 🔴 Jika Task WordPress STOPPED dengan Error "EFS Mount Failure"
**Penyebab**: IAM role tidak punya izin EFS atau access point salah.
**Solusi**:
1. Cek IAM role punya `elasticfilesystem:ClientMount` & `ClientWrite`.
2. Pastikan Access Point ID benar di task definition.
3. Pastikan UID/GID di access point = 33:33 (www-data).

### 🔴 Jika Target Group phpMyAdmin UNHEALTHY
**Penyebab**: phpMyAdmin listen di port 80 (default), tapi container port di-set 8080.
**Solusi**:
1. Set environment variable `PMA_PORT=8080` di task definition.
2. Atau ubah container port ke 80 dan target group port ke 80.
3. Health check path = `/` dengan success code `200`.

### 🔴 Jika WordPress Tidak Bisa Write ke EFS
**Penyebab**: POSIX permission di access point salah.
**Solusi**:
1. Edit access point → Owner UID/GID = 33:33.
2. Permissions = `755` atau `775`.
3. Di task definition, mount point **read only = false**.

### 🔴 Jika phpMyAdmin Tidak Bisa Connect ke RDS
**Penyebab**: SG RDS tidak izinkan dari SG phpMyAdmin.
**Solusi**:
1. Edit SG `lks-sg-rds` → Inbound rules → tambahkan source `lks-sg-phpmyadmin` port 3306.

### 🔴 Jika ALB 503 di Salah Satu Port
**Penyebab**: Listener tidak di-forward ke target group yang benar.
**Solusi**:
1. Cek tab **Listeners** di ALB.
2. Port 80 → forward ke `lks-wordpress-tg`.
3. Port 8080 → forward ke `lks-phpmyadmin-tg`.

---

## 📊 Apa yang Anda Pelajari dari Saran 3?

| Skill | Status |
|---|---|
| Multi-tier Security Groups (5 SG) | ✅ |
| RDS MySQL di private subnet | ✅ |
| **EFS + Access Points** (skill baru!) | ✅ |
| **Mount EFS ke ECS task** (skill baru!) | ✅ |
| **Multi-port ALB** (skill baru!) | ✅ |
| 2 Target Groups dalam 1 ALB | ✅ |
| 2 Task Definitions dalam 1 cluster | ✅ |
| 2 ECS Services dengan listener berbeda | ✅ |
| SSM ValueFrom untuk credential | ✅ |
| POSIX permission untuk container | ✅ |
| High availability dengan EFS persisten | ✅ |

---

## 🚀 Progression Path

```
✅ Saran 1 (1 app Nginx, manual deploy) — 1-2 jam
    ↓
✅ Saran 2 (+ RDS MySQL + SSM) — 2-3 jam
    ↓
✅ Saran 3 (+ EFS + multi-port ALB + 2 apps) — 3-4 jam ← ANDA DI SINI
    ↓
🔴 Saran 4 (+ ECR + GitHub Actions CI/CD) — 4-5 jam
    ↓
🏆 SOAL LKS DevOps CI/CD (Leantime + Gatus, 3 jam)
```

---

## 🎯 Analogi Langsung ke Soal LKS

| Yang Anda Lakukan di Saran 3 | Yang Akan Anda Lakukan di Soal LKS |
|---|---|
| WordPress task definition | **Leantime task definition** (`lks-leantime-app`) |
| phpMyAdmin task definition | **Gatus task definition** (`lks-gatus-app`) |
| 1 EFS Access Point | **3 EFS Access Points** (`/public_userfiles`, `/userfiles`, `/plugins`) |
| ALB port 80 + 8080 | **ALB port 80 + 8080** (persis sama!) |
| Manual deploy | **GitHub Actions CI/CD** |
| RDS MySQL | **RDS Aurora MySQL** |

> 💡 **Jika Anda lancar di Saran 3, Anda sudah 70% siap untuk soal LKS!** Tinggal tambah GitHub Actions pipeline dan 2 access points lagi.

---

## 🎯 Next Step

Setelah Saran 3 berhasil, Anda siap lanjut ke **Saran 4**: menambahkan **ECR** + **GitHub Actions CI/CD** untuk otomatisasi deployment.

Apakah Anda ingin saya buatkan:
1. **Panduan lengkap Saran 4** (WordPress + phpMyAdmin + ECR + GitHub Actions)?
2. **Panduan upgrade Saran 3** dengan 3 EFS Access Points (persis seperti Leantime)?
3. **Template task-definition.json** yang siap dipakai di soal LKS?

Silakan pilih, dan saya akan buatkan! 🎯