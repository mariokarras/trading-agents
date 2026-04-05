# Domain Pitfalls

**Domain:** Multi-agent trading research platform (LangGraph) — extending with options, batch analysis, portfolio tracking
**Researched:** 2026-04-04
**Overall confidence:** HIGH (based on direct codebase analysis + domain knowledge)

---

## Critical Pitfalls

Mistakes that cause rewrites, data corruption, or fundamentally broken analysis.

### Pitfall 1: AgentState Schema Migration Breaks Every Node Silently

**Severity:** CRITICAL
**Phase:** Schema refactor (early)

**What goes wrong:** The current `AgentState(MessagesState)` TypedDict is referenced by every node in the graph — all 4 analysts, bull/bear researchers, research manager, trader, 3 risk debaters, and portfolio manager. Adding new fields (portfolio context, options data, batch metadata) seems simple, but LangGraph TypedDict states have a specific behavior: fields without defaults that aren't initialized in `Propagator.create_initial_state()` will cause silent KeyError crashes deep in node execution, not at graph compilation time. The error surfaces mid-run after burning LLM tokens on earlier nodes.

**Why it happens:** `Propagator.create_initial_state()` in `propagation.py` manually initializes every field. Adding a field to `AgentState` without updating this method — or without updating every node that spreads the state — produces runtime errors that only appear when the graph reaches the node that reads the new field.

**Warning signs:**
- KeyError in a node function that worked before the schema change
- Errors appearing only after analyst nodes complete (tokens already spent)
- Tests pass on individual nodes but fail on full graph runs

**Prevention:**
1. Write a validation function that compares `AgentState.__annotations__` against `Propagator.create_initial_state()` output keys — run it as a startup assertion
2. Add new fields with Optional types and default to `None` or empty string so existing nodes don't break
3. Create a single `StateFactory.create()` method that auto-initializes all fields from the TypedDict annotations
4. Add an integration test that runs a minimal graph propagation (mock LLM) to verify state flows end-to-end

**Detection:** Add a pre-run check: `assert set(AgentState.__annotations__.keys()) - {'messages'} <= set(initial_state.keys())`

---

### Pitfall 2: Options Data Contaminating Equity-Only Analysis Path

**Severity:** CRITICAL
**Phase:** Options Analyst integration

**What goes wrong:** When adding an Options Analyst node to the graph, the options analysis (IV rank, Greeks, put-call ratio) gets injected into `AgentState` and then leaked into the existing equity-focused agents. The bull/bear researchers, risk debaters, and portfolio manager receive options data they weren't prompted to interpret. LLMs hallucinate options-based reasoning ("the elevated IV suggests...") even when the options data is stale or the user only wanted equity analysis.

**Why it happens:** The current graph is a linear pipeline: all analyst reports feed into `curr_situation` strings that every downstream agent reads (see `reflection.py` line 49-55 and `portfolio_manager.py` line 18). There's no mechanism to scope which reports are visible to which agents. Adding `options_report` to `AgentState` means it gets concatenated into every agent's context.

**Warning signs:**
- Portfolio Manager's decision references Greeks or IV when the user only asked for equity analysis
- Options data from illiquid names (where yfinance returns stale/missing chains) produces confident-sounding but wrong options commentary
- Risk debaters argue about theta decay on a stock the user has no options position in

