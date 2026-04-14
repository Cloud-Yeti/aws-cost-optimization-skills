# AWS Cost Audit

Scan an AWS account for waste using the AWS CLI. Finds idle resources, oversized instances, zombie volumes, missing reservations, unoptimized storage, and AI cost inefficiencies. Generates a prioritized savings report.

This skill is READ-ONLY. It uses `describe`, `list`, `get` CLI calls only. It never creates, modifies, terminates, or deletes any AWS resource.

## When to Use

- "Audit my AWS costs"
- "Find waste in this AWS account"
- "How much am I overpaying on AWS?"
- Before any cloud cost review meeting
- Monthly cost hygiene check

## Prerequisites

- AWS CLI configured and authenticated (`aws sts get-caller-identity` must succeed)
- Read-only IAM access
- Bash tool access in Claude Code

## The Audit

Run each section below. Collect all findings, estimate dollar waste per finding, and generate the report at the end.

---

### Step 0: Account Context and Total Spend

```bash
# Verify access
aws sts get-caller-identity

# Total spend last 30 days
aws ce get-cost-and-usage \
  --time-period Start=$(date -v-30d +%Y-%m-%d 2>/dev/null || date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics UnblendedCost \
  --query 'ResultsByTime[].Total.UnblendedCost.{Amount:Amount,Unit:Unit}' --output table

# Top 10 services by cost
aws ce get-cost-and-usage \
  --time-period Start=$(date -v-30d +%Y-%m-%d 2>/dev/null || date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[].Groups[].{Service:Keys[0],Cost:Metrics.UnblendedCost.Amount}' --output table

# Cost forecast next 30 days
aws ce get-cost-forecast \
  --time-period Start=$(date +%Y-%m-%d),End=$(date -v+30d +%Y-%m-%d 2>/dev/null || date -d '30 days' +%Y-%m-%d) \
  --granularity MONTHLY --metric UNBLENDED_COST \
  --query '{Forecast:Total.Amount,Unit:Total.Unit}' --output table
```

Record total monthly spend. This is the denominator for all savings percentages.

---

### Step 1: Zombie Resources (Delete Immediately)

These are resources costing money with zero utilization.

```bash
# Unattached EBS volumes
aws ec2 describe-volumes --filters Name=status,Values=available \
  --query 'Volumes[].{ID:VolumeId,Size:Size,Type:VolumeType,Created:CreateTime}' --output table

# Unused Elastic IPs ($3.65/mo each since Feb 2024)
aws ec2 describe-addresses --filters Name=domain,Values=vpc \
  --query 'Addresses[?AssociationId==null].{IP:PublicIp,AllocId:AllocationId}' --output table

# Stopped EC2 instances (still paying for EBS volumes)
aws ec2 describe-instances --filters Name=instance-state-name,Values=stopped \
  --query 'Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,Name:Tags[?Key==`Name`]|[0].Value}' --output table

# Old EBS snapshots (>90 days, no lifecycle)
CUTOFF=$(date -v-90d +%Y-%m-%dT00:00:00 2>/dev/null || date -d '90 days ago' +%Y-%m-%dT00:00:00)
aws ec2 describe-snapshots --owner-ids self \
  --query "Snapshots[?StartTime<='${CUTOFF}'].{ID:SnapshotId,Size:VolumeSize,Date:StartTime}" --output table

# Orphaned network interfaces
aws ec2 describe-network-interfaces --filters Name=status,Values=available \
  --query 'NetworkInterfaces[].{ID:NetworkInterfaceId,Type:InterfaceType,Desc:Description}' --output table

# ECR repos without lifecycle policy (images accumulate forever)
for repo in $(aws ecr describe-repositories --query 'repositories[].repositoryName' --output text 2>/dev/null); do
  aws ecr get-lifecycle-policy --repository-name "$repo" 2>/dev/null || echo "NO LIFECYCLE: $repo"
done

# S3 buckets without lifecycle policy
for bucket in $(aws s3api list-buckets --query 'Buckets[].Name' --output text); do
  aws s3api get-bucket-lifecycle-configuration --bucket "$bucket" 2>/dev/null || echo "NO LIFECYCLE: $bucket"
done

# Unused Secrets Manager secrets (>90 days since last access, $0.40/secret/mo)
aws secretsmanager list-secrets \
  --query 'SecretList[?LastAccessedDate!=null && LastAccessedDate<=`'$(date -v-90d +%Y-%m-%d 2>/dev/null || date -d '90 days ago' +%Y-%m-%d)'`].{Name:Name,LastAccessed:LastAccessedDate}' --output table
```

