# What Is Amazon Connect (Connect Customer)?

Amazon Connect (“Amazon Connect Customer” in some documentation) is a cloud contact center that supports multiple channels (voice, chat, email, tasks) and integrates routing, flows, analytics, and agent desktop experiences into one service.

Core building blocks:

- **Instance**: the container for configuration, identity, telephony, storage, and feature enablement.
- **Queues + routing profiles**: how contacts are routed to agents and what agents can handle.
- **Flows**: the orchestration layer for customer experience and automation.
- **Agent desktop**: Contact Control Panel (CCP) and Agent Workspace.
- **Analytics**: real-time/historical metrics, Contact Lens (conversational analytics), evaluations.
- **Data services**: Customer Profiles and Cases.
- **Integrations**: Lambda, EventBridge, Kinesis, AppIntegrations, and API-driven automation.

Where to go next in this repo:

- Getting started checklist: `core/get-started.md`
- Next steps after initial setup: `core/next-steps.md`
- Admin Guide tutorial walkthroughs: `core/tutorials/intro.md`
- Instance + identity + security fundamentals: `core/instances.md`, `core/identity-management.md`, `core/security.md`
- Flows: `flows/overview.md`, `flows/blocks.md`
- Channels: `channels/*`
- Agent Workspace: `agent-experience/*`
