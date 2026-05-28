# CCP Troubleshooting Guide

Common issues, diagnostics, and resolution steps for the Amazon Connect Contact Control Panel.

---

## No Audio Issues

### One-Way Audio (Agent Hears Customer, Customer Does Not Hear Agent)

**Symptoms:** Agent hears the customer but the customer reports silence.

**Causes and fixes:**

1. **Microphone not selected or muted.**
   - Open CCP settings (gear icon) > Audio devices.
   - Verify the correct microphone is selected.
   - Check that the microphone is not muted in the operating system's sound settings.
   - Check for a physical mute switch on the headset.

2. **Microphone permissions denied in browser.**
   - The browser must have permission to access the microphone.
   - Chrome: click the lock icon in the address bar > Site settings > Microphone > Allow.
   - Firefox: click the lock icon > Permissions > Use the Microphone > Allow.
   - Edge: click the lock icon > Site permissions > Microphone > Allow.
   - After changing permissions, refresh the page.

3. **WebRTC media path blocked by firewall.**
   - Softphone uses WebRTC for media. If the outbound UDP path on ports 3478 and 49152-65535 is blocked, media cannot flow.
   - See "Network Diagnostics" section below.

4. **VPN or proxy intercepting media traffic.**
   - Split tunneling must be configured to route media traffic directly, not through the VPN tunnel.
   - See "Corporate Proxy/VPN Issues" section below.

### One-Way Audio (Customer Hears Agent, Agent Does Not Hear Customer)

**Symptoms:** Customer can hear the agent but the agent hears silence.

**Causes and fixes:**

1. **Speaker/headset not selected.**
   - Open CCP settings > Audio devices.
   - Verify the correct output device is selected.
   - Test with the built-in audio check.

2. **Volume muted or too low.**
   - Check the system volume for the selected output device.
   - Check the browser tab volume (some operating systems allow per-tab volume control).

3. **Inbound media blocked by firewall.**
   - The firewall may allow outbound UDP but block inbound media return traffic.
   - Ensure stateful firewall rules allow return UDP traffic.

### No Audio (Both Directions)

**Symptoms:** Neither party hears the other.

**Causes and fixes:**

1. **Softphone not initialized.**
   - The CCP may not have successfully initialized the softphone.
   - Refresh the CCP page.
   - Check the browser console for errors related to `connect-rtc`.

2. **WebRTC entirely blocked.**
   - Some corporate networks block WebRTC entirely.
   - Run the Amazon Connect connectivity tool (see "Network Diagnostics").
   - Switch to desk phone mode as a workaround while network issues are resolved.

3. **Browser does not support WebRTC.**
   - Use a supported browser (see "Browser Issues").

---

## Dropped Calls

**Symptoms:** Calls disconnect unexpectedly during handling.

**Causes and fixes:**

1. **Network instability.**
   - WebRTC media requires a stable connection. Packet loss above 5% or latency above 300ms degrades call quality and may cause drops.
   - Test bandwidth and latency using a speed test to the AWS region hosting the Connect instance.
   - If on Wi-Fi, switch to a wired Ethernet connection.

2. **Session timeout.**
   - If the agent's browser session expires, the CCP loses connection.
   - Ensure the agent's browser tab remains active (not sleeping/hibernated).
   - Check for session expiry warnings from the Activity client.

3. **Contact flow timeout.**
   - Some contact flows have timeout branches that disconnect after a set period.
   - Review the contact flow for timeout configurations.

4. **Customer-side disconnect.**
   - The customer may have hung up. Check the CTR for the disconnect reason (`CUSTOMER_DISCONNECT` vs. `AGENT_DISCONNECT` vs. `SYSTEM_ERROR`).

---

## Agent Cannot Sign In

**Symptoms:** Agent sees an error when trying to access the CCP or workspace.

**Causes and fixes:**

1. **User not created in Connect instance.**
   - Verify the agent exists in the Connect console under "User management."

2. **Incorrect credentials.**
   - If using Connect-managed authentication, verify username and password.
   - If using SAML/SSO, verify the SAML identity provider configuration and attribute mappings.

3. **Security profile not assigned.**
   - Every agent must have a security profile assigned. Without one, login fails.

