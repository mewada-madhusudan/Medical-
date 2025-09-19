# Medical Shop Billing Software — PRD

This file is the main Product Requirements Document for the Medical Shop Billing Software.

It covers the overall goals, objectives, user flows, data model, and phased roadmap.

For detailed AI-powered invoice reader spec and deployment notes, see the respective documents.

## Executive Summary
This product is a billing and inventory management application tailored for small-to-medium medical shops and pharmacies.

- Application & Database: local Postgres instance
- OCR & AI Parsing: via OpenAI/ChatGPT API (cloud) or local Tesseract fallback
- Review & Correction workflow before database commit
- Secure, local-first design with optional cloud OCR/AI

## Objectives
- Rapid, accurate billing and inventory
- Minimize manual entry using OCR + AI parsing of invoices
- Hybrid medicine database (seeded + import + manual)
- Role-based local app with Postgres backend

## Target Users
- Pharmacists & shop owners
- Pharmacy chain managers
- Cashiers & stock managers

## High-Level Features
- GST-compliant billing & printing
- Inventory management with expiry & batch
- AI Invoice Reader (OCR + AI parsing + review)
- Reports (sales, expiry, GST)
- Role-based users
- Local Postgres database

## Phased Roadmap
- Phase 0: Manual DB, billing, seed medicines
- Phase 1: Local OCR (Tesseract)
- Phase 2: Cloud OCR/AI (OpenAI GPT JSON parsing)
- Phase 3: Optimizations, multi-store sync

