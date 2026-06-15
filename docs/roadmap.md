# Roadmap for n8n Helpdesk Casus

## Exercise Overview
Create an n8n workflow in shadow mode for helpdesk ticket triage and Pareto analysis using demo-data and config files provided in the casus_data folder.

## Functional Requirements (P1-P8)
- **P1 Intake**: Read tickets.csv and normalize to one schema
- **P2 Redaction**: Redact PII-like patterns (email/phone/BSN-like strings) before logging/storing
- **P3 Triage**: For each ticket produce at least:
  - category (exact 1; matches categories.json)
  - confidence (0-1 or low/med/high)
  - route_to (optional; matches queues.json)
  - route_reason (1-2 sentences)
- **P4 Concept answer**: Only for route_to != null: draft_reply (short, polite, refers to loket)
- **P5 Pareto**: Per week (or per batch) top categories with counts and %
- **P6 Trend/drift (simple)**: Compare last period with previous period and signal risers/fallers
- **P7 Output**: Write reports/latest.json (or .md) with Pareto + trends, and reports/routed_tickets.json with tickets that need to go to another loket
- **P8 Observability**: Each run gets run_id, log counts: processed, errors, routed, per category

## Non-functional Requirements (S1-S4)
- **S1 Data-minimalization**: In outputs/logs no full ticket bodies stored; only short summary or hash
- **S2 Idempotency**: Re-run of same dataset creates no duplicate outputs (overwrite per run or dedupe on id)
- **S3 Reproducibility**: One n8n "Execute workflow" should deliver the run with same result (within reasonable variation if model stochastic)
- **S4 Config-driven**: Categories/loketten adjustable via files, not by modifying node code

## Deliverables
1. docs/requirements.md (max 1 page)
   - Problem definition (short)
   - 5-10 FR (in your words)
   - 5-10 NFR (focus: security/privacy/audit/observability)
   - 10 open questions for the customer
   - 5 success metrics (how do we measure that this works?)
2. n8n/workflow-export.json
3. reports/ with example outputs of at least 1 run
4. README.md with:
   - How to run it
   - How to read input (which route chosen)
   - Which choices you made (security/logging/evaluation)

## Implementation Roadmap

### Day 1: Setup and Requirements Analysis
- [x] Access n8n editor via https://ai.educom.nu/t/<HTTP_PORT>/ (HTTP-port = SSH-port + 1000)
- [x] Read and understand the PDF exercise requirements
- [x] Review demo-data files: tickets.csv, license_requests.csv, training_completion.csv
- [x] Review config files: categories.json, queues.json
- [x] Choose input route: Route 1 (git/HTTP download) or Route 2 (container files via SSH)
- [x] Create docs/requirements.md with problem definition, FR/NFR, open questions, success metrics
- [x] Determine output format

### Day 2: Intake, Redaction, and Triage Implementation
- [x] Implement P1 Intake: Create n8n workflow nodes to read tickets.csv
- [ ] Implement P2 Redaction: Add PII redaction nodes before logging/storing
- [ ] Implement P3 Triage: Create logic to categorize tickets using categories.json
- [ ] Implement confidence scoring mechanism (0-1 or low/med/high)
- [ ] Implement route_to assignment using queues.json (optional)
- [ ] Generate route_reason (1-2 sentences)
- [ ] Implement P4 Concept answer: Generate draft_reply for routed tickets
- [ ] Set up output writing for processed tickets

### Day 3: Pareto Analysis and Reporting
- [ ] Implement P5 Pareto: Calculate top categories with counts and percentages
- [ ] Implement P6 Trend/drift: Compare periods and signal risers/fallers
- [ ] Implement P7 Output: Create reports/latest.json with Pareto + trends
- [ ] Create reports/routed_tickets.json for tickets routed to other loket
- [ ] Implement P8 Observability: Add run_id generation and logging of counts
- [ ] Set up logging for processed, errors, routed tickets per category

### Day 4: Quality Checks and Optimization
- [ ] Implement S1 Data-minimalization: Ensure no full ticket bodies in outputs/logs
- [ ] Implement S2 Idempotency: Ensure no duplicate outputs on re-run
- [ ] Implement S3 Reproducibility: Ensure workflow produces consistent results
- [ ] Implement S4 Config-driven: Ensure categories/loketten adjustable via config files
- [ ] Add quality checks and sampling validation
- [ ] Test workflow with demo-data

### Day 5: Documentation and Finalization
- [ ] Create README.md with instructions, choices, and evaluation notes
- [ ] Export n8n workflow to workflow-export.json
- [ ] Run final demonstration scenario
- [ ] Prepare live demo: Show run_id, 3 tickets (clear category, misroute with draft_reply, vague with low confidence), Pareto + trend conclusion, redaction demonstration
- [ ] Create "v2 direction to customer" reflection
- [ ] Verify all deliverables are complete

## Success Criteria
- Workflow runs successfully in n8n editor
- All functional requirements (P1-P8) implemented
- All non-functional requirements (S1-S4) satisfied
- Deliverables completed and stored in appropriate locations
- Live demo successfully demonstrates key features
