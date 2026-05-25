# Lambda Integration in Amazon Connect Flows

## Overview

Amazon Connect flows can invoke AWS Lambda functions to perform data lookups, business logic, API calls, and integrations with external systems. The `Invoke AWS Lambda function` block (in the Integrate group) calls a Lambda function synchronously and makes the response available as contact attributes.

## Prerequisites

Before using Lambda in a flow, you must register the Lambda function with your Connect instance:

1. Go to Amazon Connect console > Your instance > AWS Lambda
2. Add the Lambda function ARN
3. This grants Connect permission to invoke the function (adds a resource-based policy)

Alternatively, use the `AssociateLambdaFunction` API.

## The Invoke AWS Lambda Function Block

### Configuration

- **Function ARN**: Select from the list of registered Lambda functions.
- **Timeout**: Maximum 8 seconds. If the Lambda does not respond within the timeout, the Error branch is taken.
- **Function input parameters**: Key-value pairs sent as additional parameters alongside the standard event payload.

### Branches

- **Success**: Lambda returned a valid response.
- **Error**: Lambda timed out, threw an error, returned invalid format, or response exceeded 32 KB.

## Event Payload

When Connect invokes your Lambda function, it sends the following JSON event:

```json
{
  "Details": {
    "ContactData": {
      "Attributes": {
        "customKey1": "customValue1",
        "customKey2": "customValue2"
      },
      "Channel": "VOICE",
      "ContactId": "abc12345-def6-7890-abcd-ef1234567890",
      "CustomerEndpoint": {
        "Address": "+12065551234",
        "Type": "TELEPHONE_NUMBER"
      },
      "InitialContactId": "abc12345-def6-7890-abcd-ef1234567890",
      "InitiationMethod": "INBOUND",
      "InstanceARN": "arn:aws:connect:us-east-1:123456789012:instance/instance-id",
      "Queue": {
        "ARN": "arn:aws:connect:us-east-1:123456789012:instance/instance-id/queue/queue-id",
        "Name": "BasicQueue"
      },
      "SystemEndpoint": {
        "Address": "+18005551234",
        "Type": "TELEPHONE_NUMBER"
      },
      "MediaStreams": {
        "Customer": {
          "Audio": {
            "StartFragmentNumber": "91343852333181432392682062622220590765191907586",
            "StartTimestamp": "1565781909613",
            "StreamARN": "arn:aws:kinesisvideo:us-east-1:123456789012:stream/connect-xxx/1234567890"
          }
        }
      },
      "References": {}
    },
    "Parameters": {
      "param1": "value1",
      "param2": "value2"
    }
  },
  "Name": "ContactFlowEvent"
}
```

### ContactData Fields

| Field | Description |
|-------|-------------|
| `Attributes` | User-defined contact attributes set earlier in the flow. |
| `Channel` | `VOICE`, `CHAT`, or `TASK`. |
| `ContactId` | Unique ID for this contact. |
| `CustomerEndpoint.Address` | Customer's phone number (E.164) or chat endpoint. |
| `CustomerEndpoint.Type` | `TELEPHONE_NUMBER` or `CHAT`. |
| `InitialContactId` | The ID of the original contact (same as ContactId for initial contacts; differs for transfers). |
| `InitiationMethod` | `INBOUND`, `OUTBOUND`, `TRANSFER`, `CALLBACK`, `QUEUE_TRANSFER`, `API`. |
| `InstanceARN` | ARN of the Connect instance. |
| `Queue` | Current queue (ARN and Name). Null if not yet queued. |
| `SystemEndpoint.Address` | The phone number the customer dialed (DID/TFN). |
| `MediaStreams` | Kinesis Video Stream details if media streaming is active. |
| `References` | References attached to the contact. |

### Parameters

The `Parameters` object contains any key-value pairs configured in the `Invoke AWS Lambda function` block's "Function input parameters" section. Use these to pass flow-specific context to your Lambda.

## Response Format

Lambda must return one of two formats:

### STRING_MAP (Flat Key-Value)

```javascript
exports.handler = async (event) => {
  const phoneNumber = event.Details.ContactData.CustomerEndpoint.Address;

  // Your business logic here
  const customer = await lookupCustomer(phoneNumber);

  // Return flat key-value pairs (all values must be strings)
  return {
    customerName: customer.name,
    accountId: customer.accountId,
    tier: customer.tier,
    balance: String(customer.balance)
  };
};
```

All values must be strings. Numbers and booleans must be converted to strings.

### JSON (Nested)

```javascript
exports.handler = async (event) => {
  const phoneNumber = event.Details.ContactData.CustomerEndpoint.Address;

  const customer = await lookupCustomer(phoneNumber);

  // Return nested JSON
  return {
    customerName: customer.name,
    accountId: customer.accountId,
    address: {
      street: customer.street,
      city: customer.city,
      state: customer.state
    },
    recentOrders: [
      { orderId: "123", status: "shipped" },
      { orderId: "456", status: "delivered" }
    ]
  };
};
```

Nested values are accessible via JSONPath: `$.External.address.city`, `$.External.recentOrders[0].status`.

### Response Limits

- **Maximum response size: 32 KB**. If the response exceeds this, the Error branch is taken.
- All top-level keys in STRING_MAP responses must be strings.
- The response must be valid JSON.

## Timeout and Retry Behavior

### Timeout

- **Maximum timeout per invocation: 8 seconds.** Configure in the block settings.
- **Total Lambda chain limit: 20 seconds.** If you invoke multiple Lambda functions in sequence within a single flow, the cumulative execution time must not exceed 20 seconds. After 20 seconds, subsequent Lambda invocations will fail.

