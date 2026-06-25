# Signature Sleigh (`send-for-signature`)

Signature Sleigh is the human-friendly name for the `send-for-signature` Hermes skill: a browser-operated workflow for routing finalized Google Drive PDFs through DocuSign for signature. It combines Drive access, DocuSign browser automation, a required human approval checkpoint, and optional chat/tracker follow-up so legal-ops and contract-ops teams can route repeatable signature packets safely without a DocuSign API integration.

This README is the GitHub-facing overview. The executable agent instructions belong in `SKILL.md`; private requester defaults belong in a companion document that is not committed to the repository.

## What it does

Signature Sleigh helps an agent:

1. Confirm the requester-provided Google Drive file is a PDF intended by the requester as final.
2. Flag filenames or visible document text that suggest `DRAFT`, `WIP`, `redline`, or another non-final state.
3. Collect or load the internal signer, counter-signers, signing order, message, and optional completion-copy recipients.
4. Build a DocuSign envelope in the browser.
5. Place signature fields on the right parties' signature blocks.
6. Stop at DocuSign preview for explicit human approval.
7. Send only after approval.
8. Verify the envelope in the configured DocuSign account's sent-envelope view, where available.
9. Optionally provide an approved chat heads-up draft or post through a separately approved route.
10. Optionally log the sent envelope to a tracker if the requester's companion workflow specifies one.

The skill is designed for workflows where correctness, visibility, and auditability matter more than unattended automation.

## What this is not

Signature Sleigh is not legal review, contract drafting, approval authority, or a replacement for DocuSign account governance. It does not decide whether a document should be signed. It helps a user route an already-approved PDF through DocuSign with a visible approval checkpoint before the envelope is sent.

It is also not a DocuSign API product. Browser automation is useful when an organization has DocuSign access but has not provisioned an API-backed signing integration, but it is more sensitive to UI changes than an API integration.

## Why browser-based instead of the DocuSign API?

Signature Sleigh intentionally works through the existing DocuSign web app:

- No DocuSign API key, JWT, admin consent, or integration user is required.
- It can use an authenticated browser profile or the user's normal DocuSign login flow.
- A user can watch the live browser during sensitive sends.
- The agent works from the same web UI a human sender would normally use, including account prompts, recipient-role defaults, preview screens, and send confirmation pages.

The skill does not collect or store DocuSign passwords. Authentication must be handled through the user's normal browser/session flow, saved-password manager, SSO, or another locally approved authentication mechanism.

## Core principles

### Human approval is mandatory

Signature Sleigh must stop at preview and wait for explicit approval before sending. Silence, ambiguity, or a general prior instruction is not enough.

### The skill stays generic

The reusable skill contains the general mechanics: PDF verification, DocuSign browser steps, recipient setup, field placement, preview, send verification, and optional notification/tracker patterns.

User-specific details belong in a private companion document, not in the skill itself. Examples include:

- Which DocuSign account to use.
- Which mailbox can receive two-factor authentication codes, if the local workflow permits the agent to read it.
- Default signing order.
- Default internal signers.
- Completion-copy recipients.
- Envelope tracker location.
- Chat heads-up preferences.
- Template-specific field-placement quirks.

Never commit companion documents, credentials, 2FA routing, real signer information, tracker IDs, private client names, or contract PDFs to the public repository.

### Source-of-truth checks beat assumptions

The workflow verifies concrete state before claiming success:

- Google Drive metadata confirms the file is a PDF.
- The downloaded file can be checked as a real PDF.
- DocuSign preview is inspected before send.
- DocuSign Sent or an equivalent sent-envelope/details view is checked after send, where available.
- Tracker writes, when used, are read back.

## Requirements

Signature Sleigh assumes an agent environment with:

- Hermes Agent skills support.
- Google Workspace access through `gog` or equivalent tooling for Drive/Gmail/Docs/Sheets.
- Browser automation through the Hermes browser harness or equivalent browser-control tooling.
- A DocuSign account accessible through the browser's normal authenticated session flow.
- An approved chat/outbound delivery tool if Slack or other chat updates are part of the workflow.

