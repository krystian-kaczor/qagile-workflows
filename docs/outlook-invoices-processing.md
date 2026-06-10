# Outlook Invoices Processing

**Workflow ID:** `onRzmQggVN6TBhFg`
**Status:** Inactive (manual trigger only)
**Trigger:** Manual (▶️ Start button)

## Purpose

Reads unread emails from **two sources** — designated Outlook "Faktury" (invoices) folders **and** a Gmail account (vendors that bill to Gmail: ElevenLabs, HeyGen, fal.ai, etc.) — extracts PDF invoices (from attachments or links in email body), uses Google Gemini AI to parse invoice data, validates the data, then files the PDF into the correct OneDrive folder (JDG or Spółka) and appends a row to the corresponding Excel tracking sheet. Processed Outlook emails are marked read; processed Gmail messages are marked read via the Gmail node.

## Key External Resources

| Resource | Detail |
|---|---|
| Outlook folders | "Faktury" folders — unread emails fetched from these |
| Gmail account | Unread invoice emails fetched via Gmail node. Invoices arrive **from Stripe** (`invoice+statements+...@stripe.com`, subject "Your receipt from <vendor>") for ElevenLabs / HeyGen / fal.ai, as **PDF attachments** (`Invoice-*.pdf` + `Receipt-*.pdf`). |
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
| 4880975a | Update a message1 | microsoftOutlook | Marks email as read / moves (INC path) — **Outlook items only** (FALSE branch of Mark Read Source? (INC)) |
| d038c342 | Upload a file | microsoftOneDrive | Uploads PDF to INC month folder (`parentId = $json.folderId`) |
| 2f9f5852 | Append data to sheet1 | microsoftExcel | Appends invoice row to INC Excel sheet |

### Gmail source nodes (added 2026-05-22)

| ID | Name | Type | Notes |
|---|---|---|---|
| f95b1446 | Fetch Unread Gmail Invoices | gmail | `getAll`, `simple:false`, `downloadAttachments:true`, `readStatus:unread`. `q` = `label:QAgile-Invoices` — fetches only unread emails carrying the Gmail label **QAgile/Invoices** (id `Label_8835956035448197511`; Gmail search syntax converts `/` → `-`). This label is applied to Stripe vendor receipts (ElevenLabs/HeyGen/Qwiklabs/fal.ai) **and** Google Ads billing (`payments-noreply@google.com`) — a broader net than the previous `from:stripe.com` query. Branches off ▶️ Start in parallel with Outlook fetch. Credential `gmailOAuth2` = `oGattM1WQITzTJAj` ("Gmail account"). |
| 6e4a9ceb | Gmail Has Attachment? | if | Checks for `attachment_N` binary keys (Gmail has no `hasAttachments` field) |
| 2f41326d | Gmail Pick PDF Attachment | code | Picks PDF attachment, dedupes invoice-vs-receipt, emits `binary.data` + `messageID` + `source:'gmail'` |
| e297ad85 | Gmail Get Links | code | Extracts URLs from Gmail `html`/`text` body |
| 19045e9c | Gmail Split URLs | splitOut | One item per URL |
| 5bdbcc2d | Gmail Fetch URL | httpRequest | Downloads PDF from URL (fullResponse, neverError) |
| 55731631 | Gmail Filter PDFs | if | Content-type contains `pdf` |
| 7b56bfa9 | Gmail Prepare Body PDF | code | Normalizes fetched binary → `binary.data` + `messageID` + `source:'gmail'` |
| 8fa635b9 | Gmail Log Dropped | set | Terminal — non-PDF body links (mirrors Log Dropped Link) |
| b48756bd | Prepare Mailchimp | code | Inserted between Convert HTML to PDF and StoreFile; sets `messageID` (from Get message body) + `source:'outlook'` |
| 2b4eff98 | Mark Read Source? (JDG) | if | `$('SelectJDGFolder').item.json.source == 'gmail'` → Gmail Mark Read JDG; else → Update a message. **Reads `source` cross-node from `SelectJDGFolder`, NOT `$json`** — input is the OneDrive Upload response, which drops the custom `source` field. |
| d7f6e2e7 | Gmail Mark Read JDG | gmail | `markAsRead`, `messageId = $('SelectJDGFolder').item.json.messageID` |
| 6c4333fc | Mark Read Source? (INC) | if | `$('SelectINCFolder').item.json.source == 'gmail'` → Gmail Mark Read INC; else → Update a message1. **Reads `source` cross-node from `SelectINCFolder`, NOT `$json`** (same OneDrive-response reason). |
| 792c3b0a | Gmail Mark Read INC | gmail | `markAsRead`, `messageId = $('SelectINCFolder').item.json.messageID` |

