# Feature Landscape

**Domain:** AI-powered personal trading research platform with options support
**Researched:** 2026-04-04
**Overall confidence:** MEDIUM — based on domain knowledge of trading tools (thinkorswim, Unusual Whales, OptionStrat, Quiver Quant, OpenBB), the existing TradingAgents codebase, and PROJECT.md requirements. No live web research performed; features are derived from known tool capabilities and personal trading research platform patterns.

---

## Table Stakes

Features the tool must have or it produces less value than just reading Yahoo Finance manually.

| # | Feature | Why Expected | Complexity | Dependencies | Notes |
|---|---------|--------------|------------|-------------|-------|
| T1 | **Single-ticker deep analysis** | This is what TradingAgents already does. Without it, nothing else matters. Must produce a BUY/HOLD/SELL signal with reasoning. | Low (exists) | None | Already implemented in base framework. Preserve and extend, don't rewrite. |
| T2 | **Cited reasoning chains** | A recommendation without sources is just an opinion. Every claim ("revenue grew 23%", "bearish sentiment on Twitter") must trace to a data point or URL. | Med | T1 | Current framework produces prose reports but no structured citations. Exa and Perplexity return URLs natively — pipe them through. |
| T3 | **Conviction scoring (1-10) + time horizon** | "BUY" means nothing without strength and timeframe. Is this a 9/10 "back up the truck" or a 3/10 "slight lean"? Days or months? | Low | T1 | Add to signal processing output. LLM already reasons about confidence — just force structured output. |
| T4 | **Watchlist batch analysis** | Analyzing one ticker at a time is unusable for daily workflow. Must screen 10-30 tickers and rank by conviction. | Med | T1, T3 | Tiered depth: quick screen all, deep dive top N. Parallel execution for speed. |
| T5 | **Structured JSON output** | CLI tables are for humans reading in real-time. JSON is for storage, diffing, piping to future UI, and programmatic consumption. | Low | T1 | Output both Rich CLI tables and JSON. Define schema once, use everywhere. |
| T6 | **Portfolio context awareness** | "BUY NVDA" is different advice if you already hold 200 shares vs. zero. System must know your positions. | Med | T1 | SQLite for positions (ticker, qty, cost basis, date). Fed into every agent prompt as context. |
| T7 | **Options chain data retrieval** | Cannot recommend options strategies without knowing what's available — strikes, expirations, bid/ask, volume, OI. | Med | T1 | yfinance `option_chain()` for v1. Delayed data is fine for research (not execution). |
| T8 | **Basic options strategy output** | For each equity recommendation, suggest an options expression — at minimum directional calls/puts with specific strike and expiration. | High | T3, T7 | New Options Analyst agent. Must reason about IV, time to expiration, and cost relative to conviction. |
| T9 | **Improved news/research data** | yfinance news is thin — often just 3-5 headlines with no depth. Exa semantic search across 100+ sources is the single biggest signal improvement. | Med | T1 | Replace yfinance news tools in News Analyst and Social Media Analyst with Exa. |
| T10 | **Run history / output persistence** | Must be able to look at yesterday's recommendation and compare to today's. Without history, each run is ephemeral and you can't track signal evolution. | Low | T5 | Store JSON output per run with timestamp. SQLite or flat JSON files — simple is fine. |

---

## Differentiators

Features that make this tool meaningfully better than running TradingAgents vanilla or using free online screeners. Not expected, but high-value.

