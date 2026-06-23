# n8n Helpdesk Casus — Shadow-Mode Ticket Triage & Pareto Analysis

A fully automated n8n workflow for helpdesk ticket triage, PII redaction, Pareto reporting, and trend analysis — running in shadow mode on demo data.

**Current version:** v1.1 — 19 nodes, 150 demo tickets, Markdown reports + interactive dashboard

## Quick Start

### Import the Workflow
1. Open the n8n editor (e.g., `https://ai.educom.nu/t/<HTTP_PORT>/`).
2. Click **Import → Import from File** and select `n8n/workflow-export.json`.
3. Configure the **GitHub credential** on all three "Edit a file" nodes (required for report persistence).
4. Press **Execute Workflow** to run.

### Input Data
The workflow fetches all input from GitHub raw URLs — no local files needed:
| Resource | URL |
|----------|-----|
| Tickets CSV | `https://raw.githubusercontent.com/.../main/casus_data/demo-data/tickets.csv` |
| Categories config | `https://raw.githubusercontent.com/.../main/casus_data/config/categories.json` |
| Queues config | `https://raw.githubusercontent.com/.../main/casus_data/config/queues.json` |

Edit the HTTP Request node URLs if you fork the repo.

### Output
Reports are written to GitHub via `Edit a file` nodes:
- **`reports/latest.json`** — Full observability report (run_id, counts, Pareto analysis, trends)
- **`reports/routed_tickets.json`** — Tickets routed to specialist teams with draft replies
- **`reports/report.md`** — Human-readable Markdown report (Pareto table + trends + category breakdown)

### Dashboard
Open `docs/dashboard.html` in any browser for an interactive visualization:
- Pareto bar charts with category colors and percentages
- Trends table with colored riser/faller/stable indicators
- Routed tickets table (up to 30 tickets with full detail)
- Category breakdown sorted by count

The dashboard fetches data directly from the JSON reports on GitHub — no server needed.

---

## Architecture (19 nodes)

```
Start
 ├─ HTTP - tickets.csv → Parse CSV → P2 Redact PII ──┐
 ├─ HTTP - categories.json ───────────────────────────┤
 └─ HTTP - queues.json ───────────────────────────────┤
                                                        ↓
                                                     Merge (chooseBranch, 3 inputs)
                                                        │
                                                  P3 & P4 - Triage & Draft Reply
                                                    ┌──┼──────┐
                                                    │  │      │
                                     P5 Group → Sort → Pct   │
                                       ↓                    │
                                     P6 Trends              │
                                       ↓                    │
                                P7 & P8 Report ──────┬──────┤ P7 Routed Tickets
                                       ↓             │      ↓      ↓
                                 Edit a file1     Format     Aggregate
                              (reports/latest.json) Markdown      ↓
                                       ↓             ↓      Edit a file
                                 Edit a file   (reports/   (reports/
                                 (report.md)    report.md)  routed_tickets.json)
```

### Functional Coverage

| Requirement | Node(s) | Description |
|-------------|---------|-------------|
| **P1 Intake** | HTTP Request + Parse CSV | Fetches `tickets.csv`, normalizes to schema |
| **P2 Redaction** | P2 - Redact PII | Redacts email, phone, BSN patterns; converts Excel serial dates |
| **P3 Triage** | P3 & P4 | Categorizes each ticket + assigns confidence (low/med/high) |
| **P4 Concept Reply** | P3 & P4 | Generates Dutch draft reply for routed tickets |
| **P5 Pareto** | Group → Sort → Percentages | Category counts, sorted desc, percentage calculation |
| **P6 Trends** | P6 - Trend & Drift | Chronological midpoint split; risers/fallers/stable signals |
| **P7 Output** | Compile + Format + Edit | `latest.json` (Pareto + trends) + `routed_tickets.json` + `report.md` |
| **P8 Observability** | P2 + P7&P8 | Deterministic run_id, processed/routed/error counts per category |

---

## Design Choices

### Security & Privacy (S1 — Data-minimalization)
- Full ticket `body` is **never stored** in outputs or logs.
- `routed_tickets.json` includes only a **16-char FNV-1a hash** of each body (`body_fnva_hash`).
- PII patterns (email, phone, BSN) are redacted **before** any downstream processing via regex replacements.
- FNV-1a is implemented in pure JavaScript (no `require('crypto')` needed — works in n8n sandbox).

### Idempotency (S2)
- `run_id` is a **deterministic FNV-1a hash** of all sorted ticket IDs + count.
- Same dataset → same run_id → GitHub Edit overwrites → no duplicates on re-run.

### Reproducibility (S3)
- Report `timestamp` records the **workflow execution time** (`Date.now()`) for actionable reporting.
- All triage is keyword-based (deterministic — no LLM/API variance).
- Combined with deterministic run_id, identical datasets produce **identical reports** (trends, Pareto, routing) except for the runtime timestamp.

### Config-Driven (S4)
- Keywords come from `categories.json` (fetched at runtime).
- Queue validation uses `queues.json` (fetched at runtime).
- Only an 11-line `routingMap` in P3 code maps category→queue — the sole code-resident configuration.
- Adding a category: just update `categories.json` + one line in `routingMap`.

### Evaluation
- **Confidence scoring**: keyword match count (≥3 = high, 1-2 = medium, 0 = low → CAT_OTHER).
- **Triage fallback**: CAT_OTHER routes to Q_IT_SERVICE_DESK.
- **Draft replies** only generated for non-default-queue tickets.

---

## File Reference

| File | Purpose |
|------|---------|
| `n8n/workflow-export.json` | Exported n8n workflow (import ready, 19 nodes) |
| `docs/requirements.md` | Requirements document (FR/NFR/open questions/metrics) |
| `docs/roadmap.md` | Implementation roadmap with task tracking |
| `docs/demo_scenario.md` | Live demo walkthrough with 3 showcase tickets |
| `docs/dashboard.html` | Interactive Chart.js dashboard (open in browser) |
| `docs/v2_direction.md` | Reflection on v2 improvements (LLM, webhooks, dashboards) |
| `casus_data/config/categories.json` | 11 categories with id, name, description, keywords |
| `casus_data/config/queues.json` | 8 queues with id, name, description |
| `casus_data/demo-data/tickets.csv` | 150 demo tickets (T-0001 to T-0150, Apr-May 2026) |
| `reports/latest.json` | Latest observability report (generated by workflow) |
| `reports/report.md` | Latest Markdown report (generated by workflow) |
| `reports/routed_tickets.json` | Latest routed tickets list (generated by workflow) |
| `agent_memory/agent_config.json` | Agent configuration and learnt patterns |

---

## Tech Stack

- **n8n** v2.23.4 (Code nodes, HTTP Request, Spreadsheet File, itemLists, Sort, Set, Merge, Aggregate, GitHub)
- **Pure JavaScript** — no external dependencies in sandbox
- **GitHub API** — report persistence via `Edit a file` operation
- **Chart.js** — used by the interactive dashboard (loaded from CDN)

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| v1.0 | 2026-06-18 | Initial workflow (16 nodes, 120 tickets, JSON reports) |
| v1.1 | 2026-06-22 | Added 30 tickets (150 total), Markdown report, interactive dashboard |