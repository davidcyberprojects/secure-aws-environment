# Architecture & Design

Technical design decisions and architecture details for this secure AWS environment.

---

## Architecture Overview

```
Internet
    ↓
AWS Console/API
    ↓
VPC Endpoints (Private Network)
    ↓
EC2 (Private Subnet) - No Public IP
    ↓
CloudTrail → S3 + CloudWatch → Alarms → Email
```

**Key Principle:** Private-first architecture with zero internet exposure.

---

## Network Design

### VPC Layout
```
secure-vpc (10.0.0.0/16)
├── Public Subnet (10.0.1.0/24) - Reserved, unused
└── Private Subnet (10.0.2.0/24) - EC2 instances
```

### Why Private Subnet Only?
✅ **Zero internet exposure** - No public IPs, no internet gateway route  
✅ **Forces Session Manager** - More secure than SSH  
✅ **Demonstrates best practice** - Production apps should be in private subnets

### Security Groups
```
ec2-private-sg:
  Inbound: NONE
  Outbound: All traffic

ssmendpoint-sg:
  Inbound: HTTPS (443) from ec2-private-sg
  Outbound: All traffic
```

**Critical:** VPC endpoint SG must allow inbound from EC2 SG, or Session Manager won't work.

### VPC Endpoints vs NAT Gateway

**Why VPC Endpoints?**
- Cost: $21.60/month vs $32+/month for NAT
- Security: Traffic never leaves AWS network
- Sufficient: Only need access to AWS services, not public internet

**Trade-off:** Can't reach public internet (intentional for security)

### How Session Manager Works
```
1. User connects via AWS Console
2. Request goes to SSM service (AWS managed)
3. SSM connects to VPC endpoint in private subnet
4. Endpoint routes to EC2 instance
5. Connection established - no SSH, no public IP needed
```

**Requirements:**
- 3 VPC endpoints (ssm, ssmmessages, ec2messages)
- Private DNS enabled
- Security group allowing HTTPS from EC2

---

## IAM Design

### Role-Based Access (No Long-Lived Keys)

**AdminRole:**
- Who: IAM users with MFA
- What: Full admin access
- Use: Environment setup, maintenance

**ReadOnlyRole:**
- Who: IAM users with MFA  
- What: View-only access
- Use: Auditing, inspection

**EC2Role:**
- Who: EC2 service
- What: SSM + CloudWatch Logs only
- Use: Instance operations

### Why Roles Over Access Keys?
✅ Temporary credentials (expire after 1-6 hours)  
✅ Automatic rotation  
✅ No credentials stored on disk  
✅ Complete audit trail in CloudTrail

### Least Privilege Example
EC2Role has **only** SSM permissions. If compromised, attacker cannot:
- Launch new instances
- Access S3 buckets
- Modify IAM policies
- Change security groups

---

## Logging & Monitoring

### Three-Layer Visibility

**Layer 1: CloudTrail**
- Records all API activity
- Stores in S3 (90-day retention)
- Answers: Who did what, when, and from where?

**Layer 2: CloudWatch Logs**
- Real-time log processing
- Enables metric filters
- 5-minute delivery (faster than S3)

**Layer 3: CloudWatch Alarms**
- Monitors security events
- SNS email notifications
- 5 alarms for critical events:
  - Root account usage
  - Unauthorized API calls
  - IAM policy changes
  - Security group changes
  - Failed console logins

### Incident Response Flow
```
Event Occurs → CloudTrail Logs → Metric Filter → Alarm Triggers → Email Sent
Timeline: 5-10 minutes from event to notification
```

---

## Security Layers (Defense in Depth)

**Layer 1: Network Isolation**
- Private subnet, no public IPs
- No internet gateway route

**Layer 2: Security Groups**
- Default deny all inbound
- Explicit allow only for required traffic

**Layer 3: IAM**
- Least privilege roles
- MFA required for humans
- No access keys

**Layer 4: Encryption**
- EBS volumes encrypted
- S3 buckets encrypted (SSE-S3)
- CloudTrail logs encrypted

**Layer 5: Monitoring**
- CloudTrail logging everything
- Real-time alarms on security events
- Log file validation (tamper detection)

**Layer 6: Metadata Security**
- IMDSv2 enforced (prevents SSRF attacks)

---

## Key Design Decisions

### 1. Session Manager Over SSH Bastion
**Chosen:** Session Manager  
**Avoided:** SSH bastion in public subnet

