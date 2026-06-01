# Amazon Connect — Service-Linked Roles

Service-linked roles (SLRs) that Amazon Connect and its features use to call other AWS services on your behalf.

## Overview

A service-linked role is a unique type of IAM role linked directly to Amazon Connect. SLRs are predefined by Amazon Connect and include all the permissions the service requires to call other AWS services on your behalf — so you don't have to manually add those permissions.

Key characteristics:

- Amazon Connect defines the permissions of its SLRs. Unless defined otherwise, only Amazon Connect can assume its roles.
- The defined permissions include both the **trust policy** and the **permissions policy**.
- The permissions policy cannot be attached to any other IAM entity.
- You can delete an SLR only after first deleting its related resources. This protects your Amazon Connect resources from inadvertently losing access.

To create, edit, or delete an SLR, you must configure permissions to allow your users, groups, or roles to do so (see Service-linked role permissions in the IAM User Guide).

For a list of other services that support SLRs, see *AWS services that work with IAM* and look for services with **Yes** in the Service-linked role column.

## Core Connect SLR: `AWSServiceRoleForAmazonConnect`

The primary SLR — allows Amazon Connect to create and manage AWS resources on your behalf.

| Property | Value |
|---|---|
| Role name | `AWSServiceRoleForAmazonConnect` |
| Service principal (trust) | `connect.amazonaws.com` |
| Permissions policy | `AmazonConnectServiceLinkedRolePolicy` |

**Permissions granted:** the policy allows Amazon Connect to perform various Amazon Connect actions on your Amazon Connect resources.

### Creating

- You don't need to manually create this role. When you create an instance in the Amazon Connect console, Amazon Connect creates the SLR for you.
- Amazon Connect also supports creating the SLR using the IAM console, IAM CLI, or IAM API.
- If you delete this SLR and later need it again, the same process recreates it: creating an instance creates the SLR again.

> **Important:** If this role has been deleted, recreating an instance won't recreate the role. You may receive an error saying your account already has an instance, even though you've deleted all instances. If you hit this, see the troubleshooting section to recreate the role.

### Editing

- Amazon Connect does **not** allow you to edit the `AWSServiceRoleForAmazonConnect` SLR.
- You cannot change the role's name after creation because various entities might reference it.
- You **can** edit the role's description using IAM.

### Deleting

You must clean up the role's resources before you can manually delete it:

1. Delete your Amazon Connect instances (see *Delete your Amazon Connect instance*).
2. Use the IAM console, IAM CLI, or IAM API to delete the `AWSServiceRoleForAmazonConnect` SLR.

> **Note:** If the Amazon Connect service is using the role when you try to delete the resources, the deletion might fail. Wait a few minutes and try again.

### Supported Regions

Amazon Connect supports using SLRs in all Regions where the service is available. See Amazon Connect endpoints and quotas.

## Outbound Campaigns SLRs

Amazon Connect outbound campaigns uses a single SLR: `AWSServiceRoleForConnectCampaigns`.

### `AWSServiceRoleForConnectCampaigns`

| Property | Value |
|---|---|
| Role name | `AWSServiceRoleForConnectCampaigns` |
| Service principal (trust) | `connect-campaigns.amazonaws.com` |
| Permissions policy | `AmazonConnectCampaignsServiceLinkedRolePolicy` |

**Permissions granted:** `connect-campaigns:ListCampaigns`; `connect:BatchPutContact`/`StopContact` across instances; per-instance `connect:StartOutboundVoiceContact, GetMetricData, GetCurrentMetricData, BatchPutContact, StopContact, GetMetricDataV2, DescribeContactFlow, SendOutboundEmail`; EventBridge rule management for `ConnectCampaignsRule*`; and `wisdom:GetMessageTemplate`/`RenderMessageTemplate` on `AmazonConnectCampaignsEnabled`-tagged resources.

### Creating

You don't need to manually create this SLR. It is created when you onboard outbound campaigns for your instance (`StartInstanceOnboardingJob`).

## Customer Profiles SLR: `AWSServiceRoleForProfile`

Allows Amazon Connect Customer Profiles to access AWS services and resources on your behalf.

| Property | Value |
|---|---|
| Role name | `AWSServiceRoleForProfile_{unique-id}` (one per domain) |
| Service principal (trust) | `profile.amazonaws.com` |
| Permissions policy | `CustomerProfilesServiceLinkedRolePolicy` |

**Permissions granted:** `cloudwatch:PutMetricData`; `iam:DeleteRole`; `connect-campaigns:PutProfileOutboundRequestBatch`; and `profile:BatchGetProfile, GetRecommender, GetCalculatedAttributeForProfile, GetProfileRecommendations`.

> **Important — KMS enforcement edge case:** Before January 31, 2025, Amazon Connect Customer Profiles did **not** enforce the use of a customer managed AWS KMS key. After January 31, 2025, the service **requires** a customer managed KMS key for encrypting profile data. If you created a domain before this date, review your KMS key configuration to ensure compliance with the updated requirements.

## Managed Synchronization SLR: `AWSServiceRoleForAmazonConnectSynchronization`

Backs **Amazon Connect Global Resiliency** — synchronizing configuration/resource data across replicated instances.

| Property | Value |
|---|---|
| Role name | `AWSServiceRoleForAmazonConnectSynchronization_{unique-id}` |
| Service principal (trust) | `synchronization.connect.amazonaws.com` |
| Permissions policy | `AmazonConnectSynchronizationServiceRolePolicy` |

**Permissions granted:** broad `connect:Create*/Update*/Delete*/Describe*/Batch*/List*/Search*/Associate*/Disassociate*/Get*/Import*/Tag*/Untag*` plus `cloudwatch:PutMetricData`, with a large **deny-list** (no `Start*/Stop*/Resume*/Suspend*`, no `*Contact(s)`, no `*MetricData*`, no `CreateInstance/DeleteInstance/ReplicateInstance/GetFederationToken`, no phone-number claim/release, no traffic-distribution-group actions). Created by `ReplicateInstance`.

## Summary Table

| Role | Trust principal | Policy | Used by |
|---|---|---|---|
| `AWSServiceRoleForAmazonConnect` | `connect.amazonaws.com` | `AmazonConnectServiceLinkedRolePolicy` | Core Amazon Connect |
| `AWSServiceRoleForConnectCampaigns` | `connect-campaigns.amazonaws.com` | `AmazonConnectCampaignsServiceLinkedRolePolicy` | Outbound campaigns |
| `AWSServiceRoleForProfile` | `profile.amazonaws.com` | `CustomerProfilesServiceLinkedRolePolicy` | Customer Profiles (one per domain) |
| `AWSServiceRoleForAmazonConnectSynchronization` | `synchronization.connect.amazonaws.com` | `AmazonConnectSynchronizationServiceRolePolicy` | Global Resiliency / cross-Region sync |
