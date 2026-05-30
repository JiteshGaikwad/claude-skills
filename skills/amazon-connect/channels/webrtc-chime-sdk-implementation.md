# WebRTC Calling + Video + Screen Sharing — Chime SDK Implementation Notes

This page consolidates the Admin Guide’s “native integration” path for in‑app/web/video calling and screen sharing using Amazon Connect APIs plus Amazon Chime SDK client libraries.

## High-level sequence (native integration)

1. Customer uses your web/mobile app to start a call.
2. Your app/backend calls **`StartWebRTCContact`** and passes attributes/context for routing.
3. Your client joins the meeting using the response details with the **Amazon Chime SDK** client libraries.
4. (Optional) Use Participant Service APIs for DTMF and multi-user participation.
5. Contact is routed via flow/queue, agent accepts.
6. (Optional) Video/screen sharing can be started if enabled for both sides.

## Joining with Chime SDK (client side)

The Admin Guide describes this pattern:

- Construct a `MeetingSessionConfiguration` from `StartWebRTCContact` response.
- Instantiate `DefaultMeetingSession`.
- Start audio/video via the Chime SDK `audioVideo` APIs.

## DTMF (Participant Service)

To send DTMF tones:

1. Call `CreateParticipantConnection` to obtain a `ConnectionToken` (use the participant token from `StartWebRTCContact` response).
2. Call `SendMessage` with `contentType` set to `audio/dtmf`.

## Device selection (Chime SDK)

JavaScript patterns:

- List devices: `listAudioInputDevices()`, `listAudioOutputDevices()`
- Start input: `startAudioInput(device)`
- Choose output: `chooseAudioOutput(deviceId)`

## Mute/unmute

- JS: `realtimeMuteLocalAudio()` / `realtimeUnmuteLocalAudio()`

## Local video

- Start: `startLocalVideoTile()`
- Stop: `stopLocalVideoTile()`

## Remote video tiles (agent video)

To show agent video in your app:

- Add an observer (JS: `audioVideo.addObserver(observer)`).
- Bind tiles to UI elements (JS: `bindVideoElement(tileId, videoElement)`).
- To stop receiving video, unbind (`unbindVideoElement(tileId)`).

## Data messages (status signaling)

Use data messages to signal end-user UI state (for example: “on hold” message, or advising that video/screen share is still being transmitted).

- iOS/Android: `realtimeSendDataMessage(topic, data, lifetimeMs)`

## Multi-user participation (additional participants)

Behavioral rules to design around:

- Additional users join via `CreateParticipant` + `CreateParticipantConnection`.
- `CreateParticipant` can fail until the original caller has connected to an agent; handle this ordering constraint explicitly.
- Enforce the maximum participant count (agent + initial user + additional users).
- If the SDK reports “attendee no longer valid”, re-create participant credentials (new `CreateParticipant` + `CreateParticipantConnection`).

Contact-record implications:

- Each additional user connection creates a new contact record.
- Additional contacts reference the original via `PreviousContactId` and use initiation method `WEBRTC_API`.

## Capacity and error handling

Design for:

- “meeting is at capacity” status codes when too many participants attempt to join.
- invalid/expired participant credentials after disconnects.
- backoff/retry loops that do not hammer `CreateParticipant*` while waiting for agent connection.

