# 📘 Panduan Lengkap Pengerjaan Saran 1 di AWS Lab Learner
## "Digital Guestbook" (Buku Tamu Digital) — Versi Menengah

> **Estimasi waktu**: 6-8 jam | **Region**: `us-west-2` | **Console-based** (tanpa CLI)

---

## 📋 Daftar Isi
1. [Fase 0: Persiapan](#fase-0-persiapan)
2. [Fase 1: Networking (VPC)](#fase-1-networking-vpc)
3. [Fase 2: Security Groups](#fase-2-security-groups)
4. [Fase 3: Database (RDS + DynamoDB)](#fase-3-database)
5. [Fase 4: SSM Parameter Store](#fase-4-ssm-parameter-store)
6. [Fase 5: Storage (S3 + Lifecycle + Event)](#fase-5-storage)
7. [Fase 6: Queue (SQS + DLQ)](#fase-6-queue)
8. [Fase 7: IAM & Lambda Layer](#fase-7-iam--lambda-layer)
9. [Fase 8: Lambda Functions](#fase-8-lambda-functions)
10. [Fase 9: REST API Gateway](#fase-9-rest-api-gateway)
11. [Fase 10: WebSocket API](#fase-10-websocket-api)
12. [Fase 11: Deploy Frontend via Amplify](#fase-11-deploy-frontend-via-amplify)
13. [Fase 12: Testing End-to-End](#fase-12-testing)
14. [Fase 13: Cleanup](#fase-13-cleanup)

---

## 🎯 Fase 0: Persiapan

### Langkah 0.1: Login & Set Region
1. Login ke **AWS Lab Learner** → klik **Start Lab** → **Open AWS Console**.
2. Di pojok kanan atas, pastikan region = **US West (Oregon) `us-west-2`**.
3. Buka **Notepad** lokal untuk mencatat:
   - Account ID
   - LabRole ARN
   - VPC ID, Subnet IDs
   - API Gateway URL
   - API Key
   - WebSocket URL

### Langkah 0.2: Tagging Strategy
Setiap resource yang Anda buat, **WAJIB** isi:
- **Name tag**: `lks-<nama-resource>`
- **Description**: tujuan resource
- **Tags**: `Project=LKS-Guestbook`, `Environment=Training`

### Langkah 0.3: Siapkan Frontend Code Lokal
Buat folder baru di komputer Anda: `lks-guestbook-frontend/`. Kita akan butuh 3 file yang akan saya berikan di Fase 11.

---

## 🌐 Fase 1: Networking (VPC)

### Langkah 1.1: Buat VPC dengan Wizard
1. Buka service **VPC** → **Your VPCs** → **Create VPC**.
2. Pilih **VPC and more**.
3. Konfigurasi:
   - Name tag: `lks-vpc`
   - IPv4 CIDR: `10.0.0.0/16`
   - Number of Availability Zones: **2**
   - Number of public subnets: **2**
   - Number of private subnets: **2**
   - ✅ Centang **NAT gateways (1)** → kita akan hapus nanti
   - ✅ Centang **VPC endpoints** → pilih **S3 (Gateway)**
4. Klik **Create VPC**. Tunggu ~2 menit.

### Langkah 1.2: Catat Resource IDs
Buka **VPC** → catat:
- **VPC ID**: `vpc-xxxxxxxx`
- **Public Subnet 1**: `subnet-pub1-xxxx` (AZ a)
- **Public Subnet 2**: `subnet-pub2-xxxx` (AZ b)
- **Private Subnet 1**: `subnet-priv1-xxxx` (AZ a)
- **Private Subnet 2**: `subnet-priv2-xxxx` (AZ b)

### Langkah 1.3: Tambah VPC Endpoint untuk SSM
1. **VPC** → **Endpoints** → **Create Endpoint**.
2. Service: `com.amazonaws.us-west-2.ssm`
3. VPC: `lks-vpc`
4. Subnets: **kedua private subnet**
5. Security Group: buat baru `lks-sg-endpoint`
6. Name tag: `lks-endpoint-ssm`
7. **Create**.

### Langkah 1.4: Tambah VPC Endpoint untuk DynamoDB
1. Ulangi langkah 1.3, tapi service: `com.amazonaws.us-west-2.dynamodb`
2. Type: **Gateway**
3. Name tag: `lks-endpoint-dynamodb`

### Langkah 1.5: Hapus NAT Gateway (Hemat Biaya!)
> Karena Lambda hanya akses S3/DynamoDB/RDS/SSM (semua via VPC Endpoint), NAT tidak diperlukan.
1. **VPC** → **NAT Gateways** → pilih NAT → **Delete**.
2. Tunggu sampai status `Deleted`.
3. **Release** Elastic IP yang terkait (agar tidak ditagih).

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
1. **Create Security Group**.
2. Name: `lks-sg-lambda`
3. Description: `Security group for Lambda functions`
4. VPC: `lks-vpc`
5. **Inbound rules**: (kosongkan)
6. **Outbound rules**:
   | Type | Protocol | Port | Destination |
   |---|---|---|---|
   | All traffic | All | All | `0.0.0.0/0` |
7. **Create**.

### Langkah 2.3: SG untuk VPC Endpoint
1. Edit SG `lks-sg-endpoint` yang dibuat di Fase 1.3.
2. **Inbound rules**:
   | Type | Protocol | Port | Source |
   |---|---|---|---|
   | HTTPS | TCP | 443 | Custom → `10.0.0.0/16` |

---

## 🗄️ Fase 3: Database

### Langkah 3.1: Buat RDS PostgreSQL
1. Buka **RDS** → **Databases** → **Create database**.
2. Pilih **Standard create**.
3. Engine: **PostgreSQL**, Version: **16.x** (atau terbaru).
4. Templates: **Free tier** (db.t3.micro).
5. **Settings**:
   - DB instance identifier: `lks-rds-guestbook`
   - Master username: `lksadmin`
   - Master password: `LksPassword2026!` (catat!)
6. **Connectivity**:
   - VPC: `lks-vpc`
   - Public access: **No** ❌
   - VPC security group: `lks-sg-rds` (hapus default)
7. **Additional configuration**:
   - Initial database name: `guestbookdb`
8. **Create database**. Tunggu ~5 menit.

### Langkah 3.2: Catat Endpoint RDS
Setelah status `Available`:
- Klik DB → tab **Connectivity & security** → catat **Endpoint**:
  ```
  lks-rds-guestbook.xxxxx.us-west-2.rds.amazonaws.com
  ```

### Langkah 3.3: Buat DynamoDB Table dengan Auto Scaling
1. Buka **DynamoDB** → **Tables** → **Create table**.
2. Table name: `lks-rating-stats`
3. Partition key: `eventDate` (String)
4. **Capacity mode**: **Provisioned**
5. Read capacity: **10**
6. Write capacity: **5**
7. ✅ Centang **Use auto scaling untuk read capacity**:
   - Min: 10, Max: 100
   - Target utilization: **70%**
8. ✅ Centang **Use auto scaling untuk write capacity**:
   - Min: 5, Max: 25
   - Target utilization: **70%**
9. **Create table**.

---

## 🔑 Fase 4: SSM Parameter Store

### Langkah 4.1: Buat 4 Parameter
1. Buka **Systems Manager** → **Parameter Store** → **Create parameter** (ulangi 4x):

| Name | Type | Value |
|---|---|---|
| `/lks/guestbook/dbname` | String | `guestbookdb` |
| `/lks/guestbook/username` | String | `lksadmin` |
| `/lks/guestbook/password` | **SecureString** | `LksPassword2026!` |
| `/lks/guestbook/endpoint` | String | `lks-rds-guestbook.xxxxx.us-west-2.rds.amazonaws.com` |

---

## 📦 Fase 5: Storage (S3 + Lifecycle + Event)

### Langkah 5.1: S3 Bucket untuk Foto Bukti
1. Buka **S3** → **Create bucket**.
2. Bucket name: `lks-guestbook-photos-<namaAnda>` (harus unik global).
3. Region: `us-west-2`.
4. **Block Public Access**: ✅ ON.
5. **Create bucket**.
6. Masuk bucket → **Create folder** → nama: `bukti-kehadiran/`.

### Langkah 5.2: Setup Lifecycle Rule
1. Di bucket → tab **Management** → **Lifecycle rules** → **Create lifecycle rule**.
2. Rule name: `lks-lifecycle-photos`
3. Apply to: **All objects in the bucket** (atau prefix `bukti-kehadiran/`).
4. **Add transition**:
   - Transition to **Glacier Deep Archive** after **180 days** (6 bulan).
5. **Add expiration**:
   - Expire current version after **365 days** (1 tahun).
6. **Save**.

### Langkah 5.3: Setup S3 Event Notification ke SQS
> Kita akan setup ini setelah SQS dibuat di Fase 6.

### Langkah 5.4: S3 Bucket untuk Frontend (Amplify akan handle, skip ini)
> Amplify akan membuat bucket sendiri secara otomatis.

---

## 📬 Fase 6: Queue (SQS + DLQ)

### Langkah 6.1: Buat DLQ
1. Buka **SQS** → **Queues** → **Create queue**.
2. Type: **Standard**
3. Name: `lks-queue-notif-dlq`
4. Retention: 14 days
5. **Create**.
6. Catat **ARN DLQ**: `arn:aws:sqs:us-west-2:XXXX:lks-queue-notif-dlq`

### Langkah 6.2: Buat Main Queue
1. **Create queue** lagi.
2. Type: **Standard**
3. Name: `lks-queue-notif`
4. Visibility timeout: **30 seconds**
5. Receive message wait time: **10 seconds**
6. Message retention: **4 days**
7. Maximum message size: **256 KB**
8. **Dead-letter queue**: ✅ Enable
   - ARN DLQ: pilih `lks-queue-notif-dlq`
   - Max receive count: **3**
9. **Create**.
10. Catat **ARN Main Queue**.

### Langkah 6.3: Setup S3 Event Notification
1. Buka bucket `lks-guestbook-photos-<namaAnda>` → **Properties** → **Event notifications** → **Create event notification**.
2. Event name: `lks-s3-photo-upload`
3. Prefix: `bukti-kehadiran/`
4. Event types: ✅ **All object create events** (PUT, POST, COPY)
5. Destination: **SQS Queue**
6. SQS queue: **Enter SQS ARN** → paste ARN `lks-queue-notif`
7. **Save changes**.

> ⚠️ Jika muncul error "SQS queue policy doesn't allow S3 to write", klik **Add permission to SQS queue** → AWS akan otomatis menambahkan policy.

---

## 👤 Fase 7: IAM & Lambda Layer

### Langkah 7.1: Cari LabRole ARN
1. Buka **IAM** → **Roles** → cari `LabRole`.
2. Catat **ARN**: `arn:aws:iam::XXXXXXXXXXXX:role/LabRole`.

### Langkah 7.2: Attach Policy ke LabRole
1. Klik `LabRole` → **Add permissions** → **Create inline policy**.
2. Tab **JSON**, paste:

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
        "arn:aws:s3:::lks-guestbook-photos-*",
        "arn:aws:s3:::lks-guestbook-photos-*/*"
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
        "dynamodb:UpdateItem",
        "dynamodb:Scan",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:us-west-2:*:table/lks-*"
    },
    {
      "Sid": "ApiGatewayManage",
      "Effect": "Allow",
      "Action": [
        "execute-api:ManageConnections"
      ],
      "Resource": "arn:aws:execute-api:us-west-2:*:*/*"
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

3. **Next** → Name: `lks-guestbook-policy` → **Create policy**.

### Langkah 7.3: Buat Lambda Layer untuk `pg` Module
1. Buka **CloudShell** (ikon `>_` di top bar).
2. Jalankan:

```bash
mkdir -p pg-layer/nodejs && cd pg-layer/nodejs
npm init -y
npm install pg
cd ..
zip -r pg-layer.zip nodejs
aws s3 cp pg-layer.zip s3://lks-guestbook-photos-<namaAnda>/pg-layer.zip
```

3. Buka **Lambda** → **Layers** → **Create layer**.
   - Name: `lks-layer-pg`
   - Upload from S3: `https://s3.us-west-2.amazonaws.com/lks-guestbook-photos-<namaAnda>/pg-layer.zip`
   - Compatible runtimes: **Node.js 18.x**
   - **Create**.

---

## ⚡ Fase 8: Lambda Functions

### Langkah 8.1: Lambda #1 - `lks-guestbook-crud` (CRUD ke RDS)

1. **Lambda** → **Functions** → **Create function**.
2. Name: `lks-guestbook-crud`
3. Runtime: **Node.js 18.x**
4. Role: `LabRole`
5. **Create function**.

6. **Configuration** → **General configuration**:
   - Timeout: **30 seconds**
   - Memory: **256 MB**

7. **Configuration** → **VPC**:
   - VPC: `lks-vpc`
   - Subnets: **kedua private subnet**
   - Security groups: `lks-sg-lambda`
   - **Save**.

8. **Code** → **Layers** → **Add a layer** → Custom layers → `lks-layer-pg` → **Add**.

9. **Code source** → ganti `index.js`:

```javascript
const { Client } = require('pg');
const { SSMClient, GetParametersCommand } = require('@aws-sdk/client-ssm');

const ssm = new SSMClient({ region: 'us-west-2' });

async function getDbConfig() {
  const names = [
    '/lks/guestbook/dbname',
    '/lks/guestbook/username',
    '/lks/guestbook/password',
    '/lks/guestbook/endpoint'
  ];
  const result = await ssm.send(new GetParametersCommand({
    Names: names,
    WithDecryption: true
  }));
  const map = {};
  result.Parameters.forEach(p => { map[p.Name] = p.Value; });
  return {
    host: map['/lks/guestbook/endpoint'],
    user: map['/lks/guestbook/username'],
    password: map['/lks/guestbook/password'],
    database: map['/lks/guestbook/dbname'],
    port: 5432
  };
}

async function initTable(client) {
  await client.query(`
    CREATE TABLE IF NOT EXISTS guestbook (
      id SERIAL PRIMARY KEY,
      name VARCHAR(100) NOT NULL,
      message TEXT NOT NULL,
      photo_url TEXT,
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
    const body = event.body ? (event.isBase64Encoded 
      ? JSON.parse(Buffer.from(event.body, 'base64').toString())
      : JSON.parse(event.body)) : {};

    if (method === 'POST') {
      const { name, message, photoUrl } = body;
      if (!name || !message) {
        return { statusCode: 400, body: JSON.stringify({ error: 'name and message required' }) };
      }
      const res = await client.query(
        'INSERT INTO guestbook (name, message, photo_url) VALUES ($1, $2, $3) RETURNING *',
        [name, message, photoUrl || null]
      );
      return { 
        statusCode: 201, 
        headers: { 'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*' },
        body: JSON.stringify(res.rows[0]) 
      };
    }

    if (method === 'GET') {
      const res = await client.query('SELECT * FROM guestbook ORDER BY created_at DESC LIMIT 50');
      return { 
        statusCode: 200, 
        headers: { 'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*' },
        body: JSON.stringify(res.rows) 
      };
    }

    return { statusCode: 405, body: JSON.stringify({ error: 'Method not allowed' }) };
  } finally {
    await client.end();
  }
};
```

10. **Deploy**.

### Langkah 8.2: Lambda #2 - `lks-process-photo` (Konsumen SQS)

1. **Create function**:
   - Name: `lks-process-photo`
   - Runtime: Node.js 18.x
   - Role: `LabRole`
   - **Create**.

2. **VPC**: Tidak perlu (hanya akses DynamoDB via VPC Endpoint).

3. **Triggers** → **Add trigger**:
   - Source: **SQS**
   - SQS queue: `lks-queue-notif`
   - **Add**.

4. **Code** → ganti `index.js`:

```javascript
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, UpdateCommand } = require('@aws-sdk/lib-dynamodb');

const ddb = DynamoDBDocumentClient.from(new DynamoDBClient({ region: 'us-west-2' }));

exports.handler = async (event) => {
  console.log('Received messages:', JSON.stringify(event, null, 2));

  for (const record of event.Records) {
    try {
      const s3Event = JSON.parse(record.body);
      const bucket = s3Event.Records[0].s3.bucket.name;
      const key = decodeURIComponent(s3Event.Records[0].s3.object.key.replace(/\+/g, ' '));
      const photoUrl = `https://${bucket}.s3.amazonaws.com/${key}`;
      
      console.log(`Processing photo: ${photoUrl}`);

      // Update rating stats di DynamoDB
      const today = new Date().toISOString().split('T')[0];
      await ddb.send(new UpdateCommand({
        TableName: 'lks-rating-stats',
        Key: { eventDate: today },
        UpdateExpression: 'SET photoCount = if_not_exists(photoCount, :zero) + :inc',
        ExpressionAttributeValues: {
          ':zero': 0,
          ':inc': 1
        }
      }));

      console.log(`Updated stats for ${today}`);
    } catch (err) {
      console.error('Error processing record:', err);
      throw err; // akan masuk DLQ setelah 3x retry
    }
  }

  return { statusCode: 200, body: 'OK' };
};
```

5. **Deploy**.

### Langkah 8.3: Lambda #3 - `lks-ws-handler` (WebSocket Handler)

1. **Create function**:
   - Name: `lks-ws-handler`
   - Runtime: Node.js 18.x
   - Role: `LabRole`
   - **Create**.

2. **VPC**: Tidak perlu.

3. **Code** → ganti `index.js`:

```javascript
const { ApiGatewayManagementApiClient, PostToConnectionCommand, DeleteConnectionCommand } = require('@aws-sdk/client-apigatewaymanagementapi');

exports.handler = async (event) => {
  console.log('WebSocket event:', JSON.stringify(event, null, 2));

  const routeKey = event.requestContext.routeKey;
  const connectionId = event.requestContext.connectionId;
  const domain = event.requestContext.domainName;
  const stage = event.requestContext.stage;

  const client = new ApiGatewayManagementApiClient({
    endpoint: `https://${domain}/${stage}`
  });

  try {
    if (routeKey === '$connect') {
      console.log(`Client connected: ${connectionId}`);
      return { statusCode: 200, body: 'Connected' };
    }

    if (routeKey === '$disconnect') {
      console.log(`Client disconnected: ${connectionId}`);
      return { statusCode: 200, body: 'Disconnected' };
    }

    if (routeKey === 'sendMessage') {
      const body = JSON.parse(event.body || '{}');
      const message = body.message || 'Hello from server!';
      
      await client.send(new PostToConnectionCommand({
        ConnectionId: connectionId,
        Data: JSON.stringify({ type: 'message', content: message })
      }));
      return { statusCode: 200, body: 'Message sent' };
    }

    if (routeKey === 'broadcastMessage') {
      const body = JSON.parse(event.body || '{}');
      const message = body.message || 'Broadcast!';
      
      // Untuk broadcast, kita perlu simpan connectionId di DynamoDB
      // Untuk latihan, cukup kirim balik ke pengirim
      await client.send(new PostToConnectionCommand({
        ConnectionId: connectionId,
        Data: JSON.stringify({ type: 'broadcast', content: message })
      }));
      return { statusCode: 200, body: 'Broadcast sent' };
    }

    if (routeKey === 'getConnectionId') {
      return { 
        statusCode: 200, 
        body: JSON.stringify({ connectionId }) 
      };
    }

    return { statusCode: 200, body: 'Default handler' };
  } catch (err) {
    console.error('Error:', err);
    if (err.statusCode === 410) {
      // Connection gone, cleanup
      await client.send(new DeleteConnectionCommand({ ConnectionId: connectionId }));
    }
    return { statusCode: 500, body: 'Error: ' + err.message };
  }
};
```

4. **Deploy**.

---

## 🚪 Fase 9: REST API Gateway

### Langkah 9.1: Buat REST API
1. Buka **API Gateway** → **Create API** → **REST API** → **New API**.
2. Name: `lks-guestbook-api`
3. Endpoint type: **Regional**
4. **Create API**.

### Langkah 9.2: Buat Resources & Methods

#### Resource `/message`
1. **Actions** → **Create Resource**:
   - Name: `message`, Path: `message`
   - ✅ Enable API Gateway CORS
   - **Create Resource**.
2. Klik `/message` → **Create Method** → **POST** → ✅.
   - Integration: **Lambda Function** → `lks-guestbook-crud` → ✅ Proxy integration.
   - **Save** → OK.
3. Klik `/message` → **Create Method** → **GET** → ✅.
   - Integration: Lambda `lks-guestbook-crud` → ✅ Proxy.
   - **Save** → OK.

#### Resource `/rating`
1. **Create Resource**: `rating`, Path: `rating` → ✅ CORS.
2. Klik `/rating` → **Create Method** → **GET** → ✅.
   - Integration: Lambda `lks-guestbook-crud` → tapi kita perlu Lambda terpisah untuk rating.
   - **Solusi cepat**: Buat Lambda `lks-rating-handler` (copy dari `lks-guestbook-crud`, ubah query ke DynamoDB).

> 💡 **Alternatif sederhana**: Gabungkan logic rating di Lambda `lks-guestbook-crud` dengan cek path.

3. **Create Method** → **GET** di `/rating` → Lambda `lks-guestbook-crud`.

### Langkah 9.3: Enable CORS
Untuk setiap resource (`/message`, `/rating`):
1. Klik resource → **Actions** → **Enable CORS**.
2. Access-Control-Allow-Origin: `*`
3. Access-Control-Allow-Headers: `Content-Type,x-api-key`
4. Access-Control-Allow-Methods: `GET,POST,OPTIONS`
5. **Yes, replace**.

### Langkah 9.4: Buat API Key & Usage Plan
1. **API Keys** → **Create API Key**:
   - Name: `lks-apikey`
   - Auto generate: ✅
   - **Save**. Catat API key.

2. **Usage Plans** → **Create**:
   - Name: `lks-usage-plan`
   - Rate: 100, Burst: 50
   - **Next** → Add API Stage: `lks-guestbook-api` → stage `prod` (akan dibuat).
   - **Next** → Add API Key `lks-apikey` → **Done**.

3. **Deploy API**:
   - **Actions** → **Deploy API** → New Stage → name: `prod` → **Deploy**.
   - Catat **Invoke URL**: `https://xxxxx.execute-api.us-west-2.amazonaws.com/prod`.

### Langkah 9.5: Aktifkan API Key
1. Klik method **POST /message** → **Method Request** → API Key Required: `true` → Save.
2. Ulangi untuk semua method.
3. **Deploy API** lagi ke stage `prod`.

---

## 🔌 Fase 10: WebSocket API

### Langkah 10.1: Buat WebSocket API
1. **API Gateway** → **Create API** → **WebSocket** → **Build**.
2. Name: `lks-guestbook-ws`
3. Route selection expression: `$request.body.action`
4. **Next** → **Add route**:

#### Route `$connect`
1. Route key: `$connect`
2. **Add integration** → Lambda `lks-ws-handler` → **Next** → **Create method**.
3. **Next** → **Deploy** → Stage name: `prod` → **Deploy**.
4. Catat **WebSocket URL**: `wss://xxxxx.execute-api.us-west-2.amazonaws.com/prod`.

#### Route `$disconnect`
1. **Actions** → **Create route** → `$disconnect`.
2. Integration: Lambda `lks-ws-handler`.
3. **Deploy API** ke stage `prod`.

#### Route `sendMessage`
1. **Create route** → `sendMessage`.
2. Integration: Lambda `lks-ws-handler`.
3. **Deploy**.

#### Route `broadcastMessage`
1. **Create route** → `broadcastMessage`.
2. Integration: Lambda `lks-ws-handler`.
3. **Deploy**.

#### Route `getConnectionId`
1. **Create route** → `getConnectionId`.
2. Integration: Lambda `lks-ws-handler`.
3. **Deploy**.

#### Route `$default`
1. **Create route** → `$default`.
2. Integration: Lambda `lks-ws-handler`.
3. **Deploy**.

---

## 🎨 Fase 11: Deploy Frontend via Amplify

### Langkah 11.1: Siapkan 3 File Frontend

Buat 3 file di folder `lks-guestbook-frontend/`:

#### 📄 `index.html`
```html
<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Digital Guestbook - LKS Serverless</title>
  <script src="https://unpkg.com/vue@3/dist/vue.global.prod.js"></script>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <div id="app">
    <header>
      <h1>📖 Digital Guestbook</h1>
      <p class="subtitle">Tinggalkan kesan dan pesan Anda</p>
    </header>

    <div class="container">
      <!-- Form Tulis Pesan -->
      <div class="card">
        <h2>✍️ Tulis Pesan</h2>
        <input v-model="form.name" placeholder="Nama Anda">
        <textarea v-model="form.message" placeholder="Kesan dan pesan..." rows="4"></textarea>
        <input type="file" @change="handleFile" accept="image/*">
        <button @click="submitMessage" :disabled="loading">
          {{ loading ? 'Mengirim...' : 'Kirim Pesan' }}
        </button>
        <p v-if="successMsg" class="success">{{ successMsg }}</p>
      </div>

      <!-- Daftar Pesan -->
      <div class="card">
        <h2>💬 Pesan Tamu ({{ messages.length }})</h2>
        <div v-for="m in messages" :key="m.id" class="message-item">
          <div class="message-header">
            <strong>{{ m.name }}</strong>
            <small>{{ new Date(m.created_at).toLocaleString('id-ID') }}</small>
          </div>
          <p>{{ m.message }}</p>
          <img v-if="m.photo_url" :src="m.photo_url" class="photo-thumb">
        </div>
        <button @click="loadMessages" class="secondary">🔄 Refresh</button>
      </div>

      <!-- WebSocket Status -->
      <div class="card">
        <h2>🔌 WebSocket</h2>
        <p>Status: <span :class="wsStatus">{{ wsConnected ? 'Connected ✅' : 'Disconnected ❌' }}</span></p>
        <button @click="connectWs" v-if="!wsConnected">Connect</button>
        <button @click="disconnectWs" v-else>Disconnect</button>
        <div v-if="wsMessages.length" class="ws-messages">
          <p v-for="(msg, i) in wsMessages" :key="i">{{ msg }}</p>
        </div>
      </div>
    </div>
  </div>
  <script src="app.js"></script>
</body>
</html>
```

#### 📄 `app.js`
```javascript
// GANTI nilai di bawah dengan milik Anda!
const API_URL = 'https://XXXXXX.execute-api.us-west-2.amazonaws.com/prod';
const API_KEY = 'PASTE_API_KEY_ANDA';
const WS_URL = 'wss://XXXXXX.execute-api.us-west-2.amazonaws.com/prod';

const { createApp, ref, onMounted } = Vue;

createApp({
  setup() {
    const form = ref({ name: '', message: '', file: null });
    const messages = ref([]);
    const loading = ref(false);
    const successMsg = ref('');
    const ws = ref(null);
    const wsConnected = ref(false);
    const wsMessages = ref([]);
    const wsStatus = ref('disconnected');

    const headers = {
      'Content-Type': 'application/json',
      'x-api-key': API_KEY
    };

    const submitMessage = async () => {
      if (!form.value.name || !form.value.message) {
        alert('Nama dan pesan wajib diisi!');
        return;
      }
      loading.value = true;
      try {
        let photoUrl = null;

        // Upload foto jika ada
        if (form.value.file) {
          const reader = new FileReader();
          photoUrl = await new Promise((resolve) => {
            reader.onload = () => {
              const base64 = reader.result.split(',')[1];
              // Simpan ke S3 via API Gateway (opsional, untuk latihan cukup simpan base64)
              // Atau upload langsung ke S3 presigned URL
              resolve(`data:image/jpeg;base64,${base64}`);
            };
            reader.readAsDataURL(form.value.file);
          });
        }

        const res = await fetch(`${API_URL}/message`, {
          method: 'POST',
          headers,
          body: JSON.stringify({
            name: form.value.name,
            message: form.value.message,
            photoUrl: photoUrl
          })
        });

        if (res.ok) {
          successMsg.value = '✅ Pesan berhasil dikirim!';
          form.value = { name: '', message: '', file: null };
          loadMessages();
          setTimeout(() => successMsg.value = '', 3000);
        } else {
          alert('Error: ' + await res.text());
        }
      } catch (e) {
        alert('Network error: ' + e.message);
      } finally {
        loading.value = false;
      }
    };

    const loadMessages = async () => {
      try {
        const res = await fetch(`${API_URL}/message`, { headers });
        if (res.ok) {
          messages.value = await res.json();
        }
      } catch (e) {
        console.error(e);
      }
    };

    const handleFile = (e) => {
      form.value.file = e.target.files[0];
    };

    const connectWs = () => {
      ws.value = new WebSocket(WS_URL);
      ws.value.onopen = () => {
        wsConnected.value = true;
        wsStatus.value = 'connected';
        wsMessages.value.push('Connected to WebSocket!');
      };
      ws.value.onmessage = (event) => {
        const data = JSON.parse(event.data);
        wsMessages.value.push(`Received: ${JSON.stringify(data)}`);
      };
      ws.value.onclose = () => {
        wsConnected.value = false;
        wsStatus.value = 'disconnected';
        wsMessages.value.push('Disconnected');
      };
    };

    const disconnectWs = () => {
      if (ws.value) {
        ws.value.close();
      }
    };

    onMounted(loadMessages);

    return {
      form, messages, loading, successMsg, wsConnected, wsMessages, wsStatus,
      submitMessage, loadMessages, handleFile, connectWs, disconnectWs
    };
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
.container { max-width: 800px; margin: 0 auto; }
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
.message-item {
  padding: 15px;
  border-bottom: 1px solid #eee;
}
.message-item:last-child { border-bottom: none; }
.message-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 8px;
}
.message-header small { color: #999; }
.photo-thumb {
  max-width: 200px;
  max-height: 200px;
  border-radius: 8px;
  margin-top: 10px;
}
.connected { color: #28a745; font-weight: 600; }
.disconnected { color: #dc3545; font-weight: 600; }
.ws-messages {
  margin-top: 10px;
  padding: 10px;
  background: #f5f5f5;
  border-radius: 6px;
  max-height: 200px;
  overflow-y: auto;
  font-family: monospace;
  font-size: 12px;
}
```

### Langkah 11.2: Push ke GitHub
1. Buat repository GitHub baru: `lks-guestbook-frontend`.
2. Upload 3 file ke repository tersebut.

### Langkah 11.3: Deploy via AWS Amplify
1. Buka **AWS Amplify** → **Host web apps** → **Get started**.
2. **Deploy without Git provider** (atau connect GitHub jika diminta).
3. Pilih **GitHub** → authorize → pilih repo `lks-guestbook-frontend`.
4. Branch: `main` → **Next**.
5. Build settings: biarkan default (Amplify auto-detect).
6. **Next** → **Save and deploy**.
7. Tunggu ~2 menit → catat **URL Amplify**: `https://main.xxxxx.amplifyapp.com`.

### Langkah 11.4: Update API_URL & API_KEY
1. Edit `app.js` di GitHub → ganti `API_URL`, `API_KEY`, `WS_URL` dengan nilai asli.
2. Commit → Amplify akan auto-deploy ulang.

---

## 🧪 Fase 12: Testing End-to-End

### Test 1: Buka Frontend
```
https://main.xxxxx.amplifyapp.com
```

### Test 2: Submit Pesan
1. Isi nama & pesan → klik **Kirim Pesan**.
2. Harusnya muncul "✅ Pesan berhasil dikirim!".
3. Klik **Refresh** → pesan muncul di list.

### Test 3: Upload Foto
1. Isi nama, pesan, pilih foto → kirim.
2. Cek S3 bucket `lks-guestbook-photos-*` → folder `bukti-kehadiran/` → harus ada file baru.
3. Cek SQS `lks-queue-notif` → harus ada pesan masuk.
4. Cek CloudWatch Logs `/aws/lambda/lks-process-photo` → harus ada log processing.

### Test 4: WebSocket
1. Klik **Connect** → status berubah jadi "Connected ✅".
2. Buka console browser → kirim pesan manual via WebSocket (opsional).

### Test 5: Cek DynamoDB
1. **DynamoDB** → `lks-rating-stats` → **Explore items** → harus ada data dengan `photoCount`.

### Test 6: Cek RDS
1. Lihat CloudWatch Logs `/aws/lambda/lks-guestbook-crud` → pastikan query berhasil.

---

## 🧹 Fase 13: Cleanup

> ⚠️ **PENTING**: Hapus semua resource untuk menghindari tagihan!

Urutan penghapusan:
1. **Amplify** → Delete app.
2. **API Gateway** → Delete REST API & WebSocket API.
3. **Lambda** → Delete 3 functions.
4. **SQS** → Delete 2 queues.
5. **S3** → Empty & delete bucket `lks-guestbook-photos-*`.
6. **DynamoDB** → Delete table `lks-rating-stats`.
7. **RDS** → Delete DB instance (❌ disable snapshot).
8. **SSM Parameter Store** → Delete 4 parameters.
9. **Lambda Layer** → Delete `lks-layer-pg`.
10. **VPC** → Delete VPC `lks-vpc`.
11. **IAM** → Delete inline policy `lks-guestbook-policy` dari LabRole.

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
| 8 | S3 Event trigger ke SQS berfungsi | ☐ |
| 9 | S3 Lifecycle: 6 bulan → Glacier Deep Archive, 1 tahun → expire | ☐ |
| 10 | DynamoDB auto-scaling: RCU 10-100, WCU 5-25, target 70% | ☐ |
| 11 | REST API pakai API Key | ☐ |
| 12 | CORS enabled di API Gateway | ☐ |
| 13 | WebSocket API punya 6 routes (`$connect`, `$disconnect`, `$default`, `sendMessage`, `getConnectionId`, `broadcastMessage`) | ☐ |
| 14 | Frontend deployed via Amplify | ☐ |
| 15 | End-to-end test berhasil | ☐ |
| 16 | Environment bersih | ☐ |

---

## 💡 Tips Trainer

1. **Jika Lambda timeout connect RDS**: Cek Security Group RDS (harus izinkan dari SG Lambda) dan pastikan Lambda di private subnet yang sama VPC.
2. **Jika CORS error di browser**: Pastikan Anda sudah **Deploy API** setelah enable CORS.
3. **Jika S3 Event tidak trigger SQS**: Pastikan bucket policy SQS sudah di-update (klik "Add permission" saat setup event notification).
4. **Jika WebSocket tidak connect**: Pastikan route selection expression `$request.body.action` dan semua routes sudah di-deploy ke stage `prod`.
5. **Jika Amplify build gagal**: Pastikan file `index.html`, `app.js`, `style.css` ada di root repository.

---

## 🚀 Progression Path

```
✅ Saran 2 (Backend Only, Postman) — 2-3 jam
    ↓
✅ Saran 3 (Backend + S3 Static Frontend) — 4 jam
    ↓
🎯 Saran 1 (Backend + Amplify + WebSocket) — 6-8 jam ← ANDA DI SINI
    ↓
🏆 SOAL LKS (Sound Wave Production - 3 jam)
```

Selamat berlatih! Jika Saran 1 ini lancar, Anda siap **80-90%** untuk soal LKS sesungguhnya. 🎉