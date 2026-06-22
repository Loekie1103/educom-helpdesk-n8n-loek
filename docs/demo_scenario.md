# Demo Scenario — n8n Helpdesk Casus

This document walks through a live demonstration of the workflow using 3 specific tickets that showcase the key features.

## Setup

1. Open the n8n editor at `https://ai.educom.nu/t/<HTTP_PORT>/`.
2. Import `n8n/workflow-export.json`.
3. Ensure the GitHub credential is configured on both "Edit a file" nodes.
4. Press **Execute Workflow**.

## Showcase Tickets

### 1. Clear Category — High Confidence (T-0001)

**Subject:** "Kan niet inloggen op laptop na wachtwoord reset"

**What happens:**
| Field | Value |
|-------|-------|
| Category | `CAT_ACCESS_ACCOUNT` |
| Confidence | `high` |
| Keyword matches | 3 (`password`, `wachtwoord`, `mfa`) |
| Route to | `Q_IDENTITY_ACCESS` |
| Draft reply | Generated — referencing specialist team |

**Why this shows:** The ticket clearly falls into one category. 3 keyword matches = high confidence. The draft reply is generated because it's routed to a non-default queue.

**Demo script:** "This ticket about a password reset matched 3 keywords from the Access & Account category: `password`, `wachtwoord`, and `mfa`. Confidence is high, and it's routed to the Identity & Access team with a polite Dutch draft reply."

---

### 2. Edge Case — Routed with Draft Reply (T-0002)

**Subject:** "VPN valt steeds weg na 2 minuten"

**What happens:**
| Field | Value |
|-------|-------|
| Category | `CAT_VPN_REMOTE` |
| Confidence | `high` |
| Keyword matches | 3 (`vpn`, `tunnel`, `timeout`) |
| Route to | `Q_NETWORK` |
| Draft reply | Generated — references specialist team |

**Why this shows:** A clearly categorized ticket with a draft reply. Demonstrates the full P4 cycle: triage → route → draft_reply.

**Demo script:** "VPN issues matched 3 keywords — `vpn`, `tunnel`, and `timeout` — routing it to the Network Team. A draft reply is automatically generated. Notice the body is replaced with an FNV-1a hash in the routed tickets report — no full ticket bodies in output."

---

### 3. Vague Ticket — Low Confidence (T-0006)

**Subject:** "Wi‑Fi 'CorpNet' connectie faalt"

**What happens:**
| Field | Value |
|-------|-------|
| Category | `CAT_ACCESS_ACCOUNT` |
| Confidence | `medium` |
| Keyword matches | 1 (`access`) |
| Route to | `Q_IDENTITY_ACCESS` |
| Draft reply | Generated |

**Why this shows:** An ambiguous ticket. "Wi-Fi CorpNet" sounds like a network issue, but the single keyword match (`access`) routed it to Identity & Access with medium confidence. This demonstrates the limits of keyword-based triage and why a human review or LLM might improve accuracy in v2.

**Demo script:** "This Wi-Fi ticket only matched 1 keyword — `access` — landing in Access & Account with medium confidence. In a real production system, this might need human review or an LLM for better categorization."

---

## Pareto & Trends

After execution, open `reports/latest.json`:

```json
{
  "run_id": "RUN-XXXX",
  "summary_processed": 120,
  "summary_routed": 110,
  "pareto_analysis": [
    { "category": "CAT_ACCESS_ACCOUNT", "percentage": 20 },
    { "category": "CAT_M365_COPILOT", "percentage": 13 },
    ...
  ],
  "trends": [
    { "category": "CAT_GH_COPILOT", "signal": "faller", "difference": -3 },
    { "category": "CAT_ACCESS_ACCOUNT", "signal": "riser", "difference": 2 },
    ...
  ]
}
```

**Demo script:** "The Pareto shows Access & Account is the top category at 20%. Trends show GitHub Copilot declining (-3, faller) while Access & Account and M365 Copilot are rising (+2 each). The deterministic run_id proves this is reproducible — same dataset, same result."

---

## Redaction & S1 Data-Minimalization

Open `reports/routed_tickets.json` — every ticket has:

```json
{
  "id": "T-0001",
  "subject": "Kan niet inloggen op laptop na wachtwoord reset",
  "category": "CAT_ACCESS_ACCOUNT",
  "body_fnva_hash": "cbf29d610ee57fce",
  "draft_reply": "Beste medewerker..."
}
```

**Demo script:** "Full ticket bodies are never stored. Each routed ticket only carries a 16-character FNV-1a hash. PII like emails, phone numbers, and BSN-like numbers are redacted before processing. This satisfies S1 Data-minimalization and GDPR considerations."

---

## Key Takeaways for the Customer

1. **120 tickets processed in one run** — 110 routed to specialist teams, 10 kept at IT Service Desk.
2. **Deterministic and idempotent** — same dataset always produces the same run_id and output. No duplicates.
3. **Config-driven** — categories and queues are loaded from JSON files at runtime. Only an 11-line routing map is code-resident.
4. **Privacy-preserving** — no full ticket bodies in output, PII redacted, body hashed.
5. **Pareto-optimized** — top categories clearly visible, trends signal rising/falling patterns.