**Prevention:**
1. Make the Options Analyst opt-in per run, not always-on. Add `selected_analysts` support for `"options"` just like the existing analyst types
2. Gate options context injection: portfolio manager prompt should only include `options_report` when options analysis was requested AND the user holds options on that ticker
3. Add a `analysis_mode` field to state: `"equity_only"` vs `"equity_and_options"` — agents check this before incorporating options data
4. Separate the options recommendation from the equity recommendation in the final output (don't merge into one BUY/HOLD/SELL)

---

### Pitfall 3: Batch Analysis State Bleeding Between Tickers

**Severity:** CRITICAL
**Phase:** Batch analysis implementation

**What goes wrong:** Running `TradingAgentsGraph.propagate()` sequentially for multiple tickers reuses the same graph instance. The instance stores `self.curr_state`, `self.ticker`, and most critically, the five `FinancialSituationMemory` instances (`bull_memory`, `bear_memory`, etc.). Memories from NVDA analysis bleed into the AAPL analysis — the bull researcher retrieves NVDA-era reflections as "similar past situations" when analyzing AAPL.

**Why it happens:** `FinancialSituationMemory` is an in-memory BM25 index with no ticker-scoping. `add_situations()` appends globally. When `reflect_and_remember()` runs after ticker #1, those reflections persist and influence ticker #2's analysis. The BM25 retrieval in `get_memories()` matches on textual similarity, so "tech stock with high P/E" from NVDA analysis matches AAPL queries.

**Warning signs:**
- Second ticker in a batch run gets suspiciously specific "past memory" references that match the first ticker's sector/situation
- Batch results differ depending on ticker ordering (NVDA-first vs AAPL-first produces different AAPL recommendations)
- Memory accumulation makes later tickers in a batch slower (BM25 index grows)

**Prevention:**
1. For batch analysis: create a fresh `TradingAgentsGraph` instance per ticker, OR clear all memories between tickers with `memory.clear()`
2. Better: scope memories by ticker — key the BM25 index by `(ticker, situation)` so NVDA memories never match AAPL queries
3. For parallel execution: each ticker MUST get its own graph instance — shared mutable state across threads will corrupt results
4. Add a `reset_state()` method that clears `curr_state`, `ticker`, `log_states_dict`, and all memory instances

---

### Pitfall 4: yfinance option_chain() Silent Failures for Illiquid Names

**Severity:** CRITICAL
**Phase:** Options Analyst implementation

**What goes wrong:** `yfinance.Ticker.option_chain(date)` returns DataFrames with zero rows for illiquid options, returns stale last-trade prices (sometimes days old), and occasionally throws undocumented exceptions when Yahoo's backend changes. The Options Analyst receives empty or misleading data and generates confident analysis from it. A put-call ratio of 0.0 (no puts traded) looks like extreme bullish sentiment. IV of 0.0 (no quotes) looks like zero volatility.

**Why it happens:** yfinance scrapes Yahoo Finance HTML/JSON. Options data is particularly fragile because Yahoo doesn't guarantee completeness for non-high-volume chains. The current codebase has `yf_retry()` for transient failures but no validation of data quality/staleness.

**Warning signs:**
- Options DataFrames with `lastTradeDate` older than 3+ days
- Zero open interest across all strikes
- `impliedVolatility` of 0.0 on multiple strikes
- `option_chain()` returns for a date but the DataFrame has <5 rows

**Prevention:**
1. Build a data quality validator for options chains: reject if >50% of strikes have zero OI, if last trade is >2 days stale, or if IV is 0.0 across the board
2. When validation fails, return an explicit "Options data unavailable/unreliable for {ticker}" instead of passing garbage to the LLM
3. Log data quality metrics per ticker so you can identify which names need a better data source
4. For the Options Analyst prompt: include a preamble like "If options data quality is flagged as poor, state that and focus your analysis on what IS available"

---

## High-Severity Pitfalls

Mistakes that cause significant delays, cost overruns, or degraded quality.

### Pitfall 5: LLM Token Explosion in Batch Runs

**Severity:** HIGH
**Phase:** Batch analysis + cost management

**What goes wrong:** A full single-ticker run involves ~12-15 LLM calls. Naively running 20 tickers at full depth = 240-300 calls = $10-30+. But the real explosion comes from the debate rounds: with `max_debate_rounds=1` and `max_risk_discuss_rounds=1`, each ticker gets 2 bull/bear exchanges + 3 risk debate exchanges. Increasing debate rounds to 2 (seemingly small change) doubles the LLM calls in those phases. Combined with batch, costs compound multiplicatively.

**Why it happens:** The "tiered depth" concept (quick screen all, deep dive top candidates) requires two distinct graph configurations — but the current `TradingAgentsGraph` is initialized once with fixed parameters. There's no way to run a "light" version that skips debates.

**Warning signs:**
- Batch run costs exceed $10 (the stated budget target)
- Runs take >10 minutes for 20 tickers
- Debate rounds producing diminishing returns (round 2 just restates round 1)

**Prevention:**
1. Build two graph configurations: `ScreeningGraph` (analysts only, no debate, no risk — just produce reports and a quick signal) and `DeepAnalysisGraph` (full pipeline)
2. For screening: skip bull/bear debate and risk debate entirely — go from analyst reports straight to a signal extraction
3. Track token usage per phase (the codebase already has callback support in `TradingAgentsGraph.__init__`) — add a token budget that aborts the run if exceeded
4. Default `max_debate_rounds=1` for screening, `max_debate_rounds=2` only for deep dives on high-conviction candidates

---

### Pitfall 6: External API Rate Limits Causing Mid-Run Failures

**Severity:** HIGH
**Phase:** Exa/Perplexity/Firecrawl integration

**What goes wrong:** Exa's free tier is 1000 requests/month. If each News Analyst call makes 3 Exa queries (ticker news + industry news + macro context), a batch of 20 tickers burns 60 requests. Run this daily for a month = 1200+ requests, exceeding the free tier by day 17. The failure mode is an HTTP 429 mid-run — the News Analyst gets a rate limit error, returns a truncated or empty report, and the downstream debate agents make decisions on incomplete information without knowing data was missing.

**Why it happens:** The current tool abstraction (the `route_to_vendor` + `VENDOR_METHODS` pattern in `interface.py`) only handles `AlphaVantageRateLimitError` for fallback. New APIs need their own rate limit detection and fallback chains. More importantly, there's no request budget tracking across a run.

**Warning signs:**
- HTTP 429 errors in logs partway through a batch
- News reports for later tickers in a batch are shorter/emptier than earlier ones
- Monthly API usage spikes early in the month, then analysis quality degrades

**Prevention:**
1. Implement a `RateLimitBudget` tracker: before each API call, check remaining budget. If budget exhausted, fall back to yfinance news (degraded but functional)
2. Add all new API errors to the `route_to_vendor` fallback chain — Exa rate limit should fall back to Perplexity, then to yfinance
3. Pre-calculate API budget per run: `(tickers * queries_per_ticker)` and warn before starting if it will exceed monthly allocation
4. Add a `data_source_used` field to analyst reports so downstream agents know whether they're working with premium (Exa) or fallback (yfinance) data

---

### Pitfall 7: Parallel Ticker Execution with Shared Global Config

**Severity:** HIGH
**Phase:** Batch analysis with parallelism

**What goes wrong:** The `dataflows/config.py` module uses a global `_config` dict set by `set_config()`. When running tickers in parallel (threads or processes), all instances share this global. If one ticker's run modifies config mid-flight (unlikely now, but future features might), it corrupts others. More immediately: the `data_cache_dir` is shared, and concurrent yfinance downloads for different tickers can race on the same cache files.

**Why it happens:** `set_config(self.config)` in `TradingAgentsGraph.__init__` writes to a module-level global. The `load_ohlcv()` function in `stockstats_utils.py` likely reads/writes cached data files. Two threads calling `yf.Ticker("NVDA").history()` and `yf.Ticker("AAPL").history()` simultaneously through the retry wrapper could interleave in unexpected ways.

**Warning signs:**
- Garbled or truncated cache files in `dataflows/data_cache/`
- One ticker's analysis containing price data from another ticker
- Intermittent `FileNotFoundError` or `JSONDecodeError` on cache reads

**Prevention:**
1. Make config thread-local or pass it through the graph state instead of using a global
2. Use ticker-specific cache directories: `data_cache/{ticker}/` instead of flat files
3. For parallel execution: use `ProcessPoolExecutor` (separate memory spaces) rather than `ThreadPoolExecutor` (shared memory)
4. Add file locking on cache writes, or make cache files immutable (write once, read many)

---

### Pitfall 8: Portfolio Context Making Agents Biased

**Severity:** HIGH
**Phase:** Portfolio tracking integration

**What goes wrong:** When you tell agents "you hold 100 shares of NVDA at $850 cost basis, currently at $1200 (+41%)", the agents anchor on the existing position. The bull researcher becomes more bullish (confirmation bias — "we're already long, find reasons to stay"), the bear researcher pulls punches ("well, we have a large gain to protect"). The portfolio manager's "Hold" becomes the path of least resistance because selling means realizing the position. The system's independent analysis gets contaminated by position awareness.

**Why it happens:** LLMs exhibit anchoring bias. Giving them portfolio context (position size, P&L) before analysis skews their reasoning. The current system is designed to analyze a ticker from a neutral starting point — adding "you own this" fundamentally changes the framing.

**Warning signs:**
- Held positions consistently get more favorable ratings than non-held tickers
- "Sell" recommendations almost never appear for positions with large unrealized gains
- Removing portfolio context from a run changes the recommendation (hold -> sell or buy -> hold)

**Prevention:**
1. Separate analysis from portfolio context: run the full analyst pipeline WITHOUT portfolio information. Only inject portfolio context at the Portfolio Manager stage, after the research/debate is complete
2. The Portfolio Manager already considers "past memories" — portfolio context fits naturally here, not in analyst prompts
3. Add a "blind analysis" mode for validation: run without portfolio context periodically and compare recommendations
4. Frame portfolio context as constraints ("available capital: $X, current exposure: Y%") rather than anchors ("you bought at $Z and are up W%")

---

### Pitfall 9: Firecrawl SEC Filing Parsing Producing Garbage Context

**Severity:** HIGH
**Phase:** Firecrawl integration for Fundamentals Analyst

**What goes wrong:** SEC filings (10-K, 10-Q, 8-K) are enormous documents — a 10-K can be 100+ pages. Firecrawl returns raw scraped text that includes boilerplate legal disclaimers, table-of-contents formatting artifacts, and XBRL markup fragments. Feeding this wholesale into the Fundamentals Analyst's context window wastes tokens on noise and can exceed context limits, causing truncation of the actual financial data.

**Why it happens:** SEC filings on EDGAR have deeply nested HTML structures. Firecrawl's general-purpose scraping doesn't know which sections contain the financial statements vs. the legal boilerplate. The Fundamentals Analyst can't distinguish between useful data and formatting noise.

**Warning signs:**
- Fundamentals reports quoting boilerplate legal text as "analysis"
- Context window exceeded errors on the Fundamentals Analyst's LLM call
- Firecrawl burning through 500 free credits/month on large filing pages (each page = 1 credit)

**Prevention:**
1. Don't scrape full filings. Target specific sections: "Item 7: Management's Discussion and Analysis" for 10-Ks, "Item 2: Financial Statements" for 10-Qs
2. Use SEC EDGAR's full-text search API to find the filing URL, then Firecrawl only the relevant section anchors
3. Post-process scraped content: strip tables-of-contents, legal headers, XBRL tags. Keep only sections with financial numbers
4. Set a hard token budget for filing content (e.g., 4000 tokens max) — summarize before injecting into analyst context
5. Track Firecrawl credits per run — one 10-K scrape might cost 5-10 credits

---

## Moderate Pitfalls

Mistakes that cause technical debt, user confusion, or degraded experience.

### Pitfall 10: Signal Processing LLM Call Adding Latency and Cost for No Reason

**Severity:** MEDIUM
**Phase:** Any (existing issue, worse with batch)

**What goes wrong:** `SignalProcessor.process_signal()` makes an additional LLM call to extract BUY/HOLD/SELL from the Portfolio Manager's already-structured output. The Portfolio Manager is prompted to output a Rating as the first item. Using an LLM to regex-extract a keyword is expensive overkill, and in batch mode this adds N unnecessary LLM calls.

**Prevention:**
1. Replace the LLM-based signal extraction with regex: `re.search(r'\b(BUY|OVERWEIGHT|HOLD|UNDERWEIGHT|SELL)\b', signal, re.IGNORECASE)`
2. If the Portfolio Manager's output format is reliable (it's explicitly prompted), parse deterministically
3. Save the LLM call cost per ticker ($0.01-0.05 each, adds up in batch)

