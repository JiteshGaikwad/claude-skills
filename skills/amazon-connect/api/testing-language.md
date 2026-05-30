# Amazon Connect — Testing Language (Complete Reference)

**Version**: `2019-10-30`

The Testing language is a JSON-based representation of testing observations, events, and actions. Use it to:
- Create test cases for simulating your flows.
- Write test cases programmatically instead of using the visual test designer.

Reference order below: **Concepts → Top-level structure → Events → Actions → Observations → Operators → Full example**.

---

## 1. Concepts

The following terms are used in the Testing language.

### Observations
Observations represent each complete interaction that includes one observed event expected from the system and many actions to validate or simulate system behaviors.

### Events
Events represent expected behaviors that would come from the system, such as a prompt, a bot message, or a Lambda call.

### Actions
Actions represent what the testing framework should do in response to an event, such as sending DTMF, responding with text, asserting attribute values, or ending the test.

### Actors
Actors represent roles to be played in the testing framework.
- When **observing events**, actors can be the **System** or **Agent** — such as a play prompt coming from the system, or an agent accepting the call.
- When **simulating actions**, actors can be the **Customer**, **System**, or **Agent** — such as simulating a customer input DTMF or utterance, or simulating a system response from a Lambda function.

---

## 2. Top-level test structure

A test document has these top-level fields:

- **Version** — The API version for the testing language, such as `2019-10-30`.
- **Metadata** — Optional object containing UI-specific or non-functional (non-runtime-impacting) data.
- **Observations** — An array of observation objects that define the test flow.

### Observation object

Each observation consists of an event to observe and actions to execute when that event occurs.

- **Identifier** — Unique identifier for the observation.
- **Event** — Defines the expected event from the system to observe.
- **Actions** — Array of actions to execute when the event is observed.
- **Usage** — Defines how many times this observation should be matched.
  - **Type** — e.g. `"EXACTLY"` or `"ANY"`.
  - **Times** — Integer value for the count, when applicable.
- **Transitions** — Optional object defining flow control to the next observations.
  - **NextObservations** — Array of observation IDs to transition to.

Skeleton (verbatim from the Observations page):

```json
{
  "Version": "2019-10-30",
  "Metadata": { ... }, // Metadata to be used for data which is used for UI or any non-runtime impacting data as required.
  "Observations": [
    {
      "Identifier": "Unique identifer",
      "Event": { ... },
      "Actions": [
            {
                "Identifier": "ActionId",
                "Type": "ActionType", // Action type could be of any type mentioned in recap (ObserveEvent, SendInstruction, Assertion, OverrideSystemBehavior, EndTest)
                "Parameters": {...},
                "Transitions" : {...}
            },
            ...
        ],
      "Usage": { "Type": "ANY" },
      "Transitions" : {
        "NextObservations": ["string-id", "string-id", "string-id"]
      }
    }
    // Additional observations...
  ]
}
```

### Event object (general shape)
An `Event` inside an observation has: `Identifier`, `Type`, `Actor`, `Properties`, and (for message-style events) `MatchingCriteria`.

### Action object (general shape)
An `Action` has: `Identifier`, `Type`, `Parameters` (which contains `ActionType` and action-specific fields, sometimes `Actor`), and `Transitions` (`{ "NextAction": "<actionId or empty>" }`).

---

## 3. Events

Index text: "Events represent expected behaviors from the system that the test framework observes."

Event topics (canonical sub-pages):
- Test initiated — `testing-language-events-test-initiated.html`
- Test completed — `testing-language-events-test-completed.html`
- Message received — `testing-language-events-message-received.html`
- Flow action started — `testing-language-events-flow-action-started.html`

### 3.1 Test initiated (`TestInitiated`)

Triggered when the test execution begins. This is typically used to set up initial conditions such as override system behaviors before the actual flow execution starts.

**Parameters**
- **Identifier** — Unique identifier for the event. (API needs to specify this identifier in order for the UI to render properly.)
- **Type** — Must always be `TestInitiated`.
- **Actor** — Must always be `System`. This indicates that the event originates from the testing system.
- **Properties** — Empty object. No additional properties are required.

```json
{
    "Identifier": "unique identifier",
    "Type": "TestInitiated",
    "Actor": "System",
    "Properties": {}
}
```

### 3.2 Test completed (`TestCompleted`)

