# Step-by-Step Guides

Step-by-step guides are no-code, flow-designed UI workflows that surface inside the Amazon Connect agent workspace. They replace tribal knowledge and paper runbooks with structured, interactive screens that walk agents through processes.

---

## Overview

Guides are built entirely in the contact flow designer -- no frontend code required. They use the **Show view** block to render UI forms, display information, and collect agent input. Guides can branch based on agent selections or contact attributes, creating dynamic decision trees.

When a flow with a Show view block runs, a separate background chat contact is created in the Connect instance. This contact creates a unique CTR. If a Set event flow block is also used, the contact is associated with the inbound contact. Neither agents nor customers are aware of this background contact.

Use cases:

- Identity verification checklists.
- Troubleshooting decision trees.
- Disposition code capture during ACW.
- Upsell/cross-sell scripts.
- Compliance-required disclosures.
- Onboarding walkthroughs for new agents.
- Payment collection with sensitive data handling.
- CRM entry creation with pre-populated contact attributes.

---

## UI Builder

The UI builder is a no-code visual designer in the Connect console for creating view resources used in guides.

### Accessing the UI Builder

1. Log in to the Connect admin website.
2. Navigate to **UI Management**.
3. Choose **Create View** and specify a name and purpose type:
   - **Guide Views** -- single or multi-step workflows for agents, customers, or managers. Display contact-specific or third-party data.
   - **Workspace Views** -- workspace pages (e.g., home page) providing interface components independent of contact handling.

### Capabilities

The UI builder provides:

1. **Create panel** -- library of UI components and available templates for quick starts.
2. **Component groups** -- components organized in collapsible containers. Drag and drop onto the canvas.
3. **Canvas** -- visual preview of the view resource being designed.
4. **Customize panel** -- global settings (columns, alignment, colors) and per-component properties.

### Component Properties

Each component has a **Properties** tab where values can be set:
- **Static values** -- hardcoded text and configuration.
- **Dynamic values** -- use the lightning bolt icon to populate fields at runtime from contact attributes, integrations, or other data sources.

### Dynamic References

Reference contact attributes using JSONPath notation: `$.Attributes.MyCustomAttribute`

Reference integration output data using: `$.#[IntegrationName].[ReferenceObject]`

---

## View Resources (UI Templates)

Views are UI templates used to customize the agent workspace. Each view has three key elements:

1. **Template** -- the visual layout and component structure.
2. **Input schema** -- data passed into the view for rendering.
3. **Actions** -- interactive elements that produce output when the agent interacts.

### Data Mapping

Use the **Set JSON** option in the Show view block for the best data mapping experience. All flow namespaces can be referenced (including `$.External`), allowing data from Connect and external systems to be combined in a single view.

Complex JSON objects can be passed between the workspace and flows using the Show view block and Lambda function blocks.

---

## View Types

### Detail View

Displays read-only information to agents with a list of actions they can take. Common for screen-pops at call start.

**Properties:**

| Property | Required | Description |
|---|---|---|
| **Sections** | Yes | Body content. Can be static strings, TemplateStrings (HTML), or key-value pairs. Single data point or list. |
| **AttributeBar** | No | Persistent context bar at top. List of objects with Label, Value, and optional LinkType, ResourceId, Copyable, Url. |
| **Back** | No | Back navigation link. Required if no Actions provided. Object with Label property. |
| **Heading** | No | Page title text. |
| **Description** | No | Description text below the title. |
| **Actions** | No | List of action buttons at the bottom of the page. |

**AttributeBar LinkType options:**
- `external` -- navigates to a new browser page via the Url property.
- `case` -- navigates to a case detail in the workspace via the ResourceId property.
- `Copyable: true` -- allows users to copy the ResourceId.

**Output:**
```json
{
  "Action": "ActionSelected",
  "ViewResultData": {
    "actionName": "Action 2"
  }
}
```

### List View

Displays information as a list of items with titles and descriptions. Items can act as links with actions attached.

**Properties:**

| Property | Required | Description |
|---|---|---|
| **Items** | Yes | List of items. Each item has optional Heading, Description, Icon, and Id. When Id is defined, it is included in output. |
| **AttributeBar** | No | Persistent context bar (same structure as Detail view). |
| **Back** | No | Back navigation link. |
| **Heading** | No | Page title. |
| **SubHeading** | No | Title for the list section. |

**Output:**
```json
{
  "Action": "ActionSelected",
  "ViewResultData": {
    "actionName": "Select_Car"
  }
}
```

