---
name: receipt-expense-extractor
description: >-
  Build an expense spreadsheet from PDF files that each contain MULTIPLE receipts
  (typically one receipt per page). Use this whenever someone wants to turn a pile
  of receipt/invoice PDFs into an organized expense sheet — phrases like "create an
  expenses sheet from these receipts", "extract the totals and dates from my receipt
  PDFs", "organize my nota fiscal / invoice PDFs into a spreadsheet", "I have PDFs
  with lots of receipts and need them in Excel", or any reimbursement / tax / rental
  (aluguel) expense-tracking task. Trigger even if the user doesn't say "skill" or
  name a column layout — if the job is reading receipts out of PDFs and tabulating
  date + description + amount, this is the skill. Handles scanned and handwritten
  receipts via OCR, processes them in small reviewable batches, and produces an
  .xlsx (optionally uploaded back to Google Drive).
---

# Receipt Expense Extractor

## What this does

You are turning one or more PDF files — each holding several receipts (usually one
per page) — into a single expense spreadsheet. For every receipt you extract three
facts and record one row:

1. **Issue date** (data de emissão)
2. **Short description** of the expense
3. **Total value** (valor total)

Plus a 4th column: the **source filename**, so every row is traceable.

The output is an `.xlsx` with these four columns, in this order:

| Issue date | Description | Total value | Filename |

These receipts are real money records (reimbursement, tax, legal/rental processes),
so accuracy matters more than speed. The whole workflow is built around letting the
user **verify before anything is committed**, because OCR on scanned and handwritten
receipts is imperfect and a wrong total is worse than a slow one.

## The golden rule: extract via OCR text, do NOT download the PDF bytes

Receipt PDFs are usually scans or photos, often several MB each. The single most
important technique here: **get the text through a server-side OCR/extraction tool,
never by downloading the raw file into context.**

- Google Drive: use the Drive connector's `read_file_content` tool (it returns an
  OCR/natural-language text representation). Find files with `search_files`.
- Local / connected folder PDFs: extract text with the `pdf` skill or a tool like
  `pdftotext` / OCR (`ocrmypdf`, `pytesseract`) in the sandbox.

Why this matters: downloading a multi-MB PDF as base64 dumps hundreds of thousands of
tokens into context and will simply fail on the larger files. The OCR text is tiny by
comparison and contains everything you need. If a tool offers both "download bytes"
and "read content as text," always reach for the text one.

OCR text has no clean page breaks and garbles handwriting — that's expected. You
segment receipts by recognizing repeated markers in the text (a new store letterhead,
a new "DANFE"/"NF-e"/"CNPJ" block, a new "TOTAL", a new document header). Don't trust
that the model "sees" tidy pages; read the whole blob and split it yourself.

## Language of the description

Write each description in the **language the user asked for**. If they don't specify,
match the receipts / their request, defaulting to **Brazilian Portuguese** for
Brazilian notas fiscais. Keep descriptions short and concrete — the main item(s)
plus the store in parentheses, e.g. *"Telha São Francisco Romana (190 un.) — Vischi
Madeiras"*. The goal is something the user can scan in a list, not a full line-item dump.

## Workflow — small steps, user confirms each one

Do NOT process everything silently and dump a finished file. Move in small steps and
let the user check each one. This is what makes the skill trustworthy on fuzzy OCR.

### Step 0 — Ask what you need before starting

The trigger for this skill is often vague ("organize my receipt PDFs into a
spreadsheet"). Don't guess and don't start reading files yet. First gather the few
facts that shape the whole job, then confirm before touching anything. In Cowork,
ask these with the multiple-choice question tool so it's one quick interaction; in a
plain chat, just ask them together. Keep it to what you genuinely can't infer:

1. **Where are the PDFs?** A Google Drive folder (name or link) or a local/connected
   folder. If Drive, you'll find them with `search_files`; if local, list the folder.
2. **Output format & destination.** An `.xlsx` saved locally, and/or a copy uploaded
   back to Google Drive? What filename?
3. **Description language.** Default to the receipts' language / Brazilian Portuguese
   unless they say otherwise.
4. **Column layout.** The default is date / description / total / filename — confirm
   that order and those header labels work, or adjust.

Also briefly tell them how you'll proceed: file by file, 5 receipts at a time, pausing
for their approval before writing each batch — so the review cadence is no surprise.
Once they answer, move to Step 1.

### Step 1 — Create the sheet first (headers only)

Before reading any receipt, create the `.xlsx` with the filename and the four header
cells, and show it to the user. This confirms the layout and filename up front.
Use `scripts/expense_sheet.py init`. Then pause for the user to approve.

### Step 2 — Find and list the PDFs

Locate the folder and list the PDF files to process (Drive `search_files` by
`parentId`, or list the connected folder). Show the user the list and the order
you'll process them in.

### Step 3 — Extract one file at a time, in batches of 5 receipts

For each PDF: pull its OCR text, segment it into individual receipts, and work through
them **5 at a time**. For each receipt in the batch capture date, description, total.

As you read, watch for these recurring real-world traps:

- **Duplicate copies.** Multi-page scans often include two copies of the same pedido
  (customer copy + store copy). Deduplicate — one purchase, one row.
- **Quotes and non-payment documents vs. real receipts.** A page is not necessarily
  proof of a paid expense. Watch for *"ORÇAMENTO"* (quote/estimate), but also for sales
  orders and pre-receipts that explicitly disclaim payment — e.g. *"PEDIDO DE VENDA"*,
  *"DOCUMENTO AUXILIAR DE VENDA"*, *"não é documento fiscal"*, *"não comprova pagamento"*,
  or *"Pagamento: Nenhum"*. Any of these means "maybe not actually paid." Flag it and ask
  whether to include or exclude rather than silently adding it. (A printed NF-e/DANFE/NFC-e
  or a bank payment confirmation, by contrast, is a real receipt.)
- **Missing or illegible totals.** Handwritten receipts may have an unreadable total
  or none at all. Never guess a money value — flag it and ask the user.
- **Confidence.** Mark each receipt's confidence (high for printed NF-e/DANFE, low for
  handwritten) so the user knows where to look closely.

