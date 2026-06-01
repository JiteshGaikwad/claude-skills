# Connect AI Agents — AI Prompts

An **AI prompt** is a task/instruction for the LLM. Connect provides editable YAML templates so non-developers can customize them. You create/edit prompts in the admin site under **AI agent designer → AI prompts** (needs the **AI agent designer → AI prompts → Create** permission), or via the `qconnect` API/CLI. Publishing a prompt creates a **version** you add to an AI agent to override a system default. See [connect-ai-agents.md](connect-ai-agents.md) for how prompts attach to agents.

---

## AI prompt types

| Prompt type | Purpose |
|---|---|
| **Orchestration** | Orchestrates different use cases per customer needs (used by Orchestration AI agents). |
| **Answer generation** | Generates a solution to a query using knowledge base excerpts. |
| **Intent labelling generation** | Generates intents for the interaction; shown in the Connect assistant widget for agent selection. |
| **Query reformulation** | Constructs a query to search for relevant KB excerpts. |
| **Self-service pre-processing** | Evaluates the conversation and selects the corresponding tool to generate a response. |
| **Self-service answer generation** | Generates a solution to a query using KB excerpts (self-service context). |
| **Email response** | Helps send an email response of a conversation script to the customer. |
| **Email overview** | Provides an overview of email content. |
| **Email generative answer** | Generates answers for email responses. |
| **Email query reformulation** | Reformulates the query for email responses. |
| **Note taking** | Generates concise, structured, actionable notes in real time from the live conversation. |
| **Case Summarization** | Summarizes a case. |

---

## The four elements of a prompt

- **Instructions** — the task for the LLM.
- **Context** — external information to guide the model.
- **Input data** — the input you want a response for.
- **Output indicator** — the output type/format.

---

## Create / edit / publish (console)

