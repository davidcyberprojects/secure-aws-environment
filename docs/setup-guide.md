# Setup Guide

Step-by-step instructions for deploying this secure AWS environment.

## Prerequisites

- AWS account with root access secured by MFA
- Admin IAM user with MFA enabled
- AWS CLI installed and configured (optional)

---

## Deployment Order

### 1. VPC and Network Setup

**Create VPC:**
- Name: `secure-vpc`
- CIDR: `10.0.0.0/16`
- Enable DNS hostnames: ✅
- Enable DNS resolution: ✅

**Create Subnets:**
- **Public Subnet:** `10.0.1.0/24` (for future use)
- **Private Subnet:** `10.0.2.0/24` (for EC2 instances)

**Create Internet Gateway:**
- Attach to VPC (needed for public subnet, even if unused)

**Configure Route Tables:**
- **Public RT:** Add route `0.0.0.0/0` → Internet Gateway
- **Private RT:** No internet route (isolated)
- Associate subnets to correct route tables

---

### 2. Security Groups

**EC2 Private Security Group:**
- Name: `ec2-private-sg`
- Inbound: None
- Outbound: All traffic allowed

**SSM Endpoint Security Group:**
- Name: `ssmendpoint-sg`
- Inbound: HTTPS (443) from `ec2-private-sg`
- Outbound: All traffic allowed

---

### 3. VPC Endpoints

Create three interface endpoints in the **private subnet**:

1. **com.amazonaws.[region].ssm**
2. **com.amazonaws.[region].ssmmessages**
3. **com.amazonaws.[region].ec2messages**

**Configuration for each:**
- VPC: `secure-vpc`
- Subnets: Private subnet only
- Security Group: `ssmendpoint-sg`
- Private DNS: ✅ Enabled

---

### 4. IAM Roles

**Create EC2Role:**
1. Create role → AWS service → EC2
2. Attach trust policy: `ec2-role-trust-policy.json`
3. Attach permission policy: `ec2-role-policy.json` (or use managed `AmazonSSMManagedInstanceCore`)
4. Create instance profile and attach role

**Create AdminRole (optional):**
- Trust policy: `admin-role-trust-policy.json`
- Permission: Attach `AdministratorAccess` managed policy

**Create ReadOnlyRole (optional):**
- Trust policy: `readonly-role-trust-policy.json`
- Permission: Attach `ReadOnlyAccess` managed policy

---

### 5. S3 Bucket

**Create CloudTrail Logs Bucket:**
- Name: `cloudtrail-logs-[unique-name]`
- Region: Same as VPC
- Block all public access: ✅
- Versioning: ✅ Enabled
- Encryption: ✅ SSE-S3

**Apply bucket policy:**
- Allow CloudTrail service to write logs
- See CloudTrail setup for policy details

---

### 6. CloudTrail

**Create Trail:**
- Name: `secure-environment-trail`
- Apply to all regions: ✅
- S3 bucket: Use bucket from step 5
- Log file validation: ✅ Enabled
- CloudWatch Logs: ✅ Enabled
  - Log group: `/aws/cloudtrail/secure-environment`
  - Create new role for CloudTrail

**Start logging immediately**

---

### 7. CloudWatch Alarms

**Enable CloudWatch Logs for CloudTrail first** (if not done in step 6)

**Create 5 metric filters and alarms:**
1. Root account usage
2. Unauthorized API calls
3. IAM policy changes
4. Security group changes
5. Console sign-in failures

See `cloudwatch-alarms.md` for detailed patterns and thresholds.

**Create SNS topic:**
- Name: `SecurityAlerts`
- Add email subscription
- Confirm subscription via email

---

### 8. EC2 Instance

**Launch Instance:**
- AMI: Amazon Linux 2023 (latest)
- Instance type: `t2.micro` or `t3.micro`
- VPC: `secure-vpc`
- Subnet: **Private subnet**
- Auto-assign public IP: ❌ Disabled
- IAM role: `EC2Role`
- Security group: `ec2-private-sg`

**Storage:**
- Encrypted: ✅
- Size: 8 GB (default)

**Advanced details:**
- Metadata version: IMDSv2 required
- Metadata response hop limit: 1

---

### 9. Budget Alert

**Create Budget:**
- Budget type: Cost budget
- Amount: $10/month (or your preferred limit)
- Alert threshold: 80% of budget
- Email notification: Your email

---

### 10. Verification

**Test Session Manager:**
1. Wait 5 minutes for SSM agent to register
2. Systems Manager → Session Manager → Start session
3. Select your EC2 instance
4. Run validation script

**Check CloudTrail:**
- CloudTrail → Event history
- Verify events are being logged

**Verify Alarms:**
- CloudWatch → Alarms
- All should be in "OK" or "INSUFFICIENT_DATA" state

---

## Post-Deployment Checklist

- ✅ EC2 instance accessible via Session Manager
- ✅ No public IP on EC2 instance
- ✅ CloudTrail logging events
- ✅ CloudWatch alarms created and active
- ✅ SNS email subscription confirmed
- ✅ Budget alert configured
- ✅ S3 bucket has block public access enabled

---

## Estimated Setup Time

**Total:** 45-60 minutes

- VPC/Network: 10 min
- Security Groups: 5 min
- VPC Endpoints: 5 min
- IAM Roles: 10 min
- S3/CloudTrail: 10 min
- CloudWatch Alarms: 15 min
- EC2 Instance: 5 min

---

## Next Steps

1. Take screenshots of key configurations
2. Run security validation tests (`testing-checklist.md`)
3. Document any issues encountered
4. Review cost breakdown (`cost-breakdown.md`)
