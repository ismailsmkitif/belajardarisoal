# 📘 Panduan Lengkap Pengerjaan Saran 1 (Machine Learning)
## "Serverless Image Analyzer" — S3 → Lambda → Rekognition → DynamoDB

> **Estimasi waktu**: 1-1.5 jam | **Region**: `us-east-1` | **Console-based** (tanpa Terraform)
> **Tujuan**: Memahami logika bisnis ML pipeline sebelum dibungkus IaC di Saran selanjutnya.

---

## 🎯 Tujuan Latihan

Membangun **pipeline pemrosesan gambar otomatis** yang:
- ✅ Menerima upload gambar di S3 (`technoinput-*`)
- ✅ Memicu Lambda secara otomatis (event-driven)
- ✅ Memanggil **Amazon Rekognition** untuk deteksi objek/label
- ✅ Menyimpan hasil analisis ke **DynamoDB**
- ✅ Bisa di-query untuk melihat hasil analisis

Ini adalah **fondasi logika bisnis** dari soal LKS Machine Learning. Jika Anda paham alur ini, soal sesungguhnya hanya soal menambahkan kompleksitas (Terraform, EKS, Glue, Kinesis).

---

## 🗺️ Topologi Arsitektur

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        AWS CLOUD (us-east-1)                            │
│                                                                         │
│   ┌──────────────────┐         ┌──────────────────────────────────┐    │
│   │   USER / CLIENT  │         │      AWS LAMBDA                  │    │
│   │                  │         │   techno-lambda-s3               │    │
│   │  Upload Image    │         │                                  │    │
│   │  (PutObject)     │         │  1. Triggered by S3 event        │    │
│   └────────┬─────────┘         │  2. Call Rekognition             │    │
│            │                   │     → DetectLabels API           │    │
│            │                   │  3. Save result to DynamoDB      │    │
│            ▼                   │                                  │    │
│   ┌──────────────────┐         │  Runtime: Python 3.12            │    │
│   │  Amazon S3       │         │  Timeout: 2 minutes              │    │
│   │  technoinput-    │────────▶│  Role: LabRole                   │    │
│   │  jakarta-budi    │  Event  │                                  │    │
│   │                  │  Notif  └──────┬───────────┬───────────────┘    │
│   └──────────────────┘                │           │                    │
│                                       │           │                    │
│                          ┌────────────▼──┐   ┌────▼────────────────┐   │
│                          │  Amazon       │   │  Amazon DynamoDB    │   │
│                          │  Rekognition  │   │  Table: Tokens      │   │
│                          │               │   │  PK: token (S)      │   │
│                          │  DetectLabels │   │                     │   │
│                          │  API          │   │  Attributes:        │   │
│                          │               │   │  - token            │   │
│                          └───────────────┘   │  - image_name       │   │
│                                              │  - labels (JSON)    │   │
│                                              │  - timestamp        │   │
│                                              └─────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 🔄 Alur Kerja (Flow)
```
1. User upload gambar → S3 bucket "technoinput-jakarta-budi"
                          ↓
2. S3 Event Notification → trigger Lambda "techno-lambda-s3"
                          ↓
3. Lambda baca object dari S3 → panggil Rekognition DetectLabels
                          ↓
4. Rekognition return list of labels (dengan confidence score)
                          ↓
5. Lambda simpan hasil ke DynamoDB table "Tokens"
                          ↓
6. User bisa query DynamoDB untuk melihat hasil analisis
```

---

