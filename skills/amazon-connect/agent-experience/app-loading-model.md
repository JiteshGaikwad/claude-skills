# Agent Workspace — App Loading Model (Per-Contact vs Cross-Contact)

This page explains how the agent workspace loads third-party apps and services so you can design state management, cleanup, and contact scoping correctly.

## Key ideas

- **Apps run in iframes.** Your app is loaded in an iframe managed by the workspace.
- **Multiple instances can exist.** The workspace can load multiple instances of the same app at the same time (for example, one instance per active contact).
- **Visibility is not lifecycle.** An app can be hidden (tab switch / contact switch) without being destroyed.

## Per-contact vs cross-contact scope

When you register an integration, you choose its contact scope:

- **Per-contact**: the workspace may create a new iframe instance per contact. Your app instance should assume it is bound to exactly one contact context at a time.
- **Cross-contact**: the workspace keeps one iframe instance across contacts. Your app must handle contact switching and avoid leaking contact state between sessions.

## Hidden vs destroyed (what to expect)

Typical behaviors to design for:

- **Hidden**: the iframe still exists; timers/websockets may keep running. Do not assume you will get a destroy event when the user navigates away.
- **Destroyed**: the workspace removes the iframe (for example: the tab is closed; a per-contact app instance is torn down; the workspace is unloading).

Your cleanup strategy should be:

- Prefer **idempotent cleanup** (safe to run multiple times).
- Persist any critical state **before** you call `sendFatalError()` (fatal errors can skip destroy handshake).

## Contact isolation rules

- Treat all event/request data you receive as **scoped to the current app instance’s contact context**.
- Do not cache contact identifiers globally unless you are explicitly in cross-contact scope and you manage switching carefully.
- If your app needs cross-contact aggregation, store it externally (server-side) and fetch per contact as needed.

## CSP composition (why “Approved origins” matters)

The agent workspace’s iframe allowlist is derived from your integration configuration:

- **Access URL** (where the app is loaded from)
- **Approved origins** (additional domains the workspace allows)

If your app does any of the following, add those domains to Approved origins:

- Redirects to an IdP domain for login (or embeds an IdP in an iframe)
- Loads a different domain for an embedded sub-app
- Uses a separate domain for staging/local testing

See `agent-experience/embedding-security-and-csp.md` for concrete CSP examples.

