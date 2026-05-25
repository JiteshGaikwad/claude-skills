# Amazon Connect Flow Designer Overview

## Flow Types

Amazon Connect provides seven distinct flow types, each serving a specific purpose in the contact routing lifecycle.

### Inbound Flow (Contact Flow)

The primary flow type. Runs when a customer initiates contact (phone call, chat, task). This is where you define the complete customer experience: greetings, menus, data lookups, routing decisions, and queue placement.

- Default: "Default customer queue flow" is used if no inbound flow is assigned to a phone number or chat endpoint.
- Every phone number or chat endpoint must be associated with an inbound flow.

### Customer Queue Flow

Runs while the customer is waiting in queue. Controls the hold experience: music, position announcements, estimated wait time, callback offers.

- Triggered after `Transfer to queue` places the contact in a queue.
- Can loop prompts, offer callbacks, or check queue metrics to make dynamic decisions.
- Set via the `Set customer queue flow` block in the inbound flow.

### Customer Whisper Flow

Plays a short message to the customer immediately before the agent connects. The customer hears this; the agent does not.

- Example: "This call may be recorded for quality purposes."
- Set via the `Set whisper flow` block (customer side).
- Default: "Default customer whisper" plays a beep.

### Agent Whisper Flow

Plays a short message to the agent immediately before they connect with the customer. The agent hears this; the customer does not.

- Example: "Incoming call from the billing queue."
- Set via the `Set whisper flow` block (agent side).
- Default: "Default agent whisper" plays a beep.

### Outbound Whisper Flow

Runs when an outbound call is placed (agent-initiated or via API). Controls what happens while the call is connecting and after the customer answers.

- Used for outbound campaigns, callbacks, and agent-initiated calls.
- Set at the queue level or via API.

### Transfer to Agent Flow

Runs when a contact is transferred directly to a specific agent (not a queue). Controls the experience during and after the transfer.

- Beta feature.
- Set via the `Transfer to agent` block.

### Transfer to Queue Flow

Runs when a contact is transferred to a different queue. Allows you to modify attributes, play prompts, or apply different routing logic before the contact enters the new queue.

- Set via the `Transfer to queue` block.

## Default Flows

Every Amazon Connect instance comes with a set of default flows that cannot be deleted. They serve as fallbacks when no custom flow is assigned. You can modify default flows or create custom ones to replace them.

Default flows include:
- Default customer queue flow
- Default customer hold flow
- Default customer whisper flow (beep)
- Default agent hold flow
- Default agent whisper flow (beep)
- Default outbound flow
- Default agent transfer flow
- Default queue transfer flow

## Sample Flows

Amazon Connect provides sample flows that demonstrate common patterns:
- Sample inbound flow (first call experience)
- Sample customer queue priority
- Sample disconnect flow
- Sample queue configurations
- Sample Lambda integration
- Sample note for screenpop
- Sample AB test
- Sample secure input with agent
- Sample secure input with no agent
- Sample recording behavior
- Sample interruptible queue flow with callback

These cannot be edited directly. Clone them to create editable copies.

## Flow Modules

Flow modules are reusable, nestable sub-flows that work across all flow types.

- Create a module once, invoke it from any flow using the `Invoke module` block.
- Modules end with the `Return` block, which passes control back to the calling flow.
- Modules can set contact attributes that the calling flow can read.
- Modules can invoke other modules (nesting).
- Use modules for common patterns: authentication sequences, data lookups, standard greetings.
- Modules share the same 250-action limit per module.
- Modules are versioned independently from the flows that call them.

## Prompts

Prompts are audio files or text-to-speech (TTS) content used in flows.

### Creating Prompts
- Upload WAV files (8 kHz, 16-bit, mono PCM recommended for telephony).
- Use SSML or plain text for TTS prompts (configured per `Play prompt` or `Get customer input` block).
- Manage prompts in the Amazon Connect admin console under Routing > Prompts.

### Managing Prompts
- Prompts can be referenced by name or ARN in flow blocks.
- S3-stored prompts are supported for dynamic content via the `Get stored content` block.
- Maximum prompt file size: 50 MB.
- Supported formats: WAV.

## Import and Export

