# Analytics Data Lake

The Amazon Connect analytics data lake provides a centralized query location for all Connect operational data, enabling analysis via Amazon Athena and visualization via Amazon QuickSight.

---

## Overview

The data lake is a zero-ETL solution. Amazon Connect automatically exports data to a managed data store, and you query it using standard SQL through Athena. No custom pipelines, Glue jobs, or data transformation code is required.

---

## Data Types

The data lake includes four categories of data:

### 1. Contact Records (CTRs)

Complete contact trace records including:
- Contact metadata (timestamps, channel, initiation method, disconnect reason)
- Queue and routing information
- Agent information
- Contact attributes
- Segment attributes
- Recording and analytics references

### 2. Contact Lens Analytics

Conversational analytics output:
- Transcripts (full and redacted)
- Sentiment scores (per-turn and overall)
- Categories matched
- Talk time metrics
- Response time metrics
- Key highlights (issue, outcome, action item)
- PII detection results

### 3. Performance Evaluations

Quality management data:
- Evaluation form responses
- Scores per section and question
- Automatic fail indicators
- Evaluator information
- Generative AI recommendations

### 4. Cases

Amazon Connect Cases data:
- Case metadata (ID, status, created/resolved timestamps)
- Case fields and values
- Associated contacts
- Case templates used

---

## Data Refresh

- Data is refreshed within **1 hour** of the contact or event completing.
- This is near real-time but not instant. For true real-time data, use Kinesis streaming or the agent event stream.
- Refresh is continuous; there is no manual trigger required.

---

## Athena Integration

### Querying

Once the data lake is enabled, tables are available in Athena under a database created for your Connect instance.

```sql
-- Example: Average handle time by queue for the last 7 days
SELECT
    queue_name,
    AVG(agent_interaction_duration + after_contact_work_duration) AS avg_handle_time,
    COUNT(*) AS contacts_handled
FROM contact_records
WHERE
    disconnect_timestamp >= CURRENT_DATE - INTERVAL '7' DAY
    AND agent_username IS NOT NULL
GROUP BY queue_name
ORDER BY avg_handle_time DESC;
```

### Table Structure

Tables follow the Connect data model. Key tables include:
- `contact_record` — One row per contact segment.
- `contact_lens_conversational_analytics` — Analytics per analyzed contact.
- `evaluation` — One row per completed evaluation.
- `cases` — One row per case.

### Partitioning

Data is partitioned by date for efficient querying. Always include date filters in your queries to minimize scan costs.

---

## QuickSight Integration

Amazon QuickSight can connect directly to the Athena data source to build dashboards:

1. Create a QuickSight data source pointing to the Athena database.
2. Build datasets from the data lake tables.
3. Create analyses and dashboards with filters, calculations, and visualizations.

QuickSight SPICE can be used to cache data for faster dashboard performance.

---

## Resource Link Tables

### What Are Resource Link Tables?

Resource link tables are cross-account Athena table references that allow you to access the data lake tables from another AWS account or from a different Athena workgroup.

### Associating Tables

Use the Amazon Connect console to associate data lake tables:

1. Navigate to **Analytics and optimization > Data lake**.
2. Select the data categories to include.
3. The system creates the necessary AWS Lake Formation permissions and Athena table references.

### Managing Access

- Access to resource link tables is managed through AWS Lake Formation permissions.
- Grant SELECT permission to specific IAM roles or users.
- Revoke access to remove visibility to the data.

---

## Data Retention

| Data Type | Retention | Notes |
|---|---|---|
| Contact records | 24 months | After 24 months, data is removed from the data lake. Stream to S3 via Kinesis for longer retention. |
| Contact Lens analytics | 24 months | Same retention as contact records. |
| Evaluations | 24 months | Evaluation data follows the same lifecycle. |
| Cases | 24 months | Case data follows the same lifecycle. |

For retention beyond 24 months, configure Kinesis Data Streams or Kinesis Data Firehose to export contact records and analytics to your own S3 bucket.

---

## Zero-ETL Architecture

The data lake operates on a zero-ETL principle:

- **No extraction code** — Connect automatically exports data.
- **No transformation logic** — Data is stored in a query-ready format.
- **No loading pipelines** — No Glue crawlers, Glue jobs, or Step Functions needed.
- **No infrastructure management** — AWS manages the underlying storage and catalog.

This eliminates the operational burden of maintaining data pipelines and reduces the time from data generation to query availability.

---

## Enabling the Data Lake

### Console Setup

1. Navigate to **Analytics and optimization > Data lake** in the Amazon Connect console.
2. Enable the data lake for your instance.
3. Select which data types to include (contact records, Contact Lens, evaluations, cases).
4. Configure the Athena workgroup and database name.

### Prerequisites

- The Connect instance must be in a supported region.
- IAM permissions for Lake Formation, Athena, and S3 are required.
- A Lake Formation admin must be configured in the account.

---

## APIs

The following APIs support data lake management:

| API | Description |
|---|---|
| `CreateAnalyticsDataAssociation` | Associate a data type with the data lake. |
| `ListAnalyticsDataAssociations` | List current data type associations. |
| `DeleteAnalyticsDataAssociation` | Remove a data type association. |
| `BatchGetAnalyticsDataAssociations` | Get multiple associations in a single call. |

---

## Cost Considerations

- **Data lake storage** — Included with Amazon Connect at no additional charge.
- **Athena queries** — Standard Athena pricing applies (per TB scanned). Use partitioning and columnar filtering to minimize costs.
- **QuickSight** — Standard QuickSight pricing for users and SPICE capacity.
- **Lake Formation** — No additional charge for Lake Formation itself.

---

## Limitations

- Data is read-only. You cannot write to or modify data lake tables.
- Query performance depends on the volume of data and query complexity. Use date partitioning.
- The data lake does not include real-time streaming data. For sub-minute latency, use Kinesis.
- Cross-region data lake access is not supported. The data lake exists in the same region as the Connect instance.
- Maximum concurrent Athena queries follow standard Athena service limits.
