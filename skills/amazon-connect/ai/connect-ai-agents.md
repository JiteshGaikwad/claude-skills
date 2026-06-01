# Amazon Connect AI Agents

## Overview

Amazon Connect AI is powered by Amazon Bedrock and provides GDPR/HIPAA-compliant generative AI capabilities across two primary modes:

- **Agentic self-service** -- autonomous AI agents that handle customer interactions end-to-end without human intervention
- **Agent-assist** -- real-time AI that supports human agents during live conversations

All AI features are managed through the Amazon Q Connect service APIs and configured via the Connect admin console or programmatically.

---

## Agentic Self-Service

### Orchestrator AI Agents

Agentic self-service uses an **orchestrator AI agent** that performs autonomous multi-step reasoning to resolve customer requests. The orchestrator:

- Breaks down complex customer intents into sub-tasks
- Decides which tools to invoke and in what order
- Maintains conversation context across multiple turns
- Handles disambiguation when customer intent is unclear
- Escalates to a human agent when it determines it cannot resolve the issue
- Retries failed tool calls with adjusted parameters
- Provides natural conversational responses while executing backend actions

### Orchestrator Behavior

The orchestrator follows a reasoning loop:

1. **Observe** -- receive the customer's message and conversation history
2. **Think** -- analyze intent, determine what information is needed, plan next steps
3. **Act** -- invoke a tool (MCP, Return to Control, or Constant) or generate a response
4. **Respond** -- provide a natural language response to the customer
5. **Repeat** -- continue the loop until the issue is resolved or escalation is needed

The orchestrator can chain multiple tool calls in a single turn (e.g., look up account, check order status, process return) without requiring the customer to repeat information.

### Architecture

```
Customer --> Connect Flow --> Conversational AI Bot (Lex) --> Orchestrator AI Agent
                                                                  |
                                                          +-------+-------+
                                                          |       |       |
                                                        Tools   KB    Guardrails
```

### Tool Integration

An orchestrator AI agent has **three configurable tool types**: **MCP tools**, **Return to Control**, and **Constant**. (The `SelfServiceOrchestrator` ships with two default Return-to-Control tools — `Complete` and `Escalate`.)

#### 1. MCP Tools (Backend Actions)

Connect supports **Model Context Protocol (MCP)** tools so AI agents (self-service *and* agent-assist) retrieve information and complete actions without human intervention — invoked mid-conversation without returning control to the flow. Three sources:

- **Out-of-the-box tools** — prebuilt for common tasks (update contact attributes, retrieve case info, update cases, start tasks).
- **Flow module tools** — create new, or convert existing flow modules into MCP tools, reusing business logic across static and generative workflows; can reach third-party sources.
- **Third-party MCP tools** — via **Amazon Bedrock AgentCore Gateway**. Register AgentCore Gateways in the AWS console (like registering a third-party app) to access their tools, including remote MCP servers.

**MCP tool invocations have a 30-second timeout** — exceeding it terminates the request. When adding a tool you can: add usage instructions for that tool, override input values, and filter output values for accuracy.

**Governance:** AI-agent tool access reuses the **security-profile** framework — see "AI Agent Security Profiles" below.

#### 2. Return to Control Tools

Return to Control tools hand execution back to the Connect flow for processing.

**Built-in Return to Control tools:**
- **Complete** -- signals the conversation is resolved; the flow can perform wrap-up actions
- **Escalate** -- signals the AI cannot handle the request; the flow routes to a human agent

**Custom Return to Control tools** allow structured data to be passed back to the flow with a defined JSON input schema:

```json
{
  "type": "object",
  "properties": {
    "customerIntent": {
      "type": "string",
      "description": "The detected customer intent"
    },
    "sentiment": {
      "type": "string",
      "enum": ["positive", "neutral", "frustrated"],
      "description": "Customer's emotional state during the conversation"
    },
    "escalationSummary": {
      "type": "string",
      "maxLength": 500,
      "description": "Summary for the human agent: what was asked, what was attempted, why escalation is needed"
    },
    "escalationReason": {
      "type": "string",
      "enum": ["complex_request", "technical_issue", "customer_frustration", "policy_exception", "out_of_scope", "other"],
      "description": "Category for the escalation reason"
    }
  },
  "required": ["escalationReason", "escalationSummary", "customerIntent", "sentiment"]
}
```

