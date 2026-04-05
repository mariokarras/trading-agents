# TradingAgents Pro

## What This Is

A multi-agent investment research system built as a fork of TradingAgents (Tauric Research). It extends the existing LangGraph-powered framework with richer data sources (Exa, Perplexity, Firecrawl), portfolio tracking, options support, and enhanced output with conviction scoring and cited reasoning chains. CLI-first, no web UI. For personal use by Mario Karras.

## Core Value

Every trading recommendation traces back to cited evidence — you can audit any buy/sell call and see exactly which data, news, and reasoning produced it.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Fork TradingAgents and refactor AgentState schema to support extended data (portfolio, options, batch)
- [ ] Replace yfinance news tools with Exa semantic search (cited URLs, cross-source coverage)
- [ ] Add structured JSON output alongside Rich CLI tables
- [ ] Add conviction scoring (1-10) and time horizon to every recommendation
- [ ] Add portfolio tracking via local SQLite (manually entered positions: ticker, qty, cost basis, options contracts)
- [ ] Portfolio context fed into every agent run ("you hold X shares of NVDA at $Y")
- [ ] Batch analysis across watchlist with tiered depth (quick screen all, deep dive top candidates)
- [ ] Parallel execution for independent tickers
- [ ] New Options Analyst agent: IV rank/percentile, put-call ratio, unusual activity, Greeks
- [ ] Options chain data via yfinance option_chain() for v1
- [ ] Extended signal output: equity recommendation + options expression (directional calls/puts, vertical spreads)
- [ ] Options-aware risk assessment in the risk debate
- [ ] Add Perplexity API as real-time web research tool for News Analyst
- [ ] Add Firecrawl for SEC filings (10-K, 10-Q, 8-K) and earnings transcripts for Fundamentals Analyst
- [ ] Cross-stock impact flags (industry/relationship mapping — when news hits one stock, flag related holdings)
- [ ] Portfolio impact summary after recommendations (sector allocation shift, exposure changes)

### Out of Scope

- Web UI — CLI-first, structured JSON makes future UI easy but not building one now
- Broker API integration — positions entered manually, no automated execution
- Complex options strategies (iron condors, butterflies, calendar spreads) — directional + vertical spreads only for v1
- Real-time streaming data — analysis is on-demand or scheduled, not live
- Plugin/skill architecture for agents — future aspiration, not v1
- Multi-user support — single user (Mario) for now
- Backtesting engine — TradingAgents has backtrader integration, keep it but don't extend
- Crypto or forex — equities and options only

## Context

**Base framework:** TradingAgents v0.2.3 by Tauric Research — open-source, LangGraph-based multi-agent trading framework. Python 3.13, LangChain ecosystem, supports OpenAI/Anthropic/Google/xAI/Ollama.

**Architecture being preserved:**
- LangGraph state machine: Market Analyst → Social Media Analyst → News Analyst → Fundamentals Analyst → Bull/Bear Debate → Research Manager → Trader → Risk Debate (Aggressive/Conservative/Neutral) → Portfolio Manager → Signal
- Reflection/memory system (BM25-based retrieval from past decisions)
- Multi-provider LLM support with deep_think/quick_think model tiering
- Tool abstraction layer (data vendor config for swapping yfinance/Alpha Vantage)

**What each new data source fills:**
| Source | Gap it fills | Replaces |
|--------|-------------|----------|
| Exa | Semantic search across 100+ financial media — themed queries like "TSMC supply chain risk" | yfinance news in News + Social Media Analyst |
| Perplexity | Real-time web research with citations — "what are analysts saying about NVDA earnings" | New capability for News Analyst |
| Firecrawl | SEC filings, earnings transcripts, analyst reports | New capability for Fundamentals Analyst |

**Cost reality per run:**
| Scenario | LLM Calls | Est. Time | Est. Cost |
|----------|-----------|-----------|-----------|
| Single ticker, full analysis | ~12-15 | 2-3 min | $0.50-1.50 |
| 20 tickers, quick screen | ~20-25 | 1-2 min (parallel) | $0.30-0.80 |
| 20 tickers, screen + 5 deep dives | ~80-100 | 5-8 min (parallel) | $3-8 |
| Above + Perplexity/Exa API calls | Same LLM + API | +1-2 min | +$0.50-2.00 |

## Constraints

- **Tech stack**: Python, LangGraph/LangChain — must stay compatible with TradingAgents ecosystem
- **Data sources**: yfinance for price/fundamentals (free, adequate), Exa/Perplexity/Firecrawl for research (API key required, free tiers available)
- **Options data**: yfinance option_chain() for v1 — delayed, sometimes spotty for illiquid names. Sufficient for personal use.
- **API rate limits**: Exa 1000 req/mo free, Perplexity varies by plan, Firecrawl 500 credits/mo free. Must budget calls per run.
- **Cost**: Target <$10 per full portfolio analysis run. Use quick_think model for screening, deep_think only for final recommendations.
- **yfinance reliability**: Scrapes Yahoo Finance, breaks periodically. Alpha Vantage already in codebase as fallback.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Fork TradingAgents rather than build from scratch | Solid multi-agent architecture, LangGraph state machine, memory system — months of work already done | — Pending |
| Exa over other news APIs as first data upgrade | Semantic search with citations is the highest-signal improvement — structured, multi-source, themed queries | — Pending |
| SQLite for portfolio storage | Simple, local, no server dependency. JSON would work but SQLite handles queries better for cross-stock analysis | — Pending |
| yfinance option_chain() for v1 options data | Free, already in ecosystem. Upgrade to CBOE or dedicated provider if this becomes daily-driver | — Pending |
| Tiered analysis depth for batch runs | Full deep analysis on 20 stocks is too slow/expensive. Quick screen all, deep dive top 3-5 by conviction | — Pending |
| CLI-first, structured JSON output | No UI complexity now, but JSON output makes future web UI trivial to build on top | — Pending |

---
*Last updated: 2026-04-04 after initialization*
