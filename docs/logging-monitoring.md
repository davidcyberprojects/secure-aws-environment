# Logging and Monitoring

Comprehensive logging and monitoring configuration for visibility and security incident detection.

---

## Overview

This environment implements multiple logging layers to ensure all activity is captured, monitored, and actionable.

**Core Principle:** You can't protect what you can't see.

---

## Logging Architecture

### Three-Layer Approach

```
Layer 1: CloudTrail → What happened (API activity)
Layer 2: CloudWatch Logs → Where it's stored (log aggregation)
Layer 3: CloudWatch Alarms → When to alert (incident detection)
```

---

## CloudTrail Configuration

CloudTrail provides a complete audit trail of all AWS account activity.

### Trail Details

**Trail Name:** `secure-environment-trail`  
**Status:** Logging enabled ✅  
**Scope:** All AWS regions (multi-region trail)  
**Log File Validation:** Enabled ✅

### What Gets Logged

**Management Events (Enabled):**
- IAM actions (user logins, role assumptions, policy changes)
- EC2 actions (instance start/stop, security group modifications)
- VPC changes (endpoint creation, route table updates)
- S3 bucket operations (policy changes, bucket creation)
- CloudTrail modifications (trail updates)
- All AWS service API calls

**Data Events (Not Enabled):**
- S3 object-level operations (get/put individual files)
- Lambda function invocations
- High volume, high cost - excluded for this project

**Insights Events (Not Enabled):**
- ML-based anomaly detection
- Cost consideration - excluded

---

### Log Storage

**Primary Storage:** S3 bucket  
**Bucket Name:** `cloudtrail-logs-[unique-name]`  
**Encryption:** SSE-S3 (server-side encryption)  
**Versioning:** Enabled ✅

**S3 Bucket Configuration:**
- Block all public access ✅
- Lifecycle policy: Retain 90 days (configurable)
- Log file validation enabled (detect tampering)

**Secondary Storage:** CloudWatch Logs  
**Log Group:** `/aws/cloudtrail/secure-environment`  
**Retention:** Not set (logs kept indefinitely by default)

---

### Log Delivery Timeline

| Event Occurs | CloudTrail Processes | Logs Delivered to S3 | Logs in CloudWatch | Available in Event History |
|--------------|---------------------|---------------------|-------------------|---------------------------|
| T+0 | T+0 | T+15 min | T+5 min | T+15 min |

**Key Takeaway:** CloudWatch Logs are faster (5 min) than S3 delivery (15 min)

---

## CloudWatch Logs

Centralized log aggregation and real-time processing.

### Log Groups

**Primary Log Group:** `/aws/cloudtrail/secure-environment`

**Purpose:**
- Receives all CloudTrail events in real-time
- Enables metric filters for security events
- Allows CloudWatch alarms on API activity

**Configuration:**
- Retention: Never expire (can be changed to 7, 30, 90 days for cost savings)
- Encryption: None (CloudTrail logs already encrypted in S3)
- Size: ~100 MB/month for typical usage

**Cost:** $0.50/GB/month for storage + $0.50/GB for ingestion

---

### Metric Filters

Five metric filters transform CloudTrail logs into CloudWatch metrics:

| Filter Name | Metric Name | Pattern | Purpose |
|-------------|-------------|---------|---------|
| RootAccountUsage | RootAccountUsageCount | Root user activity | Detect root account usage |
| UnauthorizedAPICalls | UnauthorizedAPICallsCount | Access denied errors | Detect unauthorized access attempts |
| IAMPolicyChanges | IAMPolicyChangesCount | IAM policy modifications | Monitor permission changes |
| SecurityGroupChanges | SecurityGroupChangesCount | Security group updates | Track network rule changes |
| ConsoleSignInFailures | ConsoleSignInFailureCount | Failed console logins | Detect brute force attempts |

**How They Work:**
1. CloudTrail logs events to CloudWatch Logs
2. Metric filter scans log entries for pattern matches
3. When pattern found, increment metric value
4. CloudWatch alarm monitors metric and triggers notification

