# 📘 Panduan Lengkap Pengerjaan Soal LKS Serverless
## "Sound Wave Production - Concert Ticket Sales Application"

> **Estimasi waktu**: 3 jam (sesuai soal) | **Region**: `us-west-2` | **Console-based**

---

## 📋 Daftar Isi
1. [Fase 0: Persiapan & Download Source Code](#fase-0-persiapan)
2. [Fase 1: Networking (VPC)](#fase-1-networking-vpc)
3. [Fase 2: Security Groups](#fase-2-security-groups)
4. [Fase 3: Database (RDS + DynamoDB)](#fase-3-database)
5. [Fase 4: SSM Parameter Store](#fase-4-ssm-parameter-store)
6. [Fase 5: Storage (S3 + Lifecycle + Event)](#fase-5-storage)
7. [Fase 6: Queue (SQS + DLQ)](#fase-6-queue)
8. [Fase 7: IAM Policy untuk LabRole](#fase-7-iam-policy)
9. [Fase 8: Lambda Functions](#fase-8-lambda-functions)
10. [Fase 9: REST API Gateway](#fase-9-rest-api-gateway)
11. [Fase 10: WebSocket API](#fase-10-websocket-api)
12. [Fase 11: Deploy Frontend via Amplify](#fase-11-deploy-frontend)
13. [Fase 12: Testing End-to-End](#fase-12-testing)
14. [Fase 13: Cleanup](#fase-13-cleanup)

---

## 🎯 Fase 0: Persiapan & Download Source Code

### Langkah 0.1: Login & Set Region
1. Login ke **AWS Lab Learner** → klik **Start Lab** → **Open AWS Console**.
2. Di pojok kanan atas, pastikan region = **US West (Oregon) `us-west-2`**.
3. Buka **Notepad** lokal untuk mencatat:
   - Account ID
   - LabRole ARN (akan dicari di Fase 7)
   - AccessKey, SecretKey, SessionToken (dari AWS Lab Details)

### Langkah 0.2: Download Source Code dari GitHub
1. Buka browser → akses: `https://github.com/handipradana/lks-serverless.git`
2. Klik **Code** → **Download ZIP**.
3. Extract ZIP ke folder lokal.
4. Buka folder `service/` → ada file-file Lambda:
   - `auth.js` (Custom Authorizer)
   - `connection.js` (Database connection helper)
   - `writeOrder.js` (SQS consumer untuk order)
   - `payment.js` (SQS consumer untuk payment)
   - `websocket.js` (WebSocket handler)
   - Dan file Lambda lainnya untuk CRUD operations

### Langkah 0.3: Tagging Strategy
Setiap resource yang Anda buat, **WAJIB** isi:
- **Name tag**: `lks-<nama-resource>`
- **Description**: tujuan resource
- **Tags**: `Project=LKS-Serverless`, `Environment=Production`

---

## 🌐 Fase 1: Networking (VPC)

### Langkah 1.1: Buat VPC dengan Spesifikasi Soal
1. Buka service **VPC** → **Your VPCs** → **Create VPC**.
2. Pilih **VPC and more**.
3. Konfigurasi:
   - Name tag: `lks-vpc`
   - IPv4 CIDR: **`15.32.0.0/16`** (WAJIB sesuai soal)
   - Number of Availability Zones: **2**
   - Number of public subnets: **2**
   - Number of private subnets: **4** (2 per AZ)
   - ✅ Centang **NAT gateways (1)** → kita akan hapus nanti
   - ✅ Centang **VPC endpoints** → pilih **S3 (Gateway)**
4. Klik **Create VPC**. Tunggu ~2 menit.

### Langkah 1.2: Catat Resource IDs
Buka **VPC** → catat:
- **VPC ID**: `vpc-xxxxxxxx`
- **Public Subnet 1**: `subnet-pub1-xxxx` (AZ a)
- **Public Subnet 2**: `subnet-pub2-xxxx` (AZ b)
- **Private Subnet 1**: `subnet-priv1-xxxx` (AZ a)
- **Private Subnet 2**: `subnet-priv2-xxxx` (AZ a)
- **Private Subnet 3**: `subnet-priv3-xxxx` (AZ b)
- **Private Subnet 4**: `subnet-priv4-xxxx` (AZ b)

### Langkah 1.3: Tambah VPC Endpoint untuk SSM
1. **VPC** → **Endpoints** → **Create Endpoint**.
2. Service: `com.amazonaws.us-west-2.ssm`
3. VPC: `lks-vpc`
4. Subnets: **keempat private subnet**
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
3. Description: `Security group for RDS PostgreSQL - only accessible from Lambda`
4. VPC: `lks-vpc`
5. **Inbound rules**:
   | Type | Protocol | Port | Source |
   |---|---|---|---|
   | PostgreSQL | TCP | 5432 | Custom → `lks-sg-lambda` |
6. **Outbound rules**: All traffic (default)
7. **Create**.

### Langkah 2.2: SG untuk Lambda
1. **Create Security Group**.
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
1. Edit SG `lks-sg-endpoint` yang dibuat di Fase 1.3.
2. **Inbound rules**:
   | Type | Protocol | Port | Source |
   |---|---|---|---|
   | HTTPS | TCP | 443 | Custom → `15.32.0.0/16` |

---

## 🗄️ Fase 3: Database

### Langkah 3.1: Buat RDS PostgreSQL (Multi-AZ)
1. Buka **RDS** → **Databases** → **Create database**.
2. Pilih **Standard create**.
3. Engine: **PostgreSQL**, Version: **16.x** (atau terbaru).
4. Templates: **Production** (untuk Multi-AZ).
5. **Settings**:
   - DB instance identifier: `lks-rds-concertdb`
   - Master username: `lksadmin`
   - Master password: `LksPassword2026!` (catat!)
   - Confirm password.
6. **Instance configuration**:
   - DB instance class: `db.t3.medium` (atau yang sesuai budget)
7. **Connectivity**:
   - VPC: `lks-vpc`
   - Public access: **No** ❌ (WAJIB)
   - VPC security group: `lks-sg-rds` (hapus default)
   - Availability Zone: No preference
8. **Additional configuration**:
   - Initial database name: `concertdb`
   - ✅ **Multi-AZ deployment**: **Yes** (WAJIB untuk high availability)
9. **Create database**. Tunggu ~10 menit.

### Langkah 3.2: Catat Endpoint RDS
Setelah status `Available`:
- Klik DB → tab **Connectivity & security** → catat **Endpoint**:
  ```
  lks-rds-concertdb.xxxxx.us-west-2.rds.amazonaws.com
  ```

### Langkah 3.3: Buat DynamoDB Table "tokens" dengan Auto Scaling
1. Buka **DynamoDB** → **Tables** → **Create table**.
2. Table name: `tokens` (WAJIB sesuai soal)
3. Partition key: `token` (String)
4. Sort key: `deviceid` (String)
5. **Capacity mode**: **Provisioned**
6. Read capacity: **10**
7. Write capacity: **5**
8. ✅ Centang **Use auto scaling untuk read capacity**:
   - Min: **10**, Max: **100**
   - Target utilization: **70%**
9. ✅ Centang **Use auto scaling untuk write capacity**:
   - Min: **5**, Max: **25**
   - Target utilization: **70%**
10. **Create table**.

---

## 🔑 Fase 4: SSM Parameter Store

### Langkah 4.1: Buat 4 Parameter
1. Buka **Systems Manager** → **Parameter Store** → **Create parameter** (ulangi 4x):

| Name | Type | Value |
|---|---|---|
| `/lks/database/dbname` | String | `concertdb` |
| `/lks/database/username` | String | `lksadmin` |
| `/lks/database/password` | **SecureString** | `LksPassword2026!` |
| `/lks/database/endpoint` | String | `lks-rds-concertdb.xxxxx.us-west-2.rds.amazonaws.com` |

> ⚠️ Untuk `password`, pilih **KMS key source**: `aws/ssm` (default alias).

---

## 📦 Fase 5: Storage (S3 + Lifecycle + Event)

### Langkah 5.1: S3 Bucket untuk Proof of Payment
1. Buka **S3** → **Create bucket**.
2. Bucket name: `lks-YourName-YourProvince` (WAJIB format ini, contoh: `lks-john-california`)
3. Region: `us-west-2`.
4. **Block Public Access**: ✅ ON.
5. **Create bucket**.
6. Masuk bucket → **Create folder** → nama: `proofOfPayment`.

### Langkah 5.2: Setup Lifecycle Rule
1. Di bucket → tab **Management** → **Lifecycle rules** → **Create lifecycle rule**.
2. Rule name: `lks-lifecycle-payment`
3. Apply to: **Prefix** → `proofOfPayment/`
4. **Add transition**:
   - Transition to **Glacier Deep Archive** after **180 days** (6 bulan).
5. **Add expiration**:
   - Expire current version after **365 days** (1 tahun).
6. **Save**.

### Langkah 5.3: Buat SQS Queue Dulu (sebelum setup S3 Event)
> Kita akan setup S3 Event di Fase 6.2 setelah SQS dibuat.

---

## 📬 Fase 6: Queue (SQS + DLQ)

### Langkah 6.1: Buat SQS untuk Order
1. **SQS** → **Queues** → **Create queue**.
2. Type: **Standard**
3. Name: `lks-queue-order`
4. Visibility timeout: **30 seconds**
5. Receive message wait time: **5 seconds**
6. Message retention: **4 days**
7. Maximum message size: **256 KB**
8. **Dead-letter queue**: ✅ Enable
   - Buat DLQ baru: `lks-queue-order-dlq`
   - Max receive count: **3**
9. **Create**.
10. Catat **ARN**: `arn:aws:sqs:us-west-2:XXXX:lks-queue-order`

### Langkah 6.2: Buat SQS untuk Payment
1. **Create queue** lagi.
2. Type: **Standard**
3. Name: `lks-queue-payment`
4. Visibility timeout: **30 seconds**
5. Receive message wait time: **10 seconds**
6. Message retention: **2 days**
7. Maximum message size: **256 KB**
8. **Dead-letter queue**: ✅ Enable
   - Buat DLQ baru: `lks-queue-payment-dlq`
   - Max receive count: **3**
9. **Create**.
10. Catat **ARN**: `arn:aws:sqs:us-west-2:XXXX:lks-queue-payment`

### Langkah 6.3: Setup S3 Event Notification ke SQS Payment
1. Buka bucket `lks-YourName-YourProvince` → **Properties** → **Event notifications** → **Create event notification**.
2. Event name: `lks-s3-payment-upload`
3. Prefix: `proofOfPayment/`
4. Event types: ✅ **All object create events** (PUT, POST, COPY)
5. Destination: **SQS Queue**
6. SQS queue: **Enter SQS ARN** → paste ARN `lks-queue-payment`
7. **Save changes**.

> ⚠️ Jika muncul error "SQS queue policy doesn't allow S3 to write", klik **Add permission to SQS queue** → AWS akan otomatis menambahkan policy.

---

## 👤 Fase 7: IAM Policy untuk LabRole

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
        "s3:ListBucket",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::lks-*",
        "arn:aws:s3:::lks-*/*"
      ]
    },
    {
      "Sid": "SQSAccess",
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes",
        "sqs:ChangeMessageVisibility"
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
        "dynamodb:DeleteItem",
        "dynamodb:Scan",
        "dynamodb:Query"
      ],
      "Resource": [
        "arn:aws:dynamodb:us-west-2:*:table/tokens",
        "arn:aws:dynamodb:us-west-2:*:table/lks-*"
      ]
    },
    {
      "Sid": "ApiGatewayManage",
      "Effect": "Allow",
      "Action": [
        "execute-api:ManageConnections",
        "execute-api:Invoke"
      ],
      "Resource": "arn:aws:execute-api:us-west-2:*:*/*"
    },
    {
      "Sid": "VPCAccess",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DeleteNetworkInterface",
        "ec2:AssignPrivateIpAddresses",
        "ec2:UnassignPrivateIpAddresses"
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
    },
    {
      "Sid": "LambdaInvoke",
      "Effect": "Allow",
      "Action": [
        "lambda:InvokeFunction"
      ],
      "Resource": "arn:aws:lambda:us-west-2:*:function:lks-*"
    }
  ]
}
```

3. **Next** → Name: `lks-serverless-policy` → **Create policy**.

---

## ⚡ Fase 8: Lambda Functions

### Langkah 8.1: Buat Lambda Layer untuk `pg` Module
1. Buka **CloudShell** (ikon `>_` di top bar).
2. Jalankan:

```bash
mkdir -p pg-layer/nodejs && cd pg-layer/nodejs
npm init -y
npm install pg
cd ..
zip -r pg-layer.zip nodejs
aws s3 cp pg-layer.zip s3://lks-YourName-YourProvince/pg-layer.zip
```

3. Buka **Lambda** → **Layers** → **Create layer**.
   - Name: `lks-layer-pg`
   - Upload from S3: `https://s3.us-west-2.amazonaws.com/lks-YourName-YourProvince/pg-layer.zip`
   - Compatible runtimes: **Node.js 16.x**
   - **Create**.

### Langkah 8.2: Lambda #1 - `lks-auth` (Custom Authorizer)

1. **Lambda** → **Functions** → **Create function**.
2. Name: `lks-auth`
3. Runtime: **Node.js 16.x**
4. Role: `LabRole`
5. **Create function**.

6. **Configuration** → **General configuration**:
   - Timeout: **30 seconds**
   - Memory: **256 MB**

7. **Code** → ganti `index.js` dengan kode dari GitHub `auth.js`:

```javascript
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, GetItemCommand } = require('@aws-sdk/lib-dynamodb');

const ddb = DynamoDBDocumentClient.from(new DynamoDBClient({ region: 'us-west-2' }));

exports.handler = async (event) => {
  console.log('Auth event:', JSON.stringify(event, null, 2));

  const token = event.authorizationToken;
  const deviceid = event.methodContext?.deviceid || event.headers?.deviceid;

  if (!token || !deviceid) {
    return generatePolicy('anonymous', 'Deny', event.methodArn);
  }

  try {
    const result = await ddb.send(new GetItemCommand({
      TableName: 'tokens',
      Key: { token, deviceid }
    }));

    if (result.Item) {
      return generatePolicy(token, 'Allow', event.methodArn);
    } else {
      return generatePolicy(token, 'Deny', event.methodArn);
    }
  } catch (err) {
    console.error('Auth error:', err);
    return generatePolicy(token, 'Deny', event.methodArn);
  }
};

const generatePolicy = (principalId, effect, resource) => {
  const authResponse = { principalId };
  if (effect && resource) {
    const policyDocument = {
      Version: '2012-10-17',
      Statement: [{
        Action: 'execute-api:Invoke',
        Effect: effect,
        Resource: resource
      }]
    };
    authResponse.policyDocument = policyDocument;
  }
  return authResponse;
};
```

8. **Deploy**.

### Langkah 8.3: Lambda #2 - `lks-token` (Generate Token)

1. **Create function**:
   - Name: `lks-token`
   - Runtime: Node.js 16.x
   - Role: `LabRole`
   - **Create**.

2. **VPC**: Tidak perlu.

3. **Code** → ganti `index.js`:

```javascript
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, PutItemCommand } = require('@aws-sdk/lib-dynamodb');
const crypto = require('crypto');

const ddb = DynamoDBDocumentClient.from(new DynamoDBClient({ region: 'us-west-2' }));

exports.handler = async (event) => {
  console.log('Token event:', JSON.stringify(event, null, 2));

  const body = JSON.parse(event.body || '{}');
  const deviceid = body.deviceid;

  if (!deviceid) {
    return {
      statusCode: 400,
      body: JSON.stringify({ error: 'deviceid required' })
    };
  }

  const token = crypto.randomBytes(32).toString('hex');
  const expiresAt = new Date(Date.now() + 24 * 60 * 60 * 1000).toISOString();

  await ddb.send(new PutItemCommand({
    TableName: 'tokens',
    Item: {
      token,
      deviceid,
      expiresAt
    }
  }));

  return {
    statusCode: 200,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ token, expiresAt })
  };
};
```

4. **Deploy**.

### Langkah 8.4: Lambda #3 - `lks-queue-order` (SQS Producer)

1. **Create function**:
   - Name: `lks-queue-order`
   - Runtime: Node.js 16.x
   - Role: `LabRole`
   - **Create**.

2. **VPC**: Tidak perlu.

3. **Code** → ganti `index.js`:

```javascript
const { SQSClient, SendMessageCommand } = require('@aws-sdk/client-sqs');

const sqs = new SQSClient({ region: 'us-west-2' });
const QUEUE_URL = process.env.QUEUE_URL || 'arn:aws:sqs:us-west-2:XXXX:lks-queue-order';

exports.handler = async (event) => {
  console.log('Queue order event:', JSON.stringify(event, null, 2));

  const body = JSON.parse(event.body || '{}');

  await sqs.send(new SendMessageCommand({
    QueueUrl: QUEUE_URL,
    MessageBody: JSON.stringify(body),
    MessageAttributes: {
      orderId: { DataType: 'String', StringValue: body.orderId || `order-${Date.now()}` }
    }
  }));

  return {
    statusCode: 202,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ message: 'Order queued successfully' })
  };
};
```

4. **Environment variables**:
   - `QUEUE_URL`: `https://sqs.us-west-2.amazonaws.com/ACCOUNT_ID/lks-queue-order`

5. **Deploy**.

### Langkah 8.5: Lambda #4 - `lks-write-order` (SQS Consumer)

1. **Create function**:
   - Name: `lks-write-order`
   - Runtime: Node.js 16.x
   - Role: `LabRole`
   - **Create**.

2. **VPC**:
   - VPC: `lks-vpc`
   - Subnets: **keempat private subnet**
   - Security groups: `lks-sg-lambda`

3. **Triggers** → **Add trigger**:
   - Source: **SQS**
   - SQS queue: `lks-queue-order`
   - **Add**.

4. **Code** → **Layers** → **Add a layer** → Custom layers → `lks-layer-pg` → **Add**.

5. **Code** → ganti `index.js` dengan kode dari GitHub `writeOrder.js`:

```javascript
const { Client } = require('pg');
const { SSMClient, GetParametersCommand } = require('@aws-sdk/client-ssm');

const ssm = new SSMClient({ region: 'us-west-2' });

async function getDbConfig() {
  const names = [
    '/lks/database/dbname',
    '/lks/database/username',
    '/lks/database/password',
    '/lks/database/endpoint'
  ];
  const result = await ssm.send(new GetParametersCommand({
    Names: names,
    WithDecryption: true
  }));
  const map = {};
  result.Parameters.forEach(p => { map[p.Name] = p.Value; });
  return {
    host: map['/lks/database/endpoint'],
    user: map['/lks/database/username'],
    password: map['/lks/database/password'],
    database: map['/lks/database/dbname'],
    port: 5432
  };
}

async function initTable(client) {
  await client.query(`
    CREATE TABLE IF NOT EXISTS orders (
      id SERIAL PRIMARY KEY,
      order_id VARCHAR(100) UNIQUE NOT NULL,
      event_id VARCHAR(100) NOT NULL,
      ticket_id VARCHAR(100) NOT NULL,
      quantity INTEGER NOT NULL,
      total_price DECIMAL(10, 2) NOT NULL,
      status VARCHAR(50) DEFAULT 'pending',
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
  `);
}

exports.handler = async (event) => {
  console.log('Write order event:', JSON.stringify(event, null, 2));

  const config = await getDbConfig();
  const client = new Client(config);
  await client.connect();
  await initTable(client);

  try {
    for (const record of event.Records) {
      const body = JSON.parse(record.body);
      const { orderId, eventId, ticketId, quantity, totalPrice } = body;

      await client.query(
        `INSERT INTO orders (order_id, event_id, ticket_id, quantity, total_price, status)
         VALUES ($1, $2, $3, $4, $5, 'completed')
         ON CONFLICT (order_id) DO NOTHING`,
        [orderId, eventId, ticketId, quantity, totalPrice]
      );

      console.log(`Order ${orderId} written to database`);
    }

    return { statusCode: 200, body: 'Orders processed' };
  } finally {
    await client.end();
  }
};
```

6. **Deploy**.

### Langkah 8.6: Lambda #5 - `lks-payment` (SQS Consumer untuk Payment)

1. **Create function**:
   - Name: `lks-payment`
   - Runtime: Node.js 16.x
   - Role: `LabRole`
   - **Create**.

2. **VPC**: Tidak perlu.

3. **Triggers** → **Add trigger**:
   - Source: **SQS**
   - SQS queue: `lks-queue-payment`
   - **Add**.

4. **Code** → ganti `index.js` dengan kode dari GitHub `payment.js`:

```javascript
const { S3Client, GetObjectCommand } = require('@aws-sdk/client-s3');

const s3 = new S3Client({ region: 'us-west-2' });

exports.handler = async (event) => {
  console.log('Payment event:', JSON.stringify(event, null, 2));

  for (const record of event.Records) {
    try {
      const s3Event = JSON.parse(record.body);
      const bucket = s3Event.Records[0].s3.bucket.name;
      const key = decodeURIComponent(s3Event.Records[0].s3.object.key.replace(/\+/g, ' '));
      
      console.log(`Processing payment proof: s3://${bucket}/${key}`);

      // Di sini Anda bisa:
      // - Validasi file
      // - Update status order di RDS
      // - Kirim notifikasi via WebSocket
      
      console.log(`Payment proof ${key} processed successfully`);
    } catch (err) {
      console.error('Error processing record:', err);
      throw err; // akan masuk DLQ setelah 3x retry
    }
  }

  return { statusCode: 200, body: 'Payments processed' };
};
```

5. **Deploy**.

### Langkah 8.7: Lambda #6 - `lks-websocket` (WebSocket Handler)

1. **Create function**:
   - Name: `lks-websocket`
   - Runtime: Node.js 16.x
   - Role: `LabRole`
   - **Create**.

2. **VPC**: Tidak perlu.

3. **Code** → ganti `index.js` dengan kode dari GitHub `websocket.js`:

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
      await client.send(new DeleteConnectionCommand({ ConnectionId: connectionId }));
    }
    return { statusCode: 500, body: 'Error: ' + err.message };
  }
};
```

4. **Deploy**.

### Langkah 8.8: Lambda #9-15 - CRUD Functions

Buat Lambda functions berikut dengan struktur serupa (gunakan kode dari GitHub):

1. `lks-read-event` - GET events dari RDS
2. `lks-write-event` - POST/PUT/DELETE events ke RDS
3. `lks-ticket` - POST/DELETE tickets ke RDS
4. `lks-read-order` - GET orders dari RDS

> 💡 **Tips**: Gunakan template yang sama dengan `lks-write-order`, ubah query SQL sesuai kebutuhan.

---

## 🚪 Fase 9: REST API Gateway

### Langkah 9.1: Buat REST API
1. Buka **API Gateway** → **Create API** → **REST API** → **New API**.
2. Name: `lks-apiGateway`
3. Endpoint type: **Regional**
4. **Create API**.

### Langkah 9.2: Buat Custom Authorizer
1. **Authorizers** → **Create New Authorizer**.
2. Name: `lks-auth`
3. Type: **Lambda**
4. Lambda function: `lks-auth`
5. Lambda event payload: **Token**
6. Token source: `Authorization`
7. **Add header**: `deviceid`
8. Token validation: (kosongkan)
9. Authorization cache TTL: **300** seconds
10. **Create**.

### Langkah 9.3: Buat Resources & Methods

#### Resource `/token`
1. **Actions** → **Create Resource**:
   - Name: `token`, Path: `token`
   - ❌ Enable API Gateway CORS: **No**
   - **Create Resource**.
2. Klik `/token` → **Create Method** → **POST** → ✅.
   - Integration: **Lambda Function** → `lks-token` → ✅ Proxy integration.
   - **Save** → OK.
3. **Method Request** → **Settings**:
   - API Key Required: **false**
   - Authorization: **AWS_IAM**
   - **Save**.

#### Resource `/event`
1. **Create Resource**: `event`, Path: `event` → ✅ CORS.
2. Klik `/event` → **Create Method** → **GET** → ✅.
   - Integration: Lambda `lks-read-event` → ✅ Proxy.
   - **Save** → OK.
   - **Method Request** → Authorization: `lks-auth` → **Save**.
3. Klik `/event` → **Create Method** → **POST** → ✅.
   - Integration: Lambda `lks-write-event` → ✅ Proxy.
   - **Save** → OK.
   - **Method Request** → Authorization: `lks-auth` → **Save**.
4. Klik `/event` → **Create Method** → **PUT** → ✅.
   - Integration: Lambda `lks-write-event` → ✅ Proxy.
   - **Save** → OK.
   - **Method Request** → Authorization: `lks-auth` → **Save**.

#### Resource `/event/{id}`
1. **Create Resource**: `event/{id}`, Path: `{id}` → ✅ CORS.
2. Klik `/event/{id}` → **Create Method** → **DELETE** → ✅.
   - Integration: Lambda `lks-write-event` → ✅ Proxy.
   - **Save** → OK.
   - **Method Request** → Authorization: `lks-auth` → **Save**.

#### Resource `/ticket`
1. **Create Resource**: `ticket`, Path: `ticket` → ✅ CORS.
2. Klik `/ticket` → **Create Method** → **POST** → ✅.
   - Integration: Lambda `lks-ticket` → ✅ Proxy.
   - **Save** → OK.
   - **Method Request** → Authorization: `lks-auth` → **Save**.
3. Klik `/ticket` → **Create Method** → **DELETE** → ✅.
   - Integration: Lambda `lks-ticket` → ✅ Proxy.
   - **Save** → OK.
   - **Method Request** → Authorization: `lks-auth` → **Save**.

#### Resource `/ticket/{id}`
1. **Create Resource**: `ticket/{id}`, Path: `{id}` → ✅ CORS.
2. Klik `/ticket/{id}` → **Create Method** → **DELETE** → ✅.
   - Integration: Lambda `lks-ticket` → ✅ Proxy.
   - **Save** → OK.
   - **Method Request** → Authorization: `lks-auth` → **Save**.

#### Resource `/order`
1. **Create Resource**: `order`, Path: `order` → ✅ CORS.
2. Klik `/order` → **Create Method** → **GET** → ✅.
   - Integration: Lambda `lks-read-order` → ✅ Proxy.
   - **Save** → OK.
   - **Method Request** → Authorization: `lks-auth` → **Save**.
3. Klik `/order` → **Create Method** → **POST** → ✅.
   - Integration: Lambda `lks-queue-order` → ✅ Proxy.
   - **Save** → OK.
   - **Method Request** → Authorization: `lks-auth` → **Save**.

#### Resource `/payment/{filename}`
1. **Create Resource**: `payment/{filename}`, Path: `payment/{filename}` → ✅ CORS.
2. Klik `/payment/{filename}` → **Create Method** → **PUT** → ✅.
   - **Integration type**: **AWS Service**
   - AWS Service: **S3**
   - HTTP method: **PUT**
   - **Save** → OK.
   - **Integration Request**:
     - Action type: **Use path override**
     - Path override: `lks-app-2025/proofOfPayment/{filename}`
     - Execution role: `arn:aws:iam::ACCOUNT_ID:role/LabRole`
   - **URL Path Parameters**:
     - Name: `filename`
     - Mapped from: `method.request.path.filename`
   - **Save**.
3. **Method Request**:
   - Authorization: `lks-auth`
   - URL Query String Parameters: (kosongkan)
   - URL Path Parameters:
     - Add `filename` → Required: ✅
   - **Save**.

### Langkah 9.4: Enable CORS
Untuk setiap resource yang ✅ CORS:
1. Klik resource → **Actions** → **Enable CORS**.
2. Access-Control-Allow-Origin: `*`
3. Access-Control-Allow-Headers: `Authorization,deviceid,Content-Type`
4. Access-Control-Allow-Methods: `GET,POST,PUT,DELETE,OPTIONS`
5. **Yes, replace**.

### Langkah 9.5: Deploy API
1. **Actions** → **Deploy API**.
2. Deployment stage: **[New Stage]**
3. Stage name: `production`
4. **Settings** → ✅ Enable cache
5. **Deploy**.
6. Catat **Invoke URL**: `https://xxxxx.execute-api.us-west-2.amazonaws.com/production`.

---

## 🔌 Fase 10: WebSocket API

### Langkah 10.1: Buat WebSocket API
1. **API Gateway** → **Create API** → **WebSocket** → **Build**.
2. Name: `lks-websocket-api`
3. Route selection expression: `$request.body.action`
4. **Next** → **Add route**:

#### Route `$connect`
1. Route key: `$connect`
2. **Add integration** → Lambda `lks-websocket` → **Next** → **Create method**.
3. **Next** → **Deploy** → Stage name: `prod` → **Deploy**.
4. Catat **WebSocket URL**: `wss://xxxxx.execute-api.us-west-2.amazonaws.com/prod`.

#### Route `$disconnect`
1. **Actions** → **Create route** → `$disconnect`.
2. Integration: Lambda `lks-websocket`.
3. **Deploy API** ke stage `prod`.

#### Route `$default`
1. **Create route** → `$default`.
2. Integration: Lambda `lks-websocket`.
3. **Deploy**.

#### Route `sendMessage`
1. **Create route** → `sendMessage`.
2. Integration: Lambda `lks-websocket`.
3. **Deploy**.

#### Route `getConnectionId`
1. **Create route** → `getConnectionId`.
2. Integration: Lambda `lks-websocket`.
3. **Deploy**.

#### Route `broadcastMessage`
1. **Create route** → `broadcastMessage`.
2. Integration: Lambda `lks-websocket`.
3. **Deploy**.

---

## 🎨 Fase 11: Deploy Frontend via Amplify

### Langkah 11.1: Siapkan Frontend Code
1. Buka folder source code dari GitHub → folder `frontend/`.
2. Edit file `src/config.js` atau `.env`:
   ```javascript
   VUE_APP_API_URL = 'https://xxxxx.execute-api.us-west-2.amazonaws.com/production'
   VUE_APP_WS_URL = 'wss://xxxxx.execute-api.us-west-2.amazonaws.com/prod'
   ```

### Langkah 11.2: Push ke GitHub
1. Buat repository GitHub baru: `lks-frontend`.
2. Upload semua file frontend ke repository tersebut.

### Langkah 11.3: Deploy via AWS Amplify
1. Buka **AWS Amplify** → **Host web apps** → **Get started**.
2. Pilih **GitHub** → authorize → pilih repo `lks-frontend`.
3. Branch: `main` → **Next**.
4. Build settings: biarkan default (Amplify auto-detect Vue.js 3).
5. **Next** → **Save and deploy**.
6. Tunggu ~3 menit → catat **URL Amplify**: `https://main.xxxxx.amplifyapp.com`.

---

## 🧪 Fase 12: Testing End-to-End

### Test 1: Generate Token
1. Buka **Postman** atau **cURL**.
2. POST ke `https://xxxxx.execute-api.us-west-2.amazonaws.com/production/token`
3. Headers:
   - `Content-Type: application/json`
   - AWS Signature (gunakan AccessKey, SecretKey, SessionToken dari AWS Lab)
4. Body:
   ```json
   {
     "deviceid": "test-device-001"
   }
   ```
5. Harusnya dapat response:
   ```json
   {
     "token": "abc123...",
     "expiresAt": "2026-10-27T..."
   }
   ```

### Test 2: Get Events
1. GET ke `https://xxxxx.execute-api.us-west-2.amazonaws.com/production/event`
2. Headers:
   - `Authorization: <token>`
   - `deviceid: test-device-001`
3. Harusnya dapat list events.

### Test 3: Create Order
1. POST ke `https://xxxxx.execute-api.us-west-2.amazonaws.com/production/order`
2. Headers:
   - `Authorization: <token>`
   - `deviceid: test-device-001`
3. Body:
   ```json
   {
     "orderId": "order-001",
     "eventId": "event-001",
     "ticketId": "ticket-001",
     "quantity": 2,
     "totalPrice": 100.00
   }
   ```
4. Harusnya dapat response 202 Accepted.

### Test 4: Upload Payment Proof
1. PUT ke `https://xxxxx.execute-api.us-west-2.amazonaws.com/production/payment/receipt-001.jpg`
2. Headers:
   - `Authorization: <token>`
   - `deviceid: test-device-001`
   - `Content-Type: image/jpeg`
3. Body: binary file (upload gambar).
4. Cek S3 bucket `lks-YourName-YourProvince/proofOfPayment/` → harus ada file.
5. Cek SQS `lks-queue-payment` → harus ada pesan.
6. Cek CloudWatch Logs `/aws/lambda/lks-payment` → harus ada log processing.

### Test 5: WebSocket
1. Buka browser → buka URL Amplify.
2. Login dengan token yang sudah di-generate.
3. Cek console browser → WebSocket harus connected.

### Test 6: Cek Database
1. **RDS** → **Query Builder** atau lihat CloudWatch Logs.
2. Tabel `orders` harus ada data baru.

---

## 🧹 Fase 13: Cleanup

> ⚠️ **PENTING**: Hapus semua resource untuk menghindari tagihan!

Urutan penghapusan:
1. **Amplify** → Delete app.
2. **API Gateway** → Delete REST API & WebSocket API.
3. **Lambda** → Delete semua functions.
4. **SQS** → Delete 4 queues (2 main + 2 DLQ).
5. **S3** → Empty & delete bucket `lks-YourName-YourProvince`.
6. **DynamoDB** → Delete table `tokens`.
7. **RDS** → Delete DB instance (❌ disable snapshot).
8. **SSM Parameter Store** → Delete 4 parameters.
9. **Lambda Layer** → Delete `lks-layer-pg`.
10. **VPC** → Delete VPC `lks-vpc`.
11. **IAM** → Delete inline policy `lks-serverless-policy` dari LabRole.

---

## 🎓 Checklist Juri (Self-Assessment)

| # | Item | Status |
|---|---|---|
| 1 | Semua resource ber-prefix `lks-` | ☐ |
| 2 | Semua resource di-tag & ada deskripsi | ☐ |
| 3 | VPC CIDR: `15.32.0.0/16` | ☐ |
| 4 | 2 public subnets, 4 private subnets, 2 AZ | ☐ |
| 5 | RDS Multi-AZ, tidak publicly accessible | ☐ |
| 6 | RDS hanya bisa diakses dari SG Lambda | ☐ |
| 7 | DynamoDB table `tokens` dengan auto-scaling (RCU 10-100, WCU 5-25, target 70%) | ☐ |
| 8 | Password disimpan di SSM SecureString | ☐ |
| 9 | Lambda ambil config dari SSM (bukan hardcode) | ☐ |
| 10 | S3 bucket name format `lks-YourName-YourProvince` | ☐ |
| 11 | S3 Lifecycle: 6 bulan → Glacier Deep Archive, 1 tahun → expire | ☐ |
| 12 | SQS `lks-queue-order` (visibility 30s, receive 5s, retention 4 days, DLQ) | ☐ |
| 13 | SQS `lks-queue-payment` (visibility 30s, receive 10s, retention 2 days, DLQ) | ☐ |
| 14 | S3 Event trigger ke SQS payment | ☐ |
| 15 | Custom Authorizer `lks-auth` menggunakan header `Authorization` dan `deviceid` | ☐ |
| 16 | REST API stage name: `production` dengan caching | ☐ |
| 17 | REST API endpoint `/payment/{filename}` integrasi S3 dengan path override | ☐ |
| 18 | WebSocket API punya 6 routes (`$connect`, `$disconnect`, `$default`, `sendMessage`, `getConnectionId`, `broadcastMessage`) | ☐ |
| 19 | WebSocket route selection expression: `$request.body.action` | ☐ |
| 20 | Frontend deployed via Amplify | ☐ |
| 21 | End-to-end test berhasil | ☐ |
| 22 | Environment bersih | ☐ |

---

## 💡 Tips Trainer

1. **Jika Lambda timeout connect RDS**: Cek Security Group RDS (harus izinkan dari SG Lambda) dan pastikan Lambda di private subnet yang sama VPC.
2. **Jika CORS error di browser**: Pastikan Anda sudah **Deploy API** setelah enable CORS.
3. **Jika Custom Authorizer gagal**: Pastikan Lambda `lks-auth` sudah di-deploy dan IAM role punya izin DynamoDB.
4. **Jika S3 Event tidak trigger SQS**: Pastikan bucket policy SQS sudah di-update (klik "Add permission" saat setup event notification).
5. **Jika WebSocket tidak connect**: Pastikan route selection expression `$request.body.action` dan semua routes sudah di-deploy ke stage `prod`.
6. **Jika `/payment/{filename}` error 403**: Pastikan Execution Role adalah `LabRole` ARN dan path override benar: `lks-app-2025/proofOfPayment/{filename}`.
7. **Jika Amplify build gagal**: Pastikan file `package.json` ada di root repository dan build command benar.

---

## 🚀 Progression Path

```
✅ Saran 2 (Backend Only, Postman) — 2-3 jam
    ↓
✅ Saran 3 (Backend + S3 Static Frontend) — 4 jam
    ↓
✅ Saran 1 (Backend + Amplify + WebSocket) — 6-8 jam
    ↓
🏆 SOAL LKS (Sound Wave Production - 3 jam) ← ANDA DI SINI
```

Selamat mengerjakan! Jika Anda sudah mengikuti semua panduan ini dengan teliti, Anda akan berhasil menyelesaikan soal LKS dengan nilai maksimal. 🎉