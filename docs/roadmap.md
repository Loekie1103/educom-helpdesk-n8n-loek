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
- [x] Implement P2 Redaction: Add PII redaction nodes before logging/storing (including Excel date formatting)
- [x] Implement P3 Triage: Create logic to categorize tickets using categories.json
- [x] Implement confidence scoring mechanism (0-1 or low/med/high)
- [x] Implement route_to assignment using queues.json (optional)
- [x] Generate route_reason (1-2 sentences)
- [x] Implement P4 Concept answer: Generate draft_reply for routed tickets
- [x] Set up output writing for processed tickets

### Day 3: Pareto Analysis and Reporting
- [x] Implement P5 Pareto: Calculate top categories with counts and percentages (Configured for n8n v2.23.4)
- [x] Implement P6 Trend/drift: Compare periods and signal risers/fallers
- [x] Implement P7 Output: Create reports/latest.json with Pareto + trends (via GitHub API Integration)
- [x] Create reports/routed_tickets.json for tickets routed to other loket (via GitHub API Integration)
- [x] Implement P8 Observability: Add run_id generation and logging of counts (via GitHub API with unique run_ids)
- [x] Set up logging for processed, errors, routed tickets per category
- [x] Fix queue ID mismatches between P3 code and queues.json (Q_ACCESS_MANAGEMENT→Q_IDENTITY_ACCESS, etc.)
- [x] Add missing queue for CAT_M365_COPILOT (Q_M365)
- [x] Add CAT_OTHER keywords from categories.json
- [x] Fix S1 body hash (substring→SHA-256)

### Day 4: Quality Checks and Optimization
- [x] Implement S1 Data-minimalization: Ensure no full ticket bodies in outputs/logs (FNV-1a hash in routed tickets)
- [x] Implement S2 Idempotency: Deterministic run_id from ticket IDs (FNV-1a hash), GitHub Edit overwrites
- [x] Implement S3 Reproducibility: Report timestamp uses max ticket date (deterministic)
- [x] Implement S4 Config-driven: Categories & keywords loaded from GitHub-hosted categories.json, queues resolved via queues.json. Hardcoded fallback if configs fail.
- [ ] Add quality checks and sampling validation
- [ ] Test workflow with demo-data

### Day 5: Documentation and Finalization
- [x] Create README.md with instructions, choices, and evaluation notes
- [x] Export n8n workflow to workflow-export.json
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

---
# Route 1 Implementation Guide: HTTP Request Method for n8n Helpdesk Casus

## Overview
You've chosen **Route 1 (git/HTTP download)** for reading the tickets.csv file in your n8n workflow. This approach is recommended in the exercise as it avoids container file access issues and ensures reproducibility.

## Step 1: Make Your CSV File Accessible via HTTP

You have several options to make `tickets.csv` available via HTTP:

### Option A: GitHub/GitLab Raw URL (Recommended)
1. Create a new repository on GitHub/GitLab
2. Upload `casus_data/demo-data/tickets.csv` to the repository
3. Use the raw file URL:
   - GitHub: `https://raw.githubusercontent.com/yourusername/reponame/main/tickets.csv`
   - GitLab: `https://gitlab.com/yourusername/reponame/-/raw/main/tickets.csv`

### Option B: Local HTTP Server (For Development)
If you want to test locally without uploading to GitHub:

```bash
# In the casus_data/demo-data directory, start a simple HTTP server
cd casus_data/demo-data
python -m http.server 8080
```
Then use URL: `http://localhost:8080/tickets.csv`

### Option C: Use Existing Demo Server
If you have access to a demo server, upload the file there and use the provided URL.

## Step 2: n8n Workflow Setup for Route 1

### Basic Workflow Structure:
```
HTTP Request (GET) → Spreadsheet File → [Your Processing Nodes]
```

### HTTP Request Node Configuration:
1. **Method**: GET
2. **URL**: Your CSV file URL (e.g., `https://raw.githubusercontent.com/.../tickets.csv`)
3. **Response Format**: File
4. **Options**:
   - Set `Send Headers` if needed (for authentication)
   - Set `Ignore SSL Issues` to `true` for self-signed certificates (development only)

### Spreadsheet File Node Configuration:
1. **Binary Property**: `data` (default from HTTP Request)
2. **File Format**: CSV
3. **Options**:
   - **From Binary**: Enabled (this is crucial!)
   - **Include Empty Cells**: `false`
   - **Header Row**: `true` (your CSV has headers)
   - **Delimiter**: `,` (comma)
   - **Encoding**: `UTF-8`

## Step 3: Complete n8n Workflow Example

Here's a more complete example of how your n8n workflow should look:

### Node 1: HTTP Request
```json
{
  "parameters": {
    "url": "https://raw.githubusercontent.com/yourusername/helpdesk-data/main/tickets.csv",
    "method": "GET",
    "responseFormat": "file",
    "options": {
      "ignoreSSLIssues": true
    }
  }
}
```

