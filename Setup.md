# Setup & Operations Guide

Quick reference for deploying, testing, troubleshooting, and cleaning up this secure AWS environment.

---

## Quick Deployment (45-60 minutes)

### 1. VPC & Network
```
VPC: 10.0.0.0/16 (DNS resolution + hostnames enabled)
├── Public Subnet: 10.0.1.0/24
└── Private Subnet: 10.0.2.0/24 (for EC2)
```

### 2. Security Groups
```
ec2-private-sg: No inbound, all outbound
ssmendpoint-sg: HTTPS (443) from ec2-private-sg
```

### 3. VPC Endpoints (Private Subnet)
```
com.amazonaws.[region].ssm
com.amazonaws.[region].ssmmessages  
com.amazonaws.[region].ec2messages
```
⚠️ **Critical:** Enable Private DNS on all endpoints

### 4. IAM Roles
```
EC2Role: AmazonSSMManagedInstanceCore policy
AdminRole: AdministratorAccess (optional)
ReadOnlyRole: ReadOnlyAccess (optional)
```

### 5. EC2 Instance
```
AMI: Amazon Linux 2023
Subnet: Private
IAM Role: EC2Role
Public IP: Disabled
IMDSv2: Required
Storage: Encrypted
```

### 6. CloudTrail → S3 → CloudWatch
```
Trail: secure-environment-trail (all regions)
S3 Bucket: cloudtrail-logs-[name] (encrypted, versioned)
Log Group: /aws/cloudtrail/secure-environment
```

### 7. CloudWatch Alarms
```
Create 5 metric filters + alarms:
- Root account usage
- Unauthorized API calls  
- IAM policy changes
- Security group changes
- Console sign-in failures

SNS Topic: SecurityAlerts (confirm email)
```

### 8. Budget Alert
```
Amount: $10/month
Alert: 80% threshold
```

---

## Testing Checklist

**Network Isolation:**
- [ ] EC2 has no public IP
- [ ] Cannot reach internet from EC2
- [ ] Can connect via Session Manager
- [ ] AWS CLI works (VPC endpoints functioning)

**Security:**
- [ ] Root account has MFA, no access keys
- [ ] CloudTrail logging active
- [ ] All 5 alarms in OK state
- [ ] S3 bucket blocks public access

**Quick Test Script:**
```bash
# Run from Session Manager
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
echo "Private IP: $(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/local-ipv4)"
echo "IAM Role: $(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/)"
timeout 3 curl -Is https://google.com && echo "Internet: FAIL" || echo "Internet: Blocked ✓"
```

---

## Common Issues & Fixes

### Session Manager Won't Connect
**Problem:** Instance doesn't appear in Session Manager

**Fix:**
1. VPC endpoint security group must allow HTTPS (443) from EC2 security group
2. Verify all 3 endpoints exist and Private DNS enabled
3. Check EC2Role attached to instance
4. Wait 5 minutes for SSM agent registration

### Metadata Commands Return Nothing
**Problem:** `curl http://169.254.169.254/...` fails

**Fix:** IMDSv2 is enforced (good!), use token-based access:
```bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/local-ipv4
```

### Unauthorized API Calls Alarm Triggered
**Problem:** CloudWatch alarm went off

**Fix:** Check CloudTrail to verify it was your testing. Alarm will reset after 5 minutes of no errors.

---

## Cleanup (Saves ~$22/month)

**Delete in this order:**

1. **CloudWatch:** Delete alarms, metric filters, log group
2. **EC2:** Terminate instance (wait 2 mins)
3. **VPC Endpoints:** Delete all 3 (saves $21.60/month)
4. **CloudTrail:** Stop logging, delete trail
5. **S3:** Empty bucket, then delete bucket
6. **SNS:** Delete SecurityAlerts topic
7. **Security Groups:** Delete ec2-private-sg, ssmendpoint-sg
8. **IAM Roles:** Delete EC2Role (optional)
9. **Budget:** Delete budget alert (optional)

**Verify cleanup:**
```bash
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running"
aws ec2 describe-vpc-endpoints
aws cloudwatch describe-alarms
```

---

## Cost Summary

| Item | Monthly Cost |
|------|--------------|
| EC2 t2.micro | $0 (free tier) or $8.50 |
| VPC Endpoints (3) | $21.60 |
| CloudTrail + S3 + CloudWatch | ~$1 |
| **Total** | **~$22-30/month** |

**Free Tier:** First 12 months, EC2 is free (750 hrs/month)

---

## Quick Commands

```bash
# Check CloudTrail status
aws cloudtrail get-trail-status --name secure-environment-trail

# View recent events
aws cloudtrail lookup-events --max-results 10

# List alarms
aws cloudwatch describe-alarms

# Check current costs
# Go to: AWS Console → Billing Dashboard
```
