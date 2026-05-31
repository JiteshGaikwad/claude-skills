# Voicemail Express V3 (VMX3): Advanced Configuration Options

Voicemail Express V3 (VMX3) is AWS's official open-source voicemail add-on for Amazon
Connect. After the base deployment, several advanced options extend or modify the default
behavior. This reference documents four of them in depth:

1. Agent self-service voicemails (pull/pick from a worklist)
2. In-queue voicemail
3. Handling voicemails longer than 25 minutes (validated at 5 min; up to ~25 min)
4. Customer-managed KMS keys (CMKs) for data streams and S3 buckets

> **Important — these are advanced configurations.** Each option below assumes you already
> have a working VMX3 deployment and a working knowledge of Amazon Connect flows, security
> profiles, and routing profiles. Customize the flows, roles, and resource settings to match
> your own environment.

---

## 1. Agent self-service voicemails (pull/pick from a queue)

### What it enables

Amazon Connect supports letting agents **self-assign contacts** to themselves. Because VMX3
delivers voicemails as **Amazon Connect Tasks**, this self-assign capability lets agents
**manually select voicemails from a worklist of voicemails waiting in queue**, instead of
having them route automatically.

Key properties of this model:

- It works **across contact types** (it is a general Amazon Connect self-assign feature),
  and VMX3 takes advantage of it because voicemails are Tasks.
- It is configured at the **routing profile**, so delivery can be **mixed across different
  agent pools** — some agents can have voicemails routed automatically, while others
  self-service them.
- It is useful when you want voicemails addressed first thing in the morning **without
  competing with incoming contacts**.
- It can also be used to make **personal voicemails** (voicemails targeted at a specific
  agent) available via a pick list.

> **Note:** This option is currently only available for agents using the **Agent
> Workspace**.

### Configuration overview

There are three configuration areas in the Amazon Connect instance, the third of which is
optional:

| # | Area | Purpose |
| --- | --- | --- |
| 1 | Security profile | Grants the "Assign to me" permission |
| 2 | Routing profile | Enables manual assignment of Task voicemails per queue |
| 3 | Agent status (optional) | A dedicated status for working voicemails off-routing |

### Step 1: Update the security profile

Users must have a security profile that allows them to manually assign contacts to
themselves.

1. Log in to the Amazon Connect administrative interface.
2. Select **Users** and choose **Security profiles**.
3. Find the security profile you want to modify and select its name to edit it.
4. In the **Contact Actions** section, select either **Allow 'Assign to me' for my
   contact** or **Allow 'Assign to me' for any contact**, as appropriate for your use case.
5. **Save** the security profile.

#### Difference between the two permissions

**Allow 'Assign to me' for any contact** — enables agents to view contacts under any of
these conditions:

- The current agent is the only Preferred Agent on the contact.
- The current agent is one of the Preferred Agents on the contact.
- Any agent or set of agents are Preferred Agents on the contact.
- The contact has no Preferred Agents.

**Allow 'Assign to me' for my contact** — enables agents to view contacts only under these
conditions:

- The current agent is the only Preferred Agent on the contact.
- The current agent is one of the Preferred Agents on the contact.

> This Preferred Agent scoping is the access-control mechanism for **personal voicemails**:
> with the "my contact" permission, an agent only sees voicemails for which they are a
> Preferred Agent.

### Step 2: Update the routing profile

Give the agent the ability to manually select tasks for each queue you want them to service.

1. Select **Users** and choose **Routing profiles**.
2. Find the routing profile you want to modify and select its name to edit it.
3. In the **Manual Assignment** section, select the queues you want agents to be able to
   self-service, then set the **Channels** to **Task** for each.
4. If you **do not** want tasks to automatically route, make sure to **remove tasks from
   the queues** in the **Queues** section (otherwise they will still auto-route).
5. Save the routing profile.

### Step 3 (optional): Create a status for manual voicemail work

If you want agents to remove themselves from the normal contact queue but still address
work in the worklist, create a new status that makes them **unavailable for routing** while
still allowing self-assignment.

1. Select **Users** and choose **Agent status**.
2. Choose **Add new agent status**.
3. Create a new status. Give it a clear name such as `Working Voicemail` to indicate the
   agent is self-assigning work.
4. Provide a description if desired, then choose **Save**.

### Validation

1. Log an agent into the Agent Workspace and put them into the new state.
2. Place a few voicemails into queue.
3. In the Agent Workspace, select the **Apps** dropdown and choose **Worklist**.
4. Once the voicemails are ready and the Tasks have been created, they appear in the
   worklist.
