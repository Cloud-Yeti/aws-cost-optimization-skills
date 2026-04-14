# CUR Analysis — Deep Dive Into Your AWS Bill

Analyze your AWS Cost and Usage Report (CUR) data via Athena. Find top spend drivers, data transfer costs, RI/SP waste, untagged spend, cost anomalies, and AI/ML cost breakdown.

This skill is READ-ONLY. It queries existing CUR data via Athena. It never modifies any resource or data.

## When to Use

- "Analyze my CUR data"
- "Where is my AWS money going?"
- "Show me data transfer costs"
- "How much are we wasting on unused reservations?"
- "Break down costs by team/tag"
- Monthly cost review prep
- FinOps reporting

## Prerequisites

- CUR 2.0 (Data Export) configured and exporting to S3 in Parquet format
- Athena table created on top of CUR data (via Glue Crawler or manual DDL)
- AWS CLI configured with permissions for Athena queries and S3 read
- Know your Athena database name and table name (e.g., `cur_database.cur_table`)

If CUR is not set up yet, see the **Setup Guide** section at the bottom.

## The Analysis

Ask the user for their Athena database and table name, S3 output location for query results, and time period to analyze (default: last 3 months).

Run each query section below. Collect results and generate the final report.

### Helper: Run Athena Query

For each SQL query below, execute via CLI:

```bash
QUERY_ID=$(aws athena start-query-execution \
  --query-string "YOUR_SQL_HERE" \
  --query-execution-context Database=CUR_DATABASE \
  --result-configuration OutputLocation=s3://RESULTS_BUCKET/athena-results/ \
  --output text --query 'QueryExecutionId')

# Wait for completion (poll every 2 seconds)
while true; do
  STATUS=$(aws athena get-query-execution --query-execution-id $QUERY_ID \
    --query 'QueryExecution.Status.State' --output text)
  [ "$STATUS" = "SUCCEEDED" ] && break
  [ "$STATUS" = "FAILED" ] && echo "FAILED" && break
  sleep 2
done

# Get results
aws athena get-query-results --query-execution-id $QUERY_ID --output table
```

---

### Query 1: Total Spend and Top Services

```sql
SELECT
  line_item_product_code AS service,
  ROUND(SUM(line_item_unblended_cost), 2) AS total_cost,
  ROUND(SUM(line_item_unblended_cost) * 100.0 /
    SUM(SUM(line_item_unblended_cost)) OVER(), 2) AS pct_of_total
FROM CUR_DATABASE.CUR_TABLE
WHERE line_item_line_item_type NOT IN ('Credit', 'Refund')
  AND bill_billing_period_start_date >= DATE 'START_DATE'
GROUP BY 1
ORDER BY total_cost DESC
LIMIT 20;
```

---

### Query 2: Daily Spend Trend with Anomaly Detection

```sql
SELECT
  DATE(line_item_usage_start_date) AS day,
  ROUND(SUM(line_item_unblended_cost), 2) AS daily_cost,
  ROUND(AVG(SUM(line_item_unblended_cost)) OVER (
    ORDER BY DATE(line_item_usage_start_date)
    ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING
  ), 2) AS trailing_7d_avg
FROM CUR_DATABASE.CUR_TABLE
WHERE line_item_line_item_type = 'Usage'
  AND bill_billing_period_start_date >= DATE 'START_DATE'
GROUP BY 1
ORDER BY 1;
```

Flag any day where daily_cost > trailing_7d_avg * 1.25 as an ANOMALY (25%+ spike).

---

### Query 3: Cost by Tag (Team/Environment Chargeback)

```sql
SELECT
  COALESCE(resource_tags_user_team, 'UNTAGGED') AS team,
  COALESCE(resource_tags_user_environment, 'UNTAGGED') AS environment,
  line_item_product_code AS service,
  ROUND(SUM(line_item_unblended_cost), 2) AS cost
FROM CUR_DATABASE.CUR_TABLE
WHERE bill_billing_period_start_date >= DATE 'START_DATE'
  AND line_item_line_item_type = 'Usage'
GROUP BY 1, 2, 3
ORDER BY cost DESC
LIMIT 30;
```

