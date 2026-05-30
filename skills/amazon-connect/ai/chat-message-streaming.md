# Amazon Connect — Chat Message Streaming

Message streaming for AI-powered chat lets responses from AI agents and bots appear **progressively as they're generated**, rather than waiting for the complete response before showing anything to the customer. This produces a "growing text bubble" where words appear in real time, similar to watching someone type — improving the perceived responsiveness of generative-AI chat interactions in Amazon Connect.

> **Two different "streaming" features — don't confuse them.**
> - **AI message streaming** (this doc) — streams *partial AI/bot responses* to the customer inside the chat for a natural, progressive experience. Controlled by the `MESSAGE_STREAMING` instance attribute.
> - **Real-time chat message streaming** (separate feature) — subscribes an SNS endpoint to a real-time stream of chat message *events* (used for third-party messaging integrations, push notifications, analytics). Uses `StartContactStreaming` + SNS. See the chat message streaming / SNS doc.
>
> If you're enabling progressive AI responses for conversational AI interactions, this is the right doc.

---

## What it is

With **standard** chat responses, the customer waits while the AI generates its entire response, then the complete message appears all at once. With **AI message streaming**, the customer sees a growing text bubble where words appear progressively as the AI generates them.

| Aspect | Standard Chat | Streaming Chat |
|---|---|---|
| Response display | Complete message appears all at once | Text appears progressively (growing bubble) |
| Customer experience | Wait for full response with loading indicator | See words appear in real time |
| Perceived wait time | Longer (waiting for complete response) | Shorter (immediate visual feedback) |
| Conversation feel | Transactional | Natural, like chatting with a person |
| Fulfillment messages | Not available | AI can send interim status updates |
| Lex timeout handling | Subject to Lex timeout limits | Eliminates Lex timeout limitations |

### Benefits

- **Reduced perceived wait time** — customers see immediate activity instead of staring at a loading spinner.
- **More natural conversation flow** — progressive text mimics human typing.
- **Better engagement** — customers can start reading the response while it's still being generated.
- **Fulfillment messages** — AI agents can provide interim messages like *"One moment while I review your account"* during processing.
- **Eliminates Amazon Lex timeout limitations.**

---

## Integration options

Message streaming for AI-powered chat supports the following integration options:

- **Amazon Connect AI agents**
  - Eliminates Amazon Lex timeout limitations.
  - Provides fulfillment messages during processing (such as *"One moment while I review your account"*).
  - Displays partial responses with progressive text (growing text bubble).
- **Third-party bots via Amazon Lex or Lambda**
  - Eliminates Amazon Lex timeout limitations.
  - Standard bot response behavior.

---

## Enablement status: the December 2025 behavior split

Availability depends on **when the Amazon Connect instance was created**.

- **Instances created starting December 2025** are **automatically opted in**. The `MESSAGE_STREAMING` instance attribute is set to `true` by default — no additional configuration is required.
- **Instances created before December 2025** may need message streaming **enabled manually**, either through the API or the console. Check the instance's `MESSAGE_STREAMING` attribute and enable it if needed.

---

## Enable message streaming

### Option 1 — Using the API

Use the **`UpdateInstanceAttribute`** API to set the **`MESSAGE_STREAMING`** attribute to `true`.

```bash
aws connect update-instance-attribute \
  --instance-id your-instance-id \
  --attribute-type MESSAGE_STREAMING \
  --value true
```

To opt out, set the attribute to `false`.

> **When you enable via the API, you must add the required `lex:RecognizeMessageAsync` permission manually** (see below). The API path does **not** add it for you.

### Option 2 — Using the console

For **newly created instances**, message streaming is enabled by default.

For **existing instances**:

1. Open the Amazon Connect console and choose your instance.
2. In the navigation pane, choose **Flows > Amazon Lex bots**.
3. Under **Lex bots configuration**, select **Enable message streaming in Amazon Connect**.

> **When you enable streaming via the console, the required `lex:RecognizeMessageAsync` permission is automatically added to the bot alias resource-based policy.** When using the API, you must add it manually.

---

## Required Lex bot permission

AI message streaming requires the **`lex:RecognizeMessageAsync`** permission so Amazon Connect can invoke the asynchronous message recognition API that enables streaming responses. You update the **resource-based policy on each Amazon Lex bot alias** used by the instance.

### When you need to update the bot's resource-based policy

- **New instances** — any newly associated Amazon Lex bot alias will have `lex:RecognizeMessageAsync` in its alias policy by default.
- **New bot associations** — when you associate a new Lex bot with the instance, the required permission is automatically included in the bot's resource-based policy; no additional configuration is needed.
- **Existing instances with existing bots** — if the instance previously used Amazon Lex and you enable message streaming now, you **must** update the resource-based policy on **all** associated Lex bot aliases to include the new permission. This also applies to any bot associated **before** AI message streaming was enabled.

