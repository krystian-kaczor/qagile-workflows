# QAgile Monthly Newsletter

**Workflow ID:** `LzfWyVEv7t85J4dW`
**Status:** Active
**Triggers:** Monthly schedule (1st of month) + Webhook

## Purpose

Generates and sends the QAgile monthly email newsletter via Mailchimp. Fetches HTML/text templates from GitHub, retrieves upcoming events and recent blog posts via sub-workflows, assembles the newsletter HTML, creates a Mailchimp campaign, saves HTML + text versions to Google Drive, and notifies Slack on success.

## Related Workflows

| Workflow | Called by | Purpose |
|---|---|---|
| `QuHk7HIHBJQl0Moj` | (standalone) | One-shot updater for Mailchimp template 10086294 |
| Get Upcoming Events | `Call Get Upcoming Events` node | Returns upcoming course events |
| Get Recent Posts | `Call Get Recent Posts` node | Returns recent WordPress blog posts |

## Node Map

| ID | Name | Type | Notes |
|---|---|---|---|
| 0fa315bc | Monthly Trigger | scheduleTrigger | Fires 1st of each month |
| webhook-trigger-01 | Webhook Trigger | webhook | Manual/external trigger |
| 3d22e6a7 | List Files to Get | github | Lists template files in repo |
| 60057d8d | Split Out | splitOut | One item per template file |
| d5658af4 | Get File | github | Fetches each file content |
| e8f0bb85 | Decode Files | code | Base64-decodes file content |
| 90d7716a | Merge Templates | code | Merges all template parts into one object |
| cce3842c | Call Get Upcoming Events | executeWorkflow | Sub-workflow: fetch upcoming courses |
| a514126d | Call Get Recent Posts | executeWorkflow | Sub-workflow: fetch recent blog posts |
| 6f46c5fd | Merge Data | merge | Merges templates + events + posts (3 inputs) |
| 5e28b951 | Render Intro | code | Renders intro section HTML |
| 5e6f4b1f | Render Events Section | code | Renders upcoming events HTML block |
| 3438d2e5 | Render Posts Section | code | Renders recent posts HTML block |
| c0911a99 | Assemble Newsletter | code | Combines all sections into final HTML + text |
| 5bf35599 | Prepare Sheet Data | code | (disabled Update Sent Posts path) Preps rows |
| 3338ca2a | Update Sent Posts | googleSheets | **DISABLED** — was meant to track sent issues |
| ddbcb1af | Prepare Newsletter File | code | Splits output into HTML file and TXT file items |
| node-filter-html | Filter HTML | filter | Passes only the HTML item |
| node-filter-txt | Filter TXT | filter | Passes only the TXT item |
| 9541b79e | Save Newsletter to Drive | googleDrive | Saves HTML to Google Drive |
| node-drive-txt-01 | Save Text to Drive | googleDrive | Saves plain text version to Drive |
| 084dfb54 | Slack Success Notification1 | slack | Posts "newsletter saved" to Slack |
| a1b2c3d4-0002 | Create Campaign | httpRequest | Mailchimp API: creates draft campaign |
| a1b2c3d4-0003 | Set Campaign Content | httpRequest | Mailchimp API: sets HTML content on campaign |
| a1b2c3d4-0005 | Fetch Template HTML | httpRequest | **DISABLED** — alternative template upload path |
| a1b2c3d4-0006 | Upload Template | httpRequest | **DISABLED** |

## Flow

```
Monthly Trigger / Webhook
  ├─ List Files to Get → Split Out → Get File → Decode Files → Merge Templates ──┐
  ├─ Call Get Upcoming Events ──────────────────────────────────────────────────►Merge Data
  └─ Call Get Recent Posts ─────────────────────────────────────────────────────┘
       → Render Intro → Render Events Section → Render Posts Section
       → Assemble Newsletter
            ├─ Prepare Sheet Data → Update Sent Posts (DISABLED)
            ├─ Create Campaign → Set Campaign Content
            └─ Prepare Newsletter File
                 ├─ Filter HTML → Save Newsletter to Drive → Slack Success Notification
                 └─ Filter TXT → Save Text to Drive
```

## Credentials

- GitHub token (read repo files — template parts)
- Google Drive OAuth (save newsletter files)
- Google Sheets OAuth (Update Sent Posts — disabled)
- Mailchimp API key (create campaign, set content)
- Slack Bot Token

## Mailchimp Details

- **Template ID:** `10086294` (updated by companion workflow `QuHk7HIHBJQl0Moj`)
- Campaign created fresh each month via API; not using saved templates
- `Set Campaign Content` sends HTML body directly

## Template Source

Templates fetched from a GitHub repository (configured in `List Files to Get`). Multiple files are fetched, decoded, and merged before rendering.

## Known Issues / Notes

- `Update Sent Posts` is disabled — sheet tracking of sent newsletters is not active.
- The disabled `Fetch Template HTML` + `Upload Template` path was an alternative approach to uploading to Mailchimp template store; abandoned in favour of direct campaign HTML.
- The `Merge Data` node takes 3 inputs: index 0 = templates, index 1 = recent posts, index 2 = upcoming events. Order matters for downstream render nodes.