**Why:**
- No SSH keys to manage/rotate
- No public-facing host to secure
- All sessions logged in CloudTrail
- IAM-based access control

**Trade-off:** Requires VPC endpoints ($21.60/month) but eliminates bastion host maintenance

---

### 2. IMDSv2 Enforcement
**Chosen:** Require session tokens for metadata  
**Avoided:** Allow IMDSv1 (legacy)

**Why:**
- Prevents SSRF attacks from accessing instance credentials
- Industry best practice since 2019
- Minimal compatibility impact

---

### 3. S3 Block Public Access (Account-Level)
**Chosen:** Account-wide public access block  
**Avoided:** Bucket-level only

**Why:**
- Defense in depth - even if bucket policy misconfigured, account block prevents exposure
- Prevents 95% of S3 data leaks
- Easy to audit (single setting)

---

### 4. CloudWatch Alarms Over GuardDuty
**Chosen:** Custom CloudWatch alarms  
**Avoided:** AWS GuardDuty

**Why:**
- Cost: Free tier vs $4.60/month minimum
- Educational value: Learn CloudWatch
- Sufficient for demo environment

**Production:** Would enable GuardDuty for ML-based threat detection

---

### 5. Single AZ Deployment
**Chosen:** All resources in one availability zone  
**Avoided:** Multi-AZ for redundancy

**Why:**
- Cost optimization ($22/month vs $45+/month)
- Learning environment, not production
- Demonstrates core security without HA complexity

**Production:** Would use 2-3 AZs for high availability

---

## What's NOT Included (And Why)

| Service | Why Excluded | When to Use |
|---------|--------------|-------------|
| **NAT Gateway** | Cost ($32/mo), VPC endpoints sufficient | When instances need internet access |
| **VPC Flow Logs** | Cost (~$5-10/mo) | Production forensics, compliance |
| **GuardDuty** | Cost ($4.60+/mo) | Production threat detection |
| **RDS** | Out of scope | Database workloads |
| **Load Balancers** | No web apps | Multi-instance applications |
| **Auto Scaling** | Single instance demo | Variable workload |

---

## Production vs Learning Environment

| Aspect | This Project | Production |
|--------|-------------|------------|
| **Availability** | Single AZ | Multi-AZ, multi-region |
| **Monitoring** | CloudWatch alarms | + GuardDuty, Security Hub |
| **Networking** | Single VPC | Transit Gateway, VPC peering |
| **Access** | IAM users | AWS SSO, federated identity |
| **Automation** | Manual | Terraform/CloudFormation |
| **Cost** | ~$22/mo | Scale-dependent |
| **Backups** | None | AWS Backup, snapshots |

---

## Technical Specifications

### Network
- VPC CIDR: 10.0.0.0/16
- DNS Resolution: Enabled
- DNS Hostnames: Enabled
- Route Tables: Public (IGW), Private (local only)

### Compute
- Instance Type: t2.micro
- AMI: Amazon Linux 2023
- Storage: 8 GB gp3 (encrypted)
- Metadata: IMDSv2 required

### Storage
- S3 Bucket: Encrypted, versioned, public access blocked
- Lifecycle: Delete after 90 days
- Purpose: CloudTrail logs

### Monitoring
- CloudTrail: Multi-region, log validation enabled
- CloudWatch: 5 alarms, 5 custom metrics
- SNS: Email notifications
- Retention: 90 days (S3), indefinite (CloudWatch Logs)

---

## Cost Breakdown

**Monthly Recurring:**
- VPC Endpoints: $21.60 (3 × $0.01/hour × 730 hours)
- EC2: $0 (free tier) or $8.50
- CloudTrail + S3 + CloudWatch: ~$1.00
- **Total: $22-30/month**

**One-Time:** $0

**After Free Tier (12 months):** ~$30/month

---

## Validation & Testing

**Network Tests:**
- Verify no public IP on EC2
- Confirm internet access blocked
- Test Session Manager connectivity
- Validate AWS API access via endpoints

**Security Tests:**
- Root account has MFA, no keys
- CloudTrail logging active
- Alarms triggering correctly
- IMDSv2 enforced

**See SETUP.md for detailed testing checklist**

---

## References

- [VPC Endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html)
- [Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/)
- [AWS Well-Architected](https://aws.amazon.com/architecture/well-architected/)
