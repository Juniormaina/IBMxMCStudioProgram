# TrustPay — Fintech · Uganda — Launch Checklist & Guidelines

> Prepared for the IBM Dev Day whitehacking session. Pulls sector guidance from the **Sita Sector Live files** (fintech regulatory-reporting playbook) and country parameters from the **[Country dossier: Uganda](../africa-regulatory/uganda.md)**. Treat every figure as a planning baseline — verify with the named authority before acting. Technology suggestions are recommendations, not mandates.

## 1. What TrustPay is (working frame)

Assuming **TrustPay** is a Ugandan digital payments / lending fintech ("trust + pay" = payments and/or credit). The Sita fintech playbook assumes a licensed digital lender / payment platform that submits periodic returns to the central bank from a transaction-ledger pipeline (extract → master → transform → artifact).

**Reference pipeline** (from `fintech/pipelines/cbk_report_pipeline.js`): transaction/loan ledger → extract → dedupe → transformation rules → report artifact → audit log.

---

## 2. Country & sector fundamentals — Uganda

| Item | Value | Verify with |
|---|---|---|
| Central bank | Bank of Uganda | BoU licensing |
| Financial regulation | BoU licensing for payments/lending; AML/CFT | BoU |
| Data protection | Personal Data Protection Office (PDPO, under NITA-U) | PDPO / NITA-U |
| Blocs | AU; EAC; COMESA | — |
| Corporate income tax | 30% | Uganda Revenue Authority (URA) |
| VAT / sales tax | 18% | URA |
| Official / business language | English, Swahili (also Luganda, Runyankole, Acholi) | — |
| Investment promotion | Uganda Investment Authority (UIA); special economic zones; priority-sector incentives | UIA |

---

## 3. Data model (build toward the fintech playbook)

```
Account / Ledger (payments or loans)
  - account_id (PK)
  - customer_id (FK → Customer)
  - product_type: [personal, business, merchant]
  - principal / balance: DECIMAL
  - interest_rate: DECIMAL (where lending)
  - open_date: DATE
  - status: [active, defaulted, closed]

Customer
  - customer_id (PK)
  - id_number: VARCHAR (national ID)
  - kyc_tier: [basic, enhanced]
  - risk_score: INTEGER
  - created_at: TIMESTAMP

Transaction
  - transaction_id (PK)
  - account_id (FK → Account/Ledger)
  - amount: DECIMAL
  - payment_date: DATE
  - channel: [mtn_momo, airtel_money, bank, ussd]
  - direction: [debit, credit]

RegulatoryReport
  - report_id (PK)
  - regulator: [BoU, ...]           ← sub local regulator names
  - period: VARCHAR
  - generated_at: TIMESTAMP
  - status: [draft, submitted, accepted]
  - file_path: VARCHAR
```

---

## 4. Data flow & architecture

```
[Core / Payments DB]
    → Extract + validate
    → Master customer registry (dedupe on national ID / phone)
    → Transformation (apply BoU regulator schema rules)
    → [RegulatoryReport table]
    → Auto report generation + dispatch
    → Audit log entry
```

**Reference implementation layers** (from `fintech/`):
- `pipelines/*_report_pipeline.js` — pipeline config + transform
- `pipelines/transform.js` — schema mapping internal → regulator format
- `pipelines/report_scheduler.js` — cron trigger for monthly/quarterly runs
- `api/routes/reports.js` — `GET /reports`, `POST /reports/generate`
- `api/routes/compliance.js` — `POST /compliance/submit`
- `api/services/regulatoryEngine.js` — core transformation logic
- `api/services/mdmClient.js` — MDM API wrapper (identity resolution)
- `tests/*_pipeline.test.js`

---

## 5. Technology build reference (recommendations — pick the fit)

