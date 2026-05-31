# Voicemail Express V3 (VMX3) — Lambda Processing Pipeline

Reference map of the AWS open-source **Voicemail Express V3 (VMX3)** Lambda code that
powers voicemail capture, transcription, GenAI enrichment, and delivery for Amazon
Connect.

- **Runtime:** all Lambda functions run on **Python 3.13** (standardized in the 2025.09.12
  release).
- **Language/SDK:** Python with `boto3`.
- **License:** MIT-0.
- **Code location in the repo:** `Code/` (15 files). Source code version string at the time
  of writing: `2025.09.13`.

The pipeline is built from two kinds of files:

1. **Top-level pipeline functions** (`vmx3_*.py`) — each is its own Lambda with its own
   event trigger (Kinesis, S3/EventBridge, Connect invocation, separate-Lambda invoke, or
   CloudFormation).
2. **Modular helpers** (`sub_*.py`) — these are **NOT separate Lambdas**. They are Python
   modules **imported into and called in-process by `vmx3_packager`** (e.g.
   `import sub_connect_task` → `sub_connect_task.vmx3_to_connect_task(payload)`). The
   `sub_` prefix is the convention for "sub-routine of the packager." They are packaged
   into the packager's deployment artifact.

> **Capture architecture (2025.09.12):** VMX3 stopped using **Kinesis Video Streams
> (KVS)** for audio. Audio is now sourced from **Amazon Connect's native IVR call
> recording** stored in S3. The recording processor is still **triggered by the Connect
> CTR stream on a Kinesis *Data* Stream** — it decodes the CTR, finds the IVR-participant
> recording in S3, trims it to the voicemail portion, and re-saves it. See
> `troubleshooting.md` for the full changelog.

---

## End-to-end data flow

```
Caller is routed to the VMX3 voicemail experience in an Amazon Connect contact flow
        │   flow sets contact attributes incl. vmx3_flag=1, vmx3_lang, vmx3_queue_arn,
        │   vmx3_dialed_number/target, optional vmx3_genai_summary
        │   flow invokes the timestamper to stamp the voicemail start time
        ▼
[1] vmx3_voicemail_timestamper   (invoked from the contact flow, Invoke AWS Lambda block)
        │   returns vmx3_timestamp (ISO-8601, e.g. 2025-09-13T12:34:56Z) → stored as a contact attribute
        ▼
   Native Connect IVR call recording captures the voicemail audio to the recordings S3 bucket;
   the CTR for the contact is emitted to the Kinesis Data Stream
        ▼
[2] vmx3_recording_processor   (Kinesis Data Stream trigger — CTR records)
        │   base64-decodes the CTR, finds the Recording where ParticipantType == 'IVR',
        │   computes the time offset between vmx3_timestamp and the recording StartTimestamp,
        │   downloads the full IVR recording, TRIMS it to the voicemail portion (pydub),
        │   uploads recordings/<YYYY>/<MM>/<DD>/<contactId>.wav with S3 object tags
        │   (vmx3_lang, vmx3_queue_arn, vmx3_genai_summary)
        ▼
   New .wav object in the recordings bucket  →  EventBridge S3 object-created event
        ▼
[3] vmx3_transcriber   (EventBridge S3 PutObject on recordings bucket)
        │   reads the object's tags (vmx3_lang etc.), builds the recording URL,
        │   start_transcription_job(JobName='vmx3_<contactId>', WAV) → transcript JSON
        │   written to the transcripts bucket at <prefix>/<contactId>.json
        │
        ├── on SUCCESS ───────────► transcripts bucket gets <contactId>.json
        │
        └── on FAILURE (EventBridge "Transcribe Job State Change = FAILED")
                ▼
           [3b] vmx3_transcription_error_handler   (EventBridge trigger)
                │   writes a placeholder transcript JSON (a COMPLETED-shaped doc whose
                │   transcript text is a "Transcription failed…" message) to the same key
                ▼
           transcripts bucket gets placeholder <contactId>.json
        ▼  (EventBridge S3 PutObject on transcripts bucket)
[4] vmx3_packager   (EventBridge S3 PutObject on transcripts bucket — ORCHESTRATOR)
        │   ignores .write_access_check_file.temp objects, then runs 7 steps IN-PROCESS:
        │     1. init clients (lambda, transcribe, connect)
        │     2. sub_key_data_extraction.key_data_extraction(event)   → function_data + base data
        │     3. sub_process_transcript.process_transcript(payload)   → transcript text
        │            └─ calls sub_genai_summary if vmx3_genai_summary == true (Bedrock)
        │     4. sub_build_data_payload.build_data_payload(payload)    → sets vmx3_mode + delivery payload
        │     5. for task/email modes: invoke vmx3_presigner LAMBDA (RequestResponse) → presigned URL
        │            (for guided_task: stores vmx3_recording_key instead, URL made later in the flow)
        │     6. deliver by vmx3_mode:
        │            task        → sub_connect_task.vmx3_to_connect_task
        │            email       → sub_ses_email.vmx3_to_ses_email
        │            guided_task → sub_connect_guided_task.vmx3_to_connect_guided_task
        │     7. cleanup: delete_transcription_job(vmx3_<contactId>); set vmx3_flag='0' on the contact
        ▼
   Agent / recipient receives the voicemail (Connect task, guided task, or SES email)
        │
        ▼  (playback)
[5] vmx3_presigner            (invoked as a Lambda by the packager) → S3 presigned URL for task/email playback
    vmx3_guided_flow_presigner(invoked from the Guided Task agent flow) → presigned URL for the guided-flow media player

Utility (not in the runtime path):
[*] vmx3_ses_template_tool    (CloudFormation custom resource / direct invoke) — create/get/update/delete SES email templates
```

