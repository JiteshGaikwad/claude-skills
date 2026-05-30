# Get Started with Connect Customer

Use this page as an **operator/admin checklist** for standing up a new Connect Customer contact center, based on the Administrator Guide “Get started with Connect Customer” + related “Tutorials” section (Voice ID intentionally excluded).

If you prefer hands-on labs, the Admin Guide points to the AWS Workshop Studio workshop “Introduction to Connect Customer”.

## Quick setup checklist

1) **Create a Connect Customer instance**
- An instance is the container for your contact center configuration (identity, telephony, storage, routing, flows, users).
- During creation you typically decide:
  - how users sign in (identity model),
  - whether inbound/outbound calling is enabled,
  - where data is stored (S3-backed storage configuration).
- See: `core/instances.md`, `core/identity-management.md`, `core/security.md`

2) **Set up phone numbers (if you use voice)**
- Claim an AWS-provided phone number (DID) and/or port existing numbers.
- If porting, the Admin Guide recommends **claiming a temporary number first** so you can build and test while porting completes.
- See: `core/telephony.md`, `channels/voice.md`

3) **Set up routing**
- Create **queues**, **routing profiles**, and **hours of operation**.
- In routing profiles, decide which channels agents handle (voice/chat/tasks) and concurrency limits (chat/task).
- See: `core/routing-and-queues.md`

4) **Build your first flows**
- Create/confirm your inbound experience using flows.
- A single flow can handle multiple channels (voice/chat/tasks); ensure each block is configured for the channels you intend to support.
- See: `flows/overview.md`, `flows/flow-designer.md`, `flows/blocks.md`

5) **Add users and configure agent settings**
- Add your managers and agents, assign routing profiles, set telephony (softphone vs deskphone), and configure after-contact work (ACW) behavior.
- See: `core/user-management.md`

6) **Enable customer chat experience (if you use chat)**
- Connect Customer provides options for enabling customer-facing chat experiences; follow the channel guidance and hosting/embedding requirements.
- See: `channels/chat-sms.md`, `channels/communications-widget.md`, `core/customer-authentication.md` (if you plan pre-chat auth)

## What to do immediately after “Day 0”

Start here: `core/next-steps.md`

Also review hard limits (feature specifications): `core/feature-specifications.md`

## Follow the Admin Guide tutorials (recommended)

The Admin Guide’s “Tutorials: An introduction to Connect Customer” are suitable for both knowledge workers and developers and are designed as a series:

- Tutorial 1: Set up your Connect Customer instance → `core/tutorials/01-set-up-instance.md`
- Tutorial 2: Test the sample voice and chat experience → `core/tutorials/02-test-sample-voice-chat.md`
- Tutorial 3: Create an IT help desk (Lex + routing + flow) → `core/tutorials/03-it-help-desk.md`
