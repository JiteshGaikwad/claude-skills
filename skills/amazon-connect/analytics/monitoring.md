# Monitoring

Amazon Connect monitoring spans live call/chat monitoring, CloudWatch metrics, CloudTrail logging, and EventBridge integration. AWS recommends starting with a **monitoring plan** — define your goals, what data to monitor, how often, which tools (CloudWatch + CloudTrail), who performs monitoring, and who is notified on failure.

---

## Live Monitoring

Managers can silently **monitor** live voice and chat conversations, and **barge** into them. Two modes exist:

- **Three-party (default):** up to 3 participants; **monitor only — you cannot barge.** Enabled by adding a **Set recording and analytics behavior** block to the flow.
- **Multi-party / Enhanced Monitoring:** up to 6 participants; **required for barge** (voice and chat). Enable on the console via **Enable Multi-Party Calls and Enhanced Monitoring for Voice** and **Enable Multi-Party Chats and Enhanced Monitoring for Chat** (or `UpdateInstanceAttribute` with `ENHANCED_CONTACT_MONITORING`). Requires latest CCP/Agent Workspace; StreamsJS users need ≥ 2.4.2; instances need a service-linked role. Switching modes adds events to the agent event stream — can break customizations built on the old stream.

### How a manager monitors / barges

1. Log in with **CallCenterManager** (+ **Agent** to access CCP), or a profile with the needed permissions.
2. **Open the CCP first** — you need it open to connect to the conversation.
3. **Analytics and optimization → Real-time metrics → Agents**. For voice, choose the **eye icon** next to the agent; for chat, choose the number of live chats, then the conversation.
4. Toggle between **Monitor** and **Barge** (barge requires Enhanced Monitoring). There is **no limit** to how many conversations you can barge in an instance.
5. To stop, choose **End call/End chat**; monitoring also stops automatically when the agent ends the conversation. (Firefox: switch to the CCP tab after starting, or the mic won't connect.)

### Permissions

- **CallCenterManager** + **Agent** profiles, **or** a custom profile with (Analytics and optimization) **Access metrics** + **Real-time contact monitoring** (monitor) + **Real-time contact barge-in** (barge), plus (CCP) **Access Contact Control Panel**.

---

## Recorded-Conversation Playback

The page is "Monitor live **and recorded** conversations" — recordings are reviewed under **Analytics and optimization → Contact search**:

- Call recordings land in your S3 bucket shortly after disconnect, but appear in the contact record only **after the contact leaves ACW**.
- Recorded contacts show **Play**/transcript icons in the **Recording/Transcript** column (hidden if you lack permission). Open the contact ID for the full player: scrubber, playback speed, skip ±10 seconds.
- IVR/automated audio appears under **Recording and Transcript → Automated Interaction (IVR)**; a **Show flow details** toggle (needs **Flow - View** + **Flow modules - View**) shows flow execution.
- Recording permissions (precise names): **Call recordings (unredacted) - Access** (controls access *and* the S3 URLs; **Enable download button** is a separate sub-permission that only shows the button), **Call recordings (redacted) - Access**, **Contact transcripts (unredacted)/(redacted) - Access**, **Manager monitor** (monitor live + listen to recordings), **Delete recorded conversations**, and the **Automated interaction voice (IVR) recordings/transcripts** variants.

---

## CloudWatch Metrics

Amazon Connect publishes metrics to CloudWatch every 1 minute. Data is sent to the `AWS/Connect` namespace. Metrics are available for two weeks in CloudWatch, then discarded.

### Instance-Level Metrics

Dimension: `InstanceId`

| Metric | Unit | Description |
|---|---|---|
| `ConcurrentCalls` | Count | Current number of concurrent active voice calls (all active, not just agent-connected). Use Maximum/Average statistic. |
| `ConcurrentCallsPercentage` | Percent | Concurrent calls as a percentage of the instance limit. Emitted as a decimal (e.g., 0.5 = 50%). |
| `ConcurrentActiveChats` | Count | Current number of concurrent active chat contacts (all active, not just agent-connected). Use Maximum/Average statistic. |
| `ConcurrentActiveChatsPercentage` | Percent | Concurrent chats as a percentage of the instance limit. Emitted as integer (e.g., 50 = 50%). |
| `ConcurrentTasks` | Count | Current concurrent active tasks. Data sent every 5 minutes. |
| `ConcurrentTasksPercentage` | Percent | Concurrent tasks as percentage of limit. Data sent every 5 minutes. Emitted as integer. |
| `ConcurrentEmails` | Count | Current concurrent active emails (queue + assigned + outbound). Data sent every 5 minutes. |
| `ConcurrentEmailsPercentage` | Percent | Concurrent emails as percentage of limit. Data sent every 5 minutes. |
| `CallsPerInterval` | Count | Number of voice calls (inbound + outbound) received or placed per second. |
| `CallsBreachingConcurrencyQuota` | Count | Voice calls that exceeded the concurrent call limit. Use Sum statistic for total. |
| `ChatsBreachingActiveChatQuota` | Count | Valid chat start requests that exceeded the concurrent active chats quota. Use Sum statistic. |
| `TasksBreachingConcurrencyQuota` | Count | Tasks that exceeded the concurrent tasks quota. Use Sum statistic. |
| `MissedCalls` | Count | Voice calls missed by agents (not answered within 20 seconds). Use Sum for total. |
| `ThrottledCalls` | Count | Voice calls rejected because rate of calls per second exceeded the quota. Use Sum for total. |
| `MisconfiguredPhoneNumbers` | Count | Calls that failed because the phone number is not associated with a flow. |
| `SuccessfulChatsPerInterval` | Count | Chats successfully started per interval. |
| `CallRecordingUploadError` | Count | Call recordings that failed to upload to S3. |
| `TasksExpired` | Count | Tasks that expired after being active for 7 days. |
| `TasksExpiryWarningReached` | Count | Tasks active for 6 days 22 hours that reached expiry warning. |

### Contact Flow Metrics

Dimension: `ContactFlowName` (must contain only: alphanumeric, `.`, `,`, `-`, `_`, `/`, `#`, `:`, `$`, `@`, `|`, `&`, `{`, `}`, `+`, `?`, `%`, space)

| Metric | Unit | Description |
|---|---|---|
| `ContactFlowErrors` | Count | Number of times the error branch for a flow was run (non-fatal). |
| `ContactFlowFatalErrors` | Count | Number of times a flow failed to execute due to a system error (terminated the contact). |
| `PublicSigningKeyUsage` | Count | Number of times a flow security key (public signing key) was used to encrypt customer input. |

### Queue Metrics

Dimension: `QueueName` (same character restrictions as flow names)

| Metric | Unit | Description |
|---|---|---|
| `LongestQueueWaitTime` | Seconds | Wait time of the oldest contact in the queue during the refresh interval. |
| `QueueSize` | Count | Number of contacts currently in the queue (point-in-time, not a sum). |
| `QueueCapacityExceededError` | Count | Contacts rejected because the queue was full. |
| `CallBackNotDialableNumber` | Count | Queued callbacks that could not be dialed because the customer's number is in a country not allowed for outbound calls. |

### Voice Quality Metrics

Dimension: `InstanceId`, `Participant`, `Stream Type`, `Type of Connection`

| Metric | Unit | Description |
|---|---|---|
| `ToInstancePacketLossRate` | Percent | Ratio of packet loss for calls, reported every 10 seconds. Value between 0 and 100, displayed as 0-1 percent. |

---

## CloudWatch Dimensions

| Dimension | Description | Metrics |
|---|---|---|
| `InstanceId` (Instance metrics) | Filter by Connect instance. | ConcurrentCalls, ConcurrentCallsPercentage, ConcurrentActiveChats, ConcurrentActiveChatsPercentage, ConcurrentTasks, ConcurrentTasksPercentage, ConcurrentEmails, ConcurrentEmailsPercentage, CallsPerInterval, CallsBreachingConcurrencyQuota, ChatsBreachingActiveChatQuota, TasksBreachingConcurrencyQuota, MissedCalls, ThrottledCalls, MisconfiguredPhoneNumbers, SuccessfulChatsPerInterval, CallRecordingUploadError |
| `ContactFlowName` (Flow metrics) | Filter by contact flow. | ContactFlowErrors, ContactFlowFatalErrors, PublicSigningKeyUsage |
| `QueueName` (Queue metrics) | Filter by queue. | LongestQueueWaitTime, QueueSize, QueueCapacityExceededError, CallBackNotDialableNumber |
| Instance ID, Participant, Stream Type, Type of Connection | Filter by connection. | ToInstancePacketLossRate |
| Contact metrics | Filter by contact. | TasksExpiryWarningReached, TasksExpired |

---

## App Integrations Metrics

Namespace: `AWS/AppIntegrations`

| Metric | Unit | Description |
|---|---|---|
| `RecordsDownloaded` | Count | Records successfully downloaded from integration source (AppFlow). |
| `RecordsFailed` | Count | Records that failed to download. |
| `DataDownloaded` | Bytes | Bytes successfully downloaded. |
| `DataProcessingDuration` | Milliseconds | Time to process and download data in a single AppFlow execution. |
| `EventsReceived` | Count | Events successfully received from third-party source (Salesforce, Zendesk). |
| `EventsProcessed` | Count | Events successfully processed and forwarded to rules. |
| `EventsThrottled` | Count | Events throttled due to exceeding max rate. |
| `EventsFailed` | Count | Events that failed due to malformed or unsupported events. |
| `EventProcessingDuration` | Milliseconds | Time to process and forward an event. |

All metrics: Frequency 1 minute. Valid Statistics: Maximum, Sum, Minimum, Average.

Dimensions: `AccountId`, `ClientId`, `IntegrationARN`, `IntegrationType` (DataIntegration or EventIntegration), `Region`.

---

## Customer Profiles Metrics

Namespace: `AWS/CustomerProfiles`

| Metric | Unit | Description |
|---|---|---|
| `EventsProcessed` | Count | Records successfully streamed into a Kinesis Stream. |
| `EventsThrottled` | Count | PutRecord attempts that encountered throttling. |

Dimensions: `DomainName`, `DestinationType` (Kinesis), `DestinationName` (Kinesis Data Stream name).

---

## Voice ID Metrics

Namespace: `VoiceID`

| Metric | Unit | Dimension | Cadence |
|---|---|---|---|
| `RequestLatency` | Milliseconds | API | 1 min |
| `UserErrors` | Count | API | 1 min |
| `SystemErrors` | Count | API | 1 min |
| `Throttles` | Count | API | 1 min |
| `ActiveSessions` | Count | Domain | 1 min |
| `ActiveSpeakerEnrollmentJobs` | Count | Domain | 15 min |
| `ActiveFraudsterRegistrationJobs` | Count | Domain | 15 min |
| `Speakers` | Count | Domain | 15 min |
| `Fraudsters` | Count | Domain | 15 min |

Dimensions: `API` (e.g. DeleteFraudster, EvaluateSession, ListSpeakers, DeleteSpeaker, OptOutSpeaker), `Domain` (Voice ID domain). *(Voice ID end-of-support: May 20, 2026.)*

---

## Quota Calculation Formulas

### Concurrent Call Utilization

```
Total Quota = ConcurrentCalls / ConcurrentCallsPercentage
```

Example: 20 calls / 0.5 = 40 total quota.

**Important:** `ConcurrentCallsPercentage` is emitted as a decimal (not multiplied by 100). `ConcurrentTasksPercentage` and `ConcurrentActiveChatsPercentage` ARE multiplied by 100.

### Concurrent Chat Quota

```
Total Quota = (ConcurrentActiveChats / ConcurrentActiveChatsPercentage) * 100
```

### Concurrent Task Quota

```
Total Quota = (ConcurrentTasks / ConcurrentTasksPercentage) * 100
```

### Concurrent Email Quota

```
Total Quota = (ConcurrentEmails / ConcurrentEmailsPercentage) * 100
```

### When to Alert

| Threshold | Action |
|---|---|
| 60% | Monitor -- approaching capacity. |
| 80% | Warning -- request quota increase if trending upward. |
| 90%+ | Critical -- immediate risk of breaching quota. |

### Estimating Required Quota

```
Required Quota = Peak_Concurrent_Calls * 1.3 (30% headroom)
```

Use the `ConcurrentCallsPercentage` metric with a 1-hour MAX statistic to track peak utilization over time.

---

## CloudWatch Alarms (Recommended)

| Alarm | Metric | Threshold | Description |
|---|---|---|---|
| High concurrency | `ConcurrentCallsPercentage` | > 80% for 5 minutes | Approaching call capacity. |
| Calls breaching quota | `CallsBreachingConcurrencyQuota` | > 0 for 1 minute | Calls being rejected. |
| Chats breaching quota | `ChatsBreachingActiveChatQuota` | > 0 for 1 minute | Chats being rejected. |
| Tasks breaching quota | `TasksBreachingConcurrencyQuota` | > 0 for 1 minute | Tasks being rejected. |
| Contact flow errors | `ContactFlowFatalErrors` | > 5 for 5 minutes | Flow execution failures. |
| Long queue wait | `LongestQueueWaitTime` | > 300 for 5 minutes | Oldest contact waiting > 5 minutes. |
| Recording failures | `CallRecordingUploadError` | > 0 for 5 minutes | Recording upload issues. |
| Packet loss | `ToInstancePacketLossRate` | > 5% for 5 minutes | Voice quality degradation. |
| Misconfigured numbers | `MisconfiguredPhoneNumbers` | > 0 for 1 minute | Phone numbers not associated with flows. |
| Task expiry warning | `TasksExpiryWarningReached` | > 0 for 5 minutes | Tasks approaching 7-day expiry. |

---

## CloudWatch Dashboard Widgets

Recommended widgets for a Connect monitoring dashboard:

| Widget Type | Metric(s) | Statistic | Description |
|---|---|---|---|
| Number | `ConcurrentCalls` | Maximum | Current call load at a glance. |
| Line | `ConcurrentCallsPercentage` | Maximum | Capacity utilization trend over time. |
| Line | `ConcurrentActiveChats`, `ConcurrentTasks` | Maximum | Multi-channel concurrency trends. |
| Number | `CallsBreachingConcurrencyQuota` | Sum | Total breached calls in period. |
| Line | `QueueSize` | Maximum | Queue depth per queue. |
| Line | `LongestQueueWaitTime` | Maximum | Longest wait per queue. |
| Number | `ContactFlowFatalErrors` | Sum | Flow failure count. |
| Line | `ToInstancePacketLossRate` | Average | Voice quality trend. |
| Number | `CallRecordingUploadError` | Sum | Recording failures. |
| Number | `MissedCalls` | Sum | Total missed calls. |

---

## CloudTrail Logging

All Amazon Connect public API calls are logged to AWS CloudTrail as **management events** (`eventType: AwsApiCall`, `eventCategory: Management`, `managementEvent: true`). The docs do not describe data-event logging. **`eventSource`** is `connect.amazonaws.com` (Voice ID uses `voiceid.amazonaws.com` and can share the same trail/S3 bucket; Voice ID redacts PII in logs as `HIDDEN_DUE_TO_SECURITY_REASONS`). **Service-linked roles are required** for the updated admin website + CloudTrail support. CloudTrail is on at account creation (recent activity in Event History); a trail delivers to S3 and applies to all Regions by default.

### What Is Logged

- API caller identity (`userIdentity`: IAM user/role, federated identity, or another AWS service).
- Timestamp, source IP, API action name, request parameters, response elements, `readOnly`, `recipientAccountId`.

*(The docs enumerate no fixed "key actions" list — the page's example action is `GetContactAttributes`. Any subset below is illustrative, not authoritative.)*

### Example actions to alert on (illustrative)

- `CreateUser`, `DeleteUser`, `UpdateUserPhoneConfig`
- `CreateQueue`, `UpdateQueueStatus`
- `CreateContactFlow`, `UpdateContactFlowContent`
- `StartOutboundVoiceContact`, `StopContact`
- `CreateEvaluationForm`, `SubmitContactEvaluation`
- `UpdateInstanceAttribute`, `AssociateBot`
- All administrative and operational APIs.

### CloudTrail Event Example

```json
{
  "eventVersion": "1.08",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROA...:session-name",
    "arn": "arn:aws:sts::123456789012:assumed-role/role-name/session-name",
    "accountId": "123456789012"
  },
  "eventTime": "2026-05-25T14:30:00Z",
  "eventSource": "connect.amazonaws.com",
  "eventName": "UpdateContactFlowContent",
  "awsRegion": "us-east-1",
  "sourceIPAddress": "203.0.113.50",
  "requestParameters": {
    "instanceId": "instance-id",
    "contactFlowId": "flow-id"
  }
}
```

### Audit Use Cases

- Track who modified a contact flow and when.
- Identify unauthorized API calls.
- Compliance audit trails.
- Forensic analysis of incidents.

---

## EventBridge Integration

Amazon Connect publishes events to Amazon EventBridge for event-driven automation.

### Event Types

| Event | Description |
|---|---|
| **Contact events** | INITIATED, CONNECTED_TO_SYSTEM, CONTACT_DATA_UPDATED, QUEUED, CONNECTED_TO_AGENT, DISCONNECTED, PAUSED, RESUMED, COMPLETED, AMD_DISABLED, WEBRTC_API. |
| **Agent events** | Login, logout, status changes, contact state transitions (via Kinesis, not EventBridge). |
| **Contact Lens rules** | When a Contact Lens rule is matched, an event can be published. |
| **Evaluation events** | Evaluation submitted, form activated/deactivated. |
| **Case events** | Case created, updated, resolved. |
| **Screen recording events** | Screen recording state changes. |

### Event Pattern Example

```json
{
  "source": ["aws.connect"],
  "detail-type": ["Amazon Connect Contact Event"],
  "detail": {
    "eventType": ["DISCONNECTED"],
    "channel": ["VOICE"],
    "instanceArn": ["arn:aws:connect:us-east-1:123456789012:instance/abc"]
  }
}
```

### Common Targets

| Target | Use Case |
|---|---|
| **Lambda** | Custom business logic on contact events. |
| **SQS** | Queue events for async processing. |
| **SNS** | Send notifications to supervisors. |
| **Step Functions** | Orchestrate multi-step workflows triggered by events. |
| **CloudWatch Logs** | Log events for analysis. |
| **Kinesis Data Streams** | Stream events for real-time analytics. |

---

## Monitoring Best Practices

1. **Set up CloudWatch alarms** for all critical metrics before going into production.
2. **Enable CloudTrail** in a multi-region configuration and send logs to a centralized S3 bucket.
3. **Use EventBridge** for operational automation rather than polling APIs.
4. **Monitor packet loss** (`ToInstancePacketLossRate`) to proactively identify voice quality issues.
5. **Track concurrent utilization** trends weekly to anticipate quota increase requests.
6. **Create a CloudWatch dashboard** with key metrics: concurrent calls, queue size, longest wait time, flow errors, and recording failures.
7. **Review CloudTrail logs** regularly for unauthorized configuration changes.
8. **Set up IAM permissions** for CloudWatch PutMetricData if your instance was created on or before October 2018 (required for chat metrics).
