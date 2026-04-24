# GEO Citation Debugger — Claude Code Skill

A Claude Code skill that runs a systematic LLM citation audit for any domain tracked in [Peec AI](https://peec.ai), using live data to diagnose exactly why a domain is or isn't cited by AI search engines — and prescribe the right fix.

## Credit

This skill implements the **4-step GEO citation debug framework** created by **[Malte Landwehr](https://github.com/MalteBerlin)** (CPO & CMO, Peec AI), published on LinkedIn:

> *"Not getting cited by LLMs? You can debug that!"*
>
> 1️⃣ **Is your website directly cited?** → Yes: done. No: go to step 2.
>
> 2️⃣ **Is your domain retrieved but not explicitly cited?** → Make it more citeable. Add a summary with the key statement. Keep it short, clear, entity-dense, declarative.
>
> 3️⃣ **Does your domain rank for the fanout query but not get used as a source?** → Improve source-worthiness. Agree with broad consensus. Add unique data. Strengthen EEAT. Name your sources. Cover multiple fanout queries in one doc.
>
> 4️⃣ **Do you have a relevant page that doesn't rank yet?** → Fix indexing, internal/external links, crawling issues. Do SEO. If no page exists: create it.

## What it does

Given a **Peec AI project** and a **domain**, the skill:

1. Fetches all tracked prompts and their fanout search queries from Peec AI
2. Pulls per-prompt citation data for the target domain (`retrieved_percentage`, `citation_rate`)
3. Classifies every prompt into one of Malte's 4 states:

| State | Condition | Problem |
|-------|-----------|---------|
| ✅ A — Cited | `citation_rate > 0.05` | None |
| ⚠️ B — Retrieved, not cited | `retrieved_percentage > 0.05` | Citeability |
| 🔍 C — Reaches search, not retrieved | Fanout queries exist | Source-worthiness |
| ❌ D — No relevant page | No fanout queries | Content existence |

4. Samples actual AI chat responses for State B prompts to diagnose the exact citeability failure
5. Outputs a prioritised action report: **create → source-worthiness → citeability**

## Requirements

- [Claude Code](https://claude.ai/code) with the Peec AI MCP server configured
- A Peec AI account with at least one active project tracking prompts

## Usage

In Claude Code, type:

```
/peec-geo-debug
```

You'll be asked for:
- **Project name** (or ID) — which Peec AI project to audit
- **Domain** — the domain to debug (e.g. `example.com`)

The skill runs the full audit and outputs a structured report with prioritised fixes.

## Installation

Copy `SKILL.md` into your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/peec-geo-debug
cp SKILL.md ~/.claude/skills/peec-geo-debug/
```

## Methodology

For background on how prompt tracking works and how to choose the right prompts to track:

[peec.ai/blog/how-to-choose-the-right-prompts-for-llm-tracking](https://peec.ai/blog/how-to-choose-the-right-prompts-for-llm-tracking)
