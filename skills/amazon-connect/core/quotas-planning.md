# Amazon Connect — Quotas Planning and Operations

This page consolidates quota planning guidance (production launch, ongoing operations, and emergency events).

If you’re looking for **hard limits** (feature specifications that cannot be increased), see `core/feature-specifications.md`.

## Planning for production launch

Before launch:

- Treat quotas as a first-class part of your migration plan.
- Identify which quotas you will push (agents, concurrency, campaigns, cases, attachments, etc.).
- Submit quota increase requests early (don’t wait until cutover week).

Data you typically need to size quotas:

- Current number of agents
- Call/contact volume metrics
- Average contact duration
- Any seasonality or “peak event” expectations

## Ongoing operations management

Recommended operational posture:

- Monitor quota usage using CloudWatch metrics.
- Create alarms at ~**80%** of key limits so you have time to request increases before outages.

## Managing emergency events

If you need urgent help during an incident:

- Open a high-severity AWS Support case (severity depends on support plan).
- Engage your account team (TAM/SA) if applicable.

During high-volume events, balance experience and quotas:

- Use flow messaging (for example “estimated wait time”) to set expectations.
- Enable queued callbacks where appropriate.
- Reduce avoidable workload spikes (avoid unnecessary retries/loops in integrations).