4. **Instance URL incorrect.**
   - Verify the URL format: `https://{instance-alias}.my.connect.aws/agent-app-v2/` for the workspace, or `https://{instance-alias}.my.connect.aws/ccp-v2/` for standalone CCP.

5. **Browser cookies/cache stale.**
   - Clear cookies and cache for the Connect domain.
   - Try an incognito/private browsing window.

6. **Pop-up blocker preventing login window.**
   - The CCP login flow may use a pop-up window.
   - Allow pop-ups for the Connect domain.

---

## CCP Not Loading

**Symptoms:** The CCP iframe shows a blank screen, spinner, or error message.

**Causes and fixes:**

1. **JavaScript errors.**
   - Open the browser developer console (F12 > Console) and check for errors.
   - Common errors: CORS violations, missing resources, Content Security Policy (CSP) blocks.

2. **CSP blocking the iframe.**
   - If the CCP is embedded in a custom page, the page's Content Security Policy must allow the Connect domain.
   - Add `https://{instance}.my.connect.aws` to `frame-src` and `connect-src` CSP directives.

3. **Third-party cookie blocking.**
   - The CCP requires third-party cookies for authentication.
   - Chrome: Settings > Privacy > Cookies > Allow third-party cookies (or add exception for `*.my.connect.aws`).
   - Firefox: Add exception for the Connect domain in Enhanced Tracking Protection settings.
   - Safari: Safari blocks third-party cookies by default. Use a pop-up login flow instead.

4. **Incompatible browser version.**
   - See "Browser Issues" for supported versions.

5. **Network blocking WebSocket connections.**
   - The CCP uses WebSocket connections to `wss://*.transport.connect.aws`.
   - Verify WebSocket traffic is allowed through the firewall/proxy.

---

## Error States

**Symptoms:** Agent is stuck in "Error" or "Missed" state.

**Causes and fixes:**

1. **Missed contact.**
   - The agent did not accept a contact within the configured timeout.
   - The agent is placed in Error (or "Missed Contact") state.
   - Resolution: the agent must manually change their status back to "Available" or another status.

2. **System error.**
   - A system error occurred during contact handling (e.g., network failure, service interruption).
   - Resolution: refresh the CCP. If the error persists, clear cookies and re-login.

3. **Stuck in ACW.**
   - The agent cannot clear a contact from ACW.
   - Check for pending step-by-step guide completion.
   - If the guide is stuck, refresh the CCP. The contact should re-appear and can be cleared.

---

## Network Diagnostics

### Amazon Connect Connectivity Tool

AWS provides a browser-based connectivity test:

1. Navigate to `https://{instance}.my.connect.aws/ccp-v2/` and log in.
2. Use the built-in connectivity check in CCP settings, or use the standalone Amazon Connect Check tool.
3. The tool tests:
   - HTTPS connectivity to Connect APIs.
   - WebSocket connectivity for signaling.
   - WebRTC media path (TURN/STUN servers).
   - Latency to the Connect instance region.

### Check WebRTC Connectivity

WebRTC requires:

| Protocol | Port(s) | Direction | Purpose |
|---|---|---|---|
| HTTPS | 443 | Outbound | Signaling, API calls |
| WSS | 443 | Outbound | WebSocket signaling |
| UDP | 3478 | Outbound | STUN (NAT traversal) |
| UDP | 49152-65535 | Outbound | Media (RTP/SRTP) |
| TCP | 443 | Outbound | TURN fallback (when UDP is blocked) |

If UDP is blocked, WebRTC falls back to TCP via TURN servers on port 443. This works but adds latency and may reduce call quality.

### Verify Firewall Rules

Required domains to allowlist:

| Domain Pattern | Purpose |
|---|---|
| `*.my.connect.aws` | Workspace and CCP |
| `*.transport.connect.aws` | WebSocket signaling |
| `*.static.connect.aws` | Static assets |
| `*.execute-api.{region}.amazonaws.com` | API calls |
| `{region}.turn.connect.aws` | TURN servers |
| `*.cloudfront.net` | CDN for static assets |

Replace `{region}` with the AWS region of the Connect instance (e.g., `us-east-1`).