## 📋 Daftar Isi
1. [Fase 0: Persiapan](#fase-0-persiapan)
2. [Fase 1: IAM Role Preparation](#fase-1-iam)
3. [Fase 2: DynamoDB Table](#fase-2-dynamodb)
4. [Fase 3: S3 Bucket](#fase-3-s3)
5. [Fase 4: Lambda Function](#fase-4-lambda)
6. [Fase 5: S3 Event Trigger](#fase-5-trigger)
7. [Fase 6: Testing End-to-End](#fase-6-testing)
8. [Fase 7: Cleanup](#fase-7-cleanup)

---

## 🎯 Fase 0: Persiapan

### Langkah 0.1: Login & Set Region
1. Login ke **AWS Lab Learner** → klik **Start Lab** → **Open AWS Console**.
2. Di pojok kanan atas, pastikan region = **US East (N. Virginia) `us-east-1`**.
3. Buka **Notepad** lokal untuk mencatat:
   - **Account ID** (12 digit)
   - **Nama Anda** (untuk naming convention)
   - **Kota Anda** (untuk naming convention)

### Langkah 0.2: Tentukan Naming Convention
Sesuai soal LKS, semua resource WAJIB prefix **`techno-`**:

| Resource | Nama |
|---|---|
| S3 Bucket (input) | `technoinput-<kota>-<nama>` (contoh: `technoinput-jakarta-budi`) |
| DynamoDB Table | `Tokens` |
| Lambda Function | `techno-lambda-s3` |

> 💡 **Catatan Trainer**: Di soal sesungguhnya, Anda juga butuh bucket output (`technooutput-*`), tapi untuk Saran 1 kita fokus ke alur dasar dulu.

### Langkah 0.3: Siapkan Gambar Test
Download 1 gambar test ke komputer lokal (contoh: foto kucing, mobil, atau pemandangan). Simpan dengan nama sederhana: `test-image.jpg`.

---

## 👤 Fase 1: IAM Role Preparation

### Langkah 1.1: Cek LabRole
1. Buka **IAM** → **Roles** → cari `LabRole` (sudah disediakan oleh AWS Lab Learner).
2. Catat **ARN**: `arn:aws:iam::XXXXXXXXXXXX:role/LabRole`.

### Langkah 1.2: Tambah Inline Policy untuk Rekognition & DynamoDB
1. Klik `LabRole` → **Add permissions** → **Create inline policy**.
2. Pilih tab **JSON**, paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3ReadAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::technoinput-*/*"
    },
    {
      "Sid": "RekognitionAccess",
      "Effect": "Allow",
      "Action": [
        "rekognition:DetectLabels",
        "rekognition:DetectFaces",
        "rekognition:DetectText"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DynamoDBAccess",
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:UpdateItem",
        "dynamodb:Query",
        "dynamodb:Scan"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:*:table/Tokens"
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:us-east-1:*:*"
    }
  ]
}
```

3. **Next** → Name: `techno-ml-policy` → **Create policy**.

> ⚠️ **PENTING**: Tanpa policy ini, Lambda tidak bisa panggil Rekognition atau tulis ke DynamoDB.

---

## 🗄️ Fase 2: DynamoDB Table

### Langkah 2.1: Buat Table
1. Buka **DynamoDB** → **Tables** → **Create table**.
2. Konfigurasi:
   - **Table name**: `Tokens` (persis seperti soal)
   - **Partition key**: `token` (String)
   - **Sort key**: (kosongkan)
3. **Settings**:
   - Capacity mode: **On-demand** (lebih simple untuk latihan)
   - Encryption: default AWS-owned key
4. **Create table**.

### Langkah 2.2: Verifikasi
- Status table harus **Active**.
- Catat ARN: `arn:aws:dynamodb:us-east-1:XXXX:table/Tokens`.

---

## 📦 Fase 3: S3 Bucket

### Langkah 3.1: Buat Bucket Input
1. Buka **S3** → **Create bucket**.
2. Konfigurasi:
   - **Bucket name**: `technoinput-<kota>-<nama>` (contoh: `technoinput-jakarta-budi`)
     > ⚠️ Nama bucket harus **unik global** dan **lowercase**.
   - **Region**: `us-east-1`
   - **Block Public Access**: ✅ ON (biarkan default, aman)
   - **Bucket Versioning**: Disable
   - **Encryption**: SSE-S3 (default)
3. **Create bucket**.

### Langkah 3.2: Verifikasi
- Bucket sudah terbuat dan kosong.
- Catat nama bucket untuk dipakai di Lambda & trigger.

---

## ⚡ Fase 4: Lambda Function

### Langkah 4.1: Buat Lambda Function
1. Buka **Lambda** → **Functions** → **Create function**.
2. **Author from scratch**:
   - **Function name**: `techno-lambda-s3`
   - **Runtime**: **Python 3.12** (atau 3.11/3.13 jika tersedia)
   - **Architecture**: x86_64
3. **Permissions**:
   - Change default execution role → **Use an existing role**
   - Pilih: `LabRole`
4. **Basic settings** (setelah function dibuat):
   - **Timeout**: **2 minutes** (sesuai soal)
   - **Memory**: 256 MB
5. **Create function**.

### Langkah 4.2: Tulis Kode Lambda

Di tab **Code**, ganti isi `lambda_function.py` dengan:

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
        
        # 2. Generate token unik untuk identifikasi hasil
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
        
        # Simpan error ke DynamoDB untuk audit
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

### Langkah 4.3: Deploy
Klik tombol **Deploy** di atas editor untuk menyimpan perubahan.

### Langkah 4.4: Test Manual (Opsional)
Sebelum setup trigger, Anda bisa test Lambda dengan event S3 dummy:

1. Tab **Test** → **Create new event**.
2. Event name: `test-s3-event`.
3. Paste event JSON ini (ganti bucket & key):

```json
{
  "Records": [
    {
      "s3": {
        "bucket": {
          "name": "technoinput-jakarta-budi"
        },
        "object": {
          "key": "test-image.jpg"
        }
      }
    }
  ]
}
```

4. Klik **Test** → lihat **Execution result** di bawah.
5. Cek **CloudWatch Logs** untuk melihat detail proses.

> ⚠️ Test ini akan gagal jika gambar belum di-upload ke S3. Ini normal. Lanjut ke Fase 5.

---

## 🔗 Fase 5: S3 Event Trigger

### Langkah 5.1: Tambah Trigger ke Lambda
1. Di Lambda function `techno-lambda-s3`, klik **Add trigger**.
2. Pilih source: **S3**.
3. Konfigurasi:
   - **Bucket**: `technoinput-<kota>-<nama>` (pilih bucket Anda)
   - **Event types**: ✅ **All object create events**
   - **Prefix**: (kosongkan, atau isi `images/` jika ingin spesifik folder)
   - **Suffix**: `.jpg` atau `.jpeg` atau `.png` (opsional, untuk filter file)
   - ✅ **Recursive invocation**: Disable (biarkan default)
4. Klik **Add**.

### Langkah 5.2: Verifikasi Trigger
- Di tab **Function overview**, harus ada kotak **S3** terhubung ke Lambda.
- Cek di S3 bucket → tab **Properties** → **Event notifications** → harus ada event baru.

---

## 🧪 Fase 6: Testing End-to-End

### Test 1: Upload Gambar ke S3
1. Buka bucket `technoinput-<kota>-<nama>`.
2. Klik **Upload** → pilih `test-image.jpg` dari komputer.
3. **Upload**.

### Test 2: Cek Lambda Execution
1. Buka Lambda `techno-lambda-s3` → tab **Monitor** → **Logs**.
2. Klik log stream terbaru.
3. Harus muncul log seperti:
   ```
   Processing image: s3://technoinput-jakarta-budi/test-image.jpg
   Detected 12 labels
   Successfully saved to DynamoDB with token: token-1730000000-test-image
   ```

### Test 3: Cek DynamoDB
1. Buka **DynamoDB** → **Tables** → `Tokens` → **Explore items**.
2. Harus ada item baru dengan:
   - `token`: `token-xxxxx-test-image`
   - `image_name`: `test-image.jpg`
   - `labels`: JSON array berisi hasil Rekognition
   - `label_count`: jumlah label terdeteksi
   - `status`: `SUCCESS`

### Test 4: Lihat Hasil Rekognition
Klik item → lihat atribut `labels` → harus berisi JSON seperti:
```json
[
  {
    "Name": "Cat",
    "Confidence": 98.5,
    "Categories": ["Animals and Pets"]
  },
  {
    "Name": "Animal",
    "Confidence": 98.5,
    "Categories": ["Animals and Pets"]
  },
  ...
]
```

### Test 5: Upload Gambar Berbeda
Upload gambar lain (misal: foto mobil, pemandangan) → ulangi Test 1-4.
Hasil labels akan berbeda sesuai konten gambar.

### Test 6: Cek CloudWatch Logs
1. **CloudWatch** → **Log groups** → `/aws/lambda/techno-lambda-s3`.
2. Klik log stream → harus ada log lengkap proses.

---

## 🧹 Fase 7: Cleanup

> ⚠️ **PENTING**: Hapus semua resource untuk menghindari tagihan & menjaga environment bersih.

### Urutan Penghapusan:
1. **S3 Bucket** → Empty (hapus semua object) → Delete bucket.
2. **Lambda Function** → Delete `techno-lambda-s3`.
3. **CloudWatch Log Group** → `/aws/lambda/techno-lambda-s3` → Delete.
4. **DynamoDB Table** → Delete `Tokens`.
5. **IAM Inline Policy** → `techno-ml-policy` → Delete dari LabRole.

---

## 🎓 Checklist Juri (Self-Assessment)

| # | Item | Status |
|---|---|---|
| 1 | Region: `us-east-1` | ☐ |
| 2 | Bucket name sesuai format `technoinput-<kota>-<nama>` | ☐ |
| 3 | DynamoDB table name: `Tokens` dengan partition key `token` (String) | ☐ |
| 4 | Lambda name: `techno-lambda-s3` | ☐ |
| 5 | Lambda runtime: Python 3.12 (atau 3.13) | ☐ |
| 6 | Lambda timeout: 2 menit | ☐ |
| 7 | Lambda role: LabRole | ☐ |
| 8 | IAM policy mengizinkan S3 GetObject, Rekognition, DynamoDB | ☐ |
| 9 | S3 trigger aktif untuk "All object create events" | ☐ |
| 10 | Upload gambar → Lambda ter-invoke otomatis | ☐ |
| 11 | Rekognition berhasil deteksi labels | ☐ |
| 12 | Hasil tersimpan di DynamoDB dengan status SUCCESS | ☐ |
| 13 | CloudWatch logs aktif & berisi log proses | ☐ |
| 14 | Cleanup selesai | ☐ |

---

## 💡 Tips Trainer

### 🔴 Jika Lambda Error "AccessDenied" saat panggil Rekognition
**Penyebab**: IAM role tidak punya izin `rekognition:DetectLabels`.
**Solusi**: Cek inline policy `techno-ml-policy` sudah di-attach ke LabRole.

### 🔴 Jika Lambda Error "AccessDenied" saat tulis ke DynamoDB
**Penyebab**: IAM role tidak punya izin `dynamodb:PutItem`.
**Solusi**: Pastikan Resource di policy = ARN table `Tokens` yang benar.

### 🔴 Jika Lambda Tidak Ter-trigger Setelah Upload
**Penyebab**: Trigger belum di-setup atau format file tidak cocok.
**Solusi**:
1. Cek tab **Triggers** di Lambda → harus ada S3 trigger.
2. Jika pakai suffix filter (misal `.jpg`), pastikan file berekstensi `.jpg`.

### 🔴 Jika Rekognition Return 0 Labels
**Penyebab**: Gambar terlalu kecil, blur, atau format tidak didukung.
**Solusi**:
1. Gunakan gambar JPG/PNG dengan resolusi minimal 100x100 px.
2. Turunkan `MinConfidence` di kode (misal dari 70.0 ke 50.0).

### 🔴 Jika DynamoDB Item Tidak Muncul
**Penyebab**: Table name salah di kode atau IAM policy salah resource.
**Solusi**:
1. Cek variabel `DYNAMODB_TABLE = 'Tokens'` di kode Lambda.
2. Pastikan ARN di IAM policy = `arn:aws:dynamodb:us-east-1:*:table/Tokens`.

---

## 📊 Apa yang Anda Pelajari dari Saran 1?

| Skill | Status |
|---|---|
| S3 Event-Driven Architecture | ✅ |
| Lambda Python runtime | ✅ |
| Amazon Rekognition DetectLabels API | ✅ |
| DynamoDB PutItem dari Lambda | ✅ |
| IAM least privilege (S3 + Rekognition + DynamoDB) | ✅ |
| S3 → Lambda trigger configuration | ✅ |
| CloudWatch Logs debugging | ✅ |
| Error handling di Lambda | ✅ |

---

## 🚀 Progression Path

```
✅ Saran 1 (Console, S3 → Lambda → Rekognition → DynamoDB) — 1-1.5 jam ← ANDA DI SINI
    ↓
🟡 Saran 2 (Terraform + DynamoDB Streams → Kinesis) — 2 jam
    ↓
🟠 Saran 3 (Terraform + API Gateway + EKS Deployment) — 2.5 jam
    ↓
🔴 Saran 4 (Full: + Glue Crawler, SNS, IPv6, Strict Naming) — 4-5 jam
    ↓
🏆 SOAL LKS Machine Learning (6 jam, Terraform Wajib)
```

---

## 🎯 Analogi Langsung ke Soal LKS

| Yang Anda Lakukan di Saran 1 | Yang Akan Anda Lakukan di Soal LKS |
|---|---|
| Console manual | **Terraform** (wajib) |
| 1 bucket input | **2 bucket** (input + output) |
| DynamoDB on-demand | DynamoDB dengan **Streams → Kinesis** |
| Lambda panggil Rekognition | Lambda panggil Rekognition + kirim ke **SNS** |
| Tidak ada API | **API Gateway** (REST, dual-stack IPv4/IPv6) |
| Tidak ada container | **EKS** dengan Python app + LoadBalancer |
| Tidak ada analytics | **AWS Glue** (Database + Table + Crawler) |
| Prefix `techno-` sederhana | Prefix `techno-` **ketat** untuk semua resource |

---

## 🎯 Next Step

Setelah Saran 1 berhasil, Anda siap lanjut ke **Saran 2**: membungkus semua resource ini ke dalam **Terraform** + menambahkan **DynamoDB Streams → Kinesis Data Stream**.

Apakah Anda ingin saya buatkan:
1. **Panduan lengkap Saran 2** (Terraform + DynamoDB Streams → Kinesis)?
2. **Template `main.tf`** untuk Saran 1 (sebagai fondasi IaC)?
3. **Variasi test** dengan gambar berbeda untuk melihat kemampuan Rekognition?

Silakan pilih, dan saya akan buatkan! 🎯

**Selamat! Anda telah menyelesaikan fondasi logika bisnis dari soal LKS Machine Learning.** 🎉