# Agent Workspace Developer Guide

This guide covers the three integration approaches for extending the Amazon Connect agent workspace, plus the full SDK API reference.

---

## Agent Workspace Overview

The Amazon Connect agent workspace is a browser-based application that provides agents with a unified interface for handling customer interactions. It embeds the Contact Control Panel (CCP), step-by-step guides, Customer Profiles, Cases, and AI-powered agent assist tools into a single workspace shell.

Applications load inside iframes within the workspace shell. The workspace handles theming and event/request routing between all embedded applications. Apps must provide their own authentication (see Authentication below) — the workspace session is not shared with embedded apps.

There are three types of integrations:

- **Third-party applications (3P apps)**: Custom web applications with a visible UI that appear as tabs in the workspace.
- **Third-party services (headless)**: Background processes with no visible UI that run for the entire agent session.
- **AWS-managed applications**: Built-in applications managed by AWS (Customer Profiles, Cases, etc.).

Agent workspace URL: `https://{instance-alias}.my.connect.aws/agent-app-v2/`

### How apps load (important mental model)

Apps load in iframes. Design for these behaviors:

- **Per-contact apps**: the workspace can create a separate iframe instance per contact, so multiple instances of the same app may run at once (for different contacts).
- **Hidden vs destroyed**: switching contacts/tabs can hide an app without destroying it. Destroy happens when the workspace removes the iframe (for example when the tab is closed, or a per-contact app instance is torn down).
- **Contact isolation**: events/data are scoped to the contact context for that iframe instance; do not assume state is shared across contacts.
- **CSP composition**: the workspace’s iframe allowlist is derived from your integration configuration (Access URL + Approved origins). If your app does login redirects or embeds additional domains, those domains must be included as approved origins or they may be blocked.

See `agent-experience/app-loading-model.md` for patterns and pitfalls.

---

## Best Practices

- **Embedding security**: Use a CSP `frame-ancestors` allowlist so your app can only be embedded by Amazon Connect. The Agent Workspace Developer Guide examples allow both `https://*.awsapps.com` and your instance domain family (for example `https://*.my.connect.aws`).
- **Multiple domains**: If your app uses additional domains (login redirects, CDNs, staging, localhost), register them as **Approved origins** so the workspace allows them in iframes.
- **Streams vs 3P apps**: Do **not** initialize the CCP (Streams `initCCP`) inside a third-party app. Use Streams only for a custom CCP page you host; use `AmazonConnectApp.init()` for third-party applications and services.
- **Accessibility**: Follow WCAG 2.1 AA guidelines and test with automated tools plus real assistive tech (keyboard-only navigation and screen readers).
- **Theming**: If you use Cloudscape, apply the Connect theme (`@amazon-connect/theme`) using the SDK `provider`. Otherwise, use the SDK theme APIs (for example `getTheme()` / `onThemeChanged`) to react to dark-mode changes.

Example CSP header (copy/paste baseline):

```
Content-Security-Policy: frame-ancestors https://*.awsapps.com https://*.my.connect.aws;
```

Detailed checklists:
- `agent-experience/app-loading-model.md`
- `agent-experience/embedding-security-and-csp.md`
- `agent-experience/permissions-and-appconfig.md`
- `agent-experience/testing-local-and-deployed.md`
- `agent-experience/streams-vs-3p-apps.md`
- `agent-experience/appmanager-lifecycle.md`

---

## A. Third-Party Applications (3P Apps)

Third-party applications are custom web applications loaded inside the agent workspace via HTTPS iframes. They appear as tabs and interact with the workspace through the Amazon Connect SDK.

### Prerequisites — IAM Role

An IAM role with the following permissions is required to register and manage third-party applications:

- `app-integrations:CreateApplication`
- `connect:AssociateApprovedOrigin`

In the AWS console, register your application under the Amazon Connect admin console > "Third-party applications."

### Prerequisites

- For staging/production: HTTPS-hosted web application (required for normal workspace embedding).
- For local testing: `http://localhost:{port}` is supported as an Access URL/approved origin for validating the integration.
- Application registered in the Amazon Connect console under "Third-party applications."
- `@amazon-connect/sdk` packages installed.

### Create Your Application

1. Go to the Amazon Connect admin console and navigate to "Third-party applications."
2. Register your app: provide a name, namespace, and origin URL (must be HTTPS).
3. Configure permissions: select which agent data the app can access (contacts, agent state, etc.).
4. Associate the app with a security profile to control which agents see it in their workspace.

### Installation

```bash
npm install @amazon-connect/app @amazon-connect/contact @amazon-connect/core
```

The `@amazon-connect/app` package provides app initialization and lifecycle for third-party applications and must be installed by every app at a minimum. `AgentClient` and `ContactClient` both come from `@amazon-connect/contact`.

Additional SDK packages by feature:

```bash
npm install @amazon-connect/voice        # Voice call controls
npm install @amazon-connect/email        # Email handling
npm install @amazon-connect/file         # File upload/download
npm install @amazon-connect/message-template  # Message templates
npm install @amazon-connect/quick-responses   # Quick responses
npm install @amazon-connect/app-controller    # App lifecycle management
```

### Initialization

```typescript
import { AmazonConnectApp } from "@amazon-connect/app";

const { provider } = AmazonConnectApp.init({
  onCreate: (event) => {
    const { appInstanceId } = event.context;
    console.log("App initialized:", appInstanceId);
  },
  onDestroy: (event) => {
    console.log("App being destroyed");
  },
});

// Keep the { provider } reference -- it is required to create clients
// that interact with workspace events and requests.
```

