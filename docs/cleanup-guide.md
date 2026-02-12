# Cleanup Guide

Step-by-step instructions to tear down this environment and avoid ongoing charges.

---

## Important Notes

⚠️ **This will delete all resources and data**  
⚠️ **CloudTrail logs will be deleted unless backed up**  
⚠️ **No undo - follow order carefully**

**Estimated Time:** 15-20 minutes  
**Monthly Savings:** ~$22-30

---

## Before You Start

### Backup Important Data (Optional)

**CloudTrail Logs:**
```bash
# Download logs from S3 bucket
aws s3 sync s3://cloudtrail-logs-[your-name] ./cloudtrail-backup/
```

**Screenshots:**
- Ensure all project screenshots are saved locally
- Verify documentation is committed to GitHub

**Configuration Files:**
- Save any custom scripts or configurations
- Export security group rules if needed for reference

---

## Deletion Order

Delete resources in this specific order to avoid dependency issues:

---

### Step 1: Delete CloudWatch Alarms

**Why first:** No dependencies, prevents false alerts during teardown

**Console:**
1. CloudWatch → Alarms
2. Select all 5 alarms
3. Actions → Delete

**CLI:**
```bash
# List all alarms
aws cloudwatch describe-alarms --query 'MetricAlarms[*].AlarmName'

# Delete alarms
aws cloudwatch delete-alarms --alarm-names \
  RootAccountUsageAlarm \
  UnauthorizedAPICallsAlarm \
  IAMPolicyChangesAlarm \
  SecurityGroupChangesAlarm \
  ConsoleSignInFailuresAlarm
```

**Savings:** ~$0/month (within free tier)

---

### Step 2: Delete CloudWatch Metric Filters

**Console:**
1. CloudWatch → Logs → Log groups
2. Select `/aws/cloudtrail/secure-environment`
3. Metric filters tab
4. Delete all 5 metric filters

**CLI:**
```bash
# Delete each metric filter
aws logs delete-metric-filter \
  --log-group-name /aws/cloudtrail/secure-environment \
  --filter-name RootAccountUsage

# Repeat for other filters
```

---

### Step 3: Terminate EC2 Instance

**Why now:** Stop hourly charges, delete before security groups

**Console:**
1. EC2 → Instances
2. Select instance
3. Instance state → Terminate instance
4. Confirm termination

**CLI:**
```bash
# Find instance ID
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,Tags[?Key==`Name`].Value|[0]]'

# Terminate
aws ec2 terminate-instances --instance-ids i-xxxxx
```

**Wait 2-3 minutes for instance to fully terminate**

**Savings:** ~$8.50/month (if beyond free tier)

---

### Step 4: Delete VPC Endpoints

**Why now:** Stop hourly charges ($0.01/hour each)

**Console:**
1. VPC → Endpoints
2. Select all 3 SSM endpoints
3. Actions → Delete VPC endpoints
4. Type "delete" to confirm

**CLI:**
```bash
# List endpoints
aws ec2 describe-vpc-endpoints --query 'VpcEndpoints[*].[VpcEndpointId,ServiceName]'

# Delete endpoints
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids vpce-xxxxx vpce-yyyyy vpce-zzzzz
```

**Savings:** ~$21.60/month

---

### Step 5: Stop CloudTrail Logging

**Console:**
1. CloudTrail → Trails
2. Select `secure-environment-trail`
3. Click "Stop logging"
4. Then Actions → Delete

**CLI:**
```bash
# Stop logging
aws cloudtrail stop-logging --name secure-environment-trail

# Delete trail
aws cloudtrail delete-trail --name secure-environment-trail
```

---

### Step 6: Delete CloudWatch Logs

**Console:**
1. CloudWatch → Logs → Log groups
2. Select `/aws/cloudtrail/secure-environment`
3. Actions → Delete log group

**CLI:**
```bash
aws logs delete-log-group --log-group-name /aws/cloudtrail/secure-environment
```

---

### Step 7: Empty and Delete S3 Bucket

**Important:** S3 buckets must be empty before deletion

**Console:**
1. S3 → Select CloudTrail bucket
2. Click "Empty"
3. Type "permanently delete" to confirm
4. Wait for completion
5. Go back, select bucket
6. Click "Delete"
7. Type bucket name to confirm

**CLI:**
```bash
# Empty bucket
aws s3 rm s3://cloudtrail-logs-[your-name] --recursive

# Delete bucket
aws s3api delete-bucket --bucket cloudtrail-logs-[your-name]
```

**Savings:** ~$0.02/month

---

### Step 8: Delete SNS Topic and Subscriptions

**Console:**
1. SNS → Topics
2. Select `SecurityAlerts`
3. Delete

**CLI:**
```bash
# Get topic ARN
aws sns list-topics

# Delete topic (also deletes subscriptions)
aws sns delete-topic --topic-arn arn:aws:sns:region:account:SecurityAlerts
```

---

### Step 9: Delete Security Groups

**Why last:** EC2 and endpoints must be deleted first

**Console:**
1. EC2 → Security Groups
2. Select `ec2-private-sg`
3. Actions → Delete security group
4. Repeat for `ssmendpoint-sg`

**Note:** Cannot delete default VPC security group

**CLI:**
```bash
# List security groups
aws ec2 describe-security-groups --query 'SecurityGroups[*].[GroupId,GroupName]'

# Delete security groups
aws ec2 delete-security-group --group-id sg-xxxxx
aws ec2 delete-security-group --group-id sg-yyyyy
```

---

### Step 10: Delete IAM Roles (Optional)

**If you want to completely clean up:**

