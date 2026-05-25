# EventBridge Events

Amazon Connect emits events to the **default EventBridge event bus** in your AWS account. These events enable event-driven architectures for contact center automation, monitoring, and integration with downstream services.

## Event Envelope

All Connect events follow this structure:

```json
{
  "version": "0",
  "id": "event-uuid",
  "detail-type": "Amazon Connect Contact Event",
  "source": "aws.connect",
  "account": "123456789012",
  "time": "2026-05-25T14:30:00Z",
  "region": "us-east-1",
  "detail": { ... }
}
```

- **Source:** `aws.connect`
- **Detail-type:** `"Amazon Connect Contact Event"` for contact events (other detail-types for rules, evaluations, etc.)
- **Delivery:** Best effort -- events may be delayed or, in rare cases, duplicated

## Event Categories

There are **5 event categories**:

| Category | Description |
|---|---|
| Contact events | Contact lifecycle state changes (11 types) |
| Rule events | Contact Lens rule triggers |
| Performance evaluation events | Agent evaluation completions |
| Screen recording events | Screen recording state changes |

## Contact Event Types (11 Types)

| Event Type | Description |
|---|---|
| `INITIATED` | Contact has been created (inbound call received, outbound call placed, chat started, etc.) |
| `CONNECTED_TO_SYSTEM` | Customer is connected to the IVR/flow system |
| `CONTACT_DATA_UPDATED` | Contact attributes or metadata have been modified |
| `QUEUED` | Contact has been placed in a queue |
| `CONNECTED_TO_AGENT` | Contact has been routed to and accepted by an agent |
| `DISCONNECTED` | One party has disconnected (call ended, chat closed) |
| `PAUSED` | Contact has been paused (chat) |
| `RESUMED` | Previously paused contact has been resumed |
| `COMPLETED` | Contact is fully completed (including ACW) |
| `AMD_DISABLED` | Answering machine detection has been disabled for this contact |
| `WEBRTC_API` | Contact initiated via the WebRTC API |

### Contact Event Lifecycle (Voice)

```
INITIATED -> CONNECTED_TO_SYSTEM -> QUEUED -> CONNECTED_TO_AGENT -> DISCONNECTED -> COMPLETED
```

Not all events occur for every contact. For example, a call that is answered by IVR and resolved without an agent will not have `QUEUED` or `CONNECTED_TO_AGENT`.

## Data Model Objects

### AgentInfo

Present in events where an agent is involved (`CONNECTED_TO_AGENT`, `DISCONNECTED`, `COMPLETED`):

```json
{
  "AgentInfo": {
    "AgentARN": "arn:aws:connect:.../agent/agent-id",
    "HoldDuration": 15,
    "AfterContactWorkStartTimestamp": "2026-05-25T14:35:00.000Z",
    "AfterContactWorkEndTimestamp": "2026-05-25T14:37:30.000Z",
    "AfterContactWorkDuration": 150,
    "HierarchyGroups": {
      "Level1": { "GroupARN": "...", "GroupName": "Organization" },
      "Level2": { "GroupARN": "...", "GroupName": "Sales Team" }
    }
  }
}
```

| Field | Description |
|---|---|
| `AgentARN` | ARN of the assigned agent |
| `HoldDuration` | Total seconds the customer was on hold |
| `AfterContactWorkStartTimestamp` | When ACW began |
| `AfterContactWorkEndTimestamp` | When ACW ended |
| `AfterContactWorkDuration` | ACW duration in seconds |
| `HierarchyGroups` | Agent's hierarchy (up to 5 levels) |

### QueueInfo

```json
{
  "QueueInfo": {
    "QueueARN": "arn:aws:connect:.../queue/queue-id",
    "QueueType": "STANDARD",
    "EnqueueTimestamp": "2026-05-25T14:28:30.000Z",
    "DequeueTimestamp": "2026-05-25T14:29:05.000Z"
  }
}
```

| Field | Description |
|---|---|
| `QueueARN` | ARN of the queue |
| `QueueType` | `STANDARD` or `AGENT` (direct-to-agent queue) |

### RoutingCriteria

Describes how the contact was routed, including step-based routing:

```json
{
  "RoutingCriteria": {
    "Steps": [
      {
        "Expression": {
          "AttributeCondition": {
            "Name": "Language",
            "Value": "English",
            "ComparisonOperator": "NumberGreaterOrEqualTo",
            "ProficiencyLevel": 3.0
          }
        },
        "Expiry": {
          "DurationInSeconds": 30
        },
        "Status": "EXPIRED"
      }
    ]
  }
}
```

| Field | Description |
|---|---|
| `Steps` | Ordered routing steps with criteria |
| `Expression` | Attribute-based matching condition |
| `Expiry` | Time limit before falling through to next step |
| `Status` | `ACTIVE`, `EXPIRED`, `JOINED`, or `INACTIVE` |

### CustomerVoiceActivity

Tracks when the customer speaks during IVR interactions:

```json
{
  "CustomerVoiceActivity": {
    "GreetingStartTimestamp": "2026-05-25T14:28:02.000Z",
    "GreetingEndTimestamp": "2026-05-25T14:28:05.000Z"
  }
}
```

### Recordings and RecordingsInfo

```json
{
  "Recordings": [
    {
      "StorageType": "S3",
      "Location": "s3://connect-recordings-bucket/recordings/2026/05/25/contact-id.wav",
      "MediaStreamType": "AUDIO",
      "ParticipantType": "CUSTOMER",
      "FragmentStartNumber": "...",
      "FragmentStopNumber": "...",
      "Status": "AVAILABLE"
    }
  ],
  "RecordingsInfo": {
    "StorageType": "S3",
    "Location": "s3://..."
  }
}
```

