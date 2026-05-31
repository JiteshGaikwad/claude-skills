# Amazon Connect — Regions and Feature Availability

Amazon Connect and its features are not all available in every AWS Region. Each feature has its own Region availability list, and within a Region individual capabilities (channels, AI features, sub-APIs) may differ. This document captures the Region and feature-availability information as documented in the Amazon Connect Administrator Guide.

> **Availability changes over time.** New Regions and features are added regularly, and the lists below reflect a point-in-time snapshot of the documentation. For the current authoritative list, see the AWS Region table (https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/) and the "Availability of Amazon Connect features by Region" section of the Amazon Connect Administrator Guide.

> **Note on completeness.** Some feature sections in the source documentation are truncated (the source lists Region bullets that were cut off). Where a list is known to be incomplete, this is called out explicitly. Do not treat a partial list here as exhaustive — verify against the AWS docs.

---

## Resiliency model (context for Region selection)

- Amazon Connect provides **active-active resilience within an AWS Region** for all customers. This ensures high availability for all channels and applications.
- For higher resilience across Regions, use **Amazon Connect Global Resiliency** (resilience across multiple AWS Regions). Global Resiliency itself has limited Region availability (see below).
- Telephony is integrated with multiple providers over redundant dedicated network paths to **three or more Availability Zones in every Region where the service is offered**. If a component, data center, or entire AZ fails, the affected endpoint is taken out of rotation.

---

## Amazon Connect core availability by Region

The core Amazon Connect service (instances) is available in the following Regions, with the documented endpoints. Regions with FIPS endpoints are noted.

| Region Name | Region | Endpoint(s) | Protocol |
| --- | --- | --- | --- |
| US East (N. Virginia) | us-east-1 | connect.us-east-1.amazonaws.com / connect-fips.us-east-1.amazonaws.com | HTTPS |
| US West (Oregon) | us-west-2 | connect.us-west-2.amazonaws.com / connect-fips.us-west-2.amazonaws.com | HTTPS |
| Africa (Cape Town) | af-south-1 | connect.af-south-1.amazonaws.com | HTTPS |
| Asia Pacific (Seoul) | ap-northeast-2 | connect.ap-northeast-2.amazonaws.com | HTTPS |
| Asia Pacific (Singapore) | ap-southeast-1 | connect.ap-southeast-1.amazonaws.com | HTTPS |
| Asia Pacific (Sydney) | ap-southeast-2 | connect.ap-southeast-2.amazonaws.com | HTTPS |
| Asia Pacific (Tokyo) | ap-northeast-1 | connect.ap-northeast-1.amazonaws.com | HTTPS |
| Canada (Central) | ca-central-1 | connect.ca-central-1.amazonaws.com / connect-fips.ca-central-1.amazonaws.com | HTTPS |
| Europe (Frankfurt) | eu-central-1 | connect.eu-central-1.amazonaws.com | HTTPS |
| Europe (London) | eu-west-2 | connect.eu-west-2.amazonaws.com | HTTPS |
| AWS GovCloud (US-West) | us-gov-west-1 | connect.us-gov-west-1.amazonaws.com | HTTPS |

FIPS endpoints are documented for: us-east-1, us-west-2, ca-central-1.

---

## Feature-availability matrix (by Region)

This matrix summarizes which features are available in each Region, based on the per-feature lists in the documentation. **Yes** = documented as available; **No** = not listed as available; **—** = the source list is truncated/incomplete for that feature (treat as unknown, verify against AWS docs).

Regions are abbreviated: USE1 (us-east-1), USW2 (us-west-2), AFS1 (af-south-1), APNE2 (ap-northeast-2 / Seoul), APSE1 (ap-southeast-1 / Singapore), APSE2 (ap-southeast-2 / Sydney), APNE1 (ap-northeast-1 / Tokyo), CAC1 (ca-central-1), EUC1 (eu-central-1), EUW2 (eu-west-2), GOV (us-gov-west-1).

| Feature | USE1 | USW2 | AFS1 | APNE2 | APSE1 | APSE2 | APNE1 | CAC1 | EUC1 | EUW2 | GOV |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Core Amazon Connect | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| AppIntegrations | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | No |
| Cases | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | No |
| Conversational analytics (Contact Lens) | Yes | Yes | Yes* | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| Customer Profiles | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | No |
| Customer Profiles — calculated attributes API | Yes | Yes | Yes | Yes | Yes | Yes | Yes | — | — | — | — |
| Tasks | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| Communications widget | Yes | Yes | No | Yes | Yes | Yes | Yes | Yes | Yes | Yes | No |
| Data lake | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| Agent workspace & step-by-step guides | Yes | Yes | Yes | Yes | Yes | — | — | — | — | Yes | — |

\* Africa (Cape Town): Amazon Connect is present but the **real-time voice API is not available** for conversational analytics in this Region (post-call analytics endpoint not documented). See the Contact Lens section below.

> Cells marked **—** are where the source documentation lists were truncated. The matrix is conservative: an absent entry from a complete list is **No**, but a truncated list is **—**.

---

## Feature-by-feature detail

### AppIntegrations

