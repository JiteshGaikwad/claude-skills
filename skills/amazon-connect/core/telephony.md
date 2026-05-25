# Amazon Connect — Telephony & Phone Numbers

## Phone Number Types
- **DID (Direct Inward Dial)**: Local numbers, 110+ countries
- **Toll-free**: Available in select countries
- **Outbound caller ID**: Configure per queue

## Operations
- **Claim**: `ClaimPhoneNumber` — claim from available pool
- **Port**: Bring existing numbers via support case
- **Import**: `ImportPhoneNumber` — from external carrier
- **Release**: `ReleasePhoneNumber` — return to pool
- **Search**: `SearchAvailablePhoneNumbers` — find numbers by country/type/prefix

## Association
- Associate phone number to flow: `AssociatePhoneNumberContactFlow`
- One number → one flow
- Multiple numbers can point to same flow

## Outbound Calling
- 200+ outbound calling destinations
- Configure outbound caller ID per queue (`UpdateQueueOutboundCallerConfig`)
- E.164 format required for all numbers

## Telephony-as-a-Service
- AWS manages carrier network globally
- Proactive monitoring by telephony experts
- Auto-scales with demand
- No multi-year contracts or peak commitments

## Key APIs
- `ClaimPhoneNumber`, `ReleasePhoneNumber`, `ImportPhoneNumber`
- `ListPhoneNumbers`, `ListPhoneNumbersV2`, `DescribePhoneNumber`
- `SearchAvailablePhoneNumbers`
- `AssociatePhoneNumberContactFlow`, `DisassociatePhoneNumberContactFlow`
- `UpdatePhoneNumber`, `UpdatePhoneNumberMetadata`