Triggered when the test execution ends. This event is observed when the test is terminated.

**Parameters**
- **Type** — Must always be `TestCompleted`.
- **Actor** — Must always be `System`. This indicates that the event originates from the testing system.
- **Properties** — Empty object. No additional properties are required.

```json
{
    "Identifier": "unique identifier",
    "Type": "TestCompleted",
    "Actor": "System",
    "Properties": {}
}
```

### 3.3 Message received (`MessageReceived`)

Observes when the system plays a prompt or sends any voice response to the simulated customer. This event can match messages using different criteria.

**Parameters**
- **Identifier** — Unique identifier for the event. (API needs to specify this identifier in order for the UI to render properly.)
- **Type** — Must always be `MessageReceived`.
- **Actor** — Must always be `System`. This indicates the event originates from the testing system.
- **Properties** — Object containing the following message details:
  - **PromptId** (Optional) — Specific prompt ID or ARN to match. A prompt ID or prompt ARN to play to the participant along with gathering input. May not be specified if `Text` or `SSML` is also specified. Must be specified either statically or as a single valid JSONPath identifier.
  - **Text** (Optional) — Text content of the message to match. An optional string that defines text to send to the participant along with gathering input. May not be specified if `PromptId` or `SSML` is also specified. May be specified statically or dynamically.
  - **SSML** (Optional) — SSML content to match. An optional string that defines SSML to send to the participant along with gathering input. May not be specified if `Text` or `PromptId` is also specified. May be specified statically or dynamically.
  - **Media** (Optional) — External media source details:
    - **Uri** — Location of the media file / message.
    - **SourceType** — Source from which media is fetched, such as S3. (The only supported type is S3.)
    - **MediaType** — Type of media such as `"audio/mpeg"`. (The only supported type is Audio.)
  - **MatchingCriteria** — Defines how to match the message:
    - **Similarity** — Uses semantic matching to find similar messages.
    - **Inclusion** — Checks if the observed message contains the specified text.

```json
{
    "Identifier": "unique identifier",
    "Type": "MessageReceived",
    "Actor": "System",
    "Properties": {
        "PromptId": "string", // [Optional] A prompt ID or prompt ARN to play to the participant along with gathering input. May not be specified if Text or SSML is also specified. Must be specified either statically or as a single valid JSONPath identifier.
        "Text":  "string", // An optional string that defines text to send to the participant along with gathering input. May not be specified if PromptId or SSML is also specified. May be specified statically or dynamically.
        "SSML": "string", // An optional string that defines SSML to send to the participant along with gathering input. May not be specified if Text or PromptId is also specified May be specified statically or dynamically.
        "Media": { // An optional object that defines an external media source
            "Uri": "string", // Location of the message
            "SourceType": "string",// The source from which the message will be fetched. The only supported type is S3
            "MediaType": "string"// The type of the message to be played. The only supported type is Audio
        },
        "MatchingCriteria": { Type: Similarity / Inclusion }
    }
}
```

### 3.4 Flow action started (`FlowActionStarted`)

Observes when a specific flow action begins execution. This allows you to detect when particular flow actions are executed during the simulation. The `Properties.ActionType` selects which flow action is observed. There are four documented variants.

#### 3.4.1 Invoke Lambda function
Observes when a Lambda function invocation action starts.

**Parameters**
- **Identifier** — Unique identifier for the event. (In preview; API needs to specify this identifier in order for the UI to render properly.)
- **Type** — Must always be `FlowActionStarted`.
- **Actor** — Must always be `System`.
- **Properties**:
  - **ActionType** — Must always be `InvokeLambdaFunction`.
  - **ActionParameters**:
    - **LambdaFunctionARN** — The ARN of the Lambda function being invoked.

```json
{
    "Identifier": "unique identifier",
    "Type": "FlowActionStarted",
    "Actor": "System",
    "Properties": {
        "ActionType" : "InvokeLambdaFunction " // V2 JSON action type
        "ActionParameters" : { // V2 JSON action parameters
            "LambdaFunctionARN": "string"
        }
}
```

#### 3.4.2 Check hours of operation
Observes when the flow checks hours of operation.

