# Dashboards and Reports

Amazon Connect provides built-in dashboards, configurable reports, and custom metric primitives for operational visibility.

---

## Built-In Dashboards

### Conversational Analytics Dashboard

Powered by Contact Lens. Displays:
- Sentiment trends over time
- Category distribution
- Theme detection results
- Talk time breakdowns
- Key highlights aggregation
- PII detection counts

### Queue Performance Dashboard

Real-time and historical queue metrics:
- Contacts in queue
- Service level trending
- Average speed of answer
- Abandonment rate
- Contacts handled vs. queued

### Agent Performance Dashboard

Per-agent and aggregate agent metrics:
- Occupancy
- Handle time trends
- Answer rate
- ACW duration
- Contacts handled by channel
- Adherence (if scheduling is enabled)

### AI Agent Performance Dashboard

Metrics for AI-powered autonomous agents:
- AI agent invocation counts
- Success rate trends
- Handoff rate
- Completeness and faithfulness scores
- Customer satisfaction correlation
- Response helpfulness ratings

---

## Real-Time Metrics Reports

Real-time metrics reports display the current state of the contact center.

### Predefined Reports

- **Queues** — Real-time queue metrics (contacts in queue, agents available, oldest contact).
- **Agents** — Agent status roster with activity indicators.
- **Routing profiles** — Capacity metrics by routing profile.

### Customization

- Add or remove columns (metrics).
- Set column sort order.
- Configure refresh interval (default 15 seconds).
- Apply filters by queue, routing profile, agent hierarchy.
- Set thresholds for visual alerts (color coding when metrics exceed limits).

---

## Historical Metrics Reports

Historical reports query past data and can be scheduled.

### Predefined Reports

- **Contact metrics** — Contacts handled, abandoned, hold time, handle time, etc.
- **Agent metrics** — Agent activity, occupancy, answer rate, NPT.
- **Queue metrics** — Service level, wait time, abandonment.

### Scheduling

- Schedule reports to run at specific intervals (daily, weekly, monthly).
- Output to S3 in CSV format.
- Reports include the configured time range, groupings, and filters.

---

## Login/Logout Reports

Track agent login and logout events:
- Login timestamp
- Logout timestamp
- Duration of session
- Agent hierarchy group at time of login

Data sourced from agent event stream. Available in historical reports with agent grouping.

---

## Save, Share, and Publish Reports

| Action | Description |
|---|---|
| **Save** | Save a report configuration with a name for future use. Saved to your user profile. |
| **Share** | Share a saved report with other users in the same instance. |
| **Publish** | Publish a report as read-only to other users. Published reports cannot be modified by recipients. |
| **Read-only** | Recipients of published reports can view but not edit the report configuration. |

---

## Custom Metric Primitives

Amazon Connect allows you to define custom metrics by combining built-in primitives with arithmetic operations.

### Primitive Categories

Custom metrics draw from 40+ primitives organized into 4 categories:

#### 1. Contact Primitives

Metrics computed from completed contact records:
- Contacts handled
- Contacts abandoned
- Contacts queued
- Contact duration
- Handle time
- Hold time
- ACW time
- Agent interaction time
- Queue wait time
- Contacts transferred (in/out)
- Service level (configurable threshold)

#### 2. Agent Primitives

Metrics computed from agent activity:
- Agent idle time
- Agent non-productive time
- Agent occupancy
- Agent answer rate
- Contacts missed
- Contacts rejected

#### 3. Current Contact Primitives

Metrics computed from contacts currently in progress:
- Contacts in queue (current)
- Active contacts (current)
- Oldest contact age

#### 4. Current Agent Primitives

Metrics computed from current agent state:
- Agents available (current)
- Agents on contact (current)
- Agents in ACW (current)
- Agents in NPT (current)
- Agents in error (current)

### Supported Statistics

| Statistic | Description |
|---|---|
| `SUM` | Sum of values across all contacts/agents in the period. |
| `AVG` | Average value across all contacts/agents in the period. |
| `MIN` | Minimum value in the period. |
| `MAX` | Maximum value in the period. |

---

## Metric-Level Filters

