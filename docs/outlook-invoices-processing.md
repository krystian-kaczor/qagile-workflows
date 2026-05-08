# Outlook Invoices Processing

**Workflow ID:** `aGu19fNOGsMOi4BZ`
**Status:** Inactive (manual trigger only)
**Trigger:** Manual (▶️ Start button)

## Purpose

Reads unread emails from designated Outlook "Faktury" (invoices) folders, extracts PDF invoices (from attachments or links in email body), uses Google Gemini AI to parse invoice data, validates the data, then files the PDF into the correct OneDrive folder (JDG or Spółka) and appends a row to the corresponding Excel tracking sheet.

## Key External Resources

| Resource | Detail |
|---|---|
| Outlook folders | "Faktury" folders — unread emails fetched from these |
| OneDrive (JDG) | JDG invoice folder — year/month subfolder structure |
| OneDrive (Spółka/INC) | Company invoice folder — year/month subfolder structure |
| Excel tracking (JDG) | Spreadsheet for JDG invoices |
| Excel tracking (INC) | Spreadsheet for Spółka invoices |
| Google Gemini | `@n8n/n8n-nodes-langchain.googleGemini` — AI invoice data extraction |

## Node Map

| ID | Name | Type | Notes |
|---|---|---|---|
| 97863b02 | ▶️ Start | manualTrigger | Entry point |
| ee393d8d | Fetch Unread Emails From Faktury Folders | microsoftOutlook | Reads unread from Faktury folder(s) |
| e3229774 | Any Emails? | if | Stop if inbox empty |
| 060ec4a4 | Has attachment | if | Branch: attachment vs. body link |
| e3c4c94d | Get Attachments | microsoftOutlook | Fetches attachment list |
| 1d05782a | PDF? | if | Filters to PDF attachments only |
| 1bf8992d | Download attachment | microsoftOutlook | Downloads the PDF binary |
| b2a13229 | Prepare Output1 | code | Normalizes attachment binary for downstream |
| 03e9c20d | Get message body | microsoftOutlook | Fetches full HTML body |
| 06168372 | From Mailchip? | if | Detects Mailchimp-style email (has PDF link) |
| 93e03a92 | Convert HTML to PDF | htmlcsstopdf | Converts Mailchimp HTML body to PDF |
| 38f3f317 | Get links from message body | code | Extracts PDF URLs from HTML |
| 54cbb6f2 | Split URLs | splitOut | One item per URL |
| 71897beb | Fetch URL1 | httpRequest | Downloads PDF from URL |
| 004aba03 | Filter PDFs | if | Ensures content-type is PDF |
| b3087631 | Prepare Output | code | Normalizes fetched PDF |
| 9a06e439 | If | if | Additional filter |
| 0a84d8ef | Prepare Output2 | code | Final normalization |
| bcf3d15b | StoreFile | set | Stores PDF binary + metadata for AI analysis |
| aa09fae1 | Analyze document | googleGemini | AI extracts: invoice number, date, vendor, amount, VAT, entity (JDG/INC) |
| 2dfa06be | Validate Data | code | Validates required fields present and correct format |
| e124fb64 | Invoice Data Valid? | if | Branches valid vs. invalid |
| 59b75192 | JDG or INC? | switch | Routes to JDG or INC path |
| 5afeaf9d | JDG Destination | set | Sets OneDrive root path for JDG |
| a7a2eeee | Get items in a folder | microsoftOneDrive | Lists year folders in JDG root |
| 728b0762 | Check Current Year Folder | if | Checks if current year folder exists |
| acfe9a4e | Get Months Folders JDG | microsoftOneDrive | Lists month subfolders |
| 7f04b2d6 | SelectJDGFolder | code | Picks correct month folder; creates if missing |
| 54c8c8e7 | Spółka Destination | set | Sets OneDrive root path for INC |
| 53e215dd | Get items in a folder1 | microsoftOneDrive | Lists year folders in INC root |
| 67e9fc4d | Check Current Year Folder1 | if | Checks if current year folder exists |
| 0f0aa391 | Get Months Folders INC | microsoftOneDrive | Lists month subfolders |
| a09d59a2 | SelectINCFolder | code | Picks correct month folder |
| 592be37c | Update a message | microsoftOutlook | Marks email as read / moves (JDG path) |
| 5dae6470 | Upload a file1 | microsoftOneDrive | Uploads PDF to JDG month folder |
| f88e959c | Append data to sheet | microsoftExcel | Appends invoice row to JDG Excel sheet |
| 4880975a | Update a message1 | microsoftOutlook | Marks email as read / moves (INC path) |
| d038c342 | Upload a file | microsoftOneDrive | Uploads PDF to INC month folder |
| 2f9f5852 | Append data to sheet1 | microsoftExcel | Appends invoice row to INC Excel sheet |

## Flow

```
▶️ Start → Fetch Unread Emails → Any Emails?
  └─ YES → Has attachment?
       ├─ YES → Get Attachments → PDF? → Download attachment → Prepare Output1 → StoreFile
       └─ NO  → Get message body → From Mailchimp?
                  ├─ YES → Convert HTML to PDF → StoreFile
                  └─ NO  → Get links → Split URLs → Fetch URL → Filter PDFs
                              → Prepare Output → If → Prepare Output2 → StoreFile

StoreFile → Analyze document (Gemini AI) → Validate Data → Invoice Data Valid?
  └─ VALID → JDG or INC?
       ├─ JDG → JDG Destination → Get items in folder → Check Year Folder
       │          → Get Months Folders → SelectJDGFolder
       │          → Update message → Upload PDF → Append to JDG sheet
       └─ INC → Spółka Destination → Get items in folder1 → Check Year Folder1
                  → Get Months Folders INC → SelectINCFolder
                  → Update message1 → Upload PDF → Append to INC sheet
```

## Credentials

- Microsoft Outlook OAuth (read + update messages)
- Microsoft OneDrive OAuth (list + upload files)
- Microsoft Excel OAuth (append rows)
- Google Gemini API key

## Known Issues / Notes

- Workflow is **inactive** — must be run manually via the ▶️ Start button.
- Gemini AI extraction can fail on non-standard invoice layouts; `Validate Data` catches missing fields.
- The `htmlcsstopdf` node requires the `n8n-nodes-htmlcsstopdf` community package installed on the n8n instance.
- OneDrive folder structure assumed: `root/YYYY/MM - MonthName/`. `SelectJDGFolder` / `SelectINCFolder` handle matching.
