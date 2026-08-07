# Project Proposal Builder

n8n workflow that turns an Upwork job post pasted into Slack into a ready-to-review proposal, written in Marina Pereda's voice, for either Shopify development or AI automation work. A human always reviews and sends the final proposal manually; this workflow drafts it.

## How it works

```
Slack Trigger (message in channel)
        │
        ▼
   Edit Fields  (captures the raw job post text)
        │
        ▼
   AI Agent (classifier)  →  tags the job: SHOPIFY / AI_AUTOMATION / OTHER
        │
        ▼
   Route by Category (Switch)
        │
        ├── Shopify ──────────► Shopify Skills Profile ──► Shopify Proposal Writer ──┐
        │                                                                             │
        ├── AI Automation ───► Automation Skills Profile ──► AI Automation Writer ────┤
        │                                                                             ▼
        │                                                              Post Proposal to Slack
        │                                                              (thread reply, human reviews)
        └── Other ───────────► Skipped Job Note (thread reply, no proposal drafted)
```

1. **Slack Trigger** fires when a job post is pasted into the designated channel.
2. **Edit Fields** captures the message text as `Job Description`.
3. **AI Agent (classifier)** reads the post and returns `job_category`: `SHOPIFY`, `AI_AUTOMATION`, or `OTHER`. Model: Claude Sonnet.
4. **Route by Category** (Switch node) sends the job down one of three paths based on that category.
5. Each matching path loads a **Skills Profile** node (Marina's verified skills, portfolio, and proof points for that category) and passes it into a dedicated **Proposal Writer** agent (Claude Opus) that drafts the full proposal in Marina's voice.
6. **Post Proposal to Slack** replies in the original thread with the draft, ready for review and manual submission on Upwork.
7. Jobs that are neither Shopify nor AI automation get a one-line **Skipped Job Note** instead of a draft, so nothing disappears silently.

## Why two separate skills profiles

Shopify work and AI automation work draw on different, non-overlapping proof points (custom themes and CRO metrics vs. n8n and agent design). Keeping them in two Set nodes means:

- Each Proposal Writer only sees skills relevant to that job type, so it never blends Shopify jargon into an automation pitch or vice versa.
- Updating one category's skills never risks breaking the other's prompt.
- Each profile can carry its own honesty rules. The AI Automation profile, for example, is explicit that Marina has no public automation case studies yet, so the agent inserts a `[MARINA: add example]` flag instead of inventing one.

## Output format

Both Proposal Writer agents return structured JSON (enforced by a Structured Output Parser):

```json
{
  "job_type": "LONG_TECHNICAL | SHORT_VAGUE | SPECIFIC_TASKS | OPERATIONS_ROLE",
  "client_need_summary": "What the client actually needs and what would win this job",
  "fit_score": "STRONG | GOOD | WEAK",
  "fit_notes": "Budget vs rate, red flags, competition level",
  "cover_letter": "Full proposal text, ready to paste into Upwork",
  "screening_answers": [
    { "question": "Exact screening question text", "answer": "Marina's answer" }
  ],
  "review_flags": ["Anything Marina should check or confirm before submitting"]
}
```

The Slack message surfaces `fit_score` and `fit_notes` above the letter so a weak-fit job can be skipped in seconds without reading the full draft.

## Voice and content rules

Both writer prompts enforce the same non-negotiables, learned from Marina's actual winning proposals:

- No em dashes, en dashes, or hyphens used as pauses. No emojis.
- A banned-phrase list covering common AI tells ("I'd love to", "seamless", "leverage", "delve", etc.).
- Structure: a job-specific hook (2 to 4 short paragraphs) above a `---` divider, then a bio assembled from the skills profile below it.
- Every claim must trace back to the relevant Skills Profile node. Nothing is invented; anything uncertain becomes a `[MARINA: ...]` placeholder and is added to `review_flags`.

## Maintaining the skills profiles

To keep proposals accurate and improving over time, edit only these two Set nodes as new work is won or skills are confirmed:

- **Shopify Skills Profile** — portfolio stores, Shopify-specific skills, tool self-ratings, proof points (e.g. the 400% Black Friday result).
- **Automation Skills Profile** — n8n/AI agent skills, engineering stack, and (once available) real automation project examples.

Nothing else in the workflow needs to change when skills are updated.

## Required credentials

| Credential | Used by |
|---|---|
| Slack API (`Slack - Proposal Builder`) | Slack Trigger, Post Proposal to Slack, Skipped Job Note |
| Anthropic API | AI Agent (classifier), Shopify Proposal Writer, AI Automation Proposal Writer |

## Human in the loop

This workflow never submits anything to Upwork. Every draft lands in Slack for Marina to read, edit, and copy into Upwork manually. `fit_score`, `fit_notes`, and `review_flags` exist specifically to make that review fast.