---

## CloudWatch Alarms

Real-time security event notifications via SNS email.

### Alarm Configuration

**SNS Topic:** `SecurityAlerts`  
**Notification Method:** Email  
**Alarm State:**
- **OK** - No security events detected
- **ALARM** - Threshold breached, investigate
- **INSUFFICIENT_DATA** - Not enough data yet

### Active Alarms

| Alarm Name | Threshold | Evaluation Period | Purpose |
|------------|-----------|-------------------|---------|
| RootAccountUsageAlarm | >= 1 event | 5 minutes | Immediate alert on root usage |
| UnauthorizedAPICallsAlarm | >= 5 events | 5 minutes | Multiple auth failures detected |
| IAMPolicyChangesAlarm | >= 1 event | 5 minutes | Any IAM permission change |
| SecurityGroupChangesAlarm | >= 1 event | 5 minutes | Network rule modification |
| ConsoleSignInFailuresAlarm | >= 3 events | 5 minutes | Potential brute force |

**Notification Flow:**
```
Event Occurs → CloudTrail Logs → Metric Filter → Metric Increments → Alarm Triggers → SNS Email Sent
```

**Response Time:** Typically 5-10 minutes from event to notification

---

## What Gets Logged Where

### CloudTrail Event History (Console)

**Retention:** 90 days (automatic, no cost)  
**Access:** CloudTrail → Event history  
**Use Case:** Quick lookups, recent activity review

**Example Queries:**
- Who created this EC2 instance?
- What actions did this user perform today?
- When was this security group modified?

---

### CloudTrail Logs in S3

**Retention:** 90 days (configurable with lifecycle policy)  
**Access:** Direct S3 download or query with Athena  
**Use Case:** Long-term storage, compliance, forensic analysis

**File Format:** JSON, one file per ~5 minutes of activity  
**File Location:** `s3://bucket-name/AWSLogs/ACCOUNT-ID/CloudTrail/REGION/YYYY/MM/DD/`

---

### CloudWatch Logs

**Retention:** Indefinite (unless set)  
**Access:** CloudWatch → Log groups → Search  
**Use Case:** Real-time monitoring, metric filters, alarms

**Search Capabilities:**
- Filter by time range
- Search by user, IP, event name
- JSON parsing and filtering

---

## Logging Best Practices Implemented

✅ **Centralized Logging:** All API calls to one trail  
✅ **Multi-Region:** Captures activity across all regions  
✅ **Log File Validation:** Detects if logs are modified  
✅ **Encryption:** Logs encrypted at rest in S3  
✅ **Versioning:** S3 versioning prevents accidental deletion  
✅ **Real-Time Alerts:** CloudWatch alarms for critical events  
✅ **Immutability:** CloudTrail logs can't be modified  

---

## Log Retention Strategy

### Current Configuration

| Log Type | Retention | Reason |
|----------|-----------|--------|
| **CloudTrail Event History** | 90 days | AWS default, free |
| **S3 Logs** | 90 days | Lifecycle policy, cost control |
| **CloudWatch Logs** | Indefinite | Not set (can be changed) |

### Recommended Production Retention

| Environment | CloudTrail S3 | CloudWatch Logs |
|-------------|---------------|-----------------|
| **Development** | 30 days | 7 days |
| **Production** | 1-7 years | 30-90 days |
| **Compliance** | 7+ years | As required by regulation |

**Cost Trade-off:** Longer retention = higher storage costs

---

## Querying Logs

### Quick Queries via Event History

**Find who created an instance:**
```
CloudTrail → Event history
Filter: Event name = RunInstances
Time range: Last 7 days
```

**Find all actions by a user:**
```
Filter: User name = admin-user
Time range: Today
```

---

### Advanced Queries with CloudWatch Logs Insights

```sql
# Find all unauthorized API calls in last hour
fields @timestamp, userIdentity.principalId, eventName, errorCode
| filter errorCode like /Unauthorized|AccessDenied/
| sort @timestamp desc
| limit 100
```

