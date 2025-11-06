# Conversant CCI Lite — End‑to‑End AWS Console Click‑Through Guide (v1.3, Customisable QA, Production‑Ready)

> **Scope**: Single‑account, multi‑tenant, fully serverless pipeline with **customer‑defined QA forms**. Stack: **S3 → EventBridge → Lambda (Init) → Transcribe (PII redaction) → SNS/EventBridge → Lambda (Result) → Comprehend (batch) → Bedrock → S3(JSON) → Glue → Athena → QuickSight**. No Step Functions.
>
> **What’s new in v1.3**: Production hardening. Added SNS→S3 timing backoff, strict JSON repair guidance, Transcribe S3 write permissions with data-access role, idempotency and replay safety, DLQs, cost tags, and an expanded validation checklist.

---

## Tagging Convenstions

| Key           | Example Value                                | Purpose                                      |
| ------------- | -------------------------------------------- | -------------------------------------------- |
| `Project`     | `CCI-Lite`                                   | Groups all resources for the Lite pipeline.  |
| `Environment` | `Production` *(or `Dev`, `Test`)*            | Distinguishes deployments.                   |
| `Tenant`      | `demo-tenant`                                | Enables cost/usage segmentation per tenant.  |
| `Owner`       | `Conversant`                                 | Accountability for billing and audit.        |
| `Component`   | `S3-Input` / `Lambda-ResultHandler` / `Glue` | Identifies pipeline stage.                   |
| `Region`      | `eu-central-1`                               | For multi-region tracking.                   |
| `KMSKey`      | `alias/cci-lite-master-key`                  | Links encrypted resources to the master key. |
| `CostCenter`  | `CCI`                                        | Optional, for AWS cost allocation reports.   |
| `Version`     | `v1.3`                                       | Ties back to this build reference.           |


---

## 0) Prerequisites & Naming

* **Region**: `eu-central-1` for core. Bedrock calls in `us-east-1` if required by model.
* **Tenant example**: `demo-tenant`
* **Buckets**: `cci-lite-input-<accountid>-eu-central-1`, `cci-lite-results-<accountid>-eu-central-1`, `cci-lite-athena-staging-<accountid>-eu-central-1`
* **KMS key**: `alias/cci-lite-master-key`, `arn:aws:kms:eu-central-1:591338347562:key/79d23f7f-9420-4398-a848-93876d0250e5`
* **Roles**: `cci-lite-lambda-role`, `cci-lite-glue-role`, `cci-lite-quicksight-role`
* **Lambdas**: `cci-lite-job-init`, `cci-lite-result-handler` (optionally `cci-lite-analyzer` for SQS)
* **SNS Topic**: `TranscribeJobStatusTopic`
* **EventBridge rules**: `S3ObjectCreatedToJobInit`, `TranscribeCompletedToResultHandler`
* **DynamoDB table**: `cci-lite-config`
* **Glue DB**: `cci-lite-db`; **Crawler**: `cci-lite-results-crawler`
* **Athena workgroups**: `cci-lite-shared`, plus `tenant-<tenant_id>`

---

## 1) KMS — Customer Managed Key with Tenant Context

KMS → Create key → Symmetric → alias `cci-lite-master-key` → add planned service roles as Key users. After roles exist, optionally enforce tenant guard:

