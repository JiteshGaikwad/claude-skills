# Agent Workspace — Streams vs Third-Party Apps (Do’s and Don’ts)

There are two different integration paths that are easy to confuse:

## 1) Third-party apps/services (Agent Workspace SDK)

Use this when your app runs *inside* the agent workspace as a tab (UI) or as a background service.

- Initialize with `AmazonConnectApp.init()` (apps) or `AmazonConnectService.init()` (services).
- Create clients using the returned `{ provider }`.
- **Do not** initialize CCP/Streams inside the app.

## 2) StreamsJS custom CCP page (you host the workspace/CCP container)

Use this when you are embedding the CCP into a custom web page you host (for example a custom CRM).

- You load `connect-streams.js` and call `connect.core.initCCP(...)`.
- Optionally use AppManager plugin if you also embed AWS-managed apps into your custom container.

## Why this matters

The Agent Workspace Developer Guide explicitly states that initializing the CCP via Streams (even hidden) is **not supported** in third-party applications. Treat Streams as a *custom CCP embedding* tool, not a requirement for building workspace apps.

