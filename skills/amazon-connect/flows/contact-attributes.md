# Contact Attributes in Amazon Connect

Contact attributes are key-value pairs associated with a contact that carry data throughout the contact's lifecycle. They are the primary mechanism for passing information between flow blocks, Lambda functions, Lex bots, and the agent workspace.

## Attribute Types

### System Attributes

Automatically set by Amazon Connect. Read-only.

| Attribute | JSONPath | Description |
|-----------|----------|-------------|
| Customer Number | `$.SystemEndpoint.Address` | The customer's phone number (E.164 format). Same as `$.CustomerEndpoint.Address` in Lambda event. |
| Dialed Number | `$.SystemEndpoint.Address` | The number the customer dialed (DID/TFN). |
| Customer Callback Number | `$.CustomerCallback.Address` | The callback number (defaults to customer number). |
| Channel | `$.Channel` | `VOICE`, `CHAT`, or `TASK`. |
| Contact ID | `$.ContactId` | Unique identifier for the current contact. |
| Initial Contact ID | `$.InitialContactId` | The ID of the original contact in a transfer chain. |
| Previous Contact ID | `$.PreviousContactId` | The ID of the previous contact in a transfer chain. |
| Queue Name | `$.Queue.Name` | Name of the queue the contact is assigned to. |
| Queue ARN | `$.Queue.ARN` | ARN of the queue. |
| Text to Speech Voice | `$.TextToSpeechVoiceId` | Current Polly voice ID. |
| Contact Duration | `$.Contact.Duration` | Duration of the contact in seconds. |
| Instance ARN | `$.InstanceARN` | ARN of the Connect instance. |
| Language Code | `$.LanguageCode` | Language/locale code for the contact. |
| Initiation Method | `$.InitiationMethod` | How the contact was initiated: `INBOUND`, `OUTBOUND`, `TRANSFER`, `CALLBACK`, `API`, `QUEUE_TRANSFER`. |

### User-Defined (Contact) Attributes

Set by the `Set contact attributes` block or by Lambda responses saved via `Set contact attributes`. These are custom key-value pairs you define.

| JSONPath | Description |
|----------|-------------|
| `$.Attributes.{key}` | Any user-defined attribute. Key is the attribute name you chose. |

Examples:
- `$.Attributes.customerName`
- `$.Attributes.accountId`
- `$.Attributes.callerIntent`

### External Attributes

Returned by the most recent Lambda function invocation. Overwritten each time a new Lambda is invoked.

| JSONPath | Description |
|----------|-------------|
| `$.External.{key}` | A key from the Lambda response. |

Examples:
- `$.External.customerTier`
- `$.External.balance`
- `$.External.address.city` (nested JSON response)

External attributes are ephemeral. If you need them later in the flow (especially after another Lambda call), copy them to user-defined attributes using `Set contact attributes`.

### Lex Attributes

Set by Amazon Lex during a `Get customer input` interaction.

| JSONPath | Description |
|----------|-------------|
| `$.Lex.{SlotName}` | Value of a Lex slot. |
| `$.Lex.IntentName` | The resolved intent name. |
| `$.Lex.SessionAttributes.{key}` | Lex session attributes. |
| `$.Lex.IntentConfidence` | Confidence score of the matched intent. |
| `$.Lex.SentimentLabel` | Sentiment label (POSITIVE, NEGATIVE, NEUTRAL, MIXED). |
| `$.Lex.SentimentScore` | Sentiment score. |

Examples:
- `$.Lex.AccountNumber` (slot named AccountNumber)
- `$.Lex.SessionAttributes.lastIntent`

### Queue Metrics Attributes

Returned by the `Get metrics` block.

| JSONPath | Description |
|----------|-------------|
| `$.Metrics.Queue.Name` | Queue name for the metrics. |
| `$.Metrics.Queue.ARN` | Queue ARN for the metrics. |
| `$.Metrics.Queue.Size` | Number of contacts in the queue. |
| `$.Metrics.Queue.OldestContactAge` | Age (seconds) of the oldest contact in queue. |
| `$.Metrics.Agents.Available.Count` | Number of available agents. |
| `$.Metrics.Agents.Online.Count` | Number of online agents. |
| `$.Metrics.Agents.Staffed.Count` | Number of staffed agents. |
| `$.Metrics.Agents.AfterContactWork.Count` | Number of agents in ACW. |
| `$.Metrics.Agents.Busy.Count` | Number of agents on contacts. |
| `$.Metrics.Agents.MissedCount` | Number of missed contacts. |
| `$.Metrics.Agents.NonProductive.Count` | Number of agents in non-productive states. |
| `$.Metrics.Agents.Error.Count` | Number of agents in error state. |

### Media Streams Attributes

