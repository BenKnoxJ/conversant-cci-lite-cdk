# ✅ CCI Lite v1.3 — Phase 3 : IAM Roles (Completed)

### Date  
2025-11-06  
### Region  
`eu-central-1`  
### Environment  
Production  

---

## 1. Purpose  
Phase 3 establishes the IAM roles that connect all core CCI Lite components to AWS resources securely.  
Each role defines least-privilege access to S3, KMS, Glue, QuickSight, and AWS managed services required by the analytics pipeline.

---

## 2. Roles Created  

### 🟢 `cci-lite-lambda-role`  
**Trust:** `lambda.amazonaws.com`  
**Managed Policies:** `AWSLambdaBasicExecutionRole`  
**Inline Permissions:**  
- S3 read/write on 3 buckets  
- KMS Encrypt/Decrypt on `alias/cci-lite-master-key`  
- Transcribe / Comprehend / Bedrock runtime access  
- SNS, EventBridge, CloudWatch Logs, X-Ray (optional)

**Purpose:** Executes Lambda functions (`cci-lite-job-init`, `cci-lite-result-handler`) to handle file ingestion and transcription pipeline events.  

---

### 🟢 `cci-lite-glue-role`  
**Trust:** `glue.amazonaws.com`  
**Managed Policy:** `AWSGlueServiceRole`  
**Inline Permissions:**  
- Read-only access to Results bucket  
- KMS Decrypt + DescribeKey on master key  

**Purpose:** Used by Glue Crawlers and ETL jobs to index JSON output for Athena and QuickSight.  

---

### 🟢 `cci-lite-quicksight-role`  
**Trust:** `quicksight.amazonaws.com` (custom trust)  
**Inline Permissions:**  
- Athena workgroup query execution  
- S3 read on Results and Athena staging buckets  
- Glue metadata read (GetDatabase/GetTable)  
- KMS Decrypt + DescribeKey on master key  

**Purpose:** Grants QuickSight access to Athena query results and underlying S3 data for visualization.  

---

## 3. KMS Integration  
All roles successfully added to the KMS key policy for   
`arn:aws:kms:eu-central-1:591338347562:key/79d23f7f-9420-4398-a848-93876d0250e5`.

**Validated Principals:**
`arn:aws:iam::591338347562:role/cci-lite-lambda-role`
`arn:aws:iam::591338347562:role/cci-lite-glue-role`
`arn:aws:iam::591338347562:role/cci-lite-quicksight-role`


**Policy Result:** ✅ Successfully saved with no errors.  

---

## 4. Outputs & Verification  

| Check | Result |
|-------|---------|
| Roles created | ✅ 3/3 validated in IAM |
| KMS access | ✅ All roles tested and policy saved |
| Region | ✅ eu-central-1 |
| Tags | Optional (queued for later) |

---

## 5. Next Phase  
Proceed to **Phase 4 — DynamoDB (cci-lite-config)**  
This will store tenant and job configuration data used by the Lambda functions and Transcribe workflow.

---

*End of Phase 3 update.*

