# Agent Workspace — Testing (Local + Deployed)

## A) Local test (localhost)

Use this to validate SDK init, permissions, and lifecycle quickly.

1. Run your app locally (example): `http://localhost:3000`.
2. In the Connect admin console:
   - Create/update the third-party application integration.
   - Set **Access URL** to `http://localhost:3000`.
   - Add `http://localhost:3000` to **Approved origins**.
   - Associate the app to your instance.
3. In the agent’s **Security Profile**:
   - Under *Agent applications*, grant **View** for the app so it appears in the app launcher.
4. Open agent workspace and launch the app from the app launcher.

Expected behavior:

- If you open the app URL directly in a normal browser tab, the SDK will log that it failed to connect to workspace — this is expected.
- Inside workspace, `onCreate` should fire and you should see `appInstanceId` and `appConfig`.

## B) Deployed test (staging/prod)

1. Deploy your app to its production origin (HTTPS).
2. Update the integration:
   - Set Access URL to the HTTPS origin.
   - Add any IdP/login domains used during authentication to Approved origins.
3. Verify your app’s CSP `frame-ancestors` allows the workspace domain families (see `agent-experience/embedding-security-and-csp.md`).
4. Validate:
   - Loads in a normal tab
   - Loads in workspace iframe
   - Events and requests work under the configured permissions

