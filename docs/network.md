# Network Design

Network architecture and configuration for this secure AWS environment.

---

## Overview

This environment uses a **private subnet-first architecture** with VPC endpoints, eliminating the need for public internet access while maintaining secure connectivity to AWS services.

**Key Principle:** Minimize attack surface by keeping all compute resources off the public internet.

---

## VPC Configuration

**VPC Name:** `secure-vpc`  
**CIDR Block:** `10.0.0.0/16`  
**Region:** [Your AWS Region]  
**Availability Zones:** Single AZ (cost optimization)

### VPC Settings

| Setting | Value | Purpose |
|---------|-------|---------|
| **DNS Resolution** | Enabled ✅ | Required for VPC endpoints |
| **DNS Hostnames** | Enabled ✅ | Allows private DNS for endpoints |
| **Default Security Group** | Exists | Not used (custom SGs created) |
| **Default NACL** | Allows all | Network ACLs not customized |

---

## Subnet Design

### Public Subnet

**CIDR:** `10.0.1.0/24`  
**Purpose:** Reserved for future use (e.g., load balancers, NAT gateway)  
**Route Table:** Routes to Internet Gateway

**Current Usage:** None - demonstrates proper network separation

**Configuration:**
- Auto-assign public IPv4: Disabled
- Connected to Internet Gateway via route table
- Available for future public-facing services

---

### Private Subnet

**CIDR:** `10.0.2.0/24`  
**Purpose:** Hosts all EC2 instances  
**Route Table:** Local VPC traffic only (no internet route)

**Current Usage:**
- EC2 instances
- VPC endpoints

**Configuration:**
- Auto-assign public IPv4: Disabled ❌
- No route to Internet Gateway
- Routes only to:
  - `10.0.0.0/16` → local (VPC internal traffic)

**Why Private:**
- Prevents direct internet exposure
- Forces use of VPC endpoints for AWS services
- Requires Session Manager for access (no SSH from internet)

---

## Route Tables

### Public Route Table

| Destination | Target | Purpose |
|-------------|--------|---------|
| `10.0.0.0/16` | local | VPC internal traffic |
| `0.0.0.0/0` | igw-xxxxx | Internet access |

**Associated Subnets:** Public subnet (`10.0.1.0/24`)

---

### Private Route Table

| Destination | Target | Purpose |
|-------------|--------|---------|
| `10.0.0.0/16` | local | VPC internal traffic only |

**Associated Subnets:** Private subnet (`10.0.2.0/24`)

**No Internet Route:** This is intentional - instances access AWS services via VPC endpoints

---

## Internet Gateway

**Name:** `secure-vpc-igw`  
**Attached to:** `secure-vpc`  
**Purpose:** Enables public subnet connectivity (currently unused)

**Note:** Required for VPC infrastructure even if no resources currently use it. May be used for future public-facing resources like load balancers.

---

## Security Groups

Security groups act as virtual firewalls controlling inbound and outbound traffic.

### EC2 Private Security Group

**Name:** `ec2-private-sg`  
**Purpose:** Applied to EC2 instances in private subnet

**Inbound Rules:**
```
NONE - No inbound traffic allowed from anywhere
```

**Outbound Rules:**
```
All traffic → 0.0.0.0/0 (ALLOW)
```

**Why No Inbound:**
- EC2 instances don't accept connections from anywhere
- Access is outbound-only through Session Manager
- Connections initiated by instance, not to instance

**Why All Outbound:**
- Instances need to reach VPC endpoints on port 443
- Allows communication with AWS services
- Stateful - return traffic automatically allowed

---

### SSM Endpoint Security Group

**Name:** `ssmendpoint-sg`  
**Purpose:** Applied to VPC endpoints for Session Manager

**Inbound Rules:**
```
HTTPS (443) → Source: ec2-private-sg (ALLOW)
```

**Outbound Rules:**
```
All traffic → 0.0.0.0/0 (ALLOW)
```

**Why This Configuration:**
- Endpoints must accept HTTPS connections from EC2 instances
- Using security group reference (not CIDR) for dynamic updates
- Most common misconfiguration: forgetting this inbound rule ⚠️

**Critical:** Without the inbound rule, Session Manager won't work!

