# Amazon Connect Rules Function Language Reference

**Version**: `2022-11-25`

The Rules Function Language is a JSON DSL used to define programmatic conditions for Amazon Connect automation rules. Rules are created via the `CreateRule` API and trigger actions based on contact analytics, metrics, evaluations, and case events.

## Structure

```
RuleFunction (JSON string)
  └── Top-level Operator (AND / OR)
       └── Operands[] (array of conditions)
            └── Each operand: Operator + Operands (nested)
                 └── Leaf: ComparisonValue path + comparison value
```

Every rule function is a single JSON object with a top-level `AND` or `OR` operator containing an array of operand conditions.

### Limits

- A rule function **must start with an `AND` or `OR`** operator.
- Conditions may be **nested no more than 5 levels deep**.
- A single condition may contain **no more than 50 sub-conditions**, regardless of nesting depth.
- An operator's `Operands` list supports **up to 10 operands** (operator-dependent — some operators take exactly one operand).
- An optional **`Negate`** boolean on any condition inverts its result.
- Rule name pattern: `^[a-zA-Z0-9._-]{1,200}$`. `PublishStatus`: `DRAFT` or `PUBLISHED`. Up to 50 tags.

## Operators

| Operator | Description | Leaf/Branch |
|---|---|---|
| `AND` | All operands must be true | Branch |
| `OR` | At least one operand must be true | Branch |
| `CONTAINS_ANY` | Value contains any of the specified strings | Leaf |
| `EQUALS` | Value equals the specified string | Leaf |
| `NumberLessOrEqualTo` | Numeric value <= threshold | Leaf |
| `NumberGreaterOrEqualTo` | Numeric value >= threshold | Leaf |

## Trigger Event Sources

Rules are triggered by one of 17 event sources (complete `EventSourceName` enum, verbatim from the `CreateRule` API):

| Event Source | Description |
|---|---|
| `OnPostCallAnalysisAvailable` | Contact Lens post-call analysis is complete |
| `OnRealTimeCallAnalysisAvailable` | Contact Lens real-time voice analysis segment available |
| `OnRealTimeChatAnalysisAvailable` | Contact Lens real-time chat analysis segment available |
| `OnPostChatAnalysisAvailable` | Contact Lens post-chat analysis is complete |
| `OnEmailAnalysisAvailable` | Contact Lens email analysis is complete |
| `OnMetricDataUpdate` | A metric threshold is breached |
| `OnContactEvaluationSubmit` | An evaluation form is submitted |
| `OnCaseCreate` | A case is created |
| `OnCaseUpdate` | A case field is updated |
| `OnSlaBreach` | A case SLA is breached |
| `OnZendeskTicketCreate` | A Zendesk ticket is created (via integration) |
| `OnZendeskTicketStatusUpdate` | A Zendesk ticket status changes |
| `OnSalesforceCaseCreate` | A Salesforce case is created (via integration) |
| `OnAlertUpdate` | An alert is updated |
| `OnSchedulePublish` | A forecasting/scheduling schedule is published |
| `OnScheduleUpdate` | A schedule is updated |
| `OnScheduleTimeOffRequestActivity` | A time-off request activity occurs |

> Note: `OnZendeskTicketUpdate`, `OnSalesforceCaseUpdate`, and `OnPostEmailAnalysisAvailable` are NOT valid names — the correct names are `OnZendeskTicketStatusUpdate` and `OnEmailAnalysisAvailable`.

## ComparisonValue Paths

### Post-Call Analysis Paths (20+)

#### Transcript Matching

| Path | Type | Description |
|---|---|---|
| `$.ContactLens.PostCall.ExactMatch.Transcript` | String | Exact string match in transcript |
| `$.ContactLens.PostCall.SemanticMatch.Transcript` | String | Semantic/meaning-based match in transcript |
| `$.ContactLens.PostCall.SemanticMatch.Phrase` | String | Semantic match against a specific phrase |
| `$.ContactLens.PostCall.PatternMatch.Transcript` | PatternMatch | Pattern-based match (see PatternMatch operands) |

#### Sentiment

