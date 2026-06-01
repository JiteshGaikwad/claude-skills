# Contact Lens Real-Time Segment Streams (Kinesis Data Streams)

Contact Lens real-time analytics can be streamed to **Amazon Kinesis Data Streams** (KDS — not Firehose) for voice and chat. This overcomes the scaling limits of the segment REST APIs and, for **voice**, adds an **`Utterance`** segment (partial transcripts) for ultra-low-latency live agent assist.

> The aggregate post-call analytics (overall sentiment, talk time, interruptions, non-talk time, `JobDetails`, `CustomModels`, `Participants`, the full `ConversationCharacteristics` object) are part of the **post-call S3 output file**, not the segment stream — see [analytics/contact-lens.md](../analytics/contact-lens.md). This doc covers the **stream** payload only.

---

## Enable segment streams

Segment streams are **not** enabled by a flow block alone — you associate a Kinesis stream via the **`AssociateInstanceStorageConfig`** API:

1. **Create a Kinesis Data Stream** in the **same account and Region** as the Connect instance. (Use a separate stream per data type — segments vs agent events vs CTRs.)
2. **(Recommended) Server-side encryption** — three KMS options:
   - Kinesis AWS-managed key `aws/kinesis` (no setup);
   - the **same customer-managed key** already used for recordings/transcripts (grant already exists);
   - a **different CMK** — add a grant for the Connect service-linked role:
     ```bash
     aws kms create-grant --key-id <KEY_ID> \
       --grantee-principal arn:aws:iam::111122223333:role/aws-service-role/connect.amazonaws.com/AWSServiceRoleForAmazonConnect_<id> \
       --operations GenerateDataKey \
       --retiring-principal arn:aws:iam::111122223333:role/adminRole
     ```
3. **Associate the stream** with `aws connect associate-instance-storage-config`, `StorageType: KINESIS_STREAM`, `KinesisStreamConfig.StreamArn`, and resource type:
   - **`REAL_TIME_CONTACT_ANALYSIS_VOICE_SEGMENTS`** (voice)
   - **`REAL_TIME_CONTACT_ANALYSIS_CHAT_SEGMENTS`** (chat)
   - `REAL_TIME_CONTACT_ANALYSIS_SEGMENTS` is **deprecated** (voice-only, still supported; no migration needed if already used).
4. **Enable Contact Lens** for the instance and turn on real-time analytics in the flow (`Set recording and analytics behavior` block) — this is what makes analysis happen; the stream association above is what delivers it to Kinesis.

---

## Event types & envelope

| Event | When |
|---|---|
| `STARTED` | Beginning of a contact analysis session. |
| `SEGMENTS` | Repeatedly during the session — a list of analyzed segments. |
| `COMPLETED` / `FAILED` | End of the session. |

```
STARTED → SEGMENTS (repeated) → COMPLETED | FAILED
```

**Common envelope fields differ by channel:**

| | Voice (schema `1.0.0`) | Chat (schema `2.0.0`) |
|---|---|---|
| Event-type field | **`EventType`** | **`StreamingEventType`** |
| Common fields | `Version, Channel, AccountId, ContactId, InstanceId, LanguageCode, EventType` | `Version, Channel, AccountId, InstanceId, ContactId, StreamingEventType, StreamingSettings` |
| Settings | LanguageCode is top-level | `StreamingSettings { LanguageCode, Output, RedactionTypes, RedactionTypesMetadata }` |
| Redaction selector | — | `OutputType` (`Raw`/`Redacted`) on SEGMENTS |

`Channel` valid values: `VOICE`, `CHAT`, `TASK`.

---

## Segment types (by channel)

Each item in a `SEGMENTS` event's list has exactly **one** segment type:

- **Voice:** `Utterance`, `Transcript`, `Categories`, `PostContactSummary`. (`IssuesDetected` is **nested inside `Transcript`**, not its own segment.)
- **Chat:** `Attachments`, `Categories`, `Event`, `Issues`, `Transcript`, `PostContactSummary`. (`Issues` **is** its own segment, with `TranscriptItems`.)

### Utterance (voice only — partial transcripts)

The low-latency feature: partial text as the customer/agent speaks, before the final `Transcript`.

```json
{
  "Utterance": {
    "Id": "uuid",
    "TranscriptId": "uuid",
    "ParticipantId": "CUSTOMER",
    "ParticipantRole": "CUSTOMER",
    "PartialContent": "I need to cancel my",
    "BeginOffsetMillis": 160,
    "EndOffsetMillis": 2120
  }
}
```

