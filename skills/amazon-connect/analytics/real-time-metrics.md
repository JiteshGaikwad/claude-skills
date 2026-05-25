# Real-Time Metrics

Real-time metrics in Amazon Connect provide a live view of contact center operations. They reflect the current state of agents, queues, and routing profiles and update continuously.

---

## API

**GetCurrentMetricData** — Returns current metric data for the specified filters (queues, channels, routing profiles).

```
POST /metrics/current/{InstanceId}
```

### Request Structure

```json
{
  "Filters": {
    "Queues": ["queue-id-1", "queue-id-2"],
    "Channels": ["VOICE", "CHAT"],
    "RoutingProfiles": ["rp-id-1"]
  },
  "Groupings": ["QUEUE", "CHANNEL"],
  "CurrentMetrics": [
    {
      "Name": "AGENTS_AVAILABLE",
      "Unit": "COUNT"
    }
  ],
  "MaxResults": 100,
  "NextToken": "..."
}
```

### Groupings

- `QUEUE` — Group results by queue.
- `CHANNEL` — Group results by channel (VOICE, CHAT, TASK, EMAIL).
- `ROUTING_PROFILE` — Group results by routing profile.
- `INSTANCE` — Aggregate across the instance.

---

## Agent Status Metrics

| Metric Name | API Identifier | Unit | Description |
|---|---|---|---|
| Agents Available | `AGENTS_AVAILABLE` | COUNT | Agents in Available status and not on a contact. |
| Agents on Contact | `AGENTS_ON_CONTACT` | COUNT | Agents currently handling at least one contact. |
| Agents Non-Productive | `AGENTS_NON_PRODUCTIVE` | COUNT | Agents in a custom (non-productive) status. |
| Agents After Contact Work | `AGENTS_AFTER_CONTACT_WORK` | COUNT | Agents in After Contact Work state. |
| Agents Error | `AGENTS_ERROR` | COUNT | Agents in Error status (e.g., missed a contact). |
| Agents Online | `AGENTS_ONLINE` | COUNT | Agents who are not in Offline status. |
| Agents Staffed | `AGENTS_STAFFED` | COUNT | Agents who are online or in ACW. |
| Agents Count | N/A (derived) | COUNT | Total agents logged in (all statuses). Not a direct API metric; derive from summing status counts or use agent event stream. |

---

## Slot Metrics

| Metric Name | API Identifier | Unit | Description |
|---|---|---|---|
| Active Slots | `SLOTS_ACTIVE` | COUNT | Contact slots currently occupied across all agents for the filtered queues/channels. |
| Available Slots | `SLOTS_AVAILABLE` | COUNT | Contact slots available (agents available multiplied by their concurrency minus active slots). |

---

## Queue Metrics

| Metric Name | API Identifier | Unit | Description |
|---|---|---|---|
| Contacts in Queue | `CONTACTS_IN_QUEUE` | COUNT | Number of contacts currently waiting in queue. |
| Oldest Contact in Queue | `OLDEST_CONTACT_AGE` | SECONDS | Wait time of the oldest contact currently in queue. |
| Contacts Scheduled | `CONTACTS_SCHEDULED` | COUNT | Number of contacts in queue that are scheduled callbacks. |

---

## Abandonment

| Metric Name | API Identifier | Unit | Description |
|---|---|---|---|
| Abandonment Rate | Derived | PERCENT | Percentage of queued contacts that disconnected before being answered. Not a direct API metric in GetCurrentMetricData; available as a historical metric or calculated from real-time queue stats. |

For real-time abandonment tracking, use the combination of Contacts in Queue changes and the agent event stream, or reference the historical metric `ABANDONMENT_RATE` in `GetMetricDataV2` with a short time window.

---

## Agent Activity Indicator

The agent activity indicator reflects the agent's current state with a priority-based logic. When an agent has multiple concurrent activities, the highest-priority state is displayed.

### Priority Order (Highest to Lowest)

| Priority | State | Description |
|---|---|---|
| 1 | **Error** | Agent encountered a system error or failed to accept a contact. |
| 2 | **Missed** | Agent was offered a contact but did not answer within the timeout. |
| 3 | **Rejected** | Agent explicitly rejected an offered contact. |
| 4 | **On Contact** | Agent is actively connected to a contact (voice, chat, task, or email). |
| 5 | **After Contact Work** | Agent is in ACW for a recently completed contact. |
| 6 | **Incoming** | Agent has a contact being offered/ringing but has not yet accepted. |
| 7 | **Custom Status** | Agent is in a custom agent status (break, lunch, training, etc.). |
| 8 | **Offline** | Agent is logged in but set to Offline status. |

### Multi-Channel Behavior

An agent handling concurrent contacts (e.g., 2 chats + 1 task) shows the highest-priority state across all channels. If one chat is ringing (Incoming) while the agent is connected on another chat (On Contact), the indicator shows **On Contact** because priority 4 > priority 6.

---

## Refresh Behavior

- Real-time metrics in the console refresh approximately every **15 seconds**.
- API calls to `GetCurrentMetricData` return point-in-time snapshots.
- There is no push/streaming mechanism for real-time metrics; polling is required.
- API throttling: default 5 TPS for `GetCurrentMetricData`.

---

## Filters

Real-time metrics support filtering by:

| Filter | Description |
|---|---|
| **Queues** | One or more queue IDs. |
| **Channels** | VOICE, CHAT, TASK, EMAIL. |
| **Routing Profiles** | One or more routing profile IDs. |
| **Agent Hierarchy Groups** | Filter by agent hierarchy group. |

---

## Console Views

### Queue Dashboard

Shows per-queue real-time metrics including contacts in queue, agents available, agents on contact, oldest contact age, and service level.

### Agent Activity

Shows each agent's current status, duration in status, active contacts, and the agent activity indicator with the priority logic described above.

### Routing Profile

Aggregates real-time metrics by routing profile to show capacity utilization.

---

## Agent Event Stream

For event-driven real-time agent state tracking (rather than polling), use the Amazon Connect Agent Event Stream. This publishes agent state changes to a Kinesis stream in near real-time.

Events include:
- Agent login/logout
- Status changes (Available, Offline, custom statuses)
- Contact state transitions (connecting, connected, ACW, ended)

This is the recommended approach for building custom real-time dashboards that need sub-second latency.

---

## Common Patterns

### Real-Time Wallboard

Poll `GetCurrentMetricData` every 15-30 seconds with grouping by QUEUE and CHANNEL. Display:
- Contacts in queue per queue
- Agents available vs. agents on contact
- Oldest contact age (highlight if > SLA threshold)
- Service level

### Agent Roster

Use the Agent Event Stream for live agent status, or poll `GetCurrentMetricData` with `AGENTS_AVAILABLE`, `AGENTS_ON_CONTACT`, `AGENTS_NON_PRODUCTIVE`, `AGENTS_AFTER_CONTACT_WORK`, `AGENTS_ERROR` grouped by ROUTING_PROFILE.

### Capacity Planning Alert

Monitor `SLOTS_AVAILABLE`. When available slots approach zero for a queue/channel combination, trigger an alert via CloudWatch or EventBridge.
