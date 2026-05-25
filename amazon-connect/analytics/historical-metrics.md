# Historical Metrics

Amazon Connect provides 80+ historical metrics across 10 categories. These metrics are computed from contact records, agent activity events, conversational analytics, flow/bot interactions, AI agent sessions, evaluation data, and case records.

---

## API

**GetMetricDataV2** — The primary API for querying historical metrics. Replaces the legacy `GetMetricData`.

```
POST /metrics/data
```

### Request Structure

```json
{
  "ResourceArn": "arn:aws:connect:region:account:instance/instance-id",
  "StartTime": "2026-05-01T00:00:00Z",
  "EndTime": "2026-05-02T00:00:00Z",
  "Interval": {
    "TimeZone": "UTC",
    "IntervalPeriod": "DAY"
  },
  "Filters": [
    {
      "FilterKey": "QUEUE",
      "FilterValues": ["queue-id"]
    },
    {
      "FilterKey": "CHANNEL",
      "FilterValues": ["VOICE"]
    }
  ],
  "Groupings": ["QUEUE", "CHANNEL"],
  "Metrics": [
    {
      "Name": "AVG_HANDLE_TIME"
    },
    {
      "Name": "ABANDONMENT_RATE"
    }
  ]
}
```

### Interval Periods

- `FIFTEEN_MINUTES` — 15-minute intervals.
- `THIRTY_MINUTES` — 30-minute intervals.
- `HOUR` — Hourly intervals.
- `DAY` — Daily intervals.
- `WEEK` — Weekly intervals.
- `TOTAL` — Entire time range as a single interval.

### Data Availability

- Data is available approximately 15 minutes after a contact ends.
- Maximum query range: 35 days per request.
- Data retained: up to 24 months.

---

## Category 1: Contact Record Metrics

These metrics are derived directly from contact trace records (CTRs).

| Metric | API Name | Unit | Description |
|---|---|---|---|
| Abandonment Rate | `ABANDONMENT_RATE` | PERCENT | Percentage of queued contacts abandoned before agent answer. |
| After Contact Work Time | `AVG_AFTER_CONTACT_WORK_TIME` | SECONDS | Average time agents spent in ACW after contacts. |
| Agent Interaction Time | `AVG_AGENT_INTERACTION_TIME` | SECONDS | Average time agent was connected to the customer (excludes hold). |
| Agent Interaction and Hold Time | `AVG_AGENT_INTERACTION_AND_HOLD_TIME` | SECONDS | Average time agent was connected including hold time. |
| Average Active Time | `AVG_ACTIVE_TIME` | SECONDS | Average active handling time across all contacts. |
| Contact Duration | `AVG_CONTACT_DURATION` | SECONDS | Average total contact duration from initiation to disconnect. |
| Hold Time | `AVG_HOLD_TIME` | SECONDS | Average time customer spent on hold. |
| Handle Time | `AVG_HANDLE_TIME` | SECONDS | Average handle time = agent interaction time + hold time + ACW time. |
| Queue Wait Time | `AVG_QUEUE_ANSWER_TIME` | SECONDS | Average time contacts waited in queue before being answered (service level). |
| Contacts Handled | `CONTACTS_HANDLED` | COUNT | Total contacts answered by an agent. |
| Contacts Queued | `CONTACTS_QUEUED` | COUNT | Total contacts that entered a queue. |
| Contacts Abandoned | `CONTACTS_ABANDONED` | COUNT | Total contacts abandoned while in queue. |
| Contacts Transferred In | `CONTACTS_TRANSFERRED_IN` | COUNT | Contacts transferred into a queue from another queue or agent. |
| Contacts Transferred Out | `CONTACTS_TRANSFERRED_OUT` | COUNT | Contacts transferred out from a queue to another queue or agent. |
| Contacts Hold Disconnect | `CONTACTS_HOLD_DISCONNECT` | COUNT | Contacts disconnected while on hold. |
| Service Level | `SERVICE_LEVEL` | PERCENT | Percentage of contacts answered within X seconds (configurable threshold). |
| Agent Messages | `AVG_AGENT_MESSAGES` | COUNT | Average number of messages sent by agent (chat). |
| Customer Messages | `AVG_CUSTOMER_MESSAGES` | COUNT | Average number of messages sent by customer (chat). |
| Agent Response Time | `AVG_AGENT_RESPONSE_TIME` | SECONDS | Average agent response time in chat. |
| Customer Response Time | `AVG_CUSTOMER_RESPONSE_TIME` | SECONDS | Average customer response time in chat. |
| Max Queue Wait Time | `MAX_QUEUED_TIME` | SECONDS | Maximum time any contact waited in queue. |
| Contacts Handled Incoming | `CONTACTS_HANDLED_INCOMING` | COUNT | Inbound contacts handled. |
| Contacts Handled Outbound | `CONTACTS_HANDLED_OUTBOUND` | COUNT | Outbound contacts handled. |
| Callback Contacts Handled | `CALLBACK_CONTACTS_HANDLED` | COUNT | Callback contacts handled. |
| API Contacts Handled | `API_CONTACTS_HANDLED` | COUNT | Contacts initiated via API that were handled. |