```sql
# Find all security group changes
fields @timestamp, eventName, requestParameters.groupId
| filter eventName like /SecurityGroup/
| sort @timestamp desc
```

---

### S3 Log Analysis with Athena (Advanced)

Create table pointing to CloudTrail S3 bucket, then query:

```sql
SELECT 
    useridentity.principalid,
    eventname,
    sourceipaddress,
    eventtime
FROM cloudtrail_logs
WHERE eventname = 'AuthorizeSecurityGroupIngress'
ORDER BY eventtime DESC;
```

**Cost:** $5 per TB scanned

---

## Monitoring Dashboard (Conceptual)

If you wanted to create a CloudWatch dashboard, it would show:

**Security Metrics:**
- Root account usage (line chart)
- Unauthorized API calls (line chart)
- IAM policy changes (event count)
- Security group changes (event count)
- Failed console logins (line chart)

**Operational Metrics:**
- CloudTrail logging status (binary)
- Log file delivery time (histogram)
- Alarm states (widget)

**Not implemented in this project** (cost optimization), but easy to add.

---

## Log Analysis Workflow

### When an Alarm Triggers

1. **Receive SNS Email** - "ALARM: SecurityGroupChangesAlarm"

2. **Check CloudWatch Alarm** 
   - See which metric triggered
   - View alarm history

3. **Query CloudTrail Logs**
   - CloudTrail → Event history
   - Filter by event name (e.g., "AuthorizeSecurityGroupIngress")
   - Find specific event

4. **Investigate Details**
   - Who: `userIdentity.principalId`
   - What: `eventName` and `requestParameters`
   - When: `eventTime`
   - Where: `sourceIPAddress`
   - Result: `errorCode` (success or failure)

5. **Take Action**
   - If authorized: Document and dismiss alarm
   - If unauthorized: Revert change, investigate credentials, rotate keys

---

## What's NOT Logged

**Excluded for Cost/Scope:**
- VPC Flow Logs (network traffic metadata)
- S3 data events (object-level operations)
- Lambda function invocations
- Application logs (would need CloudWatch agent)
- Database query logs (no RDS in this project)

**These would be enabled in production environments**

---

## Logging Costs

### Monthly Cost Breakdown

| Service | Configuration | Monthly Cost |
|---------|---------------|--------------|
| **CloudTrail** | 1 trail, management events | $0 (first trail free) |
| **S3 Storage** | ~1 GB CloudTrail logs | $0.02 |
| **CloudWatch Logs** | Ingestion + storage | $1.00 (estimate) |
| **CloudWatch Metrics** | 5 custom metrics | $0 (first 10 free) |
| **CloudWatch Alarms** | 5 alarms | $0 (first 10 free) |
| **SNS Notifications** | Email only | $0 (first 1,000 free) |

**Total Logging Cost:** ~$1.02/month

**Free Tier Benefits:**
- CloudTrail first trail: Free
- CloudWatch first 10 alarms: Free
- SNS first 1,000 emails: Free

---

## Logging Validation Checklist

- [ ] CloudTrail is logging (status = ON)
- [ ] Events appearing in Event History within 15 minutes
- [ ] S3 bucket receiving log files
- [ ] CloudWatch Logs receiving events
- [ ] All 5 metric filters created
- [ ] All 5 alarms in OK or INSUFFICIENT_DATA state
- [ ] SNS email subscription confirmed
- [ ] Test alarm triggers and sends email

---

## Security Incident Response

### Using Logs for Investigation

**Scenario:** Unauthorized security group change detected

**Steps:**
1. CloudWatch alarm triggers → Email received
2. Go to CloudTrail Event History
3. Filter: `eventName = AuthorizeSecurityGroupIngress`
4. Find the event, check:
   - User: `userIdentity.arn`
   - IP: `sourceIPAddress`
   - Changes: `requestParameters`
5. Verify if authorized or investigate compromise
6. Revert change if malicious
7. Document incident

