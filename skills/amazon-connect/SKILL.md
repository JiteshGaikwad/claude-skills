---
name: amazon-connect
description: >
  Amazon Connect contact center — flows, routing, Contact Lens, metrics, Customer Profiles,
  Cases, Q Connect, AI agents, Lambda integration, streaming, EventBridge, and SDK/API reference.
  Use when working on Amazon Connect configuration, contact flows, real-time/historical metrics,
  Contact Lens analytics, agent workspace development, Kinesis streaming, or AWS SDK v3
  JavaScript/TypeScript code that interacts with Connect services.
---

# Amazon Connect — Comprehensive Skill Reference

Amazon Connect (now "Amazon Connect Customer") is an omnichannel cloud contact center with 9 sub-services:

| Service | What It Does |
|---------|-------------|
| **Connect Service** | Core contact center — flows, routing, queues, users, metrics, instances |
| **Contact Lens** | Conversational analytics — sentiment, transcription, categories, redaction |
| **Customer Profiles** | Unified customer data — identity resolution, integrations, segments |
| **Cases** | Case management — domains, templates, layouts, rules |
| **Q Connect** | AI-powered agent assist — knowledge bases, AI agents, prompts, guardrails |
| **Participant Service** | Chat/messaging participant management — messages, events, transcripts |
| **Outbound Campaigns** | High-volume outbound — predictive dialer, ML answering machine detection |
| **App Integrations** | Third-party data/event integrations |
| **Connect AI Agents** | Agentic self-service + agent-assist — MCP tools, multi-step reasoning |

## SDK

All code examples and patterns use **AWS SDK v3 for JavaScript/TypeScript** only:
- `@aws-sdk/client-connect`
- `@aws-sdk/client-connect-contact-lens`
- `@aws-sdk/client-customer-profiles`
- `@aws-sdk/client-connectcases`
- `@aws-sdk/client-qconnect`
- `@aws-sdk/client-connectcampaigns` / `@aws-sdk/client-connectcampaignsv2`
- `@aws-sdk/client-connectparticipant`

## Reference File Index

### Core Infrastructure
- [core/instances.md](core/instances.md) — Instance management, attributes, storage configs
- [core/telephony.md](core/telephony.md) — Phone numbers (DID/toll-free, 110+ countries)
- [core/security.md](core/security.md) — IAM policies + managed policies, per-feature KMS/CMK, data protection & retention, TBAC + tagging limits, auth profiles (IP/session), logging & monitoring, resilience, confused-deputy, compliance, best practices
- [core/global-resiliency.md](core/global-resiliency.md) — Multi-region, traffic distribution
- [core/network-requirements.md](core/network-requirements.md) — CCP network setup, firewall, ports, VDI, Direct Connect
- [core/routing-and-queues.md](core/routing-and-queues.md) — Queue types, routing profiles, priority/delay, quick connects, transfers
- [core/user-management.md](core/user-management.md) — Security profiles, hierarchy groups, bulk ops, agent status
- [core/identity-management.md](core/identity-management.md) — SAML 2.0, AD, Connect-managed, approved origins/CORS
- [core/customer-authentication.md](core/customer-authentication.md) — Customer auth (Cognito/JWT), communications widget auth
- [core/service-linked-roles.md](core/service-linked-roles.md) — All Connect SLRs (core, campaigns, sync), KMS edge cases
- [core/regions-and-feature-availability.md](core/regions-and-feature-availability.md) — Region availability, feature-by-region matrix, endpoints
- [core/language-support-matrix.md](core/language-support-matrix.md) — Languages supported per feature (TTS, ASR, Contact Lens, UI)
- [core/accessibility-and-browser-support.md](core/accessibility-and-browser-support.md) — Accessibility (ACR/screen readers), browser support, policy breakage fixes
- [core/how-contacts-are-counted.md](core/how-contacts-are-counted.md) — Concurrency quota counting rules
- [core/quotas-planning.md](core/quotas-planning.md) — Launch sizing, alarms, emergency operations

