# CloudWatch Alarms Configuration

This document details the CloudWatch alarms configured for security monitoring in the AWS environment.

## Overview

Five critical security alarms have been configured to detect suspicious activity and configuration changes. All alarms send notifications via SNS to enable rapid response to security events.

## Prerequisites

- CloudTrail enabled and logging to CloudWatch Logs
- CloudWatch Log Group: `/aws/cloudtrail/secure-environment`
- SNS Topic: `SecurityAlerts` with email subscription configured

## Configured Alarms

### 1. Root Account Usage Alarm

**Alarm Name:** `RootAccountUsageAlarm`

**What It Monitors:**  
Detects any usage of the AWS root account credentials.

**Metric Filter Pattern:**
```
{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }
```

**Threshold:** >= 1 event in 5 minutes

**Why It Matters:**  
Root account usage is a critical security event. The root account has unrestricted access to all AWS resources and should only be used for initial account setup and specific account-level tasks that require root access. Any root account usage should be investigated immediately.

**Response Actions:**
1. Verify if the root account usage was authorized
2. Check CloudTrail logs to identify what actions were performed
3. If unauthorized, immediately:
   - Change root account password
   - Review and revoke any API keys
   - Enable MFA if not already enabled
   - Check for any unauthorized resource creation or configuration changes

---

### 2. Unauthorized API Calls Alarm

**Alarm Name:** `UnauthorizedAPICallsAlarm`

**What It Monitors:**  
Detects failed API calls due to insufficient permissions (AccessDenied or UnauthorizedOperation errors).

**Metric Filter Pattern:**
```
{ ($.errorCode = "*UnauthorizedOperation") || ($.errorCode = "AccessDenied*") }
```

**Threshold:** >= 5 events in 5 minutes

**Why It Matters:**  
Multiple unauthorized API calls may indicate:
- An attacker probing for permissions
- Compromised credentials being used to enumerate access
- Misconfigured applications or scripts
- Privilege escalation attempts

**Response Actions:**
1. Review CloudTrail logs to identify the IAM principal making unauthorized calls
2. Check if the user/role credentials may be compromised
3. Verify the source IP address is expected
4. If suspicious:
   - Disable the IAM user or rotate credentials immediately
   - Review recent activity from that principal
   - Check for any successful actions that preceded the unauthorized attempts

---

### 3. IAM Policy Changes Alarm

**Alarm Name:** `IAMPolicyChangesAlarm`

**What It Monitors:**  
Detects any modifications to IAM policies, including creation, deletion, attachment, or detachment of policies.

**Metric Filter Pattern:**
```
{($.eventName=DeleteGroupPolicy)||($.eventName=DeleteRolePolicy)||($.eventName=DeleteUserPolicy)||($.eventName=PutGroupPolicy)||($.eventName=PutRolePolicy)||($.eventName=PutUserPolicy)||($.eventName=CreatePolicy)||($.eventName=DeletePolicy)||($.eventName=CreatePolicyVersion)||($.eventName=DeletePolicyVersion)||($.eventName=AttachRolePolicy)||($.eventName=DetachRolePolicy)||($.eventName=AttachUserPolicy)||($.eventName=DetachUserPolicy)||($.eventName=AttachGroupPolicy)||($.eventName=DetachGroupPolicy)}
```

**Threshold:** >= 1 event

**Why It Matters:**  
IAM policy changes directly affect permissions and access controls. Unauthorized changes could:
- Grant attackers elevated privileges
- Weaken security controls
- Provide access to sensitive resources
- Enable lateral movement within the environment

**Response Actions:**
1. Review the specific IAM policy change in CloudTrail
2. Verify if the change was authorized and follows change management procedures
3. Check who made the change and from what source IP
4. If unauthorized:
   - Immediately revert the policy change
   - Review all actions taken by the modified principal since the change
   - Investigate if credentials are compromised
   - Consider rotating credentials for affected principals

---

### 4. Security Group Changes Alarm

**Alarm Name:** `SecurityGroupChangesAlarm`

**What It Monitors:**  
Detects any modifications to security groups, including rule additions, removals, or security group creation/deletion.

**Metric Filter Pattern:**
```
{ ($.eventName = AuthorizeSecurityGroupIngress) || ($.eventName = AuthorizeSecurityGroupEgress) || ($.eventName = RevokeSecurityGroupIngress) || ($.eventName = RevokeSecurityGroupEgress) || ($.eventName = CreateSecurityGroup) || ($.eventName = DeleteSecurityGroup) }
```

