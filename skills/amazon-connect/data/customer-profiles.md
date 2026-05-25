# Customer Profiles

Amazon Connect Customer Profiles provides a unified view of customer data by combining information from external applications (Salesforce, Zendesk, ServiceNow, Marketo, S3) with Amazon Connect contact history. Agents see a single pane with cases, contact history, product purchases, and additional context during interactions.

## Core Concepts

### Domains

A Customer Profiles **domain** is the top-level container for all profile data within a Connect instance. Each instance has one domain.

```javascript
import { CustomerProfilesClient, CreateDomainCommand } from "@aws-sdk/client-customer-profiles";

const client = new CustomerProfilesClient({ region: "us-east-1" });

await client.send(new CreateDomainCommand({
  DomainName: "my-connect-domain",
  DefaultExpirationDays: 365,
  DefaultEncryptionKey: "arn:aws:kms:us-east-1:123456789012:key/key-id",
}));
```

### Profiles

A profile represents a single customer. Core profile fields include:

| Field | Description |
|---|---|
| `ProfileId` | Unique identifier |
| `AccountNumber` | Customer account number |
| `FirstName`, `MiddleName`, `LastName` | Name fields |
| `PhoneNumber`, `MobilePhoneNumber`, `HomePhoneNumber`, `BusinessPhoneNumber` | Phone numbers |
| `EmailAddress`, `PersonalEmailAddress`, `BusinessEmailAddress` | Email addresses |
| `Address`, `ShippingAddress`, `MailingAddress`, `BillingAddress` | Address objects |
| `BirthDate`, `Gender` | Demographic fields |
| `BusinessName` | Company name |
| `Attributes` | Custom key-value pairs |

### Object Types and Templates

**Object types** define the schema for data ingested into Customer Profiles. Each external data source maps to an object type.

**Templates** are pre-built object type definitions for common integrations:

| Template | Source |
|---|---|
| `_salesforceAccount` | Salesforce Account objects |
| `_salesforceContact` | Salesforce Contact objects |
| `_zendeskUsers` | Zendesk user profiles |
| `_zendeskTickets` | Zendesk tickets |
| `_serviceNowIncidents` | ServiceNow incidents |
| `_serviceNowSysUsers` | ServiceNow users |
| `_marketo` | Marketo leads |
| `_S3Connector` | Custom data from S3 |

Custom object types can be created for any data source:

```javascript
await client.send(new PutProfileObjectTypeCommand({
  DomainName: "my-domain",
  ObjectTypeName: "OrderHistory",
  Description: "Customer order history",
  ExpirationDays: 730,
  Fields: {
    orderId: { Source: "_source.orderId", Target: "_profile.Attributes.orderId" },
    orderDate: { Source: "_source.orderDate", Target: "_profile.Attributes.orderDate" },
    amount: { Source: "_source.amount", Target: "_profile.Attributes.orderAmount" },
  },
  Keys: {
    _orderId: [{ StandardIdentifiers: ["UNIQUE"], FieldNames: ["orderId"] }],
    _phone: [{ StandardIdentifiers: ["PROFILE", "LOOKUP_ONLY"], FieldNames: ["phone"] }],
  },
}));
```

## Agent View

When a contact is handled, the agent sees a unified profile panel in the agent workspace:

- **Contact history** -- previous interactions with this customer across all channels
- **Cases** -- open and closed cases associated with this customer
- **Product purchases** -- order and product data from integrated systems
- **Additional information** -- custom attributes from any connected data source

The profile is automatically matched to the incoming contact using phone number, email, or account ID.

## Identity Resolution

Identity resolution automatically merges duplicate profiles that represent the same customer.

### Auto-Merge

When enabled, Customer Profiles uses ML-based matching to identify and merge duplicates based on configurable matching rules (name + phone, email + address, etc.).

### Manual Resolution APIs

| API | Purpose |
|---|---|
| `GetMatches` | Returns pairs of profiles identified as potential duplicates |
| `GetSimilarProfiles` | Finds profiles similar to a given profile |
| `MergeProfiles` | Merges two or more profiles into one |
| `GetAutoMergingPreview` | Preview auto-merge results before enabling |

### Rule-Based Matching

Define matching rules that specify which fields must match for profiles to be considered duplicates:

```javascript
await client.send(new CreateDomainCommand({
  DomainName: "my-domain",
  Matching: {
    Enabled: true,
    AutoMerging: {
      Enabled: true,
      ConflictResolution: {
        ConflictResolvingModel: "RECENCY",
        SourceName: "salesforce",
      },
      Consolidation: {
        MatchingAttributesList: [
          ["PhoneNumber", "LastName"],
          ["EmailAddress"],
          ["AccountNumber"],
        ],
      },
      MinAllowedConfidenceScoreForMerging: 0.8,
    },
  },
}));
```