---

## Top-level pipeline functions

### `vmx3_voicemail_timestamper.py`
| | |
|---|---|
| **Runtime** | Python 3.13 |
| **Trigger** | Invoked from the Amazon Connect contact flow (Invoke AWS Lambda block). |
| **Purpose** | Stamp the moment the voicemail recording starts so the recording processor can compute where in the full IVR recording the voicemail begins. |
| **Inputs** | Connect invocation `event` (logged; not read). |
| **Logic** | Gets current UTC, formats as `%Y-%m-%dT%H:%M:%SZ` (Amazon Connect timestamp format). |
| **Outputs** | `{ "vmx3_timestamp": "<ISO-8601 Z>" }` — stored as a contact attribute by the flow. |
| **Position** | Step 1, inside the flow before recording is processed. |

### `vmx3_recording_processor.py`
| | |
|---|---|
| **Runtime** | Python 3.13 (depends on **pydub** for audio trimming). |
| **Trigger** | **Kinesis Data Stream** carrying Amazon Connect CTR records. |
| **Purpose** | From the CTR, locate the native IVR call recording in S3, trim it down to just the voicemail segment, and store it for transcription. |
| **Inputs** | Kinesis event `Records[]` (base64 CTR payload). Env: `vmx3_recordings_bucket`, `package_version`. CTR provides `ContactId`, `Attributes` (incl. `vmx3_timestamp`, `vmx3_lang`, `vmx3_queue_arn`, `vmx3_genai_summary`), and `Recordings[]`. |
| **Logic** | Decodes CTR; picks the recording with `ParticipantType == 'IVR'`; computes `vm_offset` = `vmx3_timestamp − recording StartTimestamp` (ms); downloads the source recording into memory; `AudioSegment.from_file(...)[offset:]` to trim; uploads to `recordings/<YYYY>/<MM>/<DD>/<contactId>.wav` with **S3 object tags** `vmx3_lang`, `vmx3_lang_value`, `vmx3_queue_arn`, `vmx3_genai_summary`. Defaults `vmx3_genai_summary` to `'false'` if absent. |
| **Outputs** | Trimmed `.wav` in the recordings bucket; string status. |
| **Position** | Step 2 — between native recording and transcription. |

### `vmx3_transcriber.py`
| | |
|---|---|
| **Runtime** | Python 3.13 |
| **Trigger** | **EventBridge S3 object-created** event on the recordings bucket (`event['detail']['object']['key']`). |
| **Purpose** | Start an **Amazon Transcribe** job for the trimmed voicemail recording. |
| **Inputs** | EventBridge S3 detail (bucket, key, region). Reads the object's **tags** to get `vmx3_lang`. Env: `s3_transcripts_bucket`, `package_version`. |
| **Logic** | Derives `contact_id` from the filename; builds the recording URL; `start_transcription_job(TranscriptionJobName='vmx3_<contact_id>', LanguageCode=<vmx3_lang>, MediaFormat='wav', OutputBucketName=<transcripts bucket>, OutputKey=<prefix>/<contact_id>.json)`. |
| **Outputs** | A running Transcribe job; transcript JSON lands in the transcripts bucket; string status. |
| **Position** | Step 3 — produces the transcript object that triggers the packager. |

