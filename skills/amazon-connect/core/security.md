# Amazon Connect — Security

## Shared Responsibility Model
- **AWS**: Security OF the cloud (infrastructure, availability)
- **You**: Security IN the cloud (config, access, data classification)

## Identity & Access Management (IAM)
- Service-linked role: `AWSServiceRoleForAmazonConnect`
- Resource-based policies for Lambda, Lex, S3, Kinesis
- Tag-based access control (TBAC) on Connect resources
- Security profiles control agent/admin permissions within Connect

## Security Profiles
- Predefined: Admin, Agent, CallCenterManager, QualityAnalyst
- Custom profiles with granular permissions
- Permissions grouped by resource (Routing, Flows, Metrics, etc.)
- `CreateSecurityProfile`, `UpdateSecurityProfile`, `ListSecurityProfilePermissions`

## Data Protection
- **Encryption at rest**: S3 (SSE-S3 or SSE-KMS), Kinesis (SSE)
- **Encryption in transit**: TLS 1.2+
- **Customer input encryption**: Public signing key in flows
- **PII redaction**: Contact Lens redacts from transcripts + audio

## Authentication Profiles
- `UpdateAuthenticationProfile` — configure per instance
- IP address restrictions (CIDR ranges)
- Session timeout configuration
- SAML 2.0 federation support

## Compliance
- GDPR, HIPAA eligible, PCI DSS, SOC 1/2/3, ISO 27001
- FedRAMP (GovCloud)
- HITRUST CSF

## Infrastructure Security
- Runs in AWS VPC
- No public endpoints exposed by default
- CloudTrail logs all API calls
- CloudWatch monitors instance health

## Cross-Service Confused Deputy Prevention
- Use `aws:SourceArn` and `aws:SourceAccount` condition keys
- Prevents services from being used as proxies

## Key Topics
- Data protection, IAM, logging/monitoring, tagging
- Compliance validation, resilience, infrastructure security
- Security best practices, authentication profiles