`gog` is Protocol Labs' Google Workspace command-line helper used by this skill to read Drive metadata, download PDFs, read companion docs, and optionally update Sheets. If you use a different environment, replace those calls with equivalent Google Workspace operations.

No DocuSign API credentials are required.

## Inputs

A typical request includes:

- A Google Drive PDF URL.
- Internal signer name and email.
- Counter-signer name(s) and email(s), if any.
- Signing order, or a companion document default.
- Optional DocuSign email subject/message.
- Optional chat heads-up request.

Signing order should be provided by the user or loaded from the requester's private companion document. If neither is available, the agent should ask before creating a production envelope rather than inventing a default.

## Quick start for a GitHub repository

1. Put the executable agent procedure in `send-for-signature/SKILL.md`.
2. Put this human-facing overview in `send-for-signature/README.md`.
3. Keep implementation notes and observed UI pitfalls in `send-for-signature/references/`.
4. Keep requester companion documents outside the repository.
5. Test with a non-production PDF and stop at DocuSign preview before enabling normal use.

Suggested layout:

```text
send-for-signature/
├── SKILL.md
├── README.md
└── references/
    ├── docusign-web-ui.md
    ├── docusign-login-autofill.md
    ├── slack-or-chat-heads-up-routing.md
    ├── envelope-tracker-status-refresh.md
    └── pre-rollout-test-plan.md
```

`SKILL.md` should contain the executable agent procedure. `README.md` should explain the workflow to humans reviewing the repository. Reference files should capture UI observations, pitfalls, and reusable troubleshooting notes.

## High-level workflow

### 1. Load the requester's companion document, if one exists

At the beginning of a run, the agent reads the requester's private companion document if one exists. The companion document provides user-specific defaults and prevents the generic skill from hard-coding one person's preferences.

If there is no companion document yet, the agent should collect the required details for the run. It may help the user create a reusable companion document, but only as an intentional setup step in the local/private workspace.

### 2. Verify the Drive PDF

The agent extracts the Drive file ID, reads metadata, confirms the MIME type is `application/pdf`, and downloads the PDF locally for upload into DocuSign.

If the file is a Google Doc, DOCX, or other non-PDF source, the workflow stops and asks the user for a finalized PDF. If the filename or visible document text suggests `DRAFT`, `WIP`, `redline`, or a version that may not be final, the agent flags that and asks for confirmation before proceeding.

### 3. Collect signing details

The agent confirms:

- Internal signer.
- Counter-signers.
- Signing order.
- Subject and message.
- Completion-copy recipients from the companion document, if any.
- Whether a chat heads-up is requested.

The agent should not infer signer roles from email domains or from who made the request. If the user labels someone as the internal signer or counter-signer, those labels control unless the user corrects them.

### 4. Show a pre-send checklist

Before building or sending, the agent presents a concise preview of the intended routing:

```text
Ready to prepare:
  Document: Example Agreement.pdf
  Subject:  Example Agreement
  Execution order: Counter-signer first, then internal signer
    1. Jane Smith <jane@example.com>       (counter-signer)
    2. Chris Person <chris@example.org>    (internal signer)
  Completion copies: per companion document, if configured
  Chat heads-up: draft requested
  Checklist: PDF verified; signer emails look valid; DocuSign preview required before send
```

### 5. Build the DocuSign envelope

Using browser automation, the agent opens DocuSign, uploads the PDF, adds recipients, enables signing order if needed, sets each recipient's role, adds any completion-copy recipients as `Receives a Copy`, and enters the subject/message.

### 6. Place fields carefully

The agent places only the fields each signer needs, typically Signature and Date Signed, plus Name, Initials, Title, or read-only Text fields when appropriate. It verifies fields are assigned to the correct recipient and do not cover printed text.

If the document already has printed names or titles, the agent avoids duplicating them unless the user specifically needs those fields for a test or template requirement.

### 7. Preview and wait

The agent uses DocuSign Preview to simulate the signer experience and confirm required fields are visible and assigned correctly. For watched tests, this is the natural stopping point.

The agent does not click Send until the user gives explicit approval.

### 8. Send and verify