### Test Bandwidth

Minimum bandwidth requirements:

| Channel | Minimum | Recommended |
|---|---|---|
| Voice (softphone) | 100 Kbps up + 100 Kbps down | 200 Kbps up + 200 Kbps down |
| Chat | Negligible | Negligible |
| Video (if used) | 1 Mbps up + 1 Mbps down | 2 Mbps up + 2 Mbps down |

Measure bandwidth to the AWS region hosting the instance, not just to generic speed test servers.

---

## Softphone Errors

### Microphone Permissions

When the CCP first loads, the browser requests microphone access. If denied:

- **Chrome:** shows a crossed-out microphone icon in the address bar. Click it to re-enable.
- **Firefox:** shows a notification bar. Click "Allow."
- **Edge:** similar to Chrome, click the lock icon to change permissions.

If microphone access was permanently denied, the agent must manually update browser settings:

1. Navigate to browser settings > Site settings > Microphone.
2. Find the Connect domain in the blocked list.
3. Change to "Allow."
4. Refresh the CCP.

### Headset Detection

**Headset not recognized:**

1. Verify the headset is physically connected and powered on (for Bluetooth headsets, check pairing).
2. Open operating system sound settings and verify the headset appears as an input/output device.
3. In CCP settings > Audio devices, click the refresh button to re-detect devices.
4. If the headset was connected after the CCP loaded, refresh the page.

**Headset switching mid-call:**

- If a headset is connected or disconnected during an active call, the CCP may not automatically switch.
- The agent should manually select the new device in CCP settings.
- Some browsers (Chrome) support automatic device switching; others require manual selection.

---

## Browser Issues

### Supported Browsers

| Browser | Minimum Version | Notes |
|---|---|---|
| Google Chrome | Latest 3 major versions | Recommended. Best WebRTC support. |
| Mozilla Firefox | Latest 3 major versions | Supported. May require additional permissions configuration. |
| Microsoft Edge (Chromium) | Latest 3 major versions | Supported. Same engine as Chrome. |
| Safari | Not recommended | Limited WebRTC support. Third-party cookie issues. |

### Clear Cache

When experiencing loading or display issues:

1. Clear the browser cache for the Connect domain specifically (not all sites):
   - Chrome: Settings > Privacy > Clear browsing data > Advanced > select "Cached images and files" and set time range.
   - Or use Ctrl+Shift+Delete (Cmd+Shift+Delete on Mac).
2. Alternatively, hard-refresh: Ctrl+Shift+R (Cmd+Shift+R on Mac).
3. If issues persist, clear cookies for the Connect domain as well.

### Disable Extensions

Browser extensions can interfere with the CCP:

- **Ad blockers** may block WebSocket connections or API calls.
- **Privacy extensions** may block third-party cookies required for authentication.
- **VPN extensions** may route traffic through unexpected paths.

To diagnose: try the CCP in an incognito/private window with extensions disabled. If it works there, re-enable extensions one by one to find the culprit.

### Check for Multiple CCP Instances

Running multiple CCP instances (tabs, windows, or embedded instances) causes conflicts:

- Only one CCP instance should be active per browser profile.
- Multiple instances compete for the same WebRTC media session, causing audio issues.
- Close all CCP tabs/windows except one.
- If embedding, ensure only one `initCCP()` call executes.

---

## Log Collection for Support Cases

When filing an AWS support case for CCP issues, collect:

### Browser Console Logs

1. Open the browser developer tools (F12).
2. Navigate to the Console tab.
3. Reproduce the issue.
4. Right-click in the console > "Save as" to export logs.
5. Include the full console output in the support case.

### Network Logs (HAR File)

1. Open the browser developer tools (F12).
2. Navigate to the Network tab.
3. Check "Preserve log" to prevent logs from clearing on navigation.
4. Reproduce the issue.
5. Right-click in the network tab > "Save all as HAR with content."
6. Attach the HAR file to the support case.

### CCP Logs

The CCP generates internal logs accessible via the Streams API:

```javascript
// Download CCP logs
connect.getLog().download();
```

This downloads a log file containing CCP events, errors, and timing information.

### Information to Include

