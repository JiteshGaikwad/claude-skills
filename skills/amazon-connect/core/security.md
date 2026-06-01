# Amazon Connect — Security

## Shared Responsibility Model
- **AWS**: Security OF the cloud (infrastructure, availability)
- **You**: Security IN the cloud (config, access, data classification)

## Identity & Access Management (IAM)

### IAM Resource-Level Permissions

Connect ARN format:
```
arn:aws:connect:{region}:{account}:instance/{instance-id}/{resource-type}/{resource-id}
```

Resource types addressable in Connect ARNs: **Instance, Contact, User, Routing profile, Security profile, Hierarchy group, Queue, File, Flow, Hours of operation, Phone number, Task templates, Customer profile domain, Customer profile object type, Outbound campaigns.** Notes:
- The **user** segment in an ARN is `agent`, not `user`: `arn:aws:connect:{region}:{account}:instance/{instance-id}/agent/{user-id}`.
- **Contact** ARNs use a wildcard for the dynamic contact ID: `.../instance/{instance-id}/contact/*`.
- **Phone number** has two ARN formats — V1 `.../instance/{instance-id}/phone-number/{id}` and V2 (recommended) `arn:aws:connect:{region}:{account}:phone-number/{id}`. For queue APIs in a replica Region with a number claimed to a traffic-distribution group, list the number in **both** V1 and V2 formats.
- Related services use their own ARN prefixes: Customer Profiles `arn:aws:profile:...:domains/{name}`, Voice ID `arn:aws:voiceid:...:domain/{name}`, AI agents `arn:aws:wisdom:...:assistant/{id}`, Outbound campaigns `arn:aws:connect-campaigns:...:campaign/{id}`.

#### AWS managed policies

| Policy | Purpose |
|---|---|
| `AmazonConnect_FullAccess` | Full Connect access. **Must be attached together with a custom policy** granting `iam:PutRolePolicy` on `arn:aws:iam::*:role/aws-service-role/connect.amazonaws.com/AWSServiceRoleForAmazonConnect*` (required to create an instance). |
| `AmazonConnectReadOnlyAccess` | Read-only; attach alone. (Its `connect:GetFederationTokens` was renamed to `connect:AdminGetEmergencyAccessToken` on 2024-06-15, backward compatible.) |
| `AmazonConnectVoiceIDFullAccess` | Full Voice ID access; also needs a companion custom policy and a KMS key policy granting `kms:Decrypt`/`CreateGrant`/`DescribeKey`. |
| `AmazonConnectServiceLinkedRolePolicy` | Permissions policy on the Connect service-linked role (see below). |

Connect supports **identity-based policies and service-linked roles only — no resource-based policies or ACLs**; cross-account access is via assumed roles.

#### Example: Read-Only IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "connect:Describe*",
        "connect:List*",
        "connect:Get*",
        "connect:Search*"
      ],
      "Resource": "arn:aws:connect:us-east-1:111122223333:instance/aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "connect:DescribeInstance",
        "connect:ListInstances"
      ],
      "Resource": "*"
    }
  ]
}
```

#### Example: Flow Management IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "connect:CreateContactFlow",
        "connect:UpdateContactFlowContent",
        "connect:UpdateContactFlowName",
        "connect:UpdateContactFlowMetadata",
        "connect:DescribeContactFlow",
        "connect:ListContactFlows",
        "connect:DeleteContactFlow"
      ],
      "Resource": "arn:aws:connect:us-east-1:111122223333:instance/aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee/contact-flow/*"
    }
  ]
}
```

#### IAM Condition Keys

| Condition Key | Description |
|---|---|
| `connect:InstanceId` | Restrict actions to a specific instance |
| `connect:StorageResourceType` | Restrict storage-config associations (e.g. `CONTACT_TRACE_RECORDS`) |
| `connect:SearchContactsByContactAnalysis` | Restrict Contact Lens transcript search (e.g. value `Transcript`) |
| `connect:FlowType` / `connect:Channel` / `connect:ContactInitiationMethod` / `connect:UserArn` | Restrict by flow type, channel, initiation method, or user |
| `aws:ResourceTag/{TagKey}` | Restrict based on resource tags |
| `aws:RequestTag/{TagKey}` | Require specific tags on creation |
| `aws:TagKeys` | Control which tag keys can be used |

### Service-Linked Role

