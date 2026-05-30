# Tutorial 2: Test the Sample Voice and Chat Experience

Goal: experience the agent and customer experience for voice and chat using the built-in sample flow and the Contact Control Panel (CCP), without writing code.

The Administrator Guide describes the CCP as the web UI agents use to accept and manage voice and chat contacts.

## Prerequisites

The tutorial series assumes you already have:
- an AWS account
- a configured Connect Customer instance
- an administrative user for the instance
- a claimed phone number

## Step 1: Handle a voice contact

1) In the instance navigation, go to **Dashboard**.
2) Choose **Test chat**.
3) On the Test Chat page, activate the CCP.
4) If prompted, allow microphone access.
5) If prompted, allow notifications.
6) In the CCP, set agent status to **Available**.
7) From a phone, call the number you claimed in Tutorial 1 (or locate it under Channels → Phone numbers).
8) When connected, interact with the sample inbound flow options, then choose the menu path that routes to an agent.
9) In the CCP, **Accept** the call.
10) End the call, then close the contact to return the agent to an available state.

Tip from the Admin Guide:
- You can launch the CCP from many places in the Connect Customer console using the phone icon in the top bar.

## Step 2: Handle a chat contact

1) Ensure you completed the voice-contact step first.
2) On the **Test chat** page, start a chat using the chat bubble UI.
3) Send a message as the customer; the sample inbound flow routes the chat into a queue.
4) In the CCP, accept the incoming chat contact.
5) Exchange messages, then end the chat and close the contact.

Next:
- Continue to Tutorial 3 to build a custom experience using Lex + routing + flows: `core/tutorials/03-it-help-desk.md`

Related docs:
- `agent-experience/ccp.md`
- `channels/chat-sms.md`
- `flows/blocks.md`

