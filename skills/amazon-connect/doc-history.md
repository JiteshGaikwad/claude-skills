# Documentation History — Mapping

Amazon Connect ships changes frequently. Rather than maintain a second dated
changelog here, this page maps each upstream AWS "Document history" page to the
file in this skill where the corresponding capability is documented, and points
to where dated entries live.

For this skill's own feature-level summary, see [`recent-changes.md`](recent-changes.md).

## Where AWS publishes dated changes

Each AWS guide ends with a "Document history" page listing dated entries. These
are the authoritative sources to verify exactly when a feature shipped:

- **Administrator Guide** — https://docs.aws.amazon.com/connect/latest/adminguide/doc-history.html
- **API Reference** — https://docs.aws.amazon.com/connect/latest/APIReference/doc-history.html
- **Agent Workspace Developer Guide** — https://docs.aws.amazon.com/agentworkspace/latest/devguide/ (no standalone history page; check the section updates within the guide)

Cross-service feed for net-new launches:

- **AWS What's New** — https://aws.amazon.com/about-aws/whats-new/recent/feed/

## Guide → skill-file mapping

Use this to find where a documented change is reflected in this skill.

| Upstream guide | Covers | Verify changes in this skill |
|---|---|---|
| Administrator Guide | Console setup, flows, routing, channels, analytics, security | `core/`, `flows/`, `channels/`, `analytics/`, `admin/` |
| API Reference (Connect Service) | `ConnectClient` actions and data types | `api/connect-service-api.md` |
| API Reference (Customer Profiles) | Profiles domains, object types, identity resolution | `api/customer-profiles-api.md` |
| API Reference (Cases) | Cases fields, templates, related items | `api/cases-api.md` |
| API Reference (Q Connect) | Assistants, knowledge bases, recommendations | `api/q-connect-api.md` |
| API Reference (Outbound Campaigns) | Campaigns V1/V2, dialing, limits | `api/campaigns-api.md` |
| API Reference (Contact Lens) | Real-time/post-call analytics actions | `api/contact-lens-api.md` |
| API Reference (Participant Service) | Chat participant + streaming actions | `api/participant-api.md` |
| API Reference (App Integrations) | Data integrations, event integrations | `api/app-integrations-api.md` |
| Agent Workspace Developer Guide | 3P apps, SDK clients, embedding, CSP | `agent-experience/developer-guide.md` and siblings |

## Keeping this current

When verifying a recent change:

1. Find the dated entry on the relevant AWS doc-history page above.
2. Locate the affected capability via the mapping table.
3. Update that file, then add a feature-level line to
   [`recent-changes.md`](recent-changes.md).

If full 1:1 parity with an AWS document-history table is ever needed, extract the
dated rows from the source guide and add them here as a structured list.