**Threshold:** >= 1 event

**Why It Matters:**  
Security groups are the first line of defense for network security. Changes could:
- Open resources to the public internet (0.0.0.0/0)
- Allow unauthorized protocols or ports
- Weaken network isolation
- Expose internal services

**Response Actions:**
1. Review the specific security group change in CloudTrail
2. Check if ports were opened to 0.0.0.0/0 (public internet)
3. Verify the change was authorized
4. If suspicious:
   - Immediately revert unauthorized security group rules
   - Check if any instances were accessed during the exposure window
   - Review VPC Flow Logs for unexpected connections
   - Verify no resources were compromised while exposed

---

### 5. Console Sign-In Failures Alarm

**Alarm Name:** `ConsoleSignInFailuresAlarm`

**What It Monitors:**  
Detects multiple failed AWS Console login attempts, which may indicate brute force attacks or credential stuffing.

**Metric Filter Pattern:**
```
{ ($.eventName = ConsoleLogin) && ($.errorMessage = "Failed authentication") }
```

**Threshold:** >= 3 failures in 5 minutes

**Why It Matters:**  
Multiple login failures may indicate:
- Brute force attack attempting to guess passwords
- Compromised credentials being tested
- Legitimate user with forgotten credentials (less concerning)
- Automated attack tools targeting the account

**Response Actions:**
1. Check CloudTrail for the source IP address of failed attempts
2. Verify if the IP is known or from an unexpected location
3. Check if failures are for a specific IAM user
4. If attack suspected:
   - Consider blocking the source IP at the network level if possible
   - Ensure MFA is enabled on all user accounts
   - Review password policies to ensure they meet complexity requirements
   - Notify affected users to change passwords if credential compromise suspected
   - Enable AWS GuardDuty for enhanced threat detection (if not already enabled)

---

## Notification Configuration

All alarms send notifications to the SNS topic: `SecurityAlerts`

**Email Recipients:**  
- Primary: Security team email
- Secondary: Administrator email

**To modify recipients:**
1. Go to SNS → Topics → SecurityAlerts
2. Create subscription → Email
3. Confirm subscription via email

---

## Testing Alarms

To verify alarms are working correctly:

### Test Root Account Usage Alarm:
```bash
# Sign in to AWS Console using root account credentials
# Alarm should trigger within 5-10 minutes
```

### Test Unauthorized API Calls:
```bash
# From EC2 instance without proper permissions, attempt:
aws ec2 describe-instances --region us-west-2
# (if your role doesn't have EC2 permissions in us-west-2)
```

### Test Security Group Changes:
```bash
# Add a temporary rule to any security group
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxx \
  --protocol tcp \
  --port 22 \
  --cidr 10.0.0.0/8

# Then remove it
aws ec2 revoke-security-group-ingress \
  --group-id sg-xxxxx \
  --protocol tcp \
  --port 22 \
  --cidr 10.0.0.0/8
```

---

## Alarm States

**OK** - No events detected, system operating normally  
**ALARM** - Threshold breached, investigation required  
**INSUFFICIENT_DATA** - Not enough data to evaluate (usually after initial setup)

---

## Cost Considerations

**CloudWatch Metrics:** First 10 custom metrics free, then $0.30/metric/month  
**CloudWatch Alarms:** First 10 alarms free, then $0.10/alarm/month  
**SNS Notifications:** First 1,000 email notifications free, then $2.00 per 100,000 notifications  

**Estimated Monthly Cost:** ~$0 (within free tier for this configuration)

---

## Maintenance

### Regular Reviews:
- **Weekly:** Check alarm history for any triggered events
- **Monthly:** Review alarm thresholds for false positives
- **Quarterly:** Update response procedures as needed

### Alarm Tuning:
If experiencing false positives, consider:
- Adjusting thresholds (e.g., increase from 1 to 2-3 events)
- Narrowing metric filter patterns to exclude known safe operations
- Adding time-based suppressions for maintenance windows

---

## Future Enhancements

Consider adding alarms for:
- VPC configuration changes (route table, NACL modifications)
- S3 bucket policy changes
- EC2 instance state changes (unexpected starts/stops)
- AWS Config compliance violations
- High AWS cost anomalies
- MFA device deactivation events

---

## References

- [AWS CloudWatch Alarms Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html)
- [CloudTrail Log Event Reference](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference.html)
- [AWS Security Best Practices - Monitoring and Alerting](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/detection.html)
