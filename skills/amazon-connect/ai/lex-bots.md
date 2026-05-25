# Amazon Lex Integration with Amazon Connect

## Overview

Amazon Lex is natively integrated with Amazon Connect to provide conversational IVR (Interactive Voice Response) and chatbot capabilities. Lex handles natural language understanding (NLU) for intent detection and slot filling, automatic speech recognition (ASR), and neural text-to-speech (TTS).

---

## Core Capabilities

### Natural Language Understanding (NLU)

Lex NLU performs intent detection and entity extraction:

- **Intent detection** -- identifies what the customer wants to do from their utterance
- **Slot filling** -- extracts structured data (dates, numbers, names, etc.) from free-form speech or text
- **Confirmation prompts** -- verifies extracted data with the customer before proceeding
- **Fallback intents** -- handles unrecognized utterances gracefully

### Automatic Speech Recognition (ASR)

- **25+ languages and locales** supported for speech-to-text
- Optimized for telephony audio (8kHz)
- Supports DTMF input alongside voice
- Confidence scores on transcriptions
- Custom vocabularies for domain-specific terms (product names, acronyms)

### Neural Text-to-Speech (TTS)

- **30+ languages** supported for text-to-speech
- Neural voices for natural-sounding responses
- SSML support for fine-grained speech control (pauses, emphasis, pronunciation)
- Multiple voice options per language (male/female, different styles)

---

## Generative AI Features

Lex includes several generative AI enhancements powered by Amazon Bedrock:

### LLM-Assisted Slot Resolution

The LLM helps resolve ambiguous slot values that traditional NLU would miss:

- Understands context to fill slots more accurately
- Handles synonyms and paraphrasing (e.g., "tomorrow" --> actual date)
- Resolves entity references across turns (e.g., "the first one" referring to a previously listed option)

### Conversational FAQs

Connect a knowledge base to the Lex bot for FAQ handling:

- Lex queries the knowledge base when no intent matches
- Generates natural language answers from KB content
- Falls back gracefully if no relevant content is found
- No need to define intents for every possible FAQ -- the KB handles the long tail

### Sample Utterance Generation

AI-assisted bot building:

- Provide a few example utterances for an intent
- The LLM generates additional diverse utterances automatically
- Expands coverage without manual effort
- Reduces the chance of missed intent matches in production

### Bot Creation from Natural Language Description

Create a bot by describing what it should do in plain English:

- Describe the use case: "I need a bot that handles appointment scheduling, cancellations, and rescheduling for a dental office"
- Lex generates intents, slots, sample utterances, and confirmation prompts
- Review, refine, and deploy
- Significantly accelerates bot development for standard use cases

---

## Connect Flow Integration

### Get Customer Input Block

The primary integration point is the **Get customer input** flow block:

```
Get customer input
  --> Input type: Amazon Lex
  --> Lex bot: MyBot
  --> Alias: $LATEST or production alias
  --> Intents:
      --> BookAppointment --> [next block]
      --> CancelAppointment --> [next block]
      --> FallbackIntent --> [fallback handling]
  --> Error --> [error handling]
```

Configuration options on the block:

- **Lex bot name and alias** -- which bot and version to invoke
- **Session attributes** -- key-value pairs passed to Lex (and available in Lambda fulfillment)
- **DTMF input** -- enable digit input alongside voice
- **Timeout** -- how long to wait for customer input
- **Barge-in** -- allow customers to interrupt the prompt

### Session Attributes

Lex session attributes provide a bidirectional data channel between the Connect flow and the Lex bot:

**Flow --> Lex (setting attributes before the Get Customer Input block):**

```
Set contact attributes
  Namespace: Lex
  Key: customerTier
  Value: $.Attributes.customerTier
```

**Lex --> Flow (reading attributes after the Get Customer Input block):**

```
Check contact attributes
  Namespace: Lex
  Key: appointmentDate
  Condition: Is set --> [proceed]
```

Common session attribute patterns:

| Attribute | Direction | Purpose |
|-----------|-----------|---------|
| `customerTier` | Flow --> Lex | Customize bot behavior based on customer segment |
| `language` | Flow --> Lex | Set bot language preference |
| `appointmentDate` | Lex --> Flow | Extracted slot value for downstream processing |
| `intentName` | Lex --> Flow | The matched intent name |
| `confirmationStatus` | Lex --> Flow | Whether the customer confirmed (Confirmed/Denied/None) |
| `Tool` | Lex --> Flow | Return to Control tool name (for agentic self-service) |

### Accessing Slot Values in Flows

After the Lex bot returns control, slot values are accessible as Lex session attributes:

```
Check contact attributes
  Namespace: Lex
  Key: slots.PhoneNumber
  Condition: Is set --> [proceed with phone number]
```

Or use a Lambda function for complex slot processing.

---

## Bot Architecture for Connect

### Recommended Structure

```
Lex Bot (e.g., "CustomerServiceBot")
  |
  +-- Intent: Greeting
  |     Utterances: "hello", "hi", "good morning"
  |     Fulfillment: Return to flow
  |
  +-- Intent: CheckOrderStatus
  |     Slots: OrderNumber (AMAZON.Number)
  |     Utterances: "where is my order", "check order {OrderNumber}"
  |     Fulfillment: Lambda function
  |
  +-- Intent: TransferToAgent
  |     Utterances: "speak to someone", "transfer me", "agent"
  |     Fulfillment: Return to flow (route to queue)
  |
  +-- Intent: FallbackIntent (built-in)
        Fulfillment: Conversational FAQ or escalation
```

### Lambda Fulfillment

Lex can invoke Lambda functions for slot validation and fulfillment:

```javascript
exports.handler = async (event) => {
  const intentName = event.sessionState.intent.name;
  const slots = event.sessionState.intent.slots;

  if (intentName === "CheckOrderStatus") {
    const orderNumber = slots.OrderNumber?.value?.interpretedValue;
    
    // Look up order in backend system
    const orderStatus = await lookupOrder(orderNumber);

    return {
      sessionState: {
        dialogAction: { type: "Close" },
        intent: {
          name: intentName,
          state: "Fulfilled",
          slots: slots
        },
        sessionAttributes: {
          orderStatus: orderStatus,
          orderNumber: orderNumber
        }
      },
      messages: [
        {
          contentType: "PlainText",
          content: `Your order ${orderNumber} is currently ${orderStatus}.`
        }
      ]
    };
  }
};
```

---

## Multi-Language Support

### Voice Languages (ASR) -- 25+ Locales

Partial list of supported locales:

| Language | Locale Code |
|----------|-------------|
| English (US) | `en_US` |
| English (UK) | `en_GB` |
| English (AU) | `en_AU` |
| Spanish (US) | `es_US` |
| Spanish (ES) | `es_ES` |
| French (FR) | `fr_FR` |
| French (CA) | `fr_CA` |
| German | `de_DE` |
| Italian | `it_IT` |
| Portuguese (BR) | `pt_BR` |
| Japanese | `ja_JP` |
| Korean | `ko_KR` |
| Chinese (Mandarin) | `zh_CN` |
| Hindi | `hi_IN` |
| Arabic | `ar_SA` |

### TTS Languages -- 30+

Neural TTS supports all ASR languages plus additional locales. Each language typically offers multiple voice options.

---

## Best Practices

1. **Use bot aliases** -- never point production flows at `$LATEST`; use a named alias and version pinning
2. **FallbackIntent is critical** -- always configure a meaningful fallback that either retries, escalates, or queries a FAQ knowledge base
3. **Session attributes for context** -- pass customer context (tier, language, account info) from the flow to the bot to enable personalized responses
4. **DTMF fallback** -- for IVR, always offer a DTMF option alongside voice input for accessibility
5. **Test with telephony audio** -- Lex ASR behaves differently with 8kHz telephony audio vs. 16kHz; always test in the actual Connect environment
6. **Slot elicitation prompts** -- write prompts that are natural for spoken conversation, not written text
7. **Confidence thresholds** -- configure NLU confidence thresholds to avoid false-positive intent matches; route low-confidence results to fallback