### Contact Flows
- [flows/overview.md](flows/overview.md) — Flow designer, types, modules, prompts
- [flows/blocks.md](flows/blocks.md) — All 53 flow blocks reference
- [flows/flow-language.md](flows/flow-language.md) — Programmatic JSON (56 action types)
- [flows/lambda-integration.md](flows/lambda-integration.md) — Lambda event/response, timeouts
- [flows/contact-attributes.md](flows/contact-attributes.md) — All attribute types
- [flows/media-streaming.md](flows/media-streaming.md) — Kinesis Video Streams
- [flows/nova-sonic.md](flows/nova-sonic.md) — Nova Sonic Speech-to-Speech
- [flows/proficiency-routing.md](flows/proficiency-routing.md) — Proficiency-based routing, step waterfall, preferred agent, Lambda JSON
- [flows/default-flows.md](flows/default-flows.md) — All 9 default flows with blocks, triggers, channel support
- [flows/sample-flows.md](flows/sample-flows.md) — All 13 sample flows with step-by-step structure
- [flows/flow-designer.md](flows/flow-designer.md) — Mini-map, custom block names, undo/redo, notes, copy/paste, archive/delete/restore, version control
- [flows/flow-best-practices.md](flows/flow-best-practices.md) — Flow best practices, flow logs (CloudWatch), contact initiation methods (13 types)
- [flows/flow-logging.md](flows/flow-logging.md) — CloudWatch Logs setup, log format, troubleshooting
- [flows/encryption.md](flows/encryption.md) — Sensitive input encryption, key management, PCI compliance
- [flows/prompts.md](flows/prompts.md) — Prompts: create/upload, TTS (Polly), Set voice block, dynamic prompts, S3 playback
- [flows/ssml.md](flows/ssml.md) — Supported SSML tags for TTS + strict-use rule (use only listed tags, never assume)
- [flows/queued-callback.md](flows/queued-callback.md) — Queued callback setup: flow build, Transfer to callback queue options, routing, queue limits, chat/task/email

### Channels
- [channels/voice.md](channels/voice.md) — Softphone, recording, audio enhancement
- [channels/chat-sms.md](channels/chat-sms.md) — Chat widget, SMS, WhatsApp, persistent chat
- [channels/email.md](channels/email.md) — SES integration, threading, templates
- [channels/tasks.md](channels/tasks.md) — Task management, templates, automation
- [channels/web-video.md](channels/web-video.md) — WebRTC, in-app calling, video
- [channels/in-app-web-and-video-calling-capabilities.md](channels/in-app-web-and-video-calling-capabilities.md) — Widget vs native integration overview
- [channels/webrtc-chime-sdk-implementation.md](channels/webrtc-chime-sdk-implementation.md) — Chime SDK client patterns, DTMF, multi-user

### AI & Automation
- [ai/connect-ai-agents.md](ai/connect-ai-agents.md) — Agent types, agentic self-service, MCP/Return-to-Control/Constant tools, message parsing, security profiles, Connect assistant block, sessions, locale, flow association
- [ai/ai-prompts.md](ai/ai-prompts.md) — AI prompt types, MESSAGES/TEXT_COMPLETIONS YAML, variables, prompt caching, per-region models, CLI
- [ai/ai-guardrails.md](ai/ai-guardrails.md) — Guardrail policies (content filters, denied topics, contextual grounding, word/PII filters), thresholds, full PII entity list, CLI
- [ai/q-connect.md](ai/q-connect.md) — Amazon Q Connect, knowledge bases, assistants
- [ai/lex-bots.md](ai/lex-bots.md) — Lex integration, conversational IVR
- [ai/outbound-campaigns.md](ai/outbound-campaigns.md) — Predictive dialer, campaigns v1+v2
- [ai/generative-ai.md](ai/generative-ai.md) — Post-contact summaries, gen-AI evaluations
- [ai/prompt-engineering.md](ai/prompt-engineering.md) — AI agent prompt best practices, troubleshooting, monitoring
- [ai/chat-message-streaming.md](ai/chat-message-streaming.md) — AI-powered chat message streaming, MESSAGE_STREAMING attribute, progressive responses
- [ai/bedrock-model-selection-cross-region-inference.md](ai/bedrock-model-selection-cross-region-inference.md) — Model selection + cross-region inference mapping