**Parameters**
- **Identifier** — Unique identifier for the event.
- **Type** — Must always be `FlowActionStarted`.
- **Actor** — Must always be `System`.
- **Properties**:
  - **ActionType** — Must always be `CheckHoursOfOperation`.
  - **ActionParameters**:
    - **HoursOfOperationId** — The ID or ARN of the hours of operation resource being checked.

```json
{
    "Identifier": "unique identifier",
    "Type": "FlowActionStarted",
    "Actor": "System",
    "Properties": {
        "ActionType" : "CheckHoursOfOperation" // V2 JSON action type
        "ActionParameters" : { // V2 JSON action parameters
            "HoursOfOperationId": "string"
        }
}
```

#### 3.4.3 Transfer contact to queue
Observes when a contact is being transferred to a queue.

**Parameters**
- **Identifier** — Unique identifier for the event.
- **Type** — Must always be `FlowActionStarted`.
- **Actor** — Must always be `System`.
- **Properties**:
  - **ActionType** — Must always be `TransferContactToQueue`.
  - **ActionParameters**:
    - **QueueId** — The ID or ARN of the target queue.
    - **AgentId** (Optional) — Specific agent ID if transferring to a particular agent.

```json
{
    "Identifier": "unique identifier",
    "Type": "FlowActionStarted",
    "Actor": "System",
    "Properties": {
        "ActionType" : "TransferContactToQueue" // V2 JSON action type
        "ActionParameters" : { // V2 JSON action parameters
            "QueueId": "string",
            "AgentId" : "string"
        }
}
```

#### 3.4.4 Connect participant with Lex bot
Observes when a participant is being connected to a Lex bot.

**Parameters**
- **Identifier** — Unique identifier for the event.
- **Type** — Must always be `FlowActionStarted`.
- **Actor** — Must always be `System`.
- **Properties**:
  - **ActionType** — Must always be `ConnectParticipantWithLexBot`.
  - **ActionParameters**:
    - **LexV2Bot** — Object containing bot details:
      - **AliasArn** — The ARN of the Lex V2 bot alias.

```json
{
    "Identifier": "unique identifier",
    "Type": "FlowActionStarted",
    "Actor": "System",
    "Properties": {
        "ActionType": "ConnectParticipantWithLexBot"
        "ActionParameters": {
            "LexV2Bot": {
                "AliasArn": "string"
            }
        }
    }
}
```

---

## 4. Actions

Index text: "Actions represent what the testing framework executes in response to observed events."

Action types: **SendInstruction**, **OverrideSystemBehavior**, **Assert** (used in example), **TestControl**.

### 4.1 Send Instruction (`SendInstruction`)

Simulates customer input during test execution. Sends DTMF tones, voice/text input, or a disconnect as if a customer were interacting with the flow.

#### 4.1.1 DTMF input
Simulates a customer pressing keys on their phone keypad.

**Parameters**
- **Identifier** — Unique identifier for the action.
- **Type** — Must be `SendInstruction`.
- **Parameters**:
  - **ActionType** — Must be `SendInstruction`.
  - **Actor** — `Customer` indicates this simulates customer behavior.
  - **Instruction** — Object defining the instruction type:
    - **Type** — Must be `DtmfInput`.
    - **Properties**:
      - **Value** — String or number representing the DTMF input (e.g., `"1"`, `"#"`, `"*"`).
- **Transitions**:
  - **NextAction** — The unique identifier for the next action.

```json
{
    "Identifier": "ActionId",
    "Type": "SendInstruction",
    "Parameters": {
        "ActionType": "SendInstruction",
        "Actor" : "Customer",
        "Instruction": {
            "Type": "DtmfInput",
            "Properties": {
                "Value": "string"
            }
        }
    },
    "Transitions": { "NextAction": "string" }
}
```

#### 4.1.2 Text and Utterance input
Simulates customer voice or text input. Used for Lex bot or AI agent interactions.

**Parameters**
- **Identifier** — Unique identifier for the action.
- **Type** — Must be `SendInstruction`.
- **Parameters**:
  - **ActionType** — Must be `SendInstruction`.
  - **Actor** — `Customer` indicates this simulates customer behavior.
  - **Instruction** — Object defining the instruction:
    - **Type** — Must be `TextUtterance`.
    - **Properties**:
      - **Text** (Optional) — Plain text input to send.
      - **SSML** (Optional) — SSML-formatted input to send.
      - **LanguageCode** — Language code for the input (e.g., `"en-US"`).
