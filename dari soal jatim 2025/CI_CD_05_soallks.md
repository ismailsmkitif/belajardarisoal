# 📘 PANDUAN LENGKAP SOAL LKS DEVOPS CI/CD
## **Deploy Leantime + Gatus dengan ECS Fargate, Aurora MySQL, EFS & GitHub Actions**

> **Basis dokumen**: `modul-devops CICD.pdf`
> **Alokasi waktu**: 3 jam (180 menit) | **Region wajib**: **`us-east-1` (N. Virginia)**
> **Metode**: AWS Management Console + GitHub + local editor polos (VS Code/Vim **tanpa extension**)

---

## ⚠️ BAGIAN 0 — HAL YANG PALING SERING MENGGUGURKAN PESERTA

Baca ini **sebelum** menyentuh console.

| No | Jebakan | Jawaban yang Benar |
|---|---|---|
| 1 | Salah region (terbiasa `us-west-2` dari soal Serverless) | **`us-east-1`** — semua resource |
| 2 | Health check Leantime path `/` + success `200` | Leantime **redirect 302** ke `/auth/login` → gunakan path `/auth/login`, success code **`200,302`** |
| 3 | Hardcode GitHub token / URL di `gatus/config/config.yaml` | **DILARANG** → injeksi lewat **environment variables di CI/CD** |
| 4 | Port container Leantime 80 | Wajib **8080** (`app port 8080`), containerPort = hostPort = **8080** |
| 5 | ECR tag Immutable | Wajib **Mutable** agar `latest` bisa ditimpa |
| 6 | Nama service ECS diubah jadi `lks-leantime-service` | Modul **menyebut eksplisit**: `leantime-service` & `gatus-service` (tanpa prefix) |
| 7 | Menaruh RDS/EFS di public subnet | **Private subnet**, no public access |
| 8 | Menyisakan `example-*.yml` di `.github/workflows/` | **Hapus semua**, sisakan hanya `lks-deployment-pipeline.yml` |
| 9 | Lupa `debug=false` di Leantime | Wajib **disable** sebelum final |
| 10 | Tidak pakai prefix `lks-` / tidak isi tag & description | **0 penilaian** untuk resource tanpa penamaan benar |

> 📌 **Catatan penting tentang sumber kode**: Modul merujuk template repo (`leantime/.env`, `leantime/task-definition.json`, `gatus/config/config.yaml`, `gatus/task-definition.json`) yang **isi persisnya tidak disertakan dalam PDF**. Panduan di bawah menyediakan **template lengkap yang siap pakai**, tetapi **nama key environment Leantime dan placeholder wajib Anda cocokkan dengan file asli di repo saat lomba**. Jangan menebak nama key — salin dari `.env` sample.

---

## 📋 TARGET AKHIR (Definition of Done)

```
Internet
   │
   ├── HTTP :80 ──► lks-oss-lb ──► lks-leantime-app-tg (:8080) ──► Leantime (Fargate, private)
   │                                    │            │
   │                                    │            ├──► Aurora MySQL (private)
   │                                    │            └──► EFS (3 access points)
   │                                    └──► SSM /lks/db/*
   │
   └── HTTP :8080 ► lks-oss-lb ──► lks-gatus-app-tg (:8080) ──► Gatus (Fargate, private, 2 task)
                                                                  │
                                                                  └──► GitHub Issue (alert)

GitHub push → lks-deployment-pipeline.yml → ECR lks-gatus → ECS Update Service
```

---

## 🗓️ RENCANA EKSEKUSI 180 MENIT

| Menit | Fase | Output |
|---|---|---|
| 0–10 | 0. Persiapan & clone repo | Repo siap, checklist, token |
| 10–30 | 1–2. VPC + Security Groups | Jaringan 3-tier + 4 SG |
| 30–50 | 3. Aurora MySQL | Cluster available |
| 50–70 | 4. EFS + 3 Access Points | EFS siap mount |
| 70–80 | 5. SSM Parameter Store | 5 parameter |
| 80–95 | 6–7. ECR + TG + ALB | Load balancer active |
| 95–115 | 8–10. ECS Cluster + Task Def + Service | 2 service stabil |
| 115–150 | 11–14. GitHub Token, Secrets, Workflow | Pipeline hijau |
| 150–170 | 15. Verifikasi & setup Leantime | Aplikasi berfungsi |
| 170–180 | 16. Cleanup & audit tag | Lingkungan bersih |

---

# FASE 0 — PERSIAPAN

### 0.1 Login & Region
1. AWS Academy Lab Console → **Start Lab** → **Open AWS Console**.
2. Kanan atas → **US East (N. Virginia) `us-east-1`**.
3. Catat di Notepad:
   - **Account ID** (12 digit)
   - **Access Key ID / Secret Access Key / Session Token** (dari Lab Details)
   - **LabRole ARN**

### 0.2 Clone Repository
```bash
git clone <URL_REPO_TEMPLATE_YANG_DIBERIKAN_JURI>
cd <repo>
```

Struktur yang harus ada:
```
.
├── .github/workflows/
│   ├── example-build-main.yml        ← HAPUS nanti
│   ├── example-deploy-main.yml       ← HAPUS nanti
│   └── lks-deployment-pipeline.yml   ← AKTIFKAN/ISI
├── leantime/
│   ├── .env.sample / .env
│   └── task-definition.json
└── gatus/
    ├── config/config.yaml
    ├── Dockerfile
    └── task-definition.json
```

### 0.3 Editor
Gunakan **VS Code / Vim / Nano polos** — **tanpa plugin/extension** (syarat teknis modul).

---

# FASE 1 — NETWORKING (VPC)

### 1.1 Buat VPC
**VPC → Your VPCs → Create VPC → VPC and more**

| Field | Nilai |
|---|---|
| Name tag | `lks-pm-vpc` |
| IPv4 CIDR | **`10.100.0.0/16`** |
| Tenancy | Default |
| Number of AZs | **2** (pilih `us-east-1a`, `us-east-1b`) |
| Public subnets | **2** |
| Private subnets | **2** |
| NAT gateways | **1** (single NAT, hemat biaya) |
| VPC endpoints | **Tidak perlu** (ECR/ECS butuh NAT di sini) |

