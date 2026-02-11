## S3 Bucket Configuration

**Bucket Name:** secure-project-logs  
**Purpose:** Store CloudTrail audit logs and VPC flow logs

**Security Controls:**
- Block all public access (account and bucket level)
- Versioning enabled
- Default encryption (SSE-S3)
- Access restricted via IAM policies

**Cost:** ~$0.023/GB/month (minimal for logs)
