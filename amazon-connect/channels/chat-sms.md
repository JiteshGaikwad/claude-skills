# Chat and SMS Channel

Amazon Connect supports real-time chat, SMS, and third-party messaging integrations. Agents handle chat contacts through the same CCP used for voice, with unified routing and reporting.

## Web Chat — Communications Widget

Connect provides a hosted, embeddable chat widget for websites and web applications.

**Setup:**
- Configure in the Connect console under "Communications widget"
- Generates a short JavaScript snippet to embed on your site
- Widget is hosted and served by AWS — no infrastructure to manage

**Customization:**
- Font family and size
- Primary color, background color, text color
- Widget button icon and position
- Custom header text and subtitle
- Display name for bot and agent messages

**Security:**
- Widget is secured to your specific domain(s) via allowlisting
- Only renders on approved origins — prevents unauthorized embedding
- Supports JWT-based authentication for identifying logged-in customers
- HTTPS required on the hosting domain

**Embedding — minimal code snippet:**

```html
<!-- Paste in your website's <head> or before </body> -->
<script type="text/javascript">
  (function(w, d, x, id){
    s=d.createElement('script');
    s.src='https://d3xxxxxxxxxxxx.cloudfront.net/amazon-connect-chat-interface-client.js';
    s.async=1;
    s.id=id;
    d.getElementsByTagName('head')[0].appendChild(s);
    w[x] = w[x] || function() { (w[x].ac = w[x].ac || []).push(arguments) };
  })(window, document, 'amazon_connect', 'amazon-connect-chat-widget');

  amazon_connect('styles', { openChat: { color: '#ffffff', backgroundColor: '#0052cc' }});
  amazon_connect('snippetId', 'YOUR_SNIPPET_ID');
</script>
```

**Features:**
- Rich text messages, emojis
- File attachments (images, PDFs, etc.)
- Typing indicators
- Read receipts
- Message delivery status
- Persistent chat (continue previous conversations)

## SMS Channel

Two-way SMS messaging through Amazon Connect, powered by Amazon Pinpoint SMS.

**Setup:**
1. Request a phone number or short code through Amazon Pinpoint SMS
2. Associate the SMS number with your Connect instance
3. Configure a contact flow for inbound SMS
4. Optionally integrate with Amazon Lex for automated responses

**Capabilities:**
- Two-way SMS — customers can initiate conversations and agents can reply
- Lex bot integration for automated self-service responses before agent handoff
- SMS messages routed through the same queues and routing profiles as chat
- Supports long messages (multi-segment SMS)
- Delivery receipts

**Lex auto-response pattern:**
- Customer sends SMS
- Connect contact flow routes to a Lex bot
- Bot handles FAQs, appointment confirmations, status checks
- Escalates to a live agent when needed
- Agent sees the full Lex conversation history in the CCP

## Third-Party Messaging Integrations

Connect supports messaging platforms beyond native chat and SMS.

**WhatsApp Business:**
- Integrate WhatsApp as a messaging channel
- Customers message your WhatsApp Business number
- Messages route through Connect flows and queues
- Agents respond from the CCP — no separate WhatsApp app needed
- Supports text, images, documents, and location sharing

**Facebook Messenger:**
- Connect your Facebook Page to Amazon Connect
- Customer messages from Messenger route to agents
- Supports text, images, and quick replies
- Configured via the Connect console or APIs

**Integration architecture:**
- Third-party messages arrive via Amazon Connect APIs
- Contact flows handle routing logic identically to native chat
- Agents see a unified interface regardless of the originating channel
- Contact records and analytics capture the source channel

## Chat Message Streaming

Subscribe to real-time chat events for building custom integrations, analytics, or monitoring tools.

**Real-time streaming via APIs:**
- Subscribe to new chat contacts and message events
- Receive events as they happen (new message, participant joined/left, typing)
- Build custom dashboards, logging systems, or AI-powered assist tools

**Use cases:**
- Real-time supervisor monitoring of chat conversations
- Custom analytics pipelines for chat content
- AI agent-assist that reads messages and suggests responses
- Compliance logging and archival

**Event types:**
- `PARTICIPANT_JOINED` — agent or customer enters the chat
- `PARTICIPANT_LEFT` — agent or customer leaves
- `MESSAGE` — new message sent by any participant
- `EVENT` — typing indicator, read receipt, attachment
- `DISCONNECT` — chat session ended

