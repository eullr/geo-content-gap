# geo-content-gap

A Claude Skill for GEO / LLMO content-gap analysis.

For every prompt you track, it works out where an AI engine retrieves your pages versus where it actually cites them, classifies the gap, and returns a prioritized action plan with prompt-specific benchmark sources. Output comes as Markdown, a paste-ready Google Sheets TSV, and an Excel fallback.

> **Core idea: retrieval ≠ citation.** A page the engine fetches but never cites is the most useful signal in the data. It is not a content gap, it is an optimization target. Most GEO audits never separate the two.

---

## English

### What it does

Generative Engine Optimization (GEO, also called LLMO or AEO) is about being cited as a source inside AI answers from ChatGPT, Perplexity, Gemini, and Claude. This skill turns a list of tracked prompts into a concrete editorial plan:

- Per prompt: current visibility, every one of your URLs that was retrieved or cited, and the top third-party sources the engines actually cite.
- A decision matrix that sorts each prompt into one of three states: no action, optimize the existing page, or new content needed.
- A prioritized backlog (quick wins, strategic, long-term) and optional content briefs.

### Data source

The skill ships with a reference implementation on [Peec AI](https://peec.ai), an AI-visibility tracker, using its `get_brand_report` and `get_url_report` MCP tools.

Adapt it to whatever visibility source you have. Any source works as long as it gives you, per prompt: your brand's visibility, which of your URLs were retrieved or cited, and the top third-party sources cited. Keep the method (the Mandatory Pipeline, the decision matrix, the three deliverables) and swap only the data calls.

### Install

```bash
git clone https://github.com/<your-org>/geo-content-gap.git
```

Put the `geo-content-gap/` folder (the one with `SKILL.md`) where your Claude environment loads skills, or bundle it into a plugin. Then start it with:

```
New client: [name], domain: [domain.com].
Tracked prompts:
[list]
Run the content-gap analysis.
```

### Why it holds up

The skill writes its guardrails down. An earlier version of this analysis failed in specific ways: it dropped URLs with `citation_count = 0`, reused benchmark sources across prompts, and filled gaps from general knowledge instead of querying the data. Those failures are now an explicit Mandatory Pipeline and an anti-patterns list, so the same mistakes do not come back.

---

## Deutsch

### Was der Skill macht

Generative Engine Optimization (GEO, auch LLMO oder AEO) heißt, als Quelle in den Antworten von ChatGPT, Perplexity, Gemini und Claude zitiert zu werden. Dieser Skill macht aus einer Liste getrackter Prompts einen konkreten Redaktionsplan:

- Pro Prompt: aktuelle Visibility, jede Ihrer URLs, die retrieved oder zitiert wurde, und die Top-Fremdquellen, die die Engines tatsächlich zitieren.
- Eine Entscheidungsmatrix, die jeden Prompt einordnet: kein Handlungsbedarf, bestehende Seite optimieren oder neuer Inhalt nötig.
- Ein priorisiertes Backlog (Quick Wins, strategisch, langfristig) und optionale Content-Briefs.

### Kernidee: retrieval ≠ citation

Eine Seite, die die Engine abruft, aber nie zitiert, ist das nützlichste Signal in den Daten. Das ist keine Content-Lücke, sondern ein Optimierungsziel. Die meisten GEO-Audits trennen das nicht.

### Datenquelle

Die Referenz-Implementierung nutzt [Peec AI](https://peec.ai) mit `get_brand_report` und `get_url_report`. Passen Sie die Datenschritte an Ihre eigene Visibility-Quelle an. Die Methodik bleibt gleich (Mandatory Pipeline, Entscheidungsmatrix, drei Deliverables), getauscht werden nur die Daten-Calls.

### Deliverables

1. `.md`: Vollanalyse mit Tabelle und Priorisierung
2. `.tsv`: paste-fertig für Google Sheets
3. `.xlsx`: Fallback

---

## Author

Eugen Ullrich, [eullrich.com](https://eullrich.com). GEO and AI-visibility consulting.

## License

[MIT](./LICENSE), © 2026 Eugen Ullrich
