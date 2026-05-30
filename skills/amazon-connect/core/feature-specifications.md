# Connect Customer Feature Specifications (Hard Limits)

This page summarizes **feature specifications** documented in the Connect Customer Administrator Guide.

Important:
- These are **hard limits** (feature specifications). You **can’t increase** them.
- Some other limits in Connect are **quotas** (service quotas) that may be adjustable; see `core/quotas-planning.md`.

## Global and cross-feature limits

| Item | Specification |
|------|---------------|
| Agent activity retention | 24 months from when the event occurred |
| Approved origins per instance | 100 |
| Contact record retention (all channels/subtypes) | 24 months from when the contact was initiated |
| Maximum size of data returned from an AWS Lambda function | Less than 32 KB of UTF-8 data |
| Limit on creating/deleting instances | 100 instances can be created or deleted in 30 days (account-level) |
| Searchable custom contact attributes | 50 |
| Replica instances (`ReplicateInstance`) | 5 per account |
| Traffic distribution groups | 8 per replicated instance |
| Quick connects assignable to a queue | 700 |
| Maximum size of a real-time metrics report | 200 KB |

## Attachments (general)

These limits apply to attachments used with **email, cases, chats, and tasks** (unless otherwise specified for a channel).

| Item | Specification |
|------|---------------|
| Supported attachment file extensions (default allowlist) | `.csv`, `.doc`, `.docx`, `.heic`, `.jfif`, `.jpeg`, `.jpg`, `.mov`, `.mp4`, `.pdf`, `.png`, `.ppt`, `.pptx`, `.rtf`, `.txt`, `.wav`, `.xls`, `.xlsx` |
| Custom attachment extensions | Admins can add custom extensions via the admin website or the Connect API |
| Attachment file size (default / configurable maximum) | 20 MB (configurable up to 100 MB) |
| Attachment scanner timeout | 60 seconds |

Notes:
- WhatsApp has a separate supported-file-types and max-size table; see “WhatsApp business messaging” below.

## Voice (conference calling and monitoring)

| Item | Specification |
|------|---------------|
| Conference participants (voice) | 6 (customer + agent + additional participants) |
| When multi-party calls + enhanced monitoring is enabled | Voice supports 6 participants; up to 2 supervisors can monitor (one can barge in) |
| When multi-party calls + enhanced monitoring is NOT enabled | Voice supports 3 participants on the call; up to 5 supervisors can monitor (listen-only) |

## Chat feature specifications

| Item | Specification |
|------|---------------|
| Attachments per chat conversation | 35 |
| Active chats per agent | 10 |
| Participants in a conference chat | 6 (customer, agent, plus others who can be agents) |
| Custom participants (for example, a custom bot) per contact | 1 |
| Open websocket connections per chat participant | 5 |
| Lex bot integration timeout | 10 seconds (maximum time for Lex to respond to a customer prompt) |
| Total chat duration | Up to 7 days (includes wait time) |
| Default chat duration | 25 hours |
| Configurable chat duration range | 1 hour minimum (60 minutes) to 7 days maximum (10,080 minutes) via `StartChatContact` `ChatDurationInMinutes` |
| Persistent chat past transcript file size | 5 MB |
| Persistent chat past contacts traversed | 100 |
| Monitoring (regardless of capability enablement) | Up to 5 people can monitor the same agent chat at the same time |
| Barge-in supervisors (when multi-party chat + enhanced monitoring enabled) | 1 (only one supervisor can be barged-in for a given chat) |

## Chat message size limits (by channel)

Maximum message size varies by messaging channel and direction.

| Channel | Direction | Initiator → Receiver | Limit |
|---------|-----------|----------------------|-------|
| SMS | Inbound | End customer → Agent or Lex (Connect) | 1,024 characters |
| SMS | Outbound | Agent or Lex (Connect) → End customer | 1,024 characters |
| SMS | Inbound | End customer → Lex (Connect) | 1,024 characters |
| WhatsApp | Inbound | End customer → Agent | 4,096 characters |
| WhatsApp | Outbound | Agent or Lex (Connect) → End customer | 4,096 characters |
| WhatsApp | Inbound | End customer → Lex (Connect) | 1,024 characters |
| Apple Messages for Business | Inbound | End customer → Agent | 4,096 characters |
| Apple Messages for Business | Outbound | Agent or Lex (Connect) → End customer | 4,096 characters |
| Apple Messages for Business | Inbound | End customer → Lex (Connect) | 1,024 characters |
| Chat (web chat) | Inbound | End customer → Agent | 16,384 bytes |
| Chat (web chat) | Outbound | Agent or Lex (Connect) → End customer | 16,384 bytes |

Note:
- “Agent” includes human agents and AI agents implemented as custom participants. Amazon Lex bots can have separate message-size limits.

## WhatsApp business messaging feature specifications

| Media type | Supported file types | Maximum file size |
|------------|----------------------|-------------------|
| Image | `.jpeg`, `.jpg`, `.jfif`, `.png` | 5 MB |
| Video | `.mp4`, `.3gp` | 16 MB |
| Document | `.txt`, `.pdf`, `.ppt`, `.pptx`, `.doc`, `.docx`, `.xls`, `.xlsx` | 20 MB |
| Audio | `.aac`, `.m4a`, `.mp3`, `.amr`, `.ogg` | 16 MB |
| Sticker | Not supported | Not supported |

## Email feature specifications