Each metric within a report or custom metric can be filtered independently.

| Filter | Values | Description |
|---|---|---|
| **Initiation Method** | INBOUND, OUTBOUND, TRANSFER, CALLBACK, API, QUEUE_TRANSFER, EXTERNAL_OUTBOUND | How the contact was initiated. |
| **Disconnect Reason** | CUSTOMER_DISCONNECT, AGENT_DISCONNECT, THIRD_PARTY_DISCONNECT, TELECOM_PROBLEM, CONTACT_FLOW_DISCONNECT, OTHER, EXPIRED | Why the contact ended. |
| **Channel** | VOICE, CHAT, TASK, EMAIL | Contact channel. |
| **Subtype** | connect:SMS, connect:Telephony, connect:WebRTC, etc. | Channel subtype for more specific filtering. |
| **is_abandoned** | true, false | Whether the contact was abandoned in queue. |
| **is_handled** | true, false | Whether the contact was handled by an agent. |

---

## Groupings

Reports can be grouped by one or more dimensions:

| Grouping | Description |
|---|---|
| **Agent** | Per-agent breakdown. |
| **Queue** | Per-queue breakdown. |
| **Channel** | Per-channel breakdown (VOICE, CHAT, TASK, EMAIL). |
| **Routing Profile** | Per-routing-profile breakdown. |
| **Hierarchy Level 1-5** | Per-hierarchy-group breakdown at each level. |
| **Subtype** | Per-channel-subtype breakdown. |

---

## Arithmetic Rules for Custom Metrics

When combining primitives into custom metrics, these rules apply:

### Rule 1: Same Category

All primitives in a custom metric must come from the same category. You cannot mix Contact primitives with Agent primitives in a single custom metric.

### Rule 2: Consistent Filters Within Statistic

All primitives within a single statistic operation must use the same metric-level filters. You can have different filters across different statistics in the same custom metric.

### Rule 3: Maximum 5 Components

A custom metric can contain a maximum of **5 arithmetic components** (primitives or sub-expressions).

### Rule 4: Maximum 10 Elements Per Statistic

Each statistic within a custom metric can reference a maximum of **10 elements** (primitives, constants, operators).

### Rule 5: Supported Operators

- Addition (+)
- Subtraction (-)
- Multiplication (*)
- Division (/)

Division by zero returns null (no error).

### Example

```
Custom Metric: "Transfer Rate"
= SUM(Contacts Transferred Out) / SUM(Contacts Handled) * 100
Category: Contact primitives
Filters: Channel = VOICE
```

---

## Predefined Attributes in Dashboards

Dashboards support predefined contact attributes for additional filtering and display:

- Queue name
- Agent name / username
- Channel
- Initiation method
- Disconnect reason
- Contact flow name
- Routing profile name
- Agent hierarchy (all levels)
- Contact Lens categories
- Customer endpoint (phone number)

---

## Access Control

### Hierarchy-Based Access Control

- Agents see only their own metrics.
- Supervisors see metrics for agents in their hierarchy group and below.
- Admins see all metrics across the instance.

This is controlled via security profiles and agent hierarchy group assignments.

### Tag-Based Access Control

- Resources (queues, routing profiles, agents) can be tagged.
- Security profiles can include tag-based conditions.
- Users only see metrics for resources matching their tag-based access.

Example: Tag queues with `Department:Sales` and restrict a security profile to only see resources with that tag.

---

## Change Agent Status from Dashboard

Supervisors can change an agent's status directly from the real-time metrics dashboard:

- Click on an agent in the agent roster.
- Select a new status from the dropdown (Available, Offline, any custom status).
- The agent's CCP updates immediately to reflect the new status.

This requires the `Agent status - change` permission in the supervisor's security profile.

---

## Report Data Export

| Method | Format | Description |
|---|---|---|
| **Download** | CSV | Download current report view as CSV. |
| **Scheduled export** | CSV | Scheduled reports exported to S3. |
| **API** | JSON | `GetMetricDataV2` returns JSON results for programmatic consumption. |
| **Data lake** | Parquet | Contact records and analytics available via Athena in the analytics data lake. |
