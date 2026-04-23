---
name: peec-geo-debug
description: |
  Debug why a domain isn't being cited by LLMs. Implements Malte's 4-step GEO citation framework using live Peec AI data.
  Trigger when the user says: "geo debug", "why isn't [domain] being cited", "debug citations for", "run geo debug", or "citation audit".
  Requires: a Peec AI project (name or ID) and a domain to audit (e.g. attio.com).
---

# GEO Citation Debugger
### Based on Malte Landwehr's framework: *"Not getting cited by LLMs? You can debug that!"*

> **Credit:** This skill implements the 4-step GEO citation debug framework created by **Malte Landwehr** (CPO & CMO, Peec AI).
>
> Malte's original framework:
>
> *"Let's assume I have a website and want to be the source for a set of prompts. Each prompt generates some fanout queries.*
>
> *1️⃣ Is your website directly cited in the answer? → Yes: done. No: go to step 2.*
>
> *2️⃣ Is your domain used as a source but not explicitly cited? → Yes: make it more citeable. Add a summary with the key statement. Keep it short. Write in clear, simple language. Keep entity density high. Use an authoritative, declarative voice. → No: go to step 3.*
>
> *3️⃣ Does your domain rank for the fanout query but not get used as a source? → Yes: improve the content so it is more source-worthy. Make sure you agree with broad consensus. Add unique information no other source has. Strengthen EEAT signals. Add quotes from trustworthy sources. Explicitly name your sources. Expand to cover multiple fanout queries. → No: go to step 4.*
>
> *4️⃣ Do you already have a relevant page for the fanout query that does not rank yet? → Yes: make sure it is indexed. Build internal and external links. Improve the content. Check for crawling or rendering issues. Do SEO. → No: create that page."*

You are an expert GEO analyst running a systematic citation audit. Your job is to diagnose *exactly* why a domain is or isn't cited by LLMs across a set of tracked prompts — and prescribe the right fix for each.

---

## Step 0 — Resolve inputs

If the user didn't provide a project name/ID and domain, ask for both before proceeding.

- Resolve the project: call `list_projects`, match by name, get `project_id`.
- Note the target domain (e.g. `attio.com`). Strip `https://`, `www.`, trailing slashes.
- Set date range: `end_date` = today, `start_date` = 30 days ago (or as specified by user).

---

## Step 1 — Fetch all prompts and domain citation data

Run these **in parallel**:

1. `list_prompts` with `limit: 10000` → get every prompt (`id`, `text`, `topic_id`, `tag_ids`).
2. `get_domain_report` with:
   - `dimensions: ["prompt_id"]`
   - `filters: [{ field: "domain", operator: "in", values: ["<domain>"] }]`
   - full date range
   
   This returns per-prompt: `retrieved_percentage` (0–1, was the domain pulled as a source?) and `citation_rate` (avg explicit citations per chat).

3. `list_search_queries` for the full project and date range (no prompt_id filter) to get all fanout queries. Deduplicate by `query_text`, group by `prompt_id`.

---

## Step 2 — Classify every prompt into Malte's 4 states

For each prompt, look up its domain report row and fanout queries, then assign a state:

```
citation_rate > 0.05              → STATE A  ✅  Cited
retrieved_percentage > 0.05       → STATE B  ⚠️  Retrieved but not cited
fanout queries exist for prompt   → STATE C  🔍  Reaches search, not retrieved
no fanout data                    → STATE D  ❌  No relevant page indexed
```

**The logic mirrors Malte's decision tree exactly:**
- State A: Already working. Move on to other prompts.
- State B: AI pulls the content but doesn't quote it — a *citeability* problem.
- State C: The domain has some search presence for this topic but isn't being selected as a source — a *source-worthiness* problem.
- State D: No page exists that covers this territory at all — a *content existence* problem.

---

## Step 3 — For State B prompts, sample one chat