> Setelah dibuat, **edit CIDR manual** agar persis sesuai modul:

| Subnet | Name tag | CIDR | AZ |
|---|---|---|---|
| Public 1 | `lks-pm-public-subnet-1a` | **`10.100.0.0/24`** | us-east-1a |
| Public 2 | `lks-pm-public-subnet-1b` | **`10.100.1.0/24`** | us-east-1b |
| Private 1 | `lks-pm-private-subnet-1a` | **`10.100.10.0/24`** | us-east-1a |
| Private 2 | `lks-pm-private-subnet-1b` | **`10.100.11.0/24`** | us-east-1b |

### 1.2 Internet Gateway & NAT
| Resource | Name | Lokasi |
|---|---|---|
| Internet Gateway | **`lks-pm-igw`** | attach ke `lks-pm-vpc` |
| NAT Gateway | **`lks-pm-natgw`** | **Public Subnet 1a**, allocate Elastic IP |

### 1.3 Route Tables
| Route Table | Association | Default Route (`0.0.0.0/0`) |
|---|---|---|
| **`lks-pm-public-rt`** | 2 public subnet | → `lks-pm-igw` |
| **`lks-pm-private-rt`** | 2 private subnet | → `lks-pm-natgw` |

> 🔎 **Cek wajib**: buka `lks-pm-private-rt` → harus ada route `0.0.0.0/0 → nat-xxxx`. Tanpa ini, Fargate **tidak bisa pull image** dan Gatus **tidak bisa mencapai GitHub/ALB**.

### 1.4 Tag Semua Resource
Tambahkan pada **setiap** resource manual:
```
Project     = LKS-DevOps
Environment = Production
Owner       = <NamaPeserta>
```
Description: fungsi resource (mis. `"Public tier for ALB"`).

---

# FASE 2 — SECURITY GROUPS

Buat **4 SG**, semuanya di `lks-pm-vpc`.

### 2.1 `lks-sg-alb`
| Direction | Type | Port | Source/Dest |
|---|---|---|---|
| Inbound | HTTP | **80** | `0.0.0.0/0` |
| Inbound | Custom TCP | **8080** | `0.0.0.0/0` |
| Outbound | All | All | `0.0.0.0/0` |

### 2.2 `lks-sg-app`
| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | Custom TCP | **8080** | **`lks-sg-alb`** (referensi SG, bukan IP!) |
| Outbound | All | All | `0.0.0.0/0` |

### 2.3 `lks-sg-db`
| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | MySQL/Aurora | **3306** | **`lks-sg-app`** |

### 2.4 `lks-sg-efs`
| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | NFS | **2049** | **`lks-sg-app`** |

> ❌ Jangan ada SG database/EFS dengan source `0.0.0.0/0`.
> ❌ Jangan buka 22/SSH di mana pun.

---

# FASE 3 — DATABASE (AURORA MYSQL)

**RDS → Create database**

| Field | Nilai |
|---|---|
| Engine | **MySQL** |
| Version | **8.0.x compatible** |
| Template | **Production** |
| Cluster identifier | **`lks-project-management-cluster`** |
| Credentials | Master username: `lksadmin` / password: `<PasswordKuatAnda>` |
| Instance configuration | **Aurora Standard** |
| DB instance class | **`db.t4g.medium`** |
| Aurora replica | **Enable** (minimal 1 reader) |
| VPC | `lks-pm-vpc` |
| Subnet group | private subnet 1a + 1b |
| Public access | **No** ❌ |
| VPC security group | **Existing → `lks-sg-db`** (hapus default) |
| Initial database name | **`leantime_db`** |
| Monitoring | **Database Insights: Standard** |
| Enhanced monitoring | **Disable** ❌ |
| Backup | Default (7 hari) |
| Deletion protection | Boleh ON selama latihan, **OFF sebelum cleanup** |

### Ambil Endpoint
Setelah `Available` → **Connectivity** → salin **Writer endpoint**:
```
lks-project-management-cluster-xxx.cluster-yyy.us-east-1.rds.amazonaws.com
```
> Gunakan **Cluster Writer Endpoint**, bukan instance endpoint dan bukan reader endpoint.

---

# FASE 4 — EFS + 3 ACCESS POINTS

### 4.1 File System
**EFS → Create file system**

| Field | Nilai |
|---|---|
| Name | **`lks-project-management`** |
| Type | **Regional** |
| VPC | `lks-pm-vpc` |
| Attached subnets | **2 private subnet** |
| Security group | **`lks-sg-efs`** |
| Encryption at rest | **Enable** ✅ |
| Performance mode | **General Purpose** |
| Throughput mode | **Elastic** |
| Lifecycle management | Default |

Catat **File system ID**: `fs-0xxxxxxxx`

### 4.2 Tentukan UID/GID
Modul: *"Use User ID and Group ID that are used in the Leantime environment variables"*.

Buka `leantime/.env` dari repo → cari key UID/GID (mis. `LEAN_FILE_UID` / `LEAN_FILE_GID` atau setara).
```
APP_UID = <dari repo>     # contoh umum 33 atau 1000
APP_GID = <dari repo>
```
> ⚠️ **Jangan asal isi 0 atau 1000** jika repo menentukan lain — ini penyebab #1 EFS write permission denied.

### 4.3 Buat 3 Access Point

**Access points → Create access point** — ulangi 3×:

| # | Name | Root directory path | UID | GID | Creation permission |
|---|---|---|---|---|---|
| 1 | `lks-ap-public-userfiles` | **`/public_userfiles`** | APP_UID | APP_GID | **`764`** |
| 2 | `lks-ap-userfiles` | **`/userfiles`** | APP_UID | APP_GID | **`764`** |
| 3 | `lks-ap-plugins` | **`/plugins`** | APP_UID | APP_GID | **`764`** |

**Penjelasan permission `764`** (sesuai modul: user full, group read-write, others read):
- Owner `7` = rwx
- Group `6` = rw-
- Others `4` = r--

✅ Centang **Create the root directory** agar folder otomatis dibuat.

