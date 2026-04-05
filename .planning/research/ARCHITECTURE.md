# Architecture Patterns

**Domain:** Multi-agent investment research platform (TradingAgents Pro extension)
**Researched:** 2026-04-04
**Overall confidence:** HIGH (based on direct codebase analysis, LangGraph documented patterns)

## Current Architecture (Baseline)

The existing TradingAgents system is a **single-ticker, single-pass LangGraph StateGraph** with this flow:

```
START
  → Market Analyst (tools: get_stock_data, get_indicators)
  → Social Media Analyst (tools: get_news)
  → News Analyst (tools: get_news, get_global_news, get_insider_transactions)
  → Fundamentals Analyst (tools: get_fundamentals, get_balance_sheet, get_cashflow, get_income_statement)
  → Bull/Bear Debate (configurable rounds via max_debate_rounds)
  → Research Manager (judge — synthesizes debate)
  → Trader (proposes BUY/HOLD/SELL)
  → Aggressive/Conservative/Neutral Risk Debate (configurable rounds via max_risk_discuss_rounds)
  → Portfolio Manager (final decision)
→ END
```

**Key architectural facts from the codebase:**

- `AgentState(MessagesState)` carries `company_of_interest`, `trade_date`, four analyst report strings, `InvestDebateState`, `RiskDebateState`, `trader_investment_plan`, `final_trade_decision`
- Each analyst is a closure over an LLM; tools are `@tool`-decorated functions routed through `dataflows/interface.py` vendor abstraction
- The `ToolNode` pattern: analyst calls tools → `ConditionalLogic.should_continue_X` checks for `tool_calls` → routes back to analyst or to `Msg Clear X` → next stage
- `FinancialSituationMemory` (BM25-based) stores past situation+recommendation pairs; used by bull/bear researchers, trader, research manager, portfolio manager
- `Reflector` runs post-decision to update memories based on actual returns
- No portfolio context, no batch support, no options — purely single-ticker analysis

---

## Recommended Architecture

### High-Level Component Map

```
┌──────────────────────────────────────────────────────────────────┐
│                        ORCHESTRATION LAYER                       │
│  BatchOrchestrator: manages watchlist, tiered depth, parallelism │
│  PortfolioContext: SQLite-backed, injected into every graph run  │
└──────────────┬────────────────────────────┬──────────────────────┘
               │                            │
    ┌──────────▼──────────┐     ┌───────────▼───────────┐
    │   QUICK SCREEN      │     │   DEEP ANALYSIS       │
    │   (per ticker)      │     │   (per ticker)        │
    │   Subset of agents  │     │   Full agent pipeline │
    │   + conviction sort │     │   + Options Analyst   │
    └──────────┬──────────┘     └───────────┬───────────┘
               │                            │
    ┌──────────▼────────────────────────────▼──────────────────────┐
    │                    SINGLE-TICKER GRAPH                        │
    │  (Extended TradingAgents StateGraph — unchanged flow shape)  │
    │                                                              │
    │  Market → Social → News → Fundamentals → [Options] →        │
    │  Bull/Bear Debate → Research Mgr → Trader →                  │
    │  Risk Debate → Portfolio Manager → Signal                    │
    └──────────────────────────┬───────────────────────────────────┘
                               │
    ┌──────────────────────────▼───────────────────────────────────┐
    │                     DATA TOOL LAYER                           │
    │  Vendor-routed tools (existing) + new source tools           │
    │  yfinance | Alpha Vantage | Exa | Perplexity | Firecrawl    │
    └──────────────────────────┬───────────────────────────────────┘
                               │
    ┌──────────────────────────▼───────────────────────────────────┐
    │                   PERSISTENCE LAYER                           │
    │  SQLite: portfolio positions, analysis history                │
    │  FinancialSituationMemory: BM25 reflection (existing)        │
    │  JSON logs: per-run state dumps (existing)                   │
    └─────────────────────────────────────────────────────────────┘
```