---

## Category 2: Agent Activity Metrics

These metrics are derived from agent activity events, not individual contact records.

| Metric | API Name | Unit | Description |
|---|---|---|---|
| Agent Adherence | `AGENT_ADHERENCE` | PERCENT | Percentage of time agent was in expected status per schedule. |
| Agent Answer Rate | `AGENT_ANSWER_RATE` | PERCENT | Percentage of offered contacts that the agent accepted. |
| Agent Idle Time | `AVG_AGENT_IDLE_TIME` | SECONDS | Average time agent was in Available status with no contacts. |
| Agent Non-Productive Time | `AVG_NON_PRODUCTIVE_TIME` | SECONDS | Average time in custom (non-productive) statuses. |
| Agent Connecting Time | `AVG_AGENT_CONNECTING_TIME` | SECONDS | Average time between contact being offered and agent accepting. |
| Agent Occupancy | `AGENT_OCCUPANCY` | PERCENT | Percentage of time agent was active on contacts vs. available. |
| Agent Schedule Adherence | `AGENT_SCHEDULE_ADHERENCE` | PERCENT | How closely agent followed their scheduled activities. |
| Contacts Missed | `CONTACTS_MISSED` | COUNT | Contacts offered to agent but not answered within timeout. |
| Contacts Rejected | `CONTACTS_REJECTED` | COUNT | Contacts explicitly rejected by agent. |

---

## Category 3: Conversational Analytics Metrics (Contact Lens)

These metrics require Contact Lens to be enabled.

| Metric | API Name | Unit | Description |
|---|---|---|---|
| Agent Talk Time Percent | `AVG_AGENT_TALK_TIME_PERCENT` | PERCENT | Percentage of conversation where agent was talking. |
| Customer Talk Time Percent | `AVG_CUSTOMER_TALK_TIME_PERCENT` | PERCENT | Percentage of conversation where customer was talking. |
| Non-Talk Time Percent | `AVG_NON_TALK_TIME_PERCENT` | PERCENT | Percentage of conversation with silence. |
| Agent Greeting Time | `AVG_GREETING_TIME_AGENT` | SECONDS | Average time from conversation start to agent's first utterance. |
| Interruptions (Agent) | `AVG_INTERRUPTIONS_AGENT` | COUNT | Average number of times agent interrupted the customer. |
| Interruptions (Customer) | `AVG_INTERRUPTIONS_CUSTOMER` | COUNT | Average number of times customer interrupted the agent. |
| Conversation Duration | `AVG_CONVERSATION_DURATION` | SECONDS | Average duration of the actual conversation (excludes hold, IVR). |
| Average Sentiment Score (Customer) | `AVG_CUSTOMER_SENTIMENT` | SCORE | Average customer sentiment score (-5 to +5). |
| Average Sentiment Score (Agent) | `AVG_AGENT_SENTIMENT` | SCORE | Average agent sentiment score (-5 to +5). |

---

