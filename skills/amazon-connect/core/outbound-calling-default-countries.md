# Outbound Calling — Default Country Allowlist (by Instance Region)

The AWS Region where your Amazon Connect instance is created determines which countries agents can call **by default**.

Connect uses a **default-deny** allowlisting model for outbound calling: if a destination country isn’t allowlisted for your instance/account, outbound dialing to that country is blocked.

Note: if your instance was created a while ago, your current default allowlist may differ from the lists below because service quotas and defaults have changed over time.

## Default outbound calling countries by Region

| Instance Region (where the Connect instance is created) | Countries you can call by default |
|---|---|
| US East (N. Virginia), US West (Oregon), Canada (Central), AWS GovCloud (US-West) | United States, Canada, Mexico, Puerto Rico, United Kingdom |
| Africa (Cape Town) | South Africa, United Kingdom, United States |
| Asia Pacific (Seoul) | South Korea, United Kingdom, United States |
| Asia Pacific (Singapore) | Singapore, Australia, Hong Kong, United States, United Kingdom |
| Asia Pacific (Sydney) | Australia, New Zealand, United States |
| Asia Pacific (Tokyo) | Japan, Vietnam, United States |
| EU (Frankfurt), EU (London) | United Kingdom, Italy, France, Ireland, United States |

For the full list of outbound calling destinations available by Region (beyond the default allowlist), refer to the Amazon Connect pricing page.

## Known prefix caveats (may not be allowlisted by default)

Some mobile-number prefixes may require an explicit quota/allowlist change even if the country is otherwise available.

- **United Kingdom**: mobile numbers starting with `+447` may not be allowed by default.
- **Japan**: mobile numbers starting with `+8170`, `+8180`, `+8190` may not be allowed by default.

If dialing is blocked for these prefixes, request an allowlist/quota update (next section).

## How to allow (or restrict) additional outbound calling countries

Use AWS Support to request changes to the outbound calling allowlist:

1. Open the AWS Support flow for **Account and billing** (this opens a pre-populated form).
2. Choose **Service**: “Connect (Number Management)”.
3. Choose **Category**: “Country Allowlisting for Outbound Calls”.
4. Select the severity and proceed to the “Additional information” step.
5. In the description, list the countries you want to **allow** (or **block/limit**) for outbound dialing.
6. Submit the case and wait for the Connect team review.

