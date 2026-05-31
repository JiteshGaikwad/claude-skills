# Voicemail Express V3 (VMX3) — Overview

Amazon Connect has **no native voicemail capability**. Voicemail Express (VMX3) is AWS's official open-source add-on that closes that gap, giving any standard Amazon Connect instance basic voicemail for both agents and queues.

> **Works out of the box, by default.** VMX3 is a deploy-and-go solution. It is "designed to work seamlessly behind the scenes, providing voicemail options for all agents and queues by default." Once deployed, voicemail is available to every agent and every queue without per-agent or per-queue setup. A typical deployment can be completed and validated in **under 15 minutes**.

VMX3 is an evolution of the original Voicemail Express solution built by the AWS team that worked on Service Cloud Voice (where a cloned version ships and runs at scale). V3 strips out the Salesforce-centric pieces and delivers the same easy-to-deploy voicemail experience for standard Amazon Connect customers. The solution is intentionally lightweight: as few AWS services as possible, minimal change to an existing Connect deployment, a limited and focused feature set, and **all code open and customizable**.

---

## The Problem It Solves

Amazon Connect routes live calls but has no built-in way to capture a message when no agent is available, when the contact center is closed, or when a caller specifically wants to leave a message for a person or queue. VMX3 adds that missing capability:

- Callers can leave a voicemail **for an individual agent** or **for an Amazon Connect queue**.
- Voicemails are captured during the contact flow, processed **post-call**, and delivered to agents/supervisors as work items (Tasks) or as email.
- Each voicemail can include a **transcription**, an optional **generative-AI summary**, a **presigned URL** to the recording, and the original contact data.

---

## Architecture

VMX3 is an event-driven pipeline. Audio is captured inside Amazon Connect, then a chain of Lambda functions — each triggered by the previous step's output — processes and delivers the voicemail. Everything is provisioned via CloudFormation.

| AWS Service | Role in VMX3 |
|---|---|
| **Amazon Connect** | Captures voicemail audio using **built-in IVR recording**; sets contact attributes that flag and route the voicemail; receives delivered voicemails as Tasks/Guided Tasks. |
| **Amazon Kinesis Data Streams** | Carries the Contact Record (CR) emitted at the end of the call; CR generation is the trigger that starts post-call processing. |
| **AWS Lambda** | Runs the processing pipeline (Timestamper, Recording Processor, Transcriber, Packager, Presigner, plus per-mode delivery sub-functions). All functions are **Python 3.13**. |
| **Amazon S3** | Stores trimmed voicemail `.wav` recordings and transcription output (in separate buckets); object-creation events trigger the next Lambda in the chain. |
| **Amazon Transcribe** | Generates the voicemail transcript in the caller's language. |
| **Amazon Bedrock** | Produces the optional generative-AI summary of the voicemail. |
| **Amazon SES** | Delivers voicemails via email when the email delivery mode is used. |
| **AWS KMS** | Encrypts buckets/streams; supports both AWS-managed and **customer-managed KMS keys**. |
| **AWS CloudFormation** | Deploys, upgrades, and removes the entire solution stack. |

![High-level data flow] IVR recording → S3 `.wav` → Transcribe → (optional) Bedrock summary → presigned URL → Packager → delivery (Task / Guided Task / SES email).

---

## End-to-End Workflow

1. **Caller opts in.** During the contact flow, the customer is offered the option to leave a voicemail (for whatever reason) and chooses to do so.
2. **Contact attributes are set** to mark the contact as a voicemail (see the attribute reference below). The triggering flag (`vmx3_flag`) should be set **just before recording begins** — not too early, or a caller who changes their mind and hangs up can cause errors.
3. **Built-in IVR recording starts**, capturing the customer audio. The **VMX3-Voicemail-Timestamper** Lambda is invoked *during the call* to mark the exact moment the voicemail portion begins.
4. **Caller finishes and hangs up.** Callers can simply hang up when done — no tone or prompt to wait for.
5. **A Contact Record (CR) is emitted** through the Amazon Kinesis Data Stream. *Nothing happens until the CR leaves the stream* — CR creation can take a few seconds up to a couple of minutes, so voicemail delivery is correspondingly delayed.
6. **VMX3RecordingProcessor Lambda** (triggered by the CR):
   - Identifies the contact as a voicemail and retrieves key contact data.
   - Loads the IVR recording from S3 and **trims it using the timestamp** set in step 3 (IVR recordings can contain non-voicemail portions of the call).
   - Writes the trimmed voicemail `.wav` to an S3 bucket, attaching the extracted data as object metadata.