```json
{
  "Version": "2012-10-17",
  "Id": "cci-lite-master-key-policy-v1.1",
  "Statement": [
    {
      "Sid": "AllowRootAccountFullAccess",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::591338347562:root" },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "AllowKeyAdministrators",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::591338347562:role/Admin",
          "arn:aws:iam::591338347562:role/CCIPlatformAdmin"
        ]
      },
      "Action": [
        "kms:Create*",
        "kms:Describe*",
        "kms:Enable*",
        "kms:List*",
        "kms:Put*",
        "kms:Update*",
        "kms:Revoke*",
        "kms:Disable*",
        "kms:Get*",
        "kms:Delete*",
        "kms:TagResource",
        "kms:UntagResource",
        "kms:ScheduleKeyDeletion",
        "kms:CancelKeyDeletion"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowCoreServiceRolesToUseKey",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::591338347562:role/cci-lite-lambda-role",
          "arn:aws:iam::591338347562:role/cci-lite-glue-role",
          "arn:aws:iam::591338347562:role/cci-lite-quicksight-role",
          "arn:aws:iam::591338347562:role/service-role/AmazonTranscribeServiceRole"
        ]
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowAWSServiceAccess",
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "lambda.amazonaws.com",
          "glue.amazonaws.com",
          "quicksight.amazonaws.com",
          "events.amazonaws.com",
          "transcribe.amazonaws.com",
          "s3.amazonaws.com"
        ]
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    },
    {
      "Sid": "TenantScopedEncryptionContext",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::591338347562:role/cci-lite-lambda-role"
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:EncryptionContext:tenant_id": "${aws:PrincipalTag/tenant_id}"
        }
      }
    },
    {
      "Sid": "AllowCloudWatchLogsAndEventBridgeAccess",
      "Effect": "Allow",
      "Principal": {
        "Service": [
          "logs.eu-central-1.amazonaws.com",
          "events.amazonaws.com"
        ]
      },
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey*",
        "kms:DescribeKey"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 2) S3 — Buckets, Encryption, Lifecycle, EventBridge

Create the three buckets with **SSE‑KMS**, versioning on, Block Public Access on. Lifecycle: Input delete after **1 day**, Results after **30 days**. Enable **EventBridge notifications** on the input bucket.

Structure:

```
s3://cci-lite-input-.../<tenant_id>/calls/
s3://cci-lite-results-.../<tenant_id>/YYYY/MM/DD/
```

---

## 3) IAM — Roles

Create `cci-lite-lambda-role` (Lambda trust) with permissions for S3 (scoped to three buckets), KMS key, Transcribe, Comprehend batch, Bedrock runtime, SNS, EventBridge, SQS (optional), Logs, X‑Ray.

Create `cci-lite-glue-role` (Glue trust) with read on Results bucket + KMS decrypt; attach `AWSGlueServiceRole`.

Create `cci-lite-quicksight-role` (QuickSight trust) with Athena workgroups access, S3 read on staging/results, Glue read, KMS decrypt.

---

## 4) DynamoDB — Tenant Config + QA Templates

DynamoDB → Create table: **`cci-lite-config`**, PK=`pk` (String), SK=`sk` (String), on‑demand capacity.

Seed items:

**Tenant meta**

```json
{
  "pk": "TENANT#demo-tenant",
  "sk": "META#v1",
  "name": "Demo Tenant",
  "bedrockModel": "anthropic.claude-3-haiku-20240307"
}
```

**Default QA template**

```json
{
  "pk": "TENANT#demo-tenant",
  "sk": "QA_TEMPLATE#v1",
  "criteria": [
    {"id":"greet_01","text":"Did the agent greet the customer politely at the start?","weight":1,"required":true},
    {"id":"empathy_02","text":"Did the agent acknowledge or empathize with the customer’s issue?","weight":1,"required":true},
    {"id":"resolve_03","text":"Did the agent provide a clear resolution or next step?","weight":1,"required":true},
    {"id":"confirm_04","text":"Did the agent confirm customer satisfaction before closing?","weight":0.5,"required":false},
    {"id":"compliance_05","text":"Was the required compliance statement read?","weight":1,"required":true}
  ],
  "scoring": {"pass_threshold": 0.7, "notes": "Weights sum may differ from required items."}
}
```

You can create multiple templates (e.g., `QA_TEMPLATE#sales`, `QA_TEMPLATE#support`).

---

## 5) SNS — Transcribe Status Topic

SNS → Create topic `TranscribeJobStatusTopic` (encrypted with KMS). Copy the Topic ARN.

---

## 6) SQS (Optional) — Analysis Buffer

Create `cci-lite-analysis-queue` with KMS encryption. Used if you expect high concurrency spikes; otherwise, keep analysis in `result-handler`.

---

# 7  Lambda Functions

### Overview

This phase defines the Lambda components that orchestrate transcription, enrichment, and analytics in the CCI Lite pipeline.
EventBridge has been removed; all events now flow through **S3 → SNS → SQS → Lambda**.

