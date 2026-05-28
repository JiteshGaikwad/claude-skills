# Amazon Connect — Global Resiliency

## Architecture
- **Default**: Active-active within a single AWS Region (all customers get this)
- **Global Resiliency**: Active-active across multiple AWS Regions (opt-in)

## Traffic Distribution Groups
- `CreateTrafficDistributionGroup` — link instances across regions
- `UpdateTrafficDistribution` — control % of traffic per region
- `GetTrafficDistribution` — view current distribution
- Sign-in distribution: control which region agents sign into

## Key Concepts
- **Replicated instance**: `ReplicateInstance` creates a paired instance in another region
- **Phone number portability**: Numbers can failover between regions
- **Agent sign-in**: Agents can be directed to either region
- **Contact records**: Include `GlobalResiliencyMetadata` (ActiveRegion, OriginRegion, TrafficDistributionGroupId)

## APIs
- `CreateTrafficDistributionGroup`, `DeleteTrafficDistributionGroup`
- `DescribeTrafficDistributionGroup`, `ListTrafficDistributionGroups`
- `UpdateTrafficDistribution`, `GetTrafficDistribution`
- `AssociateTrafficDistributionGroupUser`, `DisassociateTrafficDistributionGroupUser`
- `ReplicateInstance`

## Requirements
- Both instances must be in the **same AWS account**
- Both instances must share the **same identity store** (SAML or Connect-managed)
- Supported regions: `us-east-1`, `us-west-2`, `eu-west-2`, `eu-central-1`, `ap-southeast-1`, `ap-southeast-2`, `ap-northeast-1`
- Instances must be created independently in each region before linking

## Getting Started
1. **Create instances** in two or more supported regions (same account, same identity provider)
2. **Create a traffic distribution group** using `CreateTrafficDistributionGroup` — specify the source instance ARN; the replicated instance is linked automatically
3. **Claim phone numbers** to the traffic distribution group (TDG) instead of to a single instance — this enables cross-region failover
4. **Configure distribution percentages** using `UpdateTrafficDistribution` — set the % of telephony and sign-in traffic routed to each region (e.g., 100/0 for active-standby, 50/50 for balanced)
5. **Test failover** by shifting distribution to 0/100 and verifying calls land in the secondary region

## Phone Numbers Across Regions
- Use `ClaimPhoneNumber` with `TargetArn` set to the **traffic distribution group ARN** (not an instance ARN)
- Numbers claimed to a TDG can serve calls in any linked region
- **Failover**: Update the telephony distribution via `UpdateTrafficDistribution` to shift 100% of calls to the healthy region
- Number porting and release work the same as single-region, but the number is associated with the TDG
- Each phone number can belong to only one TDG at a time

## Chat Across Regions
- Chat widgets must be configured with **region-aware** endpoints — the widget config specifies which region to connect to
- There is **no automatic chat failover** — if a region goes down, chat sessions in that region are lost
- **Persistent chat** is region-bound: a persistent chat session cannot be resumed in a different region
- To achieve chat resilience, implement application-level routing that directs new chat requests to the healthy region

## Metrics Across Regions
- Each region reports metrics **independently** — real-time and historical metrics are per-region
- There is **no cross-region console view** — you must open each region's Connect console separately
- For unified reporting, export CTRs and agent events from both regions to a **data lake** (S3, Kinesis) and aggregate with Athena, QuickSight, or a custom pipeline
- Contact records include `GlobalResiliencyMetadata` fields to correlate cross-region activity

## Limitations
- **Voice and chat only** — global resiliency supports cross-region distribution for voice and chat channels
- **Tasks** are region-bound and cannot failover
- **Email** is region-bound
- **Outbound campaigns** are region-bound
- Contact flows, queues, routing profiles, and other configuration must be **maintained in both regions** independently (no automatic sync)
- Agent hierarchy and historical changes are not replicated — manage them separately per region