| Layer | Recommended (open, low-friction) | IBM option (sugg.) | Notes |
|---|---|---|---|
| Ingestion / ETL | Airbyte, dbt, Apache NiFi, cron + SQL | IBM DataStage | Repeatable, scheduled; not one-off scripts |
| Master data / dedupe | Postgres + fuzzy-match (pg_trgm), your own rules | IBM MDM | Dedupe customers across products/channels |
| Rules / compliance | Open-source rules engine (Drools), code | IBM OpenPages | Rule library per regulator |
| Analytics / scoring | Scikit-learn, XGBoost, FastAPI | IBM Watson Studio | Risk scoring, anomaly, next-best-action |
| Orchestration | Airflow, Prefect, n8n | IBM Watson Orchestrate | Alerts, dispatch, workflows |
| Reports / artifacts | ReportLab, PDFKit, templating | Watson Studio / Cognos | Generate + auto-submit |
| Sandbox / deploy | Docker + CI (GitHub Actions) | IBM TechZone / CP4D | Prototyping + deployment |
| Mobile-money | MTN MoMo / Airtel Money APIs | IBM API Connect (gateway) | Payment rails in Uganda are mobile-money heavy |

> Use mock-first development (`SITA_MOCK_MODE=1`) and only add real credentials (IBM or otherwise) in a protected secret store.

---

## 6. Regulatory & licensing checklist

- [ ] Confirm whether TrustPay is a **payments**, **lender**, or **both** platform — each attracts a different Bank of Uganda licence and capital threshold.
- [ ] Obtain **Bank of Uganda** licensing for the applicable activity before Go-Live.
- [ ] Enrol with the **Personal Data Protection Office (PDPO / NITA-U)** and register as a data processor/controller.
- [ ] Implement **AML/CFT** controls — KYC tiers, transaction monitoring, suspicious-transaction reporting.
- [ ] Put in place **consumer-protection rules** — transparent pricing, caps, fair-debt-collection.
- [ ] Confirm **fee / rate disclosure** obligations and any **interest-cap** rules in force.
- [ ] Document a **complaints and redress** channel consumers can actually use.
- [ ] Plan the **regulatory-reporting cadence**: periodic returns — automate, don't hand-roll in Excel.

## 7. Product, data & integration checklist

- [ ] Build the **ledger** model: account, customer, transaction, report tables (§3 schema).
- [ ] Map the **data flow**: extract → master (dedupe on national ID / phone) → regulator-style transform → artifact.
- [ ] Decide **mock-first** strategy for all third-party integrations (incl. MTN MoMo / Airtel Money); promote with real credentials later.
- [ ] Wire **scheduled, repeatable ingestion** (ETL) — not one-off scripts.
- [ ] Use **dedupe / master record** to avoid duplicate or mis-matched customer records.
- [ ] Define KYC tiers and downstream risk scoring on the customer record.
- [ ] Set up **scheduled jobs** with an **audit log** of every generated report.

## 8. Security & data-protection checklist

- [ ] Keep **API keys and credentials out of source control** — use `.env` + a `.gitignore` entry (the repo's `.gitignore` already excludes `.env*`, `credentials.txt`, `api-keys.txt`, `personal-info.md`). Never commit these.
- [ ] Rotate keys in **mock mode first**; only introduce real credentials in a protected secret store.
- [ ] Apply **least-privilege roles** for anyone touching the reporting pipeline.
- [ ] Encrypt **PII at rest and in transit**; pseudonymise where the schema allows.
- [ ] Log every **report generation and submission** for audit.
- [ ] Keep records of **who/what/when** accessed customer data (PDPO compliance).

## 9. Tax & finance checklist

- [ ] Register for **CIT** (30%) and **VAT** (18%).
- [ ] Plan **withholding tax** treatment on dividends/interest/royalties (commonly 5–15%, per treaty).
- [ ] Keep **transfer-pricing documentation** if part of a wider group.
- [ ] Check **UIA special-zone / priority-sector** incentives before structuring.
- [ ] Confirm whether **digital-service / telecom levy** applies to payment flows in Uganda.

## 10. Expansion notes (when scaling)

- Uganda sits in **EAC** (careful: Kenya's EACL monetary union) and **COMESA** — factor regional data-flow and licensing harmonisation; note EAC rules on payments and money transfer.
- Reuse other [Africa Regulatory Dossiers](../africa-regulatory/README.md) when TrustPay expands; each dossier carries the same sector map for its country.

---

**Status:** planning reference, verify before acting. Nothing here is legal, tax or financial advice — engage local counsel (Aikya) before commitment.