**Console:**
1. IAM → Roles
2. Select `EC2Role`
3. Delete role (detach policies first if needed)
4. Repeat for `AdminRole` and `ReadOnlyRole` if created

**CLI:**
```bash
# Remove instance profile association
aws iam remove-role-from-instance-profile \
  --instance-profile-name EC2Role-Instance-Profile \
  --role-name EC2Role

# Delete instance profile
aws iam delete-instance-profile --instance-profile-name EC2Role-Instance-Profile

# Detach policies
aws iam detach-role-policy \
  --role-name EC2Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Delete custom policy (if created)
aws iam delete-role-policy --role-name EC2Role --policy-name EC2RolePolicy

# Delete role
aws iam delete-role --role-name EC2Role
```

---

### Step 11: Delete Budget (Optional)

**Console:**
1. Billing → Budgets
2. Select your budget
3. Actions → Delete

**CLI:**
```bash
aws budgets delete-budget \
  --account-id YOUR_ACCOUNT_ID \
  --budget-name SecureEnvironmentBudget
```

---

### Step 12: Delete VPC (Optional - Advanced)

**Only if you created a dedicated VPC and want to remove everything**

**Warning:** Ensure all resources in VPC are deleted first

**Console:**
1. VPC → Your VPCs
2. Select `secure-vpc`
3. Actions → Delete VPC
4. This will also delete:
   - Subnets
   - Route tables (non-main)
   - Internet gateway associations

**CLI:**
```bash
# Detach and delete internet gateway
aws ec2 detach-internet-gateway --internet-gateway-id igw-xxxxx --vpc-id vpc-xxxxx
aws ec2 delete-internet-gateway --internet-gateway-id igw-xxxxx

# Delete subnets
aws ec2 delete-subnet --subnet-id subnet-xxxxx
aws ec2 delete-subnet --subnet-id subnet-yyyyy

# Delete VPC
aws ec2 delete-vpc --vpc-id vpc-xxxxx
```

---

## Verification

### Confirm Deletion

Run these checks to ensure everything is removed:

```bash
# Check for running instances
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running,stopped"

# Check for VPC endpoints
aws ec2 describe-vpc-endpoints

# Check for CloudWatch alarms
aws cloudwatch describe-alarms

# Check for S3 buckets
aws s3 ls

# Check for CloudTrail trails
aws cloudtrail describe-trails
```

**Expected Result:** Empty or no resources found for this project

---

### Cost Verification

1. **AWS Billing Dashboard**
   - Check current month charges
   - Should see charges drop to $0 after resources deleted

2. **Cost Explorer** (if enabled)
   - Verify service costs return to baseline
   - VPC Endpoint charges should stop immediately

---

## Common Deletion Issues

### Can't Delete Security Group

**Error:** "has dependent objects"

**Solution:**
- Ensure all EC2 instances are terminated
- Ensure all VPC endpoints are deleted
- Wait 5 minutes and retry

---

### Can't Delete VPC

**Error:** "has dependencies"

**Solution:**
1. Delete all VPC endpoints
2. Delete all NAT gateways (if any)
3. Delete all EC2 instances
4. Detach and delete internet gateway
5. Delete subnets
6. Delete custom route tables
7. Then delete VPC

---

### S3 Bucket Won't Delete

**Error:** "bucket is not empty"

**Solution:**
- Empty bucket first (including versioned objects)
- Check for delete markers
- Use `aws s3 rm --recursive` to force delete

---

## One-Command Cleanup (Advanced)

**Warning:** Use with caution, verify IDs first

```bash
#!/bin/bash
# Quick cleanup script - CUSTOMIZE WITH YOUR IDS

# Terminate instances
aws ec2 terminate-instances --instance-ids i-xxxxx

# Wait for termination
sleep 120

# Delete VPC endpoints
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids vpce-xxx vpce-yyy vpce-zzz

# Delete CloudWatch alarms
aws cloudwatch delete-alarms --alarm-names RootAccountUsageAlarm UnauthorizedAPICallsAlarm IAMPolicyChangesAlarm SecurityGroupChangesAlarm ConsoleSignInFailuresAlarm

# Delete CloudTrail
aws cloudtrail delete-trail --name secure-environment-trail

# Empty and delete S3
aws s3 rm s3://cloudtrail-logs-yourname --recursive
aws s3api delete-bucket --bucket cloudtrail-logs-yourname

# Delete security groups (wait for EC2 termination)
sleep 60
aws ec2 delete-security-group --group-id sg-xxx
aws ec2 delete-security-group --group-id sg-yyy

echo "Cleanup complete. Verify in console."
```

---

## Post-Cleanup

### What to Keep

✅ **GitHub repository** - Keep your code and documentation  
✅ **Screenshots** - Evidence of completed project  
✅ **CloudTrail logs backup** - If you downloaded them  
✅ **IAM users** - Keep your admin user for future projects

### What's Gone

❌ EC2 instance  
❌ VPC endpoints  
❌ CloudWatch alarms  
❌ CloudTrail logging  
❌ S3 bucket and logs  
❌ Custom security groups  
❌ SNS notifications

---

## Restart Instructions

To rebuild this environment later:
1. Follow `setup-guide.md` from the beginning
2. All configurations are documented
3. Estimated rebuild time: 45-60 minutes

---

## Final Cost Check

**24-48 hours after cleanup:**
- Check AWS Billing Dashboard
- Verify charges are $0 for:
  - EC2
  - VPC (endpoints)
  - S3 (CloudTrail bucket)
  - CloudWatch (alarms)

**If charges continue:**
- Review Cost Explorer for missed resources
- Check for orphaned EBS volumes or snapshots
- Verify all regions (resources may exist in other regions)