**Handling Return to Control in flows:**

When a Return-to-Control tool fires, the AI conversation ends, control returns to the flow, and the tool name + input params are stored as **Lex session attributes**. Add a **Check contact attributes** block after the **Default** output of the `Get customer input` block:

```
Check contact attributes
  Namespace: Lex
  Key: Session attributes
  Session Attribute Key: Tool
  Conditions:
    "Escalate" --> Set working queue + Transfer to queue
    "Complete" --> Disconnect
    "CustomTool" --> Custom handling branch
  No match --> Disconnect or further routing
```

To use a custom tool's input params downstream, add a **Set contact attributes** block copying them from `Lex → Session attributes` (e.g. `escalationReason`, `escalationSummary`, `customerIntent`, `sentiment`) into contact attributes. Optionally add a **Set event flow** block (event hook **Default flow for Agent UI**) to surface escalation context to the human agent.

#### 3. Constant Tools

Constant tools return a configured static string every time they're invoked and do **not** end the conversation — useful for testing/development before a real MCP tool is connected (e.g. a `getOrderStatus` Constant returning sample JSON). Create via **Add tool → Create new AI Tool**, type **Constant**, enter the **Constant value**, **Create** then **Publish**.

### Message Parsing (`<message>` tags)

Orchestrator AI agents **only display text wrapped in `<message>` tags** — the orchestration prompt must enforce this, or the customer sees nothing. The system prompt should require:

```
<formatting_requirements>
MUST format all responses with this structure:
<message>
Your response to the customer. This text is spoken aloud — write naturally and conversationally.
</message>
<thinking>
Reasoning can go here for complex decisions.
</thinking>
MUST NEVER put thinking content inside message tags.
MUST always start with <message>, even when using tools, so the customer knows you're working on it.
</formatting_requirements>
```

Multiple `<message>` blocks can appear in one response (an initial acknowledgment, then results). **Orchestration agents require chat streaming for chat contacts** (see [chat-message-streaming.md](chat-message-streaming.md)).

### AI Agent Security Profiles (tool governance)

AI-agent tool access reuses the human security-profile framework. Permissions can be assigned for **AgentCore gateway tools**, **flow modules saved as tools**, and **out-of-the-box tools**. Built-in tool permissions mirror the equivalent human-agent permission:

| AI agent tool | Required (Agent Applications) permission |
|---|---|
| Cases (Create, Update, Search) | Cases – View / Edit |
| Customer Profiles | Customer Profiles – View |
| Knowledge Base (Retrieve) | Connect assistant – View Access |
| Tasks (StartTaskContact) | Tasks – Create |

**Shared permissions:** for **Agent Assistance**, the human agent's security profile must include the **same** permissions as the AI agent's configured tools — invocations are authorized against the combination, so if the AI agent has the Cases tool, the human agent must also have Cases permissions or invocations fail. Administering AI agents needs (under **AI agent designer**) AI Agents / AI Prompts / AI Guardrails – All Access, plus **Conversational AI – All Access** and **Flows / Flow Modules – All Access** (Channels and Flows).

---

## Agent-Assist

Agent-assist provides real-time AI support to human agents during live conversations:

- **Generative responses** -- AI-generated suggested replies based on conversation context and knowledge base content
- **Document links** -- direct links to relevant knowledge base articles, SOPs, and documentation
- **Recommended actions** -- suggested next steps or actions the agent should take based on the current conversation state
- **Intent detection** -- real-time classification of customer intent displayed to the agent
- **Note taking** -- AI-generated structured notes from the conversation

Agent-assist surfaces in the agent workspace (CCP or custom agent desktop) as a panel that updates in real time as the conversation progresses.

### How Agent-Assist Works

