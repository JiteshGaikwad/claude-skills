# Web, In-App, and Video Calling

Amazon Connect supports in-app calling, web-based calling, and video calling. Customers initiate contact from within your application or website without switching to a phone — and you can pass contextual data to Connect so agents already know who the customer is.

## Overview

This channel eliminates the friction of traditional phone support. Instead of a customer calling a 1-800 number and navigating an IVR, they tap a button inside your app or website and are connected directly to an agent with full context.

**Key benefits:**
- Customer never leaves your app or website
- Contextual information (logged-in user, current page, cart contents, session data) is passed to Connect automatically
- No re-identification needed — the agent knows who the customer is before they speak
- WebRTC-based — works in modern browsers and mobile apps without plugins
- Supports voice, video, and screen sharing in a single session

## In-App Calling

Embed calling directly into your iOS, Android, or web application.

**How it works:**
1. Customer taps "Call Support" in your app
2. Your app calls the `StartWebRTCContact` API with contextual attributes
3. Connect creates a contact and routes it through a contact flow
4. The agent receives the call with all context (customer name, account, current screen, etc.)
5. Voice (and optionally video) is established via WebRTC

**Contextual data flow:**
- Your app passes attributes at call initiation (user ID, order number, page URL, error codes)
- Contact flow receives these as contact attributes
- Agent sees them in the CCP before accepting the call
- No IVR prompts needed — "What's your account number?" becomes unnecessary

**Mobile SDKs:**
- iOS: Amazon Connect Participant SDK for iOS
- Android: Amazon Connect Participant SDK for Android
- Both SDKs handle WebRTC negotiation, oICE candidates, oaudio/video streams

## Web Calling — Communications Widget

For websites, Connect provides the same hosted communications widget used for chat, extended with voice and video capabilities.

**Setup:**
- Enable voice/video in the communications widget configuration
- Same short JavaScript snippet as chat — add calling with minimal code changes
- Widget handles the WebRTC connection, oUI controls, omute/unmute, ovideo toggle

**Widget capabilities:**
- Voice calling from the browser
- Video calling (customer and agent)
- Seamless escalation from chat to voice/video within the same widget
- Customer does not need to install anything — works in Chrome, Firefox, Edge, Safari

**Embedding:**

```html
<script type="text/javascript">
  (function(w, d, x, id){
    s=d.createElement('script');
    s.src='https://d3xxxxxxxxxxxx.cloudfront.net/amazon-connect-chat-interface-client.js';
    s.async=1;
    s.id=id;
    d.getElementsByTagName('head')[0].appendChild(s);
    w[x] = w[x] || function() { (w[x].ac = w[x].ac || []).push(arguments) };
  })(window, document, 'amazon_connect', 'amazon-connect-widget');

  amazon_connect('snippetId', 'YOUR_SNIPPET_ID');
  amazon_connect('supportedMessagingContentTypes', [
    'text/plain',
    'text/markdown',
    'application/vnd.amazonaws.connect.message.interactive',
  ]);
</script>
```

## StartWebRTCContact API

The primary API for initiating in-app and web-based calls.

```javascript
import { ConnectClient, StartWebRTCContactCommand } from "@aws-sdk/client-connect";

const client = new ConnectClient({ region: "us-east-1" });

const response = await client.send(new StartWebRTCContactCommand({
  InstanceId: instanceId,
  ContactFlowId: contactFlowId,
  // Participant details
  ParticipantDetails: {
    DisplayName: "Jane Customer",
  },
  // Pass contextual data from your app
  Attributes: {
    customerId: "CUST-12345",
    currentPage: "/orders/789",
    accountTier: "Premium",
    deviceType: "iOS",
    appVersion: "3.2.1",
    sessionId: "sess-abc-def",
  },
  // Optional: enable video
  AllowedCapabilities: {
    Customer: {
      Video: "SEND", // Customer can send video
    },
    Agent: {
      Video: "SEND", // Agent can send video
    },
  },
  // Optional: related contact for context continuity
  RelatedContactId: relatedContactId,
}));

// response.ContactId — unique contact ID
// response.ParticipantId — customer's participant ID
// response.ParticipantToken — token for the customer to join the WebRTC session
// response.ConnectionData — ICE servers and signaling info for WebRTC setup
```

**ConnectionData** contains:
- ICE server URLs (STUN/TURN) for NAT traversal
- Signaling endpoint for SDP exchange
- Credentials for the WebRTC session

**Establishing the WebRTC connection (client-side):**

