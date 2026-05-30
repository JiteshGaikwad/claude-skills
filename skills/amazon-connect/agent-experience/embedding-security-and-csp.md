# Agent Workspace — Embedding Security (CSP, Approved Origins)

This is the practical checklist for making a third-party app iframeable in agent workspace *and* preventing unintended embedding elsewhere.

## 1) Set CSP `frame-ancestors`

Configure your application to only allow embedding by Amazon Connect workspace domains.

Recommended example (Agent Workspace Developer Guide style):

```
Content-Security-Policy: frame-ancestors https://*.awsapps.com https://*.my.connect.aws;
```

Notes:

- Prefer CSP `frame-ancestors` over relying only on `X-Frame-Options` (modern browsers enforce CSP more consistently).
- If you deploy in GovCloud, also allow the GovCloud connect domain family if applicable to your instance domain.

## 2) Don’t break your own auth

If your app uses an IdP flow (redirects or embedded login), make sure:

- The IdP domain(s) are included in **Approved origins** (see below).
- Your app can function in an iframe (some IdPs block framing by default).

## 3) Configure Access URL + Approved origins in Connect

The workspace’s iframe allowlist is derived from:

- **Access URL** — your primary app URL (where the iframe loads from)
- **Approved origins** — additional domains the workspace will allow for the integration

Add approved origins for:

- Login redirect domains / IdP domains
- Staging domains
- Local development origins (for example `http://localhost:3000`)
- Any other domains your app needs to iframe / navigate to inside the workspace

## 4) Common failure modes

- **Blank iframe / “refused to frame”**: your CSP `frame-ancestors` does not allow the workspace domain.
- **Login page doesn’t load**: IdP blocks framing, or the IdP domain isn’t in Approved origins.
- **Works in normal tab, fails in workspace**: iframe restrictions (CSP/Approved origins) are different from standalone navigation.