**Pricing reference:**
- Unattached gp3: $0.08/GB/month
- Old snapshots: $0.05/GB/month
- Unused EIP: $3.65/month each
- Public IPv4: $3.65/month per address (all addresses, since Feb 2024)

---

### Step 2: Idle Resources (Check Metrics, Then Decide)

```bash
# Idle EC2 instances (avg CPU <5% over 14 days)
for iid in $(aws ec2 describe-instances --filters Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[].InstanceId' --output text); do
  AVG=$(aws cloudwatch get-metric-statistics --namespace AWS/EC2 \
    --metric-name CPUUtilization --period 86400 --statistics Average \
    --start-time $(date -v-14d +%Y-%m-%dT%H:%M:%S 2>/dev/null || date -d '14 days ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date +%Y-%m-%dT%H:%M:%S) \
    --dimensions Name=InstanceId,Value=$iid \
    --query 'Datapoints[].Average' --output text 2>/dev/null)
  echo "$iid: avg CPU ${AVG:-N/A}%"
done

# Idle NAT Gateways (0 bytes in 14 days — still costs $32.85/mo just to exist)
for nat in $(aws ec2 describe-nat-gateways --filter Name=state,Values=available \
  --query 'NatGateways[].NatGatewayId' --output text); do
  BYTES=$(aws cloudwatch get-metric-statistics --namespace AWS/NATGateway \
    --metric-name BytesOutToDestination --period 1209600 --statistics Sum \
    --start-time $(date -v-14d +%Y-%m-%dT%H:%M:%S 2>/dev/null || date -d '14 days ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date +%Y-%m-%dT%H:%M:%S) \
    --dimensions Name=NatGatewayId,Value=$nat \
    --query 'Datapoints[].Sum' --output text 2>/dev/null)
  echo "$nat: ${BYTES:-0} bytes processed"
done

# Load balancers with no healthy targets
for arn in $(aws elbv2 describe-load-balancers --query 'LoadBalancers[].LoadBalancerArn' --output text 2>/dev/null); do
  for tg in $(aws elbv2 describe-target-groups --load-balancer-arn "$arn" --query 'TargetGroups[].TargetGroupArn' --output text 2>/dev/null); do
    HEALTHY=$(aws elbv2 describe-target-health --target-group-arn "$tg" \
      --query 'TargetHealthDescriptions[?TargetHealth.State==`healthy`]' --output text 2>/dev/null)
    [ -z "$HEALTHY" ] && echo "IDLE LB: $arn"
  done
done

# Idle RDS instances (0 connections over 14 days)
for db in $(aws rds describe-db-instances --query 'DBInstances[].DBInstanceIdentifier' --output text 2>/dev/null); do
  CONNS=$(aws cloudwatch get-metric-statistics --namespace AWS/RDS \
    --metric-name DatabaseConnections --period 1209600 --statistics Maximum \
    --start-time $(date -v-14d +%Y-%m-%dT%H:%M:%S 2>/dev/null || date -d '14 days ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date +%Y-%m-%dT%H:%M:%S) \
    --dimensions Name=DBInstanceIdentifier,Value=$db \
    --query 'Datapoints[].Maximum' --output text 2>/dev/null)
  echo "$db: max connections ${CONNS:-0}"
done

# Idle SageMaker endpoints (0 invocations in 7 days — costs $0.05-$32.77/hr)
for ep in $(aws sagemaker list-endpoints --query 'Endpoints[].EndpointName' --output text 2>/dev/null); do
  INV=$(aws cloudwatch get-metric-statistics --namespace AWS/SageMaker \
    --metric-name Invocations --period 604800 --statistics Sum \
    --start-time $(date -v-7d +%Y-%m-%dT%H:%M:%S 2>/dev/null || date -d '7 days ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date +%Y-%m-%dT%H:%M:%S) \
    --dimensions Name=EndpointName,Value=$ep \
    --query 'Datapoints[].Sum' --output text 2>/dev/null)
  echo "$ep: ${INV:-0} invocations in 7 days"
done

# Dead Lambda functions (0 invocations in 30 days)
for func in $(aws lambda list-functions --query 'Functions[].FunctionName' --output text 2>/dev/null); do
  INV=$(aws cloudwatch get-metric-statistics --namespace AWS/Lambda \
    --metric-name Invocations --period 2592000 --statistics Sum \
    --start-time $(date -v-30d +%Y-%m-%dT%H:%M:%S 2>/dev/null || date -d '30 days ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date +%Y-%m-%dT%H:%M:%S) \
    --dimensions Name=FunctionName,Value=$func \
    --query 'Datapoints[].Sum' --output text 2>/dev/null)
  [ -z "$INV" ] || [ "$INV" = "0.0" ] && echo "DEAD: $func"
done

# Inactive VPC interface endpoints ($7.20/mo per AZ)
aws ec2 describe-vpc-endpoints --filters Name=vpc-endpoint-type,Values=Interface \
  --query 'VpcEndpoints[].{ID:VpcEndpointId,Service:ServiceName,State:State}' --output table

# SageMaker Studio running apps (hidden cost)
aws sagemaker list-apps --query 'Apps[?Status==`InService`].{Domain:DomainId,User:UserProfileName,Type:AppType}' --output table 2>/dev/null
```

