# Amazon Q Connect (formerly Wisdom)

## Overview

Amazon Q Connect is the LLM-enhanced agent assistance and knowledge management service for Amazon Connect. It provides real-time AI-powered recommendations to agents during customer interactions and serves as the management plane for AI agents, prompts, guardrails, and knowledge bases.

Previously known as **Amazon Connect Wisdom**, it was rebranded to Amazon Q Connect when generative AI capabilities were added.

**SDK**: `@aws-sdk/client-qconnect`

---

## Core Concepts

### Assistants

An assistant is the top-level resource that ties together knowledge bases, AI agents, prompts, and guardrails for a Connect instance.

```javascript
const { QConnectClient, CreateAssistantCommand } = require("@aws-sdk/client-qconnect");

const client = new QConnectClient({ region: "us-east-1" });

const assistant = await client.send(new CreateAssistantCommand({
  name: "MyAssistant",
  type: "AGENT",  // AGENT is the only supported type currently
  serverSideEncryptionConfiguration: {
    kmsKeyId: "arn:aws:kms:us-east-1:123456789012:key/key-id"  // Optional
  },
  tags: {
    environment: "production"
  }
}));

// assistant.assistant.assistantId --> use this for all subsequent operations
```

Each Connect instance has one assistant. The assistant ID is required for most Q Connect API calls.

### Knowledge Bases

Knowledge bases store and index content that the AI uses to generate responses and recommendations.

#### Content Knowledge Base

Upload documents (PDF, HTML, Word, plain text) directly:

```javascript
const { CreateKnowledgeBaseCommand } = require("@aws-sdk/client-qconnect");

const kb = await client.send(new CreateKnowledgeBaseCommand({
  name: "ProductDocumentation",
  knowledgeBaseType: "CUSTOM",
  sourceConfiguration: {
    appIntegrations: {
      appIntegrationArn: "arn:aws:app-integrations:us-east-1:123456789012:application/app-id"
    }
  }
}));
```

#### S3 Knowledge Base

Index content from an S3 bucket:

```javascript
const kb = await client.send(new CreateKnowledgeBaseCommand({
  name: "S3Documentation",
  knowledgeBaseType: "CUSTOM",
  sourceConfiguration: {
    appIntegrations: {
      appIntegrationArn: "arn:aws:app-integrations:...",
      objectFields: ["key", "content"]
    }
  }
}));
```

#### Web Crawler Knowledge Base

Crawl and index web pages:

```javascript
const kb = await client.send(new CreateKnowledgeBaseCommand({
  name: "WebDocs",
  knowledgeBaseType: "CUSTOM",
  sourceConfiguration: {
    managedSourceConfiguration: {
      webCrawlerConfiguration: {
        urlConfiguration: {
          seedUrlConfiguration: {
            seedUrls: [
              { url: "https://docs.example.com" }
            ]
          }
        },
        crawlerLimits: {
          rateLimit: 10  // Pages per second
        },
        scope: "HOST_ONLY"  // or "SUBDOMAINS"
      }
    }
  }
}));
```

### Content Management

Add, update, and manage individual content items within a knowledge base:

```javascript
const { CreateContentCommand, UpdateContentCommand, DeleteContentCommand, SearchContentCommand } = require("@aws-sdk/client-qconnect");

// Create content
const content = await client.send(new CreateContentCommand({
  knowledgeBaseId: "kb-id",
  name: "Refund Policy",
  title: "Customer Refund Policy",
  uploadId: "upload-id",  // From StartContentUpload
  metadata: {
    department: "billing",
    category: "policies"
  },
  tags: {
    version: "2024-01"
  }
}));

// Search content
const results = await client.send(new SearchContentCommand({
  knowledgeBaseId: "kb-id",
  searchExpression: {
    filters: [
      {
        field: "NAME",
        operator: "EQUALS",
        value: "Refund Policy"
      }
    ]
  }
}));
```

### Message Templates

Templates for outbound customer communications across multiple channels:

```javascript
const { CreateMessageTemplateCommand } = require("@aws-sdk/client-qconnect");

const template = await client.send(new CreateMessageTemplateCommand({
  knowledgeBaseId: "kb-id",
  name: "OrderConfirmation",
  channelSubtype: "EMAIL",  // EMAIL, SMS, WHATSAPP, PUSH
  content: {
    email: {
      subject: "Order #{{orderNumber}} Confirmed",
      body: {
        html: {
          content: "<h1>Thank you for your order</h1><p>Order #{{orderNumber}} has been confirmed.</p>"
        },
        plainText: {
          content: "Thank you for your order. Order #{{orderNumber}} has been confirmed."
        }
      },
      headers: [
        { name: "X-Custom-Header", value: "value" }
      ]
    }
  },
  defaultAttributes: {
    customAttributes: {
      orderNumber: { values: [{ stringValue: "" }] }
    }
  }
}));
```

Supported channel subtypes:
- **EMAIL** -- full HTML/plain text with headers and attachments
- **SMS** -- plain text, character limit aware
- **WHATSAPP** -- supports WhatsApp message formatting
- **PUSH** -- mobile push notification format

### Quick Responses

Pre-written responses agents can insert into conversations with one click:

```javascript
const { CreateQuickResponseCommand } = require("@aws-sdk/client-qconnect");

const quickResponse = await client.send(new CreateQuickResponseCommand({
  knowledgeBaseId: "kb-id",
  name: "greeting_standard",
  shortcutKey: "hello",
  content: {
    quickResponseContent: {
      markdown: "Hello! Thank you for contacting us. How can I help you today?"
    }
  },
  contentType: "MARKDOWN",  // or "PLAIN_TEXT"
  channels: ["Chat"],
  language: "en-US",
  groupingConfiguration: {
    criteria: "CONTACT_LENS_CATEGORY",
    values: ["Billing"]
  }
}));
```

