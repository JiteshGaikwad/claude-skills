# Amazon Connect Flow Language (JSON DSL)

Version: `2019-10-30`

The Flow Language is the programmatic JSON format that defines Amazon Connect contact flows. Every flow exported from the console or created via API uses this format.

## Top-Level Structure

```json
{
  "Version": "2019-10-30",
  "StartAction": "<identifier-of-first-action>",
  "Metadata": {
    "entryPointPosition": { "x": 20, "y": 20 },
    "ActionMetadata": {}
  },
  "Actions": []
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `Version` | Yes | Always `"2019-10-30"`. |
| `StartAction` | Yes | The `Identifier` of the first action to execute. |
| `Metadata` | Yes | Visual metadata for the flow designer (positions, annotations). Does not affect runtime behavior. |
| `Actions` | Yes | Array of action objects. Maximum **250 actions** per flow. |

## Action Structure

Each action in the `Actions` array has the following shape:

```json
{
  "Identifier": "unique-id-here",
  "Type": "ActionTypeName",
  "Parameters": {},
  "Transitions": {
    "NextAction": "next-action-id",
    "Errors": [],
    "Conditions": []
  }
}
```

### Action Fields

| Field | Required | Description |
|-------|----------|-------------|
| `Identifier` | Yes | Unique identifier within the flow. Max 50 characters. Must be unique across all actions in the flow. |
| `Type` | Yes | The action type (see Action Types below). |
| `Parameters` | Yes | Configuration for the action. Structure varies by type. |
| `Transitions` | Yes | Defines where to go next. Contains `NextAction`, `Errors`, and `Conditions`. |

### Transitions

```json
{
  "Transitions": {
    "NextAction": "default-next-action-id",
    "Errors": [
      {
        "NextAction": "error-handler-id",
        "ErrorType": "NoMatchingError"
      }
    ],
    "Conditions": [
      {
        "NextAction": "conditional-next-id",
        "Condition": {
          "Operator": "Equals",
          "Operands": ["value"]
        }
      }
    ]
  }
}
```

- **NextAction**: The default next action if no conditions match and no error occurs.
- **Errors**: Array of error transitions. Each has an `ErrorType` and `NextAction`.
- **Conditions**: Array of conditional transitions. Evaluated in order; first match wins.

## Condition Operators

8 operators are available for condition evaluation:

| Operator | Description | Example |
|----------|-------------|---------|
| `Equals` | Exact string/number match | `"Operands": ["Sales"]` |
| `TextStartsWith` | String starts with value | `"Operands": ["re-"]` |
| `TextEndsWith` | String ends with value | `"Operands": [".com"]` |
| `TextContains` | String contains value | `"Operands": ["urgent"]` |
| `NumberGreaterThan` | Numeric greater than | `"Operands": ["100"]` |
| `NumberGreaterOrEqualTo` | Numeric greater than or equal | `"Operands": ["100"]` |
| `NumberLessThan` | Numeric less than | `"Operands": ["50"]` |
| `NumberLessOrEqualTo` | Numeric less than or equal | `"Operands": ["50"]` |

### Nesting Conditions

Conditions can be nested for complex logic:
- Maximum nesting depth: **5 levels**
- Maximum sub-conditions per condition: **50**

## Action Types by Category

### Contact Actions (27)

Actions that modify or act upon the contact.

| Type | Description |
|------|-------------|
| `CompleteOutboundCall` | Completes an outbound call after it connects. |
| `CreateCase` | Creates a new case in Amazon Connect Cases. |
| `CreateTask` | Creates a new task contact with specified attributes. |
| `CreateWisdomSession` | Creates an Amazon Q in Connect (formerly Wisdom) session. |
| `DequeueContactAndTransferToQueue` | Removes contact from current queue and transfers to another. |
| `EndFlowModuleExecution` | Ends a flow module and returns to the calling flow. |
| `GetCase` | Retrieves case data from Amazon Connect Cases. |
| `InvokeFlowModule` | Invokes a flow module (sub-flow). |
| `StartOutboundChatContact` | Initiates an outbound chat contact. |
| `TagContact` | Adds tags to the contact. |
| `TransferContactToAgent` | Transfers the contact directly to a specific agent. |
| `TransferContactToQueue` | Transfers the contact to the working queue. |
| `UnTagContact` | Removes tags from the contact. |
| `UpdateCase` | Updates an existing case. |
| `UpdateContactAttributes` | Sets user-defined contact attributes. |
| `UpdateContactCallbackNumber` | Sets or overrides the callback number. |
| `UpdateContactData` | Updates contact data fields. |
| `UpdateContactEventHooks` | Sets event flows (disconnect, hold, etc.). |
| `UpdateContactMediaProcessing` | Configures media processing (Contact Lens, etc.). |
| `UpdateContactMediaStreamingBehavior` | Starts or stops Kinesis Video Stream media streaming. |
| `UpdateContactRecordingAndAnalyticsBehavior` | Configures recording and Contact Lens analytics. |
| `UpdateContactRecordingBehavior` | Configures call recording (agent/customer/both). |
| `ResumeContact` | Resumes a paused contact. |
| `UpdateContactRoutingBehavior` | Sets routing criteria (attribute-based routing). |
| `UpdateContactTargetQueue` | Sets the target queue for the contact. |
| `UpdateContactTextToSpeechVoice` | Sets the Polly voice and language. |
| `UpdatePreviousContactParticipantState` | Updates the state of a participant in the previous contact. |

### Flow Control (15)

Actions that control flow logic, branching, and execution.

| Type | Description |
|------|-------------|
| `CheckHoursOfOperation` | Checks if current time is within hours of operation. |
| `CheckMetricData` | Branches based on queue metric values. |
| `CheckOutboundCallStatus` | Checks the progress of an outbound call. |
| `CheckVoiceId` | Checks Voice ID authentication/enrollment status. |
| `Compare` | Evaluates conditions against contact attributes. |
| `DistributeByPercentage` | Routes contacts by percentage distribution. |
| `EndFlowExecution` | Ends the flow without disconnecting. |
| `GetMetricData` | Retrieves real-time queue metrics. |
| `Loop` | Repeats a section of the flow N times. |
| `StartVoiceIdStream` | Starts Voice ID streaming for authentication. |
| `TransferToFlow` | Transfers execution to another flow. |
| `UpdateFlowAttributes` | Updates flow-level attributes (not contact attributes). |
| `UpdateFlowLoggingBehavior` | Enables or disables flow logging. |
| `UpdateRoutingCriteria` | Sets routing criteria for attribute-based routing. |
| `Wait` | Pauses execution for a duration or until an event. |

### Interactions (8)

Actions that interact with external services or customer data.

| Type | Description |
|------|-------------|
| `AssociateContactToCustomerProfile` | Links the contact to a customer profile. |
| `CreateCallbackContact` | Creates a queued callback contact. |
| `CreateCustomerProfile` | Creates a new customer profile. |
| `InvokeLambdaFunction` | Invokes an AWS Lambda function. |
| `GetCustomerProfile` | Retrieves a customer profile. |
| `GetCustomerProfileObject` | Retrieves a specific object from a customer profile. |
| `GetCalculatedAttributesForCustomerProfile` | Gets calculated attributes (e.g., total calls in 30 days). |
| `UpdateCustomerProfile` | Updates an existing customer profile. |

### Participant (6)

Actions that interact with participants (customer, agent, bot).

| Type | Description |
|------|-------------|
| `ConnectParticipantWithLexBot` | Connects the customer to an Amazon Lex bot. |
| `DisconnectParticipant` | Disconnects the participant (hangs up). |
| `GetParticipantInput` | Collects DTMF or Lex input from the participant. |
| `MessageParticipant` | Plays a prompt or sends a message to the participant. |
| `MessageParticipantIteratively` | Plays prompts in a loop (queue flow). |
| `ShowView` | Displays a UI view to the agent in the workspace. |

## Example Flow

A simple flow that plays a greeting, invokes a Lambda function, and routes based on the result.

```json
{
  "Version": "2019-10-30",
  "StartAction": "greeting",
  "Metadata": {
    "entryPointPosition": { "x": 20, "y": 20 },
    "ActionMetadata": {
      "greeting": { "position": { "x": 200, "y": 100 } },
      "lookup": { "position": { "x": 200, "y": 300 } },
      "check-result": { "position": { "x": 200, "y": 500 } },
      "transfer-vip": { "position": { "x": 50, "y": 700 } },
      "transfer-standard": { "position": { "x": 350, "y": 700 } },
      "disconnect": { "position": { "x": 200, "y": 900 } }
    }
  },
  "Actions": [
    {
      "Identifier": "greeting",
      "Type": "MessageParticipant",
      "Parameters": {
        "Text": "Thank you for calling. Please hold while we look up your account."
      },
      "Transitions": {
        "NextAction": "lookup",
        "Errors": [
          { "NextAction": "disconnect", "ErrorType": "NoMatchingError" }
        ],
        "Conditions": []
      }
    },
    {
      "Identifier": "lookup",
      "Type": "InvokeLambdaFunction",
      "Parameters": {
        "LambdaFunctionARN": "arn:aws:lambda:us-east-1:123456789012:function:CustomerLookup",
        "InvocationTimeLimitSeconds": "8"
      },
      "Transitions": {
        "NextAction": "check-result",
        "Errors": [
          { "NextAction": "transfer-standard", "ErrorType": "NoMatchingError" }
        ],
        "Conditions": []
      }
    },
    {
      "Identifier": "check-result",
      "Type": "Compare",
      "Parameters": {
        "ComparisonValue": "$.External.customerTier"
      },
      "Transitions": {
        "NextAction": "transfer-standard",
        "Errors": [
          { "NextAction": "transfer-standard", "ErrorType": "NoMatchingError" }
        ],
        "Conditions": [
          {
            "NextAction": "transfer-vip",
            "Condition": {
              "Operator": "Equals",
              "Operands": ["VIP"]
            }
          }
        ]
      }
    },
    {
      "Identifier": "transfer-vip",
      "Type": "UpdateContactTargetQueue",
      "Parameters": {
        "QueueId": "arn:aws:connect:us-east-1:123456789012:instance/xxx/queue/vip-queue-id"
      },
      "Transitions": {
        "NextAction": "do-transfer-vip",
        "Errors": [
          { "NextAction": "transfer-standard", "ErrorType": "NoMatchingError" }
        ],
        "Conditions": []
      }
    },
    {
      "Identifier": "do-transfer-vip",
      "Type": "TransferContactToQueue",
      "Parameters": {},
      "Transitions": {
        "NextAction": "disconnect",
        "Errors": [
          { "NextAction": "disconnect", "ErrorType": "QueueAtCapacity" }
        ],
        "Conditions": []
      }
    },
    {
      "Identifier": "transfer-standard",
      "Type": "UpdateContactTargetQueue",
      "Parameters": {
        "QueueId": "arn:aws:connect:us-east-1:123456789012:instance/xxx/queue/standard-queue-id"
      },
      "Transitions": {
        "NextAction": "do-transfer-standard",
        "Errors": [
          { "NextAction": "disconnect", "ErrorType": "NoMatchingError" }
        ],
        "Conditions": []
      }
    },
    {
      "Identifier": "do-transfer-standard",
      "Type": "TransferContactToQueue",
      "Parameters": {},
      "Transitions": {
        "NextAction": "disconnect",
        "Errors": [
          { "NextAction": "disconnect", "ErrorType": "QueueAtCapacity" }
        ],
        "Conditions": []
      }
    },
    {
      "Identifier": "disconnect",
      "Type": "DisconnectParticipant",
      "Parameters": {},
      "Transitions": {
        "NextAction": "",
        "Errors": [],
        "Conditions": []
      }
    }
  ]
}
```

## Programmatic Flow Management

### Create a Flow via API

Use the `CreateContactFlow` API:
- `InstanceId`: Your Connect instance ID
- `Name`: Flow name
- `Type`: `CONTACT_FLOW`, `CUSTOMER_QUEUE`, `CUSTOMER_WHISPER`, `AGENT_WHISPER`, `OUTBOUND_WHISPER`, `AGENT_TRANSFER`, `QUEUE_TRANSFER`
- `Content`: The JSON flow definition as a string

### Update a Flow via API

Use `UpdateContactFlowContent`:
- `InstanceId`: Instance ID
- `ContactFlowId`: Flow ID
- `Content`: Updated JSON flow definition

### Validate Before Deploying

Always validate flow JSON before deploying. Common issues:
- Duplicate `Identifier` values
- `StartAction` pointing to a non-existent action
- Missing `Transitions` on actions
- ARN references that do not exist in the target instance
- Exceeding the 250-action limit
