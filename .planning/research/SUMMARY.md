# Project Research Summary

**Project:** TradingAgents v0.3 -- Multi-Agent Trading Research Platform
**Domain:** AI-powered personal investment research with options support
**Researched:** 2026-04-04
**Confidence:** MEDIUM-HIGH

## Executive Summary

TradingAgents v0.3 extends an existing LangGraph-based multi-agent trading research system with options analysis, batch watchlist screening, portfolio awareness, and upgraded data sources (Exa semantic search, Perplexity real-time web research, Firecrawl SEC filing scraping). The existing architecture -- a single-ticker, single-pass StateGraph with four analysts, bull/bear debate, risk debate, and a portfolio manager -- is sound and should be preserved. The recommended approach is to extend it incrementally: widen the AgentState schema, add new data source tools through the existing vendor routing abstraction, insert an Options Analyst node into the pipeline, and layer batch orchestration on top.

The stack choices are well-validated. The existing Python/LangGraph/yfinance foundation stays. New additions (exa-py, langchain-perplexity, firecrawl-py, py_vollib, sqlmodel) are all actively maintained and fit the existing patterns. Monthly API costs for personal use land at $19-40/mo, which is reasonable. The critical architectural insight is that batch analysis must use independent graph instances per ticker -- the in-memory BM25 memories and global config pattern mean shared state will corrupt results across tickers.

The top risks are: (1) AgentState schema changes silently breaking nodes mid-run after burning LLM tokens, (2) batch analysis memory bleeding between tickers, (3) options data from illiquid names producing confident-sounding garbage analysis, and (4) LLM token cost explosion in batch mode without tiered depth screening. All four are preventable with specific patterns identified in research -- state validation assertions, fresh graph instances per ticker, options data quality validators, and a two-tier quick-screen/deep-dive batch architecture.

## Key Findings

### Recommended Stack

The existing stack (Python 3.13, LangGraph, LangChain, yfinance, Rich, Typer, pandas, Redis) remains correct. LangGraph should be upgraded to >=1.1.3 (GA since late 2025). Six new dependencies are needed, none heavy:

**Core additions:**
- **exa-py + langchain-exa**: Semantic news search -- replaces thin yfinance news with embeddings-based search across 100+ sources. $7/1K searches.
- **langchain-perplexity**: Real-time web research with citations for the News Analyst. Sonar model at $1/M tokens.
- **firecrawl-py**: SEC filing and earnings transcript scraping. Free tier 500 credits/mo, use sparingly and only for rendered pages (use EDGAR directly for structured XBRL data).
- **py_vollib**: Black-Scholes Greeks calculation (delta, gamma, theta, vega, rho). Stable math library, correct for vanilla options.
- **sqlmodel**: Pydantic + SQLAlchemy ORM over SQLite for portfolio positions and trade history. Type-safe, serializes to JSON trivially.
- **No new infrastructure**: SQLite (stdlib) for persistence. No Postgres, no external DBs, no web UI.

**Required env vars:** `EXA_API_KEY`, `PERPLEXITY_API_KEY`, `FIRECRAWL_API_KEY`

### Expected Features

**Must have (table stakes):**
- T1: Single-ticker deep analysis (exists, preserve it)
- T3: Conviction scoring (1-10) with time horizon -- low complexity, high value
- T5: Structured JSON output alongside Rich CLI tables
- T9: Exa semantic news -- single biggest signal quality improvement
- T2: Cited reasoning chains -- falls out of Exa/Perplexity integration
- T6: Portfolio context awareness via SQLite
- T4: Watchlist batch analysis with tiered depth
- T7/T8: Options chain data retrieval and basic strategy output

**Should have (differentiators):**
- D3: IV rank/percentile context (changes options strategy selection)
- D6: Perplexity real-time web research
- D8: Reasoning walkthrough mode (formatting only, low effort)
- D10: Unusual options activity detection (heuristic-based)
- D7: Signal evolution tracking (diff historical runs)