Catat ketiga **Access Point ID**: `fsap-0aaa`, `fsap-0bbb`, `fsap-0ccc`.

### 4.4 Mapping Mount (WAJIB persis)

| Access Point | Container Path | Read-only |
|---|---|---|
| `/public_userfiles` | `/var/www/html/public/userfiles` | **false** |
| `/userfiles` | `/var/www/html/userfiles` | **false** |
| `/plugins` | `/var/www/html/app/Plugins` | **false** |

> ⚠️ Di tabel modul ada penulisan `/public-files`, tetapi **Service Details** menegaskan `/public_userfiles`. Gunakan **`/public_userfiles`** sebagai root directory EFS, dan path container tetap `/var/www/html/public/userfiles`.

---

# FASE 5 — SSM PARAMETER STORE

**Systems Manager → Parameter Store → Create parameter**

| Parameter Name | Type | Value |
|---|---|---|
| **`/lks/db/host`** | String | Writer endpoint Aurora (tanpa `:3306`) |
| **`/lks/db/port`** | String | `3306` |
| **`/lks/db/name`** | String | `leantime_db` |
| **`/lks/db/username`** | **SecureString** | `lksadmin` |
| **`/lks/db/password`** | **SecureString** | `<PasswordKuatAnda>` |

KMS key: default `alias/aws/ssm`.

> ✅ Username & password **harus SecureString**. Jika String biasa → **potongan poin keamanan**.

---

# FASE 6 — ECR

**ECR → Repositories → Create repository**

| Field | Nilai |
|---|---|
| Visibility | **Private** |
| Repository name | **`lks-gatus`** |
| Tag immutability | **Disable (Mutable)** ✅ |
| Scan on push | Enable (opsional) |
| Encryption | AES-256 default |

URI target:
```
<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/lks-gatus:<commit-sha>
<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/lks-gatus:latest
```

---

# FASE 7 — TARGET GROUP & APPLICATION LOAD BALANCER

## 7.1 Target Group Gatus
**EC2 → Target Groups → Create**

| Field | Nilai |
|---|---|
| Name | **`lks-gatus-app-tg`** |
| Target type | **IP addresses** |
| Protocol / port | HTTP / **8080** |
| Protocol version | **HTTP/1.1** |
| VPC | `lks-pm-vpc` |
| Health check protocol | HTTP |
| Health check path | **`/`** |
| Healthy threshold | **5** |
| Unhealthy threshold | **3** |
| Timeout | **60 detik** |
| Interval | **60 detik** |
| Success codes | **`200`** |

## 7.2 Target Group Leantime — ⚠️ JEBAKAN UTAMA

| Field | Nilai |
|---|---|
| Name | **`lks-leantime-app-tg`** |
| Target type | **IP addresses** |
| Protocol / port | HTTP / **8080** |
| Protocol version | **HTTP/1.1** |
| VPC | `lks-pm-vpc` |
| Health check protocol | HTTP |
| Health check path | **`/auth/login`** |
| Healthy threshold | **3** |
| Unhealthy threshold | **2** |
| Timeout | **30 detik** |
| Interval | **60 detik** |
| Success codes | **`200,302`** |

> **Kenapa?** Root `/` pada Leantime selalu **302 redirect → `/auth/login`**.
> Jika path `/` dengan success `200` saja → target **selalu UNHEALTHY** → ECS terus replace task → pipeline gagal.
> Pilihan alternatif yang juga valid: path `/` dengan success codes `200-399`. **Paling aman: `/auth/login` + `200,302`.**

## 7.3 ALB
**EC2 → Load Balancers → Create → Application Load Balancer**

| Field | Nilai |
|---|---|
| Name | **`lks-oss-lb`** |
| Scheme | **internet-facing** |
| IP type | **IPv4** |
| VPC | `lks-pm-vpc` |
| Mappings | ✅ 2 **public** subnet |
| Security group | **Existing → `lks-sg-alb`** |

**Listener 1**
```
HTTP :80  →  Forward to  lks-leantime-app-tg
```

**Listener 2** (Add listener)
```
HTTP :8080  →  Forward to  lks-gatus-app-tg
```

Catat **DNS name**:
```
lks-oss-lb-xxxxxx.us-east-1.elb.amazonaws.com
```

---

# FASE 8 — ECS CLUSTER

**ECS → Clusters → Create cluster**
- Cluster name: **`lks-oss-cluster`**
- Infrastructure: **AWS Fargate (Serverless)**
- **Create**

---

# FASE 9 — SOURCE CODE & TASK DEFINITION

## 9.1 `leantime/.env` (template — cocokkan key dengan repo)

```env
# === DATABASE ===
LEAN_DB_HOST=/lks/db/host
LEAN_DB_PORT=3306
LEAN_DB_DATABASE=leantime_db
LEAN_DB_USER=<diisi dari SSM secure>
LEAN_DB_PASSWORD=<diisi dari SSM secure>

# === APPLICATION ===
LEAN_APP_URL=http://lks-oss-lb-xxxxxx.us-east-1.elb.amazonaws.com
LEAN_APP_PORT=8080
LEAN_SITENAME=LKS Leantime
LEAN_TIMEZONE=Asia/Jakarta
LEAN_DEBUG=false

# === FILESYSTEM / EFS UID-GID ===
LEAN_FILE_UID=<dari repo>
LEAN_FILE_GID=<dari repo>
```

> 🔎 **WAJIB**: nama key di atas mengikuti konvensi umum Leantime (`LEAN_*`). **Key final = nama yang ada di `.env` sample repo**. Jangan kirim password literal ke Git — yang masuk ke task definition hanyalah **`secrets` → valueFrom SSM**.

### Environment yang masuk ke task definition

