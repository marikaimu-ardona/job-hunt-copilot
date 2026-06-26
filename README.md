# Job-Hunt Copilot

An [n8n](https://n8n.io) automation that reads a job posting, scores how well it
fits my résumé using AI, and drafts a tailored cover-letter intro in my voice,
then shows the result on screen, emails it, and logs every run to a tracker.

Built as a personal project to take the repetition out of job hunting, and to
show an AI-driven n8n workflow end to end.

## How it works

```
[On form submission]   paste a job posting
        |
[Prepare inputs]       normalise fields + inject my resume
        |
[Analyse with Gemini]  LangChain LLM chain + Google Gemini + Structured Output Parser
        |              -> returns clean, typed fields (fitScore, verdict, strengths, gaps, tailoredIntro)
        |
        |--> [Show result]      fit score + intro on the form's completion screen
        |--> [Email result]     formatted summary by email (SMTP)
        |--> [Append row]       one row per job in a Google Sheet (tracker)
```

The prompt is deliberately tuned to score **strictly and honestly** and to write
**without clichés or em dashes**. Real postings come back at scores like 25, 55,
and 60 out of 100, not flattering nonsense, so the output is actually useful for
deciding where to spend effort.

## Stack

n8n · Google Gemini · LangChain (LLM chain + structured output parser) ·
Google Sheets · SMTP · prompt engineering

## Files

| File | Purpose |
|------|---------|
| `workflow.json` | The full n8n workflow. Import it, or paste it onto the canvas. |
| `BUILD.md` | Step-by-step build guide (nodes, settings, expressions, credentials). |
| `prompt.txt` | The analysis prompt used by the Gemini node. |
| `schema.json` | The JSON example that drives the Structured Output Parser. |
| `resume-snippet.txt` | The résumé text the workflow scores against. |

## Run it

1. Spin up n8n (`npx n8n`, or n8n Cloud).
2. Import `workflow.json` (or paste it onto the canvas).
3. Add credentials in the UI: a Google Gemini API key, and SMTP for email.
4. Open the form, paste a job posting, and submit.

Full details in [`BUILD.md`](./BUILD.md).

---

Built by Marikaimu Chris Ardoña — [portfolio](https://marikaimu-ardona.github.io).
