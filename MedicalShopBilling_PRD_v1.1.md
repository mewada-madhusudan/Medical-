# Medical Shop Billing Software — Product Requirements Document (PRD)

**Version:** 1.1  
**Last updated:** 2025-09-19  

**Scope:** Desktop-first billing & inventory solution for medical shops with a hybrid medicine database.  
- Application & Database (Postgres) run **locally** on the machine.  
- OCR & AI Parsing can use **cloud services like ChatGPT/OpenAI API** (with JSON schema enforcement) or fallback to local Tesseract OCR.  

---

## 1. Executive Summary
This product is a billing and inventory management application tailored for small-to-medium medical shops and pharmacies. It focuses on fast, compliant billing, accurate stock and expiry handling, supplier/purchase management, and a strong onboarding experience through a hybrid preloaded medicine database plus importer tools.  

A standout capability is an **AI-powered Invoice Reader** that can either:  
- Use **ChatGPT/OpenAI (or similar)** for OCR + parsing into structured JSON, OR  
- Use **local OCR (Tesseract)** for offline fallback.  

The system always presents a **Review & Correct** screen before committing changes to the local PostgreSQL database.  

---

## 2. Objectives
- Rapid, accurate billing and inventory updates.  
- Minimize manual entry using OCR + AI parsing of purchase invoices.  
- Local-first architecture: app + Postgres DB run on local machine.  
- Configurable OCR/Parsing provider (cloud vs local fallback).  
- Ensure user privacy and consent if cloud OCR/AI is used.  

---

## 3. Target Users
- Independent pharmacists and shop owners.  
- Chain pharmacy store managers.  
- Store cashiers and inventory managers.  

---

## 4. High-Level Features
- Billing & GST-compliant invoicing (thermal/A4 printing).  
- Inventory: batch-level stock, expiry tracking, purchase & sales ledger.  
- Hybrid medicine database: preloaded + distributor-import + manual add.  
- AI Invoice Reader: upload bill → OCR/AI → structured JSON → review → DB.  
- Reports: sales, inventory valuation, expiry, GST reports.  
- Role-based users: admin, cashier, stock-manager.  
- Local Postgres DB with backup/restore tools.  

---

# 5. Detailed Feature Spec — AI-Powered Invoice Reader (Module)

### 5.0 Overview
The Invoice Reader ingests distributor purchase bills (images/PDFs) and converts them into structured `PurchaseInvoice` records with `PurchaseLineItem` and `Stock` updates.  

**Processing options:**  
- **Primary (recommended):** Cloud OCR + Parsing via OpenAI API (ChatGPT or similar).  
- **Fallback:** Local OCR via Tesseract.  

### 5.1 User Flow
1. User uploads invoice (image/PDF).  
2. System calls the configured OCR/Parsing provider:  
   - If **OpenAI API** is selected → image/PDF sent to cloud, response returned in structured JSON.  
   - If **Tesseract (local)** is selected → local OCR → raw text → AI parsing (local or OpenAI).  
3. JSON result mapped against local medicine/supplier database.  
4. User sees **Review & Correct** screen → edits mismatches.  
5. User confirms → data saved into Postgres (purchase invoice + stock update).  

### 5.2 OCR Layer
- **Option A: Cloud OCR (OpenAI/other provider):**  
  - Send scanned image/PDF directly.  
  - Response: structured JSON with invoice metadata + line items.  

- **Option B: Local OCR (Tesseract):**  
  - Preprocess (deskew, denoise, contrast).  
  - Extract raw text.  
  - Pass to AI parsing layer (could still be OpenAI GPT for text→JSON).  

### 5.3 AI Parsing Layer
- **Cloud-first:** Use OpenAI GPT with a strict JSON schema for invoice parsing.  
- **Output schema (example):**  
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
- **Validation:** enforce JSON schema on response. If invalid, retry or fallback to manual mapping.  

### 5.4 Review & Correction UI
(Same as earlier spec: editable grid, fuzzy matches, bulk corrections, validation.)  

### 5.5 Security & Privacy
- Only **OCR + Parsing step** may use cloud APIs.  
- App + DB remain **fully local**.  
- Show user a **consent notice** before first-time cloud OCR/AI usage.  
- API keys stored securely in `.env` or `config.yaml`.  

### 5.6 Deployment Config
- Config file defines provider:  
```yaml
ocr_provider: "openai"   # or "tesseract"
api_key: "OPENAI_API_KEY"
```  

---

# 6. Data Model (Core Tables)
(unchanged from v1.0 — Postgres schema for medicines, suppliers, invoices, stock, audit logs.)  

---

# 7. UI / UX Considerations
- Upload → progress → structured response → editable grid.  
- Highlight low-confidence fields.  
- Explicit **Review & Save** step (no auto-commit).  

---

# 8. Phased Roadmap

**Phase 0:** Local DB, billing, manual medicine entry, seed DB.  
**Phase 1:** Tesseract OCR + manual mapping.  
**Phase 2:** OpenAI API integration for OCR + parsing → JSON output.  
**Phase 3:** Automation improvements, caching, active learning, optional multi-store sync.  

---

# 9. Security & Compliance
- All data remains local except uploaded invoices when using OpenAI API.  
- Ensure user awareness & opt-in.  
- Role-based access for local users.  

---

# 10. Operational Notes (Deployment)
- Local Postgres + app installer.  
- `.env` or `config.yaml` for API key & OCR provider config.  
- Backup: daily `pg_dump`.  

---

# 11. Developer Notes & Libraries
- Cloud OCR/Parsing: OpenAI GPT (images + JSON output).  
- Local fallback: `pytesseract`, `opencv-python`, `pdf2image`.  
- Fuzzy matching: `rapidfuzz`.  
- Desktop UI: `PyQt6`.  

---

# 12. Repo Structure
```
medical-billing/
├── docs/
│   ├── PRD.md
│   ├── InvoiceReaderSpec.md
│   └── Deployment.md
├── api/ (FastAPI backend)
├── desktop/ (PyQt6 UI)
├── services/ (ocr/parsing workers)
├── db/ (migrations)
├── data/ (seed_medicines.csv)
└── tests/ (invoice corpus)
```

---

# 13. Next Steps
1. Build Phase 1 with local OCR/manual mapping.  
2. Add OpenAI API OCR+Parsing integration.  
3. Validate with real distributor invoices.  

---

*End of Document v1.1*