| Type | Name | Value / Source |
|---|---|---|
| environment | `LEAN_APP_URL` | `http://<ALB_DNS>` |
| environment | `LEAN_APP_PORT` | `8080` |
| environment | `LEAN_SITENAME` | `LKS Leantime` |
| environment | `LEAN_TIMEZONE` | `Asia/Jakarta` |
| environment | `LEAN_DEBUG` | **`false`** |
| environment | `LEAN_DB_PORT` | `3306` |
| **secrets** | `LEAN_DB_HOST` | `/lks/db/host` |
| **secrets** | `LEAN_DB_PORT` | `/lks/db/port` |
| **secrets** | `LEAN_DB_DATABASE` | `/lks/db/name` |
| **secrets** | `LEAN_DB_USER` | `/lks/db/username` |
| **secrets** | `LEAN_DB_PASSWORD` | `/lks/db/password` |

> Menaruh DB host/credentials sebagai **`secrets`** (bukan `environment`) adalah praktik yang dinilai positif oleh juri.

## 9.2 `leantime/task-definition.json`

```json
{
  "family": "lks-leantime-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "runtimePlatform": {
    "operatingSystemFamily": "LINUX",
    "cpuArchitecture": "X86_64"
  },
  "executionRoleArn": "arn:aws:iam::ACCOUNT_ID:role/LabRole",
  "taskRoleArn": "arn:aws:iam::ACCOUNT_ID:role/LabRole",
  "containerDefinitions": [
    {
      "name": "lks-leantime-container",
      "image": "leantime/leantime:latest",
      "essential": true,
      "portMappings": [
        { "name": "leantime", "containerPort": 8080, "hostPort": 8080, "protocol": "tcp" }
      ],
      "environment": [
        { "name": "LEAN_APP_PORT", "value": "8080" },
        { "name": "LEAN_SITENAME", "value": "LKS Leantime" },
        { "name": "LEAN_TIMEZONE", "value": "Asia/Jakarta" },
        { "name": "LEAN_DEBUG", "value": "false" }
      ],
      "secrets": [
        { "name": "LEAN_DB_HOST",     "valueFrom": "/lks/db/host" },
        { "name": "LEAN_DB_PORT",     "valueFrom": "/lks/db/port" },
        { "name": "LEAN_DB_DATABASE", "valueFrom": "/lks/db/name" },
        { "name": "LEAN_DB_USER",     "valueFrom": "/lks/db/username" },
        { "name": "LEAN_DB_PASSWORD", "valueFrom": "/lks/db/password" }
      ],
      "mountPoints": [
        { "sourceVolume": "public-userfiles", "containerPath": "/var/www/html/public/userfiles", "readOnly": false },
        { "sourceVolume": "userfiles",        "containerPath": "/var/www/html/userfiles",        "readOnly": false },
        { "sourceVolume": "plugins",          "containerPath": "/var/www/html/app/Plugins",      "readOnly": false }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/lks/leantime-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "leantime"
        }
      }
    }
  ],
  "volumes": [
    {
      "name": "public-userfiles",
      "efsVolumeConfiguration": {
        "fileSystemId": "FS_ID",
        "transitEncryption": "ENABLED",
        "authorizationConfig": { "accessPointId": "FSAP_PUBLIC", "iam": "ENABLED" }
      }
    },
    {
      "name": "userfiles",
      "efsVolumeConfiguration": {
        "fileSystemId": "FS_ID",
        "transitEncryption": "ENABLED",
        "authorizationConfig": { "accessPointId": "FSAP_USERFILES", "iam": "ENABLED" }
      }
    },
    {
      "name": "plugins",
      "efsVolumeConfiguration": {
        "fileSystemId": "FS_ID",
        "transitEncryption": "ENABLED",
        "authorizationConfig": { "accessPointId": "FSAP_PLUGINS", "iam": "ENABLED" }
      }
    }
  ]
}
```

**Placeholder yang harus diisi:** `ACCOUNT_ID`, `FS_ID`, `FSAP_PUBLIC`, `FSAP_USERFILES`, `FSAP_PLUGINS`, dan `LEAN_APP_URL` (atau di-inject pipeline).

> Jika image Leantime di repo menggunakan custom build, **ganti `image` sesuai repo** dan gunakan port 8080 sesuai instruksi.

## 9.3 `gatus/Dockerfile`

```dockerfile
FROM ghcr.io/twin/gatus:latest

# Config tanpa secret — nilai sensitif di-inject lewat env saat deploy
COPY config/config.yaml /app/config/config.yaml

EXPOSE 8080

ENTRYPOINT ["/app/gatus"]
CMD ["--config", "/app/config/config.yaml"]
```

## 9.4 `gatus/config/config.yaml` — ⚠️ TANPA TOKEN / URL HARDCODE

```yaml
web:
  address: 0.0.0.0
  port: 8080

debug: false

alerting:
  github:
    api-key: "${GITHUB_TOKEN}"
    repository: "${GITHUB_REPOSITORY}"
    issue-labels:
      - "gatus-alert"

endpoints:
  - name: leantime
    url: "${LEANTIME_URL}"
    interval: 30s
    conditions:
      - "[STATUS] == 200"
      - "[RESPONSE_TIME] < 300"
    alerts:
      - type: github
        failure-threshold: 3
        success-threshold: 5
        send-on-resolved: true
        description: "Leantime is down or unhealthy. Check ECS service and ALB target health."

  - name: gatus
    url: "${GATUS_URL}"
    interval: 1m
    conditions:
      - "[STATUS] == 200"
      - "[RESPONSE_TIME] < 500"
    alerts:
      - type: github
        failure-threshold: 3
        success-threshold: 5
        send-on-resolved: true
        description: "Gatus status page is down or unhealthy."
```

**Nilai env yang disuplai CI/CD (bukan disimpan di Git):**

| Env | Nilai |
|---|---|
| `GITHUB_TOKEN` | `${{ secrets.TOKEN }}` |
| `GITHUB_REPOSITORY` | `owner/lks-devops-repo` |
| `LEANTIME_URL` | `${ALB_ENDPOINT}` → `http://<ALB_DNS>` (port 80) |
| `GATUS_URL` | `${ALB_ENDPOINT}:8080` → `http://<ALB_DNS>:8080` |

## 9.5 `gatus/task-definition.json`

