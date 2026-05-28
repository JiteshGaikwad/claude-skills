# Contact Control Panel (CCP) -- Daily Usage

The Contact Control Panel (CCP) is the primary interface agents use to handle contacts in Amazon Connect. It can run embedded in the agent workspace or standalone at `https://{instance}.my.connect.aws/ccp-v2/`.

---

## Agent Status Management

Agent status (availability state) controls whether contacts are routed to the agent and appears in real-time metrics.

### Built-in Statuses

| Status | Routable | Description |
|---|---|---|
| **Available** | Yes | Agent is ready to receive contacts. Contacts route based on routing profile channel/concurrency settings. |
| **Offline** | No | Agent is not available. No contacts route. This is the default state at login and after explicit sign-out. |
| **Error** | No | System-set state when a contact is missed or a system error occurs. Agent must manually change to Available or another status to resume. |

### Custom Statuses

Administrators create custom statuses in the Amazon Connect console (e.g., Break, Lunch, Training, Meeting, Coaching). Custom statuses are always **non-routable** -- agents in a custom status do not receive contacts.

Custom statuses appear in the agent status dropdown in the CCP. They are used for:

- Workforce management adherence tracking.
- Real-time metrics reporting (see which agents are on break vs. in training).
- Historical reporting on agent time allocation.

### Changing Status

- Click the agent status indicator at the top of the CCP.
- Select the desired status from the dropdown.
- If the agent has active contacts, the status change takes effect after all contacts are cleared (next-status queuing).
- Agents cannot change to Available while in Error state without first acknowledging the error.

### Next-Status Behavior

When an agent sets a new status while handling a contact:

1. The requested status is queued as the "next status."
2. The agent continues handling the current contact.
3. When all contacts are cleared, the agent transitions to the queued status automatically.
4. If the agent selects a different status before clearing, the most recent selection wins.

---

## Handling Voice Calls

### Accepting Inbound Calls

1. An incoming call notification appears in the CCP with the queue name and any available caller information.
2. Click **Accept** to connect. The call connects and the agent hears the customer (or whisper flow, if configured).
3. If the agent does not accept within the configured timeout, the contact is missed and the agent enters Error state (configurable).

### Making Outbound Calls

1. Click the phone/number pad icon in the CCP.
2. Enter the destination number using the number pad or paste from clipboard.
3. Select the outbound caller ID (if multiple are configured).
4. Click **Call**. The CCP connects to the agent first, then dials the customer.

### Hold and Resume

- **Hold** -- click the Hold button. The customer hears hold music (configured in the queue). The agent can perform other actions (look up information, consult with another party).
- **Resume** -- click the Resume button to reconnect with the customer.
- Hold time is tracked separately in metrics and CTRs.

### Transfer

Transfer sends the contact to another agent, queue, or external number:

1. Click **Transfer** (or the quick connect icon).
2. Select a quick connect destination (agent, queue, or phone number) or enter a phone number manually.
3. Two transfer types:
   - **Cold transfer (blind)** -- the contact transfers immediately. The originating agent disconnects.
   - **Warm transfer (consult)** -- the agent connects to the transfer target first, briefs them, then completes the transfer. The customer is on hold during the consult.

### Conference

Conference creates a three-way (or multi-party) call:

1. While on a call, click **Add participant** or initiate a quick connect.
2. The agent connects to the third party while the customer is on hold.
3. Click **Join** or **Conference** to merge all parties.
4. The agent can drop individual participants or leave the conference.

### Disconnect

- Click **End call** to disconnect the customer.
- The agent transitions to After Contact Work (ACW) state.
- During ACW, the agent completes post-call activities (notes, disposition) before clicking **Clear contact**.

---

## Handling Chats

### Multi-Chat Concurrency

Agents can handle multiple chat contacts simultaneously based on their routing profile concurrency settings (default: up to 10 concurrent chats).

Each active chat appears as a tab or conversation in the CCP. The agent switches between chats by clicking the tab.

### Accepting Chats

1. An incoming chat notification appears with the queue name and, if available, customer information.
2. Click **Accept**. The chat window opens with any existing conversation history (if persistent chat is enabled).

### Sending Messages

- Type in the message input field and press Enter or click Send.
- Messages support plain text. Rich formatting depends on the chat widget configuration.
- Attachments can be sent if file sharing is enabled (images, PDFs, up to configured size limits).

### Quick Responses in Chat

- Click the quick responses icon or type a shortcut prefix.
- Search for a response by keyword.
- Click to insert the response into the message field. Edit if needed before sending.

### Ending Chats

- Click **End chat** to disconnect. The agent enters ACW for that chat contact.
- If the customer disconnects first, the agent receives a notification and enters ACW.
- Chat contacts have an idle timeout -- if neither party sends a message within the configured period, the chat auto-disconnects.

---

## Handling Emails

### Accepting Emails

1. Inbound emails arrive in the agent's inbox (visible in the CCP).
2. Click **Accept** to open the email in the workspace.

### Rich Text Editor

The email editor supports:

- Bold, italic, underline, strikethrough.
- Bullet and numbered lists.
- Hyperlinks.
- Inline images.
- Font size and color adjustments.
- Code blocks.

### Templates

- Email templates are pre-built responses created by administrators.
- Agents select a template from the template picker.
- Templates can include placeholder tokens that auto-fill from contact attributes or customer profile data.
- Agents can edit the template content before sending.

### Signatures

- Agents can configure a personal email signature in their settings.
- Signatures are appended automatically to outbound emails.
- Administrators can set a default organizational signature.

### Threading