| # | Feature | Value Proposition | Complexity | Dependencies | Notes |
|---|---------|-------------------|------------|-------------|-------|
| D1 | **Cross-stock impact flags** | When TSMC news drops, auto-flag NVDA, AMD, AAPL as potentially impacted. Catches cascading effects that single-stock analysis misses. | High | T4, T6 | Requires industry/relationship mapping. Start with a simple manually-curated adjacency map, not a full knowledge graph. |
| D2 | **Vertical spread recommendations** | Beyond directional calls/puts: bull call spreads, bear put spreads. Defined-risk strategies that match conviction level to capital at risk. | High | T8 | Requires Options Analyst to reason about spread construction — max profit, max loss, breakeven. Only vertical spreads for v1 (no iron condors, butterflies). |
| D3 | **IV rank/percentile context** | "IV is at 85th percentile over 52 weeks" changes the options strategy entirely. High IV = sell premium, low IV = buy premium. | Med | T7 | Compute from historical IV data. yfinance has `impliedVolatility` on options. Need to store/compute rank over time. |
| D4 | **Portfolio impact summary** | After batch analysis, show: "If you act on all recommendations, your sector allocation shifts from X to Y, tech exposure goes from 40% to 55%." | Med | T4, T6 | Requires sector tagging (available from yfinance `info`). Computed post-analysis, not per-agent. |
| D5 | **SEC filing analysis (Firecrawl)** | Scrape 10-K, 10-Q, 8-K filings and earnings transcripts. Fundamentals Analyst gets primary-source data, not just aggregated ratios. | High | T1 | Firecrawl for extraction. Must handle large documents — summarize before feeding to LLM. Rate-limit aware (500 credits/mo free tier). |
| D6 | **Real-time web research (Perplexity)** | "What are analysts saying about NVDA earnings right now?" — current-hour information that no static data source has. | Med | T1 | Perplexity API as tool for News Analyst. Budget calls per run (expensive at scale). |
| D7 | **Signal evolution tracking** | "NVDA was a 7/10 BUY last week, now it's a 4/10 HOLD — here's what changed." Track conviction over time and surface what shifted. | Med | T3, T10 | Compare current run JSON to historical runs. Diff the reasoning chains, not just the score. |
| D8 | **Reasoning walkthrough mode** | Verbose mode that walks through the investment thesis step by step: "Here's what the fundamentals say... here's the sentiment... here's why bull wins the debate..." | Low | T1, T2 | The data already flows through the agent pipeline. Just format it as a readable narrative with section headers. |
| D9 | **Cost-aware run budgeting** | Before running batch analysis, estimate cost (LLM tokens + API calls) and confirm. Show actual cost after run. | Low | T4 | Track token usage per agent. Estimate based on historical averages. Prevents surprise $20 runs. |
| D10 | **Unusual options activity detection** | Flag tickers where options volume is abnormally high relative to average — potential smart money signal. | Med | T7 | Compute volume/OI ratio, compare to rolling average. Simple heuristic, not ML. |

---

## Anti-Features

Things to deliberately NOT build. Each one is tempting but actively harmful to the tool's value or maintainability.

| # | Anti-Feature | Why Avoid | What to Do Instead |
|---|--------------|-----------|-------------------|
| A1 | **Broker API integration / auto-execution** | One bug and you're executing real trades. The value of this tool is research and recommendations, not execution. Execution is a different product with different risk profile. | Display clear recommendations with specific strikes/expirations. Copy-paste into your broker manually. |
| A2 | **Web UI** | Adds massive surface area (auth, state management, hosting, real-time updates) for a single-user CLI tool. Every hour on UI is an hour not improving research quality. | CLI + structured JSON output. If you want a dashboard later, pipe JSON into Streamlit or similar — but don't build it now. |
| A3 | **Real-time streaming / live data** | Turns a research tool into a monitoring system. Different architecture, different cost model, different failure modes. Research is on-demand or scheduled, not live. | On-demand analysis with optional cron scheduling. "Analyze my watchlist every morning at 7am" is a cron job, not a stream. |
| A4 | **Complex multi-leg options strategies** | Iron condors, butterflies, calendar spreads require sophisticated Greeks modeling, margin calculation, and assignment risk analysis. Doing this poorly is worse than not doing it. | Stick to directional (long calls/puts) and vertical spreads (bull call, bear put). These cover 80% of personal options trading needs. |
| A5 | **Social media sentiment scraping** | Twitter/Reddit APIs are expensive, rate-limited, and noisy. Building your own sentiment pipeline is a project unto itself. | Use Exa to search for sentiment-rich sources. Perplexity can synthesize "what's the Reddit consensus on X." Don't scrape directly. |
| A6 | **Custom LLM fine-tuning** | Training data is stale the moment you create it. Markets change daily. A fine-tuned model that "learned" 2025 patterns is a liability in 2026. | Use strong general models (Claude, GPT) with rich context. The agents' value comes from data quality and reasoning structure, not model specialization. |
| A7 | **Backtesting extension** | TradingAgents has backtrader integration. Tempting to improve it, but backtesting AI-generated signals is fundamentally flawed — you're backtesting the LLM's current reasoning against historical data it was trained on. Circular. | Keep existing backtrader integration as-is. Don't invest in extending it. Track forward performance instead (signal evolution + actual P&L). |
| A8 | **Multi-user / SaaS features** | Auth, permissions, billing, multi-tenancy — none of this serves a single-user research tool. Building for hypothetical users steals time from building for the actual user. | Hardcode single-user assumptions everywhere. No user table, no auth, no permissions. If this ever becomes multi-user, refactor then. |
| A9 | **News/data caching beyond 24h** | Financial data is perishable. Caching price data or news beyond a trading day creates stale-data bugs that are hard to detect. "Why did it recommend BUY? Oh, it used last week's earnings." | Cache within a single run for deduplication. Cache price data for the current trading day. Expire everything else aggressively. |
| A10 | **Plugin/skill architecture for agents** | Premature abstraction. The agent roster is known and small (4 analysts + researchers + trader + risk + PM + options analyst). A plugin system adds indirection without users to justify it. | Hardcode agent definitions. If you need a new agent, add a Python file. Refactor to plugins only if you hit 15+ agents. |