- **Role name**: `AWSServiceRoleForAmazonConnect_{unique-id}` (trust principal `connect.amazonaws.com`); policy `AmazonConnectServiceLinkedRolePolicy`.
- Auto-created when you create an instance; cannot be edited or renamed (description editable); auto-deleted when you delete the instance.
- Grants Connect access to: **S3** (recordings bucket: Get/Delete/GetBucketLocation/GetBucketAcl; exported-reports bucket: Put/PutAcl/GetObjectAcl), **CloudWatch Logs** (flow logging) + **CloudWatch metrics** (`PutMetricData`), **Lex** (ListBots/ListBotAliases), **Customer Profiles** (`profile:*` on `amazon-connect-`-prefixed domains with a deny-list on Create/Update/Delete domain etc.), **AI agents/Wisdom** (`wisdom:*` on `AmazonConnectEnabled:True` resources), **Polly**, **Cognito**, **Chime SDK Voice Connector**, **Pinpoint SMS/push**, **SES** (email channel), **WhatsApp via End User Messaging Social**; and via inline policies as features enable: **Kinesis/Firehose**, **EventBridge**, **Voice ID**, outbound-campaigns actions. (It does **not** grant SNS publish.)
- Other SLRs: `AWSServiceRoleForConnectCampaigns` (outbound campaigns), `AWSServiceRoleForProfile` (Customer Profiles, one per domain), `AWSServiceRoleForAmazonConnectSynchronization` (Global Resiliency, principal `synchronization.connect.amazonaws.com`). See [service-linked-roles.md](service-linked-roles.md).

### Tag-Based Access Control (TBAC)

**Taggable resources** (26): agent (user), agent group, agent group level, agent state, case, contact evaluations, email addresses, evaluation forms, flow, flow module, hours of operation, instance, integration association, outbound campaign, phone number, prompts, queue agent, queues, quick connects, routing profile, security profile, task template, traffic distribution group, transfer destination, use case, vocabulary. **Contact is NOT taggable.** Some are CLI/SDK-only (no admin-website tagging): agent group, agent state, integration association, outbound campaign, phone number, quick connects, traffic distribution group, transfer destination, use case, vocabulary.

**Tag limits:** key ≤ 128 chars, value ≤ 256 chars (both case-sensitive), up to **50 tags per resource**, keys unique per resource; tag keys can't begin with `aws:` (reserved, don't count toward the limit). Value can be empty but not null. Customer Profiles, Voice ID, and AppIntegrations have **separate `TagResource` APIs**.