- **Transitions**:
  - **NextAction** — The unique identifier for the next action.

```json
{
    "Identifier": "ActionId",
    "Type": "SendInstruction",
    "Parameters": {
        "ActionType": "SendInstruction",
        "Actor" : "Customer",
        "Instruction": {
            "Type": "TextUtterance",
            "Properties": {
                "Text":  "string", // An optional string that defines text to send to the participant along with gathering input. May not be specified if PromptId or SSML is also specified. May be specified statically or dynamically.
                "SSML": "string", // An optional string that defines SSML to send to the participant along with gathering input. May not be specified if Text or PromptId is also specified May be specified statically or dynamically.,
                "LanguageCode": "en-US"
            }
        }
    },
    "Transitions": { "NextAction": "string" }
}
```

#### 4.1.3 Disconnect
Simulates customer ending the call.

**Parameters**
- **Identifier** — Unique identifier for the action.
- **Type** — Must be `SendInstruction`.
- **Parameters**:
  - **ActionType** — Must be `SendInstruction`.
  - **Actor** — `Customer` indicates this simulates customer behavior.
  - **Instruction** — Object defining the instruction:
    - **Type** — Must be `Disconnect`.
- **Transitions**:
  - **NextAction** — The unique identifier for the next action.

```json
{
    "Identifier": "ActionId",
    "Type": "SendInstruction",
    "Parameters": {
        "ActionType": "SendInstruction",
        "Actor" : "Customer",
        "Instruction": {
            "Type": "Disconnect",
        }
    },
    "Transitions": { "NextAction": "string" }
}
```

### 4.2 Override system behavior (`OverrideSystemBehavior`)

Modifies how specific flow actions behave during test execution. Lets you mock external dependencies, substitute resources, or simulate specific scenarios without modifying the actual flow. Each variant has a `Behavior.Type` of `FlowAction`, a `Properties.ActionType` selecting which flow action to override, and a `Strategy` of either `SubstituteResource` or `MockResponse`.

There are four flow-action targets: **InvokeLambdaFunction**, **CheckHoursOfOperation**, **ConnectParticipantWithLexBot**, **TransferContactToQueue**, plus **DequeueAndTransferToQueue**.

#### 4.2.1 InvokeLambdaFunction

**Substitute resource strategy** — Redirects Lambda invocations to a different function ARN.

Parameters:
- **Identifier** — Unique identifier for the action.
- **Type** — Must be `OverrideSystemBehavior`.
- **Parameters**:
  - **ActionType** — Must be `OverrideSystemBehavior`.
  - **Behavior**:
    - **Type** — Must be `FlowAction`.
    - **Properties**:
      - **ActionType** — Must be `InvokeLambdaFunction`.
      - **ActionParameters**: `LambdaFunctionARN` — The ARN of the Lambda function to override.
      - **Strategy**:
        - **Type** — Must be `SubstituteResource`.
        - **SubstituteArn** — ARN of the replacement Lambda function to use.
- **Transitions**: `NextAction` — The unique identifier for the next action.

```json
{
    "Identifier": "ActionId",
    "Type": "OverrideSystemBehavior",
    "Parameters": {
        "ActionType": "OverrideSystemBehavior",
        "Behavior": {
            "Type": "FlowAction",
            "Properties": {
                "ActionType" : "InvokeLambdaFunction",
                "ActionParameters": {
                    "LambdaFunctionARN" : "string"
                },
                "Strategy": {
                    "Type": "SubstituteResource",
                    "SubstituteArn": "string"
                }
            }
        }
    },
    "Transitions": { "NextAction": "string" }
}
```

**Mock response strategy — Success**

```json
{
    "Identifier": "ActionId",
    "Type": "OverrideSystemBehavior",
    "Parameters": {
        "ActionType": "OverrideSystemBehavior",
        "Behavior": {
            "Type": "FlowAction",
            "Properties": {
                "ActionType" : "InvokeLambdaFunction",
                "ActionParameters": {
                    "LambdaFunctionARN" : "string"
                },
                "Strategy": {
                    "Type": "MockResponse",
                    "Response": {
                        "Type" : "ExecutionResult",
                        "ExecutionResult" : {
                            "DelaySeconds" : Number,
                            "LoadedData" : "serialized JSON"
                        }
                    }
                }
            }
        }
    },
    "Transitions": { "NextAction": "string" }
}
```

