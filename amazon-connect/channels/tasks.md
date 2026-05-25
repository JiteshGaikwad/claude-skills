# Tasks Channel

Amazon Connect Tasks let you create, prioritize, assign, and track work items alongside voice, chat, and email contacts. Tasks route through the same queues and routing profiles, giving agents a unified workspace for all work.

## Overview

Tasks represent units of work that need to be completed — follow-ups, case reviews, data entry, callbacks, or any action that does not require a real-time conversation. They appear in the agent's CCP alongside other contacts.

**Key properties:**
- Tasks are contacts — they have a ContactId, appear in contact records, and generate CTR data
- Routed via the same routing profiles and queues as voice/chat/email
- Priority and routing logic apply identically
- Agents accept, work on, and complete tasks through the CCP

## Creating Tasks

### Manual Creation (Agent CCP)

Agents create tasks directly from the Contact Control Panel.

- Click "Create task" in the CCP
- Fill in task name, description, and any template-defined fields
- Assign to a queue (or self-assign)
- Set priority if allowed
- Optionally link to a previous contact (e.g., "follow up on call #ABC")

### Automatic Creation via Contact Flows

Contact flows can create tasks as part of their logic.

- Use the "Create task" block in a contact flow
- Populate task fields from contact attributes, Lambda results, or static values
- Trigger tasks based on flow conditions (e.g., create a follow-up task when a call ends with an unresolved issue)
- Chain multiple task creations in a single flow

### Automatic Creation via Rules

Connect Rules can automatically generate tasks based on events.

- Contact Lens detects a specific phrase or sentiment and triggers a task
- A contact sits in queue beyond a threshold and a supervisor task is created
- An agent evaluation score drops below a threshold and a coaching task is generated
- Rules are configured in the Connect console under "Rules"

### Programmatic Creation via APIs

Create tasks from external systems — CRM, ticketing, scheduling, or custom applications.

```javascript
import { ConnectClient, StartTaskContactCommand } from "@aws-sdk/client-connect";

const client = new ConnectClient({ region: "us-east-1" });

const response = await client.send(new StartTaskContactCommand({
  InstanceId: instanceId,
  ContactFlowId: contactFlowId,
  Name: "Follow up on billing dispute",
  Description: "Customer John Doe reported incorrect charge of $49.99 on invoice #12345. Verify and process refund if valid.",
  // Optional: link to a previous contact
  PreviousContactId: previousContactId,
  // Optional: reference to a related contact
  RelatedContactId: relatedContactId,
  // Task attributes
  Attributes: {
    customerId: "C-12345",
    issueType: "billing_dispute",
    amount: "49.99",
    urgency: "high",
  },
  // Optional: schedule for later
  ScheduledTime: new Date("2026-05-26T09:00:00Z"),
  // Optional: use a task template
  TaskTemplateId: taskTemplateId,
}));

// response.ContactId — the task's contact ID
```

## Task Templates

Templates define the structure and fields for a task, ensuring consistency and completeness.

**Template fields:**
- Name (required, always present)
- Description
- Custom fields: text, number, date, single-select, multi-select, boolean, email, URL, phone
- Each field can be marked as required or optional
- Default values can be set per field

**Creating a template:**

```javascript
import { ConnectClient, CreateTaskTemplateCommand } from "@aws-sdk/client-connect";

const client = new ConnectClient({ region: "us-east-1" });

const response = await client.send(new CreateTaskTemplateCommand({
  InstanceId: instanceId,
  Name: "Billing Dispute Follow-Up",
  Description: "Template for billing dispute resolution tasks",
  Status: "ACTIVE",
  Fields: [
    {
      Id: { Name: "customerName" },
      Type: "TEXT",
      Description: "Customer's full name",
      SingleSelectOptions: undefined,
    },
    {
      Id: { Name: "disputeAmount" },
      Type: "NUMBER",
      Description: "Disputed amount in dollars",
    },
    {
      Id: { Name: "invoiceNumber" },
      Type: "TEXT",
      Description: "Related invoice number",
    },
    {
      Id: { Name: "resolution" },
      Type: "SINGLE_SELECT",
      Description: "Resolution action",
      SingleSelectOptions: ["Refund", "Credit", "No Action", "Escalate"],
    },
    {
      Id: { Name: "dueDate" },
      Type: "DATE_TIME",
      Description: "Resolution deadline",
    },
  ],
  Defaults: {
    DefaultFieldValues: [
      {
        Id: { Name: "resolution" },
        DefaultValue: "No Action",
      },
    ],
  },
}));

// response.Id — the template ID
```

**Updating a template:**

```javascript
await client.send(new UpdateTaskTemplateCommand({
  InstanceId: instanceId,
  TaskTemplateId: templateId,
  Name: "Billing Dispute Follow-Up v2",
  Status: "ACTIVE",
  Fields: [
    // Updated field list
  ],
}));
```

**Important:**
- Templates are versioned — updates create a new version
- Active tasks using an older template version retain their original field structure
- Templates can be set to `ACTIVE` or `INACTIVE` status
- Inactive templates cannot be used to create new tasks but existing tasks are unaffected

