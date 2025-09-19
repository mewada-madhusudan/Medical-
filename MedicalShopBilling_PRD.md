# Medical Shop Billing Software — Product Requirements Document (PRD)

**Version:** 1.0

**Last updated:** 2025-09-19

**Scope:** Desktop-first billing & inventory solution for medical shops with a hybrid medicine database and an on-prem OCR + AI-assisted invoice ingestion module. Uses a local PostgreSQL instance for all data storage (no cloud by default).

---

## 1. Executive Summary
This product is a billing and inventory management application tailored for small-to-medium medical shops and pharmacies. It focuses on fast, compliant billing, accurate stock and expiry handling, supplier/purchase management, and a strong onboarding experience through a hybrid preloaded medicine database plus importer tools. A standout capability is an **AI-powered Invoice Reader** that uses local OCR and an LLM-assisted parsing step to extract purchase invoice lines and convert them into purchase entries and stock updates — with an explicit review-and-correct UX for operator validation.

## 2. Objectives
- Rapid, accurate billing and inventory updates.
- Minimal manual data entry for onboarding and daily purchase processing.
- Local-first architecture (Postgres on premises) — offline capable.
- Security, auditability, and regulatory compliance (GST, Schedule H handling).
- Scalable medicine DB seeded with authoritative datasets and enriched over time.

## 3. Target Users
- Independent pharmacists and shop owners.
- Chain pharmacy store managers.
- Store cashiers and inventory managers.

## 4. High-Level Features (summary)
- Billing & GST-compliant invoicing (thermal/A4 printing).
- Inventory: batch-level stock, expiry tracking, purchase & sales ledger.
- Hybrid medicine database: preloaded + distributor-import + manual add.
- AI Invoice Reader: image/PDF upload → OCR → AI parsing → review → import.
- Reports: sales, inventory valuation, expiry, GST reports.
- Role-based users: admin, cashier, stock-manager.
- Local Postgres DB, backup & restore.

---

# 5. Detailed Feature Spec — AI-Powered Invoice Reader (Module)

### 5.0 Overview
The Invoice Reader ingests distributor purchase bills (images/PDFs) and converts them into structured `PurchaseInvoice` records with `PurchaseLineItem` and `Stock` updates. It runs entirely on-premise: OCR via a local engine and a local or self-hosted LLM (or a tightly-controlled API if teams later choose to self-host). The system always shows a **Review & Correct** screen before committing any change.

### 5.1 User Flow
1. User navigates to **Purchases → Import Bill**.
2. User uploads an image or PDF (single/multi-page). Allowed formats: `jpg`, `png`, `tiff`, `pdf`.
3. System runs OCR on the document (background task). A progress indicator is shown.
4. OCR raw text is passed to the **AI Parsing** step which returns structured JSON representing invoice metadata & line items.
5. System performs **DB Matching** for medicine names, suppliers, and HSN/GST rates.
6. System shows the **Parsed Invoice Review** screen where each line is editable (inline) and mismatches highlighted.
7. User corrects entries (name, batch, expiry, qty, price, MRP). Options: map to existing medicine, create new medicine record, flag as unknown.
8. User clicks **Confirm & Save**. System creates `PurchaseInvoice`, `PurchaseLineItem` entries and updates `Stock` (batch-level). Audit logs saved.

### 5.2 OCR Layer
- **Primary engine:** Tesseract OCR (via `pytesseract`) for local deployment. Provide option to switch to `EasyOCR` or `PaddleOCR` if needed.
- **Preprocessing pipeline:** image deskewing, denoising, contrast enhancement, and optional region-of-interest (ROI) detection to isolate tabular areas.
- **PDF handling:** render pages to images (e.g. `pdf2image`) and run OCR per page.
- **Output:** raw text with positional coordinates (where supported) and per-line confidences.

