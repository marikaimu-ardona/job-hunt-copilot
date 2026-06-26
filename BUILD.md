# Job-Hunt Copilot — n8n build guide

An n8n workflow that takes a job posting, scores how well it fits Marikaimu's
resume, and drafts a tailored cover-letter intro in his voice. Results show on
the form's result page and get emailed to him.

Built as a learning + portfolio piece. AI provider: Google Gemini (free tier;
swap to Claude later by changing only the chat-model node).

---

## What it does (architecture)

```
[On form submission]   <- paste a job posting (+ optional URL)
        |
[Prepare inputs]       <- normalises fields + injects the resume
        |
[Analyse with Gemini]  <- Basic LLM Chain + Google Gemini Chat Model + Structured Output Parser
        |
        |--> [Email result]   (SMTP, to yourself)
        |--> [Show result]    (Form completion page)
```

Core n8n concepts this teaches: **trigger nodes**, **data normalisation with
Edit Fields**, **expressions** (`{{ $json.x }}`), the **LangChain LLM chain**
with a **chat model sub-node** and a **structured output parser**, and
**fan-out connections** (one node feeding two).

---

## Prerequisites

1. **Run n8n.** Easiest, no install:
   ```bash
   npx n8n
   ```
   Open http://localhost:5678 and create the local owner account.

2. **Google Gemini API key (free).** Get one at https://aistudio.google.com/apikey
   (sign in with a Google account, click "Create API key", copy it). No billing
   or card needed for the free tier. In n8n you will add it as a
   **Google Gemini(PaLM) API** credential when you configure the chat model node.

3. **Gmail App Password (for the email step).** Simpler than OAuth on a local
   instance. Turn on 2-Step Verification on your Google account, then create an
   App Password (Google Account > Security > App passwords). You will use it in
   an **SMTP** credential:
   - Host: `smtp.gmail.com`
   - Port: `465`  (SSL)
   - User: your Gmail address
   - Password: the 16-character app password

---

## Build it node by node

### 1. On form submission  (Form Trigger)
- Add node > search **"On form submission"**.
- Form Title: `Job-Hunt Copilot`
- Form Description: `Paste a job posting to score the fit and draft an intro.`
- Form Fields:
  | Field Label | Type | Required |
  |---|---|---|
  | `Job posting` | Textarea | yes |
  | `Job URL` | Text | no |
- Save. Click **"Test step"** later to get a form URL you can open.

### 2. Prepare inputs  (Edit Fields / Set)
Normalises the messy form keys into clean ones and injects the resume so the
AI has everything in one place.
- Add node > **Edit Fields (Set)**. Mode: **Manual Mapping**. Keep only set
  fields (turn off "Include Other Input Fields").
- Add these fields (all type String):
  - `jobText`  =  `{{ $json['Job posting'] }}`
  - `jobUrl`   =  `{{ $json['Job URL'] }}`
  - `resume`   =  paste the resume block from `resume-snippet.txt` (in this folder)

### 3. Analyse with Gemini  (Basic LLM Chain)
- Add node > **Basic LLM Chain** (under Advanced AI / LangChain).
- **Source for Prompt**: "Define below".
- **Prompt**: paste the contents of `prompt.txt` (in this folder). It already
  references `{{ $json.jobText }}` and `{{ $json.resume }}`.
- Attach the **Chat Model** sub-node: click the model connector and add
  **Google Gemini Chat Model**.
  - Credential: add your **Google Gemini(PaLM) API** key (the AI Studio key).
  - Model: pick a current Gemini model, e.g. **Gemini 2.0 Flash** (fast and
    free-tier friendly). Swapping to Claude later means only replacing this one
    sub-node with **Anthropic Chat Model**.
- Attach the **Output Parser** sub-node: add **Structured Output Parser**.
  - Schema type: "Generate from JSON example" and paste `schema.json`
    (in this folder). This forces Claude to return clean, typed fields.

After this node runs, the parsed result is available at `{{ $json.output }}`
(e.g. `{{ $json.output.fitScore }}`, `{{ $json.output.tailoredIntro }}`).

### 4a. Email result  (Send Email / SMTP)
Connect this from the **Analyse with Gemini** node.
- Add node > **Send Email** (SMTP).
- Credential: the Gmail SMTP credential above.
- From: your Gmail.  To: your Gmail.
- Subject: `{{ "Job fit " + $json.output.fitScore + "/100 — " + $json.output.role + " at " + $json.output.company }}`
- Email Format: HTML.
- HTML body:
  ```
  <h2>{{ $json.output.role }} — {{ $json.output.company }}</h2>
  <p><strong>Fit: {{ $json.output.fitScore }}/100.</strong> {{ $json.output.verdict }}</p>
  <h3>Why it fits</h3>
  <ul>{{ $json.output.strengths.map(s => '<li>' + s + '</li>').join('') }}</ul>
  <h3>Gaps to watch</h3>
  <ul>{{ $json.output.gaps.map(g => '<li>' + g + '</li>').join('') }}</ul>
  <h3>Draft intro</h3>
  <p>{{ $json.output.tailoredIntro.replace(/\n/g, '<br>') }}</p>
  ```

### 4b. Show result  (Form — Completion)
Also connect this from the **Analyse with Gemini** node (so the chain feeds
both 4a and 4b).
- Add node > **Form** (n8n Form), Operation: **Completion**.
- Completion Title: `Fit: {{ $json.output.fitScore }}/100`
- Completion Message (or "Custom" text): the verdict + the draft intro, e.g.
  `{{ $json.output.verdict }}\n\n{{ $json.output.tailoredIntro }}`

---

## Test it
1. Click **Test workflow** (or activate it).
2. Open the form URL from the Form Trigger node.
3. Paste a real job posting (try the Fantum "AI Web Builder & Prompt Artist"
   description) and submit.
4. You should see the fit score + draft intro on screen, and an email arrives.
5. Open the last execution in n8n and inspect each node's input/output. This is
   the core n8n debugging skill: data flows as an array of items, each under a
   `json` key, and you can re-run a single node.

---

## Next steps (Phase 2/3)
- **Tracker:** add a Google Sheets "Append row" node after the chain to log
  company, role, score, intro, URL, date.
- **Live URL scraping:** add an IF (does `jobUrl` exist?) + HTTP Request +
  HTML Extract so you can paste just a URL. Note many job sites block bots, so
  paste stays the reliable default.
- **Portfolio:** screenshot the canvas + a sample result, write a short
  case study, and link it from the portfolio. Add "n8n" and "workflow
  automation / AI agents" to the skills section and resume.
