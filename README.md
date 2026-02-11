# Secure AWS Environment 🔐

This project demonstrates how to design and deploy a secure AWS environment in line with security best practices.

The focus is on access control, network isolation, logging, and cost management.

## Project Scope

This project focuses on securing a basic AWS environment and does not include application development, CI/CD pipelines, or production workloads.

## About This Project

This project was built after passing the AWS Certified Cloud Practitioner exam 
as a way to apply foundational concepts and explore intermediate AWS security 
practices. While the CCP exam covers these services conceptually, this project 
implements them hands-on at a Solutions Architect Associate level.

**CCP Concepts Applied:**
- VPC and subnet design
- IAM roles and permissions
- S3 security controls
- CloudTrail logging
- Cost management

**Beyond CCP (Self-Taught):**
- VPC endpoints configuration
- Session Manager implementation
- IMDSv2 enforcement
- Security group troubleshooting

## Threat Model

By default, AWS accounts are vulnerable to misconfiguration rather than platform-level flaws. This project focuses on reducing the most common risks in small AWS environments.

### Key Risks Addressed

- **Excessive IAM Permissions** - Overly broad permissions can lead to accidental or malicious changes.
- **Public Network Exposure** - Resources exposed directly to the internet can increase the attack surface.
- **Publicly Accessible Storage** - Misconfigured S3 buckets can lead to data exposure.
- **Lack of Visibility** - Without logging and monitoring, security incidents can go undetected.
- **Uncontrolled Costs** - Forgotten resources can generate unexpected charges.

## Architecture Overview

The environment is designed with a security-first approach that prioritizes isolation and minimal exposure.

Resources are deployed within a dedicated Virtual Private Cloud (VPC) using both public and private subnets. Compute resources are placed in private subnets without public IP addresses to reduce exposure to the internet.

A small public subnet is reserved only for controlled access and required AWS-managed services.

## Security Principles & Controls

The environment is built around a small set of core security principles:

- **Least Privileged Access** - IAM permissions are scoped to only what is required for each role.
- **Reduced Attack Surface** - Compute resources are placed in private networks without public exposure.
- **Secure By Default Configurations** - Encryption and public access blocking are enabled where available.
- **Visibility and Auditability** - All account activity is logged to support detection and investigation.
- **Cost Awareness** - Resource usage is monitored to prevent unexpected charges.

These principles guide all design and implementation decisions throughout the project.

## IAM Design

Access to the AWS account is tightly controlled using IAM roles and Multi-Factor Authentication (MFA). The root account is secured with MFA and is not used for day-to-day operations.

### IAM Roles

The following roles are defined to enforce the separation of responsibilities:

- **AdminRole** - Full administrative access used only for environment setup and maintenance.
- **ReadOnlyRole** - View-only access intended for auditing and inspection.
- **EC2Role** - Limited permissions required by EC2 instances to interact with AWS services without using access keys.

Direct IAM user permissions are avoided in favor of role-based access where possible.

### EC2Role Permissions

The EC2Role is configured with the minimum permissions necessary for instances to function:

- **AmazonSSMManagedInstanceCore** - Enables AWS Systems Manager Session Manager access
- **CloudWatch Logs Write Access** - Allows instances to send application logs to CloudWatch
- **Limited S3 Access** - Read/write permissions only to specific application buckets (if needed)

This role is attached to EC2 instances via an instance profile, eliminating the need for long-lived access keys.

## Network Design

The environment uses a dedicated Virtual Private Cloud (VPC) to isolate resources from other networks.

### Subnet Strategy

**Public Subnet**
- Used only for controlled access and required AWS-managed services.
- No application workloads are deployed here.

**Private Subnet**
- Hosts compute resources such as EC2 instances.
- Instances do not have public IP addresses and are not directly accessible from the internet.

### Network Exposure Controls

- Security groups restrict inbound traffic to only what is explicitly required.
- SSH access is limited to a known IP address when enabled (though Session Manager is preferred).
- Unnecessary inbound ports are blocked by default.

This design reduces the attack surface while still allowing administrative access when needed.

### Security Group Design

Security groups follow a default-deny approach:

- **Default Policy**: All inbound traffic is denied unless explicitly allowed
- **Reference-Based Rules**: Security group rules reference other security groups rather than CIDR blocks where possible (more dynamic and secure)
- **Stateful Filtering**: Return traffic is automatically allowed for established connections
- **No Outbound Restrictions**: Private instances have unrestricted outbound access to communicate with AWS services

**Example Security Groups:**
- **EC2 Private SG**: No inbound rules, all outbound allowed
- **SSM Endpoint SG**: Inbound HTTPS (443) from EC2 Private SG, all outbound allowed

### VPC Endpoints for Private Connectivity

To enable secure access to AWS services without requiring internet connectivity, the following VPC interface endpoints are deployed:

- **com.amazonaws.[region].ssm** - AWS Systems Manager service
- **com.amazonaws.[region].ssmmessages** - Session Manager message routing
- **com.amazonaws.[region].ec2messages** - EC2 instance communication

**Key Configuration Details:**
- Endpoints are placed in the private subnet alongside EC2 instances
- Private DNS is enabled to ensure service hostnames resolve to endpoint IPs
- Endpoint security groups allow inbound HTTPS (443) from EC2 instance security groups
- This architecture eliminates the need for a NAT Gateway, reducing both cost and attack surface

**VPC DNS Settings:**
- DNS resolution: Enabled
- DNS hostnames: Enabled

These settings are required for VPC endpoints to function properly.

Detailed network design is documented in `docs/network.md`.

## Compute and Secure Access Design

Compute resources are deployed using Amazon EC2 instances placed in private subnets. Instances are configured without public IP addresses to prevent direct internet access.

### Instance Security Controls

- EC2 instances use IAM roles to access AWS services instead of long-lived access keys.
- Root storage volumes are encrypted to protect data at rest.
- Instance Metadata Service version 2 (IMDSv2) is enforced to reduce credential exposure.

### IMDSv2 Enforcement

IMDSv2 requires session tokens for metadata access, preventing SSRF attacks from accessing instance credentials:

```bash
# Require IMDSv2 on existing instances
aws ec2 modify-instance-metadata-options \
  --instance-id i-xxxxx \
  --http-tokens required \
  --http-put-response-hop-limit 1
```

### Administrative Access

Administrative access to EC2 instances is performed using AWS Systems Manager Session Manager. This approach avoids opening inbound SSH ports and provides audit logging of access sessions.

**Benefits of Session Manager:**
- No SSH keys to manage or rotate
- No bastion hosts required
- All sessions are logged in CloudTrail
- Access controlled via IAM policies
- Works entirely over HTTPS through VPC endpoints

## Secure Storage Design

Amazon S3 is used for object storage with a strong emphasis on preventing accidental public exposure.

### Storage Security Controls

- All S3 buckets block public access at the account and bucket level.
- Server-side encryption is enabled to protect data at rest.
- Bucket versioning is enabled to protect against accidental deletion or overwrite.
- Access to S3 objects is restricted using IAM policies and roles.

These controls ensure stored data remains private and auditable.

### S3 Block Public Access

Public access blocking is enforced at the AWS account level to prevent any S3 bucket from being accidentally exposed:

```bash
aws s3control put-public-access-block \
  --account-id <account-id> \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

This provides a safety net even if individual bucket policies are misconfigured.

## Encryption Strategy

Data protection is enforced through encryption at rest and in transit.

### Encryption at Rest

- **EBS Volumes**: Encrypted using AWS-managed keys (aws/ebs)
- **S3 Buckets**: Server-side encryption with Amazon S3-managed keys (SSE-S3)
- **CloudTrail Logs**: Encrypted in S3 using SSE-S3

### Encryption in Transit

- All AWS API calls use TLS 1.2 or higher
- VPC endpoints provide private network paths without traversing the public internet
- Session Manager connections are encrypted end-to-end

## Logging and Monitoring

Visibility is a core requirement for maintaining a secure AWS environment. The following services are used to record account activity and detect abnormal behavior.

### Logging Controls

- **AWS CloudTrail** is enabled to log all management API activity across the account.
- Logs are retained for a defined period to balance visibility and cost.
- CloudTrail logs are stored in a dedicated S3 bucket with restricted access.

### VPC Flow Logs

VPC Flow Logs capture network traffic metadata for security analysis:

- Records accepted and rejected traffic at the network interface level
- Helps detect unusual connection patterns or unauthorized access attempts
- Stored in CloudWatch Logs or S3 for cost-effective retention

### Monitoring and Alerts

CloudWatch is used to monitor key security-related events:

- **Root Account Usage**: Alert triggered when root credentials are used
- **Unauthorized API Calls**: Notifications for denied API requests
- **IAM Policy Changes**: Alerts for modifications to security policies
- **Console Sign-In Failures**: Detection of potential brute force attempts

These controls provide auditability and support basic incident detection.

## Cost Controls and Constraints

Cost management is treated as a security and operational concern.

### Cost Control Measures

- An AWS budget is configured with a low monthly limit to prevent unexpected charges.
- Email alerts are enabled to notify when spending approaches the defined threshold.
- Services with high baseline costs, such as NAT Gateways, are intentionally avoided for this project.

### Project Constraints

- The environment is designed for learning and demonstration purposes, not production workloads.
- Resource usage is kept minimal, and components are shut down or removed when not in use.

These constraints ensure the environment remains safe, predictable, and affordable.

## What's NOT Included

This project intentionally excludes certain AWS services to maintain focus and control costs:

- **NAT Gateway** - Cost optimization; using VPC endpoints instead for AWS service access
- **Application Load Balancers** - No public-facing applications in this environment
- **RDS Databases** - Compute-focused demonstration; databases not required
- **AWS WAF and Shield** - Application security services not needed without web applications
- **AWS Organizations and SCPs** - Single account environment; organizational controls not applicable
- **GuardDuty (ongoing)** - Threat detection service (30-day trial recommended for learning)

## AWS Account Setup

- Root account is secured with Multi-Factor Authentication (MFA), and all access keys are removed.
- Day-to-day operations are performed using an existing Admin IAM user with AdministratorAccess policy.
- MFA is also enabled on the Admin IAM user to provide an extra layer of protection.
- All administrative actions are logged for auditability.

**Note**: This project uses the existing IAM user for demonstration purposes, reflecting real-world account management practices.

## Security Testing & Validation

The following tests were performed to verify the security posture:

- ✅ **EC2 Isolation**: Verified instances in private subnets have no public IP addresses
- ✅ **S3 Public Access**: Confirmed all buckets block public access at account and bucket level
- ✅ **Session Manager**: Tested connectivity to private instances without SSH or bastion hosts
- ✅ **IAM Least Privilege**: Validated roles have only necessary permissions
- ✅ **CloudTrail Logging**: Confirmed all API activity is being recorded
- ✅ **Budget Alerts**: Tested that email notifications trigger at defined thresholds
- ✅ **VPC Endpoint Connectivity**: Verified SSM endpoints allow access from private instances
- ✅ **IMDSv2 Enforcement**: Confirmed instances require session tokens for metadata access

## Key Takeaways

Lessons learned from building this secure AWS environment:

- **VPC Endpoints**: Security groups on VPC endpoints must allow inbound traffic from instance security groups (common misconfiguration)
- **Session Manager**: Eliminates SSH key management and bastion host complexity while improving auditability
- **AWS Defaults**: AWS provides secure defaults, but they require intentional configuration and understanding
- **Cost vs Security**: Security doesn't have to be expensive; VPC endpoints can replace costly NAT Gateways
- **Defense in Depth**: Multiple layers (network isolation + IAM + encryption + logging) provide resilience against misconfigurations

## Future Enhancements

Potential improvements for production environments:

- Enable AWS Config to track resource configuration compliance
- Implement AWS GuardDuty for threat detection
- Add CloudWatch dashboards for security metrics visualization
- Configure SNS topics for centralized alerting
- Implement automated remediation using Lambda and EventBridge
- Add AWS Backup for automated EC2 and EBS snapshots

## References

- [AWS Well-Architected Framework - Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html)
- [AWS Security Best Practices](https://docs.aws.amazon.com/security/)
- [VPC Endpoints Documentation](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html)
- [Session Manager Documentation](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)

---

**Project Status**: Active Learning Environment  
**Last Updated**: February 2026  
**Maintained By**: David Devlin