### Example Lex bot alias resource-based policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "connect-us-west-2-INSTANCE_ID",
      "Effect": "Allow",
      "Principal": {
        "Service": "connect.amazonaws.com"
      },
      "Action": [
        "lex:RecognizeMessageAsync",
        "lex:RecognizeText",
        "lex:StartConversation"
      ],
      "Resource": "arn:aws:lex:us-west-2:111122223333:bot-alias/MY_BOT/MY_BOT_ALIAS",
      "Condition": {
        "StringEquals": {
          "AWS:SourceAccount": "111122223333"
        },
        "ArnEquals": {
          "AWS:SourceArn": "arn:aws:connect:us-west-2:111122223333:instance/INSTANCE_ID"
        }
      }
    }
  ]
}
```

You can add this permission programmatically by calling the Amazon Lex **`UpdateResourcePolicy`** API to add the `lex:RecognizeMessageAsync` action for the Amazon Connect instance ARN resource.

To update an existing bot policy via the console:

1. Navigate to the Amazon Lex console.
2. Select your bot and go to **Resource-based policy**.
3. Add the `lex:RecognizeMessageAsync` action to the policy statement that grants Amazon Connect access.
4. Save the updated policy.

---

## Incremental message responses (growing message bubble)

> **Incremental message responses only work with Amazon Connect AI agents of type Orchestration.**

To enable incremental responses, start a chat with a **`ParticipantConfiguration`** and set **Response Mode** to **`INCREMENTAL`**. The default Response Mode is **`COMPLETE`**.

---

## Interaction with Orchestration AI agents

Orchestration (orchestrator) AI agents **require chat streaming to be enabled** for chat contacts. Without chat streaming enabled, **some messages will fail to render**.

Orchestrator AI agents only display messages to customers when the model's response is wrapped in `<message>` tags. The prompt instructions must specify this formatting, otherwise customers will not see any messages from the AI agent. Example formatting requirement in the system prompt:

```
<formatting_requirements>
MUST format all responses with this structure:
<message>
Your response to the customer goes here. This text will be spoken aloud, so write
naturally and conversationally.
</message>
```

For agentic self-service chat, enabling AI message streaming is the recommended path for an end-to-end streaming experience.

---

## Timeout limits

- **Standard chat experience** — 10-second timeout.

(Streaming eliminates the Amazon Lex timeout limitations that otherwise apply to standard chat/bot responses.)

---

## Testing the streaming experience

You can test streaming using the Amazon Connect **Communications Widget** (the out-of-the-box, embeddable chat interface) or your own chat widget:

1. In the Amazon Connect console, go to **Channels > Communications widget**.
2. Choose **Add widget**, give it a name/description, ensure **Add chat** is selected, and pick a chat contact flow whose **Basic Settings** are configured (creates the AI session, logging, etc.).
3. Embed the generated JavaScript snippet on a test page and open it in a browser.

**What to look for** to confirm streaming is working:

| Behavior | Non-streaming (standard) | Streaming |
|---|---|---|
| Initial display | Loading indicator or typing dots | Text starts appearing immediately |
| Text appearance | Complete message appears all at once | Words appear progressively (growing bubble) |
| Response timing | Wait until AI finishes generating | See response as it's being generated |
| Visual effect | "Pop" of complete text | Smooth, flowing text like watching someone type |

---

## Limitations and notes

- Incremental (growing-bubble) responses are limited to **Orchestration**-type AI agents.
- Orchestration AI agents need chat streaming enabled or some messages will fail to render, and only `<message>`-wrapped model output is shown to customers.
- This feature applies to the **chat / messaging channel** in Amazon Connect.
- API-based enablement requires manually adding `lex:RecognizeMessageAsync` to each Lex bot alias policy; console-based enablement adds it automatically.
- Default Response Mode is `COMPLETE`; you must explicitly request `INCREMENTAL` per chat to get progressive display.

---

## Quick reference

| Item | Value |
|---|---|
| Instance attribute | `MESSAGE_STREAMING` (`true` / `false`) |
| Enable via API | `UpdateInstanceAttribute` with `--attribute-type MESSAGE_STREAMING --value true` |
| Required Lex permission | `lex:RecognizeMessageAsync` (on each Lex bot alias resource-based policy) |
| Add permission via API | Lex `UpdateResourcePolicy` |
| Auto-enabled | Instances created starting December 2025 |
| Incremental responses | `ParticipantConfiguration` Response Mode = `INCREMENTAL` (Orchestration AI agents only); default `COMPLETE` |
| Channel | Chat / messaging |