1. **AI agent designer → AI prompts → Create AI Prompt**.
2. Choose the **AI Prompt type** (table above) → **Create**. The **AI Prompt builder** opens with the template.
3. *(Optional)* In **Models**, change the model (defaults to the Region's system-default model).
4. Edit the YAML template (placeholders are written in plain-language YAML; invalid edits surface an error on Save).
5. **Save** for work-in-progress; **Publish** to create a version usable in production / to override a default.

---

## YAML formats

Two API formats; the format dictates required/optional fields. Deleting a required field or adding unsupported text errors on **Save**.

### MESSAGES — for prompts that do NOT interact with a knowledge base

| Field | Req | Notes |
|---|---|---|
| `system` | Optional | System prompt — context/role/goal for the LLM. |
| `messages` | **Required** | List of input messages. |
| `messages[].role` | Required | `user` or `assistant`. |
| `messages[].content` | Required | The turn content. |
| `tools` | Optional | List of tools the model may use. |
| `tools[].name` | Required | Tool name. |
| `tools[].description` | Required | Tool description. |
| `tools[].input_schema` | Required | JSON Schema for the tool params (see below). |

`input_schema` supported objects: `type` (only `"string"` supported), `enum` (optional allowed values), `default` (optional — makes the param effectively optional), `properties` (required), `required` (required).

```yaml
system: You are an intelligent assistant that assists with query construction.
messages:
- role: user
  content: |
    Here is a conversation between a customer support agent and a customer
    <conversation>
    {{$.transcript}}
    </conversation>
    ... formulate a query ... output <query>search query</query> and nothing else.
```

### TEXT_COMPLETIONS — for **Answer generation** prompts that interact with a KB

Uses the `contentExcerpt` and `query` variables. Only one required field:

| Field | Req | Notes |
|---|---|---|
| `prompt` | **Required** | The prompt for the LLM to complete. |

```yaml
prompt: |
  You are a multi-lingual assistant. You will receive a <query>, a list of <search_result>
  documents, and a MANDATORY <locale> (which overrides any other language).
  ... cite sources via <sources><source>ID</source></sources> ...
  Input:
  {{$.contentExcerpt}}
  <query>{{$.query}}</query>
  <locale>{{$.locale}}</locale>
```

### Assistant message-prefill caveat

The default template ends with an assistant-message prefill that reinforces wrapping output in `<message>` tags:

```yaml
  - role: assistant
    content: <message>
```

You **must delete those two lines** if you choose any of these models, or the prompt won't work correctly: `us/eu/jp/au/global.anthropic.claude-sonnet-4-6`, `openai.gpt-oss-20b-1:0`, `openai.gpt-oss-120b-1:0`. Leave the template unchanged for all other supported models. The `conversationHistory` entry stays.

---

## Variables

| Type | Format | Description |
|---|---|---|
| System | `{{$.transcript}}` | Up to the 3 most recent conversation turns. |
| System | `{{$.contentExcerpt}}` | Relevant KB document excerpts. |
| System | `{{$.locale}}` | Locale used for LLM inputs/outputs. |
| System | `{{$.query}}` | The query a Connect AI agent constructed to search the KB. |
| Customer-provided | `{{$.Custom.<VARIABLE_NAME>}}` | Any value added to the session (see AI agent sessions in [connect-ai-agents.md](connect-ai-agents.md)). If absent in the session, it interpolates as an empty string — write the prompt to handle absence. |

---

## Optimization & prompt caching

- Position **static content before variables** — caching only applies to portions that don't change between requests.
- Use prompt **prefixes of at least 1,000 tokens**; add more static content to improve latency.
- With multiple variables, create a separate ≥1,000-token prefix per variable — cache is separated per variable, and only static portions meeting token requirements benefit.
- **Prompt caching is enabled by default for all customers.**

Prompt-caching-supported models include: Claude Opus 4, Claude Sonnet 4 (us/eu/apac), Claude 3.7 Sonnet (us/eu), Claude 3.5 Haiku (incl. us), Nova Pro (us/eu/apac), Nova Lite (us/apac), Nova Micro (us/eu/apac). (Token requirements per the Amazon Bedrock prompt-caching docs.)

---

## Models

In the AI Prompt builder, the **Models** dropdown lists models available for your instance's Region; the system-default model for the Region is pre-selected (e.g. `us.amazon.nova-pro-v1:0 (Cross Region)(System Default)`). Many models use cross-region inference (CRIS / Global CRIS) for performance and availability.

### System-prompt default models (by Region)

| System prompt | us-east-1 / us-west-2 | eu-central-1 | ap-southeast-1 / ap-northeast-2 |
|---|---|---|---|
| AgentAssistanceOrchestration | Claude 4.5 Sonnet (CRIS) | Claude 4.5 Sonnet (CRIS) | Claude 4.5 Sonnet (Global CRIS) |
| AnswerGeneration / Email* | Claude Sonnet 4.5 (CRIS) | Claude Sonnet 4.5 (CRIS) | Claude Sonnet 4.5 (Global CRIS) |
| CaseSummarization | Claude Sonnet 4.5 (CRIS) | Claude Sonnet 4.5 (CRIS) | Claude Sonnet 4 (CRIS) |
| IntentLabelingGeneration | Nova Pro (CRIS) | Nova Pro (CRIS) | Nova Pro (CRIS) |
| QueryReformulation | Nova Lite (CRIS) | Nova Lite (CRIS) | Nova Lite (CRIS) |
| NoteTaking / SalesAgent | Claude 4.5 Haiku (CRIS) | Claude 4.5 Haiku (CRIS) | Claude 4.5 Haiku (Global CRIS) |
| SelfServiceAnswerGeneration / SelfServicePreProcessing | Nova Pro (CRIS) | Nova Pro (CRIS) | Nova Pro (CRIS) |
| SelfServiceOrchestration | Claude 4.5 Haiku (CRIS) | Claude 4.5 Haiku (CRIS) | Nova Pro (CRIS) |

> Notes: `SalesAgent` is **N/A** in eu-west-2. `ca-central-1` and `eu-west-2` use a mix of `global.*` CRIS, `anthropic.claude-3-haiku`, and region-local Nova for some prompts. The full per-Region matrix (8 Regions) is in the AWS "Create AI prompts" doc — verify exact model IDs there for a specific Region before relying on them.

### Custom-prompt supported models (examples by Region)

- **us-east-1 / us-west-2:** Claude 3.5 Haiku, Claude 3.7 Sonnet, Claude Sonnet 4, Claude 4.5 Haiku/Sonnet (CRIS), `global.*` Claude 4.5 (Global CRIS), Nova Pro/Lite/Micro (CRIS), `anthropic.claude-3-haiku`, OpenAI `gpt-oss-20b` / `gpt-oss-120b`.
- **eu-west-2 / eu-central-1:** `eu.*` Claude 4.5 Haiku/Sonnet, Nova Pro/Lite/Micro, Claude 3.7 Sonnet, `global.*` Claude 4.5, OpenAI gpt-oss-20b/120b.
- **ap-northeast-1/2, ap-southeast-1/2:** `apac.*` Nova Pro/Lite/Micro, `apac.*` Claude 3.5 Sonnet v2 / Claude Sonnet 4 / Claude 3 Haiku, `jp.*`/`au.*` Claude 4.5 Sonnet (CRIS), `global.*` Claude 4.5.

---

## CLI

```bash
# Create a prompt (MESSAGES)
aws qconnect create-ai-prompt --assistant-id <ASSISTANT_ID> --name example \
  --api-format MESSAGES --model-id us.anthropic.claude-3-7-sonnet-20250219-v1:0 \
  --template-type TEXT --type QUERY_REFORMULATION --visibility-status PUBLISHED \
  --template-configuration '{"textFullAIPromptEditTemplateConfiguration":{"text":"<SERIALIZED_YAML>"}}'

# Create a prompt (TEXT_COMPLETIONS) — use --api-format TEXT_COMPLETIONS --type ANSWER_GENERATION

# Version a prompt (immutable, runtime-usable). Qualify the ID as <AI_PROMPT_ID>:<VERSION_NUMBER>.
aws qconnect create-ai-prompt-version --assistant-id <ASSISTANT_ID> --ai-prompt-id <AI_PROMPT_ID>

# List SYSTEM prompt versions (omit --origin SYSTEM to also list your customized prompts; used to reset to defaults)
aws qconnect list-ai-prompt-versions --assistant-id <ASSISTANT_ID> --origin SYSTEM
```

---

## Amazon Nova Pro — tool_use example format

For **self-service pre-processing** prompts using **Amazon Nova Pro**, `tool_use` examples must be written in **Python-like format**, not JSON:

```
<tool>
    [TOOL_NAME(input_param1="{value1}", input_param2="{value2}")]
</tool>
```

(For all other models, the JSON `{"type":"tool_use","name":...,"input":{...}}` form is used.)