**Flag as waste if:**
- EC2 avg CPU < 5% over 14 days = IDLE (recommend stop/terminate)
- EC2 avg CPU < 20% = OVERSIZED (recommend downsize)
- NAT Gateway with 0 bytes = IDLE ($32.85/mo for nothing)
- RDS max connections = 0 = IDLE
- SageMaker endpoint with 0 invocations = IDLE

---

### Step 3: Configuration Waste (Fix Settings for Instant Savings)

```bash
# CloudWatch log groups with no retention (default = FOREVER = $$$)
aws logs describe-log-groups \
  --query 'logGroups[?!retentionInDays].{Name:logGroupName,StoredBytes:storedBytes}' --output table

# gp2 volumes (should be gp3 — 20% cheaper, better performance)
aws ec2 describe-volumes --filters Name=volume-type,Values=gp2 \
  --query 'Volumes[].{ID:VolumeId,Size:Size,State:State}' --output table

# Old-generation instances (t2/m4/c4/r4/m3/c3 — upgrade for 10-40% savings + Graviton)
aws ec2 describe-instances --filters Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[?contains(`t2. m4. c4. r4. m3. c3. `,join(` `,[InstanceType,` `]))].{ID:InstanceId,Type:InstanceType,Name:Tags[?Key==`Name`]|[0].Value}' --output table

# Non-Graviton instances (Graviton = 20-40% cheaper for most workloads)
aws ec2 describe-instances --filters Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[?!contains(InstanceType,`g.`)].{ID:InstanceId,Type:InstanceType}' --output table

# Lambda on x86 (arm64 = 20% cheaper)
aws lambda list-functions \
  --query 'Functions[?!contains(Architectures[0],`arm64`)].{Name:FunctionName,Arch:Architectures[0],Memory:MemorySize}' --output table

# Lambda over-provisioned memory (check if allocated >> actual usage)
aws lambda list-functions \
  --query 'Functions[?MemorySize>`512`].{Name:FunctionName,Memory:MemorySize,Runtime:Runtime}' --output table

# Lambda with old runtimes (end of support, no patches)
aws lambda list-functions \
  --query 'Functions[?Runtime==`python3.8`||Runtime==`python3.7`||Runtime==`nodejs14.x`||Runtime==`nodejs16.x`].{Name:FunctionName,Runtime:Runtime}' --output table

# RDS Multi-AZ on non-production (common waste — doubles the cost)
aws rds describe-db-instances \
  --query 'DBInstances[?MultiAZ==`true`].{ID:DBInstanceIdentifier,Class:DBInstanceClass,Engine:Engine}' --output table

# S3 buckets without Intelligent-Tiering or lifecycle
for bucket in $(aws s3api list-buckets --query 'Buckets[].Name' --output text); do
  IT=$(aws s3api list-bucket-intelligent-tiering-configurations --bucket "$bucket" 2>/dev/null)
  LC=$(aws s3api get-bucket-lifecycle-configuration --bucket "$bucket" 2>/dev/null)
  [ -z "$IT" ] && [ -z "$LC" ] && echo "NO TIERING/LIFECYCLE: $bucket"
done

# DynamoDB tables on provisioned mode with low utilization (should be on-demand)
aws dynamodb list-tables --query 'TableNames' --output text | while read table; do
  MODE=$(aws dynamodb describe-table --table-name "$table" \
    --query 'Table.BillingModeSummary.BillingMode' --output text 2>/dev/null)
  [ "$MODE" = "PROVISIONED" ] && echo "PROVISIONED MODE: $table (check if on-demand is cheaper)"
done
```