---

## Sessions

A session represents a single customer interaction. Sessions track the conversation state and AI recommendations.

```javascript
const { CreateSessionCommand, SendMessageCommand, GetNextMessageCommand } = require("@aws-sdk/client-qconnect");

// Create a session (typically done via the contact flow)
const session = await client.send(new CreateSessionCommand({
  assistantId: "assistant-id",
  name: "session-name",
  description: "Voice contact session",
  tagFilter: {
    tagCondition: {
      key: "department",
      value: "support"
    }
  }
}));

// Send a message (for chat/messaging or programmatic interaction)
const sendResult = await client.send(new SendMessageCommand({
  assistantId: "assistant-id",
  sessionId: session.session.sessionId,
  message: {
    value: {
      text: {
        value: "I need to return my order"
      }
    }
  },
  type: "TEXT",
  conversationContext: {
    selfServiceConversationHistory: [
      {
        botMessage: "How can I help you?",
        inputTranscript: "I want to return something"
      }
    ]
  }
}));

// Get the AI-generated response
const nextMessage = await client.send(new GetNextMessageCommand({
  assistantId: "assistant-id",
  sessionId: session.session.sessionId,
  nextMessageToken: sendResult.nextMessageToken
}));
```

---

## Import Jobs

Bulk import content into a knowledge base from external sources:

```javascript
const { StartImportJobCommand, GetImportJobCommand } = require("@aws-sdk/client-qconnect");

const importJob = await client.send(new StartImportJobCommand({
  knowledgeBaseId: "kb-id",
  importJobType: "QUICK_RESPONSES",  // or "CONTENT"
  uploadId: "upload-id",
  externalSourceConfiguration: {
    source: "AMAZON_S3",
    configuration: {
      amazonS3: "s3://bucket/path/to/import-file.csv"
    }
  },
  metadata: {
    importedBy: "admin"
  }
}));

// Check import status
const status = await client.send(new GetImportJobCommand({
  knowledgeBaseId: "kb-id",
  importJobId: importJob.importJob.importJobId
}));
// status.importJob.status --> "START_IN_PROGRESS" | "COMPLETE" | "FAILED"
```

---

## AI Agents, Prompts, and Guardrails

Q Connect is the management API for all AI configuration in Amazon Connect. See [connect-ai-agents.md](./connect-ai-agents.md) for detailed coverage of:

- AI agent types and configuration
- AI prompt customization (12 prompt types)
- AI guardrails (content filters, denied topics, PII handling)

Key management operations:

```javascript
// AI Agents
CreateAIAgent, UpdateAIAgent, DeleteAIAgent, GetAIAgent, ListAIAgents
CreateAIAgentVersion, ListAIAgentVersions

// AI Prompts
CreateAIPrompt, UpdateAIPrompt, DeleteAIPrompt, GetAIPrompt, ListAIPrompts
CreateAIPromptVersion, ListAIPromptVersions

// AI Guardrails
CreateAIGuardrail, UpdateAIGuardrail, DeleteAIGuardrail, GetAIGuardrail, ListAIGuardrails
CreateAIGuardrailVersion, ListAIGuardrailVersions
```

---

## Key API Operations Reference

### Assistant Management
| Operation | Description |
|-----------|-------------|
| `CreateAssistant` | Create a new assistant for a Connect instance |
| `GetAssistant` | Retrieve assistant details |
| `ListAssistants` | List all assistants in the account |
| `DeleteAssistant` | Delete an assistant |

### Knowledge Base Management
| Operation | Description |
|-----------|-------------|
| `CreateKnowledgeBase` | Create a new knowledge base |
| `GetKnowledgeBase` | Retrieve knowledge base details |
| `ListKnowledgeBases` | List all knowledge bases for an assistant |
| `DeleteKnowledgeBase` | Delete a knowledge base |
| `UpdateKnowledgeBaseTemplateUri` | Update the template URI |

### Content Operations
| Operation | Description |
|-----------|-------------|
| `CreateContent` | Add content to a knowledge base |
| `UpdateContent` | Update existing content |
| `DeleteContent` | Remove content |
| `GetContent` | Retrieve content by ID |
| `SearchContent` | Search content by filters |
| `StartContentUpload` | Initiate a content upload (returns presigned URL) |

### Session Operations
| Operation | Description |
|-----------|-------------|
| `CreateSession` | Create a new session for a contact |
| `GetSession` | Retrieve session details |
| `UpdateSession` | Update session configuration (e.g., AI agent overrides) |
| `SendMessage` | Send a message to the AI in a session |
| `GetNextMessage` | Retrieve the AI's response message |

### Recommendation Operations
| Operation | Description |
|-----------|-------------|
| `QueryAssistant` | Query the assistant for recommendations (agent-initiated search) |
| `GetRecommendations` | Get real-time AI recommendations for a session |
| `PutFeedback` | Submit agent feedback on a recommendation (helpful/not helpful) |
| `NotifyRecommendationsReceived` | Acknowledge that recommendations were displayed |

### Import Operations
| Operation | Description |
|-----------|-------------|
| `StartImportJob` | Begin bulk content import |
| `GetImportJob` | Check import job status |

---

## Integration Pattern

Typical integration flow in a Connect contact flow:

1. **Contact arrives** --> flow creates a Q Connect session via Lambda
2. **Conversation starts** --> Contact Lens streams transcript to Q Connect
3. **AI generates recommendations** --> displayed in agent workspace in real time
4. **Agent uses recommendations** --> clicks to insert, searches manually, or ignores
5. **Agent provides feedback** --> PutFeedback marks recommendations as helpful or not
6. **Contact ends** --> session is closed, feedback is used to improve future recommendations
