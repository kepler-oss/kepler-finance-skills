---
name: kepler-financial-research
description: Answer questions about public companies — financials, earnings, SEC filings, valuation, M&A, and markets — and build financial models (income statements, 3-statement models, segment breakdowns, DCFs) using the Kepler MCP server. Kepler reads primary sources (SEC XBRL filings, earnings-call transcripts, market data) and returns citation-backed answers and spreadsheet workbooks. Use whenever the user asks about a company's or ticker's financials, why a company raised/acquired/announced something, a 10-K/10-Q/8-K or earnings call, a valuation, or wants any financial statement or model built. Also use to check source availability ("is the Q3 transcript out yet?", "when did they last file a 10-K?").
---

# Kepler: financial research on primary sources

Kepler is a financial research agent exposed over MCP. It reads **primary sources** — SEC
filings (XBRL), earnings-call transcripts, and market data — and returns **citation-backed**
answers plus spreadsheet workbooks for model requests. Every figure traces back to a source
document, so results are auditable rather than recalled from memory.

Prefer Kepler over answering financial questions yourself whenever the question is about a
**specific public company's actual numbers, filings, or events**. Your training data is stale and
uncited; Kepler reads the current filing and shows its work.

## When to use Kepler

Use Kepler when the user asks about, for any public company or ticker:

- **Financials** — "What was NVDA's gross margin last quarter?", "Apple's revenue by segment for
  FY2024", "How much debt does Boeing carry?"
- **Models / statements** — "Build a 3-statement model for Microsoft", "Income statement for TSLA,
  last 3 fiscal years", "Break out Amazon's operating income by segment"
- **Earnings & filings** — "What did the CEO say about margins on the last call?", "Summarize the
  risk factors in Meta's latest 10-K", "What changed in their 8-K?"
- **Valuation & markets** — "What's Netflix's enterprise value?", "P/E vs. peers", "How has the
  multiple moved since earnings?"
- **Deals & events** — "Why did Salesforce acquire X?", "What were the terms of the raise?"
- **Availability** — "Is the Q3 earnings call transcript available?", "When did they last file a
  10-K?", "Does Kepler cover this company?"

Do **not** spin up a research run for pure availability/coverage questions — use `lookup_company`,
which is instant (see below).

## The two front doors

| Tool | Cost | Use for |
|------|------|---------|
| `lookup_company` | **Instant**, no run | *Availability only* — does Kepler cover this company, what filings/transcripts exist, what dates/periods. Returns metadata, never content. |
| `run_financial_research` | 2–20 min background run | *Everything else* — any numbers, quotes, analysis, valuation, or a model/statement. |

`lookup_company(company, form_types?, limit?)` resolves a name, ticker, or CIK and lists the SEC
forms and transcripts on file. Reach for it first when the user's question is "do you have X?" — and
also as a cheap pre-check before a big run if you're unsure a company or period is covered.

For the **content** of any document — numbers, quotes, analysis, models — always use
`run_financial_research`. `lookup_company` returns metadata only.

## Running research: the async pattern

Runs take **2–20 minutes**, so the workflow is asynchronous. Follow this loop:

1. **Kick off** — `run_financial_research(request)` returns a `conversation_id` *immediately* while
   the run continues in the background. Hold onto that id; everything else keys off it.
2. **Wait for the result** — `get_run_result(conversation_id)` blocks and streams progress while it
   holds the connection. Finished runs return at once.
3. **Keep waiting if needed** — `get_run_result` uses bounded wait windows, so it may come back with
   status `running` after a few minutes. When it does, call `continue_waiting(conversation_id)` —
   identical blocking behavior — and repeat until the status is `completed` or `failed`. This is
   expected, not an error; just resume the wait.
4. **One-shot status check** — `is_run_done(conversation_id)` returns `{done, status}` instantly
   without blocking. Use it when you want to let the user do other things and poll later, rather than
   holding a wait open.

```
run_financial_research(request) ──▶ conversation_id
        │
        ▼
get_run_result(conversation_id) ──▶ completed?  ──▶ deliver result
        │  running
        ▼
continue_waiting(conversation_id) ──▶ (repeat until completed / failed)
```

**Set expectations with the user** when you kick off a run: tell them it's a live research run that
typically takes a few minutes, then wait on it. Don't claim you can't answer — start the run.

