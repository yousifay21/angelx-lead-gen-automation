# Lead Generation Automation — Make.com + AI

Practice rebuild of an AI-powered lead generation workflow, based on a system shared by AngelX AI as a training exercise. Built to understand real-world Make.com + AI automation architecture: data flow, conditional routing, and how to make a workflow client/production-ready.

## What it does

Automates the process of finding local businesses, qualifying them as leads using AI, logging every result, and routing qualified vs. unqualified leads down different paths — one toward a structured database (Airtable), the other toward AI-personalized outreach via email.

## Architecture

```
Google Maps (Search for Places)
        │
        ▼
    Iterator  ──────  splits the list of businesses into individual records
        │
        ▼
OpenAI (ChatGPT)  ───  [what this step does: TODO — fill in once you inspect the prompt]
        │
        ▼
Google Sheets (Add a Row)  ───  logs every lead as an audit trail
        │
        ▼
      Router
      /      \
     /        \
Airtable    OpenAI (2nd pass)
(Update      │
 Record)     ▼
          Google Docs (Create a Document)
             │
             ▼
          Gmail (Send Email)
```

## Module breakdown

| Step | Module | Purpose |
|---|---|---|
| 1 | Google Maps – Search for Places | Trigger. Pulls businesses matching a search query/location. |
| 2 | Iterator | Breaks the array of results into individual items for per-lead processing. |
| 3 | OpenAI (ChatGPT) | TODO: describe the prompt — likely lead qualification/scoring or data extraction. |
| 4 | Google Sheets – Add a Row | Logs every processed lead regardless of outcome. |
| 5 | Router | TODO: describe the exact filter condition that splits the two paths. |
| 6a | Airtable – Update a Record | Qualified/simple leads stored in a structured database. |
| 6b | OpenAI (ChatGPT) #2 | TODO: describe — likely drafts personalized outreach copy. |
| 7b | Google Docs – Create a Document | Generates a document from the AI output (proposal/report?). |
| 8b | Gmail – Send an Email | Sends the outreach email to the lead. |

## Router logic

TODO: document the exact condition once inspected — e.g. "if lead has a website AND rating > X, go to Airtable path; otherwise go to AI outreach path."

## Weaknesses identified (in the original workflow)

- [ ] No visible error handling on Google Maps (what happens on 0 results?)
- [ ] No visible error handling on OpenAI steps (malformed/empty AI output not caught)
- [ ] No rate-limiting/throttling for large lead batches hitting OpenAI or Gmail
- [ ] TODO: add more as you inspect prompts and data mappings

## How I'd make this production-ready for a client

TODO — after identifying weaknesses, write 3–5 concrete improvements, e.g.:
- Add error handlers on every API-calling module (Maps, OpenAI, Gmail) with fallback/retry logic
- Validate AI output structure before it hits Google Sheets/Airtable (e.g. JSON schema check)
- Add a filter after the Router to prevent duplicate leads being processed twice
- Add logging/alerting if the scenario fails partway through
- Consider a rate limiter or batch delay for high-volume searches

## Files

- `blueprint.json` — exported Make.com scenario (source of truth for the workflow)
- `screenshots/` — visual reference of the scenario canvas

## Notes

This is a practice/learning rebuild, not a production system. Built independently to understand AI + automation workflow architecture as part of exploring roles in AI automation engineering.
