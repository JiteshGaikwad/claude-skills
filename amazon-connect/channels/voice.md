# Voice Channel

Amazon Connect's voice channel provides cloud-based telephony with softphone and PSTN options, call recording, audio enhancement, and outbound calling capabilities.

## Softphone

The Connect softphone runs in the agent's browser via WebRTC. No physical hardware or PBX infrastructure required.

**Audio quality:**
- 16 kHz wideband audio (twice the sample rate of traditional 8 kHz telephony)
- Resistant to packet loss — maintains call quality even on imperfect network connections
- Agents only need a headset and a supported browser (Chrome, Firefox)

**Requirements:**
- Stable internet connection (minimum 100 Kbps per call)
- UDP port 3478 for TURN relay
- Agents set their phone type to "Softphone" in the Contact Control Panel (CCP)

## PSTN Telephony

Connect supports traditional PSTN telephony for both inbound and outbound calls.

- Claim phone numbers (DID and toll-free) directly from the Connect console
- Port existing numbers to Connect
- Numbers available in 200+ countries and territories
- E.164 international format required for all phone numbers (e.g., `+14155551234`)

## Call Recording

Record calls for quality assurance, compliance, training, and dispute resolution. Recordings are stored in your designated S3 bucket.

**Recording modes:**
- **Agent only** — captures only the agent's audio channel
- **Customer only** — captures only the customer's audio channel
- **Agent and customer** — captures both channels (dual-channel recording)

**Storage:**
- Recordings are stored as WAV files in your configured S3 bucket
- S3 bucket must be in the same AWS region as your Connect instance
- Enable server-side encryption (SSE-S3 or SSE-KMS) for compliance
- Retention policies managed via S3 lifecycle rules

**Enabling recording:**
- Set the "Set recording and analytics behavior" block in the contact flow
- Recording starts when the agent picks up (not during IVR or queue)
- Can be started, stopped, suspended, and resumed programmatically during a call

**Recording APIs:**

| API | Purpose |
|-----|---------|
| `StartContactRecording` | Begin recording a live contact |
| `StopContactRecording` | Permanently stop recording — cannot be restarted after this call |
| `SuspendContactRecording` | Temporarily pause recording (e.g., while customer reads credit card number) |
| `ResumeContactRecording` | Resume a previously suspended recording |

**Example — suspend recording during sensitive data collection:**

```javascript
import { ConnectClient, SuspendContactRecordingCommand, ResumeContactRecordingCommand } from "@aws-sdk/client-connect";

const client = new ConnectClient({ region: "us-east-1" });

// Suspend before collecting PCI data
await client.send(new SuspendContactRecordingCommand({
  InstanceId: instanceId,
  ContactId: contactId,
  InitialContactId: initialContactId,
}));

// ... agent collects sensitive info ...

// Resume recording
await client.send(new ResumeContactRecordingCommand({
  InstanceId: instanceId,
  ContactId: contactId,
  InitialContactId: initialContactId,
}));
```

**Important behavioral notes:**
- `StopContactRecording` is final — once stopped, recording cannot be restarted for that contact
- `SuspendContactRecording` / `ResumeContactRecording` are the correct pair for temporary pauses
- If the contact disconnects while suspended, the partial recording is still saved to S3
- Recordings are available in the contact record after the call ends (not in real time)

## Audio Enhancement

Connect provides real-time audio processing to improve call quality. These features run server-side — no agent hardware changes required.

**Noise suppression:**
- Removes background noise from the agent's environment (keyboard clicks, office chatter, HVAC)
- Also applies to the customer's audio stream
- Enabled per-instance or per-contact-flow

**Voice isolation:**
- Advanced ML-based feature that isolates the primary speaker's voice
- Strips out competing voices and ambient sound more aggressively than basic noise suppression
- Particularly useful for work-from-home or open-office environments

**Agent control:**
- Agents can adjust audio enhancement settings during an active session from the CCP
- Toggle noise suppression on/off based on their current environment
- Changes take effect immediately without interrupting the call

## Outbound Calling

Place outbound calls from Connect for proactive outreach, callbacks, and follow-ups.

**Capabilities:**
- Call 200+ countries and destinations worldwide
- All numbers must be in E.164 format (e.g., `+442071234567` for UK)
- Outbound caller ID can be set per queue or per contact flow
- Supports both agent-initiated (manual dial from CCP) and API-initiated outbound calls

**Outbound campaigns:**
- High-volume outbound dialing via Amazon Connect outbound campaigns
- Predictive, progressive, and agentless dialing modes
- Integrates with Amazon Pinpoint for customer segmentation

**Caller ID considerations:**
- Set the outbound caller ID number in the queue configuration
- Must be a number claimed in your Connect instance
- Some countries require local presence — ensure compliance with local telecom regulations

## Contact Flow Integration

Voice-specific contact flow blocks:

| Block | Purpose |
|-------|---------|
| Set recording and analytics behavior | Enable/configure recording |
| Set voice | Choose Amazon Polly voice for prompts (language, neural/standard) |
| Get customer input | DTMF or Lex bot input during IVR |
| Store customer input | Capture DTMF digits (e.g., account number) |
| Play prompt | Text-to-speech or audio file playback |
| Set hold flow | Define experience while customer is on hold |
| Start media streaming | Stream real-time audio via Kinesis Video Streams |

## Real-Time Audio Streaming

Stream live call audio to external services via Kinesis Video Streams (KVS).

- Use the "Start media streaming" block in the contact flow
- Audio delivered as PCM frames to a KVS stream
- Common use cases: real-time transcription, sentiment analysis, agent assist
- Each contact gets its own KVS stream with a predictable naming convention
- Stop streaming with the "Stop media streaming" block or when the contact ends

## Key Considerations

- **Encryption:** All voice traffic is encrypted in transit (TLS) and recordings can be encrypted at rest (S3 SSE)
- **Compliance:** Supports PCI DSS, HIPAA, SOC, and other compliance frameworks
- **Capacity:** No hard limit on concurrent calls — scales automatically
- **Latency:** Connect uses AWS global infrastructure; choose the region closest to your agents and customers
- **Emergency calling:** Connect does not support 911/emergency calling — not a replacement for traditional phone service in that regard
