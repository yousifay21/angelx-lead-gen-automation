# Lead Generation Automation — Make.com + AI

Practice rebuild of an AI-powered lead generation workflow, based on a system shared by AngelX AI as a training exercise. Built independently in Make.com to understand real-world automation architecture — data flow between modules, conditional routing, AI prompt design — and then iterated on to fix real weaknesses found while building and testing it.

## What it does

Automates the process of finding local businesses via Google Maps, scoring them as leads using AI, logging every result to a spreadsheet, and routing them down two branches based on lead quality: strong leads go straight to a structured database (Airtable), while weaker leads trigger an AI-drafted outreach suggestion that gets sent internally for a human to review before any contact is made.

## Architecture (current version)

```
Google Maps (Search for Places)
        │
        ▼
    Iterator  ──────  splits the array of businesses into individual records
        │
        ▼
OpenAI #1 (gpt-3.5-turbo)  ───  returns lead score, reason, and website guess as JSON
        │
        ▼
    JSON (Parse JSON)  ───  parses the AI's JSON into separate fields: score, reason, has_website
        │
        ▼
Google Sheets (Add a Row)  ───  logs every lead as an audit trail, using the parsed fields
        │
        ▼
      Router  ───  filters on score
      /      \
     /        \
Score >= 7   Score < 7
    │            │
    ▼            ▼
Airtable    OpenAI #2 (gpt-3.5-turbo)
(Create      │   drafts a suggested outreach email
 Record)     ▼
          Google Docs (Create a Document)
             │   saves a copy of the AI draft
             ▼
          Gmail (Send Email)
             │   sends an INTERNAL notification (not to the business)
             │   with the lead's details + the AI's draft, for a human to review
```

## Module breakdown

| Step | Module | Purpose |
|---|---|---|
| 1 | Google Maps – Search for Places | Trigger. Pulls businesses matching a text search query (e.g. "Pet Insurance Auckland"). |
| 2 | Iterator | Breaks the `[bundle]` array of places from Google Maps into individual items, so everything downstream runs once per lead. |
| 3 | OpenAI #1 – Create Chat Completion | Takes one business's Name, Address, Rating, and Total Ratings and returns **strict JSON**: `{"score": <1-10>, "reason": "<one sentence>", "has_website": <true/false>}`. |
| 4 | JSON – Parse JSON | Parses the JSON string from OpenAI #1 into three separate, individually-mappable fields: `score` (number), `reason` (text), `has_website` (boolean). |
| 5 | Google Sheets – Add a Row | Logs Business Name, Address, Rating, and the three parsed AI fields into a spreadsheet as an audit trail. |
| 6 | Router | Splits into two branches based on a real filter — see Router logic below. |
| 7a | Airtable – Create a Record | Writes the lead's data into an Airtable base (`Lead Gen Leads` → `Table 1`) for leads scoring 7 or higher. |
| 7b | OpenAI #2 – Create Chat Completion | Drafts a short (3-4 sentence) personalized outreach email with subject line, for leads scoring below 7. |
| 8b | Google Docs – Create a Document | Saves a copy of the AI-drafted email as a titled Google Doc, one per lead. |
| 9b | Gmail – Send an Email | Sends an **internal notification** (to the team, not the business) containing the lead's details, AI score/reasoning, and the AI-drafted outreach copy as a suggestion for a human to review and send manually. |

## Router logic

**Implemented with a real filter**, using the parsed `score` field from the JSON module:

- **Path 1 (Airtable):** `19. Score >= 7` — strong leads get logged straight to the database.
- **Path 2 (AI outreach path):** `19. Score < 7` — weaker leads get an AI-drafted outreach suggestion instead, routed to a human for review rather than sent automatically.

This was originally unfiltered — both branches fired for every lead regardless of quality. Fixed by first restructuring the AI's output as JSON (see below), which made a reliable numeric filter possible.

## Weaknesses found — and what I did about them

### Fixed

