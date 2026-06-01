# SSML Tags in Amazon Connect (Text-to-Speech via Amazon Polly)

Amazon Connect renders text-to-speech (TTS) through Amazon Polly. SSML (Speech Synthesis Markup Language) gives you fine-grained control over pronunciation, pauses, volume, rate, pitch, and more. Connect supports a **specific subset** of Polly's SSML tags — not all of them.

---

## ⚠️ Strict-use rule (read before generating any prompt)

**When asked to generate, write, or edit a text-to-speech prompt message, use ONLY the SSML tags listed in "Supported tags" below. Do not assume a tag works just because it exists in Polly, in other TTS engines, or in generic SSML. Do not invent tags or attribute values.**

- If a tag is not in the supported list on this page, **do not use it.** In Connect, an unsupported SSML tag is **silently ignored** at processing time (the surrounding text is still spoken, but the tag does nothing).
- Common tags that are **NOT supported in Connect** and must not be used: `<emphasis>`, `<amazon:effect name="drc">`, `<amazon:effect phonation="soft">`, `<amazon:effect vocal-tract-length>`, `<amazon:auto-breaths>` / `<amazon:breath>`, `<prosody amazon:max-duration>`.
- Only use attribute values explicitly documented here (e.g. for `say-as`, only the `interpret-as` values listed). Do not guess values.
- Always wrap SSML in a single `<speak>` … `</speak>` root.
- If a request needs an effect Connect can't do with a supported tag, say so plainly rather than emitting an unsupported tag.

---

## Where SSML is used in Connect

SSML-formatted TTS can be entered in these flow blocks:

- **Play prompt**
- **Get customer input**
- **Loop prompts**
- **Store customer input**

To use SSML in a block, set the **Interpret as** field to **SSML** (the default is **Text**). When set to **Text**, any tags you type are spoken/printed literally.

**Character limits** (Play prompt / Get customer input TTS field): **3,000 billed characters** and **6,000 total characters** maximum. Exceeding 6,000 characters is an error.

**Chat channel:** SSML is **not interpreted in chat conversations.** If a prompt with SSML is delivered to a chat, both the text **and the tags themselves** are printed verbatim into the conversation. Use SSML for voice only.

**Voice/engine selection** is separate from SSML — see [prompts.md](prompts.md) (the **Set voice** block, engines, and Newscaster/Conversational speaking styles).

---

## Supported tags

Connect supports the following SSML tags (per "SSML tags supported by Connect"). Anything not in this table is ignored.

| Tag | Use to… |
|---|---|
| `<speak>` | Required root. All SSML-enhanced text must be enclosed in a pair of `<speak>` tags. |
| `<break>` | Add a pause. Maximum pause duration is **10 seconds**. |
| `<lang>` | Speak specific words in another language. |
| `<mark>` | Place a custom tag (metadata marker) within the text. |
| `<p>` | Add a pause between paragraphs. |
| `<phoneme>` | Use a phonetic pronunciation for specific text. |
| `<prosody>` | Control the volume, rate, or pitch of the selected voice. |
| `<s>` | Add a pause between lines or sentences. |
| `<say-as>` | With `interpret-as`, control how characters, words, and numbers are spoken. |
| `<sub>` | With `alias`, substitute a different word/pronunciation (e.g. for an acronym). |
| `<w>` | Customize pronunciation by specifying the word's part of speech or sense. |
| `<amazon:effect name="whispered">` | Speak the text in a whispered voice. |

Plus the **Newscaster speaking style** (`<amazon:domain name="news">`) — in Connect, supported **only for the Joanna and Matthew neural voices in en-US**.

> **Engine caveats that matter in Connect:**
> - `<amazon:effect name="whispered">` works on the **standard** engine only.
> - `<say-as interpret-as="characters">` is not supported on neural voices — Polly synthesizes that sentence with the related standard voice (still billed as neural).
> - `<prosody>` `pitch` is **not** honored on neural/generative/long-form voices (only `volume` and `rate` are). On generative voices, `<prosody>` and `<lang>` may be used only around full sentences.
> - `<mark>` produces metadata only; it does nothing on generative voices.

---

## Tag reference

### `<speak>` — root element

All SSML must be wrapped in a single `<speak>` pair.

```xml
<speak>Thank you for calling. How can I help you today?</speak>
```

---

### `<break>` — pause

Add a pause by **strength** (comma/sentence/paragraph length) or by an explicit **time** in seconds/milliseconds. Default when no attribute is given is `<break strength="medium"/>` (a comma-length pause). **Maximum time is 10 seconds.**

| Attribute | Values | Meaning |
|---|---|---|
| `strength` | `none` | No pause. Removes a normally occurring pause (e.g. after a period). |
| `strength` | `x-weak` | Same as `none` — no pause. |
| `strength` | `weak` | Pause equal to the pause after a comma. |
| `strength` | `medium` | Same as `weak` (this is the default). |
| `strength` | `strong` | Pause equal to the pause after a sentence. |
| `strength` | `x-strong` | Pause equal to the pause after a paragraph. |
| `time` | `{n}s` | Pause length in seconds. Max `10s`. |
| `time` | `{n}ms` | Pause length in milliseconds. Max `10000ms`. |