- Connect instance alias and region.
- Agent username.
- Contact ID(s) affected.
- Timestamp of the issue (with timezone).
- Browser name and version.
- Operating system and version.
- Network environment (corporate network, VPN, home network).
- Steps to reproduce.
- Screenshots or screen recordings if applicable.

---

## Corporate Proxy/VPN Issues

### Problem

Corporate networks often route all traffic through a proxy server or VPN tunnel. This can cause:

- WebRTC media failure (proxy cannot handle UDP).
- High latency (traffic routed through distant proxy servers).
- WebSocket connection drops (proxy timeout on long-lived connections).
- Certificate inspection breaking TLS (proxy MITM).

### Split Tunneling

Configure the VPN to route Connect traffic directly (not through the VPN tunnel):

**Domains to exclude from VPN/proxy:**

- `*.my.connect.aws`
- `*.transport.connect.aws`
- `*.static.connect.aws`
- `*.turn.connect.aws`
- `*.amazonaws.com` (or specifically the Connect API endpoints)

**IP ranges to exclude:**

- AWS publishes IP ranges at `https://ip-ranges.amazonaws.com/ip-ranges.json`.
- Filter for the Connect service in the relevant region.

### Proxy Configuration

If split tunneling is not possible:

1. **HTTPS proxy** -- Configure the proxy to pass through (not inspect) traffic to `*.connect.aws` domains. TLS inspection will break WebSocket and may cause authentication failures.
2. **WebSocket proxy support** -- Ensure the proxy supports WebSocket upgrades (HTTP 101 Switching Protocols). Some older proxies do not.
3. **UDP proxy** -- Most corporate proxies cannot proxy UDP traffic. In this case, WebRTC will fall back to TCP via TURN on port 443. This adds latency but works.
4. **Proxy timeout** -- Increase the proxy idle timeout for WebSocket connections to at least 600 seconds (default proxy timeouts of 30-60 seconds will drop the connection).

### Recommended Network Architecture for Corporate Environments

```
Agent Browser
  |
  +-- HTTPS/WSS (port 443) --> Direct or via proxy --> Connect APIs + Signaling
  |
  +-- UDP (ports 3478, 49152-65535) --> Direct (bypass proxy) --> TURN/STUN servers --> Media
  |
  +-- TCP 443 (fallback) --> via proxy if needed --> TURN relay --> Media
```

The key principle: **media traffic (UDP) should bypass the proxy**. Signaling traffic (HTTPS/WSS) can go through the proxy if it supports WebSocket and does not break TLS.

### Endpoint Test Utility
- Available from Connect admin console under "Troubleshooting"
- Tests: network connectivity, WebSocket, media path, browser compatibility
- Results: pass/fail for each endpoint category
- Run before deployment to validate agent workstations
- Checks TURN server reachability, DNS resolution, and WebRTC capability

### QualityMetrics in Contact Records
- Contact record field: `QualityMetrics`
- Contains: MOS (Mean Opinion Score 1-5), jitter (ms), packet loss (%), round-trip time (ms)
- Access via `GetContactAttributes` API or contact record export
- Use for: diagnosing audio quality issues after the fact, trending quality over time

### Outbound Call Issues
- **Agent can't make outbound calls**: Check routing profile has outbound queue assigned, verify security profile has "Make outbound calls" permission, confirm phone number has outbound capability
- **Caller ID not showing**: Check queue outbound caller ID config, verify claimed number supports outbound

### Mobile/Tablet Support
- CCP and agent workspace do NOT support mobile phones (iPhone/Android) or iPads
- Desktop browser required for full functionality
- Desk phone mode works from any phone but the UI must remain on a desktop browser

### Screen Recording Issues
- **Not starting**: Check Set Recording block in flow, verify screen recording enabled on instance
- **Quality issues**: Depends on agent screen resolution and network bandwidth
- **Storage**: Recordings stored in S3; check bucket permissions and encryption config

### Audio Humming/Buzzing
- **Cause**: Sample rate mismatch between headset and browser audio context
- **Fix**: Ensure headset sample rate matches browser (typically 48kHz)
- **USB headsets preferred** over Bluetooth (lower latency, fewer codec issues)
