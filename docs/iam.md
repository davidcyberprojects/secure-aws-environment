## IAM & Access Management

Access to the AWS account is secured using IAM roles instead of long-lived credentials. Roles are separated by responsibility to enforce least-privilege access and reduce blast radius.

Sensitive identifiers have been redacted in screenshots to follow security best practices.

### AdminRole
- Provides full administrative access for environment setup and maintenance.
- Intended for human users only.
- Trusted entity: AWS account.
- MFA is required for users assuming this role.
- Trust relationship screenshot: `screenshots/AdminRole-Trust.png`

### ReadOnlyRole
- Provides read-only access for auditing and inspection.
- Intended for human users.
- Trusted entity: AWS account.
- Trust relationship screenshot: `screenshots/ReadOnlyRole-Trust.png`

### EC2Role
- Assigned to EC2 instances via an instance profile.
- Eliminates the need for hard-coded AWS credentials on instances.
- Trusted entity: EC2 service.
- Attached permissions:
  - AWS managed policy: `AmazonS3ReadOnlyAccess`
  - Custom inline policy for minimal CloudWatch Logs access
- Trust relationship screenshot: `screenshots/EC2Role-Trust.png`
- Permissions screenshot: `screenshots/EC2Role-Permissions.png`
- Inline policy example: `iam/ec2-role-policy.json`

> Note: AWS managed policies are not duplicated in this repository. A custom inline policy is included to demonstrate least-privilege permissions design.
