# IAM Permissions for the Connect Admin Console (per feature)

When using **custom** IAM policies to manage who can configure the Connect admin website, users need the `connect:` actions below **plus companion-service permissions** per feature. `connect:*` grants all Connect permissions. Some pages (Tasks, Customer Profiles, Email, Cases, etc.) require extra service permissions in inline policies. (Source: "Permissions required to use custom IAM policies" — a single page of in-page anchors, no sub-pages.)

> Managed policies: **`AmazonConnect_FullAccess`** must be attached **with** a companion policy allowing `iam:PutRolePolicy` on `arn:aws:iam::*:role/aws-service-role/connect.amazonaws.com/AWSServiceRoleForAmazonConnect*` (required to create an instance). **`AmazonConnectReadOnlyAccess`** can be attached alone. To perform any **Edit** action on a detail page, users also need the corresponding **List** + **Describe**.

## Companion-service requirements by console page

| Console page / action | `connect:` actions (key) | Companion services |
|---|---|---|
| **Home — list/describe instance** | `ListInstances`, `DescribeInstance`, `List*StorageConfigs/ApprovedOrigins/SecurityKeys/LambdaFunctions/LexBots`, `DescribeInstanceAttributes` | `ds:DescribeDirectories` |
| **Home — create instance** | `CreateInstance`, `AssociateInstanceStorageConfig`, `UpdateInstanceAttribute`, `AssociateCustomerProfilesDomain` | `ds:*` (CheckAlias/CreateAlias/Authorize/CreateIdentityPoolDirectory…), `iam:CreateServiceLinkedRole`, `iam:PutRolePolicy`, `kms:CreateGrant/DescribeKey/ListAliases/RetireGrant`, `logs:CreateLogGroup`, `s3:CreateBucket/GetBucketLocation/ListAllMyBuckets`, `servicequotas:GetServiceQuota`, `profile:CreateDomain/GetDomain/Put Integration/List*` |
| **Home — delete instance** | `DeleteInstance`, `DescribeInstance` | `ds:DescribeDirectories/DeleteDirectory/UnauthorizeApplication` |
| **Overview — create SLR** | `DescribeInstance`, `UpdateInstanceAttribute`, `ListIntegrationAssociations` | `iam:CreateServiceLinkedRole`, `iam:PutRolePolicy`, `ds:DescribeDirectories`, `profile:ListAccountIntegrations` |
| **Telephony** | `DescribeInstance`, `UpdateInstanceAttribute` | (campaigns toggle adds `connect-campaigns:*`, `iam:*`, `events:*`, `kms:*`, `ds:*`) |
| **Data storage** (call/screen recording, chat transcripts, attachments, live media, exported reports) | View: `List/DescribeInstanceStorageConfig`. Edit: `Associate/Update/DisassociateInstanceStorageConfig` | `s3:ListAllMyBuckets/GetBucketLocation/GetBucketAcl/CreateBucket`, `kms:CreateGrant/DescribeKey/ListAliases/RetireGrant`, `iam:PutRolePolicy` |
| **Data streaming** (contact records, agent events) | `Associate/Update/DisassociateInstanceStorageConfig` | `kinesis:ListStreams/DescribeStream`; contact records also `firehose:ListDeliveryStreams/DescribeDeliveryStream`; `iam:PutRolePolicy` |
| **Flows** (security keys, Lex, Lambda, flow logs, Polly) | `Associate/DisassociateSecurityKey`, `Associate/DisassociateBot`/`LexBot`, `Associate/DisassociateLambdaFunction`, `ListSecurityKeys/Bots/LambdaFunctions` | `lex:*` (GetBot/CreateResourcePolicy/ListBotAliases…), `lambda:ListFunctions/AddPermission/RemovePermission`, `logs:CreateLogGroup`, `iam:PutRolePolicy` |
| **Contact Lens connectors** / **Voice transfer integrations** | `ListIntegrationAssociations`, `Create/DeleteIntegrationAssociation` | `chime:*` (Voice Connector CRUD, `CreateConnectAnalyticsConnector`, `PutVoiceConnectorExternalSystemsConfiguration`, `AssociateVoiceConnectorConnect`…), `servicequotas:GetServiceQuota` |
| **Application integration** (approved origins) | `Associate/DisassociateApprovedOrigin`, `ListApprovedOrigins` | — |
| **Customer Profiles** | `DescribeInstance`, `ListInstances` | `profile:*` (Create/Get/List/Put Domain/Integration/ObjectType/CalculatedAttribute/EventStream/SegmentDefinition…), `app-integrations:*`, `appflow:*`, `events:*`, `kms:*`, `s3:*`, `kinesis:*`, `iam:Create*/PutRolePolicy`, `cloudwatch:GetMetricData`, `sqs:ListQueues`, `ds:DescribeDirectories` |
| **Tasks** | `ListIntegrationAssociations`, `Delete IntegrationAssociation/UseCase`, `ListUseCases` | `app-integrations:*EventIntegration*`, `appflow:*`, `events:*`, `kms:*` |
| **Email** (native channel) | — | **`ses:*`** (Get/Describe/Create/Update/Delete ReceiptRule(Set), EmailIdentity, ConfigurationSet…), `iam:CreateServiceLinkedRole/PassRole/CreateRole/CreatePolicy` |
| **Cases** | `ListInstances/IntegrationAssociations`, `CreateIntegrationAssociation`, `DescribeInstance` | `cases:GetDomain/CreateDomain`, `ds:DescribeDirectories`, `iam:PutRolePolicy` |
| **Customer authentication** | `Create/Delete/ListIntegrationAssociations` | `cognito-idp:ListUserPools/DescribeUserPool/ListUserPoolClients/TagResource/CreateUserPool` |
| **Outbound campaigns** | `Create/ListIntegrationAssociations`, `UpdateInstanceAttribute`, `AssociateCustomerProfilesDomain`, `ListPhoneNumbersV2`, `SearchEmailAddresses` | `connect-campaigns:*` (onboarding + config), `iam:*`, `events:*`, `kms:*`, `profile:*`, `wisdom:CreateKnowledgeBase/ListKnowledgeBases`, `ds:DescribeDirectories` |
| **Connect AI agents (Wisdom)** | `Create/Delete/ListIntegrationAssociations`, `DescribeInstance` | `wisdom:*` (Assistant/KnowledgeBase CRUD), `app-integrations:*DataIntegration*`, `appflow:*`, `kms:*`, `iam:Put/DeleteRolePolicy`, `secretsmanager:CreateSecret/PutResourcePolicy` |
| **Voice ID** | `Create/Delete/ListIntegrationAssociations` | `voiceid:DescribeDomain/ListDomains/CreateDomain/UpdateDomain/Register+DescribeComplianceConsent`, `events:Put/Delete Rule/Targets`, `iam:PutRolePolicy` |
| **Forecasting, capacity planning & scheduling** | `Describe/Start/StopForecastingPlanningSchedulingIntegration`, `UpdateInstanceAttribute` | — |
| **Federations** | SAML: `connect:GetFederationToken`. Emergency: `connect:AdminGetEmergencyAccessToken` | — |

**`iam:PutRolePolicy`** (attaching the service-linked-role inline policy) recurs in nearly every Edit/Create flow. Note the **Email** page is the only place requiring `ses:*` — it governs the **native Connect email channel**, not outbound SMTP.

> This page covers **instance/integration administration**. Per-resource access to Users, Routing profiles, Queues, Hours of operation, Quick connects, Phone numbers, Saved reports, Metrics/dashboards, and Recording playback is governed by `connect:` resource-level actions + Connect **security profiles** (see [security.md](security.md)).
