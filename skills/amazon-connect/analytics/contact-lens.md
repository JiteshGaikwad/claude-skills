# Contact Lens — Conversational Analytics

Contact Lens provides conversational analytics for voice, chat, and email channels in Amazon Connect. It operates in two modes for voice and chat (real-time and post-contact) and in post-contact mode for email (since email is inherently asynchronous).

---

## Setup and Enabling

Contact Lens is enabled per contact flow using the **Set recording and analytics behavior** block.

### Configuration Options

| Setting | Description |
|---|---|
| **Enable analytics** | Turn on Contact Lens for the contact flow. |
| **Real-time analytics** | Must be explicitly enabled; post-contact alone does not provide streaming data. |
| **Post-contact analytics** | Runs after the interaction ends; produces a comprehensive analytics artifact in S3. |
| **Redaction** | Optional; enable within the same block after enabling analytics. |
| **Language** | Set the language for transcription and analytics. |

### Channels

| Channel | Real-Time | Post-Contact | Notes |
|---|---|---|---|
| **Voice** | Yes | Yes | Only voice has a **Post-call vs Real-time** choice in the block. Choosing **Real-time** yields *both* real-time and post-call. Requires Call recording = **Agent and Customer**. |
| **Chat** | Yes | Yes | Enabling chat analytics gives **both** real-time and post-chat automatically (no separate toggle). Each processed message is billed even if not all features apply. |
| **Email** | N/A | Yes | Asynchronous — no real-time/post distinction. Enabled via the **Set recording, analytics and processing behavior** block (Channel = Email), placed before routing; optional **Contact summary** under Contact Lens Generative AI capabilities. |

> **Enable steps:** turn on Contact Lens for the instance (console → Analytics tools → Enable Contact Lens), then add the block to your flow(s) — repeat in transfer flows. Instances created before October 2018 need extra SLR config for real-time. The **Set recording and analytics behavior** block is superseded by **Set recording, analytics and processing behavior** for new flows (and the old block takes the Error branch for Email).

---

## Real-Time Analytics (Voice & Chat)

Real-time Contact Lens analyzes conversations as they happen, enabling supervisors and automated systems to intervene during live interactions.

### Capabilities

- **Live issue detection** — Identify customer frustration, compliance violations, or escalation triggers during the call/chat.
- **Supervisor alerts** — Rules fire in real-time and can trigger alerts on the supervisor dashboard, send EventBridge events, or invoke Lambda functions.
- **Real-time categories** — Custom category rules evaluated continuously against the in-progress transcript.
- **Live transcript** — Streaming transcript visible on the Contact details page for in-progress contacts.
- **Sentiment trend** — Customer sentiment graph updates as the conversation progresses.

### APIs

The real-time segment API is **split by channel** — there is no single API for both, and **no `Get…V2`** operation exists:

| Method | Channel | Description |
|---|---|---|
| **`ListRealtimeContactAnalysisSegments`** (service `connect-contact-lens`) | **VOICE only** | Lists analysis segments for a real-time voice session. Voice data is **retained 24 hours** — call within that window. Returns `Segments[]` of `Transcript`, `Categories`, or `PostContactSummary`. |
| **`ListRealtimeContactAnalysisSegmentsV2`** (service `connect`) | **CHAT only** | Lists segments for a real-time chat session. **Raises `InvalidRequestException` if used for VOICE.** Body requires `OutputType` (`Raw`\|`Redacted`) and `SegmentTypes` (`Transcript`\|`Categories`\|`Issues`\|`Event`\|`Attachments`\|`PostContactSummary`, max 6); returns `Status` (`IN_PROGRESS`\|`FAILED`\|`COMPLETED`). Requesting `Raw` when the flow set `RedactedOnly` raises `OutputTypeNotFoundException`. |

---

## Post-Contact Analytics (Voice, Chat & Email)

Post-contact analysis runs after the interaction ends and produces a comprehensive analytics artifact stored in S3.

### Output Location

Analytics files are written to the S3 bucket configured in the instance storage settings under `Analysis - Voice/Chat/Email`. The file format is JSON and includes all analytics dimensions described below.

### S3 Path Structure

Analysis JSON is written under the configured bucket, **date-partitioned** `YYYY/MM/DD`, with the filename `{contactId}_analysis_{timestamp}.json`:

- Voice: `connect-instance-bucket/Analysis/Voice/2020/02/04/{contactId}_analysis_2020-02-04T21:14:16Z.json`
- Chat: `connect-instance-bucket/Analysis/Chat/2020/02/04/{contactId}_analysis_...json`
- Email: `connect-instance-bucket/Analysis/Email/2026/03/10/{contactId}_analysis_...json`

**Redacted** output lives under a `Redacted/` sub-prefix: `.../Analysis/Voice/Redacted/2020/02/04/{contactId}_analysis_redacted_...json`, and redacted audio is `..._call_recording_redacted_...wav`. To delete a recording you must delete **both** the redacted and unredacted files.

---

## Sentiment Analysis

Contact Lens assigns sentiment at multiple granularities:

| Level | Description |
|---|---|
| **Per-turn sentiment** | Each utterance/message is classified as `POSITIVE`, `NEGATIVE`, or `NEUTRAL`. |
| **Overall sentiment** | A numeric score from **-5** (most negative) to **+5** (most positive) computed per participant (agent and customer separately). |
| **Sentiment by period** | A score is computed for each **period/portion** of the conversation per participant (the docs say "each period of the call", not a fixed number of quarters). |
| **Sentiment trend vs. distribution** | Two distinct graphs: **trend** shows how sentiment changes as the contact progresses; **distribution** counts the turns/messages that were Positive, Neutral, and Negative across the whole contact. |
| **Sentiment shift** | A search/filter construct identifying where sentiment changed (e.g. begins ≤ -1 and ends ≥ +1), for customer or agent. |

### How Scores Are Determined

Contact Lens considers two factors for each participant turn to assign a score that ranges from -5 to +5 for each period of the call:

1. **Frequency** — The number of times the sentiment is positive, negative, or neutral within the period.
2. **Sentiment streaks** — Consecutive turns with the same sentiment weight the score more heavily.

The overall sentiment score is the average of the scores assigned during each portion of the call.

### Investigation Patterns

- **Positive-to-negative shift** — Customer started happy but left unhappy. Prioritize for quality assurance sampling.
- **Negative-to-positive shift** — Agent successfully de-escalated. Analyze to replicate successful techniques.
- **Sentiment trendline** — Visual chart showing variation in customer sentiment as the contact progresses.

---

## Transcription

Contact Lens produces high-accuracy transcripts for voice interactions using Amazon Transcribe under the hood.

### Features

- **Speaker identification** — Each segment is labeled `AGENT` or `CUSTOMER`. Contact Lens supports calls with **up to 2 participants**; with more than two parties, transcription/analytics quality degrades and AWS recommends disabling analytics for multi-party/third-party calls.
- **Timestamps** — Every segment includes `BeginOffsetMillis` and `EndOffsetMillis` relative to the start of the recording.
- **Confidence scores** — Word-level confidence scores for transcript accuracy.
- **Loudness scores** — Per-second loudness scores for each turn (array of numeric values, one per second of the turn).
- **Talk speed** — Average words per minute computed per participant (`AverageWordsPerMinute`).
- **Custom vocabularies** — Support for domain-specific terms to improve transcription accuracy.
- **Redacted transcripts** — A separate redacted version with PII replaced by placeholders (see PII Redaction below).

### Transcript JSON Structure (per turn)

```json
{
  "BeginOffsetMillis": 7370,
  "EndOffsetMillis": 11190,
  "Content": "I need to cancel my plan subscription.",
  "Id": "turn-unique-id",
  "ParticipantId": "CUSTOMER",
  "Sentiment": "NEGATIVE",
  "LoudnessScore": [77.18, 79.59, 85.23, 81.08, 73.99],
  "IssuesDetected": [...],
  "Redaction": {
    "RedactedTimestamps": [
      { "BeginOffsetMillis": 3290, "EndOffsetMillis": 3620 }
    ]
  }
}
```

---

## Categories

Categories allow you to automatically classify contacts based on rules you define.

### Rule Types

