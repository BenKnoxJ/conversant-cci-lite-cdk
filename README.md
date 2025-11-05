# Conversant CCI Lite — End‑to‑End AWS Console Click‑Through Guide (v1.2, Customisable QA)

> **Scope**: Single‑account, multi‑tenant, fully serverless pipeline with **customer‑defined QA forms**. Stack: **S3 → EventBridge → Lambda (Init) → Transcribe (PII redaction) → SNS/EventBridge → Lambda (Result) → Comprehend (batch) → Bedrock → S3(JSON) → Glue → Athena → QuickSight**. No Step Functions.
>
> **What’s new in v1.2**: Custom QA templates per tenant in DynamoDB, dynamic Bedrock prompt, structured `qa_evaluation.criteria[]` output for reporting, Athena queries to explode criteria, QuickSight visuals for pass/fail by criterion, agent, and queue.

---

## 0) Prerequisites & Naming

* **Region**: `eu-central-1` for core. Bedrock calls in `us-east-1` if required by model.
* **Tenant example**: `demo-tenant`
* **Buckets**: `cci-lite-input-<accountid>-eu-central-1`, `cci-lite-results-<accountid>-eu-central-1`, `cci-lite-athena-staging-<accountid>-eu-central-1`
* **KMS key**: `alias/cci-lite-master-key`
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
  "Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::<accountid>:role/<tenant_role>"},
  "Action": ["kms:Decrypt","kms:Encrypt","kms:GenerateDataKey"],
  "Resource": "*",
  "Condition": {"StringEquals": {"kms:EncryptionContext:tenant_id": "${aws:PrincipalTag/tenant_id}"}}
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

## 6) EventBridge — Rules

* **S3ObjectCreatedToJobInit**: Source **S3**, event type **Object Created**, bucket = input bucket. Target = `cci-lite-job-init`. Filter key prefix `/calls/` in Lambda code.
* **TranscribeCompletedToResultHandler**: Source **Transcribe**, detail filter `TranscriptionJobStatus = COMPLETED`. Target = `cci-lite-result-handler`. Add a second rule for `FAILED` to a notification Lambda or DLQ if desired.

---

## 7) SQS (Optional) — Analysis Buffer

Create `cci-lite-analysis-queue` with KMS encryption. Used if you expect high concurrency spikes; otherwise, keep analysis in `result-handler`.

---

## 8) Lambda Functions

### 8.1 Shared Environment Variables

```
INPUT_BUCKET=cci-lite-input-<accountid>-eu-central-1
OUTPUT_BUCKET=cci-lite-results-<accountid>-eu-central-1
ATHENA_STAGING_BUCKET=cci-lite-athena-staging-<accountid>-eu-central-1
KMS_KEY_ARN=<KMS_KEY_ARN>
SNS_TOPIC_ARN=<SNS_TOPIC_ARN>
BEDROCK_MODEL_ID=anthropic.claude-3-haiku-20240307
BEDROCK_REGION=us-east-1
DDB_TABLE=cci-lite-config
QA_TEMPLATE_KEY=QA_TEMPLATE#v1
LOG_LEVEL=INFO
```

Enable X‑Ray; set log retention 30 days. Reserve concurrency caps: **50** total; analyzer **10** if used.

### 8.2 `cci-lite-job-init` — Start Transcribe with PII Redaction

* Trigger: EventBridge S3 rule (failover to direct S3 trigger if you also add one)
* Timeout 2 min, Memory 1024 MB

**Key behaviour**: Validate `/calls/` prefix → derive `tenant_id` → start job with output to `results/tmp/<tenant_id>/` + SNS notifications. (Use the v1.1 code; unchanged for v1.2.)

### 8.3 `cci-lite-result-handler` — Transcript → NLP → Custom QA → Unified JSON

* Trigger: EventBridge (Transcribe COMPLETED)
* Timeout 5–10 min, Memory 1536 MB

**New logic in v1.2**: fetch tenant QA template from DynamoDB and pass to Bedrock; capture structured `qa_evaluation`.

**Code outline (replaces the summariser and adds QA):**