**Conflict resolution models:**
- `RECENCY` -- most recently updated value wins
- `SOURCE` -- value from a specified source wins

## Calculated Attributes

Define derived attributes that are computed from profile data in real-time. Examples: lifetime purchase value, average handle time, contact frequency.

```javascript
await client.send(new CreateCalculatedAttributeDefinitionCommand({
  DomainName: "my-domain",
  CalculatedAttributeName: "TotalOrderValue",
  DisplayName: "Total Order Value",
  Description: "Sum of all order amounts for this customer",
  AttributeDetails: {
    Attributes: [{ ObjectTypeName: "OrderHistory", Name: "amount" }],
    Expression: { AttributeItem: { Name: "amount" } },
  },
  Statistic: "SUM",
}));
```

**Supported statistics:** `FIRST_OCCURRENCE`, `LAST_OCCURRENCE`, `COUNT`, `SUM`, `MINIMUM`, `MAXIMUM`, `AVERAGE`, `MAX_OCCURRENCE`

## Segments

Create customer segments based on profile attributes and calculated attributes for targeted outreach.

| API | Purpose |
|---|---|
| `CreateSegmentDefinition` | Define a segment with filter criteria |
| `GetSegmentDefinition` | Retrieve a segment definition |
| `GetSegmentMembership` | Check if specific profiles belong to a segment |
| `GetSegmentSnapshot` | Get the current members of a segment |
| `ListSegmentDefinitions` | List all segment definitions |

Segments can be used to:
- Target outbound campaigns to specific customer groups
- Generate reports on customer cohorts
- Trigger workflows when customers enter or exit segments

## Event Triggers

Event triggers fire when profile data changes match specified conditions, enabling real-time automation:

```javascript
await client.send(new CreateEventTriggerCommand({
  DomainName: "my-domain",
  EventTriggerName: "HighValueCustomerAlert",
  ObjectTypeName: "OrderHistory",
  EventTriggerConditions: [
    {
      EventTriggerDimensions: [
        {
          Attributes: [
            {
              FieldName: "amount",
              DimensionType: "INCLUSIVE",
              Values: ["1000"],
            },
          ],
        },
      ],
      LogicalOperator: "ANY",
    },
  ],
  EventTriggerLimits: {
    MaxInvocationsPerProfile: 5,
  },
}));
```

## Recommenders

Recommenders provide ML-powered suggestions to agents based on customer profile data. They integrate with the agent workspace to surface relevant products, knowledge articles, or next-best-actions.

## Upload Jobs

Bulk import profile data from S3 using upload jobs:

```javascript
await client.send(new CreateBatchImportJobCommand({
  DomainName: "my-domain",
  DataFormat: "CSV",
  S3Url: "s3://my-bucket/profiles/import.csv",
  MappingId: "mapping-uuid",
}));
```

## Domain Layouts

Domain layouts control how profile data is presented in the agent workspace. Define which fields appear, their order, and grouping.

## GDPR Compliance

Customer Profiles supports GDPR requirements:

- **Right to erasure:** `DeleteProfile` removes all profile data
- **Right to access:** `SearchProfiles` and `GetProfileObjectType` retrieve stored data
- **Data retention:** `DefaultExpirationDays` auto-deletes data after a configured period
- **Encryption:** All data encrypted at rest using KMS keys

## Key APIs

| API | Purpose |
|---|---|
| `CreateDomain` | Create a Customer Profiles domain |
| `CreateProfile` | Create a new customer profile |
| `UpdateProfile` | Update profile fields |
| `SearchProfiles` | Search profiles by attributes |
| `GetProfile` | Get a specific profile |
| `DeleteProfile` | Delete a profile and all associated data |
| `PutProfileObject` | Ingest a data object into a profile |
| `ListProfileObjects` | List objects associated with a profile |
| `AddProfileKey` | Add a lookup key to a profile |
| `PutIntegration` | Configure a data source integration |
| `ListIntegrations` | List configured integrations |
| `MergeProfiles` | Merge duplicate profiles |

All APIs are available via `@aws-sdk/client-customer-profiles`.

## Key Considerations

- **Limits:** Default 100M profiles per domain, 1000 object types per domain
- **Latency:** Profile lookups are single-digit millisecond; identity resolution runs asynchronously
- **Integration lag:** External data sources (Salesforce, Zendesk) sync on a schedule, not real-time. Contact history is updated in near-real-time.
- **Encryption:** Customer-managed KMS keys supported for domain encryption
- **Cross-region:** Customer Profiles is regional. For global resiliency, profiles must be replicated manually.
- **Cost:** Billed per profile per month and per identity resolution job
