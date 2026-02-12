# IAM Configuration

Identity and Access Management (IAM) setup for secure, least-privilege access control.

---

## Overview

This environment uses **IAM roles** instead of long-lived access keys to follow AWS security best practices.

**Core Principle:** Grant minimum permissions necessary for each task.

---

## IAM Architecture

### Role-Based Access Model

```
Human Users → Assume Roles (with MFA) → Temporary Credentials
EC2 Instances → IAM Role (via instance profile) → Temporary Credentials
```

**Benefits:**
- No long-lived credentials
- Automatic credential rotation
- Centralized permission management
- Complete audit trail in CloudTrail

---

## IAM Roles

Three roles provide different levels of access to the AWS environment.

### Role 1: AdminRole

**Purpose:** Full administrative access for account management

**Who Can Use It:**
- IAM users in the same AWS account
- **Requirement:** MFA must be enabled and active

**Permissions:**
- AWS managed policy: `AdministratorAccess`
- Grants full access to all AWS services and resources

**Trust Policy:**
```json
{
  "Principal": {
    "AWS": "arn:aws:iam::ACCOUNT_ID:root"
  },
  "Condition": {
    "Bool": {
      "aws:MultiFactorAuthPresent": "true"
    }
  }
}
```

**When to Use:**
- Initial environment setup
- Creating/modifying IAM policies
- Configuring security settings
- Emergency access for troubleshooting

**Security Features:**
- MFA required (cannot be bypassed)
- All assumptions logged in CloudTrail
- Session expires after 1 hour (default)

---

### Role 2: ReadOnlyRole

**Purpose:** View-only access for auditing and inspection

**Who Can Use It:**
- IAM users in the same AWS account
- **Requirement:** MFA must be enabled and active

**Permissions:**
- AWS managed policy: `ReadOnlyAccess`
- Can view all resources but cannot modify anything

**Trust Policy:**
```json
{
  "Principal": {
    "AWS": "arn:aws:iam::ACCOUNT_ID:root"
  },
  "Condition": {
    "Bool": {
      "aws:MultiFactorAuthPresent": "true"
    }
  }
}
```

**What It Allows:**
- `Describe*`, `List*`, `Get*` operations on all services
- View security group rules
- Read CloudTrail logs
- Inspect EC2 instance details

**What It Prevents:**
- Creating, modifying, or deleting resources
- Changing configurations
- Running or stopping instances
- Modifying IAM policies

**When to Use:**
- Security audits
- Compliance reviews
- Junior team member access
- Troubleshooting without risk of changes

---

### Role 3: EC2Role

**Purpose:** Minimum permissions for EC2 instances to function

**Who Can Use It:**
- EC2 service only (not human users)
- Automatically assumed by instances with this role attached

**Permissions:**
Three statement groups provide specific capabilities:

#### Statement 1: Systems Manager Access
```json
{
  "Sid": "SystemsManagerAccess",
  "Action": [
    "ssm:UpdateInstanceInformation",
    "ssmmessages:CreateControlChannel",
    "ssmmessages:CreateDataChannel",
    "ssmmessages:OpenControlChannel",
    "ssmmessages:OpenDataChannel"
  ]
}
```
**Purpose:** Enables Session Manager connectivity

#### Statement 2: EC2 Messages Access
```json
{
  "Sid": "EC2MessagesAccess",
  "Action": [
    "ec2messages:AcknowledgeMessage",
    "ec2messages:DeleteMessage",
    "ec2messages:FailMessage",
    "ec2messages:GetEndpoint",
    "ec2messages:GetMessages",
    "ec2messages:SendReply"
  ]
}
```
**Purpose:** Allows instance to communicate with Systems Manager

#### Statement 3: CloudWatch Logs Access
```json
{
  "Sid": "CloudWatchLogsAccess",
  "Action": [
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents",
    "logs:DescribeLogStreams"
  ],
  "Resource": "arn:aws:logs:*:*:log-group:/aws/ssm/*"
}
```
**Purpose:** Send session logs to CloudWatch (optional but recommended)

