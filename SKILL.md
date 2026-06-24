---
name: geo-content-gap
description: "GEO/LLMO content-gap analysis. For each tracked prompt, determine where an AI engine retrieves vs. cites your site, classify the gap, and produce a prioritized action plan with prompt-specific benchmark sources. Reference implementation uses Peec AI; adapt the data-source steps to whatever AI-visibility source you have. Start prompt: 'New client: [name], domain: [domain.com]. Tracked prompts: [list]. Run the content-gap analysis.'"
---

# GEO / LLMO Content-Gap Agent

> A Claude Skill for **Generative Engine Optimization (GEO)** content-gap analysis.
> It finds, for every tracked prompt, *what content you need to be cited more often by AI engines* (ChatGPT, Perplexity, Gemini, etc.) — and tells apart a **content gap** from a mere **visibility gap**.

---

## Data source — read this first

This skill ships with a **reference implementation built on [Peec AI](https://peec.ai)** (an AI-visibility tracker). The tool calls below — `get_brand_report`, `get_url_report` — are Peec AI MCP tools.

**Adapt these to your own visibility data source.** Any source works as long as it can give you, per prompt:

1. the **visibility** of your brand (how often the engines surface you),
2. **which URLs of your domain** were *retrieved or cited* for that prompt, and
3. the **top third-party sources** the engines cite for that prompt.

If you use a different GEO tracker, a manual export, or your own log, keep the *methodology* (the Mandatory Pipeline, the decision matrix, the deliverables) and swap only the data-fetching calls.

---

## Goal

Systematic identification and prioritization of the content a website needs so that AI systems cite it more often as a source — based on the prompts you track.

The core insight this skill is built around:

> **Retrieval ≠ Citation.** A page with a high `retrieval_count` but low `citation_count` is your single most valuable finding: the AI engine *fetches* the page but the content *fails to convince it* to cite. That is an optimization target, not a content hole.

---

## Roles and preconditions

- **GEO/LLMO expert (you):** defines the tracked prompts with the client, supervises the analysis, makes editorial decisions.
- **Claude + visibility data source (MCP or export):** reads visibility and source data, generates structured analysis and recommendations.
- **Precondition:** a set of prompts (~25 is a good batch) is tracked for the client; the client domain is known.

---

## Step 1 — Setup

Send the following **start prompt** to set the context:

> You are my expert-level GEO / LLMO content-gap analysis assistant. I run the client **[client name]** with the domain **[domain]**. The following prompts are tracked: 
>
> [paste the list of prompts]
>
> For each tracked prompt, analyze:
> 1. Current **visibility** (in %).
> 2. **All** URLs of the client domain that were either **retrieved or cited** for this prompt (sorted by `retrieval_count` desc).
> 3. The **top-cited third-party sources** (benchmark URLs) for this prompt.
>
> Then build a table with these columns:
> - **Column A — Prompt:** the tracked prompt.
> - **Column B — Source:** **ALL** URLs of the client domain (or specified subdomain) that were **retrieved or cited** for this prompt, sorted by `retrieval_count` desc. **Never drop URLs with `citation_count = 0`** — they are an important signal: engines consider the page relevant enough to fetch, but the content does not convince for a citation. If nothing was retrieved: "–". Format: one URL per line, no `https://` prefix.
> - **Column C — Action recommendation:** structured in three blocks (see format spec below): visibility value + status sentence; "Recommendation:" with a concrete step; "Benchmark sources:" with 3–5 top third-party sources from a prompt-specific query.
> - **Column D — Action:** "No action needed" or "Action needed".
>
> **Decision logic for columns C and D:**
> - High visibility (≥ 90%) + client URL is used → No action needed; short confirmation in C, no benchmark sources required.
> - Low visibility (< 90%) + client URL is used → Action needed; recommend optimizing the existing page (add FAQ, expand content, improve structure, listicle format, Schema.org); **3–5 benchmark sources from a prompt-specific query are mandatory.**
> - Low visibility (< 90%) + no client URL → Action needed; recommend a new page or a fundamental rework; **3–5 benchmark sources from a prompt-specific query are mandatory.**
>
> Always account for:
> - The **search intent** of the prompt (informational, navigational, commercial, transactional).
> - The **target audience** (e.g. employers, SMBs, consumers, brokers).
> - Check via a **Google `site:` query** (`site:domain.com keyword`) whether suitable content already exists on the client site that could be reworked (comment briefly).
>
> Follow the **Mandatory Pipeline** (Step 2). Start with prompt 1 and work through the list systematically.

---

## Step 2 — Mandatory Pipeline (per prompt)

> ⚠️ **These four steps are mandatory for every prompt before columns C/D are written. No omissions, no carrying over data from thematically similar neighboring prompts, no derivations from general knowledge.**

The tool names below are the Peec AI reference implementation. Replace them with your data source's equivalents if needed.

**2.1 Determine visibility**
- Tool: `get_brand_report`
- Params: `dimensions=["prompt_id"]`, `filters=[{field: "brand_id", operator: "in", values: [<own_brand_id>]}]`
- Purpose: visibility score (0–1, shown as %) for exactly this prompt.

**2.2 Determine client URLs**
- Tool: `get_url_report`
- Params: `dimensions=["prompt_id"]`, `filters=[{field: "prompt_id", operator: "in", values: [<prompt_id>]}, {field: "domain", operator: "in", values: [<client_domain>]}]`, `order_by=[{field: "retrieval_count", direction: "desc"}]`
- Purpose: complete list of every client-domain URL retrieved or cited for this prompt.
- **Mandatory:** include ALL hits in column B, even where `citation_count = 0`. If a subdomain is specified (e.g. `blog.domain.com` instead of `domain.com`), filter out URLs from other subdomains manually — most trackers filter on the domain level only.

**2.3 Determine benchmark URLs**
- Tool: `get_url_report`
- Params: `dimensions=["prompt_id"]`, `filters=[{field: "prompt_id", operator: "in", values: [<prompt_id>]}]` (NO domain filter), `order_by=[{field: "retrieval_count", direction: "desc"}]`, `limit=10`
- Purpose: top third-party sources cited for exactly this prompt.
- **Mandatory:** from the top 10, name the 3–5 strongest third-party sources (not the client domain). **Never carry over from thematically similar prompts, never derive from general knowledge. If data is missing: re-run the tool call, do not guess.**

**2.4 Optional, when 2.2 is empty: Google `site:` query**
- Tool: `web_search` with query `site:<client_domain> <core keywords of the prompt>`
- Purpose: check whether suitable content already exists on the client site that just isn't retrieved by engines (= a structure/visibility problem rather than a content gap).

> **Stop rule:** the final table may only be produced after these 4 steps are completed for ALL prompts. If the tool budget runs low or processing takes long: report a short interim status, then continue. Never let data from neighboring prompts or generic knowledge leak in.

---

## Step 3 — Classification by decision matrix

| Visibility | Client URL as source? | Classification | Benchmarks mandatory? |
|------------|-----------------------|----------------|------------------------|
| High (≥ 90%) | Yes | ✅ No action needed | No |
| Low (< 90%) | Yes | ⚠️ Optimize existing page | **Yes** |
| Low (< 90%) | No | 🔴 New or reworked content needed | **Yes** |
| High (≥ 90%) | No | ℹ️ Monitor; review content mid-term | Recommended |

---

## Step 4 — Phrasing the action recommendation (Column C)

Column C has a **fixed three-part structure** (bold sub-headers, plain-text content, blank line between blocks):

```
Visibility: X%. [Status sentence: which pages are used, how the usage is
characterized (regularly / occasionally / not), where the main gap is.]

Recommendation:
[Concrete step with (a), (b), (c) enumeration of the individual
content/structure building blocks. Name search intent and audience.
Reference existing client pages where applicable.]

Benchmark sources:
domain1.com/path/page
domain2.com/path/page
domain3.com/path/page
domain4.com/path/page
domain5.com/path/page
```

**For "No action needed"** (visibility ≥ 90% + client URL used) the "Benchmark sources" block is dropped. Then only:

```
Visibility: X%. Page is consistently used as a source. Monitoring only —
ensure the answer stays prominently recognizable.
```

---

## Step 5 — Output format: THREE deliverables

> **Mandatory:** every content-gap analysis returns the following three files together.

**5.1 Markdown file (.md) — full analysis**

Contains:
- Header with client, domain, data source, time range, own brand ID, key competitors.
- Visibility overview (distribution of prompts across visibility buckets).
- Detailed content-gap table (all prompts with columns A–D).
- Prioritization of actions (quick wins / strategic / long-term).
- Cross-cutting recommendations.
- Proposed next steps (e.g. content briefs).

**5.2 TSV file (.tsv) — paste-ready table for Google Sheets**

Pure data table, one row per prompt, columns A–D. **Format spec:**

- Tab-separated (a tab character `\t` between columns).
- First row: header `Prompt\tSource (sub.domain.com)\tRecommendation\tAction` (column B name matches the client's subdomain/domain).
- Cells with line breaks are wrapped in **double quotes** (`"…"`); embedded quotes are doubled (`""`).
- Inside cells, real newlines (`\n`) are used — Sheets interprets them as in-cell line breaks on paste.
- URLs **without** `https://` prefix (matches the sheet convention: `sub.domain.com/de/page` not `https://sub.domain.com/de/page`).
- NO markdown formatting inside cells (no `**bold**`, no `` `backticks` ``, no markdown links).
- Sub-headers in column C as plain text with a colon (`Recommendation:`, `Benchmark sources:`) — the user bolds them manually after pasting into the sheet.

**5.3 Excel file (.xlsx) — fallback**

Only if TSV is not possible (e.g. special-character issues): produce an Excel file with identical content. TSV is the default.

---

## Step 6 — Quality assurance

**Self-check by Claude before output:**
- Was the Mandatory Pipeline (4 steps) run for EVERY prompt?
- Are ALL retrieved/cited client URLs listed in column B (including those with `citation_count = 0`)?
- Do ALL benchmark sources come from prompt-specific queries (not from neighboring prompts or generic knowledge)?
- Does the response deliver exactly three files (MD + TSV + optional XLSX fallback)?

**Quality check by you (user):**
- Are the recommendations realistic and a fit for the client's website?
- Do the cited benchmark URLs actually match the prompt's topic?
- Are there prompts where Claude loaded no data → Claude should flag this itself; otherwise verify manually.
- Does the TSV paste cleanly into Sheets without columns shifting?

---

## Step 7 — Action planning and prioritization

After the analysis, prioritize:

**Priority 1 — Quick wins:** existing page present + low visibility + client URL used occasionally → simple optimizations (FAQ, structured data, expand content). Especially: pages with high `retrieval_count` but low `citation_count` — already fetched, just need to become more convincing.

**Priority 2 — High-potential new content:** no existing content + low visibility + the prompt is strategically important (e.g. a core product page) → plan a new subpage.

**Priority 3 — Long-term content:** the prompt is informational and high in the funnel → a content piece, guide, or knowledge article.

---

## Step 8 — Handoff to editorial / implementation

For each "Action needed" prompt, create a short **content brief** containing:
- Target prompt (= the question to answer).
- Audience and search intent.
- Recommended URL structure and page title.
- Key content elements (based on the benchmark sources).
- Internal linking recommendations (from/to which client pages).
- Schema.org recommendations (FAQPage, Offer, Event, etc., depending on intent).

Claude can generate these briefs on request once the gap analysis is complete.

---

## Practical tips

- **Batches, not one-shot:** with more than 10 prompts, work in batches of 5–8 and output an interim table after each. Inside a batch, still run the full Mandatory Pipeline per prompt. **Never drop tool calls for efficiency** — that was the failure of the agent's first generation.
- **Retrieval ≠ Citation:** a page with high `retrieval_count` but low `citation_count` is the most valuable optimization insight: engines fetch it but the content fails to convince. Name exactly that in column C.
- **Subdomain filter manually:** when the user specifies a subdomain (e.g. "only `blog.`"), check after the fetch and exclude URLs from other subdomains of the same root domain in column B.
- **Benchmark sources always prompt-specific:** AI systems orient heavily on the structure, depth and authority of the cited sources — that information must come from the real data context of *this exact prompt*, not from assumptions about the topic area.
- **Integrate the `site:` query:** explicitly ask Claude to check via a Google `site:` query whether suitable content already exists on the client site — this avoids needless new production and identifies visibility gaps vs. content gaps.
- **Repeat regularly:** run monthly or quarterly, since visibility shifts continuously through new content, competitor activity, and AI-model updates.

---

## Worked example (one TSV row)

A neutral, fictional example for an outdoor retailer at `yourbrand.com`.

**Column A (prompt):**

```
I'm planning a multi-day hike in the Alps in autumn — what gear do I need?
```

**Column B (source):**

```
yourbrand.com/guides/alpine-hiking-checklist
yourbrand.com/guides/layering-system
yourbrand.com/category/hiking-boots
yourbrand.com/blog/autumn-hiking-tips
```

**Column C (recommendation):**

```
Visibility: 12%. The layering-system and autumn-tips pages are fetched
occasionally but rarely cited. Notable: /guides/layering-system is retrieved
4x but cited 0x — the content does not convince for a citation.

Recommendation:
Expand the existing /guides/alpine-hiking-checklist into a hub page
"Multi-day alpine hiking gear (autumn)" with
(a) a layered packing list by category,
(b) weight ranges per item,
(c) weather/temperature guidance for autumn,
(d) links to relevant product categories,
(e) an FAQ "What changes for autumn vs. summer hikes".
Search intent: informational/commercial. Audience: experienced hikers.

Benchmark sources:
example-outdoor-magazine.com/alpine-hiking-gear
example-trailguide.com/autumn-hiking-checklist
example-gearlab.com/best-hiking-layers
youtube.com/watch?v=EXAMPLE
example-alpineclub.org/safety/equipment
```

**Column D (action):**

```
Action needed
```

---

## Anti-patterns — common mistakes Claude must avoid

Based on documented failures from earlier runs:

1. **Dropping URLs with `citation_count = 0` in column B.**
   Wrong: only listing the cited URLs.
   Right: list ALL retrieved/cited URLs, sorted by `retrieval_count`.

2. **Carrying benchmark sources over from neighboring prompts.**
   Wrong: "Prompt 21 on the same topic had these top sources, I'll reuse them for prompt 1."
   Right: run a dedicated `get_url_report` query for each prompt.

3. **Deriving benchmark sources from general knowledge.**
   Wrong: "For a hiking topic the typical benchmarks are X, Y, Z."
   Right: query what the data actually returns for this exact prompt.

4. **Shortcutting the Mandatory Pipeline for efficiency.**
   Wrong: "With 25 prompts I'll just do one bulk query per batch."
   Right: bulk queries are fine, but per prompt you must have visibility + client URLs + benchmark URLs before writing C/D.

5. **Ignoring a subdomain specification.**
   Wrong: when the user says "only `blog.domain.com`", still listing URLs from other subdomains.
   Right: filter manually to the specified subdomain.

6. **Markdown formatting inside TSV cells.**
   Wrong: `**Recommendation:**` with asterisks in the TSV cell.
   Right: `Recommendation:` as plain text; formatting happens after pasting into the sheet.

7. **Delivering one file instead of three.**
   Wrong: only outputting the `.md` file.
   Right: deliver MD + TSV (or MD + XLSX fallback) together.
