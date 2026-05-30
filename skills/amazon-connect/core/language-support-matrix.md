# Amazon Connect — Language Support Matrix

Amazon Connect supports different languages depending on the feature. There is no single global language list — text-to-speech, speech recognition, Contact Lens analytics, the agent UI, and the admin website each support their own set. This page consolidates the per-feature language support documented in the Admin Guide.

Language codes are W3C language identification tags (ISO 639 for the language, ISO 3166 for the country/region), for example `en-US`, `es-US`, `fr-CA`.

> **The supported-language lists expand over time.** Treat this page as a snapshot. For the current authoritative list, see [Languages supported by Amazon Connect features](https://docs.aws.amazon.com/connect/latest/adminguide/supported-languages.html), and the per-service references linked in each section below.

---

## At a glance

| Feature area | What it covers | Language model |
| --- | --- | --- |
| AI features | Contact Lens (real-time & post-call), AI Agents, automated evaluations, summaries, sentiment, redaction, pattern matching | Per-locale matrix (below) |
| Contact Control Panel (CCP) | Agent softphone UI | Fixed UI language set |
| Chat message content | Customer ↔ agent chat text | Full Unicode — any language |
| Quick responses | Canned chat/email replies | English only |
| Admin website | Connect admin console UI | Fixed UI language set |
| Cases / Forecasting, capacity, scheduling | Admin UI for those modules | See AWS docs |
| Amazon Lex (ASR/NLU) | Conversational IVR & chatbots | Lex V2 locale list |
| Amazon Polly (TTS) | Synthesized speech in flows | Polly voice/language list |

Self-service chatbots use neural text-to-speech (TTS) in more than 30 languages and automated speech recognition (ASR) in over 25 languages/locales (via Amazon Lex).

---

## AI features (Contact Lens, AI Agents, evaluations, summaries)

The following table lists languages supported by AI features. A check (✓) means the feature supports that locale.

Columns:

- **AI Agents** — Amazon Connect AI Agents
- **Auto eval†** — Automated performance evaluations (generative AI)
- **Post-call** — Post-call analytics
- **Real-time** — Real-time call analytics
- **Email/Chat** — Email and chat analytics
- **Summaries** — Post-contact summaries
- **Sentiment** — Sentiment analysis
- **Redaction** — Sensitive-data redaction
- **Pattern** — Pattern match rules

> **Note on completeness:** The matrix below reflects only the locales present in the source extract used to build this page. The full table in the Admin Guide contains many more locales (Spanish, German, Italian, Portuguese, Japanese, Korean, Chinese variants, and others). For every locale and column, use the [AWS supported-languages page](https://docs.aws.amazon.com/connect/latest/adminguide/supported-languages.html) as the authoritative source.

| Language | Code | AI Agents | Auto eval† | Post-call | Real-time | Email/Chat | Summaries | Sentiment | Redaction | Pattern |
| --- | --- | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| Afrikaans (South Africa) | af-ZA | ✓* | ✓* | ✓* |  |  |  |  |  | ✓* |
| Arabic (Gulf) | ar-AE | ✓* | ✓* | ✓* | ✓* |  | ✓* |  |  | ✓* |
| Arabic (Modern Standard / Saudi Arabia) | ar-SA | ✓* | ✓* | ✓* | ✓* |  |  |  |  | ✓* |
| English (Singapore) | en-SG | ✓* | ✓ |  |  |  |  |  |  |  |
| English (South Africa) | en-ZA | ✓* | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| English (United Kingdom) | en-GB | ✓* | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| English (US) | en-US | ✓* | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| English (Wales) | en-WL | ✓* | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Estonian (Estonia) | et-ET | ✓* | ✓* |  | ✓* |  |  |  |  | ✓* |
| Farsi (Iran) | fa-IR | ✓* | ✓* | ✓* | ✓* |  |  |  |  | ✓* |
| Finnish (Finland) | fi-FI | ✓* | ✓* | ✓* | ✓* |  |  |  |  | ✓* |
| French (Belgium) | fr-BE | ✓* | ✓ |  |  |  |  |  |  |  |
| French (Canada) | fr-CA | ✓* | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Ukrainian (Ukraine) | uk-UA | ✓* | ✓* | ✓* |  |  |  |  |  | ✓* |
| Vietnamese (Vietnam) | vi-VN | ✓* | ✓* | ✓* | ✓* |  |  |  |  | ✓* |
| Welsh (United Kingdom) | cy-UK | ✓* |  |  |  |  |  |  |  |  |
| Xhosa (South Africa) | xh-UK | ✓* |  |  |  |  |  |  |  |  |
| Zulu (South Africa) | zu-ZA | ✓* | ✓* | ✓* |  |  |  |  |  | ✓* |

### Caveats and footnotes

- **`*` (region exclusion):** Marked entries are **not available** in the **Africa (Cape Town)** AWS Region or in **AWS GovCloud (US-West)**.
- **Redaction is post-call / chat only:** Redaction support applies to **post-call analytics and chat analytics**. It is **not supported for real-time call analytics**.
- **`†` Automated performance evaluations:** You can manually complete performance evaluations in **any language**. Automated (generative AI–filled) evaluations are **not available** in: **Africa (Cape Town)**, **Asia Pacific (Mumbai)**, **Asia Pacific (Seoul)**, and **AWS GovCloud (US-West)**.

---

## Real-time vs post-call (Contact Lens)

Contact Lens analytics splits into two analysis modes with different language coverage:

| | Real-time call analytics | Post-call analytics |
| --- | --- | --- |
| When it runs | During the live call | After the call ends |
| Redaction | **Not supported** | Supported (see redaction column) |
| Language coverage | Subset of post-call locales (see the matrix — some locales support post-call but not real-time, e.g. af-ZA, ar-SA-equivalent rows, uk-UA, zu-ZA) | Broader |

The per-locale **Real-time** and **Post-call** columns in the matrix above are the source of truth for which mode each language supports.

---

## Language support for Connect AI Agents

Additional detail beyond the per-locale matrix:

| AI Agent capability | Language support |
| --- | --- |
| Agent assistance — Proactive Intent Detection (from transcripts) | English, Spanish, Portuguese, French, Korean, Japanese, Chinese |
| Self-service — default | English |
| Self-service — other languages | Requires prompt customization |
| Guardrails | Same languages as **Amazon Bedrock Guardrails (classic tier)** — see [Languages supported by Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-supported-languages.html). Evaluating text in other languages is ineffective. |

To support self-service in languages other than English, customize these prompts and specify the target language in them:

- `SELF_SERVICE_PRE_PROCESSING`
- `SELF_SERVICE_ANSWER_GENERATION`

---

## Contact Control Panel (CCP)

The CCP agent UI is localized to a fixed set of languages. The latest CCP version supports more languages than the earlier version.

| CCP version | Supported UI languages |
| --- | --- |
| Latest | Chinese (Simplified), Chinese (Traditional), English, French, German, Italian, Japanese, Korean, Portuguese (Brazilian), Spanish |
| Earlier | English, French, German, Italian, Japanese, Korean, Portuguese (Brazilian), Spanish |

Default system messages (such as idleness messages) are displayed to agents in the CCP in all supported languages.

---

## Chat, quick responses, and admin UI

| Feature | Language support |
| --- | --- |
| Chat message content | **Full Unicode** — chat with customers in any language. |
| Quick responses (chat & email) | **English only.** |
| Admin website (console UI) | Chinese (Simplified), Chinese (Traditional), English, French, German, Italian. |

### In-flight chat redaction (Conversational Analytics)

Built-in sensitive-data redaction for chat message processing supports multiple languages, including **English, French, Portuguese, German, Italian, and Spanish variants**. For the full list, see the Conversational Analytics supported-languages reference.

---

## Speech: Lex (ASR/NLU) and Polly (TTS)

Conversational IVR/chatbots get speech recognition and understanding from Amazon Lex, and synthesized speech from Amazon Polly. These each have their own language/locale lists maintained outside the Connect Admin Guide.

| Capability | Where it comes from | Reference |
| --- | --- | --- |
| ASR / NLU (speech recognition, intent) | Amazon Lex V2 | [Languages and locales supported by Amazon Lex V2](https://docs.aws.amazon.com/lexv2/latest/dg/how-languages.html) |
| TTS (text-to-speech voices) | Amazon Polly | [Voices in Amazon Polly](https://docs.aws.amazon.com/polly/latest/dg/available-voices.html) |

### Setting language and voice in flows

- Pass the language code into the language attribute, e.g. `en-US` or `ar-AE`. Pass the voice by name, e.g. `Joanna` or `Hala`.
- **Speaking styles** can be `None`, `Conversational`, or `Newscaster`. Newscaster and Conversational styles are available (neural engine) for: **Matthew (en-US)**, **Joanna (en-US)**, **Lupe (es-US)**, **Amy (en-GB)**.
- For the Joanna and Matthew neural voices in American English (en-US), you can additionally specify a **Newscaster** speaking style.
- Some voices (e.g. **Ruth (en-US)**) do **not** support the standard engine — you must explicitly specify a supported engine (neural) or the Set voice block takes the error branch.
- The Set voice block does **not** validate the language code against the voice; only the voice is used to synthesize speech. A mismatched language code can cause erroneous behavior when paired with Lex V2 bots.

### Lex V2 language matching caveat

When using an Amazon Lex V2 bot, the Connect language attribute **must match the bot's language model** (this differs from Lex Classic):

- If the bot is built with a non-`en-US` model (e.g. `en_AU`, `fr_FR`, `es_ES`), choose a corresponding voice **and** use **Set language attribute**.
- If you use a non-`en-US` voice with a Lex V2 bot and do not set the language attribute, the Get customer input block errors.
- For multi-language bots (e.g. `en_AU` and `en_GB`), choose Set language attribute for one of the languages.

### Amazon Nova Sonic (Speech-to-Speech) — launch voice set

When configuring Nova Sonic as a Speech-to-Speech model, use a Nova Sonic–compatible expressive voice in the Set voice block:

| Voice | Locale | Gender |
| --- | --- | --- |
| Matthew | en-US | Masculine |
| Amy | en-GB | Feminine |
| Olivia | en-AU | Feminine |
| Lupe | es-US | Feminine |

---

## System language attribute

In flows, contact attributes, and Lambda payloads, the contact language is carried in the `$.LanguageCode` system attribute (Java `java.util.Locale` format, e.g. `en-US`, `ja-JP`). For multi-locale bots, set `$.LanguageCode` early in the flow using a Set contact attributes block.