---

## 7.1 Shared Environment Variables

These apply to both Lambdas. All values are production-ready for the `eu-central-1` region.

```bash
INPUT_BUCKET=cci-lite-input-591338347562-eu-central-1
OUTPUT_BUCKET=cci-lite-results-591338347562-eu-central-1
ATHENA_STAGING_BUCKET=cci-lite-athena-staging-591338347562-eu-central-1
KMS_KEY_ARN=arn:aws:kms:eu-central-1:591338347562:key/79d23f7f-9420-4398-a848-93876d0250e5
SNS_TOPIC_ARN=arn:aws:sns:eu-central-1:591338347562:TranscribeJobStatusTopic
SQS_QUEUE_URL=https://sqs.eu-central-1.amazonaws.com/591338347562/cci-lite-analysis-queue
BEDROCK_MODEL=anthropic.claude-3-haiku-20240307
BEDROCK_REGION=us-east-1
DDB_TABLE=cci-lite-config
QA_TEMPLATE_KEY=QA_TEMPLATE#v1
LOG_LEVEL=INFO
```

**Additional settings**

* Enable **X-Ray** tracing.
* CloudWatch log retention: 30 days.
* Reserve concurrency: total 50 (10 for analyzer if added).

---

## 7.2 cci-lite-job-init — Start Transcribe with PII Redaction

| Property    | Value                                                 |
| ----------- | ----------------------------------------------------- |
| **Trigger** | S3 (Object Created) on input bucket, prefix `/calls/` |
| **Timeout** | 2 minutes                                             |
| **Memory**  | 1024 MB                                               |

**Logic**

1. Triggered when a new call audio file is uploaded.
2. Validates prefix (`/calls/`) and extracts `tenant_id`.
3. Starts an Amazon Transcribe Call Analytics job with **PII redaction enabled**.
4. Output is written to `results/tmp/<tenant_id>/`.
5. Publishes completion events to `TranscribeJobStatusTopic` (SNS).

**Notes**

* Handles audio from any tenant via deterministic `tenant_id` derivation.
* Encrypted with KMS.
* Errors and retries logged in CloudWatch.

---

## 7.3 cci-lite-result-handler — Transcribe → NLP → QA → Unified JSON

| Property    | Value                                             |
| ----------- | ------------------------------------------------- |
| **Trigger** | SQS (`cci-lite-analysis-queue`) subscribed to SNS |
| **Timeout** | 10 minutes                                        |
| **Memory**  | 1536 MB                                           |

**Logic**

1. Reads message payload from SQS (subscription from Transcribe).
2. Fetches tenant QA template from DynamoDB (`cci-lite-config`).
3. Downloads transcript JSON from S3 results path.
4. Performs NLP and QA evaluation using Bedrock (`anthropic.claude-3-haiku-20240307`).
5. Merges results into a unified JSON document with sentiment, tone, QA scores, and metadata.
6. Writes enriched JSON to `cci-lite-results-<accountid>-eu-central-1/<tenant_id>/YYYY/MM/DD/`.
7. Deletes message from SQS after successful processing.

**Notes**

* Implements retry + back-off for transcript availability.
* Maintains deterministic `call_id` based on job name for idempotency.
* Fully encrypted via CMK.
* Optional: Forward summary to notification webhook (Slack/Teams).

---

## 7.4 Production Hardening (Integrated into Handler)

| Concern                     | Mitigation                                                                                      |
| --------------------------- | ----------------------------------------------------------------------------------------------- |
| **Transcript availability** | Poll `results/tmp/<tenant_id>/` for ≤ 45 s after completion; fail cleanly if not found.         |
| **JSON repair**             | If Bedrock returns malformed JSON, strip control chars and re-parse; fallback to `summary_raw`. |
| **Idempotency**             | Use `TranscriptionJobName` or `call_id`; skip if output already exists.                         |
| **Error recovery**          | Failed messages auto-redirect to `cci-lite-dlq`; monitor with CloudWatch metrics.               |
| **Scaling safety**          | Reserved concurrency and SQS buffer prevent SNS burst failures.                                 |
| **Security**                | All data KMS-encrypted; no PII stored beyond transcript text.                                   |