7. **VMX3Transcriber Lambda** (triggered by the recording object being created):
   - Retrieves the recording and its metadata.
   - Starts an Amazon Transcribe job in the caller's language; the completed transcript is written to a separate S3 bucket.
8. **VMX3Packager Lambda** (triggered by the transcript object being created):
   - Retrieves the transcript and uses it to identify the contact and locate the recording.
   - Performs the **generative-AI summary** (if enabled for the call).
   - Reads the recording's metadata.
   - Invokes **VMX3-Presigner** to generate a presigned URL for the recording (for standard **Task** and **Email** modes only).
   - Determines queue/agent, destination, etc., and invokes the **delivery sub-function** for the selected mode.
   - On successful delivery, deletes the completed transcription job.
   - Finally, **clears the voicemail flag** (`vmx3_flag → 0`) so re-processing of the same CR does not create duplicate voicemails.

---

## The Major V3 Change: Built-in IVR Recording (KVS Removed)

Earlier versions relied on **Kinesis Video Streams (KVS)** to capture voicemail audio. V3 **switched entirely to Amazon Connect's built-in IVR recording and removed KVS completely.** Why this matters:

- **Lower cost and complexity** — KVS capture/processing added both. IVR recording is native to Connect.
- **Faster and more scalable** capture.
- **GovCloud support** — with KVS gone, *all* solution components are now available in AWS GovCloud.

Because IVR recording can include parts of the call that are *not* the voicemail, V3 introduces the **VMX3-Voicemail-Timestamper** Lambda. It runs during the call and records the moment the voicemail portion starts. The Recording Processor later duplicates and trims the IVR recording at that timestamp, so the solution still produces a clean voicemail **even in environments that already use 100% IVR recording**. The Timestamper is deliberately simple and needs no permissions beyond basic Lambda execution — if a call errors out mid-flow, the cause is almost always the contact flow, not the voicemail Lambdas.

---

## Generative-AI Summary

VMX3 can generate a concise summary of each voicemail so users can quickly see **who called, why, and how to reach them back**.

- **Default model:** **Amazon Nova Lite** in all regions except GovCloud.
- **GovCloud:** **Claude Sonnet**.
- **Toggle granularity:** the summary can be turned on/off **per call**, and an **instance-wide default** can be set during implementation.
- Validated to work on voicemails up to **25 minutes** long, including the summary step.

---

## Direct Agent Routing

VMX3 can route a voicemail to a specific agent. V3 does this using the **preferred-agent routing criteria** instead of agent personal queues. Benefits:

- Managers can **easily re-allocate** voicemails if an agent is unavailable for an extended period.
- **Queue reporting data is preserved** (personal queues would otherwise distort it).
- Agents with direct customer relationships still get the personal-engagement benefit.

