# Connect AI Agents — AI Guardrails

An **AI guardrail** implements safeguards based on your use cases and responsible-AI policies — filtering harmful/inappropriate content, redacting PII, and limiting hallucinations. Connect AI agents use **Amazon Bedrock guardrails** under the hood. You create/edit them in the admin site under **AI agent designer → AI guardrails** (needs **AI agent designer → AI guardrails → Create**), or via the `qconnect` API/CLI. A customized AI agent can attach **one** guardrail; see [connect-ai-agents.md](connect-ai-agents.md).

---

## Things to know (Connect-specific)

- You can create **up to three** custom guardrails per business.
- Guardrails support the same languages as **Amazon Bedrock guardrails classic tier**; evaluating other languages is ineffective.
- The **Image content filter is NOT supported** in Connect — text only.
- AWS recommends benchmarking configurations before relying on them; some combinations have unintended effects.
- **Streaming tradeoff:** with guardrails on a streaming response, text chunks must be buffered and scanned before delivery, adding latency — mainly to **time-to-first-token**. Shorter responses feel proportionally more latency (a minimum buffer must still be scanned). It's an inherent safety-vs-speed tradeoff; consider applying guardrails selectively for latency-sensitive use cases.

---

## Create / publish (console)

1. **AI agent designer → AI guardrails → Create Guardrail** → enter name + description → **Create**. (If three already exist, you get an error — edit or delete one instead.)
2. On the **AI Guardrail builder**, configure the policy sections below.
3. **Save** (the `Latest:Draft` version always returns the saved draft).
4. **Publish** — sets Visibility status to Published and creates a new version (`Latest:Published` returns the saved published state).

---

## Policy types

### 1. Content filters

Block harmful content by category, with strength levels **NONE / LOW / MEDIUM / HIGH** configured **separately for input vs output** (`inputStrength` / `outputStrength`). Higher = more aggressive. Per-detection action is **Block** or **Detect (no action)** (`inputAction`/`outputAction` = `BLOCK | NONE`). Input modality is `TEXT` only.

| Category (API `type`) | Filters |
|---|---|
| **Hate** (`HATE`) | Discriminates/insults/dehumanizes based on identity (race, ethnicity, gender, religion, orientation, ability, national origin). |
| **Insults** (`INSULTS`) | Demeaning, humiliating, mocking, bullying language. |
| **Sexual** (`SEXUAL`) | Sexual interest/activity via references to body parts, traits, or sex. |
| **Violence** (`VIOLENCE`) | Glorification of or threats to inflict physical pain/harm. |
| **Misconduct** (`MISCONDUCT`) | Seeking/providing info on criminal activity or defrauding/harming others. |
| **Prompt Attack** | Prompt-injection / jailbreak attempts. (Configured in the builder; not a `CreateGuardrail` content-filter enum value.) |

```jsonc
"contentPolicyConfig": { "filtersConfig": [
  { "inputStrength": "HIGH", "outputStrength": "HIGH", "type": "INSULTS" }
]}
```

### 2. Denied topics

Topics to refuse. **Up to 30** per guardrail. Each: **Name** (a noun/phrase, not a description), **Definition** (≤ **200 chars**), optional **sample phrases** (≤ **5**, each ≤ **100 chars**). Type is `DENY`; per-direction via `inputEnabled`/`outputEnabled`, `inputAction`/`outputAction` (`BLOCK | NONE`). Don't put examples/instructions in the definition, and don't use denied topics to capture entities or single words (use word/PII filters).

```jsonc
"topicPolicyConfig": { "topicsConfig": [
  { "name": "Financial Advice",
    "definition": "Investment advice: financial inquiries/recommendations aimed at generating returns.",
    "examples": ["Which stocks should I invest in?", "Can you manage my finances?"],
    "type": "DENY" }
]}
```

### 3. Contextual grounding check

Detects/filters hallucinations against a reference source + the user query. Supports **summarization, paraphrasing, question answering** — **not** conversational QA / chatbot. Output-only (a model response is required). Two filter types, each with a confidence threshold:

- **GROUNDING** — is the response factually grounded in the source? (New info not in the source = un-grounded.)
- **RELEVANCE** — is the response relevant to the query?

**Threshold range: 0 to 0.99** (1 is invalid — would block everything). Responses below threshold are blocked; higher threshold = more aggressive. Character limits: grounding source ≤ **100,000**, query ≤ **1,000**, response ≤ **5,000**. Relevance is evaluated per chunk — if any chunk is relevant, the whole response is considered relevant (in streaming this can surface an irrelevant chunk before flagging).