Available in: US East (N. Virginia), US West (Oregon), Africa (Cape Town), Asia Pacific (Seoul), Asia Pacific (Singapore), Asia Pacific (Sydney), Asia Pacific (Tokyo), Canada (Central), Europe (Frankfurt), Europe (London).

- **Not available in AWS GovCloud (US-West).**
- FIPS endpoints documented for: us-east-1, us-west-2, ca-central-1.
- Endpoint pattern: `app-integrations.<region>.amazonaws.com`.

### Cases

Available in: US East (N. Virginia), US West (Oregon), Africa (Cape Town), Asia Pacific (Seoul), Asia Pacific (Singapore), Asia Pacific (Sydney), Asia Pacific (Tokyo), Canada (Central), Europe (Frankfurt), Europe (London).

- **Not available in AWS GovCloud (US-West).**
- FIPS endpoints documented for: us-east-1, us-west-2.
- Endpoint pattern: `cases.<region>.amazonaws.com`.

### Conversational analytics (Contact Lens)

| Region Name | Region | Endpoint | Notes |
| --- | --- | --- | --- |
| US East (N. Virginia) | us-east-1 | contact-lens.us-east-1.amazonaws.com | |
| US West (Oregon) | us-west-2 | contact-lens.us-west-2.amazonaws.com | |
| Africa (Cape Town) | af-south-1 | — | **Real-time voice API is not available in this Region** |
| Asia Pacific (Seoul) | ap-northeast-2 | contact-lens.ap-northeast-2.amazonaws.com | |
| Asia Pacific (Singapore) | ap-southeast-1 | contact-lens.ap-southeast-1.amazonaws.com | |
| Asia Pacific (Sydney) | ap-southeast-2 | contact-lens.ap-southeast-2.amazonaws.com | |
| Asia Pacific (Tokyo) | ap-northeast-1 | contact-lens.ap-northeast-1.amazonaws.com | |
| Canada (Central) | ca-central-1 | contact-lens.ca-central-1.amazonaws.com | |
| Europe (Frankfurt) | eu-central-1 | contact-lens.eu-central-1.amazonaws.com | |
| Europe (London) | eu-west-2 | contact-lens.eu-west-2.amazonaws.com | |
| AWS GovCloud (US-West) | us-gov-west-1 | contact-lens.us-gov-west-1.amazonaws.com | |

Endpoint pattern: `contact-lens.<region>.amazonaws.com`.

### Customer Profiles

Available in: US East (N. Virginia), US West (Oregon), Africa (Cape Town), Asia Pacific (Seoul), Asia Pacific (Singapore), Asia Pacific (Sydney), Asia Pacific (Tokyo), Canada (Central), Europe (Frankfurt), Europe (London).

- **Not available in AWS GovCloud (US-West).**
- FIPS endpoints documented for: us-east-1, us-west-2, ca-central-1.
- Endpoints follow both `profile.<region>.amazonaws.com` and `profile.<region>.api.aws` patterns (with `profile-fips.*` variants in FIPS Regions).

**Customer Profiles — calculated attributes API** is available in a narrower set:

- US East (N. Virginia)
- US West (Oregon)
- Africa (Cape Town)
- Asia Pacific (Seoul)
- Asia Pacific (Singapore)
- Asia Pacific (Sydney)
- Asia Pacific (Tokyo)

> The source list for the calculated attributes API was truncated after Asia Pacific (Tokyo); additional Regions may exist. Verify against AWS docs.

### Tasks

Available in: US East (N. Virginia), US West (Oregon), Africa (Cape Town), Asia Pacific (Seoul), Asia Pacific (Singapore), Asia Pacific (Sydney), Asia Pacific (Tokyo), Canada (Central), Europe (Frankfurt), Europe (London), AWS GovCloud (US-West).

### Communications widget

Available in: US East (N. Virginia), US West (Oregon), Asia Pacific (Seoul), Asia Pacific (Singapore), Asia Pacific (Sydney), Asia Pacific (Tokyo), Canada (Central), Europe (Frankfurt), Europe (London).

- **Not listed** for Africa (Cape Town) or AWS GovCloud (US-West).

### Data lake

Available in: US East (N. Virginia), US West (Oregon), Africa (Cape Town), Asia Pacific (Seoul), Asia Pacific (Singapore), Asia Pacific (Sydney), Asia Pacific (Tokyo), Canada (Central), Europe (Frankfurt), Europe (London), AWS GovCloud (US-West).

### Agent workspace and step-by-step guides

The source list is **truncated** in the documentation. Documented entries include:

- US East (N. Virginia)
- US West (Oregon)
- Africa (Cape Town)
- Asia Pacific (Seoul)
- Asia Pacific (Singapore)
- Europe (London)

> Intermediate Regions in this list were cut off in the source; this is not a complete list. Verify against AWS docs.

---

## Messaging integrations (by channel, by Region)

Messaging channel availability varies within a Region. The following is documented:

| Region Name | Apple Messages for Business | SMS | WhatsApp Business Messaging | Push notifications |
| --- | --- | --- | --- | --- |
| US East (N. Virginia) | Yes | Yes | Yes | Yes |
| US West (Oregon) | Yes | Yes | Yes | Yes |
| Africa (Cape Town) | No | Yes | Yes | No |
| Asia Pacific (Seoul) | Yes | Yes | Yes | Yes |
| Asia Pacific (Singapore) | Yes | Yes | Yes | Yes |
| Asia Pacific (Sydney) | Yes | Yes | Yes | Yes |
| Asia Pacific (Tokyo) | Yes | Yes | Yes | Yes |
| Canada (Central) | Yes | Yes | Yes | Yes |
| Europe (Frankfurt) | Yes | Yes | Yes | Yes |
| Europe (London) | Yes | Yes | Yes | Yes |
| AWS GovCloud (US-West) | No | No | No | No |

Notable gaps: Africa (Cape Town) supports only SMS and WhatsApp (no Apple Messages, no push). AWS GovCloud (US-West) supports **none** of these messaging integrations.

---

## Other features with documented Region constraints

The "Availability of Amazon Connect features by Region" topic in the Administrator Guide also enumerates the following features as having their own Region lists. **The detailed Region lists for these were truncated or not fully present in the source**, so they are noted here only as features that are NOT universally available across all Regions. Verify each against AWS docs before relying on it in a given Region:

- **Agent workspace third-party applications**
- **Connect AI agents**
- **Customer authentication**
- **Forecasting, capacity planning, and scheduling**
- **Generative Voice: Set Voice Block**
- **Global Resiliency** — multi-Region resilience; availability is restricted to a documented subset of Regions (list not present in source).
- **In-app, web, and video calling capabilities**
- **Live media streaming**
- **Outbound campaigns** — the source documents that not all source-Region/destination combinations are supported (e.g., you cannot make campaign calls from Europe (London) to US phone numbers, or from Europe (Frankfurt) to New Zealand phone numbers). Region-pair restrictions apply.

> Treat the above as "region-restricted — verify"; the source did not provide complete Region lists for these features.

---

## AI features and cross-region inference

Amazon Connect includes AI-powered features — such as contact summarization, semantic rule matching, and performance evaluations — that use AI models via **Amazon Bedrock**.

- For many features, Amazon Connect **fully manages the underlying AI** (model selection, prompt definition, capacity provisioning) and **changes the underlying model over time** to unlock capabilities, improve performance, and ensure availability as models reach end of life.
- **Cross-region inference:** to use an optimal model for each feature, Amazon Connect may use cross-region inference — model inference (generating output from input) can be routed across Regions. This is relevant when reasoning about where AI processing physically occurs for data-residency purposes.
- Bedrock models available to these features never store, log, share, or train on customer prompts and responses.

---

## Regional endpoint notes

- **FIPS endpoints** (`<service>-fips.<region>.amazonaws.com`) are available for a subset of services/Regions only — documented for core Connect, AppIntegrations, Cases, and Customer Profiles in select US and Canada Regions (see per-feature sections).
- **Customer Profiles** additionally exposes `*.api.aws` endpoints alongside `*.amazonaws.com`.
- **SAML / SSO endpoint selection:** most identity providers default to the **global AWS sign-in endpoint** (Application Consumer Service / ACS) hosted in **US East (N. Virginia)**. AWS recommends **overriding this to the regional endpoint that matches the Region where your instance was created.** SAML usernames are case-sensitive. Global Resiliency deployments use a separate SAML sign-in endpoint and different instructions.
- **VPC endpoints (S3):** VPC endpoints are **not supported** for the Amazon S3 storage that Connect uses for recordings and exported reports.
- **Local and toll-free numbers** can be provided in **all Regions where the service is supported**. TFN connecting to multiple carriers is **only available in the US**.

---

## Data residency and Region selection considerations

Amazon Connect Region selection is contingent upon:

- **Data governance / residency requirements** — choose the Region that satisfies where your data must reside (and consider cross-region inference for AI features).
- **Use case** — the channels and features you need (see availability matrix above; not all features exist in all Regions).
- **Services available in each Region** — confirm every dependent feature (Contact Lens, Cases, Customer Profiles, outbound campaigns, etc.) exists in the candidate Region.
- **Telephony costs** in each Region.
- **Latency** relative to your agents, contacts, and external transfer endpoint geography.

Additional operational notes:

- Use **AWS Organizations** to separate dev / staging / QA / prod accounts and centrally govern billing, access, compliance, and resource sharing.
- **Single-region telephony and softphone architecture** is the documented resilience model for telephony/softphone within a Region; cross-Region telephony resilience requires **Global Resiliency**.
- **Phone number porting:** open porting requests well in advance (months) of go-live for critical workloads, including all requirements and cutover support needs.
- **Carrier diversity (US):** prefer toll-free numbers to load-balance across multiple carriers (active-active); for DIDs, spread across multiple carriers where possible.

---

## Source

Derived from the Amazon Connect Administrator Guide ("Availability of Amazon Connect features by Region", "Region selection considerations", "AI in Amazon Connect", and related sections). Region and feature availability changes over time — always confirm the current state against the AWS Region table and the Amazon Connect Administrator Guide.