`init()` is called on the `AmazonConnectApp` module and takes `onCreate` and `onDestroy` lifecycle callbacks. The call is synchronous (it is **not** awaited) and returns an object containing the `provider`. The callbacks fire as follows:

1. `onCreate` is invoked once the app has successfully initialized in the agent workspace. Its event context provides the `appInstanceId` (and `appConfig`, plus the current `contactId` when the app is opened during an active contact).
2. `onDestroy` is invoked when the agent workspace is about to destroy the iframe the app is running in -- use it to clean up resources and persist data.

If you load the app directly outside the workspace, the SDK logs that it was unable to connect to the workspace within the allotted time; this is expected when `init` runs outside the workspace.

### Authentication

Apps in the Connect Customer agent workspace **must** provide their own authentication to their users. The workspace session is **not** shared with your app, so do not rely on it for sign-in.

It is recommended that apps use the same identity provider (IdP) that the Connect Customer instance was configured to use when it was created. This way users only need to log in once for both the agent workspace and your application, since both use the same single sign-on provider.

#### Third-party cookie deprecation (3PCD)

Apps run inside an iframe, so the Google Chrome Third-Party Cookies Deprecation (3PCD) can affect apps that use cookie-based authentication/authorization. If your app is embedded in the workspace and uses cookies for auth, it is likely to be impacted. Recommendations:

- **Temporary solution:** allow third-party cookie access for the workspace domain.
- **Permanent solution:** follow the guidance from Chrome to choose the best option for your application.

Note: on July 22, 2024, Google announced it no longer plans to deprecate third-party cookies. There is no impact to embedded apps unless users explicitly opt in to deprecation, but apps should still adopt the prevention solutions above as a forward-looking measure.

### Theme Integration

The theme package applies the standard Connect Customer theme on top of Cloudscape, so your app matches the look and feel of the agent workspace. Import it once at the application entry point and pass it the `provider` returned by `init()`:

```typescript
import { applyConnectTheme } from "@amazon-connect/theme";

// `provider` is the value returned by AmazonConnectApp.init()
await applyConnectTheme(provider);
```

From then on, Cloudscape components and design tokens can be used directly. Install with `npm install -P @amazon-connect/theme @cloudscape-design/global-styles`.

### Lifecycle Events

An app hooks into lifecycle events through the `onCreate` and `onDestroy` callbacks passed to `AmazonConnectApp.init()`:

```typescript
import { AmazonConnectApp } from "@amazon-connect/app";

const { provider } = AmazonConnectApp.init({
  // Create: fired once the app has initialized in the workspace.
  // The event context provides appInstanceId, appConfig, and the
  // contactId when the app is opened during an active contact.
  onCreate: (event) => {
    const { appInstanceId } = event.context;
  },
  // Destroy: fired when the workspace is about to remove the app's
  // iframe. Use it to clean up resources and persist data; the
  // workspace waits for the app to report cleanup is complete.
  onDestroy: (event) => {
    // cleanup
  },
});
```

To signal an unrecoverable state, call `sendError` or `sendFatalError` on the `AmazonConnectApp` object. A fatal error causes the workspace to immediately remove the iframe without running the destroy handshake, so perform any required cleanup before sending it.

### Error Handling

```typescript
import { AmazonConnectApp } from "@amazon-connect/app";

try {
  const { provider } = AmazonConnectApp.init({ onCreate: () => {}, onDestroy: () => {} });
} catch (error) {
  if (error.code === "ORIGIN_NOT_ALLOWED") {
    console.error("App origin not registered in Connect console");
  } else if (error.code === "INSTANCE_UNREACHABLE") {
    console.error("Cannot reach Connect instance");
  } else if (error.code === "AUTH_FAILED") {
    console.error("Authentication handshake failed");
  }
}
```

### SDK Without Package Manager (Bundling Guide)

When npm is not available (e.g., legacy apps, static sites), you can bundle the SDK packages into a single file:

1. Create a build project:
   ```bash
   mkdir connect-sdk-build && cd connect-sdk-build
   ```
2. Initialize the project:
   ```bash
   npm init -y
   ```
3. Install SDK packages:
   ```bash
   npm install @amazon-connect/app @amazon-connect/contact @amazon-connect/core @amazon-connect/app-controller
   ```
4. Install a bundler:
   ```bash
   npm install --save-dev esbuild
   ```
5. Create an entry file (`index.js`):
   ```javascript
   export { ContactClient, AgentClient } from "@amazon-connect/contact";
   export { AmazonConnectApp } from "@amazon-connect/app";
   export { AppControllerClient } from "@amazon-connect/app-controller";
   ```
6. Add a build script to `package.json`:
   ```json
   {
     "scripts": {
       "build": "esbuild index.js --bundle --outfile=dist/amazon-connect-sdk.js --format=iife --global-name=AmazonConnectSDK"
     }
   }
   ```
7. Build:
   ```bash
   npm run build
   ```
8. Copy `dist/amazon-connect-sdk.js` to your project.

Usage in HTML:

```html
<script src="amazon-connect-sdk.js"></script>
<script>
  const { ContactClient, AgentClient } = window.AmazonConnectSDK;
</script>
```

**StreamsJS / custom-CCP path only** (this is a *different* integration from bundling a 3P app into the workspace — do not use `initCCP` for 3P apps): Load Streams JS first (`connect-streams.js`), then the SDK bundle. Initialize CCP via `connect.core.initCCP(container, { plugins: AmazonConnectSDK.AppManagerPlugin })`. Then, **after** `connect.core.onInitialized()` fires, retrieve the provider via `connect.core.getSDKClientConfig().provider` and create SDK clients with `new ContactClient(provider)`. The `AppManagerPlugin` is only needed if you also host Connect first-party apps (Cases, Step-by-Step Guides).

