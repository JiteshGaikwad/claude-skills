# Contact Records (CTRs)

Contact Trace Records (CTRs) are the foundational data model in Amazon Connect. Every contact generates one or more CTR segments that capture the complete lifecycle of the interaction.

---

## CTR Data Model

### ContactTraceRecord Object

The top-level object for each contact segment. Key fields:

| Field | Type | Description |
|---|---|---|
| `ContactId` | String | Unique identifier for this contact. |
| `InitialContactId` | String | The ContactId of the first contact in a chain (for transfers/callbacks). Same as ContactId if this is the original contact. |
| `PreviousContactId` | String | The ContactId of the contact that preceded this one in a transfer chain. Null if this is the original contact. |
| `NextContactId` | String | The ContactId of the next contact in the chain (if this contact was transferred). |
| `RelatedContactId` | String | Links contacts that are related but not in a direct transfer chain. |
| `Channel` | String | VOICE, CHAT, TASK, or EMAIL. |
| `InitiationMethod` | String | How the contact was initiated (see below). |
| `DisconnectReason` | String | Why the contact ended (see below). |
| `InitiationTimestamp` | ISO 8601 | When the contact was initiated. |
| `ConnectedToAgentTimestamp` | ISO 8601 | When the contact was connected to an agent. Null if never connected. |
| `DisconnectTimestamp` | ISO 8601 | When the contact was disconnected. |
| `LastUpdateTimestamp` | ISO 8601 | Last update timestamp for the CTR. |
| `ScheduledTimestamp` | ISO 8601 | For callbacks, the scheduled callback time. |
| `TransferCompletedTimestamp` | ISO 8601 | When a transfer was completed. |
| `TransferredToEndpoint` | Object | The endpoint the contact was transferred to. |

### Queue Object

| Field | Type | Description |
|---|---|---|
| `ARN` | String | Queue ARN. |
| `Name` | String | Queue name. |
| `EnqueueTimestamp` | ISO 8601 | When the contact was placed in queue. |
| `DequeueTimestamp` | ISO 8601 | When the contact was removed from queue (answered or abandoned). |
| `Duration` | Integer | Time in queue in seconds. |

### Agent Object

| Field | Type | Description |
|---|---|---|
| `ARN` | String | Agent ARN. |
| `Username` | String | Agent username. |
| `FirstName` | String | Agent first name. |
| `LastName` | String | Agent last name. |
| `RoutingProfileARN` | String | Agent's routing profile ARN. |
| `HierarchyGroups` | Object | Agent's hierarchy group at each level (Level1 through Level5). |
| `AgentInteractionDuration` | Integer | Seconds agent was interacting with the contact (excludes hold). |
| `CustomerHoldDuration` | Integer | Seconds the customer was on hold. |
| `AfterContactWorkDuration` | Integer | Seconds the agent spent in ACW. |
| `AfterContactWorkStartTimestamp` | ISO 8601 | When ACW started. |
| `AfterContactWorkEndTimestamp` | ISO 8601 | When ACW ended. |
| `ConnectedToAgentTimestamp` | ISO 8601 | When agent connected to the contact. |
| `NumberOfHolds` | Integer | Number of times the agent put the customer on hold. |
| `LongestHoldDuration` | Integer | Duration of the longest hold in seconds. |

### Recording Object

| Field | Type | Description |
|---|---|---|
| `Type` | String | AUDIO or SCREEN. |
| `Status` | String | AVAILABLE, DELETING, DELETED, null. |
| `Location` | String | S3 URI of the recording file. |
| `DeletionReason` | String | Reason if recording was deleted (retention policy, manual). |
| `StorageType` | String | S3 or KINESIS_VIDEO_STREAM. |

### Attributes Object

A map of key-value pairs representing contact attributes set during the contact flow or by the agent.

```json
{
  "Attributes": {
    "CustomerType": "Premium",
    "AccountNumber": "12345",
    "Language": "en-US"
  }
}
```

Maximum 32 KB total size for all attributes combined.

### SegmentAttributes Object

Additional metadata about the contact segment:

| Field | Description |
|---|---|
| `connect:Subtype` | Channel subtype (e.g., `connect:SMS`, `connect:WebRTC`, `connect:Telephony`). |
| `connect:Direction` | INBOUND or OUTBOUND. |
| `connect:CreatedByUser` | ARN of the user who created the contact (for agent-initiated). |
| `connect:MediaStreams` | Media streaming configuration. |

---

## Initiation Methods

| Value | Description |
|---|---|
| `INBOUND` | Customer-initiated contact (incoming call, chat, email). |
| `OUTBOUND` | Agent or automated outbound contact. |
| `TRANSFER` | Contact transferred from another agent or queue. |
| `CALLBACK` | Queued callback initiated by the system. |
| `API` | Contact initiated via the StartChatContact, StartOutboundVoiceContact, or StartTaskContact API. |
| `QUEUE_TRANSFER` | Contact transferred between queues (not agent-to-agent). |
| `EXTERNAL_OUTBOUND` | Outbound contact to an external number (e.g., third-party conference). |
| `MONITOR` | Supervisor monitoring session. |
| `DISCONNECT` | System-generated disconnect contact segment. |

---

## Disconnect Reasons

| Value | Description |
|---|---|
| `CUSTOMER_DISCONNECT` | Customer hung up or left the chat. |
| `AGENT_DISCONNECT` | Agent ended the contact. |
| `THIRD_PARTY_DISCONNECT` | Third party in a conference disconnected. |
| `TELECOM_PROBLEM` | Telephony issue caused disconnection. |
| `CONTACT_FLOW_DISCONNECT` | Contact flow ended the contact (e.g., Disconnect block). |
| `OTHER` | Other/unknown reason. |
| `EXPIRED` | Contact expired (e.g., task TTL, chat idle timeout). |
| `API` | Contact ended via API (StopContact). |
| `BARGED` | Supervisor barged into the contact and it ended. |

---

## Contact States

Contacts transition through the following states during their lifecycle:

| State | Description |
|---|---|
| `INCOMING` | Contact has arrived and is in the contact flow (IVR). |
| `PENDING` | Contact is waiting to be routed (in queue). |
| `CONNECTING` | Contact is being offered to an agent (ringing). |
| `CONNECTED` | Contact is connected to an agent (active conversation). |
| `CONNECTED_ONHOLD` | Contact is connected but customer is on hold. |
| `PAUSED` | Contact is paused (task channel). |
| `MISSED` | Agent did not answer within the timeout. Contact will be re-queued. |
| `ERROR` | An error occurred during the contact. |
| `ENDED` | The contact interaction has ended but ACW may still be in progress. |
| `REJECTED` | Agent explicitly rejected the contact. |

### State Transition Flow

```
INCOMING -> PENDING -> CONNECTING -> CONNECTED -> ENDED
                                  -> CONNECTED_ONHOLD -> CONNECTED -> ENDED
                                  -> MISSED -> PENDING (re-queued)
                                  -> REJECTED -> PENDING (re-queued)
                                  -> ERROR
```

---

## Contact Chains

Contacts are linked through transfer and callback chains using three ID fields:

### Original Contact

```
ContactId: A
InitialContactId: A
PreviousContactId: null
NextContactId: B (if transferred)
```

### Transferred Contact

```
ContactId: B
InitialContactId: A (points to the original)
PreviousContactId: A (points to the contact that transferred)
NextContactId: C (if transferred again)
```

### Second Transfer

```
ContactId: C
InitialContactId: A (always points to the very first contact)
PreviousContactId: B (points to the immediate predecessor)
NextContactId: null (end of chain)
```

### Chain Rules

- `InitialContactId` always points to the **first contact** in the entire chain.
- `PreviousContactId` points to the **immediately preceding** contact.
- `NextContactId` points to the **immediately following** contact.
- All contacts in a chain share the same `InitialContactId`.
- To reconstruct a full chain, start from any contact, follow `InitialContactId` to the root, then traverse `NextContactId` forward.

---

## Conferences and Transfers Identification

### Identifying Transfers

