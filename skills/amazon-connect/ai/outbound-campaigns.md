# Amazon Connect Outbound Campaigns

## Overview

Amazon Connect Outbound Campaigns enables high-volume outbound customer contact at scale -- capable of millions of contacts per day. It supports predictive, progressive, and agentless dialing modes across voice, email, SMS, and WhatsApp channels.

Two API versions exist:

- **Campaigns V1** -- original API, voice-only
- **Campaigns V2** -- current API, multi-channel support, enhanced scheduling and priority features

Use Campaigns V2 for all new implementations.

---

## Predictive Dialer

### ML-Powered Answering Machine Detection (AMD)

The predictive dialer uses machine learning to detect whether a human or answering machine answered the call, enabling agents to skip voicemails and connect only with live customers.

### AnsweringMachineDetectionStatus Values

12 possible detection statuses:

| Status | Description |
|--------|-------------|
| `HUMAN_ANSWERED` | A human picked up the call |
| `VOICEMAIL_BEEP` | Answering machine detected, beep tone heard (can leave message) |
| `VOICEMAIL_NO_BEEP` | Answering machine detected, no beep (greeting still playing or no beep machine) |
| `AMD_UNANSWERED` | Call was not answered within the detection window |
| `AMD_UNRESOLVED` | Detection could not determine human vs. machine |
| `AMD_NOT_APPLICABLE` | AMD was not enabled for this call |
| `SIT_TONE_DETECTED` | Special Information Tone detected (number disconnected, changed, etc.) |
| `SIT_TONE_BUSY` | SIT tone indicating the line is busy |
| `SIT_TONE_INVALID_NUMBER` | SIT tone indicating the number is invalid |
| `SIT_TONE_VACANT` | SIT tone indicating the number is vacant/disconnected |
| `FAX_MACHINE_DETECTED` | Fax machine tones detected |
| `AMD_ERROR` | An error occurred during detection |

### Dialing Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| **Predictive** | ML model predicts agent availability and dials ahead; optimizes for minimal agent idle time | High-volume sales, collections |
| **Progressive** | Dials one call per available agent; no risk of abandoned calls | Compliance-sensitive campaigns |
| **Agentless** | No agent involvement; plays a message or triggers a flow | Appointment reminders, notifications |

---

## Contact Priority Ordering

Campaigns V2 supports prioritizing which contacts to dial first based on customer profile attributes:

- Up to **10 profile attributes** can be used for priority ordering
- Attributes are evaluated in order (first attribute is highest priority)
- Supports ascending and descending sort per attribute
- Examples: prioritize by account value, days past due, last contact date, VIP status

```javascript
const { ConnectCampaignsV2Client, CreateCampaignCommand } = require("@aws-sdk/client-connectcampaignsv2");

const client = new ConnectCampaignsV2Client({ region: "us-east-1" });

await client.send(new CreateCampaignCommand({
  name: "CollectionsCampaign",
  connectInstanceId: "instance-id",
  channelSubtypeConfig: {
    telephony: {
      capacity: 1.0,
      outboundMode: {
        predictive: {}
      },
      defaultOutboundConfig: {
        connectContactFlowId: "flow-arn",
        connectSourcePhoneNumber: "+15551234567",
        answerMachineDetectionConfig: {
          enableAnswerMachineDetection: true,
          awaitAnswerMachinePrompt: false
        }
      }
    }
  },
  connectCampaignFlowArn: "campaign-flow-arn",
  schedule: {
    startTime: "2024-01-15T09:00:00Z",
    endTime: "2024-01-15T17:00:00Z",
    refreshFrequency: "PT1H"
  },
  source: {
    customerProfilesSegmentArn: "segment-arn"
  },
  communicationTimeConfig: {
    telephony: {
      openHours: {
        dailyHours: {
          MONDAY: [{ startTime: "09:00", endTime: "17:00" }],
          TUESDAY: [{ startTime: "09:00", endTime: "17:00" }],
          WEDNESDAY: [{ startTime: "09:00", endTime: "17:00" }],
          THURSDAY: [{ startTime: "09:00", endTime: "17:00" }],
          FRIDAY: [{ startTime: "09:00", endTime: "17:00" }]
        }
      },
      restrictedPeriods: {
        restrictedPeriodList: [
          { name: "NewYears", startDate: "2024-01-01", endDate: "2024-01-01" }
        ]
      }
    }
  }
}));
```

---

## Segment Refresh

- Campaign contact lists are sourced from **Customer Profiles segments**
- Segments refresh on an **hourly** cadence (previously 24-hour minimum)
- Newly qualifying contacts are added to the campaign automatically
- Contacts that no longer match the segment criteria are removed
- Manual refresh can be triggered via the API

---

## Multi-Contact Time Zone Detection

The system automatically detects contact time zones and enforces calling windows:

- Uses the contact's phone number area code and/or address to determine time zone
- Respects per-time-zone calling windows (e.g., only dial 9 AM - 9 PM in the contact's local time)
- Handles contacts spanning multiple time zones within a single campaign
- Compliant with TCPA and similar regulations that restrict calling hours

---

## Channel Subtypes (V2)

Campaigns V2 supports four channel subtypes:

### Voice

Traditional outbound phone calls with predictive/progressive/agentless dialing:

```javascript
channelSubtypeConfig: {
  telephony: {
    capacity: 1.0,
    outboundMode: { predictive: {} },
    defaultOutboundConfig: {
      connectContactFlowId: "flow-arn",
      connectSourcePhoneNumber: "+15551234567",
      answerMachineDetectionConfig: {
        enableAnswerMachineDetection: true
      }
    }
  }
}
```

### Email

Outbound email campaigns:

```javascript
channelSubtypeConfig: {
  email: {
    capacity: 5.0,  // Agents can handle multiple emails simultaneously
    outboundMode: { agentless: {} },
    defaultOutboundConfig: {
      connectSourceEmailAddress: "support@example.com",
      wisdomTemplateArn: "template-arn"  // Q Connect message template
    }
  }
}
```

### SMS

Text message campaigns:

```javascript
channelSubtypeConfig: {
  sms: {
    capacity: 5.0,
    outboundMode: { agentless: {} },
    defaultOutboundConfig: {
      connectSourcePhoneNumber: "+15551234567",
      wisdomTemplateArn: "template-arn"
    }
  }
}
```

### WhatsApp

WhatsApp Business messaging:

```javascript
channelSubtypeConfig: {
  whatsApp: {
    capacity: 5.0,
    outboundMode: { agentless: {} },
    defaultOutboundConfig: {
      connectSourcePhoneNumber: "+15551234567",
      wisdomTemplateArn: "template-arn"
    }
  }
}
```

---

## Batch Dial Requests

### PutDialRequestBatch (V1 -- Voice Only)

```javascript
const { ConnectCampaignsClient, PutDialRequestBatchCommand } = require("@aws-sdk/client-connectcampaigns");

const client = new ConnectCampaignsClient({ region: "us-east-1" });

await client.send(new PutDialRequestBatchCommand({
  id: "campaign-id",
  dialRequests: [
    {
      clientToken: "unique-token-1",
      phoneNumber: "+15551234567",
      expirationTime: "2024-01-15T23:59:59Z",
      attributes: {
        customerName: "Jane Smith",
        accountNumber: "12345"
      }
    },
    {
      clientToken: "unique-token-2",
      phoneNumber: "+15559876543",
      expirationTime: "2024-01-15T23:59:59Z",
      attributes: {
        customerName: "John Doe",
        accountNumber: "67890"
      }
    }
  ]
}));
```

### PutOutboundRequestBatch (V2 -- Multi-Channel)

```javascript
const { ConnectCampaignsV2Client, PutOutboundRequestBatchCommand } = require("@aws-sdk/client-connectcampaignsv2");

const client = new ConnectCampaignsV2Client({ region: "us-east-1" });

await client.send(new PutOutboundRequestBatchCommand({
  id: "campaign-id",
  outboundRequests: [
    {
      clientToken: "unique-token-1",
      channelSubtype: "TELEPHONY",  // or "EMAIL", "SMS", "WHATSAPP"
      expirationTime: "2024-01-15T23:59:59Z",
      channelSubtypeParameters: {
        telephony: {
          destinationPhoneNumber: "+15551234567",
          attributes: {
            customerName: "Jane Smith"
          },
          connectSourcePhoneNumber: "+15550001111"
        }
      }
    }
  ]
}));
```

---

## Communication Limits

Prevent over-contacting customers:

```javascript
communicationLimitsOverride: {
  allChannelSubtypes: {
    communicationLimitsList: [
      {
        maxCountPerRecipient: 3,
        frequency: 1,
        unit: "DAY"  // DAY or WEEK
      }
    ]
  }
}
```

- Limits apply per recipient across all channels or per channel subtype
- Configurable per day or per week
- System enforces limits automatically -- contacts exceeding the limit are skipped
- Override at the campaign level or set instance-wide defaults

---

## Scheduling

Campaign scheduling controls when the campaign is active:

```javascript
schedule: {
  startTime: "2024-01-15T09:00:00Z",
  endTime: "2024-03-15T17:00:00Z",
  refreshFrequency: "PT1H"  // ISO 8601 duration -- how often to refresh the contact segment
}
```

- **Start/end time** -- overall campaign window
- **Refresh frequency** -- how often to pull new contacts from the segment (minimum `PT1H`)
- **Open hours** -- per-day calling windows (see `communicationTimeConfig` above)
- **Restricted periods** -- blackout dates (holidays, maintenance windows)
- **Time zone aware** -- all scheduling respects the contact's local time zone

---

## Key API Operations

### Campaigns V2

| Operation | Description |
|-----------|-------------|
| `CreateCampaign` | Create a new outbound campaign |
| `UpdateCampaignChannelSubtypeConfig` | Update channel configuration |
| `UpdateCampaignSchedule` | Modify campaign schedule |
| `UpdateCampaignCommunicationTime` | Update calling windows |
| `UpdateCampaignCommunicationLimits` | Update contact frequency limits |
| `UpdateCampaignSource` | Change the contact segment source |
| `PutOutboundRequestBatch` | Submit contacts for outbound delivery |
| `StartCampaign` | Start a paused/created campaign |
| `PauseCampaign` | Pause a running campaign |
| `StopCampaign` | Stop a campaign permanently |
| `DeleteCampaign` | Delete a campaign |
| `GetCampaignState` | Check campaign status |
| `ListCampaigns` | List campaigns with filtering |
