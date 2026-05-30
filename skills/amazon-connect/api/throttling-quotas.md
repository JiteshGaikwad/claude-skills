# Amazon Connect — API Throttling Quotas (Admin Guide Consolidation)

This page consolidates the Admin Guide throttling quota tables into one place so you can design client-side rate limiting correctly.

General rules:

- Throttling is **by account and Region**, not by user and not by instance.
- Multiple users/instances in the same account/Region share the same throttle buckets.

## Connect Service API throttling (core Connect Customer API)

Baseline rule: most operations are **2 rps** with **burst 5**, with exceptions.

Notable exceptions (RateLimit / BurstLimit):

- Evaluations actions: `1 / (service-defined burst)`
- `GetMetricData`: `5 / 8`
- `GetMetricDataV2`: `10 / 10`
- `GetCurrentMetricData`: `5 / 8`
- `SearchContacts`: `0.5 / 1`
- `StartContactStreaming`: `5 / 8`
- `StopContactStreaming`: `5 / 8`
- `StartChatContact`: `5 / 8`
- `CreatePersistentContactAssociation`: `5 / 8`
- `UpdateParticipantRoleConfig`: `5 / 8`
- `CreateParticipant`: `5 / 8`
- `GetContactAttributes`: `10 / 15`
- `UpdateContactAttributes`: `10 / 15`
- `DescribeContact`: `10 / 15`
- `StopContact`: `10 / 15`
- `UpdateContact`: `10 / 15`
- `ListContactReferences`: `10 / 15`
- `BatchPutContact`: `10 / 15`
- `TagContact`: `20 / 25`
- `UntagContact`: `20 / 25`
- `UpdateContactRoutingData`: `20 / 20`
- `SendChatIntegrationEvent`: `17 / 26`

Integration associations:

- `CreateIntegrationAssociation` / `DeleteIntegrationAssociation`: `2 rps` (special-case lower rate for SES identity integration types)
- `ListIntegrationAssociations`: `25 / 50`

Important operational caveat:

- Some metrics APIs may show incorrect values in the Service Quotas console; prefer the documented defaults here unless AWS Support confirms otherwise.

## Cases API throttling

Cases quotas vary by action group and are generally adjustable. Treat these as defaults:

- `GetCase`: `4 / 10`
- `UpdateCase`, `ListCasesForContact`: `2 / 2`
- Bulk gets (`BatchGetField`, `BatchGetCaseRule`): higher rates (example: `8 / 25`)

When building automation, partition work by action group and apply separate limiters.

## Contact Lens API throttling

- `ListRealtimeContactAnalysisSegments`: `1 / 2`
- `ListRealtimeContactAnalysisSegmentsV2`: `2 / 5`

## Customer Profiles API throttling (selected defaults)

The Admin Guide lists per-action TPS defaults, with a split between:

- low TPS for domain/object-type mutations (often `1`), and
- very high TPS for profile CRUD/search paths (often `100`).

If you do both provisioning and runtime lookups, put them in separate worker pools with distinct throttles.

## Outbound Campaigns API throttling (high level)

Campaign lifecycle actions (create/delete/start/stop/pause/resume and core updates) are low-rate by default; batching/lookup paths are higher.

See `api/campaigns-api.md` for action groupings, and enforce an account+Region limiter.

## Connect AI agents / templates throttling (high level)

Message template and related actions have explicit TPS defaults in the Admin Guide.

Practical rule: treat templates/metadata actions as a low-TPS admin path; do not run them inline with high-volume customer traffic.