### Analytics & Metrics
- [analytics/contact-lens.md](analytics/contact-lens.md) — Sentiment, transcription, categories
- [analytics/real-time-metrics.md](analytics/real-time-metrics.md) — RT metric definitions
- [analytics/historical-metrics.md](analytics/historical-metrics.md) — 80+ historical metrics
- [analytics/dashboards-reports.md](analytics/dashboards-reports.md) — Dashboards, custom metrics
- [analytics/data-lake.md](analytics/data-lake.md) — Athena/QuickSight, zero-ETL
- [analytics/contact-records.md](analytics/contact-records.md) — CTR data model, contact states
- [analytics/contact-dispositions.md](analytics/contact-dispositions.md) — Disposition tracking recommendations (outbound)
- [analytics/evaluations.md](analytics/evaluations.md) — Eval forms, automated/gen-AI evaluations, calibration, coaching
- [analytics/screen-recording.md](analytics/screen-recording.md) — Agent screen recording: client app, enablement, specs, playback, EventBridge, permissions
- [analytics/monitoring.md](analytics/monitoring.md) — Live monitor/barge (3-party vs enhanced), recorded playback, CloudWatch, CloudTrail
- [analytics/contact-search.md](analytics/contact-search.md) — Search, in-progress management (transfer/reschedule/end)

### Data Streaming
- [streaming/data-streaming.md](streaming/data-streaming.md) — Kinesis setup, KMS encryption
- [streaming/agent-event-streams.md](streaming/agent-event-streams.md) — Agent events data model
- [streaming/contact-lens-streams.md](streaming/contact-lens-streams.md) — Real-time CL via Kinesis
- [streaming/eventbridge-events.md](streaming/eventbridge-events.md) — Contact events (11 types)

### Agent Experience
- [agent-experience/workspace.md](agent-experience/workspace.md) — Workspace architecture
- [agent-experience/step-by-step-guides.md](agent-experience/step-by-step-guides.md) — No-code UI builder
- [agent-experience/ccp.md](agent-experience/ccp.md) — Contact Control Panel usage
- [agent-experience/wfm.md](agent-experience/wfm.md) — Forecasting, scheduling, adherence
- [agent-experience/developer-guide.md](agent-experience/developer-guide.md) — 3P apps/services, SDK (10 clients, 117 methods)
- [agent-experience/app-loading-model.md](agent-experience/app-loading-model.md) — Per-contact vs cross-contact iframe lifecycle
- [agent-experience/embedding-security-and-csp.md](agent-experience/embedding-security-and-csp.md) — CSP `frame-ancestors`, approved origins
- [agent-experience/permissions-and-appconfig.md](agent-experience/permissions-and-appconfig.md) — Permissions debugging runbook
- [agent-experience/testing-local-and-deployed.md](agent-experience/testing-local-and-deployed.md) — Local + deployed testing checklist
- [agent-experience/streams-vs-3p-apps.md](agent-experience/streams-vs-3p-apps.md) — StreamsJS vs Agent Workspace SDK
- [agent-experience/appmanager-lifecycle.md](agent-experience/appmanager-lifecycle.md) — AppManager lifecycle/visibility patterns
- [agent-experience/troubleshooting.md](agent-experience/troubleshooting.md) — CCP issues, diagnostics

### Data Services
- [data/customer-profiles.md](data/customer-profiles.md) — Identity resolution, integrations
- [data/cases.md](data/cases.md) — Domains, templates, layouts, rules
- [data/data-tables.md](data/data-tables.md) — Data tables in flows

### Voicemail (Voicemail Express V3 — out-of-the-box add-on)
- [voicemail/overview.md](voicemail/overview.md) — VMX3 architecture, end-to-end flow, works by default for all agents/queues
- [voicemail/deployment.md](voicemail/deployment.md) — Prerequisites, CloudFormation install, GovCloud, upgrade, uninstall
- [voicemail/delivery-modes.md](voicemail/delivery-modes.md) — Task, Guided Task, and SES email delivery modes
- [voicemail/advanced-options.md](voicemail/advanced-options.md) — Self-service, in-queue voicemail, longer messages, customer-managed KMS
- [voicemail/code-pipeline.md](voicemail/code-pipeline.md) — Lambda processing pipeline and data flow
- [voicemail/troubleshooting.md](voicemail/troubleshooting.md) — Common issues, media-player fix, changelog

### Admin
- [admin/workspace-admin.md](admin/workspace-admin.md) — Admin workspace themes, home dashboard, notifications

### Testing
- [testing/simulation.md](testing/simulation.md) — Visual test designer, flow simulation

