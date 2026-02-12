# Architecture Decisions

Key design choices and their rationale for this secure AWS environment.

---

## Overview

This document explains the "why" behind major architectural decisions, including what was included, excluded, and the trade-offs considered.

---

## Network Architecture

### Decision 1: VPC Endpoints Instead of NAT Gateway

**What we chose:** VPC Interface Endpoints for AWS services

**What we avoided:** NAT Gateway for internet access

**Rationale:**

✅ **Pros of VPC Endpoints:**
- Lower cost: $21.60/month vs $32+/month for NAT
- Better security: No internet exposure at all
- Traffic stays on AWS private network
- Demonstrates modern AWS best practices
- Meets project goal: secure, isolated environment

❌ **Cons of VPC Endpoints:**
- Cannot reach public internet (intentional)
- Requires endpoint for each AWS service
- Slightly more complex initial setup

**Trade-off:** We prioritized security and cost over internet connectivity, which isn't needed for this environment.

**Alternative Considered:** NAT Gateway - rejected due to cost and unnecessary for project scope

---

### Decision 2: Private Subnet for All Compute

**What we chose:** EC2 instances in private subnet only

**What we avoided:** Public subnet with instances

**Rationale:**

✅ **Pros:**
- Zero direct internet exposure
- Forces use of Session Manager (more secure than SSH)
- Demonstrates defense in depth
- Aligns with enterprise security practices

❌ **Cons:**
- Cannot install software from internet
- Requires VPC endpoints for AWS access
- No direct inbound access

**Trade-off:** We chose maximum isolation over convenience.

**Use Case:** In production, public subnets would host load balancers; private subnets host application servers. This project focuses on the secure backend layer.

---

### Decision 3: Session Manager Over SSH Bastion

**What we chose:** AWS Systems Manager Session Manager

**What we avoided:** SSH bastion host in public subnet

**Rationale:**

✅ **Pros of Session Manager:**
- No SSH keys to manage or rotate
- No inbound ports opened
- All sessions logged in CloudTrail
- IAM-based access control
- No bastion host to maintain/patch

❌ **Cons:**
- Requires VPC endpoints ($)
- Learning curve for traditional SSH users

**Trade-off:** Modern managed service over traditional architecture.

**Cost Impact:** VPC endpoints ($21.60/month) vs bastion host (~$8/month) + operational overhead

---

## Security Architecture

### Decision 4: IMDSv2 Enforcement

**What we chose:** Require IMDSv2 for EC2 metadata access

**What we avoided:** Allowing IMDSv1 (legacy metadata service)

**Rationale:**

✅ **Security Benefits:**
- Prevents SSRF attacks from accessing credentials
- Requires session token (prevents simple curl attacks)
- Industry best practice as of 2019+

❌ **Compatibility:**
- Older applications may need updates
- Requires token-based access pattern

**Trade-off:** We chose security over backward compatibility.

**Real-world Impact:** Prevents common attack vector used in cloud breaches

---

### Decision 5: IAM Roles Over Access Keys

**What we chose:** IAM roles for all AWS access

**What we avoided:** IAM users with long-lived access keys

**Rationale:**

✅ **Security Benefits:**
- No credentials stored on disk
- Automatic credential rotation (every 6 hours)
- No risk of committing keys to GitHub
- Centralized permission management

❌ **Limitations:**
- Slightly more complex setup
- Requires understanding of assume role workflow

**Trade-off:** Best practice security over simplicity.

**Alternative Considered:** Access keys - rejected due to security risks

---

### Decision 6: Block Public Access at Account Level (S3)

**What we chose:** Account-wide S3 public access block

**What we avoided:** Bucket-level controls only

**Rationale:**

✅ **Defense in Depth:**
- Even if bucket policy is misconfigured, account-level block prevents exposure
- Protects against human error
- Easy to audit (single account setting)

**Trade-off:** Cannot create public S3 websites from this account (acceptable for this use case).

