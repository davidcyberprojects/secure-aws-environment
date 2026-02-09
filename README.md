# Secure AWS Environment 🔐

This project demonstrates how to design and deploy a secure AWS environment in line with security best practices. 

The focus is on access control, network isolation, logging, and cost management.

## Project Scope 

This project focuses on securing a basic AWS environment and does not include application development, CI/CD Pipelines, or production workloads. 

## Threat Model 

By default, AWS accounts are vulnerable to misconfiguration rather than platform-level flaws.
This project focuses on reducing the most common risks in small AWS environments 

### Key Risks Addressed 

- **Excessive IAM Permissions** - Overly broad permissions can lead to accidental or malicious changes.
- **Public Network Exposure** - Resources exposed directly to the internet can increase the attack surface.
- **publicly Accessible Storage** - Misconfigured S3 buckets can lead to data exposure.
- **Lack of Visibility** - Without logging and monitoring, security incidents can go undetected.
- **Uncontrolled Costs** - Forgotten resources can generate unexpected charges. 

## Architecture Overview 

The environment is designed with a security-first approach that prioritizes isolation and minimal exposure.

Resources are deployed within a dedicated Virtual Private Cloud (VPC) using both public and private subnets. 
Compute resources are placed in private subnets without public IP addresses to reduce exposure to the internet. 

A small public subnet is reserved only for controlled access and required AWS-managed services. 

## Security Principles & Controls 

The environment is built around a small set of core security principles:

- **Least Privileged Access** - IAM permissions are scoped to only what is required for each role.
- **Reduced Attack Surface** - Compute resources are placed in private networks without public exposure.
- **Secure By Default Configurations** - Encryption and public access blocking are enabled where available.
- **Visibility and Auditability** - All account activity is logged to support detection and investigation.
- **Cost Awareness** - Resource usage is monitored to prevent unexpected charges

These principles guide all design and implementation decisions throughout the project. 

## IAM Design 

Access to the AWS account is tightly controlled using IAM Roles and Multi-Factor authentication (MFA).
The root account is secured with MFA and is not used for day-to-day operations. 

### IAM Roles 

The following roles are defined to enforce the separation of responsibilities:

- **AdminRole** - Full administrative access used only for environment setup and maintenance.
- **ReadOnlyRole** - View-only access intended for auditing and inspection.
- **EC2Role** - Limited permissions required by EC2 Instances to interact with AWS Services without using access keys.

Direct IAM user permissions are avoided in favor of role-based access where possible. 

## Network Design 

The environment uses a dedicated Virtual Private Cloud (VPC) to isolate resources from other networks. 

### Subnet Strategy 

- **Public Subnet**
  - Used only for controlled access and required AWS-managed services.
  - No application workloads are deployed here.

- **Private Subnet**
  - Hosts compute resources such as EC2 Instances.
  - Instances do not have public IP addresses and are not directly accessible from the internet.
 
### Network Exposure Controls 

- Security groups restrict inbound traffic to only what is explicitly required.
- SSH access is limited to a known IP address when enabled.
- Unnecessary inbound ports are blocked by default.

This design reduces the attack surface while still allowing administrative access when needed. 

## Compute and Secure Access Design

Compute resources are deployed using Amazon EC2 instances placed in private subnets. 
Instances are configured without public IP addresses to prevent direct internet access.

### Instance Security Controls

- EC2 instances use IAM roles to access AWS services instead of long-lived access keys.
- Root storage volumes are encrypted to protect data at rest.
- Instance metadata service version 2 (IMDSv2) is enforced to reduce credential exposure.

### Administrative Access

Administrative access to EC2 instances is performed using AWS Systems Manager Session Manager.
This approach avoids opening inbound SSH ports and provides audit logging of access sessions.

## Secure Storage Design

Amazon S3 is used for object storage with a strong emphasis on preventing accidental public exposure.

### Storage Security Controls

- All S3 buckets block public access at the account and bucket level.
- Server-side encryption is enabled to protect data at rest.
- Bucket versioning is enabled to protect against accidental deletion or overwrite.
- Access to S3 objects is restricted using IAM policies and roles.

These controls ensure stored data remains private and auditable.

## Logging and Monitoring

Visibility is a core requirement for maintaining a secure AWS environment. 
The following services are used to record account activity and detect abnormal behavior.

### Logging Controls

- AWS CloudTrail is enabled to log all management API activity across the account.
- Logs are retained for a limited period to balance visibility and cost.

### Monitoring and Alerts

- CloudWatch is used to monitor key security-related events.
- Alerts are configured for sensitive actions such as root account usage and unauthorized API calls.

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
