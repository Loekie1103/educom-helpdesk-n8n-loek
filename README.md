# n8n Helpdesk Casus – Project Overview

## How to Run the Workflow
1. Open the n8n editor (e.g., `https://ai.educom.nu/t/<HTTP_PORT>/`).
2. Click **Import** → **Import from File** and select `n8n/route1_workflow.json`.
3. Press the **Start** button (Manual Trigger) and then **Execute Workflow**.

## How to Read the Input
The workflow fetches the tickets CSV from the following raw GitHub URL:
```
https://raw.githubusercontent.com/Loekie1103/educom-helpdesk-n8n-loek/main/casus_data/demo-data/tickets.csv
```
If you move the file, edit the **HTTP Request** node’s **URL** field accordingly.

## Choices We Have Made
- **Route**: Route 1 – HTTP download of the CSV file.
- **Output format**: Both **JSON** and **Markdown** reports will be generated (JSON for downstream processing, Markdown for quick human review).