| Item | Specification |
|------|---------------|
| Maximum email body size | 5 MB |
| Email body formats | HTML (`text/html`) is the default; a plain text (`text/plain`) version is also stored |
| Maximum email body + attachments size | 25 MB |
| Attachments per email message | 10 |
| Inline images per email message | No fixed count limit, as long as total inline images size does not exceed 5 MB |
| Inline images supported formats | `image/jpg`, `image/jpeg`, `image/png`, `image/gif`, `image/svg`, `image/webp`, `image/bmp`, `image/heif`, `image/heic` |
| Inline image storage | Inline images are Base64 encoded when email messages are stored |
| Active email contact expiry | 14 days (default); adjustable up to 90 days using flow logic or the Expiry API via the `connect:ContactExpiry` segment attribute |
| Email addresses per email message | 50 total across To and CC |
| Inbound recipients | Any combination of 50 addresses across To and CC |
| Outbound recipients | 1 address in To; up to 49 addresses in CC |
| From addresses per email message | 1 |
| BCC | Not supported |
| Maximum email subject length | 998 characters |
| Maximum email address length | 255 |
| Maximum display name length | 256 |
| Email message/attachment retention | Defined by your S3 lifecycle configuration (contact record retention still applies to email contact data) |

## Task feature specifications

| Item | Specification |
|------|---------------|
| Task templates per instance | 50 |
| Task template customized fields per instance | 50 |
| Maximum duration of a task | Default 7 days; extensible up to 30 days |
| Maximum transfers per task | 11 |
| Maximum linked tasks on an existing contact | 11 |

## Forecasting, capacity planning, and scheduling feature specifications

| Item | Specification |
|------|---------------|
| Agents per schedule generation run | 5,000 |
| Agents per staffing group | 350 |
| Capacity plans per instance | 500 |
| Capacity scenarios per instance | 500 |
| Capacity plan user data uploads per instance | 500 |
| Capacity plan override uploads per instance | 5,000 |
| Concurrent uploads per instance | 20 |
| File size per upload of agent time off data | 1 GB |
| File size per upload of time off group allowance data | 1 GB (CSV can cover up to 13 months) |
| File size per upload of capacity plan user data | 1 GB |
| File size per upload of capacity plan overrides | 250 MB |
| File size per upload of forecast overrides | 250 MB |
| File size per upload of historical actuals | 1 GB |
| Historical actuals (15 or 30 minute intervals) aggregated file size limit | 2 GB |
| Historical actuals (daily interval) aggregated file size limit | 2 GB |
| Forecast groups per instance | 500 |
| Forecast override uploads per instance | 500 |
| Historical actuals (15 or 30 minute intervals) file count | 300 |
| Historical actuals (daily interval) file count | 300 |
| Queues per forecast group | 200 |
| Schedules per instance | 1,000 |
| Shift activities per instance | 500 |
| Shift activities per shift profile | 10 |
| Shift profiles per instance | 2,500 |
| Shift rotation steps per pattern | 52 |
| Shift rotation weeks per pattern | 52 |
| Shift rotations associated with a single shift profile | 1,300 |
| Shift rotations per instance | 1,300 |
| Staffing groups per forecast group | 300 |
| Staffing groups per instance | 1,300 |
| Staffing groups per supervisor/manager | 250 |
| Supervisors/managers per staffing group | 100 |

## Integration association resource feature specifications

This table describes how many of each integration association resource type can be ingested.

| Resource type | Specification |
|---------------|---------------|
| Attachment scanner | 1 |
| Amazon Pinpoint app | 1 |
| Event (used for task triggers) | 10 |
| Connect AI agents assistant | 1 |
| Connect AI agents knowledge base | 10 |
| Cases domain | 1 |

## Contact Lens feature specifications

| Item | Specification |
|------|---------------|
| Custom vocabularies | 20 |
| Contact Lens rules (post-call) | 500 |
| Contact Lens rules (post-chat) | 500 |
| Contact Lens rules (real-time) | 500 |

## Evaluation forms feature specifications

| Item | Specification |
|------|---------------|
| Maximum evaluations per agent per month | 3,000 |
| Evaluation forms per instance | 400 (historical versions don’t count; form names count) |
| Versions per form | 50 |
| Sections per form | 100 |
| Questions per form | 100 |
| Maximum nesting level of sections | 2 (sections can have sub-sections; sub-sections can’t have sub-sub-sections) |
| Definition title length | 1–128 characters |
| Section title length | 1–128 characters |
| Question title length | 1–350 characters |
| Section instructions length | Up to 1,024 characters |
| Single-select answer options | 2–256 |
| Single-select answer option text length | 1–128 characters |

## Rules feature specifications

| Item | Specification |
|------|---------------|
| Conditions in a rule | 20 |
| Rules with Natural Language condition (OnPostCallAnalysisAvailable) | 100 |
| Rules with Natural Language condition (OnPostChatAnalysisAvailable) | 100 |
| Rules with Natural Language condition (OnEmailAnalysisAvailable) | 15 |
| Rules for OnPostCallAnalysisAvailable | 500 |
| Rules for OnPostChatAnalysisAvailable | 500 |
| Rules for OnRealTimeCallAnalysisAvailable | 500 |
| Rules for OnRealTimeChatAnalysisAvailable | 500 |
| Rules for OnZendeskTicketCreate | 500 |
| Rules for OnZendeskTicketStatusUpdate | 500 |
| Rules for OnSalesforceCaseCreate | 500 |
| Rules for OnContactEvaluationSubmit | 500 |
| Rules for OnCaseUpdate | 500 |
| Rules for OnCaseCreate | 500 |
| Rules for OnMetricDataUpdate | 100 |