1. Contact Lens streams the conversation transcript to Q Connect in real time
2. Q Connect analyzes the transcript and detects customer intent
3. Knowledge base content is retrieved based on the conversation context
4. AI generates response suggestions and surfaces relevant documents
5. Agent sees recommendations in the workspace panel
6. Agent can click to insert a suggestion, open a document, or ignore
7. Agent provides feedback (helpful/not helpful) to improve future recommendations

---

## AI Agent Types

An **AI agent** is a resource that configures the end-to-end behavior for a use case: which AI prompts and AI guardrails are used, and the response locale. Each use case has a system-default AI agent you can override with your own. Connect provides these out-of-the-box AI agent types (the **AI Agent type** options when creating one):

| Agent type | What it does | AI prompt types it uses |
|---|---|---|
| **Orchestration** | Agentic agent that orchestrates use cases per customer needs; multi-turn conversation + invokes pre-configured tools. | Orchestration |
| **Answer Recommendation** | Automatic intent-based recommendations pushed to agents during a contact. | Intent labelling generation, Query reformulation, Answer generation |
| **Manual Search** | Solutions for on-demand searches an agent initiates. | Answer generation |
| **Self Service** | Solutions for customer self-service. (Locale **English only**.) | Self-service pre-processing, Self-service answer generation |
| **Email Response** | Email reply to the customer from a conversation script. | Email response |
| **Email Overview** | Overview/summary of an email thread. | Email overview |
| **Email Generative Answer** | Generated answer content for email responses. | Email generative answer |
| **Note Taking** | Structured real-time notes (invoked as a tool on the Agent Assistance orchestrator). | Note taking |
| **Agent Assistance** | Real-time orchestrator backing agent-assist. | Orchestration |
| **Case Summarization** | Summarizes a case. | Case Summarization |

**Partial override:** Answer recommendation, Self service, Email response, and Email generative answer support **two** prompt types each. If you override one prompt but not the other, the AI agent keeps the system default for the one you didn't override — so you can override just part of the default experience.

For prompt details (types, YAML, models, caching, CLI) see [ai-prompts.md](ai-prompts.md); for guardrails see [ai-guardrails.md](ai-guardrails.md).

### Default system prompts & agents

You can't edit default prompts directly, but you can **copy** one as a starting point; adding the copy to an AI agent overrides the default. System AI prompts include: `AgentAssistanceOrchestration`, `AnswerGeneration`, `CaseSummarization`, `EmailGenerativeAnswer`, `EmailOverview`, `EmailQueryReformulation`, `EmailResponse`, `IntentLabelingGeneration`, `NoteTaking`, `QueryReformulation`, `SalesAgent`, `SelfServiceAnswerGeneration`, `SelfServiceOrchestration`, `SelfServicePreProcessing`. System AI agents include: `AgentAssistanceOrchestrator`, `AnswerRecommendation`, `CaseSummarization`, `EmailGenerativeAnswer`, `EmailOverview`, `EmailResponse`, `ManualSearch`, `NoteTaking`, `SalesAgent`, `SelfService`, `SelfServiceOrchestrator`. (List with `aws qconnect list-ai-agents --origin SYSTEM` / `list-ai-prompt-versions --origin SYSTEM`.)

### Create an AI agent (console)

1. **AI agent designer → AI agents → Create AI Agent**; choose the **AI Agent type**.
2. On the **Agent builder**, set the **Locale** for the response (available for Orchestration, Answer recommendation, Manual search, and the Email types; **not** for Self-service, which is English only).
3. Choose the **published AI prompt version(s)** to override the defaults, and optionally add **one** AI guardrail.
4. **Save**, then **Publish** to make the new agent version available as a default.

### Setting which agent is used (precedence)

Precedence is **session-level > Assistant-level > system default**.