**Mock response strategy — Error** — Simulates a Lambda function error without actual invocation.

```json
{
    "Identifier": "ActionId",
    "Type": "OverrideSystemBehavior",
    "Parameters": {
        "ActionType": "OverrideSystemBehavior",
        "Behavior": {
            "Type": "FlowAction",
            "Properties": {
                "ActionType" : "InvokeLambdaFunction",
                "ActionParameters": {
                    "LambdaFunctionARN" : "lambda-arn-to-mock"
                },
                "Strategy": {
                    "Type": "MockResponse",
                    "Response": {
                        "Type" : "Error",
                        "Error" : {
                            "DelaySeconds" :  Number,
                            "Value" : "TimeLimitExceeded|NoMatchingError"
                        }
                    }
                }
            }
        }
    },
    "Transitions": { "NextAction": "string" }
}
```

#### 4.2.2 CheckHoursOfOperation
Override hours of operation checks to test different time-based scenarios.

**Substitute resource strategy** — Redirects hours of operation checks to a different hours of operation configuration.

Parameters:
- **Identifier** — Unique identifier for the action.
- **Type** — Must be `OverrideSystemBehavior`.
- **Parameters**:
  - **ActionType** — Must be `OverrideSystemBehavior`.
  - **Behavior**:
    - **Type** — Must be `FlowAction`.
    - **Properties**:
      - **ActionType** — Must be `CheckHoursOfOperation`.
      - **ActionParameters**: `HoursOfOperationId` — The ID/ARN of the hours of operation to override.
      - **Strategy**: `Type` = `SubstituteResource`; `SubstituteArn` — ARN of the replacement hours of operation resource.
- **Transitions**: `NextAction`.

```json
{
    "Identifier": "ActionId",
    "Type": "OverrideSystemBehavior",
    "Parameters": {
        "ActionType": "OverrideSystemBehavior",
        "Behavior": {
            "Type": "FlowAction",
            "Properties": {
                "ActionType" : "CheckHoursOfOperation",
                "ActionParameters": {
                    "HoursOfOperationId" : "string"
                },
                "Strategy": {
                    "Type": "SubstituteResource",
                    "SubstituteArn": "string"
                }
            }
        }
    },
    "Transitions": { "NextAction": "string" }
}
```

**Mock response strategy — Success** — Returns a predefined hours of operation check result. The `ExecutionResult.Value` is `InHours|OutOfHours`.

```json
{
    "Identifier": "ActionId",
    "Type": "OverrideSystemBehavior",
    "Parameters": {
        "ActionType": "OverrideSystemBehavior",
        "Behavior": {
            "Type": "FlowAction",
            "Properties": {
                "ActionType": "CheckHoursOfOperation",
                "ActionParameters": {
                    "HoursOfOperationId": "string"
                },
                "Strategy": {
                    "Type": "MockResponse",
                    "Response": {
                        "Type": "ExecutionResult",
                        "ExecutionResult": {
                            "Value": "InHours|OutOfHours"
                        }
                    }
                }
            }
        },
        "Transitions": {
            "NextAction": "string"
        }
    }
}
```

**Mock response strategy — Error** — Simulates an error during hours of operation check. The `Error.Value` is `NoMatchingError`.

```json
{
    "Identifier": "ActionId",
    "Type": "OverrideSystemBehavior",
    "Parameters": {
        "ActionType": "OverrideSystemBehavior",
        "Behavior": {
            "Type": "FlowAction",
            "Properties": {
                "ActionType": "CheckHoursOfOperation",
                "ActionParameters": {
                    "HoursOfOperationId": "arn:aws:connect:us-east-1:111122223333:instance/INSTANCE_ID/operating-hours/HOURS_ID"
                },
                "Strategy": {
                    "Type": "MockResponse",
                    "Response": {
                        "Type": "Error",
                        "Error": {
                            "Value": "NoMatchingError"
                        }
                    }
                }
            }
        }
    },
    "Transitions": {
        "NextAction": "string"
    }
}
```

#### 4.2.3 ConnectParticipantWithLexBot
Override Lex bot behaviors to use a different bot for testing or mock responses.

**Substitute resource strategy**