```jsonc
"contextualGroundPolicyConfig": { "filtersConfig": [
  { "type": "RELEVANCE", "threshold": 0.50 }
]}
```

> The Connect `qconnect` CLI key is `contextualGroundPolicyConfig` (Bedrock's own `CreateGuardrail` uses `contextualGroundingPolicyConfig`).

### 4. Word filters

Block words/phrases/profanity by **exact match**.

- **Profanity filter** — managed, continually-updated list (`managedWordListsConfig`, `type: PROFANITY`).
- **Custom words** (`wordsConfig`) — each item is a word or a phrase of **up to 3 words**; **up to 10,000** items. Console input: type, upload `.txt`/`.csv` (one per line, no header), or an S3 object. **API/SDK supports text only** (no file/object upload).

```jsonc
"wordPolicyConfig": { "wordsConfig": [ { "text": "Nvidia" } ] }
```

### 5. Sensitive information filters (PII)

Block or mask PII / custom regex in inputs and responses (ML-based, context-dependent). **Text output only** — does not detect PII in `tool_use` output params. Actions per direction:

- **BLOCK** — block all content, return the configured message.
- **ANONYMIZE** (console: "Mask") — redact and replace with the type, e.g. `{NAME}`, `{EMAIL}`.
- **NONE** — detect only (returns detection info, no action).

Built-in PII entity types:

- **General:** `ADDRESS`, `AGE`, `NAME`, `EMAIL`, `PHONE`, `USERNAME`, `PASSWORD`, `DRIVER_ID`, `LICENSE_PLATE`, `VEHICLE_IDENTIFICATION_NUMBER`.
- **Finance:** `CREDIT_DEBIT_CARD_CVV`, `CREDIT_DEBIT_CARD_EXPIRY`, `CREDIT_DEBIT_CARD_NUMBER`, `PIN`, `INTERNATIONAL_BANK_ACCOUNT_NUMBER`, `SWIFT_CODE`.
- **IT:** `IP_ADDRESS`, `MAC_ADDRESS`, `URL`, `AWS_ACCESS_KEY`, `AWS_SECRET_KEY`.
- **USA:** `US_BANK_ACCOUNT_NUMBER`, `US_BANK_ROUTING_NUMBER`, `US_INDIVIDUAL_TAX_IDENTIFICATION_NUMBER`, `US_PASSPORT_NUMBER`, `US_SOCIAL_SECURITY_NUMBER`.
- **Canada:** `CA_HEALTH_NUMBER`, `CA_SOCIAL_INSURANCE_NUMBER`.
- **UK:** `UK_NATIONAL_HEALTH_SERVICE_NUMBER`, `UK_NATIONAL_INSURANCE_NUMBER`, `UK_UNIQUE_TAXPAYER_REFERENCE_NUMBER`.
- **Custom regex** (`regexesConfig`): `name` (1–100 chars), `pattern` (1–500 chars), optional `description` (1–1000 chars), `action`, per-direction enabled. **Lookaround is not supported.**

Notes: detection works better with surrounding context (avoid single words). Masking applies to prompts and responses only — **not** to model-invocation logs (use CloudWatch log data protection) and **not** to guardrail trace output (returns the original value by design).

```jsonc
"sensitiveInformationPolicyConfig": { "piiEntitiesConfig": [
  { "type": "CREDIT_DEBIT_CARD_NUMBER", "action": "BLOCK" }
]}
```

### 6. Blocked messaging

Customize the message shown when input/output is blocked (`blockedInputMessaging` / `blockedOutputsMessaging`). Default: `"Blocked input text by guardrail."` Change it in the builder's **Blocked messaging** section → Save → Publish.

---

## CLI

```bash
aws qconnect update-ai-guardrail --cli-input-json '{
  "assistantId": "<ASSISTANT_ID>",
  "aiGuardrailId": "<GUARDRAIL_ID>",
  "blockedInputMessaging": "Blocked input text by guardrail",
  "blockedOutputsMessaging": "Blocked output text by guardrail",
  "visibilityStatus": "PUBLISHED",
  "contentPolicyConfig": { "filtersConfig": [ { "inputStrength":"HIGH","outputStrength":"HIGH","type":"INSULTS" } ] }
}'
```

Swap in any single policy object (`topicPolicyConfig`, `wordPolicyConfig`, `contextualGroundPolicyConfig`, `sensitiveInformationPolicyConfig`) as needed.