**Tag retention:** Connect **keeps tag metadata for deleted resources** as long as the instance is active (so removing tags on deletion can't accidentally grant access to historical metrics); tags are removed only when the whole instance is deleted.

**Admin-console TBAC (access control tags on security profiles):** define `Key:value` access-control tags on a security profile (vs. IAM-policy TBAC for API/SDK). Limits: up to **4 access-control tags per profile** (AND logic — more tags = more restrictive); a user can have at most **3 security profiles containing access-control tags** (OR logic across profiles); a profile **without** TBAC overrides one **with** TBAC on overlapping permissions; **service-linked roles are required**; grant at least **view** on TBAC'd resources; users can't see historical change logs for restricted resources.

#### Example: Restrict Supervisors to Their Team's Queues

Tag queues with `Team=TeamA` or `Team=TeamB`, then apply this IAM policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "connect:DescribeQueue",
        "connect:UpdateQueueName",
        "connect:UpdateQueueStatus",
        "connect:GetCurrentMetricData",
        "connect:GetMetricDataV2"
      ],
      "Resource": "arn:aws:connect:us-east-1:111122223333:instance/*/queue/*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Team": "TeamA"
        }
      }
    }
  ]
}
```

Use case examples:
- Restrict supervisors to only manage queues tagged with their team
- Allow flow developers to edit only flows tagged with their department
- Limit metric access to users tagged with a specific business unit

## Security Profiles

### Predefined Roles

| Profile | Access |
|---|---|
| **Admin** | Full access to all permissions across every category. Can manage users, security profiles, instance settings, flows, queues, and all configuration. |
| **Agent** | Contact Control Panel (CCP) access only. Can handle inbound/outbound contacts, transfer calls, and view the contact directory. No access to metrics, flows, or admin settings. |
| **CallCenterManager** | Metrics and reporting (real-time and historical), user management, queue management, recording playback, agent scheduling, and quality management. Cannot modify flows or instance settings. |
| **QualityAnalyst** | Recording playback and review, Contact Lens analytics, evaluation forms, quality metrics, and contact search. No user management or flow editing. |

### Full Permission Taxonomy

Security profile permissions are organized into these categories:

- **Routing** — Queues (create/edit/enable-disable), routing profiles, quick connects, hours of operation, prompts, task templates
- **Numbers** — Claim/release phone numbers, manage number associations
- **Flows** — Create/edit/publish contact flows and modules
- **Contact Control Panel (CCP)** — Make/receive calls, transfer, hold, conference, create tasks, restrict contact attributes visibility
- **Users** — Create/edit/view users, manage hierarchy groups, agent statuses, proficiencies
- **Metrics** — Access real-time metrics, historical metrics, agent activity audit, login/logout reports, saved reports
- **Recording** — Access recorded conversations, monitor live calls, barge into calls, manager listen-in, screen recording playback
- **Quality** — Evaluation forms, calibrations, Contact Lens analytics, post-contact summaries
- **Rules** — Create/edit/delete automation rules, manage rule templates
- **Contact Search** — Search contacts by attributes, view contact details, view contact flow logs, access contact records
- **Cases** — Create/edit/view cases, manage case templates, case fields, case event configurations
- **Customer Profiles** — View/edit/create customer profiles, manage profile object types, calculated attributes
- **Campaigns** — Outbound campaigns management, campaign scheduling, contact list management
- **Dashboard** — View/customize supervisor dashboards, create saved views
- **Views** — Manage step-by-step guides and agent workspace views
- **Data Tables** — Create/read/update/delete data table records
- **Workspace** — Configure agent workspace layout, app integrations
- **AI/ML** — Forecast management, capacity planning, agent scheduling, Contact Lens configuration

### Custom Security Profile Guidance

- Start from the principle of least privilege — grant only the permissions an agent or role truly needs
- Use `CreateSecurityProfile` API or the admin console under Users > Security Profiles
- `UpdateSecurityProfile` to modify, `ListSecurityProfilePermissions` to audit
- Assign multiple security profiles to a single user when a role spans categories
- Audit profiles periodically — remove permissions that are no longer needed
- Name profiles descriptively (e.g., `Tier2Support-WithRecording`, `BillingTeamLead`)

## Data Protection

### What data Connect stores

Data is segregated by **AWS account ID + instance ID**. Categories include: resource/config (queues, flows, users, routing profiles), contact metadata (connect/handle time, ANI/DNIS, contact attributes), agent performance data, call audio + recordings (when enabled), chat transcripts (when enabled), screen recordings (when enabled), attachments, knowledge documents, Voice ID voiceprints + speaker/fraudster audio, and forecasts/capacity plans/schedules. **Max 2 recordings per contact** (one IVR/automated, one agent). Recordings are held intermediately within Connect, then delivered to your S3 bucket. The customer **origination phone number is cryptographically hashed** (instance-specific key) for contact search.

**Retention notes:** Contact Lens real-time data persists **up to 24h** after the contact ends; Voice ID **original enrollment/evaluation audio is deleted after 24h** (voiceprints kept until delete/opt-out); AppIntegrations synced external-app data is stored **~1 month**.

### Encryption at rest

The baseline is **AWS KMS encryption with AWS-owned keys**; data-encryption keys aren't shared across instances. Recordings/transcripts in **your** S3 bucket are secured with the **KMS key configured when the instance was created** (envelope encryption / BYOK supported on the storage config). Per-feature customer-managed-key (CMK) support:

| Feature | CMK support |
|---|---|
| Recordings/transcripts (your S3 bucket) | KMS key set at instance creation (BYOK supported) |
| **Voice ID** | **CMK mandatory** at domain creation; changing the key triggers async re-encrypt |
| **Customer Profiles** | **CMK required before enabling Data Store**; per-object-type keys override domain key; **keys become immutable after Data Vault is enabled** |
| **Connect AI agents (Wisdom)** | Optional CMK for knowledge content; search indices use AWS-owned keys |
| **Cases** | AWS-owned keys only |
| **Outbound campaigns** | CMK or AWS-owned key; instance-specific keys |
| **AppIntegrations** | No BYOK for config data; CMK required when syncing external-app data |
| **Forecasts / capacity plans / schedules** | AWS-owned keys |

**KMS grant flow:** when you associate a key to a storage location (`AssociateInstanceStorageConfig`), Connect auto-creates a **grant** with the instance **service-linked role as grantee** — using the **caller's** permissions, so the caller needs **`kms:CreateGrant`** or the association fails. `DisassociateInstanceStorageConfig` removes the grant.

### Encryption in transit

- Browser ↔ Connect and all API calls use **TLS** — Connect **requires TLS 1.2 and recommends TLS 1.3**, with **perfect-forward-secrecy** cipher suites (DHE or ECDHE).
- **PSTN calls** routed over the public internet: signaling encrypted with TLS, audio with **SRTP**. **Softphone (agent browser)** media uses **DTLS-SRTP** over an encrypted WebSocket/TLS.
- Integrations (Lambda, Kinesis, Polly) and external-app event data are always TLS in transit.

### Customer Input Encryption
- Public signing key configured in contact flows
- Encrypts sensitive DTMF input (credit card numbers, SSNs)

### PII Redaction
- Contact Lens redacts PII from transcripts and audio
- Configurable categories: name, address, credit card, SSN, etc.

## Authentication Profiles

> **Preview feature**, configurable **only via the AWS SDK/CLI** (`list/describe/update-authentication-profile`) — no console UI. Each instance has a **default authentication profile** applied to all users.

### IP Address Restrictions

- Two lists: **`AllowedIps`** and **`BlockedIps`** — both support **IPv4 and IPv6**, individual IPs and CIDR ranges.
- **BlockedIps always takes precedence.** If `AllowedIps` is set, only those IPs are allowed (minus any blocked); if only `BlockedIps` is set, all IPs are allowed except those blocked; if both empty, no restriction.
- Applies to the authenticated browser surfaces (CCP, Agent Workspace, Admin Website) — **not** direct service-API access (that's IAM). **IP restrictions do NOT apply to the emergency admin login** — restrict that via an IAM `SourceIp` condition on `connect:AdminGetEmergencyAccessToken`.
- On block: the session is invalidated and a logout event is published in the Login/Logout report. An agent on an active call keeps the call but loses controls until re-login.

### Session Timeouts

- **Maximum session duration** — prose says it defaults to **12 hours and isn't configurable**; the API field `MaxSessionDuration` has a valid range of **360–720 minutes**.
- **Session inactivity duration** — the actually-configurable timeout: `SessionInactivityDuration` **15–720 minutes**, **off by default**, enabled via `--session-inactivity-handling-enabled`. A warning pop-up precedes logout. "Active" = mouse/keyboard on CCP/Agent Workspace/Admin Website, or an active voice contact.
- `PeriodicSessionDuration` (10–60 min) is **deprecated**. There is no `PeriodIntervalInMinutes` parameter.
- Not supported in VDI split-CCP; Streams/SDK integrations must implement activity handling before enabling.

### SAML 2.0 Federation
- Integrates with any SAML 2.0 IdP (Okta, Azure AD, OneLogin, PingFederate, etc.); enables SSO and MFA through the IdP. See [identity-management.md](identity-management.md) for setup detail.
- For instances opting into the newer `instancename.my.connect.aws` domain, SAML relay-state URLs must include `new_domain=true`.

## Compliance
- Connect is in scope for multiple AWS compliance programs and is **HIPAA-eligible** and usable in compliance with **GDPR**. The Connect compliance-validation page does not hardcode a program list — for the authoritative, current scope (PCI DSS, SOC, ISO, FedRAMP, etc.) see **AWS Services in Scope by Compliance Program**, and download audit reports via **AWS Artifact**.
- Your compliance responsibility is determined by the sensitivity of your data and applicable laws/regulations (shared-responsibility model).

## Infrastructure Security
- Connect is a **managed service** — you access it via published AWS API calls, callable from **any network location**. There is no customer-VPC isolation; restrict network access via authentication-profile IP allow-lists and IAM source-IP conditions.
- **TLS 1.2+ required** with PFS cipher suites; the `instancename.my.connect.aws` access model (default for instances created after March 2021) supports TLS 1.2+ only.
- **AWS PrivateLink (interface VPC endpoints)** are available for several Connect APIs — service names include `com.amazonaws.{region}.connect`, `.connect-fips` (FIPS endpoint), `.profile`, `.voiceid`, `.connect-campaigns`, `.wisdom`, `.app-integrations`, `.cases`.

## Cross-Service Confused Deputy Prevention

Use the `aws:SourceArn` and `aws:SourceAccount` global condition keys in **resource policies** (e.g. a KMS key or SNS topic Connect uses) to limit Connect to acting only on behalf of your account/instance. If both keys are used in one statement, the account in `aws:SourceArn` must match `aws:SourceAccount`. Prefer the exact ARN; use wildcards only for unknown portions.

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "connect.amazonaws.com" },
  "Action": "sns:Publish",
  "Resource": "arn:aws:sns:us-east-1:111122223333:TopicName",
  "Condition": {
    "StringEquals": { "aws:SourceAccount": "111122223333" },
    "ArnEquals": { "aws:SourceArn": "arn:aws:connect:us-east-1:111122223333:instance/InstanceId" }
  }
}
```

