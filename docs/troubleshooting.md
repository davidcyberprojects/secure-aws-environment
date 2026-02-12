# Troubleshooting Guide

Common issues and solutions for this secure AWS environment.

---

## Session Manager Issues

### Instance Not Appearing in Session Manager

**Symptoms:** EC2 instance doesn't show up in Systems Manager → Fleet Manager or Session Manager

**Causes & Solutions:**

1. **SSM Agent not running**
   ```bash
   # Check status (if you have console access)
   sudo systemctl status amazon-ssm-agent
   
   # Start if stopped
   sudo systemctl start amazon-ssm-agent
   ```

2. **IAM role not attached**
   - Check EC2 instance → Actions → Security → Modify IAM role
   - Ensure `EC2Role` is attached

3. **IAM role missing permissions**
   - Verify `EC2Role` has `AmazonSSMManagedInstanceCore` policy or equivalent

4. **VPC endpoints not configured**
   - Verify all three endpoints exist: `ssm`, `ssmmessages`, `ec2messages`
   - Check endpoints are in the same subnet as EC2 instance

5. **Security group misconfiguration** ⭐ **MOST COMMON**
   - VPC endpoint security group must allow **inbound HTTPS (443)** from EC2 security group
   - Fix: Edit `ssmendpoint-sg` → Add inbound rule:
     - Type: HTTPS
     - Source: `ec2-private-sg`

6. **Private DNS not enabled**
   - VPC endpoints must have "Private DNS names" enabled
   - VPC must have DNS resolution and DNS hostnames enabled

---

## Metadata Access Issues

### Commands Return Empty or 401 Errors

**Symptoms:** `curl http://169.254.169.254/latest/meta-data/...` returns nothing or 401

**Cause:** IMDSv2 is enforced (this is good!)

**Solution:** Use token-based access:
```bash
# Get token
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Use token in requests
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/local-ipv4
```

---

## CloudWatch Alarm Issues

### Alarms Stuck in INSUFFICIENT_DATA

**Cause:** Not enough time has passed for data collection

**Solution:** Wait 10-15 minutes after alarm creation

---

### Alarm Triggered: Unauthorized API Calls

**Symptoms:** Receiving email alerts for unauthorized API calls

**Common Causes:**
1. Testing AWS CLI commands without proper permissions
2. EC2 role missing specific service permissions

**Solution:**
- Check CloudTrail to verify it was your testing activity
- If expected, alarm will return to OK after 5 minutes of no errors
- If needed, add permissions to IAM role or adjust alarm threshold

---

## CloudTrail Issues

### No Events Appearing in Event History

**Causes & Solutions:**

1. **Trail not started**
   - CloudTrail → Trails → Select trail → "Logging" should show "ON"
   - If OFF, click "Start logging"

2. **Waiting for initial logs**
   - CloudTrail has 15-minute delay for log delivery
   - Wait 15-20 minutes after trail creation

3. **S3 bucket policy incorrect**
   - Verify CloudTrail has permission to write to S3 bucket
   - Check bucket policy allows `cloudtrail.amazonaws.com`

---

## Network Connectivity Issues

### Cannot Reach Public Internet from EC2

**This is expected behavior!**

**Explanation:** Instance is in private subnet without NAT Gateway

**Solution:** None needed - this is by design for security

**If you need internet access:**
- Option 1: Add NAT Gateway (costs ~$32/month)
- Option 2: Use VPC endpoints for AWS services (recommended)

---

### AWS CLI Commands Fail with Network Errors

**Symptoms:** `aws s3 ls` or similar commands timeout

**Causes & Solutions:**

1. **VPC endpoints missing**
   - AWS CLI needs VPC endpoints to reach AWS services
   - For S3: Create S3 gateway endpoint (free)
   - For other services: Create interface endpoints

2. **Security group blocking outbound**
   - Verify EC2 security group allows outbound traffic
   - Default outbound should be "All traffic" allowed

---

## Cost Issues

### Unexpected Charges Appearing

**Common Causes:**

1. **NAT Gateway running**
   - Check VPC → NAT Gateways
   - Delete if present (~$32/month + data charges)

2. **Resources left running**
   - EC2 instances: Stop when not in use
   - Elastic IPs not attached: Delete unused EIPs ($0.005/hour each)
   - VPC endpoints: Delete unused endpoints ($0.01/hour each)

3. **CloudWatch Logs accumulating**
   - Set retention period on log groups
   - Default is "Never expire" - change to 7 or 30 days

**Solution:** Run cleanup script (`cleanup-guide.md`)

---

## S3 Access Issues

### Access Denied When Accessing S3 Bucket

**Cause:** EC2Role doesn't include S3 permissions by default

**Solution:**
Add S3 permissions to `EC2Role` policy:
```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:PutObject",
    "s3:ListBucket"
  ],
  "Resource": [
    "arn:aws:s3:::your-bucket-name",
    "arn:aws:s3:::your-bucket-name/*"
  ]
}
```

---

## IAM Role Issues

### Cannot Assume AdminRole or ReadOnlyRole

**Symptoms:** "User is not authorized to perform: sts:AssumeRole"

**Causes & Solutions:**

1. **MFA not active**
   - You must have MFA enabled and provide a token code
   - Verify MFA device is active in IAM → Users → Your User → Security credentials

2. **Trust policy incorrect**
   - Verify trust policy includes your user's account
   - Check `ACCOUNT_ID` placeholder is replaced with actual account ID

3. **IAM user lacks permissions**
   - User needs `sts:AssumeRole` permission for the specific role ARN

---

## VPC Endpoint Issues

### Endpoint Shows "Failed" Status

**Cause:** Usually subnet or route table configuration

**Solution:**
1. Verify subnet has route table attached
2. Check security group allows traffic
3. Delete and recreate endpoint if necessary

---

### DNS Resolution Not Working for Endpoints

**Symptoms:** Can't resolve `ssm.region.amazonaws.com`

**Solution:**
1. VPC settings → Ensure DNS resolution enabled
2. VPC settings → Ensure DNS hostnames enabled
3. VPC endpoint → Verify "Private DNS names" is enabled

---

## Quick Diagnostic Commands

Run these from Session Manager to troubleshoot:

```bash
# Check instance metadata access
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id

# Check IAM role
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Test AWS API access
aws sts get-caller-identity

# Check SSM agent
sudo systemctl status amazon-ssm-agent

# Test internet connectivity (should fail in private subnet)
curl -I https://www.google.com --connect-timeout 5
```

---

## Getting Help

If issues persist:
1. Check CloudTrail logs for error details
2. Review VPC Flow Logs (if enabled)
3. Verify all security group rules
4. Confirm VPC endpoint configurations
5. Check AWS service health dashboard