| Path | Type | Description |
|---|---|---|
| `$.ContactLens.PostCall.Sentiment.State` | String | Overall sentiment: `POSITIVE`, `NEGATIVE`, `NEUTRAL`, `MIXED` |
| `$.ContactLens.PostCall.Sentiment.OverallScore` | Number | Overall sentiment score (-5.0 to 5.0) |
| `$.ContactLens.PostCall.Sentiment.Score.Beginning` | Number | Sentiment score for the first quarter |
| `$.ContactLens.PostCall.Sentiment.Score.End` | Number | Sentiment score for the last quarter |

#### Talk Metrics

| Path | Type | Description |
|---|---|---|
| `$.ContactLens.PostCall.NonTalkTime.TotalTimeSecs` | Number | Total silence duration in seconds |
| `$.ContactLens.PostCall.NonTalkTime.LongestTimeSecs` | Number | Longest single silence period |
| `$.ContactLens.PostCall.Interruptions.Instances` | Number | Number of interruption instances |
| `$.ContactLens.PostCall.TalkTime.TotalTimeSecs` | Number | Total talk time in seconds |
| `$.ContactLens.PostCall.Loudness.HighestLoudnessScore` | Number | Peak loudness score |

#### Agent Metrics

| Path | Type | Description |
|---|---|---|
| `$.ContactLens.PostCall.Agent.CustomerHoldDurationSecs` | Number | Total hold duration |
| `$.ContactLens.PostCall.Agent.LongestHoldDurationSecs` | Number | Longest single hold |
| `$.ContactLens.PostCall.Agent.NumberOfHolds` | Number | Number of hold events |
| `$.ContactLens.PostCall.Agent.AgentInteractionDurationSecs` | Number | Agent interaction duration |
| `$.ContactLens.PostCall.Agent.AfterContactWorkDurationSecs` | Number | ACW duration |
| `$.ContactLens.PostCall.Agent.NonTalkTimePct` | Number | Percentage of non-talk time |
| `$.ContactLens.PostCall.Agent.CustomerHoldDurationPct` | Number | Hold time as percentage of call |
| `$.ContactLens.PostCall.Agent.RoutingProfile.ARN` | String | Agent's routing profile ARN |
| `$.ContactLens.PostCall.Agent.HierarchyGroup.ARN` | String | Agent's hierarchy group ARN |
| `$.ContactLens.PostCall.Agent.AgentId` | String | Agent user ID |

#### Queue & Contact Metadata

| Path | Type | Description |
|---|---|---|
| `$.ContactLens.PostCall.Queue.QueueId` | String | Queue ID the contact was routed to |
| `$.ContactLens.PostCall.InitiationMethod` | String | How the contact was initiated |
| `$.ContactLens.PostCall.DisconnectReason` | String | Why the contact disconnected |
| `$.ContactLens.PostCall.PotentialDisconnectIssue` | String | Potential disconnect issue detected |

#### Dynamic Attributes

| Path | Type | Description |
|---|---|---|
| `$.ContactLens.PostCall.ContactAttribute.{KEY}` | String | Contact attribute by key name |
| `$.ContactLens.PostCall.SegmentAttributes.UserDefined.{KEY}` | String | User-defined segment attribute |
| `$.ContactLens.PostCall.AiAgent.IdWithVersion` | String | AI agent ID and version |

## FilterClause

Filters narrow the scope of transcript analysis:

### ParticipantRole

Filter transcript matching to a specific participant:

```json
{
  "ParticipantRole": "CUSTOMER"
}
```

Values: `CUSTOMER`, `AGENT`, `ANY`

### PostCallContactPeriodSeconds

Limit analysis to the first or last N seconds of the call:

```json
{
  "PostCallContactPeriodSeconds": {
    "First": 60
  }
}
```

or

```json
{
  "PostCallContactPeriodSeconds": {
    "Last": 120
  }
}
```

### PatternMatchLanguageFilter

Restrict pattern matching to specific languages (11 supported):

```json
{
  "PatternMatchLanguageFilter": ["en-US", "es-US"]
}
```

