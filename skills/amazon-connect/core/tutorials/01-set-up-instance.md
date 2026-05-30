# Tutorial 1: Set Up Your Connect Customer Instance

Goal: open the Connect Customer console, create an instance, and claim a phone number for testing.

The Administrator Guide notes you can have multiple instances; each instance contains resources for a contact center (phone numbers, users, queues, etc.).

## Step 1: Launch Connect Customer

1) Sign in to the AWS Management Console.
2) Open the Services menu, search for “Connect Customer”, and open it.
3) If you see the welcome page, choose **Get started**.

Output: you can access the Connect Customer console.

## Step 2: Create an instance

1) On the instances page, choose **Add an instance**.
2) **Set identity**: enter a globally-unique Access URL alias for the instance, then continue.
3) **Add administrator**: create the first admin user for this instance (you’ll use it to sign in later via the Access URL).
   - Username is case sensitive.
   - Password requirements: 8–64 characters; include at least one uppercase letter, one lowercase letter, and one number.
4) **Set telephony**: accept defaults to allow inbound and outbound calling (adjust later if needed).
5) **Data storage**: accept defaults (you can revisit storage configuration later).
6) Review and create the instance.
7) After creation completes, choose **Get started**, then skip any optional onboarding prompts to reach the dashboard.

Outputs:
- An instance exists in the selected region.
- Your instance alias appears in the instance URL.

Related docs:
- `core/instances.md`
- `core/identity-management.md`
- `core/security.md`

## Step 3: Claim a phone number

1) In the instance navigation, go to **Channels → Phone numbers**.
2) Choose **Claim a number**.
3) Select the **DID** tab.
4) Choose a country/region (and optionally an area code, if available), then select an available number.
5) Record the claimed number (you’ll use it in Tutorial 2).
6) Add a description noting the number is for testing.
7) Assign the default sample inbound flow (the “first contact experience” sample flow) to the number, then save.

Outputs:
- A phone number is associated with the instance.
- The number routes into the sample inbound flow, so you can test voice immediately.

Related docs:
- `core/telephony.md`
- `channels/voice.md`
- `flows/overview.md`

