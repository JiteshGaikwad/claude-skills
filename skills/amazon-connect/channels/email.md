# Email Channel

Amazon Connect's email channel lets agents send and receive emails through the same CCP used for voice and chat. Built on Amazon SES for delivery, with full threading, templates, attachments, and Contact Lens analytics.

## Setup and Configuration

Email in Connect requires Amazon SES integration for sending and receiving.

**Domain setup:**
- Configure up to **5 custom domains** per Connect instance
- Domains must be verified in Amazon SES (DKIM + SPF records)
- SES handles email delivery, bounce management, and reputation monitoring

**Email addresses:**
- Create up to **100 email addresses** across your domains
- Common patterns: `support@example.com`, `sales@example.com`, `billing@example.com`
- Each address can be associated with a specific contact flow
- Multiple "from" addresses can be configured per queue — agents select which to send from

**Inbound routing:**
- Incoming emails hit SES, which triggers a Connect contact flow
- Contact flow applies routing logic (queue, priority, attributes)
- Email contacts land in agent queues alongside voice and chat

## Email Threading

Emails within the same conversation are threaded chronologically.

- Connect tracks threads via standard email headers (`In-Reply-To`, `References`)
- Agents see the full conversation history when handling a reply
- New emails from the same customer on the same subject are grouped into the thread
- Thread display is chronological — oldest at top, newest at bottom
- Agents can scroll through the entire conversation before responding

## Agent Experience

**Rich text editor:**
- Agents compose replies using a full rich text editor in the CCP
- Formatting options: bold, italic, underline, bullet lists, numbered lists, hyperlinks
- HTML email output — customers receive properly formatted messages

**Templates:**
- Pre-built email templates for common responses
- Templates support dynamic variables (customer name, case ID, etc.)
- Admins create and manage templates in the Connect console
- Agents select a template and customize before sending

**Signatures:**
- Configurable email signatures appended to outgoing messages
- Can be set per agent, per queue, or per routing profile
- Support HTML formatting (logo, links, disclaimers)

**Quick responses:**
- Pre-written snippets agents can insert into emails
- Different from templates — quick responses are partial content blocks, not full email bodies
- Useful for standard paragraphs, legal disclaimers, or common instructions

**Content protection:**
- Agents cannot manipulate or edit the customer's original content in replies
- Customer's message is quoted as-is in the reply thread
- Prevents accidental or intentional alteration of what the customer said

## Auto-Responses

Configure automatic email replies for specific scenarios.

- Acknowledgment emails when a customer's email is received ("We got your message, a representative will respond within 24 hours")
- Out-of-hours auto-responses with expected response times
- Configured in contact flows using the "Send email" block or Lambda functions
- Auto-responses include thread headers so they appear in the correct conversation thread

## Forwarding

Agents can forward emails to external addresses.

- Forward to colleagues, escalation teams, or external partners
- Forwarded email preserves the original thread and attachments
- The forwarded recipient can reply back into the Connect thread
- Useful for cross-team collaboration on complex cases

## Multiple "From" Addresses per Queue

A single queue can have multiple outbound email addresses configured.

- Agents select the appropriate "from" address when composing or replying
- Example: a "General Support" queue might send from `support@example.com`, `help@example.com`, or `info@example.com`
- The default "from" address is set at the queue level
- Routing rules can set the "from" address automatically based on contact attributes

## File Attachments

Email supports file attachments for both inbound and outbound messages.

- Maximum attachment size: **100 MB** per email
- Multiple attachments per message
- Common file types supported (PDF, DOCX, XLSX, images, ZIP, etc.)
- Attachments are stored in the Connect-managed S3 bucket
- Virus scanning is recommended via S3 event triggers (not built into Connect)
- Large attachments count against the 100 MB total per email, not per file

## Contact Lens for Email

Contact Lens analytics apply to email contacts just as they do to voice and chat.

**Capabilities:**
- **Categorization:** Automatically classify emails by topic, intent, or issue type using rules
- **PII redaction:** Detect and redact sensitive information (SSN, credit card numbers, addresses) in email content
- **Summaries:** AI-generated summaries of email threads — useful for long multi-reply conversations
- **Sentiment analysis:** Assess customer sentiment from email text
- **Evaluation:** Include email contacts in agent quality evaluations

**Configuration:**
- Enable Contact Lens for email in the "Set recording and analytics behavior" flow block
- Redaction rules apply to stored transcripts and analytics output
- Results available in the Connect analytics dashboard and via APIs

## Email APIs

### CreateEmailAddress

Provision a new email address for your Connect instance.