5. Selecting an item loads a **preview** of the Task.
6. Select **Assign to me**, then choose **Assign** in the popup. The Task routes to the
   agent.

---

## 2. In-queue voicemail

### What it enables

In-queue voicemail offers the customer the option to **leave a voicemail instead of
remaining in queue**. To do this, you effectively reproduce the functionality of the
`VMX3_Main_VM_Module` flow inside your **customer queue flow**, with one key difference:
**you must provide a mechanism to clear the voicemail flag if an agent takes the contact.**

### Why the flag must be cleared (the core problem)

In Amazon Connect, when a customer is in a queue flow and an eligible agent becomes
available, the caller is **routed to that agent regardless of what they are doing in the
queue flow**. This is desirable — we want customers to connect to agents. But if the
`vmx3_flag` is still set when the agent connects, the **entire conversation will be streamed
and processed as if it were a voicemail**.

To prevent this, an **agent whisper flow** clears the `vmx3_flag` setting when the agent
connects. The `VMX3_AWS_Test_Flow` has been updated to provide an example of how to set the
queue flow, agent whisper, and other attributes that make the in-queue experience possible.

### The three provided contact flows

This version of Voicemail Express provides three contact flows to deliver the experience:

| Flow | Role |
| --- | --- |
| `VMX3_In_Queue_Option` | Sample **customer queue flow** that loops recordings and provides the voicemail offer experience. Also sets the agent whisper flow. |
| `VMX3_In_Queue_Agent_Whisper` | Looks for the presence of `vmx3_flag` and sets it to `0` if it exists. |
| `VMX3_In_Queue_Customer_Whisper` | Lets the customer know they are now connecting to an agent. |

#### `VMX3_In_Queue_Option`

Provides the option to leave a voicemail instead of remaining in queue. It **tracks the
customer response to the offer** and **does not offer the option again if the customer
declines**, so the caller is not repeatedly asked the same question. If the customer does
select to leave a voicemail, the `VMX3_In_Queue_Agent_Whisper` and
`VMX3_In_Queue_Customer_Whisper` flows are set, and the rest of the experience mimics the
normal voicemail experience.

> This "ask once" behavior is how you implement an **uninterruptable / non-repeating**
> in-queue offer — the offer is presented and tracked, and a declined offer is not looped
> back to the customer.

#### `VMX3_In_Queue_Agent_Whisper`

Checks for the presence of `vmx3_flag` and, if it exists, **resets the value to `0`** so
that this contact is **not interpreted as a voicemail**. This is the safeguard that lets a
live agent connection proceed normally even though the customer was in the in-queue
voicemail experience.

#### `VMX3_In_Queue_Customer_Whisper`

Lets the customer know that an agent has become available and is connecting to the call.

### Implementing in your environment

- Use the **test flow** (`VMX3_AWS_Test_Flow`) as an example of how to configure the VMX3
  attributes **prior to queuing**.
- **Set the customer queue flow before doing the transfer.**
- Once you transfer to the provided customer queue flow (`VMX3_In_Queue_Option`), the
  **agent and customer whisper flows are set for you**.
- These whisper flows are **critical** should an agent become available and the call
  transfers — without them, the flag would not be cleared and the live conversation would
  be processed as a voicemail.

### Key contact attribute

| Attribute | Value | Set where | Purpose |
| --- | --- | --- | --- |
| `vmx3_flag` | (set) | `VMX3_In_Queue_Option` (when customer opts in) | Marks the contact for voicemail processing |
| `vmx3_flag` | `0` | `VMX3_In_Queue_Agent_Whisper` | Clears the flag when an agent connects, so the live conversation is not processed as a voicemail |

---

## 3. Handling voicemails longer than 25 minutes

### Validated baseline and the two considerations

Voicemail Express has been **validated at scale with messages up to 5 minutes long**. This
covers the majority of use cases. If, however, you need to support longer voicemails, there
are **two considerations**:

1. How much **memory** the Recording Processor Lambda function needs to process the message,
   and
2. How much **time** it takes to do it.

### Lambda defaults and why memory (not time) is the constraint

For the default deployment, the **Recording Processor** function is configured with **512 MB
of memory** and a **15-minute timeout**.

- **Timeout is not the concern.** With the new recording configuration, the function timeout
  should not be a concern, as the function can comfortably handle **nearly 4 hours** of audio
  processing time.
- **Memory is the concern.** The default Lambda memory is **128 MB**, which was **insufficient
  for a 5-minute message**, so it was moved up to **512 MB** (the next pricing-tier limit). At
  512 MB it should be able to comfortably handle **up to 25 minutes of audio** — however, this
  **has not been validated at scale**.