For 3P apps (not StreamsJS), do not call `connect.core.initCCP()`; initialize with `AmazonConnectApp.init()` and use the returned `{ provider }` as shown in the Initialization section above.

### Agent Data Integration

Use the Agent Client to subscribe to agent state and retrieve role/queue information for your app:

- Subscribe to agent state changes: `agentClient.onStateChanged(callback)`
- Get agent routing profile: `agentClient.getRoutingProfile()`
- Get available states: `agentClient.listAvailabilityStates()`
- Use case: show agent-specific data in your app based on their assigned queues or role.

### Events and Requests Pattern

The SDK uses two communication patterns between the workspace and your app:

- **Events**: workspace-to-app notifications (agent state changed, contact incoming, theme changed). Events use a subscribe/unsubscribe pattern with `on*` / `off*` methods.
- **Requests**: app-to-workspace actions (set agent state, accept contact, launch another app). Requests use async methods that return promises.

```typescript
// Event pattern: subscribe to notifications
contactClient.onIncoming((event) => {
  console.log("New contact:", event.contactId);
});

// Request pattern: perform an action
await contactClient.accept(contactId);
```

### Troubleshooting

**Events not received:**
- Verify the app origin is registered in the Connect console (exact match, including protocol and port).
- Check browser console for `postMessage` origin errors.
- Ensure `AmazonConnectApp.init()` was called and its returned `{ provider }` was captured before subscribing to events.

**Provider is undefined:**
- For 3P apps: ensure `AmazonConnectApp.init()` was called and its return value (the `{ provider }`) was captured.
- For StreamsJS / custom CCP: access the provider only **after** `connect.core.onInitialized()` fires, via `connect.core.getSDKClientConfig().provider`.

**Requests failing:**
- Confirm the agent's security profile grants permissions for the requested resource.
- Check that the SDK package version matches the Connect instance version.
- Look for CORS errors if the app makes direct API calls.

**Permission errors (events/requests):**
- Inspect `event.context.appConfig.permissions` in `onCreate` to confirm what the workspace granted.
- Common error patterns to look for:
  - “App attempted to subscribe to topic without permission …” (event subscription blocked)
  - “App does not have permission for this request” (request blocked)
- Fix path: update the integration permissions and ensure the agent’s security profile has **Agent applications → View** for the app, then reload the workspace tab.

**Testing locally:**
- For local testing, set the integration **Access URL** to `http://localhost:{port}` (for example `http://localhost:3000`) as described in the Agent Workspace Developer Guide.
- Add the same origin as an **Approved origin**.
- Ensure the agent’s **Security Profile** grants the app “View” access under *Agent applications* so it appears in the app launcher.
- Running the app outside the workspace should log that it failed to connect to the workspace — this is expected.

**Testing deployed:**
- Register the production domain as an allowed origin.
- Verify the app loads in a standalone browser tab before testing inside the workspace.
- Check the browser network tab for blocked requests or failed resource loads.

For a full end-to-end checklist (console fields + security profile + expected error messages), see `agent-experience/testing-local-and-deployed.md`.

---

## B. Third-Party Services (Headless)

Third-party services are background processes that run for the duration of the agent's workspace session. They have no visible UI -- they execute logic in response to workspace events.

### Use Cases

- **Auto-launch apps on ACW** -- listen for the ACW event and programmatically launch a specific 3P app.
- **Custom auth flows** -- perform additional authentication steps when the agent logs in.
- **Contact event listeners** -- log contact events to an external system, trigger webhooks, or update external CRM records.
- **App focus control** -- automatically switch focus to a specific app when a contact arrives.
- **Background data sync** -- periodically sync data between the workspace and an external system.

### Service Setup

1. In the Amazon Connect console, navigate to "Third-party applications."
2. Create a new application with the "Service" type (not "Application").
3. Provide the HTTPS URL of the service endpoint.
4. The service URL is loaded in a hidden iframe -- no UI is rendered.
5. The service initializes via `AmazonConnectService.init()` (from `@amazon-connect/app`) and subscribes to events. It must complete the initial handshake within its configured `InitializationTimeout` (in milliseconds), or the agent workspace will fail to load. All initialization must complete within the overall 30-second window.

### Example: Auto-Launch App on ACW

```typescript
import { AmazonConnectService } from "@amazon-connect/app";
import { ContactClient } from "@amazon-connect/contact";
import { AppControllerClient } from "@amazon-connect/app-controller";

const { provider } = AmazonConnectService.init({
  onCreate: (event) => {
    const { instanceId } = event.context;
    console.log("Service creation complete:", instanceId);
  },
});

const contactClient = new ContactClient({ provider });
const appController = new AppControllerClient({ provider });

// Listen for ACW state
contactClient.onStartingAcw((event) => {
  // Auto-launch the disposition app
  appController.launch({
    appId: "disposition-app-id",
    contactId: event.contactId,
  });
});
```

### Agent Workspace Startup Process

1. Agent opens the workspace URL.
2. Workspace loads the CCP, AWS-managed apps, and all registered third-party apps.
3. Third-party services are initialized in the background (no visible UI).
4. Services receive lifecycle events and can interact with apps via AppController.

### Example: Contact Event Listener with App Launch

```typescript
import { AmazonConnectService } from "@amazon-connect/app";
import { ContactClient } from "@amazon-connect/contact";
import { AppControllerClient } from "@amazon-connect/app-controller";

const { provider } = AmazonConnectService.init({ onCreate: () => {} });
const contactClient = new ContactClient({ provider });
const appController = new AppControllerClient({ provider });

contactClient.onIncoming(async (contact) => {
  const attrs = await contactClient.getAttributes(contact.contactId);
  if (attrs["customerTier"] === "premium") {
    await appController.launch({ appId: "premium-support-app" });
  }
});
```

