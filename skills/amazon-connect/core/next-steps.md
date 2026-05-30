# Next Steps (After Initial Setup)

After you’ve created your instance, claimed numbers, set up routing, and built initial flows, the Administrator Guide calls out a couple of high-value next steps to optimize operations.

## 1) Monitor live and recorded conversations

Monitoring and review are core for coaching and quality.

- **Voice**: enable call recording via flow configuration (recording is controlled in flows, not only at the instance level).
- **Chat**: enable chat transcript recording at the instance level (storage configuration must be in place).

Related docs in this repo:
- `flows/blocks.md` (recording-related blocks and flow patterns)
- `analytics/monitoring.md`
- `core/instances.md` (storage configuration; recordings/transcripts)

## 2) Add conversational AI bots (Lex)

Use Amazon Lex to reduce agent load and automate common interactions. Typical patterns:

- Use a bot to handle the first interaction (triage) before routing to an agent.
- Use a bot to answer frequently asked questions without transferring to an agent.

Related docs in this repo:
- `ai/lex-bots.md`
- `flows/blocks.md` (Lex integration blocks and routing patterns)

## Free online classes (listed in the Admin Guide)

The Admin Guide calls out these free classes to build operator proficiency:

- “Introduction to Connect Customer and the Contact Control Panel (CCP)”
- “Connect Customer: Introduction to the Administrative Interface”
- “Connect Customer: Creating and Managing Connect Customer Instances”
- “Connect Customer: Implementing Chat in Connect Customer”
- “Connect Customer: Implementing Tasks in Connect Customer”

