# Amazon Connect — How Contacts Are Counted (Quotas)

This page summarizes how Amazon Connect counts “active contacts” for concurrency-style quotas, and how to reason about the most common edge cases.

## Counted as concurrent active contacts

The following are counted toward “concurrent active calls/contacts per instance” style quotas:

- Contacts that are **handled by a flow**
- Contacts that are **waiting in queue**
- Contacts that are **handled by an agent**
- **Outbound calls** created/placed by the system

## Not counted (until they become active)

- **Callbacks waiting in a callback queue** are not counted until the callback is actually offered to an available agent.
- **External transfers** are not counted.

## What happens when you exceed the quota

For voice contacts, exceeding the active-call quota results in a **reorder tone** (fast busy), indicating there is no available transmission path.

## How to determine your configured quota

Two practical methods:

1) **CloudWatch-derived calculation**
- Use CloudWatch metrics to compute the current/max concurrent usage and compare to the configured quota.
- Pair this with alarms (see `core/quotas-planning.md`).

2) **Queue contact-limit trick (voice/chat/tasks combined)**
- In the queue settings UI, set “Maximum contacts in queue” to an exceptionally large number.
- The validation error reveals that the value must be at least “1 less than the combined quota”, effectively exposing the combined quota for calls + chats + tasks.

Keep this as a diagnostic only; don’t run production operations with artificially high queue limits.

