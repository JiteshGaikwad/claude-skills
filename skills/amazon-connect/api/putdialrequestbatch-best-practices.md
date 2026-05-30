# Outbound Campaigns — `PutDialRequestBatch` Best Practices

This page captures the Developer Guide narrative guidance for using `PutDialRequestBatch` reliably (V1 campaigns).

Primary API reference:

- `api/campaigns-api.md`

## Preconditions

- Ensure the Connect instance is onboarded for campaigns.
- Create the campaign and **start** it before submitting dial requests.

## Idempotency and client tokens

- Always supply a stable `clientToken` per dial request.
- If you need to retry submission, retry the same request with the same `clientToken` to avoid duplicates.

## Expiration time

- Set `expirationTime` to a reasonable future time window (long enough for the dialer to place the attempt, short enough to avoid stale outreach).
- If a request expires, treat it as “not attempted” and decide whether to resubmit with a new time window (and a new token if you consider it a new request).

## Batch sizing and throttling

- Submit in **small batches** (API limit is 25 per request).
- Rate limit submissions to stay within throttling limits.
- Use controlled concurrency and retries with backoff for throttling errors.

## Failure handling

Treat failures as either:

- **Retryable** (throttling / transient) → retry with backoff.
- **Permanent** (invalid phone number, invalid campaign state) → fix data or campaign state and resubmit.

## Operational checklist

- Track acceptance/rejection per dial request (log the response).
- Monitor campaign state and stop submitting if the campaign is stopped/paused.
- Keep dial-request submissions idempotent and resumable (job checkpoints).