| Rule Type | Description |
|---|---|
| **Words or phrases — Exact Match** | Exact word/phrase (singular or plural), not case-sensitive. Enter manually or **Import from word collection** (user collections: full CRUD; system collections: predefined). |
| **Words or phrases — Pattern Match** | `*` wildcard for related words; **List of values** (interchangeable); **Number** `[0-9]`; **Proximity** (word-distance, e.g. "credit" not within 1 word of "card"). |
| **Words or phrases — Semantic Match** | Matches synonyms/intent; **up to 4 intents/keywords per card**; **post-call/chat only**. |
| **Natural Language — Semantic Match** (generative AI) | Evaluates a natural-language true/false statement against the transcript. Requires **Rules - Generative AI** permission; **post-call/chat only**; uses transcript only. |
| **Conditions** | Plus a large condition set: Sentiment (Time-period vs Entire-contact), Interruptions, Non-talk time, Talk time, Response time, Hold time, Loudness, Queues, Routing profile, Agent/hierarchy, AI agent (+ Escalation), Initiation method, Disconnect reason, ACW, Contact attributes (**max 5/rule**), etc. |

**Note:** rules apply to **new/in-progress** contacts — they cannot be run against past/stored conversations. The "When" trigger is one of: post-call, real-time, post-chat, real-time chat, email analysis. **Word-logic:** for category rules, words **within a line are AND'd**, lines within a card are OR'd, and cards are AND'd together.

### Evaluation Modes

- **Post-contact** — Rules evaluated after the full transcript is available. Most accurate. Supports all match types including semantic match.
- **Real-time** — Rules evaluated incrementally during the conversation. Enables live alerting. Only supports exact match and pattern match.

### Logic Model

- Within a single card of words/phrases: each line is evaluated with **OR** logic.
- Between multiple cards: cards are connected with **AND** logic.
- Example: (Card 1 line 1 OR Card 1 line 2) AND (Card 2 line 1 OR Card 2 line 2).

### Additional Conditions

Rules can be further scoped with:
- **Queue filter** — Apply rule only to specific queues.
- **Contact attributes** — Apply when contact attributes have certain values.
- **Sentiment score threshold** — Apply when sentiment scores meet certain criteria.

### Use Cases

- Compliance monitoring (agent read disclosure script)
- Escalation detection (customer mentions "supervisor" or "cancel")
- Quality assurance categorization (greeting compliance, hold procedure)
- Script adherence (verify agent spoke required phrases)

---

## Contact Lens Rules — Actions

When a rule matches, Contact Lens can perform the following actions:

| Action | Description |
|---|---|
| **Assign contact category** | The matched category name is always assigned (and surfaces on the Contact details page / Current agent performance widget for real-time). |
| **Generate EventBridge event** | Publish to EventBridge. Subscribe with `source = aws.connect` and a `detail-type` of: Contact Lens **Post Call / Realtime / Realtime Chat / Post Chat / Evaluation / Metrics** Rules Matched. Payload `detail` has `ruleName`, `actionName`, `instanceArn`, `contactArn`, `agentArn`, `queueArn`. |
| **Create task** | Create a follow-up Connect task. **Not available for real-time chat.** Task gets its own CTR linked to the contact as Previous contact ID. |
| **Send email notification** | Sent from `no-reply@amazonconnect.com` (not customizable); default limit **500/day** (exceeding blocks the instance 24h); SAML users need a secondary email. |
| **Create Case** | Create a Connect Cases case. |
| **Submit an automated evaluation** | Auto-submit an evaluation form. |

There is **no discrete "Supervisor alert" action** — real-time supervisor alerting happens because the matched category surfaces on the **Current agent performance** widget (calls) / **Contact details** page (chat).

### Rule Management

- Rules are created via **Analytics and optimization > Rules > Create a rule > Conversational analytics** in the Connect console.
- Rules can also be managed programmatically via the Connect Rules APIs.
- Requires **CallCenterManager** security profile or explicit **Rules** permissions (generative-AI conditions also need **Rules - Generative AI**).

### Feature limits (hard caps, not increasable)

| Item | Limit |
|---|---|
| Contact Lens rules per event source (post-call, post-chat, real-time call, real-time chat, …) | **500** each |
| Conditions in a rule | **20** |
| Rules with a Natural-Language (generative-AI) condition | **100** post-call, **100** post-chat, **15** email |
| Words/phrases entries — Exact match / Pattern match | **100** each |
| Words/phrases — Semantic match (per card) | **4** |
| Natural-Language semantic-match entries (per condition) | **1** |
| Queue / Agent condition entries | **100** each |
| Custom attribute conditions per rule | **5** |
| Sentiment (time-period / entire-contact) / Interruptions entries | **5** each |
| Response time threshold (post-chat only) | max **4 hours** |
| Non-talk time threshold (post-call only) | max **5 hours** |
| Custom vocabularies | **20** |

