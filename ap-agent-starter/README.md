# Accounts Payable Agent — feature-upgraded local demo

This is a safety-first local development scaffold for an Accounts Payable (AP) workflow. It now includes the original vendor/invoice/mock-payment flow plus invoice document extraction, PO/GRN/invoice matching, in-app workflow notifications, analytics, and normalized ERP export previews.

## Important safety boundaries

- **Real transfers are disabled in code.** `PAYMENT_MODE` must be `mock`; the app refuses to start otherwise.
- A completed payment is a **simulated record only**. No bank account, wallet, SAP or Oracle connection is made.
- Every simulated payment still requires explicit confirmation and a named approver in the local demo.
- New vendors are unapproved by default. Invoice duplicates are rejected, and PO/receipt mismatches remain held for review.
- AI/OCR extraction produces draft fields only. It never creates or pays an invoice automatically. Verify every extracted field.
- ERP features create export previews only. SAP and Oracle adapters are **not connected** and do not upload data.
- This is not production-secure: the local UI does not authenticate user identity or implement role-based access. Do not expose it publicly or use real vendor bank details.

## Windows PowerShell setup

Requires Python 3.12 (Python 3.11+ should also work).

```powershell
cd ap-agent-starter
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
Copy-Item .env.example .env
uvicorn app.main:app --reload
```

Open `http://127.0.0.1:8000` for the UI and `http://127.0.0.1:8000/docs` for API testing.

If `.env` already exists, do not overwrite it blindly. Add the optional `OPENAI_API_KEY` and `OPENAI_MODEL` values manually if you want LLM extraction. Keep keys private.

## New features

### 1. Invoice reader (OCR + optional LLM)

UI: **Invoice AI**. Upload a PDF, PNG/JPG, or TXT file (maximum 10 MB). PDFs with selectable text are read with `pypdf`; images use Tesseract OCR; scanned PDFs try a PyMuPDF render followed by Tesseract. If `OPENAI_API_KEY` is set, extracted text is passed to the configured OpenAI model for structured field extraction. Otherwise, a conservative regex fallback is used.

Image/scanned-PDF OCR requires the Tesseract application on Windows. Install it separately and ensure it is on PATH, or set `TESSERACT_CMD` in `.env`. AI results may be wrong; treat them as suggestions, not accounting evidence.

Endpoints:
- `POST /api/invoices/ingest` (multipart form field named `file`)
- `GET /api/invoices/documents`

### 2. Three-way matching

UI: **PO / GRN match**. Create a purchase order, record the goods receipt, then match the invoice against the PO and receipt. Checks include vendor, currency, invoice-vs-PO amount, invoice-vs-received amount, and receipt-to-PO linkage. A mismatch holds the invoice. A passing match still requires human approval and is still subject to spend limits.

Endpoints:
- `POST/GET /api/purchase-orders`
- `POST/GET /api/goods-receipts`
- `POST /api/matching/run?invoice_id=1&po_number=PO-2026-001&grn_number=GRN-2026-001`
- `GET /api/matching/history`

### 3. Approval/workflow notifications

UI: **Notifications**. The app records local alerts when invoices need review/approval, vendors are approved or their demo destination changes, three-way matching passes/fails, and mock payments are recorded. Notifications are in-app only; email and Microsoft Teams delivery are not configured.

Endpoints:
- `GET /api/notifications?unread_only=true`
- `PATCH /api/notifications/{notification_id}/read`

### 4. Analytics

UI: **Analytics / ERP**. Shows invoice status counts, held invoices, simulated paid totals by currency, simulated spend by vendor/month, and selected safety-block counts.

Endpoint: `GET /api/analytics/overview`

### 5. ERP integration scaffolding

The UI can prepare normalized export previews for invoice, vendor, and payment records using generic, SAP-style, or Oracle-style target labels. It does **not** establish live connectivity or send records to either ERP system.

Endpoints:
- `GET /api/integrations/erp/status`
- `POST /api/integrations/erp/export-preview` with `{"target":"sap","data_type":"invoices"}` (or `oracle` / `generic`; `invoices` / `vendors` / `payments`)
- `GET /api/integrations/erp/history`

## Existing core workflow

- Vendor creation and demo approval
- Invoice creation with duplicate-reference blocking and PO amount mismatch hold
- Explicitly confirmed mock payment with vendor approval, destination, duplicate-payment and spend-limit checks
- Payment ledger and audit log
- Recurring bill schedule (does not auto-create or auto-pay invoices)
- Agent request intake, disabled until `AGENT_API_TOKEN` is configured
- Currency-separated report: `GET /api/reports/summary`

## Suggested safe demo flow

1. Create a vendor using a fake destination reference and approve it for demo use.
2. Create a PO and a GRN for that same vendor and currency.
3. Create an invoice with an amount matching the PO and received amount.
4. Run the three-way match. Resolve any failed check; do not force a match.
5. Use the explicit human approval flow only if the invoice is eligible and under the configured limit.
6. Review Notifications, Analytics, Payments, Audit trail, and the ERP export preview.
7. Upload a sample invoice document and compare extracted fields to the original manually.

## Existing database note

The new feature tables are created automatically on startup. Existing tables are not automatically altered; this feature pack deliberately uses separate new tables for PO/GRN, documents, notifications, match history, and ERP export history. Back up `ap_agent.db` before testing. Do not delete a database if it contains records you need.

## Before any real deployment

Add verified SSO/OIDC authentication and role-based authorization, maker-checker separation, MFA, security review, database migrations/backups, audit integrity protections, safe file scanning, retention rules, rate limits, provider reconciliation, and tested notification delivery. Configure real ERP adapters only after credential management, environment isolation, audit logging, and integration tests are approved. Real payment integration is intentionally out of scope and must remain disabled until independently reviewed.

## 6. Conversational AP Assistant

A floating **Ask AP Assistant** chat is available from every dashboard view. The local retrieval assistant can:
- Explain the dashboard panels and the purpose of each workflow.
- Answer questions using the full `docs/accounts-payable-agent.md` specification, including payment safety, idempotency, approvals, spend limits, payment rails, recurring bills, audit trails, and AP summaries.
- Show a live snapshot of vendor, invoice, pending-review, and simulated payment records from the same local database.
- Clearly state that payments are mock-only and SAP/Oracle export features are previews, not live integrations.

The assistant uses local section retrieval from the MD file and live database queries; it does not require an API key and is not an external LLM. Answers are grounded in matching source sections and may return several related sections for broad questions.

Endpoint: `POST /api/chat` with JSON `{"message":"What are the payment rails?"}`.
