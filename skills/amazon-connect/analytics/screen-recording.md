# Agent Screen Recording

Contact Lens screen recording captures the agent's desktop during contacts for quality management, compliance, and coaching. It requires a **client application installed on the agent's device** plus instance + flow configuration — it is **not** browser-only.

---

## What it captures

- Records **all open applications** on the agent's monitor(s), **up to three monitors**.
- Channels: **voice, chat, and task** contacts. (The `Set recording and analytics behavior` block's contact-type table allows the block on Voice/Chat; the screen-recording feature itself supports voice/chat/task.)
- Format: **MP4**, **5 fps** (not configurable), **OpenH264** codec, ~**1.5 MB/minute**. No max duration, no service quota.
- Only the **unredacted** call audio is merged into the video — there is no redacted-audio option, so the **Screen recording - Access** permission also exposes unredacted audio.
- Available only for **Completed** contacts.

---

## Prerequisites

1. **Service-linked role** — instances created before October 2018 must migrate to the Connect SLR first.
2. **Connect Customer Client Application** installed on the agent desktop/VDI (see below). Recordings are captured only from Connect domains in the app's allowlist.
3. Instance-level screen recording enabled in **Data storage** with an S3 bucket (+ optional KMS).
4. A **Set recording and analytics behavior** block with **Screen Recording = On** in each relevant flow.

### System & network requirements

- CPU 2.0 GHz (4 cores/vCPU recommended), 4 GB memory, ~**500 kbps per concurrent recorded contact**.
- OS: 64-bit **Windows 10/11**, or **Chrome OS 140+** (enrolled in a Google Enterprise Domain). Windows multi-session needs client **v2.0.0+**.
- Browsers: Chromium (Chrome, Edge). **Chrome v147 / Edge v147 (April 2026)** enforce Local Network Access restrictions that block the local WebSocket and break recording — deploy the **`LoopbackNetworkAllowedForUrls`** enterprise policy with your CCP domain (e.g. `[*.]my.connect.aws`).
- Local WebSocket port: **5431** (Windows), **25431** (Chrome OS).
- Firewall allowlist: `connect-recording-staging-*.s3.dualstack.<region>.amazonaws.com` (from CCP) and `https://<instance-alias>.my.connect.aws/taps/client/auth` (from the client app).

---

## Enable (3 parts)

**1. Instance level** — Console → instance alias → **Data storage** → **Screen recordings** → **Edit** → **Enable screen recording** → create/select an S3 bucket → optionally **Enable encryption** (KMS). The **same KMS key must be used at the S3 bucket and in the data-storage config** — they can't differ.

**2. Install the client app** (see below).

**3. Flow block** — add a **Set recording and analytics behavior** block after the flow's entry point, set **Screen Recording = On**. (Superseded by **Set recording, analytics and processing behavior** for new flows.)

**Tips:** set a custom attribute (e.g. `screen recording = true`) before the block so supervisors can search for recorded contacts; use a **Distribute by percentage** block to record a sample; use `SuspendContactRecording`/`ResumeContactRecording` to keep sensitive info out.

### Client application install

- **Windows** — latest **v2.0.3** (GovCloud US-West requires v2.0.3+). MSI `Amazon.Connect.Client.Service.Setup.<version>.msi`. Programmatic install via SCCM with the `ALLOWED_CONNECT_DOMAINS` msiexec parameter (comma-separated). Allowlist rules: A-Z a-z 0-9 hyphen period only, no protocol/wildcards, max 500 entries / 256 chars each / 128,000 total. Processes: `Amazon.Connect.Client.Service` (always) + `Amazon.Connect.Client.RecordingSession` (after first recorded contact accepted).
- **Chrome OS** — via Google Enterprise Admin Console: an **Isolated Web App** (managed-config key `allowListedDomain`; grant Direct sockets / Screen recording / Window management) + a **Chrome browser extension** (Force Install). Both auto-update.

---

## Lifecycle

- Starts when the agent **accepts** a recorded contact (the `RecordingSession` process spawns then).
- **Continues during hold** (does not stop).
- If the browser closes before upload completes, the recording publishes the next time the agent logs into CCP.
- S3 location is in the contact record's **`RecordingsInfo.Location`** field.

---

## Review / playback

- **Analytics and optimization → Contact search** → open the contact → **Contact details** → **Recording** section has a video player **synchronized with the voice recording and transcript**.
- Controls: zoom, fit-to-window, full-screen, **picture-in-picture**, download (if permitted). A **Show screen recording** toggle must be on.
- **Playback is not supported on the legacy `awsapps.com` domain** — use `https://<alias>.my.connect.aws/`.

---

## EventBridge status tracking

`detail-type` = **"Screen Recording Status Changed"**, `source` = **"aws.connect"**. `recordingStatus` values: **INITIATED** (agent accept), **COMPLETED** (ended on desktop, not yet uploaded), **PUBLISHED** (in S3; payload adds `location`, `durationInMillis`, `sizeInBytes`, `startTime`/`endTime`/`publishTime`), **FAILED** (`failureInfo`). Payload also has `clientInfo.appVersion`, `instanceArn`, `contactArn`, `agentArn`.

---

## Security profile permissions (Analytics and optimization)

- **Screen recording - Access** — view/review recordings (also lets the user hear the unredacted audio merged into the video).
- **Screen recording - Enable download button** — shows the download button on Contact details.

---

## Limitations

- Up to 3 monitors; 5 fps / OpenH264 fixed; unredacted audio only; Completed contacts only.
- Not supported when an agent is logged into multiple CCP instances simultaneously.
- Custom CCP/agent desktops supported only via `amazon-connect-streams` (test first).
- No playback on the legacy `awsapps.com` domain; Chrome/Edge v147+ need the loopback enterprise policy.
- Troubleshooting logs (Windows): `C:\ProgramData\Amazon\Amazon.Connect.Client.Service\logs` and `%USERPROFILE%\AppData\Local\Amazon\Amazon.Connect.Client.RecordingSession\Logs`.
