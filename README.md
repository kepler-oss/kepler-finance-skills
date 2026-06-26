# Kepler Finance Skills

Claude [Agent Skills](https://www.anthropic.com/news/skills) that guide Claude to use the
**Kepler MCP server** for financial questions about public companies.

[Kepler](https://kepler.ai) is a financial research agent that reads primary sources — SEC filings
(XBRL), earnings-call transcripts, and market data — and returns citation-backed answers and
spreadsheet models. This skill teaches Claude *when* to reach for Kepler and *how* to drive its
asynchronous, multi-tool workflow well.

## Skills in this repository

| Skill | Description |
|-------|-------------|
| [`kepler-financial-research`](./kepler-financial-research/SKILL.md) | Answer questions about public companies — financials, earnings, SEC filings, valuation, M&A, markets — and build financial models, using the Kepler MCP server. Reads primary sources and returns citation-backed answers and spreadsheet workbooks. |

## What the skill covers

- **When to invoke** — company/ticker financials, earnings & filings, valuation, M&A, source
  availability, and model/statement builds.
- **The two front doors** — instant `lookup_company` availability checks vs. full
  `run_financial_research` runs.
- **The async pattern** — kick off → `get_run_result` → `continue_waiting` until done, with
  `is_run_done` for one-shot polling.
- **Writing precise requests** — company, periods, structure, units.
- **Working with results** — preserving citations, and pulling full workbook data as CSV via
  `get_workbook_data`.
- **Stateful follow-ups** — `continue_research` to refine models and drill into sources without
  losing context.

## Prerequisite

The [Kepler MCP server](https://kepler.ai) must be connected to your Claude client (Claude.ai,
Claude Desktop, or Claude Code). The skill orchestrates that server's tools; it does not replace it.

## License

MIT