- Emails maintain thread context -- replies are grouped with the original email.
- Agents see the full email thread history when handling a reply.
- Thread IDs link related emails in the CTR and reporting.

### Forwarding

- Agents can forward emails to external addresses or internal queues.
- Forwarded emails include the original message content and any attachments.

### Attachments

- Agents can attach files to outbound emails.
- Attachment size limits are configured per instance.
- Supported file types are configurable by administrators.

---

## Handling Tasks

### Accepting Tasks

1. Task contacts appear in the CCP inbox like calls and chats.
2. Click **Accept** to start working on the task.
3. The task details (description, references, linked contacts) appear in the workspace.

### Creating Tasks

1. Click the **Create task** button in the CCP.
2. Select a task template (if configured) or use the default form.
3. Fill in required fields (name, description, assignee/queue).
4. Optionally link the task to the current contact.
5. Optionally schedule the task for a future date/time.
6. Click **Create**.

### Pause and Resume

- **Pause** -- click Pause on an active task. Select a pause reason. The task stops counting against the agent's concurrency, allowing them to handle other contacts.
- **Resume** -- click Resume to reactivate the task. It counts against concurrency again.
- Paused tasks have a configurable expiry -- if not resumed within the expiry window, they can be auto-routed back to the queue.

### Completing Tasks

- Click **End task** or **Complete** when the work is done.
- The agent enters ACW for the task contact.
- Clear the contact to finish.

---

## Transfers and Conferences

### Quick Connects

Quick connects are pre-configured transfer destinations that appear in the transfer dialog:

| Type | Destination |
|---|---|
| **Agent** | Transfer to a specific agent. |
| **Queue** | Transfer to a queue (re-enters routing). |
| **Phone number** | Transfer to an external phone number. |

Administrators configure quick connects in the Connect console and assign them to queues. Agents only see quick connects associated with the queue of the current contact.

### Transfer to Phone Number

For ad-hoc transfers to numbers not in quick connects:

1. Click Transfer.
2. Select "Phone number" tab.
3. Enter the destination number.
4. Choose cold or warm transfer.

### Conference Workflow

1. Agent places customer on hold.
2. Agent dials the third party (via quick connect or number pad).
3. Agent briefs the third party.
4. Agent clicks **Join all** to create the conference.
5. All parties can speak.
6. Agent can leave the conference (customer and third party continue) or end it.

---

## Device Settings

### Phone Type

| Setting | Description |
|---|---|
| **Softphone** | Audio handled through the browser using WebRTC. Requires microphone access and stable network. This is the default and recommended setting. |
| **Desk phone** | Audio routed to a physical phone number (agent's desk phone or mobile). The CCP handles signaling only. Useful when browser audio is unreliable. |

Agents set their phone type in the CCP settings (gear icon). Changes take effect on the next contact.

### Audio Device Selection

When using softphone:

- Select the microphone input device.
- Select the speaker/headset output device.
- Adjust ringer volume and ringer device (can ring on speakers while audio goes to headset).
- Test audio devices using the built-in audio check.

---

## Language Preferences

Agents can set their preferred language for the CCP interface:

- The CCP supports multiple languages for UI labels and controls.
- Language preference is set in the agent's settings.
- Language changes take effect immediately.
- The language setting does not affect contact routing or customer-facing language -- it only changes the CCP UI.

Supported languages include English, Spanish, French, German, Japanese, Korean, Portuguese, Chinese (Simplified and Traditional), and others based on the Connect instance region.

---

## Number Pad and DTMF

The number pad in the CCP serves two purposes:

### Dialing

- Used to enter phone numbers for outbound calls.
- Supports copy-paste from clipboard.
- Validates number format based on outbound calling configuration.

### DTMF (Dual-Tone Multi-Frequency)

- During an active call, the number pad sends DTMF tones.
- Used when navigating external IVR systems (e.g., transferring to an external queue that requires menu selections).
- Each key press sends the corresponding DTMF tone in real time.

---

## Inbox for Inbound Contacts

The inbox is the contact queue visible in the CCP where inbound contacts arrive:

- Contacts appear in the inbox when routed to the agent.
- Each contact shows: channel type (voice/chat/email/task), queue name, wait time, and available customer information.
- Voice calls auto-present with a ringtone -- the agent must accept or the contact is missed.
- Chats and emails queue in the inbox and wait for the agent to accept.
- Tasks queue similarly to chats.
- The inbox respects the agent's routing profile concurrency -- if the agent is at max capacity for a channel, no additional contacts of that channel type are presented.

### Contact Priority

Contacts are presented based on:

1. Priority setting (configurable in contact flows).
2. Age in queue (longest-waiting first within the same priority).
3. Channel routing order (configurable in routing profile).

### Launch and Login
- **CCP URL**: `https://{instance-alias}.my.connect.aws/ccp-v2/`
- **Login options**: Connect-managed credentials, SAML SSO redirect, AD credentials
- **First-time setup**: Allow microphone access, allow popups, enable third-party cookies
- **Cookie requirements**: Third-party cookies must be enabled for CCP to function

### Forward Calls to Mobile
- Set agent phone type to "Desk phone"
- Enter mobile number in E.164 format (e.g., +14155551234)
- Incoming calls ring the mobile device
- **Limitations**: No softphone UI features (hold/mute buttons); use DTMF or physical phone
- **Use case**: Remote agents without reliable internet for softphone

### View Agent Schedule
- Schedule tab in agent workspace (requires WFM enabled)
- Shows today's timeline: shifts, breaks, training, time off
- Submit time-off requests directly from schedule view
- Weekly view for upcoming schedule
