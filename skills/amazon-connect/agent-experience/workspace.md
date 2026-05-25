# Agent Workspace Architecture

The Amazon Connect agent workspace is the unified, browser-based application where agents handle customer contacts across all channels. It consolidates eight integrated components into a single pane of glass.

**URLs:**

- Agent workspace: `https://{instance}.my.connect.aws/agent-app-v2/`
- CCP (standalone): `https://{instance}.my.connect.aws/ccp-v2/`

---

## 1. Contact Control Panel (CCP)

The CCP is the core telephony and contact-handling widget embedded in the workspace. It handles:

- **Voice calls** -- accept, reject, hold, resume, transfer, conference, disconnect.
- **Chats** -- accept, send messages, transfer, end conversation. Supports concurrent multi-chat.
- **Emails** -- accept, reply, forward, compose drafts with rich text editor. Full threading support.
- **Tasks** -- accept, create, pause, resume, transfer, complete. Tasks can be assigned via flows, APIs, or other agents.

The CCP surfaces agent status controls, the number pad (DTMF), quick connects for transfers, and device settings (softphone vs. desk phone).

When embedded in the workspace, the CCP appears as a narrow panel on the left side. It can also run standalone at the `/ccp-v2/` URL for minimal deployments or third-party CRM integrations.

---

## 2. Third-Party Applications (Apps Launcher)

Third-party applications are external web applications loaded inside the workspace via HTTPS iframes. They appear as tabs in the workspace and can be launched from the Apps launcher button.

- Built with any frontend framework (React, Angular, Vue, plain HTML/JS).
- Communicate with the workspace via the `@amazon-connect/sdk` packages.
- Receive contact events, agent state changes, and theme updates from the workspace.
- Can be configured to auto-launch on specific contact events (e.g., incoming call, ACW).

Applications are registered in the Amazon Connect admin console under "Third-party applications." Each app defines a display name, namespace, access URL, allowed origins, and launch permissions.

---

## 3. Connect AI Agents (Real-Time Recommendations)

Amazon Connect AI agents (formerly Amazon Q in Connect / Wisdom) provide real-time recommendations to agents during live contacts:

- Automatically detects customer intent from the conversation transcript.
- Surfaces relevant knowledge articles, recommended responses, and next-best actions.
- Recommendations update in real time as the conversation progresses.
- Agents can search the knowledge base manually if automatic detection does not surface the right content.
- Supports custom AI guardrails and self-service configurations.

The AI agents panel appears as a tab in the workspace alongside the contact details.

---

## 4. Tasks

Tasks are a contact channel for tracking follow-up work, assignments, and non-real-time activities:

- **Create tasks** -- agents create tasks manually from the workspace, or tasks are generated automatically via contact flows, rules, or API calls.
- **Assign tasks** -- route to specific agents, queues, or quick connects.
- **Task templates** -- define structured fields (required/optional) for consistent task creation.
- **Task scheduling** -- schedule tasks for future dates and times.
- **Linked tasks** -- associate tasks with the originating contact for context.
- **Pause and resume** -- agents can pause a task (with a reason) and resume later. Paused tasks do not count against concurrency.

Tasks appear in the CCP contact list alongside calls, chats, and emails.

---

## 5. Cases Tab

Amazon Connect Cases provides case management directly in the workspace:

- Agents view, create, and update cases linked to customer profiles.
- Each case has fields (status, priority, summary, custom fields), a timeline of events, and linked contacts.
- Cases can be created automatically via contact flows or rules.
- Case templates define the required and optional fields for case creation.
- Cases persist across contacts -- an agent can reference a case from a previous interaction.

The Cases tab appears in the workspace when Cases is enabled for the instance.

---

## 6. Step-by-Step Guides

Guides are no-code, flow-designed UI workflows that surface inside the agent workspace. They walk agents through structured processes (identity verification, troubleshooting scripts, disposition capture).

- Created in the contact flow designer using the "Show view" block.
- Can be invoked at the start of a contact, during handling, or during After Contact Work (ACW).
- Support form inputs, dropdowns, radio buttons, and conditional branching.
- Can read and write contact attributes for dynamic content.
- Support PII redaction for sensitive fields.
- Default ACW guides auto-launch when the agent enters ACW state.

See `step-by-step-guides.md` for full details.

---

## 7. Customer Profile Tab

