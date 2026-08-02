---
name: david-ocr-skill
description: Use when extracting invoice data from PDF invoices for accounting/QuickBooks-style workflows. Uses PyMuPDF for digital PDFs, marker-pdf for scanned/image PDFs, and returns clean structured JSON only.
version: 1.0.0
author: David OCR Skill
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [ocr, invoices, pdf, accounting, quickbooks, json, pymupdf, marker-pdf]
    related_skills: [ocr-and-documents]
---

# David OCR Skill

## Overview

Use this skill to read invoice PDFs accurately and extract invoice data into structured JSON. The goal is to produce clean machine-readable output that another program can map into QuickBooks, accounting software, spreadsheets, or an approval workflow.

This skill uses:

- **PyMuPDF / pymupdf** for digital PDFs where text is selectable or extractable.
- **marker-pdf** for scanned invoices, image-based PDFs, poor OCR, complex layouts, tables, forms, and mixed image/text documents.

The final invoice-processing response should be **JSON only** — no prose, markdown, explanations, or commentary.

## When to Use

Use this skill when:

- The user provides a PDF invoice and wants structured data extracted.
- The invoice may be scanned, photographed, faxed, or image-based.
- The user wants JSON output for later accounting import.
- The user needs fields commonly required for QuickBooks Bills, Expenses, or Vendor Invoices.

Do not use this skill for:

- Posting directly to QuickBooks without explicit authorization.
- Guessing missing invoice values.
- Legal/tax/accounting advice beyond extracting what the invoice says.

## Dependencies

Install the required tools:

```bash
pip install pymupdf marker-pdf
```

Optional but useful:

```bash
pip install python-dateutil pydantic
```

Notes:

- `pymupdf` is lightweight and fast.
- `marker-pdf` may download large OCR/layout models on first use.
- For scanned invoices, marker-pdf is preferred because it handles OCR and layout better.

## PDF Type Detection

Always detect the PDF type before extraction.

### Rule

1. Try PyMuPDF first.
2. If text is selectable/extractable and has meaningful invoice content, treat it as a **digital PDF**.
3. If text is empty, garbled, incomplete, or missing table/line-item data, treat it as a **scanned PDF** and use marker-pdf.
4. If unsure, run both and prefer the extraction with better field coverage and validation results.

### PyMuPDF text check

```python
import fitz  # pymupdf
from pathlib import Path

def extract_text_with_pymupdf(pdf_path: str) -> dict:
    doc = fitz.open(pdf_path)
    pages = []
    for i, page in enumerate(doc):
        text = page.get_text("text") or ""
        pages.append({"page": i + 1, "text": text})
    combined = "\n\n".join(p["text"] for p in pages)
    return {
        "file_name": Path(pdf_path).name,
        "page_count": len(doc),
        "text": combined,
        "text_length": len(combined.strip())
    }

def looks_digital_pdf(extracted: dict) -> bool:
    text = extracted.get("text", "").strip()
    if len(text) < 100:
        return False
    invoice_keywords = ["invoice", "bill to", "subtotal", "total", "tax", "amount due", "due date"]
    hits = sum(1 for k in invoice_keywords if k.lower() in text.lower())
    return hits >= 2
```

## Extraction Workflow

### Digital PDF workflow: PyMuPDF

Use PyMuPDF when the PDF has selectable text.

```python
import fitz

def pymupdf_invoice_text(pdf_path: str) -> str:
    doc = fitz.open(pdf_path)
    parts = []
    for page_number, page in enumerate(doc, start=1):
        text = page.get_text("text") or ""
        parts.append(f"\n--- PAGE {page_number} ---\n{text}")
    return "\n".join(parts)
```

For line-item tables, also try structured block extraction:

```python
import fitz

def pymupdf_blocks(pdf_path: str) -> list:
    doc = fitz.open(pdf_path)
    results = []
    for page_number, page in enumerate(doc, start=1):
        blocks = page.get_text("blocks")
        for block in blocks:
            x0, y0, x1, y1, text, block_no, block_type = block
            results.append({
                "page": page_number,
                "bbox": [x0, y0, x1, y1],
                "text": text,
                "block_no": block_no,
                "block_type": block_type
            })
    return results
```

