# Agent Workspace — Permissions & AppConfig Troubleshooting

This page turns the PDF troubleshooting guidance into a repeatable workflow: **inspect AppConfig → verify permissions → map failures to fixes**.

## 1) Inspect AppConfig at runtime

Your app’s create event context includes `appConfig`. Use it to check what the workspace believes your app is allowed to do (especially `permissions`).

```ts
import { AmazonConnectApp } from "@amazon-connect/app";

AmazonConnectApp.init({
  onCreate: (event) => {
    const { appConfig } = event.context;
    console.log("AppConfig:", appConfig);
    console.log("Permissions:", appConfig?.permissions);
  },
  onDestroy: () => {},
});
```

## 2) Recognize the two most common permission errors

These show up when your app subscribes to an event or makes a request you don’t have permissions for.

- **Event subscription without permission**
  - Symptom: subscribing to a topic fails immediately.
  - Error pattern (as documented): “App attempted to subscribe to topic without permission …”
- **Request without permission**
  - Symptom: calling a client method rejects.
  - Error pattern (as documented): “App does not have permission for this request”

## 3) Fix checklist

When you see either pattern:

1. Confirm you are running the app from the correct **Access URL** / origin that is registered.
2. Confirm the app is associated to the correct **instance**.
3. In Connect admin, confirm:
   - The app integration has the required **permissions** selected (events/requests you need).
   - The agent’s **Security Profile** allows the app to be viewed/launched (Agent applications → View).
4. Re-open the app in the workspace so it re-initializes with the updated permissions.

## 4) Practical rule

If the app can’t subscribe or request, the fastest path is:

- Print `appConfig.permissions`
- Compare to what you expect to use
- Update the integration configuration and security profile accordingly