---

### Pitfall 11: Memory System Not Persisting Across Sessions

**Severity:** MEDIUM
**Phase:** Portfolio tracking / reflection

**What goes wrong:** `FinancialSituationMemory` stores everything in-memory (Python lists + BM25 index). When the process exits, all learned reflections are lost. The reflection system (which is a core feature of TradingAgents — "learn from past decisions") becomes useless for a daily-use tool because it restarts fresh every session.

**Warning signs:**
- Memory-dependent recommendations ("based on past experience...") only appear within a single long-running session
- No improvement in recommendation quality over time despite regular use

**Prevention:**
1. Add persistence: serialize `documents` and `recommendations` lists to a JSON or SQLite file on each `add_situations()` call
2. Load persisted memories on `FinancialSituationMemory.__init__()`
3. Scope persistence by ticker: `memories/{ticker}/bull_memory.json`
4. The SQLite portfolio database (already planned) could store memories in a `reflections` table

---

### Pitfall 12: Cross-Stock Impact Flags Creating Circular Analysis

**Severity:** MEDIUM
**Phase:** Cross-stock impact feature

**What goes wrong:** When implementing "news hits NVDA, flag related holdings (TSMC, AMD, AVGO)", there's a risk of circular triggering: NVDA news triggers TSMC re-analysis, which finds TSMC supply chain news that mentions NVDA, which triggers another NVDA flag. In batch mode with parallel execution, this could create an infinite loop of re-analysis.