```bash
# Create / version (qualify as <AI_AGENT_ID>:<VERSION_NUMBER>)
aws qconnect create-ai-agent --type ANSWER_RECOMMENDATION --visibility-status PUBLISHED \
  --configuration '{"answerRecommendationAIAgentConfiguration":{"answerGenerationAIPromptId":"<ID:VER>","intentLabelingGenerationAIPromptId":"<ID:VER>","queryReformulationAIPromptId":"<ID:VER>"}}'
aws qconnect create-ai-agent-version --assistant-id <ID> --ai-agent-id <ID>

# Set the default on the Assistant (takes effect next contact/session)
aws qconnect update-assistant-ai-agent --assistant-id <ID> --ai-agent-type MANUAL_SEARCH \
  --configuration '{"aiAgentId":"<ID:VERSION>"}'

# Override per session
aws qconnect update-session --assistant-id <ID> --session-id <ID> \
  --ai-agent-configuration '{"ANSWER_RECOMMENDATION":{...},"MANUAL_SEARCH":{...}}'
```

For a partial config, omit the prompt IDs you want to keep as the default. Manual search has a single prompt (`manualSearchAIAgentConfiguration.answerGenerationAIPromptId`), so no partial config applies.

### Overriding System Defaults (SDK)

```javascript
const { QConnectClient, CreateAIAgentCommand } = require("@aws-sdk/client-qconnect");

const client = new QConnectClient({ region: "us-east-1" });

await client.send(new CreateAIAgentCommand({
  assistantId: "assistant-id",
  name: "CustomOrchestrator",
  type: "SELF_SERVICE",
  configuration: {
    selfServiceAIAgentConfiguration: {
      selfServiceAIGuardrailId: "guardrail-id",
      selfServicePreProcessingConfiguration: {
        aiPromptId: "custom-preprocess-prompt-id"
      },
      selfServiceAnswerGenerationConfiguration: {
        aiPromptId: "custom-answer-prompt-id"
      }
    }
  }
}));
```

### Versioning

AI agents support versioning:

- Each update creates a new version
- Previous versions can be referenced and restored
- Flows can pin to a specific version or use `LATEST`
- Enables safe rollback if a prompt change degrades quality

```javascript
// Create a version snapshot
await client.send(new CreateAIAgentVersionCommand({
  assistantId: "assistant-id",
  aiAgentId: "agent-id"
}));

// List all versions
const versions = await client.send(new ListAIAgentVersionsCommand({
  assistantId: "assistant-id",
  aiAgentId: "agent-id"
}));
```

---

## AI Prompts

Connect supports **12 AI prompt types** (Orchestration, Answer generation, Intent labelling generation, Query reformulation, Self-service pre-processing, Self-service answer generation, Email response/overview/generative-answer/query-reformulation, Note taking, Case Summarization). Prompts are YAML templates in one of two formats (`MESSAGES` or `TEXT_COMPLETIONS`), support system variables (`$.transcript`, `$.contentExcerpt`, `$.locale`, `$.query`) and custom variables (`$.Custom.<NAME>`), and use prompt caching (on by default).

**Full reference — types, YAML fields, the assistant-prefill caveat, variables, optimization/caching, per-Region models, and CLI — is in [ai-prompts.md](ai-prompts.md).**

---

## AI Guardrails

Connect AI agents use Amazon Bedrock guardrails (max **3** custom per business; an agent attaches **one**). Six policy areas: **content filters** (Hate/Insults/Sexual/Violence/Misconduct/Prompt Attack, strength NONE/LOW/MEDIUM/HIGH per input+output), **denied topics** (up to 30), **contextual grounding** (GROUNDING/RELEVANCE, threshold 0–0.99), **word filters** (exact match; up to 10,000 custom items), **sensitive information / PII** (Block / Anonymize-mask / None, full entity list + custom regex), and **blocked messaging**. Image filtering is not supported; enabling guardrails on streaming adds time-to-first-token latency.

**Full reference — every policy's options, thresholds, limits, the complete PII entity list, and CLI — is in [ai-guardrails.md](ai-guardrails.md).**


---

## Setup Sequence

### Self-Service Setup

