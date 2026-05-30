# Streams + AppManager — Lifecycle Management Notes

This page captures the lifecycle details that are easy to miss when hosting Amazon Connect apps inside a StreamsJS custom CCP container.

## AppHost lifecycle events

When you launch/host apps via AppManager, manage the iframe lifecycle explicitly:

- `onCreated` — iframe created and ready
- `onDestroying` — teardown begins
- `onDestroyed` — teardown complete

Practical rule:

- Do **not** remove the iframe DOM node until you receive `onDestroyed`.

## Visibility management

If you implement tabbing/panels:

- Call `appHost.stop()` when an app is hidden.
- Call `appHost.start()` when it becomes visible again.

## Prevent duplicate instances

If your UI can launch the same app multiple times, use a stable `launchKey` strategy so AppManager can avoid duplicates (and so your app doesn’t initialize multiple times unexpectedly).

## Dynamic changes

If your container supports apps being added/removed at runtime, wire the management events:

- `onAppHostAdded`
- `onAppHostRemoved`
- `onAppHostFocused`

## Catalog expectations

`getAppCatalog()` is filtered by what the current agent is allowed to see (security-profile gated). Don’t assume it returns the full set of apps configured on the instance.