Parameters:
- **Identifier**, **Type** = `OverrideSystemBehavior`.
- **Parameters.ActionType** = `OverrideSystemBehavior`.
- **Behavior.Type** = `FlowAction`.
- **Properties.ActionType** = `ConnectParticipantWithLexBot`.
- **ActionParameters.LexV2Bot.AliasArn** — ARN of the Lex bot alias to override.
- **Strategy.Type** = `SubstituteResource`; **SubstituteArn** — ARN of the replacement Lex bot alias.
- **Transitions.NextAction**.

```json
{
    "Identifier": "ActionId",
    "Type": "OverrideSystemBehavior",
    "Parameters": {
        "ActionType": "OverrideSystemBehavior",
        "Behavior": {
            "Type": "FlowAction",
            "Properties": {
                "ActionType" : "ConnectParticipantWithLexBot",
                "ActionParameters" : {
                    "LexV2Bot": {
                       "AliasArn": "string"
                    }
                },
                "Strategy": {
                    "Type": "SubstituteResource",
                    "SubstituteArn": "string"
                }
             }
         }
    },
    "Transitions": { "NextAction": "string" }
}
```

**Mock response strategy — Success** — Returns a predefined successful response without invoking the actual Lex bot.

```json
{
    "Identifier": "ActionId",
    "Type": "OverrideSystemBehavior",
    "Parameters": {
        "ActionType": "OverrideSystemBehavior",
        "Behavior": {
            "Type": "FlowAction",
            "Properties": {
                "ActionType": "ConnectParticipantWithLexBot",
                "ActionParameters": {
                    "LexV2Bot": {
                        "AliasArn": "string"
                    }
                },
                "Strategy": {
                    "Type": "MockResponse",
                    "Response": {
                        "Type": "ExecutionResult",
                        "ExecutionResult": {
                            "DelaySeconds": Number,
                            "LoadedData": "serialized JSON"
                        }
                    }
                }
            }
        }
    },
    "Transitions": { "NextAction": "string" }
}
```

**Mock response strategy — Error** — Simulates a Lex bot error without actual invocation. `Error.Value` is `TimeLimitExceeded|NoMatchingError`.

```json
{
    "Identifier": "ActionId",
    "Type": "OverrideSystemBehavior",
    "Parameters": {
        "ActionType": "OverrideSystemBehavior",
        "Behavior": {
            "Type": "FlowAction",
            "Properties": {
                "ActionType": "ConnectParticipantWithLexBot",
                "ActionParameters": {
                    "LexV2Bot": {"AliasArn": "string"}
                },
                "Strategy": {
                    "Type": "MockResponse",
                    "Response": {
                        "Type": "Error",
                        "Error": {
                            "DelaySeconds": Number,
                            "Value": "TimeLimitExceeded|NoMatchingError"
                        }
                    }
                }
            }
        }
    },
    "Transitions": { "NextAction": "string" }
}
```

#### 4.2.4 TransferContactToQueue
Override queue transfer behavior to substitute queues or simulate transfer failures.

**Substitute resource strategy** — Redirects queue transfers to a different queue.

Parameters:
- **Identifier**, **Type** = `OverrideSystemBehavior`.
- **Parameters.ActionType** = `OverrideSystemBehavior`.
- **Behavior.Type** = `FlowAction`.
- **Properties.ActionType** = `TransferContactToQueue`.
- **ActionParameters.QueueId** — ID/ARN of the queue to override.
- **Strategy.Type** = `SubstituteResource`; **SubstituteArn** — ARN of the replacement queue.
- **Transitions.NextAction**.

```json
{
    "Identifier": "ActionId",
    "Type": "OverrideSystemBehavior",
    "Parameters": {
        "ActionType": "OverrideSystemBehavior",
        "Behavior": {
            "Type": "FlowAction",
            "Properties": {
                "ActionType" : "TransferContactToQueue",
                "ActionParameters" : {
                    "QueueId": "string"
                },
                "Strategy": {
                    "Type": "SubstituteResource",
                    "SubstituteArn": "string"
                }
             }
         }
    },
    "Transitions": { "NextAction": "string" }
}
```

**Mock response strategy — Error** — Simulates queue transfer failures for testing error paths. `Error.Value` is `QueueAtCapacity|NoMatchingError`.

