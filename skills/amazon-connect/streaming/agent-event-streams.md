# Agent Event Streams

Agent event streams provide near-real-time agent activity data via Amazon Kinesis Data Streams. They enable external systems to track agent state, routing configuration changes, and contact lifecycle without polling the Connect API.

## Event Types

There are **4 event types**:

| EventType | Trigger | Frequency |
|---|---|---|
| `LOGIN` | Agent logs in to the CCP | Once per session |
| `LOGOUT` | Agent logs out of the CCP | Once per session |
| `STATE_CHANGE` | Agent status, conversation state, or configuration changes | Per change |
| `HEART_BEAT` | Periodic pulse for connected agents | Every **120 seconds** |

### STATE_CHANGE Subtypes

`STATE_CHANGE` fires for several categories of change:

- **Status change** -- Agent moves between Available, Offline, or custom statuses (e.g., Break, Lunch)
- **Conversation state change** -- Contact transitions through states (INCOMING, CONNECTED, ENDED, etc.)
- **Configuration change** -- Any of these are modified:
  - Routing profile assignment
  - Queue membership
  - Auto-accept call setting
  - SIP address
  - Hierarchy group assignment
  - Language preference

### HEART_BEAT Behavior

- Emitted every **120 seconds** while the agent has an active session
- Continues for **1 hour after logout** -- this lets consumers detect stale sessions
- Contains the same `CurrentAgentSnapshot` as the most recent `STATE_CHANGE`
- Useful as a liveness signal for dashboards and monitoring systems

## Data Model

### Top-Level AgentEvent

```json
{
  "AgentARN": "arn:aws:connect:us-east-1:123456789012:instance/i-id/agent/a-id",
  "AWSAccountId": "123456789012",
  "EventId": "unique-event-uuid",
  "EventTimestamp": "2026-05-25T14:30:00.000Z",
  "EventType": "STATE_CHANGE",
  "InstanceARN": "arn:aws:connect:us-east-1:123456789012:instance/i-id",
  "CurrentAgentSnapshot": { ... },
  "PreviousAgentSnapshot": { ... },
  "Version": "2017-10-01"
}
```

| Field | Type | Description |
|---|---|---|
| `AgentARN` | String | ARN of the agent |
| `AWSAccountId` | String | AWS account ID |
| `EventId` | String | Unique identifier for this event |
| `EventTimestamp` | ISO 8601 | When the event occurred |
| `EventType` | String | `LOGIN`, `LOGOUT`, `STATE_CHANGE`, or `HEART_BEAT` |
| `InstanceARN` | String | ARN of the Connect instance |
| `CurrentAgentSnapshot` | Object | Agent state after the event |
| `PreviousAgentSnapshot` | Object | Agent state before the event (null for LOGIN) |
| `Version` | String | Schema version |

### AgentSnapshot

Both `CurrentAgentSnapshot` and `PreviousAgentSnapshot` share this structure:

```json
{
  "AgentStatus": {
    "ARN": "arn:aws:connect:.../agent-status/status-id",
    "Name": "Available",
    "StartTimestamp": "2026-05-25T14:28:00.000Z",
    "Type": "ROUTABLE"
  },
  "NextAgentStatus": {
    "ARN": "arn:aws:connect:.../agent-status/status-id",
    "Name": "Break",
    "EnqueuedTimestamp": "2026-05-25T14:29:55.000Z"
  },
  "Configuration": {
    "FirstName": "Jane",
    "LastName": "Smith",
    "Username": "jsmith",
    "RoutingProfile": { ... },
    "HierarchyGroups": { ... },
    "Proficiencies": [ ... ]
  },
  "Contacts": [ ... ]
}
```

### AgentStatus

| Field | Type | Description |
|---|---|---|
| `ARN` | String | ARN of the agent status |
| `Name` | String | Display name (e.g., "Available", "Break", "Offline") |
| `StartTimestamp` | ISO 8601 | When the agent entered this status |
| `Type` | Enum | `ROUTABLE` (available for contacts), `CUSTOM` (user-defined), or `OFFLINE` |

### NextAgentStatus

Present when an agent has queued a status change (e.g., selected "Break" while still on a call). The agent will transition to this status after their current contact ends.

| Field | Type | Description |
|---|---|---|
| `ARN` | String | ARN of the next agent status |
| `Name` | String | Display name of the upcoming status |
| `EnqueuedTimestamp` | ISO 8601 | When the agent selected the next status |

### Configuration

| Field | Type | Description |
|---|---|---|
| `FirstName` | String | Agent's first name |
| `LastName` | String | Agent's last name |
| `Username` | String | Agent's login username |
| `RoutingProfile` | Object | Current routing profile (see below) |
| `HierarchyGroups` | Object | Agent's position in the hierarchy |
| `Proficiencies` | Array | Language and skill proficiencies |

### RoutingProfile

