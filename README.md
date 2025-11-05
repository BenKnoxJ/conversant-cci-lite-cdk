# CCI Lite — Production Implementation Guide (AWS Console, Step‑by‑Step)

This guide is a complete, click‑by‑click procedure to deploy **CCI Lite** for production and customer use in AWS, using the **AWS Console**. CLI snippets are provided as optional checks. Follow the sequence exactly.

---

## 0) Scope and Success Criteria

**Goal:** Upload a call audio file to S3 → automatic transcript → sentiment + AI summary → JSON written to S3 → queryable in Athena → visualised in QuickSight. Multi‑tenant safe. No audio stored long‑term.

**Pass if:**

* Uploading `demo-tenant/calls/sample.wav` triggers a result JSON under `cci-lite-results/demo-tenant/YYYY/MM/DD/<call_id>.json`.
* Athena returns rows for the file within 5 minutes.
* QuickSight dashboard refresh shows the new call.
* CloudWatch Alarms stay green.

---

## 1) Prerequisites

* AWS account with admin access for initial setup.
* Region: **eu-central-1** for all resources except **Amazon Bedrock** (use **us-east-1** unless your chosen model is available in eu-central-1).
* Chosen tenant ID for first deployment, e.g. `demo-tenant`.
* Test audio file: 8–60 seconds, `.wav` or `.mp3`, G.711/PCM preferred.

**Standard Tags** (apply to every resource):

```
Project=CCI-Lite, Env=prod, Owner=Conversant, Data=Voice
```

---

## 2) Create the KMS Key

**Console:** KMS → Create key → Symmetric → Next.

* **Alias:** `alias/cci-lite-master-key`
* **Key administrators:** your admin user/role.
* **Key users:** leave empty for now; you will add roles in section 4.
* Finish creation.

**Validation:** Key appears with the alias and correct administrators.

---

## 3) Create S3 Buckets and Lifecycle

### 3.1 Input bucket

**S3 → Create bucket**

* Name: `cci-lite-input`
* Region: eu-central-1
* Block Public Access: **On**
* Default encryption: **SSE-KMS** → select `alias/cci-lite-master-key`
* Create bucket.

**Lifecycle rule:**

* Name: `input-retention-1d`
* Current object expiration: **1 day**
* Expired object delete markers: **Delete** (enabled)

**Create prefix:** `demo-tenant/calls/`

### 3.2 Results bucket

**S3 → Create bucket**

* Name: `cci-lite-results`
* Region: eu-central-1
* Block Public Access: **On**
* Default encryption: **SSE-KMS** → `alias/cci-lite-master-key`

**Lifecycle rule:**

* Name: `results-retention-30d`
* Current object expiration: **30 days**
* Expired object delete markers: **Delete** (enabled)

**Create prefixes:** `tmp/` and `demo-tenant/`

### 3.3 Bucket policies

