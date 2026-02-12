# Cost Breakdown

Estimated monthly costs for this secure AWS environment.

---

## Monthly Cost Summary

| Service | Configuration | Monthly Cost | Notes |
|---------|--------------|--------------|-------|
| **EC2 Instance** | t2.micro (750 hrs free tier) | $0 - $8.50 | Free tier: 750 hours/month |
| **VPC Endpoints** | 3 interface endpoints | $21.60 | $0.01/hour × 3 × 730 hours |
| **EBS Volume** | 8 GB gp3 (30 GB free tier) | $0 | Within free tier |
| **CloudTrail** | 1 trail, management events | $0 | First trail free |
| **S3 Storage** | CloudTrail logs (~1 GB) | $0.02 | $0.023/GB/month |
| **CloudWatch** | 5 alarms, 5 metrics | $0 | First 10 free |
| **SNS** | Email notifications | $0 | First 1,000 emails free |
| **Data Transfer** | Minimal (VPC internal) | $0 | No internet egress |

**Total Estimated Cost:** **$21.62 - $30.12/month**

---

## Cost Optimization Decisions

### What We're Paying For

**VPC Endpoints ($21.60/month):**
- Required for Session Manager in private subnet
- **Worth it because:** Eliminates need for NAT Gateway ($32/month)
- **Savings:** $10.40/month vs NAT Gateway approach

### What We're NOT Paying For

❌ **NAT Gateway** - Not needed (using VPC endpoints instead)  
❌ **Elastic IPs** - No public-facing resources  
❌ **Application Load Balancer** - No web applications  
❌ **Additional EBS volumes** - Single 8 GB volume sufficient  
❌ **RDS Database** - Not needed for this project  
❌ **VPC Flow Logs** - Optional (would add ~$5-10/month)

---

## Free Tier Benefits

This environment is designed to maximize AWS Free Tier usage:

| Service | Free Tier Allowance | Usage in This Project |
|---------|---------------------|----------------------|
| **EC2** | 750 hours/month t2.micro | ~730 hours (1 instance) ✅ |
| **EBS** | 30 GB General Purpose SSD | 8 GB ✅ |
| **CloudTrail** | 1 trail with management events | 1 trail ✅ |
| **CloudWatch** | 10 alarms, 10 metrics | 5 alarms, 5 metrics ✅ |
| **S3** | 5 GB standard storage | <1 GB ✅ |
| **Data Transfer** | 100 GB/month outbound | Minimal usage ✅ |

**Free Tier Valid:** 12 months from AWS account creation

---

## Cost by Usage Pattern

### If Running 24/7

**Monthly Cost:** ~$22-30

- EC2 always running
- VPC endpoints always active
- Continuous CloudTrail logging

### If Running During Work Hours Only (8hrs/day, 5 days/week)

**Monthly Cost:** ~$15-20

- Stop EC2 when not in use (save ~$6/month)
- VPC endpoints still charged 24/7
- CloudTrail continues logging

### If Running for Testing Only (few hours/week)

**Monthly Cost:** ~$5-10

- Stop EC2 between sessions
- Consider deleting VPC endpoints when not in use (recreate as needed)
- Keep CloudTrail for audit history

---

## How to Minimize Costs

### Daily Operations

1. **Stop EC2 when not in use**
   ```bash
   aws ec2 stop-instances --instance-ids i-xxxxx
   ```
   Saves: ~$0.28/day for t2.micro

2. **Delete unused Elastic IPs**
   - Unattached EIPs cost $0.005/hour ($3.60/month)
   - Check EC2 → Elastic IPs → Release unused

3. **Set S3 lifecycle policies**
   ```bash
   # Delete CloudTrail logs older than 90 days
   # Saves: ~$0.02/month per GB deleted
   ```

### Weekly Maintenance

1. **Review CloudWatch Logs retention**
   - Set to 7 or 30 days instead of "Never expire"
   - Logs cost $0.50/GB/month

2. **Check for orphaned resources**
   - Snapshots: $0.05/GB/month
   - Unused security groups: Free but good to clean up

### Monthly Review

1. **Cost Explorer**
   - Check AWS Cost Explorer for unexpected charges
   - Look for trends or anomalies

2. **Budget Alert**
   - Verify budget alert is working
   - Adjust threshold if needed

---

## Cost Comparison: Different Architectures

### This Project (VPC Endpoints)
- **Cost:** $22/month
- **Pros:** Secure, no public internet exposure
- **Cons:** Endpoints cost money even when unused

### With NAT Gateway
- **Cost:** $32/month + data processing fees
- **Pros:** Full internet access from private subnet
- **Cons:** More expensive, broader attack surface

### Public Subnet Only
- **Cost:** $8/month (just EC2)
- **Pros:** Cheapest option
- **Cons:** Poor security, public exposure, can't use for secure environment demo

---

## Expected Charges Timeline

**Day 1-30 (First Month):**
- VPC Endpoints: ~$21.60
- EC2 (if beyond free tier): ~$8.50
- Misc (S3, CloudWatch): ~$0.02
- **Total:** ~$22-30

**After 12 Months (Free Tier Expired):**
- All charges continue
- EC2 no longer free (add $8.50/month)
- **Total:** ~$30/month

---

## Cost Alerts Setup

**Budget Configuration:**
- Monthly budget: $10 (or $25 if beyond free tier)
- Alert threshold: 80% ($8 or $20)
- Notification: Email

**Why this threshold:**
- Normal operation: ~$22/month
- Alert triggers if approaching $10+ over budget
- Prevents surprise charges from forgotten resources

---

## Teardown Cost Savings

When project is complete, delete resources in this order:

1. **EC2 Instance** → Saves $8.50/month
2. **VPC Endpoints** → Saves $21.60/month
3. **CloudTrail** → Minimal savings (~$0)
4. **S3 Bucket** → Saves $0.02/month

**Total Monthly Savings After Cleanup:** ~$30/month

See `cleanup-guide.md` for step-by-step teardown instructions.

---

## Cost Monitoring Commands

```bash
# Check current month's costs (requires Cost Explorer enabled)
aws ce get-cost-and-usage \
  --time-period Start=2024-02-01,End=2024-02-28 \
  --granularity MONTHLY \
  --metrics UnblendedCost

# List running instances
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,LaunchTime]'

# Check for unused Elastic IPs
aws ec2 describe-addresses \
  --query 'Addresses[?AssociationId==null]'
```

---

## References

- [AWS Pricing Calculator](https://calculator.aws/)
- [EC2 Pricing](https://aws.amazon.com/ec2/pricing/)
- [VPC Pricing](https://aws.amazon.com/vpc/pricing/)
- [Free Tier Details](https://aws.amazon.com/free/)
