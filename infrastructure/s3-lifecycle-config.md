# S3 Lifecycle Configuration — Bucket: cci-lite-data

## Rule 1 – Audio Cleanup
- Prefix: `audio-input/`
- Expiration: 3 days
- Action: Permanently delete
- Status: Enabled

## Rule 2 – Transcript Expiration
- Prefix: `transcripts/`
- Expiration: 30 days
- Action: Delete
- Status: Enabled

## Rule 3 – Enriched Archive/Delete
- Prefix: `enriched/`
- Transition: Glacier Instant Retrieval (optional)
- Expiration: 90 days
- Action: Delete
- Status: Enabled

## Notes
- All rules verified via S3 Management → Lifecycle.  
- Public access blocked.  
- Default encryption: KMS (`cci-lite-key`).
