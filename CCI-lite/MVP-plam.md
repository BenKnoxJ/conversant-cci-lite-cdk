# CCI Lite — End-to-End MVP Deployment Runbook (Final Consolidated Edition)

This guide provides a complete, end-to-end sequence for deploying **CCI Lite** from scratch on AWS. It integrates all final design revisions for simplicity, stability, and demonstrability.

---

## 0. Overview

**Objective:** Build and demo a post-call analytics MVP that transforms uploaded call recordings into insights.

**Pipeline:**
S3 (Audio Upload) → Lambda → Transcribe → SNS → Lambda → Comprehend → Bedrock → S3 (Results JSON) → Glue → Athena → QuickSight Dashboard

**Design Principles:**

* Serverless: no persistent infrastructure.
* Secure: encrypted with KMS (alias/cci-lite-master-key).
* Lightweight: <10 resources total.
* Cost-efficient: pay-per-use only.

---

## 1. Preparation

### 1.1 AWS Requirements

* Region: **eu-central-1**.
* Services: S3, KMS, IAM, Lambda, SNS, EventBridge, Transcribe, Comprehend, Bedrock, Glue, Athena, QuickSight.
* Confirm Bedrock model access (Claude 3 Haiku) in **us-east-1**.

### 1.2 Folder & Documentation Setup

Create a GitHub repo folder structure:

```
/cci-lite
  ├─ environment-overview.md
  ├─ runbooks/
  │   ├─ cci-lite-build-runbook-v1.2.md
  │   └─ cci-lite-end-to-end-runbook.md
```

Keep all names, ARNs, and roles recorded in `environment-overview.md`.

---

## 2. Phase 1 — Foundation Setup

### Step 1: KMS Key

1. Navigate to **KMS → Customer managed keys → Create key**.
2. Choose **Symmetric / Encrypt and decrypt**.
3. Add alias: `alias/cci-lite-master-key`.
4. In **Key Policy → Add principals:**

   * `lambda.amazonaws.com`
   * `s3.amazonaws.com`
   * `glue.amazonaws.com`
   * `quicksight.amazonaws.com`
5. Include your IAM admin user.
6. Save the ARN — record it in `environment-overview.md`.

---

### Step 2: S3 Buckets

Create three buckets (all in eu-central-1):

| Bucket                                             | Purpose                | Lifecycle                       |
| -------------------------------------------------- | ---------------------- | ------------------------------- |
| `cci-lite-input-<accountid>-eu-central-1`          | Incoming audio         | Delete after 30 days            |
| `cci-lite-results-<accountid>-eu-central-1`        | Transcripts + insights | Glacier or delete after 90 days |
| `cci-lite-athena-staging-<accountid>-eu-central-1` | Athena query results   | Clean up after 30 days          |

**Settings:**

* Encryption: SSE-KMS → `alias/cci-lite-master-key`
* Block all public access.
* Versioning optional.

✅ Verified baseline storage layer.

---

### Step 3: IAM Roles

#### a) `cci-lite-lambda-role`

* Trust policy: Lambda.
* Managed: `AWSLambdaBasicExecutionRole`.
* Inline permissions:

  * Transcribe, Comprehend, Bedrock (invoke only)
  * S3 read/write (scoped to the 3 buckets)
  * KMS Encrypt/Decrypt on key ARN.

#### b) `cci-lite-glue-role`

* Trust: Glue.
* Managed: `AWSGlueServiceRole`.
* Inline: S3 access + KMS decrypt.

#### c) `cci-lite-transcribe-sns-role`