```python
import os, json, boto3, datetime

s3 = boto3.client('s3')
comprehend = boto3.client('comprehend')
bedrock = boto3.client('bedrock-runtime', region_name=os.environ['BEDROCK_REGION'])
ddb = boto3.resource('dynamodb').Table(os.environ['DDB_TABLE'])

OUTPUT_BUCKET = os.environ['OUTPUT_BUCKET']
MODEL_ID = os.environ['BEDROCK_MODEL_ID']
QA_TEMPLATE_KEY = os.environ['QA_TEMPLATE_KEY']


def _load_transcript_from_s3(output_bucket, tenant_id):
    prefix = f"tmp/{tenant_id}/"
    resp = s3.list_objects_v2(Bucket=output_bucket, Prefix=prefix)
    for o in resp.get('Contents', []):
        if o['Key'].endswith('.json'):
            return json.loads(s3.get_object(Bucket=output_bucket, Key=o['Key'])['Body'].read())
    raise RuntimeError('Transcript JSON not found')


def _get_qa_template(tenant_id):
    resp = ddb.get_item(Key={"pk": f"TENANT#{tenant_id}", "sk": QA_TEMPLATE_KEY})
    return resp.get('Item', {"criteria": []})


def _bedrock_eval(text, qa_template):
    # Trim to protect token budget
    t = text[:45000]
    prompt = (
        "You are a strict call QA evaluator. Given the transcript and the QA criteria, return pure JSON.\n"
        "Required keys: issue, resolution, outcome, action_items[], tone_score(0-100), sentiment_summary,\n"
        "qa_evaluation: {criteria:[{id,question,passed,score,reason}], total_score, max_score, percentage, failed:[{id,reason}]}\n"
        "Scoring: score is 0 or the provided weight (if given). If required=true and not met, passed=false.\n"
        f"QA_CRITERIA_JSON: {json.dumps(qa_template.get('criteria', []))}\n"
        f"TRANSCRIPT:\n{t}"
    )
    body = {"anthropic_version":"bedrock-2023-05-31","messages":[{"role":"user","content":[{"type":"text","text":prompt}]}],"max_tokens":1800}
    resp = bedrock.invoke_model(modelId=MODEL_ID, body=json.dumps(body))
    payload = json.loads(resp['body'].read())
    content = payload.get('output_text') or payload.get('content', [{}])[0].get('text') or json.dumps(payload)
    try:
        return json.loads(content)
    except Exception:
        return {"summary_raw": content}


def handler(event, context):
    detail = event.get('detail', {})
    job_name = detail.get('TranscriptionJobName') or detail.get('TranscriptionJob',{}).get('TranscriptionJobName')
    tenant_id = job_name.split('_')[1] if job_name and '_' in job_name else 'unknown'

    # Load transcript assembled by Transcribe
    transcript = _load_transcript_from_s3(OUTPUT_BUCKET, tenant_id)
    texts = transcript.get('results', {}).get('transcripts', [])
    call_text = "\n".join([x.get('transcript','') for x in texts])

    # Comprehend NLP
    comp_in = [call_text]
    sentiment = comprehend.batch_detect_sentiment(TextList=comp_in, LanguageCode='en')
    phrases = comprehend.batch_detect_key_phrases(TextList=comp_in, LanguageCode='en')
    entities = comprehend.batch_detect_entities(TextList=comp_in, LanguageCode='en')

    # QA template per tenant
    qa_template = _get_qa_template(tenant_id)

    # Bedrock evaluation with custom QA
    ai = _bedrock_eval(call_text, qa_template)

    # Unified JSON v1.2
    now = datetime.datetime.utcnow()
    out = {
        "tenant_id": tenant_id,
        "call_id": job_name,
        "call_date": now.strftime('%Y-%m-%d'),
        "sentiment": sentiment.get('ResultList',[{}])[0],
        "key_phrases": phrases.get('ResultList',[{}])[0],
        "entities": entities.get('ResultList',[{}])[0],
        "ai_summary": {
            "issue": ai.get('issue'),
            "resolution": ai.get('resolution'),
            "outcome": ai.get('outcome'),
            "action_items": ai.get('action_items'),
            "tone_score": ai.get('tone_score'),
            "sentiment_summary": ai.get('sentiment_summary')
        },
        "qa_evaluation": ai.get('qa_evaluation', {}),
        "source": {"transcribe_job": job_name, "pii_redacted": True}
    }

    key = f"{tenant_id}/{now.year:04d}/{now.month:02d}/{now.day:02d}/{job_name}.json"
    s3.put_object(Bucket=OUTPUT_BUCKET, Key=key, Body=json.dumps(out).encode('utf-8'),
                  ServerSideEncryption='aws:kms', SSEKMSKeyId=os.environ['KMS_KEY_ARN'],
                  Tagging=f"tenant_id={tenant_id}")

    return {"written": f"s3://{OUTPUT_BUCKET}/{key}"}
```