### Example: Authentication Popup Pattern

```typescript
import { AmazonConnectService } from "@amazon-connect/app";
import { AppControllerClient } from "@amazon-connect/app-controller";

const { provider } = AmazonConnectService.init({ onCreate: () => {} });
const appController = new AppControllerClient({ provider });

// Service that handles OAuth popup for an external CRM
await appController.launch({
  appId: "crm-auth-popup",
  launchMode: "popup",
  metadata: { returnUrl: window.location.href },
});
```

### Service Best Practices

- Create services sparingly — each one adds to workspace startup time.
- Use services for cross-cutting concerns (authentication, logging, routing logic).
- Do not duplicate functionality that an app already handles.
- Services persist for the entire agent session — clean up event listeners on destroy.

### Differences from 3P Apps

| Aspect | 3P App | 3P Service |
|---|---|---|
| UI | Visible tab in workspace | Hidden (no UI) |
| Lifecycle | Loaded when tab is active or on event | Loaded at workspace startup, runs entire session |
| Use case | Interactive features | Background automation |
| User interaction | Agent interacts directly | No direct interaction |

---

## C. Streams + AppManager Integration

For custom CCP embedding and advanced workspace integrations, use the `amazon-connect-streams` library alongside `@amazon-connect/app-manager`.

### amazon-connect-streams

The Streams library (`connect-streams.js`) enables embedding the CCP in a custom web page and programmatically controlling it.

```html
<script src="https://cdn.jsdelivr.net/npm/amazon-connect-streams/release/connect-streams.min.js"></script>
```

Or install via npm:

```bash
npm install amazon-connect-streams
```

### @amazon-connect/app-manager

The AppManager library manages AWS-managed applications within an embedded workspace.

```bash
npm install @amazon-connect/app-manager
```

### Architecture

```
Your Web Page
  |
  +-- connect.core.initCCP(containerDiv, config)
  |     |
  |     +-- Initializes CCP in containerDiv (iframe)
  |     +-- Sets up event handlers (contact, agent)
  |
  +-- AppManager plugin (optional)
        |
        +-- connect.appManager.launchApp(appConfig)
              |
              +-- Creates AppHost
                    |
                    +-- Renders app in iframe
```

### Basic CCP Embedding

```javascript
// Initialize CCP
connect.core.initCCP(document.getElementById("ccp-container"), {
  ccpUrl: "https://my-instance.my.connect.aws/ccp-v2/",
  loginPopup: true,
  loginPopupAutoClose: true,
  loginOptions: {
    autoClose: true,
    height: 600,
    width: 400,
  },
  softphone: {
    allowFramedSoftphone: true,
    disableRingtone: false,
  },
  region: "us-east-1",
});

// Agent events
connect.agent((agent) => {
  console.log("Agent connected:", agent.getName());

  agent.onStateChange((stateChange) => {
    console.log("Agent state:", stateChange.newState);
  });

  agent.onRoutingProfileChanged((routingProfile) => {
    console.log("Routing profile:", routingProfile.name);
  });
});

// Contact events
connect.contact((contact) => {
  console.log("New contact:", contact.getContactId());

  contact.onConnected(() => {
    console.log("Contact connected");
  });

  contact.onEnded(() => {
    console.log("Contact ended");
  });
});
```

### CCP with AppManager Plugin

```javascript
import "amazon-connect-streams";
import { AppManager } from "@amazon-connect/app-manager";

// Initialize CCP with AppManager plugin
connect.core.initCCP(document.getElementById("ccp-container"), {
  ccpUrl: "https://my-instance.my.connect.aws/ccp-v2/",
  loginPopup: true,
  softphone: { allowFramedSoftphone: true },
  region: "us-east-1",
});

// Initialize AppManager after CCP
const appManager = new AppManager({
  // Configuration for managed apps
});

// Launch a third-party app
appManager.launchApp({
  appId: "my-app-id",
  containerId: "app-container",  // DOM element to render into
});
```

### AppHost lifecycle and visibility (important)

When using AppManager, treat hosted apps as lifecycled iframe instances (similar to workspace apps):

- **Lifecycle events**: handle `onCreated`, `onDestroying`, and `onDestroyed` for each AppHost.
  - Practical rule: do not remove the iframe DOM node until `onDestroyed` fires.
- **Visibility management**: if your UI hides/shows app containers, call:
  - `appHost.stop()` when hidden
  - `appHost.start()` when visible again
- **Avoid duplicates**: if your page can launch the same app multiple times, use a stable `launchKey` strategy so you don’t accidentally create multiple instances for the same logical tab/contact.
- **Dynamic app changes**: if you maintain a dynamic layout, wire management hooks such as:
  - `onAppHostAdded` / `onAppHostRemoved`
  - `onAppHostFocused`
- **Catalog expectations**: the app catalog is filtered by what the current agent is allowed to see (security-profile gated).

See `agent-experience/appmanager-lifecycle.md` for notes and patterns.

### React Example with Dynamic App Management