### 5.3 AI Parsing Layer
- **Purpose:** convert noisy OCR text into structured invoice JSON: invoice-level metadata and line items.
- **Implementation options (local-first):**
  - Option A: Local LLM (if you host LLM on premise) with a parsing prompt and JSON schema.
  - Option B: Lightweight rule-based + regex fallback combined with a small on-device transformer/parsing model.
- **Example output schema:**
```json
{
  "invoice_number": "INV-123",
  "invoice_date": "2025-08-25",
  "supplier": "ABC Pharma Pvt Ltd",
  "total_amount": 12345.67,
  "currency": "INR",
  "line_items": [
    {"line_no": 1, "medicine_name": "Paracetamol 500mg", "batch": "B123", "expiry": "2026-06-30", "qty": 10, "purchase_rate": 12.5, "mrp": 20.0},
    ...
  ]
}
```
- **Prompting guidelines (if using LLM):** give the model a strict output JSON schema, examples, and rules for ambiguous rows (e.g., when OCR merges columns). Enforce `json` output only and validate it.
- **Confidence scoring:** each parsed field gets a confidence value from AI + OCR; fields under a threshold are flagged for manual review.

### 5.4 Name Matching & Fuzzy Matching
- **Exact match**: match `medicine_name` to canonical `medicine.name` using normalized tokens (lowercase, remove punctuation, ignore packaging indicators like "10x10").
- **Fuzzy match**: compute Levenshtein distance or token-set ratio (e.g., `fuzzywuzzy` or `python-Levenshtein`).
- **Similarity thresholding:**
  - >95%: auto-match.
  - 70-95%: suggest top 3 matches for user to pick.
  - <70%: mark as unknown and suggest adding new medicine.
- **Dedup & canonicalization:** strip extra words like "Syrup", "Tab", pack size markers and compare base drug name + strength.

### 5.5 Supplier Matching
- Match supplier name to `Supplier` table using normalized string matching. Allow one-click to create a new supplier if none found.

### 5.6 Review & Correction UI
- **Sidebar:** invoice metadata (supplier, date, invoice no, totals) with edit option.
- **Main grid:** parsed line items in rows:
  - Columns: `#`, `Medicine (parsed)` (editable), `Matched Medicine (linkable)`, `Batch`, `Expiry`, `Quantity`, `Purchase Rate`, `MRP`, `Confidence`, `Action` (Map/Create/Ignore).
- **Inline helpers:**
  - Dropdown of suggested matches (top 3 fuzzy matches) with match scores.
  - Button `Create New Medicine` that opens a lightweight modal to enter essential properties (generic, manufacturer, HSN, gst_rate).
  - Quick edit shortcuts: double-click cell, Tab-to-next, bulk apply supplier or MRP.
- **Validation:** expiry format, numeric checks for qty/price, negative values blocking.
- **Bulk actions:** mark multiple rows to create new medicine entries, or adjust MRP multiplier, or discard rows.

### 5.7 Import Rules & Stock Update
- After confirmation:
  - Create a `PurchaseInvoice` record (invoice header).
  - For each line, create `PurchaseLineItem` with references to `Medicine.id` (or null if new/created) and `Stock` entries (batch-level) with `quantity`, `expiry`, `purchase_rate`, `mrp`, and `supplier_id`.
  - Update inventory & `Stock` aggregates.
  - Generate an audit log entry per created/modified DB row with `user_id` and timestamp.

### 5.8 Error Handling & Fallback
- **OCR fails**: show user with an option to download OCR text, switch OCR engine, or manually enter invoice.
- **Parsing fails or invalid JSON**: fall back to table-detection + regex extraction and show raw lines for mapping.
- **Partial match/confidence low**: force manual confirmation for those lines.
- **Malformed invoice (columns shifted)**: provide a guided column-mapping UI where user drags OCR-detected columns to target fields.

### 5.9 Security & Privacy
- All processing occurs locally on the machine or within the local network.
- Postgres connection must be secured by strong password & local firewall rules.
- Optional full-disk encryption for laptop deployments.
- Audit logs stored in DB with user attribution.