**Trust Policy:**
```json
{
  "Principal": {
    "Service": "ec2.amazonaws.com"
  },
  "Action": "sts:AssumeRole"
}
```

**What EC2Role Does NOT Have:**
- ❌ No S3 access (unless explicitly added)
- ❌ No EC2 instance management permissions
- ❌ No IAM permissions
- ❌ No ability to modify security groups
- ❌ No access to other AWS services

**Why This Matters:**
If an EC2 instance is compromised, the attacker only has SSM and logging permissions—they cannot:
- Launch new instances
- Access S3 data
- Modify IAM policies
- Change network configuration

---

## How IAM Roles Work

### Trust Policies vs Permission Policies

**Trust Policy (Who):**
- Defines WHO can assume the role
- Principal: IAM user, AWS service, or another account
- Can include conditions (like MFA requirement)

**Permission Policy (What):**
- Defines WHAT the role can do once assumed
- Actions: Specific API operations allowed
- Resources: Which AWS resources the actions apply to

**Example:**
```
AdminRole:
  Trust Policy → IAM users with MFA can assume this role
  Permission Policy → Full access to all AWS services
```

---

## Assuming Roles (Human Users)

### How Admin Users Assume AdminRole

**Via AWS Console:**
1. Log in as IAM user with MFA
2. Click user name (top right) → Switch Role
3. Enter:
   - Account: Your account ID
   - Role: AdminRole
4. Click "Switch Role"
5. Now operating with admin permissions
6. Session expires after 1 hour

**Via AWS CLI:**
```bash
aws sts assume-role \
  --role-arn arn:aws:iam::ACCOUNT_ID:role/AdminRole \
  --role-session-name admin-session \
  --serial-number arn:aws:iam::ACCOUNT_ID:mfa/USERNAME \
  --token-code 123456
```

Returns temporary credentials (Access Key, Secret Key, Session Token) valid for 1 hour.

---

## EC2 Instance Profiles

EC2 instances cannot directly use IAM roles—they use **instance profiles**.

### What is an Instance Profile?

**Instance Profile:**
- Container that holds an IAM role
- Attached to EC2 instance at launch
- Provides credentials via instance metadata service

**Relationship:**
```
EC2 Instance → Instance Profile → IAM Role → Permissions
```

### How It Works

1. **Launch:** EC2 instance launched with instance profile attached
2. **Credentials:** AWS automatically provides temporary credentials
3. **Metadata:** Credentials available via `http://169.254.169.254/latest/meta-data/iam/security-credentials/`
4. **Rotation:** Credentials automatically rotate every 6 hours
5. **Usage:** AWS CLI/SDK automatically use these credentials

**Example:**
```bash
# From inside EC2 instance - no configuration needed
aws sts get-caller-identity

# Returns:
{
  "Account": "123456789012",
  "UserId": "AIDAI...:i-0123456789abcdef",
  "Arn": "arn:aws:sts::123456789012:assumed-role/EC2Role/i-0123456789abcdef"
}
```

---

## IAM Best Practices Implemented

✅ **Least Privilege:** Each role has only necessary permissions  
✅ **No Access Keys:** Use roles instead of long-lived credentials  
✅ **MFA Required:** Human access requires multi-factor authentication  
✅ **Temporary Credentials:** All credentials expire automatically  
✅ **Role Separation:** Different roles for different purposes  
✅ **Service Roles:** EC2 uses service-specific trust policy  
✅ **Audit Trail:** All role assumptions logged in CloudTrail

---

## IAM Security Features

### Multi-Factor Authentication (MFA)

**Requirement:** MFA required for AdminRole and ReadOnlyRole

**Enforcement Mechanism:** Trust policy condition
```json
"Condition": {
  "Bool": {
    "aws:MultiFactorAuthPresent": "true"
  }
}
```