```javascript
import { ConnectClient, CreateEmailAddressCommand } from "@aws-sdk/client-connect";

const client = new ConnectClient({ region: "us-east-1" });

const response = await client.send(new CreateEmailAddressCommand({
  InstanceId: instanceId,
  EmailAddress: "support@example.com",
  DisplayName: "Customer Support",
  Description: "Main support inbox",
}));
// response.EmailAddressId — unique identifier for this email address
// response.EmailAddressArn — ARN for IAM policies
```

### SearchEmailAddresses

Find email addresses configured in your instance.

```javascript
const response = await client.send(new SearchEmailAddressesCommand({
  InstanceId: instanceId,
  MaxResults: 25,
  SearchCriteria: {
    StringCondition: {
      FieldName: "emailAddress",
      Value: "support",
      ComparisonType: "CONTAINS",
    },
  },
}));
// response.EmailAddresses — array of matching email address objects
```

### SendOutboundEmail

Send an email from Connect without creating a full contact (e.g., notifications, receipts).

```javascript
const response = await client.send(new SendOutboundEmailCommand({
  InstanceId: instanceId,
  FromEmailAddress: {
    EmailAddressId: emailAddressId,
  },
  DestinationEmailAddress: {
    EmailAddressId: destinationEmailId,
    // or DisplayName + raw address
  },
  EmailMessage: {
    Subject: { Value: "Your order has shipped", Charset: "UTF-8" },
    Body: {
      Html: { Value: "<p>Your order #123 shipped today.</p>", Charset: "UTF-8" },
      Text: { Value: "Your order #123 shipped today.", Charset: "UTF-8" },
    },
  },
  // Optional: attach files via S3 references
}));
```

### StartEmailContact

Create a new inbound email contact that enters a contact flow.

```javascript
const response = await client.send(new StartEmailContactCommand({
  InstanceId: instanceId,
  ContactFlowId: contactFlowId,
  FromEmailAddress: {
    EmailAddress: "customer@example.com",
    DisplayName: "John Customer",
  },
  DestinationEmailAddress: "support@yourcompany.com",
  Name: "Email from John regarding billing",
  EmailMessage: {
    Subject: { Value: "Billing question", Charset: "UTF-8" },
    Body: {
      Text: { Value: "I have a question about my invoice.", Charset: "UTF-8" },
    },
  },
}));
// response.ContactId — the new contact's ID
```

### StartOutboundEmailContact

Proactively initiate an email contact that routes through a flow and assigns to an agent.

```javascript
const response = await client.send(new StartOutboundEmailContactCommand({
  InstanceId: instanceId,
  ContactFlowId: outboundFlowId,
  DestinationEmailAddress: {
    EmailAddress: "customer@example.com",
    DisplayName: "Jane Customer",
  },
  FromEmailAddress: {
    EmailAddressId: fromEmailAddressId,
  },
  EmailMessage: {
    Subject: { Value: "Follow-up on your recent case", Charset: "UTF-8" },
    Body: {
      Html: { Value: "<p>Hi Jane, following up on case #789...</p>", Charset: "UTF-8" },
    },
  },
}));
```

## Routing Behavior

- Email contacts enter queues and are routed via the same routing profiles as voice and chat
- Priority and delay settings apply — email can be lower priority than voice/chat
- Agents handle one email at a time by default (configurable concurrency)
- Email does not ring — it appears in the agent's queue and they accept it
- After-contact work (ACW) applies to email contacts

## Email Lifecycle

| Stage | Description |
|-------|-------------|
| Received | SES receives the email, triggers Connect contact flow |
| Queued | Contact flow routes to a queue based on rules |
| Assigned | Agent accepts the email from their queue |
| Composing | Agent reads thread, drafts reply using rich text editor |
| Sent | Reply sent via SES, threaded with the original conversation |
| ACW | Agent completes after-contact work (notes, disposition) |
| Closed | Contact record finalized, analytics processed |

## Key Considerations

- **SES limits:** SES sending quotas apply — request increases for high-volume email operations
- **Bounce handling:** SES manages bounces and complaints; high bounce rates can affect your sending reputation
- **Encryption:** TLS in transit, S3 SSE at rest for stored emails and attachments
- **Compliance:** Email content can be redacted via Contact Lens for PCI/HIPAA compliance
- **Threading limits:** Very long email threads may be truncated in the agent view for performance
- **No BCC visibility:** Agents cannot see BCC recipients on inbound emails (standard email behavior)
- **Spam filtering:** Inbound emails pass through SES spam/virus filtering before reaching Connect