**Input bucket policy** (S3 → Permissions → Bucket policy):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowLambdaAndUser",
      "Effect": "Allow",
      "Principal": {"AWS": [
        "arn:aws:iam::ACCOUNT_ID:role/cci-lite-lambda-role"
      ]},
      "Action": ["s3:PutObject","s3:GetObject","s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::cci-lite-input",
        "arn:aws:s3:::cci-lite-input/*"
      ],
      "Condition": {
        "Bool": {"aws:SecureTransport": "true"},
        "StringEquals": {"s3:x-amz-server-side-encryption": "aws:kms"}
      }
    },
    {
      "Sid": "DenyNonTLSorUnencrypted",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::cci-lite-input/*",
      "Condition": {
        "Bool": {"aws:SecureTransport": "false"},
        "StringNotEqualsIfExists": {"s3:x-amz-server-side-encryption": "aws:kms"}
      }
    }
  ]
}
```

**Results bucket policy** (allow Transcribe to write temporary output):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowTranscribeWritesTmp",
      "Effect": "Allow",
      "Principal": {"Service": "transcribe.amazonaws.com"},
      "Action": ["s3:PutObject"],
      "Resource": "arn:aws:s3:::cci-lite-results/tmp/*"
    }
  ]
}
```

**Validation:** Try a manual upload without SSE‑KMS → denied. With SSE‑KMS → allowed.

---

## 4) Create IAM Roles (Least Privilege)

### 4.1 `cci-lite-lambda-role`

**Console:** IAM → Roles → Create role → Trusted entity: **AWS service** → Lambda.
Attach a **custom policy** `CCI-Lite-Lambda-Policy`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Action": ["s3:GetObject","s3:PutObject"], "Resource": [
      "arn:aws:s3:::cci-lite-input/*",
      "arn:aws:s3:::cci-lite-results/*"
    ]},
    {"Effect": "Allow", "Action": ["logs:CreateLogGroup","logs:CreateLogStream","logs:PutLogEvents"], "Resource": "*"},
    {"Effect": "Allow", "Action": ["transcribe:StartTranscriptionJob","transcribe:GetTranscriptionJob"], "Resource": "*"},
    {"Effect": "Allow", "Action": ["comprehend:DetectSentiment"], "Resource": "*"},
    {"Effect": "Allow", "Action": ["bedrock:InvokeModel"], "Resource": "*"}
  ]
}
```

Create role and attach the policy.

### 4.2 `cci-lite-transcribe-role`

**Console:** IAM → Roles → Create role → Trusted entity: **Custom trust policy** → paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow","Principal": {"Service": "transcribe.amazonaws.com"},"Action": "sts:AssumeRole"}
  ]
}
```

Attach inline policy `CCI-Lite-Transcribe-WriteResults`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow","Action": ["s3:PutObject"],"Resource": "arn:aws:s3:::cci-lite-results/tmp/*"}
  ]
}
```

### 4.3 Grant KMS key usage

KMS → the key → **Key users** → add both roles above.

**Validation:** KMS key shows the two roles under Key users.

---

## 5) Parameter Store (optional but recommended)

**Systems Manager → Parameter Store → Create parameter**

* Name: `/cci-lite/config/models`
* Type: String
* Value:

```json
{"bedrock_model_summary":"anthropic.claude-3-haiku-20240307","bedrock_region":"us-east-1"}
```

* Repeat per tenant as needed: `/cci-lite/config/tenants/demo-tenant`.

---

## 6) Create the Lambda Function

**Console:** Lambda → Create function → Author from scratch.

* Name: `cci-lite-processor`
* Runtime: **Python 3.12**
* Architecture: arm64 or x86_64
* Execution role: **Use existing** → `cci-lite-lambda-role`
* Create.

**Configuration → Environment variables:**

```
INPUT_BUCKET=cci-lite-input
OUTPUT_BUCKET=cci-lite-results
TMP_PREFIX=tmp/
BEDROCK_REGION=us-east-1
MODEL_ID_CLAUDE=anthropic.claude-3-haiku-20240307
TRANSCRIBE_ROLE_ARN=arn:aws:iam::ACCOUNT_ID:role/cci-lite-transcribe-role
```

**General configuration:**

* Memory: **1024 MB**
* Timeout: **5 minutes**
* Ephemeral storage: default is fine.

**Code (index.py):**

```python
import json, os, time, boto3, urllib.parse
s3 = boto3.client('s3')
transcribe = boto3.client('transcribe')
comprehend = boto3.client('comprehend')
bedrock = boto3.client('bedrock-runtime', region_name=os.environ['BEDROCK_REGION'])

OUTPUT_BUCKET = os.environ['OUTPUT_BUCKET']
INPUT_BUCKET = os.environ['INPUT_BUCKET']
TMP_PREFIX = os.environ['TMP_PREFIX']
MODEL_ID = os.environ['MODEL_ID_CLAUDE']
TRANSCRIBE_ROLE_ARN = os.environ['TRANSCRIBE_ROLE_ARN']

def lambda_handler(event, context):
    record = event['Records'][0]
    key = urllib.parse.unquote(record['s3']['object']['key'])
    tenant = key.split('/')[0]
    call_id = key.split('/')[-1].rsplit('.', 1)[0]

    job_name = f"cci-lite-{int(time.time())}-{call_id}"
    media_uri = f"s3://{INPUT_BUCKET}/{key}"

    transcribe.start_transcription_job(
        TranscriptionJobName=job_name,
        Media={'MediaFileUri': media_uri},
        IdentifyLanguage=True,
        OutputBucketName=OUTPUT_BUCKET,
        OutputKey=f"{TMP_PREFIX}{tenant}/{job_name}/",
        OutputEncryptionKMSKeyId='alias/cci-lite-master-key',
        DataAccessRoleArn=TRANSCRIBE_ROLE_ARN
    )

    while True:
        resp = transcribe.get_transcription_job(TranscriptionJobName=job_name)
        status = resp['TranscriptionJob']['TranscriptionJobStatus']
        if status in ('COMPLETED','FAILED'):
            break
        time.sleep(5)

    if status != 'COMPLETED':
        raise RuntimeError(f"Transcription failed: {resp}")

    tmp_key = f"{TMP_PREFIX}{tenant}/{job_name}/transcription.json"
    tr_data = json.loads(s3.get_object(Bucket=OUTPUT_BUCKET, Key=tmp_key)['Body'].read())
    text = tr_data['results']['transcripts'][0]['transcript'] if tr_data['results']['transcripts'] else ''

    sent = comprehend.detect_sentiment(Text=text[:4500], LanguageCode='en')

    prompt = (
        "Return JSON with keys: summary, action_items[] (strings), risks[] (strings).\n"
        "Text:" + text[:12000]
    )
    br = bedrock.invoke_model(
        modelId=MODEL_ID,
        body=json.dumps({"prompt": prompt, "max_tokens": 600})
    )
    br_payload = json.loads(br['body'].read())
    ai_summary = br_payload.get('completion') or br_payload

    result = {
        "tenant": tenant,
        "call_id": call_id,
        "summary": ai_summary,
        "sentiment": sent,
        "timestamp": time.strftime('%Y-%m-%dT%H:%M:%SZ', time.gmtime())
    }

    out_key = f"{tenant}/{time.strftime('%Y/%m/%d')}/{call_id}.json"
    s3.put_object(Bucket=OUTPUT_BUCKET, Key=out_key, Body=json.dumps(result), ServerSideEncryption='aws:kms')

    return {"ok": True, "result_key": out_key}
```

**Save** and **Deploy**.

---

## 7) Configure S3 Event Trigger

S3 → `cci-lite-input` → **Properties** → Event notifications → Create event notification.

* Name: `on-audio-upload`
* Event types: **PUT**
* Prefix: `demo-tenant/calls/`
* Suffix: `.wav` and repeat for `.mp3` (create 2 notifications) **or** use one rule without suffix but filter in code.
* Destination: **Lambda function** → `cci-lite-processor`.

**Validation:** Lambda shows the S3 trigger in its Triggers tab.

---

## 8) First End‑to‑End Test

1. Upload `sample.wav` to `s3://cci-lite-input/demo-tenant/calls/`.
2. CloudWatch Logs → `cci-lite-processor` log group → confirm no errors.
3. S3 → `cci-lite-results/demo-tenant/YYYY/MM/DD/` → verify `<call_id>.json` exists.

**Expected JSON keys:** `tenant, call_id, summary, sentiment, timestamp`.

---

## 9) CloudWatch Log Retention and Alarms

**Logs:** Lambda → Monitor → Logs → Set retention to **30 days**.

**Alarms:** CloudWatch → Alarms → Create

* `LambdaErrorsHigh`: Metric = **Errors**, Statistic = Sum, Period = 5m, Threshold > 0 for 2 datapoints.
* `LambdaDurationHigh`: Metric = **Duration**, Threshold > 250000 ms for 2 datapoints.
  Notification to your SNS topic or email.

---

## 10) Glue + Athena (Data Catalog)

**Glue Database:** `cci_lite`

**Crawler:**

* Data source: `s3://cci-lite-results/`
* IAM role: auto‑created or a dedicated Glue role with read on results bucket.
* Target database: `cci_lite`
* Run **once**.

**Athena:**

* Workgroup: primary
* Query: `SELECT tenant, call_id, timestamp FROM "cci_lite"."cci_lite_results" ORDER BY timestamp DESC LIMIT 10;`

**Validation:** At least one row appears for your test call.

---

## 11) QuickSight Dashboard

* QuickSight → Manage data → New dataset → **Athena** → choose the table created by the crawler.
* Import to SPICE or Direct Query.
* Build visuals: calls by day, average sentiment score, word cloud from summary terms (optional).
* Schedule refresh.

---

## 12) Hardening for Production

* Replace broad IAM with least‑privilege policies shown above.
* S3 Block Public Access: **On** for both buckets.
* KMS key policy review; limit admins; enable key rotation.
* Add **AWS Budgets**: monthly cost alert for Transcribe, Comprehend, Bedrock.
* Enable CloudTrail and GuardDuty in the account.
* Optionally put Lambda in a VPC and use **Gateway/Interface VPC Endpoints** for S3, Logs, STS, Bedrock (where supported).
* Enable CloudWatch log **metric filters** for `ERROR` strings in logs.

---

## 13) Multi‑Tenant Onboarding Procedure

**For each new tenant `<tenant_id>`:**

1. Create prefixes: `cci-lite-input/<tenant_id>/calls/` and `cci-lite-results/<tenant_id>/`.
2. Add or adjust S3 event notification prefix if you isolate per tenant; otherwise keep single rule.
3. Optionally store `/cci-lite/config/tenants/<tenant_id>` in Parameter Store.
4. Validate by uploading a short test file.

**Data isolation model:** All tenant data is separated by S3 prefixes. If exposing externally, add S3 Access Points per tenant and IAM conditions on `s3:prefix`.

---

## 14) Cost Controls

* S3 lifecycle already deletes input after 1 day and results after 30 days.
* Budgets: Alerts at 50%, 80%, 100% of monthly target.
* Optional Lambda **reserved concurrency** to cap spend.
* Use **smaller Bedrock model** for summaries if quality is acceptable.

---

## 15) Operations Runbooks

**A. Transcribe job stuck in IN_PROGRESS**

* Check IAM `cci-lite-transcribe-role` trust and S3 permissions on `cci-lite-results/tmp/`.
* Confirm KMS key grants to Transcribe role.

**B. Lambda access denied to S3**

* Verify bucket policy and Lambda role permissions.
* Ensure SSE‑KMS used and Lambda role is a Key user.

**C. Empty transcript or wrong language**

* Confirm audio codec and clarity; IdentifyLanguage is enabled; try a longer sample.

**D. Bedrock `AccessDeniedException`**

* Ensure model access is granted in Bedrock; region set to `us-east-1` env var.

**E. Athena table missing columns**

* Rerun Glue crawler; ensure sample JSONs exist; check partition projection if added later.

---

## 16) Production Readiness Checklist

* [ ] All resources tagged.
* [ ] KMS key with rotation and least‑privilege.
* [ ] Buckets encrypted, lifecycle rules active, public access blocked.
* [ ] IAM policies least‑privilege and reviewed.
* [ ] S3 event notifications working.
* [ ] Lambda memory/timeout sized for typical file length.
* [ ] Alarms configured and tested.
* [ ] Glue, Athena, QuickSight pipeline working.
* [ ] Budgets created; alerts received in email/SNS.
* [ ] Runbooks documented and stored.

---

## 17) Rollback Plan

* Disable S3 event notifications to stop processing.
* Set Lambda **reserved concurrency = 0** to halt invocation.
* Remove Alarms and budget alerts after decommission.
* Delete Glue crawler and database if not reused.
* Empty and delete S3 buckets; then **schedule KMS key deletion** (with a waiting period).

---

## 18) Appendices

### A) Result JSON (minimum schema)

```json
{
  "tenant": "demo-tenant",
  "call_id": "a1b2c3",
  "summary": "...",
  "sentiment": {"Sentiment": "NEUTRAL", "SentimentScore": {"Positive":0.01,"Negative":0.02,"Neutral":0.95,"Mixed":0.02}},
  "timestamp": "2025-11-05T12:00:00Z"
}
```

### B) Optional CLI sanity checks

```bash
aws s3 ls s3://cci-lite-input/demo-tenant/calls/
aws logs tail /aws/lambda/cci-lite-processor --since 1h
aws athena start-query-execution --query-string "SELECT count(*) FROM \"cci_lite\".\"cci_lite_results\";" --work-group primary
```

### C) Supported audio hints

* Prefer 8 kHz or 16 kHz PCM WAV. Keep under 30 MB for quick tests.

---

**Done. Follow the steps in order. Use the checklist before go‑live.**