Channel notes: Semantic match, Sentiment-entire-contact, and Interruptions are **not supported for real-time**; Response time is **post-chat only**; Non-talk time is **post-call only**.

---

## PII Redaction

Contact Lens can detect and redact personally identifiable information from transcripts, audio recordings, and email content.

### Supported PII Entity Types

Redaction uses NLU to detect sensitive data such as **name, address, and credit card information**. You can **Redact All PII data** or **select specific entities** to redact — the full selectable list is shown in the block's **Data redaction** section in the console. Entity codes confirmed in the output schema include `CREDIT_DEBIT_NUMBER`, `NAME`, and `USERNAME` (the selected entities appear in `RedactionEntitiesRequested`). *(AWS docs don't publish a full text list of entity codes — choose from the console list rather than assuming codes.)*

### Redaction Targets

| Target | Description |
|---|---|
| **Transcript** | PII replaced with `[PII]` placeholder (when `RedactionMaskMode` = `PII`) or entity type placeholder like `[NAME]` (when `RedactionMaskMode` = `ENTITY_TYPE`). |
| **Audio** | PII segments replaced with silence in the audio recording (.wav file). Silent portions are NOT flagged as non-talk time. |
| **Email** | PII redacted from email body and subject in analytics output. |

### Output Files When Redaction Is Enabled

| File | Description |
|---|---|
| **Redacted file** | Generated by default. Output schema with sensitive data redacted. |
| **Original (raw) analyzed file** | Generated only when "Get redacted and original transcripts with redacted audio" is selected. Contains the complete unredacted conversation. |
| **Redacted audio file (.wav)** | For voice contacts. Sensitive data replaced with silence. |

### Configuration

- Configured in the recording/analytics block: **Redact sensitive data** → **Redact All PII data** (or pick specific entities) → **Data redaction replacement** mask (**Replace with placeholder PII** = `[PII]`, or `ENTITY_TYPE` = `[NAME]` etc.). For voice, redaction applies to the transcript **and** audio (silence) together — there is no transcript-only/audio-only toggle.
- **Dynamic redaction** via a contact attribute (`redaction_option`, case-sensitive: `None` / `RedactedOnly` / `RedactedAndOriginal`) set by a Set-contact-attributes block or Lambda; language can be set dynamically too (`language`).
- For voice, redaction is applied **after the call disconnects**; for email, after the email contact ends.
- ⚠️ The **original (raw) analyzed file is the only place the complete conversation is stored** — deleting it leaves no record of what was redacted. You currently cannot download redacted chat files or voice transcripts.

### Important Limitations

- Redaction uses machine learning and may not catch all instances of PII. Review redacted output.
- Does not meet de-identification requirements under HIPAA. Continue treating redacted data as protected health information.
- Redaction is supported for post-call analytics and chat analytics in supported languages. Not supported for real-time call analytics.

---

## Theme Detection

Theme detection groups contacts with similar issues to surface previously unknown / emerging themes (e.g. "cancel reservation", "delayed order"). It is **post-contact only** — it runs on the **issues detected** in already-analyzed contacts.

- **Generated from Contact search**, not a live dashboard: apply filters → **Save search** → **Generate themes report** (the button is enabled only with **≥ 300 contacts that have issues detected**).
- A report covers the **3,000 most recent** contacts in the saved search; reports are retained **30 days**; the most recent **20** reports per saved search are kept.
- Only contacts created **on or after 2023-01-30** are included.
- Drill into a theme label to view its contacts, listen to recordings, and read transcripts.
- Permissions: **Contact search - Access**, **Contact Lens - theme detection - Create/View**.

---

## Talk Time Metrics

Contact Lens measures detailed talk time breakdowns for voice interactions:

