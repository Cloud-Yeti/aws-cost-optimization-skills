# AWS Cost Optimization Skills for Claude Code

**Stop overpaying for AWS.** These Claude Code skills find waste, recommend fixes, and track savings — all from your terminal.

Built by a 13x AWS certified ex-Amazon engineer who spent 6 years running quarterly business reviews on enterprise cloud accounts.

## Skills

| Skill | What It Does | Install |
|-------|-------------|---------|
| **[aws-cost-audit](aws-cost-audit/)** | Full AWS account waste scan. Finds idle resources, oversized instances, missing reservations, unoptimized storage. | `claude skill install Cloud-Yeti/aws-cost-optimization-skills/aws-cost-audit` |
| *bedrock-cost-optimizer* | Analyze Bedrock model usage, recommend routing (cheap models for simple tasks). | Coming soon |
| *gpu-rightsizer* | Check GPU instance utilization, recommend downsizing or Spot. | Coming soon |
| *ri-savings-plan-advisor* | Analyze RI/SP coverage gaps, recommend purchases with ROI math. | Coming soon |
| *s3-lifecycle-optimizer* | Find S3 buckets without lifecycle policies, estimate savings from tiering. | Coming soon |
| *4t-framework-audit* | Full Tokens/Tiers/Time/Tags audit for AI + cloud cost optimization. | Coming soon |

## Quick Start

```bash
# Clone and symlink
git clone https://github.com/Cloud-Yeti/aws-cost-optimization-skills.git
ln -s $(pwd)/aws-cost-optimization-skills/aws-cost-audit ~/.claude/skills/aws-cost-audit

# Or just copy the SKILL.md into your project
cp aws-cost-optimization-skills/aws-cost-audit/SKILL.md .claude/skills/aws-cost-audit/SKILL.md
```

Then in Claude Code:
```
Run an AWS cost audit on this account
```

## What the Cost Audit Finds

The `aws-cost-audit` skill checks for the most common sources of AWS waste:

- **Idle EC2 instances** — Running instances with <5% CPU for 7+ days
- **Oversized instances** — Instances using <20% of their allocated resources
- **Unattached EBS volumes** — Volumes not attached to any instance (you're still paying)
- **Old EBS snapshots** — Snapshots older than 90 days with no lifecycle policy
- **S3 without lifecycle** — Buckets in Standard tier that should be in IA/Glacier
- **Unused Elastic IPs** — Allocated but unattached ($3.65/month each since Feb 2024)
- **NAT Gateway data transfer** — Often the silent budget killer
- **Missing Reserved Instances / Savings Plans** — On-demand when you could save 30-60%
- **Idle RDS instances** — Databases with 0 connections
- **Oversized Lambda functions** — Over-allocated memory burning money per invocation
- **Bedrock/SageMaker AI costs** — Model selection, batch vs real-time, caching opportunities
- **Untagged resources** — Can't optimize what you can't attribute

## Requirements

- AWS CLI configured (`aws sts get-caller-identity` should work)
- Read-only IAM permissions (the skill never modifies your infrastructure)
- Claude Code with Bash tool access

## The 4T Framework

These skills are built on the **4T Framework for AI & Cloud Cost Optimization**:

1. **Tokens** — Prompt optimization, caching, compression (50-90% savings on AI costs)
2. **Tiers** — Model routing: cheap models for simple tasks, premium for complex (25-40% savings)
3. **Time** — Batch vs real-time, Spot instances, scheduled scaling (50-80% savings)
4. **Tags** — Cost allocation, chargeback/showback, accountability (the governance foundation)

## Want a Hands-On Audit?

These skills give you the playbook. If you want someone to run it for you:

**[Book a consultation](https://cloudyeti.io/meet)** — I'll audit your AWS + AI infrastructure costs and deliver a prioritized savings roadmap.

## About

I'm [Saurav Sharma](https://github.com/ravsau) — 13x AWS certified, 6 years at Amazon (Senior TAM + SDE), HashiCorp Ambassador. I built LLM platforms at Amazon and now help teams cut cloud and AI infrastructure costs through [CloudYeti](https://cloudyeti.io).

- YouTube: [CloudYeti](https://youtube.com/cloudyeti) (15K+ subscribers)
- LinkedIn: [sauravsharma93](https://linkedin.com/in/sauravsharma93)

## License

MIT