Note: Tag column names depend on your tag keys. Common patterns: `resource_tags_user_team`, `resource_tags_user_project`, `resource_tags_user_cost_center`. Adjust column names to match your actual tags.

---

### Query 4: Untagged Spend (The Blind Spot)

```sql
SELECT
  line_item_product_code AS service,
  ROUND(SUM(line_item_unblended_cost), 2) AS untagged_cost,
  COUNT(DISTINCT line_item_resource_id) AS resource_count
FROM CUR_DATABASE.CUR_TABLE
WHERE (resource_tags_user_team IS NULL OR resource_tags_user_team = '')
  AND line_item_line_item_type = 'Usage'
  AND line_item_resource_id != ''
  AND bill_billing_period_start_date >= DATE 'START_DATE'
GROUP BY 1
ORDER BY untagged_cost DESC
LIMIT 20;
```

Calculate: untagged_cost / total_cost = your attribution gap percentage. Target: < 20%.

---

### Query 5: Cost by Linked Account (Multi-Account)

```sql
SELECT
  line_item_usage_account_id AS account_id,
  ROUND(SUM(line_item_unblended_cost), 2) AS total_cost,
  ROUND(SUM(line_item_unblended_cost) * 100.0 /
    SUM(SUM(line_item_unblended_cost)) OVER(), 2) AS pct_of_total
FROM CUR_DATABASE.CUR_TABLE
WHERE bill_billing_period_start_date >= DATE 'START_DATE'
  AND line_item_line_item_type NOT IN ('Credit', 'Refund')
GROUP BY 1
ORDER BY total_cost DESC;
```

---

### Query 6: Data Transfer Breakdown (The Hidden Cost)

```sql
SELECT
  line_item_product_code AS service,
  line_item_usage_type AS transfer_type,
  ROUND(SUM(line_item_usage_amount), 2) AS gb_transferred,
  ROUND(SUM(line_item_unblended_cost), 2) AS cost
FROM CUR_DATABASE.CUR_TABLE
WHERE (line_item_usage_type LIKE '%DataTransfer%'
   OR line_item_usage_type LIKE '%Bytes%'
   OR line_item_usage_type LIKE '%NatGateway%')
  AND bill_billing_period_start_date >= DATE 'START_DATE'
GROUP BY 1, 2
ORDER BY cost DESC
LIMIT 20;
```

Data transfer is typically 15-30% of the bill and poorly understood. Flag: NAT Gateway processing, cross-AZ transfer, internet egress.

---

### Query 7: Most Expensive Individual Resources

```sql
SELECT
  line_item_resource_id AS resource_id,
  line_item_product_code AS service,
  product_instance_type AS instance_type,
  ROUND(SUM(line_item_unblended_cost), 2) AS cost
FROM CUR_DATABASE.CUR_TABLE
WHERE line_item_resource_id != ''
  AND bill_billing_period_start_date >= DATE 'START_DATE'
  AND line_item_line_item_type = 'Usage'
GROUP BY 1, 2, 3
ORDER BY cost DESC
LIMIT 25;
```

---

### Query 8: On-Demand vs Reserved vs Spot vs Savings Plan

```sql
SELECT
  CASE
    WHEN line_item_line_item_type = 'DiscountedUsage' THEN 'Reserved Instance'
    WHEN line_item_line_item_type = 'SavingsPlanCoveredUsage' THEN 'Savings Plan'
    WHEN line_item_usage_type LIKE '%SpotUsage%' THEN 'Spot'
    WHEN line_item_line_item_type = 'Usage' THEN 'On-Demand'
    ELSE line_item_line_item_type
  END AS pricing_model,
  ROUND(SUM(line_item_unblended_cost), 2) AS cost,
  ROUND(SUM(line_item_unblended_cost) * 100.0 /
    SUM(SUM(line_item_unblended_cost)) OVER(), 2) AS pct
FROM CUR_DATABASE.CUR_TABLE
WHERE line_item_product_code = 'AmazonEC2'
  AND bill_billing_period_start_date >= DATE 'START_DATE'
GROUP BY 1
ORDER BY cost DESC;
```