### Component Boundaries

| Component | Responsibility | Communicates With | New vs Existing |
|-----------|---------------|-------------------|-----------------|
| **BatchOrchestrator** | Accepts watchlist, determines depth tier per ticker, manages parallel execution, aggregates results, runs cross-stock impact analysis | PortfolioStore, SingleTickerGraph, CrossStockAnalyzer | NEW |
| **PortfolioStore** | SQLite CRUD for positions (ticker, qty, cost_basis, options_contracts). Produces `PortfolioContext` dict for injection. | BatchOrchestrator, AgentState (read-only injection) | NEW |
| **SingleTickerGraph** | The LangGraph StateGraph — extended with Options Analyst node and portfolio context in state. Same flow shape. | All agents, tool nodes, PortfolioStore (read) | MODIFIED |
| **Options Analyst** | New analyst node: fetches options chain, calculates IV rank/percentile, put-call ratio, unusual activity, Greeks. Produces `options_report`. | Options tool node (yfinance option_chain), AgentState | NEW |
| **Options-aware Risk Debate** | Existing risk debate agents receive options report in prompt context. Options expressions added to output. | AgentState (options_report field) | MODIFIED |
| **Exa Tool** | `@tool` function: semantic search for financial news with cited URLs. Replaces yfinance news in Social Media + News Analyst. | `dataflows/interface.py` vendor routing | NEW |
| **Perplexity Tool** | `@tool` function: real-time web research with citations for News Analyst. | `dataflows/interface.py` vendor routing | NEW |
| **Firecrawl Tool** | `@tool` function: scrape SEC filings, earnings transcripts. For Fundamentals Analyst. | `dataflows/interface.py` vendor routing | NEW |
| **CrossStockAnalyzer** | Post-run analysis: given all ticker results + portfolio positions, flags cross-stock impacts (sector exposure shifts, correlated holdings, industry contagion). | BatchOrchestrator, PortfolioStore | NEW |
| **SignalProcessor** | Extended to output structured JSON with conviction score, time horizon, options expression, cited sources. | Portfolio Manager output, CLI renderer | MODIFIED |
| **ConditionalLogic** | Extended with `should_continue_options` for the new analyst. | SingleTickerGraph | MODIFIED |
| **FinancialSituationMemory** | Unchanged — BM25 retrieval for past reflections. | Bull/Bear researchers, Trader, Research Mgr, Portfolio Mgr | EXISTING |
| **Reflector** | Extended to include options analyst reflection. | All memories | MODIFIED |

---

### Data Flow

#### 1. Batch Analysis Flow

```
User provides: [NVDA, AAPL, TSLA, MSFT, ...] + depth config

BatchOrchestrator:
  1. Load PortfolioContext from SQLite
  2. For each ticker (parallel via asyncio/ThreadPool):
     a. Run SingleTickerGraph(ticker, date, portfolio_context, depth="quick")
     b. Collect { ticker, conviction_score, signal, summary }
  3. Rank by conviction_score descending
  4. Select top N for deep analysis
  5. For each top-N ticker (parallel):
     a. Run SingleTickerGraph(ticker, date, portfolio_context, depth="full")
     b. Collect full analysis result
  6. Run CrossStockAnalyzer on all results + portfolio
  7. Aggregate into BatchResult with per-ticker + portfolio-level output
```

**Quick screen vs full analysis:**
- Quick screen: runs Market Analyst + Fundamentals Analyst only (skip Social, News, Options), single debate round, quick_think LLM only. Output: conviction 1-10 + signal.
- Full analysis: all analysts including Options Analyst, configurable debate rounds, deep_think LLM for managers. Output: full structured recommendation.

#### 2. Single-Ticker Graph Data Flow (Extended)

