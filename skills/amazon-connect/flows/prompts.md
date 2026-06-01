# Prompts in Amazon Connect

Prompts are audio played in flows (greetings, messages, hold music). Connect ships a default prompt library; you can add your own recordings, generate audio from text with Amazon Polly (text-to-speech), or play audio stored in Amazon S3. Manage prompts in the admin site under **Routing > Prompts**, or programmatically via the [Prompt actions API](api/) (`CreatePrompt`, etc.).

> For SSML formatting of text-to-speech prompts, see [ssml.md](ssml.md). For the voice/language/engine reference, see "Set voice block" below.

---

## Create a prompt (admin site)

Requires the security profile permission **Numbers and flows > Prompts - Create**.

1. In the navigation menu, choose **Routing**, then **Prompts**.
2. Choose **Add prompt**.
3. Enter a **name**.
4. Enter a **Description** (recommended — useful for accessibility).
5. Choose one of:
   - **Upload** — choose a `.wav` file you have legal permission to use.
   - **Record** — choose **Start recording**, speak into your microphone, then **Stop recording**. Use **Crop** to trim, or **Clear recording** to start over.
6. In **Prompt Settings**, optionally add **tags** to identify, organize, search, filter, and control access to the prompt.

On the **Prompts** page you can filter by **Name**, **Description**, and **Tags**, and use the **Copy** icon to copy a prompt's full ARN (needed when selecting prompts dynamically).

To create prompts programmatically, use `CreatePrompt`.

### Supported file types

- Upload a pre-recorded `.wav` file, or record one in the web app.
- Recommended: **8 kHz `.wav`**, **< 50 MB**, **< 5 minutes**.
- Higher-rate files (e.g. 16 kHz) are **downsampled to 8 kHz** because of PSTN limitations (G.711), which can reduce audio quality.

### Maximum length

Prompts must be **< 50 MB** and **< 5 minutes** long.

### Bulk upload

**Not supported** — there is no bulk prompt upload via the console, API, or CLI. Each prompt is added individually.

---

## Text-to-speech (Amazon Polly)

Connect converts text to lifelike speech with Amazon Polly. You can enter TTS in these flow blocks:

- **Get customer input**
- **Loop prompts**
- **Play prompt**
- **Store customer input**

In the block **Properties**, choose **Text-to-speech**, then enter plain text (e.g. *"Thank you for calling"*) or SSML. To enter SSML, set the **Interpret as** field to **SSML** (default is **Text**). See [ssml.md](ssml.md) for the supported tags and the strict-use rule.

**Character limits** (TTS field on Play prompt / Get customer input): **3,000 billed characters**, **6,000 total characters** max.

### Pricing

- Polly **Neural** and **Standard** voices are **free** in Connect.
- **Generative** voices are **billed** (see Amazon Polly pricing). If your account is onboarded to Next Gen Amazon Connect, generative voices are included in Next Gen pricing.
- **Custom/Brand voices** associated with your account are also billed.

### Use the best available voice