Default behavior of a bare `<break/>` depends on adjacent punctuation: next to a comma it upgrades to `strong`; next to a period it upgrades to `x-strong`; otherwise `medium`.

```xml
<speak>
  Mary had a little lamb <break time="3s"/> whose fleece was white as snow.
</speak>
```

---

### `<p>` — paragraph pause

Encloses a paragraph and inserts a long pause, equivalent to `<break strength="x-strong"/>`. Takes no attributes.

```xml
<speak>
  <p>This is the first paragraph. There should be a pause after this text is spoken.</p>
  <p>This is the second paragraph.</p>
</speak>
```

---

### `<s>` — sentence pause

Encloses a sentence/line and inserts a pause equivalent to `<break strength="strong"/>` (the same as ending the sentence with a period). Takes no attributes. Useful for poetry or line-organized text.

```xml
<speak>
  <s>Mary had a little lamb</s>
  <s>Whose fleece was white as snow</s>
  And everywhere that Mary went, the lamb was sure to go.
</speak>
```

---

### `<say-as>` — how to speak characters, words, and numbers

Use the `interpret-as` attribute to remove ambiguity in how text is read.

```xml
<say-as interpret-as="{value}">text to be interpreted</say-as>
```

| `interpret-as` value | What it does | Example |
|---|---|---|
| `characters` or `spell-out` | Spells out each letter (a-b-c). Not supported on neural voices (falls back to standard for that sentence). | `<say-as interpret-as="characters">abc</say-as>` |
| `cardinal` or `number` | Reads as a cardinal number (1,234). | `<say-as interpret-as="cardinal">1234</say-as>` |
| `ordinal` | Reads as an ordinal number (1,234th). | `<say-as interpret-as="ordinal">1234</say-as>` |
| `digits` | Reads each digit individually (1-2-3-4). | `<say-as interpret-as="digits">1234</say-as>` |
| `fraction` | Reads as a fraction (see below). | `<say-as interpret-as="fraction">2/9</say-as>` → "two ninths" |
| `unit` | Reads as a measurement; number/fraction followed by a unit with no space (`1/2inch`, `1meter`). | `<say-as interpret-as="unit">1meter</say-as>` |
| `date` | Reads as a date; requires the `format` attribute (see below). | `<say-as interpret-as="date" format="mdy">12-31-1900</say-as>` |
| `time` | Reads as a duration in minutes and seconds (`1'21"`). | `<say-as interpret-as="time">1'21"</say-as>` |
| `address` | Reads as part of a street address. | `<say-as interpret-as="address">...</say-as>` |
| `expletive` | "Beeps out" the enclosed content. | `<say-as interpret-as="expletive">...</say-as>` |
| `telephone` | Reads as a 7- or 10-digit phone number; handles extensions (`2025551212x345`). | `<say-as interpret-as="telephone">2025551212</say-as>` |

**Fractions:**
- Simple: `{cardinal}/{cardinal}`, e.g. `2/9` → "two ninths".
- Mixed (non-negative): `{cardinal}+{cardinal}/{cardinal}`, e.g. `3+1/2` → "three and a half". The `+` is required; `3 1/2` is not supported.

**Date `format` values:** `mdy`, `dmy`, `ymd`, `md`, `dm`, `ym`, `my`, `d`, `m`, `y`, and `yyyymmdd`. With `yyyymmdd` you can mask parts with `?`:

```xml
<say-as interpret-as="date" format="yyyymmdd">????0922</say-as>  <!-- "September 22nd" -->
```

**Telephone:** Polly often interprets formatted numbers (e.g. `202-555-1212`) correctly without the tag; use the tag for unformatted strings (`2025551212`). Pronunciation is language-specific (UK English groups repeated digits as "double five", etc.). `telephone` is available for en (AU/GB/IN/US/GB-WLS), es (ES/MX/US), fr (FR/CA), pt (BR/PT), de-DE, it-IT, ja-JP, ru-RU; some languages (e.g. arb) handle phone numbers automatically.

```xml
<speak>
  Your confirmation number is <say-as interpret-as="digits">48213</say-as>.
  We will call you back on <say-as interpret-as="telephone">2025551212</say-as>.
</speak>
```

---

### `<prosody>` — volume, rate, pitch

Must contain at least one attribute; can combine multiple in one tag. Values are voice-relative (no absolute units).