Supported languages: `en-US`, `en-GB`, `en-AU`, `en-IN`, `es-US`, `fr-CA`, `fr-FR`, `de-DE`, `it-IT`, `ja-JP`, `pt-BR`

## PatternMatch Operands

PatternMatch supports four operand types:

### PLAIN

Simple string match:

```json
{
  "Operator": "CONTAINS_ANY",
  "Operands": [
    {
      "Type": "PLAIN",
      "Value": "cancel my account"
    }
  ]
}
```

### LIST

Array of strings (any match triggers):

```json
{
  "Operator": "CONTAINS_ANY",
  "Operands": [
    {
      "Type": "LIST",
      "Value": [
        { "Type": "PLAIN", "Value": "cancel" },
        { "Type": "PLAIN", "Value": "close my account" },
        { "Type": "PLAIN", "Value": "terminate service" }
      ]
    }
  ]
}
```

### PROXIMITY

Words within a specified distance of each other:

```json
{
  "Operator": "CONTAINS_ANY",
  "Operands": [
    {
      "Type": "PROXIMITY",
      "Value": {
        "Distance": 5,
        "IsWithin": true
      }
    }
  ]
}
```

- `Distance` — maximum number of words between the target words
- `IsWithin` — `true` if words must be within distance, `false` if they must be farther apart

### NUMERICAL

Numeric comparison within transcript:

```json
{
  "Operator": "CONTAINS_ANY",
  "Operands": [
    {
      "Type": "NUMERICAL",
      "Value": {
        "Decimal": 100.0
      }
    }
  ]
}
```

## Negate Flag

Any condition can be negated with the `Negate` flag:

```json
{
  "Operator": "CONTAINS_ANY",
  "Operands": [...],
  "Negate": true
}
```

This inverts the condition — true becomes false and vice versa.

## Complete Example

This rule triggers on post-call analysis when:
1. The customer mentioned "cancel" or "close account" AND
2. The customer sentiment score at the end was negative AND
3. The call had more than 2 holds

```json
{
  "Version": "2022-11-25",
  "Operator": "AND",
  "Operands": [
    {
      "Operator": "CONTAINS_ANY",
      "ComparisonValue": "$.ContactLens.PostCall.SemanticMatch.Transcript",
      "Operands": [
        { "Type": "PLAIN", "Value": "cancel my account" },
        { "Type": "PLAIN", "Value": "close my account" },
        { "Type": "PLAIN", "Value": "terminate my service" }
      ],
      "FilterClause": {
        "ParticipantRole": "CUSTOMER",
        "PostCallContactPeriodSeconds": {
          "Last": 300
        }
      }
    },
    {
      "Operator": "NumberLessOrEqualTo",
      "ComparisonValue": "$.ContactLens.PostCall.Sentiment.Score.End",
      "Operands": [-2.0]
    },
    {
      "Operator": "NumberGreaterOrEqualTo",
      "ComparisonValue": "$.ContactLens.PostCall.Agent.NumberOfHolds",
      "Operands": [3]
    }
  ]
}
```

## Using Rules via the SDK

```typescript
import { ConnectClient, CreateRuleCommand } from '@aws-sdk/client-connect';

const client = new ConnectClient({ region: 'us-east-1' });

await client.send(new CreateRuleCommand({
  InstanceId: 'instance-xxx',
  Name: 'churn-risk-detection',
  TriggerEventSource: {
    EventSourceName: 'OnPostCallAnalysisAvailable',
  },
  Function: JSON.stringify({
    Version: '2022-11-25',
    Operator: 'AND',
    Operands: [
      {
        Operator: 'CONTAINS_ANY',
        ComparisonValue: '$.ContactLens.PostCall.SemanticMatch.Transcript',
        Operands: [
          { Type: 'PLAIN', Value: 'cancel my account' },
          { Type: 'PLAIN', Value: 'switching to competitor' },
        ],
        FilterClause: {
          ParticipantRole: 'CUSTOMER',
        },
      },
      {
        Operator: 'NumberLessOrEqualTo',
        ComparisonValue: '$.ContactLens.PostCall.Sentiment.OverallScore',
        Operands: [-1.0],
      },
    ],
  }),
  Actions: [
    {
      ActionType: 'CREATE_TASK',
      CreateTaskAction: {
        Name: 'Churn Risk Follow-up',
        Description: 'Customer expressed intent to cancel with negative sentiment',
        ContactFlowId: 'flow-xxx',
      },
    },
    {
      ActionType: 'SEND_NOTIFICATION',
      SendNotificationAction: {
        DeliveryMethod: 'EMAIL',
        Subject: 'Churn Risk Alert',
        Content: 'A customer expressed cancellation intent. Review the contact.',
        ContentType: 'PLAIN_TEXT',
        Recipient: {
          UserTags: { Department: 'Retention' },
        },
      },
    },
  ],
  PublishStatus: 'PUBLISHED',
}));
```