### Form View

Provides input fields to gather data and submit to backend systems. Multiple sections with headers, various input field types in column or grid layout.

**Properties:**

| Property | Required | Description |
|---|---|---|
| **Sections** | Yes | List of sections containing input/display fields. Each section has SectionProps. |
| **AttributeBar** | No | Persistent context bar. |
| **Back** | No | Back navigation link. Object or string with Label. |
| **Next** | No | Submit/continue action. Object (FormActionProps) or string. Used when step is not the last. |
| **Previous** | No | Previous step action. Object (FormActionProps) or string. Used when step is not the first. |
| **Cancel** | No | Cancel action. Object (FormActionProps) or string. |
| **Edit** | No | Edit action for DataSection type sections. Object (FormActionProps) or string. |
| **Heading** | No | Page title string. |
| **SubHeading** | No | Secondary message. |
| **ErrorText** | No | Server-side error display. ErrorProps with Header and Content. |
| **Wizard** | No | Progress tracker displayed on the left side. Each step has Heading (required), Description (optional), Optional (optional). |

**Section Properties (SectionProps):**

| Property | Description |
|---|---|
| **Heading** | Section header text. |
| **Type** | `FormSection` (user input fields) or `DataSection` (display label-value pairs). |
| **Items** | List of data/form components based on Type. |
| **isEditable** | Boolean. Shows edit button at header for DataSection types. |

**Form Input Types:**
- `FormInput` -- text input with InputType (text, number, email, etc.), Label, Name, DefaultValue, Fluid.
- `DatePicker` -- date selector with Label, Name, DefaultValue.
- `TimeInput` -- time selector with Label, Name, DefaultValue.
- Dropdowns, radio buttons, checkboxes (via form component types).

**Layout Configuration:**
```json
{
  "LayoutConfiguration": {
    "Grid": [{
      "colspan": { "default": "6", "xs": "4" }
    }]
  }
}
```

**FormActionProps:**
```json
{
  "Label": "Confirm Reservation",
  "Details": {
    "endpoint": "awesomecustomer.com/submit"
  }
}
```

**Output:**
```json
{
  "Action": "Submit",
  "ViewResultData": {
    "FormData": {
      "pickup-location": "Seattle",
      "pickup-day": "2022-10-10",
      "pickup-time": "13:00"
    },
    "StepName": "Pickup and drop off"
  }
}
```

### Confirmation View

Displayed after form submission or action completion. Shows summary, next steps, and prompts.

**Properties:**

| Property | Required | Description |
|---|---|---|
| **Next** | Yes | Action button for navigation. Object with Label string. |
| **AttributeBar** | No | Persistent context bar. |
| **Heading** | No | Page title (e.g., "Your reservation has been updated"). |
| **SubHeading** | No | Secondary message (e.g., "You will receive a confirmation shortly"). |
| **Graphic** | No | Displays an icon/image. Object with `Include: true/false`. |

**Output:**
```json
{
  "Action": "Next",
  "ViewResultData": {
    "Label": "Go Home"
  }
}
```

### Cards View

Presents a grid of clickable cards for topic selection. Cards expand to show details when selected.

**Properties:**

| Property | Required | Description |
|---|---|---|
| **Cards** | Yes | List of card objects, each with Summary and Detail. |
| **AttributeBar** | No | Persistent context bar. |
| **Heading** | No | Page title (e.g., "Customer may be contacting about..."). |
| **Back** | No | Back navigation link. |
| **NoMatchFound** | No | Button below cards for fallback (e.g., "Can't find match?"). |

**Card Summary Properties:**
- `Id` -- unique identifier, included in output.
- `Icon` -- icon name (e.g., "Airplane", "Car Side View", "Suitcase").
- `Heading` -- card title.
- `Status` -- status text (e.g., "Upcoming Sept 17, 2022").
- `Description` -- card description.

**Card Detail Properties:**
- `Heading` -- detail page title.
- `Description` -- detail description.
- `Sections` -- body content using TemplateString (HTML supported).
- `Actions` -- list of action buttons in the detail view.

**Output:**
```json
{
  "Action": "ActionSelected",
  "ViewResultData": {
    "actionName": "Update the trip"
  }
}
```

### Custom Views (Customer-Managed)

Fully custom views created via API or UI builder. Render custom HTML/JSON templates with arbitrary layouts and components. Custom views support the **Apply Sample Data** option in the Show view block to pre-populate input with sample JSON schema.

---

