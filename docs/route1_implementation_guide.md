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