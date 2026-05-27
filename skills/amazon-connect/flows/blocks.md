# Amazon Connect Flow Blocks Reference

53 flow blocks organized by category. Voice ID blocks are excluded.

## Interact

Blocks that interact with the contact (play audio, collect input, display views).

| Block | Description | Channels | Key Details |
|-------|-------------|----------|-------------|
| **Play prompt** | Plays an audio prompt or TTS message to the contact. | Voice, Chat | Supports SSML, dynamic text via attributes, uploaded audio files. |
| **Get customer input** | Collects DTMF input or invokes a Lex bot for NLU. | Voice, Chat | Configure timeout, max digits, Lex bot association. Branches on Lex intents or DTMF input. Encrypts sensitive input if configured. |
| **Store customer input** | Captures and stores DTMF input (credit card, SSN, etc.) with optional encryption. | Voice | Stores as contact attribute. Supports customer-key encryption for PCI compliance. |
| **Loop prompts** | Plays a sequence of prompts in a loop while the contact waits. | Voice, Chat | Used in customer queue flows. Configure interrupt behavior. |
| **Hold customer or agent** | Places the customer or agent on hold. | Voice | Branches: Success, Error. |
| **Get stored content** | Retrieves content from S3 or Amazon Q in Connect knowledge bases. | Voice, Chat | Returns content to play or send as a message. |
| **Connect assistant** | Invokes Amazon Nova Sonic for speech-to-speech conversational AI. | Voice | Enables natural voice AI interactions without Lex. See nova-sonic.md. |
| **Show view** | Displays a UI view (form, list, detail) to the agent in the agent workspace. | Chat, Task | Step-by-step guides. Renders JSON-defined views. Branches on view actions. |
| **Send message** | Sends a chat message to the customer. | Chat | Supports plain text and rich formatting. |
| **Authenticate Customer** | Initiates customer authentication via Amazon Connect Voice ID or custom auth. | Voice | Branches on authentication result. |

## Set

Blocks that set configuration, attributes, or behavior for the contact.

| Block | Description | Channels | Key Details |
|-------|-------------|----------|-------------|
| **Set contact attributes** | Sets user-defined contact attributes (key-value pairs). | Voice, Chat, Task | Attributes persist for the life of the contact. Accessible as `$.Attributes.{key}`. |
| **Set customer queue flow** | Assigns a customer queue flow to the contact. | Voice, Chat, Task | Determines what the customer experiences while waiting in queue. |
| **Set hold flow** | Assigns a hold flow for the customer or agent hold experience. | Voice | Separate from queue flow; used when agent puts customer on hold. |
| **Set whisper flow** | Assigns a whisper flow (agent-side or customer-side). | Voice, Chat | Plays before agent/customer connection. |
| **Set disconnect flow** | Assigns a flow that runs when the contact disconnects. | Voice, Chat, Task | Used for post-call cleanup, surveys, or ACW automation. |
| **Set event flow** | Assigns a flow that runs on specific events (e.g., customer reconnect in chat). | Chat | Handles chat-specific lifecycle events. |
| **Set working queue** | Sets the queue that the contact will be routed to. | Voice, Chat, Task | Must be set before `Transfer to queue`. |
| **Set callback number** | Sets or overrides the callback phone number for the contact. | Voice | Used before creating a callback. Defaults to customer's ANI. |
| **Set voice** | Sets the TTS voice (Amazon Polly voice) and language for the contact. | Voice | Supports standard and neural voices. Set language code for multilingual flows. |
| **Set logging behavior** | Enables or disables contact flow logging for the contact. | Voice, Chat, Task | Logs go to CloudWatch Logs. Disable for sensitive flows. |
| **Set recording and analytics behavior** | Configures call recording (agent, customer, or both) and Contact Lens analytics. | Voice | Enables/disables recording. Enables Contact Lens for real-time or post-call analytics. |
| **Set recording analytics and processing behavior** | Extended version of recording behavior. Configures recording, Contact Lens, screen recording, and evaluation forms. | Voice, Chat | Newer block with additional analytics options. |
| **Set routing criteria** | Defines agent routing criteria (required/preferred attributes, expiration). | Voice, Chat, Task, Email | Proficiency-based routing with step waterfall. Steps use AND/OR/NOT on predefined attributes with proficiency levels (1–5). Supports preferred agent targeting by user ID (up to 10 agents). Steps expire and relax over time; last step can be non-expiring. Set manually or dynamically via Lambda JSON. See [proficiency-routing.md](proficiency-routing.md) for full details, JSON schema, and Lambda examples. |
| **Set Touchtone Buffer Behavior** | Controls DTMF buffering behavior. | Voice | Enable/disable DTMF buffering. Useful for preventing DTMF bleed between blocks. |
| **Contact tags** | Adds or removes tags on the contact. | Voice, Chat, Task | Tags are searchable in contact search. Used for categorization and reporting. |
| **Set voice** | (See above) | | |

## Branch

Blocks that make routing decisions based on conditions.

