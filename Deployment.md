# Deployment.md

## Deployment & Operations

### Architecture
- Application: PyQt6 Desktop App
- Backend: FastAPI (local services)
- Database: Local PostgreSQL instance
- OCR/AI: Configurable (OpenAI API or local Tesseract)

### Setup
1. Install local Postgres & create schema
2. Install Python dependencies
3. Configure `.env` or `config.yaml` with OCR provider & API keys
4. Launch desktop app

### Security
- All data stored locally in Postgres
- Only uploaded invoices transmitted externally (if cloud OCR enabled)
- Role-based local user accounts

### Backup
- Daily `pg_dump` backups to local drive
- Manual restore utility included