```json
{
  "family": "lks-gatus-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "runtimePlatform": {
    "operatingSystemFamily": "LINUX",
    "cpuArchitecture": "X86_64"
  },
  "executionRoleArn": "arn:aws:iam::ACCOUNT_ID:role/LabRole",
  "taskRoleArn": "arn:aws:iam::ACCOUNT_ID:role/LabRole",
  "containerDefinitions": [
    {
      "name": "lks-gatus-container",
      "image": "ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/lks-gatus:latest",
      "essential": true,
      "portMappings": [
        { "name": "gatus", "containerPort": 8080, "hostPort": 8080, "protocol": "tcp" }
      ],
      "environment": [
        { "name": "GATUS_WEB_PORT", "value": "8080" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/lks/gatus-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "gatus"
        }
      }
    }
  ]
}
```

> `LEANTIME_URL`, `GATUS_URL`, `GITHUB_TOKEN`, `GITHUB_REPOSITORY` **di-inject oleh GitHub Actions** saat render task definition — tidak pernah commit ke Git.

### 🔐 Opsi hardening (disarankan jika waktu cukup)
Alih-alih menaruh token di `environment` task definition, buat parameter tambahan:
```
/lks/gatus/github-token  → SecureString
```
lalu di task definition gunakan:
```json
"secrets": [
  { "name": "GITHUB_TOKEN", "valueFrom": "/lks/gatus/github-token" }
]
```
Ini menjaga token tidak tersimpan plaintext di revision task definition. **5 parameter wajib `/lks/db/*` tetap harus ada.**

---

# FASE 10 — ECS SERVICE

## 10.1 Service Leantime
**ECS → Cluster `lks-oss-cluster` → Services → Create**

| Field | Nilai |
|---|---|
| Compute / Capacity provider | **Fargate**, base **0**, weight **1**, platform **LATEST** |
| Task definition | **`lks-leantime-app` : latest revision** |
| Service name | **`leantime-service`** *(nama persis sesuai modul)* |
| Desired tasks | **1** (akan auto-scale) |
| VPC / Subnets | `lks-pm-vpc` / **2 private subnet** |
| Public IP | **Disable** ❌ |
| Security group | **`lks-sg-app`** |
| Load balancing | **Application Load Balancer** |
| ALB | `lks-oss-lb` |
| Container | `lks-leantime-container` : **8080** |
| Listener / TG | **80** → `lks-leantime-app-tg` |
| Health check grace period | **120–300 detik** |
| Deployment type | Rolling |

**Auto Scaling**
| Field | Nilai |
|---|---|
| Minimum tasks | **1** |
| Maximum tasks | **5** |
| Scaling type | **Target tracking** |
| Metric | **CPU utilization** |
| Target value | **75%** |
| Scale-out cooldown | **300 detik** |
| Scale-in cooldown | **300 detik** |

## 10.2 Service Gatus

| Field | Nilai |
|---|---|
| Capacity provider | Fargate, base **0**, weight **1**, LATEST |
| Task definition | **`lks-gatus-app` : latest** |
| Service name | **`gatus-service`** |
| Desired tasks | **2** |
| Subnets | 2 **private** |
| Public IP | **Disable** |
| SG | **`lks-sg-app`** |
| ALB | `lks-oss-lb` |
| Container | `lks-gatus-container` : **8080** |
| Listener / TG | **8080** → `lks-gatus-app-tg` |
| Grace period | **120 detik** |

### CloudWatch Log Group (buat manual agar pasti ada)
**CloudWatch → Log groups → Create log group**
```
/ecs/lks/leantime-app     (retention 7 days)
/ecs/lks/gatus-app        (retention 7 days)
```
Region: `us-east-1`. Tag `Project=LKS-DevOps`.

---

# FASE 11 — IAM LABROLE

**IAM → Roles → `LabRole` → Add permissions → Create inline policy**

Pastikan sudah ada **AWS managed**:
- `AmazonECSTaskExecutionRolePolicy`
- `AmazonECS_FullAccess` (atau izin ECS yang diperlukan)

Lalu tambahkan inline policy **`lks-devops-policy`**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRPullAndPush",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage"
      ],
      "Resource": "*"
    },
    {
      "Sid": "SSMSecrets",
      "Effect": "Allow",
      "Action": ["ssm:GetParameters", "ssm:GetParameter"],
      "Resource": "arn:aws:ssm:us-east-1:*:parameter/lks/*"
    },
    {
      "Sid": "KMSDecrypt",
      "Effect": "Allow",
      "Action": ["kms:Decrypt"],
      "Resource": "*"
    },
    {
      "Sid": "EFSMount",
      "Effect": "Allow",
      "Action": [
        "elasticfilesystem:ClientMount",
        "elasticfilesystem:ClientWrite",
        "elasticfilesystem:ClientRootAccess",
        "elasticfilesystem:DescribeMountTargets",
        "elasticfilesystem:DescribeFileSystems"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ECSDeploy",
      "Effect": "Allow",
      "Action": [
        "ecs:RegisterTaskDefinition",
        "ecs:DescribeTaskDefinition",
        "ecs:DescribeServices",
        "ecs:UpdateService",
        "ecs:DescribeClusters",
        "ecs:ListTasks",
        "ecs:DescribeTasks"
      ],
      "Resource": "*"
    },
    {
      "Sid": "PassRole",
      "Effect": "Allow",
      "Action": ["iam:PassRole"],
      "Resource": "arn:aws:iam::*:role/LabRole",
      "Condition": {
        "StringEquals": {
          "iam:PassedToService": ["ecs-tasks.amazonaws.com"]
        }
      }
    },
    {
      "Sid": "VPCInterfaces",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DeleteNetworkInterface",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeVpcs",
        "ec2:AssignPrivateIpAddresses",
        "ec2:UnassignPrivateIpAddresses"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Logs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
      ],
      "Resource": "arn:aws:logs:us-east-1:*:*"
    }
  ]
}
```

---

# FASE 12 — GITHUB TOKEN & SECRETS

## 12.1 Fine-Grained Personal Access Token
**GitHub → Settings → Developer settings → Fine-grained tokens → Generate new token**

| Field | Nilai |
|---|---|
| Token name | `lks-cicd-token` |
| Expiration | Sesuai kebutuhan lomba |
| Repository access | **Only select repositories** → repo Anda **SAJA** |
| Permissions → **Issues** | **Read and write** |
| Permissions → **Metadata** | **Read-only** |
| Permission lain | **Tidak perlu** |

Generate → **copy token segera**.

## 12.2 Repository Secrets
**Repo → Settings → Secrets and variables → Actions → New repository secret**

| Secret Name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Dari AWS Lab Details |
| `AWS_SECRET_ACCESS_KEY` | Dari AWS Lab Details |
| `AWS_SESSION_TOKEN` | Dari AWS Lab Details |
| `ALB_ENDPOINT` | `http://lks-oss-lb-xxxxxx.us-east-1.elb.amazonaws.com` |
| `TOKEN` | Fine-grained token |

