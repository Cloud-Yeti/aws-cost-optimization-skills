# AWS Cost Optimization Skills for Claude Code

**Stop overpaying for AWS.** These Claude Code skills find waste, recommend fixes, and track savings — all from your terminal using the AWS CLI.

Built by a 13x AWS certified ex-Amazon engineer who spent 6 years running quarterly business reviews on enterprise cloud accounts.

## Skills

| Skill | What It Does |
|-------|-------------|
| **[aws-cost-audit](aws-cost-audit/)** | Full AWS account waste scan. Finds idle resources, oversized instances, zombie volumes, missing reservations, unoptimized storage, AI/ML cost waste. Generates a prioritized savings report. |
| **[cur-analysis](cur-analysis/)** | Deep dive into your Cost and Usage Report. Top spend by service, tag, account. Data transfer breakdown. RI/SP utilization gaps. Anomaly detection. |

## Quick Start

```bash
# Clone and symlink
git clone https://github.com/Cloud-Yeti/aws-cost-optimization-skills.git
ln -s $(pwd)/aws-cost-optimization-skills/aws-cost-audit ~/.claude/skills/aws-cost-audit
ln -s $(pwd)/aws-cost-optimization-skills/cur-analysis ~/.claude/skills/cur-analysis
```

Then in Claude Code:
```
Run an AWS cost audit on this account
```
or
```
Analyze my CUR data for the last 3 months
```

## Requirements

- AWS CLI configured (`aws sts get-caller-identity` must work)
- Read-only IAM permissions (these skills never modify your infrastructure)
- For CUR analysis: CUR 2.0 export configured to S3 + Athena table

## What the Cost Audit Checks

The `aws-cost-audit` skill runs 15+ checks across these categories:

**Zombie Resources** (delete immediately)
- Unattached EBS volumes, unused Elastic IPs, old snapshots, stopped instances still paying for EBS, orphaned network interfaces, ECR repos without lifecycle policies, S3 incomplete multipart uploads

**Idle Resources** (check metrics, then decide)
- EC2 with <5% CPU over 14 days, idle NAT Gateways, load balancers with no healthy targets, RDS with 0 connections, idle SageMaker endpoints, dead Lambda functions, inactive VPC endpoints

**Configuration Waste** (fix settings)
- CloudWatch logs with no retention (default = forever), gp2 volumes (should be gp3), old-generation instances (t2/m4/c4), Lambda on x86 instead of arm64, S3 without Intelligent-Tiering

**Cost Intelligence** (spend analysis)
- Top 10 services by cost, Cost Optimization Hub recommendations, RI/SP coverage gaps, Compute Optimizer rightsizing, cost forecast, data transfer breakdown

**AI/ML Costs** (if applicable)
- Bedrock model spend, SageMaker idle endpoints, Studio running apps, provisioned throughput utilization

**Networking** (the silent killer)
- NAT Gateway data processing costs, missing VPC endpoints for S3/DynamoDB/ECR, cross-AZ data transfer patterns

## What CUR Analysis Does

The `cur-analysis` skill queries your Cost and Usage Report via Athena:

- **Top spend breakdown** by service, linked account, region, tag
- **Data transfer costs** broken down by type (inter-AZ, NAT, internet, VPC endpoints)
- **RI/SP utilization** — are you using what you paid for?
- **Daily cost trends** — spot anomalies before they become budget problems
- **Untagged spend** — how much of your bill can't be attributed to a team or project
- **AI/ML cost tracking** — Bedrock and SageMaker spend by model, endpoint, and usage type

## Want a Hands-On Audit?

These skills give you the playbook. If you want someone to run it for you:

**[Book a consultation](https://cloudyeti.io/meet)** — I'll audit your AWS + AI infrastructure costs and deliver a prioritized savings roadmap.

## About

I'm [Saurav Sharma](https://github.com/ravsau) — 13x AWS certified, 6 years at Amazon (Senior TAM + SDE), HashiCorp Ambassador. I built LLM platforms at Amazon and now help teams cut cloud and AI infrastructure costs through [CloudYeti](https://cloudyeti.io).

- YouTube: [CloudYeti](https://youtube.com/cloudyeti) (15K+ subscribers)
- LinkedIn: [sauravsharma93](https://linkedin.com/in/sauravsharma93)

## License

MIT