### Step 4 — Present the batch and wait for approval

Show the batch as a small table: number, date, description, total, and a confidence
note. Call out every uncertain field explicitly (e.g. "the day is illegible — 30/03 or
30/05?", "total ambiguous between R$ 1.097 and R$ 1.197"). Include a link to the
original file so the user can check. Then **wait** — do not write anything yet.

### Step 5 — Append the approved rows, then continue

Once the user approves (with any corrections), append exactly those rows with
`scripts/expense_sheet.py append`, then move to the next batch / next file. Repeat
until every file is done.

### Step 6 — Finish up

When all files are processed, report a short summary (count and total per file, grand
total). Then offer useful finishing touches the user may want:

- sort the rows by date (`scripts/expense_sheet.py sort`),
- add a totals row,
- upload a copy back to Google Drive (Drive `create_file` with the `.xlsx`, into the
  source folder's `parentId`).

## Formatting conventions

- **Dates**: store as text in the user's locale format. For Brazilian receipts use
  `DD/MM/YYYY`. Keeping dates as strings avoids spreadsheet apps silently
  re-interpreting them under a different locale.
- **Totals**: store as real numbers and apply a currency number format
  (`R$ #,##0.00` for BRL) so the column stays summable. Don't store "R$ 1.197,00" as text.
- **Font**: a clean, consistent font (Arial) for header and data.
- Freeze the header row so it stays visible as the list grows.

## Helper script

`scripts/expense_sheet.py` handles the spreadsheet mechanics so you don't rewrite
openpyxl each time. It reads row data as JSON to avoid quoting headaches.

```bash
# 1) create the sheet with headers (run once, in Step 1)
#    (the currency format is applied later, during append — init doesn't take it)
python scripts/expense_sheet.py init \
  --path "Expenses.xlsx" \
  --headers '["Data de Emissão","Descrição","Valor Total","Arquivo"]'

# 2) append approved rows (each row: [date, description, total, filename])
python scripts/expense_sheet.py append \
  --path "Expenses.xlsx" \
  --rows '[["02/02/2021","Ventilador de teto (Fênix Ventisol)",194.90,"benfeitorias-2021.pdf"]]'

# 3) sort all rows by date ascending (date is column 1, format DD/MM/YYYY)
python scripts/expense_sheet.py sort --path "Expenses.xlsx" --date-format "%d/%m/%Y"

# optional: append a TOTAL row summing the value column
python scripts/expense_sheet.py total --path "Expenses.xlsx" --label "TOTAL"
```

The script keeps the value column numeric with the currency format applied, sets the
font, and freezes the header. If the user wants a different column language/order or
currency, pass different `--headers` / `--currency-format`; the workflow is identical.

## Quick example of a good batch presentation

> **benfeitorias-2021.pdf** — 2 receipts:
>
> | # | Data | Descrição | Valor | Confiança |
> |---|------|-----------|-------|-----------|
> | 1 | 02/02/2021 | Ventilador de teto 3 pás (Fênix Ventisol) | R$ 194,90 | ✅ Alta (NF-e) |
> | 2 | 22/04/2021 ⚠️ | Automatizador de portão, central e cremalheira | R$ 1.197,00 ⚠️ | ⚠️ Baixa (manuscrito) |
>
> Receipt #2 is handwritten — please confirm the total (R$ 1.097 vs 1.197) and the
> date. Original: <link>. Approve and I'll write these two rows.

This is the texture to aim for: confident where the print is clean, explicitly humble
and specific where the OCR is shaky, and never committing a number the user hasn't seen.