| Block | Description | Channels | Key Details |
|-------|-------------|----------|-------------|
| **Check contact attributes** | Branches based on contact attribute values. | Voice, Chat, Task | Supports conditions: Equals, StartsWith, EndsWith, Contains, GreaterThan, LessThan, etc. Multiple conditions per branch. |
| **Check hours of operation** | Branches based on whether the current time is within configured hours of operation. | Voice, Chat, Task | References a hours-of-operation resource. Branches: In Hours, Out of Hours, Error. |
| **Check queue status** | Branches based on queue metrics (queue size, oldest contact age, agents available). | Voice, Chat, Task | Use to make overflow or callback decisions. |
| **Check staffing** | Checks if agents are staffed/available in a queue. | Voice, Chat, Task | Branches: True (agents available), False, Error. |
| **Distribute by percentage** | Routes contacts by percentage distribution across branches. | Voice, Chat, Task | A/B testing, gradual rollouts. Configure up to 11 branches with percentages that sum to 100. |
| **Check call progress** | Checks the progress of an outbound call (answered, voicemail, busy, etc.). | Voice (Outbound) | Used in outbound flows. Branches: Call Answered, Voicemail (beep), Voicemail (no beep), Not Detected, Error. |
| **Get metrics** | Retrieves real-time queue metrics for use in branching. | Voice, Chat, Task | Returns: Queue Size, Oldest Contact in Queue, Agents Online, Agents Available, Agents Staffed, Agents in ACW, Agents on Contact, Agents on Call, Agents Non-Productive, Agents Error. |

## Integrate

Blocks that integrate with external services and AWS services.

| Block | Description | Channels | Key Details |
|-------|-------------|----------|-------------|
| **Invoke AWS Lambda function** | Invokes a Lambda function and uses the response in the flow. | Voice, Chat, Task | Max timeout: 8 seconds. Response max: 32 KB. Returns STRING_MAP or JSON. See lambda-integration.md. |
| **Customer profiles** | Retrieves or creates customer profile data from Amazon Connect Customer Profiles. | Voice, Chat, Task | Search by phone, email, account ID. Returns profile attributes as contact attributes. |
| **Cases** | Creates, updates, or retrieves Amazon Connect Cases. | Voice, Chat, Task | Integrates with the Cases feature for case management workflows. |
| **Data Table** | Reads from or writes to Amazon Connect Data Tables. | Voice, Chat, Task | Key-value lookups without Lambda. |
| **Create task** | Creates a new task contact. | Voice, Chat, Task | Tasks are routed to queues like calls/chats. Set task attributes, description, scheduled time. |
| **Create persistent contact association** | Links related contacts together for continuity. | Voice, Chat, Task | Associates contacts across interactions for the same customer issue. |

## Transfer and Disconnect

Blocks that route contacts to destinations or end the interaction.

| Block | Description | Channels | Key Details |
|-------|-------------|----------|-------------|
| **Transfer to queue** | Transfers the contact to the working queue. | Voice, Chat, Task | Must set working queue first. Branches: At Capacity, Error. |
| **Transfer to agent (beta)** | Transfers directly to a specific agent by agent ID or username. | Voice, Chat, Task | Beta. Specify agent. Branches: At Capacity, Error. |
| **Transfer to phone number** | Transfers the contact to an external phone number. | Voice | Specify country code and number. Optional: caller ID, timeout. |
| **Transfer to flow** | Transfers the contact to another flow. | Voice, Chat, Task | Chain flows together. Max 20 flows in a chain. |
| **Disconnect / hang up** | Ends the contact. | Voice, Chat, Task | Terminates the interaction. Triggers disconnect flow if set. |
| **End flow / Resume** | Ends the current flow execution without disconnecting. | Voice, Chat, Task | Used in event flows and disconnect flows. |
| **Resume contact** | Resumes a paused contact (from a task or after hold). | Voice, Chat, Task | Reconnects the contact after a pause. |

## Flow Control

Blocks that control flow logic and structure.

| Block | Description | Channels | Key Details |
|-------|-------------|----------|-------------|
| **Invoke module** | Calls a flow module (sub-flow). | Voice, Chat, Task | Passes control to the module. Resumes after module's `Return` block. |
| **Return** | Returns from a flow module back to the calling flow. | Voice, Chat, Task | Only valid inside a flow module. |
| **Loop** | Repeats a section of the flow a specified number of times. | Voice, Chat, Task | Configure loop count. Branches: Looping, Complete. |
| **Wait** | Pauses the flow for a specified duration or until an event. | Chat, Task | Configure timeout (up to 7 days for chat/task). Used for async workflows. |

## Media and Streaming

Blocks that control media streaming behavior.

| Block | Description | Channels | Key Details |
|-------|-------------|----------|-------------|
| **Start media streaming** | Starts streaming the customer's audio to a Kinesis Video Stream. | Voice | Enables real-time audio processing. See media-streaming.md. |
| **Stop media streaming** | Stops the media stream. | Voice | Call after processing is complete to free resources. |

## Outbound

Blocks specific to outbound calling flows.

| Block | Description | Channels | Key Details |
|-------|-------------|----------|-------------|
| **Call phone number** | Initiates an outbound call from within a flow. | Voice | Specify the destination number and optional caller ID. |

## Routing

Blocks that influence contact routing priority and behavior.

| Block | Description | Channels | Key Details |
|-------|-------------|----------|-------------|
| **Change routing priority / age** | Adjusts the routing priority or age of the contact in queue. | Voice, Chat, Task | Increase priority to move contact up in queue. Adjust age to simulate longer/shorter wait. |

## Block Count by Category

| Category | Count |
|----------|-------|
| Interact | 10 |
| Set | 15 |
| Branch | 8 |
| Integrate | 6 |
| Transfer and Disconnect | 7 |
| Flow Control | 4 |
| Media and Streaming | 2 |
| Outbound | 1 |
| Routing | 1 |
| **Total** | **53** |

Note: Some blocks appear in slightly different groupings in the Connect console. The categorization above follows the functional purpose of each block.
