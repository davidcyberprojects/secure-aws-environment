# Security Testing Checklist

Validation steps to verify the security posture of this AWS environment.

---

## Pre-Testing Setup

- [ ] All resources deployed per `setup-guide.md`
- [ ] EC2 instance running and accessible via Session Manager
- [ ] CloudTrail logging for at least 15 minutes
- [ ] All CloudWatch alarms in "OK" state

---

## Network Isolation Tests

### Test 1: Verify No Public IP Address

**What it tests:** EC2 instance is truly private

**Steps:**
1. Connect to EC2 via Session Manager
2. Run:
   ```bash
   TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
   curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-ipv4
   ```

**Expected Result:** Empty response or 404 error

**Status:** ☐ Pass ☐ Fail

---

### Test 2: Verify Internet Isolation

**What it tests:** Instance cannot reach public internet

**Steps:**
1. From Session Manager, run:
   ```bash
   curl -I https://www.google.com --connect-timeout 5
   ```

**Expected Result:** Connection timeout or failure

**Status:** ☐ Pass ☐ Fail

---

### Test 3: Verify AWS Service Access via VPC Endpoints

**What it tests:** Instance can reach AWS services without internet

**Steps:**
1. From Session Manager, run:
   ```bash
   aws sts get-caller-identity
   ```

**Expected Result:** Returns account ID and role ARN

**Status:** ☐ Pass ☐ Fail

---

## IAM and Access Control Tests

### Test 4: Verify IAM Role Attached

**What it tests:** EC2 uses IAM role, not access keys

**Steps:**
1. From Session Manager, run:
   ```bash
   TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
   curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/
   ```

**Expected Result:** Shows `EC2Role` (or your role name)

**Status:** ☐ Pass ☐ Fail

---

### Test 5: Verify IMDSv2 Enforcement

**What it tests:** Metadata requires session token (IMDSv2)

**Steps:**
1. Try accessing metadata without token:
   ```bash
   curl http://169.254.169.254/latest/meta-data/instance-id
   ```

**Expected Result:** 401 Unauthorized or empty response

**Status:** ☐ Pass ☐ Fail

---

### Test 6: Verify Least Privilege (EC2Role)

**What it tests:** EC2 role has only necessary permissions

**Steps:**
1. Try unauthorized action:
   ```bash
   aws ec2 describe-instances
   ```

**Expected Result:** Access Denied error (unless EC2 permissions added)

**Status:** ☐ Pass ☐ Fail

---

## Storage Security Tests

### Test 7: Verify S3 Block Public Access

**What it tests:** S3 buckets cannot be made public

**Steps:**
1. AWS Console → S3 → Select CloudTrail bucket
2. Check "Block all public access" is enabled
3. Try to make bucket public (should be prevented)

**Expected Result:** Cannot enable public access

**Status:** ☐ Pass ☐ Fail

---

### Test 8: Verify S3 Encryption

**What it tests:** S3 bucket has default encryption

**Steps:**
1. S3 → CloudTrail bucket → Properties → Default encryption

**Expected Result:** Shows "Server-side encryption enabled (SSE-S3)"

**Status:** ☐ Pass ☐ Fail

---

### Test 9: Verify S3 Versioning

**What it tests:** Bucket versioning protects against deletion

**Steps:**
1. S3 → CloudTrail bucket → Properties → Bucket Versioning

**Expected Result:** Shows "Enabled"

**Status:** ☐ Pass ☐ Fail

---

### Test 10: Verify EBS Encryption

**What it tests:** EC2 volumes are encrypted at rest

**Steps:**
1. EC2 → Instances → Select instance → Storage tab
2. Click on volume ID
3. Check "Encryption" field

**Expected Result:** Shows "Encrypted"

**Status:** ☐ Pass ☐ Fail

---

## Logging and Monitoring Tests

### Test 11: Verify CloudTrail Logging

**What it tests:** All API activity is being logged

**Steps:**
1. CloudTrail → Event history
2. Filter: Last hour
3. Look for recent events (EC2 actions, IAM activity, etc.)

**Expected Result:** Events visible within 15 minutes of activity

**Status:** ☐ Pass ☐ Fail

---

### Test 12: Verify CloudWatch Alarms Active

**What it tests:** Security alarms are monitoring

**Steps:**
1. CloudWatch → Alarms → All alarms
2. Check status of all 5 alarms

**Expected Result:** All in "OK" state (or "INSUFFICIENT_DATA" if recent)

**Status:** ☐ Pass ☐ Fail

---

### Test 13: Test CloudWatch Alarm (Security Group Change)

**What it tests:** Alarms trigger on security events