**Real-world Impact:** Prevents 95% of S3 data exposure incidents

---

## Monitoring Architecture

### Decision 7: CloudWatch Alarms for Security Events

**What we chose:** 5 targeted CloudWatch alarms

**What we avoided:** AWS GuardDuty (ML-based threat detection)

**Rationale:**

✅ **Pros of CloudWatch Alarms:**
- Free tier eligible (10 alarms free)
- Customizable to specific threats
- Immediate notifications via SNS
- Educational value (learn CloudWatch)

❌ **Limitations:**
- Manual pattern creation
- Reactive, not proactive
- No ML-based anomaly detection

**Trade-off:** Cost optimization and learning value over advanced features.

**Cost Impact:** $0 vs $4.60/month for GuardDuty

**Future Enhancement:** Enable GuardDuty for 30-day free trial

---

### Decision 8: CloudTrail in CloudWatch Logs

**What we chose:** Send CloudTrail to both S3 and CloudWatch Logs

**What we avoided:** S3 only (default)

**Rationale:**

✅ **Benefits:**
- Enables real-time metric filters
- Allows CloudWatch alarms on API events
- Faster query capabilities
- Educational demonstration of log integration

❌ **Cost:**
- CloudWatch Logs storage: ~$0.50/GB/month
- Worth it for small log volumes

**Trade-off:** Pay small fee for real-time monitoring capability.

---

## Cost Architecture

### Decision 9: Free Tier Optimization

**What we chose:** Design maximizing AWS Free Tier usage

**What we avoided:** Enterprise-scale services (RDS, EKS, etc.)

**Rationale:**

✅ **Alignment with Project Goals:**
- Demonstration/learning environment
- Not production workload
- Cost-conscious design
- Shows real-world constraint handling

**Free Tier Utilization:**
- EC2: 750 hours/month (within limit)
- CloudTrail: First trail free
- CloudWatch: 10 alarms free
- S3: 5 GB storage free

**Trade-off:** Feature richness vs affordability.

---

### Decision 10: No VPC Flow Logs (Initially)

**What we chose:** Skip VPC Flow Logs in initial setup

**What we avoided:** Enable flow logs to all subnets

**Rationale:**

❌ **Cost Consideration:**
- Flow logs cost ~$0.50/GB ingested
- High volume for 24/7 network capture
- Not critical for this demonstration

✅ **Can Enable Later:**
- Easy to add when needed
- Useful for troubleshooting
- Recommended in production

**Trade-off:** Cost savings over network visibility.

**Production Difference:** Enterprise environments would enable flow logs for security forensics.

---

## Scope Decisions

### Decision 11: Single-Account Architecture

**What we chose:** Everything in one AWS account

**What we avoided:** Multi-account setup with AWS Organizations

**Rationale:**

✅ **Simplicity:**
- Easier to demonstrate and understand
- No cross-account IAM complexity
- Lower operational overhead

**Production Difference:**
- Large organizations use AWS Organizations
- Separate accounts for dev/test/prod
- Service Control Policies (SCPs) for guardrails

**Trade-off:** Learning focus over enterprise complexity.

---

### Decision 12: No Application Layer

**What we chose:** Infrastructure-focused security demonstration

**What we avoided:** Deploying applications, databases, load balancers

**Rationale:**

✅ **Project Scope:**
- Focus on foundational security
- Network isolation and access control
- Monitoring and logging

**Not Included:**
- Application security (WAF, rate limiting)
- Database security (RDS, encryption at rest)
- Container security (ECS, EKS)
- Content delivery (CloudFront, S3 static hosting)

**Trade-off:** Deep dive on fundamentals vs broad coverage.

---

## Technology Choices

### Decision 13: Amazon Linux 2023 for EC2

**What we chose:** Amazon Linux 2023 (latest)

**What we avoided:** Ubuntu, CentOS, Windows

**Rationale:**

