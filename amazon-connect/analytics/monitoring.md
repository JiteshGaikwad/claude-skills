# Monitoring

Amazon Connect monitoring spans live call/chat monitoring, CloudWatch metrics, CloudTrail logging, and EventBridge integration.

---

## Live Monitoring

### Voice Monitoring

Supervisors can listen to live voice calls in real-time.

- Access from the **Real-time metrics** page by clicking the eye icon next to an active agent.
- The supervisor hears both the agent and customer but is not heard by either party (silent monitoring).
- Requires the `Access metrics - Real-time metrics` and `Manager monitor` permissions in the supervisor's security profile.

### Chat Monitoring

Supervisors can monitor live chat conversations.

- View the chat transcript in real-time from the Real-time metrics page.
- The supervisor sees messages from both agent and customer.
- Silent monitoring — the customer and agent are not aware of the supervisor.

### Barge

Supervisors can barge into live voice conversations:

- From the monitoring state, the supervisor can switch to **Barge** mode.
- In Barge mode, the supervisor joins the call as an active participant.
- Both the agent and customer can hear the supervisor.
- Useful for critical situations where supervisor intervention is needed immediately.

### Toggle Between Monitor and Barge

- While monitoring, click **Barge** to join the conversation.
- While barged, the supervisor can choose to disconnect and return to monitor-only mode.
- The transition is seamless — no call drop or reconnection required.

### Monitoring from Real-Time Metrics

1. Navigate to **Analytics and optimization > Real-time metrics > Agents**.
2. Find the agent with an active contact.
3. Click the **eye icon** (monitor) next to the agent's name.
4. The supervisor's CCP opens with the monitoring session.
5. Toggle between Monitor and Barge as needed.

---

## CloudWatch Metrics

Amazon Connect publishes 25+ metrics to the `AWS/Connect` CloudWatch namespace.

### Instance-Level Metrics

| Metric | Unit | Description |
|---|---|---|
| `ConcurrentCalls` | Count | Current number of concurrent active voice calls. |
| `ConcurrentCallsPercentage` | Percent | Concurrent calls as a percentage of the instance limit. |
| `ConcurrentActiveChats` | Count | Current number of concurrent active chat contacts. |
| `ConcurrentActiveChatsPercentage` | Percent | Concurrent chats as a percentage of the instance limit. |
| `ConcurrentTasks` | Count | Current number of concurrent active tasks. |
| `ConcurrentTasksPercentage` | Percent | Concurrent tasks as a percentage of the instance limit. |
| `ConcurrentEmails` | Count | Current number of concurrent active email contacts. |
| `ConcurrentEmailsPercentage` | Percent | Concurrent emails as a percentage of the instance limit. |
| `CallsPerInterval` | Count | Number of voice calls received per interval. |
| `CallsBreachingConcurrencyQuota` | Count | Voice calls that could not be placed because the concurrent call limit was reached. |
| `MissedCalls` | Count | Voice calls that were missed by agents. |
| `ThrottledCalls` | Count | Voice calls rejected because API rate limits were exceeded. |

### Contact Flow Metrics

| Metric | Unit | Description |
|---|---|---|
| `ContactFlowErrors` | Count | Number of errors in contact flow execution (non-fatal). |
| `ContactFlowFatalErrors` | Count | Number of fatal errors in contact flow execution that terminated the contact. |
| `PublicSigningKeyUsage` | Count | Contact flows using the public signing key. |

### Queue Metrics

| Metric | Unit | Description |
|---|---|---|
| `LongestQueueWaitTime` | Seconds | Wait time of the oldest contact in the queue. |
| `QueueSize` | Count | Number of contacts currently in the queue. |
| `QueueCapacityExceededError` | Count | Contacts that could not be queued because the queue capacity was reached. |

### Voice Quality Metrics

| Metric | Unit | Description |
|---|---|---|
| `ToInstancePacketLossRate` | Percent | Packet loss rate for audio going to the Connect instance. |
| `CallRecordingUploadError` | Count | Errors uploading call recordings to S3. |

### App Integrations Metrics

Metrics for data integrations (e.g., third-party CRM data flowing into Customer Profiles).

| Metric | Unit | Description |
|---|---|---|
| `RecordsDownloaded` | Count | Records downloaded from the integration source. |
| `EventsReceived` | Count | Events received from the integration source. |
| `RecordsFailed` | Count | Records that failed to process. |
| `RecordsProcessed` | Count | Records successfully processed. |

### Customer Profiles Metrics

| Metric | Unit | Description |
|---|---|---|
| `EventsProcessed` | Count | Customer profile events successfully processed. |
| `EventsThrottled` | Count | Customer profile events throttled due to rate limits. |
| `EventsFailed` | Count | Customer profile events that failed processing. |

---

## CloudWatch Dimensions

Metrics can be filtered by the following dimensions:

| Dimension | Description |
|---|---|
| `InstanceId` | Filter by Connect instance. |
| `ContactFlowName` | Filter by contact flow (for flow metrics). |
| `QueueName` | Filter by queue (for queue metrics). |
| `MetricGroup` | Group of related metrics (e.g., `CallsPerInterval`). |

---

## Quota Calculation Formulas

### Concurrent Call Utilization

```
Utilization % = (ConcurrentCalls / ServiceQuota_ConcurrentCalls) * 100
```

### When to Alert

| Threshold | Action |
|---|---|
| 60% | Monitor — approaching capacity. |
| 80% | Warning — request quota increase if trending upward. |
| 90%+ | Critical — immediate risk of `CallsBreachingConcurrencyQuota`. |

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
| Contact flow errors | `ContactFlowFatalErrors` | > 5 for 5 minutes | Flow execution failures. |
| Long queue wait | `LongestQueueWaitTime` | > 300 for 5 minutes | Oldest contact waiting > 5 minutes. |
| Recording failures | `CallRecordingUploadError` | > 0 for 5 minutes | Recording upload issues. |
| Packet loss | `ToInstancePacketLossRate` | > 5% for 5 minutes | Voice quality degradation. |

---

## CloudTrail Logging

All Amazon Connect public API calls are logged to AWS CloudTrail.

### What Is Logged

- API caller identity (IAM user/role, federated identity).
- Timestamp of the API call.
- Source IP address.
- API action name.
- Request parameters.
- Response elements.

### Key API Actions Logged

- `CreateUser`, `DeleteUser`, `UpdateUserPhoneConfig`
- `CreateQueue`, `UpdateQueueStatus`
- `CreateContactFlow`, `UpdateContactFlowContent`
- `StartOutboundVoiceContact`, `StopContact`
- `CreateEvaluationForm`, `SubmitContactEvaluation`
- `UpdateInstanceAttribute`, `AssociateBot`
- All administrative and operational APIs.

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
| **Contact events** | INITIATED, QUEUED, CONNECTED_TO_AGENT, DISCONNECTED, COMPLETED. |
| **Agent events** | Login, logout, status changes, contact state transitions. |
| **Contact Lens rules** | When a Contact Lens rule is matched, an event can be published. |
| **Evaluation events** | Evaluation submitted, form activated/deactivated. |
| **Case events** | Case created, updated, resolved. |

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