**Defer (v2+):**
- D1: Cross-stock impact flags (high complexity, needs relationship mapping)
- D2: Vertical spread recommendations (depends on solid T8)
- D5: SEC filing analysis (high complexity, rate-limited)
- D4: Portfolio impact summary (post-batch aggregation)
- D9: Cost-aware run budgeting

**Anti-features (do not build):**
- Broker API integration / auto-execution
- Web UI (CLI + JSON is the product)
- Real-time streaming / live data
- Complex multi-leg options strategies
- Social media sentiment scraping (use Exa/Perplexity instead)
- Custom LLM fine-tuning
- Multi-user / SaaS features

### Architecture Approach

The architecture is a layered extension of the existing system: an Orchestration Layer (BatchOrchestrator + PortfolioContext) sits above the existing SingleTickerGraph, which gains an Options Analyst node and portfolio context injection. A Data Tool Layer adds Exa/Perplexity/Firecrawl through the existing vendor routing pattern. A Persistence Layer uses SQLite for portfolio and analysis history.

**Major components:**
1. **BatchOrchestrator** -- manages watchlist, tiered depth (quick screen all, deep dive top N), parallel execution via asyncio, aggregates results
2. **PortfolioStore** -- SQLite CRUD for positions/transactions/snapshots, produces read-only portfolio context dict injected into every graph run
3. **SingleTickerGraph (modified)** -- extended AgentState with portfolio_context, options_report, analysis_depth, structured_decision fields; Options Analyst node inserted between Fundamentals and Bull/Bear debate
4. **Options Analyst** -- new analyst node following the existing closure pattern; tools for options chain, IV rank, unusual activity
5. **SignalProcessor (modified)** -- outputs structured JSON with conviction, time horizon, options expression, cited sources; replaces LLM-based signal extraction with regex
6. **CrossStockAnalyzer** -- post-batch LLM call for sector exposure shifts and correlated holding flags (Phase 6, deferred)

**Key pattern: portfolio context is read-only.** Loaded once at batch start, passed as immutable dict. No write contention between parallel ticker runs. Portfolio mutations happen only through CLI commands.

### Critical Pitfalls

1. **AgentState schema migration breaks nodes silently** -- Adding fields without updating `Propagator.create_initial_state()` causes KeyError mid-run after burning tokens. Prevention: validation assertion comparing annotations to initial state keys, Optional types with defaults for new fields.

2. **Batch analysis memory bleeding between tickers** -- BM25 `FinancialSituationMemory` has no ticker scoping. NVDA reflections contaminate AAPL analysis. Prevention: fresh `TradingAgentsGraph` instance per ticker, never reuse across tickers in a batch.

3. **yfinance option_chain() silent failures** -- Returns empty/stale data for illiquid names. Zero OI looks like extreme bullish sentiment to the LLM. Prevention: data quality validator that rejects chains with >50% zero OI or stale last-trade dates.

4. **LLM token explosion in batch runs** -- 20 tickers at full depth = 240-300 LLM calls = $10-30+. Prevention: two-tier architecture -- quick screen (analysts only, no debate) for all tickers, full pipeline only for top N by conviction.

5. **Portfolio context biasing agent analysis** -- LLMs anchor on existing positions. Prevention: inject portfolio context only at Portfolio Manager stage, after independent analysis completes. Run analyst pipeline blind.

## Implications for Roadmap

### Phase 1: Foundation -- AgentState + Portfolio Store + Structured Output
**Rationale:** Everything depends on the extended AgentState schema and structured output. Portfolio store is needed for context injection. These are low-risk, high-dependency changes.
**Delivers:** Extended state schema with validation, SQLite portfolio CRUD, conviction scoring (1-10), structured JSON output, Rich CLI rendering.
**Addresses:** T3 (conviction scoring), T5 (JSON output), T6 (portfolio context), T10 (run history persistence)
**Avoids:** Pitfall 1 (schema migration) by building validation from day one. Pitfall 8 (portfolio bias) by designing context injection to target only Portfolio Manager.
**Stack:** sqlmodel, sqlite3 (stdlib), rich (existing)

