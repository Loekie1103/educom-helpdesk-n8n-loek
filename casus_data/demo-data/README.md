# Demo data

Deze map bevat **realistische dummy-data** (geen echte persoonsgegevens) om de opdracht `20260506_opdracht_yoran_1week_n8n-ai-helpdesk-triage.md` direct te kunnen uitvoeren.

## Bestanden

- `tickets.csv`
  - Simuleert helpdesk-mails/tickets voor: categorisatie, misroute-detectie, Pareto/trends.
  - Bevat expres een klein percentage **PII-achtige** patronen (fake) om redaction te testen.
- `license_requests.csv`
  - Simuleert aanvragen voor “Copilot for GitHub” licenties.
- `training_completion.csv`
  - Simuleert trainingsregistratie waartegen je een aanvraag kunt toetsen.

## Let op (privacy)

- In `tickets.csv` kunnen strings staan die lijken op e-mail/telefoon/BSN, maar ze zijn **fictief**.
- Gebruik deze data alleen als testinput (shadow mode).