### Source flag (`source`)

A `source` field (`'outlook'` / `'gmail'`) is set on every item at its prep node and survives end-to-end so the terminal mark-as-read step routes correctly:
- Set in: `Prepare Output1`, `Prepare Output2`, `Prepare Mailchimp` (outlook); `Gmail Pick PDF Attachment`, `Gmail Prepare Body PDF` (gmail).
- `StoreFile` reads `messageID`/`source` from `$json` (was hardcoded to `$('Has attachment').item.json.id`).
- `Validate Data` copies `source` (and `messageId` from `storeItem.json.messageID`) into its output.
- `SelectJDGFolder`/`SelectINCFolder` carry `source` into `baseJson`.
- `Mark Read Source? (JDG/INC)` branch on it — reading `$('SelectJDGFolder'/'SelectINCFolder').item.json.source`, **not** `$json.source`. Their direct input is the OneDrive `Upload a file` response, which does not preserve the custom `source` field, so a `$json.source` test always evaluates `undefined` (FALSE).

Both `StoreFile` (5 inbound) and the two `Append data to sheet` nodes (2 inbound each) merge multiple connections — n8n appends inbound streams; each invoice item is single-source, so no double rows.

## Flow

```
▶️ Start ─┬─ Fetch Unread Emails (Outlook) → Any Emails?
          │     └─ YES → Has attachment?
          │          ├─ YES → Get Attachments → Tag Parent → PDF? → Dedupe → Download attachment → Prepare Output1 ─┐
          │          └─ NO  → Get message body → From Mailchimp?                                                    │
          │                     ├─ YES → Convert HTML to PDF → Prepare Mailchimp ───────────────────────────────────┤
          │                     └─ NO  → Get links → Split URLs → Fetch URL → Filter PDFs                            │
          │                                 → Prepare Output → If → Prepare Output2 ─────────────────────────────────┤
          │                                                                                                          │
          └─ Fetch Unread Gmail Invoices → Gmail Has Attachment?                                                     │
                ├─ TRUE → Gmail Pick PDF Attachment ───────────────────────────────────────────────────────────────┤
                └─ FALSE → Gmail Get Links → Gmail Split URLs → Gmail Fetch URL → Gmail Filter PDFs                   │
                              ├─ TRUE → Gmail Prepare Body PDF ───────────────────────────────────────────────────────┤
                              └─ FALSE → Gmail Log Dropped (terminal)                                                  │
                                                                                                                      ▼
                                                                                                                  StoreFile
StoreFile → Analyze document (Gemini AI) → Validate Data → Invoice Data Valid?
  └─ VALID → JDG or INC?
       ├─ JDG → JDG Destination → Get items in folder → Check Year Folder
       │          → Get Months Folders JDG → SelectJDGFolder → Folder Needs Creation? (JDG)
       │               ├─ TRUE  → Create Month Folder JDG → Fill Folder ID (JDG) ─┐
       │               └─ FALSE ──────────────────────────────────────────────────┤
       │          → Upload PDF ($json.folderId) → Mark Read Source? (JDG)
       │               ├─ gmail   → Gmail Mark Read JDG ─┐
       │               └─ outlook → Update a message ────┤→ Append to JDG sheet
       └─ INC → Spółka Destination → Get items in folder1 → Check Year Folder1
                  → Get Months Folders INC → SelectINCFolder → Folder Needs Creation? (INC)
                       ├─ TRUE  → Create Month Folder INC → Fill Folder ID (INC) ─┐
                       └─ FALSE ──────────────────────────────────────────────────┤
                  → Upload PDF ($json.folderId) → Mark Read Source? (INC)
                       ├─ gmail   → Gmail Mark Read INC ─┐
                       └─ outlook → Update a message1 ───┤→ Append to INC sheet
```

