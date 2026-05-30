# Contact Dispositions — Tracking Recommendations

This page consolidates the Developer Guide’s recommendations for tracking outbound contact outcomes (“dispositions”) based on contact termination reasons.

Primary data-model reference:

- `analytics/contact-records.md` (CTR fields like `DisconnectReason`)

## Why this matters

For outbound campaigns and automated outreach, you want consistent reporting that answers:

- Did the attempt succeed?
- If it didn’t, should we retry? If yes, when and how?

## Recommended buckets

Group outcomes into three operational buckets:

- **Success** — reached the customer (or completed as intended).
- **Expired** — request wasn’t placed in time (or was abandoned due to time window).
- **Failed** — telecom/system failures where retry policy depends on the exact reason.

## Practical workflow

1. Capture the raw reason field(s) (for example `DisconnectReason` and any campaign-specific response fields).
2. Map raw reasons into the buckets above for dashboards and decisioning.
3. For “Expired” and some “Failed” reasons, decide whether to resubmit with a new time window.

## Recommendation matrix (outline)

Use a decision table keyed by your observed reasons, for example:

- If outcome indicates **expired** → resubmit (new time window; choose token policy consistently).
- If outcome indicates **telecom problem** → retry with backoff and a cap (to avoid repeated bad numbers).
- If outcome indicates **resource error** → investigate capacity/configuration, then retry after remediation.

This repository keeps the authoritative raw reason/value lists in `analytics/contact-records.md`; keep the mapping table here aligned with those values.