1. **Create orchestrator agent** -- define the agent with its base configuration, select model, attach guardrail
2. **Create custom prompts (optional)** -- customize orchestration, pre-processing, and answer generation prompts
3. **Create guardrail (optional)** -- configure content filters, denied topics, PII handling
4. **Add tools** -- attach MCP tools, Return to Control tools, and/or Constant tools with input/output schemas
5. **Associate knowledge bases** -- link one or more KBs with content tag filters and search type configuration
6. **Configure orchestration prompt** -- customize the system prompt that guides agent reasoning and tool selection behavior
7. **Set as default self-service agent** -- on the **AI Agents** page, set it in **Default AI Agent Configurations → Self Service** (or override per-session via Lambda).
8. **Create a Conversational AI bot** -- **Routing → Flows → Conversational AI**, with the Connect Customer AI agent intent enabled (this is the conversational front end).
9. **Build a contact flow** -- a `Get customer input` block invoking the bot, plus a `Check contact attributes` block to route on the Return-to-Control tool selected.
10. **Test end-to-end** -- verify tool invocations, escalation paths, and guardrail enforcement. (For chat, enable chat streaming.)

### Agent-Assist Setup

1. **Create/configure AI agent** -- customize the Answer Recommendation or Agent Assistance agent type.
2. **Associate knowledge bases** -- link KBs containing support documentation.
3. **Enable Contact Lens** -- required for voice (real-time transcript streaming); **not** required for chat. Add a `Set recording and analytics behavior` block configured for Contact Lens real-time.
4. **Add the `Connect assistant` flow block** -- associates the assistant domain to the contact (for default behavior). To customize agents, use a Lambda + `AWS Lambda function` block instead.
5. **Enable in agent workspace** -- ensure agents have **Connect assistant – View Access** so recommendations appear.
6. **Test with live contacts** -- verify recommendations appear and are relevant.

### Security Profiles for AI Agents

- **Admins** (configure agents/prompts/guardrails): under **AI agent designer** — AI Agents, AI Prompts, AI Guardrails (View or All Access). Creating agents/flows also needs **Conversational AI** and **Flows / Flow Modules** (Channels and Flows).
- **Agents** (receive recommendations / use guides): **Connect assistant – View Access**, plus **Custom views – Access** for step-by-step guides.
- **AI-agent tool governance** mirrors human permissions — see "AI Agent Security Profiles" above.

---

## Knowledge Base Configuration

When configuring an AI agent with a knowledge base, the following parameters are available:

```javascript
{
  associationId: "kb-association-id",       // Links the KB to the agent
  contentTagFilter: {                        // Filter KB content by tags
    tagCondition: {
      key: "department",
      value: "billing"
    }
  },
  maxResults: 5,                             // Max documents to retrieve (1-100)
  overrideKnowledgeBaseSearchType: "SEMANTIC" // "SEMANTIC" or "HYBRID"
}
```

- **SEMANTIC** -- vector similarity search; best for natural language queries
- **HYBRID** -- combines semantic search with keyword matching; better for queries containing specific terms, codes, or product names

### Multiple Knowledge Base Setup

- Create separate KBs for different topics (billing, technical support, policies)
- Tag KBs with metadata for content segmentation
- AI agent queries relevant KB based on contact context and intent
- Best practice: segment by department or product line, not by document type
- Use `CreateContentAssociation` to link specific content items to step-by-step guides

---

## Supported Models by Region

Model availability varies by AWS region:

| Model | Best For | Availability |
|-------|---------|-------------|
| Claude Sonnet 4.5 | Complex reasoning, nuanced responses | Select regions (us-east-1, us-west-2, eu-west-2, ap-southeast-2) |
| Amazon Nova Pro | General-purpose, balanced performance/cost | All Connect regions |
| Claude Haiku | Fast responses, simple queries, lowest cost | Select regions |

