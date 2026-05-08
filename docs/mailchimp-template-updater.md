# One-shot: Update Mailchimp Template 10086294

**Workflow ID:** `QuHk7HIHBJQl0Moj`
**Status:** Active
**Triggers:** Manual or Webhook

## Purpose

Single-purpose utility workflow. Fetches the latest newsletter HTML template from its source (GitHub/Drive) and PATCHes it to Mailchimp template ID `10086294`. Run this whenever the newsletter template design changes.

## Node Map

| ID | Name | Type | Notes |
|---|---|---|---|
| node-trigger-01 | Manual Trigger | manualTrigger | Manual run |
| node-webhook-01 | Webhook Trigger | webhook | External/automated trigger |
| node-fetch-01 | Fetch Template HTML | httpRequest | Fetches current template HTML |
| node-patch-01 | PATCH Mailchimp Template | httpRequest | PATCH `/templates/10086294` with new HTML |

## Flow

```
Manual / Webhook → Fetch Template HTML → PATCH Mailchimp Template
```

## Credentials

- Mailchimp API key (Basic auth: `anystring:<API_KEY>`)

## Notes

- This is a companion to the `QAgile Monthly Newsletter` workflow.
- Template ID `10086294` is hardcoded in `PATCH Mailchimp Template`.
- Run this after any CSS/HTML changes to the template before running the newsletter workflow.