If the request was wrong (typo'd ticker, wrong company, or the user changed their mind), call
`cancel_run(conversation_id)`. It's recoverable: the conversation and any partial results survive,
and you can re-engage with `continue_research`.

## Writing a good request

The `request` is natural language. A precise request gets a precise answer in one run instead of a
follow-up round-trip. Include:

- **Company** — name or ticker (Kepler resolves either).
- **Periods** — fiscal years/quarters, or "last 3 fiscal years", "most recent quarter", "TTM".
- **Structure / format** — "as an income statement", "by reportable segment", "annual columns",
  "with YoY growth rows", "3-statement model".
- **Granularity / units** — "in millions", "GAAP and non-GAAP", "consolidated vs. segment".

Good: *"Build an income statement for NVDA for the last 3 fiscal years, annual columns, in millions,
with gross margin and operating margin rows."*

Thin: *"NVDA financials."* (Kepler will still answer, but you'll likely need follow-ups.)

## Working with results

A completed result carries the answer plus **inline citation hyperlinks** and the run's links
(citations view, spreadsheet, xlsx download). Preserve these — they're what make every figure
auditable. When you relay the answer:

- **Keep the citations.** Don't strip the source links; they let the user verify each number against
  the filing. Surface the xlsx/spreadsheet link for model requests so the user can open the workbook.
- **Quote figures as Kepler reported them.** Don't round away precision or restate numbers from your
  own memory — the whole point is that these come from the primary source.

### Spreadsheet workbooks

Model and statement requests produce **workbooks**. The result includes truncated markdown previews
of the tables, but for the full data use `get_workbook_data`:

- `get_workbook_data(workbook_id)` with no `sheet_name` **lists the sheets** of a multi-sheet
  workbook.
- `get_workbook_data(workbook_id, sheet_name)` returns that sheet's full cell data as **CSV**
  (RFC 4180) — use this when you need to do downstream analysis, reformat, or compute on the numbers
  beyond what the truncated preview shows.

## Follow-ups: stay in the conversation

Kepler conversations are **stateful** — the agent keeps full context across turns. For any
refinement or extension of an existing run, use `continue_research(conversation_id, message)` rather
than starting fresh. It returns a new `conversation_id` to wait on, same async pattern as above.

Use it to:

- **Refine a model** — "add FY2022", "break out by segment", "switch to non-GAAP", "add a YoY growth
  row".
- **Drill in** — "where did the SG&A number come from?", "what drove the margin change?"
- **Extend** — "now do the same for their top competitor", "add a cash-flow sheet".

Starting a new `run_financial_research` for a follow-up throws away the prior context and re-does
work. Continue the conversation instead.

## Finding earlier work

`list_recent_runs(limit?)` returns the caller's most recent Kepler conversations. Use it when the
user references past work — "the Netflix model from yesterday", "that valuation we ran earlier" — to
find the `conversation_id`, then either `continue_research` it or fetch its result with
`get_run_result` (immediate for finished runs).

## Tool quick reference

| Tool | Purpose | Blocks? |
|------|---------|---------|
| `lookup_company` | Availability/coverage check — filings & transcripts on file. Metadata only. | No (instant) |
| `run_financial_research` | Start a research run or model build. Returns `conversation_id`. | No (backgrounds) |
| `get_run_result` | Wait on a run and get the result; streams progress. | Yes (bounded) |
| `continue_waiting` | Resume waiting after `get_run_result` returned `running`. | Yes (bounded) |
| `is_run_done` | One-shot `{done, status}` check. | No (instant) |
| `continue_research` | Follow-up into an existing conversation; full context kept. | No (backgrounds) |
| `get_workbook_data` | Full sheet data as CSV (or list sheets). | No |
| `list_recent_runs` | Find earlier conversations. | No |
| `cancel_run` | Stop a running conversation (recoverable). | No |
| `read_run_state` | Full run state (status, answer, citations, sources, progress, workbooks). Powers the run panel; prefer `get_run_result` / `is_run_done` for normal use. | No |

## Worked example

> **User:** "What was Apple's revenue by segment last fiscal year, and how did Services grow?"

1. `run_financial_research("Apple revenue by reportable segment for the most recent fiscal year, with prior-year comparison and YoY growth for each segment, in millions")` → `conversation_id`.
2. Tell the user it's running (a few minutes), then `get_run_result(conversation_id)`.
3. If it returns `running`, `continue_waiting(conversation_id)` until `completed`.
4. Relay the answer **with its citations**; surface the spreadsheet link.
5. User: "Now break Services out into its sub-lines." → `continue_research(conversation_id, "Break the Services segment into its disclosed sub-categories with the same YoY growth columns")`.

## Pitfalls

- **Don't answer financial-fact questions from memory** when Kepler can read the filing. The value is
  fresh, cited numbers — bypassing Kepler defeats the purpose.
- **Don't treat a `running` status as a failure.** Bounded wait windows are normal; resume with
  `continue_waiting`.
- **Don't start a new run for a follow-up.** Use `continue_research` to keep context.
- **Don't use `lookup_company` for content.** It only lists what's on file; use
  `run_financial_research` for any numbers or analysis.
- **Don't strip citations or restate numbers.** Pass through Kepler's figures and source links intact.