```tsx
import React, { useEffect, useRef, useState } from "react";
import "amazon-connect-streams";
import { AppManager } from "@amazon-connect/app-manager";

interface ActiveApp {
  id: string;
  name: string;
  containerId: string;
}

export function ConnectWorkspace() {
  const ccpRef = useRef<HTMLDivElement>(null);
  const [activeApps, setActiveApps] = useState<ActiveApp[]>([]);
  const [appManager, setAppManager] = useState<AppManager | null>(null);

  useEffect(() => {
    if (!ccpRef.current) return;

    // Initialize CCP
    connect.core.initCCP(ccpRef.current, {
      ccpUrl: "https://my-instance.my.connect.aws/ccp-v2/",
      loginPopup: true,
      softphone: { allowFramedSoftphone: true },
      region: "us-east-1",
    });

    // Initialize AppManager
    const manager = new AppManager({});
    setAppManager(manager);

    // Listen for contact events to auto-launch apps
    connect.contact((contact) => {
      contact.onConnected(() => {
        const channelType = contact.getType();
        if (channelType === connect.ContactType.VOICE) {
          launchApp(manager, "voice-assist-app");
        }
      });
    });

    return () => {
      // Cleanup on unmount
      manager?.destroy();
    };
  }, []);

  function launchApp(manager: AppManager, appId: string) {
    const containerId = `app-${appId}-${Date.now()}`;
    manager.launchApp({
      appId,
      containerId,
    });
    setActiveApps((prev) => [
      ...prev,
      { id: appId, name: appId, containerId },
    ]);
  }

  function closeApp(app: ActiveApp) {
    appManager?.closeApp(app.id);
    setActiveApps((prev) => prev.filter((a) => a.containerId !== app.containerId));
  }

  return (
    <div style={{ display: "flex", height: "100vh" }}>
      {/* CCP Panel */}
      <div ref={ccpRef} style={{ width: 320, flexShrink: 0 }} />

      {/* App Panels */}
      <div style={{ flex: 1, display: "flex", flexDirection: "column" }}>
        {activeApps.map((app) => (
          <div key={app.containerId} style={{ flex: 1, position: "relative" }}>
            <button onClick={() => closeApp(app)}>Close {app.name}</button>
            <div id={app.containerId} style={{ width: "100%", height: "100%" }} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## D. SDK API Reference

Complete reference for all 10 SDK clients. Total: approximately 117 methods across all clients.

---

### Activity Client

**Package:** `@amazon-connect/core` (built-in)

Manages agent session activity and keep-alive.

| Method | Description |
|---|---|
| `onSessionExpiryWarning(callback)` | Fires when the agent session is approaching expiry. The callback receives the time remaining. Use this to prompt the agent to extend their session. |
| `offSessionExpiryWarning(callback)` | Unsubscribes from session expiry warning events. |
| `onSessionExpiryCleared(callback)` | Fires when the session expiry warning is cleared (agent extended session or the warning condition resolved). |
| `offSessionExpiryCleared(callback)` | Unsubscribes from session expiry cleared events. |
| `onExtensionError(callback)` | Fires when a session extension attempt fails. The callback receives the error details. |
| `offExtensionError(callback)` | Unsubscribes from extension error events. |
| `reportActive()` | Reports agent activity to extend the session. Call periodically to prevent session timeout. Returns a Promise that resolves when the report is acknowledged. |

```typescript
import { ActivityClient } from "@amazon-connect/core";

const activityClient = new ActivityClient({ provider });

activityClient.onSessionExpiryWarning((event) => {
  console.log(`Session expires in ${event.timeRemaining}ms`);
  // Prompt agent or auto-extend
  activityClient.reportActive();
});
```

---

### Agent Client

**Package:** `@amazon-connect/contact`

Manages agent state, routing profile, and channel configuration.

| Method | Signature | Description |
|---|---|---|
| `getARN()` | `() => Promise<string>` | Returns the agent's Amazon Resource Name (ARN). |
| `getName()` | `() => Promise<string>` | Returns the agent's display name. |
| `getState()` | `() => Promise<AgentState>` | Returns the current agent state (Available, Offline, custom status). |
| `getRoutingProfile()` | `() => Promise<RoutingProfile>` | Returns the agent's routing profile (queues, channels, concurrency). |
| `getChannelConcurrency()` | `() => Promise<ChannelConcurrency>` | Returns per-channel concurrency settings (e.g., voice: 1, chat: 5). |
| `getExtension()` | `() => Promise<string>` | Returns the agent's phone extension (for desk phone mode). |
| `listAvailabilityStates()` | `() => Promise<AvailabilityState[]>` | Lists all available agent states (built-in + custom statuses). |
| `listQuickConnects()` | `() => Promise<QuickConnect[]>` | Lists quick connects available to the agent. |
| `setAvailabilityState(stateARN)` | `(stateARN: string) => Promise<void>` | Sets the agent's state by ARN. |
| `setAvailabilityStateByName(name)` | `(name: string) => Promise<void>` | Sets the agent's state by display name (e.g., "Available", "Break"). |
| `setOffline()` | `() => Promise<void>` | Sets the agent to Offline state. |
| `onStateChanged(callback)` | `(cb: (event: StateChangedEvent) => void) => void` | Fires when the agent's state changes. Event includes old and new state. |
| `offStateChanged(callback)` | `(cb) => void` | Unsubscribes from state change events. |
| `onRoutingProfileChanged(callback)` | `(cb: (event: RoutingProfileEvent) => void) => void` | Fires when the agent's routing profile changes (admin reassignment). |
| `offRoutingProfileChanged(callback)` | `(cb) => void` | Unsubscribes from routing profile change events. |
| `onEnabledChannelListChanged(callback)` | `(cb: (event: ChannelListEvent) => void) => void` | Fires when the agent's enabled channel list changes. |
| `offEnabledChannelListChanged(callback)` | `(cb) => void` | Unsubscribes from channel list change events. |

```typescript
import { AgentClient } from "@amazon-connect/contact";

const agentClient = new AgentClient({ provider });

const state = await agentClient.getState();
console.log(`Agent is: ${state.name}`);