## Rule Actions (`Actions`)

Actions are NOT part of the function language — they are the separate `Actions` array of `CreateRule`/`UpdateRule`. The function defines *when* the rule fires; the actions define *what runs*.

### Complete `ActionType` enum

`CREATE_TASK`, `ASSIGN_CONTACT_CATEGORY`, `GENERATE_EVENTBRIDGE_EVENT`, `SEND_NOTIFICATION`, `CREATE_CASE`, `UPDATE_CASE`, `ASSIGN_SLA`, `END_ASSOCIATED_TASKS`, `SUBMIT_AUTO_EVALUATION`

### Per-action parameters

| Action Type | Struct | Parameters |
|---|---|---|
| `ASSIGN_CONTACT_CATEGORY` | `AssignContactCategoryAction` | `{}` — no params; tags the contact with the rule's category |
| `CREATE_TASK` | `TaskAction` | `Name`, `Description`, `ContactFlowId`, `References{<key>:{Value, Type, Status, Arn, StatusReason}}` |
| `GENERATE_EVENTBRIDGE_EVENT` | `EventBridgeAction` | `Name` |
| `SEND_NOTIFICATION` | `SendNotificationAction` | `DeliveryMethod`(=`EMAIL`), `Subject`, `Content`, `ContentType`(=`PLAIN_TEXT`), `Recipient{UserTags, UserIds[]}`, `Exclusion{UserTags, UserIds[]}` |
| `CREATE_CASE` | `CreateCaseAction` | `Fields[]{Id, Value{BooleanValue\|DoubleValue\|EmptyValue\|StringValue}}`, `TemplateId` |
| `UPDATE_CASE` | `UpdateCaseAction` | `Fields[]{Id, Value{...}}` |
| `ASSIGN_SLA` | `AssignSlaAction` | `SlaAssignmentType`(=`CASES`), `CaseSlaConfiguration{Name, Type(=CaseField), FieldId, TargetFieldValues[]{Bool\|Double\|Empty\|String}, TargetSlaMinutes(long)}` |
| `END_ASSOCIATED_TASKS` | `EndAssociatedTasksAction` | `{}` — no params |
| `SUBMIT_AUTO_EVALUATION` | `SubmitAutoEvaluationAction` | `EvaluationFormId` |

`TaskAction.References` `Type` ∈ `URL`, `ATTACHMENT`, `CONTACT_ANALYSIS`, `NUMBER`, `STRING`, `DATE`, `EMAIL`, `EMAIL_MESSAGE`, `EMAIL_MESSAGE_PLAIN_TEXT`, `EMAIL_MESSAGE_PLAIN_TEXT_REDACTED`, `EMAIL_MESSAGE_REDACTED`. `Status` ∈ `AVAILABLE`, `DELETED`, `APPROVED`, `REJECTED`, `PROCESSING`, `FAILED`.

### Action support by event source

Not every action is valid for every event source:

| Action | Supported event sources |
|---|---|
| `CREATE_TASK` | `OnZendeskTicketCreate`, `OnZendeskTicketStatusUpdate`, `OnSalesforceCaseCreate` |
| `GENERATE_EVENTBRIDGE_EVENT` | `OnPostCallAnalysisAvailable`, `OnRealTimeCallAnalysisAvailable`, `OnRealTimeChatAnalysisAvailable`, `OnPostChatAnalysisAvailable`, `OnContactEvaluationSubmit`, `OnMetricDataUpdate` |
| `ASSIGN_CONTACT_CATEGORY` | `OnPostCallAnalysisAvailable`, `OnRealTimeCallAnalysisAvailable`, `OnRealTimeChatAnalysisAvailable`, `OnPostChatAnalysisAvailable`, `OnZendeskTicketCreate`, `OnZendeskTicketStatusUpdate`, `OnSalesforceCaseCreate` |
| `SEND_NOTIFICATION` | `OnPostCallAnalysisAvailable`, `OnRealTimeCallAnalysisAvailable`, `OnRealTimeChatAnalysisAvailable`, `OnPostChatAnalysisAvailable`, `OnContactEvaluationSubmit`, `OnMetricDataUpdate` |
| `CREATE_CASE` | `OnPostCallAnalysisAvailable`, `OnPostChatAnalysisAvailable` |
| `UPDATE_CASE` | `OnCaseCreate`, `OnCaseUpdate` |
| `END_ASSOCIATED_TASKS` | `OnCaseUpdate` |
| `ASSIGN_SLA` | `OnCaseCreate`, `OnCaseUpdate` |

### Full `Actions` JSON skeleton

```json
[
  {
    "ActionType": "CREATE_TASK|ASSIGN_CONTACT_CATEGORY|GENERATE_EVENTBRIDGE_EVENT|SEND_NOTIFICATION|CREATE_CASE|UPDATE_CASE|ASSIGN_SLA|END_ASSOCIATED_TASKS|SUBMIT_AUTO_EVALUATION",
    "TaskAction": {
      "Name": "string",
      "Description": "string",
      "ContactFlowId": "string",
      "References": { "<key>": { "Value": "string", "Type": "URL", "Status": "AVAILABLE", "Arn": "string", "StatusReason": "string" } }
    },
    "EventBridgeAction": { "Name": "string" },
    "AssignContactCategoryAction": {},
    "SendNotificationAction": {
      "DeliveryMethod": "EMAIL",
      "Subject": "string",
      "Content": "string",
      "ContentType": "PLAIN_TEXT",
      "Recipient": { "UserTags": { "string": "string" }, "UserIds": ["string"] },
      "Exclusion": { "UserTags": { "string": "string" }, "UserIds": ["string"] }
    },
    "CreateCaseAction": {
      "Fields": [ { "Id": "string", "Value": { "BooleanValue": true, "DoubleValue": 0.0, "EmptyValue": {}, "StringValue": "string" } } ],
      "TemplateId": "string"
    },
    "UpdateCaseAction": {
      "Fields": [ { "Id": "string", "Value": { "StringValue": "string" } } ]
    },
    "AssignSlaAction": {
      "SlaAssignmentType": "CASES",
      "CaseSlaConfiguration": {
        "Name": "string",
        "Type": "CaseField",
        "FieldId": "string",
        "TargetFieldValues": [ { "StringValue": "string" } ],
        "TargetSlaMinutes": 0
      }
    },
    "EndAssociatedTasksAction": {},
    "SubmitAutoEvaluationAction": { "EvaluationFormId": "string" }
  }
]
```

### CloudFormation form (`AWS::Connect::Rule`)

In CloudFormation, `Actions` uses **arrays per action type** (not a single `ActionType` discriminator): `AssignContactCategoryActions`, `CreateCaseActions`, `EndAssociatedTasksActions`, `EventBridgeActions`, `SendNotificationActions`, `TaskActions`, `UpdateCaseActions`. Required properties: `Actions`, `Function` (the conditions string), `InstanceArn`, `Name`, `PublishStatus`, `TriggerEventSource`. Returns `RuleArn` via `Fn::GetAtt`.

### Getting the exact `Function` for any event source

The per-source condition pages render client-side only, so the most reliable way to see the exact operators and `ComparisonValue` paths for a given event source is to build the rule in the Amazon Connect console, then call `DescribeRule` (or `aws connect describe-rule`) and read back the returned `Function` string. Use `aws connect create-rule --generate-cli-skeleton input` for the full `Actions` skeleton.