---

## VPC Endpoints

VPC endpoints enable private connectivity to AWS services without traversing the public internet.

### Why VPC Endpoints Instead of NAT Gateway?

| Aspect | VPC Endpoints | NAT Gateway |
|--------|--------------|-------------|
| **Cost** | $21.60/month | $32+/month |
| **Security** | Traffic stays on AWS network | Exposes to internet |
| **Use Case** | AWS services only | General internet access |
| **Maintenance** | Managed by AWS | Requires monitoring |

**Decision:** VPC endpoints chosen for cost and security.

---

### Endpoint 1: SSM Endpoint

**Service Name:** `com.amazonaws.[region].ssm`  
**Type:** Interface endpoint  
**Purpose:** Core Systems Manager API

**Configuration:**
- VPC: `secure-vpc`
- Subnets: Private subnet
- Security Group: `ssmendpoint-sg`
- Private DNS: Enabled ✅

**Enables:**
- SSM agent registration
- Managed instance operations
- Systems Manager API calls

---

### Endpoint 2: SSM Messages Endpoint

**Service Name:** `com.amazonaws.[region].ssmmessages`  
**Type:** Interface endpoint  
**Purpose:** Session Manager message routing

**Configuration:**
- VPC: `secure-vpc`
- Subnets: Private subnet
- Security Group: `ssmendpoint-sg`
- Private DNS: Enabled ✅

**Enables:**
- Session Manager shell sessions
- Interactive command execution
- Encrypted communication channel

---

### Endpoint 3: EC2 Messages Endpoint

**Service Name:** `com.amazonaws.[region].ec2messages`  
**Type:** Interface endpoint  
**Purpose:** EC2 instance communication with Systems Manager

**Configuration:**
- VPC: `secure-vpc`
- Subnets: Private subnet
- Security Group: `ssmendpoint-sg`
- Private DNS: Enabled ✅

**Enables:**
- Run Command operations
- State Manager functionality
- Inventory collection

---

### All Three Required

❌ **Missing any endpoint breaks Session Manager**

These three endpoints work together:
1. **ssm** - Management API
2. **ssmmessages** - Interactive sessions
3. **ec2messages** - Instance-to-service communication

---

## Network Flow Diagrams

### Session Manager Connection Flow

```
User (Internet)
    ↓
AWS Console/CLI
    ↓
Systems Manager Service (AWS managed)
    ↓
VPC Endpoint (ssmmessages) - Private network
    ↓
EC2 Instance (Private subnet) - 10.0.2.x
```

**Key Points:**
- No inbound internet connection to EC2
- Connection initiated by EC2 outbound to endpoint
- All traffic encrypted and on AWS backbone

---

### AWS API Call Flow (from EC2)

```
EC2 Instance (10.0.2.x)
    ↓
Security Group (ec2-private-sg) - Outbound allowed
    ↓
VPC Endpoint (ssm/ssmmessages/ec2messages)
    ↓
Security Group (ssmendpoint-sg) - Inbound HTTPS allowed from ec2-private-sg
    ↓
AWS Service (SSM/EC2/etc)
```

**Why This Works:**
- Outbound from EC2 always allowed
- Inbound to endpoint allowed from EC2 SG
- Stateful connection - return traffic automatic
- Never touches public internet

---

## Network Access Control

### What CAN Access EC2 Instances

✅ **Session Manager** - Via VPC endpoints  
✅ **AWS Services** - Through IAM role and endpoints  
✅ **Internal VPC** - Other resources in same VPC (if any)

### What CANNOT Access EC2 Instances

❌ **Public Internet** - No route, no public IP  
❌ **SSH from Internet** - No inbound rules, no public IP  
❌ **Other AWS Accounts** - No peering configured  
❌ **On-Premises** - No VPN/Direct Connect

---

## DNS Resolution

### How Private DNS Works

**VPC Endpoints with Private DNS Enabled:**

```
EC2 tries to reach: ssm.us-east-1.amazonaws.com
    ↓
VPC DNS resolution (enabled)
    ↓
Resolves to: 10.0.2.x (VPC endpoint private IP)
    ↓
Traffic stays in VPC, never goes to internet
```