agentClient.onStateChanged((event) => {
  console.log(`State changed: ${event.previousState.name} -> ${event.newState.name}`);
});

// Set agent to available
await agentClient.setAvailabilityStateByName("Available");
```

---

### AppController Client

**Package:** `@amazon-connect/app-controller`

Controls third-party application lifecycle within the workspace.

| Method | Signature | Description |
|---|---|---|
| `close(appId)` | `(appId: string) => Promise<void>` | Closes a running third-party application. |
| `focus(appId)` | `(appId: string) => Promise<void>` | Brings a third-party application to the foreground. |
| `getApp(appId)` | `(appId: string) => Promise<AppInfo>` | Returns metadata about a specific registered application. |
| `getCatalog()` | `() => Promise<AppCatalog>` | Returns the full catalog of registered third-party applications. |
| `getConfig()` | `() => Promise<AppControllerConfig>` | Returns the AppController configuration. |
| `getActiveApps()` | `() => Promise<ActiveApp[]>` | Returns a list of currently active (running) applications. |
| `launch(config)` | `(config: LaunchConfig) => Promise<void>` | Launches a third-party application. Config includes appId and optional parameters (contactId, containerId). |

```typescript
import { AppControllerClient } from "@amazon-connect/app-controller";

const appController = new AppControllerClient({ provider });

// Get available apps
const catalog = await appController.getCatalog();
console.log("Available apps:", catalog.apps.map(a => a.name));

// Launch an app
await appController.launch({ appId: "my-crm-app" });

// Focus an app
await appController.focus("my-crm-app");

// Close an app
await appController.close("my-crm-app");
```

---

### Contact Client

**Package:** `@amazon-connect/contact`

Core contact handling -- accept, transfer, disconnect, and events for all channel types.

| Method | Signature | Description |
|---|---|---|
| `accept(contactId)` | `(contactId: string) => Promise<void>` | Accepts an incoming contact. |
| `addParticipant(config)` | `(config: AddParticipantConfig) => Promise<void>` | Adds a participant to a contact (conference/transfer). |
| `clear(contactId)` | `(contactId: string) => Promise<void>` | Clears a contact after ACW is complete. |
| `disconnectParticipant(config)` | `(config: DisconnectConfig) => Promise<void>` | Disconnects a specific participant from the contact. |
| `engagePreviewContact(contactId)` | `(contactId: string) => Promise<void>` | Engages a preview contact (outbound preview campaigns). |
| `getAttribute(contactId, key)` | `(contactId: string, key: string) => Promise<string>` | Gets a single contact attribute by key. |
| `getAttributes(contactId)` | `(contactId: string) => Promise<Record<string, string>>` | Gets all contact attributes. |
| `getChannelType(contactId)` | `(contactId: string) => Promise<ChannelType>` | Returns the contact's channel (VOICE, CHAT, EMAIL, TASK). |
| `getContact(contactId)` | `(contactId: string) => Promise<ContactDetails>` | Returns full contact details. |
| `getInitialContactId(contactId)` | `(contactId: string) => Promise<string>` | Returns the initial contact ID (for transferred contacts, this is the original). |
| `getParticipant(contactId, participantId)` | `(contactId: string, participantId: string) => Promise<Participant>` | Returns details about a specific participant. |
| `getParticipantState(contactId, participantId)` | `(contactId: string, participantId: string) => Promise<ParticipantState>` | Returns the state of a specific participant (connected, on hold, etc.). |
| `getPreviewConfiguration(contactId)` | `(contactId: string) => Promise<PreviewConfig>` | Returns preview configuration for outbound preview contacts. |
| `getQueue(contactId)` | `(contactId: string) => Promise<Queue>` | Returns the queue associated with the contact. |
| `getQueueTimestamp(contactId)` | `(contactId: string) => Promise<Date>` | Returns when the contact entered the queue. |
| `getStateDuration(contactId)` | `(contactId: string) => Promise<number>` | Returns how long the contact has been in its current state (ms). |
| `isPreviewMode(contactId)` | `(contactId: string) => Promise<boolean>` | Returns whether the contact is in preview mode. |
| `listContacts()` | `() => Promise<ContactSummary[]>` | Lists all active contacts for the agent. |
| `listParticipants(contactId)` | `(contactId: string) => Promise<Participant[]>` | Lists all participants in the contact. |
| `transfer(config)` | `(config: TransferConfig) => Promise<void>` | Transfers the contact to a queue, agent, or phone number. |
| `onConnected(callback)` | `(cb: (event) => void) => void` | Fires when a contact is connected. |
| `offConnected(callback)` | `(cb) => void` | Unsubscribes from connected events. |
| `onCleared(callback)` | `(cb: (event) => void) => void` | Fires when a contact is cleared (ACW complete). |
| `offCleared(callback)` | `(cb) => void` | Unsubscribes from cleared events. |
| `onMissed(callback)` | `(cb: (event) => void) => void` | Fires when a contact is missed (not accepted in time). |
| `offMissed(callback)` | `(cb) => void` | Unsubscribes from missed events. |
| `onIncoming(callback)` | `(cb: (event) => void) => void` | Fires when an inbound contact is presented to the agent. |
| `offIncoming(callback)` | `(cb) => void` | Unsubscribes from incoming events. |
| `onParticipantAdded(callback)` | `(cb: (event) => void) => void` | Fires when a participant is added to the contact. |
| `offParticipantAdded(callback)` | `(cb) => void` | Unsubscribes from participant added events. |
| `onParticipantDisconnected(callback)` | `(cb: (event) => void) => void` | Fires when a participant disconnects from the contact. |
| `offParticipantDisconnected(callback)` | `(cb) => void` | Unsubscribes from participant disconnected events. |
| `onParticipantStateChanged(callback)` | `(cb: (event) => void) => void` | Fires when a participant's state changes (e.g., placed on hold). |
| `onStartingAcw(callback)` | `(cb: (event) => void) => void` | Fires when the agent enters After Contact Work state for a contact. |
| `offStartingAcw(callback)` | `(cb) => void` | Unsubscribes from ACW events. |

```typescript
import { ContactClient } from "@amazon-connect/contact";