### Scanned PDF workflow: marker-pdf

Use marker-pdf when the invoice is scanned or image-based.

CLI example:

```bash
marker_single invoice.pdf --output_dir ./invoice_output
```

Batch example:

```bash
marker /path/to/invoice_folder --workers 4 --output_dir ./invoice_output
```

Expected result: marker-pdf will produce Markdown/JSON-like extracted content depending on version and options. Use the extracted Markdown/text as the source for structured invoice JSON.

If marker supports JSON output in your installed version, use it. Otherwise use the generated Markdown.

## Structured JSON Output

When processing an invoice, return **only JSON** in this schema:

```json
{
  "vendor_name": "",
  "vendor_address": "",
  "vendor_phone": "",
  "vendor_email": "",
  "invoice_number": "",
  "invoice_date": "",
  "due_date": "",
  "payment_terms": "",
  "po_number": "",
  "bill_to": "",
  "ship_to": "",
  "currency": "USD",
  "subtotal": null,
  "tax": null,
  "shipping": null,
  "discount": null,
  "total": null,
  "amount_due": null,
  "line_items": [
    {
      "description": "",
      "quantity": null,
      "unit_price": null,
      "amount": null,
      "sku": "",
      "service_date": "",
      "account_category": ""
    }
  ],
  "confidence": {
    "overall": 0.0,
    "vendor_name": 0.0,
    "invoice_number": 0.0,
    "invoice_date": 0.0,
    "total": 0.0,
    "line_items": 0.0
  },
  "validation": {
    "subtotal_plus_tax_matches_total": null,
    "line_items_sum_matches_subtotal": null,
    "warnings": []
  },
  "source": {
    "file_name": "",
    "pdf_type": "digital_or_scanned",
    "extraction_method": "pymupdf_or_marker_pdf",
    "page_count": null
  }
}
```

## Field Rules

- Preserve invoice numbers exactly as written.
- Preserve vendor names exactly as written.
- Normalize money values to numbers without dollar signs or commas.
- Use `null` for unknown numeric values.
- Use empty string `""` for unknown text fields.
- Do not guess missing values.
- Dates should be ISO format `YYYY-MM-DD` when confidently parsed.
- If a date is ambiguous, keep the original text and add a warning.
- If a total is unclear, set confidence lower and add a warning.
- If line items cannot be reliably extracted, return an empty list and add a warning.

## Validation Rules

After extraction, validate:

1. Line items sum to subtotal.
2. Subtotal + tax + shipping - discount equals total.
3. Amount due matches total unless invoice shows payments/credits.
4. Required accounting fields are present: vendor, invoice number, invoice date, total.

Use tolerances for rounding:

- Money comparisons should allow a difference up to $0.01.
- If a validation check cannot be performed, use `null`.
- Put all issues in `validation.warnings`.

Example warnings:

```json
[
  "Invoice date was not found.",
  "Line item sum differs from subtotal by 0.02.",
  "PDF appears scanned; OCR confidence may be lower.",
  "Due date is ambiguous: 03/04/25."
]
```

## Confidence Scoring

Use confidence values from 0.0 to 1.0.

Suggested scoring:

- `1.0`: exact, clear, validated.
- `0.8`: clear extraction but not independently validated.
- `0.6`: likely correct but formatting/layout is imperfect.
- `0.4`: uncertain OCR or ambiguous location.
- `0.0`: missing or not extractable.

Overall confidence should reflect the weakest important fields: vendor, invoice number, date, total, and line items.

## QuickBooks Readiness

This skill does **not** post to QuickBooks. It prepares structured data that can be mapped later.

Common QuickBooks mapping:

- `vendor_name` → Vendor
- `invoice_number` → Bill No. / Ref No.
- `invoice_date` → Bill Date / Transaction Date
- `due_date` → Due Date
- `total` / `amount_due` → Amount
- `line_items[].description` → Description
- `line_items[].quantity` → Qty
- `line_items[].unit_price` → Rate
- `line_items[].amount` → Amount
- `line_items[].account_category` → Expense Category / Account
- Original PDF → Attachment

Require human review before creating or posting accounting records.

## End-to-End Example

```python
from pathlib import Path
import json
import subprocess
import fitz


def extract_pymupdf_text(pdf_path):
    doc = fitz.open(pdf_path)
    text_parts = []
    for page_number, page in enumerate(doc, start=1):
        text_parts.append(f"--- PAGE {page_number} ---\n" + (page.get_text("text") or ""))
    return {
        "file_name": Path(pdf_path).name,
        "page_count": len(doc),
        "text": "\n\n".join(text_parts)
    }


def should_use_marker(extracted):
    text = extracted["text"].strip().lower()
    if len(text) < 100:
        return True
    required_keywords = ["invoice", "total", "date"]
    return sum(1 for k in required_keywords if k in text) < 2


def run_marker(pdf_path, output_dir="./marker_output"):
    Path(output_dir).mkdir(parents=True, exist_ok=True)
    subprocess.run([
        "marker_single",
        pdf_path,
        "--output_dir",
        output_dir
    ], check=True)
    return output_dir


pdf_path = "invoice.pdf"
extracted = extract_pymupdf_text(pdf_path)

if should_use_marker(extracted):
    marker_dir = run_marker(pdf_path)
    source_text = f"Use marker-pdf output from {marker_dir}"
    method = "marker_pdf"
    pdf_type = "scanned"
else:
    source_text = extracted["text"]
    method = "pymupdf"
    pdf_type = "digital"

print(json.dumps({
    "source": {
        "file_name": Path(pdf_path).name,
        "pdf_type": pdf_type,
        "extraction_method": method,
        "page_count": extracted["page_count"]
    },
    "next_step": "Send source_text to the LLM/extraction layer and require the final invoice JSON schema only."
}, indent=2))
```

## Prompt Template for Invoice Extraction

Use this prompt after extracting text/OCR from the invoice:

```text
Extract invoice data from the following PDF/OCR text.

Return JSON only. No markdown. No prose.

Rules:
- Do not guess missing values.
- Use null for unknown numbers.
- Use empty strings for unknown text.
- Preserve exact invoice numbers and vendor names.
- Normalize money to numbers.
- Validate totals and line items.
- Add warnings for uncertainty.

Use this schema:
{PASTE THE JSON SCHEMA HERE}

Invoice text:
{PASTE EXTRACTED TEXT HERE}
```

## Common Pitfalls

1. **Using OCR for every PDF**  
   PyMuPDF is faster and cleaner for digital PDFs. OCR should be fallback or used for scanned invoices.

2. **Trusting OCR totals without validation**  
   Always check subtotal, tax, shipping, discount, total, and line-item sums.

3. **Guessing vendor or category**  
   Do not invent missing accounting categories. Leave blank unless clearly present or provided by business rules.

4. **Changing invoice numbers**  
   Invoice numbers must be preserved exactly, including prefixes, dashes, and leading zeros.

5. **Returning explanations instead of JSON**  
   For automation, final output must be raw JSON only.

6. **Posting directly to QuickBooks**  
   This skill extracts and validates data. It does not authorize accounting entries.

## Verification Checklist

Before returning final JSON:

- [ ] PDF type was detected: digital or scanned.
- [ ] Correct extractor was used: PyMuPDF or marker-pdf.
- [ ] Vendor name was extracted or left blank.
- [ ] Invoice number was extracted exactly or left blank.
- [ ] Invoice date and due date were extracted or warnings were added.
- [ ] Subtotal, tax, shipping, discount, total, and amount due are numeric or null.
- [ ] Line items are structured, or warning explains why unavailable.
- [ ] Totals were validated where possible.
- [ ] Confidence scores were assigned.
- [ ] Final response is valid JSON only.