| Attribute | Allowed values | Meaning |
|---|---|---|
| `volume` | `default` | Reset to the voice's default volume. |
| `volume` | `silent`, `x-soft`, `soft`, `medium`, `loud`, `x-loud` | Predefined volume levels. |
| `volume` | `+ndB` / `-ndB` | Relative change. `+0dB` = no change, `+6dB` ≈ 2× volume, `-6dB` ≈ half. |
| `rate` | `x-slow`, `slow`, `medium`, `fast`, `x-fast` | Predefined speaking rate. |
| `rate` | `n%` | Non-negative percentage. `100%` = no change, `200%` = 2×, `50%` = half. Range 20–200%. |
| `pitch` | `default` | Reset to the voice's default pitch. |
| `pitch` | `x-low`, `low`, `medium`, `high`, `x-high` | Predefined pitch. |
| `pitch` | `+n%` / `-n%` | Relative pitch change. |

**Engine support:** standard voices support all three attributes. Neural / generative / long-form voices support `volume` and `rate` but **not `pitch`**. On generative voices, `<prosody>` may wrap only full sentences.

```xml
<speak>
  Please listen carefully, <prosody rate="slow" volume="loud">as our menu options have changed.</prosody>
</speak>
```

Attributes can be combined and tags nested:

```xml
<speak>
  <prosody rate="85%">Sometimes combining attributes <prosody volume="loud">changes the impression</prosody> as well.</prosody>
</speak>
```

---

### `<phoneme>` — phonetic pronunciation

Both attributes are **required**: `alphabet` and `ph`.

| Attribute | Values | Meaning |
|---|---|---|
| `alphabet` | `ipa` | International Phonetic Alphabet. |
| `alphabet` | `x-sampa` | Extended SAMPA. |
| `alphabet` | `x-amazon-pinyin` | Pinyin (Mandarin Chinese). |
| `alphabet` | `x-amazon-yomigana` | Yomigana (Japanese). |
| `alphabet` | `x-amazon-pron-kana` | Pronunciation Kana (Japanese). |
| `ph` | phonetic symbol string | The pronunciation in the chosen alphabet. |

```xml
<speak>
  You say <phoneme alphabet="ipa" ph="pɪˈkɑːn">pecan</phoneme>,
  I say <phoneme alphabet="ipa" ph="ˈpi.kæn">pecan</phoneme>.
</speak>
```

---

### `<lang>` — language for specific words

Speak a word/phrase/sentence in another language using `xml:lang` (e.g. `fr-FR`, `en-US`). Pronunciation is still shaped by the base voice's native language (a US-English voice gives an American-accented rendering of French).

```xml
<speak>
  Mi piace <lang xml:lang="en-US">Bruce Springsteen.</lang>
</speak>
```

---

### `<mark>` — custom metadata marker

Inserts a named marker. Polly takes no audible action but returns the mark's location in the SSML metadata stream. `name` can be any string.

```xml
<speak>
  Mary had a little <mark name="animal"/>lamb.
</speak>
```

---

### `<sub>` — substitution / alias

Speak the `alias` text in place of the enclosed text (acronyms, symbols, abbreviations).

```xml
<speak>
  My favorite element is <sub alias="Mercury">Hg</sub>.
</speak>
```

---

### `<w>` — part of speech / word sense

Force a pronunciation by specifying the word's role via the `role` attribute.

| `role` value | Meaning |
|---|---|
| `amazon:VB` | Verb (present simple). |
| `amazon:VBD` | Past-tense verb. |
| `amazon:DT` | Determiner. |
| `amazon:IN` | Preposition. |
| `amazon:JJ` | Adjective. |
| `amazon:NN` | Noun. |
| `amazon:DEFAULT` | Default sense of the word. |
| `amazon:SENSE_1` | Non-default sense (e.g. "bass" the fish vs. the musical range). |

```xml
<speak>
  The word read may be present <w role="amazon:VB">read</w> or past <w role="amazon:VBD">read</w>.
</speak>
```

---

### `<amazon:effect name="whispered">` — whisper

Speak the enclosed text in a whisper. **Standard engine only.** Slowing the rate slightly enhances the effect.

```xml
<speak>
  <amazon:effect name="whispered">If you make any noise,</amazon:effect> she said,
  <amazon:effect name="whispered"><prosody rate="-10%">they will hear us.</prosody></amazon:effect>
</speak>
```

---

### `<amazon:domain name="news">` — Newscaster style

In Connect, available **only for the Joanna and Matthew neural voices in en-US**. Reads the enclosed text in a broadcast-news style.

```xml
<speak>
  <amazon:domain name="news">
    The maiden voyage of the largest ship ever launched has ended in disaster.
  </amazon:domain>
</speak>
```

---

## Quick checklist for writing a Connect prompt

1. Wrap everything in one `<speak>` … `</speak>`.
2. Use **only** the tags in the supported table above — nothing else.
3. Keep total length ≤ 6,000 characters (≤ 3,000 billed).
4. Keep any `<break time>` ≤ 10 seconds.
5. For account numbers / confirmation codes, prefer `say-as interpret-as="digits"`; for phone numbers, `telephone`.
6. Remember `pitch` and `whispered` and Newscaster are engine/voice-limited (see caveats above).
7. SSML is voice-only — never rely on it for chat.
