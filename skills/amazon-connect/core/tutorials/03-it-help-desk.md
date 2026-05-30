# Tutorial 3: Create an IT Help Desk (Lex + Routing + Flow)

Goal: create an IT Help Desk contact center by adding an Amazon Lex bot, setting up routing, creating a contact flow, assigning it to the phone number, and testing the custom voice + chat experience.

The Administrator Guide describes the Lex bot as the first step to collect customer intent (for example, password reset vs network issue), then uses that input in a flow to route to the correct queue.

## Prerequisites

The Admin Guide frames this tutorial as part of a series. If you didn’t complete Tutorial 1, you need:
- an AWS account
- a configured Connect Customer instance
- a Connect Customer administrative account
- a claimed phone number

## Tutorial outline (Admin Guide “Contents”)

1) Create an Amazon Lex bot
2) Add permissions to the Lex bot
3) Set up routing
4) Create a contact flow
5) Assign the contact flow to the phone number
6) Test a custom voice and chat experience

## Step 1 (expanded): Create an Amazon Lex bot

The Admin Guide notes:
- This step uses the **Amazon Lex console** (not the Connect Customer console).
- If it’s your first time using the Lex console, the “Get Started” experience differs from returning users.

It breaks Step 1 into parts:

- Part 1: Create an Amazon Lex bot
- Part 2: Add intents to your Lex bot
- Part 3: Build and test the Lex bot

Example configuration values used by the Admin Guide (adjust as needed):
- Bot name: `HelpDesk`
- IAM permissions: create a role with basic Lex permissions
- COPPA: choose appropriately for your use case
- Idle session timeout: choose per your experience requirements
- Language + voice: choose appropriate options; the guide calls out a common default voice (“Joanna”) for Connect Customer

Related docs:
- `ai/lex-bots.md`
- `flows/blocks.md`
- `core/routing-and-queues.md`

