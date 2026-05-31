# Voicemail Express (VMX3) Deployment Reference

Complete, click-by-click deployment reference for **Voicemail Express V3 (VMX3)** — the open-source voicemail add-on for Amazon Connect. Covers prerequisites, standard (commercial-region) installation, the CloudFormation parameter reference, GovCloud deployment, upgrades, and uninstall.

Current template version referenced throughout: **2025.09.13**. Always confirm the latest version at the end of the project README before deploying or upgrading.

Account IDs in this document use the placeholder `111122223333`.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Standard Installation (all regions except GovCloud)](#2-standard-installation-all-regions-except-govcloud)
3. [CloudFormation Parameter Reference](#3-cloudformation-parameter-reference)
4. [Assign a Test Number and Validate](#4-assign-a-test-number-and-validate)
5. [GovCloud Deployment](#5-govcloud-deployment)
6. [Upgrade](#6-upgrade)
7. [Uninstall](#7-uninstall)

---

## 1. Prerequisites

These **must be completed** before deploying and using Voicemail Express. They ensure your Amazon Connect instance is ready.

### 1.1 Core prerequisites (all deployments)

- **Contact records streaming via Amazon Kinesis Data Stream.** Configure your Amazon Connect instance to stream Contact Trace Records (CTRs) to a **Kinesis Data Stream**.
  - **Amazon Kinesis Data Firehose is NOT supported.** The solution is designed to receive CTRs via Kinesis Data Streams only. It WILL NOT work with a Kinesis Firehose.
  - Reference: AWS docs "Set up data streaming for your instance" (data-streaming).
- **Amazon Bedrock model access** (only required if you want generative AI summaries of voicemail):
  - **Amazon Nova Lite** enabled in Amazon Bedrock for commercial regions.
  - **Claude Sonnet 3.5** enabled in Amazon Bedrock for **GovCloud**.
  - Enable via the Bedrock console "Model access" page (model-access-modify).
  - **Note:** Model access for all models is allowed by default on AWS accounts beginning **October 2025**. On older/existing accounts you may still need to request access manually.

### 1.2 Delivery-mode-specific prerequisites

VMX3 supports multiple delivery modes. Complete only the prerequisites for the mode(s) you intend to enable.

#### Amazon Connect Task or Guided Task delivery

For Task or Guided Task delivery, you simply need the following configured on your Amazon Connect instance:

- **Tasks enabled in Channel Settings** of the relevant **routing profiles**, and for the appropriate queues.
- **Agents assigned to those routing profiles.**

Guided Task delivery additionally requires the agent to use the **Agent Workspace** or a **custom CCP/UI with agent guides enabled** (the voicemail is presented through an agent guide with a media-player experience).

#### Email delivery (Amazon SES)

Email delivery uses **Amazon SES**. If SES is not already configured in your account, you must perform minimal configuration to validate email addresses.

There are two types of email addresses in this solution: **`from`** and **`to`**.

- **`from` address** — indicates where the email is from, e.g. `contact_center@yourcompany.com`. You can dynamically assign the from address or simply use a default. This decision can be made on a call-by-call basis.
- **`to` address** — where the email is delivered:
  - For **agent** voicemails, this should be the agent's email address, e.g. `joe.smith@yourcompany.com`.
  - For **queue** voicemails, this would be a group address or distribution list, e.g. `customer_support@yourcompany.com`.
- **Verify identities/domains in SES.** For SES to send using your addresses, you must validate ownership of either the individual addresses or the entire email domain. Follow the AWS docs for "Creating and verifying identities in Amazon SES" (creating-identities). **You must use verified identities/domains, or the emails will not be delivered.**

##### Voicemail-to-agent emails

The solution is designed to use the **agent's email address configured in Amazon Connect** as the delivery (`to`) address. When deploying, you select which agent field stores that address (see the `VMAgentAddressKey` parameter). If you wish to use a different email address, the packager function would need to be modified to account for that.

##### Voicemail-to-queue emails

For queues, the solution extracts the email address from the **queue tags**, allowing you to set the address per queue. If no address is specified, the solution uses the provided default (`VMToEmailDefaultTo`).

##### Add the email-address tag to a queue

1. Login to the Amazon Connect administration interface.
2. Select **Routing**, then choose **Queues**.
3. Select the Queue that you wish to modify.
4. In the **Tags** section, create a new tag with the **key** `vmx3_queue_email` and the **value** set to the email address that you want voicemails delivered to for this queue.
5. Save the queue.

> Once prerequisites are complete, proceed to the standard installation for all regions except GovCloud, or to the GovCloud deployment section for GovCloud.

---

## 2. Standard Installation (all regions except GovCloud)

Voicemail Express is deployed via **AWS CloudFormation**. The template builds all AWS resources required to make Voicemail Express work, including several nested stacks.

> **GovCloud:** Do not use this procedure. Use [Section 5 — GovCloud Deployment](#5-govcloud-deployment).

### 2.1 Gather required information (all deployments)

Before launching the template, collect:

- **ARN for the Amazon Kinesis Data Stream** used for streaming your CTRs (from the Amazon Kinesis Data Streams console).
  - Reminder: Kinesis **Data Streams** only, **not** Firehose.
- **Amazon Connect Instance Alias** (from the Amazon Connect console).
- **Amazon Connect Instance ARN** (from the Amazon Connect console).
- **Amazon Connect Call Recordings bucket ARN** (Connect console → Data storage → Call recordings).
- **Default agent ID** to use for testing.
- **Default queue ARN** to use for testing.

### 2.2 Gather required information (email deployments ONLY)

- **Default FROM email address** — the default address used to send emails FROM if no other address is configured or provided.
- **Default TO email address** — the default address used to send emails TO if no other address is configured or provided.

### 2.3 Deploy the CloudFormation template

1. Open a new browser tab and log into the **AWS console**. Set your **region to match the region where Amazon Connect is deployed**, then return.
2. Select the **launch link that matches your region** to open the CloudFormation "create stack" wizard pre-populated with the template. The stack name defaults to `VMX3`. Supported regions and their source buckets follow the pattern `https://vmx-source-<region>.s3.<region>.amazonaws.com/vmx3/2025.09.13/cloudformation/vmx3.yaml`:
   - **us-east-1**
   - **us-west-2**
   - **af-south-1**
   - **ap-southeast-1**
   - **ap-southeast-2**
   - **ap-northeast-1**
   - **ap-northeast-2**
   - **ca-central-1**
   - **eu-central-1**
   - **eu-west-2**
3. Verify that the **Amazon S3 URL** is for the same region you are deploying to, then choose **Next**.
4. Update the **stack name** to include your instance alias, for example `VMX3-MyInstanceName`.
5. **Complete the parameters** using the information you gathered (see [Section 3](#3-cloudformation-parameter-reference) for every parameter).
6. Once the parameters are complete, choose **Next**.
7. Scroll to the bottom and select the boxes to **acknowledge that IAM resources will be created**.
8. Scroll to the bottom and select **Next**.
9. Select **Submit**.
10. The deployment takes **3–5 minutes**. During this time, multiple **nested stacks** are deployed. Once the **main stack shows `CREATE_COMPLETE`**, you are ready to proceed.

> The deployed solution creates several nested stacks: a core stack (`VMXCoreStack` — S3 buckets, etc.), a contact-flow stack (`VMXContactFlowStack`), an SES setup stack (`VMXSESSetupStack`, when email is enabled), a Lambda functions stack (`VMXLambdaStack`), a policy stack (`VMXPolicyStack`), and a triggers stack (`VMXTriggersStack`).

After the stack completes, continue to [Section 4 — Assign a Test Number and Validate](#4-assign-a-test-number-and-validate).

---

## 3. CloudFormation Parameter Reference

These are the **exact parameter keys** from `vmx3.yaml` (template version 2025.09.13), grouped as they appear in the CloudFormation console. The console shows friendly labels; the keys below are what the template uses internally.

### Group 1 — Environment Configuration

| Parameter (key) | Console label | Default | Allowed values | Description |
|---|---|---|---|---|
| `AWSRegion` | Which AWS region is your Amazon Connect Instance deployed to? | `us-east-1` | `us-east-1`, `us-west-2`, `af-south-1`, `ap-southeast-1`, `ap-southeast-2`, `ap-northeast-1`, `ap-northeast-2`, `ca-central-1`, `eu-central-1`, `eu-west-2`, `us-gov-west-1` | All resources must reside in the same region. Validate which region your Amazon Connect instance is in and select it. Run the template from this region as well. |
| `ConnectInstanceAlias` | What is your Amazon Connect instance alias? | `REPLACEME` | (free text) | Instance alias for your Amazon Connect instance (shown in the list of instances in the Connect console). **Use lowercase letters ONLY.** |
| `ConnectInstanceARN` | What is your Amazon Connect instance ARN? | `Looks like - arn:aws:connect:REGION:ACCOUNT:instance/REPLACEME` | (free text) | Instance ARN for your Amazon Connect instance. |
| `ConnectRecordingsBucketArn` | What is the ARN for the S3 bucket used to store Amazon Connect recordings? | `Looks like - arn:aws:s3:::YOURBUCKETNAME` | (free text) | Full ARN for the S3 bucket. Find the bucket name in the Connect console at **Data Storage > Call recordings**. |
| `ConnectCTRStreamARN` | What is the ARN for the Kinesis Data Stream configured for Contact Record streaming? | `Looks like - arn:aws:kinesis:REGION:ACCOUNT:stream/REPLACEME` | (free text) | ARN for the Kinesis Data Stream that receives your contact records. **Kinesis Data Streams only — not Firehose.** |
| `LambdaLoggingLevel` | For the included AWS Lambda functions, what should the default logging level be? | `INFO` | `ERROR`, `WARN`, `INFO`, `DEBUG` | Default logging level for the included Lambda functions. Can be changed later. |

### Group 2 — Voicemail Express AWS Delivery Modes

| Parameter (key) | Console label | Default | Allowed values | Description |
|---|---|---|---|---|
| `EnableVMToConnectGuidedTask` | Enable Voicemail delivery as an Amazon Connect Guided Task? (DEFAULT MODE) | `yes` | `yes`, `no` | Deliver voicemails as **Guided Tasks**. Like a normal Task, but presented via an agent guide with a media-player experience. Requires Agent Workspace or a custom UI that incorporates agent guides. |
| `EnableVMToConnectTask` | Enable Voicemail delivery as an Amazon Connect Task? | `no` | `yes`, `no` | Deliver voicemails as **Amazon Connect Tasks**. Works with Agent Workspace, CCP, or streams. The incoming task contains the transcript and a presigned URL to the recording. |
| `EnableVMToEmail` | Enable Voicemail delivery as an Email using Amazon SES? | `no` | `yes`, `no` | Deliver voicemails as external **emails**. These are **not tracked or routed via Amazon Connect**. |
| `VMDefaultMode` | What is your default delivery mode? | `AmazonConnectGuidedTask` | `AmazonConnectGuidedTask`, `AmazonConnectTask`, `AmazonSimpleEmailService` | If no other voicemail delivery model can be identified, which model should be used. |

### Group 3 — AI Option and Recording Retention Settings

| Parameter (key) | Console label | Default | Allowed values | Description |
|---|---|---|---|---|
| `EnableGenAISummary` | Do you want to generate a summary of the voicemail using generative AI? | `no` | `yes`, `no` | Use generative AI to create a summary of the recording and include it in the delivered voicemail. Can be overridden on a case-by-case basis. (Requires the relevant Bedrock model access from prerequisites.) |
| `EnableBucketVersioning` | Enable S3 Versioning for the recordings and transcription buckets? | `no` | `yes`, `no` | Enable versioning on both buckets. Versioning helps recover objects from accidental deletion or overwrite. |
| `RecordingsExpireInXDays` | How long should we keep voicemail recordings before deleting or archiving? (# of days) | `7` | (free text — number of days) | Number of days recordings are saved before they are lifecycled. When the limit is hit, the recording is lifecycled per `ExpiredRecordingBehavior`. |
| `ExpiredRecordingBehavior` | When the recording reaches its lifecycle, what action should be taken? | `delete` | `delete`, `keep`, `glacier` | On expiry, delete the recording, keep it, or move it to Glacier. |

### Group 4 — Amazon Connect Guided Task Parameters (only for Guided Task delivery)

| Parameter (key) | Console label | Default | Allowed values | Description |
|---|---|---|---|---|
| `GuidedTaskRecordingLinksExpireInXMinutes` | How long should the presigned recording URL be valid for? (# of minutes) | `2` | `2`, `5`, `10` | Minutes recordings are accessible via the presigned link. Limited by the maximum time a presigned URL can be active. |

### Group 5 — Amazon Connect Task Parameters (only for standard Task delivery)

| Parameter (key) | Console label | Default | Allowed values | Description |
|---|---|---|---|---|
| `TaskRecordingLinksExpireInXDays` | How long should the presigned recording URL be valid for? (# of days) | `1` | `1`, `2`, `3`, `4`, `5`, `6`, `7` | Number of days the presigned recording URL is valid. |

### Group 6 — Amazon Connect Email Parameters (only for email delivery)

| Parameter (key) | Console label | Default | Allowed values | Description |
|---|---|---|---|---|
| `VMToEmailDefaultFrom` | Fallback FROM email for voicemails if no address is provided? | `voicemail_from@test.com` | (free text) | Email address used to send voicemails when no other FROM address could be identified. Also configured as the **test FROM** address. |
| `VMToEmailDefaultTo` | Fallback TO email for voicemails if no address is provided? | `voicemail_to@test.com` | (free text) | Email address to receive voicemails when no other TO address could be identified. Also configured as the **test TO** address. |
| `VMAgentAddressKey` | Do you store agent email addresses in the Email, Secondary Email, or Username field? | `Email` | `Email`, `SecondaryEmail`, `Username` | Which Connect agent field holds the email address. Typically **SAML** configs use `SecondaryEmail` and **non-SAML** configs use `Email`. If you set the Username value to the agent email address, select `Username`. |
| `EmailRecordingLinksExpireInXDays` | How long should the presigned recording URL be valid for? (# of days) | `1` | `1`, `2`, `3`, `4`, `5`, `6`, `7` | Number of days the presigned recording URL is valid. |

### Group 7 — Voicemail Express Test Parameters

| Parameter (key) | Console label | Default | Allowed values | Description |
|---|---|---|---|---|
| `VMTestAgentId` | Which agent ID will you use for testing? | `Looks like - jdoe or jdoe@test.com` | (free text) | Agent ID used by the voicemail test function. |
| `VMTestQueueARN` | Which queue will you use for testing? | `Looks like - arn:aws:connect:REGION:ACCOUNT:instance/INSTANCEID/queue/QUEUEID` | (free text) | Queue ARN for the test function. Find it in the Connect UI: **Routing > Queue**, choose the queue, then expand **Show additional queue information**. |

### Group 7 (Advanced) — Use only in customized deployments

> The template labels this group "7. Advanced Settings" (the numbering "7" repeats in the template metadata).

| Parameter (key) | Console label | Default | Allowed values | Description |
|---|---|---|---|---|
| `EXPDevBucketPrefix` | (ADVANCED USE ONLY) What is the bucket prefix for your custom code S3 bucket? | `''` (empty) | (free text) | Not required for normal deployments; used for development. **In GovCloud this field carries the deployment-bucket prefix** (see Section 5). |
| `EXPTemplateVersion` | (ADVANCED USE ONLY) What version do you wish to deploy? | `2025.09.13` | (free text) | Template version. **ONLY change if you have built a custom version** — or when upgrading, to point at a newer published version. |

> **Total: 24 deployment parameters** across the eight metadata groups (6 environment, 4 delivery modes, 4 AI/retention, 1 Guided Task, 1 Task, 4 email, 2 test, 2 advanced).

---

## 4. Assign a Test Number and Validate

### 4.1 Assign a test number

1. Login to the Amazon Connect administration interface.
2. Select **Channels** from the navigation menu, then choose **Phone numbers**.
3. Either select an existing number, or claim a new number.
4. Set the contact flow for the number to **`VMX3-AWSTestFlow-YOURINSTANCE`**.
5. Select **Save**.

### 4.2 Test flow — DTMF menu map

The test line (`VMX3-AWSTestFlow-...`) presents a first menu to choose the **delivery mode**, then a second menu to choose the **target** (agent vs queue), then optionally enables the **GenAI summary**. The first-menu DTMF differs per test scenario:

| Scenario | First menu DTMF | Second menu |
|---|---|---|
| Guided Task delivery | Press **2** | **1** = agent, **2** = queue |
| Standard Task delivery | Press **1** | **1** = agent, **2** = queue |
| Email (SES) delivery | Press **3** | **1** = agent, **2** = queue |
| In-Queue Voicemail (Task) | Press **4** | **1** = leave a voicemail |

### 4.3 Test: Guided Task delivery

If you deployed with the Guided Tasks delivery option:

1. **Dial** the phone number configured for the Voicemail Test Line.
2. At the first menu, **press 2** to select Guided Task delivery.
3. At the next menu, **press 1** to leave a voicemail for an agent or **press 2** to leave a voicemail for a queue.
4. Select the appropriate option to **enable the generative AI summary**, if desired.
5. When you hear the tone, **record your voicemail**. Hang up any time after recording.
6. After recording, **wait approximately 2 minutes**.
7. Log the appropriate agent in **to the agent workspace or a custom CCP with guides enabled** and put them into the **Available** state. The Guided Task should arrive shortly.

### 4.4 Test: standard Task delivery

If you deployed with the Tasks delivery option:

1. **Dial** the test line.
2. At the first menu, **press 1** to select Task delivery.
3. At the next menu, **press 1** (agent) or **press 2** (queue).
4. Enable the GenAI summary if desired.
5. Record your voicemail at the tone; hang up any time after recording.
6. **Wait approximately 2 minutes.**
7. Log the agent in and set them to **Available**. The Task should arrive shortly.

### 4.5 Test: Email delivery via SES

If you deployed with the email delivery option:

1. **Dial** the test line.
2. At the first menu, **press 3** to select email delivery.
3. At the next menu, **press 1** (agent) or **press 2** (queue).
4. Enable the GenAI summary if desired.
5. Record your voicemail at the tone; hang up any time after recording.
6. **Wait approximately 2 minutes.**
7. Access the appropriate email box to verify delivery of the voicemail.

### 4.6 Test: In-Queue Voicemail (Task)

If you deployed with the Tasks delivery option, validate the in-queue experience:

1. **Dial** the test line.
2. At the first menu, **press 4** to select the in-queue Task scenario.
3. At the next menu, **press 1** to leave a voicemail.
4. Enable the GenAI summary if desired.
5. Record your voicemail at the tone; hang up any time after recording.
6. **Wait approximately 2 minutes.**
7. Log the agent in and set them to **Available**. The Task should arrive shortly.

**Voicemail validation is complete.**

---

## 5. GovCloud Deployment

GovCloud deployments follow the same overall pattern as standard deployments. The **primary difference** is that the deployment package (CloudFormation templates + Lambda ZIPs) must be hosted in an **S3 bucket within your own account** to conform with GovCloud's more stringent security requirements. Outside of that, the process is the same.

> **Model difference:** GovCloud GenAI summaries use **Claude Sonnet 3.5** (commercial regions use Amazon Nova Lite). Ensure that model access is enabled in Bedrock (see prerequisites).

- First complete the [installation prerequisites](#1-prerequisites).

### 5.1 Gather required information (all deployments)

Same as standard:

- ARN for the Amazon Kinesis Data Stream used for streaming CTRs (Kinesis Data Streams only, not Firehose).
- Amazon Connect Instance Alias.
- Amazon Connect Instance ARN.
- Amazon Connect Call Recordings bucket ARN.
- Default agent ID to use for testing.
- Default queue ARN to use for testing.

### 5.2 Gather required information (email deployments ONLY)

- Default FROM email address.
- Default TO email address.

### 5.3 Create an S3 bucket to host the templates and code

During deployment, several Lambda functions are deployed; their code lives in ZIP files in S3. For typical deployments those ZIPs are in public-facing buckets. In GovCloud, you host the ZIPs and CloudFormation templates within your own account (this also lets you run virus/security scans against the resources if desired). The bucket must be in the **same region** as your Amazon Connect instance.

1. Create a new S3 bucket that follows this naming convention:
   - **`<unique prefix>` + `vmx-source-us-gov-west-1`**
   - **Example:** `mygovcloud-vmx-source-us-gov-west-1` (a documented example uses a different prefix; the pattern is what matters)
   - **Remember your prefix** — you will need it as a parameter (e.g. `mygovcloud-`).
2. Once created, create a new folder named **`vmx3`** in the new bucket.
3. Download the **full VMX deployment package zip file**: `https://vmx-source-us-gov-west-1.s3.us-west-2.amazonaws.com/vmx3_govcloud_version_2025.09.13.zip` (the GovCloud version zip; use the version matching your target).
4. Extract the **top-level contents** of the file and upload the **version folder** into the `vmx3` folder you created in your S3 bucket. The resulting structure resembles the documented S3 object layout (a version-named folder containing the `cloudformation/` templates and the Lambda ZIPs).
5. **Copy the Object URL for the `vmx3.yaml` file** in your bucket — you will paste it into CloudFormation.

> **Critical:** The solution must deploy from resources in **your** environment. **Do not** attempt to deploy to GovCloud from the standard public template — the deployment will fail.

### 5.4 Deploy the solution

1. Open the **AWS Console**.
2. Navigate to **CloudFormation**.
3. Select **Create stack** and choose **With new resources (standard)**.
4. In the **Specify template** section, select **Amazon S3 URL** and paste the **Object URL for the `vmx3.yaml`** file you copied earlier.
5. Update the **stack name** to include your instance alias, e.g. `VMX3-MyInstanceName`.
6. In the **1. Environment Configuration** section, set the **"What is the prefix for your deployment bucket?"** field (the `EXPDevBucketPrefix` parameter) to the bucket prefix you used earlier. For example, if your bucket was `mygovcloud-vmx-source-us-gov-west-1`, enter **`mygovcloud-`**.
7. **Complete the remaining parameters** using the information you gathered (see [Section 3](#3-cloudformation-parameter-reference)). Set `AWSRegion` to `us-gov-west-1`.
8. Once the parameters are complete, choose **Next**.
9. Scroll to the bottom and select **Next**.
10. Scroll to the bottom and select the boxes to **acknowledge that IAM resources will be created**.
11. Select **Submit**.
12. The deployment takes **3–5 minutes**; multiple nested stacks are deployed. Once the **main stack shows `CREATE_COMPLETE`**, you are ready to proceed.

### 5.5 Assign a test number and validate (GovCloud)

Identical to the standard procedure — assign the `VMX3-AWSTestFlow-YOURINSTANCE` flow to a phone number, then run the DTMF test flows. See [Section 4](#4-assign-a-test-number-and-validate).

---

## 6. Upgrade

When new versions of Voicemail Express are released, upgrade using the **Update** feature of AWS CloudFormation.

### 6.1 Considerations before upgrading

1. **The upgrade overwrites any customizations you have made.** Best practice: note your modifications so they can be migrated to the new version. Alternatively, maintain your own versions of the contact flows and Lambda functions so you can deploy without overwriting your modifications.
2. **When you load the new template, it retains your previous settings.** You **must** change the **template version field at the bottom** (`EXPTemplateVersion`) to actually pull the new version.

### 6.2 Upgrade instructions

1. Open the Voicemail Express CloudFormation template (`CloudFormation/vmx3.yaml`) in a new tab. **Right-click / control-click the `raw` button and save the template locally.**
2. Login to the **AWS Console**.
3. Navigate to the **AWS CloudFormation console**.
4. Make sure you are in the **correct region** for your deployment.
5. Select **Stacks**, then choose the Voicemail Express stack you deployed. If you don't recall the name, the description includes **Voicemail Express**.
6. Choose **Update stack**, then choose **Make a direct update**.
7. Select **Replace existing template**, then choose **Upload a template file**.
8. Select **Choose file**.
9. Navigate to the **`vmx3.yaml`** file you downloaded and choose **Open**.
10. Wait a moment for the S3 URL to update, then select **Next**.
11. Find the parameter field labeled **"(ADVANCED USE ONLY) What is version you wish to deploy?"** (`EXPTemplateVersion`).
12. Change that value to the **latest version available** (find it at the end of the project README).
13. There may be **additional new fields and changes**. Validate/update all other fields as well.
    > **Important:** When updating **from a version before 2025.09.12**, you **MUST** provide the bucket that Amazon Connect uses to store recordings (the `ConnectRecordingsBucketArn` parameter). This field was introduced in/around that version and is required going forward.
14. Once the parameters are filled out, choose **Next**.
15. Scroll to the bottom and select **Next**.
16. Scroll to the bottom and select the boxes to **acknowledge that IAM resources will be created**.
17. Select **Submit**.
18. The deployment takes **3–5 minutes**; multiple nested stacks are deployed. Once the **main stack shows `UPDATE_COMPLETE`**, the upgrade is complete.

---

## 7. Uninstall

Removing Voicemail Express is a straightforward process, but the S3 buckets holding recordings/transcripts have a **retain policy** and require manual handling.

### 7.1 Prepare the Amazon Connect instance

1. Log into your Amazon Connect instance as an administrator.
2. Select **Channels**, then choose **Phone numbers**.
3. Make sure that **none of the VMX default flows** are set as the **Contact flow / IVR** for any phone numbers in your instance. The default flows all begin with **`VMX3`**.

> **Caution:** You may have referenced some of these flows from within your other flows. While not strictly required to uninstall, you should validate that these default flows are not referenced elsewhere — otherwise removing them could cause flow failures.

### 7.2 Move your voicemail recordings/transcripts

The CloudFormation retention policy does **not** allow deletion of the S3 buckets holding voicemail recordings and temporary transcripts **if they contain objects**. First either copy the contents elsewhere (to retain copies for archival) or delete the contents.

To determine which S3 buckets are in use:

1. Login to the **AWS Console**.
2. Navigate to the **AWS CloudFormation console**.
3. Make sure you are in the **correct region** for your deployment.
4. Select **Stacks**, then choose the **`VMXCoreStack`** stack.
5. Select the **Resources** tab.
6. In the **Logical ID** column, find both **`VMXS3RecordingsBucket`** and **`VMXS3TranscriptsBucket`**.
7. The link in the corresponding **Physical ID** column takes you to each bucket.
8. **Export or delete the objects** as desired.

### 7.3 Delete the CloudFormation stack

Deleting the stack removes **ALL** AWS resources created, with the conditional exception of the S3 buckets. By default, if the S3 buckets contain any objects, they will not be deleted and their contents remain in place.

1. Login to the **AWS Console**.
2. Navigate to the **AWS CloudFormation console**.
3. Make sure you are in the **correct region** for your deployment.
4. Select **Stacks**, then choose the Voicemail Express stack you deployed. If you don't recall the name, the description includes **Voicemail Express**.
5. Select **Delete**, then confirm by choosing **Delete** in the popup.
6. **IF** the stack deleted completely, you are done.
7. **IF** the stack failed to fully delete, it is most likely due to content remaining in an S3 bucket. In that case, select the **nested stack that failed** (most likely **`VMXCoreStack`**) and choose **Delete**.
8. In the popup, you will likely see the resources that could not be deleted (probably the S3 buckets). **Select both to retain them**, then select **Delete**.
9. Once the nested stack deletes, **reselect the main stack** and select **Delete** again. This time the resources should delete without further issue.

You have successfully removed Voicemail Express.
