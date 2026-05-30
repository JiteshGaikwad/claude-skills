# Amazon Connect Customer APIs — Best Practices (Developer Guide Narrative)

This page consolidates the “Programming with the Connect Customer APIs” narrative guidance from the Developer Guide into a practical checklist for building reliable integrations.

## Error types (how to think about failures)

Treat API failures in three broad buckets:

- **Client errors (4xx)**: invalid parameters, missing permissions, resource not found, invalid state.
- **Service errors (5xx)**: transient service-side issues.
- **Throttling / quotas**: requests rejected due to rate limits or account/resource quotas.

Practical rule: always check each API’s documented `Errors` list for the exact exception types and retryability.

## Throttling basics (2 TPS reality)

Many Connect Customer APIs enforce low default throttles (commonly around **2 TPS** on some action families). Designing for this up front avoids brittle integrations.

What to do:

- **Assume low TPS by default** unless you have verified per-action quotas.
- Implement **client-side rate limiting** for the specific action families you use.
- Use **bounded concurrency** for API calls (don’t “fan out” list/describe calls unbounded).
- Prefer **Search APIs** when they reduce the number of calls required.

## Retry strategy (recommended baseline)

Use an exponential backoff strategy with jitter for retryable failures:

- Retry on throttling responses (`ThrottlingException`, `TooManyRequestsException`, HTTP 429) and transient 5xx errors.
- Do not retry 4xx validation errors (fix request).

Keep retries bounded:

- Cap max delay.
- Cap total attempts.
- Add per-request timeouts.

Implementation patterns: see `api/sdk-patterns.md`.

## Read APIs (Describe/Get/List/Search) — client configuration

For read-heavy integrations:

- Use a single shared SDK client per process.
- Add an application-level rate limiter per action family you call frequently.
- Prefer pagination with **maximum page size** (when safe) to reduce total requests.

## Making 2 TPS work for List APIs

When you must use `List*` APIs:

- Use `MaxResults` at the maximum supported by the API.
- Paginate sequentially and throttle your loop so you don’t exceed TPS.
- Treat pagination tokens as fragile and handle token-invalid situations gracefully (restart from a known checkpoint).

When possible, use `Search*` APIs instead of repeatedly listing + describing.

## Making 2 TPS work for Create/Update APIs

Create/Update patterns are more failure-sensitive because they change state:

- Use **idempotency tokens** when the API supports them.
- Serialize operations per resource (avoid concurrent updates to the same logical object).
- For bulk onboarding, run a controlled job with:
  - fixed concurrency,
  - rate limiting,
  - checkpointing,
  - and retries only for retryable failures.

## Resource quotas: delete stale resources to unblock

If you hit a quota (resource limits, not throttling):

- Identify unused or stale resources (old flows, prompt versions, abandoned workspaces/pages/media, etc.).
- Delete or archive them to free capacity.
- Prefer automation for cleanup in non-production accounts.

## Requesting throttling quota increases

Before requesting an increase:

- Confirm the limit you’re hitting is throttling (429 / throttling exceptions) vs a hard resource quota.
- Capture:
  - the action name(s),
  - current request rate,
  - desired request rate,
  - Region(s),
  - and the business justification.

Then use AWS Service Quotas (or AWS Support, depending on the quota type) to request an increase.

## Supported SDKs

AWS offers SDKs for many languages. This skill repository is intentionally scoped to **AWS SDK v3 for JavaScript/TypeScript** examples.

## Eventual consistency

Some Connect resources may not be immediately readable after creation/update (or after association changes).

Practical guidance:

- Expect delays from **seconds up to minutes** in edge cases.
- If a subsequent read fails with a retryable “not ready”/invalid-state flavor of error, retry with backoff.
- Design workflows so you can re-run safely (idempotent operations and checkpoints).

## Integrations: CloudFormation, CloudTrail, EventBridge

You can integrate Connect with:

- **CloudFormation** (when supported) to manage resources as code.
- **CloudTrail** to audit API calls and security-relevant actions.
- **EventBridge** for event-driven workflows (see `streaming/eventbridge-events.md`).