- **AI output was one unstructured block of text.** The first OpenAI prompt asked for three things (score, reason, website guess) but returned them as free-form prose, which meant the same messy text was getting duplicated into three separate spreadsheet columns instead of being usable as real data. **Fix:** rewrote the prompt to return strict JSON only, and added a Parse JSON module right after it to split the response into clean `score`, `reason`, and `has_website` fields. Every downstream module (Sheets, Airtable, the Router filter) now uses these clean fields instead of raw text.
- **The Router had no actual filter.** Both branches were wired but open, so every lead flowed down both paths regardless of quality — there was no real triage happening. **Fix:** once `score` existed as a clean number (from the JSON fix above), added a real filter: `score >= 7` to Airtable, `score < 7` to the AI outreach path.
- **The Gmail step had no real recipient.** Google Maps' Search for Places data has no email field — only phone, website, address, and ratings — so the original design (AI drafts an email → send it straight to the business) had no actual address to send to. Hardcoding a test address to make the pipeline runnable masked this rather than solving it. **Fix:** redesigned the branch as an internal notification instead of external outreach — the email now goes to the team with the lead's data, AI score/reasoning, and the AI-drafted copy included as a *suggestion*, for a human to review and send manually if it's a good fit. This also sidesteps the real risk of an automated system cold-emailing scraped contacts with no verification.

### Still open

- **No error handling anywhere in the scenario.** Confirmed directly: when my OpenAI account ran out of API credit mid-test, the OpenAI module failed with a `429 RateLimitError`, but nothing downstream was notified or stopped cleanly — the Iterator's bundles just came through empty, and every module after it silently processed blank data with no error surfaced until I manually checked each module's execution log. A production system needs error handlers on every external API call (Google Maps, OpenAI, Gmail) with at minimum a notification/alert on failure, ideally a retry.
- **No de-duplication.** If the scenario runs on a schedule (Make.com defaults to every 15 minutes), the same businesses from a repeated Google Maps search would be processed and logged again. A production version needs to check Airtable/Sheets for an existing record (by Google Place ID) before creating a new one.
- **No rate limiting for larger batches.** Tested with a limit of 3 results. At real scale (e.g. limit of 50+), this would fire 50+ OpenAI calls in quick succession with no throttling, risking API rate limits.
- **Google Docs step's purpose could be clearer.** Currently saves a copy of the AI-drafted suggestion as a record — reasonable now that the outreach is internal, but still worth confirming this is actually the intended use rather than something inherited from the original diagram without a clear reason.

## How I'd make this production-ready for a client

1. Add error handlers on every external API module (Maps, OpenAI x2, Gmail, Airtable, Sheets, Docs, JSON parser) with at least a notification on failure and a retry where sensible.
2. Add a de-duplication check (by Google Place ID) before creating new Airtable records, so repeated scenario runs don't re-log the same business.
3. Add a rate limiter / delay between iterations for larger batches, to stay within API limits.
4. If real outreach to businesses is eventually needed (not just internal notification), add a proper email-enrichment step (website scrape or a service like Hunter.io) before attempting to contact anyone directly — and keep a human-review step before anything sends automatically.
5. Consider applying the same "strict JSON + parse" pattern used on OpenAI #1 to OpenAI #2's outreach draft, so the subject line and body could be split cleanly rather than living in one block of text.

## Debugging notes (real issues hit while building)

- Google Cloud requires an active billing method to enable the Places API, even though usage stays within the $200/month free credit — the "Enable APIs" onboarding shortcut failed silently until billing was added.
- The Iterator's `Array` field mapping was lost after reconnecting the Google Maps API key partway through the build, which caused every downstream module to receive empty data with no error shown, until traced back by checking each module's individual execution output.
- OpenAI's API has no free tier — separate from ChatGPT's web interface — and requires prepaid credit to make any call at all, confirmed via a `429 RateLimitError` when credit ran out mid-test.
- Manually defining the Parse JSON module's expected structure (rather than using Make's "Generate" auto-detect, which needs a live sample) worked fine and didn't require any OpenAI credit — useful for continuing to build while blocked on billing.

## Files

- `screenshots/` — visual reference of the scenario canvas
- `blueprint.json` — not yet exported (pending a fully successful end-to-end run with active OpenAI credit)

## Notes

This is a practice/learning rebuild, not a production system. Built independently as part of an onboarding exercise to understand AI + automation workflow architecture for a role in AI automation engineering, then iterated on to fix real issues found through hands-on testing rather than just describing what a fix could look like.
