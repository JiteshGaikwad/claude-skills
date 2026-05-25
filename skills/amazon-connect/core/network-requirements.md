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