For each STATE B prompt (retrieved but not cited), call `get_chat` on one recent chat for that prompt. Read the actual AI response to understand:
- What language does the AI use when it *does* mention the domain?
- Is it vague ("you might check…") or specific ("according to [source]…")?
- What's the structure of a response that IS cited vs. one that isn't?

Use this to sharpen the action recommendation for that prompt.

---

## Step 4 — Output the report

Print this exact structure:

---

```
╔══════════════════════════════════════════════════════════════════════╗
║  GEO Citation Debug Report                                          ║
║  Domain: <domain>  ·  Project: <name>  ·  Last 30 days             ║
╠══════════════════════════════════════════════════════════════════════╣
║  SUMMARY                                                            ║
║  ✅  <N>  prompts  —  Cited (healthy)                               ║
║  ⚠️  <N>  prompts  —  Retrieved but not cited                       ║
║  🔍  <N>  prompts  —  Reaches search, not retrieved                 ║
║  ❌  <N>  prompts  —  No relevant page                              ║
╚══════════════════════════════════════════════════════════════════════╝
```

Then output **three prioritized action groups**:

---

### 🔴 Priority 1 — Create these pages (State D)
*These topics have zero coverage. Every prompt here is a missed buyer.*

For each STATE D prompt:
```
Prompt: "<text>"
Fanout queries to target: <list up to 4>
Action: Create a page covering [specific angle based on fanout queries].
        Target these fanout queries in H2s: [list]
        Internal link from: [suggest 1-2 logical internal link sources based on what you know of the domain]
```

---

### 🟡 Priority 2 — Improve source-worthiness (State C)
*Pages exist but AI models don't trust them enough to retrieve.*

For each STATE C prompt:
```
Prompt: "<text>"
Fanout queries AI used: <list>
Action: Your page likely covers this but lacks authority signals. Add:
        • A named author with credentials visible above the fold
        • At least 2 external citations from recognisable sources
        • Original data or a unique claim no competitor makes
        • Explicit source attribution ("According to [Source], ...")
        Expand to cover all fanout queries above in one authoritative doc.
```

---

### 🟠 Priority 3 — Make it citeable (State B)
*AI retrieves your content but won't quote it. One targeted fix per prompt.*

For each STATE B prompt, use what you read from the chat sample to be specific:
```
Prompt: "<text>"
Retrieved: <retrieved_pct>%  |  Citation rate: <rate>
What AI says now: "<quote or paraphrase from sampled chat>"
Fix: Add a summary block immediately after your intro with this structure:
     • Lead with the key claim in one declarative sentence
     • Follow with 2-3 supporting facts (numbers, named entities, specific comparisons)
     • Close with the entity relationship made explicit: "[Brand] is [category] for [ICP] because [differentiator]"
     Keep it under 100 words. No hedging language.
```

---

### ✅ Healthy — No action needed (State A)
List prompt texts only, one line each. No detail needed.

---

## Final line

```
Total: <N> prompts  ·  <N> need action  ·  Est. fix time: <N> hours
Methodology: peec.ai/blog/how-to-choose-the-right-prompts-for-llm-tracking
```

---

## Guardrails

- **Never fabricate data.** If a prompt has no domain report row, its retrieved_pct and citation_rate are both 0.
- **If fewer than 10 prompts have data** (i.e., tracking hasn't run yet), say so clearly: *"No tracking data yet for this project. Run the prompts first, then re-run this audit."* Do not attempt to classify without data.
- **State C vs D distinction**: If there are fanout queries for a prompt but retrieved_pct = 0, that means the AI *searched* for this topic but didn't land on the domain — that's State C. If there are zero fanout queries at all, that's State D.
- **Domain matching**: Match `retrieved_percentage` against the exact domain string. Do not match subdomains unless the user specified one (e.g., `blog.attio.com` ≠ `attio.com`).
- **Prioritisation rationale**: State D before C before B because: creating a missing page unlocks all future citation potential, improving source-worthiness converts existing traffic into LLM citations, and citeability tweaks are the fastest wins but smallest surface area.
