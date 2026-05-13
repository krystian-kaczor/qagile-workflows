# Outlook Invoices Processing

**Workflow ID:** `onRzmQggVN6TBhFg`
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
| 7f04b2d6 | SelectJDGFolder | code | Picks correct month folder; emits `needsCreation` flag if missing |
| — | Folder Needs Creation? (JDG) | if | TRUE → create folder; FALSE → upload directly |
| — | Create Month Folder JDG | microsoftOneDrive | Creates missing month folder under JDG year folder |
| — | Fill Folder ID (JDG) | code | Merges created folder's id back onto invoice item with binary |
| 54c8c8e7 | Spółka Destination | set | Sets OneDrive root path for INC |
| 53e215dd | Get items in a folder1 | microsoftOneDrive | Lists year folders in INC root |
| 67e9fc4d | Check Current Year Folder1 | if | Checks if current year folder exists |
| 0f0aa391 | Get Months Folders INC | microsoftOneDrive | Lists month subfolders |
| a09d59a2 | SelectINCFolder | code | Picks correct month folder; emits `needsCreation` flag if missing |
| — | Folder Needs Creation? (INC) | if | TRUE → create folder; FALSE → upload directly |
| — | Create Month Folder INC | microsoftOneDrive | Creates missing month folder under INC year folder |
| — | Fill Folder ID (INC) | code | Merges created folder's id back onto invoice item with binary |
| 592be37c | Update a message | microsoftOutlook | Marks email as read / moves (JDG path) |
| 5dae6470 | Upload a file1 | microsoftOneDrive | Uploads PDF to JDG month folder (`parentId = $json.folderId`) |
| f88e959c | Append data to sheet | microsoftExcel | Appends invoice row to JDG Excel sheet |
| 4880975a | Update a message1 | microsoftOutlook | Marks email as read / moves (INC path) |
| d038c342 | Upload a file | microsoftOneDrive | Uploads PDF to INC month folder (`parentId = $json.folderId`) |
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
       │          → Get Months Folders JDG → SelectJDGFolder → Folder Needs Creation? (JDG)
       │               ├─ TRUE  → Create Month Folder JDG → Fill Folder ID (JDG) ─┐
       │               └─ FALSE ──────────────────────────────────────────────────┤
       │          → Update message → Upload PDF ($json.folderId) → Append to JDG sheet
       └─ INC → Spółka Destination → Get items in folder1 → Check Year Folder1
                  → Get Months Folders INC → SelectINCFolder → Folder Needs Creation? (INC)
                       ├─ TRUE  → Create Month Folder INC → Fill Folder ID (INC) ─┐
                       └─ FALSE ──────────────────────────────────────────────────┤
                  → Update message1 → Upload PDF ($json.folderId) → Append to INC sheet