Polly periodically releases improved voices/speaking styles. You can have Connect auto-resolve TTS to the most natural variant of a voice (e.g. flows using Joanna resolve to Joanna's conversational style; if no neural version exists, Connect defaults to the standard voice).

To enable: Amazon Connect console → choose your instance → **Flows** → in the **Amazon Polly** section, choose **Use the best available voice**.

---

## Set voice block (voice, language, engine, speaking style)

The **Set voice** block sets the TTS language and voice for the rest of the flow. After it runs, every later TTS invocation resolves to the selected voice/engine.

- **Default voice:** Joanna (Conversational speaking style, neural).
- Choose **Override speaking style** to use Neural or Generative voices.
  - **Neural** voices improve pitch, inflection, intonation, and tempo.
  - **Generative** voices are the most human-like and adaptive (billed; included in Next Gen pricing).
- In **chat/task/email**, this block has no effect and takes the **Success** branch (voice-only block). Branches: **Success** and **Error**.

**Supported channels:** Voice = Yes; Chat / Task / Email = No (Success branch).

### Setting values dynamically

You can set language, voice, engine, and style dynamically (from contact attributes), subject to these rules:

- If **language** is selected dynamically, the **voice** must also be selected dynamically.
- If the **voice** is dynamic and the speaking style is overridden, the **engine** and **style** must also be dynamic.
- If the voice/engine is invalid, or the voice doesn't support the engine, the **Error** branch is taken.
- Language code is only passed into the flow if **Set language attribute** is selected. Invalid language codes do **not** take the Error branch here, but can cause erroneous behavior with Lex V2 bots.
- If a Play prompt follows the **Error** branch, it defaults to **Joanna/standard**.
- If the defined speaking style isn't supported by the voice, the **None** style is used.

### Configuration values

- **Language:** pass the language code, e.g. `en-US`, `ar-AE`.
- **Voice:** pass the voice name, e.g. `Joanna`, `Hala`, `Ruth`.
- **Engine:** Connect supports `standard`, `neural`, and `generative`. **If you don't specify an engine, `standard` is used by default.** Some voices (e.g. **Ruth**, en-US) do not support standard — you must specify a supported engine or the block fails / takes Error.
- **Speaking style:** `None`, `Conversational`, or `Newscaster`. Conversational and Newscaster are available on the **neural** engine for: **Matthew (en-US), Joanna (en-US), Lupe (es-US), Amy (en-GB)**.

| Language | Voice | Engine | Style | Result |
|---|---|---|---|---|
| en-US | Ruth | (none) | (none) | **Error** — defaults to standard; Ruth doesn't support standard. |
| en-US | Ruth | neural | none | **Success** — Ruth supports neural. |
| en-US | Ruth | neural | conversational | **Success** — style unsupported by Ruth, so synthesized with no style (no error). |
| ar-AE | Ruth | neural | none | **Success** — block doesn't validate language code; only the voice is used. (Wrong language code can misbehave with Lex V2.) |

### Generative engine IAM note

If your instance predates October 2018 and you migrated to a Service-Linked Role, add this to your Service Role to use generative engines:

```json
{
  "Sid": "AllowPollyActions",
  "Effect": "Allow",
  "Action": ["polly:SynthesizeSpeech"],
  "Resource": ["*"]
}
```

### Lex V2 bots

If using a Lex V2 bot, the Connect language attribute must match the bot's language model:

- For a non-`en-US` bot (e.g. `en_AU`, `fr_FR`, `es_ES`), pick a matching **Voice** and select **Set language attribute**.
- If you use a non-`en-US` voice with a Lex V2 bot and don't select **Set language attribute**, the **Get customer input** block errors.
- For multi-language bots (e.g. `en_AU` + `en_GB`), select **Set language attribute** for one of the languages.

> A short pointer page also exists ("Choose the text-to-speech voice and language") whose tip notes: if you enter text not supported for the chosen Polly voice, that part won't play, but other supported text in the prompt still plays.

---

## Dynamic text strings in Play prompt

Personalize a TTS message by referencing contact attributes in the **Play prompt** text:

- **External** attributes (e.g. from a Lambda return): `$.External.FirstName $.External.LastName`
  - *"Hello `$.External.FirstName $.External.LastName`, thank you for calling."*
- **User-defined** attributes set earlier (e.g. via Set contact attributes or the API): `$.Attributes.nameOfAttribute`
  - *"Hello `$.Attributes.FirstName $.Attributes.LastName`, thank you for calling."*

**Dynamic key resolution with backticks:** enclose a dynamic key in backticks to resolve it at runtime. If the key to use is stored in `$.Attributes.NameToPlay`:

```
Hello $.External.['`$.Attributes.NameToPlay`'], thank you for calling.
```

---

## Dynamically select which prompt to play

Choose the prompt at runtime via an attribute:

1. Add **Set contact attributes** blocks, each configured to hold the appropriate audio prompt (e.g. one for open hours, one for closed). Name the user-defined attribute anything (e.g. `CompanyWelcomeMessage`).
2. In the **Play prompt** block, choose **User Defined** and enter that attribute name.
3. Connect the **Set contact attributes** blocks to the **Play prompt** block.

This lets the same Play prompt block play different recordings depending on which Set contact attributes block ran. (When referencing a prompt by ARN, copy the full ARN from the Prompts page.)

---

## Play prompts from an Amazon S3 bucket

The **Get customer input**, **Loop prompts**, **Play prompt**, and **Store customer input** blocks can use an S3 bucket as the prompt source, accessed in real time via contact attributes. Store as many prompts as needed.

### Requirements

- **Format:** `.wav`, **8 kHz, mono, U-Law encoding** (convert with a third-party tool if needed) — otherwise the prompt won't play.
- **Size:** < 50 MB and < 5 minutes.
- **Opt-in Regions** (e.g. Africa (Cape Town)): the bucket must be in the **same Region**.

### Bucket policy

Grant the Connect service principal `connect.amazonaws.com` permission to `s3:ListBucket` and `s3:GetObject`. Replace the bucket name, Region, account ID, and instance ID:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "statement1",
      "Effect": "Allow",
      "Principal": { "Service": "connect.amazonaws.com" },
      "Action": ["s3:ListBucket", "s3:GetObject"],
      "Resource": [
        "arn:aws:s3:::amzn-s3-demo-bucket",
        "arn:aws:s3:::amzn-s3-demo-bucket/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:SourceAccount": "111122223333",
          "aws:SourceArn": "arn:aws:connect:region:111122223333:instance/instance-id"
        }
      }
    }
  ]
}
```

### Encryption (KMS)

Connect **cannot** play prompts from an S3 bucket encrypted with an **AWS managed key**. Use a **customer managed key** and grant the Connect service principal `kms:decrypt`:

```json
{
  "Sid": "Enable Connect",
  "Effect": "Allow",
  "Principal": { "Service": "connect.amazonaws.com" },
  "Action": "kms:decrypt",
  "Resource": ["arn:aws:kms:region:account-ID:key/key-ID"]
}
```

After the bucket policy is in place, configure the block to play from the bucket. (For the S3 URI syntax and static-vs-dynamic examples, see the **Play prompt** block reference in [blocks.md](blocks.md).)

---

## SSML in prompts

- Enable by setting **Interpret as: SSML** on the block (default is **Text**).
- Use **only** the SSML tags Connect supports, and don't assume — full reference and strict-use rule in [ssml.md](ssml.md).
- **Chat:** SSML is **not interpreted** — both the text and the tags print literally into the conversation. SSML is for voice only.