### Additional Data Objects

| Object | Description |
|---|---|
| `ChatMetrics` | Chat-specific metrics (response times, message counts) |
| `ParticipantMetrics` | Per-participant metrics (agent and customer) |
| `ContactEvaluations` | Performance evaluations completed for this contact |
| `StateTransitions` | History of contact state changes with timestamps |
| `OutboundStrategy` | Outbound campaign configuration (dialing mode, retry settings) |
| `GlobalResiliencyMetadata` | Cross-region failover information |

## Timestamps

Events include multiple timestamps to track the full contact lifecycle:

| Timestamp | Description |
|---|---|
| `InitiationTimestamp` | When the contact was created |
| `ConnectedToSystemTimestamp` | When the customer connected to the IVR/system |
| `EnqueueTimestamp` | When the contact entered a queue |
| `ConnectedToAgentTimestamp` | When the agent accepted the contact |
| `DisconnectTimestamp` | When disconnection occurred |
| `ScheduledTimestamp` | For scheduled callbacks: the scheduled time |

All timestamps are ISO 8601 format in UTC.

## DisconnectReason (26 Codes)

The `DisconnectReason` field explains why a contact ended. The 26 reason codes include:

| Code | Description |
|---|---|
| `CUSTOMER_DISCONNECT` | Customer hung up or left chat |
| `AGENT_DISCONNECT` | Agent ended the contact |
| `THIRD_PARTY_DISCONNECT` | Third party on a conference call disconnected |
| `TELECOM_PROBLEM` | Telephony network issue |
| `CONTACT_FLOW_DISCONNECT` | Flow terminated the contact |
| `EXPIRED` | Contact expired (e.g., chat timeout) |
| `API` | Contact ended via API call |
| `BARGE` | Supervisor barged and ended the contact |
| `OTHER` | Other/unknown reason |

Additional codes cover specific scenarios: `IDLE_DISCONNECT`, `SILENT_MONITOR_DISCONNECT`, `SUPERVISOR_DISCONNECT`, transfer-related disconnects, campaign-related disconnects, and system error disconnects.

## AnsweringMachineDetectionStatus (12 Values)

For outbound calls with answering machine detection (AMD) enabled:

| Status | Description |
|---|---|
| `HUMAN_ANSWERED` | A human answered the call |
| `VOICEMAIL_BEEP` | Voicemail detected (with beep) |
| `VOICEMAIL_NO_BEEP` | Voicemail detected (no beep) |
| `AMD_UNANSWERED` | No answer detected |
| `AMD_UNRESOLVED` | Detection could not determine |
| `AMD_NOT_APPLICABLE` | AMD not enabled for this call |
| `SIT_TONE_DETECTED` | Special information tone detected (invalid number) |
| `SIT_TONE_BUSY` | Busy SIT tone |
| `SIT_TONE_INVALID_NUMBER` | Invalid number SIT tone |
| `FAX_MACHINE_DETECTED` | Fax machine detected |
| `AMD_ERROR` | Error during detection |
| `AMD_DISABLED` | AMD was disabled mid-call |

## InitiationMethod (12 Values)

| Method | Description |
|---|---|
| `INBOUND` | Customer-initiated inbound contact |
| `OUTBOUND` | Agent-initiated outbound call |
| `TRANSFER` | Transferred from another agent or queue |
| `CALLBACK` | Queued callback |
| `QUEUE_TRANSFER` | Transferred between queues |
| `DISCONNECT` | Post-disconnect flow execution |
| `MONITOR` | Supervisor monitoring |
| `EXTERNAL_OUTBOUND` | Outbound campaign call |
| `WEBRTC_API` | WebRTC API-initiated |
| `FLOW` | Flow-initiated contact |
| `AGENT_REPLY` | Agent reply to a contact |
| `API` | API-initiated contact |

## Subscribing to Events

Create an EventBridge rule in the AWS console or via infrastructure as code:

```javascript
// EventBridge rule pattern to match all Connect contact events
{
  "source": ["aws.connect"],
  "detail-type": ["Amazon Connect Contact Event"]
}
```

```javascript
// Pattern to match only specific event types
{
  "source": ["aws.connect"],
  "detail-type": ["Amazon Connect Contact Event"],
  "detail": {
    "eventType": ["DISCONNECTED", "COMPLETED"]
  }
}
```

**Common targets:**
- Lambda function (process event, update database)
- SQS queue (buffer events for batch processing)
- SNS topic (fan-out to multiple subscribers)
- Step Functions (orchestrate multi-step workflows)
- CloudWatch Logs (audit trail)

## Key Considerations

- **Delivery guarantee:** Best effort. Events may arrive late or be duplicated. Design consumers to be idempotent.
- **Latency:** Events are typically delivered within seconds but can be delayed under high load.
- **Event size:** Large contacts with many state transitions can produce sizable event payloads. Monitor Lambda payload limits (6 MB synchronous, 256 KB asynchronous).
- **Filtering:** Use EventBridge content-based filtering to route only relevant events to each target, reducing Lambda invocations and cost.
- **Cross-account:** EventBridge supports cross-account event delivery if your processing infrastructure is in a different account.
- **Ordering:** No ordering guarantee across events. Use timestamps for sequencing.
- **Region:** Events are emitted to the event bus in the same region as the Connect instance.
