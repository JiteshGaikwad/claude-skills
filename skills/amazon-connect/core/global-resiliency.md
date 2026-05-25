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