### Phase 2: Data Source Upgrade -- Exa + Perplexity + Firecrawl
**Rationale:** Independent of Phase 1 schema work. Can run in parallel. Exa integration is the single biggest signal quality improvement. Plugs into existing vendor routing pattern with zero analyst code changes.
**Delivers:** Semantic news search, real-time web research, SEC filing scraping, cited sources in all reports.
**Addresses:** T9 (improved news), T2 (cited reasoning), D6 (Perplexity research), D5 (SEC filings -- basic)
**Avoids:** Pitfall 6 (rate limits) by implementing budget tracker and fallback chains. Pitfall 9 (SEC filing garbage) by targeting specific filing sections with token budgets.
**Stack:** exa-py, langchain-exa, langchain-perplexity, firecrawl-py

### Phase 3: Options Analyst
**Rationale:** Requires Phase 1 (AgentState has options_report field). New analyst node follows the exact same closure pattern as existing analysts -- well-understood integration point.
**Delivers:** Options chain data retrieval, Greeks calculation, IV rank/percentile, basic options strategy output (directional calls/puts).
**Addresses:** T7 (options chain data), T8 (options strategy output), D3 (IV rank/percentile), D10 (unusual activity detection)
**Avoids:** Pitfall 2 (options contaminating equity analysis) by making Options Analyst opt-in with analysis_mode gating. Pitfall 4 (yfinance silent failures) by building data quality validator.
**Stack:** py_vollib, yfinance (existing), numpy/scipy (existing)

### Phase 4: Batch Orchestration + Tiered Depth
**Rationale:** Requires Phase 1 (portfolio context for injection) and Phase 1's conviction scoring (for ranking tickers). This is what makes the tool usable as a daily workflow instead of a one-ticker-at-a-time toy.
**Delivers:** Watchlist batch analysis, quick screen all tickers, deep dive top N, parallel execution, cost tracking.
**Addresses:** T4 (batch analysis), D9 (cost budgeting), D8 (reasoning walkthrough mode)
**Avoids:** Pitfall 3 (memory bleeding) by using fresh graph instances per ticker. Pitfall 5 (token explosion) by implementing two-tier quick/deep architecture. Pitfall 7 (global config corruption) by using process isolation or ticker-scoped config.
**Stack:** asyncio (stdlib), no new dependencies

### Phase 5: Signal Evolution + Memory Persistence
**Rationale:** Requires run history from Phase 1 and batch results from Phase 4. Lower priority -- the tool is fully usable without this, but it makes it genuinely better over time.
**Delivers:** Signal tracking over time, conviction trend visualization, memory persistence across sessions, "what changed" diffs between runs.
**Addresses:** D7 (signal evolution tracking), Pitfall 11 (memory not persisting)
**Avoids:** Pitfall 12 (circular analysis) by keeping cross-stock flags informational only.
**Stack:** No new dependencies

### Phase 6: Cross-Stock Impact + Portfolio-Level Analysis
**Rationale:** Requires batch infrastructure from Phase 4 and portfolio data from Phase 1. Highest complexity, most speculative value. Only build after batch analysis proves useful in daily workflow.
**Delivers:** Cross-stock impact flags, sector concentration warnings, portfolio allocation shift projections.
**Addresses:** D1 (cross-stock impact), D4 (portfolio impact summary)
**Stack:** No new dependencies

### Phase Ordering Rationale

- **Phases 1 and 2 can start simultaneously** -- Phase 1 touches state/persistence, Phase 2 touches the tool layer. No overlap.
- **Phase 3 depends on Phase 1** (schema) but not Phase 2 (data sources). It can start as soon as Phase 1's AgentState changes land.
- **Phase 4 depends on Phases 1 and 2** being complete -- needs conviction scoring for ranking and upgraded data for meaningful screening.
- **Phases 5 and 6 are additive** -- the platform is fully functional after Phase 4. These add depth, not capability.
- **The critical path is: Phase 1 -> Phase 4.** Everything else is either parallel or additive.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 3 (Options Analyst):** py_vollib API surface needs verification. Unusual activity detection heuristics need tuning thresholds. yfinance option_chain edge cases for illiquid names need testing.
- **Phase 4 (Batch Orchestration):** Global config thread-safety needs investigation. FinancialSituationMemory isolation strategy needs prototyping. Token cost estimation model needs calibration.