After approval, the agent sends the envelope and verifies it in DocuSign Sent or the configured account's equivalent sent-envelope/details view. Verification should identify the envelope/document name, current status, recipients, and first pending signer where the UI exposes those details.

### 9. Notify and log, if configured

If a chat heads-up is requested, the default behavior is to draft the message for approval first. Direct posting to channels or DMs should happen only when explicitly approved and technically reachable through the local delivery tooling.

If the requester's companion document specifies an envelope tracker, the agent logs the sent envelope idempotently and reads the row back before reporting that it was logged.

## Safety guardrails

Signature Sleigh should not:

- Send without preview and explicit approval.
- Accept non-PDF source files as final signature packets.
- Guess DocuSign passwords or ask the user to reveal passwords.
- Bypass MFA or ask users to disclose authentication secrets.
- Hard-code one user's DocuSign account, 2FA routing, CC recipients, tracker, or chat preferences into the generic skill.
- Continue clicking through DocuSign when the UI does not match expectations.
- Post chat heads-ups without approval.
- Claim an envelope was sent, logged, or verified unless the agent inspected the relevant source of truth during the run.

## Privacy and data handling

This workflow touches contracts, signer emails, DocuSign account state, chat messages, and optional tracker rows. Treat all local outputs as sensitive.

Do not commit:

- Downloaded PDFs or signature packets.
- Companion documents.
- DocuSign account names, passwords, tokens, or 2FA routing.
- Real signer names/emails unless they are already part of a public example and safe to publish.
- Tracker spreadsheet IDs or private workspace links.
- Slack/channel/DM identifiers from a real workspace.

Clean up downloaded PDFs and screenshots according to your local retention policy.

## Companion document pattern

A companion document keeps user-specific setup separate from the reusable skill. A minimal companion document can include:

```markdown
# Send for Signature — <User>'s preferences

## DocuSign login
- Account / saved login to use:
- Verification-code email the agent can read, if permitted by local policy:

## Signing defaults
- Default signing order:
- Usual internal signer(s):
- Completion-copy recipients and timing:
- Chat person/channel for heads-ups:
- Envelope tracker, if any:

## Template / field-placement notes
- Standard signature-block quirks:
```

This separation makes the GitHub skill reusable across people and teams while preserving each user's private operational preferences outside the public repository.

## Example prompt

```text
Run Signature Sleigh for this PDF:

Drive PDF: https://drive.google.com/file/d/<file-id>/view
Internal signer: Chris Person, chris@example.org
Counter-signer: Jane Smith, jane@example.com
Signing order: counter-signer first, internal last
Chat heads-up: draft one for approval

Build the DocuSign envelope and stop at preview before sending.
```

## Testing and dry runs

Before using the workflow on production documents:

1. Use a non-production PDF with clearly fake content.
2. Use test recipients or a DocuSign sandbox/test account if available.
3. Exercise the workflow through field placement and DocuSign Preview.
4. Stop at preview unless the test account and recipients are safe to send to.
5. Verify that cleanup steps remove or abandon unsent drafts where appropriate.

For watched tests, the user can keep the live browser open while the agent performs the run. The chat thread remains the control channel for checkpoints and approvals.

## Limitations

- DocuSign UI changes can break browser automation and may require reference updates.
- Browser automation is slower and more fragile than API integration.
- Human approval is required; the workflow is intentionally not unattended send automation.
- The skill may not support every DocuSign account configuration, template, recipient role, field type, or regional UI variant.
- It does not replace legal, finance, or business approval for the underlying document.

## Contributing

When updating the skill:

- Keep reusable mechanics in `SKILL.md` or `references/`.
- Keep user-specific defaults in private companion documents.
- Add observed DocuSign UI changes to a reference file with date and reproduction notes.
- Preserve the preview-and-explicit-approval guardrail.
- Test changes with a dry run before relying on them for a production envelope.

## Status

Signature Sleigh is a browser-operated signing workflow for careful, human-supervised legal-ops and contract-ops use. It is appropriate when the requester wants visibility and approval before any envelope leaves DocuSign, and when a DocuSign API integration is unavailable or intentionally out of scope.