## Logging & Monitoring

- **CloudTrail** logs **all public Connect API calls** as **management events** (`eventSource: connect.amazonaws.com`; Voice ID uses `voiceid.amazonaws.com` and can share the trail, redacting PII). Service-linked roles are required for CloudTrail support. The docs do not state Connect supports CloudTrail *data* events.
- **CloudWatch metrics** (namespace `AWS/Connect`, sent every 1 min — Concurrent Emails/Tasks every 5 min, `ToInstancePacketLossRate` every 10 s; retained ~2 weeks): e.g. `ConcurrentCalls(Percentage)`, `ConcurrentActiveChats(Percentage)`, `MissedCalls`, `ThrottledCalls`, `CallRecordingUploadError`, `ContactFlowErrors`, `ContactFlowFatalErrors`, `QueueSize`, `LongestQueueWaitTime`, `CallBackNotDialableNumber`, `MisconfiguredPhoneNumbers`, `TasksExpired`. Dimensions: Instance, Queue, Flow, Contact, Connection type. Other namespaces: `VoiceID`, `AWS/AppIntegrations`, `AWS/CustomerProfiles`. (For CTR-derived business KPIs, see the `connect-metrics` skill.)

## Resilience

- Each Connect instance spans a **minimum of 3 Availability Zones** in **active-active-active** configuration; if an AZ fails, that node is removed from rotation with no production impact (enabling zero-downtime maintenance).
- Telephony is integrated with **multiple carriers** over redundant paths to ≥3 AZs per Region; US toll-free is routed across carriers via RespOrg; outbound is load-balanced across providers; the agent browser selects from ≥2 servers across AZs.
- The above is **single-Region** resiliency. **Amazon Connect Global Resiliency** (multi-Region replicated instances + traffic distribution groups) is a separate capability — the `AWSServiceRoleForAmazonConnectSynchronization` SLR (created by `ReplicateInstance`) backs it.