> ⚠️ **Session token AWS Lab bersifat sementara.** Jika expired di tengah lomba → refresh lab/credential → **update secret** → re-run workflow.
>
> ⚠️ `ALB_ENDPOINT` **tanpa trailing slash**, karena pipeline akan menambahkan `:8080` untuk Gatus.

---

# FASE 13 — GITHUB ACTIONS WORKFLOW

## 13.1 Bersihkan workflow contoh
```bash
rm -f .github/workflows/example-*.yml
```
Yang tersisa **hanya**: `lks-deployment-pipeline.yml`.

## 13.2 `.github/workflows/lks-deployment-pipeline.yml`

```yaml
name: LKS Deployment Pipeline

on:
  push:
    branches: [main]
    paths:
      - 'leantime/**'
      - 'gatus/**'
      - '.github/workflows/lks-deployment-pipeline.yml'

env:
  AWS_REGION: us-east-1
  ECS_CLUSTER: lks-oss-cluster

jobs:
  # ============================================================
  # JOB 1 — DEPLOY LEANTIME (paralel dengan job Gatus)
  # ============================================================
  deploy-leantime:
    name: Deploy Leantime
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Resolve infrastructure values
        id: infra
        run: |
          FS_ID=$(aws efs describe-file-systems \
            --query "FileSystems[?Name=='lks-project-management'].FileSystemId" \
            --output text)
          FSAP_PUBLIC=$(aws efs describe-access-points \
            --query "AccessPoints[?Name=='lks-ap-public-userfiles'].AccessPointId" \
            --output text)
          FSAP_USER=$(aws efs describe-access-points \
            --query "AccessPoints[?Name=='lks-ap-userfiles'].AccessPointId" \
            --output text)
          FSAP_PLUGIN=$(aws efs describe-access-points \
            --query "AccessPoints[?Name=='lks-ap-plugins'].AccessPointId" \
            --output text)
          ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

          sed -i "s|ACCOUNT_ID|${ACCOUNT_ID}|g" leantime/task-definition.json
          sed -i "s|FS_ID|${FS_ID}|g"           leantime/task-definition.json
          sed -i "s|FSAP_PUBLIC|${FSAP_PUBLIC}|g" leantime/task-definition.json
          sed -i "s|FSAP_USERFILES|${FSAP_USER}|g" leantime/task-definition.json
          sed -i "s|FSAP_PLUGINS|${FSAP_PLUGIN}|g" leantime/task-definition.json

      - name: Render task definition
        id: render
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: leantime/task-definition.json
          container-name: lks-leantime-container
          image: leantime/leantime:latest
          environment-variables: |
            LEAN_APP_URL=${{ secrets.ALB_ENDPOINT }}
            LEAN_APP_PORT=8080
            LEAN_SITENAME=LKS Leantime
            LEAN_TIMEZONE=Asia/Jakarta
            LEAN_DEBUG=false

      - name: Deploy to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v2
        with:
          task-definition: ${{ steps.render.outputs.task-definition }}
          cluster: ${{ env.ECS_CLUSTER }}
          service: leantime-service
          wait-for-service-stability: true
          wait-for-minimum-stability: 1200

  # ============================================================
  # JOB 2 — BUILD + DEPLOY GATUS (digabung sesuai modul)
  # ============================================================
  build-and-deploy-gatus:
    name: Build and Deploy Gatus
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set image metadata
        id: meta
        run: |
          ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
          echo "registry=${{ steps.ecr.outputs.registry }}" >> $GITHUB_OUTPUT
          echo "account=${ACCOUNT_ID}" >> $GITHUB_OUTPUT
          echo "image=${{ steps.ecr.outputs.registry }}/lks-gatus" >> $GITHUB_OUTPUT

      - name: Build, tag and push to ECR
        run: |
          docker build \
            -t ${{ steps.meta.outputs.image }}:${{ github.sha }} \
            -t ${{ steps.meta.outputs.image }}:latest \
            ./gatus
          docker push ${{ steps.meta.outputs.image }}:${{ github.sha }}
          docker push ${{ steps.meta.outputs.image }}:latest

      - name: Resolve account id in task definition
        run: |
          sed -i "s|ACCOUNT_ID|${{ steps.meta.outputs.account }}|g" \
            gatus/task-definition.json

      - name: Render Gatus task definition with secrets from CI/CD
        id: render
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: gatus/task-definition.json
          container-name: lks-gatus-container
          image: ${{ steps.meta.outputs.image }}:${{ github.sha }}
          environment-variables: |
            LEANTIME_URL=${{ secrets.ALB_ENDPOINT }}
            GATUS_URL=${{ secrets.ALB_ENDPOINT }}:8080
            GITHUB_TOKEN=${{ secrets.TOKEN }}
            GITHUB_REPOSITORY=${{ github.repository }}

      - name: Deploy Gatus to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v2
        with:
          task-definition: ${{ steps.render.outputs.task-definition }}
          cluster: ${{ env.ECS_CLUSTER }}
          service: gatus-service
          wait-for-service-stability: true
          wait-for-minimum-stability: 900
```

> 📌 Jika template repo **memisahkan** `build-gatus` dan `deploy-gatus`, pecah job di atas menjadi dua dan hubungkan dengan `needs: build-gatus`. Logikanya identik; yang dinilai adalah **build → push ECR → register task def → update service** berjalan benar.

