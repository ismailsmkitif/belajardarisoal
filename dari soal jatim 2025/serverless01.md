# 📘 Panduan Lengkap Pengerjaan Saran 3 di AWS Lab Learner
## "Event RSVP & Feedback API + Simple Frontend"

> **Estimasi waktu**: 4 jam | **Region**: `us-west-2` | **Console-based** (tanpa CLI)

---

## 📋 Daftar Isi
1. [Fase 0: Persiapan](#fase-0-persiapan)
2. [Fase 1: Networking (VPC)](#fase-1-networking-vpc)
3. [Fase 2: Security Groups](#fase-2-security-groups)
4. [Fase 3: Database (RDS + DynamoDB)](#fase-3-database)
5. [Fase 4: SSM Parameter Store](#fase-4-ssm-parameter-store)
6. [Fase 5: Storage & Queue (S3 + SQS)](#fase-5-storage--queue)
7. [Fase 6: IAM Role & Lambda Layer](#fase-6-iam-role--lambda-layer)
8. [Fase 7: Lambda Functions](#fase-7-lambda-functions)
9. [Fase 8: API Gateway](#fase-8-api-gateway)
10. [Fase 9: Deploy Frontend](#fase-9-deploy-frontend)
11. [Fase 10: Testing End-to-End](#fase-10-testing)
12. [Fase 11: Cleanup](#fase-11-cleanup)

---

## 🎯 Fase 0: Persiapan

### Langkah 0.1: Login & Set Region
1. Login ke **AWS Lab Learner** → klik **Start Lab** → **Open AWS Console**.
2. Di pojok kanan atas, pastikan region = **US West (Oregon) `us-west-2`**.
3. Buka **Notepad** lokal untuk mencatat:
   - Account ID
   - LabRole ARN (akan dicari di Fase 6)
   - VPC ID (akan diisi di Fase 1)

### Langkah 0.2: Tagging Strategy
Setiap resource yang Anda buat, **WAJIB** isi:
- **Name tag**: `lks-<nama-resource>`
- **Description**: tujuan resource
- **Tags**: `Project=LKS-Serverless`, `Environment=Training`

---

## 🌐 Fase 1: Networking (VPC)

### Langkah 1.1: Buat VPC
1. Buka service **VPC** → **Your VPCs** → **Create VPC**.
2. Pilih **VPC and more**.
3. Konfigurasi:
   - Name tag: `lks-vpc`
   - IPv4 CIDR: `10.0.0.0/16`
   - Number of Availability Zones: **2**
   - Number of public subnets: **2**
   - Number of private subnets: **2**
   - ✅ Centang **NAT gateways (1 per AZ)** → akan mahal, tapi kita ganti nanti dengan VPC Endpoint
   - ✅ Centang **VPC endpoints** → pilih **S3 (Gateway)**
4. Klik **Create VPC**. Tunggu ~2 menit.

> 💡 **Catatan Trainer**: NAT Gateway akan kita hapus nanti untuk hemat biaya. Kita akan pakai **VPC Gateway Endpoint** (gratis) untuk S3 dan SSM.

### Langkah 1.2: Catat Resource yang Dibuat
Buka **VPC** → catat:
- **VPC ID**: `vpc-xxxxxxxx`
- **Public Subnet 1 ID**: `subnet-xxxxxxxx` (AZ a)
- **Private Subnet 1 ID**: `subnet-xxxxxxxx` (AZ a)
- **Private Subnet 2 ID**: `subnet-xxxxxxxx` (AZ b)

### Langkah 1.3: Tambah VPC Endpoint untuk SSM (Agar Lambda bisa baca Parameter Store)
1. **VPC** → **Endpoints** → **Create Endpoint**.
2. Service category: **AWS services**.
3. Cari: `com.amazonaws.us-west-2.ssm` → pilih.
4. VPC: `lks-vpc`.
5. Subnets: pilih **kedua private subnet**.
6. Security Group: pilih `lks-vpc-EndpointSG` (dibuat otomatis) atau buat baru.
7. Policy: **Full access** (default).
8. Name tag: `lks-endpoint-ssm`.
9. **Create**.

> Ulangi untuk endpoint `com.amazonaws.us-west-2.secretsmanager` (opsional) dan `com.amazonaws.us-west-2.dynamodb` (agar Lambda hemat biaya akses DynamoDB).

### Langkah 1.4: Hapus NAT Gateway (Hemat Biaya!)
> ⚠️ **HAPUS hanya jika Lambda tidak perlu akses internet**. Karena Lambda kita hanya akses S3/DynamoDB/RDS/SSM (semua via VPC Endpoint), NAT tidak diperlukan.
1. **VPC** → **NAT Gateways** → pilih NAT → **Actions** → **Delete NAT Gateway**.
2. Tunggu sampai status `Deleted`.

---

## 🔒 Fase 2: Security Groups

### Langkah 2.1: SG untuk RDS
1. **VPC** → **Security Groups** → **Create Security Group**.
2. Name: `lks-sg-rds`
3. Description: `Security group for RDS PostgreSQL`
4. VPC: `lks-vpc`
5. **Inbound rules**:
   | Type | Protocol | Port | Source |
   |---|---|---|---|
   | PostgreSQL | TCP | 5432 | Custom → `lks-sg-lambda` |
6. **Create**.

### Langkah 2.2: SG untuk Lambda
1. **Create Security Group** lagi.
2. Name: `lks-sg-lambda`
3. Description: `Security group for Lambda functions`
4. VPC: `lks-vpc`
5. **Inbound rules**: (kosongkan, Lambda tidak perlu inbound)
6. **Outbound rules**:
   | Type | Protocol | Port | Destination |
   |---|---|---|---|
   | All traffic | All | All | `0.0.0.0/0` |
7. **Create**.

### Langkah 2.3: SG untuk VPC Endpoint
1. Edit SG `lks-vpc-EndpointSG` (atau buat baru `lks-sg-endpoint`).
2. Inbound: HTTPS (443) dari `lks-vpc` CIDR `10.0.0.0/16`.

---

## 🗄️ Fase 3: Database

### Langkah 3.1: Buat RDS PostgreSQL
1. Buka service **RDS** → **Databases** → **Create database**.
2. Pilih **Standard create**.
3. Engine: **PostgreSQL**, Version: **16.x** (atau yang terbaru).
4. Templates: **Free tier** (db.t3.micro atau db.t4g.micro).
5. **Settings**:
   - DB instance identifier: `lks-rds-eventdb`
   - Master username: `lksadmin`
   - Master password: `LksPassword2026!` (catat ini!)
   - Confirm password.
6. **Connectivity**:
   - VPC: `lks-vpc`
   - Public access: **No** ❌
   - VPC security group: **Choose existing** → `lks-sg-rds` (hapus yang default)
   - Availability Zone: No preference
7. **Additional configuration**:
   - Initial database name: `eventdb`
8. **Create database**. Tunggu ~5 menit.

### Langkah 3.2: Catat Endpoint RDS
Setelah status `Available`:
- Klik DB → tab **Connectivity & security** → catat **Endpoint**:
  ```
  lks-rds-eventdb.xxxxx.us-west-2.rds.amazonaws.com
  ```

### Langkah 3.3: Buat DynamoDB Table
1. Buka **DynamoDB** → **Tables** → **Create table**.
2. Table name: `lks-feedback-db`
3. Partition key: `feedbackId` (String)
4. Settings:
   - Capacity mode: **On-demand** (lebih simple dari provisioned)
5. **Create table**.

---

## 🔑 Fase 4: SSM Parameter Store

### Langkah 4.1: Buat 4 Parameter
1. Buka **Systems Manager** → **Application Management** → **Parameter Store**.
2. **Create parameter** (ulangi 4x):

| Name | Type | Value |
|---|---|---|
| `/lks/eventdb/dbname` | String | `eventdb` |
| `/lks/eventdb/username` | String | `lksadmin` |
| `/lks/eventdb/password` | **SecureString** | `LksPassword2026!` |
| `/lks/eventdb/endpoint` | String | `lks-rds-eventdb.xxxxx.us-west-2.rds.amazonaws.com` |

3. Untuk `password`, pilih **KMS key source**: `aws/ssm` (default alias).

---

## 📦 Fase 5: Storage & Queue

### Langkah 5.1: S3 Bucket untuk Upload Gambar
1. Buka **S3** → **Create bucket**.
2. Bucket name: `lks-event-images-<namaAnda>` (harus unik global).
3. Region: `us-west-2`.
4. **Block Public Access**: ✅ ON (biarkan default).
5. **Bucket Versioning**: Disable.
6. **Create bucket**.
7. Masuk bucket → **Create folder** → nama: `feedback-img/`.

### Langkah 5.2: S3 Bucket untuk Frontend
1. **Create bucket** lagi.
2. Bucket name: `lks-frontend-<namaAnda>`.
3. **Block Public Access**: ❌ **OFF** (centang acknowledge warning).
4. **Create bucket**.
5. Masuk bucket → **Properties** → **Static website hosting** → **Edit**:
   - Enable
   - Index document: `index.html`
   - Error document: `index.html`
   - **Save**.
6. Catat **Bucket website endpoint**:
   ```
   http://lks-frontend-<namaAnda>.s3-website.us-west-2.amazonaws.com
   ```
7. **Permissions** → **Bucket policy** → paste:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadGetObject",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::lks-frontend-<namaAnda>/*"
  }]
}
```
*(ganti `<namaAnda>` dengan nama bucket Anda)*

### Langkah 5.3: SQS Queue + DLQ
1. Buka **SQS** → **Queues** → **Create queue**.
2. **Queue 1 - DLQ**:
   - Type: **Standard**
   - Name: `lks-queue-image-dlq`
   - Retention: 14 days (default)
   - **Create**.
   - Catat **ARN DLQ**: `arn:aws:sqs:us-west-2:XXXX:lks-queue-image-dlq`

3. **Queue 2 - Main**:
   - Type: **Standard**
   - Name: `lks-queue-image-process`
   - Visibility timeout: **30 seconds**
   - Receive message wait time: **10 seconds**
   - Message retention: **4 days**
   - Maximum message size: **256 KB**
   - **Dead-letter queue**: ✅ Enable
     - ARN DLQ: pilih `lks-queue-image-dlq`
     - Max receive count: **3**
   - **Create**.

---

## 👤 Fase 6: IAM Role & Lambda Layer

### Langkah 6.1: Cari LabRole ARN
1. Buka **IAM** → **Roles**.
2. Cari role `LabRole` (sudah disediakan AWS Lab Learner).
3. Catat **ARN**: `arn:aws:iam::XXXXXXXXXXXX:role/LabRole`.

### Langkah 6.2: Attach Policy ke LabRole
1. Klik `LabRole` → **Add permissions** → **Create inline policy**.
2. Pilih tab **JSON**, paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SSMRead",
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters",
        "kms:Decrypt"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3Access",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::lks-event-images-*",
        "arn:aws:s3:::lks-event-images-*/*"
      ]
    },
    {
      "Sid": "SQSAccess",
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:us-west-2:*:lks-queue-*"
    },
    {
      "Sid": "DynamoDBAccess",
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:Scan",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:us-west-2:*:table/lks-feedback-db"
    },
    {
      "Sid": "VPCAccess",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DeleteNetworkInterface"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:us-west-2:*:*"
    }
  ]
}
```

3. **Next** → Name: `lks-labrole-policy` → **Create policy**.

### Langkah 6.3: Buat Lambda Layer untuk `pg` Module
Karena Lambda Node.js runtime tidak include `pg`, kita buat layer via **CloudShell**:

1. Buka **CloudShell** (ikon `>_` di top bar AWS Console).
2. Tunggu terminal siap, lalu jalankan:

```bash
mkdir -p pg-layer/nodejs && cd pg-layer/nodejs
npm init -y
npm install pg
cd ..
zip -r pg-layer.zip nodejs
```

3. Upload ZIP ke S3 sementara:
```bash
aws s3 cp pg-layer.zip s3://lks-frontend-<namaAnda>/pg-layer.zip
```

4. Buka **Lambda** → **Layers** → **Create layer**.
   - Name: `lks-layer-pg`
   - Upload file dari **Amazon S3**:
     - URL: `https://s3.us-west-2.amazonaws.com/lks-frontend-<namaAnda>/pg-layer.zip`
   - Compatible runtimes: **Node.js 16.x** (atau 18.x/20.x)
   - **Create**.

---

## ⚡ Fase 7: Lambda Functions

### Langkah 7.1: Lambda #1 - `lks-rsvp-handler` (CRUD ke RDS)

1. **Lambda** → **Functions** → **Create function**.
2. **Author from scratch**:
   - Function name: `lks-rsvp-handler`
   - Runtime: **Node.js 18.x** (atau 16.x)
   - Architecture: x86_64
3. **Permissions** → Change default execution role → **Use an existing role** → pilih `LabRole`.
4. **Create function**.

5. **Configuration** → **General configuration** → **Edit**:
   - Timeout: **30 seconds**
   - Memory: **256 MB**

6. **Configuration** → **VPC** → **Edit**:
   - VPC: `lks-vpc`
   - Subnets: pilih **kedua private subnet**
   - Security groups: `lks-sg-lambda`
   - **Save**.

7. **Code** tab → **Layers** → **Add a layer** → **Custom layers**:
   - Pilih `lks-layer-pg` → **Add**.

8. **Code source** → ganti isi `index.js` dengan:

```javascript
const { Client } = require('pg');
const { SSMClient, GetParameterCommand } = require('@aws-sdk/client-ssm');

const ssm = new SSMClient({ region: 'us-west-2' });

async function getDbConfig() {
  const params = ['/lks/eventdb/dbname', '/lks/eventdb/username', '/lks/eventdb/password', '/lks/eventdb/endpoint'];
  const result = await ssm.send(new GetParameterCommand({
    Names: params,
    WithDecryption: true
  }));
  const map = {};
  result.Parameters.forEach(p => { map[p.Name] = p.Value; });
  return {
    host: map['/lks/eventdb/endpoint'],
    user: map['/lks/eventdb/username'],
    password: map['/lks/eventdb/password'],
    database: map['/lks/eventdb/dbname'],
    port: 5432
  };
}

async function initTable(client) {
  await client.query(`
    CREATE TABLE IF NOT EXISTS rsvp (
      id SERIAL PRIMARY KEY,
      name VARCHAR(100) NOT NULL,
      email VARCHAR(100) NOT NULL,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
  `);
}

exports.handler = async (event) => {
  const config = await getDbConfig();
  const client = new Client(config);
  await client.connect();
  await initTable(client);

  try {
    const method = event.httpMethod || event.requestContext?.http?.method;
    const body = event.body ? JSON.parse(event.body) : {};

    if (method === 'POST') {
      const { name, email } = body;
      if (!name || !email) {
        return { statusCode: 400, body: JSON.stringify({ error: 'name and email required' }) };
      }
      const res = await client.query(
        'INSERT INTO rsvp (name, email) VALUES ($1, $2) RETURNING *',
        [name, email]
      );
      return { statusCode: 201, body: JSON.stringify(res.rows[0]) };
    }

    if (method === 'GET') {
      const res = await client.query('SELECT * FROM rsvp ORDER BY created_at DESC LIMIT 50');
      return { statusCode: 200, body: JSON.stringify(res.rows) };
    }

    return { statusCode: 405, body: JSON.stringify({ error: 'Method not allowed' }) };
  } finally {
    await client.end();
  }
};
```

9. **Deploy** (tombol **Deploy** di atas editor).

### Langkah 7.2: Lambda #2 - `lks-feedback-text` (Tulis ke DynamoDB)

1. **Create function** baru:
   - Name: `lks-feedback-text`
   - Runtime: Node.js 18.x
   - Role: `LabRole`
   - **Create**.

2. **VPC**: **Tidak perlu** (DynamoDB via VPC Endpoint).

3. **Code** → ganti `index.js`:

```javascript
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, PutCommand, ScanCommand } = require('@aws-sdk/lib-dynamodb');

const ddb = DynamoDBDocumentClient.from(new DynamoDBClient({ region: 'us-west-2' }));
const TABLE = 'lks-feedback-db';

exports.handler = async (event) => {
  const method = event.httpMethod || event.requestContext?.http?.method;
  const body = event.body ? JSON.parse(event.body) : {};

  if (method === 'POST') {
    const item = {
      feedbackId: `fb-${Date.now()}-${Math.random().toString(36).slice(2, 8)}`,
      text: body.text || '',
      rating: body.rating || 5,
      createdAt: new Date().toISOString()
    };
    await ddb.send(new PutCommand({ TableName: TABLE, Item: item }));
    return { statusCode: 201, body: JSON.stringify(item) };
  }

  if (method === 'GET') {
    const res = await ddb.send(new ScanCommand({ TableName: TABLE }));
    return { statusCode: 200, body: JSON.stringify(res.Items || []) };
  }

  return { statusCode: 405, body: JSON.stringify({ error: 'Method not allowed' }) };
};
```

4. **Deploy**.

### Langkah 7.3: Lambda #3 - `lks-image-processor` (Konsumen SQS)

1. **Create function**:
   - Name: `lks-image-processor`
   - Runtime: Node.js 18.x
   - Role: `LabRole`
   - **Create**.

2. **VPC**: **Tidak perlu**.

3. **Triggers** → **Add trigger**:
   - Source: **SQS**
   - SQS queue: `lks-queue-image-process`
   - **Add**.

4. **Code** → ganti `index.js`:

```javascript
exports.handler = async (event) => {
  console.log('Received messages:', JSON.stringify(event, null, 2));

  for (const record of event.Records) {
    const body = JSON.parse(record.body);
    console.log(`Processing image: ${body.key || 'unknown'}`);
    // Di sini Anda bisa: resize image, simpan metadata ke DynamoDB, dll
    // Untuk latihan, cukup log saja.
  }

  return { statusCode: 200, body: 'OK' };
};
```

5. **Deploy**.

---

## 🚪 Fase 8: API Gateway

### Langkah 8.1: Buat REST API
1. Buka **API Gateway** → **Create API** → **REST API** → **New API**.
2. Name: `lks-event-api`
3. Endpoint type: **Regional**
4. **Create API**.

### Langkah 8.2: Buat Resource & Method
1. **Actions** → **Create Resource**:
   - Resource name: `rsvp`
   - Resource path: `rsvp`
   - ✅ Enable API Gateway CORS
   - **Create Resource**.

2. Klik resource `/rsvp` → **Actions** → **Create Method** → pilih **POST** → ✅.
3. Integration type: **Lambda Function**.
4. ✅ Use Lambda Proxy integration.
5. Lambda function: `lks-rsvp-handler`.
6. **Save** → OK (untuk add permission).

7. Ulangi untuk **GET** method pada `/rsvp` → integrasi ke `lks-rsvp-handler`.

8. **Create Resource** lagi: `feedback` → ✅ Enable CORS.
9. Di `/feedback` → **Create Method** → **POST** → Lambda `lks-feedback-text` (proxy).
10. Di `/feedback` → **Create Method** → **GET** → Lambda `lks-feedback-text` (proxy).

11. **Create Resource**: `upload` → ✅ Enable CORS.
12. Di `/upload` → **Create Method** → **POST** → Lambda `lks-image-processor` (proxy).

### Langkah 8.3: Enable CORS di Setiap Resource
Untuk setiap resource (`/rsvp`, `/feedback`, `/upload`):
1. Klik resource → **Actions** → **Enable CORS**.
2. Access-Control-Allow-Origin: `*`
3. Access-Control-Allow-Headers: `Content-Type,x-api-key`
4. Access-Control-Allow-Methods: `GET,POST,OPTIONS`
5. **Yes, replace existing values**.
6. **Deploy API** ke stage baru (nanti).

### Langkah 8.4: Buat API Key & Usage Plan
1. Di sidebar kiri → **API Keys** → **Create API Key**.
   - Name: `lks-apikey`
   - Auto generate: ✅
   - **Save**.
   - Catat **API key**: `abc123xxxx...`

2. **Usage Plans** → **Create**.
   - Name: `lks-usage-plan`
   - Rate: 100, Burst: 50
   - **Next** → **Add API Stage**:
     - API: `lks-event-api`
     - Stage: `prod` (akan dibuat saat deploy)
     - Throttle: default
   - **Next** → **Add API Key** → pilih `lks-apikey` → **Done**.

3. **Deploy API**:
   - **Actions** → **Deploy API**.
   - Deployment stage: **[New Stage]**
   - Stage name: `prod`
   - **Deploy**.
   - Catat **Invoke URL**: `https://xxxxx.execute-api.us-west-2.amazonaws.com/prod`.

### Langkah 8.5: Aktifkan API Key di Setiap Method
1. Klik method **POST /rsvp** → **Method Request**.
2. **Settings** → **API Key Required**: `true` → ✅ (save).
3. Ulangi untuk semua method (POST /feedback, GET /rsvp, GET /feedback, POST /upload).
4. **Deploy API** lagi ke stage `prod` (agar perubahan aktif).

---

## 🎨 Fase 9: Deploy Frontend

### Langkah 9.1: Siapkan 3 File Frontend

Buat 3 file di komputer lokal:

#### 📄 `index.html`
```html
<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8">
  <title>LKS Event RSVP</title>
  <script src="https://unpkg.com/vue@3/dist/vue.global.prod.js"></script>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <div id="app">
    <header>
      <h1>🎉 Event RSVP & Feedback</h1>
      <p class="subtitle">Aplikasi Latihan Serverless AWS</p>
    </header>

    <div class="container">
      <!-- Form RSVP -->
      <div class="card">
        <h2>📝 RSVP Kehadiran</h2>
        <input v-model="rsvp.name" placeholder="Nama Lengkap">
        <input v-model="rsvp.email" type="email" placeholder="Email">
        <button @click="submitRsvp" :disabled="loading">
          {{ loading ? 'Mengirim...' : 'Kirim RSVP' }}
        </button>
        <p v-if="message" class="success">{{ message }}</p>
      </div>

      <!-- Form Feedback -->
      <div class="card">
        <h2>💬 Feedback & Upload Foto</h2>
        <textarea v-model="feedback.text" placeholder="Kesan Anda tentang event..." rows="3"></textarea>
        <input type="file" @change="handleFile" accept="image/*">
        <button @click="submitFeedback" :disabled="loading">
          {{ loading ? 'Mengirim...' : 'Kirim Feedback' }}
        </button>
      </div>

      <!-- List RSVP -->
      <div class="card">
        <h2>👥 Daftar Tamu ({{ rsvpList.length }})</h2>
        <ul class="guest-list">
          <li v-for="r in rsvpList" :key="r.id">
            <strong>{{ r.name }}</strong> — {{ r.email }}
          </li>
        </ul>
        <button @click="loadRsvp" class="secondary">🔄 Refresh</button>
      </div>
    </div>
  </div>
  <script src="app.js"></script>
</body>
</html>
```

#### 📄 `app.js`
```javascript
// GANTI 2 nilai di bawah dengan milik Anda!
const API_URL = 'https://XXXXXX.execute-api.us-west-2.amazonaws.com/prod';
const API_KEY = 'PASTE_API_KEY_ANDA_DI_SINI';

const { createApp, ref, onMounted } = Vue;

createApp({
  setup() {
    const rsvp = ref({ name: '', email: '' });
    const feedback = ref({ text: '', file: null });
    const rsvpList = ref([]);
    const loading = ref(false);
    const message = ref('');

    const headers = {
      'Content-Type': 'application/json',
      'x-api-key': API_KEY
    };

    const submitRsvp = async () => {
      if (!rsvp.value.name || !rsvp.value.email) {
        alert('Nama dan email wajib diisi!');
        return;
      }
      loading.value = true;
      try {
        const res = await fetch(`${API_URL}/rsvp`, {
          method: 'POST',
          headers,
          body: JSON.stringify(rsvp.value)
        });
        if (res.ok) {
          message.value = '✅ RSVP berhasil dikirim!';
          rsvp.value = { name: '', email: '' };
          loadRsvp();
          setTimeout(() => message.value = '', 3000);
        } else {
          alert('Error: ' + await res.text());
        }
      } catch (e) {
        alert('Network error: ' + e.message);
      } finally {
        loading.value = false;
      }
    };

    const submitFeedback = async () => {
      if (!feedback.value.text) {
        alert('Feedback wajib diisi!');
        return;
      }
      loading.value = true;
      try {
        // Kirim teks ke DynamoDB
        await fetch(`${API_URL}/feedback`, {
          method: 'POST',
          headers,
          body: JSON.stringify({ text: feedback.value.text, rating: 5 })
        });

        // Jika ada file, upload via /upload endpoint
        if (feedback.value.file) {
          const reader = new FileReader();
          reader.onload = async () => {
            const base64 = reader.result.split(',')[1];
            await fetch(`${API_URL}/upload`, {
              method: 'POST',
              headers: { 'Content-Type': 'application/json', 'x-api-key': API_KEY },
              body: JSON.stringify({
                filename: feedback.value.file.name,
                content: base64
              })
            });
          };
          reader.readAsDataURL(feedback.value.file);
        }

        alert('✅ Feedback terkirim!');
        feedback.value = { text: '', file: null };
      } catch (e) {
        alert('Error: ' + e.message);
      } finally {
        loading.value = false;
      }
    };

    const loadRsvp = async () => {
      try {
        const res = await fetch(`${API_URL}/rsvp`, {
          method: 'GET',
          headers
        });
        if (res.ok) {
          rsvpList.value = await res.json();
        }
      } catch (e) {
        console.error(e);
      }
    };

    const handleFile = (e) => {
      feedback.value.file = e.target.files[0];
    };

    onMounted(loadRsvp);

    return { rsvp, feedback, rsvpList, loading, message, submitRsvp, submitFeedback, loadRsvp, handleFile };
  }
}).mount('#app');
```

#### 📄 `style.css`
```css
* { box-sizing: border-box; margin: 0; padding: 0; }
body {
  font-family: 'Segoe UI', Arial, sans-serif;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  min-height: 100vh;
  padding: 20px;
  color: #333;
}
header { text-align: center; color: white; margin-bottom: 30px; }
header h1 { font-size: 2.5em; margin-bottom: 8px; }
.subtitle { opacity: 0.9; font-size: 1.1em; }
.container { max-width: 700px; margin: 0 auto; }
.card {
  background: white;
  padding: 25px;
  margin-bottom: 20px;
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0,0,0,0.1);
}
.card h2 { margin-bottom: 15px; color: #667eea; }
input, textarea, button {
  display: block;
  width: 100%;
  padding: 10px 12px;
  margin: 8px 0;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 14px;
}
input:focus, textarea:focus { outline: none; border-color: #667eea; }
button {
  background: #667eea;
  color: white;
  border: none;
  cursor: pointer;
  font-weight: 600;
  transition: background 0.2s;
}
button:hover:not(:disabled) { background: #5568d3; }
button:disabled { opacity: 0.6; cursor: not-allowed; }
button.secondary { background: #eee; color: #333; }
.success { color: #28a745; font-weight: 600; margin-top: 10px; }
.guest-list { list-style: none; max-height: 300px; overflow-y: auto; }
.guest-list li {
  padding: 10px;
  border-bottom: 1px solid #eee;
}
.guest-list li:last-child { border-bottom: none; }
```

### Langkah 9.2: Upload ke S3 Frontend Bucket
1. Buka bucket `lks-frontend-<namaAnda>` → **Upload**.
2. Drag & drop 3 file → **Upload**.

### Langkah 9.3: Update API_URL & API_KEY di `app.js`
1. Edit `app.js` lokal → ganti 2 konstanta di baris atas:
   ```javascript
   const API_URL = 'https://xxxxxx.execute-api.us-west-2.amazonaws.com/prod';
   const API_KEY = 'abc123xxxx...';
   ```
2. Upload ulang `app.js` ke S3 (overwrite).

---

## 🧪 Fase 10: Testing End-to-End

### Test 1: Buka Frontend
```
http://lks-frontend-<namaAnda>.s3-website.us-west-2.amazonaws.com
```
Harus muncul halaman RSVP.

### Test 2: Submit RSVP
1. Isi nama & email → klik **Kirim RSVP**.
2. Harusnya muncul pesan "✅ RSVP berhasil dikirim!".
3. Klik **Refresh** → nama Anda muncul di list.

### Test 3: Cek Database RDS
1. Buka **RDS** → **Query Builder** (atau pakai **EC2** + psql jika ada).
2. Atau lihat di **CloudWatch Logs** → `/aws/lambda/lks-rsvp-handler`.

### Test 4: Submit Feedback
1. Isi feedback → klik **Kirim Feedback**.
2. Cek **DynamoDB** → table `lks-feedback-db` → **Explore items** → harus ada data baru.

### Test 5: Cek SQS
1. Buka **SQS** → `lks-queue-image-process` → **Send and receive messages**.
2. Atau cek **CloudWatch Logs** → `/aws/lambda/lks-image-processor`.

---

## 🧹 Fase 11: Cleanup

> ⚠️ **PENTING**: Hapus semua resource untuk menghindari tagihan!

Urutan penghapusan:
1. **API Gateway** → Delete API `lks-event-api`.
2. **Lambda** → Delete 3 functions.
3. **SQS** → Delete 2 queues.
4. **S3** → Empty & delete 2 buckets.
5. **DynamoDB** → Delete table `lks-feedback-db`.
6. **RDS** → Delete DB instance (❌ disable snapshot).
7. **SSM Parameter Store** → Delete 4 parameters.
8. **Lambda Layer** → Delete `lks-layer-pg`.
9. **VPC** → Delete VPC `lks-vpc` (akan hapus subnet, route table, SG, endpoints otomatis).
10. **IAM** → Delete inline policy `lks-labrole-policy` dari LabRole.

---

## 🎓 Checklist Juri (Self-Assessment)

| # | Item | Status |
|---|---|---|
| 1 | Semua resource ber-prefix `lks-` | ☐ |
| 2 | Semua resource di-tag & ada deskripsi | ☐ |
| 3 | RDS tidak publicly accessible | ☐ |
| 4 | RDS hanya bisa diakses dari SG Lambda | ☐ |
| 5 | Password disimpan di SSM SecureString | ☐ |
| 6 | Lambda ambil config dari SSM (bukan hardcode) | ☐ |
| 7 | SQS punya DLQ | ☐ |
| 8 | API Gateway pakai API Key | ☐ |
| 9 | CORS enabled di API Gateway | ☐ |
| 10 | Frontend bisa diakses publik via S3 | ☐ |
| 11 | End-to-end test berhasil | ☐ |
| 12 | Environment bersih (tidak ada resource sampah) | ☐ |

---

## 💡 Tips Trainer

1. **Jika Lambda timeout saat connect RDS**: Cek Security Group RDS (harus izinkan dari SG Lambda) dan pastikan Lambda di private subnet yang sama VPC.
2. **Jika CORS error di browser**: Pastikan Anda sudah **Deploy API** setelah enable CORS.
3. **Jika "Missing Authentication Token"**: API Key belum aktif di method tersebut, atau header `x-api-key` salah.
4. **Jika Lambda tidak bisa baca SSM**: Pastikan VPC Endpoint untuk SSM sudah dibuat.

Selamat berlatih! Jika Saran 3 ini lancar, Anda siap 70% untuk soal LKS sesungguhnya. 🚀