| Metric | Description |
|---|---|
| **Total conversation duration** | End-to-end duration of the voice interaction (`TotalConversationDurationMillis`). |
| **Agent talk time** | Total time the agent was speaking (`TalkTime.DetailsByParticipant.AGENT.TotalTimeMillis`). |
| **Customer talk time** | Total time the customer was speaking (`TalkTime.DetailsByParticipant.CUSTOMER.TotalTimeMillis`). |
| **Total talk time** | Combined agent + customer talk time (`TalkTime.TotalTimeMillis`). |
| **Non-talk time** | Hold time **plus** any silence where both participants aren't talking for **more than 3 seconds** (`NonTalkTime.TotalTimeMillis`; the 3-second threshold isn't customizable). Searchable by duration or percentage (calls only). |
| **Agent talk time %** | Agent talk time as a percentage of total conversation. |
| **Customer talk time %** | Customer talk time as a percentage of total conversation. |
| **Non-talk time %** | Silence as a percentage of total conversation. |
| **Longest non-talk time** | The longest single stretch of silence. |
| **Interruptions (total)** | Total count of interruptions (`Interruptions.TotalCount`). |
| **Interruptions (total time)** | Total time spent in interruptions (`Interruptions.TotalTimeMillis`). |
| **Interruptions by interrupter** | Broken down by AGENT and CUSTOMER, with begin/end offsets and duration for each instance. |
| **Talk speed (agent)** | Average words per minute for the agent (`TalkSpeed.DetailsByParticipant.AGENT.AverageWordsPerMinute`). |
| **Talk speed (customer)** | Average words per minute for the customer. |

These metrics are valuable for coaching: excessive agent talk time may indicate the agent is not listening, while excessive non-talk time may indicate holds without proper communication.

---

## Response Time Metrics

Response-time metrics are **chat-only**:

| Metric | Description |
|---|---|
| **Agent greeting time** | First response time — how fast the agent engaged after joining the chat. |
| **Agent response time** | **Average and Maximum** time the agent takes to respond after the customer. |
| **Customer response time** | **Average and Maximum** time the customer takes to respond after the agent. |

Searchable by average or maximum (with min/max supported values in the Rules feature specs).

---

## Key Highlights

Contact Lens automatically identifies key highlights. Issues, outcomes, and action items are detected via conversational analytics; the **post-contact summary** is the generative-AI capability (no specific foundation model is named in the docs):

| Highlight | JSON Field | Description |
|---|---|---|
| **Issue** | `IssuesDetected` | The primary reason for the contact as detected from the conversation. Includes character offsets and text. |
| **Outcome** | `OutcomesDetected` | Whether the issue was resolved and how. Includes character offsets and text. |
| **Action item** | `ActionItemsDetected` | Any follow-up actions mentioned during the conversation. Includes character offsets and text. |
| **Post-contact summary** | `ContactSummary.PostContactSummary.Content` | AI-generated summary of the entire conversation. |

A contact has **at most one issue, one outcome, and one action item** (some contacts have none — the CCP shows "There are no key highlights for this transcript"). Highlights also surface to agents in the CCP during/after the contact (see `key-highlights.html`). These appear on the Contact details page and in the output JSON.

---

## Email Analytics

For the email channel, Contact Lens provides:

| Capability | Description |
|---|---|
| **Categorization** | Auto-categorize emails using the same rules engine as voice/chat. |
| **PII redaction** | Detect and redact PII from email body and subject line. |
| **Summaries** | AI-generated summaries of email threads. |
| **Sentiment** | Not available for email (email is asynchronous). |

There is no real-time vs. post-contact distinction for email since email is an asynchronous channel. Analytics are produced after each email message is processed.

---

## Custom Vocabulary

Improve transcription accuracy for domain-specific terms (product names, medical terms, jargon).

### Setup

1. Navigate to **Analytics and optimization > Custom vocabularies** in the Connect console.
2. Choose **Add custom vocabulary**, enter a name, and select a language.
3. Download the sample file (English only) or create a tab-separated file with the header: `Phrase`, `IPA`, `SoundsLike`, `DisplayAs`.
4. Upload the file. Multi-word phrases use hyphens (not spaces) in the Phrase column.
5. Set the vocabulary as **default** for it to be applied to analyses.

### File Format

| Column | Required | Description |
|---|---|---|
| **Phrase** | Yes | The word or hyphen-separated phrase to recognize. |
| **IPA** | No | International Phonetic Alphabet pronunciation. |
| **SoundsLike** | No | Phonetic hint using similar-sounding words. |
| **DisplayAs** | No | How the word should appear in the transcript. |

