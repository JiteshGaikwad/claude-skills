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

Amazon Connect outbound campaigns uses the following SLRs:

- `AWSServiceRoleForConnectCampaigns`
- `AWSServiceRoleForConnectCampaignsExecution`
- `AWSServiceRoleForAmazonConnectVoiceIDAccess`

### `AWSServiceRoleForConnectCampaigns`

| Property | Value |
|---|---|
| Role name | `AWSServiceRoleForConnectCampaigns` |
| Service principal (trust) | `connect-campaigns.amazonaws.com` |
| Permissions policy | `AmazonConnectCampaignsServiceRolePolicy` |

**Permissions granted:** allows Amazon Connect outbound campaigns to perform connect-campaigns related actions to manage outbound campaigns.

### `AWSServiceRoleForConnectCampaignsExecution`

| Property | Value |
|---|---|
| Role name | `AWSServiceRoleForConnectCampaignsExecution` |
| Service principal (trust) | `connect-campaigns.amazonaws.com` |
| Permissions policy | `AmazonConnectCampaignsExecutionRolePolicy` |

**Permissions granted:** allows the service to perform actions such as starting and managing outbound campaign executions, and interacting with Amazon Connect and Amazon Pinpoint resources.

### `AWSServiceRoleForAmazonConnectVoiceIDAccess`

Listed among the SLRs used by outbound campaigns. (The source provides no further trust/policy detail for this specific role beyond its inclusion in the outbound campaigns list.)

### Creating

You don't need to manually create these SLRs. When you enable outbound campaigns for your Amazon Connect instance, the service creates them for you.

## Customer Profiles SLR: `AWSServiceRoleForProfile`

Allows Amazon Connect Customer Profiles to access AWS services and resources on your behalf.

| Property | Value |
|---|---|
| Role name | `AWSServiceRoleForProfile` |
| Service principal (trust) | `profile.amazonaws.com` |

**Permissions granted:** allows Amazon Connect Customer Profiles to perform actions such as accessing customer profile data and integrating with other AWS services.

> **Important — KMS enforcement edge case:** Before January 31, 2025, Amazon Connect Customer Profiles did **not** enforce the use of a customer managed AWS KMS key. After January 31, 2025, the service **requires** a customer managed KMS key for encrypting profile data. If you created a domain before this date, review your KMS key configuration to ensure compliance with the updated requirements.

## Voice ID SLR: `AWSServiceRoleForAmazonConnectVoiceID`

Allows Amazon Connect Voice ID to access AWS resources on your behalf.

| Property | Value |
|---|---|
| Role name | `AWSServiceRoleForAmazonConnectVoiceID` |
| Service principal (trust) | `voiceid.amazonaws.com` |

**Permissions granted:** allows Amazon Connect Voice ID to perform speaker recognition and fraud detection tasks.

## Managed Synchronization SLR

Amazon Connect uses a service-linked role for managed synchronization to keep resources synchronized across Regions or services. The role allows Amazon Connect to synchronize configuration and resource data automatically.

(The source does not specify the role name, trust principal, or permissions policy for the managed synchronization SLR.)

## Summary Table

| Role | Trust principal | Used by |
|---|---|---|
| `AWSServiceRoleForAmazonConnect` | `connect.amazonaws.com` | Core Amazon Connect |
| `AWSServiceRoleForConnectCampaigns` | `connect-campaigns.amazonaws.com` | Outbound campaigns |
| `AWSServiceRoleForConnectCampaignsExecution` | `connect-campaigns.amazonaws.com` | Outbound campaign execution |
| `AWSServiceRoleForAmazonConnectVoiceIDAccess` | (not specified in source) | Outbound campaigns (Voice ID access) |
| `AWSServiceRoleForProfile` | `profile.amazonaws.com` | Customer Profiles |
| `AWSServiceRoleForAmazonConnectVoiceID` | `voiceid.amazonaws.com` | Voice ID |
| Managed synchronization SLR | (not specified in source) | Cross-Region/service resource sync |