### 5.10 Performance & Scaling
- Typical invoice: 20–200 line items. Target parse time: <10s for a 2-page invoice on a modest modern machine (quad-core, 8GB RAM).
- Use asynchronous job queue (e.g., `celery` or `rq`) for OCR and parsing to avoid UI blocking.
- Cache fuzzy match indices (e.g., trigram indexes) for faster name matching; consider `pg_trgm` extension in Postgres.

### 5.11 Testing & Validation
- Create a test corpus of distributor bills (good-quality, low-quality, rotated, multi-column, merged cells) and measure:
  - OCR character accuracy.
  - Line-level extraction accuracy.
  - Field-level precision/recall for medicine name, qty, price, batch, expiry.
- Define acceptance criteria: >95% invoice-level correctness after user review in Phase 1; >99% after Phase 3.

---

# 6. Data Model (Core Tables) — simplified

```sql
-- Medicines
CREATE TABLE medicines (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  generic_name TEXT,
  manufacturer TEXT,
  hsn_code TEXT,
  gst_rate NUMERIC(5,2),
  packaging TEXT,
  schedule TEXT,
  created_at TIMESTAMP DEFAULT now()
);

-- Suppliers
CREATE TABLE suppliers (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  gstin TEXT,
  contact JSONB,
  created_at TIMESTAMP DEFAULT now()
);

-- Purchase Invoices
CREATE TABLE purchase_invoices (
  id SERIAL PRIMARY KEY,
  supplier_id INTEGER REFERENCES suppliers(id),
  invoice_number TEXT,
  invoice_date DATE,
  total_amount NUMERIC(14,2),
  created_by INTEGER,
  created_at TIMESTAMP DEFAULT now()
);

-- Purchase Line Items
CREATE TABLE purchase_line_items (
  id SERIAL PRIMARY KEY,
  purchase_invoice_id INTEGER REFERENCES purchase_invoices(id),
  medicine_id INTEGER REFERENCES medicines(id),
  raw_name TEXT,
  batch_no TEXT,
  expiry_date DATE,
  quantity INTEGER,
  purchase_rate NUMERIC(14,4),
  mrp NUMERIC(14,2),
  confidence JSONB,
  created_at TIMESTAMP DEFAULT now()
);

-- Stock (batch-level)
CREATE TABLE stock_batches (
  id SERIAL PRIMARY KEY,
  medicine_id INTEGER REFERENCES medicines(id),
  batch_no TEXT,
  expiry_date DATE,
  quantity INTEGER,
  purchase_rate NUMERIC(14,4),
  supplier_id INTEGER REFERENCES suppliers(id),
  created_at TIMESTAMP DEFAULT now()
);

-- Audit log
CREATE TABLE audit_logs (
  id SERIAL PRIMARY KEY,
  user_id INTEGER,
  action TEXT,
  payload JSONB,
  created_at TIMESTAMP DEFAULT now()
);
```

Notes:
- Enable `pg_trgm` extension for fast fuzzy matching (recommended): `CREATE EXTENSION pg_trgm;`
- Add indexes on `lower(name)` and trigram indexes for `medicines.name` and `suppliers.name`.

---

# 7. UI / UX Considerations
- Desktop-first layout; optimized for keyboard-driven data entry.
- Large fonts and clear buttons for pharmacy counters.
- Dedicated **Purchase Import** screen with progress, parsing summary, and prominent Confirm / Edit controls.
- Undo capability for imports (soft-delete + revert stock changes).
- Accessible design (contrast, spacing) for staff who may have poor lighting.

---

# 8. Phased Roadmap (detailed)

**Phase 0 — Seed & Core (MVP)**
- Local Postgres setup & schema.
- Billing, printing, simple inventory, medicine DB seeded (~25k common SKU list).
- Distributor CSV/Excel import module.
- Manual add/edit medicine UI.