### Key Details

- File must use **LF** line endings (CRLF is rejected); multi-word phrases use **hyphens, not spaces** (spaces → **Failed** state).
- One active (default) vocabulary per language; up to **20** files can be uploaded and activated simultaneously. Underlying Amazon Transcribe limits: **≤ 100 files per AWS account**, **≤ 50 KB per file**, **≤ 256 chars per entry**, same Region as transcription.
- States: **Processing** → **Ready** (valid, not applied) → **Ready (default)** (applied to both real-time and post-call) → **Deleting** → **Failed**. Only **Ready** files can be downloaded/viewed.
- Deletion takes ~**90 minutes**. Transcription is one-time — vocabularies are **not** applied retroactively.
- Speech analytics only (not chat). Permission: **Analytics and Optimization → Contact Lens - custom vocabularies** (default on Admin / CallCenterManager).

### APIs

- `CreateVocabulary` — Create a new custom vocabulary.
- `AssociateDefaultVocabulary` — Set a vocabulary as the default for a language.

---

## Language Support

The following table shows Contact Lens feature support by language. Languages marked with * are not available in Africa (Cape Town) or AWS GovCloud (US-West).

### Full Feature Support (Post-Call + Real-Time + Sentiment + Redaction + Summaries)

| Language | Code | Post-Call | Real-Time | Sentiment | Redaction | Summaries | Pattern Rules |
|---|---|---|---|---|---|---|---|
| English (US) | en-US | Yes | Yes | Yes | Yes | Yes | Yes |
| English (UK) | en-GB | Yes | Yes | Yes | Yes | Yes | Yes |
| English (Australia) | en-AU | Yes | Yes | Yes | Yes | Yes | Yes |
| English (India) | en-IN | Yes | Yes | Yes | Yes | Yes | Yes |
| English (Ireland) | en-IE | Yes | Yes | Yes | Yes | Yes | Yes |
| English (New Zealand) | en-NZ | Yes | Yes | Yes | Yes | Yes | Yes |
| English (Scotland) | en-AB | Yes | Yes | Yes | Yes | Yes | Yes |
| English (South Africa) | en-ZA | Yes | Yes | Yes | Yes | Yes | Yes |
| English (Wales) | en-WL | Yes | Yes | Yes | Yes | Yes | Yes |
| French (Canada) | fr-CA | Yes | Yes | Yes | Yes | Yes | Yes |
| French (France) | fr-FR | Yes | Yes | Yes | Yes | Yes | Yes |
| German (Germany) | de-DE | Yes | Yes | Yes | Yes | Yes | Yes |
| Italian (Italy) | it-IT | Yes | Yes | Yes | Yes | Yes | Yes |
| Portuguese (Brazil) | pt-BR | Yes | Yes | Yes | Yes | Yes | Yes |
| Portuguese (Portugal) | pt-PT | Yes | Yes | Yes | Yes | Yes | Yes |
| Spanish (Spain) | es-ES | Yes | Yes | Yes | Yes | Yes | Yes |
| Spanish (US) | es-US | Yes | Yes | Yes | Yes | Yes | Yes |

### Post-Call + Real-Time + Sentiment (No Redaction)

| Language | Code | Post-Call | Real-Time | Sentiment | Summaries |
|---|---|---|---|---|---|
| Chinese Simplified | zh-CN | Yes | Yes | Yes | Yes |
| German (Switzerland) | de-CH | Yes | Yes | Yes | Yes |
| Hindi (India) | hi-IN | Yes | Yes | Yes | No |
| Japanese (Japan) | ja-JP | Yes | Yes | Yes | Yes |
| Korean (South Korea) | ko-KR | Yes | Yes | Yes | Yes |

### Post-Call + Real-Time (No Redaction)