```
AgentState initialized with:
  - company_of_interest, trade_date
  - portfolio_context: { positions: [...], total_value, sector_allocation }
  - analysis_depth: "quick" | "full"
  - Empty report fields (market, sentiment, news, fundamentals, options)

Flow (depth="full"):
  Market Analyst → writes market_report
  Social Media Analyst → writes sentiment_report (uses Exa if configured)
  News Analyst → writes news_report (uses Exa + Perplexity if configured)
  Fundamentals Analyst → writes fundamentals_report (uses Firecrawl if configured)
  Options Analyst → writes options_report (yfinance option_chain)
  Bull/Bear Debate → writes investment_debate_state
  Research Manager → writes investment_plan
  Trader → writes trader_investment_plan
  Risk Debate → writes risk_debate_state (now options-aware)
  Portfolio Manager → writes final_trade_decision (structured)
```

#### 3. Portfolio Context Injection

```
PortfolioStore.get_context(ticker) → {
  "current_position": { qty, cost_basis, unrealized_pnl } | null,
  "portfolio_summary": { total_value, cash, sector_weights },
  "related_positions": [ positions in same sector/industry ],
  "options_positions": [ { contract, strike, expiry, type, qty } ]
}
```

This context is injected into AgentState at initialization. Every agent sees it in state but only the Portfolio Manager and Trader explicitly reference it in prompts. The Research Manager uses it to frame the investment plan relative to existing exposure.

#### 4. Cross-Stock Impact Flow

```
CrossStockAnalyzer receives:
  - All ticker analysis results (reports, signals)
  - Full portfolio positions
  - Industry/sector mapping (from fundamentals data)

Produces:
  - Impact flags: "NVDA downgrade may affect AMD position (same sector, 40% correlation)"
  - Sector concentration warnings
  - Portfolio allocation shift if all recommendations executed
```

This runs as a **post-graph LLM call** (not inside the StateGraph). It operates on aggregated results, not per-ticker state.

---

### AgentState Schema Changes

Current `AgentState` fields are preserved. New fields added:

```python
class AgentState(MessagesState):
    # === EXISTING (unchanged) ===
    company_of_interest: Annotated[str, "Company ticker"]
    trade_date: Annotated[str, "Trade date"]
    sender: Annotated[str, "Agent that sent this message"]
    market_report: Annotated[str, "Report from Market Analyst"]
    sentiment_report: Annotated[str, "Report from Social Media Analyst"]
    news_report: Annotated[str, "Report from News Analyst"]
    fundamentals_report: Annotated[str, "Report from Fundamentals Analyst"]
    investment_debate_state: Annotated[InvestDebateState, "Debate state"]
    investment_plan: Annotated[str, "Plan from Research Manager"]
    trader_investment_plan: Annotated[str, "Plan from Trader"]
    risk_debate_state: Annotated[RiskDebateState, "Risk debate state"]
    final_trade_decision: Annotated[str, "Final decision text"]

    # === NEW ===
    # Portfolio context (read-only, injected at init)
    portfolio_context: Annotated[dict, "Current portfolio state"]
    # Options analysis
    options_report: Annotated[str, "Report from Options Analyst"]
    # Analysis depth control
    analysis_depth: Annotated[str, "quick or full"]
    # Structured output
    structured_decision: Annotated[dict, "Structured JSON decision output"]
```

The `structured_decision` dict schema:

```python
{
    "ticker": str,
    "signal": "BUY" | "OVERWEIGHT" | "HOLD" | "UNDERWEIGHT" | "SELL",
    "conviction": int,  # 1-10
    "time_horizon": str,  # "1-2 weeks", "1-3 months", etc.
    "equity_action": str,  # "Buy 50 shares at market"
    "options_expression": str | None,  # "Buy NVDA Jan 2027 $150 call" or None
    "key_drivers": [str],  # Top 3 reasons
    "cited_sources": [{"title": str, "url": str, "relevance": str}],
    "risk_factors": [str],
    "portfolio_impact": {
        "sector_exposure_change": str,
        "position_sizing_note": str,
    }
}
```

---

### How Batch Execution Works

**Parallel per ticker with shared portfolio context (read-only).**

