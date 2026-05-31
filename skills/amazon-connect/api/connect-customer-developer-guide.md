# Amazon Connect Customer — Developer Guide (How to Use This Skill)

This page mirrors the “front matter” of the Amazon Connect Customer Developer Guide and explains how this repository is organized so you can find the right deep-dive quickly.

## What this guide is

Amazon Connect Customer (“Amazon Connect”) is a contact center platform with multiple APIs and sub-services (Connect Service, Contact Lens, Customer Profiles, Cases, Participant Service, Outbound Campaigns, AppIntegrations, Q Connect, etc.).

This skill repository is structured to match those areas:

- **Core concepts & administration**: `core/*`
- **Contact flows**: `flows/*`
- **Channels** (voice/chat/email/tasks/web-video): `channels/*`
- **Analytics & reporting**: `analytics/*`
- **Streaming & events**: `streaming/*`
- **Agent experience & Agent Workspace**: `agent-experience/*`
- **Service API references (action catalogs)**: `api/*`

## Who this is for

Use this skill when you are:

- Building or integrating with Amazon Connect Customer APIs (AWS SDK v3 JS/TS).
- Designing contact flows, routing, queue behavior, or channel experiences.
- Implementing Agent Workspace third-party applications/services.
- Troubleshooting throttling, quotas, attachments, and operational behaviors.

## What’s in this guide

The Connect Customer Developer Guide spans:

- **Programming/best practices** (throttling, errors, quotas, eventual consistency, integrations).
- **Attachments** (Participant Service vs Connect attached files).
- **Outbound campaigns** best practices (`PutDialRequestBatch`).
- **Contact disposition tracking** recommendations.
- Reference catalogs for Flow Language / Rules Function Language / Testing Language.

In this repo, those map to:

- Best practices narrative: `api/connect-customer-api-best-practices.md`
- Attachments narrative: `api/attachments.md`
- PutDialRequestBatch best practices: `api/putdialrequestbatch-best-practices.md`
- Contact dispositions: `analytics/contact-dispositions.md`
- Flow language JSON: `flows/flow-language.md`
- Rules Function Language: `api/rules-language.md`
- Testing Language: `api/testing-language.md`
- API action catalogs: `api/connect-service-api.md`, `api/customer-profiles-api.md`, `api/cases-api.md`, `api/participant-api.md`, `api/campaigns-api.md`, `api/q-connect-api.md`, `api/app-integrations-api.md`, `api/contact-lens-api.md`

## How to use this guide

1. Start with the narrative page that matches your problem:
   - Throttling / 2 TPS / quotas: `api/connect-customer-api-best-practices.md`
   - Attachments: `api/attachments.md`
   - Outbound dial batching: `api/putdialrequestbatch-best-practices.md`
   - Disposition reporting: `analytics/contact-dispositions.md`
2. Then jump into the specific service action catalog under `api/*`.
3. For flow automation, use `flows/*` (blocks, designer behavior, flow-language JSON).

## Skill policy notes

- **SDK examples**: this skill uses **AWS SDK v3 for JavaScript/TypeScript** patterns.
