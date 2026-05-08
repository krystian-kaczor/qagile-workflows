# QAgile Workflows — Claude Instructions

## Before working on any workflow

Read the corresponding doc in `docs/` before touching a workflow JSON or making changes via MCP:

| Workflow file | Doc |
|---|---|
| `workflows/course-audit.json` | `docs/course-audit.md` |
| `workflows/score-reports-scrumorg.json` | `docs/score-reports-scrumorg.md` |
| `workflows/outlook-invoices-processing.json` | `docs/outlook-invoices-processing.md` |
| `workflows/qagile-monthly-newsletter.json` | `docs/qagile-monthly-newsletter.md` |

The docs contain: workflow ID, node map, flow diagram, credentials, spreadsheet IDs, tab naming conventions, and known issues. Reading them avoids re-fetching the full workflow definition each session.

## After changing a workflow

Update the corresponding `docs/` file to reflect any node additions, logic changes, or new credentials. The pre-commit hook will warn if you forgot.

## Workflow IDs (quick reference)

| Name | ID |
|---|---|
| Course Audit | `1MLHq9gEDw5lD8Y7` |
| Score Reports — scrum.org | `CQ6zEuIEIDS67Tjf` |
| Outlook Invoices Processing | `aGu19fNOGsMOi4BZ` |
| QAgile Monthly Newsletter | `LzfWyVEv7t85J4dW` |
| Mailchimp Template Updater | `QuHk7HIHBJQl0Moj` |

## n8n instance

`https://n8n.qagile.pl`

## Partial update syntax reminder

Always use `patches` array format:
```json
{ "id": "<workflow-id>", "patches": [ { "type": "updateNode", "nodeName": "...", "updates": { ... } } ] }
```

Never use flat parameters with `n8n_update_partial_workflow`.