## Category 4: Flow and Bot Metrics

| Metric | API Name | Unit | Description |
|---|---|---|---|
| Bot Conversation Time | `AVG_BOT_CONVERSATION_TIME` | SECONDS | Average time spent in Lex bot conversation within a flow. |
| Bot Conversation Turns | `AVG_BOT_CONVERSATION_TURNS` | COUNT | Average number of turns in Lex bot conversation. |
| Flow Time | `AVG_FLOW_TIME` | SECONDS | Average time contacts spent in contact flows (IVR). |
| Contacts Flow Out | `CONTACTS_FLOW_OUT` | COUNT | Contacts that exited a flow without being queued or handled. |

---

## Category 5: AI Agent Metrics

Metrics for Amazon Connect AI Agents (Q in Connect, autonomous agents).

| Metric | API Name | Unit | Description |
|---|---|---|---|
| AI Agent Invocations | `AI_AGENT_INVOCATIONS` | COUNT | Number of times AI agent was invoked. |
| AI Agent Success Rate | `AI_AGENT_SUCCESS_RATE` | PERCENT | Percentage of AI agent invocations that completed successfully. |
| AI Agent Response Helpful | `AI_AGENT_RESPONSE_HELPFUL` | COUNT | Number of AI responses rated helpful by agent/customer. |
| AI Agent Response Not Helpful | `AI_AGENT_RESPONSE_NOT_HELPFUL` | COUNT | Number of AI responses rated not helpful. |
| AI Agent Conversation Turns | `AI_AGENT_CONVERSATION_TURNS` | COUNT | Average number of conversation turns in AI agent sessions. |

---

## Category 6: AI Session Metrics

| Metric | API Name | Unit | Description |
|---|---|---|---|
| AI Session Handoffs | `AI_SESSION_HANDOFFS` | COUNT | Number of AI sessions that were handed off to a human agent. |
| AI Session Handoff Rate | `AI_SESSION_HANDOFF_RATE` | PERCENT | Percentage of AI sessions resulting in handoff. |
| AI Completeness Score | `AI_COMPLETENESS_SCORE` | SCORE | How completely the AI agent addressed the customer's question. |
| AI Faithfulness Score | `AI_FAITHFULNESS_SCORE` | SCORE | How accurately the AI agent's response reflected source material. |
| AI Goal Success Score | `AI_GOAL_SUCCESS_SCORE` | SCORE | How successfully the AI agent achieved the interaction goal. |

---

## Category 7: AI Prompt and Tool Metrics

| Metric | API Name | Unit | Description |
|---|---|---|---|
| AI Prompt Invocations | `AI_PROMPT_INVOCATIONS` | COUNT | Number of AI prompt invocations. |
| AI Prompt Latency | `AVG_AI_PROMPT_LATENCY` | MILLISECONDS | Average latency of AI prompt responses. |
| AI Tool Invocations | `AI_TOOL_INVOCATIONS` | COUNT | Number of AI tool use invocations. |
| AI Tool Latency | `AVG_AI_TOOL_LATENCY` | MILLISECONDS | Average latency of AI tool invocations. |
| AI Tool Accuracy | `AI_TOOL_ACCURACY` | PERCENT | Accuracy rate of AI tool invocations. |

---

## Category 8: Case Metrics

Metrics from Amazon Connect Cases.

| Metric | API Name | Unit | Description |
|---|---|---|---|
| Case Resolution Time | `AVG_CASE_RESOLUTION_TIME` | SECONDS | Average time from case creation to resolution. |
| Contacts Per Case | `AVG_CONTACTS_PER_CASE` | COUNT | Average number of contacts associated with each case. |
| Cases Created | `CASES_CREATED` | COUNT | Number of cases created in the period. |
| Cases Resolved | `CASES_RESOLVED` | COUNT | Number of cases resolved in the period. |

---

## Category 9: Evaluation Metrics

