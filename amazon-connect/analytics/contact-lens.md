# Contact Lens — Conversational Analytics

Contact Lens provides conversational analytics for voice, chat, and email channels in Amazon Connect. It operates in two modes for voice and chat (real-time and post-contact) and in post-contact mode for email (since email is inherently asynchronous).

---

## Real-Time Analytics (Voice & Chat)

Real-time Contact Lens analyzes conversations as they happen, enabling supervisors and automated systems to intervene during live interactions.

### Capabilities

- **Live issue detection** — Identify customer frustration, compliance violations, or escalation triggers during the call/chat.
- **Supervisor alerts** — Rules fire in real-time and can trigger alerts on the supervisor dashboard, send EventBridge events, or invoke Lambda functions.
- **Real-time categories** — Custom category rules evaluated continuously against the in-progress transcript.
- **Live transcript** — Streaming transcript visible on the Contact details page for in-progress contacts.

### Enabling Real-Time

Real-time analytics is enabled per contact flow using the `Set recording and analytics behavior` block. You must explicitly enable real-time analytics; post-contact analytics alone does not provide streaming data.

---

## Post-Contact Analytics (Voice, Chat & Email)

Post-contact analysis runs after the interaction ends and produces a comprehensive analytics artifact stored in S3.

### Output Location

Analytics files are written to the S3 bucket configured in the instance storage settings under `Analysis - Voice/Chat/Email`. The file format is JSON and includes all analytics dimensions described below.

---

## Sentiment Analysis

Contact Lens assigns sentiment at multiple granularities:

| Level | Description |
|---|---|
| **Per-turn sentiment** | Each utterance/message is classified as `POSITIVE`, `NEGATIVE`, or `NEUTRAL`. |
| **Overall sentiment** | A numeric score from **-5** (most negative) to **+5** (most positive) computed per participant (agent and customer separately). |
| **Sentiment shift** | Indicates whether sentiment improved, worsened, or remained stable between the beginning and end of the conversation. Useful for identifying whether the agent successfully de-escalated. |

### Sentiment Score Interpretation

| Score Range | Meaning |
|---|---|
| +3 to +5 | Strongly positive |
| +1 to +2 | Mildly positive |
| 0 | Neutral |
| -1 to -2 | Mildly negative |
| -3 to -5 | Strongly negative |

---

## Transcription

Contact Lens produces high-accuracy transcripts for voice interactions using Amazon Transcribe under the hood.

### Features

- **Speaker identification** — Each segment is labeled as `AGENT` or `CUSTOMER`. For multi-party calls (conferences), additional participants are identified.
- **Timestamps** — Every segment includes `BeginOffsetMillis` and `EndOffsetMillis` relative to the start of the recording.
- **Confidence scores** — Word-level confidence scores for transcript accuracy.
- **Custom vocabularies** — Support for domain-specific terms to improve transcription accuracy.
- **Redacted transcripts** — A separate redacted version with PII replaced by placeholders (see PII Redaction below).

---

## Categories

Categories allow you to automatically classify contacts based on rules you define.

### Rule Types

| Rule Type | Description |
|---|---|
| **Keywords/phrases** | Match exact words or phrases spoken by agent or customer. |
| **Sentiment** | Match based on sentiment of a turn or overall. |
| **Interruptions** | Match when interruption count exceeds a threshold. |
| **Non-talk time** | Match when silence duration exceeds a threshold. |
| **Composite** | Combine multiple conditions with AND/OR/NOT logic. |

### Evaluation Modes

- **Post-contact** — Rules evaluated after the full transcript is available. Most accurate.
- **Real-time** — Rules evaluated incrementally during the conversation. Enables live alerting.

### Use Cases

- Compliance monitoring (agent read disclosure script)
- Escalation detection (customer mentions "supervisor" or "cancel")
- Quality assurance categorization (greeting compliance, hold procedure)

---

## PII Redaction

Contact Lens can detect and redact personally identifiable information from transcripts, audio recordings, and email content.