The PortfolioStore is read once at batch start and passed as an immutable dict into each graph run. Individual graph runs do not modify portfolio state — that remains manual. This means:

- No write contention between parallel runs
- Each ticker sees the same portfolio snapshot
- Portfolio is only mutated by the user (manual position entry via CLI)

**Implementation approach:**

```python
class BatchOrchestrator:
    def __init__(self, config, portfolio_store):
        self.config = config
        self.portfolio_store = portfolio_store

    async def run_batch(self, tickers: list[str], date: str):
        portfolio_ctx = self.portfolio_store.get_full_context()

        # Phase 1: Quick screen (parallel)
        quick_results = await asyncio.gather(*[
            self._run_single(ticker, date, portfolio_ctx, depth="quick")
            for ticker in tickers
        ])

        # Phase 2: Rank and select top N
        ranked = sorted(quick_results, key=lambda r: r["conviction"], reverse=True)
        deep_tickers = [r["ticker"] for r in ranked[:self.config["deep_dive_count"]]]

        # Phase 3: Deep analysis on top candidates (parallel)
        deep_results = await asyncio.gather(*[
            self._run_single(ticker, date, portfolio_ctx, depth="full")
            for ticker in deep_tickers
        ])

        # Phase 4: Cross-stock analysis
        cross_impact = self._analyze_cross_impact(quick_results, deep_results, portfolio_ctx)

        return BatchResult(quick=quick_results, deep=deep_results, cross_impact=cross_impact)
```

**Thread safety:** Each `TradingAgentsGraph` instance is independent (own LLM clients, own state). No shared mutable state between parallel runs. The `FinancialSituationMemory` instances are per-graph — if memory sharing is desired across batch runs, it would need a shared read-only snapshot (not write-sharing).

---

### Options Analyst Integration

The Options Analyst slots into the existing analyst chain between Fundamentals Analyst and the Bull/Bear debate.

**Graph modification in `setup.py`:**

```
... → Fundamentals Analyst → Msg Clear Fundamentals → Options Analyst → Msg Clear Options → Bull Researcher → ...
```

**Options Analyst node pattern** (follows exact same pattern as other analysts):

```python
def create_options_analyst(llm):
    def options_analyst_node(state):
        tools = [get_options_chain, get_iv_rank, get_unusual_activity]
        # ... same ChatPromptTemplate pattern as market_analyst
        # Writes: {"messages": [result], "options_report": report}
    return options_analyst_node
```

**Options tool:** A new `@tool` function `get_options_chain` in `tradingagents/agents/utils/options_tools.py` that wraps `yfinance.Ticker(symbol).option_chain()`. Registered in `_create_tool_nodes` alongside existing tools.

**Conditional skip:** When `analysis_depth == "quick"`, the Options Analyst is not included in `selected_analysts`. The existing `selected_analysts` parameter on `TradingAgentsGraph.__init__` already supports this — just omit `"options"` from the list.

---

### New Data Source Tools Integration

All three new sources integrate through the existing vendor abstraction layer in `dataflows/interface.py`.

**New vendor category additions to `TOOLS_CATEGORIES`:**

```python
"web_research": {
    "description": "Web-based research and semantic search",
    "tools": ["search_web_semantic", "search_web_realtime"]
},
"filings_data": {
    "description": "SEC filings and earnings transcripts",
    "tools": ["get_sec_filing", "get_earnings_transcript"]
}
```

**New entries in `VENDOR_METHODS`:**

```python
"search_web_semantic": {
    "exa": exa_semantic_search,      # Primary
    "yfinance": get_news_yfinance,   # Fallback (degraded)
},
"search_web_realtime": {
    "perplexity": perplexity_search,
},
"get_sec_filing": {
    "firecrawl": firecrawl_sec_filing,
},
"get_earnings_transcript": {
    "firecrawl": firecrawl_earnings_transcript,
}
```

**Tool files to create:**

