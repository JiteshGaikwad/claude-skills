# AI in Amazon Connect — Bedrock, Model Selection, Cross‑Region Inference

This page captures the Admin Guide guidance for AI features that use Amazon Bedrock-backed models.

## Bedrock data handling (high level)

Amazon Bedrock is the managed service used by Amazon Connect for certain AI features (summaries, semantic matching, evaluations, etc.).

Admin-guide posture to design around:

- Inputs (prompts) and outputs (responses) are not used to train the underlying foundation models.
- Treat prompts/responses as sensitive data anyway: avoid placing secrets/credentials in prompts.

## Model selection (service-managed)

For many features, Amazon Connect manages:

- model selection,
- prompt definition,
- and capacity provisioning.

Implication:

- The underlying model can change over time (feature improvements, model end-of-life transitions).
- Build monitoring around output quality and latency; don’t hard-code assumptions about a specific model.

## Cross‑region inference (what it means)

To use an optimal model for each feature, Amazon Connect may process inference in a different AWS Region than the Connect instance Region.

Key properties:

- Amazon Connect automatically selects an eligible inference region.
- Data is transmitted encrypted over AWS’s network (no public internet path required).

### Instance Region → possible inference Regions (Admin Guide snapshot)

This matrix is a point-in-time snapshot from the Admin Guide; it changes over time.

- `us-east-1` (N. Virginia) → `us-east-1`, `us-east-2`, `us-west-2`
- `us-west-2` (Oregon) → `us-east-1`, `us-east-2`, `us-west-2`
- `af-south-1` (Cape Town) → `af-south-1` + other commercial AWS Regions
- `ap-northeast-2` (Seoul) → `ap-northeast-2` + other commercial AWS Regions
- `ap-southeast-1` (Singapore) → `ap-southeast-1` + other commercial AWS Regions
- `ap-southeast-2` (Sydney) → `ap-southeast-2`, `ap-southeast-4`
- `ap-northeast-1` (Tokyo) → `ap-northeast-1`, `ap-northeast-3`
- `ca-central-1` (Canada Central) → `ca-central-1`
- `eu-central-1` (Frankfurt) → `eu-central-1`, `eu-west-1`, `eu-south-1`, `eu-west-3`, `eu-south-2`, `eu-north-1`
- `eu-west-2` (London) → `eu-west-2`, `eu-central-1`, `eu-west-1`, `eu-south-1`, `eu-west-3`, `eu-south-2`, `eu-north-1`
- `us-gov-west-1` (GovCloud West) → `us-gov-west-1`, `us-gov-east-1`

## Compliance

Both Amazon Connect and Amazon Bedrock participate in many AWS compliance programs. For compliance-specific requirements, validate against your organization’s compliance posture and the current AWS documentation.