**What This Prevents:**
- Stolen password alone cannot assume admin role
- Requires both password AND MFA device
- Greatly reduces risk of unauthorized access

**MFA Options:**
- Virtual MFA app (Google Authenticator, Authy)
- Hardware MFA device (YubiKey)
- SMS (not recommended)

---

### Credential Rotation

**Human Users:**
- Temporary credentials from assume role expire after 1 hour
- Must re-assume role to continue
- No permanent credentials to rotate

**EC2 Instances:**
- Credentials automatically rotate every 6 hours
- Happens transparently to applications
- AWS SDK handles rotation automatically

**Benefits:**
- Compromised credentials expire quickly
- No manual rotation required
- Reduced risk of credential exposure

---

## IAM Monitoring

All IAM activity is logged to CloudTrail:

### What Gets Logged

**Role Assumptions:**
- Event: `AssumeRole`
- Who assumed the role
- When it was assumed
- Source IP address
- MFA device used (if applicable)

**Permission Usage:**
- Every API call made using role credentials
- Success or failure
- Resources accessed
- Changes made

**Policy Changes:**
- Creating/modifying/deleting roles
- Attaching/detaching policies
- Changing trust relationships

### CloudWatch Alarms for IAM

**Active Alarm:** `IAMPolicyChangesAlarm`

**Triggers On:**
- CreatePolicy, DeletePolicy
- AttachRolePolicy, DetachRolePolicy
- PutUserPolicy, PutRolePolicy
- Any IAM permission modification

**Purpose:** Immediate notification of permission changes

---

## Permission Boundaries (Not Implemented)

**What They Are:**
- Maximum permissions an IAM entity can have
- Additional safety layer beyond policies
- Not implemented in this project (scope)

**When to Use:**
- Delegate role creation to non-admin users
- Prevent privilege escalation
- Enterprise environments with multiple teams

---

## Service Control Policies (Not Applicable)

**What They Are:**
- Organization-wide permission guardrails
- Require AWS Organizations
- Not applicable to single-account setup

**Production Use Case:**
- Prevent any user from disabling CloudTrail
- Restrict regions where resources can be created
- Enforce encryption requirements

---

## IAM Role Usage Examples

### Example 1: Admin Performing Maintenance

```bash
# 1. Admin user logs in with MFA
# 2. Assumes AdminRole
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/AdminRole \
  --role-session-name maintenance-2024 \
  --serial-number arn:aws:iam::123456789012:mfa/admin \
  --token-code 123456

# 3. Export temporary credentials
export AWS_ACCESS_KEY_ID=ASIA...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...

# 4. Perform admin tasks
aws ec2 describe-instances
aws iam create-role ...

# 5. Credentials expire after 1 hour
```

---

### Example 2: EC2 Instance Using IAM Role

```bash
# Application code on EC2 instance
# No credentials needed - AWS SDK handles automatically

import boto3

# This works without any configuration
# SDK gets credentials from instance metadata
ssm = boto3.client('ssm')
response = ssm.describe_instance_information()
```

**Behind the Scenes:**
1. AWS SDK checks for credentials
2. Finds instance profile attached
3. Queries metadata service for temp credentials
4. Uses credentials transparently
5. Rotates before expiration

---

## IAM Testing

### Test AdminRole

**Verify MFA requirement:**
```bash
# Try to assume without MFA (should fail)
aws sts assume-role \
  --role-arn arn:aws:iam::ACCOUNT_ID:role/AdminRole \
  --role-session-name test

# Expected: Error - MFA required
```

**With MFA (should succeed):**
```bash
aws sts assume-role \
  --role-arn arn:aws:iam::ACCOUNT_ID:role/AdminRole \
  --role-session-name test \
  --serial-number arn:aws:iam::ACCOUNT_ID:mfa/username \
  --token-code 123456

# Expected: Returns temporary credentials
```

---

### Test ReadOnlyRole

