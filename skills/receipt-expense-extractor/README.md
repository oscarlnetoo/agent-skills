# receipt-expense-extractor

Turn a pile of PDF files — where each PDF holds **many receipts** (often one per page) —
into a single, clean expense spreadsheet.

For every receipt the skill records one row:

| Issue date | Description | Total value | Source filename |

## Why it exists

Expense paperwork for reimbursement, tax, or legal/rental processes usually means dozens
of receipts scanned into a few big PDFs — printed notes, faded thermal slips, even
handwritten ones. Transcribing them by hand is slow and error-prone. This skill does the
extraction, but keeps a human in the loop so the numbers can be trusted.

## How it works

- **Reads receipts via OCR text, never by downloading the raw PDF bytes.** For Google
  Drive it uses the connector's text-extraction tool; for local PDFs it uses standard OCR.
  This keeps even multi-megabyte scans cheap to process.
- **Works in small, reviewable batches** — five receipts at a time — and waits for your
  approval before writing anything to the sheet.
- **Is honest about uncertainty.** Printed fiscal notes (NF-e/DANFE/NFC-e) are high
  confidence; handwritten totals get flagged for you to confirm rather than guessed.
- **Avoids common traps**: duplicate page copies of the same purchase, and quotes /
  "não comprova pagamento" documents that aren't actually paid receipts.
- **Produces an `.xlsx`** with locale-aware dates and a currency-formatted total column,
  and can upload a copy back to Google Drive.

It starts by asking a few quick questions (where the PDFs are, output format and
filename, description language, column layout) before doing any work.

## Install

Download [`receipt-expense-extractor.skill`](receipt-expense-extractor.skill) and install
it in a compatible agent environment (in Claude Cowork mode: open the file → **Save
skill**). It then triggers on requests like *"organize my receipt PDFs into a
spreadsheet"* or *"extract the totals and dates from these nota fiscal PDFs."*

## Contents

```
receipt-expense-extractor/
├── SKILL.md                       # the agent instructions
├── scripts/
│   └── expense_sheet.py           # helper: init / append / sort / total for the .xlsx
└── receipt-expense-extractor.skill
```

### The helper script

`scripts/expense_sheet.py` handles the spreadsheet mechanics so the workflow doesn't
re-implement them each run:

```bash
python scripts/expense_sheet.py init   --path Expenses.xlsx
python scripts/expense_sheet.py append --path Expenses.xlsx --rows '[["02/02/2021","Item (Store)",194.90,"file.pdf"]]'
python scripts/expense_sheet.py sort   --path Expenses.xlsx
python scripts/expense_sheet.py total  --path Expenses.xlsx
```

Requires `openpyxl`. Headers, currency format, and date format are all configurable via
flags, so the same script works for other languages or currencies.