```json
{
  "ARN": "arn:aws:connect:.../routing-profile/rp-id",
  "Name": "Basic Routing Profile",
  "InboundQueues": [
    {
      "ARN": "arn:aws:connect:.../queue/q-id",
      "Name": "BasicQueue"
    }
  ],
  "DefaultOutboundQueue": {
    "ARN": "arn:aws:connect:.../queue/q-id",
    "Name": "OutboundQueue"
  },
  "Concurrency": [
    {
      "AvailableSlots": 1,
      "Channel": "VOICE",
      "MaximumSlots": 1
    },
    {
      "AvailableSlots": 3,
      "Channel": "CHAT",
      "MaximumSlots": 5
    }
  ]
}
```

| Field | Type | Description |
|---|---|---|
| `ARN` | String | Routing profile ARN |
| `Name` | String | Routing profile name |
| `InboundQueues` | Array | Queues the agent receives contacts from |
| `DefaultOutboundQueue` | Object | Queue used for outbound contacts |
| `Concurrency` | Array | Per-channel slot configuration |

**Concurrency fields:**

| Field | Description |
|---|---|
| `AvailableSlots` | Remaining capacity for this channel |
| `Channel` | `VOICE`, `CHAT`, or `TASK` |
| `MaximumSlots` | Maximum concurrent contacts for this channel |

### Contacts Array

Each element in the `Contacts` array represents an active or recently ended contact:

```json
{
  "ContactId": "contact-uuid",
  "InitialContactId": "initial-contact-uuid",
  "Channel": "VOICE",
  "InitiationMethod": "INBOUND",
  "State": "CONNECTED",
  "StateStartTimestamp": "2026-05-25T14:29:00.000Z",
  "ConnectedToAgentTimestamp": "2026-05-25T14:29:05.000Z",
  "QueueTimestamp": "2026-05-25T14:28:30.000Z",
  "Queue": {
    "ARN": "arn:aws:connect:.../queue/q-id",
    "Name": "Support"
  }
}
```

**Contact State values (10 states):**

| State | Description |
|---|---|
| `INCOMING` | Contact is ringing on the agent's CCP |
| `PENDING` | Contact is pending acceptance (chat/task) |
| `CONNECTING` | Outbound call is connecting |
| `CONNECTED` | Agent and customer are actively connected |
| `CONNECTED_ONHOLD` | Customer is on hold |
| `MISSED` | Agent did not accept the contact in time |
| `PAUSED` | Contact is paused (chat) |
| `REJECTED` | Agent rejected the contact |
| `ERROR` | An error occurred during contact handling |
| `ENDED` | Contact has ended, agent is in ACW |

**InitiationMethod values (12 methods):**

`INBOUND`, `OUTBOUND`, `TRANSFER`, `CALLBACK`, `QUEUE_TRANSFER`, `DISCONNECT`, `MONITOR`, `EXTERNAL_OUTBOUND`, `WEBRTC_API`, `FLOW`, `AGENT_REPLY`, `API`

## Calculating ACW Duration

After-call work (ACW) time can be derived from agent event streams by measuring the time between the contact entering `ENDED` state and the next `STATE_CHANGE` event:

```javascript
// ACW = time from contact ENDED to the next state change (agent becomes Available, etc.)
const acwStart = contact.StateStartTimestamp; // when State became ENDED
const acwEnd = nextStateChangeEvent.EventTimestamp; // when agent changed status

const acwDurationMs = new Date(acwEnd) - new Date(acwStart);
```

The `ENDED` state represents the ACW period -- the agent is completing post-contact work. When the agent sets themselves back to an available status (or another status), the ACW period ends.

## Consumer Patterns

**Real-time agent dashboard:**
```javascript
import { KinesisClient, GetRecordsCommand, GetShardIteratorCommand } from "@aws-sdk/client-kinesis";

// Process agent events from Kinesis
// Filter by EventType to track specific agent activities
// Use CurrentAgentSnapshot.AgentStatus.Type === "ROUTABLE" to count available agents
// Use Contacts[].State === "CONNECTED" to count active contacts
```

**Detecting stale sessions:**
- If a `HEART_BEAT` is not received for an agent within 300 seconds (2.5x the 120s interval), consider the session potentially stale
- `HEART_BEAT` events continue for 1 hour after `LOGOUT`, so a missing heartbeat before logout is more concerning

## Key Considerations

- **Delivery:** Near-real-time, not exactly real-time. Expect sub-second to a few seconds of latency.
- **Ordering:** Events for a single agent are ordered within a shard. Use `AgentARN` as the partition key for per-agent ordering.
- **Duplicates:** At-least-once delivery. Consumer logic should be idempotent.
- **Volume:** One event per state change per agent, plus a heartbeat every 120s per active agent. Plan shard capacity accordingly.
- **Retention:** Configure Kinesis stream retention based on your recovery needs (default 24 hours, configurable up to 365 days).