---

### Step 4: Networking (The Silent Budget Killer)

```bash
# NAT Gateway costs (often #1 surprise on the bill)
aws ce get-cost-and-usage \
  --time-period Start=$(date -v-30d +%Y-%m-%d 2>/dev/null || date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY --metrics UnblendedCost \
  --filter '{"Dimensions":{"Key":"USAGE_TYPE","Values":["NatGateway-Bytes","NatGateway-Hours"]}}' \
  --query 'ResultsByTime[].Total.UnblendedCost.Amount' --output text

# Check if S3/DynamoDB gateway endpoints exist (FREE, saves NAT cost)
aws ec2 describe-vpc-endpoints --filters Name=vpc-endpoint-type,Values=Gateway \
  --query 'VpcEndpoints[].{Service:ServiceName,State:State}' --output table

# Check if ECR VPC endpoint exists (saves NAT cost for container pulls)
aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=*ecr* \
  --query 'VpcEndpoints[].{Service:ServiceName,State:State}' --output table

# Data transfer breakdown
aws ce get-cost-and-usage \
  --time-period Start=$(date -v-30d +%Y-%m-%d 2>/dev/null || date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY --metrics UnblendedCost \
  --filter '{"Dimensions":{"Key":"USAGE_TYPE","Values":["DataTransfer-Out-Bytes","DataTransfer-Regional-Bytes","NatGateway-Bytes"]}}' \
  --query 'ResultsByTime[].Groups[]'
```

**Recommend:** Add S3 and DynamoDB gateway endpoints (free). Add ECR endpoint if running containers. These can eliminate 50-80% of NAT Gateway data processing charges.

---

### Step 5: Reserved Instances and Savings Plans

```bash
# RI coverage (target: 70%+)
aws ce get-reservation-coverage \
  --time-period Start=$(date -v-30d +%Y-%m-%d 2>/dev/null || date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY --output table

# Savings Plans coverage
aws ce get-savings-plans-coverage \
  --time-period Start=$(date -v-30d +%Y-%m-%d 2>/dev/null || date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY --output table

# RI purchase recommendations (with ROI math)
aws ce get-reservation-purchase-recommendation \
  --service "Amazon Elastic Compute Cloud - Compute" \
  --term-in-years ONE_YEAR --payment-option NO_UPFRONT \
  --lookback-period-in-days SIXTY_DAYS

# Savings Plans purchase recommendations
aws ce get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP --lookback-period-in-days SIXTY_DAYS \
  --term-in-years ONE_YEAR --payment-option NO_UPFRONT
```

---

### Step 6: AI/ML Costs

```bash
# Bedrock spend (last 30 days)
aws ce get-cost-and-usage \
  --time-period Start=$(date -v-30d +%Y-%m-%d 2>/dev/null || date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity DAILY --metrics UnblendedCost \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Bedrock"]}}' \
  --query 'ResultsByTime[].{Date:TimePeriod.Start,Cost:Total.UnblendedCost.Amount}' --output table

# SageMaker spend
aws ce get-cost-and-usage \
  --time-period Start=$(date -v-30d +%Y-%m-%d 2>/dev/null || date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity DAILY --metrics UnblendedCost \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon SageMaker"]}}' \
  --query 'ResultsByTime[].{Date:TimePeriod.Start,Cost:Total.UnblendedCost.Amount}' --output table

# Bedrock provisioned throughput (purchased but possibly underutilized)
aws bedrock list-provisioned-model-throughputs \
  --query 'provisionedModelSummaries[].{Name:provisionedModelName,Model:modelId,Status:status}' --output table 2>/dev/null

# SageMaker training jobs still running
aws sagemaker list-training-jobs --status-equals InProgress \
  --query 'TrainingJobSummaries[].{Name:TrainingJobName,Created:CreationTime}' --output table 2>/dev/null
```