### Node 2: Spreadsheet File
```json
{
  "parameters": {
    "binaryPropertyName": "data",
    "fileFormat": "csv",
    "options": {
      "fromBinary": true,
      "headerRow": true,
      "delimiter": ",",
      "encoding": "UTF-8"
    }
  }
}
```

### Node 3: Function Node (Optional - Data Validation)
Add a Function node to verify the data structure:
```javascript
// Check if data is properly loaded
const items = $input.all();
const firstItem = items[0].json;

if (!firstItem.id || !firstItem.subject || !firstItem.body) {
  throw new Error('CSV data missing required fields');
}

// Log sample for debugging
console.log('First ticket:', {
  id: firstItem.id,
  subject: firstItem.subject.substring(0, 50) + '...',
  body: firstItem.body.substring(0, 100) + '...'
});

return items;
```

## Step 4: Testing Your Setup

1. **Test HTTP Request**: Execute just the HTTP Request node to ensure it downloads the file
2. **Test CSV Parsing**: Connect HTTP Request → Spreadsheet File and execute to verify parsing
3. **Verify Data**: Check that you get 122 items (tickets) with correct fields:
   - `id`, `created_at`, `channel`, `language`, `requester_type`, `subject`, `body`, `assigned_queue`

## Step 5: Handling Multiple Files (Optional)

If you also want to load the other demo files:
- `license_requests.csv`
- `training_completion.csv`

You can:
1. Create separate HTTP Request → Spreadsheet File chains for each file
2. Or use a single HTTP Request with different URLs in a loop
3. Merge data using "Merge" node if needed

## Step 6: Error Handling

Add error handling nodes:
1. **Error Trigger** node to catch HTTP errors
2. **Set** node to add error context
3. **IF** node to decide whether to continue or stop

Example error handling structure:
```
HTTP Request → On Error → Error Trigger → Set (error context) → [Handle Error]
HTTP Request → On Success → Spreadsheet File → [Continue Processing]
```

## Step 7: Next Steps After Route 1 Implementation

Once you have the CSV data loaded in n8n:

1. **Implement P2 Redaction**: Add nodes to redact PII (email, phone, BSN-like patterns)
2. **Implement P3 Triage**: Create logic to categorize tickets using `categories.json`
3. **Add Confidence Scoring**: Implement scoring mechanism (0-1 or low/med/high)
4. **Route Assignment**: Use `queues.json` for optional routing
5. **Generate Responses**: Create `draft_reply` for routed tickets

## Common Issues & Solutions

### Issue 1: HTTP Request returns HTML instead of CSV
- **Solution**: Ensure you're using the RAW file URL, not the GitHub page URL

### Issue 2: Spreadsheet File node shows "No binary data found"
- **Solution**: Check that HTTP Request Response Format is set to "File", not "JSON" or "Auto"

### Issue 3: CSV parsing errors
- **Solution**: Verify delimiter matches your CSV (comma), and encoding is UTF-8

### Issue 4: Rate limiting on GitHub
- **Solution**: For production, consider:
  - Using GitLab or another host
  - Implementing caching in n8n
  - Using local file method (Route 2) for development

## Best Practices for Route 1

1. **Version Control**: Keep your CSV files in git for reproducibility
2. **URL Management**: Store URLs as n8n variables or credentials for easy updates
3. **Caching**: Consider adding cache logic if you run the workflow frequently
4. **Monitoring**: Add logging to track download success/failure rates
5. **Fallback**: Consider implementing Route 2 as a fallback option

## Integration with Your Current Progress

Since you've completed the first 5 steps of Day 1, your next actions are:

1. **Implement Route 1** using the guide above
2. **Determine output format for reports** (JSON recommended for n8n compatibility)
3. **Move to Day 2 tasks**: Implement P1 Intake (which you're doing now), P2 Redaction, P3 Triage, etc.

## Example Workflow Export Snippet

Here's a minimal JSON snippet for your n8n workflow export:
```json
{
  "nodes": [
    {
      "name": "HTTP Request - Get Tickets CSV",
      "type": "n8n-nodes-base.httpRequest",
      "position": [250, 300],
      "parameters": {
        "url": "={{$vars.CSV_URL}}",
        "method": "GET",
        "responseFormat": "file"
      }
    },
    {
      "name": "Parse CSV",
      "type": "n8n-nodes-base.spreadsheetFile",
      "position": [450, 300],
      "parameters": {
        "binaryPropertyName": "data",
        "fileFormat": "csv",
        "options": {
          "fromBinary": true,
          "headerRow": true
        }
      }
    }
  ]
}
```

Remember to test each node individually before connecting them, and verify the data flows correctly through your workflow!