> For high volume, send a compact payload to SQS and move NLP/Bedrock to an `analyzer` Lambda.

---

## 9) CloudWatch — Logs, Metrics, Alarms

* Errors ≥ 1 per function → notify.
* Transcribe `FAILED` path → dedicated rule to notify or DLQ.
* Optional custom metric filter for Bedrock latency > 10s.
* Retention 30 days.

---

## 10) Glue — Database & Crawler

* Database: `cci-lite-db`.
* Crawler: name `cci-lite-results-crawler`, source = Results bucket, role `cci-lite-glue-role`, schedule Daily, output to `cci-lite-db`.
* Run once to create `results` table from JSON. (For large scale, later convert to Parquet with a Glue job.)

---

## 11) Athena — Workgroups & Queries

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

## 12) QuickSight — Datasets, RLS, Dashboards

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

## 13) Budgets & Concurrency Guards

* Budgets for Transcribe, Bedrock, Lambda.
* Reserve concurrency caps. Optionally per‑function caps to prioritise the result‑handler.

---

## 14) CloudTrail — Data Events

Trail with Data events for all three buckets, plus management events. Encrypt with KMS.

---

## 15) Security Hardening

* S3 bucket policies that require `s3:ExistingObjectTag/tenant_id` to match caller’s principal tag.
* Write object tag `tenant_id` on all results (already done in code).
* Enforce KMS encryption context after you tag tenant principals.
* QuickSight: use row‑level security and separate workgroups if stronger isolation is needed.

---

## 16) End‑to‑End Validation

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

## 17) Troubleshooting Focus for v1.2

* **No QA in JSON**: Check DynamoDB item path (`TENANT#<id>` + `QA_TEMPLATE#v1`), env var `QA_TEMPLATE_KEY`, and Bedrock output parsing.
* **Model output not JSON**: Add stricter instruction and a one‑line repair step (e.g., wrap with `{"qa_evaluation": ...}`); log the raw text to debug.
* **Crawler mis‑types arrays**: Use custom classifier or create a curated external table via Athena DDL for the exploded view.
* **High Bedrock latency**: Reduce transcript slice, use summarised chunks, or switch to a cheaper/faster model.
* **Comprehend throttling**: Insert SQS buffer or reduce reserved concurrency.

---

## 18) Optional Enhancements

* **Versioned QA**: Store multiple templates (`QA_TEMPLATE#v2`) and include version used in the output (`qa_template_version`).
* **Agent/Queue metadata**: Join with external CRM/CCaaS webhook data and add to the JSON for richer reporting.
* **ETL to Parquet**: Nightly Glue job to flatten and write Parquet tables for faster Athena and cheaper queries.
* **DLQs**: SQS DLQ for both Lambdas; EventBridge rule for Transcribe `FAILED` → notify.

---

## 19) Example Unified JSON (v1.2)

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

## 20) Operator Runbook (Daily)

* Check CloudWatch alarms.
* Review Budgets.
* Verify Glue crawler success and QuickSight SPICE refresh.
* Run the **Explode QA** query; export weekly failure leaderboard.

---

**Complete.** This v1.2 guide gives you a cu