If On-Demand > 30% of EC2 spend, recommend Savings Plans or Reserved Instances.

---

### Query 9: Savings Plan Utilization

```sql
SELECT
  savings_plan_savings_plan_a_r_n AS sp_arn,
  ROUND(SUM(savings_plan_used_commitment), 2) AS used,
  ROUND(SUM(savings_plan_total_commitment_to_date), 2) AS committed,
  ROUND(SUM(savings_plan_used_commitment) * 100.0 /
    NULLIF(SUM(savings_plan_total_commitment_to_date), 0), 1) AS utilization_pct
FROM CUR_DATABASE.CUR_TABLE
WHERE savings_plan_savings_plan_a_r_n != ''
  AND bill_billing_period_start_date >= DATE 'START_DATE'
GROUP BY 1
ORDER BY utilization_pct ASC;
```

Utilization < 80% = money committed but wasted. Each % below 100% is pure loss.

---

### Query 10: Reserved Instance Unused Hours

```sql
SELECT
  reservation_reservation_a_r_n AS ri_arn,
  line_item_product_code AS service,
  ROUND(SUM(reservation_unused_quantity), 2) AS unused_hours,
  ROUND(SUM(reservation_unused_recurring_fee), 2) AS wasted_cost
FROM CUR_DATABASE.CUR_TABLE
WHERE reservation_reservation_a_r_n != ''
  AND bill_billing_period_start_date >= DATE 'START_DATE'
GROUP BY 1, 2
HAVING SUM(reservation_unused_quantity) > 0
ORDER BY wasted_cost DESC;
```

---

### Query 11: AI/ML Cost Breakdown (Bedrock + SageMaker)

```sql
-- Bedrock daily spend
SELECT
  DATE(line_item_usage_start_date) AS day,
  line_item_usage_type AS usage_type,
  ROUND(SUM(line_item_unblended_cost), 2) AS cost
FROM CUR_DATABASE.CUR_TABLE
WHERE line_item_product_code = 'AmazonBedrock'
  AND bill_billing_period_start_date >= DATE 'START_DATE'
GROUP BY 1, 2
ORDER BY 1, cost DESC;

-- SageMaker by usage type (endpoints, training, notebooks, storage)
SELECT
  line_item_usage_type AS usage_type,
  ROUND(SUM(line_item_unblended_cost), 2) AS cost,
  ROUND(SUM(line_item_usage_amount), 2) AS usage_amount
FROM CUR_DATABASE.CUR_TABLE
WHERE line_item_product_code = 'AmazonSageMaker'
  AND bill_billing_period_start_date >= DATE 'START_DATE'
GROUP BY 1
ORDER BY cost DESC;
```

---

### Query 12: Month-over-Month Cost Trend

```sql
SELECT
  DATE_FORMAT(bill_billing_period_start_date, '%Y-%m') AS month,
  ROUND(SUM(line_item_unblended_cost), 2) AS total_cost
FROM CUR_DATABASE.CUR_TABLE
WHERE line_item_line_item_type NOT IN ('Credit', 'Refund')
GROUP BY 1
ORDER BY 1 DESC
LIMIT 12;
```

---

### Query 13: Cost by Region

```sql
SELECT
  product_region AS region,
  ROUND(SUM(line_item_unblended_cost), 2) AS cost
FROM CUR_DATABASE.CUR_TABLE
WHERE product_region != ''
  AND bill_billing_period_start_date >= DATE 'START_DATE'
GROUP BY 1
ORDER BY cost DESC;
```

Multi-region sprawl is a common cost driver. If significant spend in regions with no active workloads, flag for investigation.

---

## Output: The CUR Report

Generate a markdown report:

```markdown
# AWS CUR Analysis Report
**Account(s):** [Account IDs]
**Period:** START_DATE to END_DATE
**Total Spend:** $X,XXX

## Cost Summary
- Top 5 services: [with $ and %]
- Month-over-month trend: [up/down X%]
- Cost forecast: $X,XXX next month

## Cost Anomalies
| Date | Daily Cost | 7-Day Avg | Spike % | Likely Cause |
|------|-----------|-----------|---------|-------------|

## Pricing Model Mix (EC2)
| Model | Cost | % of EC2 |
|-------|------|----------|
| On-Demand | $X | X% |
| Savings Plan | $X | X% |
| Reserved | $X | X% |
| Spot | $X | X% |

## Savings Plan / RI Waste
| Commitment | Used | Utilization | Wasted $/mo |
|------------|------|-------------|-------------|

## Data Transfer Costs
| Type | GB | Cost | Recommendation |
|------|-----|------|---------------|

## Untagged Spend
- Total untagged: $X,XXX (X% of total)
- Top untagged services: [list]

## AI/ML Costs
- Bedrock: $X/month [by model if available]
- SageMaker: $X/month [by usage type]

## Top 25 Most Expensive Resources
| Resource | Service | Cost/mo |
|----------|---------|---------|

## Recommendations
1. [Specific action with estimated savings]
2. ...
```

---

## CUR Setup Guide (If Not Yet Configured)

### Step 1: Create S3 Bucket

```bash
aws s3 mb s3://YOUR-CUR-BUCKET --region us-east-1

aws s3api put-bucket-policy --bucket YOUR-CUR-BUCKET --policy '{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "bcm-data-exports.amazonaws.com"},
    "Action": ["s3:PutObject", "s3:GetBucketPolicy"],
    "Resource": [
      "arn:aws:s3:::YOUR-CUR-BUCKET",
      "arn:aws:s3:::YOUR-CUR-BUCKET/*"
    ]
  }]
}'
```

### Step 2: Create CUR 2.0 Data Export

```bash
aws bcm-data-exports create-export --export '{
  "Name": "daily-cur-export",
  "DataQuery": {
    "QueryStatement": "SELECT * FROM COST_AND_USAGE_REPORT",
    "TableConfigurations": {
      "COST_AND_USAGE_REPORT": {
        "TIME_GRANULARITY": "DAILY",
        "INCLUDE_RESOURCES": "TRUE",
        "INCLUDE_SPLIT_COST_ALLOCATION_DATA": "TRUE"
      }
    }
  },
  "DestinationConfigurations": {
    "S3Destination": {
      "S3Bucket": "YOUR-CUR-BUCKET",
      "S3Prefix": "cur-data",
      "S3Region": "us-east-1",
      "S3OutputConfigurations": {
        "OutputType": "CUSTOM",
        "Format": "PARQUET",
        "Compression": "PARQUET",
        "Overwrite": "OVERWRITE_REPORT"
      }
    }
  },
  "RefreshCadence": {"Frequency": "SYNCHRONOUS"}
}'
```

### Step 3: Set Up Athena Table

```bash
# Create Glue database
aws glue create-database --database-input '{"Name": "cur_database"}'

# Create Glue crawler
aws glue create-crawler \
  --name cur-crawler \
  --role arn:aws:iam::ACCOUNT_ID:role/GlueCrawlerRole \
  --database-name cur_database \
  --targets '{"S3Targets": [{"Path": "s3://YOUR-CUR-BUCKET/cur-data/"}]}'

# Run crawler
aws glue start-crawler --name cur-crawler
```

CUR data typically takes 24 hours to appear after first setup. Parquet format is recommended (smaller, faster queries, lower Athena costs).

### Quick Alternative: AWS Cloud Intelligence Dashboards

```bash
pip install cid-cmd
cid-cmd deploy --dashboard-id cudos-v5 --athena-database cur_database
```

This deploys pre-built QuickSight dashboards (CUDOS, Cost Intelligence Dashboard, KPI Dashboard) on top of your CUR data.

---

## Credits

Built by [Saurav Sharma](https://github.com/ravsau) — 13x AWS Certified, ex-Amazon (6 years as Senior TAM + SDE), [CloudYeti](https://cloudyeti.io).

Want a hands-on cost analysis? **[Book a consultation](https://cloudyeti.io/meet)**