```json
{
    "Identifier": "ActionId",
    "Type": "OverrideSystemBehavior",
    "Parameters": {
        "ActionType": "OverrideSystemBehavior",
        "Behavior": {
            "Type": "FlowAction",
            "Properties": {
                "ActionType": "TransferContactToQueue",
                "ActionParameters": {
                    "QueueId": "string"
                },
                "Strategy": {
                    "Type": "MockResponse",
                    "Response": {
                        "Type": "Error",
                        "Error": {
                            "Value": "QueueAtCapacity|NoMatchingError"
                        }
                    }
                }
            }
        }
    },
    "Transitions": {
        "NextAction": "string"
    }
}
```

#### 4.2.5 DequeueAndTransferToQueue
Override behavior when dequeuing a contact and transferring to another queue.

**Substitute resource strategy**

Parameters:
- **Identifier**, **Type** = `OverrideSystemBehavior`.
- **Parameters.ActionType** = `OverrideSystemBehavior`.
- **Behavior.Type** = `FlowAction`.
- **Properties.ActionType** = `DequeueAndTransferToQueue`.
- **ActionParameters.QueueId** — ID/ARN of the queue to override.
- **Strategy.Type** = `SubstituteResource`; **SubstituteArn** — ARN of the replacement queue.
- **Transitions.NextAction**.

```json
{
    "Identifier": "ActionId",
    "Type": "OverrideSystemBehavior",
    "Parameters": {
        "ActionType": "OverrideSystemBehavior",
        "Behavior": {
            "Type": "FlowAction",
            "Properties": {
                "ActionType" : "DequeueAndTransferToQueue",
                "ActionParameters" : {
                    "QueueId": "string"
                },
                "Strategy": {
                    "Type": "SubstituteResource",
                    "SubstituteArn": "string"
                }
             }
         }
    },
    "Transitions": { "NextAction": "string" }
}
```

**Mock response strategy — Error** — Simulates dequeue and transfer failures. `Error.Value` is `QueueAtCapacity|NoMatchingError`.

```json
{
    "Identifier": "ActionId",
    "Type": "OverrideSystemBehavior",
    "Parameters": {
        "ActionType": "OverrideSystemBehavior",
        "Behavior": {
            "Type": "FlowAction",
            "Properties": {
                "ActionType": "DequeueAndTransferToQueue",
                "ActionParameters": {
                    "QueueId": "string"
                },
                "Strategy": {
                    "Type": "MockResponse",
                    "Response": {
                        "Type": "Error",
                        "Error": {
                            "Value": "QueueAtCapacity|NoMatchingError"
                        }
                    }
                }
            }
        }
    },
    "Transitions": {
        "NextAction": "string"
    }
}
```

### 4.3 Assert (`Assert`)

Validates that a runtime attribute equals/compares to an expected value. (Shown in the worked example; not given a standalone documented page in the fetched set.)

**Parameters** (from the example):
- **Namespace** — JSONPath-style reference to the runtime value, e.g. `"$.Queue.Name"`.
- **Operator** — comparison operator, e.g. `"Equals"`.
- **Operand** — the expected value, e.g. `"Basic Queue"`.

```json
{
    "Identifier": "assertqueue",
    "Type": "Assert",
    "Parameters": {
        "Namespace": "$.Queue.Name",
        "Operator": "Equals",
        "Operand": "Basic Queue"
    },
    "Transitions": {
        "NextAction": "endthetest"
    }
}
```

### 4.4 Test Control (`TestControl`)

Special actions that control test execution flow and debugging. Two commands: **EndTest** and **LogData**.

#### 4.4.1 End Test
Explicitly terminates the test execution. Use this when you want to end the test at a specific point rather than waiting for natural completion.

**Parameters**
- **Identifier** — Unique identifier for the action.
- **Type** — Must be `TestControl`.
- **Parameters**:
  - **ActionType** — Must be `TestControl`.
  - **Command** — Object defining the control command:
    - **Type** — Must be `EndTest`.
- **Transitions**: `NextAction` — The unique identifier for the next action.

```json
{
    "Identifier": "ActionId",
    "Type": "TestControl",
    "Parameters": {
        "ActionType": "TestControl"
        "Command": {
           "Type": "EndTest"
        }
    },
    "Transitions": { "NextAction": "string" }
}
```

#### 4.4.2 Log Data
Captures and logs specific attribute values during test execution for debugging and validation purposes. The logged data appears in the test execution results.

