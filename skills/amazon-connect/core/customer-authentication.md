# Amazon Connect — Customer Authentication

Customer authentication lets you verify that an incoming chat request comes from a user who has already signed in to your own website or application, so you can securely personalize the conversation. In the hosted communications widget, this is done by having your web server issue a **JSON Web Token (JWT)** for each new chat request. The widget includes the JWT in the chat request, and Amazon Connect validates it using a shared security key before routing the chat to an agent.

Related capabilities the Admin Guide groups under this area:

- **Communications widget JWT authentication** — verify chat requests came from authenticated users on your website (the focus of this doc).
- **Authenticate Customer flow block** — a flow block for authenticating customers inside a flow (see the flow block reference and AWS docs for details).
- **In-app, web, and video calling** — if a customer is already logged into your app, they do not need to identify or authenticate themselves when they request a call or video conversation; you can pass their context (such as profile attributes or prior in-app actions) straight to Connect.
- **Voice ID** — voice biometrics for caller authentication (covered separately).

> The Admin Guide content for this topic is primarily about communications-widget JWT authentication and passing authenticated context. For the **Authenticate Customer** flow block configuration and Amazon Cognito / OIDC identity-provider setup, the guide points to the flow block definition reference and the broader AWS documentation rather than detailing console steps here.

## Supported Regions

Customer authentication is available in (per the Admin Guide):

- US East (N. Virginia)
- US West (Oregon)
- Africa (Cape Town)
- Asia Pacific (Seoul)
- Asia Pacific (Singapore)
- Asia Pacific (Sydney)

For the authoritative, current list, see **Customer authentication availability by Region** in the Amazon Connect documentation.

## How widget authentication works

When you secure the communications widget with JWTs:

1. A customer signs in to your website or application.
2. The customer chooses the **Start chat** icon on your site.
3. The communications widget calls your web server (via the `authenticate` callback) to request a JWT.
4. Your web server generates a JWT, signing it with the 44-character security key that Amazon Connect provides.
5. The widget includes the JWT as part of the customer's chat request to Amazon Connect.
6. Amazon Connect uses the shared secret key to validate (decrypt) the token. If validation succeeds, this confirms the request was issued by your web server, and Connect routes the chat to your contact center agents.

Because the security key lives only on your web server, this gives you control over which chat requests are accepted and lets you verify they come from authenticated users.

## 1. Enable security on the communications widget

These steps are part of setting up the hosted communications widget (Customize communications widget in the admin website).

1. Log in to the admin website at `https://INSTANCE_NAME.my.connect.aws/` and choose **Customize communications widget**.
2. On the **Communications widgets** page, choose **Add communications widget** (or edit an existing one), then enter a **Name** and **Description**.
3. In **Communications options**, choose how customers can engage (for example, chat), then **Save and continue**.
4. Specify the website **domains** where the widget may appear, then continue.
5. Under **Add security for your communications widget**, choose **Yes** and work with your website administrator to set up your web servers to issue JWTs for new chat requests.

Choosing **Yes** results in:

- Amazon Connect providing a **44-character security key** on the next page, used to create JWTs.
- Amazon Connect adding a callback function inside the widget embed script that checks for a JWT when a chat is initiated.

You must implement the `authenticate` callback in the embedded snippet:

```javascript
amazon_connect('authenticate', function(callback) {
  window.fetch('/token').then(res => {
    res.json().then(data => {
      callback(data.data);
    });
  });
});
```

6. Choose **Save**.

## 2. Domain allowlist behavior

Chat loads only on the website domains you allow (up to **50** domains). Allowlist rules:

- **Subdomains are automatically included.** If you allow `example.com`, then `sub.example.com` is also allowed.
- **Protocol must match exactly.** Specify `http://` or `https://` to match your configuration. (Use `https://` for production.)
- **All URL paths are automatically allowed.** If `example.com` is allowed, every page under it (`example.com/cart`, `example.com/checkout`) is accessible. You cannot allow or block specific subdirectories.

> Include the full URL starting with `https://` and double-check the domains are valid.

## 3. Get the security key and generate JWTs

On the confirm/copy step you can copy the widget code and, if you chose JWTs, the security key(s).

- The **44-character security key** is used by your web server to generate JWTs.
- You can **rotate** keys: Connect issues a new key while keeping the previous one valid until you deploy the new one, then you delete the old key.

### JWT specifics