| Setting | Default value | Notes |
| --- | --- | --- |
| Recording Processor memory | **512 MB** | Raised from 128 MB (which was insufficient for a 5-minute message). 512 MB should handle up to ~25 min of audio (unvalidated at scale). |
| Recording Processor timeout | **15 minutes** | Not a constraint — can comfortably handle nearly 4 hours of audio processing time at the current configuration. |

### How do I know if my function is running out of memory?

If your voicemail **never makes it past the Recording Processor function**, check the
CloudWatch logs. You will likely see an entry similar to this:

```json
{
    "time": "2024-12-30T23:20:29.823Z",
    "type": "platform.report",
    "record": {
        "requestId": "671dc475-78c6-416b-b7ac-56b52940dd13",
        "metrics": {
            "durationMs": 32973.233,
            "billedDurationMs": 32974,
            "memorySizeMB": 128,
            "maxMemoryUsedMB": 125
        },
        "status": "error",
        "errorType": "Runtime.OutOfMemory"
    }
}
```

This indicates that you do not have enough memory on your function to complete the audio
processing. **Increase the amount of memory** assigned to your function and re-test.

### Configuring your flow module to accommodate longer messages

In the standard VMX module, the maximum amount of time you can allow for voicemail messages
is **180 seconds** — that is the **max timeout for a Get customer input block**. If you want
to extend that longer, you will need to **hold the call in the flow** using a number of
different approaches.

The most simple approach is to **create a loop with a Play prompt inside it**:

1. Using **SSML**, create a prompt that just **waits for up to 10 seconds**.
2. **Loop** until you have hit your maximum time.
   - Example: **100 loops of 10 seconds = 16 minutes and 40 seconds.**
   - Combined with the **180 seconds** of time in your Get customer input block, that provides
     **up to 19 minutes and 40 seconds**.

> **Worked example from the docs:** a flow with **3 minutes of Get customer input time +
> 5 minutes of loop time = 8 minutes** of recording time total.

| Component | Max contribution | Mechanism |
| --- | --- | --- |
| Get customer input block | 180 seconds (3 min) | The block's own max timeout |
| Play-prompt loop | up to 16 min 40 sec | 100 loops × 10-second SSML wait prompt |
| **Combined ceiling** | **~19 min 40 sec** | Get customer input + loop |

### Putting it together for longer messages

1. Confirm the **flow module** can hold the call long enough (Get customer input + SSML loop
   approach above) for your target message length.
2. Ensure the **Recording Processor memory** is high enough for the audio length (512 MB
   handles up to ~25 min; increase if you see `Runtime.OutOfMemory`).
3. The **timeout** generally does not need adjusting (15-minute default handles far more than
   the audio length).
4. **Test and re-test** at your target length, watching CloudWatch for out-of-memory errors.

---

## 4. Customer-managed KMS keys (CMKs) for data streams and S3 buckets

### What it covers

If you want to use **customer-managed KMS keys** for either the **Kinesis Data Stream** or
your **Amazon Connect recordings S3 bucket**, you must update the **VMX3 Lambda function
roles** to include a policy that allows access to those keys.

The easiest approach is to **create a new IAM policy** and attach it to whichever roles need
it.

### Roles that most likely need access

| Role | Resource access it needs |
| --- | --- |
| `VMX3_Recording_Processor_Role` | S3 `GetObject`; Kinesis Data Streams `GetRecords` |
| `VMX3_Transcriber_Role` | S3 `GetObject` |
| `VMX3_Guided_Flow_Role` | S3 `GetObject` |
| `VMX3_Presigner_Role` | S3 `GetObject` |
| `VMX3_Packager_Role` | S3 `GetObject` |

### Required KMS actions

The policy you create must include:

- `kms:Decrypt`
- `kms:GenerateDataKey`

### Example policy

Replace the account ID and key ID placeholders with your own values. The example below uses
the placeholder account `111122223333`:

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "VisualEditor0",
			"Effect": "Allow",
			"Action": [
				"kms:Decrypt",
				"kms:GenerateDataKey"
			],
			"Resource": "arn:aws:kms:us-east-1:111122223333:key/%YOUR_KEY_ID%"
		}
	]
}
```

### Least-privilege guidance

Use the **least-privilege permissions required**, limiting access to the **specific key(s)
and resources** rather than granting blanket KMS access. Scope the `Resource` element to the
exact key ARN(s) the VMX3 functions need, and attach the policy only to the roles from the
table above that actually require it.