*(Exception: **ar-AE** (Arabic Gulf) also supports **sentiment** — it's not a no-sentiment language. The others below are no-sentiment, no-redaction.)*

| Language | Code |
|---|---|
| Arabic (Gulf)* — has sentiment | ar-AE |
| Arabic (Modern Standard)* | ar-SA |
| Catalan (Spain)* | ca-ES |
| Croatian (Croatia)* | hr-HR |
| Czech (Czech Republic)* | cs-CZ |
| Danish (Denmark)* | da-DK |
| Dutch (Netherlands)* | nl-NL |
| Farsi (Iran)* | fa-IR |
| Finnish (Finland)* | fi-FI |
| Galician (Spain)* | gl-ES |
| Greek (Greece)* | el-GR |
| Hebrew (Israel)* | he-IL |
| Indonesian (Indonesia)* | id-ID |
| Latvian (Latvia)* | lv-LV |
| Malay (Malaysia)* | ms-MY |
| Norwegian (Norway)* | no-NO |
| Polish (Poland)* | pl-PL |
| Romanian (Romania)* | ro-RO |
| Russian (Russia)* | ru-RU |
| Serbian (Serbia)* | sr-RS |
| Slovak (Slovakia)* | sk-SK |
| Swedish (Sweden)* | sv-SE |
| Tagalog (Philippines)* | tl-PH |
| Thai (Thailand)* | th-TH |
| Ukrainian (Ukraine)* | uk-UA |
| Vietnamese (Vietnam)* | vi-VN |

### Post-Call Only (No Real-Time)

| Language | Code |
|---|---|
| Afrikaans (South Africa)* | af-ZA |
| Bengali (Bangladesh)* | bn-IN |
| Bosnian (Bosnia)* | bs-BA |
| Bulgarian (Bulgaria)* | bg-BG |
| Estonian (Estonia)* | et-ET |
| Hungarian (Hungary)* | hu-HU |
| Kannada (India)* | kn-IN |
| Lithuanian (Lithuania)* | lt-LT |
| Macedonian (Macedonia)* | mk-MK |
| Malayalam (India)* | ml-IN |
| Marathi (India)* | mr-IN |
| Sinhala (Sri Lanka)* | si-LK |
| Slovenian (Slovenia)* | sl-SI |
| Somali (Somalia)* | so-SO |
| Sundanese (Indonesia)* | su-ID |
| Tamil (India)* | ta-IN |
| Telugu (India)* | te-IN |
| Turkish (Turkey)* | tr-TR |
| Zulu (South Africa)* | zu-ZA |

Notes: The authoritative `supported-languages.html` table now also has **Amazon Connect AI Agents** and **Automated performance evaluations** columns (the latter excluded in Africa (Cape Town), Mumbai, Seoul, GovCloud US-West), and additional locales such as **zh-HK** (Cantonese — email/chat + summaries only) and AI-Agents-only locales (es-MX, en-SG, fr-BE, de-AT, nl-BE, ga-IE, …). `de-CH` has summaries+sentiment but **no redaction and no pattern rules**. Redaction `*` languages are unavailable in Africa (Cape Town) and GovCloud (US-West). The list changes — verify against the live table for a specific Region.

---

## External Voice System Integration

Apply Contact Lens analytics to calls handled by a **non-Connect** voice system — **not** by uploading audio files, but via **live audio replication using a Contact Lens Connector and SIPREC**. A read-only copy of the in-progress call audio is forked from your external system into Connect; the external call flow keeps operating normally for agents while Contact Lens produces real-time and post-call analytics on the replica.

- **Mechanism:** your Session Border Controller (SBC) sends a SIPREC replica of the call audio to the Contact Lens Connector's fully-qualified host name, plus call metadata. No phone number is claimed in Connect.
- **Setup:** create an instance + add agents/hierarchies → request service-quota increases ("Contact Lens connectors per account", "Maximum active recording sessions from external voice systems per instance") → create a Contact Lens connector → configure the SBC to send SIPREC audio + metadata (supported `ContactCenterSystemTypes`/`SessionBorderControllerTypes` per Amazon Chime SDK `PutVoiceConnectorExternalSystemsConfiguration`) → grant security-profile perms (Analytics and Optimization – Contact Lens connectors – View/Edit; Channels and Flows – Flows – View) → create + associate a flow with a `Set recording and analytics behavior` block → optionally a Lambda to parse the SIPREC request + metadata.
- **Agent identification is required:** if no agent is identified for a call, the replica call terminates and no recording/analytics are produced.
- Pages: `contact-lens-integration.html`, `create-contact-lens-connector.html`, `configure-external-voice-system.html`, `callmetadata-contactlens-integration.html`, `contactlens-integration-multiregion.html`. (Distinct from `external-voice-transfer.html`, an unrelated outbound-transfer feature.)

---

## Output Artifacts — JSON Schema

The post-contact analytics JSON file includes the following top-level fields:

| Field | Description |
|---|---|
| `Version` | Schema version (e.g., "1.1.0"). |
| `AccountId` | AWS account ID. |
| `Channel` | VOICE, CHAT, or EMAIL. |
| `ContentMetadata.Output` | "Raw" for original file, "Redacted" for redacted file. |
| `JobStatus` | COMPLETED or FAILED. |
| `JobDetails.SkippedAnalysis[]` | Skipped features: `Feature` (e.g. `CATEGORIZATION`), `ReasonCode` (`QUOTA_EXCEEDED` \| `FAILED_SAFETY_GUIDELINES`), `SkippedEntities[]` (`CategoryName`, `RuleId`). |
| `LanguageCode` | Language code used for analysis. |
| `Participants` | Array of participants with `ParticipantId` and `ParticipantRole`. |
| `Categories` | `MatchedCategories[]` and `MatchedDetails.<CategoryName>.PointsOfInterest[].{BeginOffsetMillis, EndOffsetMillis}`. |
| `ConversationCharacteristics` | `TotalConversationDurationMillis`; `Sentiment.{OverallSentiment.{AGENT,CUSTOMER}, SentimentByPeriod.QUARTER.{AGENT,CUSTOMER}[].{BeginOffsetMillis,EndOffsetMillis,Score}}` (overall score can be fractional, e.g. 3.1); `Interruptions.{TotalCount, TotalTimeMillis, InterruptionsByInterrupter.{AGENT,CUSTOMER}[].{BeginOffsetMillis,DurationMillis,EndOffsetMillis}}`; `NonTalkTime.{TotalTimeMillis, Instances[]}`; `TalkSpeed.DetailsByParticipant.{AGENT,CUSTOMER}.AverageWordsPerMinute`; `TalkTime.{TotalTimeMillis, DetailsByParticipant...}`; `ContactSummary.PostContactSummary.Content`. |
| `CustomModels` | Custom vocabulary references (`Type: TRANSCRIPTION_VOCABULARY`, `Name`, `Id`). |
| `Transcript[]` | Per turn: `BeginOffsetMillis`, `Content`, `EndOffsetMillis`, `Id`, `ParticipantId`, `Sentiment`, `LoudnessScore[]` (one value/second), per-turn `IssuesDetected[]` / `OutcomesDetected[]` / `ActionItemsDetected[]` (each `{CharacterOffsets.{BeginOffsetChar,EndOffsetChar}, Text}`), and `Redaction.RedactedTimestamps[]` on PII turns. |

### Redacted File Additions

| Field | Description |
|---|---|
| `ContentMetadata.RedactionTypes` | Array of redaction types (e.g., `["PII"]`). |
| `ContentMetadata.RedactionTypesMetadata.PII` | `RedactionEntitiesRequested` (entity types), `RedactionMaskMode` (`PII` or `ENTITY_TYPE`). |

---

## APIs and Data Access

| Method | Description |
|---|---|
| **`ListRealtimeContactAnalysisSegments`** | Real-time **voice** segments (service `connect-contact-lens`; 24h retention). |
| **`ListRealtimeContactAnalysisSegmentsV2`** | Real-time **chat** segments (service `connect`; voice raises `InvalidRequestException`). |
| **S3 analytics files** | Post-contact analytics written as JSON to the configured S3 bucket. |
| **Contact Lens rules** | Managed via the Amazon Connect console or Rules APIs. |
| **Data lake** | Contact Lens data available in the Connect analytics data lake for Athena queries. |

---

## Pricing Considerations

| Dimension | Pricing Model |
|---|---|
| **Voice (post-call)** | Per minute of voice analyzed. |
| **Voice (real-time)** | Per minute of voice analyzed (priced separately from post-call). |
| **Chat** | Per message analyzed. Each processed message is billed the same way regardless of which features apply to that message. |
| **Email** | Per email message analyzed. |
| **PII redaction** | Additional charge on top of base analytics pricing. |
| **Theme detection** | Additional charge. |
| **Generative AI features** | Summaries, key highlights may incur additional charges. |

Real-time and post-contact are priced separately. See [Amazon Connect Pricing](https://aws.amazon.com/connect/pricing/) for current rates.
