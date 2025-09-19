# InvoiceReaderSpec.md

## AI-Powered Invoice Reader (Module Spec)

### Overview
The Invoice Reader processes distributor purchase bills (images/PDFs) and converts them into structured JSON mapped into the local Postgres DB.

### Processing Options
- **Primary:** Cloud OCR + Parsing via OpenAI API (ChatGPT or similar)
- **Fallback:** Local OCR via Tesseract

### Workflow
1. User uploads invoice
2. OCR provider selected (cloud/local)
3. Response parsed into structured JSON:
```json
{
  "invoice_number": "INV-123",
  "invoice_date": "2025-08-25",
  "supplier": "ABC Pharma Pvt Ltd",
  "line_items": [
    {"medicine_name": "Paracetamol 500mg", "batch": "B123", "expiry": "2026-06-30", "qty": 10, "purchase_rate": 12.5, "mrp": 20.0}
  ]
}
```
4. User reviews/corrects entries in grid UI
5. On confirmation, records saved to Postgres

### Config
Defined in `config.yaml` or `.env`:
```yaml
ocr_provider: "openai"   # or "tesseract"
api_key: "OPENAI_API_KEY"
```

### Security & Privacy
- Only OCR + parsing step may use cloud
- API keys stored locally & securely
- User consent required for cloud use