---

### ✅ Outcome

Both Lambdas (`cci-lite-job-init` and `cci-lite-result-handler`) operate within a fully event-driven, KMS-encrypted pipeline:

```
S3 → Lambda(job-init) → Transcribe → SNS → SQS → Lambda(result-handler) → S3(results)
```

This configuration replaces EventBridge completely and is production-ready for CCI Lite Alpha/Pilot deployments.

---

## 8) CloudWatch — Logs, Metrics, Alarms

* Errors ≥ 1 per function → notify.
* Transcribe `FAILED` path → dedicated rule to notify or DLQ.
* Optional custom metric filter for Bedrock latency > 10s.
* Retention 30 days.

---

## 9) Glue — Database & Crawler

* Database: `cci-lite-db`.
* Crawler: name `cci-lite-results-crawler`, source = Results bucket, role `cci-lite-glue-role`, schedule Daily, output to `cci-lite-db`.
* Run once to create `results` table from JSON. (For large scale, later convert to Parquet with a Glue job.)

---

## 10) Athena — Workgroups & Queries

Workgroups: `cci-lite-shared`, `tenant-<id>` with enforced result locations in the staging bucket.

**Explode QA criteria (typical reporting view):**

```sql
-- Replace table/column paths to match your crawler output
WITH base AS (
  SELECT
    tenant_id,
    call_id,
    call_date,
    json_extract(ai_summary, '$.tone_score')        AS tone_score,
    json_extract(ai_summary, '$.sentiment_summary') AS sentiment_summary,
    json_extract(qa_evaluation, '$.criteria')       AS criteria
  FROM "cci-lite-db"."results"
  WHERE tenant_id = 'demo-tenant'
)
SELECT
  tenant_id,
  call_id,
  call_date,
  json_extract_scalar(citem, '$.id')        AS criterion_id,
  json_extract_scalar(citem, '$.question')  AS question,
  CAST(json_extract_scalar(citem, '$.passed') AS boolean) AS passed,
  CAST(json_extract_scalar(citem, '$.score') AS double)   AS score
FROM base
CROSS JOIN UNNEST(cast(json_parse(criteria) as array(json))) AS t(citem);
```

**Failure rate per criterion:**

```sql
SELECT criterion_id,
       question,
       1 - avg(CASE WHEN passed THEN 1.0 ELSE 0.0 END) AS fail_rate,
       count(*) AS calls
FROM (<previous query>)
GROUP BY 1,2
ORDER BY fail_rate DESC, calls DESC;
```

---

## 11) QuickSight — Datasets, RLS, Dashboards

1. Create Athena dataset from `cci-lite-db.results`.
2. Create a second dataset using the **Explode QA criteria** query as a custom SQL dataset.
3. RLS: mapping of `user_email` → `tenant_id` dataset; attach to both datasets.
4. Visuals:

   * KPI: Overall QA% = avg(passed) by time (use custom SQL dataset)
   * Table: Top failed criteria with fail_rate and call count
   * Heatmap: Agent × Criterion fail rate (once agent id is available in metadata)
   * Trend: Tone score and sentiment over time
5. SPICE refresh: Daily.

---

## 12) Budgets & Concurrency Guards

* Budgets for Transcribe, Bedrock, Lambda.
* Reserve concurrency caps. Optionally per‑function caps to prioritise the result‑handler.

---

## 13) CloudTrail — Data Events

Trail with Data events for all three buckets, plus management events. Encrypt with KMS.

---

## 14) Security Hardening

* S3 bucket policies that require `s3:ExistingObjectTag/tenant_id` to match caller’s principal tag.
* Write object tag `tenant_id` on all results (already done in code).
* Enforce KMS encryption context after you tag tenant principals.
* QuickSight: use row‑level security and separate workgroups if stronger isolation is needed.

---

## 15) End‑to‑End Validation