Phases with standard patterns (skip research-phase):
- **Phase 1 (Foundation):** SQLite + sqlmodel is straightforward. AgentState extension follows existing patterns. Structured output is a schema definition exercise.
- **Phase 2 (Data Sources):** All three APIs have LangChain integrations. Vendor routing pattern already exists. This is plumbing.
- **Phase 5 (Signal Evolution):** JSON diffing and trend tracking are well-understood patterns.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | MEDIUM-HIGH | All versions verified via PyPI. API pricing verified via vendor sites. Exa semantic quality claim not personally benchmarked. |
| Features | MEDIUM | Based on domain knowledge of trading platforms (thinkorswim, Unusual Whales, OpenBB), not live web research. Feature prioritization is sound but MVP scope could shift. |
| Architecture | HIGH | Based on direct codebase analysis. Existing patterns (analyst closures, vendor routing, state propagation) are well-understood. Extension points are clear. |
| Pitfalls | HIGH | Based on direct codebase analysis of specific files (agent_states.py, memory.py, propagation.py, interface.py). Pitfalls are concrete and reproducible, not speculative. |

**Overall confidence:** MEDIUM-HIGH

### Gaps to Address

- **py_vollib API stability:** Last PyPI update not recent. The math is stable (Black-Scholes doesn't change), but the Python wrapper may have compatibility issues with Python 3.13. Validate during Phase 3 planning.
- **Unusual activity detection thresholds:** Volume/OI ratio > 3x and volume spike > 2 std dev are starting heuristics. Need tuning against real data. Start collecting data in Phase 1, tune in Phase 3.
- **Exa semantic search quality for financial queries:** Claimed superiority over keyword search not personally benchmarked. Run a comparison during Phase 2 implementation against yfinance news baseline.
- **LangGraph 1.x migration:** Current pyproject specifies >=0.4.8. Upgrading to >=1.1.3 may introduce breaking API changes. Verify migration path before Phase 1.
- **FinancialSituationMemory thread safety:** BM25 index uses Python lists internally. Concurrent reads during parallel batch runs may be safe (Python GIL), but this needs verification.
- **Firecrawl SEC EDGAR effectiveness:** General-purpose scraper may struggle with EDGAR's nested HTML. Test against specific filing types (10-K Item 7, 10-Q Item 2) during Phase 2.

## Sources

### Primary (HIGH confidence)
- Direct codebase analysis of `/Users/mariokarras/trading-agents/TradingAgents/` -- agent architecture, state schema, signal processing, data flows, memory system, configuration patterns
- LangGraph StateGraph patterns from `setup.py`, `trading_graph.py`, `propagation.py`
- PyPI version verification for all recommended packages (April 2026)

### Secondary (MEDIUM confidence)
- [Exa API Pricing](https://exa.ai/pricing) -- $7/1K searches, SDK versions
- [Perplexity API Pricing](https://docs.perplexity.ai/docs/getting-started/pricing) -- Sonar/Sonar Pro token costs, rate limits
- [Firecrawl Pricing](https://www.firecrawl.dev/pricing) -- credit model, rate limits
- [yfinance Options Documentation](https://ranaroussi.github.io/yfinance/) -- option_chain() API
- Domain knowledge of trading platforms (thinkorswim, Unusual Whales, OptionStrat, OpenBB, Quiver Quant)

### Tertiary (LOW confidence)
- Unusual activity detection heuristics -- custom thresholds, need tuning
- py_vollib Python 3.13 compatibility -- not tested
- Firecrawl effectiveness on SEC EDGAR HTML -- not tested against specific filings

---
*Research completed: 2026-04-04*
*Ready for roadmap: yes*