## Credentials

- Microsoft Outlook OAuth (read + update messages) — Azure AD App ID: `1263101f-d064-42b0-b326-40f740b3edf1`
- Microsoft OneDrive OAuth (list + upload files)
- Microsoft Excel OAuth (append rows)
- Google Gemini API key
- **Gmail OAuth2 (`gmailOAuth2`)** — credential `oGattM1WQITzTJAj` ("Gmail account"), attached to the 3 Gmail nodes (Fetch Unread Gmail Invoices, Gmail Mark Read JDG, Gmail Mark Read INC). Created & connected in the n8n UI on 2026-05-28.
- **Secret rotation:** Last rotated 2026-05-13. Next expiry: ~2028-05-13. Set a calendar reminder 2 months before.

## Known Issues / Notes

- Workflow is **inactive** — must be run manually via the ▶️ Start button.
- Gemini AI extraction can fail on non-standard invoice layouts; `Validate Data` catches missing fields.
- The `htmlcsstopdf` node requires the `n8n-nodes-htmlcsstopdf` community package installed on the n8n instance.
- OneDrive folder structure assumed: `root/YYYY/MM - MonthName/`. `SelectJDGFolder` / `SelectINCFolder` handle matching.
- **Gmail invoices come from Stripe, not vendor domains.** ElevenLabs/HeyGen/fal.ai bill through Stripe; the email is `invoice+statements+acct_*@stripe.com`, subject "Your receipt from <vendor>", with two PDF attachments (`Invoice-*.pdf` + `Receipt-*.pdf`). The Gmail branch takes the **attachment path**; `Gmail Pick PDF Attachment` keeps the `Invoice-*` PDF when both are present.
- **Gmail attachment binary keys** are named `attachment_0…N` (n8n default `dataPropertyAttachmentsPrefixName`). `Gmail Has Attachment?` and `Gmail Pick PDF Attachment` rely on the `attachment_` prefix.
- **`Gmail Get Links` / body-link path** is a fallback only; its regex now keeps only PDF-looking URLs (`.pdf`, `/pdf`, `files.stripe.com`, `receipts/invoices`) to avoid the earlier bug where 544 tracking/stylesheet URLs were fetched and timed out. `Gmail Fetch URL` has `retryOnFail` + `onError: continueRegularOutput` so a single dead URL can't fail the whole run.

## Changes (2026-06-10)

### Gmail fetch now filters by label `QAgile/Invoices`

- Changed `Fetch Unread Gmail Invoices` `q` from the vendor/Stripe query to `label:QAgile-Invoices`. The node now pulls every **unread** email tagged with the Gmail label **QAgile/Invoices** (id `Label_8835956035448197511`), regardless of sender.
- Gmail search syntax represents the nested label `QAgile/Invoices` as `QAgile-Invoices` (slash → hyphen). Verified: the query resolves to 201 labeled threads (Stripe receipts for ElevenLabs/HeyGen/Qwiklabs + Google Ads billing PDFs).
- Broader coverage than before — Google Ads invoices (`payments-noreply@google.com`) are now caught, which the old `from:stripe.com` filter missed. Any new vendor just needs the label applied (manually or via a Gmail filter rule).

### Fixed "Id is malformed" in Outlook `Update a message` (Gmail items mis-routed)

Every invoice — Gmail-sourced included — was being routed into the Outlook `Update a message` / `Update a message1` nodes, which then threw **"Id is malformed"** on a Gmail message id.

- **Root cause:** `Mark Read Source? (JDG)` (2b4eff98) and `Mark Read Source? (INC)` (6c4333fc) tested `={{ $json.source }}`. Their input is the OneDrive `Upload a file` response, which carries only OneDrive file metadata — **not** the custom `source` field. So `$json.source` was always `undefined`, `source == 'gmail'` was always FALSE, and all items fell through to the Outlook branch.
- **Fix:** changed each IF condition `leftValue` to read `source` cross-node from the `Select*Folder` node — the same node the mark-read nodes already trust for `messageID`. Used `.first()` (not `.item`) because the OneDrive Upload boundary makes `.item` pairing unreliable here, and each JDG/INC branch carries exactly one invoice per run:
  - JDG: `={{ $('SelectJDGFolder').first().json.source }}`
  - INC: `={{ $('SelectINCFolder').first().json.source }}`
