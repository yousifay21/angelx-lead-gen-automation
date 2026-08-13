# Lead Generation Automation — Make.com + AI

Practice rebuild of an AI-powered lead generation workflow, based on a system shared by AngelX AI as a training exercise. Built independently in Make.com to understand real-world automation architecture: data flow between modules, conditional routing, AI prompt design, and what it actually takes to make a workflow client/production-ready.

## What it does

Automates the process of finding local businesses via Google Maps, qualifying them as leads using AI, logging every result to a spreadsheet, and routing leads down two branches — one toward a structured database (Airtable), the other toward AI-drafted, personalized outreach via email.

## Architecture (as built)

```
Google Maps (Search for Places)
        │
        ▼
    Iterator  ──────  splits the array of businesses into individual records
        │
        ▼
OpenAI #1 (gpt-3.5-turbo)  ───  scores each lead 1-10, gives a reason, guesses if they have a website
        │
        ▼
Google Sheets (Add a Row)  ───  logs every lead as an audit trail
        │
        ▼
      Router
      /      \
     /        \
Airtable    OpenAI #2 (gpt-3.5-turbo)
(Create      │   drafts a short personalized outreach email
 Record)     ▼
          Google Docs (Create a Document)
             │   saves a copy of the sent email as a record
             ▼
          Gmail (Send Email)
```

## Module breakdown

| Step | Module | Purpose |
|---|---|---|
| 1 | Google Maps – Search for Places | Trigger. Pulls businesses matching a text search query (e.g. "Pet Insurance Auckland"). |
| 2 | Iterator | Breaks the `[bundle]` array of places from Google Maps into individual items, so everything downstream runs once per lead. |
| 3 | OpenAI #1 – Create Chat Completion | Takes one business's Name, Address, Rating, and Total Ratings and asks the model to: (1) score lead quality 1-10, (2) give a one-sentence reason, (3) guess whether the business has a website. |
| 4 | Google Sheets – Add a Row | Logs Business Name, Address, Rating, and the raw AI output into a spreadsheet as an audit trail. |
| 5 | Router | Splits into two branches. **Not currently filtered** — see Router logic below. |
| 6a | Airtable – Create a Record | Writes the same lead data into an Airtable base (`Lead Gen Leads` → `Table 1`). |
| 6b | OpenAI #2 – Create Chat Completion | Drafts a short (3-4 sentence) personalized outreach email with subject line, based on the same lead data. |
| 7b | Google Docs – Create a Document | Saves a copy of the AI-drafted email as a titled Google Doc, one per lead. |
| 8b | Gmail – Send an Email | Sends the AI-drafted email. **Hardcoded to a test address during development** — see Weaknesses. |

## Router logic

**Not implemented.** Both branches are currently open with no filter condition — every lead goes down both paths simultaneously rather than being routed based on lead quality. In the original AngelX AI workflow shown to me, the router presumably splits qualified vs. unqualified leads (e.g. by score or rating threshold), but no filter was visible in the screenshot I was given, and I did not infer one for this rebuild.

**Fix for production:** add a filter on one branch, e.g. `AI Score >= 7 → Airtable path`, `AI Score < 7 → AI outreach path`, or split on Google rating instead once the AI output is returned in structured (JSON) format rather than free text.

## Weaknesses identified

Found through hands-on rebuilding and testing, not just reading the diagram:

- **No email address in the data.** Google Maps' Search for Places output has no email field — only phone, website, address, and ratings. The Gmail step as designed has no real recipient to send to for an actual lead. A production version needs an email-finding step (website scrape, Hunter.io-style API, or a manual enrichment step) before Gmail can be used for real outreach.
- **No error handling anywhere in the scenario.** Confirmed directly: when my OpenAI account ran out of API credit mid-test, the OpenAI module failed with a `429 RateLimitError`, but nothing downstream was notified or stopped cleanly — the Iterator's bundles just came through empty, and Google Sheets/Airtable/Docs/Gmail silently processed blank data with no error surfaced until I manually checked each module's execution log. A production system needs error handlers on every external API call (Google Maps, OpenAI, Gmail) with at minimum a notification/alert on failure, ideally a retry.
- **Single AI response gets crammed into multiple fields.** My first OpenAI prompt asks for three things (score, reason, website guess) but returns them as one block of free text. Right now that entire block would need to be duplicated into three separate spreadsheet/Airtable columns, which is messy and not really usable as structured data. Fix: prompt the model to return valid JSON (e.g. `{"score": 8, "reason": "...", "has_website": true}`) and parse it before writing to columns.
- **Unclear purpose of the Google Docs step.** A Doc isn't needed to send an email — Gmail can send text directly. My best interpretation is that it's meant as a saved record/archive of what was sent, separate from the act of sending. Worth clarifying directly with whoever designed the original, since the diagram alone doesn't make the reasoning obvious.
- **No de-duplication.** If the same scenario runs on a schedule (e.g. every 15 minutes, which Make.com defaults to), the same businesses from a repeated Google Maps search would be processed and emailed again. A production version needs to check Airtable/Sheets for an existing record (by Place ID) before creating a new one or sending outreach again.
- **No rate limiting for larger batches.** Tested with a limit of 3 results. At real scale (e.g. limit of 50+), this would fire 50+ OpenAI calls and 50+ Gmail sends in quick succession with no throttling, risking API rate limits or looking like spam to Gmail's sending limits.

## How I'd make this production-ready for a client

1. Add error handlers on every external API module (Maps, OpenAI x2, Gmail, Airtable, Sheets, Docs) with at least a notification on failure and a retry where sensible.
2. Switch both OpenAI prompts to request structured JSON output, and parse it before writing to Sheets/Airtable, so each field lands in its own column cleanly.
3. Add a real filter condition on the Router based on lead score or rating, so leads are actually triaged rather than both branches firing for everyone.
4. Add a de-duplication check (by Google Place ID) before creating new Airtable records or sending outreach, so repeated scenario runs don't re-contact the same business.
5. Add an email-enrichment step before Gmail, since Google Maps data alone doesn't include a usable contact email.
6. Add a rate limiter / delay between iterations for larger batches, to stay within API limits and avoid triggering spam flags on the sending account.

## Debugging notes (real issues hit while building)

- Google Cloud requires an active billing method to enable the Places API, even though usage stays within the $200/month free credit — the "Enable APIs" onboarding shortcut failed silently until billing was added.
- The Iterator's `Array` field mapping was lost after reconnecting the Google Maps API key partway through the build, which caused every downstream module to receive empty data with no error shown, until traced back by checking each module's individual execution output.
- OpenAI's API has no free tier — separate from ChatGPT's web interface — and requires prepaid credit to make any call at all, confirmed via a `429 RateLimitError` when credit ran out mid-test.

## Files

- `screenshots/` — visual reference of the scenario canvas
- `blueprint.json` — not yet exported (pending a fully successful end-to-end run)

## Notes

This is a practice/learning rebuild, not a production system. Built independently as part of an onboarding exercise to understand AI + automation workflow architecture for a role in AI automation engineering.