- **Algorithm:** `HS256`
- **Claims:**
  - `sub`: the `widgetId` (replace with your own widget ID; find it in the communications widget script).
  - `iat`: Issued At Time.
  - `exp`: Expiration — **10 minute maximum**.
  - `segmentAttributes` (optional): system-defined key-value pairs stored on contact segments via an attribute map (see `SegmentAttributes` in the `StartChatContact` API).
  - `attributes` (optional): string-to-string key-value pairs; must follow `StartChatContact` API limits.
  - `relatedContactId` (optional): a valid contact ID; must follow `StartChatContact` API limits.
  - `customerId` (optional): a Customer Profiles ID, or a custom identifier from an external system such as a CRM.

Example JWT generation in Python (`PyJWT` required — `pip install PyJWT`):

```python
import jwt
import datetime

CONNECT_SECRET = "your-securely-stored-jwt-secret"  # security key from Amazon Connect
WIDGET_ID = "widget-id"
JWT_EXP_DELTA_SECONDS = 500

payload = {
    'sub': WIDGET_ID,
    'iat': datetime.datetime.utcnow(),
    'exp': datetime.datetime.utcnow() + datetime.timedelta(seconds=JWT_EXP_DELTA_SECONDS),
    'customerId': "your-customer-id",
    'relatedContactId': 'your-relatedContactId',
    'segmentAttributes': {"connect:Subtype": {"ValueString": "connect:Guide"}},
    'attributes': {
        "name": "Jane",
        "memberID": "123456789",
        "email": "jane@example.com",
        "isPremiumUser": "true",
        "age": "45"
    }
}

header = {'typ': "JWT", 'alg': 'HS256'}
encoded_token = jwt.encode(payload, CONNECT_SECRET, algorithm="HS256", headers=header)
```

## 4. Pass authenticated contact attributes

You can carry information about the authenticated customer into the contact by adding an `attributes` claim to the JWT. Those attributes are then available in the flow and can be shown to the agent in the Contact Control Panel (CCP) — for example to greet the customer by name or surface account/member IDs.

To do this:

1. Enable widget security (choose **Yes**) and use the security key to generate JWTs, as above.
2. Add your contact attributes to the JWT payload as the `attributes` claim.
3. Pass the resulting encoded token to `callback(data)` in the `authenticate` snippet — no additional snippet changes are needed.

Things to know:

- **Token size limit:** the encoded token must be within **6144 bytes**. Because JavaScript uses UTF-16 (2 bytes per character), the practical maximum is about **3000 characters**.
- **Integrity, not secrecy:** the JWT protects integrity (a bad actor cannot tamper with the data if you safeguard the shared secret), but `attributes` are only **encoded, not encrypted** — they can be decoded and read. Do not put secrets in claims.

## 5. Pass a customer display name (related)

You can pass the customer's display name during contact initialization so it appears to both the customer and agent and is recorded in the transcript. Use the `customerDisplayName` callback:

```javascript
amazon_connect('customerDisplayName', function(callback) {
  const displayName = 'Jane Doe';
  callback(displayName);
});
```

Notes:

- Only **one** `customerDisplayName` function can exist at a time.
- The name must be a string **1–256 characters** (per `StartChatContact` limits). Empty/`null`/`undefined` is invalid — the widget logs `Invalid customerDisplayName provided` and falls back to the default display name `Customer`.
- The snippet runs in your website front end, so **do not pass sensitive data** as the display name.

## Starting authenticated chats with StartChatContact

For custom chat applications (not the hosted widget), start chats with the **`StartChatContact`** API. The JWT claims above (`segmentAttributes`, `attributes`, `relatedContactId`, `customerId`) follow the same limits this API defines, so use the `StartChatContact` API reference as the source of truth for field constraints.

When you first explore chat, note that chats are **not** counted in the `Contacts Incoming` metric initially, because the contact record's **Initiation Method** is `API`. After the chat is transferred to an agent, `Contacts Incoming` is incremented.

A CloudFormation-based open-source example (API Gateway + Lambda) is available to bootstrap a custom chat backend; see the Amazon Connect open source chat library and the Participant Service / Streams documentation.

## Limitations and notes

- **JWT expiration** is capped at **10 minutes** (`exp` claim).
- **Encoded token size** is capped at **6144 bytes** (~3000 characters in the widget).
- JWTs use **HS256** with the 44-character shared security key; rotate keys via the admin website.
- Widget domain allowlist supports up to **50** domains; subdomains and all paths are auto-included, and protocol must match exactly.
- `attributes` in the JWT are encoded, not encrypted — never include secrets.
- Customer authentication has limited regional availability (see **Supported Regions** above).