**Phase 1 — OCR + Manual Mapping**
- Integrate local Tesseract OCR, basic preprocessing.
- Upload PDF/Image → OCR → show raw lines for manual mapping to fields.
- Review UI to accept/save purchase invoices.

**Phase 2 — AI-Assisted Parsing (Recommended target)**
- Integrate AI parsing layer (local LLM or light parsing model + heuristics).
- Fuzzy matching and suggestion dropdown.
- Confidence scoring & auto-approve rules.
- Performance tuning and caching.

**Phase 3 — Automation & Improvements**
- Improve parsing accuracy with active learning (user-corrected rows used to re-train or fine-tune parsing heuristics).
- Optional self-hosted LLM fine-tuning for domain-specific patterns.
- Add cloud sync as optional (for multi-store chains) — opt-in only.

---

# 9. Acceptance Criteria & Success Metrics
- Invoice import time: < 10s for a 2-page invoice for OCR+parse on target hardware.
- After review, >95% of parsed fields require no correction (Phase 2 target).
- Inventory update atomicity: no partial updates; operations fully auditable.
- Users can correct all fields prior to commit; rollback available.

---

# 10. Operational Notes (Dev/Ops)
- **Local Postgres setup:** Provide an installer or scripts to create DB, user, and necessary extensions (`pg_trgm`).
- **Backups:** daily scheduled `pg_dump` to a configurable local folder with retention.
- **Deployment:** single binary/installer for Windows and Linux (Electron or PyInstaller for PyQt builds).
- **Logs & Support:** log OCR/parsing failures with sample masked images for debugging (respecting privacy).

---

# 11. Security & Compliance
- Role-based access control (RBAC) and password policy.
- Optionally enable encrypted DB credentials per deployment.
- Data export only via authenticated admin UI; logs record exports.
- Do not transmit invoices or patient data off-premises by default.

---

# 12. Developer Notes & Suggested Libraries
- OCR: `pytesseract`, `opencv-python`, `pdf2image`.
- Fuzzy matching: `python-Levenshtein`, `fuzzywuzzy` (or `rapidfuzz`).
- Job queue: `rq` or `celery` (with Redis) — or threaded worker for small installs.
- Backend: `FastAPI` or `Flask`.
- Frontend (desktop): `PyQt6` (native) or `Electron` (web stack). PyQt6 recommended for lower resource footprint.
- LLM integration: modular adapter so teams can swap `local LLM` or `in-house LLM` later.

---

# 13. Directory structure & deliverables (suggested repo layout)
```
medical-billing/
├── docs/
│   ├── PRD.md
│   ├── InvoiceReaderSpec.md
│   └── Deployment.md
├── api/
│   └── (FastAPI app code)
├── desktop/
│   └── (PyQt6 UI code)
├── services/
│   ├── ocr_worker/
│   └── parsing_worker/
├── db/
│   └── migrations/
├── data/
│   └── seed_medicines.csv
└── tests/
    └── invoices_corpus/
```

---

# 14. Next steps / Recommendations
1. Prepare a **seed medicine dataset** (India-focused): combine NPPA price lists + a curated supplier CSV. Clean & normalize names.
2. Build a small invoice corpus (20–50 real-world invoices, masked) to iterate OCR & parsing heuristics.
3. Implement Phase 1 (OCR + manual mapping) first — this delivers immediate value and reduces manual entry pain.
4. After Phase 1 stable, add AI parsing + fuzzy matching (Phase 2) and measure accuracy.

---

# 15. Appendix — Example prompts for AI parsing (if using an LLM)
**System prompt skeleton:**
```
You are an invoice-parser. Input: noisy OCR text from a distributor invoice. Output: valid JSON strictly matching schema described below. If a field is missing or uncertain, set its value to null and add a confidence score (0-1). Do not include any extra commentary.
```
**Example JSON schema:** (see 5.3 above)

---

*End of document.*
