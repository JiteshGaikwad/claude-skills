# Data Streaming

Amazon Connect can stream contact records, agent events, and Customer Profiles data to Amazon Kinesis for real-time processing, analytics, and long-term retention beyond the 24-month in-instance limit.

## Enabling Data Streaming

Data streaming is configured from the **Data streaming** page in the Amazon Connect console (Amazon Connect > Instance > Data streaming).

Three data categories can be streamed independently:

| Data Category | Kinesis Data Firehose | Kinesis Data Stream |
|---|---|---|
| Contact records (CTRs) | Yes | Yes |
| Agent events | No | Yes (stream only) |
| Customer Profiles | No | Yes (stream only) |

- **Contact records** can go to either Firehose (for direct delivery to S3, Redshift, OpenSearch, etc.) or a Kinesis Data Stream (for custom consumers).
- **Agent events** and **Customer Profiles** support Kinesis Data Streams only -- Firehose is not an option for these categories.

## Contact Records Streaming

Contact records contain the metadata and details of every contact handled by your instance. By default, CTRs are available in the Connect instance for **24 months**. Streaming to Kinesis enables longer retention and integration with external analytics systems.

**What a contact record includes:**
- Contact identifiers (ContactId, InitialContactId, PreviousContactId)
- Agent information (ARN, timestamps, hierarchy groups)
- Queue information (ARN, name, enqueue/dequeue timestamps)
- Recording location (S3 bucket, key, type, status)
- Attributes (contact attributes set during the flow)
- Disconnect details (reason, timestamp, who disconnected)
- Channel and initiation method

**Delivery behavior:**
- CTRs are emitted after the contact is fully completed (after ACW)
- Records are delivered as JSON to the configured Kinesis stream or Firehose
- Each record is a self-contained JSON document

## Agent Event Streaming

Agent events provide near-real-time visibility into agent activity. See [agent-event-streams.md](./agent-event-streams.md) for the full data model and event types.

Configuration: select a Kinesis Data Stream on the Data streaming page. Agent events cannot be sent to Firehose.

## Customer Profiles Streaming

When Customer Profiles data streaming is enabled, profile changes (creates, updates, merges) are published to the configured Kinesis Data Stream. This is useful for keeping external systems synchronized with Connect's unified customer view.

## Server-Side Encryption with Customer-Managed KMS Keys

By default, Kinesis streams use AWS-managed encryption. To use a **customer-managed KMS key** (CMK), you must grant the Connect service-linked role permission to generate data keys.

**Step 1 — Get the service-linked role ARN:**

The Connect service-linked role follows this pattern:

```
arn:aws:iam::<ACCOUNT_ID>:role/aws-service-role/connect.amazonaws.com/AWSServiceRoleForAmazonConnect_<SUFFIX>
```

You can find the exact ARN in the IAM console under Roles, or via the CLI:

```bash
aws iam list-roles --query "Roles[?contains(RoleName, 'AWSServiceRoleForAmazonConnect')].[Arn]" --output text
```

**Step 2 — Construct the KMS key policy statement:**

Add a policy statement to your KMS key that grants the service-linked role the `kms:GenerateDataKey` permission:

```json
{
  "Sid": "AllowConnectToGenerateDataKey",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::<ACCOUNT_ID>:role/aws-service-role/connect.amazonaws.com/AWSServiceRoleForAmazonConnect_<SUFFIX>"
  },
  "Action": "kms:GenerateDataKey",
  "Resource": "*"
}
```

**Step 3 — Add the statement to the KMS key policy:**

```bash
# Get current policy
aws kms get-key-policy --key-id <KEY_ID> --policy-name default --output text > policy.json

# Edit policy.json to add the statement above

# Put updated policy
aws kms put-key-policy --key-id <KEY_ID> --policy-name default --policy file://policy.json
```

Without this permission, Connect cannot encrypt records before writing them to the Kinesis stream, and streaming will fail silently.

## Retention and Archival

| Storage Location | Retention |
|---|---|
| Connect instance (native) | 24 months |
| Kinesis Data Stream | Configurable (1-365 days, or unlimited with enhanced fan-out) |
| Kinesis Firehose to S3 | Indefinite (managed by S3 lifecycle rules) |

For long-term retention, the recommended pattern is:
1. Stream CTRs to **Kinesis Data Firehose**
2. Firehose delivers to **S3** with partitioning by date
3. Apply **S3 lifecycle rules** for tiered storage (Standard -> IA -> Glacier)
4. Query archived records with **Athena** using a Glue catalog table

## Key APIs

- Data streaming is configured via the Connect console, not via API
- Kinesis stream/Firehose resources are managed via their respective AWS SDKs
- `KinesisClient` / `FirehoseClient` from `@aws-sdk/client-kinesis` / `@aws-sdk/client-firehose`

## Key Considerations

- **Ordering:** Records within a single contact are ordered, but records across contacts may arrive out of order
- **Duplicates:** At-least-once delivery -- consumers should be idempotent
- **Latency:** CTRs stream after contact completion; agent events stream in near-real-time
- **Cross-region:** The Kinesis stream must be in the same region as the Connect instance
- **Permissions:** The Connect service-linked role needs `kinesis:PutRecord` and `kinesis:PutRecords` on the target stream (granted automatically when configured via console)
