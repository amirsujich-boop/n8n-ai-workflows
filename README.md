# n8n-ai-workflows

Three n8n workflows I use with clients, cleaned up so you can import them as they are. Each one puts Claude in the middle of a boring business process and keeps a human at the end of it.

| Workflow | Trigger | What it does |
|---|---|---|
| [01 · AI proposal generator](workflows/01-ai-proposal-generator.json) | n8n form | Form with client + scope → Claude writes a structured proposal → Gmail sends the draft to you |
| [02 · Inbox triage](workflows/02-inbox-triage.json) | Gmail (every minute) | Classifies each new email (lead / support / invoice / spam), labels it, drafts a reply in the sender's language, pings Slack when urgent |
| [03 · Lead → CRM → follow-up](workflows/03-lead-to-crm-follow-up.json) | Webhook (website form) | Scores the lead 1–10, upserts the contact in HubSpot, logs to a sheet, answers hot leads within minutes, nurtures the rest after 2 days |

## Import

1. In n8n: **Workflows → ⋯ → Import from file**, pick a JSON from `workflows/`.
2. Open each red node and select a credential:
   - **Anthropic** – API key from console.anthropic.com
   - **Gmail** – OAuth2 (Google Cloud project with the Gmail API enabled)
   - **Slack**, **HubSpot**, **Google Sheets** – only where the workflow uses them
3. Replace the placeholders that are specific to your account: Gmail label IDs (`Label_LEAD`, `Label_SUPPORT`), Slack channel names, the Google Sheet document.
4. Run once with **Test workflow**, then switch **Active** on.

Tested on n8n 1.80+ (self-hosted, Docker). The Anthropic node is `@n8n/n8n-nodes-langchain.anthropic`, which ships with n8n; no community nodes needed.

## Design notes

- **JSON out of the model, always.** Where Claude has to decide something (category, score) the prompt asks for JSON only, and a Code node parses it defensively: if parsing fails the item is routed to a safe default instead of crashing the run.
- **Drafts, not sends.** Inbox triage never sends email on its own. It creates a Gmail draft in the thread; you read it and press send. The lead workflow is the one exception, and only for leads that score ≥ 7, with a reply that asks one qualifying question rather than pitching.
- **Cheap model for classification, better model for writing.** Haiku for triage and scoring (fast, a fraction of a cent per run), Sonnet for the proposal text.
- **Idempotent CRM step.** HubSpot upsert by email, so re-running a webhook does not create duplicates.

## Changing the prompts

The prompts live in the Anthropic node's message field. Things worth adapting:

- `01` – the five proposal headings and the tone line ("direct, confident, no filler")
- `02` – the category list; add `partnership` or `hr` if your inbox needs it, and add a matching Switch rule + label
- `03` – the scoring guide; tighten it to your actual ICP (industry, company size, geography)

## Cost

Rough figures from my own usage, Anthropic pricing as of autumn 2026:

- proposal: ~1.5k input / 800 output tokens on Sonnet → under $0.02 per draft
- triage: ~1.2k input / 150 output on Haiku → ~$0.0015 per email, so a 200-email/day inbox is about $9/month
- lead scoring: similar to triage

## License

MIT. Use them, change them, ship them.
