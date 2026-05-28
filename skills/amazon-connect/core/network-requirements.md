# Amazon Connect — Network Requirements for CCP

## Browser Requirements
- Google Chrome (latest 3 versions)
- Mozilla Firefox (latest 3 versions)
- Microsoft Edge Chromium (latest 3 versions)

## Network Configuration
- CCP requires WebRTC for voice
- Minimum bandwidth: 100 Kbps per active call
- Recommended: low-latency connection (<150ms RTT to Connect endpoint)

## Firewall / Proxy Rules
Allow outbound HTTPS (443) and media (UDP/TCP) to:
- `*.my.connect.aws` — agent workspace/CCP
- `*.static.connect.aws` — static assets
- Connect IP ranges (published in `ip-ranges.json` under service `AMAZON_CONNECT`)
- WebRTC media: UDP ports 3478+ for TURN/STUN

## DNS
- Resolve `*.my.connect.aws` domains
- Do not cache DNS aggressively — IPs may change

## TLS
- TLS 1.2+ required
- No SSL inspection on WebRTC media traffic (breaks encryption)

## Common Issues
- Corporate proxies blocking WebRTC → audio one-way or no audio
- VPN split tunneling needed for media traffic
- Firewall blocking UDP → forces TCP fallback (higher latency)

## Softphone Requirements
- Headset with USB connection recommended
- Built-in echo cancellation in CCP
- 16kHz audio quality

## Domain Allowlist
All of the following domains must be reachable from the agent workstation:
- `*.my.connect.aws` — agent workspace, CCP login, API calls
- `*.transport.connect.aws` — WebRTC signaling and media transport
- `*.static.connect.aws` — static assets (JS, CSS, images)
- `*.turn.connect.aws` — TURN/STUN relay servers for NAT traversal
- Wildcard entries are required because subdomains vary by instance and region

## Stateless Firewalls
- Stateless firewalls (e.g., AWS NACLs, some hardware firewalls) do **not** track connection state
- You must create **explicit inbound rules** to allow return UDP traffic on ephemeral ports (typically 1024–65535)
- Without inbound rules for return traffic, WebRTC media will fail even if outbound is allowed
- Stateful firewalls (security groups, most enterprise firewalls) handle return traffic automatically

## Agents Using Connect Remotely
- **Split tunneling** is strongly recommended when agents use a VPN — route Connect media traffic directly to the internet, not through the VPN tunnel
- Forcing media through a VPN concentrator adds latency and jitter, degrading call quality
- Minimum **100 Kbps per active voice call** (both directions)
- If split tunneling is not possible, ensure the VPN path has low latency (<150ms RTT) and sufficient bandwidth
- Test with the Connect endpoint connectivity tool before go-live

## VDI Environment
- Running CCP inside Citrix, RDP, or other VDI platforms introduces additional audio latency
- **CCP v2 media optimization**: Some VDI vendors support redirecting WebRTC media from the virtual session to the local endpoint — this is the recommended approach
- Audio should route **directly** from the agent's local device to Connect, bypassing the VDI host
- Without media optimization, expect 100–300ms added latency and reduced audio quality
- Citrix HDX RealTime and VMware Horizon support WebRTC redirection — check vendor documentation for current compatibility

## Using AWS Direct Connect
- AWS Direct Connect is **not required** but can improve call quality for on-premises agents
- Use a **Public VIF** (Virtual Interface) — Connect endpoints are public, not inside a VPC
- Direct Connect provides a more consistent network path compared to internet routing
- Does not eliminate the need for the domain allowlist — DNS and TLS requirements still apply
- For hybrid setups, combine Direct Connect for office agents with internet for remote agents

## Region Selection
- Choose the region **closest to your agents** to minimize network latency
- Consider **data residency** requirements — call recordings, CTRs, and other data are stored in the selected region
- If agents are distributed globally, use Global Resiliency (multi-region) to place agents in their nearest region
- Available regions are listed in the AWS Connect documentation — not all regions support all features

## Agent Workstation Requirements
- **CPU**: 2+ cores (4 cores recommended if running CRM alongside CCP)
- **RAM**: 4 GB minimum (8 GB recommended)
- **Network**: 100 Kbps minimum per active voice call; 200+ Kbps recommended for concurrent voice + screen sharing
- **Browser**: Chrome, Firefox, or Edge — latest 3 major versions
- **Audio**: USB headset recommended; built-in speakers/mic are supported but may cause echo
- **OS**: Windows 10+, macOS 10.15+, or supported Linux distributions
- **Display**: 1280x720 minimum resolution for the agent workspace
