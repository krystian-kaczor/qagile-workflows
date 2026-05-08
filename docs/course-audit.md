# Course Audit — Check Publishing Status

**Workflow ID:** `1MLHq9gEDw5lD8Y7`
**Status:** Active
**Trigger:** Webhook (Slack slash-command) or manual

## Purpose

Compares upcoming courses from Outlook Calendar against four publishing platforms and posts a summary to Slack showing which courses are published everywhere, partially published, or not yet published anywhere.

## Platforms checked

| Platform | How fetched |
|---|---|
| Outlook Calendar | Microsoft Outlook node — reads upcoming calendar events |
| qagile.pl | HTTP GET → HTML extraction → Code parse |
| online.qagile.pl | Two HTTP GETs (courses page + listing) → Code parse |
| scrum.org | HTTP GET → Code parse |
| BUR (rejestr.parp.gov.pl) | HTTP GET → Code parse |

## Node Map

| ID | Name | Type | Notes |
|---|---|---|---|
| n1 | Webhook Trigger | webhook | Entry point from Slack |
| 6e0adf03 | When clicking 'Execute workflow' | manualTrigger | Manual test trigger |
| n16 | Respond to Slack | respondToWebhook | Immediate 200 ACK so Slack doesn't time out |
| n2 | Get Outlook Calendar Events | microsoftOutlook | Fetches upcoming events |
| n3 | Filter Course Events | code | Normalizes course titles, filters non-course events, builds canonical course list with ABBREV_MAP |
| n4 | Fetch qagile.pl Events | httpRequest | GET qagile.pl events page |
| n5 | Extract Events HTML | html | CSS selector extraction |
| n6 | Parse qagile.pl Events | code | Normalizes scraped titles to match Outlook entries |
| n7 | Fetch online.qagile.pl Courses | httpRequest | GET online.qagile.pl courses API/page |
| 83c0bfbd | Fetch online.qagile.pl Listing | httpRequest | Second fetch for listing page |
| n8 | Parse online.qagile.pl Courses | code | Merges both fetches, normalizes |
| n10 | Fetch scrum.org Courses | httpRequest | GET scrum.org trainer course list |
| n11 | Parse scrum.org Courses | code | Normalizes |
| n12 | Fetch BUR Courses | httpRequest | GET BUR registry |
| n13 | Parse BUR Courses | code | Normalizes |
| n9 | Build Audit Report | code | Cross-references all 5 sources; produces 3 buckets: `publishedEverywhere`, `partial`, `notPublished` |
| n15 | Post to Slack | slack | Posts formatted Slack message |

## Flow

```
Webhook / Manual → Respond to Slack (ACK) → Get Outlook Events
  → Filter Course Events
  → Fetch qagile.pl → Extract HTML → Parse qagile.pl
  → Fetch online.qagile.pl → Fetch Listing → Parse online
  → Fetch scrum.org → Parse scrum.org
  → Fetch BUR → Parse BUR
  → Build Audit Report → Post to Slack
```

## Slack Output Format

```
*17 in Outlook:* 11 ✅ everywhere | 3 ⚠️ partial | 3 📋 not yet

✅ *Published everywhere (11):*
• PSM I — 15 maja ...

⚠️ *Partial (3):*
• PSPO I — 20 maja | ✅ qagile | ❌ online | ✅ scrumorg | ✅ BUR

📋 *Not yet published (3):*
• PSM II — 10 czerwca ...
```

## Credentials

- Microsoft Outlook OAuth (calendar read)
- Slack Bot Token (post to `#trainings` or configured channel)

## Key Logic — Build Audit Report (n9)

Three buckets:
- **publishedEverywhere**: all 4 platforms `status === 'found'`
- **partial**: at least one `found` AND at least one `missing`
- **notPublished**: none of the 4 platforms have `found`

Course abbreviation normalization uses `ABBREV_MAP` + `ABBREV_ALIASES` constants inside `Filter Course Events`.

## Known Issues / Notes

- BUR sometimes returns stale data; false positives possible.
- Outlook events without a recognized course abbreviation are silently dropped by `Filter Course Events`.
- The `totalOutlookCourses` count in the summary is the authoritative source of truth; all 3 buckets must sum to this number.