✅ **Benefits:**
- Optimized for AWS services
- SSM agent pre-installed
- Long-term support (until 2028)
- Regular security updates
- Free tier eligible

**Alternative Considered:** Ubuntu 22.04 - also good choice, but Amazon Linux is AWS-native.

---

### Decision 14: Session Manager Over SSH + Bastion

Already covered above, but worth emphasizing as a key modern vs traditional decision.

**Modern Approach:** Session Manager (this project)  
**Traditional Approach:** SSH bastion host

**Why Modern Wins:**
- No key management
- Better audit trail
- No exposed SSH port
- IAM-based access

---

## What We Deliberately Excluded

### Excluded Services & Why

| Service | Why Excluded | When to Use |
|---------|--------------|-------------|
| **NAT Gateway** | Cost ($32/mo), unnecessary | When instances need internet access |
| **RDS** | Not needed for demo | When running databases in production |
| **Auto Scaling** | Single instance sufficient | Production workloads with variable traffic |
| **Load Balancers** | No public apps | Multi-instance web applications |
| **Lambda** | Out of scope | Serverless computing needs |
| **GuardDuty** | Cost optimization | Production threat detection |
| **AWS WAF** | No web apps | Protecting web applications |
| **EKS/ECS** | Complexity | Container orchestration needs |

---

## Design Principles Applied

### 1. Least Privilege
- IAM roles with minimum permissions
- No overly broad "allow all" policies
- Security groups with minimal inbound rules

### 2. Defense in Depth
- Multiple security layers (network, IAM, encryption)
- Account-level S3 blocks + bucket-level
- IMDSv2 + IAM roles

### 3. Visibility
- CloudTrail logging all API calls
- CloudWatch alarms for security events
- SNS notifications for awareness

### 4. Cost Awareness
- Free tier optimization
- Budget alerts
- No expensive services (NAT, GuardDuty in steady state)

### 5. Security by Default
- Encryption enabled automatically
- Public access blocked by default
- Private networking first

---

## Lessons Learned

### What Worked Well

✅ VPC endpoints eliminated need for NAT Gateway  
✅ Session Manager provided secure access without SSH complexity  
✅ CloudWatch alarms effectively caught security events  
✅ Free tier covered most services

### What Was Challenging

⚠️ VPC endpoint security groups were non-obvious (required troubleshooting)  
⚠️ IMDSv2 required different access patterns  
⚠️ CloudWatch metric filters have learning curve

### What We'd Change

🔄 **For Production:**
- Enable VPC Flow Logs
- Add GuardDuty
- Implement multi-account structure
- Add redundancy (multi-AZ)

🔄 **For Learning:**
- Add more detailed troubleshooting steps earlier
- Include common error patterns
- Provide more example CloudWatch patterns

---

## Production vs Learning Trade-offs

| Aspect | This Project | Production Environment |
|--------|-------------|------------------------|
| **Availability** | Single AZ | Multi-AZ, multi-region |
| **Monitoring** | CloudWatch alarms | GuardDuty, Security Hub, SIEM |
| **Access** | Admin user with MFA | AWS SSO, federated identity |
| **Networking** | Single VPC | Transit Gateway, VPC peering |
| **Cost** | Minimal ($22/mo) | Based on scale |
| **Automation** | Manual | Terraform, CloudFormation |
| **Backups** | None | Automated snapshots, AWS Backup |
| **DR** | None | Cross-region replication |

---

## References

- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [AWS Security Best Practices](https://docs.aws.amazon.com/security/)
- [VPC Endpoints vs NAT Gateway](https://aws.amazon.com/blogs/architecture/reduce-cost-and-increase-security-with-amazon-vpc-endpoints/)
- [IMDSv2 Deep Dive](https://aws.amazon.com/blogs/security/defense-in-depth-open-firewalls-reverse-proxies-ssrf-vulnerabilities-ec2-instance-metadata-service/)