```javascript
// After receiving ConnectionData from StartWebRTCContact
const peerConnection = new RTCPeerConnection({
  iceServers: connectionData.IceServers.map(server => ({
    urls: server.Urls,
    username: server.Username,
    credential: server.Password,
  })),
});

// Add local audio (and video if enabled)
const localStream = await navigator.mediaDevices.getUserMedia({
  audio: true,
  video: true, // set to false for voice-only
});
localStream.getTracks().forEach(track => {
  peerConnection.addTrack(track, localStream);
});

// Handle remote stream (agent's audio/video)
peerConnection.ontrack = (event) => {
  const remoteVideo = document.getElementById('remote-video');
  remoteVideo.srcObject = event.streams[0];
};

// ICE candidate exchange via Connect signaling channel
// ... (handled by the Connect Participant SDK in production)
```

## Video Calling

Video extends the voice channel with face-to-face interaction.

**Capabilities:**
- Two-way video between customer and agent
- Customer and agent can independently toggle video on/off during the call
- Video is optional — either party can participate with voice-only
- Video quality adapts to available bandwidth

**Use cases:**
- Technical support with visual troubleshooting ("show me what you see")
- Identity verification (document review via camera)
- Healthcare telehealth consultations
- Financial advisory meetings
- Insurance claims — customer shows damage via video

**Agent experience:**
- Agent sees the customer's video feed in the CCP (if the customer enables video)
- Agent can toggle their own camera on/off
- Video does not affect voice quality — they run on separate media tracks
- Agent can handle video calls alongside other contact types per routing profile

## Screen Sharing

Share screens during a call for collaborative troubleshooting and guided walkthroughs.

**StartScreenSharing API:**

```javascript
import { ConnectClient, StartScreenSharingCommand } from "@aws-sdk/client-connect";

const client = new ConnectClient({ region: "us-east-1" });

await client.send(new StartScreenSharingCommand({
  InstanceId: instanceId,
  ContactId: contactId,
  // Screen sharing configuration
}));
```

**Capabilities:**
- Agent shares their screen with the customer (guided walkthrough)
- Customer shares their screen with the agent (troubleshooting)
- Selective sharing — share entire screen, specific window, or browser tab
- Screen share can be started and stopped during the call without disconnecting

**Use cases:**
- Agent walks customer through a form or application step-by-step
- Customer shows agent an error message or confusing UI element
- Agent demonstrates how to navigate a portal or complete a process
- Technical support for software configuration

**Privacy and security:**
- Screen sharing requires explicit consent from the sharing party
- The browser prompts to select what to share (screen, window, or tab)
- Sharing can be stopped at any time by either party
- Screen share data is encrypted in transit via DTLS-SRTP (same as WebRTC media)

## Architecture

```
Customer App/Website
    |
    |-- StartWebRTCContact API --> Amazon Connect
    |                                   |
    |-- WebRTC (STUN/TURN) -----------> Contact Flow
    |   (audio + video + screen)        |
    |                                   +--> Queue --> Agent CCP
    |                                   |
    |                                   +--> Contact Record (CTR)
    |
    +-- Context Attributes -----------> Agent sees customer info
        (userId, page, session)         before answering
```

## Contact Flow Integration

Web/video contacts flow through standard Connect contact flows with additional capabilities.

| Block | Purpose |
|-------|---------|
| Check contact attributes | Branch on contextual data passed from the app (e.g., accountTier, deviceType) |
| Set contact attributes | Enrich with additional data from Lambda or flow logic |
| Transfer to queue | Route based on context (Premium customers to specialized queue) |
| Play prompt | Audio prompts play to the customer while waiting |
| Get customer input | DTMF or Lex interaction (voice-only, not video-specific) |

**Context-based routing example:**
- Customer calls from the billing page of your app
- `currentPage` attribute is `/billing`
- Contact flow checks this attribute and routes directly to the billing queue
- Agent receives the call with billing context — no "How can I help you?" needed

## Routing Behavior

- Web/video contacts are routed through the same routing profiles as voice
- They consume a voice slot in the agent's concurrency configuration
- Priority and queue delay settings apply
- Agents can receive web calls mixed with PSTN calls based on queue membership
- Video capability does not affect routing — it is an optional media upgrade during the call

## Key Considerations

- **Browser support:** WebRTC requires Chrome, Firefox, Edge, or Safari (latest versions)
- **Bandwidth:** Voice requires ~100 Kbps; video requires 300 Kbps-1.5 Mbps depending on quality
- **Firewall/NAT:** TURN servers handle NAT traversal; ensure UDP 3478 is open
- **Mobile:** Native SDKs handle WebRTC complexity; web SDK works in mobile browsers
- **Recording:** WebRTC calls can be recorded same as PSTN calls (audio only — video is not recorded)
- **Encryption:** All WebRTC media encrypted with DTLS-SRTP; signaling encrypted with TLS
- **Fallback:** If WebRTC fails (bad network), consider offering a callback on PSTN as fallback
- **No emergency calling:** Same as standard Connect — WebRTC calls do not support 911/emergency services
- **Concurrent sessions:** Each WebRTC session consumes resources; monitor CloudWatch metrics for capacity
- **Contact Lens:** Real-time and post-contact analytics apply to WebRTC voice (transcription, sentiment, etc.)
