# ✅ CCI Lite v1.3 — S3 & KMS Foundation Setup (Completed)

### Date

2025-11-06

### Region

`eu-central-1` (core) — Bedrock calls use `eu-central-1` if model requires

### Environment

Production

---

## 1. Purpose

This phase establishes the **secure storage and event backbone** for the CCI Lite pipeline.
All subsequent components (Lambda, Transcribe, Glue, Athena) depend on these encrypted, versioned S3 buckets and the customer-managed KMS key.

---

## 2. Components Implemented

### 🔑 KMS — `alias/cci-lite-master-key`

**ARN:**
`arn:aws:kms:eu-central-1:591338347562:key/79d23f7f-9420-4398-a848-93876d0250e5`

* Created as **symmetric** customer-managed key.
* Region locked to `eu-central-1`.
* Rotation: enabled.
* Used by all S3, Lambda, Glue, and QuickSight resources.
* Policy includes root + admin + AWS service principals (Lambda, Glue, QuickSight, Transcribe, EventBridge, S3).
* Tenant guard condition reserved for future isolation enforcement:

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

* Tag set applied:

  ```
  Project = CCI-Lite
  Environment = Production
  Owner = Conversant
  Version = v1.3
  ```

---

### 🧳 S3 Buckets (Created)

| Bucket                                              | Purpose                     | Lifecycle            | Encryption                      | Versioning |
| --------------------------------------------------- | --------------------------- | -------------------- | ------------------------------- | ---------- |
| `cci-lite-input-591338347562-eu-central-1`          | Upload call audio           | Delete after 1 day   | SSE-KMS (`cci-lite-master-key`) | Enabled    |
| `cci-lite-results-591338347562-eu-central-1`        | Store JSON analytics output | Delete after 30 days | SSE-KMS                         | Enabled    |
| `cci-lite-athena-staging-591338347562-eu-central-1` | Athena query results        | None                 | SSE-KMS                         | Enabled    |

**Global settings:**

* Block all public access
* ACLs disabled
* EventBridge notifications enabled on the **input bucket**

**Structure (auto-created by Lambdas and Transcribe):**

```
s3://cci-lite-input-.../<tenant_id>/calls/
s3://cci-lite-results-.../<tenant_id>/YYYY/MM/DD/
```

---

## 3. Why These Choices

| Design Goal              | Implementation Decision                                                    |
| ------------------------ | -------------------------------------------------------------------------- |
| Security & Compliance    | SSE-KMS using customer key + restricted policy.                            |
| Cost & Retention Control | Lifecycle rules enforce short-term storage (1 day input, 30 days results). |
| Automation & Integration | EventBridge notifications trigger Lambda `cci-lite-job-init`.              |
| Multi-Tenant Isolation   | Tenant prefixes (`demo-tenant/`) prepare for logical separation.           |
| Observability            | Versioning supports recovery & audit of S3 object changes.                 |

---

## 4. Replication Guide for New Tenants

When onboarding a new customer or tenant, repeat these **tenant-specific** actions only:

1. **Create prefixes** (automatically done on first upload):
   `s3://cci-lite-input-.../<tenant_id>/calls/`
2. **Add tenant config** record to DynamoDB `cci-lite-config`.
3. **Optionally** enforce tenant guard in KMS policy:

   ```json
   "Principal": { "AWS": "arn:aws:iam::<account_id>:role/cci-lite-lambda-role" },
   "Condition": { "StringEquals": { "kms:EncryptionContext:tenant_id": "<tenant_id>" } }
   ```
4. Tag the new resources or prefixes:

   ```
   Tenant = <tenant_id>
   Project = CCI-Lite
   Environment = Production
   Component = S3
   Region = eu-central-1
   Version = v1.3
   ```
5. Verify EventBridge is still enabled on the input bucket (applies globally).

No new buckets are created per tenant; the design is **multi-tenant, prefix-partitioned**.

---

## 5. Next Phase

Proceed to **Step 3 — IAM Roles** creation:

* `cci-lite-lambda-role`
* `cci-lite-glue-role`
* `cci-lite-quicksight-role`

Each role will require **KMS decrypt/encrypt access** on
`arn:aws:kms:eu-central-1:591338347562:key/79d23f7f-9420-4398-a848-93876d0250e5`.

---

*End of S3 & KMS Foundation update.*