The Customer Profile tab displays a unified customer view assembled from multiple data sources:

- Shows customer identity (name, phone, email, account number) resolved via Customer Profiles domain.
- Displays contact history, case history, and product/asset information.
- Data ingested from S3, Salesforce, ServiceNow, Zendesk, Segment, Shopify, and custom integrations.
- Identity resolution uses ML to merge duplicate profiles across sources.
- Agents can edit profile fields directly from the workspace.
- Contact flows can auto-populate the profile tab based on caller ID or IVR-collected data.

---

## 8. Voice ID (Excluded -- End of Life)

Amazon Connect Voice ID reached end of life and is excluded from new implementations. Previously provided real-time caller authentication via voiceprint and fraud detection.

---

## Theme Customization

Administrators can customize the workspace appearance using the `UpdateWorkspaceTheme` API:

### Palette

Define custom colors for the workspace chrome:

| Token | Purpose |
|---|---|
| `primaryColor` | Buttons, links, active states |
| `secondaryColor` | Secondary actions, hover states |
| `backgroundColor` | Workspace background |
| `surfaceColor` | Cards, panels, elevated surfaces |
| `textColor` | Primary text |
| `errorColor` | Error states, destructive actions |
| `warningColor` | Warning indicators |
| `successColor` | Success indicators |

### Fonts

Specify custom font families and sizes:

```json
{
  "fontFamily": "Inter, system-ui, sans-serif",
  "headerFontFamily": "Inter, system-ui, sans-serif",
  "fontSize": {
    "small": "12px",
    "medium": "14px",
    "large": "16px"
  }
}
```

### Logo

Upload a custom logo (PNG or SVG, recommended 200x40px) that replaces the Amazon Connect branding in the workspace header:

```
UpdateWorkspaceTheme --instance-id {id} --logo-url "https://assets.example.com/logo.svg"
```

Third-party applications receive theme changes via the SDK theme integration and can adapt their UI to match.

---

## Persona-Based Workspace Pages

Use `CreateWorkspacePage` to define custom workspace layouts tailored to specific agent roles or teams:

- Each workspace page defines which components (tabs, panels, apps) are visible and their arrangement.
- Assign workspace pages to routing profiles or security profiles.
- Examples:
  - **Sales agent page** -- CCP + Customer Profile + CRM app + Cases.
  - **Support agent page** -- CCP + AI Agents + Knowledge Base app + Step-by-Step Guides.
  - **Supervisor page** -- Real-time metrics + Agent monitoring + Quality Management.

```
CreateWorkspacePage
  --instance-id {id}
  --name "Sales Agent Workspace"
  --components '[
    {"type": "CCP", "position": "LEFT"},
    {"type": "CUSTOMER_PROFILE", "position": "CENTER"},
    {"type": "THIRD_PARTY_APP", "appId": "crm-app-id", "position": "CENTER"},
    {"type": "CASES", "position": "RIGHT"}
  ]'
```

Workspace pages are evaluated at login time based on the agent's security profile assignment.

---

## Disposition Codes

Disposition codes let agents categorize the outcome of a contact:

- Configured as a list of codes in the admin console or via step-by-step guides.
- Agents select a disposition during or after the contact (typically during ACW).
- Disposition values are stored as contact attributes and appear in contact records.
- Can be made required (agent cannot clear the contact without selecting a disposition).
- Dispositions can trigger downstream rules (e.g., auto-create a follow-up task, update a case).

When using guides for disposition capture, the guide form collects the disposition and writes it to a contact attribute via the flow.

---

## Admin Workspaces Setup

To configure the agent workspace:

1. **Enable the workspace** -- In the Amazon Connect console, navigate to "Agent application" under "Application integration."
2. **Register third-party apps** -- Add each app under "Third-party applications" with its access URL, allowed origins, and permissions.
3. **Create workspace pages** -- Define persona-based layouts via `CreateWorkspacePage` API or console.
4. **Apply themes** -- Use `UpdateWorkspaceTheme` to set branding.
5. **Configure guides** -- Build step-by-step guides in the flow designer and assign them to contact flows.
6. **Enable AI agents** -- Set up knowledge bases and enable Amazon Q in Connect.
7. **Set security profiles** -- Control which components each agent role can access via security profile permissions.
8. **Distribute the URL** -- Agents access the workspace at `https://{instance}.my.connect.aws/agent-app-v2/`.
