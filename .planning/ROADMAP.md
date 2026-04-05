# Roadmap: TradingAgents Pro

## Overview

TradingAgents Pro extends an existing LangGraph multi-agent trading framework with richer data sources, portfolio awareness, batch analysis, and structured output with conviction scoring. The roadmap starts with foundational schema and output changes (everything depends on these), then upgrades data sources and prompts in parallel, layers batch orchestration on top, and finishes with observability and signal tracking. The critical path is Phase 1 to Phase 4 -- the system is fully usable for daily workflows after Phase 4 completes.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Foundation** - AgentState schema refactor, structured output, conviction scoring, portfolio store, run persistence
- [ ] **Phase 2: Data Sources** - Exa semantic search, Perplexity real-time research, Firecrawl SEC filings, cited reasoning chains
- [ ] **Phase 3: Prompt Engineering** - Domain-expert analyst prompts, sharpened debate structures, enhanced decision frameworks
- [ ] **Phase 4: Batch & Portfolio** - Portfolio context injection, watchlist batch analysis, parallel execution, cross-stock flags, cost tracking
- [ ] **Phase 5: Observability & Signals** - LangSmith tracing, run grouping, historical signal comparison

## Phase Details

### Phase 1: Foundation
**Goal**: User can run single-ticker analysis and get structured, auditable output with conviction scoring, persisted to disk
**Depends on**: Nothing (first phase)
**Requirements**: FOUND-01, FOUND-02, FOUND-03, FOUND-04, PORT-01
**Success Criteria** (what must be TRUE):
  1. User runs analysis on a ticker and receives a conviction score (1-10) with time horizon (days/weeks/months) in the output
  2. Every analysis run produces both a Rich CLI table and a structured JSON file on disk with timestamp
  3. User can pass a verbose flag and see the full reasoning walkthrough per agent (analyst reports, debate transcript, trader reasoning)
  4. User can add, remove, and list portfolio positions via CLI commands, stored in local SQLite
  5. AgentState schema includes validation that catches missing fields before any LLM calls are made
**Plans**: TBD

Plans:
- [ ] 01-01: TBD
- [ ] 01-02: TBD
- [ ] 01-03: TBD

### Phase 2: Data Sources
**Goal**: Analysts pull from semantically-searched news, real-time web research, and SEC filings -- every factual claim traced to a source URL
**Depends on**: Nothing (independent of Phase 1 -- plugs into existing vendor routing layer)
**Requirements**: DATA-01, DATA-02, DATA-03, DATA-04
**Success Criteria** (what must be TRUE):
  1. News Analyst and Social Media Analyst use Exa semantic search instead of yfinance news, and every claim in their reports includes a source URL
  2. News Analyst can invoke Perplexity for real-time web research, returning cited answers about current events and analyst sentiment
  3. Fundamentals Analyst can scrape SEC filings (10-K, 10-Q, 8-K) and earnings transcripts via Firecrawl, with extracted data appearing in its report
  4. The final recommendation output includes a cited reasoning chain where each factual claim traces to a specific source URL or data point
**Plans**: TBD

Plans:
- [ ] 02-01: TBD
- [ ] 02-02: TBD

### Phase 3: Prompt Engineering
**Goal**: Every agent produces higher-quality analysis through domain-expert prompts with structured reasoning frameworks
**Depends on**: Phase 1 (needs structured output formats and extended schema)
**Requirements**: PROMPT-01, PROMPT-02, PROMPT-03
**Success Criteria** (what must be TRUE):
  1. All analyst prompts include domain-specific expertise framing, structured output format instructions, and explicit reasoning frameworks (not generic "analyze this stock" prompts)
  2. Bull/bear debate produces structured arguments with specific evidence requirements, counterargument handling, and scored conviction per argument
  3. Trader and risk management prompts use explicit decision frameworks with position sizing logic and risk/reward criteria visible in output
**Plans**: TBD

Plans:
- [ ] 03-01: TBD
- [ ] 03-02: TBD

### Phase 4: Batch & Portfolio
**Goal**: User can analyze an entire watchlist with portfolio-aware context, parallel execution, and cross-stock impact awareness
**Depends on**: Phase 1 (portfolio store, conviction scoring), Phase 2 (upgraded data for meaningful screening)
**Requirements**: PORT-02, PORT-03, PORT-04, PORT-05, PORT-06, FOUND-05
**Success Criteria** (what must be TRUE):
  1. Portfolio context (current positions with cost basis) is injected into agent prompts so recommendations account for existing holdings
  2. User can run batch analysis on a watchlist with tiered depth -- quick screen all tickers, then deep dive top N ranked by conviction
  3. Independent tickers execute in parallel, and each gets its own isolated graph instance (no memory bleeding between tickers)
  4. After batch analysis, system shows portfolio impact summary with sector allocation shifts and exposure changes
  5. When analysis on one stock impacts related holdings, system flags cross-stock effects for tickers in the portfolio/watchlist
  6. Before a batch run, system estimates token cost and displays it; after completion, shows actual cost incurred
**Plans**: TBD

Plans:
- [ ] 04-01: TBD
- [ ] 04-02: TBD
- [ ] 04-03: TBD

### Phase 5: Observability & Signals
**Goal**: Every agent call is traced for debugging and cost analysis, and users can track how signals evolve over time
**Depends on**: Phase 1 (run persistence for signal history), Phase 4 (batch runs generate meaningful trace volume)
**Requirements**: OBS-01, OBS-02, SIG-01
**Success Criteria** (what must be TRUE):
  1. Every agent call in a run is traced in LangSmith with input/output, token usage, latency, and cost visible in the LangSmith dashboard
  2. All traces from a single analysis run are grouped together, so user can view one run's complete execution in LangSmith
  3. User can compare the current run for a ticker against historical runs -- seeing how conviction score, recommendation direction, and key reasoning points changed over time
**Plans**: TBD

Plans:
- [ ] 05-01: TBD
- [ ] 05-02: TBD

## Progress

**Execution Order:**
Phases 1 and 2 can execute in parallel. Phase 3 follows Phase 1. Phase 4 follows Phases 1+2. Phase 5 follows Phases 1+4.

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation | 0/3 | Not started | - |
| 2. Data Sources | 0/2 | Not started | - |
| 3. Prompt Engineering | 0/2 | Not started | - |
| 4. Batch & Portfolio | 0/3 | Not started | - |
| 5. Observability & Signals | 0/2 | Not started | - |