- One condition per IF node; wiring/operator/branches unchanged. Gmail items now route TRUE → `Gmail Mark Read JDG/INC`; Outlook items route FALSE → `Update a message`/`Update a message1`.
- **MCP gotcha:** applying this via `n8n_update_partial_workflow` with a dotted path `parameters.conditions.conditions[0].leftValue` does **not** index into the array — it silently creates a junk sibling key `"conditions[0]"` and leaves the real condition unchanged. The first two attempts had no effect for this reason. Fix: replace the whole `parameters.conditions` object in one `updateNode`.
- **Verified:** run 2327 (2026-06-10) processed a mixed Outlook+Gmail batch — Gmail HeyGen invoice → `Gmail Mark Read JDG` (UNREAD label cleared); Outlook g43office invoice → `Update a message` (`isRead: true`, no malformed-id error). `status: success`.
- **Batch caveat:** `.first()` is correct only while each JDG/INC branch carries a single invoice per run. A fully batch-safe fix would carry `source`/`messageID` as real `$json` fields through the OneDrive Upload nodes.

## Changes (2026-05-22)

### Added Gmail as a second invoice source

- Goal: process invoice emails arriving on Krystian's Gmail account (ElevenLabs, HeyGen, fal.ai, others) alongside the existing Outlook source.
- **Strategy:** parallel Gmail sub-pipeline (dedicated nodes) merging into the existing `StoreFile` node. Everything downstream of `StoreFile` is source-agnostic except the two terminal mark-as-read nodes, which now branch on a `source` flag.
- **Added:** 9 Gmail branch nodes (fetch → attachment/body-link → StoreFile), `Prepare Mailchimp` (so the Mailchimp path also carries `messageID`/`source`), and 2 `Mark Read Source?` IF nodes + 2 `Gmail Mark Read` nodes. See the Gmail node-map table and the `source` flag section above.
- **Modified:** `StoreFile` (reads `messageID`/`source` from `$json`, adds `source`), `Prepare Output1`/`Prepare Output2` (emit `messageID` + `source:'outlook'`), `Validate Data` (carries `source`), `SelectJDGFolder`/`SelectINCFolder` (carry `source`).
- **Open:** the `gmailOAuth2` credential is not yet created on the instance — attach it manually before the Gmail nodes can run (see Credentials).

## Changes (2026-05-28)

### Fixed Gmail attachment processing (first live run)

First run pulled 16 ElevenLabs **marketing newsletters** (CATEGORY_PROMOTIONS), none with attachments, and the body-link path exploded to 544 URLs → `Gmail Fetch URL` timed out (`ETIMEDOUT` on a SendGrid tracking pixel). Root causes + fixes:

- **Wrong `q` query.** Original `from:elevenlabs.io OR from:heygen.com OR from:fal.ai` never matches the invoices, because the vendors bill **through Stripe** — the email is from `invoice+statements+acct_*@stripe.com` with subject "Your receipt from <vendor>", carrying `Invoice-*.pdf` + `Receipt-*.pdf` attachments. New `q`: `from:stripe.com subject:receipt (ElevenLabs OR "Eleven Labs" OR HeyGen OR "fal.ai" OR fal)`. These now flow the **attachment path** (`Gmail Has Attachment?` TRUE → `Gmail Pick PDF Attachment`), which already picks the `Invoice-*` PDF.
- **Runaway link extraction.** `Gmail Get Links` matched every URL in the HTML body (544). Now filters to PDF-looking URLs only (`.pdf`, `/pdf`, `files.stripe.com`, `receipts/invoices`) + dedupes. This path is now only a fallback.
- **No HTTP error handling.** `Gmail Fetch URL` got `retryOnFail` (2 tries) + `onError: continueRegularOutput` so a single unreachable URL no longer kills the whole execution.
- **Credential created.** `gmailOAuth2` `oGattM1WQITzTJAj` ("Gmail account") connected in the UI and attached to all 3 Gmail nodes.

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
