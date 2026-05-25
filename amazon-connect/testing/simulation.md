# Testing and Simulation

Amazon Connect provides a visual test designer and simulation capabilities to validate contact flows, business logic, and routing behavior before deploying changes to production. Tests run within the Connect console without requiring live contacts or agents.

## Visual Test Designer

The test designer is available in the Amazon Connect console and provides a no-code interface for creating, executing, and reviewing test cases.

### Creating Test Cases

A test case defines:

1. **Entry point** -- the flow or phone number where the simulated contact begins
2. **Contact attributes** -- pre-set attributes that simulate different customer scenarios (e.g., account tier, language, region)
3. **Customer inputs** -- DTMF digits or Lex bot intents the simulated customer provides
4. **Expected experiences** -- the flow path and blocks the contact should traverse
5. **Assertions** -- specific conditions that must be true for the test to pass

**Example test scenarios:**
- VIP customer calls during business hours -> routed to priority queue
- Customer enters invalid account number 3 times -> transferred to agent
- After-hours call -> plays closed message and offers voicemail
- Full queue with 30+ contacts -> plays overflow message and offers callback

### Specifying Experiences

Define the expected path through the flow:

- Which blocks the contact should pass through
- Which branch (success, error, timeout) should be taken at each decision point
- Which queue the contact should be placed in
- Which prompt should be played

### Attribute Assertions

Assert that contact attributes have expected values at specific points in the flow:

| Assertion Type | Description |
|---|---|
| Attribute equals | Contact attribute matches an exact value |
| Attribute exists | Contact attribute is set (non-null) |
| Queue assignment | Contact is in the expected queue |
| Flow transition | Contact moved to the expected flow |

## Executing Test Cases

Run tests from the console:

1. Select one or more test cases
2. Click **Execute**
3. Connect simulates the contact through the flow using the defined attributes and inputs
4. Results are available immediately after execution completes

### Execution Modes

- **Single test** -- run one test case and review detailed results
- **Batch execution** -- run multiple test cases simultaneously for regression testing
- **Re-run** -- re-execute a previously run test case (useful after flow changes)

## Reviewing Results

### Results Summary

After execution, each test case shows:

| Result | Description |
|---|---|
| **Pass** | All assertions passed, flow path matched expectations |
| **Fail** | One or more assertions failed or flow deviated from expected path |
| **Error** | Execution could not complete (invalid flow, missing resources) |

### Deviation Analysis

When a test fails, the results highlight **deviations from the expected path**:

- Which block produced an unexpected branch
- What the expected vs actual attribute values were
- Where the flow diverged from the expected path
- The exact block and branch where the deviation occurred

This makes it straightforward to identify whether the failure is due to a flow change, a configuration issue, or a test case that needs updating.

## Chat Testing and Simulation

Test chat flows with simulated chat contacts:

- Define chat messages the simulated customer sends
- Test Lex bot interactions within chat flows
- Validate chat-specific blocks (e.g., interactive messages, list pickers)
- Assert response messages and routing behavior

## Flow Simulation

The flow designer includes an inline simulation mode:

- Step through a flow block by block
- Set contact attributes at each step
- See which branch is taken at each decision point
- Useful for ad-hoc debugging during flow development

This is lighter weight than a full test case and does not require saving a test definition.

## Business Condition Testing

Test flows under specific business conditions without waiting for those conditions to naturally occur:

| Condition | How to Simulate |
|---|---|
| After hours | Set simulated time to outside business hours |
| Holiday | Set simulated date to a holiday |
| Full queue | Configure queue size attribute to exceed threshold |
| Agent unavailable | Simulate with no available agents |
| Specific customer tier | Set customer attributes to match tier criteria |
| Failed Lambda | Simulate Lambda error branch |

This is particularly valuable for testing edge cases and error handling paths that are difficult to trigger with live contacts.

## Test Dashboard

The test dashboard provides an aggregate view of test execution history:

- **Pass/fail trends** -- track test reliability over time
- **Recent executions** -- quick access to latest results
- **Test coverage** -- which flows have associated test cases
- **Failure analysis** -- common failure patterns across test cases

Use the dashboard to identify flows that are frequently breaking and need attention.

## APIs

| API | Purpose |
|---|---|
| `CreateTestCase` | Create a new test case definition |
| `UpdateTestCase` | Modify an existing test case |
| `DescribeTestCase` | Get the full definition of a test case |
| `DeleteTestCase` | Delete a test case |
| `SearchTestCases` | Search for test cases by name, flow, or status |
| `StartTestCaseExecution` | Execute a test case |
| `StopTestCaseExecution` | Cancel a running test execution |
| `GetTestCaseExecutionSummary` | Get pass/fail summary for an execution |
| `ListTestCaseExecutions` | List all executions for a test case |
| `ListTestCaseExecutionRecords` | Get detailed step-by-step execution records |

### Example -- Create and Run a Test Case

```javascript
import {
  ConnectClient,
  CreateTestCaseCommand,
  StartTestCaseExecutionCommand,
  GetTestCaseExecutionSummaryCommand,
} from "@aws-sdk/client-connect";

const client = new ConnectClient({ region: "us-east-1" });

// Create a test case for after-hours routing
const { TestCaseId } = await client.send(new CreateTestCaseCommand({
  InstanceId: instanceId,
  Name: "After Hours - Voicemail Offered",
  Description: "Verifies that after-hours calls receive the closed message and voicemail option",
  ContactAttributes: {
    "SystemEndpoint": "+15551234567",
  },
  SimulatedTime: "2026-05-25T23:30:00Z", // 11:30 PM
  CustomerInputs: [
    { Type: "DTMF", Value: "1" }, // Press 1 for voicemail
  ],
  Assertions: [
    {
      Type: "FLOW_TRANSITION",
      ExpectedValue: "After Hours Flow",
    },
    {
      Type: "PROMPT_PLAYED",
      ExpectedValue: "We are currently closed",
    },
  ],
}));

// Execute the test
const { ExecutionId } = await client.send(new StartTestCaseExecutionCommand({
  InstanceId: instanceId,
  TestCaseId,
}));

// Check results (after execution completes)
const summary = await client.send(new GetTestCaseExecutionSummaryCommand({
  InstanceId: instanceId,
  TestCaseId,
  ExecutionId,
}));

console.log(`Result: ${summary.Status}`); // PASSED or FAILED
console.log(`Deviations: ${summary.DeviationCount}`);
```

## Best Practices

- **Create test cases for every flow** -- especially flows handling business-critical routing
- **Test edge cases** -- after hours, holidays, full queues, Lambda failures, invalid inputs
- **Run regression tests after flow changes** -- batch-execute all related test cases before publishing
- **Use descriptive names** -- name test cases by the scenario they validate, not by internal IDs
- **Version control test cases** -- export test definitions and track changes alongside flow exports
- **Monitor the dashboard** -- review pass/fail trends weekly to catch regressions early

## Key Considerations

- **No live traffic** -- tests run in simulation mode and do not affect live contacts or agents
- **Lambda calls** -- simulated contacts can invoke real Lambda functions during testing. Use test-specific attributes to prevent side effects in production systems.
- **Lex bots** -- test cases can interact with live Lex bots. Ensure test utterances do not trigger unintended actions.
- **Quotas** -- test case executions are subject to Connect API throttling limits
- **Cost** -- test executions do not incur per-minute telephony charges, but Lambda and Lex invocations during tests are billed normally
- **Permissions** -- creating and running tests requires appropriate security profile permissions in Connect