**AI cost flags:**
- Bedrock: Are you using the cheapest model that works? (Nova Micro vs Claude Sonnet = 535x price difference)
- Bedrock: Batch inference is 25% cheaper than on-demand — is it being used for non-real-time tasks?
- SageMaker: Endpoints running 24/7 for bursty workloads should be serverless inference
- SageMaker: Training on on-demand when managed Spot gives 70% off

---

### Step 7: Cost Optimization Hub (AWS's Own Recommendations)

```bash
# Opt in first (if not already)
aws cost-optimization-hub update-enrollment-status --status Active 2>/dev/null

# All recommendations sorted by savings (quick wins first)
aws cost-optimization-hub list-recommendations \
  --order-by dimension=estimatedMonthlySavings,order=Desc \
  --query 'items[].{Type:currentResourceType,ID:currentResourceSummary,Action:recommendedResourceType,Savings:estimatedMonthlySavings.value}' --output table 2>/dev/null

# Summary by resource type
aws cost-optimization-hub list-recommendation-summaries \
  --group-by ResourceType 2>/dev/null
```

---

### Step 8: Compute Optimizer (ML-Based Rightsizing)

```bash
aws compute-optimizer get-ec2-instance-recommendations \
  --query 'instanceRecommendations[].{Instance:instanceArn,Finding:finding,Current:currentInstanceType,Recommended:recommendationOptions[0].instanceType,MonthlySavings:recommendationOptions[0].estimatedMonthlySavings.value}' --output table
```

---

### Step 9: Tagging and Governance

```bash
# Untagged resources (can't optimize what you can't attribute)
aws resourcegroupstaggingapi get-resources \
  --query 'ResourceTagMappingList[?Tags==`[]`].ResourceARN' --output text | head -50

# Check if cost allocation tags are active
aws ce list-cost-allocation-tags \
  --status Active \
  --query 'CostAllocationTags[].{Key:TagKey,Type:Type,Status:Status}' --output table

# Check if budgets exist
aws budgets describe-budgets --account-id $(aws sts get-caller-identity --query Account --output text) \
  --query 'Budgets[].{Name:BudgetName,Limit:BudgetLimit.Amount,Actual:CalculatedSpend.ActualSpend.Amount}' --output table 2>/dev/null

# Check if cost anomaly detection is enabled
aws ce get-anomaly-monitors \
  --query 'AnomalyMonitors[].{Name:MonitorName,Type:MonitorType}' --output table 2>/dev/null
```

**Governance flags:**
- No budgets configured = no guardrails
- No cost anomaly detection = no early warning
- Untagged resources > 20% of total = attribution problem

---

## Output: The Report

Generate a markdown report:

```markdown
# AWS Cost Audit Report
**Account:** [Account ID]
**Date:** YYYY-MM-DD
**Monthly Spend:** $X,XXX
**Forecasted Next Month:** $X,XXX

## Executive Summary
- Total estimated waste: $X,XXX/month ($XX,XXX/year)
- X% of monthly spend is potentially recoverable
- Top 3 quick wins: [list with dollar amounts]

## Findings

### Critical (>$500/month savings each)
| Finding | Resource | Monthly Cost | Potential Savings | Action |
|---------|----------|-------------|-------------------|--------|

### High (>$100/month savings each)
| Finding | Resource | Monthly Cost | Potential Savings | Action |

### Medium (<$100/month savings each)
| Finding | Resource | Monthly Cost | Potential Savings | Action |

## Quick Wins (Do This Week)
1. [Action with exact AWS CLI command]
2. ...

## 30-Day Recommendations
1. ...

## 90-Day Architecture Recommendations
1. ...

## RI/Savings Plan Recommendations
- Current coverage: X%
- Recommended purchases with ROI math
- Estimated annual savings: $X,XXX

## Governance Gaps
- Tagging coverage: X%
- Budget alerts: [configured/missing]
- Cost anomaly detection: [enabled/missing]
```

Save the report to `aws-cost-audit-YYYY-MM-DD.md` in the current directory.

## Credits

Built by [Saurav Sharma](https://github.com/ravsau) — 13x AWS Certified, ex-Amazon (6 years as Senior TAM + SDE), [CloudYeti](https://cloudyeti.io).

Want a hands-on audit with expert recommendations? **[Book a consultation](https://cloudyeti.io/meet)**