## Priority and Routing

Tasks are routed through the same system as all other contact types.

**Priority:**
- Tasks can have a priority value (lower number = higher priority)
- Priority determines order in queue — higher-priority tasks are offered to agents first
- Priority can be set at creation time or modified in the contact flow

**Routing:**
- Tasks are assigned to queues and routed via routing profiles
- An agent's routing profile determines which queues they receive tasks from
- Task concurrency is configured separately from voice/chat concurrency in the routing profile
- Example: an agent might handle 1 voice call + 3 chats + 2 tasks simultaneously

**Queue behavior:**
- Tasks wait in queue like any other contact
- Queue metrics (oldest contact, contacts in queue) include tasks
- Supervisors can see task queue depth in real-time metrics

## Task Expiry

Tasks have a configurable expiration window.

| Setting | Value |
|---------|-------|
| Default expiry | 7 days |
| Minimum expiry | Not specified (minutes) |
| Maximum expiry | 90 days |

- Expired tasks are automatically closed with a disposition of "expired"
- Expiry is set at task creation time via the `TaskExpirationInSeconds` parameter or in the contact flow
- Expired tasks still appear in contact records and historical metrics
- Set expiry based on the business SLA for the task type

## Pausing and Resuming Tasks

Agents can pause a task and return to it later.

**How it works:**
- Agent accepts a task and begins working
- Agent pauses the task (e.g., waiting for information from another team)
- Task returns to a paused state — the agent's slot is freed for other work
- Agent (or another agent) resumes the task later
- Full context is preserved across pause/resume cycles

**Use cases:**
- Waiting for customer callback
- Pending approval from a manager
- Blocked on information from an external system
- Multi-day tasks that require incremental progress

**Behavioral details:**
- Paused tasks do not count against the agent's concurrency limit
- Paused tasks can be reassigned to a different agent or queue
- The pause timestamp and resume timestamp are recorded in the contact record

## Follow-Up Automation

Tasks enable structured follow-up workflows.

**Patterns:**
- A voice call ends and a flow automatically creates a follow-up task linked to that contact
- A task is completed and a rule creates a subsequent task (task chaining)
- A scheduled task fires at a specific date/time for proactive outreach
- An SLA breach triggers an escalation task to a supervisor queue

**Scheduled tasks:**
- Set the `ScheduledTime` parameter when creating a task via API
- The task will not be routed to an agent until the scheduled time
- Useful for callbacks, reminders, and time-sensitive follow-ups

```javascript
await client.send(new StartTaskContactCommand({
  InstanceId: instanceId,
  ContactFlowId: contactFlowId,
  Name: "Scheduled callback - Jane Doe",
  ScheduledTime: new Date("2026-05-26T14:00:00Z"), // Route at 2 PM UTC
  Attributes: {
    callbackNumber: "+14155551234",
    reason: "Follow up on claim #789",
  },
}));
```

## Task APIs Summary

| API | Purpose |
|-----|---------|
| `StartTaskContact` | Create a new task contact |
| `CreateTaskTemplate` | Define a new task template with custom fields |
| `UpdateTaskTemplate` | Modify an existing task template |
| `GetTaskTemplate` | Retrieve a task template definition |
| `ListTaskTemplates` | List all task templates in an instance |
| `DeleteTaskTemplate` | Remove a task template |
| `TransferContact` | Transfer a task to another queue or agent |
| `StopContact` | Complete/close a task |
| `UpdateContact` | Update task attributes or description |
| `PauseContact` | Pause an active task |
| `ResumeContact` | Resume a paused task |

## Contact Flow Integration

Task-specific flow blocks and behaviors:

| Block/Feature | Purpose |
|---------------|---------|
| Create task | Generate a new task from within a flow |
| Check contact attributes | Branch logic based on task attributes |
| Set contact attributes | Add/modify task metadata |
| Invoke Lambda | Enrich task data from external systems |
| Transfer to queue | Route task to a specific queue |
| Set working queue | Change the task's target queue |

## Metrics and Reporting

Tasks appear in both real-time and historical metrics.

**Real-time metrics:**
- Tasks in queue
- Agents handling tasks
- Oldest task in queue
- Task acceptance rate

**Historical metrics:**
- Tasks created, completed, expired, transferred
- Average handle time for tasks
- Task-specific agent performance
- Filter by queue, agent, routing profile, or time range

**Contact records:**
- Every task generates a Contact Trace Record (CTR)
- CTR includes: creation time, assignment time, completion time, agent, queue, attributes, linked contacts
- CTRs available via Kinesis Data Stream or S3 export

## Key Considerations

- **Not real-time:** Tasks are asynchronous work items, not live conversations
- **Linking:** Tasks can be linked to previous contacts via `PreviousContactId` for full context chains
- **No customer participant:** Tasks do not have a customer-facing component — the agent works on them independently
- **Concurrency:** Task slots are separate from voice/chat slots in routing profile configuration
- **Automation:** Combine tasks with Rules, Contact Lens, and Lambda for powerful workflow automation
- **SLA tracking:** Use scheduled tasks and expiry to enforce business SLAs programmatically