### API Reference
- [api/overview.md](api/overview.md) — Architecture, best practices, common errors
- [api/connect-customer-developer-guide.md](api/connect-customer-developer-guide.md) — Guide intro, how to use this repo
- [api/connect-customer-api-best-practices.md](api/connect-customer-api-best-practices.md) — Throttling/2 TPS, errors, quotas, eventual consistency
- [api/attachments.md](api/attachments.md) — Attachments narrative (Participant vs Attached Files)
- [api/putdialrequestbatch-best-practices.md](api/putdialrequestbatch-best-practices.md) — Outbound dial batching best practices
- [api/throttling-quotas.md](api/throttling-quotas.md) — Admin Guide throttling quota tables (by service)
- [api/connect-service-api.md](api/connect-service-api.md) — 200+ actions, 530 data types
- [api/contact-lens-api.md](api/contact-lens-api.md) — 2 actions, 8 data types
- [api/customer-profiles-api.md](api/customer-profiles-api.md) — 80+ actions, 130+ data types
- [api/cases-api.md](api/cases-api.md) — 30+ actions, 80+ data types
- [api/q-connect-api.md](api/q-connect-api.md) — 70+ actions, 200+ data types
- [api/participant-api.md](api/participant-api.md) — 11 actions, 17 data types
- [api/campaigns-api.md](api/campaigns-api.md) — V1+V2, 85 data types
- [api/app-integrations-api.md](api/app-integrations-api.md) — 23 actions, 20 data types
- [api/rules-language.md](api/rules-language.md) — Rules Function Language DSL
- [api/testing-language.md](api/testing-language.md) — Testing Language DSL
- [api/sdk-patterns.md](api/sdk-patterns.md) — JS SDK v3 patterns

### Release Notes
- [recent-changes.md](recent-changes.md) — Latest features (2026)
- [doc-history.md](doc-history.md) — Document history mapping (AWS PDF doc history pointers)

## Examples

- "How do I invoke a Lambda function from a contact flow?" → Read `flows/lambda-integration.md`
- "What real-time metrics can I track?" → Read `analytics/real-time-metrics.md`
- "How do I set up agentic self-service with MCP tools?" → Read `ai/connect-ai-agents.md`
- "How do I customize an AI prompt / which models or YAML format?" → Read `ai/ai-prompts.md`
- "How do I configure AI guardrails (content filters, PII, denied topics)?" → Read `ai/ai-guardrails.md`
- "Show me how to use GetMetricDataV2 with the JS SDK" → Read `api/sdk-patterns.md`
- "What EventBridge events does Connect emit?" → Read `streaming/eventbridge-events.md`
- "How do I build a 3P app for the agent workspace?" → Read `agent-experience/developer-guide.md`
- "How do I route contacts based on agent skills/language?" → Read `flows/proficiency-routing.md`
- "How do I set up SAML SSO for Connect?" → Read `core/identity-management.md`
- "What are the best practices for AI agent prompts?" → Read `ai/prompt-engineering.md`
- "How do I encrypt credit card input in a flow?" → Read `flows/encryption.md`
- "How do I add voicemail to Amazon Connect?" → Read `voicemail/overview.md` (Voicemail Express V3 — out-of-the-box add-on)
- "How do I create a prompt / add text-to-speech / play audio from S3?" → Read `flows/prompts.md`
- "Write/generate a TTS prompt with SSML" or "which SSML tags does Connect support?" → Read `flows/ssml.md` first, then use only the tags listed there
- "How do I set up a queued callback / offer a callback in queue?" → Read `flows/queued-callback.md`

## Guidelines

- All code examples must use AWS SDK v3 for JavaScript/TypeScript — never Java, Python, Go, or .NET
- **When generating any text-to-speech / prompt message with SSML, FIRST read `flows/ssml.md` and use ONLY the SSML tags listed there. Never assume a tag is supported — Connect supports a specific subset and silently ignores the rest.**
- When answering, read the relevant sub-file first rather than guessing from the routing index
- If a feature seems unfamiliar or recent, check the Living Documentation changelog URLs below
- Reference files are loaded on-demand — read only the files relevant to the question, not the whole skill at once

## Living Documentation

When information in this skill seems outdated, a feature is unfamiliar, or the user asks about something recent, check these canonical doc history pages:

- **Admin Guide changelog**: https://docs.aws.amazon.com/connect/latest/adminguide/doc-history.html
- **Agent Workspace Dev Guide changelog**: https://docs.aws.amazon.com/agentworkspace/latest/devguide/doc-history.html

After finding new information, update the relevant skill reference file so future queries don't need to re-fetch.