**Without Private DNS:**
- Would resolve to public AWS IP
- Traffic blocked (no internet route)
- Connection fails

**Requirements:**
- VPC DNS Resolution: Enabled
- VPC DNS Hostnames: Enabled
- Endpoint Private DNS: Enabled

---

## Network Security Layers

### Layer 1: Network Isolation
- Private subnet with no internet gateway route
- No public IPs assigned to instances

### Layer 2: Security Groups
- Default deny all inbound
- Explicit allow for required traffic only
- Security group references for dynamic updates

### Layer 3: VPC Endpoints
- Private network paths to AWS services
- No exposure to public internet
- Endpoint policies (not implemented, but possible)

### Layer 4: NACLs (Default)
- Not customized in this environment
- Allow all traffic (default)
- Security groups provide sufficient control

**Defense in Depth:** Multiple layers ensure even if one fails, others protect resources.

---

## Network Troubleshooting

### Common Issues

**Session Manager Not Working:**
1. Check all 3 VPC endpoints exist and are "Available"
2. Verify `ssmendpoint-sg` allows HTTPS from `ec2-private-sg`
3. Confirm Private DNS enabled on endpoints
4. Verify VPC has DNS resolution/hostnames enabled

**Cannot Reach AWS Services:**
1. Check security group outbound rules
2. Verify VPC endpoints for the service exist
3. Confirm IAM role has necessary permissions

**Diagnostic Commands:**
```bash
# From EC2 instance, test endpoint connectivity
curl -I https://ssm.us-east-1.amazonaws.com --connect-timeout 5

# Check if DNS resolves to private IP
nslookup ssm.us-east-1.amazonaws.com

# Verify IAM role
aws sts get-caller-identity
```

---

## Network Cost Breakdown

| Component | Quantity | Unit Cost | Monthly Cost |
|-----------|----------|-----------|--------------|
| **VPC** | 1 | Free | $0 |
| **Subnets** | 2 | Free | $0 |
| **Internet Gateway** | 1 | Free | $0 |
| **Route Tables** | 2 | Free | $0 |
| **Security Groups** | 2 | Free | $0 |
| **VPC Endpoints** | 3 | $0.01/hour | $21.60 |
| **Data Transfer** | Internal | Free | $0 |

**Total Network Cost:** $21.60/month

**Savings vs NAT Gateway:** $10.40/month

---

## Production Considerations

### Current Setup (Learning/Demo)
- Single AZ deployment
- Minimal redundancy
- Cost-optimized

### Production Enhancements Would Include:
- **Multi-AZ deployment** - Subnets in 2+ AZs
- **NAT Gateway** - For internet access if needed (one per AZ)
- **VPC Flow Logs** - Network traffic analysis
- **Transit Gateway** - If connecting multiple VPCs
- **Custom NACLs** - Additional network layer security
- **VPC Peering** - Connect to other VPCs
- **AWS PrivateLink** - For SaaS integrations

---

## Network Expansion Options

### To Add Internet Access:
1. Deploy NAT Gateway in public subnet
2. Add route in private route table: `0.0.0.0/0` → NAT Gateway
3. Cost: +$32/month

### To Add More AWS Services:
1. Create additional VPC endpoints as needed
2. Examples: S3 (gateway endpoint - free!), CloudWatch, Lambda
3. Add security group rules if required

### To Add High Availability:
1. Create subnets in second AZ
2. Deploy endpoints in both AZs
3. Add load balancer in public subnets
4. Cost: Doubles endpoint costs

---

## Network Validation Checklist

- [ ] VPC has DNS resolution enabled
- [ ] VPC has DNS hostnames enabled
- [ ] Private subnet has no internet gateway route
- [ ] EC2 instances have no public IPs
- [ ] All 3 VPC endpoints are "Available"
- [ ] VPC endpoints have Private DNS enabled
- [ ] `ssmendpoint-sg` allows HTTPS from `ec2-private-sg`
- [ ] `ec2-private-sg` allows all outbound traffic
- [ ] Session Manager can connect to instances

---

## References

- [VPC Endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html)
- [Security Groups](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)
- [Session Manager Prerequisites](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-prerequisites.html)
- [VPC Best Practices](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-best-practices.html)
