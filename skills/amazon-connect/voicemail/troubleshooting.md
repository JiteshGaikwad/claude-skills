# Voicemail Express V3 (VMX3) — Troubleshooting, Support & Changelog

This reference covers diagnosing common problems with Voicemail Express 3 (VMX3), the
agent-workspace media player fix for affected regions, how to get support, and the full
version changelog.

---

## Table of Contents

- [Troubleshooting](#troubleshooting)
  - [General Troubleshooting Tips](#general-troubleshooting-tips)
  - [The VMX3 Event Chain](#the-vmx3-event-chain)
  - [General Issues](#general-issues)
  - [Email Delivery Issues](#email-delivery-issues)
  - [Removal Issues](#removal-issues)
  - [Upgrade Issues](#upgrade-issues)
- [Fixing the Media Player in the Agent Workspace](#fixing-the-media-player-in-the-agent-workspace)
- [Getting Support](#getting-support)
- [Changelog](#changelog)

---

## Troubleshooting

Below are common problems customers have encountered, with the appropriate resolution
for each.

### General Troubleshooting Tips

- **Enable DEBUG logging.** If you are experiencing issues, set the logging level of the
  Lambda functions to `DEBUG`. You can change this setting by going to the four core
  functions — `VMXRecordingProcessor`, `VMXTranscriber`, `VMXPackager`, and
  `VMXPresigner` — and changing the application logging level to `DEBUG`. This will
  provide the most detail. Refer to the
  [Configuring advanced logging controls for Lambda functions](https://docs.aws.amazon.com/lambda/latest/dg/monitoring-cloudwatchlogs-advanced.html)
  section of the AWS Lambda guide for more details.

- **Know the event flow.** Troubleshooting is far easier when you understand the order in
  which artifacts are produced. The flow of events, at a high level, is:

  - CTR is emitted
  - `VMXRecordingProcessor` function
  - `VMXTranscriber` function
  - `VMXPackager` function
  - `VMXPresigner` function (called by `VMXPackager`)

- **Start at the beginning of the chain.** Begin troubleshooting at the
  `VMXRecordingProcessor` function. Check the CloudWatch logs to see if the function
  succeeded and that the file was created and placed in S3.

- **Walk the chain in order.** Once you validate that the file was created, look to see
  if there were any errors in the `VMXTranscriber` function. Depending on how closely
  you are monitoring progression, you can also look in the Amazon Transcribe console to
  see if the transcription process is still running, completed, or returned an error.

### The VMX3 Event Chain

```
CTR emitted
   └─► VMXRecordingProcessor   → writes recording to S3
          └─► VMXTranscriber    → writes transcript to S3
                 └─► VMXPackager → packages & delivers the voicemail
                        └─► VMXPresigner (called by VMXPackager) → presigned playback URLs
```

When a voicemail does not arrive, the failure is in the first function in this chain
that stops producing the expected output. Each function has its own CloudWatch log
group, so inspect them in this order.

---

### General Issues

#### I am not getting any voicemails

Make sure that you are setting the **`vmx3_flag`** value to `1` in your contact flows.
This must be set in order for the recording-processor function to know that it should
process this record as a voicemail.

> Note: the recording-processor function is named `VMXRecordingProcessor` in current
> versions; in earlier (KVS-based) versions this function was named `VMXKVStoS3`, and the
> `vmx3_flag` requirement applies in both cases.

---

### Email Delivery Issues

#### I see no errors, but email is not being delivered

1. Validate that the email addresses or domain have been validated in Amazon SES.
2. For agent voicemails, validate that the agent ID is their email address.
3. For queue voicemails, validate that you have added the **`vmx3_queue_email`** tag to
   the queue and populated it with a valid email address.
4. If you are using a custom template, make sure all variables referenced in the
   template exist as contact attributes on the call.

---

### Removal Issues

#### The VMXContactFlowStack fails to delete, causing the parent stack to fail as well

Make sure that you remove the test flow from any phone numbers in your Amazon Connect
instance. If it is associated to a phone number, the delete will fail. Once you make
that change, re-try the delete and it should now succeed.

#### The VMXCoreStack fails to delete, causing the parent stack to fail as well

If there are any files in the S3 buckets created by the stack, the delete will fail
intentionally. You must decide what to do with your files. You can opt to delete them,
move them, or re-run the delete, choosing the option to keep the bucket resources. If you
delete them or move them, re-run the delete and it should succeed.

---

### Upgrade Issues

#### The upgrade succeeded but voicemails are not being delivered. The `VMXRecordingProcessor` function fails at step 1

When upgrading from previous versions, make sure to complete the CloudFormation template
parameters. In older versions, access to the Amazon Connect recording store was not
necessary. That was added in **2025.09.12**. If you did not include the recording bucket
ARN in the template, the solution will fail. Re-run the update, making sure to complete
the parameters.

---

## Fixing the Media Player in the Agent Workspace

In some regions, the rendering of the view is incorrectly formatting the media player
URL. To address this, make the following changes.

1. In the Amazon Connect UI, open the **VMX3_Guided_Task_Agent_Flow_%INSTANCE%** flow.

2. Between the **Generate Presigned URL** and the **Show VMX Guided Task view** blocks,
   add a **Set contact attributes** block.

3. In the new block, set the following attributes as **FLOW ATTRIBUTES**:

   - key: `media_player`, value (Set manually):

     ```json
     {"TemplateString":"<div><audio controls controlsList='nodownload' preload='auto' src='$.External.vmx3_presigned_url' style='width: 100%;'></audio><p>&nbsp;</p></div>"}
     ```

   - key: `customer_data`, value (Set manually):

     ```json
     {"Items":[{"Value":"$.Attributes.vmx3_source_queue"},{"Value":"$.Attributes.vmx3_target"},{"Value":"$.Attributes.vmx3_callback_number"},{"Value":"$.Attributes.vmx3_timestamp"}]}
     ```

   - key: `genai_summary`, value (Set manually):

     ```json
     {"TemplateString":"<h2><b>Generative AI Summary</b></h2>  <p>$.Attributes.vmx3_genai_summary</p>"}
     ```

   - key: `vm_transcript`, value (Set manually):

     ```json
     {"TemplateString":"<div>$.Attributes.vmx3_transcript</div>"}
     ```

4. **Save** the block.

5. Connect the **Success** and **Error** branches of the **Generate Presigned URL** block
   to the input of the new **Set contact attributes** block.

6. Connect the exit paths of the **Set contact attributes** block to the
   **Show VMX Guided Task view** block.

7. Modify the **Show VMX Guided Task view** block so each view input is bound dynamically
   to the matching flow attribute:

   | View input | Setting | Namespace | Key |
   | --- | --- | --- | --- |
   | `customer_data` | Set Dynamically | Flow | `customer_data` |
   | `GenAISummary` | Set Dynamically | Flow | `genai_summary` |
   | `media_player` | Set Dynamically | Flow | `media_player` |
   | `transcript_box` | Set Dynamically | Flow | `vm_transcript` |

8. **Save** the block.

9. **Save** the flow.

10. **Publish** the flow.

Once this is complete, run some test voicemails through the guided flow experience to
validate functionality.

---

## Getting Support

Voicemail Express 3 is a solution designed, developed, and maintained by Amazon Connect
Solutions Architects. While great effort has been spent to validate its use and
effectiveness, it is not an AWS service or product, is not aligned to any AWS Service
Team, and support is on a best-effort basis with the resources available. There are,
however, three primary pathways to support:

1. **GitHub Issues** — Use the
   [Issues](https://github.com/amazon-connect/voicemail-express-amazon-connect/issues)
   feature of the repository for defects, errors, or deployment issues. Any issues posted
   here are routed to the team that maintains Voicemail Express 3, which has the most
   direct hands-on knowledge. Additionally, previously raised issues are archived and
   explained for future reference to help other users in their troubleshooting efforts.
   No AWS support plan is required for this.

2. **GitHub Discussions** — Use the
   [Discussions](https://github.com/amazon-connect/voicemail-express-amazon-connect/discussions)
   feature of the repository for feature requests, general use questions, modification
   guidance, etc. As with the Issues section, no AWS support plan is required for this.

3. **AWS Support** — You can open an [AWS Support](https://console.aws.amazon.com/support)
   case. You will need to ensure that the case is opened against this solution's
   repository to be properly routed to this team. You must have an AWS support plan to
   open a case.

---

## Changelog

Version entries are listed newest first.

### 2025.09.13

- Fixed an issue with the `sub_build_data_payload` code that prevented agent emails from
  being correctly set.
- Fixed an issue with the `sub_ses_email` code that prevented queue emails from being set
  correctly.
- Fixed an issue with the default agent email template that was causing agent-specific
  emails to fail.
- Updated email documentation with correct example code for templates.
- Added GovCloud version to repo.

### 2025.09.12

- Changed the VMX3 process to use Amazon Connect's IVR recording feature instead of KVS,
  including the ability to specifically extract the voicemail from full IVR recording use
  cases.
- Added a Generative AI summary option.
- Switched from agent personal queues to preferred agent routing for better control over
  tasks when agents are unavailable.
- Added an in-queue voicemail option and examples.
- Configuration now supports GovCloud.
- Added support for bucket versioning.
- Validated and documented operation with customer-managed KMS keys.
- Validated voicemails up to 25 minutes long.
- Performance tested up to 2000 voicemails per hour.
- Added documentation for supporting long voicemails.
- Added documentation for supporting self-service voicemails.
- Truncated voicemail transcripts to meet task reference limits.
- Additional voicemail filtering to support high-volume environments.
- Updated recording and transcript bucket folder structure to make files easier to find.
- Updated all functions to Python 3.13.
- Updated logging configuration on all functions.
- Updated HTML code for email delivery.
- Updated the agent view when using the guided tasks option.
- Added tagging to all resources.
- Added tagging to contacts.
- Fixed the MIME type to improve browser playback compatibility.

### 2024.09.01

- Added a guided task option, visualizing a player and obfuscating the short-lived URL.
- IAM roles and policies by function instead of one central role/policy.
- Updated the email process to allow users to select the user field which contains the
  agent email.
- Fixed an issue with email templates that prevented dynamic template selection.

### 2024.08.01

- Rewrote the KVStoS3 function in Python, completing conversion of the code to
  Python 3.12.
- Switched to the `GetMediaForFragmentList` API for Kinesis Video Streams Archived Media
  to extract the audio from KVS.
- Adopted native Lambda logging in all functions and improved logging.
- Improved error handling in all functions.
- Removed the Node common layer.
- Load-tested thousands of voicemails.
- Improved the Troubleshooting section.
- Added the Getting Support section.
- Added the Changelog.

### 2024.07.03

- Updated the KVS-to-S3 function to reduce error conditions in environments with heavy
  KVS use or long retention windows.
- Updated all Lambda functions to use the native Lambda logging configuration.
- Changed code/template buckets to further separate from VMX2.

### 2024.07.02

- Improved CR filtering to reduce non-VMX records.
- Changed trigger configuration to reduce errors blocking subsequent records.

### 2024.07.01

- Added support for voicemail delivery via Amazon Simple Email Service (SES).

### 2024.06.01

- Improved performance of the KVStoS3 function.
- Kinesis Data Stream filtering for records to reduce Lambda invocations.
- Added voicemail date/time info to the task.
- Updated flow naming for consistency.
- Additional load testing.

### 2024.05.01

- Resolved an edge case that could allow a voicemail task to be duplicated.
- Resolved an issue where corrupted or invalid audio files would cause the transcription
  to fail, resulting in a lost voicemail.
- Added new messaging to identify KVS startup issues on first run, and to hopefully
  resolve them.
- Upgraded all Python functions to 3.12.
- Reduced the layer size for the Python layer.

### 2024.03.20

- Simplified the deployment process.
- Removed Salesforce-centric deployment options.
- All voicemails are delivered as Amazon Connect tasks. The option to add other delivery
  modes will come in a future release.
- An Amazon Connect flow module named **VMX3VoicemailCoreModule** is provided. This
  provides a standard voicemail experience, sets all required attributes, and records the
  voicemail. You can use this module in any standard Amazon Connect inbound contact flow
  to provide the voicemail experience without needing to create a custom flow.
- The VMX3TestFlow has been modified to use the **VMX3VoicemailCoreModule**.
- Modified the transcribe job name to eliminate conflicts.
