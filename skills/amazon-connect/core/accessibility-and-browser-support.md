# Amazon Connect — Accessibility & Browser Support

Covers Amazon Connect's accessibility posture (compliance reports, supported screen readers) and supported browsers, plus the browser-policy changes that break the Contact Control Panel (CCP) / agent workspace and how to fix them.

## Accessibility compliance

AWS strives to provide an accessible user interface for Amazon Connect. Accessibility Conformance Reports (ACRs) are regularly published in **AWS Artifact**.

- For the current ACR/VPAT, go to **AWS Artifact** (see *Getting started with AWS Artifact*) and download the latest Amazon Connect accessibility compliance report. The ACR in AWS Artifact is the source of truth for the current WCAG conformance level — do not assume a specific level; pull the published report.
- For AWS compliance programs more broadly, see *Compliance validation in Amazon Connect*.

> The Admin Guide does not state a specific WCAG level inline. The conformance level is whatever the current ACR in AWS Artifact documents.

## Supported screen readers

Screen readers can be used optionally by people who have difficulty seeing websites or applications. Amazon Connect supports three popular screen readers:

- **JAWS**
- **NVDA**
- **VoiceOver** — for Apple devices. No download required: on an Apple device go to **Settings → Accessibility → VoiceOver**.

## Supported browsers

Verify your browser is supported before working with Amazon Connect.

| Browser | Supported versions | How to check |
|---|---|---|
| Google Chrome | Latest three versions | Type `chrome://version` in the address bar; version is in the **Google Chrome** field. |
| Microsoft Edge (Chromium) | Latest three versions | Menu → **Help and feedback** → **About Microsoft Edge**. |
| Mozilla Firefox | Latest three versions | Menu → **Help** icon → **About Firefox**. |
| Mozilla Firefox ESR | Supported until each version's Firefox end-of-life date (see the Firefox ESR release calendar) | Menu → **Help** icon → **About Firefox**. |

**Safari is not supported** (for the console/CCP/agent workspace).

Each supported browser has an associated browser-policy caveat: Chrome → third-party cookies; Edge → v146 autoplay policy; Firefox → Enhanced Tracking Protection. See the playbooks below.

### Mobile browsers

The Amazon Connect console, CCP, and agent workspace **do not support mobile browsers**. Agents can forward the audio portion of a call to their mobile device instead (see *Forward calls in the CCP to a mobile device*).

### In-app, web, and video calling (communications widget / JS SDK)

Different support matrix than the CCP — this covers the customer-facing widget, not the agent desktop:

- **Amazon Chime SDK for iOS and Android:**
  - iOS version 13 and later
  - Android OS version 8.1 and later, ARM and ARM64 architecture
- **Web browsers for the out-of-the-box communications widget and JS SDK:** latest three versions of **Google Chrome, Firefox, Safari, and Microsoft Edge Chromium** on macOS, Windows, iOS, and Android. (Safari *is* supported here, unlike the CCP.)
- **Voice Focus (VF) and Echo Reduction (ER):** not universally supported. Lower-spec devices may not support Amazon Voice Focus regardless of form factor. Where VF is unsupported, the browser's built-in noise suppression is used instead.

## Browser-policy breakage playbooks

These are the known browser-policy changes that break agent audio/auth and their fixes.

### Chrome — third-party cookies (3PCD)

On July 22, 2024, Google changed its plans: rather than deprecating third-party cookies by default, it offers an **opt-in mechanism** for users to disable them.

- **Impact:** for businesses that embed the CCP into a custom workspace, if agents use Google's opt-in mechanism to disable third-party cookies, it causes **authentication issues** in the CCP. Connect relies on third-party cookies to help authentication.
- **Fix:** ensure **third-party cookies are enabled** in agents' browser settings.

### Firefox — Enhanced Tracking Protection (ETP) / Total Cookie Protection

As of February 2024, Firefox enabled **Total Cookie Protection** by default for all users (including those whose Enhanced Tracking Protection is set to **Standard**). This **prevents the CCP from being embedded in another application**, so agents cannot handle contacts.

- **Fix (per user):**
  1. In Firefox, go to **Settings → Privacy & Security**.
  2. In the **Custom** box, for **Cookies** choose **Cross-site tracking cookies**.

### Firefox — microphone access (tab focus)

The CCP conforms to Firefox microphone usage guidance and only has microphone access **when the CCP tab is in focus**. This can cause **missed-call scenarios** when the CCP tab is not focused (e.g., the agent is on a different tab or app).

- **Fix:** agents must focus the CCP or Agent Workspace Firefox tab when accepting/connecting to a voice contact.

### Microsoft Edge v146 — autoplay policy change

Edge v146 (released March 13, 2026) changed autoplay policy behavior. When the `AutoplayAllowed` enterprise policy is set to **"Disabled"**, it now maps to **"Block"** (websites cannot autoplay media). In Edge versions 92–144 the same setting mapped to **"Limit"** and permitted audio playback on active WebRTC streams.

- **Impact** (only for customers on Edge v146+ with group policy `AutoplayAllowed` = "Disabled"):
  - Agents cannot hear ringtones or audio on incoming voice contacts (one-way audio).
  - Agents cannot see or hear the end customer on video calls.
- **Fix:** configure the `AutoplayAllowlist` policy to explicitly permit autoplay on your Amazon Connect instance URL.
  - **Group Policy path:** Administrative Templates/Microsoft Edge
  - **Policy name:** Allow media autoplay on specific sites
  - **Registry path:** `SOFTWARE\Policies\Microsoft\Edge\AutoplayAllowlist`
- **URLs to allowlist:**
  - Standard: `https://[your-instance-name].my.connect.aws`
  - AWS GovCloud: `https://[your-instance-name].govcloud.connect.aws`
  - If you use a custom CCP with `connect-rtc-js` (audio element loaded on your own page), also add your hosting domain: `https://[your-hosting-domain]`
- **Note:** the wildcard `*` is **not** accepted by this policy — use the exact instance URL. Configure via Group Policy (`MSEdge.admx`), Microsoft Intune, or the Windows Registry. See *AutoplayAllowlist policy* and *AutoplayAllowed policy* in the Microsoft Edge documentation.

## Flow Designer performance on multi-GPU Windows systems

On Windows systems with dual GPUs, Flow Designer animations may feel less smooth in Firefox than in Chrome because browsers default to the power-saving GPU. Chrome defaults to 60 FPS output; Firefox may cap at 30 FPS.

- **Fix:** point the browser at the dedicated GPU.
  1. Open **Windows Settings → Display → Graphics → Browse**.
  2. Navigate to the install folder:
     - Firefox: `C:\Program Files\Mozilla Firefox`
     - Chrome: `C:\Program Files\Google\Chrome\Application`
  3. Select `firefox.exe` or `chrome.exe`.
  4. Choose **Options → High Performance** to use the dedicated GPU.
  5. Save and restart the browser.

## Cross-references

- Microphone access setup → *Grant microphone access in Chrome, Firefox, or Edge*
- Audio troubleshooting (humming/sample rate) → *Verify Firefox sample rate*, *Verify Chrome sample rate*
- Screen recording on Chrome 147 / Edge 147+ → *Troubleshoot screen recording failures*
- Headset and workstation requirements → *Agent headset and workstation requirements for using the CCP*