1. Upload `.wav` to `s3://cci-lite-input-.../demo-tenant/calls/test.wav`.
2. EventBridge → `job-init` starts Transcribe; SNS wired.
3. On COMPLETED, `result-handler` runs.
4. Check S3 results path for JSON; confirm fields `qa_evaluation.criteria[]` exist.
5. Run Glue crawler; confirm/update schema.
6. Athena queries return rows; exploded criteria query works.
7. QuickSight dashboards show QA pass/fail, failure leaders.
8. Confirm alarms and budgets are quiet.

**Success**: Per‑criterion reporting available. Cost per 10‑min call ≤ target. PII redacted. Multi‑tenant isolation intact.

---

## 16) Troubleshooting Focus for v1.2

* **No QA in JSON**: Check DynamoDB item path (`TENANT#<id>` + `QA_TEMPLATE#v1`), env var `QA_TEMPLATE_KEY`, and Bedrock output parsing.
* **Model output not JSON**: Add stricter instruction and a one‑line repair step (e.g., wrap with `{"qa_evaluation": ...}`); log the raw text to debug.
* **Crawler mis‑types arrays**: Use custom classifier or create a curated external table via Athena DDL for the exploded view.
* **High Bedrock latency**: Reduce transcript slice, use summarised chunks, or switch to a cheaper/faster model.
* **Comprehend throttling**: Insert SQS buffer or reduce reserved concurrency.

---

## 17) Optional Enhancements

* **Versioned QA**: Store multiple templates (`QA_TEMPLATE#v2`) and include version used in the output (`qa_template_version`).
* **Agent/Queue metadata**: Join with external CRM/CCaaS webhook data and add to the JSON for richer reporting.
* **ETL to Parquet**: Nightly Glue job to flatten and write Parquet tables for faster Athena and cheaper queries.
* **DLQs**: SQS DLQ for both Lambdas; EventBridge rule for Transcribe `FAILED` → notify.

---

## 18) Example Unified JSON (v1.2)

```json
{
  "tenant_id": "demo-tenant",
  "call_id": "cci_demo-tenant_test_wav",
  "call_date": "2025-11-05",
  "sentiment": {"ResultList": [{"Sentiment": "NEUTRAL"}]},
  "key_phrases": {"ResultList": [{"KeyPhrases": [{"Text": "order number"}]}]},
  "entities": {"ResultList": [{"Entities": [{"Text": "Alice", "Type": "PERSON"}]}]},
  "ai_summary": {
    "issue": "Billing query",
    "resolution": "Explained invoice line items",
    "outcome": "Resolved",
    "action_items": ["Email revised invoice"],
    "tone_score": 73,
    "sentiment_summary": "Mostly neutral"
  },
  "qa_evaluation": {
    "criteria": [
      {"id": "greet_01", "question": "Did the agent greet the customer politely at the start?", "passed": true,  "score": 1,   "reason": "Greeting present"},
      {"id": "empathy_02","question": "Did the agent acknowledge or empathize with the customer’s issue?", "passed": false, "score": 0,   "reason": "No empathy statement"},
      {"id": "resolve_03","question": "Did the agent provide a clear resolution or next step?",            "passed": true,  "score": 1,   "reason": "Clear resolution"},
      {"id": "confirm_04","question": "Did the agent confirm customer satisfaction before closing?",       "passed": true,  "score": 0.5, "reason": "Asked for confirmation"},
      {"id": "compliance_05","question": "Was the required compliance statement read?",                   "passed": false, "score": 0,   "reason": "No compliance script"}
    ],
    "total_score": 2.5,
    "max_score": 3.5,
    "percentage": 71,
    "failed": [
      {"id": "empathy_02", "reason": "No empathy statement"},
      {"id": "compliance_05", "reason": "No compliance script"}
    ]
  },
  "source": {"transcribe_job": "cci_demo-tenant_test_wav", "pii_redacted": true}
}
```

---

## 19) Operator Runbook (Daily)

* Check CloudWatch alarms.
* Review Budgets.
* Verify Glue crawler success and QuickSight SPICE refresh.
* Run the **Explode QA** query; export weekly failure leaderboard.

---

**Complete.** This v1.2 guide gives you a customisable QA and insights product on CCI Lite with full click‑through steps and reporting ready for customers.