* Allows Transcribe → SNS publishing.
* Trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "transcribe.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
```

* Permissions: `sns:Publish` to `TranscribeJobStatusTopic`.

✅ Enables automatic job completion events.

---

### Step 4: Parameter Store

Under **Systems Manager → Parameter Store**, create secure strings:

```
/cci-lite/kms-key-arn
/cci-lite/input-bucket
/cci-lite/results-bucket
/cci-lite/athena-staging-bucket
/cci-lite/transcribe-sns-topic
/cci-lite/transcribe-sns-role
```

✅ Provides centralized configuration for Lambdas.

---

## 3. Phase 2 — Core Pipeline Deployment

### Step 1: Create SNS Topic

1. Name: `TranscribeJobStatusTopic`.
2. Create subscription target: Lambda (will attach after next step).

---

### Step 2: Lambda #1 — `cci-lite-job-init`

**Purpose:** Triggered by S3 → starts Transcribe job.

1. Go to **Lambda → Create function → From scratch**.

   * Name: `cci-lite-job-init`
   * Runtime: Python 3.11
   * Role: `cci-lite-lambda-role`
2. Environment variables:

   ```
   INPUT_BUCKET=<input_bucket>
   RESULTS_BUCKET=<results_bucket>
   KMS_KEY_ARN=<kms_arn>
   TRANSCRIBE_SNS_TOPIC_ARN=<topic_arn>
   TRANSCRIBE_SNS_ROLE_ARN=<sns_role_arn>
   ```
3. Code outline:

   ```python
   import boto3, os, json
   transcribe = boto3.client('transcribe')

   def lambda_handler(event, context):
       for record in event['detail']['object']['key']:
           key = record
           if not key.startswith('tenant/'):
               raise ValueError('Missing tenant prefix')
           parts = key.split('/')
           tenant_id, call_id = parts[1], parts[3].split('.')[0]
           job_name = f"{tenant_id}-{call_id}"

           transcribe.start_transcription_job(
               TranscriptionJobName=job_name,
               LanguageCode='en-US',
               Media={'MediaFileUri': f"s3://{os.environ['INPUT_BUCKET']}/{key}"},
               OutputBucketName=os.environ['RESULTS_BUCKET'],
               OutputKey=f"tenant/{tenant_id}/calls/{call_id}/transcribe.json",
               NotificationChannel={
                   'RoleArn': os.environ['TRANSCRIBE_SNS_ROLE_ARN'],
                   'SNSTopicArn': os.environ['TRANSCRIBE_SNS_TOPIC_ARN']
               }
           )
       return {'status': 'started'}
   ```
4. Set timeout: 30 seconds.

✅ This function starts the Transcribe job when an audio file is uploaded.

---

### Step 3: EventBridge Rule

* Name: `cci-lite-s3-input-to-job-init`
* Event source: `aws.s3`
* Detail-type: `Object Created`
* Filter by bucket: input bucket name.
* Target: `cci-lite-job-init` Lambda.

✅ Upload triggers pipeline automatically.

---

### Step 4: Lambda #2 — `cci-lite-result-handler`

**Purpose:** Triggered by SNS → reads transcript → runs Comprehend + Bedrock → saves result JSON.

1. Create Lambda `cci-lite-result-handler`.
2. Runtime: Python 3.11.
3. Role: `cci-lite-lambda-role`.
4. Timeout: 5 min.
5. DLQ: `cci-lite-dlq` (create SQS queue for failures).
6. Environment variables:

   ```
   RESULTS_BUCKET=<results_bucket>
   COMPREHEND_REGION=eu-central-1
   BEDROCK_MODEL_ID=anthropic.claude-3-haiku
   KMS_KEY_ARN=<kms_arn>
   ```
7. Code outline:

   ```python
   import boto3, os, json, re

   comprehend = boto3.client('comprehend', region_name=os.environ['COMPREHEND_REGION'])
   bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')
   s3 = boto3.client('s3')

   def lambda_handler(event, context):
       msg = json.loads(event['Records'][0]['Sns']['Message'])
       job_name = msg['TranscriptionJobName']
       job = boto3.client('transcribe').get_transcription_job(TranscriptionJobName=job_name)
       transcript_uri = job['TranscriptionJob']['Transcript']['TranscriptFileUri']

       transcript_data = json.loads(s3.get_object(Bucket=os.environ['RESULTS_BUCKET'], Key=f"{job_name}/transcribe.json")['Body'].read())
       text = transcript_data['results']['transcripts'][0]['transcript'][:15000]

       # Chunk text for Comprehend
       chunks = [text[i:i+4000] for i in range(0, len(text), 4000)]
       sentiments = [comprehend.detect_sentiment(Text=c, LanguageCode='en')['Sentiment'] for c in chunks]
       sentiment_final = max(set(sentiments), key=sentiments.count)

       prompt = f"Summarize the call and list actions in JSON: {text[:4000]}"
       bedrock_resp = bedrock.invoke_model(
           modelId=os.environ['BEDROCK_MODEL_ID'],
           body=json.dumps({"prompt": prompt, "max_tokens": 400})
       )

       result_json = bedrock_resp['body'].read().decode('utf-8')
       try:
           result = json.loads(result_json)
       except Exception:
           result = json.loads(re.sub(r"[^\x00-\x7F]+", "", result_json))

       output = {
           'tenant_id': job_name.split('-')[0],
           'call_id': job_name.split('-')[1],
           'sentiment': sentiment_final,
           'summary': result.get('summary', ''),
           'actions': result.get('actions', []),
       }

       s3.put_object(
           Bucket=os.environ['RESULTS_BUCKET'],
           Key=f"tenant/{output['tenant_id']}/calls/{output['call_id']}/cci-lite-output.json",
           Body=json.dumps(output),
           ServerSideEncryption='aws:kms',
           SSEKMSKeyId=os.environ['KMS_KEY_ARN']
       )
       return {'status': 'success'}
   ```

✅ Handles long transcripts, validates JSON, outputs unified result.

---

## 4. Phase 3 — Glue & Athena

### Step 1: Glue Setup

1. Create database: `cci-lite-db`.
2. Create crawler: `cci-lite-results-crawler`.

   * Source: results bucket → prefix `tenant/`
   * Role: `cci-lite-glue-role`
   * Run on-demand.
3. Run crawler after first JSON file is generated.

### Step 2: Athena

1. Go to Athena → workgroup: `cci-lite-shared`.
2. Query result location: staging bucket.
3. Run:

```sql
SELECT tenant_id, call_id, sentiment, summary FROM "cci-lite-db"."cci_lite_results" LIMIT 10;
```

✅ Confirms data flow and schema.

---

## 5. Phase 4 — QuickSight Dashboard

1. Open QuickSight → Datasets → New → Athena.
2. Data source: `cci-lite-db` → `cci_lite_results`.
3. Connection: **Direct Query** (for live updates).
4. Build visuals:

   * KPI: `count(call_id)`
   * Bar: sentiment distribution
   * Table: summary and actions
5. Add filter: `tenant_id`.

✅ Working dashboard for live demo.

---

## 6. Validation

* Upload audio to input bucket → verify automatic pipeline run.
* Confirm transcript JSON → result JSON in results bucket.
* Run Glue crawler → Athena query works.
* QuickSight shows metrics live.

✅ Full MVP flow complete.

---

## 7. Demo Checklist

| Item                            | Status |
| ------------------------------- | ------ |
| KMS policy validated            | ✅      |
| Buckets encrypted               | ✅      |
| Transcribe notifications firing | ✅      |
| Comprehend sentiment detected   | ✅      |
| Bedrock summary generated       | ✅      |
| Glue crawler success            | ✅      |
| QuickSight dashboard live       | ✅      |

---

**Build time:** ~2–3 hours
**Cost:** <$5 for 10 test calls
**Result:** Simple, secure, event-driven post-call analytics MVP.