---

## Feature Dependencies

```
T1 (Single-ticker analysis) ─── exists, foundation for everything
├── T2 (Cited reasoning) ─── requires Exa/Perplexity integration
├── T3 (Conviction scoring) ─── structured output from signal processing
│   └── T8 (Options strategy output) ─── needs conviction to size positions
│       └── D2 (Vertical spreads) ─── extends T8 with spread construction
├── T5 (JSON output) ─── schema definition
│   └── T10 (Run history) ─── persist JSON per run
│       └── D7 (Signal evolution) ─── diff historical runs
├── T6 (Portfolio context) ─── SQLite schema + position entry
│   ├── D4 (Portfolio impact) ─── post-analysis aggregation
│   └── D1 (Cross-stock impact) ─── requires knowing holdings
├── T7 (Options chain data) ─── yfinance option_chain()
│   ├── T8 (Options strategy output)
│   ├── D3 (IV rank/percentile) ─── historical IV computation
│   └── D10 (Unusual activity) ─── volume/OI heuristics
├── T9 (Improved news data) ─── Exa integration
│   └── D6 (Perplexity research) ─── additional data source
└── T4 (Batch analysis) ─── orchestration + parallel execution
    ├── D1 (Cross-stock impact)
    ├── D4 (Portfolio impact)
    └── D9 (Cost budgeting) ─── token tracking per agent

D5 (SEC filings) ─── independent, Firecrawl integration
D8 (Reasoning walkthrough) ─── formatting only, no new data
```

---

## MVP Recommendation

**For MVP (Phase 1-2), prioritize in this order:**

1. **T3 — Conviction scoring + time horizon** (Low complexity, high value, no new dependencies)
2. **T5 — Structured JSON output** (Low complexity, unlocks T10 and D7 later)
3. **T9 — Exa news integration** (Med complexity, single biggest signal quality improvement)
4. **T2 — Cited reasoning chains** (Med complexity, falls out naturally from Exa integration)
5. **T6 — Portfolio context** (Med complexity, changes how every recommendation reads)
6. **T4 — Batch analysis** (Med complexity, makes the tool usable as daily workflow)

**Defer to post-MVP:**

- **T7/T8 (Options):** High complexity, requires new agent. Get the equity research pipeline right first, then layer options on top.
- **D1 (Cross-stock impact):** High complexity, requires relationship mapping. Do after batch analysis proves useful.
- **D2 (Vertical spreads):** Depends on T8 being solid. Phase 3+ territory.
- **D5 (SEC filings):** High complexity, rate-limited. Nice-to-have after core pipeline is strong.
- **D3/D10 (IV rank, unusual activity):** Requires historical data accumulation. Start collecting data early but defer the features.

**Critical path:** T1 (exists) -> T3 + T5 -> T9 + T2 -> T6 -> T4 -> T7 + T8

---

## Sources

- **TradingAgents codebase** (HIGH confidence): Direct inspection of `/Users/mariokarras/trading-agents/TradingAgents/` — agent architecture, state schema, signal processing, data flows
- **PROJECT.md** (HIGH confidence): Requirements and constraints defined by user at `/Users/mariokarras/trading-agents/.planning/PROJECT.md`
- **Domain knowledge of trading platforms** (MEDIUM confidence): Based on known capabilities of thinkorswim, Unusual Whales, OptionStrat, OpenBB Terminal, Quiver Quantitative, and personal trading research workflow patterns. Not verified via live web research this session.
- **yfinance option_chain() API** (MEDIUM confidence): Known to return calls/puts DataFrames with strike, bid, ask, volume, OI, IV. Delayed data. API stability varies — training-data knowledge, not live-verified.