const contactClient = new ContactClient({ provider });

// Listen for incoming contacts
contactClient.onIncoming((event) => {
  console.log(`Incoming contact: ${event.contactId}, channel: ${event.channelType}`);
});

// Accept a contact
contactClient.onIncoming(async (event) => {
  await contactClient.accept(event.contactId);
});

// Get contact attributes
const attrs = await contactClient.getAttributes(contactId);
console.log("Customer tier:", attrs["CustomerTier"]);

// Transfer to a queue
await contactClient.transfer({
  contactId,
  queueARN: "arn:aws:connect:us-east-1:123456789:instance/abc/queue/xyz",
});

// Listen for ACW
contactClient.onStartingAcw((event) => {
  console.log(`Contact ${event.contactId} entering ACW`);
});
```

---

### Email Client

**Package:** `@amazon-connect/email`

Handles email-specific operations (drafts, metadata, threading).

| Method | Signature | Description |
|---|---|---|
| `createDraft(config)` | `(config: DraftConfig) => Promise<DraftInfo>` | Creates a new email draft for a contact. Config includes subject, body (HTML), recipients, attachments. |
| `getMetadata(contactId)` | `(contactId: string) => Promise<EmailMetadata>` | Returns email metadata (subject, from, to, cc, bcc, timestamp, headers). |
| `getTree(contactId)` | `(contactId: string) => Promise<EmailThread>` | Returns the full email thread tree (original + all replies/forwards). |
| `sendDraft(draftId)` | `(draftId: string) => Promise<void>` | Sends a previously created draft. |
| `onAccepted(callback)` | `(cb: (event) => void) => void` | Fires when an email contact is accepted by the agent. |
| `offAccepted(callback)` | `(cb) => void` | Unsubscribes from email accepted events. |
| `onDraftCreated(callback)` | `(cb: (event) => void) => void` | Fires when a draft is created. |
| `offDraftCreated(callback)` | `(cb) => void` | Unsubscribes from draft created events. |

```typescript
import { EmailClient } from "@amazon-connect/email";

const emailClient = new EmailClient({ provider });

// Get email thread
const thread = await emailClient.getTree(contactId);
console.log("Thread messages:", thread.messages.length);

// Create and send a reply
const draft = await emailClient.createDraft({
  contactId,
  subject: "Re: Your inquiry",
  body: "<p>Thank you for contacting us...</p>",
  recipients: { to: ["customer@example.com"] },
});
await emailClient.sendDraft(draft.draftId);
```

---

### File Client

**Package:** `@amazon-connect/file`

Manages file uploads and downloads for attachments across channels.

| Method | Signature | Description |
|---|---|---|
| `batchGetMetadata(fileIds)` | `(fileIds: string[]) => Promise<FileMetadata[]>` | Returns metadata for multiple files (name, size, type, upload status). |
| `completeUpload(uploadId)` | `(uploadId: string) => Promise<void>` | Completes a multipart upload that was started with `startUpload`. |
| `delete(fileId)` | `(fileId: string) => Promise<void>` | Deletes a file. |
| `download(fileId)` | `(fileId: string) => Promise<Blob>` | Downloads a file. Returns the file content as a Blob. |
| `startUpload(config)` | `(config: UploadConfig) => Promise<UploadInfo>` | Initiates a file upload. Config includes file name, size, content type, and the associated contact ID. Returns upload URL and upload ID. |

```typescript
import { FileClient } from "@amazon-connect/file";

const fileClient = new FileClient({ provider });

// Upload a file
const upload = await fileClient.startUpload({
  contactId,
  fileName: "report.pdf",
  fileSizeInBytes: file.size,
  contentType: "application/pdf",
});

// Upload file content to the presigned URL
await fetch(upload.uploadUrl, {
  method: "PUT",
  body: file,
  headers: { "Content-Type": "application/pdf" },
});

// Complete the upload
await fileClient.completeUpload(upload.uploadId);

// Download a file
const blob = await fileClient.download(fileId);
```

---

### MessageTemplate Client

**Package:** `@amazon-connect/message-template`

Manages message templates for email and chat responses.

| Method | Signature | Description |
|---|---|---|
| `getContent(templateId)` | `(templateId: string) => Promise<TemplateContent>` | Returns the full content of a message template (subject, body, placeholders). |
| `isEnabled()` | `() => Promise<boolean>` | Returns whether message templates are enabled for the instance. |
| `search(query)` | `(query: SearchQuery) => Promise<TemplateSearchResult>` | Searches message templates by keyword, category, or channel type. |

```typescript
import { MessageTemplateClient } from "@amazon-connect/message-template";

const templateClient = new MessageTemplateClient({ provider });

// Check if templates are enabled
const enabled = await templateClient.isEnabled();

// Search for templates
const results = await templateClient.search({
  query: "password reset",
  channelType: "EMAIL",
});

// Get template content
const template = await templateClient.getContent(results.templates[0].id);
console.log("Template body:", template.body);
```

---

### QuickResponses Client

**Package:** `@amazon-connect/quick-responses`

Manages quick responses (canned messages with shortcuts).

| Method | Signature | Description |
|---|---|---|
| `isEnabled()` | `() => Promise<boolean>` | Returns whether quick responses are enabled for the instance. |
| `search(query)` | `(query: SearchQuery) => Promise<QuickResponseSearchResult>` | Searches quick responses by keyword, shortcut, or category. Returns matching responses with their content and shortcuts. |

```typescript
import { QuickResponsesClient } from "@amazon-connect/quick-responses";