**Steps:**
1. Modify any security group (add harmless rule)
2. Wait 5 minutes
3. Check email for SNS notification
4. Revert the change

**Expected Result:** Email received, alarm shows "ALARM" state

**Status:** ☐ Pass ☐ Fail

---

## Access Method Tests

### Test 14: Verify Session Manager Access

**What it tests:** Remote access works without SSH

**Steps:**
1. Systems Manager → Session Manager → Start session
2. Select EC2 instance
3. Verify shell access

**Expected Result:** Shell prompt appears, no SSH keys needed

**Status:** ☐ Pass ☐ Fail

---

### Test 15: Verify SSH is Blocked

**What it tests:** SSH port is not exposed

**Steps:**
1. Check security group `ec2-private-sg`
2. Verify no inbound rule for port 22

**Expected Result:** No SSH rule exists

**Status:** ☐ Pass ☐ Fail

---

## VPC Endpoint Tests

### Test 16: Verify VPC Endpoints Exist

**What it tests:** Required endpoints are configured

**Steps:**
1. VPC → Endpoints
2. Verify these exist and are "Available":
   - `com.amazonaws.[region].ssm`
   - `com.amazonaws.[region].ssmmessages`
   - `com.amazonaws.[region].ec2messages`

**Expected Result:** All 3 endpoints show "Available" status

**Status:** ☐ Pass ☐ Fail

---

### Test 17: Verify Endpoint Security Groups

**What it tests:** Endpoints allow traffic from EC2

**Steps:**
1. Select each endpoint → Security groups tab
2. Verify `ssmendpoint-sg` is attached
3. Check `ssmendpoint-sg` has inbound rule: HTTPS from `ec2-private-sg`

**Expected Result:** Correct security group with proper rules

**Status:** ☐ Pass ☐ Fail

---

### Test 18: Verify Private DNS Enabled

**What it tests:** Endpoints resolve via private DNS

**Steps:**
1. VPC → Endpoints → Select SSM endpoint
2. Check "Private DNS names" field

**Expected Result:** Shows "Enabled"

**Status:** ☐ Pass ☐ Fail

---

## Account Security Tests

### Test 19: Verify Root Account MFA

**What it tests:** Root account is secured

**Steps:**
1. Sign in as root user (or check via admin)
2. IAM → Dashboard → Security recommendations
3. Check MFA status

**Expected Result:** "MFA is activated"

**Status:** ☐ Pass ☐ Fail

---

### Test 20: Verify No Root Access Keys

**What it tests:** Root has no programmatic access

**Steps:**
1. Sign in as root
2. My Security Credentials → Access keys

**Expected Result:** No active access keys

**Status:** ☐ Pass ☐ Fail

---

### Test 21: Verify Budget Alert

**What it tests:** Cost monitoring is active

**Steps:**
1. Billing → Budgets
2. Verify budget exists with email notification

**Expected Result:** Budget configured with alert threshold

**Status:** ☐ Pass ☐ Fail

---

## Final Security Score

**Total Tests:** 21  
**Passed:** ___  
**Failed:** ___  
**Success Rate:** ___%

---

## Recommended Pass Rate

- **90-100%** (19-21 passed): Excellent security posture ✅
- **80-89%** (17-18 passed): Good, minor improvements needed ⚠️
- **Below 80%** (<17 passed): Review failed tests, significant gaps ❌

---

## Failed Test Remediation

If any tests fail, refer to:
- `troubleshooting.md` for common issues
- `setup-guide.md` to verify correct configuration
- CloudTrail logs for detailed error information

---

## Testing Frequency

**Before project demo:** Run all tests  
**After any changes:** Run relevant tests  
**Monthly (if live):** Run all tests to verify no drift

---

## Documentation

After completing tests:
1. Screenshot passing tests (especially Session Manager, alarms)
2. Document any failures and how you resolved them
3. Add test results to project README
4. Include in `screenshots/` folder

---

## Quick Test Script

Run this comprehensive validation from Session Manager:

```bash
#!/bin/bash
echo "=== Security Validation ==="
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

echo "✓ User: $(whoami)"
echo "✓ Private IP: $(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/local-ipv4)"
echo "✓ IAM Role: $(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/)"

curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-ipv4 > /dev/null 2>&1
[ $? -ne 0 ] && echo "✓ No public IP" || echo "✗ Public IP found"

timeout 3 curl -Is https://www.google.com > /dev/null 2>&1
[ $? -ne 0 ] && echo "✓ Internet blocked" || echo "✗ Internet accessible"

aws sts get-caller-identity > /dev/null 2>&1
[ $? -eq 0 ] && echo "✓ AWS API accessible" || echo "✗ AWS API failed"

echo "=== Tests Complete ==="
```