### `vmx3_transcription_error_handler.py`
| | |
|---|---|
| **Runtime** | Python 3.13 |
| **Trigger** | **EventBridge** rule on Transcribe job state change = **FAILED** (`event['detail']['TranscriptionJobName']`). |
| **Purpose** | Resilience — still deliver a voicemail when transcription fails instead of losing it. |
| **Inputs** | EventBridge Transcribe detail; `event['account']`. Env: `s3_transcripts_bucket`. |
| **Logic** | Strips the `vmx3_` prefix from the job name to derive the key/`contact_id`; writes a **placeholder transcript** shaped like a normal Transcribe result (`status: COMPLETED`) whose transcript text is: *"Transcription failed. Please refer to the recording link below…"*. Writes it to the transcripts bucket at the original transcript key — which itself triggers the packager. |
| **Outputs** | Placeholder transcript object (drives the packager); status. |
| **Position** | Step 3b — failure branch parallel to the transcriber's success path. |

### `vmx3_packager.py`  ← **central orchestrator**
| | |
|---|---|
| **Runtime** | Python 3.13 (bundles the `sub_*` modules). |
| **Trigger** | **EventBridge S3 object-created** event on the transcripts bucket (real or placeholder transcript). |
| **Purpose** | Drive the 7-step enrichment + delivery sequence by **calling the imported `sub_*` modules in-process** and invoking the presigner Lambda. |
| **Inputs** | EventBridge S3 detail. Env: `AWS_LAMBDA_FUNCTION_NAME`, `package_version`, `presigner_function_arn`. |
| **Logic (7 steps)** | 1) Skip `*.write_access_check_file.temp`; init `lambda`/`transcribe`/`connect` clients. 2) `sub_key_data_extraction.key_data_extraction(event)` → core `function_data` + attributes. 3) `sub_process_transcript.process_transcript(payload)` (which internally calls `sub_genai_summary` when `vmx3_genai_summary == 'true'`). 4) `sub_build_data_payload.build_data_payload(payload)` → sets `vmx3_mode`. 5) For `task`/`email`: invoke `vmx3_presigner` Lambda (`RequestResponse`) → `vmx3_presigned_url` (on failure, substitutes a docs URL and continues); for `guided_task`: stores `vmx3_recording_key`. 6) Deliver per `vmx3_mode` → `sub_connect_task` / `sub_ses_email` / `sub_connect_guided_task`. 7) `delete_transcription_job('vmx3_<contact_id>')`; `update_contact_attributes(... vmx3_flag='0')`. |
| **Outputs** | A delivered voicemail; `{ status: complete, result: success }`. |
| **Position** | Step 4 — the hub that fans out to all `sub_*` helpers + the presigner. |

### `vmx3_presigner.py`
| | |
|---|---|
| **Runtime** | Python 3.13 |
| **Trigger** | Invoked **as a Lambda by the packager** (`RequestResponse`). |
| **Purpose** | Generate a long-lived **S3 presigned URL** for the recording, used in the task/email so the recipient can play it back. |
| **Inputs** | `{ recording_bucket, recording_key, vmx3_mode }`. Env: `aws_region`, `secrets_key_id`, `tasks_url_expire`, `email_url_expire`. |
| **Logic** | Retrieves dedicated IAM keys from **AWS Secrets Manager** (`vmx_iam_key_id`/`vmx_iam_key_secret`); builds an S3 client with **SigV4** scoped to those keys; expiry = `tasks_url_expire` or `email_url_expire` (in **days** × 86400), default 7 days; `generate_presigned_url('get_object', …)`. |
| **Outputs** | `{ result: success, presigned_url: <url> }`. |
| **Position** | Step 5 — playback URL for task/email delivery. |

### `vmx3_guided_flow_presigner.py`
| | |
|---|---|
| **Runtime** | Python 3.13 |
| **Trigger** | Invoked from the **Guided Task agent flow** (Connect invocation). |
| **Purpose** | Generate the presigned recording URL on demand for the guided-flow **media player view** (so the URL is fresh, short-lived). |
| **Inputs** | `event['Details']['ContactData']['Attributes']['vmx3_recording_key']`. Env: `aws_region`, `s3_recordings_bucket`. |
| **Logic** | Builds a SigV4 S3 client using the **function's execution role** (no Secrets Manager keys, unlike `vmx3_presigner`); `generate_presigned_url('get_object', ExpiresIn=600)` (10 minutes). |
| **Outputs** | `{ result: success, vmx3_presigned_url: <url> }` (or an `ERROR` result). |
| **Position** | Step 5 — playback URL for `guided_task` delivery. See the regional media-player rendering issue and full fix in `troubleshooting.md`. |

