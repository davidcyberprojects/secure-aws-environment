## CloudTrail Configuration

**Trail Name:** secure-environment-trail  
**Scope:** Multi-region (all AWS regions)  
**Storage:** S3 bucket with versioning and encryption  
**Log File Validation:** Enabled

**What's Logged:**
- All management events (API calls)
- IAM activity (logins, role changes)
- Security group modifications
- EC2 instance changes
- S3 bucket policy updates

**Retention:** Logs stored indefinitely in S3 (can set lifecycle policy to archive/delete after X days)

**Cost:** ~$2/month for 100k management events + S3 storage costs
