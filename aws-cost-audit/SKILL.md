# AWS Cost Audit

Scan an AWS account for waste. Find idle resources, oversized instances, missing reservations, unoptimized storage, and AI cost inefficiencies. Generate a prioritized savings report.

## When to Use

- "Audit my AWS costs"
- "Find waste in this AWS account"
- "How much am I overpaying on AWS?"
- "Run a cost optimization scan"
- Before any cloud cost review meeting
- Monthly cost hygiene check

## Prerequisites

- AWS CLI configured and authenticated (`aws sts get-caller-identity` must succeed)
- Read-only IAM access (this skill NEVER creates, modifies, or deletes resources)
- Bash tool access in Claude Code

## The Audit

Run each check below using the AWS CLI. Collect findings, calculate estimated waste, and generate the final report.

### Step 0: Verify Access and Get Account Context

```bash
aws sts get-caller-identity
aws ce get-cost-and-usage \
  --time-period Start=$(date -v-30d +%Y-%m-%d 2>/dev/null || date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --query 'ResultsByTime[].Total.BlendedCost.Amount' \
  --output text
```

Record total monthly spend. This is the denominator for all savings percentages.

### Step 1: EC2 — Idle and Oversized Instances

```bash
# Find running instances
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,Launch:LaunchTime,Name:Tags[?Key==`Name`]|[0].Value}' \
  --output table

# Check CPU utilization (last 7 days) for each instance
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=INSTANCE_ID \
  --start-time $(date -v-7d +%Y-%m-%dT%H:%M:%S 2>/dev/null || date -d '7 days ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date +%Y-%m-%dT%H:%M:%S) \
  --period 86400 \
  --statistics Average \
  --output table
```

**Flag as waste if:**
- Average CPU < 5% over 7 days = IDLE (recommend stop or terminate)
- Average CPU < 20% = OVERSIZED (recommend downsize)
- Instance running 24/7 with no Reserved Instance = ON-DEMAND WASTE

**Estimate savings:** Look up instance hourly price, multiply by 730 hours/month.

### Step 2: EBS — Unattached Volumes and Old Snapshots

```bash
# Unattached volumes (you're paying for storage with zero use)
aws ec2 describe-volumes \
  --filters "Name=status,Values=available" \
  --query 'Volumes[].{ID:VolumeId,Size:Size,Type:VolumeType,Created:CreateTime}' \
  --output table

# Snapshots older than 90 days
aws ec2 describe-snapshots \
  --owner-ids self \
  --query 'Snapshots[?StartTime<=`'$(date -v-90d +%Y-%m-%d 2>/dev/null || date -d '90 days ago' +%Y-%m-%d)'`].{ID:SnapshotId,Size:VolumeSize,Date:StartTime,Desc:Description}' \
  --output table
```

**Estimate savings:**
- Unattached gp3: $0.08/GB/month
- Old snapshots: $0.05/GB/month

### Step 3: S3 — Buckets Without Lifecycle Policies

```bash
# List all buckets
aws s3api list-buckets --query 'Buckets[].Name' --output text

# For each bucket, check lifecycle configuration
# (script this in a loop)
for bucket in $(aws s3api list-buckets --query 'Buckets[].Name' --output text); do
  lifecycle=$(aws s3api get-bucket-lifecycle-configuration --bucket $bucket 2>/dev/null)
  if [ $? -ne 0 ]; then
    size=$(aws s3api list-objects-v2 --bucket $bucket --query 'Contents[].Size' --output text 2>/dev/null | awk '{sum+=$1} END {printf "%.2f GB\n", sum/1073741824}')
    echo "NO LIFECYCLE: $bucket ($size)"
  fi
done
```

**Flag as waste if:** Bucket has >10GB in Standard tier with no lifecycle policy. Recommend Intelligent-Tiering or IA transition rules.

### Step 4: Elastic IPs — Unused (Charged Since Feb 2024)

```bash
aws ec2 describe-addresses \
  --query 'Addresses[?AssociationId==null].{IP:PublicIp,AllocId:AllocationId}' \
  --output table
```

**Cost:** $3.65/month per unused EIP (since AWS started charging Feb 2024). Also $0.005/hr for ALL public IPv4 addresses.

### Step 5: NAT Gateway — The Silent Budget Killer

```bash
# Find NAT Gateways
aws ec2 describe-nat-gateways \
  --filter "Name=state,Values=available" \
  --query 'NatGateways[].{ID:NatGatewayId,Subnet:SubnetId,VPC:VpcId}' \
  --output table

# Check data processing costs (last 30 days)
aws cloudwatch get-metric-statistics \
  --namespace AWS/NATGateway \
  --metric-name BytesOutToDestination \
  --dimensions Name=NatGatewayId,Value=NAT_GW_ID \
  --start-time $(date -v-30d +%Y-%m-%dT%H:%M:%S 2>/dev/null || date -d '30 days ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date +%Y-%m-%dT%H:%M:%S) \
  --period 2592000 \
  --statistics Sum \
  --output text
```

**Cost:** $0.045/hour ($32.85/month) per NAT GW + $0.045/GB processed. Recommend VPC endpoints for S3/DynamoDB to reduce data transfer.