| Metric | API Name | Unit | Description |
|---|---|---|---|
| Automatic Fails Percent | `AUTOMATIC_FAILS_PERCENT` | PERCENT | Percentage of evaluations where the agent automatically failed due to a critical section. |
| Average Evaluation Score | `AVG_EVALUATION_SCORE` | PERCENT | Average evaluation score across all completed evaluations. |

---

## Category 10: Routing and Capacity

| Metric | API Name | Unit | Description |
|---|---|---|---|
| Contacts Transferred In from Queue | `CONTACTS_TRANSFERRED_IN_FROM_QUEUE` | COUNT | Contacts transferred in from another queue. |
| Contacts Transferred Out from Queue | `CONTACTS_TRANSFERRED_OUT_FROM_QUEUE` | COUNT | Contacts transferred out to another queue. |
| Contacts Transferred In by Agent | `CONTACTS_TRANSFERRED_IN_BY_AGENT` | COUNT | Contacts transferred in by a specific agent. |
| Contacts Transferred Out by Agent | `CONTACTS_TRANSFERRED_OUT_BY_AGENT` | COUNT | Contacts transferred out by a specific agent. |
| Contacts Put on Hold | `CONTACTS_PUT_ON_HOLD` | COUNT | Number of contacts put on hold. |
| Hold Count | `AVG_HOLDS` | COUNT | Average number of holds per contact. |

---

## Groupings

Historical metrics can be grouped by:

| Grouping | Description |
|---|---|
| `QUEUE` | Results per queue. |
| `CHANNEL` | Results per channel (VOICE, CHAT, TASK, EMAIL). |
| `AGENT` | Results per individual agent. |
| `ROUTING_PROFILE` | Results per routing profile. |
| `AGENT_HIERARCHY_LEVEL_ONE` through `LEVEL_FIVE` | Results per agent hierarchy group at each level. |
| `FEATURE` | Results per feature (e.g., Contact Lens). |
| `CONTACT_FLOW` | Results per contact flow. |
| `CASE_TEMPLATE` | Results per case template. |

---

## Filters

| Filter Key | Values |
|---|---|
| `QUEUE` | Queue IDs |
| `CHANNEL` | VOICE, CHAT, TASK, EMAIL |
| `ROUTING_PROFILE` | Routing profile IDs |
| `AGENT` | Agent IDs |
| `AGENT_HIERARCHY_LEVEL_ONE` through `FIVE` | Hierarchy group IDs |
| `FEATURE` | VOICE_ANALYTICS (Contact Lens) |
| `INITIATION_METHOD` | INBOUND, OUTBOUND, TRANSFER, CALLBACK, API, QUEUE_TRANSFER, EXTERNAL_OUTBOUND |
| `DISCONNECT_REASON` | CUSTOMER_DISCONNECT, AGENT_DISCONNECT, THIRD_PARTY_DISCONNECT, TELECOM_PROBLEM, CONTACT_FLOW_DISCONNECT, OTHER, EXPIRED |
| `CONTACT_FLOW_TYPE` | Various flow types |

---

## Statistic Types

Each metric can be requested with different statistics:

| Statistic | Description |
|---|---|
| `SUM` | Total across all contacts in the period. |
| `AVG` | Average across all contacts in the period. |
| `MIN` | Minimum value in the period. |
| `MAX` | Maximum value in the period. |

Not all statistics apply to all metrics. COUNT metrics typically use SUM. Duration metrics support all four.

---

## Common Patterns

### Daily Queue Performance Report

Query `CONTACTS_HANDLED`, `CONTACTS_ABANDONED`, `AVG_QUEUE_ANSWER_TIME`, `SERVICE_LEVEL`, `AVG_HANDLE_TIME` grouped by `QUEUE` with `DAY` interval.

### Agent Scorecard

Query `AGENT_ANSWER_RATE`, `AVG_HANDLE_TIME`, `AVG_AFTER_CONTACT_WORK_TIME`, `AGENT_OCCUPANCY`, `CONTACTS_HANDLED` grouped by `AGENT` with `DAY` interval.

### Trend Analysis

Query key metrics with `FIFTEEN_MINUTES` or `HOUR` interval over multiple days to identify peak hours and staffing gaps.
