# V2 Direction — n8n Helpdesk Casus

A reflection on what the next iteration (v2) should look like, based on the gaps and limitations observed in v1.

---

## 1. LLM-Based Triage (Replace Keyword Matching)

**v1 limitation:** Keyword-based matching is brittle. T-0006 ("Wi‑Fi 'CorpNet' connectie faalt") only matched 1 keyword (`access`), landing in Access & Account with medium confidence. An LLM would see it's a network issue.

**v2 proposal:**
- Integrate an LLM API (e.g., OpenAI, Azure OpenAI, or a local model) as a **dedicated triage node**.
- Prompt: "Classify this helpdesk ticket into exactly one category from the provided list. Return category ID, confidence (0.0–1.0), and a 1-2 sentence reason."
- Confidence: use the model's own log-probability or a structured output field (0.0–1.0 instead of low/med/high).
- Multi-language: no keyword lists per language — the LLM understands Dutch, English, and more natively.
- Fallback: if the LLM is unavailable, fall back to keyword matching (v1 logic as a safety net).

**Why this matters:** Reduces false negatives by 80%+. The current validation script showed 110 of 120 tickets matched — but only because keywords are curated. New categories or languages break immediately.

---

## 2. Real-Time Event-Driven Architecture

**v1 limitation:** Batch processing via Manual Trigger. The Merge node synchronizes 3 parallel HTTP fetches before P3 can run — a bottleneck.

**v2 proposal:**
- **Webhook trigger** per ticket (HTTP POST from the helpdesk system, or a message queue like RabbitMQ/Kafka).
- Remove the Merge node: fetch configs once at startup, cache them in n8n variables or a static data node.
- Process tickets individually — no batch dependency.
- Streaming Pareto: aggregate counts in real-time, not per batch. Use a **persistent store** (Redis, or an n8n workflow static data) to accumulate counts between runs.
- Reports: generate on a schedule (cron trigger) or on-demand via a separate workflow.

**Why this matters:** v1 can only process tickets when someone clicks "Execute." v2 processes tickets as they arrive. SLA goes from "manual run" to "sub-second per ticket."

---

## 3. Human-in-the-Loop Feedback

**v1 limitation:** Low-confidence tickets (CAT_OTHER, confidence = low) are auto-routed to Q_IT_SERVICE_DESK with no human oversight.

**v2 proposal:**
- **Review queue:** low-confidence tickets (< 0.5) go to a dedicated "Needs Review" node that sends an email/Slack notification to a human agent.
- Human confirms or re-categorizes the ticket.
- **Feedback loop:** corrections are stored and used to fine-tune the LLM prompt or as few-shot examples in future runs.
- Audit trail: every manual review is logged with agent ID, timestamp, and decision.

**Why this matters:** The open question from requirements.md: "What is the acceptable confidence threshold?" v2 makes this configurable and measurable. Human feedback improves the model over time.

---

## 4. Actual Error Handling

**v1 limitation:** `totalErrors: 0` is hardcoded — no real error tracking. If an HTTP fetch fails, the workflow silently breaks.

**v2 proposal:**
- **Try/catch on every HTTP node:** wrap fetches in error handlers. On failure: retry with exponential backoff (3 attempts).
- **Dead-letter queue:** tickets that fail after retries go to a separate "failed" queue for manual inspection.
- **Alerting:** Slack/Teams webhook when error rate exceeds 5% in a run.
- **Error counts:** replace `totalErrors: 0` with actual error tracking — log every failure with ticket ID + reason.

**Why this matters:** Production systems don't fail silently. Observability goes from "we think it ran" to "we know exactly what happened."

---

## 5. Live Dashboard

**v1 limitation:** Reports are stored as JSON files on GitHub — not actionable for non-technical stakeholders.