### 13.3 Commit & Push
```bash
git add .github/workflows/lks-deployment-pipeline.yml leantime/ gatus/
git commit -m "feat: deploy Leantime and Gatus with CI/CD"
git push origin main
```

---

# FASE 14 — JALANKAN & AWASI PIPELINE

**GitHub → Actions → LKS Deployment Pipeline**

Yang harus terlihat:
```
Deploy Leantime ────────────► ✅
Build and Deploy Gatus ─────► ✅
```

Checklist tiap run:
- [ ] `aws ecr login` sukses
- [ ] `docker push` menghasilkan 2 tag: `<commit-sha>` dan `latest`
- [ ] Task definition `lks-leantime-app` revision baru terdaftar
- [ ] Task definition `lks-gatus-app` revision baru terdaftar
- [ ] `aws ecs update-service` berhasil
- [ ] Service stabil (rolling deployment selesai)

**Verifikasi ECR**: Repositories → `lks-gatus` → Images → ada `latest` + SHA commit.

---

# FASE 15 — VERIFIKASI AKHIR (WAJIB DI-LAKUKAN, BUKAN DIASUMSIKAN)

## 15.1 Health Check Target
**EC2 → Target Groups**

| Target Group | Expected |
|---|---|
| `lks-leantime-app-tg` | **healthy**, minimal 1 target |
| `lks-gatus-app-tg` | **healthy**, 2 target |

Jika unhealthy → lihat Troubleshooting di bawah, **jangan langsung lanjut**.

## 15.2 Akses Leantime
```
http://lks-oss-lb-xxxxxx.us-east-1.elb.amazonaws.com
```
- Muncul **Leantime setup wizard**
- Buat admin user & password
- Setelah login: nama situs **`LKS Leantime`**, timezone **`Asia/Jakarta`**
- Buat 1 project + 1 task → bukti aplikasi benar-benar menulis ke Aurora

## 15.3 Uji EFS (bukti mount berfungsi)
1. Di Leantime → upload **logo / file lampiran**.
2. **EFS → Access points → `lks-ap-public-userfiles` / `lks-ap-userfiles`** → file muncul.
3. Restart task Leantime → **file tetap ada** (persisten).

## 15.4 Akses Gatus
```
http://lks-oss-lb-xxxxxx.us-east-1.elb.amazonaws.com:8080
```
Harus tampil **2 endpoint**:

| Endpoint | Interval | Conditions |
|---|---|---|
| `leantime` | **30s** | STATUS == 200, RESPONSE_TIME < **300ms** |
| `gatus` | **1m** | STATUS == 200, RESPONSE_TIME < **500ms** |

> 💡 Jika `RESPONSE_TIME < 300ms` sering gagal karena cold start, **jangan ubah syarat soal**. Perbaiki performa (task stabil, NAT sehat), atau biarkan beberapa siklus hingga stabil. Threshold ini adalah **syarat penilaian**.

## 15.5 Uji Alerting GitHub Issue
1. **Stop** service Leantime sementara (desired tasks = 0) — atau ubah `LEANTIME_URL` ke path salah sementara.
2. Tunggu **3 siklus gagal** (±90–120 detik).
3. **GitHub → Issues** → muncul issue baru berlabel `gatus-alert`.
4. Kembalikan service → tunggu **5 siklus sukses**.
5. Issue **tertutup otomatis** (`send-on-resolved: true`). ✅

## 15.6 Cek CloudWatch
```
/ecs/lks/leantime-app  → log Apache/PHP + DB connection OK
/ecs/lks/gatus-app     → log endpoint evaluation + alert dispatch
```

## 15.7 Cek Keamanan
- [ ] Aurora **Public access = No**
- [ ] EFS mount target di **private subnet**
- [ ] `lks-sg-app` inbound hanya dari `lks-sg-alb`
- [ ] `lks-sg-db` inbound hanya dari `lks-sg-app` :3306
- [ ] `lks-sg-efs` inbound hanya dari `lks-sg-app` :2049
- [ ] `LEAN_DEBUG=false`
- [ ] **Tidak ada** token/URL hardcoded di `config.yaml` atau Git history
- [ ] Fine-grained token hanya scoped ke 1 repo, hanya Issues RW + Metadata R
- [ ] Hanya `lks-deployment-pipeline.yml` di folder workflows

---

# 🛠️ TROUBLESHOOTING CEPAT

| Gejala | Penyebab | Solusi |
|---|---|---|
| Task `STOPPED`, `ResourceInitializationError` | Execution role kurang izin SSM/KMS/EFS/ECR | Cek inline policy Fase 11 + `AmazonECSTaskExecutionRolePolicy` |
| Task `CannotPullContainerError` | NAT mati / route private salah / ECR URI salah | Cek `0.0.0.0/0 → lks-pm-natgw`, SG outbound allow |
| Leantime target **UNHEALTHY** terus | Health check `/` + success 200 | Ganti `/auth/login` + `200,302` |
| Gatus target UNHEALTHY | Config YAML gagal parse / token kosong / port bukan 8080 | Cek log `/ecs/lks/gatus-app`, pastikan env ter-inject |
| Leantime **500 / DB connection failed** | SG DB salah, password SSM salah, atau nama DB beda | Cocokkan `/lks/db/*` dengan kredensial Aurora; cek `lks-sg-db` |
| EFS **Permission denied** | UID/GID access point ≠ UID proses container | Samakan dengan env repo, permission `764`, `readOnly=false` |
| EFS mount timeout | `lks-sg-efs` tidak allow 2049 dari `lks-sg-app` | Perbaiki inbound SG |
| GitHub Actions **AccessDenied** | Session token expired | Refresh credential Lab → update 3 secrets → re-run |
| ECR push **403** | `ecr:GetAuthorizationToken` Resource bukan `*` | Perbaiki policy |
| ECS update **PassRole denied** | Belum ada `iam:PassRole` | Tambahkan dengan condition `ecs-tasks.amazonaws.com` |
| Gatus tidak membuat Issue | Token scope Issues tidak RW / repo salah | Cek Fine-grained token + `GITHUB_REPOSITORY` (`owner/repo`) |
| ALB 503 | Belum ada healthy target | Tunggu grace period, cek SG app & health check |

