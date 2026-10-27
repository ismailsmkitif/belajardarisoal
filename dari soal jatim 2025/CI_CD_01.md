# 📘 Panduan Lengkap Pengerjaan Saran 1 (DevOps CI/CD)
## "Simple Container Deployment" — Nginx di ECS Fargate + ALB

> **Estimasi waktu**: 1-2 jam | **Region**: `us-east-1` (N. Virginia) | **Console-based**

---

## 🎯 Tujuan Latihan

Membangun fondasi **container deployment** di AWS dengan:
- ✅ 1 aplikasi web statis (Nginx custom)
- ✅ ECS Fargate (tanpa manage server)
- ✅ Application Load Balancer (internet-facing)
- ✅ ECR (container registry)
- ✅ VPC dengan public + private subnet
- ✅ Security Groups yang ketat

Ini adalah **step 0** sebelum Anda mengerjakan soal LKS DevOps CI/CD yang sesungguhnya (Leantime + Gatus + GitHub Actions).

---

## 📋 Daftar Isi
1. [Fase 0: Persiapan](#fase-0-persiapan)
2. [Fase 1: Networking (VPC)](#fase-1-networking-vpc)
3. [Fase 2: Security Groups](#fase-2-security-groups)
4. [Fase 3: ECR (Container Registry)](#fase-3-ecr)
5. [Fase 4: Build & Push Docker Image](#fase-4-build--push-docker-image)
6. [Fase 5: Application Load Balancer + Target Group](#fase-5-alb--target-group)
7. [Fase 6: ECS Cluster](#fase-6-ecs-cluster)
8. [Fase 7: Task Definition](#fase-7-task-definition)
9. [Fase 8: ECS Service](#fase-8-ecs-service)
10. [Fase 9: Testing End-to-End](#fase-9-testing)
11. [Fase 10: Cleanup](#fase-10-cleanup)

---

## 🎯 Fase 0: Persiapan

### Langkah 0.1: Login & Set Region
1. Login ke **AWS Lab Learner** → klik **Start Lab** → **Open AWS Console**.
2. Di pojok kanan atas, pastikan region = **US East (N. Virginia) `us-east-1`**.
3. Buka **Notepad** lokal untuk mencatat:
   - Account ID (12 digit, ada di pojok kanan atas)
   - VPC ID (akan diisi di Fase 1)
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
   - Name tag: `lks-simple-vpc`
   - IPv4 CIDR: `10.50.0.0/16`
   - Number of Availability Zones: **2**
   - Number of public subnets: **2**
   - Number of private subnets: **2**
   - ✅ Centang **NAT gateways (1)** — kita akan pakai untuk pull image ECR
   - ❌ **Tidak perlu** VPC endpoints (kita pakai NAT saja untuk simplicity)
4. Klik **Create VPC**. Tunggu ~2 menit.

### Langkah 1.2: Catat Resource IDs
Buka **VPC** → catat:
- **VPC ID**: `vpc-xxxxxxxx`
- **Public Subnet 1**: `subnet-pub1-xxxx` (AZ a)
- **Public Subnet 2**: `subnet-pub2-xxxx` (AZ b)
- **Private Subnet 1**: `subnet-priv1-xxxx` (AZ a)
- **Private Subnet 2**: `subnet-priv2-xxxx` (AZ b)

### Langkah 1.3: Rename Resources (Sesuai Naming Convention)
1. **Internet Gateway** → rename ke `lks-simple-igw`
2. **NAT Gateway** → rename ke `lks-simple-natgw`
3. **Public Route Table** → rename ke `lks-simple-public-rt`
4. **Private Route Table** → rename ke `lks-simple-private-rt`

> 💡 **Catatan Trainer**: NAT Gateway diperlukan agar ECS task di private subnet bisa pull image dari ECR (ECR ada di internet). Pada soal LKS sesungguhnya, Anda akan pakai VPC Endpoint untuk hemat biaya.

---

## 🔒 Fase 2: Security Groups

### Langkah 2.1: SG untuk ALB (Public)
1. **VPC** → **Security Groups** → **Create Security Group**.
2. Name: `lks-sg-alb`
3. Description: `Security group for ALB - allows HTTP from internet`
4. VPC: `lks-simple-vpc`
5. **Inbound rules**:
   | Type | Protocol | Port | Source | Description |
   |---|---|---|---|---|
   | HTTP | TCP | 80 | `0.0.0.0/0` | Allow HTTP from internet |
6. **Outbound rules**: All traffic (default)
7. **Create**.

### Langkah 2.2: SG untuk ECS Task (Private)
1. **Create Security Group** lagi.
2. Name: `lks-sg-app`
3. Description: `Security group for ECS tasks - only from ALB`
4. VPC: `lks-simple-vpc`
5. **Inbound rules**:
   | Type | Protocol | Port | Source | Description |
   |---|---|---|---|---|
   | Custom TCP | TCP | 80 | `lks-sg-alb` | Allow traffic from ALB only |
6. **Outbound rules**: All traffic (default)
7. **Create**.

> ⚠️ **PENTING**: Perhatikan Source = **`lks-sg-alb`** (bukan `0.0.0.0/0`). Ini adalah prinsip keamanan paling dasar: aplikasi hanya bisa diakses melalui ALB.

---

## 📦 Fase 3: ECR (Container Registry)

### Langkah 3.1: Buat ECR Repository
1. Buka **ECR** → **Repositories** → **Create repository**.
2. Visibility: **Private**
3. Name: `lks-nginx-app`
4. Tag immutability: **Disable** (mutable, agar tag `latest` bisa di-update)
5. Scan on push: ✅ Enable (opsional, untuk security scanning)
6. **Create repository**.

### Langkah 3.2: Catat Repository URI
Setelah dibuat, catat URI:
```
<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/lks-nginx-app
```

---

## 🐳 Fase 4: Build & Push Docker Image

### Langkah 4.1: Siapkan File Docker
Buat 2 file di komputer lokal:

#### 📄 `Dockerfile`
```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
COPY style.css /usr/share/nginx/html/style.css
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

#### 📄 `index.html`
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>LKS Simple Container - Success!</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <div class="container">
    <h1>🎉 Selamat!</h1>
    <h2>Container Deployment Berhasil</h2>
    <p>Aplikasi Nginx Anda berjalan di <strong>AWS ECS Fargate</strong></p>
    <p>Region: <code>us-east-1</code></p>
    <p>Load Balancer: <strong>Active</strong></p>
    <div class="badge">
      <span>✅ VPC</span>
      <span>✅ ECR</span>
      <span>✅ ECS Fargate</span>
      <span>✅ ALB</span>
    </div>
    <p class="footer">Latihan LKS DevOps CI/CD - Saran 1</p>
  </div>
</body>
</html>
```

#### 📄 `style.css`
```css
* { box-sizing: border-box; margin: 0; padding: 0; }
body {
  font-family: 'Segoe UI', Arial, sans-serif;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
}
.container {
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  padding: 40px;
  border-radius: 20px;
  text-align: center;
  max-width: 600px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.2);
}
h1 { font-size: 3em; margin-bottom: 10px; }
h2 { font-size: 1.5em; margin-bottom: 20px; opacity: 0.9; }
p { margin: 10px 0; font-size: 1.1em; }
code {
  background: rgba(0, 0, 0, 0.3);
  padding: 2px 8px;
  border-radius: 4px;
  font-family: 'Courier New', monospace;
}
.badge {
  display: flex;
  gap: 10px;
  justify-content: center;
  flex-wrap: wrap;
  margin: 20px 0;
}
.badge span {
  background: rgba(255, 255, 255, 0.2);
  padding: 8px 16px;
  border-radius: 20px;
  font-weight: 600;
}
.footer {
  margin-top: 30px;
  font-size: 0.9em;
  opacity: 0.7;
}
```

### Langkah 4.2: Build Image via AWS CloudShell

> 💡 **Kenapa CloudShell?** Karena AWS Lab Learner biasanya tidak mengizinkan Docker di mesin lokal, dan CloudShell sudah include Docker.

1. Buka **CloudShell** (ikon `>_` di top bar AWS Console).
2. Tunggu terminal siap (~30 detik).
3. Buat folder kerja:
```bash
mkdir -p ~/lks-nginx && cd ~/lks-nginx
```

4. Buat file `Dockerfile`:
```bash
cat > Dockerfile << 'EOF'
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
COPY style.css /usr/share/nginx/html/style.css
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF
```

5. Buat file `index.html`:
```bash
cat > index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>LKS Simple Container - Success!</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <div class="container">
    <h1>🎉 Selamat!</h1>
    <h2>Container Deployment Berhasil</h2>
    <p>Aplikasi Nginx Anda berjalan di <strong>AWS ECS Fargate</strong></p>
    <p>Region: <code>us-east-1</code></p>
    <p>Load Balancer: <strong>Active</strong></p>
    <div class="badge">
      <span>✅ VPC</span>
      <span>✅ ECR</span>
      <span>✅ ECS Fargate</span>
      <span>✅ ALB</span>
    </div>
    <p class="footer">Latihan LKS DevOps CI/CD - Saran 1</p>
  </div>
</body>
</html>
EOF
```

6. Buat file `style.css`:
```bash
cat > style.css << 'EOF'
* { box-sizing: border-box; margin: 0; padding: 0; }
body {
  font-family: 'Segoe UI', Arial, sans-serif;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
}
.container {
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  padding: 40px;
  border-radius: 20px;
  text-align: center;
  max-width: 600px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.2);
}
h1 { font-size: 3em; margin-bottom: 10px; }
h2 { font-size: 1.5em; margin-bottom: 20px; opacity: 0.9; }
p { margin: 10px 0; font-size: 1.1em; }
code {
  background: rgba(0, 0, 0, 0.3);
  padding: 2px 8px;
  border-radius: 4px;
  font-family: 'Courier New', monospace;
}
.badge {
  display: flex;
  gap: 10px;
  justify-content: center;
  flex-wrap: wrap;
  margin: 20px 0;
}
.badge span {
  background: rgba(255, 255, 255, 0.2);
  padding: 8px 16px;
  border-radius: 20px;
  font-weight: 600;
}
.footer {
  margin-top: 30px;
  font-size: 0.9em;
  opacity: 0.7;
}
EOF
```

### Langkah 4.3: Login ke ECR & Build Image

1. **Login ke ECR** (CloudShell otomatis punya credential):
```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
```
> Ganti `<ACCOUNT_ID>` dengan Account ID Anda (12 digit).

2. **Build Docker image**:
```bash
docker build -t lks-nginx-app:latest .
```

3. **Tag image** untuk ECR:
```bash
docker tag lks-nginx-app:latest <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/lks-nginx-app:latest
```

4. **Push image ke ECR**:
```bash
docker push <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/lks-nginx-app:latest
```

5. **Verifikasi** di ECR Console → repository `lks-nginx-app` → harus ada image dengan tag `latest`.

---

## 🌐 Fase 5: ALB + Target Group

### Langkah 5.1: Buat Target Group
1. Buka **EC2** → **Target Groups** → **Create target group**.
2. Target type: **IP addresses** (WAJIB untuk Fargate)
3. Name: `lks-nginx-tg`
4. Protocol: **HTTP**
5. Port: **80**
6. VPC: `lks-simple-vpc`
7. **Health check settings**:
   - Protocol: HTTP
   - Path: `/`
   - Success code: `200`
   - Healthy threshold: **3**
   - Unhealthy threshold: **2**
   - Timeout: **5 seconds**
   - Interval: **30 seconds**
8. **Create**.

### Langkah 5.2: Buat Application Load Balancer
1. **EC2** → **Load Balancers** → **Create Load Balancer**.
2. Pilih **Application Load Balancer** → **Create**.
3. **Basic configuration**:
   - Name: `lks-simple-alb`
   - Scheme: **Internet-facing**
   - IP address type: **IPv4**
4. **Network mapping**:
   - VPC: `lks-simple-vpc`
   - Mappings: ✅ centang **kedua public subnet** (AZ a & b)
5. **Security groups**:
   - Hapus default, pilih: `lks-sg-alb`
6. **Listeners**:
   - Protocol: **HTTP**, Port: **80**
   - Default action: **Forward to** → `lks-nginx-tg`
7. **Create load balancer**. Tunggu ~2 menit.

### Langkah 5.3: Catat ALB DNS Name
Setelah status **Active**:
- Klik ALB → tab **Description** → catat **DNS name**:
  ```
  lks-simple-alb-xxxxxxxx.us-east-1.elb.amazonaws.com
  ```

---

## 🎯 Fase 6: ECS Cluster

### Langkah 6.1: Buat ECS Cluster
1. Buka **ECS** → **Clusters** → **Create cluster**.
2. Cluster name: `lks-simple-cluster`
3. Infrastructure: **AWS Fargate (Serverless)**
4. **Create**.

---

## 📝 Fase 7: Task Definition

### Langkah 7.1: Buat Task Definition
1. **ECS** → **Task definitions** → **Create new task definition**.
2. **Task definition configuration**:
   - Name: `lks-nginx-task`
   - Launch compatibility: **Fargate** ✅
3. **Infrastructure requirements**:
   - OS: **Linux/X86_64**
   - Task CPU: **256** (.25 vCPU)
   - Task memory: **512 MB**
   - Task IAM role: `LabRole`
   - Task execution role: `LabRole`
4. **Container - 1**:
   - Name: `lks-nginx-container`
   - Image URI: `<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/lks-nginx-app:latest`
   - Container port: **80** (TCP)
   - Port name: `http`
5. **Logging**:
   - Log driver: **awslogs**
   - Log group: `/ecs/lks/nginx-app` (auto-create)
   - Region: `us-east-1`
6. **Monitoring**:
   - ✅ Enable Container Insights (opsional)
7. **Create**.

> 💡 **Catatan Trainer**: Jika `LabRole` belum punya izin ECR pull, tambahkan policy `AmazonECSTaskExecutionRolePolicy` ke LabRole di IAM.

### Langkah 7.2: Verifikasi IAM Role
1. Buka **IAM** → **Roles** → `LabRole`.
2. Pastikan ada policy:
   - `AmazonECSTaskExecutionRolePolicy` (untuk pull image ECR & CloudWatch logs)
   - `AmazonECS_FullAccess` (untuk ECS operations)
3. Jika belum ada → **Add permissions** → **Attach policies** → cari & pilih → **Add**.

---

## 🚀 Fase 8: ECS Service

### Langkah 8.1: Buat ECS Service
1. **ECS** → **Clusters** → klik `lks-simple-cluster`.
2. Tab **Services** → **Create**.
3. **Environment**:
   - Cluster: `lks-simple-cluster`
   - Capacity provider: **Fargate**
   - Platform version: **LATEST**
4. **Deployment configuration**:
   - Application type: **Task**
   - Task definition: `lks-nginx-task` (pilih revision terbaru)
   - Service name: `lks-nginx-service`
   - Desired tasks: **2** (untuk high availability)
5. **Deployment options**:
   - Deployment type: **Rolling update**
   - Minimum healthy percentage: **100**
   - Maximum percentage: **200**
6. **Networking**:
   - ✅ Use existing VPC: `lks-simple-vpc`
   - Subnets: pilih **kedua private subnet**
   - Security group: `lks-sg-app`
   - Public IP: ❌ **Disable**
7. **Load balancing**:
   - ✅ **Application Load Balancer**
   - Select existing: `lks-simple-alb`
   - Container to load balance: `lks-nginx-container:80`
   - Listener: **80:HTTP** → forward to `lks-nginx-tg`
8. **Health check grace period**: **60 seconds**
9. **Auto scaling**: ❌ Disable (untuk latihan sederhana)
10. **Create**.

### Langkah 8.2: Tunggu Service Stabil
- Tunggu ~3-5 menit sampai status service = **ACTIVE**.
- Tab **Tasks** → harus ada 2 tasks dengan status **RUNNING**.
- Jika task status **STOPPED** → klik task → cek **Events** & **Logs** untuk troubleshooting.

---

## 🧪 Fase 9: Testing End-to-End

### Test 1: Akses via ALB DNS
1. Buka browser → akses:
   ```
   http://lks-simple-alb-xxxxxxxx.us-east-1.elb.amazonaws.com
   ```
2. Harus muncul halaman "🎉 Selamat! Container Deployment Berhasil".

### Test 2: Cek Target Group Health
1. **EC2** → **Target Groups** → klik `lks-nginx-tg`.
2. Tab **Targets** → harus ada 2 targets dengan status **Healthy**.

### Test 3: Cek ECS Tasks
1. **ECS** → **Clusters** → `lks-simple-cluster` → **Tasks**.
2. Harus ada 2 tasks **RUNNING** di subnet berbeda (multi-AZ).

### Test 4: Cek CloudWatch Logs
1. **CloudWatch** → **Log groups** → `/ecs/lks/nginx-app`.
2. Klik log stream → harus ada log Nginx access.

### Test 5: Test High Availability (Opsional)
1. **ECS** → Tasks → **Stop** 1 task.
2. Tunggu ~1 menit → ECS akan otomatis spawn task baru.
3. Akses ALB lagi → halaman tetap muncul (downtime minimal).

---

## 🧹 Fase 10: Cleanup

> ⚠️ **PENTING**: Hapus semua resource untuk menghindari tagihan!

### Urutan Penghapusan (WAJIB berurutan):

1. **ECS Service** → ECS → Clusters → `lks-simple-cluster` → Services → pilih `lks-nginx-service` → **Update** → Desired tasks: 0 → **Save** → tunggu task berhenti → **Delete service**.

2. **ECS Task Definition** → Task definitions → `lks-nginx-task` → **Deregister** → **Delete**.

3. **ECS Cluster** → Clusters → `lks-simple-cluster` → **Delete cluster**.

4. **ALB** → EC2 → Load Balancers → `lks-simple-alb` → **Actions** → **Delete**.

5. **Target Group** → Target Groups → `lks-nginx-tg` → **Actions** → **Delete**.

6. **ECR Repository** → ECR → Repositories → `lks-nginx-app` → **Delete**.

7. **Security Groups** → VPC → Security Groups → delete `lks-sg-alb` dan `lks-sg-app`.

8. **NAT Gateway** → VPC → NAT Gateways → delete `lks-simple-natgw`.

9. **Elastic IP** → EC2 → Elastic IPs → **Release** (agar tidak ditagih!).

10. **VPC** → VPC → `lks-simple-vpc` → **Actions** → **Delete VPC** (akan hapus subnet, route tables, IGW otomatis).

11. **CloudWatch Log Group** → CloudWatch → Log groups → `/ecs/lks/nginx-app` → **Delete**.

---

## 🎓 Checklist Juri (Self-Assessment)

| # | Item | Status |
|---|---|---|
| 1 | Semua resource ber-prefix `lks-` | ☐ |
| 2 | Semua resource di-tag & ada deskripsi | ☐ |
| 3 | Region: `us-east-1` | ☐ |
| 4 | VPC: `10.50.0.0/16` dengan 2 public + 2 private subnet | ☐ |
| 5 | SG ALB hanya izinkan HTTP 80 dari `0.0.0.0/0` | ☐ |
| 6 | SG App hanya izinkan port 80 dari SG ALB | ☐ |
| 7 | ECR repository `lks-nginx-app` dengan tag mutable | ☐ |
| 8 | Docker image berhasil di-push ke ECR | ☐ |
| 9 | ALB internet-facing dengan 2 listener (HTTP 80) | ☐ |
| 10 | Target Group: IP target, health check `/`, success 200 | ☐ |
| 11 | ECS Cluster Fargate (serverless) | ☐ |
| 12 | Task Definition: Fargate, 256 CPU, 512 MB, awslogs | ☐ |
| 13 | ECS Service: 2 tasks, multi-AZ, private subnet | ☐ |
| 14 | Task status RUNNING di 2 subnet berbeda | ☐ |
| 15 | Target group health: 2 targets HEALTHY | ☐ |
| 16 | Halaman Nginx bisa diakses via ALB DNS | ☐ |
| 17 | CloudWatch logs aktif | ☐ |
| 18 | Cleanup selesai (tidak ada resource tersisa) | ☐ |

---

## 💡 Tips Trainer

### 🔴 Jika Task STOPPED dengan Error "CannotPullContainerError"
**Penyebab**: IAM role tidak punya izin pull dari ECR.
**Solusi**: 
1. Buka IAM → Roles → `LabRole`.
2. Attach policy `AmazonECSTaskExecutionRolePolicy`.
3. Restart task (Stop → ECS akan auto-spawn baru).

### 🔴 Jika Target Group UNHEALTHY
**Penyebab**: Security Group App tidak izinkan traffic dari SG ALB.
**Solusi**:
1. Cek SG `lks-sg-app` → Inbound rules → Source harus `lks-sg-alb` (bukan IP).
2. Pastikan port = 80, protocol = TCP.

### 🔴 Jika ALB 503 Service Unavailable
**Penyebab**: Target group tidak punya healthy target.
**Solusi**:
1. Tunggu ~2 menit (health check butuh waktu).
2. Cek tab **Targets** di target group → status harus **Healthy**.
3. Jika masih unhealthy → cek Security Group & task logs.

### 🔴 Jika NAT Gateway Tidak Bekerja
**Penyebab**: Route table private subnet tidak point ke NAT.
**Solusi**:
1. VPC → Route Tables → `lks-simple-private-rt`.
2. Routes → harus ada: `0.0.0.0/0` → target `lks-simple-natgw`.

---

## 📊 Apa yang Anda Pelajari dari Saran 1?

| Skill | Status |
|---|---|
| VPC dengan public + private subnet | ✅ |
| Security Groups dengan referensi SG | ✅ |
| ECR (build, tag, push Docker image) | ✅ |
| Dockerfile dasar (Nginx + custom HTML) | ✅ |
| ALB + Target Group (health check) | ✅ |
| ECS Cluster Fargate | ✅ |
| Task Definition (Fargate spec) | ✅ |
| ECS Service (multi-AZ, private subnet) | ✅ |
| CloudWatch Logs integration | ✅ |
| Cleanup yang benar (urutan penting!) | ✅ |

---

## 🚀 Progression Path

```
✅ Saran 1 (1 app Nginx, manual deploy) — 1-2 jam ← ANDA DI SINI
    ↓
🟡 Saran 2 (+ RDS MySQL + SSM Parameter Store) — 2-3 jam
    ↓
🟠 Saran 3 (+ EFS + multi-port ALB + 2 apps) — 3-4 jam
    ↓
🔴 Saran 4 (+ ECR + GitHub Actions CI/CD) — 4-5 jam
    ↓
🏆 SOAL LKS DevOps CI/CD (Leantime + Gatus, 3 jam)
```

---

## 🎯 Next Step

Setelah Saran 1 berhasil, Anda siap lanjut ke **Saran 2**: menambahkan **RDS MySQL** + **SSM Parameter Store** untuk deploy aplikasi yang butuh database (misal WordPress).

Apakah Anda ingin saya buatkan:
1. **Panduan lengkap Saran 2** (WordPress + RDS + SSM)?
2. **Template troubleshooting** untuk error yang sering muncul di ECS Fargate?
3. **Script cleanup otomatis** untuk menghapus semua resource sekaligus?

Silakan pilih, dan saya akan buatkan! 🎯