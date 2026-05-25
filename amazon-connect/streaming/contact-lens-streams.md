# Contact Lens Real-Time Streaming

Contact Lens real-time analytics can be streamed via Amazon Kinesis Data Streams, providing live transcription, sentiment analysis, category matches, and issue detection during active contacts. This overcomes the scaling limitations of the REST API (`ListRealtimeContactAnalysisSegments`) and enables building low-latency agent assist applications.

## Enabling Contact Lens Streaming

Contact Lens streaming is enabled per-contact via a flow block:

1. Add the **"Set recording, analytics and processing behavior"** block to your contact flow
2. Enable **Contact Lens real-time analytics** within the block
3. Configure the output Kinesis Data Stream in the Connect instance settings

The stream receives events for all contacts that pass through flows with Contact Lens enabled.

## Event Types

| Event Type | When Emitted | Purpose |
|---|---|---|
| `STARTED` | Beginning of Contact Lens analysis | Signals that real-time analysis has begun for a contact |
| `SEGMENTS` | During the contact (continuously) | Contains analyzed data: transcripts, sentiment, categories, issues |
| `COMPLETED` | Contact ends successfully | Signals that analysis finished normally |
| `FAILED` | Analysis encounters an error | Signals that analysis could not complete |

### Event Lifecycle

```
Contact begins -> STARTED -> SEGMENTS (repeated) -> COMPLETED or FAILED
```

- `STARTED` is emitted once per contact when Contact Lens begins processing
- `SEGMENTS` events are emitted continuously throughout the contact as new data is analyzed
- `COMPLETED` or `FAILED` is emitted once when the contact ends

## SEGMENTS Event Data

The `SEGMENTS` event is the primary carrier of analytics data. Each SEGMENTS event contains one or more of the following:

### Transcript Segments

Real-time transcription of the conversation, delivered as individual utterances.

```json
{
  "Transcript": {
    "Id": "segment-uuid",
    "ParticipantId": "AGENT",
    "ParticipantRole": "AGENT",
    "Content": "Thank you for calling, how can I help you today?",
    "BeginOffsetMillis": 1200,
    "EndOffsetMillis": 4500,
    "Sentiment": "POSITIVE"
  }
}
```

**ParticipantRole values:** `AGENT`, `CUSTOMER`

**Sentiment values:** `POSITIVE`, `NEGATIVE`, `NEUTRAL`, `MIXED`

### Utterance Segments (Partial Transcripts)

For ultra-low-latency agent assist scenarios, Contact Lens emits **utterance** segment types containing partial (in-progress) transcriptions before the speaker finishes their turn.

- Partial transcripts update as the speaker continues talking
- Final transcript replaces partials when the utterance is complete
- Enables real-time agent assist UIs that display what the customer is saying as they speak

### Category Matches

When a Contact Lens rule category is triggered during the contact:

```json
{
  "Categories": {
    "MatchedCategories": ["Cancellation Request", "Frustrated Customer"],
    "MatchedDetails": {
      "Cancellation Request": {
        "PointsOfInterest": [
          {
            "BeginOffsetMillis": 15000,
            "EndOffsetMillis": 22000
          }
        ]
      }
    }
  }
}
```

Categories are defined in the Connect console under Contact Lens rules. They use keyword matching, sentiment thresholds, and other criteria.

### Issue Detection

Automatically identified customer issues:

```json
{
  "Issues": {
    "IssuesDetected": [
      {
        "CharacterOffsets": {
          "BeginOffsetChar": 0,
          "EndOffsetChar": 45
        },
        "Text": "I need to cancel my subscription immediately"
      }
    ]
  }
}
```

### Sentiment

Per-utterance sentiment is included in transcript segments. Overall contact sentiment trends can be derived by aggregating utterance-level sentiments over time.

## Voice vs Chat Data Models

Contact Lens uses **separate data models** for voice and chat channels:

**Voice-specific fields:**
- `BeginOffsetMillis` / `EndOffsetMillis` -- time offsets relative to call start
- Audio-level metadata (loudness, talk speed)
- Non-talk time detection

**Chat-specific fields:**
- `AbsoluteTime` -- wall-clock timestamp for each message
- Message type metadata (text, attachment, event)
- No audio-related fields

When building consumers, check the `Channel` field in the contact metadata to determine which data model to expect.

## Consumer Architecture

### Real-Time Agent Assist Pattern

```javascript
import { KinesisClient, GetRecordsCommand } from "@aws-sdk/client-kinesis";

// 1. Consume from the Contact Lens Kinesis stream
// 2. Filter SEGMENTS events by ContactId
// 3. Extract transcript segments for display
// 4. Monitor category matches for alerts
// 5. Push to agent UI via WebSocket (API Gateway + Lambda)
```

### Scaling Advantages Over REST API

| Approach | Scalability | Latency | Use Case |
|---|---|---|---|
| `ListRealtimeContactAnalysisSegments` REST API | Limited by API throttling | Request-response | Low-volume, on-demand queries |
| Kinesis Data Stream | Scales with shard count | Sub-second push | High-volume, real-time agent assist |

The REST API has per-instance throttle limits that become a bottleneck at scale. Kinesis streaming scales horizontally by adding shards.

## Key Considerations

- **Stream configuration:** The Kinesis stream must be in the same region as the Connect instance
- **Shard planning:** Each concurrent contact generates multiple SEGMENTS events per second. Plan shard capacity based on peak concurrent contacts.
- **Ordering:** Events for a single contact are ordered within a shard (ContactId is used as the partition key)
- **Latency:** Transcript segments typically arrive within 1-3 seconds of the spoken word
- **Partial transcripts:** Utterance segments arrive faster but may change -- always use the final transcript segment for persistence
- **Encryption:** Enable server-side encryption on the Kinesis stream; see [data-streaming.md](./data-streaming.md) for KMS key policy setup
- **Cost:** Contact Lens real-time analysis is billed per minute of analyzed audio, in addition to Kinesis stream costs
- **Language support:** Real-time transcription supports a subset of languages compared to post-call -- check the Connect documentation for current language availability