Flows can be exported as JSON and imported into the same or different instances.

- Export: Select a flow in the designer, choose Export. Produces a JSON file in the Flow Language format.
- Import: Use the Import button in the flow designer. The JSON is validated before import.
- Use export/import for promoting flows between dev/staging/prod instances.
- Contact flow modules can also be imported/exported.
- Phone numbers, queue ARNs, Lambda ARNs, and Lex bot references are instance-specific and must be updated after import.

## Migration

When migrating flows between instances or regions:
1. Export flows as JSON from the source instance.
2. Update all ARN references (Lambda functions, Lex bots, queues, prompts) to match the target instance.
3. Import into the target instance.
4. Test thoroughly: attribute references, Lambda integrations, and queue assignments.
5. Use the Amazon Connect API (`CreateContactFlow`, `UpdateContactFlowContent`) for programmatic migration.

## Keyboard Shortcuts

The flow designer supports keyboard shortcuts for efficiency:
- `Ctrl+C` / `Ctrl+V` — Copy/paste blocks
- `Ctrl+Z` / `Ctrl+Y` — Undo/redo
- `Ctrl+A` — Select all blocks
- `Delete` / `Backspace` — Delete selected blocks
- `Ctrl+S` — Save flow
- Arrow keys — Move selected blocks
- Mouse wheel — Zoom in/out
- Space + drag — Pan the canvas

## Permissions

Flow management requires specific security profile permissions:
- **Contact flows — Create**: Create and clone flows
- **Contact flows — Edit**: Modify existing flows
- **Contact flows — Delete**: Remove flows
- **Contact flows — Publish**: Save and publish flows (making them live)
- **Contact flows — View**: View flow list and details
- **Contact flow modules — All**: Separate permissions for modules
- **Prompts — All**: Create, edit, delete prompts used in flows

Assign these via Security Profiles in the Amazon Connect admin console.

## Conversational AI Bots

Flows integrate with conversational AI through the `Get customer input` block:
- **Amazon Lex**: Configure a Lex bot for intent recognition and slot filling. Supports both voice and chat channels.
- **Amazon Lex V2**: Preferred. Supports multiple languages, streaming, and improved NLU.
- The `Get customer input` block can be configured for DTMF, Lex, or both.
- Lex slot values are available as contact attributes: `$.Lex.{SlotName}`.
- Lex session attributes can pass context between the flow and the bot.

## Agent-Initiated Flows

Agents can trigger outbound flows:
- Outbound calls use the outbound whisper flow assigned to the queue.
- Quick connects can invoke transfer flows.
- The Connect Agent Workspace or CCP initiates the flow.
- Agent-initiated flows have access to the agent's contact attributes and the destination number.

## Callbacks

### Queued Callback
- Offered to customers waiting in queue via the `Create callback` block (inside a customer queue flow or transfer to queue flow).
- The customer provides a callback number (or uses the inbound number).
- The callback is placed in the queue at the same priority/position.
- When an agent becomes available, Connect dials the customer.
- Retry logic: configurable number of retry attempts and delay between retries.

### Customer-First Mode
- In customer-first callback mode, Connect dials the customer first.
- Only after the customer answers does Connect route to an available agent.
- Reduces agent idle time waiting for the customer to pick up.
- Configured in the callback block settings.

## Nova Sonic Speech-to-Speech

Amazon Nova Sonic enables speech-to-speech AI interactions in contact flows. Instead of the traditional IVR (TTS prompt -> speech recognition -> Lex NLU -> response TTS), Nova Sonic provides a single model that processes speech input and generates speech output directly, enabling more natural, conversational voice AI experiences.

- Configured via the `Connect assistant` block in the flow designer.
- Supports real-time, bidirectional voice interactions.
- See `nova-sonic.md` for detailed configuration.

## Limits

- **Maximum 250 actions per flow.** This is a hard limit. If your flow exceeds this, break it into modules or transfer to another flow.
- Maximum 20 flows can be chained via `Transfer to flow`.
- Maximum flow execution time: 5 minutes for voice, 7 days for chat/task.
- Maximum number of flows per instance: 500 (soft limit, can be increased).
- Maximum number of modules per instance: 200 (soft limit).
