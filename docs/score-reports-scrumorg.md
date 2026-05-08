# Score Reports — scrum.org

**Workflow ID:** `CQ6zEuIEIDS67Tjf`
**Status:** Active
**Trigger:** Webhook (Slack slash-command) or manual

## Purpose

Logs into scrum.org as a trainer, downloads score report Excel files for each active course, posts results to Slack, and writes student assessment data (name, score, pass/fail, attempts) into the **Zapisy PSM 2026** Google Sheet.

## Key External Resources

| Resource | Detail |
|---|---|
| Google Sheet | **Zapisy PSM 2026** — spreadsheet ID in `Resolve Sheet Targets` node |
| Sheet tabs | Named by course dates, e.g. `"22 kwietnia"` (day + Polish genitive month, no year) |
| scrum.org login | FusionAuth OAuth flow — credentials stored in n8n DataTable |
| Slack channel | Configured in `Post to Slack` node |

## Node Map

| ID | Name | Type | Notes |
|---|---|---|---|
| sr1 | Webhook Trigger | webhook | Entry from Slack |
| sr9 | When clicking Execute workflow | manualTrigger | Manual test |
| sr2 | Respond to Slack | respondToWebhook | Immediate ACK |
| readCreds | Read Credentials | dataTable | Reads scrum.org login credentials |
| login2b | Build Login Body | code | Assembles POST body for FusionAuth login |
| login3 | Post Login Form | httpRequest | POST to FusionAuth login endpoint |
| login4 | Extract Redirect URL | code | Extracts OAuth redirect from response |
| login9 | Extract Session Cookie | code | Extracts `SSESS*` session cookie |
| sr3 | Fetch Course List | httpRequest | GET trainer course list from scrum.org; date range from webhook body or default last 30 days |
| sr4 | Parse Course List | code | Parses HTML/JSON course list |
| ifHasCourses | Has Score Reports? | if | Branches: has reports vs no reports |
| sr5 | Download Score Report | httpRequest | Downloads Excel score report per course |
| sr6 | Parse Excel | extractFromFile | Extracts rows from .xlsx binary |
| sr6b | Extract Students | code | Normalizes student rows: name, score, pass/fail, attempts |
| sr7 | Build Slack Message | code | Formats per-course result summary |
| sr8 | Post to Slack | slack | Posts to Slack |
| noReports | No Reports Message | code | Builds "no reports available" message |
| gs_resolve | Resolve Sheet Targets | code | Maps course titles → spreadsheet ID + tab name |
| gs_loop | Loop Courses | splitInBatches | Iterates over each course batch |
| gs_read2 | Read Sheet Tab | googleSheets | Reads existing rows from the target tab |
| gs_read_err | Handle Read Error | code | Catches tab-not-found / auth errors; emits structured error object |
| gs_match | Match Students | code | Merges new scores with existing rows; skips if score+attempts not improved |
| gs_skip | Skip If No Updates | if | Skips write if no rows changed |
| gs_collect | Collect Updates | code | Aggregates all changed rows |
| gs_write3 | Write Assessment Data | httpRequest | Google Sheets batchUpdate API (USER_ENTERED) |
| gs_err_slack | Notify Sheet Error | slack | Posts error to Slack if tab not found or auth failed |

### Disabled nodes (login flow remnants — kept for reference)

`Get Login Page` (login1), `Extract Login Form` (login2), `Get FusionAuth Form` (login1b), `GET Consent Page` (login5b), `Build Callback URL` (login6b), `GET OAuth Callback` (login7c), `GET Meta Refresh URL` (login8), `Build Consent URL` (login6), `Parse Meta Refresh` (login7), `GET Complete Registration` (login5)

## Flow

```
Webhook / Manual → Respond to Slack → Read Credentials → Build Login Body
  → Post Login Form → Extract Redirect URL → Extract Session Cookie
  → Fetch Course List → Parse Course List → Has Score Reports?
    ├─ YES → Download Score Report → Parse Excel → Extract Students
    │          ├─ Build Slack Message → Post to Slack
    │          └─ Resolve Sheet Targets → Loop Courses
    │               └─ Read Sheet Tab
    │                    ├─ OK → Match Students → Skip If No Updates
    │                    │         ├─ skip → Loop (next batch)
    │                    │         └─ has updates → Collect Updates → Write Assessment Data → Loop
    │                    └─ ERROR → Handle Read Error → Skip If No Updates (emit 0 rows)
    │                                                 └─ Notify Sheet Error (Slack)
    └─ NO  → No Reports Message → Post to Slack
```

## Google Sheet Write Logic

- **Operation:** `batchUpdate` via HTTP (not the Sheets node) with `valueInputOption: USER_ENTERED`
- **Column mapping** (set in `Collect Updates`): `Imię i Nazwisko`, `Egzamin`, `Wynik`, `Próby`
- **Update condition** (`Match Students`): a row is updated only if the incoming score OR attempts count is higher than what's already in the sheet. Prevents overwriting a pass with a re-attempt fail.
- **Tab naming:** Polish genitive month, e.g. `"22 kwietnia"`, `"15 maja"`. No year in tab name.

## Credentials

- n8n DataTable: scrum.org login credentials (email + password)
- Google Sheets OAuth (re-authorization needed if token expires — manual action in n8n UI)
- Slack Bot Token

## Date Range / Triggering

`Fetch Course List` queries `https://www.scrum.org/admin/courses/manage` with `node_date_range_start` and `node_date_range_end` parameters.

**Default (no argument):** last 30 days → today.

**Override from Slack:** type a date after the command:
```
/score-reports 2026-01-01
```
The `text` field Slack sends is read as `from` if it matches `YYYY-MM-DD`. The `to` date is always today.

**Override via direct HTTP POST** (e.g. curl/Postman):
```json
{ "from": "2026-01-01", "to": "2026-05-08" }
```

The manual trigger always uses the default (no body to read from).

## Known Issues / Notes

- **Google OAuth expiry:** `Read Sheet Tab` fails with "authorization grant invalid" when token is expired. Fix: re-authorize the Google Sheets credential in n8n UI. The `Notify Sheet Error` Slack node will alert when this happens.
- **Tab name mismatch:** `Resolve Sheet Targets` must generate tab names matching actual sheet tabs exactly. Format is `"D MMMM"` in Polish (e.g. `"22 kwietnia"`). Multi-day courses like `"22-24 kwietnia"` may differ — verify against actual sheet.
- **Disabled login nodes:** The FusionAuth login flow was rebuilt several times; old nodes left disabled for reference.
