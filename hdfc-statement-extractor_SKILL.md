---
name: hdfc-statement-extractor
description: >
  Extracts domestic and international transaction data from HDFC Infinia Credit Card
  PDF statements and produces a formatted .xlsx workbook matching the house style
  (HDFC blue header, alternating row shading, green credit rows, totals footer,
  separate Domestic and International sheets). Use this skill whenever the user
  uploads an HDFC credit card statement PDF and asks to extract transactions, convert
  to Excel, or produce a transaction register — even if they phrase it as "get the
  data out", "put it in a spreadsheet", or "same format as last time".
---

# HDFC Statement → Excel Extractor

## Purpose

Read an HDFC Infinia Credit Card statement PDF (any billing period) and output a
clean `.xlsx` workbook with two sheets — **Domestic** and **International** — formatted
to match the established house style.

---

## Step 0 — Read required skills first

Before writing any code, call `view` on:
- `/mnt/skills/public/xlsx/SKILL.md` — Excel construction rules (openpyxl patterns, recalc script)
- `/mnt/skills/public/pdf-reading/SKILL.md` — PDF extraction strategy (if statement not already in context)

---

## Step 1 — Ingest the PDF

The statement is usually already visible in the conversation context (rendered as
images/text by the document block). If it is, read all pages from context — no tool
call needed.

If the content is NOT in context (only a file path is mentioned), use the pdf-reading
skill to extract text page by page.

**What to look for on each page:**

| Field | Notes |
|---|---|
| Billing Period | "08 Mar, 2026 – 07 Apr, 2026" — use for sheet title |
| Cardholder sub-sections | Bold lines: "MR PRAMOD", "Priyal Maheshwari", "SHILPA MAHESHWARI" |
| DATE & TIME | Two separate tokens, e.g. "07/03/2026\| 05:32" |
| TRANSACTION DESCRIPTION | Full string as printed |
| REWARDS | "+495", "-20", blank for no reward |
| AMOUNT | Numeric; note the ₹ symbol — strip it |
| Credit indicator | Rows shown with green text / "+" prefix on amount in statement = CR |

**Identifying credit/refund rows:**
- Amount shown as `+ ₹X,XXX.XX` (with leading `+`) → CR
- Description includes "HDFC Bank Ltd" payment credit → CR
- Rewards column shows negative (e.g. `-65`) → CR

---

## Step 2 — Parse and structure data

Build two Python lists:

```python
# (date, time, card_holder, description, rewards_str, amount_float, cr_dr)
domestic     = [ ... ]
international = [ ... ]
```

Rules:
- `card_holder`: Preserve casing from statement (`MR PRAMOD`, `PRIYAL MAHESHWARI`,
  `SHILPA MAHESHWARI`). Use UPPER for add-on cardholders.
- `rewards_str`: Keep the `+` prefix if present; empty string `""` if blank.
- `amount_float`: Pure `float` — no ₹, no commas.
- `cr_dr`: `"CR"` for credits/refunds, `""` for normal debits.

---

## Step 3 — Build the workbook

Use **openpyxl** (not pandas) for full formatting control.

### House style constants

```python
HDFC_BLUE  = "FF003087"   # Header fill + title font
WHITE      = "FFFFFFFF"
LIGHT_GREY = "FFF5F5F5"   # Odd data rows
CR_GREEN   = "FF006400"   # Font for CR rows
CR_ROW_BG  = "FFE8F5E9"   # Fill for CR rows
ORANGE_BG  = "FFFFE0B2"   # Totals footer fill
```

### Column layout (both sheets)

| Col | Header | Width |
|-----|--------|-------|
| A | DATE | 14 |
| B | TIME | 10 |
| C | CARD HOLDER | 22 |
| D | TRANSACTION DESCRIPTION | 60 |
| E | REWARDS | 10 |
| F | AMOUNT (₹) | 18 |
| G | CR/DR | 8 |

### Row rules

- **Row 1**: Merged A1:G1, title `"HDFC Infinia Credit Card  —  Domestic Transactions  (Mmm YYYY)"`, HDFC blue bold font, height 22.
- **Row 2**: Header row — white bold font on HDFC blue fill, height 18.
- **Rows 3+**: Alternate `LIGHT_GREY` / `WHITE` fill; CR rows use `CR_ROW_BG` fill and `CR_GREEN` font regardless of alternation.
- **Footer**: Two rows below last data row, merged D:E, orange fill, text:
  `"Total Debits: ₹{X:,.2f}     |     Total Credits: ₹{Y:,.2f}"`

### Alignment

- Columns A, B, E, G: centre
- Column D: left
- Column C: left
- Column F: right

### Font

Arial, size 9 for data rows; size 10 bold white for headers; size 11 bold HDFC blue for title.

---

## Step 4 — Recalculate and verify

```bash
python /mnt/skills/public/xlsx/scripts/recalc.py /home/claude/<output>.xlsx 30
```

Confirm `"status": "success"` and `"total_errors": 0` before proceeding.

---

## Step 5 — Deliver

```python
import shutil
shutil.copy("/home/claude/<output>.xlsx", "/mnt/user-data/outputs/<output>.xlsx")
```

Call `present_files` with the output path.

---

## Naming convention

Output filename: `HDFC_Infinia_<Period>_Transactions.xlsx`

Examples:
- `HDFC_Infinia_Mar_Apr2026_Transactions.xlsx`
- `HDFC_Infinia_Jan2026_Transactions.xlsx`

Derive the period label from the statement's billing period dates.

---

## Edge cases

| Situation | Handling |
|---|---|
| Multiple add-on cardholders | Each has their own sub-section in the PDF; preserve all with correct name |
| DCC / IGST fee rows | Include as normal debit rows; REWARDS field will be blank |
| Refund rows (amount in green with `+`) | Mark as CR; use absolute value for AMOUNT |
| Payment credit (e.g. HDFC Bank Ltd) | Mark as CR with `"+"` in REWARDS column |
| Blank rewards column | Use empty string `""` |
| Statement already in another XLSX | Read that file to confirm column structure before writing; match exactly |

---

## Reference: Jan 2026 baseline format

If the user uploads a prior-period XLSX alongside the new PDF (e.g. "same format as
last time"), load it with openpyxl to confirm column widths, fonts, and fill colours
before building the new file. Use `load_workbook(..., data_only=True)` to inspect
values; do not overwrite it.
