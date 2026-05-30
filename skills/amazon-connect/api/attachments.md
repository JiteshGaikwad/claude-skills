# Amazon Connect — Attachments (Developer Guide Narrative)

This page consolidates attachment guidance into one place. There are two common attachment paths that are easy to confuse:

- **Chat attachments** (customer ↔ agent chat): use the **Participant Service**.
- **Case attachments / attached files** (resources like Cases): use **Connect Service Attached Files** APIs.

## 1) Chat attachments (Participant Service)

Use the Participant Service APIs for chat attachments because they operate in the participant context.

Primary reference:

- `api/participant-api.md`

Practical flow (high level):

1. Obtain a participant connection.
2. Request an upload URL from Participant Service.
3. Upload the bytes to the provided URL.
4. Send the message referencing the uploaded attachment.

## 2) Attached files (Connect Service)

Use Connect Service “Attached Files” actions when you are attaching files to Connect-managed resources (for example, Cases).

Primary references:

- `api/connect-service-api.md` (Attached Files actions list)
- `api/overview.md` (high-level workflow notes)

Typical flow:

1. `StartAttachedFileUpload` — get a presigned upload URL.
2. Upload bytes to the URL.
3. `CompleteAttachedFileUpload` — confirm completion.
4. Associate/reference the file to the target resource (for example as a related item in Cases).

### Cases permission nuance

When attaching a file to a **Case**, the caller must also have permission to create the related item on that case (for example `cases:CreateRelatedItem`), in addition to whatever permissions are required for the attached files APIs.

## 3) Choosing the correct path

- If the file is part of a **chat transcript** experience → Participant Service.
- If the file belongs to a **case** or other Connect-managed resource → Connect Attached Files + Cases related items.

