# KMS Key — cci-lite-key

**ARN:** arn:aws:kms:eu-central-1:<ACCOUNT_ID>:key/10071a89-fba0-4254-b37b-66aaf99e0831  
**Alias:** cci-lite-key  
**Rotation:** Enabled  
**Usage:** Encrypts all S3 objects, DynamoDB tables, and logs  
**Policy Summary:**
- Allows full access for root account
- Grants Encrypt/Decrypt/DataKey to:
  - `CciLiteLambdaRole`
  - `CciLiteQuickSightRole`
- Enforces tenant isolation via EncryptionContext `tenant_id`
