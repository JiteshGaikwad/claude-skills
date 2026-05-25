# Step-by-Step Guides

Step-by-step guides are no-code, flow-designed UI workflows that surface inside the Amazon Connect agent workspace. They replace tribal knowledge and paper runbooks with structured, interactive screens that walk agents through processes.

---

## Overview

Guides are built entirely in the contact flow designer -- no frontend code required. They use the **Show view** block to render UI forms, display information, and collect agent input. Guides can branch based on agent selections or contact attributes, creating dynamic decision trees.

Use cases:

- Identity verification checklists.
- Troubleshooting decision trees.
- Disposition code capture during ACW.
- Upsell/cross-sell scripts.
- Compliance-required disclosures.
- Onboarding walkthroughs for new agents.

---

## Creating Guides in the Flow Designer

Guides are contact flows of type **Agent whisper**, **Customer whisper**, **Agent hold**, or dedicated **View** flows. The key building block is the **Show view** block.

### Show View Block

The Show view block renders a UI view in the agent workspace. Each block defines:

- **View resource** -- which pre-built or custom view template to use.
- **Input data** -- JSON object passed to the view (dynamic values from contact attributes, system variables, or literals).
- **Output mapping** -- maps agent input from the view back to contact attributes for downstream use.

Pre-built view templates:

| View | Purpose |
|---|---|
| `DetailView` | Display read-only information (customer details, order summary). |
| `FormView` | Collect structured input (text fields, dropdowns, radio buttons, checkboxes). |
| `ConfirmationView` | Show a confirmation screen with accept/reject actions. |
| `ListSelectView` | Present a selectable list of options. |
| `CardsView` | Display a grid of clickable cards (e.g., product catalog). |
| `CustomView` | Render a fully custom HTML/JSON template. |

### Flow Structure Example

```
Start
  |
  v
[Show view: "VerifyIdentityForm"]
  |  (agent enters last 4 SSN, DOB)
  |
  v
[Check contact attribute: verification_result]
  |
  +--> Success --> [Show view: "AccountSummary"]
  |
  +--> Failure --> [Show view: "EscalateToSupervisor"]
```

---

## Invoking Guides

### At Start of Contact

Attach the guide flow as the **agent event flow** on the queue or routing profile. When a contact arrives, the guide launches automatically in the workspace:

1. Agent accepts the contact.
2. The guide's first Show view block renders immediately.
3. The agent follows the screens step by step.

### During After Contact Work (ACW)

Use **default ACW guides** to auto-launch a guide when the agent enters ACW state:

1. Create a flow with Show view blocks for disposition capture, notes, follow-up scheduling.
2. In the queue configuration, set the flow as the **default ACW flow**.
3. When the contact disconnects and the agent enters ACW, the guide launches automatically.
4. The agent completes the guide and then clears the contact.

This is the primary mechanism for structured disposition capture.

### On-Demand (During Contact)

Agents can manually invoke guides from the workspace if the guide is exposed as a quick action or embedded in a step-by-step flow that offers multiple paths.

---

## PII Redaction in Guides

Guides support PII redaction for sensitive fields:

- Mark fields as **sensitive** in the Show view block configuration.
- Sensitive field values are masked in the UI after the agent moves to the next step (e.g., SSN shows as `***-**-1234`).
- Sensitive values are redacted in contact trace records (CTRs) and logs.
- Redaction applies to both the display and the stored contact attribute value.

Configure redaction behavior:

| Setting | Behavior |
|---|---|
| `REDACT_AND_MASK` | Value stored as masked string. |
| `REDACT_AND_REMOVE` | Value removed entirely from records. |
| `NO_REDACTION` | Value stored as-is (default). |

---

## Display Contact Attributes

Guides can read and display any contact attribute dynamically:

- **System attributes** -- queue name, contact ID, channel, initiation method.
- **User-defined attributes** -- set by contact flows (e.g., customer tier, IVR selections, account number).
- **External attributes** -- fetched by Lambda functions invoked in the flow.

In the Show view block, reference attributes using JSONPath notation in the input data:

```json
{
  "customerName": "$.Attributes.CustomerName",
  "accountTier": "$.Attributes.AccountTier",
  "queueName": "$.Queue.Name",
  "contactId": "$.ContactId"
}
```

Attributes update in real time -- if a Lambda or flow block updates an attribute mid-contact, the next Show view block reflects the new value.

---

## Disposition Codes via Guides

The recommended pattern for disposition capture:

1. Create a guide flow with a `FormView` Show view block.
2. Define a dropdown or radio group with disposition options (e.g., "Resolved," "Escalated," "Follow-up needed," "Customer hung up").
3. Map the selected disposition to a contact attribute (e.g., `disposition_code`).
4. Optionally add conditional logic -- if "Follow-up needed," show a second view to collect follow-up details.
5. Set this flow as the default ACW guide on the queue.

The disposition value flows into the CTR as a contact attribute and is available in reporting, rules, and analytics.

### Making Disposition Required

To enforce disposition capture:

- Do not include a "Skip" or "Close" action on the disposition view.
- The agent must complete the guide to clear the contact.
- Alternatively, use a rule that flags contacts missing a disposition attribute.

---

## Deploying Guides in Chats

Guides work across channels. For chat contacts:

- Show view blocks render as interactive forms in the agent workspace (not in the customer's chat window).
- The agent fills out the guide while simultaneously handling the chat conversation.
- Guide data can be used to populate canned responses or trigger actions in the chat flow.

Chat-specific considerations:

- Guides can be triggered per chat message event or at chat start.
- Multi-chat concurrency means an agent may have multiple guides open simultaneously (one per chat contact).

---

## Manager Workspace Guides

Guides are not limited to agents. Supervisors and managers can use guides for:

- **Coaching forms** -- structured evaluation forms surfaced during agent monitoring.
- **Escalation workflows** -- step-by-step processes when an agent requests supervisor assistance.
- **Quality review checklists** -- standardized evaluation criteria.

Manager guides are associated with the supervisor's security profile and workspace page configuration.

---

## View Integrations with Connect Resources

Show view blocks can interact with other Connect resources:

| Resource | Integration |
|---|---|
| **Customer Profiles** | Display profile data in views; update profile fields from view inputs. |
| **Cases** | Create or update cases from guide form submissions. |
| **Tasks** | Create follow-up tasks with data collected in the guide. |
| **Contact attributes** | Read/write attributes throughout the guide flow. |
| **Lambda functions** | Invoke Lambdas between view steps for external lookups or writes. |
| **Amazon Q / AI Agents** | Surface AI recommendations alongside guide steps. |

---

## Quick Responses

Quick responses are pre-written message snippets that agents can search and insert during chat and email contacts:

- **Search** -- agents type in a search box to find relevant quick responses by keyword or category.
- **Type shortcuts** -- agents type a shortcut prefix (e.g., `brb`) and the full response auto-expands ("I will be right back. Please hold for a moment.").
- **Categories** -- organize quick responses into groups (greetings, closings, troubleshooting, policies).
- **Personalization** -- quick responses can include placeholder tokens (e.g., `{customer_name}`) that auto-fill from contact attributes.
- **Management** -- administrators create, edit, and retire quick responses in the Connect console.

Quick responses appear in the agent workspace as a searchable panel during chat and email handling.

---

## Default ACW Guides

Default ACW guides auto-launch when the agent enters After Contact Work state:

### Setup

1. Build a contact flow with Show view blocks for post-contact activities (disposition, notes, follow-up).
2. Navigate to the queue settings in the admin console.
3. Under "After Contact Work," select the flow as the default ACW guide.
4. Save the queue configuration.

### Behavior

- When the contact disconnects, the agent transitions to ACW.
- The default ACW guide launches automatically in the workspace -- no agent action required.
- The agent completes the guide steps.
- On completion, the contact clears (or the agent manually clears if ACW timeout has not expired).

### Best Practices

- Keep ACW guides short (2-4 screens max) to minimize ACW time.
- Always include a disposition step.
- Include an optional notes/comments field for free-text input.
- Use conditional branching sparingly in ACW -- agents are under time pressure.
- Monitor ACW duration metrics to ensure guides are not increasing handle time excessively.