Utterances are faster but **may change** — use the final `Transcript` segment for persistence.

### Transcript

```json
{
  "Transcript": {
    "Id": "uuid",
    "ParticipantId": "CUSTOMER",
    "ParticipantRole": "CUSTOMER",
    "Content": "My name is [PII] and I need help.",
    "BeginOffsetMillis": 160,
    "EndOffsetMillis": 4640,
    "Sentiment": "NEUTRAL",
    "IssuesDetected": [
      { "CharacterOffsets": { "BeginOffsetChar": 0, "EndOffsetChar": 55 }, "Text": "I need to cancel my plan subscription" }
    ],
    "Redaction": { "RedactedTimestamps": [ { "BeginOffsetMillis": 3290, "EndOffsetMillis": 3620 } ] }
  }
}
```

- **Voice** transcript: time offsets (`BeginOffsetMillis`/`EndOffsetMillis`), and (when speech analytics produce them) loudness data; `IssuesDetected` nested here.
- **Chat** transcript adds `DisplayName`, `ContentType`, and uses **`Time.AbsoluteTime`** (ISO-8601 wall clock, e.g. `2024-03-14T19:39:26.715Z`) instead of millis offsets.
- `Sentiment`: `POSITIVE` / `NEGATIVE` / `NEUTRAL` / `MIXED`.

### Redaction

A `Redaction.RedactedTimestamps[]` appears on a turn only when it contains PII. Multiple PII in a turn → offsets ordered (first offset = first PII). For voice, the corresponding audio is silenced in the redacted recording.

### Categories

```json
{
  "Categories": {
    "MatchedCategories": ["Cancellation"],
    "MatchedDetails": {
      "Cancellation": { "PointsOfInterest": [ { "BeginOffsetMillis": 7370, "EndOffsetMillis": 11190 } ] }
    }
  }
}
```

Voice `PointsOfInterest` use millis offsets; **chat** uses `TranscriptItems` (each with `Id` + `CharacterOffsets`).

### Chat-only: Issues, Event, Attachments

- **`Issues`** — standalone segment with `TranscriptItems` (id + character offsets pointing at the issue text).
- **`Event`** — chat participant events, e.g. `EventType: application/vnd.amazonaws.connect.event.participant.left`.
- **`Attachments`** — attachment metadata exchanged in the chat.

### PostContactSummary

```json
{ "PostContactSummary": { "Content": "Customer asked to cancel; agent applied a discount and retained the account.", "Status": "COMPLETED" } }
```

Emitted near the end of the session (a top-level SEGMENTS property for voice). `Status` e.g. `COMPLETED` / `FAILED`.

---

## Consumer architecture

```javascript
import { KinesisClient, GetRecordsCommand } from "@aws-sdk/client-kinesis";

// 1. Consume from the segment-stream Kinesis stream
// 2. Branch on Channel + EventType/StreamingEventType
// 3. Voice: show Utterance for live text, persist on final Transcript
// 4. Surface Categories / nested IssuesDetected for real-time alerts
// 5. Push to the agent UI via WebSocket (API Gateway + Lambda)
```

### Scaling vs the segment REST APIs

| Approach | Scalability | Latency | Use case |
|---|---|---|---|
| `ListRealtimeContactAnalysisSegments` (voice) / `...SegmentsV2` (chat) | Limited by API throttling | Request-response | Low-volume, on-demand |
| Kinesis segment stream | Scales with shard count | Sub-second push (Utterance faster) | High-volume live agent assist |

---

## Key considerations

- Stream must be in the **same account and Region** as the instance.
- **Shard planning:** each concurrent contact emits multiple SEGMENTS/Utterance events per second — size shards to peak concurrency.
- **Utterance segments may change** — persist from the final `Transcript`, not partials.
- **Encryption:** enable SSE on the stream (see KMS options above and [data-streaming.md](data-streaming.md)).
- **Redaction** in the stream uses the same entity selection as post-call; configured when you enable analytics (not real-time call analytics → no redaction for live voice; see [analytics/contact-lens.md](../analytics/contact-lens.md)).
- The Connect docs for this feature do **not** specify a partition key or ordering guarantee — don't assume ContactId-as-partition-key.
- Real-time language support is a subset of post-call — check the supported-languages matrix.