### Supported PII Entity Types

- Credit/debit card numbers
- Social Security numbers
- Names
- Addresses
- Email addresses
- Phone numbers
- Bank account numbers
- Date of birth
- Other custom-defined sensitive data

### Redaction Targets

| Target | Description |
|---|---|
| **Transcript** | PII replaced with `[PII]` placeholder in the text transcript. |
| **Audio** | PII segments replaced with silence in the audio recording file. |
| **Email** | PII redacted from email body and subject in analytics output. |

### Configuration

PII redaction is configured per contact flow. You can choose which entity types to redact and whether to redact from transcript only, audio only, or both. The original (unredacted) files can optionally be retained in a separate S3 location with restricted access.

---

## Theme Detection

Theme detection uses unsupervised machine learning to identify recurring topics across your contact center interactions.

- Automatically groups contacts by emerging themes without requiring predefined categories.
- Helps identify new or trending issues that you haven't built categories for yet.
- Available in the Contact Lens dashboard in the Amazon Connect console.
- Themes are surfaced with representative phrases and contact counts.

---

## Talk Time Metrics

Contact Lens measures detailed talk time breakdowns for voice interactions:

| Metric | Description |
|---|---|
| **Total conversation duration** | End-to-end duration of the voice interaction. |
| **Agent talk time** | Total time the agent was speaking. |
| **Customer talk time** | Total time the customer was speaking. |
| **Non-talk time** | Total silence duration (neither party speaking). |
| **Agent talk time %** | Agent talk time as a percentage of total conversation. |
| **Customer talk time %** | Customer talk time as a percentage of total conversation. |
| **Non-talk time %** | Silence as a percentage of total conversation. |
| **Longest non-talk time** | The longest single stretch of silence. |

These metrics are valuable for coaching: excessive agent talk time may indicate the agent is not listening, while excessive non-talk time may indicate holds without proper communication.

---

## Response Time Metrics

| Metric | Description |
|---|---|
| **Agent greeting time** | Time from conversation start to the agent's first utterance. |
| **Agent response time (avg)** | Average time the agent takes to respond after the customer finishes speaking. |
| **Customer response time (avg)** | Average time the customer takes to respond after the agent finishes speaking. |

These are particularly useful for chat where response delays are more visible to the customer.

---

## Key Highlights

Contact Lens automatically identifies key highlights from the conversation:

- **Issue** — The primary reason for the contact as detected from the conversation.
- **Outcome** — Whether the issue was resolved and how.
- **Action item** — Any follow-up actions mentioned during the conversation.

These are generated using generative AI and appear on the Contact details page.

---

## Email Analytics

For the email channel, Contact Lens provides:

| Capability | Description |
|---|---|
| **Categorization** | Auto-categorize emails using the same rules engine as voice/chat. |
| **PII redaction** | Detect and redact PII from email body and subject line. |
| **Summaries** | AI-generated summaries of email threads. |
| **Sentiment** | Sentiment analysis on email content. |

There is no real-time vs. post-contact distinction for email since email is an asynchronous channel. Analytics are produced after each email message is processed.

---

## APIs and Data Access

| Method | Description |
|---|---|
| **ListRealtimeContactAnalysisSegmentsV2** | Stream real-time transcript and analytics segments for an in-progress contact. |
| **S3 analytics files** | Post-contact analytics written as JSON to the configured S3 bucket. |
| **Contact Lens rules** | Managed via the Amazon Connect console or Rules APIs. |
| **Data lake** | Contact Lens data available in the Connect analytics data lake for Athena queries. |
| **Kinesis** | Real-time analytics segments can be streamed to Kinesis for custom processing. |

---

## Pricing Considerations

- Contact Lens is priced per minute of voice analyzed and per message for chat.
- Real-time and post-contact are priced separately.
- PII redaction, theme detection, and generative AI features may incur additional charges.
- Email analytics pricing is per email message analyzed.