### Step 6: RDS — Idle Databases

```bash
aws rds describe-db-instances \
  --query 'DBInstances[].{ID:DBInstanceIdentifier,Class:DBInstanceClass,Engine:Engine,MultiAZ:MultiAZ,Status:DBInstanceStatus}' \
  --output table

# Check connections (last 7 days)
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=DB_INSTANCE_ID \
  --start-time $(date -v-7d +%Y-%m-%dT%H:%M:%S 2>/dev/null || date -d '7 days ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date +%Y-%m-%dT%H:%M:%S) \
  --period 86400 \
  --statistics Maximum \
  --output text
```

**Flag as waste if:** Max connections = 0 over 7 days. Also flag Multi-AZ on dev/staging databases.

### Step 7: Reserved Instances and Savings Plans Coverage

```bash
# Current RI coverage
aws ce get-reservation-coverage \
  --time-period Start=$(date -v-30d +%Y-%m-%d 2>/dev/null || date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --output table

# Savings Plans coverage
aws ce get-savings-plans-coverage \
  --time-period Start=$(date -v-30d +%Y-%m-%d 2>/dev/null || date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --output table

# RI purchase recommendations
aws ce get-reservation-purchase-recommendation \
  --service "Amazon Elastic Compute Cloud - Compute" \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --lookback-period-in-days SIXTY_DAYS
```

**Target:** 70%+ coverage. Every % below that is on-demand premium you're paying unnecessarily.

### Step 8: Lambda — Over-Allocated Memory

```bash
aws lambda list-functions \
  --query 'Functions[].{Name:FunctionName,Memory:MemorySize,Runtime:Runtime,Timeout:Timeout}' \
  --output table
```

Cross-reference with CloudWatch invocation metrics. Functions with 512MB+ allocated but <128MB actual usage are over-provisioned. Use AWS Lambda Power Tuning to find the optimal memory setting.

### Step 9: Bedrock / SageMaker AI Costs (if applicable)

```bash
# Bedrock model invocation costs (last 30 days)
aws ce get-cost-and-usage \
  --time-period Start=$(date -v-30d +%Y-%m-%d 2>/dev/null || date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity DAILY \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Bedrock"]}}' \
  --metrics BlendedCost \
  --output table

# SageMaker endpoints (often left running)
aws sagemaker list-endpoints \
  --query 'Endpoints[].{Name:EndpointName,Status:EndpointStatus,Created:CreationTime}' \
  --output table
```

**Flag:** SageMaker endpoints running 24/7 for batch workloads. Recommend serverless inference or async endpoints. Check if Bedrock batch inference (50% cheaper) can replace real-time calls.

### Step 10: Untagged Resources

```bash
# Find resources without cost allocation tags
aws resourcegroupstaggingapi get-resources \
  --query 'ResourceTagMappingList[?Tags==`[]`].ResourceARN' \
  --output text | head -50
```

**Why it matters:** Untagged resources can't be attributed to teams or projects. No attribution = no accountability = no optimization. Recommend a tagging policy with at minimum: Environment, Team, Project, CostCenter.

### Step 11: Compute Optimizer Recommendations (Free)

```bash
aws compute-optimizer get-ec2-instance-recommendations \
  --query 'instanceRecommendations[].{Instance:instanceArn,Finding:finding,Current:currentInstanceType,Recommended:recommendationOptions[0].instanceType,Savings:recommendationOptions[0].estimatedMonthlySavings.value}' \
  --output table
```

AWS Compute Optimizer uses ML to recommend rightsizing. It's free and often has actionable recommendations already waiting.

## Output: The Report

Generate a markdown report with this structure:

```markdown
# AWS Cost Audit Report
**Account:** [Account ID]
**Date:** YYYY-MM-DD
**Monthly Spend:** $X,XXX

## Executive Summary
- Total estimated waste: $X,XXX/month ($XX,XXX/year)
- X% of monthly spend is potentially recoverable
- Top 3 findings: [list]

## Findings by Category

### Critical (>$500/month savings)
| Finding | Resource | Current Cost | Savings | Action |
|---------|----------|-------------|---------|--------|
| ... | ... | ... | ... | ... |

### High (>$100/month savings)
| ... |

### Medium (<$100/month savings)
| ... |

## Quick Wins (Do This Week)
1. [Actionable item with exact AWS CLI command to fix]
2. ...
3. ...

## 30-Day Recommendations
1. ...

## 90-Day Architecture Recommendations
1. ...

## Reserved Instance / Savings Plan Recommendations
- Current coverage: X%
- Recommended purchases: [list with ROI math]
- Estimated annual savings: $X,XXX
```

Save the report to `aws-cost-audit-YYYY-MM-DD.md` in the current directory.

## Safety

This skill is READ-ONLY. It uses `describe`, `list`, `get` API calls only. It never creates, modifies, terminates, or deletes any AWS resource. All recommendations are advisory.

## Credits

Built by [Saurav Sharma](https://github.com/ravsau) — 13x AWS Certified, ex-Amazon (6 years as Senior TAM + SDE), [CloudYeti](https://cloudyeti.io).

Want a hands-on audit with expert recommendations? **[Book a consultation](https://cloudyeti.io/meet)**
