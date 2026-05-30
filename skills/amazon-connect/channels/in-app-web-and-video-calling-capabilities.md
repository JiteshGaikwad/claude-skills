# In‑App, Web, Video Calling, and Screen Sharing (Overview)

Amazon Connect supports customer-facing calling experiences that run inside your own web/mobile application. This doc captures the Admin Guide “what/why/how” overview and points to the deeper Chime SDK integration notes.

## What this enables

- Customers can contact you from inside your app (web or mobile).
- You can pass context into Amazon Connect (attributes, profile identifiers, app state).
- Optional video and screen sharing can be enabled.

## Important behavior note (privacy / hold)

During video or screen sharing, agents may continue to see the customer’s video/screen share even when the customer is on hold. If you need different behavior, use a custom CCP and/or a custom communications widget integration.

## Two integration options

### Option 1: Out-of-the-box communications widget

- Configure chat/voice/video/screen sharing from the Communications widgets UI.
- Amazon Connect generates an embed snippet.
- You can apply brand customization and optionally secure the widget so it can only launch from your domains.

### Option 2: Native integration (build your own widget)

- You call `StartWebRTCContact` to create the contact.
- You join the call using the **Amazon Chime SDK** client libraries (iOS, Android, JavaScript).

See `channels/webrtc-chime-sdk-implementation.md` for the detailed steps and multi-user behavior.

## Multi-user calling limits (high level)

Amazon Connect supports adding additional participants to a call (within a fixed max participant limit). Use `CreateParticipant` and `CreateParticipantConnection` for additional users/agents, and design for capacity limits and ordering constraints (agent must be connected before adding participants).