**Timeline:** 5-10 minutes from event to investigation start

---

## Compliance and Audit

### What This Logging Provides

✅ **Who** - Every action tied to IAM principal  
✅ **What** - Full details of API calls and parameters  
✅ **When** - Precise timestamp for all events  
✅ **Where** - Source IP address captured  
✅ **Result** - Success or failure with error codes  
✅ **Immutability** - Logs can't be altered (with validation enabled)

**Use Cases:**
- Security audits
- Compliance reporting (PCI-DSS, HIPAA, SOC 2)
- Incident investigation
- User activity tracking
- Change management verification

---

## Log Retention Automation

### S3 Lifecycle Policy Example

Already configured to delete logs after 90 days:

```json
{
  "Rules": [{
    "Id": "DeleteOldLogs",
    "Status": "Enabled",
    "Expiration": {"Days": 90},
    "Filter": {"Prefix": "AWSLogs/"}
  }]
}
```

**Saves:** ~$0.02/GB/month in storage costs

---

### CloudWatch Logs Retention

Set retention on log group:

```bash
# Set 30-day retention
aws logs put-retention-policy \
  --log-group-name /aws/cloudtrail/secure-environment \
  --retention-in-days 30
```

**Options:** 1, 3, 5, 7, 14, 30, 60, 90, 120, 150, 180, 365, 400, 545, 731, 1827, 3653 days

---

## Future Enhancements

### Logging Improvements for Production

**VPC Flow Logs:**
- Capture network traffic metadata
- Cost: ~$0.50/GB ingested
- Use case: Detect port scanning, unusual traffic patterns

**GuardDuty:**
- ML-based threat detection
- Cost: $4.60/month (minimum)
- Use case: Automated threat identification

**AWS Config:**
- Track resource configuration changes
- Cost: ~$2/month per rule
- Use case: Compliance monitoring, config drift detection

**Centralized Logging (Multi-Account):**
- Aggregate logs from multiple accounts
- Use AWS Organizations + S3 bucket policies
- Use case: Enterprise environments

---

## Troubleshooting Logging

### CloudTrail Not Logging

**Check:**
1. Trail status is ON
2. S3 bucket policy allows CloudTrail to write
3. CloudWatch Logs role has correct permissions
4. No service control policies blocking CloudTrail

**Verify:**
```bash
aws cloudtrail get-trail-status --name secure-environment-trail
```

Expected: `"IsLogging": true`

---

### Alarms Not Triggering

**Check:**
1. Metric filter pattern is correct
2. CloudWatch Logs receiving events
3. Alarm threshold configuration
4. SNS subscription is confirmed

**Test:**
```bash
# Trigger security group alarm manually
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxx \
  --protocol tcp --port 22 --cidr 10.0.0.0/8

# Wait 5 minutes, check for alarm
```

---

### SNS Emails Not Received

**Check:**
1. Subscription status is "Confirmed"
2. Email not in spam folder
3. SNS topic has correct subscriptions
4. Alarm action configured to notify SNS topic

**Verify:**
```bash
aws sns list-subscriptions-by-topic \
  --topic-arn arn:aws:sns:region:account:SecurityAlerts
```

---

## Logging Commands

### Useful CLI Commands

```bash
# View recent CloudTrail events
aws cloudtrail lookup-events --max-results 10

# Check trail status
aws cloudtrail get-trail-status --name secure-environment-trail

# List metric filters
aws logs describe-metric-filters \
  --log-group-name /aws/cloudtrail/secure-environment

# View alarm status
aws cloudwatch describe-alarms

# Test SNS topic
aws sns publish \
  --topic-arn arn:aws:sns:region:account:SecurityAlerts \
  --message "Test notification"
```

---

## References

- [CloudTrail Documentation](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/)
- [CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/)
- [CloudWatch Alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html)
- [Monitoring Security Events](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudwatch-alarms-for-cloudtrail.html)
- [Log File Validation](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-log-file-validation-intro.html)