### Retry

- Connect automatically retries Lambda invocations up to **3 times** on:
  - Throttling (429 / `TooManyRequestsException`)
  - Server errors (500-series)
- Retries count toward the 8-second timeout. If all retries exhaust the timeout, the Error branch is taken.
- Retries do NOT occur on client errors (4xx other than 429) or on responses that are simply too large.

## Accessing Lambda Response in the Flow

### Direct Access

After a successful Lambda invocation, response values are available as External attributes:

- `$.External.customerName`
- `$.External.accountId`
- `$.External.address.city` (for nested JSON responses)

Use these in:
- `Check contact attributes` blocks for branching
- `Play prompt` blocks for dynamic TTS: "Hello, $.External.customerName"
- Other Lambda invocations as parameters

### Saving to Contact Attributes

Use the `Set contact attributes` block to copy External attributes to user-defined contact attributes:

- Source type: External
- Source attribute: `customerName`
- Destination key: `customerName`

This is recommended if you need the values to persist beyond the immediate next block, or if you plan to invoke another Lambda (which overwrites `$.External`).

## Best Practices

### Play a Prompt Between Lambda Functions

If you chain multiple Lambda invocations, insert a `Play prompt` block between them:
- Provides feedback to the customer ("One moment while I look that up...")
- Helps stay within the 20-second total chain limit by giving each Lambda its own window
- Prevents silence on the line during processing

### Error Handling

Always connect the Error branch to a meaningful fallback:

```javascript
exports.handler = async (event) => {
  try {
    const result = await riskyOperation();
    return { status: "success", data: result };
  } catch (error) {
    console.error("Lambda error:", error);
    // Return an error indicator instead of throwing
    // This takes the Success branch but lets you branch on status
    return { status: "error", errorMessage: error.message };
  }
};
```

Alternatively, throw an error to trigger the Error branch:

```javascript
exports.handler = async (event) => {
  const result = await riskyOperation();
  if (!result) {
    throw new Error("Customer not found");
  }
  return { customerName: result.name };
};
```

### Keep Lambda Functions Fast

- Connect flows are real-time voice interactions. Every second of Lambda execution is a second of silence (or prompt) for the customer.
- Aim for sub-1-second Lambda execution.
- Use provisioned concurrency for critical Lambdas to avoid cold starts.
- Cache frequently accessed data (DynamoDB DAX, ElastiCache, or in-memory caching for the Lambda execution environment).

### Logging

Log the incoming event for debugging:

```javascript
exports.handler = async (event) => {
  console.log("Connect event:", JSON.stringify(event, null, 2));

  const contactId = event.Details.ContactData.ContactId;
  const channel = event.Details.ContactData.Channel;
  const callerNumber = event.Details.ContactData.CustomerEndpoint.Address;
  const attributes = event.Details.ContactData.Attributes;
  const params = event.Details.Parameters;

  // Your logic here

  return { status: "ok" };
};
```

### Input Validation

Always validate incoming data:

```javascript
exports.handler = async (event) => {
  const contactData = event?.Details?.ContactData;
  if (!contactData) {
    throw new Error("Missing ContactData");
  }

  const phoneNumber = contactData.CustomerEndpoint?.Address;
  if (!phoneNumber) {
    return { status: "error", reason: "no_phone_number" };
  }

  // Continue with validated data
};
```

## Common Patterns

### Customer Lookup

```javascript
const { DynamoDBClient } = require("@aws-sdk/client-dynamodb");
const { DynamoDBDocumentClient, QueryCommand } = require("@aws-sdk/lib-dynamodb");

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

exports.handler = async (event) => {
  const phoneNumber = event.Details.ContactData.CustomerEndpoint.Address;

  const result = await docClient.send(new QueryCommand({
    TableName: "Customers",
    IndexName: "phone-index",
    KeyConditionExpression: "phoneNumber = :phone",
    ExpressionAttributeValues: { ":phone": phoneNumber }
  }));

  if (result.Items && result.Items.length > 0) {
    const customer = result.Items[0];
    return {
      found: "true",
      customerName: customer.name,
      accountId: customer.accountId,
      tier: customer.tier || "standard"
    };
  }

  return { found: "false" };
};
```

### Hours Override Check

```javascript
const { DynamoDBClient } = require("@aws-sdk/client-dynamodb");
const { DynamoDBDocumentClient, GetCommand } = require("@aws-sdk/lib-dynamodb");

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

exports.handler = async (event) => {
  const result = await docClient.send(new GetCommand({
    TableName: "SystemConfig",
    Key: { configKey: "hoursOverride" }
  }));

  const override = result.Item;
  if (override && override.active) {
    return {
      overrideActive: "true",
      message: override.message || "We are currently closed for a scheduled maintenance."
    };
  }

  return { overrideActive: "false" };
};
```

### CRM Integration

```javascript
exports.handler = async (event) => {
  const phoneNumber = event.Details.ContactData.CustomerEndpoint.Address;
  const contactId = event.Details.ContactData.ContactId;

  const response = await fetch("https://api.crm.example.com/contacts/search", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${process.env.CRM_API_KEY}`
    },
    body: JSON.stringify({ phone: phoneNumber })
  });

  if (!response.ok) {
    return { found: "false" };
  }

  const data = await response.json();

  if (data.contacts && data.contacts.length > 0) {
    const contact = data.contacts[0];
    return {
      found: "true",
      crmId: contact.id,
      customerName: `${contact.firstName} ${contact.lastName}`,
      openCases: String(contact.openCaseCount),
      lastInteraction: contact.lastInteractionDate
    };
  }

  return { found: "false" };
};
```
