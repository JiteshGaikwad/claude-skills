# Amazon Connect — Instance Management

## Instance Lifecycle
- **Create**: Via console, CLI (`CreateInstance`), or CloudFormation (`AWS::Connect::Instance`)
- **Identity**: SAML 2.0, Connect-managed, or linked to existing directory
- **Alias**: Appears in URL `https://{alias}.my.connect.aws/`
- **Delete**: `DeleteInstance` — permanent, removes all config

## Instance Attributes
Enable/disable features per instance via `UpdateInstanceAttribute`:
- `INBOUND_CALLS`, `OUTBOUND_CALLS`
- `CONTACT_LENS` — conversational analytics
- `AUTO_RESOLVE_BEST_VOICES` — neural TTS
- `CONTACTFLOW_LOGS` — flow logging to CloudWatch
- `EARLY_MEDIA` — audio before call connects
- `MULTI_PARTY_CONFERENCE` — 3+ party calls
- `HIGH_VOLUME_OUTBOUND` — outbound campaigns
- `ENHANCED_CONTACT_MONITORING` — barge/monitor

## Storage Configuration
Configure via `AssociateInstanceStorageConfig`:
- **Call recordings** → S3 bucket (with optional KMS encryption)
- **Chat transcripts** → S3 bucket
- **Screen recordings** → S3 bucket
- **Exported reports** → S3 bucket
- **Contact Lens** → S3 bucket for analyzed output
- **Agent events** → Kinesis Data Stream
- **Contact records** → Kinesis Data Stream or Firehose

## Service Quotas
- Default concurrent calls: varies by region
- Default concurrent chats: 2,500
- Default concurrent tasks: 2,500
- Flows per instance: 500
- Queues per instance: 1,000
- Routing profiles: 1,000
- Users: 10,000
- Request via Service Quotas console for increases

## Key APIs
- `CreateInstance`, `DescribeInstance`, `ListInstances`, `DeleteInstance`
- `UpdateInstanceAttribute`, `DescribeInstanceAttribute`
- `AssociateInstanceStorageConfig`, `ListInstanceStorageConfigs`
- `ReplicateInstance` (for Global Resiliency)

## Getting Started
1. Create instance in console (choose identity management)
2. Claim phone number (DID or toll-free)
3. Create queues and routing profiles
4. Create users and assign routing profiles
5. Create flows (or use defaults)
6. Associate phone number with flow
7. Test by calling the number