const qrClient = new QuickResponsesClient({ provider });

// Search for quick responses
const results = await qrClient.search({ query: "greeting" });
results.responses.forEach((qr) => {
  console.log(`Shortcut: ${qr.shortcut}, Content: ${qr.content}`);
});
```

---

### User Client

**Package:** `@amazon-connect/core` (built-in)

Manages user preferences and language settings.

| Method | Signature | Description |
|---|---|---|
| `getLanguage()` | `() => Promise<string>` | Returns the agent's language preference (e.g., "en-US", "es-ES"). |
| `onLanguageChanged(callback)` | `(cb: (event: LanguageEvent) => void) => void` | Fires when the agent changes their language preference. |
| `offLanguageChanged(callback)` | `(cb) => void` | Unsubscribes from language change events. |

```typescript
const language = await app.getUser().getLanguage();
console.log("Agent language:", language);

app.getUser().onLanguageChanged((event) => {
  console.log("Language changed to:", event.language);
  // Update app localization
});
```

---

### Voice Client

**Package:** `@amazon-connect/voice`

Controls voice-specific operations (hold, resume, conference, outbound, DTMF, voice enhancement).

| Method | Signature | Description |
|---|---|---|
| `hold(contactId)` | `(contactId: string) => Promise<void>` | Places the customer on hold. |
| `resume(contactId)` | `(contactId: string) => Promise<void>` | Resumes the customer from hold. |
| `conference(contactId)` | `(contactId: string) => Promise<void>` | Merges all participants into a conference call. |
| `createOutbound(config)` | `(config: OutboundConfig) => Promise<OutboundResult>` | Initiates an outbound call. Config includes destination number, outbound caller ID, and optional contact flow. |
| `getCustomerNumber(contactId)` | `(contactId: string) => Promise<string>` | Returns the customer's phone number. |
| `getOutboundPermission()` | `() => Promise<OutboundPermission>` | Returns whether the agent has outbound calling permission and allowed countries. |
| `isOnHold(contactId)` | `(contactId: string) => Promise<boolean>` | Returns whether the customer is currently on hold. |
| `listDialableCountries()` | `() => Promise<Country[]>` | Lists countries the agent is allowed to dial. |
| `getVoiceEnhancementMode(contactId)` | `(contactId: string) => Promise<VoiceEnhancementMode>` | Returns the current voice enhancement mode (noise cancellation, etc.). |
| `setVoiceEnhancementMode(contactId, mode)` | `(contactId: string, mode: VoiceEnhancementMode) => Promise<void>` | Sets the voice enhancement mode for a contact. |
| `canResumeParticipant(contactId, participantId)` | `(contactId: string, participantId: string) => Promise<boolean>` | Returns whether a specific held participant can be resumed. |
| `canResumeSelf(contactId)` | `(contactId: string) => Promise<boolean>` | Returns whether the agent can resume themselves from hold. |
| `getVoiceEnhancementModelPaths()` | `() => Promise<string[]>` | Returns available voice enhancement model paths. |
| `onHold(callback)` | `(cb: (event) => void) => void` | Fires when a participant is placed on hold. |
| `offHold(callback)` | `(cb) => void` | Unsubscribes from hold events. |
| `onResume(callback)` | `(cb: (event) => void) => void` | Fires when a participant is resumed from hold. |
| `offResume(callback)` | `(cb) => void` | Unsubscribes from resume events. |
| `onConference(callback)` | `(cb: (event) => void) => void` | Fires when a conference is established. |
| `offConference(callback)` | `(cb) => void` | Unsubscribes from conference events. |
| `onCapabilityChanged(callback)` | `(cb: (event) => void) => void` | Fires when voice capabilities change (e.g., hold/resume availability). |
| `offCapabilityChanged(callback)` | `(cb) => void` | Unsubscribes from capability change events. |
| `onVoiceEnhancementChanged(callback)` | `(cb: (event) => void) => void` | Fires when voice enhancement mode changes. |
| `offVoiceEnhancementChanged(callback)` | `(cb) => void` | Unsubscribes from voice enhancement change events. |
| `onOutboundConnected(callback)` | `(cb: (event) => void) => void` | Fires when an outbound call connects. |
| `offOutboundConnected(callback)` | `(cb) => void` | Unsubscribes from outbound connected events. |
| `onOutboundFailed(callback)` | `(cb: (event) => void) => void` | Fires when an outbound call fails. |
| `offOutboundFailed(callback)` | `(cb) => void` | Unsubscribes from outbound failed events. |

```typescript
import { VoiceClient } from "@amazon-connect/voice";

const voiceClient = new VoiceClient({ provider });

// Hold and resume
await voiceClient.hold(contactId);
console.log("Customer on hold:", await voiceClient.isOnHold(contactId));
await voiceClient.resume(contactId);

// Make outbound call
const result = await voiceClient.createOutbound({
  destinationNumber: "+15551234567",
  outboundCallerId: "+15559876543",
});

// Voice enhancement
const mode = await voiceClient.getVoiceEnhancementMode(contactId);
await voiceClient.setVoiceEnhancementMode(contactId, "NOISE_CANCELLATION");

// Events
voiceClient.onHold((event) => {
  console.log(`Participant ${event.participantId} placed on hold`);
});

voiceClient.onResume((event) => {
  console.log(`Participant ${event.participantId} resumed`);
});
```
