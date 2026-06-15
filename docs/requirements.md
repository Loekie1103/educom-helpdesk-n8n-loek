# Requirements Document for n8n Helpdesk Casus

## Problem Definition
The goal is to build a **shadow‑mode n8n workflow** that automatically processes helpdesk tickets. The workflow must ingest ticket data, perform PII redaction, categorize each ticket, optionally route it to the correct queue, generate a draft reply for mis‑routed tickets, and produce weekly Pareto and trend reports. All processing must be traceable, secure‑by‑default, and reproducible using only the provided demo‑data and configuration files.

## Functional Requirements (FR)
1. **P1 Intake** – Read `tickets.csv` (and optionally other demo files) and normalize to a single schema.
2. **P2 Redaction** – Detect and redact PII‑like patterns (email, phone, BSN‑like strings) before any logging or storage.
3. **P3 Triage** – For each ticket output:
   - `category` (exactly one, matching `categories.json`)
   - `confidence` (0‑1 or low/med/high)
   - `route_to` (optional, matching `queues.json`)
   - `route_reason` (1‑2 sentences)
4. **P4 Concept answer** – When `route_to` is not null, generate a short, polite `draft_reply` that references the appropriate loket.
5. **P5 Pareto** – Produce a weekly (or per‑batch) list of top categories with counts and percentages.
6. **P6 Trend/Drift** – Compare the current period with the previous period and flag rising or falling categories.
7. **P7 Output** – Write:
   - `reports/latest.json` (or `.md`) containing Pareto + trends
   - `reports/routed_tickets.json` with tickets that need to be sent to another loket.
8. **P8 Observability** – Assign a unique `run_id` to each execution and log counts of processed, errored, routed tickets per category.

## Non‑Functional Requirements (NFR)
1. **S1 Data‑minimalisation** – Never store full ticket bodies in logs or outputs; keep only a short summary or a hash.
2. **S2 Idempotency** – Re‑running the workflow on the same dataset must not create duplicate outputs (overwrite or dedupe by ticket ID).
3. **S3 Reproducibility** – A single "Execute workflow" must yield the same results (within reasonable variance for stochastic models).
4. **S4 Config‑driven** – All categories and queues must be configurable via `categories.json` and `queues.json` without code changes.
5. **S5 Security** – All external calls (e.g., HTTP request for CSV) must use HTTPS and respect the principle of least privilege.

## Open Questions for the Customer
1. Which model or rule‑set should be used for the triage classification?
2. What is the acceptable confidence threshold for automatic routing?
3. Should the workflow support multilingual tickets beyond Dutch and English?
4. How often should the Pareto/trend reports be generated (daily, weekly, on‑demand)?
5. Are there any retention policies for the generated reports?
6. Which logging platform should be used for observability (e.g., Elastic, CloudWatch)?
7. Is there a preferred format for the `draft_reply` (plain text, HTML, markdown)?
8. Should the workflow send notifications (e.g., Slack) when a mis‑routed ticket is detected?
9. Are there any rate‑limits or authentication requirements for the GitHub raw CSV URL?
10. What is the expected SLA for processing a batch of tickets?

## Success Metrics
1. **Processing Accuracy** – ≥ 90 % of tickets correctly categorized according to `categories.json`.
2. **Redaction Coverage** – 100 % of PII patterns are redacted before any log entry.
3. **Report Timeliness** – Pareto and trend reports are generated within 2 minutes of workflow execution.
4. **Idempotent Runs** – Re‑executing the same batch produces identical output files (hash comparison).
5. **User Satisfaction** – Stakeholders rate the draft replies as useful in ≥ 80 % of mis‑routed cases.