## Persistent Chat

Allow customers to return to a previous conversation without losing context. Powered by `CreatePersistentContactAssociation`.

**How it works:**
- When a chat ends, create a persistent association using `CreatePersistentContactAssociation`
- When the customer returns, the widget loads the previous conversation history
- Context carries over — no need for the customer to re-explain their issue
- The new contact can route to the same agent (if available) or a different one

**Key details:**
- Association is tied to a source (e.g., customer ID, session token)
- History is displayed to the agent in the CCP
- Works with both the hosted widget and custom chat implementations
- Persistent associations have a configurable TTL

```javascript
import { ConnectClient, CreatePersistentContactAssociationCommand } from "@aws-sdk/client-connect";

const client = new ConnectClient({ region: "us-east-1" });

await client.send(new CreatePersistentContactAssociationCommand({
  InstanceId: instanceId,
  InitialContactId: initialContactId,
  RehydrationType: "ENTIRE_PAST_SESSION", // or FROM_SEGMENT
  SourceContactId: sourceContactId,
}));
```

## Chat APIs

### StartChatContact

Initiate a new inbound chat contact programmatically (e.g., from a custom UI or backend trigger).

```javascript
import { ConnectClient, StartChatContactCommand } from "@aws-sdk/client-connect";

const client = new ConnectClient({ region: "us-east-1" });

const response = await client.send(new StartChatContactCommand({
  InstanceId: instanceId,
  ContactFlowId: contactFlowId,
  ParticipantDetails: {
    DisplayName: "Jane Customer",
  },
  Attributes: {
    customerName: "Jane",
    accountId: "12345",
  },
  // Optional: set chat duration (minutes)
  ChatDurationInMinutes: 1440, // 24 hours
}));

// response.ContactId — unique ID for this contact
// response.ParticipantId — customer's participant ID
// response.ParticipantToken — token for the customer to connect to the chat
```

### StartOutboundChatContact

Proactively reach out to a customer via chat (e.g., order update, appointment reminder).

```javascript
const response = await client.send(new StartOutboundChatContactCommand({
  InstanceId: instanceId,
  ContactFlowId: outboundFlowId,
  DestinationEndpoint: {
    Type: "CONNECT_PHONENUMBER", // or other endpoint types
    Address: "+14155551234",
  },
  ParticipantDetails: {
    DisplayName: "Support Team",
  },
}));
```

### SendChatIntegrationEvent

Inject events into a chat contact from an external system (e.g., a CRM sending a notification into an active chat).

```javascript
const response = await client.send(new SendChatIntegrationEventCommand({
  SourceId: "external-crm-system",
  DestinationId: destinationId,
  Event: {
    Type: "MESSAGE",
    Content: JSON.stringify({
      Content: "Your order #456 has shipped!",
      ContentType: "text/plain",
    }),
  },
  Subtype: "connect:sms", // or other subtypes
}));
```

## Routing and Queue Behavior

- Chat contacts are routed through the same routing profiles and queues as voice
- Agents can handle multiple concurrent chats (configurable concurrency per routing profile)
- Default chat concurrency is typically 2-5 simultaneous chats
- Chat does not block voice — agents can take a call while handling chats (if configured)
- Queue priority and routing logic apply identically to chat contacts

## Chat Timeouts and Lifecycle

| Timeout | Default | Configurable | Purpose |
|---------|---------|-------------|---------|
| Chat duration | 24 hours | Yes (via `ChatDurationInMinutes`) | Maximum lifetime of a chat session |
| Customer idle | 15 minutes | Yes (via contact flow) | Disconnect if customer stops responding |
| Agent idle | None | Custom via Lambda | Optional agent inactivity detection |
| After-contact work | Same as voice ACW | Yes | Time for agent to wrap up after chat ends |

## Key Considerations

- **Encryption:** All chat messages encrypted in transit (TLS) and at rest
- **Transcripts:** Full chat transcripts stored in S3 (configure in instance settings)
- **Contact Lens:** Chat analytics available — sentiment, categorization, PII redaction
- **Lex integration:** Bots can handle initial interactions before agent handoff
- **Transfer:** Agents can transfer chats to other agents or queues (warm or cold)
- **Attachments:** Supported in chat — configurable size limits and file type restrictions