The full per-Region model matrix (system-default models per prompt, and the models supported for custom prompts) is in [ai-prompts.md](ai-prompts.md). Check the [Amazon Connect documentation](https://docs.aws.amazon.com/connect/latest/adminguide/) for current availability.

### Model Upgrade Guide

- Upgrade via `UpdateAIAgent` API -- change the model reference in the agent config
- Test in non-production first -- different models may interpret prompts differently
- No downtime during model switch -- takes effect on next contact
- Rollback: revert to previous model version via `UpdateAIAgent`
- Version your agents before switching models for safe rollback

---

## Associating AI Agents with Flows

- **Default behavior:** add the **Connect assistant** flow block (see below).
- **Override behavior** (custom agents / per-session): create a Lambda and add it via the **AWS Lambda function** block at runtime:

```javascript
// In your Lambda function invoked from the contact flow
exports.handler = async (event) => {
  const { QConnectClient, UpdateSessionCommand } = require("@aws-sdk/client-qconnect");
  const client = new QConnectClient({ region: "us-east-1" });

  await client.send(new UpdateSessionCommand({
    assistantId: "assistant-id",
    sessionId: event.Details.ContactData.Attributes.qconnectSessionId,
    aiAgentConfiguration: {
      [agentType]: {
        aiAgentId: "custom-agent-id"
      }
    }
  }));

  return { success: true };
};
```

### Connect assistant flow block

Associates a Connect assistant domain to a contact to enable real-time recommendations.

- **Config:** the full ARN of the Connect assistant domain, and the **Orchestration AI agent** to use for Agent Assistance.
- **Channels:** Voice, Chat, Task, Email (all Yes). *Outbound email hitting this block does nothing functionally but is still **charged** — guard with a `Check contact attributes` block.*
- **Flow types:** Inbound, Customer Queue, Outbound whisper, Transfer to Agent, Transfer to Queue.
- **Voice requires Contact Lens real-time** (`Set recording and analytics behavior` block, placement doesn't matter); chat does not.
- **Branches:** Success, Error.
- If you **customize** agents, use a Lambda + `AWS Lambda function` block instead of this block.

---

## AI Agent Sessions (custom data)

Add custom data to an AI-agent session so prompts can reference it via `{{$.Custom.<KEY>}}`.

1. Add the **Connect assistant** block (creates/associates the session for the contact).
2. After it, add an **AWS Lambda function** block whose Lambda calls **`UpdateSessionData`**. The session ID comes from **`DescribeContact`** (using the `assistantId` from the Connect assistant block).

```bash
aws qconnect update-session-data --assistant-id <ASSISTANT_ID> --session-id <SESSION_ID> \
  --data '[ { "key": "productId", "value": { "stringValue": "ABC-123" } } ]'
```

Reference it in a prompt as `{{$.Custom.productId}}`. If the value isn't in the session, it interpolates as an **empty string** — write the prompt to handle absence.

---

## Language / Locale

Set the response language on an AI agent via its **Locale** (AI agent builder dropdown → Save → Publish, or the `locale` field in the agent config, e.g. `"locale": "es_ES"`). Locale is settable for Orchestration, Answer recommendation, Manual search, and the Email types; **Self-service is English only.** Connect AI agents support ~66 locale codes (e.g. `en_US`, `es_ES`, `fr_FR`, `de_DE`, `ja_JP`, `ko_KR`, `pt_BR`, `zh_CN`, `ar`, `hi_IN`, … through `zu_ZA`). If text isn't supported for the chosen language, that part isn't returned.

---

## Integration with Step-by-Step Guides

AI agents can be integrated with step-by-step guides (agent workspace views) using the `CreateContentAssociation` API:

```javascript
const { QConnectClient, CreateContentAssociationCommand } = require("@aws-sdk/client-qconnect");

const client = new QConnectClient({ region: "us-east-1" });

await client.send(new CreateContentAssociationCommand({
  knowledgeBaseId: "kb-id",
  contentId: "content-id",
  associationType: "AMAZON_CONNECT_GUIDE",
  association: {
    amazonConnectGuideAssociation: {
      flowId: "flow-arn"  // ARN of the guide flow
    }
  }
}));
```

This allows AI-generated recommendations to include "Launch Guide" actions that open the relevant step-by-step guide in the agent workspace.

**Getting the IDs:** `knowledgeBaseId` via `aws qconnect list-knowledge-bases`; `contentId` via `aws qconnect list-contents --knowledge-base-id <id>`; the guide's `flowARN` via the flow designer (**About this flow → View ARN**) or `aws connect list-contact-flows --instance-id <id>`. **One** content association per content resource, but a guide can be associated with **multiple** content resources. Verify with `aws qconnect list-content-associations --knowledge-base-id <id> --content-id <id>`. Agents need **Connect AI agents – View** (to see/receive content; auto-recommendations require Contact Lens conversational analytics) and **Custom views – Access** (to see the guides).

---

## CloudWatch Monitoring

Monitor AI agent performance with CloudWatch. *(The metric names below are illustrative of what to track — confirm exact metric/namespace names in the CloudWatch console for your instance.)*

| Metric | Description |
|---|---|
| `SessionCount` | Number of Q Connect sessions created |
| `RecommendationCount` | Number of recommendations generated |
| `RecommendationAcceptedCount` | Number of recommendations accepted by agents |
| `QueryLatency` | Time to retrieve knowledge base results |
| `ResponseGenerationLatency` | Time to generate AI responses |
| `ToolInvocationCount` | Number of tool invocations by the orchestrator |
| `ToolInvocationErrorCount` | Number of failed tool invocations |
| `EscalationCount` | Number of self-service escalations to human agents |
| `GuardrailBlockedCount` | Number of responses blocked by guardrails |

Set up CloudWatch alarms for:
- High escalation rates (may indicate prompt or tool issues)
- High tool error rates (may indicate backend service issues)
- Elevated response latency (may indicate model or KB issues)
- Low recommendation acceptance rates (may indicate relevance issues)

---

## Troubleshooting

### AI Agent Not Responding

- Verify the assistant ID is correct and the agent is associated with the session
- Check that the Lex bot is properly configured and the flow invokes it
- Verify the AI agent type is set as default or overridden in the session
- Check CloudWatch logs for errors in the Q Connect service

### Recommendations Not Appearing

- Verify Contact Lens is enabled and streaming transcripts
- Check that knowledge bases have indexed content
- Verify the Q Connect session was created for the contact
- Ensure the agent workspace has the Q Connect panel enabled

### Tool Invocation Failures

- Check MCP tool endpoint availability and authentication
- Verify input schema matches what the orchestrator is sending
- Review CloudWatch logs for tool invocation errors
- Test the tool endpoint independently

### Poor Response Quality

- Review and customize prompts for your use case
- Ensure knowledge base content is relevant and up-to-date
- Check guardrail settings -- overly restrictive filters may block good responses
- Try switching search type between SEMANTIC and HYBRID
- Increase `maxResults` for knowledge base retrieval

---

## Legacy generative-AI self-service

Before agentic self-service, generative-AI self-service returned control to the contact flow whenever a **custom tool** was selected (rather than continuing the conversation). It is **legacy** (no new features) — use agentic self-service for new builds.

- Enable the `AMAZON.QinConnectIntent` intent in the Lex bot; add a **Connect assistant** block; add **Get customer input**; optionally a **Check contact attributes** block (Namespace = Lex, Key = Session attributes, Session Attribute Key = `Tool`) to branch on the selected tool.
- **Default system tools:** `QUESTION`, `ESCALATION` (takes the Error branch of Get customer input), `CONVERSATION`, `COMPLETE`, `FOLLOW_UP_QUESTION`.
- Tools here are defined in **YAML** (not JSON). Example:

```yaml
tools:
- name: HANDOFF
  description: Hand off the customer to a human agent with a summary of why they're calling.
  input_schema:
    type: object
    properties:
      message:
        type: string
        description: Restatement of what they're calling about; end by saying you're handing off to an agent.
      summary:
        type: string
        description: Reasons the customer reached out, as <SummaryItems><Item>...</Item></SummaryItems>.
    required:
    - message
    - summary
```

**Recommendation:** prefer agentic self-service (orchestrator + MCP tools) for new implementations — multi-step reasoning, in-conversation tool invocation, and access to new features.