A contact was transferred if:
- `NextContactId` is not null (this contact was the source of a transfer).
- `InitiationMethod` is `TRANSFER` or `QUEUE_TRANSFER` (this contact is the result of a transfer).
- `TransferCompletedTimestamp` is populated.

### Identifying Conferences

A conference (multi-party call) is identified when:
- Multiple agent segments exist for the same `ContactId`.
- The `DisconnectReason` of the original agent segment shows `THIRD_PARTY_DISCONNECT` if they left the conference.
- Conference participants appear as additional agent records in the CTR.

### Warm vs. Cold Transfer

- **Warm (consultative) transfer** — The original agent stays on the line while the new agent joins. Both agents appear in the CTR for a period. The original agent then disconnects.
- **Cold (blind) transfer** — The original agent disconnects immediately after initiating the transfer. The `TransferCompletedTimestamp` is very close to the disconnect time.

---

## Queued Callbacks

Callback contacts have specific CTR behavior:

| Behavior | Description |
|---|---|
| **InitiationMethod** | Set to `CALLBACK`. |
| **ScheduledTimestamp** | The time the callback was scheduled for. |
| **InitialContactId** | Points to the original inbound contact that requested the callback. |
| **Contact flow** | The callback-specific contact flow is invoked when the callback is initiated. |
| **Retry behavior** | If the customer doesn't answer, the callback may retry based on the queue's callback configuration. Each retry creates a new CTR segment. |

---

## Contact Events

Contact events provide real-time state change notifications via EventBridge.

### Event Sequence

| Event | Description |
|---|---|
| `INITIATED` | Contact has been created. |
| `QUEUED` | Contact has been placed in a queue. |
| `CONNECTED_TO_AGENT` | Contact has been connected to an agent. |
| `PAUSED` | Contact has been paused (task). |
| `RESUMED` | Contact has been resumed from paused state. |
| `HOLD` | Customer placed on hold. |
| `UNHOLD` | Customer taken off hold. |
| `TRANSFERRED` | Contact has been transferred. |
| `DISCONNECTED` | Contact has been disconnected. |
| `COMPLETED` | Contact processing is fully complete (including ACW). |

### EventBridge Integration

Contact events are published to the default EventBridge bus. You can create rules to route these events to Lambda, SQS, SNS, Step Functions, or other targets.

```json
{
  "source": "aws.connect",
  "detail-type": "Amazon Connect Contact Event",
  "detail": {
    "contactId": "12345-abcde",
    "eventType": "CONNECTED_TO_AGENT",
    "instanceArn": "arn:aws:connect:...",
    "channel": "VOICE"
  }
}
```

---

## Data Retention

- CTRs are retained for **24 months** in Amazon Connect.
- After 24 months, CTRs are no longer available via the Connect console or APIs.
- For longer retention, stream CTRs to an external store.

### Kinesis Streaming for Longer Retention

Configure a Kinesis Data Stream or Kinesis Data Firehose in the instance settings:

1. Navigate to **Data storage** in the Connect console.
2. Under **Contact trace records**, enable Kinesis streaming.
3. Select an existing Kinesis Data Stream or Firehose delivery stream.
4. CTRs are published to Kinesis in near real-time as JSON.

The Kinesis consumer (e.g., Firehose to S3, Lambda processing) stores the CTRs for your desired retention period.

---

## CTR Size and Limits

| Limit | Value |
|---|---|
| Contact attributes total size | 32 KB |
| CTR availability after contact ends | ~15 minutes |
| Maximum retention in Connect | 24 months |
| Kinesis record size | Up to 1 MB per CTR record |

---

## Accessing CTRs

| Method | Description |
|---|---|
| **Console** | Contact search page — search and view individual CTRs. |
| **API** | `DescribeContact`, `SearchContacts`, `GetContactAttributes`. |
| **Kinesis** | Real-time streaming of CTRs as JSON records. |
| **Data lake** | Query CTRs via Athena in the analytics data lake. |
| **S3 export** | Scheduled historical reports export CTR data to S3 in CSV format. |
