# ✅ CCI Lite v1.3 — Phase 5 : SNS (Transcribe Job Status Topic)

### Date

2025-11-06

### Region

`eu-central-1`

### Environment

Production

---

## 1. Purpose

This phase establishes the **Amazon SNS topic** that notifies the Lambda result handler when an Amazon Transcribe job completes. The topic is now a **Standard SNS topic** (not FIFO) to ensure compatibility with Amazon Transcribe notifications and Lambda subscriptions. The topic remains encrypted with the same customer-managed KMS key for consistency and tenant data protection.

---

## 2. SNS Configuration

**Topic Name:** `TranscribeJobStatusTopic`
**Type:** Standard
**Encryption:** Customer-managed KMS → `alias/cci-lite-master-key`
**Region:** eu-central-1
**Tags:**
`Project=CCI-Lite`, `Environment=Production`, `Version=v1.3`, `Region=eu-central-1`

**Topic ARN:**
`arn:aws:sns:eu-central-1:591338347562:TranscribeJobStatusTopic`

---

## 3. IAM Role Updates

The `cci-lite-lambda-role` was updated to include minimal permissions for publishing to the SNS topic if required (for example, when a job initiation function needs to send a status message). SNS **invocation of Lambda** is handled via a **subscription**, so `sns:Subscribe` and `sns:Receive` are not required for the Lambda itself.

### Updated SNS Access Policy (optional)

```json
{
  "Sid": "SNSAccess",
  "Effect": "Allow",
  "Action": [
    "sns:Publish",
    "sns:GetTopicAttributes",
    "sns:SetTopicAttributes",
    "sns:ListSubscriptionsByTopic"
  ],
  "Resource": "arn:aws:sns:eu-central-1:591338347562:TranscribeJobStatusTopic"
}
```

> Note: This block can be omitted entirely if the Lambda only **receives** messages from SNS (i.e., is subscribed). The permission is only necessary if the Lambda **publishes** messages to this topic.

---

## 4. KMS Integration

* **Encryption Key:** `arn:aws:kms:eu-central-1:591338347562:key/79d23f7f-9420-4398-a848-93876d0250e5`
* **Alias:** `alias/cci-lite-master-key`
* The key policy already authorizes the `sns.amazonaws.com` service principal. No additional entries are needed.

---

## 5. Validation Checklist

| Check                                      | Result |
| ------------------------------------------ | ------ |
| SNS Topic created                          | ✅      |
| Topic Type = Standard                      | ✅      |
| Encryption key applied                     | ✅      |
| IAM permissions for Lambda updated         | ✅      |
| Region alignment                           | ✅      |
| Topic ARN captured for Lambda subscription | ✅      |

---

## 6. Outputs / Notes

* SNS topic now acts as the **event bridge** between Amazon Transcribe and the Lambda result handler (to be created in a later phase).
* Using a **Standard** topic ensures Amazon Transcribe can publish completion events successfully.
* Encryption remains consistent under the single CMK.
* Lambda subscription will be created once the result handler function exists.
* Optional (future): add DLQ (SQS Standard) or CloudWatch delivery logging for reliability.

---

## 7. Next Phase

Proceed to **Phase 6 — EventBridge and Lambda Integration**
Tasks:

* Deploy `cci-lite-job-init` Lambda to start Transcribe jobs.
* Deploy `cci-lite-result-handler` Lambda to process completed jobs.
* Subscribe `cci-lite-result-handler` to this SNS topic once deployed.

---

*End of updated Phase 5 (SNS) documentation — aligned with Standard topic configuration and current infrastructure.*