To direct a voicemail to an agent, set `vmx3_target` to `agent` and supply `vmx3_preferred_agent` (the agent's username).

---

## Delivery Modes

VMX3 supports three delivery modes, selectable per call via the `vmx3_mode` attribute or set as an implementation default. (See **[delivery-modes.md](./delivery-modes.md)** for full detail on each.)

**Task** — The voicemail is delivered as a standard Amazon Connect Task containing the transcript, optional AI summary, contact data, and a presigned URL to the recording. The agent opens the task and clicks the link to listen.

**Guided Task** — Delivered as an Amazon Connect Guided Task (a custom view/agent-guide experience). Built around an embedded media player and a richer guided layout, this mode does not require a presigned URL on the task itself. It is the **default delivery mode** (`vmx3_mode` defaults to `guided_task`).

**Email** — The voicemail is sent as an email via **Amazon SES**, with an HTML layout containing the transcript, optional AI summary, contact data, and a presigned URL to the recording. Adds email routing/delivery time on top of the normal processing delay.

---

## Contact Attributes Reference

Voicemail processing depends entirely on contact attributes set in the flow. Set them just before recording begins.

| Attribute | Required | Values / Format | Purpose |
|---|---|---|---|
| `vmx3_flag` | Yes | `0` / `1` | **The trigger.** Set to `1` to flag the call for processing. Packager resets it to `0` when done. Anything other than `1` (or missing) = the CR is ignored and **no voicemail is created**. |
| `vmx3_target` | Yes | `agent` / `queue` | Whether the voicemail goes to a queue or sets preferred-agent routing. |
| `vmx3_from` | Yes | E.164 phone number | Caller's number. Typically set from `$.CustomerEndpoint.Address`. |
| `vmx3_queue_arn` | Yes | Queue ARN | Target queue ARN; used to derive queue, instance ID, etc. Typically set from `$.Queue.ARN`. |
| `vmx3_preferred_agent` | Required for agent-directed routing | Agent username | The agent the voicemail is directed to. |
| `vmx3_lang` | Yes | Transcribe language code (default `en-US`) | Language Amazon Transcribe uses. |
| `vmx3_mode` | Yes | `email` / `task` / `guided_task` (default `guided_task`) | Delivery mode for this voicemail. |
| `vmx3_timestamp` | Yes | Set by the Timestamper Lambda | Marks the start of the voicemail so the recording can be trimmed. |

### Optional / delivery-mode attributes

All optional; each falls back to a deployment default. Set only to override per call.

| Attribute | Mode | Values / Format | Purpose |
|---|---|---|---|
| `vmx3_genai_summary` / `vmx3_do_genai_summary` | any | `true` / `false` | Per-call GenAI toggle. The recording processor reads/defaults `vmx3_genai_summary`; the Packager acts on `vmx3_do_genai_summary` (`true` enables). Falls back to the deployment default (`false` if unset). |
| `vmx3_task_flow` | task | Contact-flow ID | Custom task flow to process the generated task. Default: `default_task_flow`. |
| `vmx3_guided_task_flow` | guided_task | Contact-flow ID | Custom guided-task flow. Default: `default_guided_task_flow`. |
| `vmx3_email_from` | email | Email address | FROM address. Default: `default_email_from`. |
| `vmx3_email_to` | email | Email address | TO address (must contain `@`). If absent and not derivable from agent/queue data, falls back to `default_email_target`. |
| `vmx3_email_template` | email | SES template name | Template to render this email. Default: agent vs queue template chosen by `vmx3_target`. |

For **email** routing, the agent email field is selected at deployment
(`Email`, `SecondaryEmail`, or `Username`; SAML uses the latter two). Queue email comes
from the queue tag `vmx3_queue_email`. The Packager also injects template helpers
referenceable via `{{double_braces}}`: `vmx3_from`, `vmx3_agent_name`,
`vmx3_presigned_url`, `vmx3_transcript_contents`, `vmx3_genai_summary`. See
**[delivery-modes.md](./delivery-modes.md)**.

---

## Key Specifications & Limits

| Item | Value / Behavior |
|---|---|
| **Presigned URL lifetime** | Maximum **7 days**; invalid after that. |
| **Recording lifecycle** | Separately configurable retention window; option to **keep, archive, or delete** recordings once the window is met. Set during deployment. |
| **Max voicemail length (default)** | **1 minute** (increase/decrease via the longer-messages configuration). |
| **Validated max voicemail length** | Up to **25 minutes**, including generative summary. |
| **Transcript truncation** | Transcripts longer than **2048 bytes are truncated** to ensure Task compatibility. |
| **Load tested** | Up to **2,000 voicemails per hour**. |
| **Lambda runtime** | **Python 3.13** across all functions. |
| **Security** | S3 bucket versioning supported; validated with **customer-managed KMS keys**. |
| **In-queue voicemail** | Supported, with example flows for clean, uninterruptable in-queue voicemail. |
| **Self-service** | Documented configuration allows agents to self-service voicemails. |

**Roadmap direction:** keep the solution lightweight and replace components with native Amazon Connect features as they become available — e.g., reduce external-service dependence, move to Contact Lens-generated transcriptions once available for IVR recordings, and attach recordings directly to Tasks (removing the need for presigned URLs) once supported.

---

## Deeper Documentation in This Skill

- **[deployment.md](./deployment.md)** — Prerequisites, installation (standard and GovCloud), upgrade, and uninstall.
- **[delivery-modes.md](./delivery-modes.md)** — Task, Guided Task, and email modes in detail.
- **[advanced-options.md](./advanced-options.md)** — Self-service, in-queue voicemail, longer messages, customer-managed KMS keys.
- **[code-pipeline.md](./code-pipeline.md)** — The Lambda functions, S3/Kinesis triggers, and processing internals.
- **[troubleshooting.md](./troubleshooting.md)** — Common voicemail issues and how to diagnose them.

---

*Source: documents the public repository `amazon-connect/voicemail-express-amazon-connect` (Voicemail Express V3, release 2025.09).*