**Parameters**
- **Identifier** — Unique identifier for the action.
- **Type** — Must be `TestControl`.
- **Parameters**:
  - **ActionType** — Must be `TestControl`.
  - **Command** — Object defining the control command:
    - **Type** — Must be `LogData`.
    - **Properties**:
      - **Expressions** — Object containing key-value pairs where:
        - **Key** — A descriptive label for the logged value.
        - **Value** — JSON path to extract the attributes (e.g., `"$.Queue.Name"`).
- **Transitions**: `NextAction` — The unique identifier for the next action.

```json
{
    "Identifier": "ActionId",
    "Type": "TestControl",
    "Parameters": {
        "ActionType": "TestControl",
        "Command": {
            "Type": "LogData",
            "Properties": {
                "Expressions": {
                    "string":"string", //"myContactId": "$.ContactId"
                ...
                }
            }
        }
    },
    "Transitions": { "NextAction": "string" }
}
```

---

## 5. Observations (detail)

The Observations page defines the top-level parameters (`Version`, `Metadata`, `Observations`) and the observation object shape (see Section 2). Key points:

- Each **observation object** = one observed `Event` + an `Actions` array to run when that event is observed, plus `Usage` (match count) and `Transitions` (`NextObservations`).
- **Usage.Type** values: `"EXACTLY"`, `"ANY"`. `Usage.Times` is an integer count used when applicable.
- **Transitions.NextObservations** is an array of observation `Identifier`s controlling which observation(s) the test may match next.

### Matching criteria (for message events)
For `MessageReceived`, `MatchingCriteria` controls how the observed prompt is compared to the expected `Text`/`SSML`/`PromptId`:
- **Similarity** — semantic match.
- **Inclusion** — substring/contains match.

---

## 6. Operators

The only comparison operator shown in the AWS documentation example is **`Equals`** (used by the `Assert` action via `Operator: "Equals"`). The documented pages in this guide section do not enumerate a broader operator set (e.g. NotEquals/Contains/GreaterThan/LessThan); only `Equals` appears. Error-value enumerations used by `MockResponse` strategies are:
- Lambda / Lex: `TimeLimitExceeded | NoMatchingError`
- Hours of operation: `NoMatchingError` (success value: `InHours | OutOfHours`)
- Queue transfer / dequeue: `QueueAtCapacity | NoMatchingError`

---

## 7. Full Example Test (verbatim)

This example mocks the hours of operation when the test is initiated, validates that a welcome prompt is played, simulates a contact's DTMF input to be placed in a queue and connected to an agent, then verifies the queue placement before concluding. It consists of three interactions with three simulated actions.

```json
{
    "Version": "2019-10-30",
    "Metadata": {},
    "Observations": [
        {
            "Identifier": "TriggerHoursCheck",
            "Event": {
                "Identifier": "Unique identifer"
                "Type": "TestInitiated",
                "Actor": "System",
                "Properties": {}
            },
            "Usage": {
                "Type": "EXACTLY"
            },
            "Actions": [
                {
                    "Identifier": "ActionId",
                    "Type": "OverrideSystemBehavior",
                    "Parameters": {
                        "ActionType": "OverrideSystemBehavior",
                        "Behavior": {
                            "Type": "FlowAction",
                            "Properties": {
                                "ActionType": "CheckHoursOfOperation",
                                "ActionParameters": {
                                    "HoursOfOperationId": "arn:aws:connect:us-west-2:123456789012:instance/abc123/flow/BasicHours"
                                },
                                "Strategy": {
                                    "Type": "SubstituteResource",
                                    "SubstituteArn": "arn:aws:connect:us-west-2:123456789012:instance/abc123/flow/AlwaysOnHours"
                                }
                            }
                        }
                    },
                    "Transitions": {
                        "NextAction": ""
                    }
                }
            ],
            "Transitions": {
                "NextObservations": [
                    "Welcome Message"
                ]
            }
        },
        {
            "Identifier": "Welcome Message",
            "Event": {
                "Identifier": "Unique identifer"
                "Type": "MessageReceived",
                "Actor": "System",
                "Properties": {
                    "Text": "Press 1 to be connected to an agent"
                },
                "MatchingCriteria": "Similarity"
            },
            "Usage": {
                "Type": "EXACTLY"
            },
            "Actions": [
