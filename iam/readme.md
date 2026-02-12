# IAM Configuration

This document explains the IAM roles and policies configured in this secure AWS environment.

## Overview

This environment uses IAM roles rather than IAM users with access keys to follow the principle of least privilege. Three roles are defined, each with specific trust and permission policies:

- **AdminRole** - Full administrative access for environment setup and maintenance
- **ReadOnlyRole** - View-only access for auditing and inspection
- **EC2Role** - Limited permissions for EC2 instances to interact with AWS services

## IAM Roles Architecture

```
IAM Role = Trust Policy + Permission Policy

Trust Policy → WHO can assume the role
Permission Policy → WHAT the role can do once assumed
```

---

## Role 1: AdminRole

### Purpose
Provides full administrative access for authorized users to manage the AWS environment. Used for setup, configuration changes, and maintenance tasks.

### Trust Policy
**File:** `admin-role-trust-policy.json`

Defines **WHO** can assume the AdminRole.

**Explanation:**
- **Principal:** Allows IAM users in the same AWS account to assume this role
- **Condition:** Requires MFA to be enabled and active when assuming the role (security best practice)
- **Use Case:** Admin IAM user can `AssumeRole` to get temporary administrative credentials

### Permission Policy
**AWS Managed Policy:** `AdministratorAccess`

This is an AWS-managed policy that grants full access to all AWS services and resources. No custom permission policy file is needed.

**What it allows:**
- Create, modify, and delete all AWS resources
- Modify IAM policies and roles
- Access all services without restriction

---

## Role 2: ReadOnlyRole

### Purpose
Provides read-only access for auditing, inspection, and compliance verification without the ability to make changes.

### Trust Policy
**File:** `readonly-role-trust-policy.json`

Defines **WHO** can assume the ReadOnlyRole.

**Explanation:**
- **Principal:** Allows IAM users in the same AWS account to assume this role
- **Condition:** Requires MFA for added security
- **Use Case:** Auditors or junior team members can view resources without risk of accidental changes

### Permission Policy
**AWS Managed Policy:** `ReadOnlyAccess`

This is an AWS-managed policy that grants read-only access to all AWS services.

**What it allows:**
- View (describe, list, get) all AWS resources
- Read CloudTrail logs
- View security group configurations
- Inspect EC2 instances, S3 buckets, etc.

**What it prevents:**
- Creating, modifying, or deleting any resources
- Changing configurations
- Launching or terminating instances

---

## Role 3: EC2Role

### Purpose
Provides EC2 instances with the minimum permissions needed to function without requiring long-lived access keys stored on the instance.

### Trust Policy
**File:** `ec2-role-trust-policy.json`

Defines **WHO** can assume the EC2Role.

**Explanation:**
- **Principal:** Allows the EC2 service (not users) to assume this role
- **Service:** `ec2.amazonaws.com` means only EC2 instances can use this role
- **No MFA Required:** EC2 instances authenticate via instance metadata, not user credentials
- **How it works:** When an EC2 instance launches with this role attached (via instance profile), it automatically gets temporary credentials

### Permission Policy
**File:** `ec2-role-policy.json`

Defines **WHAT** EC2 instances with this role can do.

## How Roles Are Used

### AdminRole Usage Flow

1. Admin IAM user logs into AWS Console with MFA
2. User switches to AdminRole (assumes the role)
3. AWS provides temporary credentials with admin permissions
4. User performs administrative tasks
5. Session expires after a set time (default: 1 hour)

**Benefits:**
- Audit trail shows who assumed the role and when
- MFA requirement prevents unauthorized access
- Temporary credentials automatically expire

### ReadOnlyRole Usage Flow

1. Auditor or team member logs in with MFA
2. User assumes ReadOnlyRole
3. User can view all resources but cannot make changes
4. Attempts to modify resources are denied
5. Session expires automatically

**Benefits:**
- Safe environment exploration without risk
- Useful for training and knowledge transfer
- Clear separation of responsibilities

### EC2Role Usage Flow

1. EC2 instance is launched with EC2Role attached (via instance profile)
2. Instance automatically receives temporary credentials
3. AWS CLI and SDKs use these credentials without configuration
4. Credentials rotate automatically every 6 hours
5. No access keys stored on the instance

**Benefits:**
- No long-lived credentials to compromise
- Credentials never appear in code or configuration files
- Automatic credential rotation
- Easy to audit what instances are doing via CloudTrail

---

## Security Best Practices Implemented

✅ **Least Privilege:** Each role has only the permissions it needs  
✅ **MFA Required:** Human users must use MFA to assume roles  
✅ **No Access Keys:** EC2 instances use roles instead of access keys  
✅ **Temporary Credentials:** All role assumptions provide temporary credentials  
✅ **Audit Trail:** All role assumptions logged in CloudTrail  
✅ **Separation of Duties:** Different roles for different purposes

---

## Common IAM Troubleshooting

**Issue:** "User is not authorized to perform: sts:AssumeRole"  
**Solution:** Verify the trust policy allows your user/service and MFA is enabled

**Issue:** "Access Denied" when EC2 instance tries to use AWS CLI  
**Solution:** Ensure EC2Role is attached to the instance via instance profile

**Issue:** Session Manager not working  
**Solution:** Verify EC2Role has SSM permissions and VPC endpoints are configured

---

## Future Enhancements

Consider adding:
- **DeveloperRole** - Limited permissions for application development
- **S3AccessRole** - Specific S3 bucket access for applications
- **LambdaExecutionRole** - If Lambda functions are added
- **Permission boundaries** - Additional guardrails for role creation

---

## References

- [IAM Roles Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)
- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [Using Instance Profiles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html)
- [Policy Evaluation Logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html)