## Security Best Practices Checklist

1. **Use restrictive security profiles** — Never assign Admin to agents. Create custom profiles scoped to the exact permissions each role requires.

2. **Enable MFA** — Configure MFA through your SAML IdP. If using Connect-managed auth, enable RADIUS MFA integration.

3. **Emergency access URL** — The emergency login URL bypasses federation. Use it only when your IdP is down. Restrict knowledge of this URL to a small number of administrators.

4. **Prevent instance deletion with SCPs** — Apply a Service Control Policy at the AWS Organizations level:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyConnectInstanceDeletion",
      "Effect": "Deny",
      "Action": ["connect:DeleteInstance"],
      "Resource": ["arn:aws:connect:*:*:instance/*"]
    },
    {
      "Sid": "DenyConnectRoleDeletion",
      "Effect": "Deny",
      "Action": ["iam:DeleteRole"],
      "Resource": ["arn:aws:iam::*:role/ConnectUserRole"]
    }
  ]
}
```

5. **Enable CloudWatch logging for flows** — In instance settings, enable contact flow logs. Set CloudWatch alarms for flow errors and Lambda invocation failures.

6. **Archive logs with lifecycle policies** — Export CloudWatch logs to S3 with lifecycle rules (e.g., transition to Glacier after 90 days, expire after 7 years for compliance).

7. **Restrict IP ranges** — Use authentication profiles to limit CCP and console access to corporate CIDR ranges.

8. **Protect chat from XSS** — If building custom chat UIs:
   - Always encode output before rendering chat messages
   - Set a Content Security Policy (CSP) header
   - Never use `innerHTML` to render agent or customer messages
   - Sanitize any user-provided content before display

9. **Protect WebRTC participant tokens** — For custom CCP or video implementations:
   - Authenticate the user before issuing a participant token
   - Always use HTTPS for token delivery
   - Minimize token exposure time — generate tokens on demand, not ahead of time

10. **Monitor with CloudTrail** — Enable CloudTrail for the Connect instance. Key events to alert on:
    - `DeleteInstance`, `DeleteUser`, `DeleteContactFlow`
    - `CreateUser` with Admin security profile
    - `UpdateSecurityProfile` changes
    - `AssociateBot`, `AssociateLambdaFunction` (new integrations)

11. **Rotate credentials regularly** — Rotate KMS keys, review IAM policies quarterly, and audit security profile assignments monthly.

12. **Use resource tagging** — Tag all Connect resources for cost allocation, access control, and audit trail purposes.