**Can read:**
```bash
# After assuming ReadOnlyRole
aws ec2 describe-instances  # Success
aws s3 ls  # Success
```

**Cannot write:**
```bash
aws ec2 run-instances ...  # Fails with AccessDenied
aws s3 cp file.txt s3://bucket/  # Fails with AccessDenied
```

---

### Test EC2Role

**From EC2 instance via Session Manager:**
```bash
# Should work - has SSM permissions
aws sts get-caller-identity

# Should fail - no EC2 permissions
aws ec2 describe-instances

# Should fail - no S3 permissions (unless added)
aws s3 ls
```

---

## IAM Costs

**IAM Service:** Completely free

**No Charges For:**
- Creating roles
- Policies
- Users
- Groups
- Role assumptions
- Credential rotations

**Related Costs:**
- CloudTrail logging (first trail free)
- MFA devices (if purchasing hardware tokens)

---

## IAM Validation Checklist

- [ ] Root account has MFA enabled
- [ ] Root account has no access keys
- [ ] AdminRole requires MFA
- [ ] ReadOnlyRole requires MFA
- [ ] EC2Role attached to EC2 instance
- [ ] EC2Role has minimum necessary permissions
- [ ] No IAM users with access keys (except admin user)
- [ ] All role assumptions logged in CloudTrail
- [ ] IAM policy changes trigger CloudWatch alarm

---

## Troubleshooting IAM

### Cannot Assume Role

**Error:** "User is not authorized to perform: sts:AssumeRole"

**Causes:**
1. MFA not provided (for AdminRole/ReadOnlyRole)
2. User not in trust policy principal
3. User's account doesn't match trust policy
4. MFA device not activated

**Solution:**
- Verify MFA is active
- Check trust policy allows your user/account
- Provide correct MFA token

---

### EC2 Instance Can't Access AWS Services

**Error:** "Unable to locate credentials"

**Causes:**
1. No IAM role attached to instance
2. Instance profile not created
3. Role has wrong trust policy
4. IMDSv2 required but not configured

**Solution:**
```bash
# Check if role attached
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Should return role name
# If empty, attach instance profile via console
```

---

### Access Denied Errors

**Error:** "User is not authorized to perform: [action]"

**Causes:**
1. Role doesn't have required permission
2. Resource policy blocks access
3. Service control policy restricts action
4. Permission boundary limits access

**Solution:**
- Check role's permission policy
- Add required action to policy
- Verify resource-based policies
- Check CloudTrail for exact error

---

## IAM Evolution

### Current State (Learning Environment)

- 3 IAM roles (Admin, ReadOnly, EC2)
- MFA required for human access
- Least privilege for EC2
- Manual role assumption

### Production Improvements

**For Small Teams:**
- Add DeveloperRole for app deployment
- Create separate roles per application
- Implement permission boundaries

**For Enterprises:**
- AWS Organizations + SSO
- Service control policies (SCPs)
- Cross-account access roles
- Automated provisioning with IaC

---

## Quick Reference

### IAM Role Summary

| Role | Principal | MFA Required | Permissions | Use Case |
|------|-----------|--------------|-------------|----------|
| **AdminRole** | IAM users | Yes ✅ | Full access | Setup, maintenance |
| **ReadOnlyRole** | IAM users | Yes ✅ | Read-only | Auditing, inspection |
| **EC2Role** | EC2 service | No | SSM + logging | Instance operations |

### Common Commands

```bash
# List roles
aws iam list-roles

# Get role details
aws iam get-role --role-name EC2Role

# List attached policies
aws iam list-attached-role-policies --role-name EC2Role

# Assume role
aws sts assume-role --role-arn ARN --role-session-name SESSION

# Get current identity
aws sts get-caller-identity
```

---

## References

- [IAM Roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)
- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [Using Instance Profiles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html)
- [MFA for AWS](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa.html)
- [Policy Evaluation Logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html)