---

# 🎯 CHECKLIST JURI / RUBRIK SELF-ASSESSMENT

| # | Kriteria | Poin fokus |
|---|---|---|
| 1 | Region `us-east-1` | 🔴 |
| 2 | `lks-pm-vpc` CIDR `10.100.0.0/16`, public `/10.100.0.0/24`, private `/10.100.10.0/24` | 🔴 |
| 3 | `lks-pm-igw`, `lks-pm-natgw` (1), `lks-pm-public-rt`, `lks-pm-private-rt` | 🔴 |
| 4 | 4 SG dengan least privilege, app hanya dari ALB | 🔴 |
| 5 | Aurora `lks-project-management-cluster`, MySQL 8.0, Production, Aurora Standard, `t4g.medium`, replica ON, no public, Insights Standard, Enhanced OFF | 🔴 |
| 6 | DB awal `leantime_db` | 🟠 |
| 7 | EFS `lks-project-management`, Regional, encrypted, General Purpose, Elastic | 🔴 |
| 8 | **3 access point**: `/public_userfiles`, `/userfiles`, `/plugins` + UID/GID benar + permission 764 | 🔴 |
| 9 | Mount container sesuai 3 path, `readOnly=false` | 🔴 |
| 10 | SSM `/lks/db/{host,port,name,username,password}`; username & password **SecureString** | 🔴 |
| 11 | ECR `lks-gatus` private, **Mutable** | 🟠 |
| 12 | TG `lks-gatus-app-tg`: IP, 8080, `/`, healthy 5 / unhealthy 3, timeout 60s, interval 60s, 200 | 🔴 |
| 13 | TG `lks-leantime-app-tg`: IP, 8080, `/auth/login`, healthy 3 / unhealthy 2, timeout 30s, interval 60s, **302 ditangani** | 🔴 |
| 14 | ALB `lks-oss-lb`, internet-facing, IPv4, listener 80 → Leantime, 8080 → Gatus | 🔴 |
| 15 | Cluster `lks-oss-cluster`, Fargate | 🔴 |
| 16 | Task `lks-leantime-app`: awsvpc, FARGATE, **1024 CPU / 2048 MB**, port 8080, log `/ecs/lks/leantime-app` | 🔴 |
| 17 | Task `lks-gatus-app`: awsvpc, FARGATE, **512 CPU / 1024 MB**, port 8080, log `/ecs/lks/gatus-app` | 🔴 |
| 18 | Leantime env: app URL benar, port 8080, sitename `LKS Leantime`, timezone `Asia/Jakarta`, **debug false** | 🟠 |
| 19 | Service `leantime-service`: private, no public IP, ALB, autoscale **1–5**, CPU **75%**, cooldown **300/300** | 🔴 |
| 20 | Service `gatus-service`: desired **2**, private, ALB | 🔴 |
| 21 | Fine-grained token: repo-scoped, Issues RW, Metadata R | 🔴 |
| 22 | 5 GitHub Secrets lengkap | 🔴 |
| 23 | `lks-deployment-pipeline.yml`, trigger `main` + path `leantime/`, `gatus/`, workflow file | 🔴 |
| 24 | Job paralel: Deploy Leantime + Build/Deploy Gatus | 🟠 |
| 25 | Image push dengan tag **commit SHA + latest** | 🟠 |
| 26 | Gatus: URL & token **tidak hardcoded**, injeksi via CI/CD | 🔴 |
| 27 | Gatus endpoint: 30s/300ms & 1m/500ms, threshold 3/5, send-on-resolved | 🔴 |
| 28 | GitHub Issue terbuka saat down & tertutup saat recover | 🔴 |
| 29 | Tag + description semua resource manual | 🟠 |
| 30 | Resource sampah & `example-*.yml` dihapus | 🟠 |

---

# 🧹 FASE 16 — CLEANUP (URUTAN PENTING)

1. **GitHub**: revoke Fine-grained token, hapus secrets.
2. **ECS**: autoscaling policy → delete → service `leantime-service` & `gatus-service` desired 0 → delete.
3. **Task definitions**: deregister semua revision `lks-leantime-app`, `lks-gatus-app`.
4. **ECS cluster** `lks-oss-cluster` → delete.
5. **ALB** `lks-oss-lb` → delete (tunggu selesai).
6. **Target groups** `lks-leantime-app-tg`, `lks-gatus-app-tg` → delete.
7. **ECR** `lks-gatus` → delete image + repository.
8. **RDS**: nonaktifkan deletion protection → delete cluster, **skip final snapshot** (lab).
9. **EFS**: delete 3 access point → delete mount target jika perlu → delete `lks-project-management`.
10. **SSM**: delete `/lks/db/*` (+ `/lks/gatus/github-token` jika dibuat).
11. **CloudWatch**: delete 2 log group.
12. **Security groups**: `lks-sg-alb`, `lks-sg-app`, `lks-sg-db`, `lks-sg-efs`.
13. **NAT Gateway** → delete → **Release Elastic IP**.
14. **VPC** `lks-pm-vpc` → delete (route table, subnet, IGW ikut).
15. **IAM**: hapus inline policy `lks-devops-policy` (jangan hapus LabRole bawaan lab).
16. Audit **Cost Explorer** → pastikan tidak ada resource tersisa.

---

## 💬 PENUTUP DARI JURI

> **Kunci nilai tinggi pada soal ini bukan kecepatan klik, melainkan tiga hal:**
> 1. **Health check Leantime yang memahami perilaku 302** — ini pembeda peserta paham vs peserta copy-paste.
> 2. **Secret hygiene** — kredensial di SSM SecureString, token Gatus di-inject CI/CD, tidak ada satu pun secret di Git.
> 3. **Pipeline yang benar-benar hijau dan aplikasi yang benar-benar berfungsi** — bukan hanya resource ter-create.
>
> Kuasai urutan **Saran 1 → 2 → 3 → 4**, lalu kerjakan panduan ini. Jika Saran 4 sudah lancar dalam ≤5 jam, target 3 jam pada soal asli sudah realistis.

Selamat berkompetisi. 🚀