### `vmx3_ses_template_tool.py`  ← utility, not in the runtime path
| | |
|---|---|
| **Runtime** | Python 3.13 |
| **Trigger** | Direct invoke / CloudFormation-style at deploy/admin time. |
| **Purpose** | Manage the **Amazon SES (SESv2)** email templates used to format voicemail emails. |
| **Inputs** | `{ mode: create\|get\|update\|delete, template_name, template_subject, template_text, template_html }`. |
| **Logic** | Switches on `mode`: `create_email_template` / `get_email_template` / `update_email_template` / `delete_email_template` via the `sesv2` client. |
| **Outputs** | Status string (or the template content for `get`). |
| **Position** | Deploy/admin — provisions templates that `sub_ses_email` uses at runtime. |

---

## Modular helpers (`sub_*.py`) — imported and called by the packager

All `sub_` modules run **inside the packager Lambda's process** (not separate Lambdas).
The packager `import`s them and calls their named entry function, threading a shared
`function_payload` dict through each step.

### `sub_key_data_extraction.py` — `key_data_extraction(event)`
| | |
|---|---|
| **Called when** | Always (packager step 2). |
| **Purpose** | Build the core data from the S3 event: derive bucket/key names, transcript key, recording key, and load the recording object's **tags** (language, queue ARN, genai flag), seeding `function_data` and the working payload. |
| **Output** | Payload with `function_data` (`contact_id`, `instance_id`, `recording_bucket`, `recording_key`, …) and base attributes. |

### `sub_process_transcript.py` — `process_transcript(function_payload)`
| | |
|---|---|
| **Called when** | Always (packager step 3). |
| **Purpose** | Load the Transcribe output from S3 and flatten it into clean transcript text (`vmx3_transcript_contents`); gracefully handles the error-handler placeholder. **Calls `sub_genai_summary` when enabled.** |
| **Output** | Processed transcript text merged into `vmx_data`. |

### `sub_genai_summary.py` — `genai_summarizer(function_payload)`
| | |
|---|---|
| **Called when** | `vmx3_genai_summary == 'true'` (invoked from within `sub_process_transcript`). |
| **Purpose** | Send the transcript (`vmx3_transcript_contents`) to **Amazon Bedrock** to produce a concise voicemail summary. |
| **Output** | GenAI summary text for inclusion in the task/email. |

### `sub_build_data_payload.py` — `build_data_payload(function_payload)`
| | |
|---|---|
| **Called when** | Always (packager step 4). |
| **Purpose** | Determine the **VMX delivery mode** (`task` / `email` / `guided_task`) and assemble the final delivery payload (attributes + transcript + summary + recording reference + metadata). |
| **Output** | `vmx3_mode` + consolidated delivery payload merged into `vmx_data`. |

### `sub_connect_task.py` — `vmx3_to_connect_task(function_payload)`
| | |
|---|---|
| **Called when** | `vmx3_mode == 'task'`. |
| **Purpose** | Create an **Amazon Connect task** (via the `connect` client) routed to the target queue/agent, embedding transcript, summary, and the presigned recording link. |
| **Output** | `{ result: success/... }`. |

### `sub_connect_guided_task.py` — `vmx3_to_connect_guided_task(function_payload)`
| | |
|---|---|
| **Called when** | `vmx3_mode == 'guided_task'`. |
| **Purpose** | Create a Connect task tied to the **step-by-step guide (Guided Task agent flow)** for a guided playback/disposition experience; stores `vmx3_recording_key` so the guided flow can later call `vmx3_guided_flow_presigner` for a fresh URL. |
| **Output** | `{ result: success/... }`. |

### `sub_ses_email.py` — `vmx3_to_ses_email(function_payload)`
| | |
|---|---|
| **Called when** | `vmx3_mode == 'email'`. |
| **Purpose** | Send the voicemail notification email via **Amazon SES** using the configured template (managed by `vmx3_ses_template_tool`), populating transcript, summary, and the presigned recording link. Supports agent emails (from the User field) and queue emails (from the `vmx3_queue_email` queue tag). |
| **Output** | `{ result: success/... }`. |

---

## Quick reference: trigger → function

| Trigger type | Function(s) |
|---|---|
| Contact flow (Invoke Lambda) | `vmx3_voicemail_timestamper` |
| Guided Task agent flow (Invoke Lambda) | `vmx3_guided_flow_presigner` |
| Kinesis Data Stream (CTR records) | `vmx3_recording_processor` |
| EventBridge S3 PutObject (recordings bucket) | `vmx3_transcriber` |
| EventBridge S3 PutObject (transcripts bucket) | `vmx3_packager` |
| EventBridge (Transcribe FAILED) | `vmx3_transcription_error_handler` |
| Lambda invoke by packager | `vmx3_presigner` |
| Deploy / admin (custom resource) | `vmx3_ses_template_tool` |
| Imported & called in-process by packager | all `sub_*` modules |