Available when media streaming is active (after `Start media streaming` block).

| JSONPath | Description |
|----------|-------------|
| `$.MediaStreams.Customer.Audio.StreamARN` | ARN of the Kinesis Video Stream for the customer's audio. |
| `$.MediaStreams.Customer.Audio.StartTimestamp` | Timestamp when streaming started. |
| `$.MediaStreams.Customer.Audio.StartFragmentNumber` | Fragment number where streaming began. |

## How to Set Contact Attributes

### In the Flow Designer

Use the **Set contact attributes** block:

1. Add the block to your flow.
2. Configure the attribute:
   - **Destination key**: The attribute name (e.g., `callerIntent`).
   - **Value**: Can be:
     - **Use text**: A static string value.
     - **Use attribute**: A reference to another attribute using JSONPath (e.g., `$.External.tier`).

### From Lambda

Return attributes from Lambda, then use `Set contact attributes` to save them:

```javascript
// Lambda returns:
return {
  customerName: "Jane Doe",
  accountId: "ACC-12345"
};
```

In the flow after the Lambda block, add `Set contact attributes`:
- Source type: External
- Source attribute: `customerName`
- Destination key: `customerName`

Alternatively, contact attributes passed via `UpdateContactAttributes` in the Lambda code directly:

```javascript
const { ConnectClient, UpdateContactAttributesCommand } = require("@aws-sdk/client-connect");

const client = new ConnectClient({});

exports.handler = async (event) => {
  const { ContactId, InstanceARN } = event.Details.ContactData;
  const instanceId = InstanceARN.split("/").pop();

  await client.send(new UpdateContactAttributesCommand({
    InstanceId: instanceId,
    InitialContactId: ContactId,
    Attributes: {
      customerName: "Jane Doe",
      accountId: "ACC-12345"
    }
  }));

  return { status: "ok" };
};
```

## How to Reference Contact Attributes

### In Play Prompt (TTS)

Use JSONPath directly in the text:

```
Hello, $.Attributes.customerName. Your account number is $.Attributes.accountId.
```

The flow engine substitutes the attribute values at runtime.

### In Check Contact Attributes (Branching)

1. Add the `Check contact attributes` block.
2. Set the **Attribute to check**: Use the JSONPath (e.g., `$.Attributes.callerIntent`).
3. Add conditions:
   - Equals `billing` -> route to billing queue
   - Equals `support` -> route to support queue
   - No match -> default branch

### In Lambda Parameters

In the `Invoke AWS Lambda function` block, add function input parameters:
- Key: `accountId`
- Value: `$.Attributes.accountId`

These appear in `event.Details.Parameters.accountId` in the Lambda function.

### In Lex Session Attributes

Pass contact attributes to Lex as session attributes in the `Get customer input` block:
- Source: `$.Attributes.customerName`
- Lex session attribute key: `customerName`

## JSONPath Syntax Reference

Amazon Connect uses a subset of JSONPath for attribute references:

| Syntax | Meaning |
|--------|---------|
| `$.Attributes.{key}` | User-defined contact attribute |
| `$.External.{key}` | Lambda response attribute |
| `$.Lex.{SlotName}` | Lex slot value |
| `$.Lex.SessionAttributes.{key}` | Lex session attribute |
| `$.Queue.Name` | Current queue name |
| `$.Queue.ARN` | Current queue ARN |
| `$.SystemEndpoint.Address` | System endpoint (dialed number) |
| `$.CustomerEndpoint.Address` | Customer endpoint (caller number) |
| `$.Channel` | Contact channel |
| `$.ContactId` | Contact ID |
| `$.InitialContactId` | Initial contact ID |
| `$.Metrics.Queue.Size` | Queue size metric |
| `$.MediaStreams.Customer.Audio.StreamARN` | Media stream ARN |

## Attribute Limits

| Limit | Value |
|-------|-------|
| Maximum number of user-defined attributes per contact | 200 |
| Maximum attribute key length | 256 characters |
| Maximum attribute value length | 32,768 characters |
| Maximum total attributes size | 32 KB |
| Attribute key allowed characters | Alphanumeric, hyphens, periods, underscores |

## Best Practices

- **Copy External to Contact attributes** immediately after Lambda invocation if you need the values later. External attributes are overwritten by the next Lambda call.
- **Use meaningful attribute names**: `customerTier` not `ct`, `accountStatus` not `as`.
- **Keep attribute values small**: The 32 KB total limit is shared across all attributes on the contact.
- **Do not store sensitive data in attributes** unless encrypted. Contact attributes appear in contact records, CloudWatch Logs (if flow logging is enabled), and the agent workspace.
- **Use contact tags** (via `Contact tags` block) for categorization and search; use attributes for data that drives flow logic.