## Show View Block Configuration

The Show view block is the flow block that renders views in the workspace.

### Configuration Methods

1. **Set manually** -- enter static text for each field in the block properties.
2. **Set dynamically** -- choose a namespace and key from contact attributes. Values render dynamically at runtime (e.g., `$.Customer.LastName`).
3. **Set JSON** -- paste a JSON object defining the complete view input. Supports all namespaces including `$.External`. Best for complex configurations.

### Contact Type Support

| Contact Type | Supported |
|---|---|
| Voice | Yes |
| Chat | Yes |
| Task | Yes |
| Email | Yes |

### Flow Type Support

| Flow Type | Supported |
|---|---|
| Inbound flow | Yes |
| Customer hold / whisper / outbound whisper | No |
| Agent hold / whisper | No |
| Transfer to agent / queue | No |

### Block Branches

- **Conditional branches** -- depend on which view is selected and which action the agent takes (e.g., Back, Next, custom actions).
- **No Match** -- triggered when an action component has a custom value that does not match any condition.
- **Error** -- failure to render the view or capture output (e.g., network issues).
- **Timeout** -- configurable time limit for step completion. When exceeded, the flow follows timeout logic. The customer remains connected during timeout.

### Sensitive Data Handling

Enable **This view has sensitive data** when collecting credit card data, addresses, or other sensitive information:
- Data submitted will not be recorded in transcripts or contact records.
- Data will not be visible to agents by default.
- Turn off logging (Set Logging Behavior) to prevent sensitive data in flow logs.
- Build reusable payment modules using a flow module with this setting + Lambda + prompts.

### Data Generated

At runtime, the Show view block generates:
- `$.Views.Action` -- the action taken on the view UI (represents a branch).
- `$.Views.ViewResultData` -- the output data from the view interaction.

These values are determined by which components the agent interacted with.

### Security Profile Permissions

| Permission | Who | Purpose |
|---|---|---|
| Agent Applications - Custom views - All | Agents | See step-by-step guides in workspace |
| Channels and flows - Views | Managers/analysts | Configure step-by-step guides in flows |
| Flows - Edit, Create | Managers/analysts | Create the flows that guides are built in (guides are created using flows) |

**Service quota:** guide workflows run as **chat contacts**, so increase the **concurrent active chats per instance** quota by the number of concurrent contacts you expect to use guides. Note: Disconnect-flow workflows count as their own contacts — if you set both a `DefaultFlowID` and a `DisconnectFlowID`, they count as **two** active contacts.

---

## Invoking Guides

### At Start of Contact

Use a **Set event flow** block configured with the **`DefaultFlowForAgentUI`** event hook (UI label: **Default flow for Agent UI**) to choose which guide flow to send to the agent.

1. Add a **Set event flow** block to your flow and set the **DefaultFlowForAgentUI** event hook to the guide flow's ID.
2. The guide starts **as soon as the contact is offered to the agent** — it does **not** wait for the agent to accept.
3. The agent follows the screens step by step.

To pick the guide dynamically, branch with a **Check attribute** block (on IVR responses, queue name, customer info, etc.) and set the flow ID via **Set event flow** accordingly.

### During After Contact Work (ACW)

Surface a guide after the contact ends (during ACW) by setting the **`DisconnectFlowForAgentUI`** contact attribute:

1. Create a flow with Show view blocks for disposition capture, notes, follow-up scheduling.
2. Set the `DisconnectFlowForAgentUI` attribute in the contact flow **before the contact ends** (as long as it's set before the contact ends, the agent UI surfaces this form after the contact ends).
3. When the contact disconnects and the agent enters ACW, the guide launches automatically.
4. The agent completes the guide and then clears the contact.

This is the primary mechanism for structured disposition capture.

### On-Demand (During Contact)

Agents can manually invoke guides from the workspace if the guide is exposed as a quick action or embedded in a step-by-step flow that offers multiple paths.

---

## PII Redaction in Guides

By default, any information passed through a guide is included in the **contact record transcript**. To prevent PII from appearing there:

1. Use the **Set recording and analytics behavior** block in the guide flow.
2. **Enable Contact Lens** (conversational analytics) on that block.
3. Enable the **redaction of sensitive data**.

Notes:
- Redaction is performed by Contact Lens (ML/NLU) and is applied **after the call disconnects** — it is **not** real-time.
- It acts on the **contact record transcript / recording**, not the live guide UI.
- It **may not identify and remove all instances** of sensitive data, and does **not** meet HIPAA de-identification requirements.

> This is a separate control from the Show view block's **"This view has sensitive data"** toggle (see [View types / sensitive data handling](#show-view-block-configuration)), which keeps a view's input/output out of transcripts and contact records. Use whichever fits — they are independent.

---

## Display Contact Attributes

Guides can read and display any contact attribute dynamically:

- **System attributes** -- queue name, contact ID, channel, initiation method.
- **User-defined attributes** -- set by contact flows (e.g., customer tier, IVR selections, account number).
- **External attributes** -- fetched by Lambda functions invoked in the flow (via `$.External` namespace).

In the Show view block, reference attributes using JSONPath notation in the input data:

```json
{
  "customerName": "$.Attributes.CustomerName",
  "accountTier": "$.Attributes.AccountTier",
  "queueName": "$.Queue.Name",
  "contactId": "$.ContactId",
  "channel": "$.Channel"
}
```

Attributes update in real time -- if a Lambda or flow block updates an attribute mid-contact, the next Show view block reflects the new value.

---

## Disposition Codes via Guides

The recommended pattern for disposition capture:

1. Create a guide flow with a `FormView` Show view block.
2. Define a dropdown or radio group with disposition options (e.g., "Resolved," "Escalated," "Follow-up needed," "Customer hung up").
3. Use a **Set contact attributes** block to save the selected disposition to a user-defined attribute (e.g., `disposition_code`).
4. Optionally add a **Lambda function** block to send data to an external system.
5. Set the `DisconnectFlowForAgentUI` custom attribute in the inbound flow to dynamically surface the disposition form after contact ends.

The disposition value flows into the CTR as a contact attribute and is available in reporting, rules, and analytics.

### Making Disposition Required

To enforce disposition capture:

- Do not include a "Skip" or "Close" action on the disposition view.
- The agent must complete the guide to clear the contact.
- Alternatively, use a rule that flags contacts missing a disposition attribute.

### Conditional Logic

- If "Follow-up needed" is selected, show a second view to collect follow-up details.
- Use Check contact attribute blocks between Show view blocks for branching.
- Use the Loop block to limit retries if the Show view block takes an error or timeout branch.

---

## Deploying Guides in Chats

You can present the **same guide you built for agents to the customer inside the chat widget**, creating an interactive **self-service** experience that gathers context and then transfers the customer (with that context) to an agent.

**Setup:**

1. First **enable and configure guides for agents** (confirm they pop when a contact is reserved for an agent).
2. In the **chat flow**, invoke views with the **Show View** block exactly as you would for an agent (e.g. run through a couple of views before transferring the chat to an agent).
3. Create a **hosted chat widget** from the admin page and set its **chat flow** to the one you created. This generates the widget `<script>` snippet.
4. In that snippet, extend the `supportedMessagingContentTypes` array to include the interactive message types so guides render in chat:
   ```js
   amazon_connect('supportedMessagingContentTypes', [
     'text/plain',
     'application/vnd.amazonaws.connect.message.interactive',
     'application/vnd.amazonaws.connect.message.interactive.response'
   ]);
   ```
5. Add `{{your-website-url}}/views/renderer/` to your **URL allow-list** so guides work within chat. (If you use a CSP for the chat widget, you'll already have the CloudFront URL such as `https://{{unique-id}}.cloudfront.net/amazon-connect-chat-interface.js`.)

You can also use guides in chat with a **custom-built communications widget** — see the `amazon-connect/amazon-connect-chat-interface` project on GitHub.

---

## Manager Workspace Guides

Guides are not limited to agents. Supervisors and managers can use guides in persona-based workspaces:

### Setup

1. Create a persona-based workspace for managers.
2. In the UI builder, drag the **Connect Application** component onto the canvas.
3. Configure the component:
   - **Application Namespace** -- select "Guide."
   - **ContactFlowId** -- specify the guide's contact flow ID.

### Behavior

- Users start the guide by choosing the "Begin" button (creates a background chat contact).
- After completing the workflow, users restart with the "Restart" button.
- Nesting a guide application component inside a view already used in a guide is not supported.
- The guide can only be embedded in a static view used as a workspace page.

### Use Cases

- **Coaching forms** -- structured evaluation forms surfaced during agent monitoring.
- **Escalation workflows** -- step-by-step processes when an agent requests supervisor assistance.
- **Quality review checklists** -- standardized evaluation criteria.
- **Emergency protocol execution** -- data table-driven operations via workspace views.

Manager guides are associated with the supervisor's security profile and workspace page configuration.

---

## View Integrations with Connect Resources

Views can poll live data sources at specified intervals using view integrations, configured in the UI builder's global settings panel.

### Integration Configuration

Each tool (integration point) requires:

| Property | Description |
|---|---|
| **Integration Name** | Custom name used to reference data in the view. |
| **Integration Type** | Format of the integration between view and Connect resource. |
| **Tool** | Integration source (e.g., Flow modules). |
| **Version or Alias** | Flow module version/alias to call for data. |
| **Enable refresh** | Boolean to enable periodic data polling. |
| **Refresh intervals** | Polling interval in seconds. |
| **Tool input object** | JSON object sent to the source to fetch data. |

### Referencing Integration Data

Reference integration output in UI components using: `$.#[IntegrationName].[ReferenceObject]`

Ensure the reference object format matches what the component property expects.

### Runtime Behavior

- On view load, data from the integration populates as defined in the schema.
- If refresh intervals are set, the view invokes the integration at each interval.
- When new data is available, the view prompts the user to refresh.
- Polling works best in single-tab environments. Multiple tabs each poll independently, potentially causing throttling.

### Resource Integration Matrix

| Resource | Integration Capability |
|---|---|
| **Customer Profiles** | Display profile data in views; update profile fields from view inputs. |
| **Cases** | Create or update cases from guide form submissions. |
| **Tasks** | Create follow-up tasks with data collected in the guide. |
| **Contact attributes** | Read/write attributes throughout the guide flow. |
| **Lambda functions** | Invoke Lambdas between view steps for external lookups or writes. |
| **Amazon Q / AI Agents** | Surface AI recommendations alongside guide steps. |
| **Flow modules** | Poll live data at intervals for real-time view updates. |

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

## After-Contact (ACW) Guides

Surface a guide automatically when the agent enters After Contact Work, by setting the `DisconnectFlowForAgentUI` attribute:

### Setup

1. Build a contact flow with Show view blocks for post-contact activities (disposition, notes, follow-up).
2. Set the `DisconnectFlowForAgentUI` attribute in the inbound contact flow **before the contact ends**.
3. When the contact disconnects, the agent transitions to ACW and the guide launches automatically.

### Behavior

- The ACW guide launches automatically in the workspace -- no agent action required.
- The agent completes the guide steps.
- On completion, the contact clears (or the agent manually clears if ACW timeout has not expired).

### Best Practices

- Keep ACW guides short (2-4 screens max) to minimize ACW time.
- Always include a disposition step.
- Include an optional notes/comments field for free-text input.
- Use conditional branching sparingly in ACW -- agents are under time pressure.
- Monitor ACW duration metrics to ensure guides are not increasing handle time excessively.
- Use the Loop block to limit retries on error/timeout branches to prevent infinite loops.

---

## APIs

| API | Purpose |
|---|---|
| `CreateView` | Create a new view resource. |
| `UpdateViewContent` | Modify the content/schema of an existing view. |
| `CreateViewVersion` | Create a versioned snapshot of a view. |
| `ShowView` (Flow action) | Render a view in the agent workspace from a contact flow. |

### ShowView Flow Language

```json
{
  "Parameters": {
    "ViewResource": {
      "Id": "arn:aws:connect:{region}:aws:view/form:1"
    },
    "InvocationTimeLimitSeconds": "300",
    "ViewData": {
      "Heading": "$.Customer.LastName",
      "SubHeading": "$.Customer.FirstName",
      "Sections": "...",
      "AttributeBar": [
        { "Label": "Queue", "Value": "Sales" },
        { "Label": "Case", "Value": "123456", "LinkType": "case", "ResourceId": "123456", "Copyable": true }
      ],
      "Back": { "Label": "Back" },
      "Next": "Next",
      "Previous": "Previous",
      "Wizard": {
        "Heading": "Progress tracker",
        "Selected": "Step Selected"
      }
    }
  },
  "Type": "ShowView",
  "Transitions": {
    "Conditions": [
      { "Condition": { "Operator": "Equals", "Operands": ["Back"] } },
      { "Condition": { "Operator": "Equals", "Operands": ["Next"] } },
      { "Condition": { "Operator": "Equals", "Operands": ["Step"] } }
    ],
    "Errors": [
      { "ErrorType": "NoMatchingCondition" },
      { "ErrorType": "NoMatchingError" },
      { "ErrorType": "TimeLimitExceeded" }
    ]
  }
}
```