**Prevention:**
1. Impact flags are informational only — they annotate existing results, they don't trigger new analysis runs
2. If you do implement re-analysis triggers, add a depth limit (max 1 hop: NVDA -> TSMC, but TSMC doesn't re-trigger NVDA)
3. Use a simple industry/sector mapping rather than dynamic news-based relationship detection for v1

---

### Pitfall 13: SQLite Concurrent Access in Parallel Batch Runs

**Severity:** MEDIUM
**Phase:** Portfolio tracking with SQLite

**What goes wrong:** SQLite handles concurrent reads fine but concurrent writes cause `database is locked` errors. If parallel ticker analysis tries to write results or update portfolio state simultaneously, writes fail silently or raise exceptions.

**Prevention:**
1. Use WAL (Write-Ahead Logging) mode: `PRAGMA journal_mode=WAL;` — allows concurrent reads with a single writer
2. Use a connection pool or a single writer thread that serializes writes
3. Alternatively: read portfolio data once before batch starts, write all results after batch completes (no mid-run writes)

---

### Pitfall 14: Conviction Scoring Without Calibration

**Severity:** MEDIUM
**Phase:** Conviction scoring feature

**What goes wrong:** Adding a 1-10 conviction score to recommendations without calibration means the LLM assigns scores inconsistently. One run gives NVDA a 7/10 BUY, another run with identical data gives it a 9/10 BUY. Users (you) can't compare scores across tickers or across time because the scale has no anchor.

**Prevention:**
1. Define explicit rubric in the Portfolio Manager's prompt: "10 = unanimous bull/bear/risk consensus with strong fundamentals + positive catalysts; 5 = mixed signals, balanced risk/reward; 1 = strong consensus against with deteriorating fundamentals"
2. Include 2-3 calibration examples in the prompt (few-shot)
3. Track scores over time and correlate with actual outcomes — adjust rubric based on what scores actually predicted
4. Consider deriving conviction from structured signals (count of bullish vs bearish factors) rather than asking the LLM for a number

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation | Severity |
|-------------|---------------|------------|----------|
| AgentState schema refactor | Fields not initialized in Propagator, silent KeyError mid-run | Validation function comparing annotations to initial state | Critical |
| Options Analyst integration | Options data leaking into equity-only analysis; stale/empty option chains treated as valid | Opt-in gating + data quality validator | Critical |
| Batch analysis | Memory bleeding between tickers; state not resetting | Fresh instances or explicit memory.clear() per ticker | Critical |
| Exa/Perplexity integration | Rate limit mid-run leaving partial data; no fallback chain | Budget tracker + fallback to yfinance | High |
| Parallel execution | Global config corruption; cache file races | Process isolation + ticker-scoped cache dirs | High |
| Portfolio tracking | Position awareness biasing "neutral" analysis | Inject portfolio context only at Portfolio Manager stage | High |
| Firecrawl for SEC filings | 100-page filings blowing context windows; credit burn | Section-targeted scraping + token budget | High |
| Conviction scoring | Inconsistent scores without calibration | Explicit rubric with few-shot examples | Medium |
| Cross-stock flags | Circular re-analysis triggers | Flags are informational only, no re-triggers | Medium |
| SQLite portfolio DB | Concurrent write locks in parallel mode | WAL mode + serialized writes | Medium |
| Memory persistence | Reflections lost on process exit | Serialize to JSON/SQLite per ticker | Medium |

## Sources

- Direct codebase analysis of TradingAgents v0.2.3 fork at `/Users/mariokarras/trading-agents/TradingAgents/`
- Key files examined: `agent_states.py`, `trading_graph.py`, `propagation.py`, `interface.py`, `memory.py`, `signal_processing.py`, `conditional_logic.py`, `portfolio_manager.py`, `fundamentals_analyst.py`, `y_finance.py`, `yfinance_news.py`, `default_config.py`
- yfinance option_chain behavior: known from yfinance documentation and community reports of delayed/missing data for low-volume names
- LangGraph TypedDict state behavior: based on LangGraph documentation for StateGraph with TypedDict schemas
- SQLite concurrency: SQLite documentation on WAL mode and concurrent access patterns
- LLM anchoring bias in financial contexts: established pattern in prompt engineering literature