| File | Tools | Used By |
|------|-------|---------|
| `agents/utils/exa_tools.py` | `search_web_semantic(query, ticker, num_results)` | News Analyst, Social Media Analyst |
| `agents/utils/perplexity_tools.py` | `search_web_realtime(query, ticker)` | News Analyst |
| `agents/utils/firecrawl_tools.py` | `get_sec_filing(ticker, filing_type)`, `get_earnings_transcript(ticker, quarter)` | Fundamentals Analyst |
| `agents/utils/options_tools.py` | `get_options_chain(ticker)`, `get_iv_rank(ticker)`, `get_unusual_activity(ticker)` | Options Analyst |

**Graceful degradation:** Each new source should catch API errors and return a degraded-but-usable response. If Exa is unavailable, fall back to yfinance news (already in the vendor routing system). If Perplexity/Firecrawl are unavailable, return "Data source unavailable" so the analyst proceeds without that signal.

---

## Patterns to Follow

### Pattern 1: Analyst Node as Closure

**What:** Every analyst is a factory function `create_X_analyst(llm) -> callable` that returns a closure. The closure reads from `state`, calls `llm.bind_tools(tools).invoke()`, and returns a partial state update.

**When:** Adding any new analyst (Options Analyst, or future additions).

**Why it works:** Consistent interface, easy to test, LangGraph just needs a callable that takes state and returns dict.

### Pattern 2: Tool Vendor Routing

**What:** `@tool` functions in `agents/utils/` call `route_to_vendor(method_name, *args)` which delegates to vendor-specific implementations via `VENDOR_METHODS` map. Fallback chain handles rate limits.

**When:** Adding any new data source tool.

**Why it matters:** Exa/Perplexity/Firecrawl all fit cleanly into this pattern. No changes to analyst code needed when swapping vendors.

### Pattern 3: Portfolio Context as Read-Only State

**What:** Portfolio data is loaded once, serialized to a dict, and injected into `AgentState` at initialization. Agents read it but never write to it.

**When:** Any agent needs portfolio awareness.

**Why:** Avoids write contention in parallel batch runs. Portfolio mutations happen outside the analysis pipeline (CLI commands).

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Shared Mutable State Across Parallel Graph Runs

**What:** Having multiple parallel ticker analyses write to the same memory or portfolio store during execution.

**Why bad:** Race conditions, inconsistent recommendations, unpredictable behavior.

**Instead:** Snapshot portfolio context once at batch start. Memory reflection happens sequentially after all runs complete (or per-ticker with independent memory instances).

### Anti-Pattern 2: Monolithic AgentState

**What:** Putting every possible field directly on AgentState, making it grow unboundedly as features are added.

**Why bad:** Every agent receives the full state on every invocation. Token waste and confusion.

**Instead:** Keep AgentState flat but disciplined. Use `portfolio_context` as a single dict field rather than 10 portfolio sub-fields. Use `structured_decision` as a single dict rather than separate fields for conviction, time_horizon, etc.

### Anti-Pattern 3: Tight Coupling Between Analysts and Data Sources

**What:** Having analyst prompt code directly reference Exa or Perplexity by name, or having tool implementations that are analyst-specific.

**Why bad:** Can't swap sources, can't reuse tools across analysts.

**Instead:** Tools are generic (`search_web_semantic`), vendor routing handles implementation, analyst prompts describe what data they need (not where it comes from).

---

## Suggested Build Order

Dependencies between components dictate this order. Each phase is independently testable.

### Phase 1: AgentState + Portfolio Store (Foundation)

- Extend `AgentState` with new fields (`portfolio_context`, `options_report`, `analysis_depth`, `structured_decision`)
- Build `PortfolioStore` (SQLite CRUD)
- Modify `Propagator.create_initial_state()` to accept and inject portfolio context
- Modify `Portfolio Manager` prompt to reference portfolio context
- **Dependency:** Nothing depends on this being done first, but everything depends on it being done. Build it first.

### Phase 2: Structured Output + Conviction Scoring

