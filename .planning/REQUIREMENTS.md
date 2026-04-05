# Requirements — TradingAgents Pro v1

## v1 Requirements

### Foundation & Output

- [ ] **FOUND-01**: User can run analysis on a single ticker and receive a conviction score (1-10) with time horizon (days/weeks/months)
- [ ] **FOUND-02**: Every analysis produces structured JSON output alongside Rich CLI tables
- [ ] **FOUND-03**: Each run's output is persisted to disk with timestamp for later comparison
- [ ] **FOUND-04**: User can run in verbose mode to see full reasoning walkthrough per agent (analyst reports, debate transcript, trader reasoning)
- [ ] **FOUND-05**: Before a batch run, system estimates token cost and shows it; after run, shows actual cost

### Data Sources

- [ ] **DATA-01**: News Analyst and Social Media Analyst use Exa semantic search instead of yfinance news, returning cited URLs with every claim
- [ ] **DATA-02**: News Analyst can use Perplexity API for real-time web research with source citations
- [ ] **DATA-03**: Fundamentals Analyst can use Firecrawl to scrape SEC filings (10-K, 10-Q, 8-K) and earnings transcripts
- [ ] **DATA-04**: Every recommendation includes a cited reasoning chain — each factual claim traces to a source URL or data point

### Portfolio & Batch

- [ ] **PORT-01**: User can manage a portfolio in local SQLite (add/remove positions: ticker, quantity, cost basis, date acquired)
- [ ] **PORT-02**: Portfolio context is fed into agent prompts ("you hold 100 shares of NVDA at $850")
- [ ] **PORT-03**: User can run batch analysis across a watchlist with tiered depth — quick screen all tickers, deep dive top N by conviction
- [ ] **PORT-04**: Independent tickers run in parallel for speed
- [ ] **PORT-05**: After batch analysis, system shows portfolio impact summary (sector allocation shifts, exposure changes)
- [ ] **PORT-06**: When news/analysis on one stock impacts related holdings, system flags cross-stock effects

### Prompt Engineering

- [ ] **PROMPT-01**: All analyst prompts rewritten with domain-specific expertise, structured output formats, and clear reasoning frameworks
- [ ] **PROMPT-02**: Bull/bear debate prompts sharpened — specific argument structures, evidence requirements, counterargument handling
- [ ] **PROMPT-03**: Trader and risk management prompts enhanced with decision frameworks (position sizing logic, risk/reward criteria)

### Observability

- [ ] **OBS-01**: LangSmith integration — every agent call traced with input/output, token usage, latency, and cost
- [ ] **OBS-02**: Traces grouped per analysis run for easy debugging and quality evaluation

### Signal Intelligence

- [ ] **SIG-01**: User can compare current run to historical runs for the same ticker — see how conviction and reasoning changed over time

---

## v2 (Deferred — Options Layer)

- Options chain data retrieval (yfinance option_chain)
- Options strategy output (directional calls/puts with strike/expiration)
- Vertical spread recommendations (bull call, bear put)
- IV rank/percentile context (historical IV computation)
- Unusual options activity detection (volume/OI heuristics)

## Out of Scope

- Web UI — CLI-first, structured JSON makes future UI trivial
- Broker integration / auto-execution — recommendations only, manual execution
- Complex multi-leg options (iron condors, butterflies, calendar spreads) — directional + verticals only in v2
- Real-time streaming data — on-demand or scheduled analysis, not live
- Plugin/skill architecture for agents — future aspiration, not current
- Multi-user / SaaS features — single user (Mario), no auth/permissions
- Backtesting extension — keep existing backtrader as-is, don't extend
- Crypto / forex — equities only (options in v2)
- Custom LLM fine-tuning — use strong general models with rich context
- Direct social media scraping — use Exa/Perplexity for sentiment, not Twitter/Reddit APIs

---

## Traceability

*(Updated by roadmap — maps requirements to phases)*

| Requirement | Phase |
|-------------|-------|
| FOUND-01 | — |
| FOUND-02 | — |
| FOUND-03 | — |
| FOUND-04 | — |
| FOUND-05 | — |
| DATA-01 | — |
| DATA-02 | — |
| DATA-03 | — |
| DATA-04 | — |
| PORT-01 | — |
| PORT-02 | — |
| PORT-03 | — |
| PORT-04 | — |
| PORT-05 | — |
| PORT-06 | — |
| PROMPT-01 | — |
| PROMPT-02 | — |
| PROMPT-03 | — |
| OBS-01 | — |
| OBS-02 | — |
| SIG-01 | — |

---
*Last updated: 2026-04-04 after requirements definition*