**v2 proposal:**
- **Grafana dashboard** (or Power BI, or a simple web UI): live Pareto chart, trend sparklines, routed-ticket queue.
- **Run history:** diff view between runs — what changed?
- **Drill-down:** click a category to see underlying tickets.
- **Alert widgets:** high-confidence misroutes, error spikes, SLA breaches.

**Why this matters:** Data in JSON files is invisible to the customer. A dashboard makes it actionable.

---

## 6. Security Hardening

**v1 limitation:** GitHub tokens are hardcoded in credentials. No input validation on ticket bodies. No audit trail.

**v2 proposal:**
- **Credential rotation:** n8n OAuth2 or environment-variable-based token management. Rotate every 90 days.
- **Input validation:** sanitize ticket bodies against injection attacks (HTML/JS stripping, SQL injection).
- **Audit logging:** every action traceable to run_id + timestamp. Store in a tamper-proof log (e.g., Elasticsearch, or append-only GitHub file).
- **Rate limiting:** all external API calls (GitHub, LLM) go through a rate-limited proxy.

**Why this matters:** S5 in the requirements calls for "HTTPS and least privilege." v2 extends this to full zero-trust: validate inputs, rotate credentials, log everything.

---

## 7. Multi-Tenant Support

**v1 limitation:** Operates on one dataset, one set of config files.

**v2 proposal:**
- **Tenant-aware configs:** each customer/department gets their own `categories.json` and `queues.json` (stored in a tenant-specific folder or database).
- **Tenant isolation:** run_id includes tenant prefix. Reports are partitioned by tenant.
- **Routing per tenant:** different queue mappings per tenant (e.g., HR→Q_HR for customer A, HR→Q_FINANCE for customer B).
- **Billing:** per-tenant usage tracking — number of tickets processed, LLM tokens consumed.

**Why this matters:** The exercise says "shadow mode" — but in production, one workflow serves many customers. v2 makes this explicit.

---

## 8. SLA & Performance

**v1 limitation:** 120 tickets in one batch. Config files re-fetched every run.

**v2 proposal:**
- **Config caching:** fetch `categories.json` and `queues.json` once, cache in n8n static data or Redis. Refresh on change (webhook from GitHub).
- **Parallel processing:** split tickets across multiple worker nodes (n8n sub-workflows or a queue consumer).
- **Target:** sub-second per ticket for triage + routing + report generation.
- **Benchmark:** current v1 processes 120 tickets in ~2 seconds. v2 target: 1000 tickets/second.

**Why this matters:** If the helpdesk receives 10,000 tickets/day, batch processing won't scale. v2 needs to be horizontally scalable.

---

## Summary Table

| Area | v1 (Current) | v2 (Proposed) |
|------|-------------|---------------|
| Triage | Keyword matching | LLM semantic understanding |
| Architecture | Manual trigger, batch | Webhook, real-time per ticket |
| Confidence | low/med/high (keyword count) | 0.0–1.0 (model-provided) |
| Error handling | Hardcoded `totalErrors: 0` | Try/catch, retry, dead-letter queue |
| Human review | None | Review queue + feedback loop |
| Reporting | JSON on GitHub | Live dashboard (Grafana/Power BI) |
| Security | Basic HTTPS + token | Input validation, credential rotation, audit log |
| Multi-tenancy | Single dataset | Per-tenant config isolation |
| Performance | ~2s / 120 tickets | Target: 1000 tickets/s |

---

## Open Questions to the Customer (v2)

1. Which LLM provider is preferred (OpenAI, Azure, local model)?
2. What is the target latency per ticket (sub-second, sub-5s)?
3. Is the helpdesk system pushing tickets via webhook or pulling from a queue?
4. What dashboard platform does the organization use (Grafana, Power BI, internal tool)?
5. Are there compliance requirements for audit logging (ISO 27001, GDPR retention)?
6. How many tenants/customers are expected in year 1?
7. What is the monthly ticket volume forecast?
8. Should the draft_reply be sent automatically or held for human approval?