- Extend `SignalProcessor` to produce structured JSON with conviction score, time horizon
- Modify Portfolio Manager to output structured format
- Add Rich CLI table rendering for structured output
- **Dependency:** Requires Phase 1 (AgentState schema). Can run in parallel with Phase 3.

### Phase 3: New Data Source Tools (Exa, Perplexity, Firecrawl)

- Create tool files (`exa_tools.py`, `perplexity_tools.py`, `firecrawl_tools.py`)
- Register in `dataflows/interface.py` vendor routing
- Update `_create_tool_nodes` to include new tools in relevant analyst ToolNodes
- Update analyst prompts to reference new tool capabilities
- **Dependency:** Independent of Phase 1/2. Can be built in parallel. Just needs the existing codebase.

### Phase 4: Options Analyst

- Create `options_tools.py` (yfinance option_chain wrapper)
- Create `agents/analysts/options_analyst.py` (follows analyst closure pattern)
- Add `"options"` to `selected_analysts` support in `setup.py`
- Add `should_continue_options` to `ConditionalLogic`
- Wire into graph between Fundamentals and Bull/Bear debate
- Update risk debate agents to include options context
- **Dependency:** Requires Phase 1 (AgentState has `options_report` field).

### Phase 5: Batch Orchestration + Tiered Depth

- Build `BatchOrchestrator` with quick/full depth tiers
- Implement parallel execution (asyncio.gather over independent graph runs)
- Quick screen uses subset of analysts (market + fundamentals only)
- Rank by conviction, select top N for deep analysis
- **Dependency:** Requires Phase 1 (portfolio context), Phase 2 (conviction scoring for ranking).

### Phase 6: Cross-Stock Impact Analysis

- Build `CrossStockAnalyzer` as post-batch LLM call
- Industry/sector mapping from fundamentals data
- Portfolio impact summary (sector allocation shifts)
- **Dependency:** Requires Phase 5 (batch results to analyze), Phase 1 (portfolio positions).

### Phase Dependency Graph

```
Phase 1 (AgentState + Portfolio)
  ├── Phase 2 (Structured Output)
  │     └── Phase 5 (Batch Orchestration)
  │           └── Phase 6 (Cross-Stock Impact)
  ├── Phase 4 (Options Analyst)
  └── Phase 3 (Data Sources) ← independent, can start immediately
```

**Phases 1 and 3 can start simultaneously.** Phase 3 is the most independent — it only touches the tool layer and doesn't require schema changes. Phase 1 is the most critical dependency.

---

## Scalability Considerations

| Concern | 1 ticker | 20 tickers (batch) | 100 tickers |
|---------|----------|--------------------|-------------|
| LLM calls | ~12-15 | ~25 quick + ~75 deep (top 5) | Budget: quick screen only, deep top 10 |
| Execution time | 2-3 min | 5-8 min (parallel) | 10-15 min (parallel, rate limited) |
| Memory (BM25) | In-memory, trivial | Per-graph instances, ~10MB total | May need shared read-only snapshot |
| SQLite | Single read | Single read at batch start | Single read, no contention |
| API rate limits | No issue | Exa 1000/mo = 50 calls per ticker | Must budget: 5-10 Exa calls per deep ticker |

**The bottleneck at scale is LLM API rate limits and cost, not architecture.** The parallel-per-ticker model scales horizontally. SQLite is read-once. Memory is independent per graph.

---

## Sources

- Direct codebase analysis of `/Users/mariokarras/trading-agents/TradingAgents/` (HIGH confidence — primary source for all architectural claims)
- LangGraph StateGraph patterns from existing `setup.py`, `trading_graph.py`, `propagation.py` (HIGH confidence)
- yfinance `option_chain()` API: documented in yfinance PyPI page (MEDIUM confidence — API may have changed)
- Exa, Perplexity, Firecrawl API patterns: based on general knowledge of these APIs (MEDIUM confidence — should verify with Context7/official docs during implementation)