```

## Credentials

- Microsoft Outlook OAuth (read + update messages) — Azure AD App ID: `1263101f-d064-42b0-b326-40f740b3edf1`
- Microsoft OneDrive OAuth (list + upload files)
- Microsoft Excel OAuth (append rows)
- Google Gemini API key
- **Secret rotation:** Last rotated 2026-05-13. Next expiry: ~2028-05-13. Set a calendar reminder 2 months before.

## Known Issues / Notes

- Workflow is **inactive** — must be run manually via the ▶️ Start button.
- Gemini AI extraction can fail on non-standard invoice layouts; `Validate Data` catches missing fields.
- The `htmlcsstopdf` node requires the `n8n-nodes-htmlcsstopdf` community package installed on the n8n instance.
- OneDrive folder structure assumed: `root/YYYY/MM - MonthName/`. `SelectJDGFolder` / `SelectINCFolder` handle matching.

## Changes (2026-05-13)

### Bug fixes applied

**1. Non-invoice emails processed as invoices** (`From Mailchip?` + Gemini hallucination)
- Gemini prompt updated: returns `{"isInvoice": false}` for non-invoice documents (marketing emails, subscription notices, training confirmations). Removed biased category examples (`mailchimp`, `pspo-training`) from prompt.
- `Validate Data` now short-circuits immediately on `isInvoice: false` and also asserts that `binary.currentFile.fileName` exists before passing validation. Non-invoice emails no longer produce Excel rows.

**2. Upload nodes used wrong `binaryPropertyName`** (expression returned object, not key string)
- Both `Upload a file` (INC) and `Upload a file1` (JDG) had `binaryPropertyName` set to an expression `={{ $('SelectJDGFolder').item.binary.currentFile }}` which evaluated to the binary object. Changed to literal string `currentFile`. Upload now correctly reads the renamed PDF binary.
- `fileName` expression prefix also fixed from `{{ }}` to `={{ }}` by autofix.

**3. `JDG Destination` / `Spółka Destination` ran unconditionally before validation**
- Both Set nodes were wired directly from `Any Emails?` (TRUE branch), running before any invoice validation and feeding folder listings back into `JDG or INC?` independently — causing item-index pairing bugs in `SelectJDGFolder`/`SelectINCFolder`.
- Rewired: `JDG or INC?` output 0 → `JDG Destination`, output 1 → `Spółka Destination`. `Get Months Folders JDG` → `SelectJDGFolder`, `Get Months Folders INC` → `SelectINCFolder`.
- Folder listing now only runs after a valid invoice is confirmed.

**4. `Any Emails?` and `Has attachment` IF nodes** upgraded from typeVersion 1 to typeVersion 2 conditions format (required by validator after connection rewire triggered auto-sanitization).

**5. `fileName` column still empty after fixes 1–4** (Excel append referenced OneDrive response, not pre-computed name)
- `Append data to sheet` (JDG) had `fileName` column mapped to `={{ $json.name }}` — the OneDrive Upload response's `name` field, which is absent or inconsistently shaped for certain file types.
- `Append data to sheet1` (INC) had `fileName` mapped to `={{ $('Upload a file').item.json.name }}` — same fragile dependency.
- Both `SelectJDGFolder` / `SelectINCFolder` already build a deterministic filename (`YYYYMMDD-<category>.pdf`) and store it in `$json.fileName`. That string is also passed to the Upload node's `fileName` parameter.
- Fix: both Excel nodes now read `$('SelectJDGFolder').item.json.fileName` / `$('SelectINCFolder').item.json.fileName` — guaranteed non-empty whenever a row is appended.

**6. Anthropic invoice appears twice in Excel** (multi-attachment emails produced one row per PDF)
- `Get Attachments` emits one item per attachment. Anthropic emails carry two PDFs: `Invoice-…` and `Receipt-…`. Both pass `PDF?` → both downloaded → both processed by Gemini → both produce identical output → two Excel rows, one OneDrive file.
- Fix: inserted two new nodes between `Get Attachments` and `Download attachment`:
  - `Tag Parent Message` (Set, runs per item) — copies `$('Has attachment').item.json.id` onto each attachment as `parentMessageId`, preserving `name`, `contentType`, `id` fields.
  - `Dedupe Invoice vs Receipt` (Code, runs once for all items) — groups attachments by `parentMessageId`. If a group contains both an `invoice`-named and a `receipt`-named PDF, only the invoice is kept. Single-PDF groups and groups without both patterns pass through unchanged.
- Flow: `Get Attachments → Tag Parent Message → PDF? → Dedupe Invoice vs Receipt → Download attachment`

**7. Non-PDF body-link responses dropped silently** (Scrum.org login-gated URLs)
- `Filter PDFs` FALSE branch was unconnected — items with `content-type: text/html` vanished with no trace in execution logs.
- Fix: added `Log Dropped Link` (Set node) connected to `Filter PDFs` FALSE branch. Records `droppedUrl`, `contentType`, `messageId`, and reason. No downstream connection — terminates the branch but makes the drop visible in n8n execution UI.
- Note: Scrum.org license renewal PDFs typically arrive as direct attachments; if the email has an attachment, re-marking unread and re-running processes it correctly via the attachment path. Link-only emails from login-gated portals require manual download.

**8. Missing month folder caused silent invoice drop** (Scrum.org invoice, `05 Maj` not yet created)
- `SelectJDGFolder` / `SelectINCFolder` returned `[]` when no month folder matched — invoice silently dropped.
- Fix: both Select nodes now always emit exactly one item with a `needsCreation` boolean flag and `parentFolderId` (the year folder id from `parentReference.id`).
- Added per-side creation chain: `Folder Needs Creation? (JDG/INC)` IF node → `Create Month Folder JDG/INC` (microsoftOneDrive `folder.create`, `parentFolderId` under `options`) → `Fill Folder ID (JDG/INC)` Code node that merges the new folder's `id` back onto the invoice item (restoring binary from `SelectJDGFolder`/`SelectINCFolder`).
- Both upload nodes (`Upload a file`, `Upload a file1`) updated: `parentId` now reads `$json.folderId` from the current item — correct for both existing-folder (FALSE branch) and newly-created-folder (TRUE branch) paths.
