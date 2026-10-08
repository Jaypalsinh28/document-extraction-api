# Automated Document Processing & Information Extraction

A REST API that reads invoices (PDF, scanned image or text), pulls out the key fields with **OCR + NLP/LLM**, validates them, and stores the results in a **database**.

**Tech stack:** Python, OCR (Tesseract), NLP, LLMs (Anthropic API), FastAPI REST API, SQLAlchemy (SQLite / PostgreSQL)

## How it works

```
upload -> text (PDF text layer, or OCR for scans/images)
       -> extraction (LLM if API key set, otherwise rule-based; LLM errors fall back to rules)
       -> normalise (dates, amounts) -> validate (missing fields, subtotal + tax = total)
       -> save to database -> JSON response (needs_review + warnings)
```

- `app/ocr.py`: PDF text layer first, Tesseract OCR for scanned PDFs and images.
- `app/llm_extractor.py`: LLM returns the fields as JSON.
- `app/regex_extractor.py`: offline rule-based extractor (also the fallback).
- `app/pipeline.py`: normalisation and validation. Anything suspicious sets `needs_review`.
- `app/db.py`: SQLAlchemy model; SQLite by default, PostgreSQL via `DATABASE_URL`.
- `app/main.py`: FastAPI endpoints.

## Quick start

Install [Tesseract](https://github.com/tesseract-ocr/tesseract) for OCR (skip if you only use text/digital PDFs).

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt
cp .env.example .env            # optional: add ANTHROPIC_API_KEY to enable the LLM
uvicorn app.main:app --reload   # http://127.0.0.1:8000/docs
pytest
```

## Endpoints

| Method | Path | What it does |
|---|---|---|
| POST | `/documents` | Upload a PDF / image / .txt invoice |
| POST | `/documents/text` | Submit already-extracted text as JSON |
| GET | `/documents` | List documents (`?needs_review=true` to filter) |
| GET | `/documents/{id}` | Get one document |
| DELETE | `/documents/{id}` | Delete a document |

```bash
curl -X POST http://127.0.0.1:8000/documents -F "file=@data/sample_invoice.txt"
```

```json
{"id": 1, "vendor": "Acme Supplies Pvt Ltd", "invoice_number": "INV-2026-0042",
 "invoice_date": "2026-09-15", "currency": "INR", "subtotal": 1000.0, "tax": 180.0,
 "total": 1180.0, "method": "regex", "needs_review": false, "warnings": []}
```

## Docker

```bash
docker build -t doc-extraction .
docker run -p 8000:8000 -e ANTHROPIC_API_KEY=... doc-extraction
```

## Notes

- Ambiguous dates such as `03/04/2026` are read day-first.
- Rule-based extraction suits common invoice layouts; use the LLM for varied formats and always check `needs_review`.
- OCR output quality depends on scan quality.

## Author

Jaypalsinh Parmar — [GitHub](https://github.com/Jaypalsinh28)
